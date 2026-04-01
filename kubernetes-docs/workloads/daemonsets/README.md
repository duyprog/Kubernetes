# DaemonSets — Node-Level Agents

A DaemonSet ensures that **one Pod runs on every node** (or a subset of nodes) in the cluster. When a new node joins, the DaemonSet automatically schedules a Pod on it. When a node is removed, the Pod is garbage collected.

---

## Use Cases

| Use Case | Example |
|---|---|
| Log collection | Fluentd, Filebeat, Promtail |
| Node monitoring | Prometheus Node Exporter, Datadog Agent |
| Network plugins | Calico, Cilium, Flannel (CNI) |
| Storage drivers | AWS EBS CSI driver node plugin |
| Security agents | Falco, CrowdStrike Falcon |
| Cluster DNS | CoreDNS on each node (sometimes) |

---

## DaemonSet Spec

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-monitor
  namespace: monitoring
  labels:
    app: node-monitor
spec:
  selector:
    matchLabels:
      app: node-monitor

  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1       # update one node at a time

  template:
    metadata:
      labels:
        app: node-monitor
    spec:
      # Access node-level resources
      hostNetwork: false
      hostPID: false

      # Required: tolerate NoSchedule taints on system/control-plane nodes
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
        - key: node.kubernetes.io/not-ready
          operator: Exists
          effect: NoSchedule
        - key: node.kubernetes.io/unreachable
          operator: Exists
          effect: NoSchedule

      containers:
        - name: agent
          image: my-monitoring-agent:1.0
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"

          # Access node filesystem
          volumeMounts:
            - name: host-proc
              mountPath: /host/proc
              readOnly: true
            - name: host-sys
              mountPath: /host/sys
              readOnly: true

          # Expose node info to container
          env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName

      volumes:
        - name: host-proc
          hostPath:
            path: /proc
        - name: host-sys
          hostPath:
            path: /sys
```

---

## Tolerations for System Nodes

By default, DaemonSet Pods **do not** run on control plane nodes or tainted nodes. Add tolerations to extend coverage.

```yaml
tolerations:
  # Run on control-plane nodes
  - key: node-role.kubernetes.io/control-plane
    operator: Exists
    effect: NoSchedule

  # Run on nodes that are not yet ready
  - key: node.kubernetes.io/not-ready
    operator: Exists
    effect: NoSchedule

  # Run on unreachable nodes (agent should still attempt)
  - key: node.kubernetes.io/unreachable
    operator: Exists
    effect: NoSchedule

  # Run on dedicated node groups (e.g., spot instances)
  - key: "node-type"
    operator: "Equal"
    value: "spot"
    effect: "NoSchedule"
```

---

## Running on a Subset of Nodes

Use `nodeSelector` or `nodeAffinity` to limit which nodes get the DaemonSet Pod.

```yaml
# Only nodes with this label
spec:
  template:
    spec:
      nodeSelector:
        logging: "enabled"
```

```yaml
# Only nodes in specific AZs
spec:
  template:
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
```

---

## Update Strategy

| Strategy | Behavior |
|---|---|
| `RollingUpdate` | Updates Pods one node at a time (default) |
| `OnDelete` | New Pod spec only takes effect when you manually delete the old Pod |

```bash
# Trigger a DaemonSet rollout
kubectl rollout restart daemonset/node-monitor -n monitoring

# Watch rollout progress
kubectl rollout status daemonset/node-monitor -n monitoring
```

---

## DaemonSet vs Deployment

| | DaemonSet | Deployment |
|---|---|---|
| Placement | One Pod per node (or subset) | N replicas anywhere in cluster |
| Scaling | Determined by node count | Manually or via HPA |
| Use case | Per-node agents | Application workloads |

---

## Commands

```bash
# List DaemonSet Pods with their nodes
kubectl get pods -n monitoring -l app=node-monitor -o wide

# Check DaemonSet status (desired vs ready)
kubectl get daemonset node-monitor -n monitoring

# Describe for events and conditions
kubectl describe daemonset node-monitor -n monitoring
```

---

## Examples

- [fluentd-daemonset.yaml](examples/fluentd-daemonset.yaml)
