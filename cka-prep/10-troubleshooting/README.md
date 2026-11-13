# 10 - Troubleshooting

Topics: kubectl describe/logs/exec/events, CrashLoopBackOff, ImagePullBackOff, DNS, pending pods

CKA me **~30% weightage** isi ka hai. Roz practice karo.

## Golden workflow

1. **Kya toota hai?** (`k get ... -o wide`)
2. **Kyun?** (`k describe`, **Events**)
3. **App ka haal?** (`k logs`, `--previous`)
4. **Andar jaake dekho** (`k exec`, debug pod)
5. **Fix** -> **verify** (`k get`, `curl`)

Top-down: Application -> Pod -> Service/Network -> Node -> Control plane.

## 1. Core kubectl tools

```bash
k get pods -A -o wide
k get pod <pod> -o yaml
k describe pod <pod>                       # niche Events section sabse important
k get events -A --sort-by=.metadata.creationTimestamp
k get events --field-selector type=Warning
k logs <pod> [-c container] [-f] [--previous] [--tail=50] [--since=10m]
k logs -l app=web --all-containers
k exec -it <pod> -- sh
k exec <pod> -c <container> -- cat /etc/config/app.conf
k debug -it <pod> --image=busybox --target=<container>      # ephemeral container
k debug node/<node> -it --image=ubuntu                       # node debug
k top pod / node
k get pod <pod> -o jsonpath='{.status.containerStatuses[*].lastState}'
k explain pod.spec
k api-resources
```

## 2. Pod status errors

### Pending
Scheduler ne node nahi diya (ya volume bind nahi hua). `k describe pod` ka Events dekho:
- `Insufficient cpu/memory` -> requests kam karo ya node add
- `node(s) had untolerated taint` -> toleration/taint
- `didn't match Pod's node affinity/selector` -> labels check
- `unbound immediate PersistentVolumeClaims` -> PVC Pending (storageClass, size, access mode)
- Scheduler hi down: `k get pods -n kube-system` (static pod manifest check)
- Namespace quota: `exceeded quota`

### ImagePullBackOff / ErrImagePull
- Image name/tag galat (typo), registry unreachable
- Private registry ka `imagePullSecrets` missing/galat
- Rate limit (Docker Hub)
Fix: `k describe pod` (Failed to pull image ...), `k set image ...`, `k create secret docker-registry`.

### CrashLoopBackOff
Container start hota hai, crash hota hai, kubelet backoff ke saath restart karta hai.
```bash
k logs <pod> --previous
k describe pod <pod>      # Exit Code, Reason, Last State
```
Exit codes: `1` app error, `137` SIGKILL (OOMKilled ya liveness fail), `139` segfault, `143` SIGTERM, `127` command not found.
Causes: galat command/args, missing env/ConfigMap/Secret, liveness probe galat, memory limit kam (OOMKilled), file permission, dependency (DB) ready nahi.

### CreateContainerConfigError
ConfigMap/Secret missing ya key galat (`describe` me clearly likha hota hai).

### ContainerCreating (atka hua)
Volume mount fail, CNI problem, image pull ho raha hai, secret/configmap volume missing.

### Running par 0/1 READY
Readiness probe fail. Probe path/port check karo.

### Evicted / OOMKilled / Terminating (stuck)
- Evicted: node pressure (disk/memory). `k describe node`.
- Stuck Terminating: finalizers ya node down. `k delete pod x --force --grace-period=0` (sochke).

## 3. Service / Network troubleshooting

```bash
k get svc,ep <svc>                                  # endpoints khaali = selector/labels ya readiness issue
k describe svc <svc>
k get pods --show-labels
k run t --image=curlimages/curl --rm -it --restart=Never -- curl -sv <svc>:<port>
k run t --image=busybox:1.28 --rm -it --restart=Never -- nslookup <svc>
k get netpol -A
```
Checklist:
1. Service selector == Pod labels?
2. `targetPort` == container port?
3. Pods Ready?
4. NetworkPolicy block to nahi kar rahi?
5. kube-proxy Pods chal rahe hain? (`k get ds kube-proxy -n kube-system`)
6. CNI healthy?

### DNS
```bash
k get pods -n kube-system -l k8s-app=kube-dns
k logs -n kube-system -l k8s-app=kube-dns
k get svc kube-dns -n kube-system ; k get ep kube-dns -n kube-system
k exec <pod> -- cat /etc/resolv.conf
k get cm coredns -n kube-system -o yaml
```
Causes: CoreDNS Pods down, Corefile galat, NetworkPolicy egress 53 block, node `resolv.conf` loop, CNI problem. Dusre namespace ke liye FQDN use karo.

## 4. Node troubleshooting

```bash
k get nodes
k describe node <node>            # Conditions: Ready, MemoryPressure, DiskPressure, PIDPressure; Events; Taints
```
`NotReady` hone par node par SSH karke:
```bash
systemctl status kubelet
journalctl -u kubelet -f --no-pager | tail -100
systemctl status containerd         # runtime
crictl ps -a ; crictl logs <id> ; crictl pods
df -h ; free -m ; top
cat /var/lib/kubelet/config.yaml
cat /etc/kubernetes/kubelet.conf
```
Common causes: kubelet stopped/crashed (`systemctl start kubelet`, `enable`), kubelet config/path galat (`/etc/systemd/system/kubelet.service.d/10-kubeadm.conf`), containerd down, certificate expired, swap on (kubelet fail), CNI missing, node ka network/firewall, disk full, kubelet API server se connect nahi kar pa raha (galat `server:` in kubelet.conf).

## 5. Control plane troubleshooting

kubeadm cluster me control plane **static Pods** hain: `/etc/kubernetes/manifests/`
- `kube-apiserver.yaml`, `kube-controller-manager.yaml`, `kube-scheduler.yaml`, `etcd.yaml`

```bash
k get pods -n kube-system
k logs -n kube-system kube-apiserver-<node>
crictl ps -a | grep kube-apiserver               # API server down ho to kubectl nahi chalega
crictl logs <container-id>
journalctl -u kubelet | grep -i apiserver
ls /etc/kubernetes/manifests ; ls /etc/kubernetes/pki
```
Symptoms -> cause:
- `kubectl` "connection refused": API server down (manifest typo, galat cert path, etcd unreachable), ya kubeconfig me galat server/port.
- Naye Pods `Pending` par koi event nahi: **scheduler** down.
- Deployment replicas create nahi hote: **controller-manager** down.
- Manifest me typo: flags, volume paths (`hostPath`) ya cert filenames. Kubelet manifest dobara apply karega (file save karne par auto restart).
- etcd issues: `crictl logs`, data-dir path, cert paths.

Cert check: `kubeadm certs check-expiration`, `openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text | grep -A2 Validity`.

## 6. Application aur resource issues

- Logs me error -> config/env verify (`k exec ... env`).
- Resource: `k top`, `k describe node` (Allocated resources).
- Quota/LimitRange: `k describe quota -n ns`, `k describe limitrange -n ns`.
- RBAC: `Forbidden` -> `k auth can-i ... --as ...`.
- Storage: PVC `Pending`, mount errors in events.
- Multi-container: `-c` flag lagana mat bhulo.

## 7. Exam tips

- Pehla command hamesha `k describe`/`k get events`.
- Fix ke baad **verify karo** (Pod Running/Ready, curl 200).
- Control-plane fix ke baad 1-2 minute lag sakte hain, `watch k get pods -n kube-system`.
- YAML edit se pehle backup: `cp kube-apiserver.yaml /tmp/` (manifests folder se bahar!).
- Static Pod ka backup file manifests folder ke andar mat rakho.

## Quick reference table

| Symptom | Pehle check karo |
|---------|------------------|
| Pending | describe events: resources/taints/PVC |
| ImagePullBackOff | image name, secret |
| CrashLoopBackOff | logs --previous, exit code |
| 0/1 Ready | readiness probe, ports |
| Service not reachable | endpoints, labels, targetPort |
| DNS fail | CoreDNS Pods/logs, netpol |
| Node NotReady | kubelet status/journal, containerd |
| kubectl timeout | apiserver static pod, kubeconfig |
| New pods don't schedule | scheduler Pod |

## Practice labs (break and fix)

1. Deployment me image typo karo -> ImagePullBackOff -> fix.
2. Liveness probe galat port pe daalo -> restart loop dekho.
3. Service selector me typo -> endpoints empty -> fix.
4. Worker node par `systemctl stop kubelet` -> NotReady -> recover.
5. `kube-scheduler.yaml` me galat flag/typo daalo -> Pods Pending -> fix.
6. `kube-apiserver.yaml` me galat etcd cert path -> API down -> `crictl` se debug karke fix.
7. CoreDNS Deployment scale 0 karo -> DNS fail -> recover.
