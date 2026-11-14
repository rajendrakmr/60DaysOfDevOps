# 05 - Scheduling

Topics: Labels, selectors, taints, tolerations, affinity, topology spread, priority

## Scheduler kaise kaam karta hai

`kube-scheduler` unscheduled Pods (jinka `spec.nodeName` empty hai) dekhta hai:
1. **Filtering**: kaun se nodes feasible hain (resources, taints, selectors, affinity)
2. **Scoring**: feasible nodes me best chuno
3. **Binding**: Pod ko node assign (`nodeName` set)

Scheduler down ho to naye Pods `Pending` rahenge.

## 1. Labels aur Selectors

Labels = key/value metadata. Selectors = labels se objects chunna.

```bash
k label pod nginx env=prod tier=fe
k label pod nginx env-                 # label hatana
k get pods --show-labels
k get pods -l env=prod,tier!=be
k get pods -l 'env in (prod,stage)'
k label node node01 disk=ssd
```

Selector types:
- **Equality**: `env=prod`
- **Set-based**: `env in (a,b)`, `notin`, `exists`
- YAML me `matchLabels` aur `matchExpressions`

Annotations: non-identifying metadata (selectable nahi).

## 2. Manual scheduling

- `spec.nodeName: node01` -> scheduler bypass (Pod directly us node pe)
- Scheduler dead ho to ye kaam aata hai.

## 3. nodeSelector

```yaml
spec:
  nodeSelector:
    disk: ssd
```
Simple AND match. Complex logic ke liye affinity.

## 4. Node Affinity

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:     # hard
        nodeSelectorTerms:
        - matchExpressions:
          - key: disk
            operator: In
            values: ["ssd", "nvme"]
      preferredDuringSchedulingIgnoredDuringExecution:    # soft
      - weight: 80
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values: ["a"]
```
Operators: `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`.
"IgnoredDuringExecution" = Pod ke chalne ke baad label badalne se Pod evict nahi hota.

## 5. Pod Affinity / Anti-affinity

Dusre **Pods** ki location ke hisaab se. `topologyKey` batata hai "same" ka matlab kya (node, zone).

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels: {app: web}
      topologyKey: kubernetes.io/hostname     # har replica alag node par
  podAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 50
      podAffinityTerm:
        labelSelector:
          matchLabels: {app: cache}
        topologyKey: topology.kubernetes.io/zone
```

## 6. Taints aur Tolerations

**Taint** node ki taraf se: "mujh pe sirf wahi Pods aayein jo ise tolerate karein". **Toleration** Pod ki taraf se.

```bash
k taint nodes node01 key=value:NoSchedule
k taint nodes node01 key=value:NoSchedule-        # remove (end me '-')
k describe node node01 | grep -i taint
```

Effects:
- `NoSchedule`: naye Pods nahi (existing safe)
- `PreferNoSchedule`: koshish karo ki na aayein
- `NoExecute`: naye nahi **aur existing bhi evict** (jab tak tolerate na karein)

```yaml
tolerations:
- key: "key"
  operator: "Equal"        # ya Exists
  value: "value"
  effect: "NoSchedule"
  # tolerationSeconds: 300   (NoExecute ke saath)
```

Notes:
- Taint sirf "andar aane se rokta" hai, Pod ko us node pe **bhejta nahi**. Dedicated node ke liye taint + nodeSelector/affinity dono.
- Control-plane node taint: `node-role.kubernetes.io/control-plane:NoSchedule`.
- Auto taints: `node.kubernetes.io/not-ready`, `unreachable`, `disk-pressure`, `memory-pressure` etc.

## 7. Topology Spread Constraints

Pods ko zones/nodes me evenly failure domains me failao.

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule       # ya ScheduleAnyway
    labelSelector:
      matchLabels: {app: web}
```
`maxSkew` = kisi do domains ke Pod count me max difference.

## 8. Priority aur Preemption

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority     # ya Never
---
spec:
  priorityClassName: high
```
High-priority Pod `Pending` ho to scheduler low-priority Pods ko **preempt (evict)** kar sakta hai. Built-ins: `system-cluster-critical`, `system-node-critical`.

## 9. Related topics (exam me aate hain)

- **Static Pods**: `/etc/kubernetes/manifests` (kubelet config `staticPodPath`). Mirror Pod dikhta hai, name me node name suffix.
- **Multiple schedulers**: Pod spec `schedulerName`.
- **Resource requests** scheduling ka basis (02-configuration).
- **Cordon/Drain**: `k cordon node01` (naye Pods nahi), `k drain node01 --ignore-daemonsets --delete-emptydir-data`, `k uncordon node01`.

## Debug

```bash
k describe pod <pod>           # Events: "0/3 nodes are available: 1 node(s) had untolerated taint..., 2 Insufficient cpu"
k get events --sort-by=.metadata.creationTimestamp
k get nodes --show-labels
```

## Common mistakes

- `NoSchedule` taint hatane ke liye trailing `-` bhulna.
- Required affinity ka label node par hai hi nahi -> Pending.
- Taint lagayi par toleration/nodeSelector dono nahi diye.
- topologyKey galat spelling.
- DaemonSet control-plane par nahi chala (toleration nahi).

## Practice labs

1. Node ko label do aur Pod ko nodeSelector se wahi bhejo.
2. Node par `NoSchedule` taint lagao, Pod Pending dekho, toleration dalke schedule karo.
3. 3 replica Deployment ko anti-affinity se alag-alag nodes pe rakho.
4. `NoExecute` taint lagao aur existing Pods ka evict hona dekho.
5. PriorityClass bana ke low-priority Pods ko preempt hote dekho.
6. Scheduler Pod (static) ka manifest hatao, naya Pod `Pending` dekho, phir wapas lagao.
