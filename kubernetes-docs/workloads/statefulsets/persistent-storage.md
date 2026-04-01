# StatefulSet Persistent Storage — PVC per Pod

---

## How `volumeClaimTemplates` Works

Each StatefulSet Pod gets its **own dedicated PersistentVolumeClaim**. The PVC name follows the pattern:

```
<volumeClaimTemplate-name>-<statefulset-name>-<pod-index>
```

For a StatefulSet named `postgres` with a PVC template named `data`:

```
data-postgres-0   → bound to postgres-0
data-postgres-1   → bound to postgres-1
data-postgres-2   → bound to postgres-2
```

---

## volumeClaimTemplates Spec

```yaml
volumeClaimTemplates:
  - metadata:
      name: data
      labels:
        app: postgres
    spec:
      accessModes:
        - ReadWriteOnce       # one node at a time (standard for block storage)
      storageClassName: gp3   # AWS EBS gp3 StorageClass
      resources:
        requests:
          storage: 50Gi
```

### Access Modes

| Mode | Abbreviation | Description |
|---|---|---|
| `ReadWriteOnce` | RWO | One node read/write at a time |
| `ReadOnlyMany` | ROX | Many nodes read-only |
| `ReadWriteMany` | RWX | Many nodes read/write (requires NFS/EFS) |
| `ReadWriteOncePod` | RWOP | Only one Pod can mount (k8s 1.22+) |

---

## PVC Retention on Scale-Down or Delete

By default, PVCs created by `volumeClaimTemplates` are **not deleted** when:
- The StatefulSet is scaled down
- The StatefulSet is deleted

This is intentional — data is retained for safety.

### `persistentVolumeClaimRetentionPolicy` (k8s 1.23+)

```yaml
spec:
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain     # or Delete
    whenScaled: Retain      # or Delete
```

| Value | Behavior |
|---|---|
| `Retain` | PVC is kept (default) — manual cleanup required |
| `Delete` | PVC is deleted when Pod is removed |

> In production, use `Retain` to avoid accidental data loss. Clean up manually after confirming data is no longer needed.

---

## Manually Deleting PVCs After Scale-Down

```bash
# After scaling down from 3 → 1, clean up orphaned PVCs
kubectl delete pvc data-postgres-1 data-postgres-2 -n production
```

---

## Expanding PVC Storage (EBS/EFS)

Storage can be expanded without downtime if the StorageClass has `allowVolumeExpansion: true`.

```bash
# Edit PVC to increase size
kubectl edit pvc data-postgres-0 -n production
# Change: storage: 50Gi → storage: 100Gi
```

The PVC will show `FileSystemResizePending` until the Pod is restarted, then the filesystem expands automatically.

---

## Backup Strategies

### Snapshot via VolumeSnapshot (AWS EBS)

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-0-snapshot
  namespace: production
spec:
  volumeSnapshotClassName: csi-aws-vsc
  source:
    persistentVolumeClaimName: data-postgres-0
```

### Restore from Snapshot

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-postgres-0-restored
  namespace: production
spec:
  dataSource:
    name: postgres-0-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3
  resources:
    requests:
      storage: 50Gi
```

---

## Troubleshooting PVCs in StatefulSets

| Symptom | Cause | Fix |
|---|---|---|
| Pod stuck in `Pending` | PVC not bound — no available PV or StorageClass missing | Check `kubectl describe pvc <name>` |
| PVC stuck in `Pending` | StorageClass does not exist or no provisioner | Verify StorageClass: `kubectl get storageclass` |
| Pod stuck in `Pending` after scale-up | PVC exists from old Pod with wrong node binding | Check if EBS AZ matches node AZ |
| Data lost after Pod restart | No PVC — using `emptyDir` or no volume | Add `volumeClaimTemplates` to StatefulSet |

```bash
# Check PVC status
kubectl get pvc -n production -l app=postgres

# Describe a stuck PVC
kubectl describe pvc data-postgres-0 -n production

# Check PV that the PVC is bound to
kubectl get pv | grep data-postgres-0
```
