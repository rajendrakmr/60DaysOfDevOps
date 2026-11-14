# 08 - Scaling and Availability

Topics: HPA, VPA concepts, Cluster Autoscaler, PDB, rollout strategies

## 1. Manual scaling

```bash
k scale deployment web --replicas=5
k scale statefulset db --replicas=3
k scale --replicas=0 deployment web            # pause jaisa
```

## 2. Horizontal Pod Autoscaler (HPA)

Pods ki **sankhya** metrics ke hisaab se badhata/ghatata hai (Deployment/StatefulSet/ReplicaSet).

**Prerequisites**
- **metrics-server** install ho (`k top pods` kaam kare).
- Target Pods me **resource requests** set hone chahiye (CPU % requests ke relative hota hai).

```bash
k autoscale deployment web --cpu-percent=60 --min=2 --max=10
k get hpa
k describe hpa web
```

Formula: `desiredReplicas = ceil(currentReplicas * currentMetric / targetMetric)`

`autoscaling/v2` YAML:
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 400Mi
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300      # flapping rokne ke liye
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
```
Metric types: `Resource`, `Pods` (custom per-pod), `Object`, `External` (custom/external metrics adapter, e.g. Prometheus adapter, KEDA).

Gotchas:
- HPA aur manual `replicas` ya Deployment YAML me `replicas` conflict kar sakte hain.
- `TARGETS: <unknown>/60%` = metrics-server missing ya requests set nahi.
- Scale-down me default stabilization 5 min.

## 3. Vertical Pod Autoscaler (VPA) - concept

Pods ke **requests/limits** ko recommend ya auto-adjust karta hai (size badalta hai, count nahi).
- Core Kubernetes me built-in nahi; alag add-on (CRD `VerticalPodAutoscaler`) hai.
- Modes: `Off` (sirf recommendation), `Initial`, `Recreate`/`Auto` (Pod restart karke apply).
- Components: Recommender, Updater, Admission Controller.
- Same metric (CPU/memory) pe HPA aur VPA saath me mat chalao (conflict). In-place resize (Pod resize feature) newer versions me develop ho raha hai, version ke hisaab se check karo.

## 4. Cluster Autoscaler (CA)

**Nodes** ki sankhya badhata/ghatata hai.
- **Scale up**: Pods `Pending` hain kyunki resources nahi (unschedulable) -> naye node add.
- **Scale down**: node kaafi der under-utilized ho aur Pods kahin aur ja sakein -> node remove.
- Node groups (ASG/MIG/VMSS) ke saath cloud provider integration. Alternative: **Karpenter** (AWS me popular, just-in-time node provisioning).
- Scale-down rokne wale: PDB violation, local storage, `cluster-autoscaler.kubernetes.io/safe-to-evict: "false"` annotation, kube-system Pods bina PDB.
- Order: HPA Pods badhata hai -> Pending -> CA nodes badhata hai.

## 5. PodDisruptionBudget (PDB)

**Voluntary disruptions** (drain, upgrade, CA scale-down) ke time minimum availability guarantee.

```bash
k create pdb web-pdb --selector=app=web --min-available=2
k create pdb web-pdb --selector=app=web --max-unavailable=1
```
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2          # ya maxUnavailable (dono ek saath nahi)
  selector:
    matchLabels: {app: web}
```
- `k drain` PDB respect karta hai; allowed disruptions 0 ho to drain atak jata hai.
- Involuntary disruptions (node crash, OOM) pe PDB lagu nahi hoti.
- Single replica + `minAvailable: 1` = drain kabhi nahi hoga.

## 6. Rollout strategies

### Built-in (Deployment)
**RollingUpdate** (default): dhire-dhire naye Pods lao, purane hatao.
- `maxSurge`: desired se kitne extra Pods (number ya %)
- `maxUnavailable`: kitne Pods unavailable ho sakte hain
- Zero-downtime ke liye: readiness probe sahi + `maxUnavailable: 0`.

**Recreate**: pehle sab purane band, phir naye. Downtime hota hai; jab do versions saath na chal sakein (e.g. schema/RWO volume).

Aur Deployment fields: `minReadySeconds`, `progressDeadlineSeconds`, `revisionHistoryLimit`.

```bash
k rollout status deploy/web
k rollout history deploy/web --revision=2
k rollout undo deploy/web --to-revision=1
k annotate deploy/web kubernetes.io/change-cause="upgrade to 1.27"
```

### Advanced (extra tooling)
- **Blue/Green**: do Deployments (blue, green) + Service selector switch.
  ```bash
  k patch svc web -p '{"spec":{"selector":{"app":"web","version":"green"}}}'
  ```
- **Canary**: chhota Deployment (10% replicas) same Service label ke andar, ya Ingress/Gateway weights, ya Argo Rollouts/Flagger.
- StatefulSet: `updateStrategy.rollingUpdate.partition`.

## 7. Availability best practices

- Multiple replicas + **topology spread** / **anti-affinity** (05-scheduling).
- **Probes** (readiness) sahi rakho.
- **PDB** lagao critical apps pe.
- Requests/limits sahi rakho; PriorityClass critical workloads ke liye.
- Multi-AZ node pools; HA control plane (3 etcd/API server).
- `terminationGracePeriodSeconds` aur `preStop` hook graceful shutdown ke liye.

## Common mistakes

- metrics-server nahi, ya requests set nahi -> HPA `<unknown>`.
- HPA me `minReplicas` kam rakhna aur sudden load.
- PDB `minAvailable` replicas ke barabar -> drain/upgrade block.
- Recreate strategy production me bina soche.
- HPA + VPA same CPU metric.

## Practice labs

1. metrics-server install karo, HPA banao, `busybox` se load generate karo (`while true; do wget -q -O- http://web; done`), scale up/down dekho.
2. PDB lagao (`minAvailable: 2`) aur `k drain` try karo, behaviour dekho.
3. `maxSurge=1, maxUnavailable=0` ke saath image update karo aur `k get pods -w` se observe karo.
4. Blue/Green: do versions deploy karke Service selector switch karo.
5. Galat image se update karo aur `rollout undo` se wapas aao.
