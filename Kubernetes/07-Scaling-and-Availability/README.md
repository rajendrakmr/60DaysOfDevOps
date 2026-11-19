Kubernetes – HPA, VPA, Cluster Autoscaler, PDB & Rollout Strategies

HPA (Horizontal Pod Autoscaler)

What is HPA?

HPA automatically increases or decreases the number of Pod replicas based on resource utilization or custom metrics.

Why use HPA?
- Handle increased application traffic.
- Maintain application performance.
- Reduce unnecessary Pod replicas when traffic decreases.

**Example:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

**Meaning:**
- Minimum 2 Pods.
- Maximum 10 Pods.
- If average CPU utilization exceeds 70%, HPA may increase replicas.
- When CPU usage decreases, HPA may scale down.

**Commands:**

```bash
kubectl get hpa
kubectl describe hpa nginx-hpa
kubectl autoscale deployment nginx --cpu-percent=70 --min=2 --max=10
```

**Important:** HPA requires Metrics Server for resource metrics such as CPU and memory.

---

## 2. VPA (Vertical Pod Autoscaler)

**What is VPA?**

VPA automatically recommends or adjusts CPU and memory requests for containers based on their actual resource usage.

**Why use VPA?**
- Avoid CPU and memory under-allocation.
- Reduce unnecessary resource requests.
- Improve resource utilization.
- Help prevent resource-related performance issues.

**Example:**

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: nginx-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  updatePolicy:
    updateMode: Auto
```

**Update modes:**

| Mode | Meaning |
|---|---|
| Off | Only provides recommendations |
| Initial | Applies recommendations when Pods are created |
| Recreate | Updates resources by recreating Pods |
| Auto | Automatically applies recommendations; behavior depends on VPA implementation/version |

**Important:** VPA is an add-on; it is not automatically installed with Kubernetes.

---

## 3. Cluster Autoscaler (CA)

**What is Cluster Autoscaler?**

Cluster Autoscaler automatically increases or decreases the number of worker nodes based on Pod scheduling requirements and node utilization.

**Why use it?**
- Add nodes when Pods cannot be scheduled due to insufficient resources.
- Remove underutilized nodes when workloads can safely move elsewhere.
- Optimize infrastructure costs.

**Example:**

Suppose an EKS node group has:

```text
Minimum Nodes: 2
Desired Nodes: 2
Maximum Nodes: 10
```

If existing nodes cannot accommodate new Pods, Cluster Autoscaler can increase the node count, up to the configured maximum.

**Important points:**
- HPA scales Pods.
- VPA adjusts Pod resource requests.
- Cluster Autoscaler scales worker nodes.
- Cluster Autoscaler needs a supported infrastructure provider/node group.
- In EKS, it commonly works with Auto Scaling Groups or managed node groups.

---

## 4. PDB (Pod Disruption Budget)

**What is PDB?**

Pod Disruption Budget defines how many Pods must remain available during voluntary disruptions.

**Why use PDB?**
- Maintain application availability.
- Protect workloads during node maintenance.
- Control voluntary Pod evictions, such as during `kubectl drain`.

**Example:**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nginx-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: nginx
```

Suppose 3 nginx Pods are running.

```text
Total Pods: 3
minAvailable: 2
Allowed voluntary disruptions: 1
```

Only one Pod can be voluntarily disrupted at a time while maintaining the budget.

**Two important fields:**

| Field | Meaning |
|---|---|
| minAvailable | Minimum number or percentage of Pods that must remain available |
| maxUnavailable | Maximum number or percentage of Pods that can be unavailable |

**Commands:**

```bash
kubectl get pdb
kubectl describe pdb nginx-pdb
kubectl get pdb -A
```

**Important:** PDB primarily protects against voluntary disruptions. It does not prevent involuntary failures such as node crashes.

---

## 5. Kubernetes Rollout Strategies

A rollout strategy defines how Kubernetes updates an application from one version to another.

### A. Rolling Update

Updates Pods gradually instead of replacing all Pods at once.

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

**Meaning:**
- `maxSurge: 1` – Kubernetes can create one extra Pod during the update.
- `maxUnavailable: 0` – No desired Pod should be unavailable due to the rollout.

**Use case:** Normal production deployments where minimal downtime is required.

### B. Recreate Strategy

Terminates old Pods before creating new Pods.

```yaml
strategy:
  type: Recreate
```

**Flow:**

```text
Stop old Pods
      ↓
Create new Pods
      ↓
Application becomes available
```

**Use case:** Applications that cannot run multiple versions simultaneously.

### C. Blue-Green Deployment

Maintain two environments:

- Blue = Current production version.
- Green = New application version.

**Flow:**

```text
Blue (v1) → Live Traffic
Green (v2) → Testing
              ↓
        Switch Traffic
              ↓
Green (v2) → Live Traffic
```

**Benefit:** Quick traffic switching and easier rollback.

**Note:** Blue-Green deployment generally requires additional configuration, such as separate Deployments and Service selector switching.

### D. Canary Deployment

Release a new version to a small percentage of users first.

**Example:**

```text
Version 1 → 90% Traffic
Version 2 → 10% Traffic
```

Monitor metrics and errors. If everything looks good, gradually increase traffic to v2.

**Benefit:** Reduces risk by testing a new version with limited traffic.

**Note:** Canary traffic splitting generally requires a service mesh, Gateway API implementation, ingress controller feature, or another traffic-management solution.

---

## 6. Quick Comparison – Interview Revision

| Component | What does it scale/manage? | Main purpose |
|---|---|---|
| HPA | Pod replicas | Horizontal scaling |
| VPA | CPU/Memory requests | Vertical scaling |
| Cluster Autoscaler | Worker nodes | Infrastructure scaling |
| PDB | Pod availability during voluntary disruptions | Availability protection |
| Rolling Update | Application versions | Gradual deployment |
| Recreate | Application versions | Replace old Pods |
| Blue-Green | Application traffic | Switch between environments |
| Canary | Application traffic | Gradual release |

## 7. Real-World Production Scenario

Suppose an e-commerce application experiences high traffic during a sale.

1. **HPA:** Increases frontend and backend Pod replicas.
2. **Cluster Autoscaler:** Adds worker nodes if existing nodes cannot schedule new Pods.
3. **VPA:** Helps right-size CPU and memory requests.
4. **PDB:** Protects minimum application availability during node maintenance.
5. **Rolling Update / Canary:** Deploys new application versions with controlled risk.

## 8. Important Interview Questions

1. What is the difference between HPA and VPA?
2. How does Cluster Autoscaler work?
3. What happens when HPA creates Pods but there is no available node capacity?
4. What is the difference between `minAvailable` and `maxUnavailable` in PDB?
5. Does PDB protect against node failure?
6. What is the difference between Rolling Update and Recreate?
7. What is Blue-Green deployment?
8. How does Canary deployment reduce deployment risk?
9. Can HPA and VPA work together?
10. What happens if a PDB blocks a node drain?

**Remember:**

- HPA = More or fewer Pods.
- VPA = More or fewer resources per Pod.
- CA = More or fewer worker nodes.
- PDB = Protect available Pods during voluntary disruptions.
- Rollout = Safely deploy application updates.