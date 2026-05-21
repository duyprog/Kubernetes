# Priority & Preemption

PriorityClasses allow you to assign an importance score to Pods. When the cluster is under resource pressure, higher-priority Pods can **evict** lower-priority ones to get scheduled.

---

## How It Works

```
Cluster is full. New high-priority Pod arrives and cannot be scheduled.

Scheduler checks:
  → If we evict some lower-priority Pods, can this Pod fit?
  → Yes → evict those Pods → schedule the high-priority Pod

The evicted Pods re-enter the queue and will be re-scheduled when capacity is available.
```

---

## PriorityClass

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000            # integer priority — higher = more important
globalDefault: false      # if true, assigned to all Pods without a priorityClassName
preemptionPolicy: PreemptLowerPriority   # or Never
description: "Critical production services that must always run"
```

### Assign to a Pod / Deployment

```yaml
spec:
  priorityClassName: high-priority
  containers:
    - name: app
      image: my-app:1.0
```

---

## Priority Values — Recommended Tiers

```
2,000,001,000  system-node-critical   (built-in — kubelet, containerd)
2,000,000,000  system-cluster-critical (built-in — CoreDNS, kube-proxy, CNI)
─────────────────────────────────────────────────────────────
  1,000,000    high-priority          (critical production services)
    100,000    medium-priority        (standard production workloads)
     10,000    low-priority           (non-critical services, staging)
        100    batch-priority         (batch jobs, background tasks)
          0    (default if no priorityClassName set)
```

Higher values = more important. Values below 0 are also valid (useful for overprovisioning placeholder Pods — see [cluster-autoscaler.md](../operations/aws-eks/cluster-autoscaler.md)).

---

## Built-in PriorityClasses

Kubernetes ships with two built-in classes. Never assign these to your workloads.

```bash
kubectl get priorityclass
```

| Name | Value | For |
|---|---|---|
| `system-node-critical` | 2,000,001,000 | Node-level components: kubelet, containerd |
| `system-cluster-critical` | 2,000,000,000 | Cluster components: CoreDNS, kube-proxy, metrics-server |

---

## preemptionPolicy

| Value | Behavior |
|---|---|
| `PreemptLowerPriority` | This Pod can preempt lower-priority Pods to get scheduled (default) |
| `Never` | This Pod has a high priority score but will **not** preempt others — it waits in the queue |

`Never` is useful when you want to express importance for **eviction ordering** (which Pod gets evicted first under memory pressure) without enabling aggressive preemption.

---

## Priority vs Eviction

Priority affects two separate mechanisms:

### 1. Scheduling Preemption

When a high-priority Pod cannot be scheduled, the scheduler evicts lower-priority Pods to make room.

```
High-priority Pod → Pending (no room)
Scheduler finds: if we evict low-priority Pod A and B → room opens
Scheduler evicts A and B → high-priority Pod scheduled
A and B re-enter queue → scheduled when capacity returns
```

### 2. Node-Level Eviction (OOM/Disk Pressure)

When a node is under memory pressure, kubelet evicts Pods — lowest priority first, then highest memory usage within the same priority.

```
Node memory pressure:
  BestEffort Pods evicted first
  Then Burstable Pods (lowest priority first)
  Guaranteed Pods last
```

---

## Full Tier Example

```yaml
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical
value: 1000000
preemptionPolicy: PreemptLowerPriority
description: "Core revenue-critical services"

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: standard
value: 100000
preemptionPolicy: PreemptLowerPriority
description: "Standard production workloads"

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch
value: 100
preemptionPolicy: Never        # batch jobs wait; don't evict others
description: "Background batch jobs and data processing"

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: overprovisioning
value: -1                      # negative — below everything
preemptionPolicy: Never
description: "Placeholder Pods for warm node capacity"
```

---

## Protecting Against Preemption: PodDisruptionBudget

A PDB does **not** prevent preemption directly, but the scheduler respects it — it will not preempt Pods if doing so would violate the PDB.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
  namespace: production
spec:
  minAvailable: 2     # scheduler won't preempt if it would drop below 2 available Pods
  selector:
    matchLabels:
      app: my-app
```

---

## Commands

```bash
# List PriorityClasses
kubectl get priorityclass

# Check Pod priority
kubectl get pod <pod-name> -n <namespace> \
  -o jsonpath='{.spec.priority}'

# Find which Pods have no priorityClassName
kubectl get pods --all-namespaces \
  -o json | jq '.items[] | select(.spec.priorityClassName == null) | 
  "\(.metadata.namespace)/\(.metadata.name)"'
```

---

## Best Practices

- Define at least 3 tiers: critical, standard, batch
- Set `globalDefault: true` on your `standard` class so Pods without a class still get a reasonable priority
- Use `preemptionPolicy: Never` for batch jobs — they should wait, not evict others
- Always pair high-priority Deployments with a PDB so they can't be fully preempted
- Combine priority with Guaranteed QoS (`requests == limits`) for the most protection under resource pressure
