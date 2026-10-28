# 5. Kubernetes

[← Back to index](README.md)

**Q40. Explain Kubernetes architecture.**
Control plane: API server as the single entry point handling authentication, authorisation and admission; etcd storing all state; the scheduler placing pods on nodes; controller managers running reconcile loops; the cloud controller manager talking to AWS for load balancers and volumes. Each node runs kubelet, which starts containers and runs probes; kube-proxy, which programs Service routing; a container runtime like containerd; and a CNI plugin, on EKS the VPC CNI giving pods real VPC IPs. On EKS, AWS runs the control plane.

---

**Q41. Pod vs Deployment vs ReplicaSet vs StatefulSet vs DaemonSet?**
A Pod is one or more containers sharing a network namespace and volumes; it's ephemeral. A ReplicaSet keeps N identical pods running. A Deployment manages ReplicaSets to give rolling updates and rollbacks for stateless apps. A StatefulSet gives stable pod names and persistent volumes per replica with ordered rollout, for databases and queues. A DaemonSet runs one pod per node, for log shippers and monitoring agents.

---

**Q42. How do Services and Ingress work?**
A Service selects pods by label and gives them a stable ClusterIP and DNS name; kube-proxy programs iptables or IPVS so traffic to the ClusterIP is load-balanced to ready pod IPs tracked in EndpointSlices. NodePort exposes on every node; LoadBalancer provisions an NLB on EKS. Ingress is layer 7 routing by host and path, implemented by a controller; on EKS the AWS Load Balancer Controller turns an Ingress into an ALB with target-type IP pointing directly at pods. Gateway API is the newer standard.

---

**Q43. Liveness vs readiness vs startup probes?**
Readiness decides whether a pod receives traffic; failing removes it from Service endpoints without restarting. Liveness decides whether the container is alive; failing restarts it. Startup gates both for slow-starting apps. The classic mistake is a liveness probe that calls the database: when the DB has a blip, every pod restarts and you've turned a degradation into an outage. Liveness should be cheap and self-contained; readiness can check dependencies.

---

**Q44. Requests vs limits?**
Requests are what the scheduler reserves and determine QoS class. Limits are the ceiling: exceeding the CPU limit throttles the container; exceeding the memory limit kills it with OOMKilled. I set requests from observed usage, always set memory limits, and often skip CPU limits for latency-sensitive services to avoid throttling, enforcing fairness through namespace ResourceQuota and LimitRange instead.

---

**Q45. How do you achieve zero-downtime deployments on Kubernetes?**
Multiple replicas spread across zones, RollingUpdate with `maxUnavailable: 0` and `maxSurge: 1`, a correct readiness probe so new pods only get traffic when ready, graceful shutdown on SIGTERM with a `preStop` sleep so the endpoint is removed before the process exits, `terminationGracePeriodSeconds` matched to the ALB deregistration delay, a PodDisruptionBudget for node drains, and backward-compatible database migrations. I verify with `kubectl rollout status` and keep `kubectl rollout undo` ready.

---

**Q46. How do you inject secrets into a pod and how do you secure them?**
Reference a Secret with `env.valueFrom.secretKeyRef`, `envFrom.secretRef`, or mount it as a volume. Secrets are only base64-encoded, so on EKS I enable envelope encryption of etcd with a KMS key, restrict `get secrets` via RBAC, and prefer not to store them in git at all: External Secrets Operator or the Secrets Store CSI driver pulls from AWS Secrets Manager using the pod's IAM identity. For GitOps, Sealed Secrets or SOPS.

---

**Q47. Explain RBAC.**
Roles and ClusterRoles define rules: API groups, resources, verbs. RoleBindings and ClusterRoleBindings attach them to subjects: users, groups, or ServiceAccounts. Roles are namespaced; ClusterRoles cover cluster-wide or non-namespaced resources. Least privilege: a CI service account that needs to restart a deployment gets `patch` on deployments in that namespace, nothing on secrets. On EKS, humans map in through access entries and IAM; I verify with `kubectl auth can-i --as`.

---

**Q48. How do pods get AWS permissions on EKS?**
Not from the node role. IRSA: annotate the ServiceAccount with a role ARN; the pod gets a projected OIDC token and the role trusts the cluster's OIDC provider with a condition on the service account subject. EKS Pod Identity is the newer, simpler version: an agent on the node and an association between cluster, namespace, service account and role, no OIDC provider per cluster. Either way the node role stays minimal.

---

**Q49. What is Helm and what problems does it solve?**
Helm is the Kubernetes package manager: a chart is templated manifests plus a `values.yaml`; a release is an installed chart with specific values, tracked as Secrets in the namespace. It solves parameterisation across environments, versioned upgrades and rollbacks (`helm rollback`), and dependency management. I use `--atomic` so a failed upgrade rolls back automatically, and `helm template | kubectl apply --dry-run=server` in CI. Kustomize is the overlay-based alternative.

---

**Q50. What is an Operator?**
A custom controller plus a Custom Resource Definition that encodes how to run a specific application. You declare a `PostgresCluster` or `Kafka` resource and the operator handles provisioning, failover, backups and upgrades. Examples: Prometheus Operator, cert-manager, External Secrets, ArgoCD. It's the Kubernetes way to turn runbooks into software.

---

**Q51. How would you secure an EKS cluster?**
Private API endpoint, access entries with least privilege, audit logging to CloudWatch, secrets encryption with KMS, current version with managed add-ons patched. Nodes on Bottlerocket or a Packer-hardened AMI, IMDSv2 with hop limit 1, SSM instead of SSH, minimal node role. Workloads under Pod Security Standards `restricted`: non-root, read-only filesystem, dropped capabilities, resource limits; default-deny NetworkPolicies; image scanning in ECR and signing with cosign; admission policies via Kyverno; GuardDuty EKS protection at runtime.

---

**Q52. HPA vs VPA vs Cluster Autoscaler vs Karpenter?**
HPA scales pod replicas on CPU, memory or custom metrics. VPA adjusts requests and limits. Cluster Autoscaler scales existing node groups through ASGs. Karpenter provisions right-sized nodes directly from pending pod requirements, consolidates underused nodes, and handles Spot well; EKS Auto Mode bundles it. Pods scale with HPA, nodes with Karpenter.

---

**Q53. A pod is in CrashLoopBackOff. How do you debug it?**
`kubectl describe pod` for events and the last termination reason; `kubectl logs --previous` for the crashed container's output. Typical causes: missing env or secret, wrong command, port already bound, failing dependency at startup, OOMKilled when the exit code is 137. Fix the manifest, apply, watch `rollout status`.

---

**Q54. Pods are Running but the Service returns 503. Where do you look?**
`kubectl get endpoints` for the Service. Empty endpoints means either the selector doesn't match pod labels or the readiness probe is failing, which shows as `0/1 Ready`. `describe pod` shows the probe error: wrong path, wrong port, or the app bound to localhost. If endpoints are populated, check the Ingress or ALB target group health, the `targetPort`, and security groups between ALB and pods.

---

[← Back to index](README.md)
