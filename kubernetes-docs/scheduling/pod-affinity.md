# Pod Affinity & Anti-Affinity

Where node affinity controls *which nodes* a Pod can run on, pod affinity controls placement *relative to other Pods* — co-locating or spreading them within a topology domain.

---

## Pod Affinity

Schedule this Pod **near** Pods matching a label selector — on the same node, zone, or any other topology domain.

```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: redis
          topologyKey: kubernetes.io/hostname   # "near" = same node
```

**Use case:** App Pod should run on the same node as its Redis cache to minimize latency.

### Soft pod affinity (prefer co-location)

```yaml
spec:
  affinity:
    podAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: redis
            topologyKey: kubernetes.io/hostname
```

---

## Pod Anti-Affinity

Schedule this Pod **away from** Pods matching a label selector — spread replicas across nodes, zones, or racks.

### Hard: never two replicas on the same node

```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: my-app
          topologyKey: kubernetes.io/hostname
```

> Hard anti-affinity on `hostname` limits max replicas to the number of nodes. If you have 3 nodes and request 4 replicas, the 4th Pod will be `Pending`.

### Soft: prefer spreading across AZs

```yaml
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: my-app
            topologyKey: topology.kubernetes.io/zone
```

### Hard + Soft combined (HA pattern)

Never on the same node, but prefer different AZs:

```yaml
spec:
  affinity:
    podAntiAffinity:
      # Hard: never two replicas on same node
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: my-app
          topologyKey: kubernetes.io/hostname

      # Soft: prefer different AZs
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: my-app
            topologyKey: topology.kubernetes.io/zone
```

---

## topologyKey

Defines the **scope** of "near" or "away from". The scheduler groups nodes by the value of this label.

| topologyKey | Scope | Example grouping |
|---|---|---|
| `kubernetes.io/hostname` | Per node | Each node is its own domain |
| `topology.kubernetes.io/zone` | Per AZ | `us-east-1a`, `us-east-1b`, `us-east-1c` |
| `topology.kubernetes.io/region` | Per region | `us-east-1` |
| Custom label | Any grouping | `rack=rack-1`, `datacenter=dc-east` |

---

## namespaceSelector

By default, pod affinity only looks at Pods in the **same namespace**. Use `namespaceSelector` or `namespaces` to cross namespaces.

```yaml
requiredDuringSchedulingIgnoredDuringExecution:
  - labelSelector:
      matchLabels:
        app: redis
    namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: shared-services
    topologyKey: kubernetes.io/hostname
```

---

## Practical Patterns

### Co-locate app + sidecar cache (affinity)

```yaml
# Redis Pod label: app=redis, tier=cache
# App Pod should run on the same node as Redis

spec:
  affinity:
    podAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: redis
                tier: cache
            topologyKey: kubernetes.io/hostname
```

### Spread a Deployment across all 3 AZs (anti-affinity)

```yaml
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: my-app
            topologyKey: topology.kubernetes.io/zone
```

### Keep Pods away from a noisy-neighbor service

```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: batch-processor    # CPU-heavy batch job
          topologyKey: kubernetes.io/hostname
```

---

## Pod Affinity vs Topology Spread Constraints

Both can spread Pods across domains. The difference:

| | Pod Anti-Affinity | topologySpreadConstraints |
|---|---|---|
| Goal | Keep Pods away from matching Pods | Even distribution with a max skew |
| Scale-out behavior | May leave nodes empty | Fills domains evenly |
| Syntax | Verbose | Cleaner |
| Best for | Strict "never same node" | Even spread across AZs/nodes |

> For most HA spread scenarios, [topologySpreadConstraints](topology-spread.md) is cleaner and more predictable. Use pod anti-affinity when you need strict isolation from a *specific* other workload.

---

## Performance Note

Pod affinity/anti-affinity is computationally expensive — the scheduler must compare every candidate node against all existing Pods. For large clusters (1000+ nodes, many Pods), prefer `topologySpreadConstraints` for simple spreading scenarios.
