# Scaling — HPA, VPA & Manual Scaling

---

## Manual Scaling

```bash
# Scale immediately
kubectl scale deployment/my-app --replicas=5 -n production

# Scale via patch
kubectl patch deployment/my-app -p '{"spec":{"replicas":5}}' -n production
```

> For production, prefer HPA over manual scaling — manual changes are overwritten by HPA if one is configured.

---

## Horizontal Pod Autoscaler (HPA)

Automatically scales the number of **Pod replicas** based on observed metrics.

### How It Works

```
HPA controller (runs every 15s)
  → queries metrics (CPU, memory, custom)
  → calculates desired replicas:
      desiredReplicas = ceil(currentReplicas × (currentMetric / targetMetric))
  → updates Deployment/StatefulSet replicas
```

### CPU-Based HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app

  minReplicas: 2
  maxReplicas: 10

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70    # % of requests.cpu
```

> CPU requests **must** be defined on the target container — HPA cannot calculate utilization without a baseline.

### Memory-Based HPA

```yaml
metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: "400Mi"
```

> Memory-based scaling is less reliable because memory is not released automatically by most runtimes. Prefer CPU or custom metrics.

### Multi-Metric HPA

HPA evaluates **all** metrics and takes the **highest** desired replica count.

```yaml
metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: "400Mi"
```

### Custom Metrics HPA (e.g., requests per second)

Requires a custom metrics adapter (Prometheus Adapter, Datadog Cluster Agent, etc.).

```yaml
metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
```

### HPA Scaling Behavior

Control scale-up and scale-down speed to prevent flapping.

```yaml
spec:
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0      # react immediately to load
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15              # max double replicas every 15s
    scaleDown:
      stabilizationWindowSeconds: 300    # wait 5 min before scaling down
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60             # remove max 1 Pod per minute
```

### HPA Commands

```bash
# View current HPA status
kubectl get hpa -n production
kubectl describe hpa my-app-hpa -n production

# Watch HPA in real time
kubectl get hpa my-app-hpa -n production -w
```

---

## Vertical Pod Autoscaler (VPA)

Automatically adjusts **CPU and memory requests** for containers. Does not change replica count.

### VPA Update Modes

| Mode | Behavior |
|---|---|
| `Off` | Recommend only — no changes applied |
| `Initial` | Apply recommendations only at Pod creation |
| `Auto` | Apply recommendations and evict/restart Pods to apply changes |
| `Recreate` | Same as `Auto` but only evicts, never in-place update |

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Off"       # start with Off, review, then switch to Auto
  resourcePolicy:
    containerPolicies:
      - containerName: app
        minAllowed:
          cpu: "50m"
          memory: "64Mi"
        maxAllowed:
          cpu: "4"
          memory: "4Gi"
        controlledResources: ["cpu", "memory"]
```

```bash
# View VPA recommendations
kubectl describe vpa my-app-vpa -n production
```

> **Important:** Do not use HPA (CPU/memory based) and VPA (Auto) together on the same Deployment — they conflict. Use HPA for replica scaling + VPA in `Off` mode for recommendations, or use HPA with custom metrics + VPA for right-sizing.

---

## Cluster Autoscaler

Scales the **number of nodes** in the cluster when Pods cannot be scheduled due to insufficient resources.

- Scale up: when Pods are `Pending` due to insufficient node capacity
- Scale down: when a node has been underutilized for 10+ minutes and its Pods can be rescheduled elsewhere

> See [cluster-autoscaler.md](../../operations/aws-eks/cluster-autoscaler.md) for EKS-specific setup.

---

## Scaling Decision Matrix

| Scenario | Solution |
|---|---|
| Traffic spikes requiring more Pods | HPA (CPU or custom metrics) |
| Containers sized too small/large | VPA in `Off` mode for recommendations |
| Cluster out of capacity | Cluster Autoscaler |
| Batch job with variable load | KEDA (event-driven autoscaling) |
| Stateful app needing replicas | HPA on StatefulSet (careful with shared state) |
