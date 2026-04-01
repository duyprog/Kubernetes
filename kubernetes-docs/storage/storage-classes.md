# StorageClasses — Dynamic Provisioning on AWS EKS

A StorageClass defines how PersistentVolumes are **dynamically provisioned** — the type of storage, performance tier, encryption, and binding behavior.

---

## StorageClass Spec

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"   # make this the default
provisioner: ebs.csi.aws.com
parameters:
  type: gp3                   # EBS volume type
  encrypted: "true"           # encrypt with AWS KMS
  kmsKeyId: ""                # empty = AWS managed key; set ARN for customer-managed key
  throughput: "125"           # MiB/s (gp3: 125–1000)
  iops: "3000"                # IOPS (gp3: 3000–16000)
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
allowVolumeExpansion: true
mountOptions:
  - noatime
```

---

## AWS EKS StorageClasses

### EBS gp3 (General Purpose SSD — Recommended)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
allowVolumeExpansion: true
```

**Use for:** General workloads, web servers, small to medium databases.

### EBS io2 (High Performance — Databases)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: io2-high-iops
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  iops: "10000"
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain         # Retain for databases
allowVolumeExpansion: true
```

**Use for:** Production databases (PostgreSQL, MySQL, MongoDB) requiring consistent high IOPS.

### EFS (Elastic File System — Shared Storage)

Requires the [AWS EFS CSI Driver](https://github.com/kubernetes-sigs/aws-efs-csi-driver).

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap           # use EFS Access Points
  fileSystemId: fs-0abc123def456789  # your EFS filesystem ID
  directoryPerms: "700"
  basePath: "/dynamic_provisioning"
  uid: "1000"
  gid: "1000"
volumeBindingMode: Immediate          # EFS is multi-AZ, no need to wait
reclaimPolicy: Delete
```

**Use for:** Shared logs, ML training data, CMS assets — anything needing `ReadWriteMany`.

---

## EBS vs EFS Comparison

| | EBS gp3 | EBS io2 | EFS |
|---|---|---|---|
| Access mode | RWO | RWO | RWX |
| Multi-AZ | No | No | Yes |
| Latency | ~1ms | <1ms | ~2-10ms |
| Throughput | Up to 1 GiB/s | Up to 4 GiB/s | Scales with usage |
| IOPS | Up to 16,000 | Up to 64,000 | Bursts/provisioned |
| Pricing | Moderate | Higher | Pay per GB used |
| Use case | General, databases | High-perf DBs | Shared files |

---

## Default StorageClass

Only one StorageClass should be marked as default. PVCs without `storageClassName` use the default.

```bash
# Check which StorageClass is default
kubectl get storageclass

# Change default StorageClass
kubectl patch storageclass gp2 \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

kubectl patch storageclass gp3 \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

> On EKS, the default StorageClass is often `gp2`. Replace it with `gp3` for better cost/performance.

---

## EBS CSI Driver Setup on EKS

### 1. Create IAM Role with IRSA

```bash
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster my-cluster \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve
```

### 2. Install the EBS CSI Driver Add-on

```bash
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name aws-ebs-csi-driver \
  --service-account-role-arn arn:aws:iam::<account-id>:role/AmazonEKS_EBS_CSI_DriverRole
```

### 3. Create gp3 StorageClass

```bash
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
allowVolumeExpansion: true
EOF
```

---

## EFS CSI Driver Setup on EKS

```bash
# Install EFS CSI Driver via Helm
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
helm upgrade --install aws-efs-csi-driver \
  aws-efs-csi-driver/aws-efs-csi-driver \
  --namespace kube-system \
  --set controller.serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::<account-id>:role/AmazonEKS_EFS_CSI_DriverRole
```

---

## Commands

```bash
# List StorageClasses
kubectl get storageclass

# Describe a StorageClass
kubectl describe storageclass gp3

# Watch PVC dynamic provisioning
kubectl get pvc -n production -w

# Find PVs created by a StorageClass
kubectl get pv -o jsonpath='{range .items[?(@.spec.storageClassName=="gp3")]}{.metadata.name}{"\n"}{end}'
```
