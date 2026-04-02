# Scaling — Overview & Decision Guide

Kubernetes offers multiple layers of scaling. Picking the right one depends on **what you're scaling** and **what drives the load**.

---

## The Three Layers

```
┌─────────────────────────────────────────────────────────┐
│  Layer 3 — Cluster Scaling                              │
│  Add / remove NODES                                     │
│  → Cluster Autoscaler, Karpenter                        │
├─────────────────────────────────────────────────────────┤
│  Layer 2 — Workload Horizontal Scaling                  │
│  Add / remove PODS (replicas)                           │
│  → HPA, KEDA                                            │
├─────────────────────────────────────────────────────────┤
│  Layer 1 — Workload Vertical Scaling                    │
│  Resize CPU / memory per POD                            │
│  → VPA                                                  │
└─────────────────────────────────────────────────────────┘
```

All three layers work together. A typical production setup:
- **VPA** (Off mode) → right-size requests/limits
- **HPA** → scale replicas based on CPU or custom metrics
- **Cluster Autoscaler / Karpenter** → add nodes when HPA needs more capacity

---

## Decision Matrix

| Scenario | Solution |
|---|---|
| Traffic spikes (CPU/memory driven) | [HPA](hpa.md) |
| Traffic spikes (queue depth, events) | [KEDA](keda.md) |
| Containers are over/under-resourced | [VPA](vpa.md) in `Off` mode → review → `Auto` |
| Cluster runs out of nodes | [Cluster Autoscaler](../operations/aws-eks/cluster-autoscaler.md) or Karpenter |
| Scale to zero when idle | [KEDA](keda.md) with `minReplicaCount: 0` |
| Batch job workers (SQS, Kafka) | [KEDA](keda.md) |
| Stateful app needing more replicas | HPA on StatefulSet (careful — shared state) |
| Unknown correct resource sizing | [VPA](vpa.md) `Off` mode first |

---

## Combining Scalers

| Combination | Verdict |
|---|---|
| HPA (CPU) + VPA (Auto) | **Conflict** — both adjust CPU, fight each other |
| HPA (custom metrics) + VPA (Auto) | **OK** — HPA scales replicas, VPA resizes containers |
| HPA (CPU) + VPA (Off) | **Recommended** — VPA gives recommendations, HPA scales |
| KEDA + VPA (Auto) | **OK** — different axes (replicas vs resources) |
| HPA + KEDA | **OK** — KEDA creates an HPA under the hood; don't create a separate HPA for the same target |

---

## Manual Scaling

Quick override — useful for emergencies or pre-scaling before a known event.

```bash
# Scale a Deployment
kubectl scale deployment my-app --replicas=10 -n production

# Scale a StatefulSet
kubectl scale statefulset postgres --replicas=3 -n production

# Patch directly
kubectl patch deployment my-app \
  -p '{"spec":{"replicas":10}}' \
  -n production
```

> If HPA is active on the same workload, manual scaling is temporary — HPA will override it on the next evaluation cycle (every 15s).

---

## Scaling Scope by Workload Type

| Workload | Horizontal (replicas) | Vertical (resources) | Notes |
|---|---|---|---|
| Deployment | HPA / KEDA / manual | VPA | Most flexible |
| StatefulSet | HPA / manual | VPA | Careful with shared state; PVCs created per replica |
| DaemonSet | Not applicable | VPA | Always one Pod per node |
| Job | `parallelism` field | VPA | Completions-based, not replica-based |
| CronJob | Inherited from Job | VPA | Per-run parallelism |

---

## Further Reading

- [HPA — Horizontal Pod Autoscaler](hpa.md)
- [VPA — Vertical Pod Autoscaler](vpa.md)
- [KEDA — Event-Driven Autoscaling](keda.md)
- [Cluster Autoscaler (EKS)](../operations/aws-eks/cluster-autoscaler.md)
