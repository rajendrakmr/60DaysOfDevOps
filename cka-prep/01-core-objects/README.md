# 01 - Core Objects

Topics: Pod, ReplicaSet, Deployment, StatefulSet, DaemonSet, Job, CronJob, Namespace

## 1. Pod

Pod Kubernetes ki sabse chhoti deployable unit hai. Ek Pod me ek ya zyada containers hote hain jo:
- same **network namespace** share karte hain (ek IP, `localhost` pe ek dusre se baat kar sakte hain)
- same **volumes** share kar sakte hain
- same node pe schedule hote hain

Pod ephemeral hai: mar gaya to wahi Pod wapas nahi aata. Isliye hum Deployment jaise controllers use karte hain.

### Pod types / patterns
- **Single container**: sabse common.
- **Multi-container**: sidecar (log shipper, proxy), adapter, ambassador.
- **Init containers**: main containers se pehle, ek ke baad ek, sequence me chalte hain. Fail hue to Pod aage nahi badhta. DB ready hone ka wait, config download, etc.
- **Static Pods**: kubelet directly manage karta hai (API server nahi). Manifest folder: `/etc/kubernetes/manifests` (kubeadm). Control plane components (kube-apiserver, etcd, scheduler, controller-manager) isi tarah chalte hain. Exam me kaafi aata hai.

### Pod lifecycle phases
`Pending` -> `Running` -> `Succeeded` / `Failed` (aur `Unknown`). Container states: `Waiting`, `Running`, `Terminated`.

`restartPolicy`: `Always` (default), `OnFailure`, `Never`.

### Commands
```bash
k run nginx --image=nginx
k run nginx --image=nginx --port=80 --labels="app=web,tier=fe" $do > pod.yaml
k run busybox --image=busybox --restart=Never -- sleep 3600
k run tmp --image=busybox --rm -it --restart=Never -- sh      # temporary debug pod
k get pods -o wide
k get pod nginx -o yaml
k describe pod nginx
k logs nginx                  # -c <container> multi-container me, -f follow, --previous
k exec -it nginx -- sh        # -c <container>
k delete pod nginx $now
k edit pod nginx              # sirf limited fields editable (image, activeDeadlineSeconds, tolerations)
```

### Multi-container + init container YAML
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi
spec:
  initContainers:
  - name: init-wait
    image: busybox
    command: ["sh", "-c", "until nslookup myservice; do sleep 2; done"]
  containers:
  - name: app
    image: nginx
  - name: sidecar
    image: busybox
    command: ["sh", "-c", "tail -f /var/log/app.log"]
```

## 2. ReplicaSet

Ye ensure karta hai ki **N identical Pods** hamesha chal rahe hon. Pod delete hua to naya bana deta hai. Selector `matchLabels` se Pods pehchanta hai.

Practically seedha ReplicaSet kam banate hain; **Deployment** ise manage karta hai.

```bash
k get rs
k scale rs myrs --replicas=5
```

Dhyan rahe: ReplicaSet ka selector Pod template ke labels se match hona chahiye, warna create fail hota hai.

## 3. Deployment

Stateless apps ke liye default controller. Deployment -> ReplicaSet -> Pods.

Features: rolling update, rollback, scaling, pause/resume.

```bash
k create deployment web --image=nginx:1.25 --replicas=3
k scale deployment web --replicas=5
k set image deployment/web nginx=nginx:1.27
k rollout status deployment/web
k rollout history deployment/web
k rollout undo deployment/web                 # --to-revision=2
k rollout restart deployment/web
k rollout pause deployment/web ; k rollout resume deployment/web
k edit deployment web
```

Strategy:
```yaml
spec:
  strategy:
    type: RollingUpdate          # ya Recreate
    rollingUpdate:
      maxSurge: 25%              # extra Pods allowed during update
      maxUnavailable: 25%        # kitne Pods down ho sakte hain
```
(Detail "08-scaling-and-availability" me.)

Update tabhi trigger hota hai jab `.spec.template` badle (image, env, labels, etc.), sirf replicas badalne se nahi.

## 4. StatefulSet

Stateful apps (DB, Kafka, ZooKeeper) ke liye. Guarantees:
- **Stable names**: `web-0`, `web-1`, `web-2`
- **Ordered** create/delete/scale (default `OrderedReady`; `podManagementPolicy: Parallel` bhi hota hai)
- **Stable storage**: `volumeClaimTemplates` se har Pod ka apna PVC. Pod delete hone par PVC bachta hai.
- **Headless Service** zaroori: `serviceName: web` aur Service me `clusterIP: None`. DNS: `web-0.web.<ns>.svc.cluster.local`.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: web
  replicas: 3
  selector:
    matchLabels: {app: web}
  template:
    metadata:
      labels: {app: web}
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

Update strategy: `RollingUpdate` (reverse ordinal order me) with `partition` for canary.

## 5. DaemonSet

Har node (ya selected nodes) pe **ek Pod**. Use cases: log agent (fluentd), monitoring (node-exporter), CNI, kube-proxy.

- Naya node aaye to Pod automatically bante hain.
- Control-plane nodes pe chalana ho to **toleration** chahiye (taint `node-role.kubernetes.io/control-plane:NoSchedule`).
- Subset ke liye `nodeSelector` / affinity.

Exam trick: DaemonSet ka imperative command nahi hai. Deployment ka YAML generate karo aur edit karo:
```bash
k create deployment ds --image=nginx $do > ds.yaml
# kind: Deployment -> DaemonSet; replicas aur strategy hatao
```

## 6. Job

Run-to-completion task. Pod successfully khatam ho jaye tak retry karta hai.

```bash
k create job pi --image=perl -- perl -Mbignum=bpi -wle 'print bpi(2000)'
```

Important fields:
- `completions`: kitne successful Pods chahiye
- `parallelism`: ek saath kitne chalenge
- `backoffLimit`: retry limit (default 6)
- `activeDeadlineSeconds`: total time limit
- `ttlSecondsAfterFinished`: finished Job auto-delete
- `restartPolicy`: `OnFailure` ya `Never` (Always allowed nahi)

## 7. CronJob

Schedule par Job banata hai (cron syntax, `min hour dom mon dow`).

```bash
k create cronjob hello --image=busybox --schedule="*/5 * * * *" -- echo hi
k create job manual-run --from=cronjob/hello      # CronJob se turant Job
```

Fields: `concurrencyPolicy` (`Allow`/`Forbid`/`Replace`), `successfulJobsHistoryLimit`, `failedJobsHistoryLimit`, `startingDeadlineSeconds`, `suspend`.

## 8. Namespace

Cluster ke andar logical isolation: names, RBAC, ResourceQuota, NetworkPolicy namespace scope me lagte hain.

Default namespaces: `default`, `kube-system`, `kube-public`, `kube-node-lease`.

```bash
k create ns dev
k get pods -n dev
k get pods -A                    # all namespaces
k config set-context --current --namespace=dev
```

- Namespaced vs cluster-scoped: `k api-resources --namespaced=true|false`. Node, PV, StorageClass, Namespace, ClusterRole cluster-scoped hain.
- Cross-namespace DNS: `svc.namespace.svc.cluster.local`.
- Namespace delete karne par uske andar sab kuch delete ho jata hai.

## Common mistakes

- Namespace `-n` bhul jana (galat namespace me object ban gaya).
- Pod ko `k edit` karke unsupported field badalna (Deployment edit karo).
- StatefulSet ke saath headless Service na banana.
- Job me `restartPolicy: Always` likhna (invalid).
- Selector aur template labels match na hona.

## Practice labs

1. 3 replica Deployment `nginx:1.24` banao, `1.27` pe update karo, rollback karo.
2. Ek Pod banao jisme init container `/work/ready` file banaye aur main container use check kare.
3. 3-replica StatefulSet + headless Service banao, `web-0` delete karo aur dekho wahi naam wapas aata hai, data bacha rehta hai.
4. DaemonSet banao jo control-plane nodes pe bhi chale.
5. Har minute chalne wala CronJob banao, 3 history jobs rakho.
6. `k explain pod.spec --recursive | less` se fields explore karo.
