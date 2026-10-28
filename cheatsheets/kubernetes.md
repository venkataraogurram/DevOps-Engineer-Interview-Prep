# Kubernetes / kubectl cheatsheet

[← Cheatsheet index](README.md)

## Context and config

```bash
aws eks update-kubeconfig --name prod --region ap-south-1 --alias prod
kubectl config get-contexts; kubectl config use-context prod; kubectl config current-context
kubectl config set-context --current --namespace=payments
kubectl cluster-info; kubectl version; kubectl api-resources; kubectl api-versions
kubectl get --raw /healthz; kubectl get --raw /metrics | head
alias k=kubectl; source <(kubectl completion bash); export KUBECONFIG=~/.kube/prod:~/.kube/dev
kubectx / kubens / k9s / stern / kubecolor         # worth installing
```

## Get / describe / explain

```bash
k get pods -A -o wide; k get po -n ns -w; k get all -n ns
k get po --show-labels; k get po -l app=api,tier!=db; k get po --field-selector status.phase!=Running
k get po -o yaml; k get po -o json | jq; k get po -o name
k get po -o jsonpath='{.items[*].metadata.name}'; k get po -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}'
k get po -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,IMAGE:.spec.containers[0].image
k get po --sort-by=.status.startTime; k get po --sort-by='.status.containerStatuses[0].restartCount'
k get events -n ns --sort-by=.lastTimestamp; k get events -A --field-selector type=Warning
k describe po POD; k describe node NODE; k describe svc SVC; k describe ing ING
k explain deploy.spec.strategy; k explain pod.spec.containers.livenessProbe --recursive
k get deploy,rs,po,svc,ep,ing,cm,secret,sa,pvc,hpa,pdb,netpol -n ns
k get nodes -o wide; k top nodes; k top pods -A --sort-by=memory; k top po --containers
k get crd; k get <crd-kind> -A
```

## Create / apply / edit / delete

```bash
k apply -f f.yaml; k apply -f dir/ -R; k apply -k overlays/prod; k apply -f - <<EOF ... EOF
k apply -f f.yaml --dry-run=client -o yaml; --dry-run=server; k diff -f f.yaml
k create deploy api --image=nginx --replicas=3 --port=80 --dry-run=client -o yaml > deploy.yaml   # scaffold
k create svc clusterip api --tcp=80:8080 --dry-run=client -o yaml
k create cm app-config --from-literal=LOG=info --from-file=app.properties --from-env-file=.env
k create secret generic db --from-literal=PASSWORD=x; k create secret tls web-tls --cert=c.pem --key=k.pem; k create secret docker-registry regcred --docker-server=... --docker-username=... --docker-password=...
k create ns payments; k create sa api -n payments; k create job test --image=busybox -- sh -c 'echo hi'; k create cronjob x --image=busybox --schedule='*/5 * * * *' -- date
k create job manual --from=cronjob/nightly
k run tmp --rm -it --image=busybox:1.36 --restart=Never -- sh          # throwaway pod
k run curl --rm -it --image=curlimages/curl --restart=Never -- curl -s http://api.payments.svc/health
k edit deploy api; k set image deploy/api api=repo/api:1.2.4; k set env deploy/api LOG_LEVEL=debug; k set resources deploy/api -c api --limits=memory=512Mi
k patch deploy api -p '{"spec":{"replicas":5}}'; k patch deploy api --type json -p '[{"op":"replace","path":"/spec/template/spec/containers/0/image","value":"repo/api:1.2.5"}]'
k label po POD tier=web --overwrite; k label node NODE disk=ssd; k annotate deploy api kubernetes.io/change-cause="v1.2.4"
k scale deploy api --replicas=0; k autoscale deploy api --min=2 --max=10 --cpu-percent=60
k delete -f f.yaml; k delete po POD --grace-period=0 --force; k delete po -l app=api; k delete ns x (wait)
k replace --force -f f.yaml
```

## Rollouts

```bash
k rollout status deploy/api --timeout=180s; k rollout history deploy/api; k rollout history deploy/api --revision=3
k rollout undo deploy/api; k rollout undo deploy/api --to-revision=2
k rollout restart deploy/api            # re-create pods (picks up new ConfigMap/Secret)
k rollout pause deploy/api; k rollout resume deploy/api
k get rs -l app=api                      # old RS kept per revisionHistoryLimit
```

## Logs, exec, debug, copy

```bash
k logs POD; k logs POD -c sidecar; k logs -f --tail=100 POD; k logs --previous POD; k logs --since=10m POD
k logs -l app=api --all-containers --prefix -f; k logs deploy/api; stern api -n ns
k exec -it POD -- sh; k exec POD -c api -- env; k exec POD -- cat /etc/app/config.yaml
k debug -it POD --image=nicolaka/netshoot --target=api      # ephemeral container sharing pid ns
k debug node/NODE -it --image=busybox                       # host at /host
k cp POD:/tmp/heap.hprof ./heap.hprof -c api; k cp ./f POD:/tmp/
k port-forward svc/api 8080:80; k port-forward po/POD 5005:5005; k port-forward -n monitoring svc/grafana 3000:80
k proxy; curl localhost:8001/api/v1/namespaces/default/pods
k attach POD -c api
```

## Nodes

```bash
k get no -o wide; k describe no NODE | sed -n '/Conditions/,/Events/p'; k describe no NODE | grep -A5 Allocated
k cordon NODE; k drain NODE --ignore-daemonsets --delete-emptydir-data --grace-period=60; k uncordon NODE
k taint nodes NODE dedicated=gpu:NoSchedule; k taint nodes NODE dedicated-   # remove
k get no -l eks.amazonaws.com/nodegroup=app --show-labels
k get po -A -o wide --field-selector spec.nodeName=NODE
```

## RBAC / auth

```bash
k auth can-i create deployments -n payments; k auth can-i get secrets --as=system:serviceaccount:ci:deployer -n payments
k auth can-i --list -n payments --as alice
k create role pod-reader --verb=get,list,watch --resource=pods -n ns; k create rolebinding x --role=pod-reader --serviceaccount=ns:sa -n ns
k create clusterrolebinding x --clusterrole=view --group=developers
k get clusterrolebindings -o wide | grep cluster-admin
k create token api -n payments --duration=1h                 # SA token (1.24+)
k whoami (1.28+); aws sts get-caller-identity
# EKS access entries
aws eks list-access-entries --cluster-name prod; aws eks create-access-entry --cluster-name prod --principal-arn arn:aws:iam::123:role/Dev
aws eks associate-access-policy --cluster-name prod --principal-arn ... --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy --access-scope type=namespace,namespaces=dev
```

## Secrets / ConfigMaps

```bash
k get secret db -o jsonpath='{.data.PASSWORD}' | base64 -d; k get secret db -o go-template='{{range $k,$v := .data}}{{$k}}={{$v|base64decode}}{{"\n"}}{{end}}'
k create secret generic db --from-literal=PASSWORD=x --dry-run=client -o yaml | kubeseal -o yaml   # sealed secrets
k get cm app-config -o yaml; k create cm app-config --from-file=cfg/ -o yaml --dry-run=client | k apply -f -   # update cm
k rollout restart deploy/api     # env-based consumers don't reload; volume mounts do (~1 min)
```

## Storage

```bash
k get sc; k get pv; k get pvc -A; k describe pvc data-0
k patch sc gp3 -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
k patch pvc data -p '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'   # expand (allowVolumeExpansion)
k get volumeattachment
# EBS CSI: gp3 RWO, WaitForFirstConsumer; EFS CSI: RWX, access points
```

## Networking

```bash
k get svc -A; k get ep SVC; k get endpointslices -l kubernetes.io/service-name=api
k get ing -A; k describe ing api | grep -A10 Rules; k get ingressclass
k get netpol -A; k describe netpol deny-all
k run dns --rm -it --image=busybox:1.36 --restart=Never -- nslookup api.payments.svc.cluster.local
k get po -n kube-system -l k8s-app=kube-dns; k logs -n kube-system -l k8s-app=kube-dns
k get po -n kube-system -l k8s-app=aws-node -o wide   # VPC CNI; k describe ds aws-node -n kube-system | grep -i -E 'ENABLE_PREFIX|NETWORK_POLICY'
k get targetgroupbindings -A                           # AWS LB controller
k -n kube-system logs deploy/aws-load-balancer-controller | tail
```

Service types: ClusterIP (default), NodePort (30000–32767), LoadBalancer (NLB on EKS), ExternalName (CNAME), Headless (`clusterIP: None`).
DNS: `<svc>.<ns>.svc.cluster.local`, pod: `<ip-dashed>.<ns>.pod.cluster.local`.

## Resource / scheduling fields to remember

```yaml
nodeSelector: { disk: ssd }
tolerations: [{ key: dedicated, operator: Equal, value: gpu, effect: NoSchedule }]
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector: { matchLabels: { app: api } }
        topologyKey: kubernetes.io/hostname
topologySpreadConstraints: [{ maxSkew: 1, topologyKey: topology.kubernetes.io/zone, whenUnsatisfiable: ScheduleAnyway, labelSelector: { matchLabels: { app: api } } }]
priorityClassName: high
terminationGracePeriodSeconds: 30
dnsPolicy: ClusterFirst
hostNetwork: false
```

## Helm (see helm.md) · Kustomize

```bash
kustomize build overlays/prod | k apply -f -; k apply -k overlays/prod; k kustomize overlays/prod
# kustomization.yaml: resources, namespace, namePrefix, commonLabels, images: [{name, newTag}], patches, configMapGenerator, secretGenerator
```

## Pod status decoder

| Status | Meaning → check |
|---|---|
| Pending | unschedulable: `describe` Events (Insufficient cpu/memory, taint, affinity, PVC, no IPs) |
| ContainerCreating | pulling image, mounting volume/secret (missing Secret/ConfigMap!), CNI IP |
| ImagePullBackOff / ErrImagePull | tag/repo/auth (`describe` shows exact registry error) |
| CrashLoopBackOff | app exits → `logs --previous` |
| OOMKilled | memory limit → raise / fix leak |
| Error | container exited non-zero (Job) |
| Completed | Job done |
| Terminating (stuck) | finalizers, volume detach, node gone → `--force --grace-period=0` last resort |
| Evicted | node pressure (disk/mem) → `describe` message |
| Running 0/1 | readiness failing → `describe` probe events |
| Init:0/1, Init:CrashLoopBackOff | init container failing |
| CreateContainerConfigError | missing ConfigMap/Secret key referenced in env |
| NodeLost / Unknown | node NotReady |

## Useful one-liners

```bash
# restarts > 0
k get po -A -o json | jq -r '.items[] | select(any(.status.containerStatuses[]?; .restartCount>0)) | "\(.metadata.namespace)/\(.metadata.name) \([.status.containerStatuses[].restartCount]|add)"'
# images in cluster
k get po -A -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.image}{"\n"}{end}{end}' | sort | uniq -c | sort -rn
# pods without resource limits
k get po -A -o json | jq -r '.items[] | select(.spec.containers[].resources.limits==null) | "\(.metadata.namespace)/\(.metadata.name)"'
# delete all evicted
k get po -A --field-selector status.phase=Failed -o name | xargs -r k delete
# force delete stuck ns
k get ns x -o json | jq '.spec.finalizers=[]' | k replace --raw /api/v1/namespaces/x/finalize -f -
# which node is a pod on + node AZ
k get po POD -o jsonpath='{.spec.nodeName}' | xargs -I{} k get no {} -o jsonpath='{.metadata.labels.topology\.kubernetes\.io/zone}'
# decode all secrets in ns (careful)
for s in $(k get secret -o name); do echo "== $s"; k get $s -o go-template='{{range $k,$v := .data}}{{$k}}: {{$v|base64decode}}{{"\n"}}{{end}}'; done
```

## EKS specifics

```bash
aws eks describe-cluster --name prod --query 'cluster.{v:version,ep:resourcesVpcConfig.endpointPrivateAccess,oidc:identity.oidc.issuer,logs:logging}'
aws eks list-nodegroups --cluster-name prod; aws eks describe-nodegroup --cluster-name prod --nodegroup-name app --query 'nodegroup.{v:version,ami:releaseVersion,scaling:scalingConfig,update:updateConfig}'
aws eks update-nodegroup-version --cluster-name prod --nodegroup-name app      # rolling node AMI upgrade
aws eks list-addons --cluster-name prod; aws eks describe-addon-versions --addon-name vpc-cni --kubernetes-version 1.31 --query 'addons[].addonVersions[0].addonVersion'
aws eks update-addon --cluster-name prod --addon-name coredns --addon-version v1.11.3-eksbuild.1 --resolve-conflicts PRESERVE
aws eks list-pod-identity-associations --cluster-name prod; aws eks create-pod-identity-association --cluster-name prod --namespace payments --service-account api --role-arn arn:aws:iam::123:role/payments-api
aws eks update-cluster-config --name prod --logging '{"clusterLogging":[{"types":["api","audit","authenticator"],"enabled":true}]}'
eksctl get cluster; eksctl create cluster -f cluster.yaml; eksctl utils associate-iam-oidc-provider --cluster prod --approve
# upgrade order: control plane (1 minor at a time) → add-ons (vpc-cni, coredns, kube-proxy) → node groups → verify
```
