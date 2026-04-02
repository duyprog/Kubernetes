# HPA — Horizontal Pod Autoscaler

HPA automatically adjusts the number of Pod replicas in a Deployment, StatefulSet, or ReplicaSet based on observed metrics.

---

## How HPA Works

```
Every 15 seconds (default):

HPA controller
  → queries Metrics API (CPU/memory) or custom metrics adapter
  → calculates desired replicas:

      desiredReplicas = ceil( currentReplicas × (currentMetric / targetMetric) )

  → applies cooldown windows (scaleUp / scaleDown stabilization)
  → updates .spec.replicas on the target workload
```

**Example:**
- Current replicas: 3, CPU target: 50%, current CPU: 90%
- `ceil(3 × (90 / 50))` = `ceil(5.4)` = **6 replicas**

---

## Prerequisites

- `metrics-server` must be installed for CPU/memory HPA
- Container `resources.requests.cpu` must be defined — HPA calculates utilization as `current usage / request`
- For custom metrics: a metrics adapter (Prometheus Adapter, Datadog, etc.) must be installed

---

## CPU-Based HPA (most common)

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
  maxReplicas: 20

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70    # % of requests.cpu across all Pods
```

---

## Memory-Based HPA

```yaml
metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: "400Mi"   # average memory per Pod
```

> Memory scaling is less reliable — most runtimes don't release memory eagerly, so replicas won't scale down even after load drops. Prefer CPU or custom metrics for scale-down to work well.

---

## Multi-Metric HPA

HPA evaluates **all** metrics independently and takes the **highest** desired replica count from any of them.

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

  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1k"
```

---

## Custom Metrics HPA

Requires a custom metrics adapter deployed in the cluster.

### Prometheus Adapter (most common)

Exposes Prometheus queries as custom metrics via the `custom.metrics.k8s.io` API.

```yaml
# custom-metrics-adapter values.yaml
rules:
  - seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
    resources:
      overrides:
        namespace: {resource: "namespace"}
        pod: {resource: "pod"}
    name:
      matches: "^(.*)_total$"
      as: "${1}_per_second"
    metricsQuery: 'rate(<<.Series>>{<<.LabelMatchers>>}[2m])'
```

```yaml
# HPA using the custom metric
metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "500"    # 500 req/s per Pod
```

### External Metrics (metrics not attached to Pods)

For metrics from outside the cluster — SQS queue depth, Datadog metric, etc.

```yaml
metrics:
  - type: External
    external:
      metric:
        name: sqs_approximate_number_of_messages_visible
        selector:
          matchLabels:
            queue: my-queue
      target:
        type: AverageValue
        averageValue: "30"    # scale to keep ~30 messages per Pod
```

---

## Scaling Behavior (cooldowns and rate limits)

Without tuning, HPA can react too aggressively — scaling up and down rapidly ("flapping"). Use `behavior` to control the pace.

```yaml
spec:
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0      # react to spikes immediately
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60              # at most double replicas per minute
        - type: Pods
          value: 4
          periodSeconds: 60             # or add at most 4 Pods per minute
      selectPolicy: Max                  # use whichever policy allows more Pods

    scaleDown:
      stabilizationWindowSeconds: 300    # wait 5 min of sustained low load before scaling down
      policies:
        - type: Pods
          value: 1
          periodSeconds: 120            # remove at most 1 Pod every 2 minutes
      selectPolicy: Min                  # use whichever policy removes fewer Pods (conservative)
```

### stabilizationWindowSeconds

HPA looks back over this window and uses the **highest** desired replica count seen. Prevents scaling down on a brief traffic dip.

```
scaleDown stabilizationWindowSeconds: 300

Load drops at T=0 → HPA wants 2 replicas
At T=60  → still wants 2
At T=120 → still wants 2
At T=300 → 5 min of consistently wanting 2 → scale down to 2
```

---

## Target Types

| Type | Description | Example |
|---|---|---|
| `Utilization` | % of the container's `requests` value | CPU at 70% of request |
| `AverageValue` | Absolute average value per Pod | 400Mi memory per Pod |
| `Value` | Total value across all Pods | Only for External metrics |

---

## Checking HPA Status

```bash
# Current state — shows current metric vs target
kubectl get hpa -n production
kubectl get hpa my-app-hpa -n production

# Example output:
# NAME          REFERENCE         TARGETS         MINPODS   MAXPODS   REPLICAS
# my-app-hpa   Deployment/my-app  45%/70%         2         20        3

# Full details — events, conditions, last scale time
kubectl describe hpa my-app-hpa -n production

# Watch in real time
kubectl get hpa my-app-hpa -n production -w
```

### HPA Conditions

From `kubectl describe hpa`:

| Condition | Meaning |
|---|---|
| `AbleToScale: True` | HPA can scale the target |
| `ScalingActive: True` | At least one metric is valid and HPA is active |
| `ScalingLimited: True` | Desired replicas was clamped by min/maxReplicas or rate limit |

---

## Common Issues

| Symptom | Cause | Fix |
|---|---|---|
| `unknown` in TARGETS column | metrics-server not installed or CPU requests not set | Install metrics-server; set `resources.requests.cpu` |
| HPA never scales down | scaleDown stabilization window too long, or memory metric | Tune `stabilizationWindowSeconds`; avoid memory-only scaling |
| HPA flapping (rapid up/down) | No stabilization window, narrow target band | Add `scaleDown.stabilizationWindowSeconds` |
| Replicas stuck at max | Load genuinely exceeds capacity | Add nodes (Cluster Autoscaler) |
| HPA conflicts with manual scale | HPA overrides manual changes | Use `kubectl annotate hpa ... autoscaling.alpha.kubernetes.io/current-metrics-` to pause |

---

## HPA + Deployment Strategy

For zero-downtime scaling, pair HPA with proper readiness probes and PodDisruptionBudgets:

```yaml
# Ensure at least 2 Pods are always available during scale-down or rolling update
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: my-app
```
