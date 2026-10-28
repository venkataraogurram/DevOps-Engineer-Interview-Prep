# Kubernetes / kubectl cheatsheet (annotated)

[← Cheatsheet index](README.md) · Manifests and concepts: [../05-refresher-kubernetes.md](../05-refresher-kubernetes.md)

Format: command → what it does and when you reach for it. `k` = `kubectl`.

## Context and config

| Command | What it does / when to use |
|---|---|
| `aws eks update-kubeconfig --name prod --region ap-south-1 --alias prod` | Writes a kubeconfig entry that uses your IAM identity to authenticate to the EKS cluster. Run once per cluster; `--alias` gives a short context name. |
| `k config get-contexts` | Lists clusters you can talk to and marks the current one. Check this before any destructive command. |
| `k config use-context prod` | Switches cluster. |
| `k config current-context` | Prints the active context. Put it in your shell prompt to avoid running prod commands by accident. |
| `k config set-context --current --namespace=payments` | Sets a default namespace so you can drop `-n payments`. |
| `k cluster-info` | Shows API server and CoreDNS endpoints. Quick "am I connected" check. |
| `k version` | Client and server versions. Skew of more than one minor between kubectl and API server causes odd errors. |
| `k api-resources` | Lists every resource kind, short names, API group, and whether namespaced. Use to find the right `kind`/short name. |
| `k api-versions` | Lists available API groups/versions. Use before an upgrade to see if `autoscaling/v2` etc. exist. |
| `k get --raw /healthz` | Hits the API server health endpoint directly. Confirms the control plane is up when kubectl behaves oddly. |
| `alias k=kubectl; source <(kubectl completion bash)` | Shorthand and tab completion. Saves minutes per day. |
| `export KUBECONFIG=~/.kube/prod:~/.kube/dev` | Merges several kubeconfig files for the session. |
| `kubectx` / `kubens` / `k9s` / `stern` / `kubecolor` | Context switcher, namespace switcher, terminal UI, multi-pod log tailer, coloured output. Install them; interviewers like seeing you know the ecosystem. |

## Read: get, describe, explain

| Command | What it does / when to use |
|---|---|
| `k get pods -A -o wide` | All pods in all namespaces with node and IP. The first command in any incident. |
| `k get po -n ns -w` | Watches pod changes live. Use while a rollout or scale event is happening. |
| `k get all -n ns` | Common workload resources in a namespace (not literally everything: no ConfigMaps/Secrets/Ingress). |
| `k get po --show-labels` | Shows labels. Use to verify a Service selector will match. |
| `k get po -l app=api,tier!=db` | Filter by label. |
| `k get po --field-selector status.phase!=Running` | Filter by a field. Quick "what's not running" across the cluster. |
| `k get po POD -o yaml` | Full object as stored, including status. Use to see what defaults were applied and the exact state. |
| `k get po -o json \| jq ...` | Machine-readable for scripting. |
| `k get po -o name` | Just `pod/name` lines, pipe into `xargs`. |
| `k get po -o jsonpath='{.items[*].metadata.name}'` | Extract specific fields. `{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}` for tables. |
| `k get po -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,IMAGE:.spec.containers[0].image` | Your own table columns. Use for audits (which image is where). |
| `k get po --sort-by=.status.startTime` | Sort output. `--sort-by='.status.containerStatuses[0].restartCount'` finds crashloopers. |
| `k get events -n ns --sort-by=.lastTimestamp` | Namespace events in time order: scheduling failures, probe failures, pulls. Often faster than `describe`. |
| `k get events -A --field-selector type=Warning` | Only warnings cluster-wide. Good incident sweep. |
| `k describe po POD` | Human summary with Events at the bottom: this answers 80% of "why isn't my pod running". |
| `k describe node NODE` | Capacity, allocated resources, conditions (MemoryPressure, DiskPressure), taints. Use for Pending pods and NotReady nodes. |
| `k describe svc SVC` | Shows Endpoints; empty means selector mismatch or no ready pods. |
| `k describe ing ING` | Rules, backends and the controller's events (ALB creation errors show here). |
| `k explain deploy.spec.strategy` | Offline field documentation. `--recursive` prints the whole subtree. Use instead of guessing field names in the coding round. |
| `k get deploy,rs,po,svc,ep,ing,cm,secret,sa,pvc,hpa,pdb,netpol -n ns` | One shot of everything that matters in a namespace. |
| `k get nodes -o wide` | Nodes with IPs, OS, kubelet and runtime versions. Use during upgrades to confirm versions. |
| `k top nodes` / `k top pods -A --sort-by=memory` / `k top po --containers` | Live CPU/memory from metrics-server. Use to spot saturation and right-size requests. |
| `k get crd` / `k get <crd-kind> -A` | Custom resource definitions and their instances (e.g. `k get rollouts -A`). |

## Write: create, apply, edit, delete

| Command | What it does / when to use |
|---|---|
| `k apply -f f.yaml` | Declaratively create or update from a file. The default way to change anything. |
| `k apply -f dir/ -R` | Apply every manifest in a directory tree. |
| `k apply -k overlays/prod` | Apply a Kustomize overlay. |
| `k apply -f f.yaml --dry-run=client -o yaml` | Validates YAML structure locally and prints the result. First check in the coding round. |
| `k apply -f f.yaml --dry-run=server` | Sends to the API server for full validation and admission webhooks without persisting. Catches policy violations before apply. |
| `k diff -f f.yaml` | Shows what would change versus the live object. Use before applying to prod. |
| `k create deploy api --image=nginx --replicas=3 --port=80 --dry-run=client -o yaml > deploy.yaml` | Scaffold a correct manifest instead of writing from memory. Do this for Deployment, Service, ConfigMap, Secret, Job. |
| `k create svc clusterip api --tcp=80:8080 --dry-run=client -o yaml` | Scaffold a Service (port 80 → targetPort 8080). |
| `k create cm app-config --from-literal=LOG=info --from-file=app.properties --from-env-file=.env` | ConfigMap from literals, files or an env file. |
| `k create secret generic db --from-literal=PASSWORD=x` | Opaque Secret; base64 is done for you. `--dry-run=client -o yaml` to get YAML for git (then seal/encrypt it). |
| `k create secret tls web-tls --cert=c.pem --key=k.pem` | TLS secret for Ingress. |
| `k create secret docker-registry regcred --docker-server=... --docker-username=... --docker-password=...` | Image pull secret for private registries (not needed for ECR on EKS nodes). |
| `k create ns payments` / `k create sa api -n payments` | Namespace and ServiceAccount. |
| `k create job test --image=busybox -- sh -c 'echo hi'` | One-off Job. |
| `k create cronjob x --image=busybox --schedule='*/5 * * * *' -- date` | Scheduled job. |
| `k create job manual --from=cronjob/nightly` | Trigger a CronJob now. Use to test a schedule without waiting. |
| `k run tmp --rm -it --image=busybox:1.36 --restart=Never -- sh` | Throwaway pod with a shell, deleted on exit. Use for in-cluster DNS/connectivity tests. |
| `k run curl --rm -it --image=curlimages/curl --restart=Never -- curl -s http://api.payments.svc/health` | One-shot HTTP check from inside the cluster. |
| `k edit deploy api` | Opens the live object in `$EDITOR`. Fine for emergencies; the change isn't in git, so follow up with a PR. |
| `k set image deploy/api api=repo/api:1.2.4` | Change a container image: triggers a rollout. Common in pipelines; GitOps prefers a commit. |
| `k set env deploy/api LOG_LEVEL=debug` | Add/change an env var (rolls pods). Quick debugging lever. |
| `k set resources deploy/api -c api --limits=memory=512Mi` | Change requests/limits. |
| `k patch deploy api -p '{"spec":{"replicas":5}}'` | Strategic-merge patch of a field. `--type json -p '[{"op":"replace","path":"/spec/template/spec/containers/0/image","value":"..."}]'` for precise JSON patches. |
| `k label po POD tier=web --overwrite` / `k label node NODE disk=ssd` | Add labels (for selectors, nodeSelector). |
| `k annotate deploy api kubernetes.io/change-cause="v1.2.4"` | Records why; shows in `rollout history`. |
| `k scale deploy api --replicas=0` | Manual scale. `--replicas=0` is a fast "stop" without deleting. |
| `k autoscale deploy api --min=2 --max=10 --cpu-percent=60` | Creates an HPA. |
| `k delete -f f.yaml` | Delete what a file defines. |
| `k delete po POD --grace-period=0 --force` | Force delete a stuck Terminating pod. Last resort; the container may still be running on the node. |
| `k delete po -l app=api` | Delete by label (Deployment recreates them: a crude restart). |
| `k delete ns x` | Deletes the namespace and everything in it; blocks until finalizers finish. |
| `k replace --force -f f.yaml` | Delete and recreate (for immutable field changes). Causes downtime for that object. |

## Rollouts

| Command | What it does / when to use |
|---|---|
| `k rollout status deploy/api --timeout=180s` | Blocks until the rollout finishes or fails. The gate in every deploy pipeline. |
| `k rollout history deploy/api` | Lists revisions with change-cause. `--revision=3` shows that revision's template. |
| `k rollout undo deploy/api` | Roll back to the previous revision. `--to-revision=2` for a specific one. The fastest rollback in Kubernetes. |
| `k rollout restart deploy/api` | Recreates pods with the same spec. Use to pick up a changed ConfigMap/Secret consumed as env vars. |
| `k rollout pause deploy/api` / `resume` | Pause mid-rollout to inspect a canary batch, then continue. |
| `k get rs -l app=api` | Old ReplicaSets kept per `revisionHistoryLimit`; the ones with 0 replicas are rollback targets. |

## Logs, exec, debug, copy

| Command | What it does / when to use |
|---|---|
| `k logs POD` / `k logs POD -c sidecar` | Container stdout/stderr. Specify `-c` in multi-container pods. |
| `k logs -f --tail=100 POD` | Follow, starting from the last 100 lines. |
| `k logs --previous POD` | Logs of the crashed previous container. The command for CrashLoopBackOff. |
| `k logs --since=10m POD` | Time-bounded. |
| `k logs -l app=api --all-containers --prefix -f` | Aggregate logs from all pods of an app, prefixed with pod name. `stern api -n ns` does this better. |
| `k logs deploy/api` | Logs from one pod of the Deployment (random pick). |
| `k exec -it POD -- sh` | Shell inside the container. Use to check env, files, and local connectivity. Audited in a bank; prefer `debug` where possible. |
| `k exec POD -c api -- env` | Print the container's environment: verifies Secret/ConfigMap injection. |
| `k debug -it POD --image=nicolaka/netshoot --target=api` | Attaches an ephemeral debug container sharing the target's process namespace. Use when the app image has no shell (distroless). |
| `k debug node/NODE -it --image=busybox` | Pod on the node with the host filesystem mounted at `/host`. Use to read kubelet logs or disk usage without SSH. |
| `k cp POD:/tmp/heap.hprof ./heap.hprof -c api` | Copy a file out (needs `tar` in the container). |
| `k port-forward svc/api 8080:80` | Tunnel a local port to a Service or pod. Use to hit an internal service from your laptop without an Ingress. |
| `k proxy` | Local proxy to the API server; `curl localhost:8001/api/v1/...` for raw API exploration. |
| `k attach POD -c api` | Attach to the running process's stdin/stdout. Rare. |

## Nodes

| Command | What it does / when to use |
|---|---|
| `k describe no NODE \| sed -n '/Conditions/,/Events/p'` | Just the conditions block: Ready, MemoryPressure, DiskPressure, PIDPressure. |
| `k describe no NODE \| grep -A5 Allocated` | Requested vs allocatable resources. Use when pods are Pending for insufficient CPU/memory. |
| `k cordon NODE` | Mark unschedulable; existing pods stay. Step one of a manual node replacement. |
| `k drain NODE --ignore-daemonsets --delete-emptydir-data --grace-period=60` | Evict pods respecting PDBs, so they reschedule elsewhere. Step two before terminating a node. Hangs if a PDB can't be satisfied. |
| `k uncordon NODE` | Make schedulable again. |
| `k taint nodes NODE dedicated=gpu:NoSchedule` | Repel pods without a matching toleration. `dedicated-` (trailing dash) removes the taint. |
| `k get no -l eks.amazonaws.com/nodegroup=app --show-labels` | Nodes of one managed node group. |
| `k get po -A -o wide --field-selector spec.nodeName=NODE` | What's running on a node. Check before draining. |

## RBAC and auth

| Command | What it does / when to use |
|---|---|
| `k auth can-i create deployments -n payments` | Checks your own permission. |
| `k auth can-i get secrets --as=system:serviceaccount:ci:deployer -n payments` | Impersonates a ServiceAccount or user to test a Role. Use to prove least privilege in a review. |
| `k auth can-i --list -n payments --as alice` | Everything a subject can do in a namespace. |
| `k create role pod-reader --verb=get,list,watch --resource=pods -n ns` | Scaffold a Role (add `--dry-run=client -o yaml`). |
| `k create rolebinding x --role=pod-reader --serviceaccount=ns:sa -n ns` | Bind a Role to a ServiceAccount. |
| `k create clusterrolebinding x --clusterrole=view --group=developers` | Cluster-wide read-only for a group. |
| `k get clusterrolebindings -o wide \| grep cluster-admin` | Who has cluster-admin. Audit item. |
| `k create token api -n payments --duration=1h` | Mint a short-lived ServiceAccount token (1.24+ no longer auto-creates secret tokens). |
| `k whoami` | Shows the identity the API server sees you as (1.28+). On EKS pair with `aws sts get-caller-identity`. |
| `aws eks list-access-entries --cluster-name prod` | IAM principals mapped into the cluster (replaces the aws-auth ConfigMap). |
| `aws eks create-access-entry --cluster-name prod --principal-arn arn:aws:iam::123:role/Dev` | Grant an IAM role cluster access. |
| `aws eks associate-access-policy --cluster-name prod --principal-arn ... --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy --access-scope type=namespace,namespaces=dev` | Attach an EKS access policy, optionally scoped to namespaces. Use instead of writing RBAC for IAM users. |

## Secrets and ConfigMaps

| Command | What it does / when to use |
|---|---|
| `k get secret db -o jsonpath='{.data.PASSWORD}' \| base64 -d` | Decode one key. Reminder: base64 is not encryption. |
| `k get secret db -o go-template='{{range $k,$v := .data}}{{$k}}={{$v\|base64decode}}{{"\n"}}{{end}}'` | Decode all keys in one go. |
| `k create secret generic db --from-literal=PASSWORD=x --dry-run=client -o yaml \| kubeseal -o yaml` | Produce a SealedSecret that is safe to commit (Bitnami Sealed Secrets). |
| `k create cm app-config --from-file=cfg/ -o yaml --dry-run=client \| k apply -f -` | Idempotently update a ConfigMap from files (plain `create` fails if it exists). |
| `k rollout restart deploy/api` | Needed after changing a Secret/ConfigMap used as env vars; volume-mounted ones refresh automatically within ~1 minute. |

## Storage

| Command | What it does / when to use |
|---|---|
| `k get sc` | StorageClasses; the one marked `(default)` is used by PVCs without a class. On EKS: `gp2` legacy, `gp3` via EBS CSI. |
| `k get pv` / `k get pvc -A` / `k describe pvc data-0` | Volumes and claims; `describe` Events show why a claim is Pending (no SC, zone mismatch, quota). |
| `k patch sc gp3 -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'` | Make gp3 the default class (remove the annotation from gp2). |
| `k patch pvc data -p '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'` | Expand a volume online (SC needs `allowVolumeExpansion: true`). |
| `k get volumeattachment` | Which PVs are attached to which nodes. Use when a pod is stuck ContainerCreating on "Multi-Attach error". |

## Networking

| Command | What it does / when to use |
|---|---|
| `k get svc -A` | Services with ClusterIP/NodePort/LB address. |
| `k get ep SVC` / `k get endpointslices -l kubernetes.io/service-name=api` | Pod IPs behind a Service. Empty = selector mismatch or no Ready pods. |
| `k get ing -A` / `k describe ing api` | Ingress rules and the address the controller assigned (ALB DNS). |
| `k get ingressclass` | Which controllers exist (`alb`, `nginx`). A missing class means the Ingress is ignored. |
| `k get netpol -A` / `k describe netpol deny-all` | Network policies in force. First check when pods suddenly can't talk. |
| `k run dns --rm -it --image=busybox:1.36 --restart=Never -- nslookup api.payments.svc.cluster.local` | In-cluster DNS test. |
| `k get po -n kube-system -l k8s-app=kube-dns` / `k logs -n kube-system -l k8s-app=kube-dns` | CoreDNS health and logs. |
| `k get po -n kube-system -l k8s-app=aws-node -o wide` | VPC CNI pods, one per node. Pending pods with "failed to assign an IP" point here. |
| `k describe ds aws-node -n kube-system \| grep -i -E 'ENABLE_PREFIX\|NETWORK_POLICY'` | Checks prefix delegation (more IPs per node) and network policy enforcement flags. |
| `k get targetgroupbindings -A` | AWS LB Controller's mapping of Services to ALB/NLB target groups. |
| `k -n kube-system logs deploy/aws-load-balancer-controller \| tail` | Why an Ingress didn't get an ALB (missing subnet tags, IAM, cert ARN). |

Service types: ClusterIP (internal only, default) · NodePort (30000–32767 on every node) · LoadBalancer (NLB via LB Controller on EKS) · ExternalName (CNAME to outside) · Headless `clusterIP: None` (returns pod IPs; StatefulSets).
DNS names: `<svc>.<ns>.svc.cluster.local`; pods `<ip-dashed>.<ns>.pod.cluster.local`.

## Scheduling fields

```yaml
nodeSelector: { disk: ssd }                                  # simple "only nodes with this label"
tolerations: [{ key: dedicated, operator: Equal, value: gpu, effect: NoSchedule }]   # allowed onto tainted nodes
affinity:
  podAntiAffinity:                                           # spread replicas: don't put two on the same node
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector: { matchLabels: { app: api } }
        topologyKey: kubernetes.io/hostname
topologySpreadConstraints:                                   # even spread across zones; softer than anti-affinity
  - { maxSkew: 1, topologyKey: topology.kubernetes.io/zone, whenUnsatisfiable: ScheduleAnyway, labelSelector: { matchLabels: { app: api } } }
priorityClassName: high                                      # preempt lower-priority pods under pressure
terminationGracePeriodSeconds: 30                            # time between SIGTERM and SIGKILL; align with LB drain
dnsPolicy: ClusterFirst                                      # default; Default = use node DNS
hostNetwork: false                                           # true shares node network namespace (avoid)
```

## Kustomize

| Command | What it does / when to use |
|---|---|
| `k kustomize overlays/prod` | Renders the overlay to stdout without applying. Review the result. |
| `k apply -k overlays/prod` | Build and apply. |
| `kustomization.yaml` keys | `resources` (bases/files) · `namespace` · `namePrefix` · `commonLabels` · `images: [{name, newTag}]` (bump tags without touching manifests) · `patches` (strategic/JSON6902) · `configMapGenerator`/`secretGenerator` (hash-suffixed names so pods roll on change) |

## Pod status decoder

| Status | Meaning → what to check |
|---|---|
| Pending | Unschedulable → `describe` Events: Insufficient cpu/memory, taint without toleration, affinity unmet, PVC unbound, CNI out of IPs. |
| ContainerCreating | Pulling image, mounting volume/Secret/ConfigMap (missing one hangs here), waiting for CNI IP. |
| ImagePullBackOff / ErrImagePull | Wrong tag/repo, registry auth, private ECR in another account → `describe` shows the registry's exact error. |
| CrashLoopBackOff | App exits repeatedly → `logs --previous`; bad config/secret, port in use, dependency down at startup. |
| OOMKilled (exit 137) | Memory over limit → raise limit or fix leak; check `describe` Last State. |
| Error | Container exited non-zero (Jobs). |
| Completed | Job finished successfully. |
| Terminating (stuck) | Finalizer, volume detach, or node gone → investigate; `--force --grace-period=0` as last resort. |
| Evicted | Node pressure (disk/memory) → `describe` message says which; fix node sizing or limits. |
| Running 0/1 | Readiness probe failing → `describe` probe events; wrong path/port, app not ready. |
| Init:0/1, Init:CrashLoopBackOff | Init container failing → `logs POD -c init-name`. |
| CreateContainerConfigError | Env references a missing ConfigMap/Secret key. |
| Unknown / NodeLost | Node NotReady → `describe node`. |

## One-liners

```bash
# Pods with restarts > 0, with counts – find the crashloopers
k get po -A -o json | jq -r '.items[] | select(any(.status.containerStatuses[]?; .restartCount>0)) | "\(.metadata.namespace)/\(.metadata.name) \([.status.containerStatuses[].restartCount]|add)"'
# Every image running in the cluster with counts – audit for :latest or old versions
k get po -A -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.image}{"\n"}{end}{end}' | sort | uniq -c | sort -rn
# Pods without resource limits – policy gap finder
k get po -A -o json | jq -r '.items[] | select(.spec.containers[].resources.limits==null) | "\(.metadata.namespace)/\(.metadata.name)"'
# Delete all Failed (Evicted) pods – housekeeping after node pressure
k get po -A --field-selector status.phase=Failed -o name | xargs -r k delete
# Force-finalize a namespace stuck Terminating – only after confirming nothing real is left
k get ns x -o json | jq '.spec.finalizers=[]' | k replace --raw /api/v1/namespaces/x/finalize -f -
# Which AZ a pod is in – for spread checks
k get po POD -o jsonpath='{.spec.nodeName}' | xargs -I{} k get no {} -o jsonpath='{.metadata.labels.topology\.kubernetes\.io/zone}'
```

## EKS CLI

| Command | What it does / when to use |
|---|---|
| `aws eks describe-cluster --name x --query 'cluster.{v:version,ep:resourcesVpcConfig.endpointPrivateAccess,oidc:identity.oidc.issuer,logs:logging}'` | Version, whether the API endpoint is private, OIDC issuer (for IRSA), enabled control-plane logs. |
| `aws eks list-nodegroups --cluster-name x` / `describe-nodegroup ... --query 'nodegroup.{v:version,ami:releaseVersion,scaling:scalingConfig,update:updateConfig}'` | Node group K8s version, AMI release, min/max/desired, `maxUnavailable` for rolling updates. |
| `aws eks update-nodegroup-version --cluster-name x --nodegroup-name app` | Rolling node replacement to the latest AMI for the cluster version. Respects PDBs; fails if drains can't complete. |
| `aws eks list-addons --cluster-name x` / `describe-addon-versions --addon-name vpc-cni --kubernetes-version 1.31` | Managed add-ons installed and which versions are compatible with a K8s version. Check before upgrading. |
| `aws eks update-addon --cluster-name x --addon-name coredns --addon-version v1.11.3-eksbuild.1 --resolve-conflicts PRESERVE` | Upgrade an add-on; PRESERVE keeps your custom config instead of overwriting. |
| `aws eks list-pod-identity-associations --cluster-name x` / `create-pod-identity-association --namespace ns --service-account sa --role-arn ...` | Map a ServiceAccount to an IAM role (Pod Identity, the successor to IRSA). |
| `aws eks update-cluster-config --name x --logging '{"clusterLogging":[{"types":["api","audit","authenticator"],"enabled":true}]}'` | Turn on control-plane logs to CloudWatch; audit log is required in regulated environments. |
| `eksctl get cluster` / `eksctl create cluster -f cluster.yaml` | Opinionated CLI for quick clusters; fine for labs, Terraform for real environments. |
| `eksctl utils associate-iam-oidc-provider --cluster x --approve` | Creates the IAM OIDC provider needed for IRSA. |

Upgrade order: control plane (one minor at a time) → add-ons (vpc-cni, coredns, kube-proxy) → node groups → verify workloads. Check deprecated APIs first with `kubent` or `pluto`.
