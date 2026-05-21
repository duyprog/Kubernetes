# Topology Spread Constraints

Topology Spread Constraints distribute Pods **evenly** across topology domains (zones, nodes, racks) using a simple `maxSkew` model. They are the cleaner, more predictable alternative to pod anti-affinity for spreading workloads.

---

## The Problem They Solve

Without spread constraints, the scheduler may pack all replicas into one AZ:

```
AZ-a: [Pod Pod Pod Pod Pod]   ← all 5 here
AZ-b: []
AZ-c: []

→ AZ-a outage = total service outage
```

With spread constraints:

```
AZ-a: [Pod Pod]
AZ-b: [Pod Pod]
AZ-c: [Pod]

→ AZ-a outage = 3/5 Pods still running
```

---

## Core Concept: maxSkew

`maxSkew` is the **maximum allowed difference** in Pod count between the most-loaded and least-loaded topology domain.

```
3 AZs, 6 Pods, maxSkew: 1

Allowed:      AZ-a:2  AZ-b:2  AZ-c:2   (skew = 0) ✓
Also allowed: AZ-a:3  AZ-b:2  AZ-c:1   (skew = 2) ✗ — violates maxSkew:1
              AZ-a:2  AZ-b:2  AZ-c:1   (skew = 1) ✓
```

---

## Spec

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: my-app
      minDomains: 3               # require at least 3 AZs to be eligible (k8s 1.25+)
      nodeAffinityPolicy: Honor   # k8s 1.26+: respect nodeAffinity when counting
      nodeTaintsPolicy: Honor     # k8s 1.26+: exclude tainted nodes from count
```

---

## whenUnsatisfiable

What to do when spreading cannot satisfy `maxSkew`:

| Value | Behavior |
|---|---|
| `DoNotSchedule` | Hard — Pod stays `Pending` if constraint cannot be met |
| `ScheduleAnyway` | Soft — Pod schedules on the node that minimizes skew violation |

---

## Common Patterns

### Spread across AZs (hard)

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: my-app
```

### Spread across nodes (hard)

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: my-app
```

### Spread across both AZs and nodes (two constraints)

Both constraints must be satisfied simultaneously.

```yaml
topologySpreadConstraints:
  # No more than 1 Pod difference between AZs
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: my-app

  # No more than 1 Pod per node
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: my-app
```

### Soft spread (best effort)

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway   # soft
    labelSelector:
      matchLabels:
        app: my-app
```

Use soft spreading when you want even distribution but not at the cost of Pods getting stuck `Pending`.

### Mixed: hard node spread + soft AZ spread

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule   # hard: never two on same node
    labelSelector:
      matchLabels:
        app: my-app

  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway  # soft: prefer AZ balance
    labelSelector:
      matchLabels:
        app: my-app
```

---

## minDomains (k8s 1.25+)

Require a minimum number of eligible topology domains. If fewer than `minDomains` exist, the constraint treats `maxSkew` as if there are `minDomains` empty domains — preventing all Pods from piling into the one available domain.

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    minDomains: 3       # require 3 AZs; Pods stay Pending if fewer than 3 exist
    labelSelector:
      matchLabels:
        app: my-app
```

---

## Cluster-Level Defaults

Set default spread constraints for all Pods in the cluster (via scheduler config). Useful so every Deployment benefits without per-manifest boilerplate.

```yaml
# kube-scheduler config
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler
    pluginConfig:
      - name: PodTopologySpread
        args:
          defaultConstraints:
            - maxSkew: 1
              topologyKey: topology.kubernetes.io/zone
              whenUnsatisfiable: ScheduleAnyway
            - maxSkew: 1
              topologyKey: kubernetes.io/hostname
              whenUnsatisfiable: ScheduleAnyway
          defaultingType: List   # only apply to Pods without explicit constraints
```

---

## Topology Spread vs Pod Anti-Affinity

| | topologySpreadConstraints | podAntiAffinity |
|---|---|---|
| Goal | Even distribution with max skew | Keep Pod away from matching Pods |
| Scale behaviour | Fills all domains before skewing | May leave nodes/zones empty |
| Multiple domains | Single constraint covers all domains | One rule per topology level |
| Syntax | Concise | Verbose |
| Performance | Efficient at scale | Expensive at scale (O(n×m)) |
| "Never same node" | `maxSkew:1` + `hostname` | `required` + `hostname` |
| **Recommendation** | Preferred for HA spreading | Use for strict isolation from another workload |

---

## Troubleshooting

```bash
# Pod stuck Pending — check scheduling failure reason
kubectl describe pod <pod-name> -n <namespace>
# Look for: "didn't match pod topology spread constraints"

# Check how many nodes exist per zone
kubectl get nodes -L topology.kubernetes.io/zone

# Check how many Pods are in each zone
kubectl get pods -n production -o wide | awk '{print $7}' | sort | uniq -c
```

**Common causes of constraint violations:**
- Not enough nodes in each zone (cluster needs more capacity)
- `minDomains` set higher than available AZs
- `labelSelector` not matching any running Pods (wrong labels)
- Hard (`DoNotSchedule`) constraint impossible to satisfy → use `ScheduleAnyway` or add nodes
