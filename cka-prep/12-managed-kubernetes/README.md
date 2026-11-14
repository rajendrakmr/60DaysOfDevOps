# 12 - Managed Kubernetes

Topics: EKS, AKS, GKE concepts, IAM integration, node pools, managed control plane

> Ye topic CKA exam ke core me nahi hai (exam kubeadm-style self-managed clusters pe hota hai), par real-world kaam, interviews aur cloud certifications me zaroori hai.

## 1. Managed Kubernetes kya hai?

Cloud provider **control plane** (API server, etcd, scheduler, controller-manager) manage karta hai: HA, patching, backup, upgrades. Aap **worker nodes aur workloads** manage karte ho (kitna ye service ke mode par nirbhar hai).

| Layer | Self-managed (kubeadm) | Managed (EKS/AKS/GKE) |
|-------|------------------------|------------------------|
| Control plane | Aap | Provider |
| etcd backup | Aap | Provider |
| Control plane upgrade | Aap (`kubeadm upgrade`) | Provider (aap trigger karte ho) |
| Worker nodes | Aap | Aap / semi-managed / fully managed (Autopilot, Fargate) |
| Add-ons (CNI, CoreDNS, kube-proxy) | Aap | Managed add-ons ya aap |
| Cost | Infra | Infra + control plane fee (EKS ~ per cluster/hour; AKS free/standard tier; GKE per-cluster fee with free tier) |

> Pricing aur features badalte rehte hain; current details provider docs se verify karo.

## 2. Comparison

| | **EKS (AWS)** | **AKS (Azure)** | **GKE (Google)** |
|---|---------------|-----------------|------------------|
| Node options | Managed node groups, self-managed, **Fargate** (serverless), Karpenter / EKS Auto Mode | Node pools (VMSS), virtual nodes (ACI), Automatic mode | Standard node pools, **Autopilot** (fully managed nodes) |
| IAM -> Pod | **IRSA** (IAM Roles for Service Accounts, OIDC) / **EKS Pod Identity** | **Microsoft Entra Workload ID** | **Workload Identity Federation for GKE** |
| User auth | IAM + `aws-auth` ConfigMap / **EKS access entries** | Microsoft Entra ID (+ Azure RBAC) | Google IAM + K8s RBAC |
| Default CNI | Amazon VPC CNI (Pod = VPC IP) | Azure CNI / kubenet / CNI Overlay / Cilium | VPC-native (alias IP), Dataplane V2 (Cilium) |
| LB / Ingress | AWS Load Balancer Controller (ALB/NLB) | Azure LB / App Gateway (AGIC), App Routing add-on | Cloud Load Balancing, GKE Ingress/Gateway |
| Storage (CSI) | EBS CSI, EFS CSI | Azure Disk, Azure Files CSI | Persistent Disk CSI, Filestore CSI |
| Autoscaling | Cluster Autoscaler, **Karpenter** | Cluster Autoscaler, NAP (Karpenter-based) | Cluster Autoscaler, Node auto-provisioning, Autopilot |
| CLI | `eksctl`, `aws eks` | `az aks` | `gcloud container clusters` |
| Registry | ECR | ACR | Artifact Registry |

## 3. Managed control plane

- Multi-AZ HA, auto-patched, SLA-backed.
- API server endpoint: **public**, **private**, ya dono (security ke liye private + restricted CIDR).
- Control plane logs (audit, authenticator, scheduler) cloud logging me bhej sakte ho (CloudWatch / Azure Monitor / Cloud Logging).
- Aap etcd ya control-plane nodes tak SSH nahi kar sakte; `kubeadm`/static pod troubleshooting yahan applicable nahi.
- Upgrade: provider control plane upgrade karta hai (ek minor at a time), phir aap node pools upgrade karte ho (rolling/surge). Kai providers **release channels / auto-upgrade** dete hain.
- Version support window limited (~14 months standard) -> upgrade plan rakhna padta hai.

## 4. Node pools / node groups

Same configuration (VM size, OS, labels, taints, disk) ke nodes ka group.
- **System pool** (CoreDNS, metrics) vs **User pools** (apps).
- Alag pools: GPU, high-memory, **Spot/Preemptible** (sasta, interrupt ho sakta hai; tolerations + PDB use karo), ARM (Graviton/Ampere).
- Multi-AZ spread, autoscaling min/max, `nodeSelector` / taints se workloads ko pool tak bhejna.
- Upgrade: surge nodes (`maxSurge`), blue/green pool swap.
- Fully managed options: **Fargate / Autopilot / Automatic** (node management nahi, per-Pod billing; kuch features restrict, e.g. DaemonSets/hostPath/privileged).

## 5. IAM integration

Do alag directions:

**A) Humans/CI -> Cluster (kubectl access)**
- AWS: IAM identity -> EKS access entries (ya `aws-auth`) -> K8s RBAC groups. `aws eks update-kubeconfig --name <cluster>`.
- Azure: Entra ID users/groups + Azure RBAC for Kubernetes. `az aks get-credentials`.
- GCP: IAM roles (`container.developer` etc.) + RBAC. `gcloud container clusters get-credentials`.
- Hamesha **least privilege**, groups-based RBAC.

**B) Pods -> Cloud APIs (S3, Key Vault, GCS)**
Static cloud keys Secret me **mat** rakho. Workload identity use karo:
```
Pod -> ServiceAccount (annotated) -> OIDC federation -> Cloud IAM role -> short-lived credentials
```
- EKS IRSA: SA annotation `eks.amazonaws.com/role-arn: arn:aws:iam::<acct>:role/<role>`
- AKS Workload ID: SA annotation `azure.workload.identity/client-id`, Pod label `azure.workload.identity/use: "true"`
- GKE: SA annotation `iam.gke.io/gcp-service-account` (ya direct principal binding)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader
  namespace: app
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/s3-read
```

## 6. Networking aur storage notes

- **Pod IP exhaustion**: EKS VPC CNI me Pod ko VPC IP milta hai (subnet size aur ENI limits plan karo; prefix delegation, custom networking). Azure CNI Overlay IP bachata hai.
- Service type LoadBalancer provider ka LB banata hai (annotations se internal/external, NLB/ALB, etc.).
- Ingress controller / Gateway API ke cloud-native implementations.
- StorageClass defaults: `gp2/gp3` (EKS), `managed-csi` (AKS), `standard-rwo` (GKE). EBS/Disk zonal hote hain -> Pod ko usi AZ me chalna padta hai (`WaitForFirstConsumer`).
- Private clusters: nodes ko public IP nahi, NAT/egress via gateway, private endpoints.

## 7. Security aur operations

- Private API endpoint, authorized IP ranges, network policies, Pod Security Admission.
- Image: private registries (ECR/ACR/Artifact Registry), scanning.
- Secrets: external secret stores (AWS Secrets Manager, Key Vault, GCP Secret Manager) + CSI driver / External Secrets Operator; envelope encryption of etcd with KMS.
- Observability: Container Insights / Azure Monitor / Cloud Operations, ya Prometheus + Grafana.
- Cost: right-size, Spot, autoscaling, namespace/label cost allocation (Kubecost).
- IaC: Terraform / eksctl / Bicep / Pulumi / Crossplane.

## 8. kubeadm vs Managed (kab kya)

- **Managed**: kam ops overhead, faster start, cloud integrations. Limited control-plane customization.
- **Self-managed**: poora control, on-prem/air-gapped, custom flags. Zyada operational kaam.
- CKA exam ke liye kubeadm practice karo; production ke liye aksar managed choose hota hai.

## Common mistakes

- Static cloud credentials Secret me rakhna (workload identity use karo).
- Sab kuch ek node pool me chalana (system + apps + GPU).
- Subnet/IP range chhoti rakhna (Pod IP exhaustion).
- Spot nodes pe stateful/critical workloads bina PDB/replicas ke.
- Control plane upgrade ke baad node pools purane chhod dena (version skew).
- Public API endpoint bina CIDR restriction.

## Practice (free tier / low cost)

1. `eksctl create cluster`, `az aks create`, ya `gcloud container clusters create-auto` se ek chhota cluster banao (cost dhyan se, kaam ke baad delete karo!).
2. Naya node pool (alag VM type + taint) banao aur workload usi pe bhejo.
3. Workload Identity se Pod ko object storage read access do (bina keys ke).
4. Control plane ke baad node pool upgrade karo.
5. Local alternative: `kind`/`minikube` pe IAM ke bina baaki concepts practice karo.
