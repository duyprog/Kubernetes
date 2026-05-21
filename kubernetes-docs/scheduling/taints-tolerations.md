# Taints & Tolerations

Taints and tolerations work together to **repel Pods from nodes**. A taint on a node pushes Pods away; a toleration on a Pod says "I can handle that taint."

```
Node with taint  ──repels──►  Pod without toleration   (not scheduled)
Node with taint  ──allows──►  Pod with matching toleration  (scheduled)
```

Tolerations are **permissive** — they allow a Pod to schedule on a tainted node, but don't guarantee it. Combine with [nodeAffinity](node-affinity.md) to both allow and attract.

---

## Taint Effects

| Effect | Behavior on Pods WITHOUT a matching toleration |
|---|---|
| `NoSchedule` | New Pods will **not** be scheduled on this node. Existing Pods are unaffected. |
| `PreferNoSchedule` | Scheduler **tries to avoid** placing Pods here, but will if no other node fits. |
| `NoExecute` | New Pods will not be scheduled AND existing Pods **are evicted** (unless they have `tolerationSeconds`). |

---

## Managing Taints on Nodes

```bash
# Add a taint
kubectl taint node <node-name> <key>=<value>:<effect>

# Examples
kubectl taint node node-1 dedicated=gpu:NoSchedule
kubectl taint node node-2 spot=true:NoSchedule
kubectl taint node node-3 env=production:NoSchedule

# Remove a taint (append -)
kubectl taint node node-1 dedicated=gpu:NoSchedule-

# View all node taints
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints

# Describe a node to see its taints
kubectl describe node <node-name> | grep -A5 Taints
```

---

## Toleration Syntax

```yaml
spec:
  tolerations:
    # Match key + value + effect exactly
    - key: "dedicated"
      operator: "Equal"
      value: "gpu"
      effect: "NoSchedule"

    # Match key + effect, any value (operator: Exists)
    - key: "spot"
      operator: "Exists"
      effect: "NoSchedule"

    # Match key only, any effect
    - key: "dedicated"
      operator: "Exists"

    # Match ALL taints (use carefully — e.g. DaemonSets)
    - operator: "Exists"
```

### Operators

| Operator | Behavior |
|---|---|
| `Equal` | Key, value, and effect must all match |
| `Exists` | Key (and optionally effect) must match; value is ignored |

---

## tolerationSeconds (for NoExecute)

When a `NoExecute` taint is added to a node, Pods are evicted immediately — unless they have a toleration with `tolerationSeconds`, which delays the eviction.

```yaml
tolerations:
  - key: "node.kubernetes.io/not-ready"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 300    # Pod stays on node for 5 min after it becomes not-ready
```

Kubernetes automatically adds these two tolerations to every Pod (default 300s):
```yaml
- key: node.kubernetes.io/not-ready
  effect: NoExecute
  tolerationSeconds: 300
- key: node.kubernetes.io/unreachable
  effect: NoExecute
  tolerationSeconds: 300
```

This means Pods survive brief node blips (network partition, kubelet restart) without being immediately evicted.

---

## Common Use Cases

### Dedicated GPU node group

Only GPU workloads run on GPU nodes — regular workloads are not accidentally scheduled there (wasting expensive GPU instances).

```bash
# Taint all GPU nodes
kubectl taint node gpu-node-1 nvidia.com/gpu=true:NoSchedule
kubectl taint node gpu-node-2 nvidia.com/gpu=true:NoSchedule
```

```yaml
# Only ML Pods tolerate the taint — and are attracted via nodeAffinity
spec:
  tolerations:
    - key: "nvidia.com/gpu"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: nvidia.com/gpu
                operator: Exists
  resources:
    limits:
      nvidia.com/gpu: "1"
```

### Spot instance node group

Batch / non-critical workloads run on Spot to save cost; production services are protected.

```bash
kubectl taint node spot-node-1 spot=true:NoSchedule
kubectl taint node spot-node-2 spot=true:NoSchedule
```

```yaml
# Only batch Pods tolerate spot
spec:
  tolerations:
    - key: "spot"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          preference:
            matchExpressions:
              - key: eks.amazonaws.com/capacityType
                operator: In
                values:
                  - SPOT
```

### Environment isolation (production vs staging on same cluster)

```bash
kubectl taint node prod-node-1 env=production:NoSchedule
kubectl taint node prod-node-2 env=production:NoSchedule
```

```yaml
# Production Pods only
spec:
  tolerations:
    - key: "env"
      operator: "Equal"
      value: "production"
      effect: "NoSchedule"
  nodeSelector:
    env: production
```

### Drain a node for maintenance

`kubectl drain` applies a `NoExecute` taint under the hood and evicts all Pods.

```bash
# Cordon — prevent new Pods from scheduling
kubectl cordon <node-name>

# Drain — evict existing Pods gracefully (respects PDBs and terminationGracePeriod)
kubectl drain <node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60

# After maintenance — uncordon to allow scheduling again
kubectl uncordon <node-name>
```

### System / infrastructure DaemonSets

DaemonSets like Fluentd, Prometheus Node Exporter, and CNI plugins need to run on **every** node — including tainted ones. They tolerate all taints:

```yaml
tolerations:
  - operator: "Exists"    # tolerate every taint on every node
```

---

## Built-in Node Taints (Kubernetes-managed)

Kubernetes automatically applies these taints to nodes in certain states:

| Taint | Applied When |
|---|---|
| `node.kubernetes.io/not-ready` | Node health check failing |
| `node.kubernetes.io/unreachable` | Node unreachable from control plane |
| `node.kubernetes.io/memory-pressure` | Node reports memory pressure |
| `node.kubernetes.io/disk-pressure` | Node reports disk pressure |
| `node.kubernetes.io/pid-pressure` | Node reports PID pressure |
| `node.kubernetes.io/unschedulable` | Node is cordoned |
| `node.kubernetes.io/network-unavailable` | Node network not configured |
| `node.cloudprovider.kubernetes.io/uninitialized` | Cloud node not yet initialized |

All built-in taints use `NoSchedule` or `NoExecute`. The default Pod tolerations (300s) handle brief `not-ready` and `unreachable` conditions.

---

## Taints + Tolerations vs Node Affinity

These two mechanisms are complementary and often used together:

| Mechanism | Direction | Effect |
|---|---|---|
| Taint | Node → repels Pods | Keeps unwanted Pods **off** a node |
| Toleration | Pod → accepts taint | Allows a Pod to **land on** a tainted node |
| nodeAffinity | Pod → attracts to node | **Pulls** a Pod toward specific nodes |

**Taint + toleration alone** is not enough to guarantee a Pod lands on the intended node — other Pods with the same toleration could also land there. Add `nodeAffinity` to both allow **and** attract.

```yaml
# Full dedicated-node pattern
spec:
  # Tolerate the node's taint
  tolerations:
    - key: "dedicated"
      value: "ml-workload"
      effect: "NoSchedule"
  # Also require the node
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: dedicated
                operator: In
                values:
                  - ml-workload
```
