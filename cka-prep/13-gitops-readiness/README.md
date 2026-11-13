# 13 - GitOps Readiness

Topics: Declarative desired state, reconciliation, drift, promotion, secrets handling

> CKA exam me GitOps tools (Argo CD/Flux) nahi aate, par ye concepts modern Kubernetes operations ka hissa hain.

## 1. GitOps kya hai?

Principle: **Git = single source of truth** for cluster ka desired state. Changes Git (PR) se hote hain, `kubectl apply` manually nahi.

4 principles (OpenGitOps):
1. **Declarative**: system ka desired state declaratively express ho (YAML/Helm/Kustomize).
2. **Versioned and immutable**: state Git me versioned, history/audit/rollback.
3. **Pulled automatically**: agent cluster ke andar se Git se **pull** karta hai.
4. **Continuously reconciled**: agent actual state ko desired state se match karta rehta hai.

## 2. Declarative desired state

Kubernetes khud declarative hai: aap bataate ho "kya chahiye" (`replicas: 3`), controllers "kaise" nikalte hain.
- Imperative: `kubectl scale ...` (history nahi, drift banata hai)
- Declarative: YAML Git me commit -> apply

Repo me kya rakhein: Deployments, Services, ConfigMaps, RBAC, NetworkPolicy, Ingress, namespaces, CRDs, Helm values, Kustomize overlays, policies. **Plain Secrets nahi**.

Repo structure (example):
```
gitops-repo/
├── apps/
│   └── web/
│       ├── base/
│       └── overlays/
│           ├── dev/
│           ├── staging/
│           └── prod/
├── infrastructure/        # ingress, cert-manager, monitoring
├── clusters/
│   ├── dev/
│   └── prod/
└── README.md
```
Common strategies: **mono-repo** vs **repo per team/env**; env ke liye **folders** (recommended) vs branches.

## 3. Reconciliation loop

```
Git (desired) --> GitOps controller --> compare --> Cluster (actual)
                        ^                              |
                        └──────── apply diff <─────────┘
```
- Controller periodically (ya webhook se) Git fetch karta hai, manifests render (Helm/Kustomize), cluster se diff karta hai, difference apply karta hai.
- Kubernetes controllers ka hi pattern: observe -> diff -> act.
- **Sync status**: `Synced` / `OutOfSync`; **Health**: `Healthy` / `Progressing` / `Degraded`.
- Options: automated sync, **prune** (Git se hata to cluster se bhi hatao), **self-heal** (manual change wapas revert).

## 4. Tools

| | **Argo CD** | **Flux** |
|---|-------------|----------|
| Style | UI + CLI, Application CRD | Toolkit of controllers (source, kustomize, helm, notification) |
| Core CRDs | `Application`, `ApplicationSet`, `AppProject` | `GitRepository`, `Kustomization`, `HelmRelease`, `HelmRepository` |
| Multi-cluster | Hub-and-spoke from one Argo | Per-cluster Flux (or hub) |
| Extras | Rich UI, SSO/RBAC, sync waves/hooks | Image automation, native Kubernetes feel |

Argo CD Application:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/gitops-repo.git
    targetRevision: main
    path: apps/web/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: web
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

Flux:
```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata: {name: gitops-repo, namespace: flux-system}
spec:
  interval: 1m
  url: https://github.com/org/gitops-repo
  ref: {branch: main}
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata: {name: web, namespace: flux-system}
spec:
  interval: 5m
  path: ./apps/web/overlays/prod
  prune: true
  sourceRef: {kind: GitRepository, name: gitops-repo}
```
(API versions releases ke saath badalte hain, tool docs se verify karo.)

## 5. Drift

**Drift** = cluster ka actual state Git ke desired state se alag ho jana (kisi ne `kubectl edit/scale` kiya, HPA ne replicas badle, mutating webhook ne fields add kiye).
- Detect: controller `OutOfSync` dikhata hai, `argocd app diff`, `flux diff`.
- Handle: **self-heal** (auto revert) ya alert; sahi tareeka - change Git me PR se.
- False positives: HPA-managed `replicas`, controller-injected fields, defaults -> **ignoreDifferences** (Argo) ya field ownership/server-side apply se handle.
- Break-glass: emergency `kubectl` ke baad Git me wahi fix commit karo.
- Policy: normal humans ka cluster write access **RBAC se limit**, changes sirf GitOps controller se.

## 6. Promotion (dev -> staging -> prod)

Same artifact, alag environment config; badlav **Git me PR** se.
- **Image promotion**: CI image build karke tag/digest push karta hai (`app:1.4.2` ya SHA digest, `latest` nahi). Image tag overlay me update (PR/auto-commit; Flux image-automation, Argo Image Updater, Renovate).
- **Folder-based**: `overlays/dev` -> `staging` -> `prod`; promotion = us folder me version/values update.
- Gates: automated tests, approvals (CODEOWNERS/PR review), progressive delivery (**Argo Rollouts / Flagger**: canary, blue/green, auto-rollback via metrics).
- **Rollback** = Git `revert` commit (controller sync kar dega).
- Pipeline split: **CI** (build/test/push, manifests update) aur **CD** (GitOps controller pull/sync).

## 7. Secrets handling

Plain Kubernetes Secret (sirf base64) ko **Git me commit mat karo**. Options:

| Approach | Kaise kaam karta hai | Note |
|----------|---------------------|------|
| **Sealed Secrets** | `kubeseal` public key se encrypt -> `SealedSecret` Git me; cluster controller decrypt karke Secret banata hai | Cluster-specific key; key backup zaroori |
| **SOPS** (+ age / KMS / PGP) | Secret YAML ke values encrypt, Git me; Flux native decrypt, Argo plugin ke saath | Keys KMS me rakh sakte ho |
| **External Secrets Operator (ESO)** | `ExternalSecret` CRD cloud/Vault secret store se Secret sync karta hai; Git me sirf reference | Source of truth external store |
| **Secrets Store CSI Driver** | Pod me secret volume ke roop me mount (Vault, Key Vault, Secrets Manager) | K8s Secret optional |
| **HashiCorp Vault** (agent/injector, VSO) | Dynamic, short-lived secrets | Zyada operational complexity |

```yaml
# External Secrets example
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-creds
spec:
  refreshInterval: 1h
  secretStoreRef: {name: aws-store, kind: ClusterSecretStore}
  target: {name: db-creds}
  data:
  - secretKey: password
    remoteRef: {key: prod/db, property: password}
```
Best practices: RBAC se Secret read limit, etcd encryption at rest, secret rotation, git history me leaked secret ho gaya to **rotate** karo (sirf delete kaafi nahi), pre-commit scanners (gitleaks).

## 8. GitOps readiness checklist

- [ ] Saare manifests declarative aur Git me
- [ ] Plain Secrets repo me nahi (Sealed/SOPS/ESO)
- [ ] Image tags/digests pinned, `latest` nahi
- [ ] Environments ka clear structure (overlays/folders)
- [ ] PR review + branch protection + CODEOWNERS
- [ ] Controller (Argo CD/Flux) bootstrap hua, pull-based
- [ ] Automated sync + prune + self-heal decide kiya (prod me manual approval option)
- [ ] Humans ka direct `kubectl` write access limited
- [ ] Health checks (readiness probes) aur sync status alerts
- [ ] Rollback = `git revert` tested
- [ ] Policy as code (Kyverno/Gatekeeper) aur CI validation (`kubeconform`, `kustomize build`, `helm lint`)
- [ ] Backup: Git + cluster state (Velero) + controller config
- [ ] Documented break-glass process

## Common mistakes

- Secrets plain Git me.
- Imperative `kubectl` se hotfix aur Git me wapas na daalna (drift).
- Prune enable karke galti se resources delete ho jaana (pehle dry-run/diff).
- Environment ke liye branches (merge conflicts, drift); folders prefer.
- HPA ke `replicas` aur Git ke `replicas` ka tug-of-war.
- Controller ko cluster-admin dena bina scope (AppProject / RBAC limit karo).

## Practice labs

1. kind cluster par Argo CD (ya Flux) install karo, ek Git repo se app sync karo.
2. `kubectl scale` se manual change karo aur self-heal se wapas aate dekho.
3. dev -> prod overlay promotion PR flow banao.
4. Sealed Secrets ya SOPS se Secret encrypt karke Git me commit karo aur cluster me decrypt hote dekho.
5. Galat commit ko `git revert` se rollback karo.
