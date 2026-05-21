# Scheduling — How Kubernetes Places Pods on Nodes

The **kube-scheduler** is responsible for assigning unscheduled Pods to nodes. It never runs containers — it only decides *where* they go.

---

## Scheduling Pipeline

Every unscheduled Pod goes through two phases:

```
Unscheduled Pod (nodeName = "")
         ↓
┌────────────────────────────────────────┐
│  Phase 1: FILTERING                    │
│  Eliminate nodes that cannot run Pod   │
│                                        │
│  • Insufficient CPU / memory           │
│  • Node is cordoned (Unschedulable)    │
│  • Taint without matching toleration   │
│  • nodeSelector label mismatch         │
│  • nodeAffinity hard rule mismatch     │
│  • Pod anti-affinity hard rule         │
│  • Volume zone / topology mismatch     │
│  • HostPort conflict on node           │
└──────────────┬─────────────────────────┘
               │ remaining candidate nodes
┌──────────────▼─────────────────────────┐
│  Phase 2: SCORING                      │
│  Rank remaining nodes (0–100 per rule) │
│                                        │
│  • LeastAllocated   — most free CPU+mem│
│  • BalancedResource — even CPU/mem use │
│  • NodeAffinity     — preferred rules  │
│  • PodAffinity      — co-locate Pods   │
│  • PodAntiAffinity  — spread Pods      │
│  • ImageLocality    — image cached?    │
│  • TaintToleration  — prefer no taints │
└──────────────┬─────────────────────────┘
               │ highest scoring node wins
               ↓
       nodeName = "node-3" written to Pod spec
               ↓
       kubelet on node-3 picks up the Pod and starts containers
```

If **no node passes filtering**, the Pod stays `Pending` with a `FailedScheduling` event.

---

## Scheduling Mechanisms — Quick Reference

| Mechanism | Controls | Scope | Hard / Soft |
|---|---|---|---|
| [nodeSelector](node-selector.md) | Which nodes | Node labels | Hard only |
| [Node Affinity](node-affinity.md) | Which nodes | Node labels | Hard + Soft |
| [Pod Affinity](pod-affinity.md) | Near which Pods | Pod labels | Hard + Soft |
| [Pod Anti-Affinity](pod-affinity.md#pod-anti-affinity) | Away from which Pods | Pod labels | Hard + Soft |
| [Taints & Tolerations](taints-tolerations.md) | Repel Pods from nodes | Node taints | Hard + Soft |
| [Topology Spread](topology-spread.md) | Even distribution across domains | Pod labels | Hard + Soft |
| [Priority & Preemption](priority-preemption.md) | Evict lower-priority Pods | PriorityClass | — |

---

## Choosing the Right Mechanism

```
"I need the Pod on a specific type of node"
  → nodeSelector (simple) or nodeAffinity (expressive)

"I need replicas spread across AZs / nodes"
  → topologySpreadConstraints (cleanest)
  → podAntiAffinity (more control, more verbose)

"I want to dedicate nodes to specific workloads"
  → Taints + Tolerations (repel others) + nodeAffinity (attract the right ones)

"I need a Pod near another Pod (low latency)"
  → podAffinity

"I need critical workloads to survive resource pressure"
  → PriorityClass + Preemption
```

---

## Diagnosing Scheduling Failures

```bash
# The Events section is the most useful — always read it first
kubectl describe pod <pod-name> -n <namespace>

# Common FailedScheduling reasons:
# "Insufficient cpu"              → increase node capacity or reduce requests
# "node(s) had taints"            → add toleration or remove taint
# "node(s) didn't match nodeSelector" → fix label or nodeSelector
# "node(s) didn't match Pod's nodeAffinity" → fix affinity rule or node labels
# "pod has unbound PersistentVolumeClaims" → fix PVC/StorageClass issue
# "0/3 nodes are available"       → all nodes filtered out — check all constraints

# See node labels (for debugging affinity / nodeSelector)
kubectl get nodes --show-labels
kubectl get nodes -L topology.kubernetes.io/zone,node.kubernetes.io/instance-type

# See node taints
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

---

## Further Reading

- [nodeSelector](node-selector.md)
- [Node Affinity](node-affinity.md)
- [Pod Affinity & Anti-Affinity](pod-affinity.md)
- [Taints & Tolerations](taints-tolerations.md)
- [Topology Spread Constraints](topology-spread.md)
- [Priority & Preemption](priority-preemption.md)
