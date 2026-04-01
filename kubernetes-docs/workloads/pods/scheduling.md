# Scheduling — How Kubernetes Selects the Right Node

The **kube-scheduler** assigns unscheduled Pods to nodes. It runs through two phases for each Pod:

1. **Filtering** — eliminate nodes that cannot run the Pod (insufficient resources, taints without tolerations, etc.)
2. **Scoring** — rank remaining nodes and pick the best one

You can influence this process using the mechanisms below.

---

## nodeSelector (simple)

The simplest way to constrain a Pod to nodes with specific labels. All labels must match — it is a hard AND rule.

```yaml
spec:
  nodeSelector:
    disktype: ssd
    kubernetes.io/arch: amd64
```

```bash
# Label a node
kubectl label node <node-name> disktype=ssd

# Verify node labels
kubectl get nodes --show-labels
kubectl get nodes -l disktype=ssd
```

**Limitation:** No OR conditions, no soft preferences. Use `nodeAffinity` for more control.

---

## Node Affinity

Node affinity is a more expressive version of `nodeSelector`. It supports operators (`In`, `NotIn`, `Exists`, `Gt`, `Lt`) and separates hard from soft rules.

### requiredDuringSchedulingIgnoredDuringExecution (hard rule)

Pod will **not be scheduled** unless a matching node exists.

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              # One expression per item — all must match (AND within a term)
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                  - us-east-1a
                  - us-east-1b
              - key: kubernetes.io/arch
                operator: In
                values:
                  - amd64

          # A second term is an OR — the Pod can go to either term's matching nodes
          - matchExpressions:
              - key: node.kubernetes.io/instance-type
                operator: In
                values:
                  - m5.xlarge
                  - m5.2xlarge
```

> Multiple `matchExpressions` within one `nodeSelectorTerms` item are **AND**-ed.
> Multiple items in `nodeSelectorTerms` are **OR**-ed.

### preferredDuringSchedulingIgnoredDuringExecution (soft rule)

Scheduler prefers matching nodes but places the Pod elsewhere if no match exists. Use `weight` (1–100) to prioritize.

```yaml
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80
          preference:
            matchExpressions:
              - key: node-type
                operator: In
                values:
                  - on-demand
        - weight: 20
          preference:
            matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                  - us-east-1a
```

### IgnoredDuringExecution

Both modes include this suffix — it means: **if node labels change after the Pod is running, the Pod is not evicted**. A future `requiredDuringExecutionIgnoredDuringScheduling` may be added.

### Combining Hard and Soft

```yaml
spec:
  affinity:
    nodeAffinity:
      # Hard: must be in a specific region
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/region
                operator: In
                values:
                  - us-east-1

      # Soft: prefer SSD nodes
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

## Anti-Node Affinity

"Anti-node affinity" is not a separate field — it is achieved using `NotIn` or `DoesNotExist` operators within `nodeAffinity`.

### Avoid Specific Node Types

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              # Never schedule on spot instances
              - key: node.kubernetes.io/lifecycle
                operator: NotIn
                values:
                  - spot
```

### Avoid Specific Availability Zones

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: NotIn
                values:
                  - us-east-1c     # avoid this AZ (e.g., capacity issues)
```

---

## Pod Affinity & Anti-Affinity

Control placement relative to **other Pods**, not nodes.

### Pod Affinity (co-locate)

Schedule this Pod **near** Pods with matching labels.

```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: redis
          topologyKey: kubernetes.io/hostname   # same node
```

**Use case:** App Pod should run on the same node as its Redis cache to minimize latency.

### Pod Anti-Affinity (spread out)

Schedule this Pod **away from** Pods with matching labels.

```yaml
spec:
  affinity:
    podAntiAffinity:
      # Hard: never two replicas on the same node
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: my-app
          topologyKey: kubernetes.io/hostname

      # Soft: prefer spreading across AZs
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: my-app
            topologyKey: topology.kubernetes.io/zone
```

**Use case:** Ensure replicas are spread across nodes and AZs for high availability.

### `topologyKey` values

| topologyKey | Scope |
|---|---|
| `kubernetes.io/hostname` | Per node |
| `topology.kubernetes.io/zone` | Per availability zone |
| `topology.kubernetes.io/region` | Per region |
| Custom label | Any grouping you define on nodes |

---

## Topology Spread Constraints

A cleaner way to spread Pods evenly across nodes, zones, or any topology domain — without complex anti-affinity rules.

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1                          # max difference between domain counts
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule   # hard: or ScheduleAnyway (soft)
      labelSelector:
        matchLabels:
          app: my-app

    - maxSkew: 1
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: ScheduleAnyway  # soft
      labelSelector:
        matchLabels:
          app: my-app
```

**Example:** With 3 AZs and `maxSkew: 1`, Kubernetes ensures no AZ has more than 1 extra Pod compared to the least-loaded AZ.

```
AZ-a: [Pod Pod Pod]
AZ-b: [Pod Pod Pod]   ← balanced (skew = 0)
AZ-c: [Pod Pod Pod]

vs. unbalanced:

AZ-a: [Pod Pod Pod Pod]
AZ-b: [Pod]           ← skew = 3 — violates maxSkew: 1
AZ-c: [Pod]
```

---

## Taints and Tolerations

Taints are applied to **nodes** to repel Pods. Tolerations are applied to **Pods** to allow scheduling on tainted nodes.

### Applying Taints to Nodes

```bash
# Add a taint
kubectl taint node <node-name> key=value:effect

# Remove a taint
kubectl taint node <node-name> key=value:effect-

# Examples
kubectl taint node node-1 gpu=true:NoSchedule
kubectl taint node node-2 environment=production:NoSchedule
kubectl taint node node-3 spot=true:PreferNoSchedule
```

### Taint Effects

| Effect | Behavior |
|---|---|
| `NoSchedule` | New Pods without toleration will **not** be scheduled on this node |
| `PreferNoSchedule` | Scheduler **tries to avoid** placing Pods without toleration, but is not guaranteed |
| `NoExecute` | New Pods without toleration are not scheduled **AND** existing Pods without toleration are **evicted** |

### Tolerations in Pod Spec

```yaml
spec:
  tolerations:
    # Exact match — tolerate a specific taint
    - key: "gpu"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"

    # Exists — tolerate any value for this key
    - key: "spot"
      operator: "Exists"
      effect: "NoSchedule"

    # Tolerate all taints (use sparingly — e.g., DaemonSets)
    - operator: "Exists"

    # With tolerationSeconds — eviction delayed after NoExecute taint appears
    - key: "node.kubernetes.io/not-ready"
      operator: "Exists"
      effect: "NoExecute"
      tolerationSeconds: 300    # Pod stays on node for 5 min before eviction
```

### Common Use Cases

**Dedicated GPU node group:**

```yaml
# Node taint
kubectl taint node gpu-node-1 nvidia.com/gpu=true:NoSchedule

# Only ML Pods tolerate this
spec:
  tolerations:
    - key: "nvidia.com/gpu"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
  resources:
    limits:
      nvidia.com/gpu: "1"
```

**Spot instance node group:**

```yaml
# All spot nodes are tainted
kubectl taint node spot-node-1 spot=true:NoSchedule

# Only batch/non-critical Pods tolerate spot
spec:
  tolerations:
    - key: "spot"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
```

**Drain a node gracefully (NoExecute taint):**

```bash
# Cordon (prevent new scheduling) + drain (evict existing Pods)
kubectl cordon <node-name>
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# The above is equivalent to applying a NoExecute taint
```

---

## Scheduling Summary

| Mechanism | Controls | Hard/Soft | Level |
|---|---|---|---|
| `nodeSelector` | Which nodes | Hard only | Node labels |
| `nodeAffinity` | Which nodes | Hard + Soft | Node labels (expressive) |
| `podAffinity` | Near which Pods | Hard + Soft | Pod labels |
| `podAntiAffinity` | Away from which Pods | Hard + Soft | Pod labels |
| `topologySpreadConstraints` | Even distribution | Hard + Soft | Any topology |
| Taints + Tolerations | Repel Pods from nodes | Hard + Soft | Node taints |

> **Best practice for HA:** Always use `podAntiAffinity` or `topologySpreadConstraints` to spread replicas across nodes and zones. A Deployment with 3 replicas on 1 node provides no redundancy.

---

## Commands

```bash
# Check why a Pod is not scheduled
kubectl describe pod <pod-name> -n <namespace>
# Look for: "Events: FailedScheduling" and the reason

# Check node taints
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints

# Check node labels
kubectl get nodes --show-labels
kubectl get nodes -l topology.kubernetes.io/zone=us-east-1a

# Simulate scheduling (dry-run)
kubectl apply -f pod.yaml --dry-run=server
```
