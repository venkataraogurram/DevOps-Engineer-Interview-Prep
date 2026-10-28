# 18. Kubernetes – advanced and scenario questions

[← Back to index](README.md)

**Q169. Walk through what happens from `kubectl apply` of a Deployment to a running pod.**
kubectl sends the manifest to the API server; authentication (IAM via aws-iam-authenticator on EKS), RBAC authorisation, admission controllers (mutating like injecting sidecars, validating like Pod Security or Kyverno), then it's persisted to etcd. The Deployment controller sees a new Deployment and creates a ReplicaSet; the ReplicaSet controller creates Pod objects with no node. The scheduler filters nodes by resources, taints, affinity, then scores and binds the pod to a node. That node's kubelet sees the binding, asks the CNI for an IP, pulls the image via containerd, mounts volumes and secrets, starts init containers then app containers, and runs probes. The endpoint controller adds the ready pod IP to the Service's EndpointSlice; kube-proxy programs the rule.

---

**Q170. Explain the EKS upgrade procedure and the risks.**
Read the release notes for API removals (`kubectl deprecations` plugin or Pluto to find deprecated APIs in manifests and Helm releases). Upgrade order: control plane one minor version at a time, then managed add-ons (vpc-cni, coredns, kube-proxy) to versions compatible with the new control plane, then node groups via managed rolling update or new node groups with cordon/drain, then verify workloads. Risks: removed APIs breaking Deployments or CRDs, PDBs blocking drains, pods without readiness probes dropping traffic, add-on version skew, webhooks that fail on new API versions. Do it in non-prod, keep the control plane within the supported version window (extended support costs more).

---

**Q171. How do you debug a pod that can't reach an AWS API with IRSA?**
Check the pod has `AWS_ROLE_ARN` and `AWS_WEB_IDENTITY_TOKEN_FILE` env vars (injected by the webhook when the ServiceAccount is annotated and the pod was created after the annotation). Confirm the role's trust policy references the cluster's OIDC provider and the `sub` condition matches `system:serviceaccount:<ns>:<sa>` exactly. Check the SDK version supports web identity. Then IAM permissions themselves via CloudTrail AccessDenied. With Pod Identity instead, check the association exists for that namespace/SA, the eks-pod-identity-agent DaemonSet is running, and the role trusts `pods.eks.amazonaws.com`.

---

**Q172. HPA isn't scaling. What do you check?**
`kubectl describe hpa` for the current metric value and conditions: "unable to get metrics" means metrics-server is missing or pods have no CPU requests (utilisation is relative to requests). Target is a percentage; if pods are at 55% and target 60%, no scaling is correct. Check `maxReplicas` reached, the stabilisation window delaying scale-down, or a second HPA or KEDA ScaledObject fighting it. For custom metrics, the adapter and the metric query. For nodes, if pods are Pending after HPA scaled, the problem is the cluster autoscaler or Karpenter, not HPA.

---

**Q173. Explain Pod Security Standards and how you'd enforce them.**
Three profiles: privileged (none), baseline (blocks known escalations: hostPath, privileged, hostNetwork), restricted (also requires non-root, drops all capabilities, seccomp RuntimeDefault, read-only root encouraged). Enforce via Pod Security Admission labels on namespaces: `pod-security.kubernetes.io/enforce: restricted` with `warn` and `audit` first to see what breaks. For exceptions and richer rules (image registries, required labels, no `latest`), add Kyverno or Gatekeeper policies with audit mode, then enforce.

---

**Q174. Compare Deployment strategies with Argo Rollouts versus native.**
Native Deployment supports RollingUpdate and Recreate only. Blue/green natively means two Deployments and switching a Service selector by hand. Argo Rollouts replaces Deployment with a Rollout CRD supporting blue/green with preview services and canary with weighted steps, pauses, and automated analysis against Prometheus or CloudWatch metrics that abort and roll back on failure, integrating with the AWS LB Controller for traffic weights. Flagger is the Flux-side equivalent. Use them when the business impact of a bad release justifies the extra controller.

---

**Q175. How do you handle a stateful workload like Kafka or PostgreSQL on Kubernetes, or do you?**
First answer: in AWS, prefer managed services (MSK, RDS/Aurora) for Tier-1 data; the operational and compliance burden of running them on EKS rarely pays off. If you must: StatefulSet with volumeClaimTemplates on EBS gp3, pod anti-affinity across AZs with topology-aware volume binding, PodDisruptionBudgets, a mature operator (Strimzi, CloudNativePG) to handle failover and backups, dedicated node groups, backups to S3 with restore drills, and resource guarantees (Guaranteed QoS).

---

**Q176. Explain NetworkPolicy and a default-deny design.**
NetworkPolicy selects pods and whitelists ingress and egress by pod selector, namespace selector, or IP block; it's enforced by the CNI, so VPC CNI needs network policy enabled. Default-deny per namespace: one policy selecting all pods with empty ingress and egress rules, then explicit allows: ingress from the ingress controller namespace, egress to CoreDNS (UDP/TCP 53 in kube-system), egress to the database CIDR on 5432, egress to specific service namespaces. Roll out in audit mode or namespace by namespace; DNS is the thing everyone forgets.

---

**Q177. Nodes go NotReady periodically. Causes and investigation?**
`kubectl describe node` conditions: MemoryPressure, DiskPressure, PIDPressure, or kubelet stopped posting status. Causes: kubelet OOM because pods without limits consumed node memory (reserve with `--system-reserved`/`--kube-reserved`), disk filled with image layers or logs (set `imageGCHighThreshold`, log rotation), too many pods for the ENI IP limit, Spot interruption, or a kernel issue. Check `journalctl -u kubelet` on the node via SSM, CloudWatch node metrics, and the ASG activity history. Karpenter or managed node group health auto-replaces, but you fix the root cause.

---

**Q178. How does CoreDNS work and how do you tune it?**
CoreDNS runs as a Deployment in kube-system behind the `kube-dns` Service; pods use it via `/etc/resolv.conf` with `ndots:5` and search domains, which means an external lookup like `api.example.com` tries five cluster suffixes first, multiplying queries. Tune with `dnsConfig` options `ndots: 2` for pods that mostly call external names, or trailing dots on FQDNs. Scale CoreDNS with the cluster-proportional autoscaler, enable NodeLocal DNSCache for heavy workloads, watch `coredns_dns_request_count_total` and SERVFAIL rates, and set `cache` in the Corefile.

---

**Q179. Image pull is slow or rate-limited. Fixes?**
Use ECR in-region (pull-through cache for Docker Hub and public registries to avoid rate limits), smaller images (distroless, multi-stage), `imagePullPolicy: IfNotPresent` with immutable tags, pre-pull critical images via a DaemonSet or Bottlerocket's image cache, VPC endpoints for ECR (`ecr.api`, `ecr.dkr`, plus S3 gateway) so pulls don't go through NAT, and node-level containerd cache sizing.

---

**Q180. How do you design multi-tenancy on a shared EKS cluster for several teams?**
Namespace per team or app with ResourceQuota and LimitRange, RBAC bound to IAM Identity Center groups via access entries, Pod Security restricted, default-deny NetworkPolicy, Kyverno policies for labels and registries, separate node groups with taints for sensitive workloads, IRSA/Pod Identity so no shared AWS credentials, cost allocation by namespace labels (Kubecost or CloudWatch Container Insights), per-namespace logging and metrics scoping. Hard multi-tenancy for different security domains means separate clusters; namespaces are soft isolation.

---

**Q181. Explain how Karpenter decides what node to launch.**
It watches for Pending pods the scheduler couldn't place, batches them, and computes the cheapest instance types from the NodePool's allowed set (families, sizes, capacity type Spot/On-Demand, architectures, zones) that satisfy the pods' requests, affinities, and topology constraints, then launches directly via EC2 Fleet using the EC2NodeClass (AMI family, subnets, SGs, user data). It consolidates by replacing underused or expensive nodes when a cheaper configuration can host the same pods, respecting PDBs and `do-not-disrupt` annotations. Set disruption budgets and expiry so nodes are refreshed onto new AMIs automatically.

---

**Q182. GitOps with Argo CD or Flux: how does it change the pipeline?**
CI stops at building, scanning and pushing the image and then updates the image tag in a config repo (Kustomize overlay or Helm values) by commit. Argo CD or Flux in the cluster continuously reconciles the cluster to the repo: pull-based, so the cluster needs no inbound CI credentials, every change is a commit with author and review, drift is detected and self-healed, rollback is `git revert`, and multi-cluster promotion is directory per environment. Add Argo Image Updater or Flux image automation to automate the tag bump; approval for prod becomes a PR review on the config repo.

---

**Q183. A Job must run database migrations before the new app version starts. Design it.**
Helm pre-upgrade hook Job (or Argo Rollouts/Argo CD PreSync hook) running the migration image with the same tag as the app, `backoffLimit` low, `activeDeadlineSeconds`, idempotent migration tool (Flyway/Liquibase/Alembic) with its own lock table so parallel runs are safe, `hook-delete-policy: before-hook-creation` so reruns are clean, secrets via the same Secret or external-secrets, and the hook must succeed before the Deployment rolls. Migrations must be backward-compatible with the previous app version for safe rollback (expand/contract).

[← Back to index](README.md)
