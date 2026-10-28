# Refresher – Kubernetes (coding-round topic)

## 1. Architecture in two minutes

Control plane: **kube-apiserver** (front door, authn/authz/admission), **etcd** (state), **scheduler** (pod → node), **controller-manager** (reconcile loops), **cloud-controller-manager** (LBs, volumes). On EKS AWS runs all of this.

Node: **kubelet** (runs pods, probes), **kube-proxy** (Service → pod routing via iptables/IPVS), **container runtime** (containerd), **CNI** (VPC CNI on EKS: pods get VPC IPs).

Everything is a declarative object reconciled by a controller: you write desired state, the controller drives actual state toward it.

## 2. Core objects

| Object | Purpose | Key fields |
|---|---|---|
| Pod | 1+ containers sharing network/storage; ephemeral | `containers[]`, `volumes`, `serviceAccountName`, `nodeSelector/affinity`, `tolerations` |
| ReplicaSet | keeps N identical pods | managed by Deployment; don't create directly |
| Deployment | stateless apps; rollouts/rollbacks | `replicas`, `selector`, `template`, `strategy` |
| StatefulSet | stable identity + storage (DBs, Kafka) | `serviceName`, `volumeClaimTemplates`, ordered rollout |
| DaemonSet | one pod per node (agents, CNI, log shippers) | tolerations for control-plane/tainted nodes |
| Job / CronJob | run-to-completion / scheduled | `backoffLimit`, `completions`, `schedule`, `concurrencyPolicy` |
| Service | stable virtual IP + DNS for pods | `type: ClusterIP\|NodePort\|LoadBalancer\|ExternalName`, `selector`, `ports[].port/targetPort` |
| Ingress | L7 HTTP routing via a controller | `ingressClassName`, `rules[].host/paths`, `tls` |
| ConfigMap | non-secret config | `data`, consumed as env or volume |
| Secret | base64-encoded (not encrypted by default) | `type: Opaque\|kubernetes.io/tls\|dockerconfigjson`, `stringData` for plain text |
| Namespace | logical isolation; RBAC/quota boundary | ResourceQuota, LimitRange |
| ServiceAccount | pod identity | IRSA / Pod Identity annotation on EKS |
| PV / PVC / StorageClass | storage | EBS CSI (RWO), EFS CSI (RWX); `reclaimPolicy`, `volumeBindingMode: WaitForFirstConsumer` |
| HPA | scale replicas on metrics | needs metrics-server; CPU/memory/custom |
| PDB | limit voluntary disruption | `minAvailable` / `maxUnavailable` |
| NetworkPolicy | pod firewall | needs CNI support (VPC CNI with network policy enabled, Calico, Cilium) |

## 3. Manifests you must write from memory

```yaml
apiVersion: v1
kind: Namespace
metadata: { name: payments }
---
apiVersion: v1
kind: ConfigMap
metadata: { name: app-config, namespace: payments }
data:
  LOG_LEVEL: "info"
  DB_HOST: "payments-db.cluster-abc.ap-south-1.rds.amazonaws.com"
---
apiVersion: v1
kind: Secret
metadata: { name: app-secrets, namespace: payments }
type: Opaque
stringData:                      # plain text here; k8s stores base64
  DB_PASSWORD: "change-me"
  API_KEY: "abc123"
# or: kubectl create secret generic app-secrets -n payments --from-literal=DB_PASSWORD=x --dry-run=client -o yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments-api
  namespace: payments
  labels: { app: payments-api }
spec:
  replicas: 3
  revisionHistoryLimit: 5
  selector:
    matchLabels: { app: payments-api }          # MUST match template labels
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxSurge: 25%, maxUnavailable: 0 }
  template:
    metadata:
      labels: { app: payments-api, version: v1.4.2 }
    spec:
      serviceAccountName: payments-api
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        fsGroup: 10001
        seccompProfile: { type: RuntimeDefault }
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector: { matchLabels: { app: payments-api } }
      containers:
        - name: api
          image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/payments-api:1.4.2   # never :latest
          imagePullPolicy: IfNotPresent
          ports:
            - { name: http, containerPort: 8080 }
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef: { name: app-secrets, key: DB_PASSWORD }
            - name: LOG_LEVEL
              valueFrom:
                configMapKeyRef: { name: app-config, key: LOG_LEVEL }
            - name: POD_NAME
              valueFrom: { fieldRef: { fieldPath: metadata.name } }
          envFrom:                                   # bulk inject
            - configMapRef: { name: app-config }
            - secretRef: { name: app-secrets }
          resources:
            requests: { cpu: 250m, memory: 256Mi }
            limits:   { memory: 512Mi }              # CPU limit optional; memory limit always
          startupProbe:
            httpGet: { path: /healthz, port: http }
            failureThreshold: 30
            periodSeconds: 2
          readinessProbe:
            httpGet: { path: /ready, port: http }
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet: { path: /healthz, port: http }
            periodSeconds: 10
            failureThreshold: 3
          lifecycle:
            preStop:
              exec: { command: ["sh", "-c", "sleep 5"] }   # let endpoints drain
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: { drop: ["ALL"] }
          volumeMounts:
            - { name: tmp, mountPath: /tmp }
            - { name: config, mountPath: /etc/app, readOnly: true }
      terminationGracePeriodSeconds: 30
      volumes:
        - name: tmp
          emptyDir: {}
        - name: config
          configMap: { name: app-config }
---
apiVersion: v1
kind: Service
metadata: { name: payments-api, namespace: payments }
spec:
  type: ClusterIP
  selector: { app: payments-api }
  ports:
    - { name: http, port: 80, targetPort: http, protocol: TCP }
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: payments-api
  namespace: payments
  annotations:
    alb.ingress.kubernetes.io/scheme: internal
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /ready
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-south-1:123456789012:certificate/...
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
spec:
  ingressClassName: alb
  rules:
    - host: payments.internal.idfc
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: payments-api, port: { number: 80 } } }
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: payments-api, namespace: payments }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: payments-api }
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 60 } }
  behavior:
    scaleDown: { stabilizationWindowSeconds: 300 }
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: payments-api, namespace: payments }
spec:
  minAvailable: 2
  selector: { matchLabels: { app: payments-api } }
```

## 4. Secrets – what interviewers probe

- Base64 is encoding, not encryption. etcd encryption at rest: EKS envelope-encrypts with a KMS key (enable at cluster creation / `encryption_config`).
- Mount as env (`secretKeyRef`, `envFrom.secretRef`) or as volume (file per key; auto-updates on change, env does not).
- Better: **External Secrets Operator** or **Secrets Store CSI Driver** pulling from AWS Secrets Manager/SSM so git never has secrets; **Sealed Secrets** / SOPS for GitOps.
- RBAC: restrict `get secrets`; audit `kubectl get secret -o yaml`.
- Verify injection: `kubectl exec deploy/payments-api -- env | grep DB_`, `kubectl get secret app-secrets -o jsonpath='{.data.DB_PASSWORD}' | base64 -d`.

## 5. Probes, resources, QoS

- **startup**: for slow boots; disables liveness/readiness until it passes.
- **readiness**: fail → removed from Service endpoints (no traffic), not restarted. Use for dependency warm-up and graceful drain.
- **liveness**: fail → container restarted. Must be cheap and self-contained (no DB calls).
- Probe types: `httpGet`, `tcpSocket`, `exec`, `grpc`.
- **QoS**: Guaranteed (requests == limits for all), Burstable, BestEffort (none). Eviction order under node pressure: BestEffort → Burstable → Guaranteed.
- CPU limit → CFS throttling; memory over limit → OOMKilled (exit 137). `kubectl describe pod` shows `Last State: Terminated Reason: OOMKilled`.
- LimitRange sets defaults per namespace; ResourceQuota caps the namespace.

## 6. Rollouts

```bash
kubectl apply -f deploy.yaml --dry-run=client -o yaml     # schema check locally
kubectl apply -f deploy.yaml --dry-run=server             # admission/webhooks too
kubectl diff -f deploy.yaml
kubectl apply -f deploy.yaml
kubectl rollout status deploy/payments-api -n payments --timeout=180s
kubectl rollout history deploy/payments-api -n payments
kubectl rollout undo deploy/payments-api -n payments [--to-revision=3]
kubectl rollout pause|resume deploy/payments-api
kubectl set image deploy/payments-api api=…/payments-api:1.4.3 -n payments   # imperative; prefer git change
kubectl annotate deploy/payments-api kubernetes.io/change-cause="bump to 1.4.3"
```

Strategies: RollingUpdate (default), Recreate (downtime; for singletons), blue/green (two Deployments, switch Service selector or Ingress weight), canary (Argo Rollouts / Flagger / ALB weighted target groups).

Zero-downtime checklist: readiness probe correct, `maxUnavailable: 0`, graceful SIGTERM handling, `preStop` sleep, PDB, ALB `deregistration_delay` ≤ grace period, multiple replicas across AZs, DB migrations backwards-compatible.

## 7. Networking

- Pod IPs flat and routable cluster-wide (VPC CNI: real VPC IPs; watch ENI/IP limits per instance type; prefix delegation to increase density).
- Service ClusterIP is virtual; kube-proxy DNATs to pod IPs. DNS: `svc.ns.svc.cluster.local` via CoreDNS. Headless Service (`clusterIP: None`) returns pod IPs (StatefulSets).
- NodePort 30000–32767 on every node. LoadBalancer on EKS → NLB via AWS LB Controller (annotation `service.beta.kubernetes.io/aws-load-balancer-type: external`).
- Ingress controllers: AWS LB Controller (ALB, IngressClass `alb`), ingress-nginx, Traefik. **Gateway API** (HTTPRoute) is the successor.
- NetworkPolicy default deny then allow per namespace/app; needs CNI support (enable `enableNetworkPolicy` in VPC CNI).
- Service mesh (Istio, Linkerd, App Mesh deprecated): mTLS, retries, traffic splitting, telemetry.

## 8. RBAC and identity

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payments-api
  namespace: payments
  # IRSA: annotations: { eks.amazonaws.com/role-arn: arn:aws:iam::123:role/payments-api }
  # EKS Pod Identity: no annotation; create aws_eks_pod_identity_association(cluster, ns, sa, role)
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role                       # namespaced; ClusterRole for cluster-wide / non-namespaced
metadata: { name: config-reader, namespace: payments }
rules:
  - apiGroups: [""]              # core
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { name: payments-api-config-reader, namespace: payments }
subjects:
  - { kind: ServiceAccount, name: payments-api, namespace: payments }
roleRef: { kind: Role, name: config-reader, apiGroup: rbac.authorization.k8s.io }
```

- Human access on EKS: **access entries** (`aws_eks_access_entry` + `aws_eks_access_policy_association`, e.g. `AmazonEKSClusterAdminPolicy`) replaced the `aws-auth` ConfigMap. `kubectl auth can-i get pods --as system:serviceaccount:payments:payments-api`.
- Admission: Pod Security Admission labels on namespace (`pod-security.kubernetes.io/enforce: restricted`), OPA Gatekeeper / Kyverno policies (no `:latest`, required labels, no privileged).
- Audit logs → CloudWatch (`enabled_cluster_log_types = ["api","audit","authenticator"]`).

## 9. Helm

```bash
helm repo add eks https://aws.github.io/eks-charts && helm repo update
helm search repo eks/aws-load-balancer-controller
helm install alb eks/aws-load-balancer-controller -n kube-system --set clusterName=prod --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller
helm upgrade --install payments ./charts/payments -n payments -f values-prod.yaml --atomic --timeout 5m   # --atomic rolls back on failure
helm template payments ./charts/payments -f values-prod.yaml | kubectl apply --dry-run=server -f -
helm diff upgrade ... (plugin)   helm history payments   helm rollback payments 3   helm uninstall payments
helm lint ./charts/payments   helm package   helm dependency update
```

Chart layout: `Chart.yaml`, `values.yaml`, `templates/` with `{{ .Values.x }}`, `_helpers.tpl`, `NOTES.txt`, `tests/`. Release state stored as Secrets in the namespace. Values precedence: `values.yaml` < `-f file` < `--set`.

Kustomize alternative: `base/` + `overlays/prod/kustomization.yaml` with patches; `kubectl apply -k`.

## 10. Operators and CRDs

Operator = CRD (new kind, e.g. `Kafka`, `PostgresCluster`) + controller that reconciles it. Encodes operational knowledge (backup, failover, upgrade). Examples: Prometheus Operator, cert-manager, ExternalDNS, External Secrets, ArgoCD, Strimzi, CloudNativePG. Build with Kubebuilder/Operator SDK or Go controller-runtime. `kubectl get crd`, `kubectl explain prometheus.spec`.

## 11. Scaling

- HPA (pods, needs metrics-server; custom metrics via Prometheus Adapter / KEDA for SQS depth etc.).
- VPA (right-size requests; don't mix with HPA on CPU).
- Cluster Autoscaler (ASG-based) vs **Karpenter** (provisions right-sized nodes directly, consolidates, Spot-aware; EKS Auto Mode bundles it).
- EKS managed node groups vs Fargate (serverless pods; no DaemonSets, no privileged) vs Auto Mode.

## 12. Troubleshooting flow

```bash
kubectl get pods -n ns -o wide                 # status: Pending / ContainerCreating / CrashLoopBackOff / ImagePullBackOff / OOMKilled / Evicted
kubectl describe pod POD -n ns                 # Events at bottom – 90% of answers
kubectl logs POD -c api --previous             # crashed container's last logs
kubectl get events -n ns --sort-by=.lastTimestamp
kubectl exec -it POD -- sh ; env ; nc -zv svc 80 ; nslookup svc
kubectl get endpoints svc -n ns                # empty → selector/readiness mismatch
kubectl top pods/nodes                         # metrics-server
kubectl debug -it POD --image=busybox --target=api   # ephemeral debug container
kubectl get pod POD -o yaml | kubectl neat (plugin)
kubectl explain deployment.spec.strategy       # field docs offline
```

| Symptom | Usual cause |
|---|---|
| Pending | insufficient CPU/mem, no node matches selector/affinity/taint, PVC unbound, VPC CNI out of IPs |
| ImagePullBackOff | wrong tag, private registry auth (ECR IAM on node role), typo in repo |
| CrashLoopBackOff | app exits: bad env/secret, missing config file, port in use; check `logs --previous` |
| OOMKilled | memory limit too low / leak |
| Readiness failing, Service has no endpoints | wrong probe path/port, app slow, selector label mismatch |
| 502/504 from ALB | target-type ip but SG blocks node→pod? pods not ready, healthcheck path wrong, `deregistration_delay` |
| `error: You must be logged in to the server (Unauthorized)` | kubeconfig role not in access entries / aws-auth |
| Deployment stuck | `progressDeadlineSeconds` exceeded; PDB blocking; quota |
| Pods can't reach AWS API | IRSA/Pod Identity misconfigured (check `AWS_WEB_IDENTITY_TOKEN_FILE` env, trust policy `sub` condition) or no egress/endpoints |

## 13. Security checklist (say it in order)

Cluster: private endpoint, access entries least privilege, audit logs, secrets KMS encryption, latest version, managed add-ons patched.
Nodes: Bottlerocket or hardened AMI (Packer!), IMDSv2 hop limit 1, SSM not SSH, node role minimal (pods use Pod Identity).
Workloads: Pod Security `restricted`, non-root, read-only FS, drop caps, resource limits, no `hostNetwork/hostPath`, NetworkPolicy default-deny, image scanning + signing (ECR + cosign), pinned digests, admission policies (Kyverno).
Supply chain: SBOM, Trivy in CI, private ECR with pull-through cache.
Runtime: GuardDuty EKS protection, Falco.
