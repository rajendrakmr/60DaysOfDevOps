# 03 - Networking

Topics: Service types, ClusterIP, NodePort, LoadBalancer, DNS, CNI, NetworkPolicy

## Kubernetes network model (4 rules)

1. Har Pod ko **apna unique IP** milta hai.
2. Saare Pods bina NAT ke ek dusre se baat kar sakte hain (kisi bhi node par).
3. Nodes aur Pods bina NAT ke baat kar sakte hain.
4. Pod ko apna IP wahi dikhta hai jo dusre use dekhte hain.

Ye model **CNI plugin** implement karta hai.

## 1. CNI (Container Network Interface)

Plugin jo Pod ko network deta hai (IP assign, routes). Examples: Calico, Cilium, Flannel, Weave, Antrea.

- Config: `/etc/cni/net.d/`, binaries: `/opt/cni/bin/`
- CNI install nahi hai to nodes `NotReady` aur CoreDNS `Pending/ContainerCreating` rahega.
- **Flannel NetworkPolicy support nahi karta**; Calico/Cilium karte hain.
- Pod CIDR kubeadm me: `kubeadm init --pod-network-cidr=...`. Service CIDR: `--service-cidr`.

```bash
ls /etc/cni/net.d/
k get pods -n kube-system -o wide
k cluster-info dump | grep -m2 -E "cluster-cidr|service-cluster-ip-range"
```

## 2. Service

Pods ephemeral hain aur IP badalte hain. Service ek **stable virtual IP + DNS name** deta hai aur label **selector** se Pods me load-balance karta hai. Backend list: **EndpointSlice** (purana: Endpoints).

```bash
k expose deployment web --port=80 --target-port=8080 --name=web-svc
k expose pod nginx --port=80 --type=NodePort
k get svc,ep
k get endpointslices
```

### Types

| Type | Kya karta hai |
|------|----------------|
| **ClusterIP** (default) | Sirf cluster ke andar virtual IP |
| **NodePort** | Har node par ek port (30000-32767 default), `NodeIP:NodePort`. ClusterIP bhi milta hai |
| **LoadBalancer** | Cloud provider ka external LB (NodePort + ClusterIP upar). Bare-metal me MetalLB chahiye |
| **ExternalName** | DNS CNAME alias (e.g. `db.example.com`), koi proxying nahi |
| **Headless** (`clusterIP: None`) | Virtual IP nahi; DNS seedhe Pod IPs return karta hai. StatefulSet ke saath |

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80          # Service port
    targetPort: 8080  # container port
    nodePort: 30080   # optional
```

Selector ke bina Service banao to Endpoints manually banane padte hain (external backend ke liye).

Extra: `sessionAffinity: ClientIP`, `externalTrafficPolicy: Local` (source IP preserve), named `targetPort`.

### kube-proxy
Har node par chalta hai (DaemonSet), Service IP -> Pod IP rules banata hai. Modes: `iptables` (default), `ipvs`, (nftables newer versions me). Kuch CNI (Cilium) kube-proxy replace kar sakte hain.

## 3. DNS (CoreDNS)

CoreDNS `kube-system` me Deployment (Service name `kube-dns`) hai. Config: ConfigMap `coredns` (Corefile).

DNS names:
- Service: `<svc>.<ns>.svc.cluster.local` (same namespace me sirf `<svc>`)
- Pod: `<pod-ip-dashes>.<ns>.pod.cluster.local`
- StatefulSet Pod (headless): `<pod>.<svc>.<ns>.svc.cluster.local`

Pod ki `/etc/resolv.conf`: `nameserver <kube-dns ClusterIP>`, `search <ns>.svc.cluster.local svc.cluster.local cluster.local`, `ndots:5`.

```bash
k run dns-test --image=busybox:1.28 --rm -it --restart=Never -- nslookup web-svc
k run dns-test --image=busybox:1.28 --rm -it --restart=Never -- nslookup web-svc.default.svc.cluster.local
k get cm coredns -n kube-system -o yaml
k logs -n kube-system -l k8s-app=kube-dns
```
Pod `dnsPolicy`: `ClusterFirst` (default), `Default`, `None` (+ `dnsConfig`), `ClusterFirstWithHostNet`.

## 4. NetworkPolicy

Default me **sab Pods ek dusre se baat kar sakte hain**. NetworkPolicy jab kisi Pod ko select karti hai to us Pod pe sirf **allowed** traffic jaata/aata hai. Enforcement CNI karta hai.

Rules:
- `podSelector` : policy kin Pods pe lagegi (`{}` = namespace ke saare)
- `policyTypes`: `Ingress`, `Egress`
- Multiple policies **additive** (union) hoti hain.
- `from`/`to` me: `podSelector`, `namespaceSelector`, `ipBlock`.

**Default deny all ingress:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: prod
spec:
  podSelector: {}
  policyTypes: ["Ingress"]
```

**db Pods me sirf backend se (port 5432) allow:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow-backend
spec:
  podSelector:
    matchLabels: {app: db}
  policyTypes: ["Ingress"]
  ingress:
  - from:
    - podSelector:
        matchLabels: {app: backend}
    ports:
    - protocol: TCP
      port: 5432
```

**Important gotcha (exam favourite):**
```yaml
from:
- namespaceSelector: {matchLabels: {team: a}}
  podSelector:   {matchLabels: {role: client}}     # AND (ek hi list item)
```
vs
```yaml
from:
- namespaceSelector: {matchLabels: {team: a}}
- podSelector:       {matchLabels: {role: client}} # OR (alag items)
```

Egress deny karo to **DNS (port 53 UDP/TCP)** allow karna mat bhulo.

Namespace selector ke liye label chahiye: `kubernetes.io/metadata.name=<ns>` automatically hota hai.

## Debug commands

```bash
k get svc,ep -n <ns>                      # Endpoints khaali? selector/labels check karo
k describe svc web-svc
k run curl --image=curlimages/curl --rm -it --restart=Never -- curl -s web-svc:80
k port-forward svc/web-svc 8080:80
k exec <pod> -- nslookup kubernetes.default
k get netpol -A ; k describe netpol <name>
```

## Common mistakes

- Service selector aur Pod labels mismatch -> endpoints empty.
- `port` aur `targetPort` confuse karna.
- NetworkPolicy lagayi par CNI (Flannel) support nahi karta.
- Egress policy me DNS allow na karna.
- AND vs OR selector ki indentation.

## Practice labs

1. Deployment expose karo ClusterIP, NodePort aur LoadBalancer type se, difference dekho.
2. Dusre namespace ke Pod se `svc.ns.svc.cluster.local` se access karo.
3. Default-deny + specific allow policy lagao aur `curl` se test karo.
4. CoreDNS Pods scale/delete karke dekho DNS pe kya asar padta hai.
5. Headless Service banao aur `nslookup` se Pod IPs dekho.
