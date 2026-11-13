# 04 - Storage

Topics: PV, PVC, StorageClass, CSI, access modes, reclaim policies

## Big picture

Pod ka container filesystem temporary hota hai. Persistent data ke liye:

```
Pod -> PVC (request) -> PV (actual storage) -> backend (EBS, NFS, Ceph, local disk...)
                         ^
                 StorageClass (dynamic provisioning)
```

## 1. PersistentVolume (PV)

Cluster-level storage resource (cluster-scoped, namespace nahi). Admin banata hai (static) ya StorageClass se automatically banta hai (dynamic).

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-demo
spec:
  capacity:
    storage: 5Gi
  accessModes: ["ReadWriteOnce"]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data          # sirf testing/exam
```

## 2. PersistentVolumeClaim (PVC)

User ki storage request (**namespaced**). Match hone par PV se bind hoti hai (1:1).

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-demo
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 3Gi
  storageClassName: manual
```

Pod me use:
```yaml
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: pvc-demo
```

### Binding rules
PVC bind tabhi hoti hai jab PV me:
- `storageClassName` match kare (`""` ka matlab "koi class nahi", omit karne ka matlab default class)
- `accessModes` PVC ke modes ko support kare
- `capacity` >= requested size
- (optional) selector/labels match ho

Status: PVC `Pending` (koi PV match nahi ya WaitForFirstConsumer), `Bound`. PV: `Available`, `Bound`, `Released`, `Failed`.

## 3. Access modes

| Mode | Short | Matlab |
|------|-------|--------|
| ReadWriteOnce | RWO | Ek **node** se read-write |
| ReadOnlyMany | ROX | Kai nodes se read-only |
| ReadWriteMany | RWX | Kai nodes se read-write (NFS, CephFS) |
| ReadWriteOncePod | RWOP | Sirf ek **Pod** se read-write |

Access mode backend ki capability hai, har backend har mode support nahi karta (EBS = RWO only).

## 4. Reclaim policies

PVC delete hone ke baad PV ka kya hoga:

| Policy | Behaviour |
|--------|-----------|
| **Retain** | PV `Released` ho jata hai, data bachta hai, manual cleanup/reuse (claimRef hatana padta hai) |
| **Delete** | PV aur backend storage dono delete (dynamic provisioning ka default) |
| **Recycle** | Deprecated |

Badalne ke liye:
```bash
k patch pv pv-demo -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

## 5. StorageClass

Dynamic provisioning ka template: provisioner, parameters, reclaimPolicy, volumeBindingMode, allowVolumeExpansion.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

- `volumeBindingMode: Immediate` (default) vs `WaitForFirstConsumer` (Pod schedule hone ke baad volume banata hai, zone-aware, local volumes ke liye zaroori).
- `kubernetes.io/no-provisioner` = dynamic provisioning nahi (local PV).
- Default StorageClass annotation se set hoti hai.

```bash
k get sc
k patch sc fast -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

## 6. CSI (Container Storage Interface)

Storage vendors ke liye standard plugin interface. Purane in-tree drivers ki jagah. Components:
- **Controller plugin** (provision/attach/snapshot)
- **Node plugin** (mount on node)
- Sidecars: external-provisioner, external-attacher, external-resizer, external-snapshotter
- Objects: `CSIDriver`, `CSINode`, `VolumeSnapshot`, `VolumeSnapshotClass`

## 7. Volume expansion

StorageClass me `allowVolumeExpansion: true` ho to PVC ka `spec.resources.requests.storage` **badha** sakte ho (shrink nahi).
```bash
k patch pvc pvc-demo -p '{"spec":{"resources":{"requests":{"storage":"10Gi"}}}}'
```

## 8. Other volume types (recap)

`emptyDir`, `hostPath`, `local` (PV with nodeAffinity), `nfs`, `projected`, `configMap`, `secret`. Raw block ke liye `volumeMode: Block`.

## Exam style flow

```bash
# 1. PV banao (YAML), 2. PVC banao, 3. Pod/Deployment me mount karo
k get pv,pvc
k describe pvc pvc-demo        # Events me bind na hone ki wajah
k exec pod -- sh -c 'echo hi > /data/x; cat /data/x'
# Pod delete karke recreate -> data bacha?
```

## Common mistakes

- PVC `Pending`: storageClassName mismatch, size zyada, access mode unsupported, ya WaitForFirstConsumer me Pod abhi bana hi nahi.
- PV/PVC namespace confuse (PV cluster-wide, PVC namespaced).
- `Retain` PV `Released` me hai to naya PVC bind nahi hota jab tak `claimRef` na hatao.
- hostPath/local PV ke saath Pod dusre node pe schedule hona.
- StatefulSet delete karne se PVC delete nahi hoti (alag se karni padti hai).

## Practice labs

1. hostPath PV (1Gi) + PVC + Pod; file likho, Pod delete karo, naye Pod me data check karo.
2. Reclaim policy `Retain` pe PVC delete karo, PV ka status dekho, phir PV ko reuse karo.
3. Default StorageClass ke saath bina `storageClassName` ke PVC banao, dynamic PV dekho.
4. PVC ko expand karo (agar backend support kare).
5. `WaitForFirstConsumer` class pe PVC banao aur dekho Pod aane tak `Pending` rehti hai.
