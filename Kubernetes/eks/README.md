# Amazon EKS Cluster Setup & Management

**AWS | Amazon EKS | eksctl | Kubernetes | Managed Node Groups | Envoy Gateway | cert-manager**

## 📌 Project Overview

This project demonstrates how to create and manage Amazon EKS clusters using `eksctl`.

It covers two approaches to EKS cluster provisioning:

1. **EKS Cluster with Managed Node Group** – Create an EKS cluster along with worker nodes.
2. **EKS Cluster without Node Group** – Create only the EKS control plane and provision worker nodes separately.

Additionally, this project covers:

- EKS cluster configuration and management.
- Managed Node Group creation and scaling.
- IAM OIDC provider association.
- Kubernetes context configuration.
- Envoy Gateway installation using Helm.
- SSL certificate management using cert-manager.
- AWS CloudFormation stack cleanup.

---

## 🛠️ Prerequisites

Install and configure the following tools:

| Tool | Purpose |
|---|---|
| AWS CLI | AWS resource management |
| eksctl | EKS cluster provisioning |
| kubectl | Kubernetes cluster management |
| Helm | Kubernetes package management |

### Verify Installation

```bash
aws --version
eksctl version
kubectl version --client
helm version
```

### Configure AWS Credentials

```bash
aws configure
```

Verify AWS identity:

```bash
aws sts get-caller-identity
```

---

# 1. Create EKS Cluster with Managed Node Group

In this approach, the EKS control plane and managed worker nodes are created together.

### Cluster Configuration

| Parameter | Value |
|---|---|
| Cluster Name | clusterwithnode |
| Region | eu-west-1 |
| Kubernetes Version | 1.37 |
| Availability Zones | eu-west-1a, eu-west-1b |
| Node Group | worker-nodes |
| Instance Type | t3.small |
| Desired Nodes | 2 |
| Minimum Nodes | 1 |
| Maximum Nodes | 3 |
| Volume Size | 30 GB |
| Volume Type | gp3 |

### Create Cluster

```bash
eksctl create cluster \
  --name clusterwithnode \
  --region eu-west-1 \
  --version 1.37 \
  --zones eu-west-1a,eu-west-1b \
  --nodegroup-name worker-nodes \
  --node-type t3.small \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --node-volume-size 30 \
  --node-volume-type gp3 \
  --managed \
  --asg-access \
  --tags environment=dev,team=devops
```

This command creates:

- EKS control plane.
- Managed Node Group.
- Two worker nodes.
- Auto Scaling Group.
- EC2 instances distributed across the selected Availability Zones.
- EBS gp3 volumes.

### Associate IAM OIDC Provider

OIDC association enables IAM Roles for Service Accounts (IRSA), allowing Kubernetes service accounts to assume IAM roles.

```bash
eksctl utils associate-iam-oidc-provider \
  --region eu-west-1 \
  --cluster clusterwithnode \
  --approve
```

### Configure kubectl

```bash
aws eks update-kubeconfig \
  --region eu-west-1 \
  --name clusterwithnode \
  --alias clusterwithnode
```

### Verify Cluster

```bash
kubectl get nodes -o wide

kubectl get pods -A

kubectl cluster-info

eksctl get cluster --region eu-west-1
```

---

# 2. Managed Node Group Scaling

EKS Managed Node Groups use EC2 Auto Scaling Groups to manage worker node capacity.

You can configure minimum, maximum and desired node counts.

### Update Minimum Nodes

```bash
eksctl scale nodegroup \
  --cluster clusterwithnode \
  --region eu-west-1 \
  --name worker-nodes \
  --nodes-min 2
```

### Update Maximum Nodes

```bash
eksctl scale nodegroup \
  --cluster clusterwithnode \
  --region eu-west-1 \
  --name worker-nodes \
  --nodes-max 5
```

### Increase Desired Node Count

```bash
eksctl scale nodegroup \
  --cluster clusterwithnode \
  --region eu-west-1 \
  --name worker-nodes \
  --nodes 4
```

**Important:**

- `nodes-min`: Minimum number of worker nodes.
- `nodes-max`: Maximum number of worker nodes.
- `nodes`: Desired number of worker nodes.

The desired node count must be between the configured minimum and maximum values.

### Get Node Group Details

```bash
eksctl get nodegroup \
  --cluster clusterwithnode \
  --region eu-west-1
```

### Verify Worker Nodes

```bash
kubectl get nodes
```

---

# 3. Create EKS Cluster Without Node Group

In this approach, we first create the EKS control plane without worker nodes.

After the cluster becomes available, we create a separate managed node group.

This approach provides more flexibility when managing cluster infrastructure independently from worker nodes.

### Step 1: Create EKS Control Plane

```bash
eksctl create cluster \
  --name clusterwithoutnode \
  --version 1.37 \
  --region eu-west-1 \
  --without-nodegroup
```

### Step 2: Associate IAM OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider \
  --region eu-west-1 \
  --cluster clusterwithoutnode \
  --approve
```

### Step 3: Create Managed Node Group

```bash
eksctl create nodegroup \
  --cluster clusterwithoutnode \
  --region eu-west-1 \
  --name without-worker-nodes \
  --node-type t3.small \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --node-volume-size 30 \
  --node-volume-type gp3 \
  --managed
```

### Step 4: Configure kubectl

```bash
aws eks update-kubeconfig \
  --region eu-west-1 \
  --name clusterwithoutnode \
  --alias clusterwithoutnode
```

### Step 5: Verify Cluster

```bash
kubectl get nodes -o wide

kubectl get pods -A

eksctl get nodegroup \
  --cluster clusterwithoutnode \
  --region eu-west-1
```

---

# 4. Install Envoy Gateway Using Helm

Envoy Gateway provides Kubernetes Gateway API-based traffic management.

It supports HTTP, HTTPS, gRPC and other supported traffic protocols.

### Step 1: Install Envoy Gateway

```bash
helm install eg \
  oci://docker.io/envoyproxy/gateway-helm \
  --version v1.10.0 \
  -n envoy-gateway-system \
  --create-namespace
```

### Step 2: Wait for Deployment

```bash
kubectl wait \
  --timeout=5m \
  -n envoy-gateway-system \
  deployment/envoy-gateway \
  --for=condition=Available
```

### Step 3: Verify Installation

```bash
kubectl get pods -n envoy-gateway-system

kubectl get svc -n envoy-gateway-system

kubectl get deployment -n envoy-gateway-system
```

### Step 4: Verify Gateway API Resources

```bash
kubectl get gatewayclass

kubectl get gateways -A

kubectl get httproutes -A
```

**Note:** Ensure the required Gateway API CRDs are installed before creating Gateway API resources.

---

# 5. Install cert-manager for SSL Certificate Management

cert-manager automates TLS certificate issuance and renewal inside Kubernetes.

It can integrate with Let's Encrypt to obtain free SSL/TLS certificates.

### Step 1: Add Jetstack Helm Repository

```bash
helm repo add jetstack https://charts.jetstack.io

helm repo update
```

### Step 2: Install cert-manager

```bash
helm install cert-manager jetstack/cert-manager \
  -n cert-manager \
  --create-namespace \
  --set crds.enabled=true \
  --set config.apiVersion="controller.config.cert-manager.io/v1alpha1" \
  --set config.kind="ControllerConfiguration" \
  --set config.enableGatewayAPI=true
```

### Step 3: Verify cert-manager

```bash
kubectl get pods -n cert-manager

kubectl get deployment -n cert-manager

kubectl get crds | grep cert-manager
```

**Important:** Gateway API CRDs must exist before cert-manager starts when Gateway API integration is enabled. Install the required Gateway API CRDs first if they are not already present.

### SSL Certificate Workflow

The general certificate management workflow:

1. Install cert-manager.
2. Configure an ACME ClusterIssuer.
3. Configure Let's Encrypt email and ACME server.
4. Configure HTTP-01 or DNS-01 challenge.
5. Create a Certificate or configure Gateway TLS integration.
6. Verify certificate issuance.
7. cert-manager automatically renews certificates before expiration.

---

# 6. AWS CloudFormation Stack Management

eksctl provisions AWS infrastructure using CloudFormation stacks.

You can inspect and manage these stacks from AWS CLI.

### List CloudFormation Stacks

```bash
aws cloudformation list-stacks \
  --region eu-west-1
```

### Disable Termination Protection

If termination protection is enabled on a stack, disable it before deletion.

```bash
aws cloudformation update-termination-protection \
  --region eu-west-1 \
  --stack-name eksctl-clusterwithnode-cluster \
  --no-enable-termination-protection
```

### Delete EKS Cluster (Recommended)

Use eksctl to delete the cluster and its associated managed resources.

```bash
eksctl delete cluster \
  --name clusterwithnode \
  --region eu-west-1
```

For the second cluster:

```bash
eksctl delete cluster \
  --name clusterwithoutnode \
  --region eu-west-1
```

### Verify Stack Deletion

If deleting stacks manually:

```bash
aws cloudformation wait stack-delete-complete \
  --region eu-west-1 \
  --stack-name eksctl-clusterwithnode-cluster
```

**Important:** Manual CloudFormation deletion is not the preferred way to remove an entire eksctl-managed cluster. Use `eksctl delete cluster` to clean up the associated resources properly.

---

# 7. Useful EKS Commands

### Cluster Management

```bash
eksctl get cluster --region eu-west-1

eksctl get nodegroup --cluster clusterwithnode --region eu-west-1

kubectl config get-contexts

kubectl config current-context
```

### Kubernetes Node Management

```bash
kubectl get nodes -o wide

kubectl describe nodes

kubectl top nodes
```

### Namespace and Pod Management

```bash
kubectl get namespaces

kubectl get pods -A

kubectl get services -A

kubectl get deployments -A
```

### Helm Management

```bash
helm list -A

helm status eg -n envoy-gateway-system

helm status cert-manager -n cert-manager
```

---

# 8. Important Concepts Learned

| Concept | Description |
|---|---|
| Amazon EKS | Managed Kubernetes control plane provided by AWS |
| eksctl | CLI tool for creating and managing EKS clusters |
| Managed Node Group | AWS-managed worker node infrastructure |
| Availability Zones | Distribute infrastructure across multiple AZs |
| Auto Scaling Group | Manages EC2 instance capacity |
| OIDC Provider | Enables IAM identity federation for Kubernetes workloads |
| kubeconfig | Connects kubectl to an EKS cluster |
| Helm | Kubernetes package manager |
| Envoy Gateway | Gateway API implementation |
| cert-manager | Automates TLS certificate issuance and renewal |
| CloudFormation | Infrastructure provisioning and management |

---

# 9. Project Architecture

```text
                       AWS CLOUD
                           |
                    Amazon EKS Cluster
                           |
              +------------+------------+
              |                         |
        EKS Control Plane          Managed Node Group
              |                         |
              |                 +-------+-------+
              |                 |               |
              |              Worker 1        Worker 2
              |                 |               |
              |                 +-------+-------+
              |                         |
              |                  Kubernetes Pods
              |                         |
              |                   Envoy Gateway
              |                         |
              |                    HTTP / HTTPS
              |                         |
              |                    cert-manager
              |                         |
              |                   Let's Encrypt
              |                         |
              +-------------------------+
                         |
                   SSL Certificate
```

---

# 10. Learning Outcomes

After completing this project, you will understand:

- How to provision Amazon EKS clusters using eksctl.
- Difference between creating clusters with and without node groups.
- How to create and manage EKS Managed Node Groups.
- How to configure worker node scaling.
- How to configure IAM OIDC integration.
- How to connect kubectl with EKS.
- How to install Kubernetes components using Helm.
- How to deploy Envoy Gateway.
- How to configure cert-manager for automated SSL certificate management.
- How to manage and clean up AWS infrastructure.

---

## ⚠️ Cost Considerations

AWS resources may incur charges even when no application is deployed.

Consider the following:

- EKS cluster control plane charges.
- EC2 worker node charges.
- EBS gp3 storage charges.
- Load Balancer charges.
- Public IPv4 address charges.
- Data transfer charges.

Delete unused clusters and associated resources after completing your practice.

---

**Project Status:** Hands-on Learning Project

**Cloud Provider:** AWS

**Region:** eu-west-1 (Ireland)

**Tools:** AWS CLI, eksctl, kubectl, Helm

**Kubernetes:** Amazon EKS