# PersistentVolumes — Types, Reclaim Policies & Static Provisioning

---

## PV Spec Reference

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
  labels:
    type: ebs
    environment: production
spec:
  capacity:
    storage: 50Gi

  volumeMode: Filesystem    # or Block (raw block device)
  accessModes:
    - ReadWriteOnce

  persistentVolumeReclaimPolicy: Retain

  storageClassName: gp3

  mountOptions:
    - noatime
    - nodiratime

  # One of: csi, nfs, hostPath, local, etc.
  csi:
    driver: ebs.csi.aws.com
    volumeHandle: vol-0abc123def456789
    fsType: ext4
    volumeAttributes:
      storage.kubernetes.io/csiProvisionerIdentity: "..."
```

---

## PV Types (CSI Drivers on AWS)

### EBS — Elastic Block Store

```yaml
csi:
  driver: ebs.csi.aws.com
  volumeHandle: vol-0abc123def456789
  fsType: ext4            # or xfs
```

- Block storage, `ReadWriteOnce` only
- Must be in the same AZ as the node
- Best performance for databases

### EFS — Elastic File System

```yaml
csi:
  driver: efs.csi.aws.com
  volumeHandle: fs-0abc123def456789::fsap-0abc123def456789
```

- Network filesystem, `ReadWriteMany`
- Accessible from multiple AZs simultaneously
- Higher latency than EBS

### NFS (on-premises or self-managed)

```yaml
nfs:
  server: nfs-server.example.com
  path: /exports/data
```

---

## Reclaim Policies

### Retain

PV transitions to `Released` phase when PVC is deleted. Data is preserved. Manual intervention required to reuse the PV.

**Manual cleanup process:**
```bash
# 1. Delete the PVC
kubectl delete pvc my-pvc -n production

# 2. PV is now in "Released" state — still references old PVC
kubectl get pv my-pv

# 3. Edit PV to remove claimRef (makes it Available again)
kubectl edit pv my-pv
# Remove the entire .spec.claimRef section

# 4. PV is now Available — can be rebound to a new PVC
```

### Delete

PV and underlying storage resource are deleted when PVC is deleted. Default for dynamically provisioned PVs.

```bash
# Verify reclaim policy before deleting PVC
kubectl get pv <pv-name> -o jsonpath='{.spec.persistentVolumeReclaimPolicy}'
```

---

## Static Provisioning (Pre-existing Storage)

Use when you have existing EBS volumes, NFS mounts, or other storage that should be imported into Kubernetes.

### Step 1 — Create the PV

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: existing-ebs-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: gp3
  csi:
    driver: ebs.csi.aws.com
    volumeHandle: vol-0existingvolumeid
    fsType: ext4
```

### Step 2 — Create a PVC that binds to it

Use a `selector` or matching `storageClassName` + capacity to target the specific PV.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: existing-data-pvc
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClassName: gp3
  # Optional: bind to a specific PV by label
  selector:
    matchLabels:
      type: ebs
```

Or bind explicitly by name (set `claimRef` on the PV):

```yaml
# In PV spec
spec:
  claimRef:
    name: existing-data-pvc
    namespace: production
```

---

## Expanding PersistentVolumes

Requires `allowVolumeExpansion: true` on the StorageClass.

```bash
# Edit the PVC to request more storage
kubectl edit pvc my-pvc -n production
# Change: storage: 50Gi → storage: 100Gi
```

For `Filesystem` volumes, the filesystem is expanded the next time the Pod starts (or immediately for online expansion if supported by the CSI driver).

```bash
# Check expansion status
kubectl describe pvc my-pvc -n production
# Look for: "Resizing" or "FileSystemResizePending" in conditions
```

---

## Volume Snapshots

Create point-in-time snapshots of PVs (requires VolumeSnapshot CRDs and a compatible CSI driver).

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: data-snapshot-2024-01-15
  namespace: production
spec:
  volumeSnapshotClassName: csi-aws-vsc
  source:
    persistentVolumeClaimName: my-pvc
```

```bash
# Check snapshot status
kubectl get volumesnapshot -n production
kubectl describe volumesnapshot data-snapshot-2024-01-15 -n production
```

---

## Local Volumes (Node-Local SSD)

High performance — data is tied to a specific node. If the node goes down, the Pod cannot reschedule.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-ssd-pv
spec:
  capacity:
    storage: 500Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/ssd/data
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - node-with-ssd
```

> Use only when you need maximum disk I/O performance and can tolerate node-local affinity. Prefer EBS gp3 for most workloads.
