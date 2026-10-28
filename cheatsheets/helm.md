# Helm cheatsheet

[← Cheatsheet index](README.md)

## Repos and charts

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami; helm repo add eks https://aws.github.io/eks-charts
helm repo update; helm repo list; helm search repo nginx --versions; helm search hub prometheus
helm show values bitnami/postgresql > values.yaml; helm show chart X; helm show readme X; helm show crds X
helm pull bitnami/redis --untar --version 19.0.0
# OCI registries (ECR)
aws ecr get-login-password | helm registry login --username AWS --password-stdin 123.dkr.ecr.ap-south-1.amazonaws.com
helm push app-1.2.3.tgz oci://123.dkr.ecr.ap-south-1.amazonaws.com/charts; helm install app oci://123.dkr.ecr.ap-south-1.amazonaws.com/charts/app --version 1.2.3
```

## Install / upgrade / rollback

```bash
helm install api ./charts/api -n payments --create-namespace -f values-prod.yaml --set image.tag=1.2.3
helm upgrade --install api ./charts/api -n payments -f values-prod.yaml --atomic --timeout 5m --wait --history-max 10
helm upgrade api ./charts/api --reuse-values --set replicaCount=5          # careful: reuse-values ignores new chart defaults
helm upgrade api ./charts/api --reset-values -f values-prod.yaml
helm install --dry-run --debug api ./charts/api -f values.yaml              # render + server-side validation
helm template api ./charts/api -f values.yaml > rendered.yaml; helm template ... | kubectl apply --dry-run=server -f -
helm list -A; helm list -n payments --all; helm status api -n payments; helm get values api -n payments; helm get values api --all; helm get manifest api; helm get notes api
helm history api -n payments; helm rollback api 3 -n payments --wait
helm uninstall api -n payments; helm uninstall api --keep-history
helm diff upgrade api ./charts/api -f values-prod.yaml                      # plugin: helm plugin install https://github.com/databus23/helm-diff
helm test api -n payments
```

## Authoring

```bash
helm create api; helm lint ./charts/api -f values-prod.yaml; helm package ./charts/api --version 1.2.3 --app-version 1.2.3
helm dependency update ./charts/api; helm dependency build; helm dependency list
```

```
charts/api/
├── Chart.yaml          apiVersion: v2, name, version (chart semver), appVersion, dependencies: [{name, version, repository, condition}]
├── values.yaml
├── values.schema.json  (optional validation)
├── templates/
│   ├── _helpers.tpl    {{- define "api.fullname" -}} ... {{- end }}
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── serviceaccount.yaml
│   ├── NOTES.txt
│   └── tests/test-connection.yaml
├── crds/               installed first, never upgraded by helm
└── charts/             vendored dependencies
```

Template snippets:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "api.fullname" . }}
  labels: {{- include "api.labels" . | nindent 4 }}
  annotations:
    checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}   # restart on config change
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          {{- with .Values.resources }}
          resources: {{- toYaml . | nindent 12 }}
          {{- end }}
          env:
            {{- range $k, $v := .Values.env }}
            - name: {{ $k }}
              value: {{ $v | quote }}
            {{- end }}
{{- if .Values.ingress.enabled }}
---
apiVersion: networking.k8s.io/v1
kind: Ingress
...
{{- end }}
```

Functions: `default`, `quote`, `toYaml`, `nindent`, `indent`, `include`, `tpl`, `required "msg" .Values.x`, `lookup "v1" "Secret" "ns" "name"`, `b64enc`, `sha256sum`, `printf`, `trunc 63`, `trimSuffix "-"`, `has`, `hasKey`, `ternary`, `semverCompare ">=1.25-0" .Capabilities.KubeVersion.Version`, `.Release.Name/.Namespace/.Revision`, `.Values`, `.Chart`, `.Capabilities.APIVersions.Has "autoscaling/v2"`.

Hooks: `helm.sh/hook: pre-install,pre-upgrade` (DB migrations Job), `helm.sh/hook-weight`, `helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded`.

## Values precedence (lowest → highest)

chart `values.yaml` → parent chart values for subchart → `-f a.yaml` → `-f b.yaml` → `--set` / `--set-string` / `--set-file` (later wins). Subchart values nested under its name; `global:` shared.

## Gotchas

- Release state lives in Secrets `sh.helm.release.v1.<name>.v<n>` in the namespace; `--history-max` to cap.
- `--atomic` implies `--wait`; `--wait` needs correct readiness probes or it times out.
- CRDs in `crds/` are not upgraded; manage separately or via a dedicated chart.
- Immutable fields (Deployment `selector`, Job spec) → upgrade fails; `--force` deletes/recreates (downtime).
- "another operation is in progress" → stuck `pending-upgrade` release: `helm rollback` or delete the release secret with the highest revision.
- `helm upgrade` + removed resource from chart → deleted from cluster (unless `helm.sh/resource-policy: keep`).
- Argo CD / Flux render charts themselves; `helm list` won't show them.
