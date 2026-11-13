# 09 - Helm and Packaging

Topics: Charts, values, templates, releases, dependencies, repositories (+ Kustomize)

CKA curriculum me **Helm aur Kustomize** dono "Cluster Architecture, Installation & Configuration" me aate hain.

## 1. Helm kya hai?

Kubernetes ka package manager. Kai YAML files ko ek **chart** me package karke **values** se customize karte hain.

Terms:
- **Chart**: package (templates + default values + metadata)
- **Release**: cluster me chart ka ek installed instance (naam + namespace). Ek chart ke kai releases ho sakte hain.
- **Repository**: charts ka index host karne wali jagah (HTTP ya OCI registry)
- **Revision**: release ka har upgrade/rollback ek naya revision
- Helm 3: **Tiller nahi**, release info cluster me Secrets me store hoti hai (`sh.helm.release.v1.*`).

## 2. Chart structure

```
mychart/
├── Chart.yaml          # name, version, appVersion, dependencies
├── values.yaml         # default values
├── charts/             # downloaded dependencies
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── _helpers.tpl    # reusable named templates
│   ├── NOTES.txt       # install ke baad message
│   └── tests/
└── .helmignore
```

`Chart.yaml`:
```yaml
apiVersion: v2
name: mychart
description: Demo app
type: application
version: 0.1.0          # chart version
appVersion: "1.27"      # app version
dependencies:
- name: postgresql
  version: "15.x.x"
  repository: https://charts.bitnami.com/bitnami
  condition: postgresql.enabled
```

## 3. Templates (Go templates + Sprig)

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-web
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
      - name: web
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        {{- if .Values.resources }}
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
        {{- end }}
```
```yaml
# values.yaml
replicaCount: 2
image:
  repository: nginx
  tag: ""
resources: {}
```

Built-in objects: `.Values`, `.Release` (Name, Namespace, Revision, IsInstall), `.Chart`, `.Capabilities`, `.Template`, `.Files`.
Constructs: `if/else`, `range`, `with`, `define`/`include`, `default`, `quote`, `toYaml`, `nindent`, `required`.

## 4. Core commands

### Repository
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm repo list
helm search repo nginx                     # added repos me
helm search repo nginx --versions
helm search hub wordpress                  # Artifact Hub
helm show values bitnami/nginx > values.yaml
helm show chart bitnami/nginx
helm pull bitnami/nginx --untar
```

### Install / upgrade / rollback
```bash
helm install my-nginx bitnami/nginx -n web --create-namespace
helm install my-nginx bitnami/nginx --set replicaCount=3 --set service.type=NodePort
helm install my-nginx bitnami/nginx -f custom-values.yaml
helm install my-nginx ./mychart --dry-run --debug

helm upgrade my-nginx bitnami/nginx -f custom-values.yaml
helm upgrade --install my-nginx ./mychart -n web       # idempotent (CI/CD)
helm upgrade my-nginx ./mychart --reuse-values --set image.tag=1.27

helm list -A
helm status my-nginx -n web
helm history my-nginx -n web
helm rollback my-nginx 1 -n web
helm uninstall my-nginx -n web                          # --keep-history optional
```

### Authoring / debug
```bash
helm create mychart
helm lint ./mychart
helm template my-rel ./mychart -f values.yaml          # cluster ke bina render
helm get values my-nginx                                # user-supplied values (--all for computed)
helm get manifest my-nginx                              # actual installed YAML
helm package ./mychart                                  # .tgz
helm dependency update ./mychart                        # charts/ me download, Chart.lock
helm dependency build ./mychart
helm test my-nginx
```

### Values precedence (low -> high)
chart `values.yaml` < parent chart values < `-f file` (right-most file wins) < `--set` / `--set-string` / `--set-file`

### Flags jo kaam aate hain
`--atomic` (fail pe auto-rollback), `--wait`, `--timeout`, `--version`, `-n`, `--create-namespace`.

### Hooks (concept)
Annotation `helm.sh/hook: pre-install, post-upgrade ...` se lifecycle pe Jobs chalao (DB migration). Delete policy: `helm.sh/hook-delete-policy`.

### OCI registries
```bash
helm registry login ghcr.io
helm push mychart-0.1.0.tgz oci://ghcr.io/myorg/charts
helm install my oci://ghcr.io/myorg/charts/mychart --version 0.1.0
```

## 5. Kustomize (template-free)

Base YAML + **overlays** (patches) se environments (dev/prod) manage karta hai. `kubectl` me built-in: `kubectl apply -k`, `kubectl kustomize`.

```
app/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/kustomization.yaml
    └── prod/
        ├── kustomization.yaml
        └── replica-patch.yaml
```
`base/kustomization.yaml`:
```yaml
resources:
- deployment.yaml
- service.yaml
```
`overlays/prod/kustomization.yaml`:
```yaml
resources:
- ../../base
namePrefix: prod-
namespace: production
commonLabels:
  env: prod
images:
- name: nginx
  newTag: "1.27"
replicas:
- name: web
  count: 5
patches:
- path: replica-patch.yaml
configMapGenerator:
- name: app-config
  literals:
  - LOG=info
```
```bash
k kustomize overlays/prod          # render dekho
k apply -k overlays/prod
k delete -k overlays/prod
```

### Helm vs Kustomize
| | Helm | Kustomize |
|---|------|-----------|
| Approach | Templating + packaging | Patch-based overlays |
| Release tracking/rollback | Haan | Nahi (kubectl/GitOps se) |
| Learning curve | Zyada | Kam |
| Sharing | Chart repos | Git directories |
Dono saath me bhi use hote hain (Helm output ko Kustomize se patch).

## Extension concepts (CKA curriculum se related)

- **CRD** (CustomResourceDefinition) naye API types add karta hai: `k get crd`.
- **Operator** = CRD + custom controller (domain knowledge automate: DB backup, failover).
- `k explain <crd-kind>` CRD fields dikhata hai.

## Common mistakes

- `helm upgrade` me `--reuse-values` aur naye chart defaults ka conflict.
- `helm upgrade` me values dobara na dena (default pe wapas chale jaate hain).
- Release ka namespace (`-n`) bhul jana; `helm list` sirf current namespace dikhata hai (`-A` se sab).
- `helm template` me cluster-specific lookup functions kaam nahi karte.
- YAML indentation (`nindent` count galat).

## Practice labs

1. Bitnami nginx chart install karo `--set service.type=NodePort`, phir upgrade karke replicas badhao, rollback karo.
2. `helm create demo`, templates badlo, `helm lint` aur `helm template` se verify karo, phir install.
3. `helm get values` / `helm get manifest` se installed state dekho.
4. Kustomize base + dev/prod overlays banao aur `k kustomize` se difference dekho.
5. Ek chart me dependency add karke `helm dependency update` chalao.
