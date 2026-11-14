# 02 - Configuration

Topics: ConfigMap, Secret, env, volumes, probes, requests, limits

## 1. ConfigMap

Non-sensitive config (key-value ya poori file) ko Pod se alag rakhne ke liye. Limit ~1 MiB.

```bash
k create configmap app-config --from-literal=ENV=prod --from-literal=LOG=debug
k create configmap app-file --from-file=app.properties
k create configmap app-dir --from-file=./configs/
k get cm app-config -o yaml
```

### Pod me use karne ke 3 tareeke

**a) Single env var**
```yaml
env:
- name: APP_ENV
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: ENV
```

**b) Saare keys env me**
```yaml
envFrom:
- configMapRef:
    name: app-config
```

**c) Volume ke roop me (files)**
```yaml
volumes:
- name: cfg
  configMap:
    name: app-file
containers:
- name: app
  image: nginx
  volumeMounts:
  - name: cfg
    mountPath: /etc/config        # har key ek file banti hai
    # subPath: app.properties     # sirf ek file mount (is case me auto-update nahi hota)
```

Notes:
- Env vars Pod start par fix ho jaate hain; ConfigMap badalne par **update nahi hote** (Pod restart chahiye).
- Volume-mounted ConfigMap kuch der me auto-update ho jata hai (subPath ke bina).
- `immutable: true` set karne se accidental change rukta hai.

## 2. Secret

Sensitive data (password, token, TLS keys). Data **base64 encoded** hota hai, **encrypted nahi**. Isliye:
- etcd me **encryption at rest** enable karo (`EncryptionConfiguration`)
- RBAC se Secret read access limit karo

```bash
k create secret generic db-secret --from-literal=username=admin --from-literal=password='S3cr3t'
k create secret docker-registry regcred --docker-server=reg.io --docker-username=u --docker-password=p
k create secret tls my-tls --cert=tls.crt --key=tls.key
k get secret db-secret -o jsonpath='{.data.password}' | base64 -d
```

Types: `Opaque` (generic), `kubernetes.io/tls`, `kubernetes.io/dockerconfigjson`, `kubernetes.io/service-account-token`, `kubernetes.io/basic-auth`, `kubernetes.io/ssh-auth`.

Use karna ConfigMap jaisa hi hai:
```yaml
env:
- name: DB_PASS
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: password
volumes:
- name: sec
  secret:
    secretName: db-secret
```

Private registry image: Pod spec me `imagePullSecrets: [{name: regcred}]`.

YAML me `data:` (base64) ya `stringData:` (plain text, API server encode kar deta hai).

## 3. Environment variables (summary)

- `env[].value` : direct
- `env[].valueFrom` : `configMapKeyRef`, `secretKeyRef`, `fieldRef`, `resourceFieldRef`
- `envFrom` : poora ConfigMap/Secret

Downward API (Pod ki apni info):
```yaml
env:
- name: MY_POD_NAME
  valueFrom:
    fieldRef:
      fieldPath: metadata.name
- name: MY_CPU_LIMIT
  valueFrom:
    resourceFieldRef:
      containerName: app
      resource: limits.cpu
```

## 4. Volumes (Pod level)

| Type | Use |
|------|-----|
| `emptyDir` | Pod ke life tak temp storage; containers ke beech sharing. `medium: Memory` se tmpfs |
| `hostPath` | Node ki directory (risky, sirf special cases/testing) |
| `configMap`, `secret` | config/secret files |
| `persistentVolumeClaim` | durable storage (04-storage) |
| `projected` | configMap + secret + downwardAPI + serviceAccountToken ek jagah |

```yaml
volumes:
- name: cache
  emptyDir: {}
containers:
- name: app
  volumeMounts:
  - name: cache
    mountPath: /cache
```

## 5. Probes

Kubelet container ki health check karta hai.

| Probe | Sawal | Fail hone par |
|-------|-------|---------------|
| `startupProbe` | App start ho gaya? (slow apps) | Container restart; iske pass hone tak baaki probes disabled |
| `readinessProbe` | Traffic le sakta hai? | Service endpoints se **hata deta hai**, restart nahi |
| `livenessProbe` | Zinda hai ya hang? | Container **restart** |

Mechanisms: `httpGet`, `tcpSocket`, `exec`, `grpc`.

```yaml
containers:
- name: app
  image: nginx
  readinessProbe:
    httpGet: {path: /ready, port: 80}
    initialDelaySeconds: 5
    periodSeconds: 10
  livenessProbe:
    httpGet: {path: /healthz, port: 80}
    initialDelaySeconds: 15
    periodSeconds: 20
    failureThreshold: 3
  startupProbe:
    httpGet: {path: /healthz, port: 80}
    failureThreshold: 30
    periodSeconds: 10
```

Tunables: `initialDelaySeconds`, `periodSeconds`, `timeoutSeconds`, `successThreshold`, `failureThreshold`.

Aam galti: liveness probe bahut aggressive rakhna -> app restart loop me.

## 6. Resource requests aur limits

```yaml
resources:
  requests:
    cpu: 250m          # 0.25 core
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```

- **requests** = scheduler ke liye guaranteed minimum. Node ke allocatable me fit hona chahiye warna Pod `Pending`.
- **limits** = runtime cap. CPU limit cross -> **throttle**. Memory limit cross -> **OOMKilled**.
- Units: CPU `1 = 1 core`, `100m = 0.1`; memory `Mi/Gi` (binary) vs `M/G` (decimal).

### QoS classes
- **Guaranteed**: har container me requests == limits (cpu aur memory dono)
- **Burstable**: kam se kam ek request/limit set, par Guaranteed nahi
- **BestEffort**: koi request/limit nahi. Memory pressure me sabse pehle evict.

### Namespace level controls
- **LimitRange**: per-container/pod default aur min/max
- **ResourceQuota**: poore namespace ka total cap (11-cluster-operations)

```bash
k top pod ; k top node        # metrics-server chahiye
```

## Common mistakes

- Secret ko encrypted samajhna.
- ConfigMap badal ke expect karna ki env vars auto-update honge.
- `readinessProbe` path galat -> Pod `Running` par `0/1 READY`.
- Requests node se badi rakhna -> Pending (`Insufficient cpu/memory`).
- Memory limit bahut kam -> `OOMKilled` (`k describe pod` me `Last State: Terminated, Reason: OOMKilled`).

## Practice labs

1. ConfigMap se env aur file dono tareeke se Pod me inject karo, `k exec` se verify karo.
2. Secret banao aur `base64 -d` se decode karke verify karo.
3. Aisa Pod banao jo Guaranteed QoS me aaye. `k describe` me QoS Class check karo.
4. Readiness probe fail karke dekho Pod `Ready` kyun nahi hota aur Service endpoints me kyun nahi aata.
5. Container ko 10Mi memory limit do aur memory khane wala process chalakar `OOMKilled` dekho.
