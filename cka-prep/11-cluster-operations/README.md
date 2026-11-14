# 11 - Cluster Operations

Topics: Upgrade, backup/restore etcd, node maintenance, resource quotas

## Control plane architecture (refresher)

**Control plane**: `kube-apiserver`, `etcd`, `kube-scheduler`, `kube-controller-manager` (+ cloud-controller-manager).
**Node**: `kubelet`, `kube-proxy`, container runtime (containerd/CRI-O).

HA control plane: 3 (ya 5) control-plane nodes, stacked etcd (har control-plane node pe) ya external etcd, aage load balancer.

## 1. kubeadm: cluster bootstrap (concept)

```bash
# control-plane
kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=<ip>
mkdir -p $HOME/.kube && cp /etc/kubernetes/admin.conf $HOME/.kube/config
# CNI install (Calico/Flannel/Cilium) karo

# worker
kubeadm join <cp-ip>:6443 --token <t> --discovery-token-ca-cert-hash sha256:<h>

# naya join command (token expire hone par)
kubeadm token create --print-join-command
```
Prerequisites: swap off, containerd, `kubeadm/kubelet/kubectl` same minor, `br_netfilter`/`overlay` modules, `net.ipv4.ip_forward=1`, ports open.

## 2. Cluster upgrade (kubeadm) - EXAM ME PAKKA AATA HAI

Rules:
- Ek baar me **sirf ek minor version** (1.30 -> 1.31 -> 1.32).
- Order: **control plane pehle, phir workers**. Kubelet apiserver se naya nahi ho sakta.
- Version skew: kubelet apiserver se max 3 minor purana (n-3 in recent versions), kubectl +-1.

### Control plane node
```bash
# 1. Package repo me naya version available hai? (docs ke hisaab se repo version badlo)
apt update
apt-cache madison kubeadm

# 2. kubeadm upgrade
apt-mark unhold kubeadm
apt-get install -y kubeadm=1.31.x-*
apt-mark hold kubeadm
kubeadm version
kubeadm upgrade plan
kubeadm upgrade apply v1.31.x

# 3. Node drain karo (control-plane ko apne se drain)
kubectl drain <cp-node> --ignore-daemonsets

# 4. kubelet + kubectl upgrade
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.31.x-* kubectl=1.31.x-*
apt-mark hold kubelet kubectl
systemctl daemon-reload
systemctl restart kubelet

# 5. Uncordon
kubectl uncordon <cp-node>
```

### Worker node (ek ek karke)
```bash
# control plane / kubectl wale machine se:
kubectl drain <worker> --ignore-daemonsets --delete-emptydir-data

# worker par SSH:
apt-mark unhold kubeadm && apt-get install -y kubeadm=1.31.x-* && apt-mark hold kubeadm
kubeadm upgrade node                      # (apply nahi, "node")
apt-mark unhold kubelet kubectl && apt-get install -y kubelet=1.31.x-* kubectl=1.31.x-* && apt-mark hold kubelet kubectl
systemctl daemon-reload && systemctl restart kubelet

# wapas control plane se:
kubectl uncordon <worker>
kubectl get nodes
```
Additional control-plane nodes (HA) par: `kubeadm upgrade node` (pehle node par `apply`).

Upgrade se pehle: **etcd backup**, release notes/deprecations padho, PDB check.

## 3. etcd backup aur restore - EXAM ME PAKKA AATA HAI

etcd cluster ka saara state (objects) store karta hai.

### Info nikaalna
```bash
# etcd static pod ke args se cert paths
k -n kube-system describe pod etcd-<node>        # ya /etc/kubernetes/manifests/etcd.yaml
# --cert-file, --key-file, --trusted-ca-file, --data-dir, --listen-client-urls
ls /etc/kubernetes/pki/etcd/
```

### Backup (snapshot)
```bash
ETCDCTL_API=3 etcdctl snapshot save /opt/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

etcdutl snapshot status /opt/etcd-backup.db -w table     # verify (ya etcdctl snapshot status)
```
(`etcdctl` na ho to `apt install etcd-client` ya etcd container se.)

### Restore
```bash
# 1. snapshot ko NAYE data dir me restore (etcd restore me certs/endpoints nahi chahiye)
etcdutl snapshot restore /opt/etcd-backup.db --data-dir=/var/lib/etcd-restored
# (purane tareeke: ETCDCTL_API=3 etcdctl snapshot restore ... --data-dir=...)

# 2. etcd static Pod manifest me data-dir badlo
vim /etc/kubernetes/manifests/etcd.yaml
#   volumes:
#   - hostPath:
#       path: /var/lib/etcd-restored       # <- naya path
#       type: DirectoryOrCreate
#     name: etcd-data
#   (container ke --data-dir flag ko hostPath ke mount path ke saath consistent rakho)

# 3. kubelet etcd Pod ko restart karega; wait karo
crictl ps | grep etcd
k get pods -n kube-system
```
Tips:
- Restore ke baad ownership/permissions dhyan se (etcd user).
- Restored cluster me purane object states aa jaate hain, backup ke baad ke changes gayab.
- **External etcd** me alag `systemctl stop etcd`, restore, `--data-dir` systemd unit me update, `systemctl start etcd`.
- Backup regularly (cron) aur off-node store karo.

## 4. Node maintenance

```bash
k cordon <node>                    # unschedulable mark (existing Pods chalte rahenge)
k drain <node> --ignore-daemonsets --delete-emptydir-data   # Pods evict + cordon
k drain <node> --ignore-daemonsets --force                  # bare Pods (controller ke bina) bhi delete
k uncordon <node>
k get nodes
```
- `drain` PDB respect karta hai.
- DaemonSet Pods ignore karne ke liye `--ignore-daemonsets` zaroori.
- Bare Pod (controller ke bina) drain me delete ho jaata hai wapas nahi aata (`--force` ke saath).
- Node permanently hataana: drain -> `kubectl delete node <node>` -> node par `kubeadm reset`.
- Pod-eviction-timeout / node failure ke baad 5 min default tolerations (`tolerationSeconds: 300`).

## 5. Resource Quota aur LimitRange

### ResourceQuota (namespace total cap)
```bash
k create quota team-a --hard=pods=10,requests.cpu=4,requests.memory=8Gi,limits.cpu=8,limits.memory=16Gi -n team-a
k describe quota -n team-a
```
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a
  namespace: team-a
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    persistentvolumeclaims: "5"
    services.nodeports: "2"
    count/deployments.apps: "10"
```
- Quota me compute resources ho to us namespace me Pods ko requests/limits **dene hi padte hain** (ya LimitRange defaults), warna Pod create reject.
- Scopes: `Terminating`, `NotTerminating`, `BestEffort`, `PriorityClass`.

### LimitRange (per-object defaults aur bounds)
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limits
  namespace: team-a
spec:
  limits:
  - type: Container
    default:           {cpu: 500m, memory: 256Mi}     # default limit
    defaultRequest:    {cpu: 250m, memory: 128Mi}     # default request
    max:               {cpu: "1", memory: 512Mi}
    min:               {cpu: 50m, memory: 32Mi}
```

## 6. Certificates

```bash
kubeadm certs check-expiration
kubeadm certs renew all                    # phir control plane static Pods restart
ls /etc/kubernetes/pki
```
kubeadm certs 1 saal valid; upgrade karne par renew ho jaate hain. CA 10 saal.

## 7. Other ops tasks

- **kubeconfig**: `/etc/kubernetes/admin.conf`
- **Kubelet config**: `/var/lib/kubelet/config.yaml`; **runtime**: containerd config `/etc/containerd/config.toml` (SystemdCgroup = true).
- **Logs**: `journalctl -u kubelet`, `/var/log/pods`, `/var/log/containers`.
- **Namespace/RBAC/quota** per team.
- **Add node**: `kubeadm token create --print-join-command`.
- **Backup**: etcd + important manifests (+ PV data via Velero, concept).
- **HA/Failure**: multiple API servers behind LB; etcd quorum `(n/2)+1` (3 nodes = 1 failure tolerate, 5 nodes = 2).

## Common mistakes

- Worker par `kubeadm upgrade apply` chalana (sahi: `kubeadm upgrade node`).
- Dono (kubeadm aur kubelet) upgrade bhul jana; sirf kubeadm karke chhod dena.
- Minor version skip karna.
- etcd restore ke baad hostPath update na karna.
- Cordon karke uncordon bhool jana.
- Backup file ko manifests directory ke andar rakhna.

## Practice labs

1. 2-3 node kubeadm cluster banao (VM/vagrant), ek minor version upgrade karo (control plane + worker).
2. etcd snapshot lo, kuch objects delete karo, restore karke wapas aate dekho.
3. Worker drain/uncordon, PDB ke saath drain behaviour dekho.
4. ResourceQuota lagao, quota cross karke error dekho; LimitRange defaults ke saath Pod banao.
5. `kubeadm certs check-expiration` aur renew practice karo.
