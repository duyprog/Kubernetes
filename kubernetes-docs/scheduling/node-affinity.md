# Node Affinity

Node affinity is the expressive replacement for `nodeSelector`. It supports operators, OR logic between terms, and separates hard rules from soft preferences.

---

## Two Rule Types

| Field | Type | Behavior |
|---|---|---|
| `requiredDuringSchedulingIgnoredDuringExecution` | Hard | Pod **will not schedule** if no node matches |
| `preferredDuringSchedulingIgnoredDuringExecution` | Soft | Scheduler **prefers** matching nodes; places Pod elsewhere if none match |

> **IgnoredDuringExecution** — if a node's labels change *after* the Pod is running, the Pod is **not evicted**. A future `requiredDuringExecution` variant would evict, but it doesn't exist yet.

---

## Hard Rule (required)

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                  - us-east-1a
                  - us-east-1b
              - key: kubernetes.io/arch
                operator: In
                values:
                  - amd64
```

**Logic:** expressions within one `nodeSelectorTerms` item are **AND**-ed. Multiple items are **OR**-ed.

```
nodeSelectorTerms:
  - [expr1 AND expr2]   ← term A
  - [expr3]             ← term B

Result: (expr1 AND expr2) OR expr3
```

---

## Soft Rule (preferred)

Use `weight` (1–100) to rank preferences. Higher weight = stronger preference.

```yaml
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80
          preference:
            matchExpressions:
              - key: node.kubernetes.io/lifecycle
                operator: NotIn
                values:
                  - spot          # strongly prefer on-demand nodes
        - weight: 20
          preference:
            matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                  - us-east-1a   # weakly prefer AZ-a
```

The scheduler adds all matching weights to each node's score and picks the highest.

---

## Hard + Soft Combined

```yaml
spec:
  affinity:
    nodeAffinity:
      # Must be in us-east-1 region
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/region
                operator: In
                values:
                  - us-east-1

      # Prefer SSD nodes, but not required
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          preference:
            matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
```

---

## Operators

| Operator | Meaning | Example |
|---|---|---|
| `In` | Label value is in the list | `zone In [us-east-1a, us-east-1b]` |
| `NotIn` | Label value is NOT in the list | `lifecycle NotIn [spot]` |
| `Exists` | Label key exists (any value) | `gpu Exists` |
| `DoesNotExist` | Label key does not exist | `experimental DoesNotExist` |
| `Gt` | Label value (integer) is greater than | `generation Gt [2]` |
| `Lt` | Label value (integer) is less than | `generation Lt [5]` |

---

## Anti-Node Affinity

There is no `nodeAntiAffinity` field — use `NotIn` or `DoesNotExist` within `nodeAffinity` to achieve the same result.

### Avoid spot instances

```yaml
requiredDuringSchedulingIgnoredDuringExecution:
  nodeSelectorTerms:
    - matchExpressions:
        - key: eks.amazonaws.com/capacityType
          operator: NotIn
          values:
            - SPOT
```

### Avoid nodes without a specific label

```yaml
requiredDuringSchedulingIgnoredDuringExecution:
  nodeSelectorTerms:
    - matchExpressions:
        - key: approved-for-production
          operator: Exists      # only nodes explicitly labeled for production
```

### Avoid a specific AZ

```yaml
requiredDuringSchedulingIgnoredDuringExecution:
  nodeSelectorTerms:
    - matchExpressions:
        - key: topology.kubernetes.io/zone
          operator: NotIn
          values:
            - us-east-1c        # AZ with capacity issues
```

---

## Practical Patterns

### Dedicated GPU nodes

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: nvidia.com/gpu
                operator: Exists
  tolerations:
    - key: nvidia.com/gpu
      operator: Exists
      effect: NoSchedule
```

> Node affinity *attracts* the Pod to GPU nodes. Toleration *allows* it to schedule on tainted GPU nodes. Both are needed.

### Graviton (ARM64) node group on EKS

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: kubernetes.io/arch
                operator: In
                values:
                  - arm64
              - key: eks.amazonaws.com/nodegroup
                operator: In
                values:
                  - graviton-workers
```

### Prefer on-demand, tolerate spot

```yaml
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          preference:
            matchExpressions:
              - key: eks.amazonaws.com/capacityType
                operator: In
                values:
                  - ON_DEMAND
  tolerations:
    - key: spot
      operator: Equal
      value: "true"
      effect: NoSchedule
```

---

## nodeSelector vs nodeAffinity

| Need | Use |
|---|---|
| Simple label match, no logic | `nodeSelector` |
| OR logic across label values | `nodeAffinity` + `In` with multiple values |
| Soft preference with weight | `nodeAffinity` `preferred` |
| NOT a label value | `nodeAffinity` + `NotIn` |
| Label key must exist | `nodeAffinity` + `Exists` |
| Numeric range | `nodeAffinity` + `Gt` / `Lt` |

`nodeSelector` and `nodeAffinity` can coexist — both must be satisfied.
