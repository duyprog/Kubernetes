# Storage — PV, PVC & StorageClass Lifecycle

---

## Storage Architecture

```
Pod
└── volumeMount → PersistentVolumeClaim (PVC)
                       └── binds to PersistentVolume (PV)
                                  └── backed by physical storage
                                      (EBS, EFS, NFS, local disk...)

StorageClass → defines how PVs are dynamically provisioned
```

---

## The Three Objects

### PersistentVolume (PV)

A piece of storage **provisioned in the cluster** — either manually (static) or automatically (dynamic via StorageClass). Cluster-scoped (not namespace-specific).

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 50Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: gp3
  csi:
    driver: ebs.csi.aws.com
    volumeHandle: vol-0abc123def456      # AWS EBS volume ID
    fsType: ext4
```

### PersistentVolumeClaim (PVC)

A **request for storage** by a Pod. Namespace-scoped. Kubernetes binds the PVC to a matching PV.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: gp3   # must match PV or StorageClass
```

### StorageClass

Defines a **type of storage** and how to provision it dynamically.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer   # provision in same AZ as Pod
reclaimPolicy: Delete
allowVolumeExpansion: true
```

---

## Lifecycle

### Static Provisioning

```
Admin creates PV manually → PVC is created → Kubernetes binds PVC to PV → Pod uses PVC
```

### Dynamic Provisioning (recommended)

```
PVC is created with storageClassName → StorageClass triggers provisioner
→ PV is created automatically → PVC binds → Pod uses PVC
```

---

## PVC Binding Rules

Kubernetes finds the best PV that:
1. Has matching `storageClassName`
2. Has matching or larger `capacity`
3. Has compatible `accessModes`
4. Is in `Available` phase

If no PV matches → PVC stays `Pending` until one becomes available (or StorageClass dynamically creates one).

---

## Access Modes

| Mode | Abbreviation | Description | Typical Storage |
|---|---|---|---|
| `ReadWriteOnce` | RWO | One node, read/write | EBS, local disk |
| `ReadOnlyMany` | ROX | Many nodes, read-only | NFS, EFS |
| `ReadWriteMany` | RWX | Many nodes, read/write | EFS, NFS, CephFS |
| `ReadWriteOncePod` | RWOP | One Pod only, read/write | EBS (k8s 1.22+) |

---

## Reclaim Policies

What happens to the PV when the PVC is deleted.

| Policy | Behavior |
|---|---|
| `Retain` | PV stays, data preserved — admin must manually clean up |
| `Delete` | PV and underlying storage (EBS volume) are deleted |
| `Recycle` | Deprecated — basic scrub then makes PV available again |

> Use `Retain` for databases and critical data. Use `Delete` for ephemeral, reproducible storage.

---

## PV / PVC Phases

### PV Phases

| Phase | Description |
|---|---|
| `Available` | Ready to be bound, not yet claimed |
| `Bound` | Bound to a PVC |
| `Released` | PVC was deleted; PV is not yet reclaimed |
| `Failed` | Automatic reclaim failed |

### PVC Phases

| Phase | Description |
|---|---|
| `Pending` | Waiting for a matching PV or dynamic provisioning |
| `Bound` | Successfully bound to a PV |
| `Lost` | Bound PV has been deleted |

---

## Volume Binding Mode

| Mode | Behavior |
|---|---|
| `Immediate` | PV provisioned as soon as PVC is created — may be in wrong AZ |
| `WaitForFirstConsumer` | PV provisioned when a Pod using the PVC is scheduled — uses Pod's AZ |

> Always use `WaitForFirstConsumer` for block storage (EBS) in multi-AZ clusters. Otherwise a Pod on node in `us-east-1a` might try to attach an EBS volume in `us-east-1b`.

---

## Commands

```bash
# List PVs and their binding status
kubectl get pv

# List PVCs in a namespace
kubectl get pvc -n production

# Describe a stuck PVC
kubectl describe pvc my-pvc -n production

# Check which PV a PVC is bound to
kubectl get pvc my-pvc -n production -o jsonpath='{.spec.volumeName}'

# List all StorageClasses
kubectl get storageclass

# Check default StorageClass
kubectl get storageclass | grep "(default)"
```

---

## Further Reading

- [persistent-volumes.md](persistent-volumes.md) — PV types, reclaim policies, static provisioning
- [storage-classes.md](storage-classes.md) — Dynamic provisioning on AWS EKS (EBS, EFS)
