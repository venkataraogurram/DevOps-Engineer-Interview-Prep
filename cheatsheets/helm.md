# Helm cheatsheet (annotated)

[← Cheatsheet index](README.md)

Format: command → what it does and when you reach for it.

## Repos and charts

- `helm repo add eks https://aws.github.io/eks-charts`

  Register a chart repository under a name. Needed before `helm search repo` / `install repo/chart`.

- `helm repo update`

  Refresh the index of all added repos. Run before installing to get current versions.

- `helm repo list`

  Registered repos.

- `helm search repo nginx --versions`

  Find charts in added repos; `--versions` lists every chart version so you can pin one.

- `helm search hub prometheus`

  Search Artifact Hub (public catalogue) when you don't know the repo.

- `helm show values bitnami/postgresql > values.yaml`

  Dump the chart's default values. Start every customisation from this file.

- `helm show chart X` / `helm show readme X` / `helm show crds X`

  Metadata / docs / CRDs the chart will install. Check CRDs before installing into a shared cluster.

- `helm pull bitnami/redis --untar --version 19.0.0`

  Download and unpack a chart to read its templates. Use to understand what a chart really creates.

- `aws ecr get-login-password | helm registry login --username AWS --password-stdin 123.dkr.ecr.ap-south-1.amazonaws.com`

  Authenticate to ECR as an OCI chart registry.

- `helm push app-1.2.3.tgz oci://123.dkr.ecr.ap-south-1.amazonaws.com/charts`

  Publish a packaged chart to ECR. Private charts live next to images.

- `helm install app oci://123.dkr.ecr.ap-south-1.amazonaws.com/charts/app --version 1.2.3`

  Install straight from the OCI registry.

## Install, upgrade, rollback

- `helm install api ./charts/api -n payments --create-namespace -f values-prod.yaml --set image.tag=1.2.3`

  First install of a release named `api` from a local chart, with an environment values file and an inline override.

- `helm upgrade --install api ./charts/api -n payments -f values-prod.yaml --atomic --timeout 5m --wait --history-max 10`

  The idempotent pipeline command: install if missing, else upgrade. `--atomic` rolls back automatically on failure, `--wait` blocks until resources are ready, `--history-max` caps stored revisions.

- `helm upgrade api ./charts/api --reuse-values --set replicaCount=5`

  Keep previous values and change one. Caution: `--reuse-values` ignores new defaults from an upgraded chart.

- `helm upgrade api ./charts/api --reset-values -f values-prod.yaml`

  Discard previous values and apply only what you pass. The safer habit in pipelines.

- `helm install --dry-run --debug api ./charts/api -f values.yaml`

  Render templates and validate against the API server without installing. Shows the final YAML and any template errors.

- `helm template api ./charts/api -f values.yaml > rendered.yaml`

  Render locally with no cluster. Pipe into `kubectl apply --dry-run=server -f -` for admission checks or into `conftest` for policy.

- `helm list -A` / `helm list -n payments --all`

  Releases in all namespaces / including failed and pending ones.

- `helm status api -n payments`

  Release state, last deployed time, notes.

- `helm get values api -n payments` / `--all`

  Values you supplied / merged with chart defaults. Use to answer "what is actually configured".

- `helm get manifest api` / `helm get notes api`

  The rendered manifests Helm applied / the NOTES.txt output.

- `helm history api -n payments`

  Revision list with status and chart version. Pick a target for rollback.

- `helm rollback api 3 -n payments --wait`

  Roll back to revision 3 and wait for readiness.

- `helm uninstall api -n payments`

  Delete the release and its resources. `--keep-history` keeps revision records so you can roll back a deletion.

- `helm diff upgrade api ./charts/api -f values-prod.yaml`

  Shows what an upgrade would change (plugin: `helm plugin install https://github.com/databus23/helm-diff`). Review before prod upgrades.

- `helm test api -n payments`

  Runs the chart's `tests/` pods (smoke tests) against the release.

## Authoring

- `helm create api`

  Scaffold a chart with a Deployment/Service/Ingress/HPA/ServiceAccount and helpers. Start here, delete what you don't need.

- `helm lint ./charts/api -f values-prod.yaml`

  Checks chart structure and renders with the given values to catch template errors. Run in CI.

- `helm package ./charts/api --version 1.2.3 --app-version 1.2.3`

  Build the `.tgz`. `version` is the chart version, `appVersion` the software version.

- `helm dependency update ./charts/api` / `dependency build` / `dependency list`

  Fetch subcharts declared in `Chart.yaml` into `charts/` / rebuild from lock / show them.

```
charts/api/
├── Chart.yaml          # apiVersion: v2, name, version (chart semver), appVersion, dependencies: [{name, version, repository, condition}]
├── values.yaml         # defaults; every template value should have one here
├── values.schema.json  # optional JSON schema; makes bad values fail at install time
├── templates/
│   ├── _helpers.tpl    # named templates: {{- define "api.fullname" -}} ... {{- end }}
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── serviceaccount.yaml
│   ├── NOTES.txt       # printed after install: URLs, next steps
│   └── tests/test-connection.yaml   # run by `helm test`
├── crds/               # installed before templates, never upgraded or deleted by Helm
└── charts/             # vendored subcharts
```

```yaml
# templates/deployment.yaml – the patterns you'll be asked about
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "api.fullname" . }}                 # helper keeps names consistent and ≤63 chars
  labels: {{- include "api.labels" . | nindent 4 }}    # standard labels block; nindent handles indentation
  annotations:
    checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}   # pods restart when the ConfigMap changes
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}                 # omit replicas when HPA manages them, or they fight
  {{- end }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"   # tag falls back to appVersion
          {{- with .Values.resources }}
          resources: {{- toYaml . | nindent 12 }}      # pass a whole values block through as YAML
          {{- end }}
          env:
            {{- range $k, $v := .Values.env }}         # map → env list
            - name: {{ $k }}
              value: {{ $v | quote }}                  # quote so numbers/bools stay strings
            {{- end }}
{{- if .Values.ingress.enabled }}
---
apiVersion: networking.k8s.io/v1
kind: Ingress
...
{{- end }}
```

- `default "x" .Values.y`

  Fallback when a value is unset.

- `quote`

  Wrap in quotes; essential for env values and annotations.

- `toYaml` + `nindent N`

  Emit a values sub-tree as YAML at the right indentation.

- `include "name" .`

  Call a named template and capture its output (pipe-able, unlike `template`).

- `tpl .Values.x .`

  Render a string from values that itself contains template syntax. Lets users put `{{ .Release.Name }}` in values.

- `required "msg" .Values.x`

  Fail the render with a clear message if a value is missing.

- `lookup "v1" "Secret" "ns" "name"`

  Read a live object during render (e.g. keep an existing random password). Empty in `template`/`--dry-run`.

- `b64enc` / `sha256sum` / `printf` / `trunc 63` / `trimSuffix "-"`

  Encode secrets, hash for checksums, build names that fit DNS limits.

- `has` / `hasKey` / `ternary`

  Conditionals on lists/maps; `ternary "a" "b" cond`.

- `semverCompare ">=1.25-0" .Capabilities.KubeVersion.Version`

  Branch on cluster version (e.g. API changes).

- `.Capabilities.APIVersions.Has "autoscaling/v2"`

  Emit the right API version for the target cluster.

- `.Release.Name` / `.Release.Namespace` / `.Release.Revision` / `.Values` / `.Chart`

  Built-in objects.

Hooks: annotate a Job with `helm.sh/hook: pre-install,pre-upgrade` to run DB migrations before the app rolls; `helm.sh/hook-weight` orders hooks; `helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded` cleans up finished hook jobs so re-runs don't collide.

## Values precedence (lowest → highest)

chart `values.yaml` → parent chart's values for a subchart → `-f a.yaml` → `-f b.yaml` (later file wins) → `--set` / `--set-string` / `--set-file` (last wins). Subchart values nest under the subchart name; `global:` is visible to all.

## Gotchas

- **Where is release state?**

  Secrets named `sh.helm.release.v1.<name>.v<n>` in the release namespace. Deleting them orphans resources. `--history-max` caps them.

- `--atomic` timed out and rolled back

  `--atomic` implies `--wait`; readiness probes must pass within `--timeout`. Fix probes or raise the timeout.

- **CRDs not updated on upgrade**

  Helm never upgrades `crds/`. Apply CRD changes separately (`kubectl apply -f crds/`) or ship them in a dedicated chart.

- `field is immutable` on upgrade

  Deployment `selector`, Job spec, Service `clusterIP` can't change. `--force` deletes and recreates (downtime) or change the release/resource name.

- `another operation (install/upgrade/rollback) is in progress`

  A previous run died leaving `pending-upgrade`. `helm rollback` to the last good revision, or delete the pending release Secret.

- **Resource vanished after upgrade**

  You removed it from the chart, so Helm deleted it. Protect with annotation `helm.sh/resource-policy: keep`.

- `helm list` doesn't show my app

  Argo CD/Flux render charts themselves and don't create Helm release Secrets. Look in the GitOps tool.

- **Values not applied**

  Wrong indentation/key path, or `--reuse-values` masking a new default. `helm get values --all` shows the truth.

