# CKA (Certified Kubernetes Administrator) - Study Pack

Ye pack CKA exam ke liye hai. Har topic ka apna folder hai aur uske andar `README.md`.

## Folder index

| # | Folder | Topics |
|---|--------|--------|
| 01 | [01-core-objects](01-core-objects/README.md) | Pod, ReplicaSet, Deployment, StatefulSet, DaemonSet, Job, CronJob, Namespace |
| 02 | [02-configuration](02-configuration/README.md) | ConfigMap, Secret, env, volumes, probes, requests, limits |
| 03 | [03-networking](03-networking/README.md) | Service types, DNS, CNI, NetworkPolicy |
| 04 | [04-storage](04-storage/README.md) | PV, PVC, StorageClass, CSI, access modes, reclaim policies |
| 05 | [05-scheduling](05-scheduling/README.md) | Labels, taints, tolerations, affinity, topology spread, priority |
| 06 | [06-ingress-and-gateway](06-ingress-and-gateway/README.md) | Ingress, TLS, routing, Gateway API |
| 07 | [07-security](07-security/README.md) | RBAC, ServiceAccounts, PSS, securityContext, admission |
| 08 | [08-scaling-and-availability](08-scaling-and-availability/README.md) | HPA, VPA, Cluster Autoscaler, PDB, rollouts |
| 09 | [09-helm-and-packaging](09-helm-and-packaging/README.md) | Helm charts, values, releases, Kustomize |
| 10 | [10-troubleshooting](10-troubleshooting/README.md) | describe/logs/exec/events, common errors |
| 11 | [11-cluster-operations](11-cluster-operations/README.md) | kubeadm upgrade, etcd backup/restore, drain, quotas |
| 12 | [12-managed-kubernetes](12-managed-kubernetes/README.md) | EKS, AKS, GKE concepts |
| 13 | [13-gitops-readiness](13-gitops-readiness/README.md) | Desired state, reconciliation, drift, secrets |

## CKA exam ka structure (domains aur weightage)

Weightage curriculum ke hisaab se hai. Exam se pehle **official CNCF curriculum PDF** se current percentages aur Kubernetes version zaroor verify kar lena, kyunki ye time ke saath change hote hain.

| Domain | Approx. weight | Is pack me kahan |
|--------|---------------|------------------|
| Cluster Architecture, Installation & Configuration | ~25% | 07 (RBAC), 09 (Helm/Kustomize), 11 (kubeadm, etcd) |
| Workloads & Scheduling | ~15% | 01, 02, 05, 08 |
| Services & Networking | ~20% | 03, 06 |
| Storage | ~10% | 04 |
| Troubleshooting | ~30% | 10 (sabse zyada weight, sabse zyada practice) |

- Exam **hands-on, performance-based** hota hai (terminal me tasks solve karne hote hain), MCQ nahi.
- Duration aam taur par **2 ghante**, passing score aam taur par **66%**.
- Exam me `kubernetes.io/docs` (aur kuch allowed official links) khol sakte ho. Docs search karna practice karo.
- Topic 12 (Managed K8s) aur 13 (GitOps) CKA exam ke core me nahi hain, par real-world kaam aur interviews ke liye zaroori hain.

## Exam day setup (pehle 2 minute me ye karo)

```bash
alias k=kubectl
source <(kubectl completion bash)
complete -o default -F __start_kubectl k
export do="--dry-run=client -o yaml"
export now="--force --grace-period 0"

# vim ke liye ~/.vimrc
cat >> ~/.vimrc <<'EOF'
set tabstop=2 shiftwidth=2 expandtab
set number
EOF
```

(Exam environment me alias/completion pehle se set ho sakta hai. Check kar lo.)

Use:
```bash
k run nginx --image=nginx $do > pod.yaml
k delete pod nginx $now
```

## Time management

- Har question ka weightage dikhta hai. Heavy questions pehle, easy ones turant.
- 5 minute se zyada atak jao to **flag** karo aur aage badho.
- Har task me **context switch** karna na bhulo: question ke upar ek command diya hota hai, e.g. `kubectl config use-context k8s-c1`. Ye hamesha chalao.
- Task ke baad verify karo (`k get`, `k describe`).

## Imperative commands pehle, YAML baad me

Fast kaam ke liye imperative commands use karo:

```bash
k create deployment web --image=nginx --replicas=3
k expose deployment web --port=80 --type=NodePort
k create configmap app --from-literal=K=V
k create secret generic s --from-literal=pw=123
k create role / rolebinding / serviceaccount ...
k create job / cronjob ...
k explain pod.spec.containers --recursive   # field names bhul gaye to
```

## Study order (suggested)

1. 01 Core objects -> 02 Configuration -> 05 Scheduling
2. 03 Networking -> 06 Ingress/Gateway -> 04 Storage
3. 07 Security -> 08 Scaling
4. 11 Cluster operations -> 10 Troubleshooting (isko roz practice karo)
5. 09 Helm/Kustomize, phir 12 aur 13 for real-world context

## Practice environments

- `kind` ya `minikube` (local), ya `kubeadm` se multi-node VM cluster (kubeadm practice ke liye best).
- killer.sh simulator (CKA exam purchase ke saath aata hai) and killercoda.com free labs.
- Kubernetes docs ke tasks khud type karke chalao, copy-paste ke bajay.
