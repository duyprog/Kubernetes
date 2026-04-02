# VPA — Vertical Pod Autoscaler

VPA automatically adjusts **CPU and memory requests/limits** for containers based on observed usage. Where HPA adds more Pods, VPA makes each Pod the right size.

---

## How VPA Works

```
VPA components:
  ┌─────────────────┐   ┌──────────────────┐   ┌─────────────────────┐
  │  Recommender    │   │  Updater          │   │  Admission Plugin   │
  │                 │   │                  │   │                     │
  │ Watches metrics │   │ Evicts Pods that  │   │ Injects recommended │
  │ Builds history  │   │ are out of range  │   │ resources at Pod    │
  │ Produces recs   │   │ (in Auto mode)    │   │ creation time       │
  └────────┬────────┘   └──────────────────┘   └─────────────────────┘
           │
           ▼
     VPA object (.status.recommendation)
           │
           ▼
     Updater reads → evicts Pod → Admission Plugin injects new values → Pod restarts with right sizing
```

**Key point:** VPA changes take effect by **restarting Pods** (evict + recreate). This causes brief downtime per Pod unless `PodDisruptionBudget` is set and multiple replicas exist.

---

## VPA Spec

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
    updateMode: "Off"        # Off | Initial | Recreate | Auto

  resourcePolicy:
    containerPolicies:
      - containerName: app   # or "*" for all containers
        minAllowed:
          cpu: "50m"
          memory: "64Mi"
        maxAllowed:
          cpu: "4"
          memory: "4Gi"
        controlledResources: ["cpu", "memory"]
        controlledValues: RequestsAndLimits   # or RequestsOnly
```

---

## Update Modes

| Mode | What Happens | Use When |
|---|---|---|
| `Off` | Recommendations calculated but **never applied** | Starting point — review before committing |
| `Initial` | Applied at **Pod creation** only — no evictions | Safe for stateful apps; won't disrupt running Pods |
| `Recreate` | Evicts Pods out of range and recreates with new values | Controlled updates with PDB protection |
| `Auto` | Same as Recreate + may apply in-place when supported | Fully automated right-sizing |

> **Start with `Off`.** Let VPA collect data for 24–72 hours, review the recommendations, then switch to `Initial` or `Auto`.

---

## Reading Recommendations

```bash
kubectl describe vpa my-app-vpa -n production
```

Example output:

```
Recommendation:
  Container Recommendations:
    Container Name: app
    Lower Bound:
      Cpu:     50m
      Memory:  128Mi
    Target:              ← use this for requests
      Cpu:     250m
      Memory:  312Mi
    Uncapped Target:     ← target ignoring min/maxAllowed
      Cpu:     250m
      Memory:  312Mi
    Upper Bound:         ← use this for limits
      Cpu:     500m
      Memory:  600Mi
```

| Field | Meaning |
|---|---|
| `Target` | Recommended `requests` value — set this |
| `Lower Bound` | Minimum safe value — going below risks OOM/throttle |
| `Upper Bound` | Recommended `limits` ceiling |
| `Uncapped Target` | Target without `minAllowed`/`maxAllowed` constraints |

---

## controlledValues

```yaml
controlledValues: RequestsAndLimits   # adjusts both requests AND limits proportionally
controlledValues: RequestsOnly        # adjusts requests only; limits unchanged
```

Use `RequestsOnly` when you want to keep limits fixed (e.g., Guaranteed QoS where requests == limits).

---

## Workflow: Right-Sizing with VPA

```
1. Deploy VPA in Off mode
        ↓
2. Wait 24–72 hours of production traffic
        ↓
3. Review recommendations:
   kubectl describe vpa -n production
        ↓
4. Apply Target values to Deployment resources.requests
   Apply Upper Bound values to resources.limits
        ↓
5. Optional: switch VPA to Initial or Auto for ongoing adjustments
        ↓
6. Monitor: kubectl top pod, check for OOMKills and throttling
```

---

## VPA + HPA Compatibility

| HPA metric | VPA mode | Compatible? |
|---|---|---|
| CPU utilization | Auto | **No** — VPA changes requests, HPA recalculates utilization → feedback loop |
| CPU utilization | Off / Initial | **Yes** — VPA recommends, you apply manually |
| Custom metrics (req/s, queue depth) | Auto | **Yes** — different scaling axes |
| Memory | Auto | **Risky** — similar conflict risk as CPU |

**Recommended pattern:**
```
HPA (CPU utilization)  +  VPA (Off mode for recommendations only)
```
or
```
HPA (custom metrics like req/s)  +  VPA (Auto for right-sizing)
```

---

## Limitations

- VPA requires Pod **restart** to apply changes — not suitable for workloads that cannot tolerate restarts
- **Minimum 2 replicas** recommended when using `Auto` or `Recreate` mode — paired with a PodDisruptionBudget
- VPA and HPA **cannot both target CPU** on the same workload (see above)
- VPA needs at least **8 days** of data for high-confidence recommendations (works with less, but accuracy improves over time)
- Not suitable for single-replica StatefulSets with critical state (Updater will evict the only Pod)

---

## Installation

VPA is not installed by default. Install from the official repo:

```bash
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh
```

Or via Helm:

```bash
helm repo add fairwinds-stable https://charts.fairwinds.com/stable
helm upgrade --install vpa fairwinds-stable/vpa \
  --namespace vpa \
  --create-namespace
```

Verify components are running:

```bash
kubectl get pods -n kube-system | grep vpa
# vpa-admission-controller-...   Running
# vpa-recommender-...            Running
# vpa-updater-...                Running
```

---

## Commands

```bash
# List all VPAs
kubectl get vpa -n production

# View recommendations
kubectl describe vpa my-app-vpa -n production

# Check if VPA admission controller is working
kubectl get mutatingwebhookconfiguration | grep vpa

# See what VPA would recommend without installing it (using goldilocks)
kubectl label namespace production goldilocks.fairwinds.com/enabled=true
```

### Goldilocks (VPA recommendation dashboard)

[Goldilocks](https://github.com/FairwindsOps/goldilocks) runs VPA in `Off` mode for every workload in a namespace and shows recommendations in a web UI.

```bash
helm upgrade --install goldilocks fairwinds-stable/goldilocks \
  --namespace goldilocks \
  --create-namespace
kubectl port-forward svc/goldilocks-dashboard 8080:80 -n goldilocks
```
