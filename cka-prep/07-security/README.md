# 07 - Security

Topics: RBAC, ServiceAccounts, Pod Security Standards, securityContext, admission controls

(CKA ke "Cluster Architecture, Installation & Configuration" domain me RBAC bahut important hai.)

## API request ka flow

```
Request -> Authentication (kaun?) -> Authorization (allowed?) -> Admission (mutate/validate) -> etcd
```

## 1. Authentication (short)

- Users cluster ke andar objects nahi hote. Identity client certificates (CN = user, O = group), tokens, OIDC se aati hai.
- **ServiceAccounts** Pods/processes ke liye hain.
- kubeconfig: clusters + users + contexts.

```bash
k config view
k config get-contexts
k config use-context <ctx>
k config set-credentials jane --client-certificate=jane.crt --client-key=jane.key
k config set-context jane-ctx --cluster=kubernetes --user=jane
```

### CSR se user certificate (exam me aata hai)
```bash
openssl genrsa -out jane.key 2048
openssl req -new -key jane.key -subj "/CN=jane/O=dev" -out jane.csr
```
```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane
spec:
  request: <base64 -w0 jane.csr>
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400
  usages: ["client auth"]
```
```bash
k certificate approve jane
k get csr jane -o jsonpath='{.status.certificate}' | base64 -d > jane.crt
```

## 2. RBAC (Role-Based Access Control)

Authorization mode: `--authorization-mode=Node,RBAC` (API server flag). RBAC **additive** hai: sirf allow rules, deny nahi.

| Object | Scope | Kaam |
|--------|-------|------|
| `Role` | Namespace | Namespace ke andar permissions |
| `ClusterRole` | Cluster | Cluster-scoped resources (nodes, PVs) ya sab namespaces ya non-resource URLs |
| `RoleBinding` | Namespace | Role/ClusterRole ko subject se jodta hai us namespace me |
| `ClusterRoleBinding` | Cluster | ClusterRole ko poore cluster me |

Subjects: `User`, `Group`, `ServiceAccount`.

```bash
k create role pod-reader --verb=get,list,watch --resource=pods -n dev
k create rolebinding jane-pods --role=pod-reader --user=jane -n dev
k create clusterrole node-reader --verb=get,list,watch --resource=nodes
k create clusterrolebinding jane-nodes --clusterrole=node-reader --user=jane
k create rolebinding sa-bind --clusterrole=view --serviceaccount=dev:mysa -n dev
```

YAML:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: pod-reader
rules:
- apiGroups: [""]                 # "" = core group
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
  # resourceNames: ["specific-pod"]
```

Verbs: `get, list, watch, create, update, patch, delete, deletecollection`. Subresources: `pods/exec`, `pods/log`.

### Check permissions
```bash
k auth can-i create deployments -n dev
k auth can-i list pods --as jane -n dev
k auth can-i '*' '*' --as system:serviceaccount:dev:mysa
k auth whoami
k api-resources -o wide            # apiGroups aur verbs dekhne ke liye
```

Gotchas: RoleBinding ClusterRole ko refer kar sakti hai (permissions sirf us namespace me). `roleRef` immutable hai. Built-in ClusterRoles: `cluster-admin`, `admin`, `edit`, `view`.

## 3. ServiceAccounts

Pod ki identity jab wo API server se baat kare.

```bash
k create sa mysa -n dev
k create token mysa -n dev                 # short-lived bound token
```
Pod me:
```yaml
spec:
  serviceAccountName: mysa
  automountServiceAccountToken: false      # zarurat na ho to band
```
- Namespace me `default` SA hota hai.
- Naye versions me tokens **bound, time-limited, projected volume** se aate hain (`/var/run/secrets/kubernetes.io/serviceaccount`). Long-lived Secret tokens auto nahi bante (zarurat ho to manually Secret type `kubernetes.io/service-account-token`).
- `imagePullSecrets` SA me bhi laga sakte ho.

## 4. securityContext

Pod level (`spec.securityContext`) aur container level (`spec.containers[].securityContext`; container wala override karta hai).

```yaml
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    runAsNonRoot: true
    seccompProfile: {type: RuntimeDefault}
  containers:
  - name: app
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      privileged: false
      capabilities:
        drop: ["ALL"]
        add: ["NET_BIND_SERVICE"]
```
Capabilities sirf container level pe hoti hain.

## 5. Pod Security Standards (PSS) / Pod Security Admission (PSA)

PodSecurityPolicy **removed** (v1.25). Ab built-in **Pod Security Admission** namespace labels se.

Levels:
- **privileged**: unrestricted
- **baseline**: known privilege escalations roke (hostNetwork, privileged, hostPath etc. blocked)
- **restricted**: hardened best practices (non-root, drop ALL caps, seccomp, no privilege escalation)

Modes: `enforce` (reject), `audit` (log), `warn` (user ko warning).

```bash
k label ns dev pod-security.kubernetes.io/enforce=restricted
k label ns dev pod-security.kubernetes.io/warn=restricted pod-security.kubernetes.io/audit=restricted
k label ns dev pod-security.kubernetes.io/enforce-version=latest
```

## 6. Admission controllers

API request authN/authZ ke baad, persist hone se pehle intercept karte hain.
- **Mutating** (object badalte hain, e.g. default StorageClass, ServiceAccount) -> phir **Validating**.
- Built-in plugins API server flag `--enable-admission-plugins` / `--disable-admission-plugins` se. Examples: `NamespaceLifecycle`, `LimitRanger`, `ResourceQuota`, `ServiceAccount`, `PodSecurity`, `NodeRestriction`.
- Dynamic: `MutatingWebhookConfiguration`, `ValidatingWebhookConfiguration` (aur `ValidatingAdmissionPolicy`, CEL-based, built-in).
- Policy engines: Kyverno, OPA Gatekeeper.

Check: `k -n kube-system describe pod kube-apiserver-<node> | grep admission` ya `/etc/kubernetes/manifests/kube-apiserver.yaml`.

## 7. Aur security topics (awareness)

- **Secrets**: encryption at rest (`EncryptionConfiguration`), RBAC limit.
- **NetworkPolicy** (03-networking).
- **TLS certs**: `/etc/kubernetes/pki`. `kubeadm certs check-expiration`, `kubeadm certs renew all`.
- **Image security**: trusted registries, scanning (Trivy), `imagePullPolicy`, digest pinning.
- **Node/Runtime**: AppArmor/seccomp, runtimeClass (gVisor/Kata).
- **Audit logs**: API server `--audit-policy-file`, `--audit-log-path`.
- API server hardening: `--anonymous-auth=false`, kubelet auth, `--authorization-mode`.

## Common mistakes

- Role me galat apiGroup (`""` core, `apps` Deployments ke liye, `batch` Jobs ke liye).
- RoleBinding me ServiceAccount ka namespace nahi dena (`--serviceaccount=ns:name`).
- `can-i` bina `-n` ke default namespace check karna.
- PSA restricted namespace me Pod create fail (securityContext missing).
- Container-level aur pod-level securityContext fields ko mix-up karna.

## Practice labs

1. User `jane` ke liye CSR flow, kubeconfig context aur `dev` namespace me sirf pods read access do.
2. ServiceAccount banao, Pod me use karo aur Pod ke andar se API call (curl with token) test karo.
3. Namespace ko `restricted` PSS pe lagao, non-compliant Pod reject hote dekho, phir fix karo.
4. Container ko non-root, read-only rootfs aur sab capabilities drop ke saath chalao.
5. `k auth can-i --list --as jane -n dev` se permission audit karo.
