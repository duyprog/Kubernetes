# Resource Management — Requests, Limits & QoS

Defining resource requests and limits is **required** for production workloads. Without them, Pods compete for resources unpredictably and get evicted during node pressure.

---

## Requests vs Limits

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

| Field | Role | Enforced By |
|---|---|---|
| `requests` | Minimum guaranteed — used by scheduler to find a node with enough capacity | kube-scheduler |
| `limits` | Maximum allowed — enforced at runtime | kubelet / cgroups |

### CPU Behavior

- `1 CPU = 1000m` (millicores)
- **Throttled** when exceeding CPU limit — the process slows down, does not get killed
- `250m` = 0.25 of one CPU core

### Memory Behavior

- **OOM-killed** when exceeding memory limit — container is terminated and restarted per `restartPolicy`
- `Mi` = Mebibytes (1 MiB = 1,048,576 bytes), `Gi` = Gibibytes

> **Key difference:** CPU is compressible (throttled), memory is not (killed).

---

## Quality of Service (QoS) Classes

Kubernetes assigns a QoS class to every Pod based on how requests and limits are configured. This determines eviction priority under node memory pressure.

### Guaranteed

`requests == limits` for **every** container in the Pod (both CPU and memory).

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

- **Eviction priority:** Last — only evicted when no Burstable or BestEffort Pods remain
- Use for: production critical workloads, databases

### Burstable

At least one container has `requests != limits`, or only one of the two is defined.

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "1"
    memory: "1Gi"
```

- **Eviction priority:** Medium
- Use for: most production workloads that benefit from burst capacity

### BestEffort

No `requests` or `limits` defined on **any** container.

```yaml
# No resources field at all
```

- **Eviction priority:** Highest — first to be evicted
- Never use for production workloads

---

## Sizing Guidelines

### Determining Requests

1. Run without limits in staging, observe actual usage via `kubectl top pod`
2. Set requests at P50 usage (typical load)
3. Set limits at P99 or 2× requests

```bash
kubectl top pod -n <namespace>
kubectl top pod -n <namespace> --sort-by=memory
```

### Common Starting Points

| Workload Type | CPU Request | Memory Request |
|---|---|---|
| Lightweight API | 50–100m | 64–128Mi |
| Typical web service | 100–250m | 128–256Mi |
| JVM application | 500m–1 | 512Mi–1Gi |
| Data processing | 500m–2 | 512Mi–2Gi |

---

## LimitRange

Set default requests/limits for Pods in a namespace — catches Pods deployed without resource definitions.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "4"
        memory: "4Gi"
      min:
        cpu: "50m"
        memory: "64Mi"
```

---

## ResourceQuota

Limit total resource consumption across a namespace.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: namespace-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
    pods: "50"
```

> See [resource-quotas.md](../../operations/resource-quotas.md) for full details.

---

## Vertical Pod Autoscaler (VPA)

VPA automatically adjusts requests based on observed usage. Useful when correct sizing is unknown.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Off"    # Off = recommend only; Auto = apply and restart
  resourcePolicy:
    containerPolicies:
      - containerName: app
        minAllowed:
          cpu: "50m"
          memory: "64Mi"
        maxAllowed:
          cpu: "2"
          memory: "2Gi"
```

Start with `updateMode: "Off"` to review recommendations before enabling auto-updates.

```bash
kubectl describe vpa my-app-vpa -n <namespace>
# Look for "Recommendation:" section
```

---

## Checklist

- [ ] Every container has `requests` and `limits` defined
- [ ] Production critical Pods use `Guaranteed` QoS (`requests == limits`)
- [ ] `LimitRange` applied to each namespace to catch un-resourced Pods
- [ ] `ResourceQuota` applied to non-system namespaces
- [ ] `kubectl top pod` checked to validate sizing after deployment
