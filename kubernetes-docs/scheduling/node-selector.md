# nodeSelector

The simplest way to constrain a Pod to nodes that have specific labels. Every label in `nodeSelector` must match — it is a hard **AND** rule with no soft preference or OR logic.

---

## Syntax

```yaml
spec:
  nodeSelector:
    disktype: ssd
    kubernetes.io/arch: amd64
```

The Pod will only schedule on nodes that have **both** labels present with those exact values.

---

## Well-Known Node Labels

Kubernetes and cloud providers automatically attach these labels to nodes:

| Label | Example Values | Description |
|---|---|---|
| `kubernetes.io/hostname` | `ip-10-0-1-42.ec2.internal` | Unique node name |
| `kubernetes.io/arch` | `amd64`, `arm64` | CPU architecture |
| `kubernetes.io/os` | `linux`, `windows` | Node OS |
| `topology.kubernetes.io/zone` | `us-east-1a` | Availability zone |
| `topology.kubernetes.io/region` | `us-east-1` | Cloud region |
| `node.kubernetes.io/instance-type` | `m5.xlarge` | EC2 instance type |
| `eks.amazonaws.com/nodegroup` | `general` | EKS managed node group name |
| `eks.amazonaws.com/capacityType` | `ON_DEMAND`, `SPOT` | EKS node capacity type |
| `node.kubernetes.io/lifecycle` | `normal`, `spot` | Node lifecycle (AWS) |

---

## Common Examples

### Schedule on SSD nodes only

```yaml
spec:
  nodeSelector:
    disktype: ssd
```

### Schedule on ARM64 (Graviton on EKS)

```yaml
spec:
  nodeSelector:
    kubernetes.io/arch: arm64
```

### Schedule only in a specific AZ

```yaml
spec:
  nodeSelector:
    topology.kubernetes.io/zone: us-east-1a
```

> Pinning to one AZ removes HA — use [topologySpreadConstraints](topology-spread.md) instead for spreading across AZs.

### Schedule on On-Demand nodes only (avoid Spot)

```yaml
spec:
  nodeSelector:
    eks.amazonaws.com/capacityType: ON_DEMAND
```

---

## Managing Node Labels

```bash
# Add a custom label to a node
kubectl label node <node-name> disktype=ssd

# Remove a label
kubectl label node <node-name> disktype-

# Overwrite an existing label
kubectl label node <node-name> disktype=nvme --overwrite

# List nodes filtered by label
kubectl get nodes -l disktype=ssd

# Show specific label columns
kubectl get nodes -L disktype,topology.kubernetes.io/zone
```

---

## Limitations

| Limitation | Solution |
|---|---|
| Only hard rules — no soft preference | Use [nodeAffinity](node-affinity.md) `preferred` rules |
| No OR conditions across labels | Use [nodeAffinity](node-affinity.md) `In` operator with multiple values |
| No expressions (NotIn, Gt, Lt) | Use [nodeAffinity](node-affinity.md) |
| Cannot express "prefer but don't require" | Use [nodeAffinity](node-affinity.md) |

For anything more complex than a simple label match, use `nodeAffinity`.
