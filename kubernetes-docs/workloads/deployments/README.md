# Deployments — Rolling Updates & Rollback

A Deployment manages a ReplicaSet, which manages Pods. It adds declarative rolling updates, rollbacks, and scaling on top of raw ReplicaSets.

---

## How It Works

```
Deployment
└── ReplicaSet (current)        ← manages desired Pods
└── ReplicaSet (previous)       ← kept for rollback
└── ReplicaSet (older, scaled to 0)
```

When you update a Deployment (new image, env var, etc.), Kubernetes creates a **new ReplicaSet** and gradually shifts traffic by scaling up new Pods and scaling down old ones.

---

## Deployment Spec

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
  labels:
    app: my-app
spec:
  replicas: 3

  selector:
    matchLabels:
      app: my-app         # must match template labels — immutable after creation

  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: my-registry/my-app:1.2.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # extra Pods above desired count during update
      maxUnavailable: 0     # Pods that can be unavailable during update

  revisionHistoryLimit: 5   # number of old ReplicaSets to keep for rollback
  progressDeadlineSeconds: 600
```

---

## Update Strategies

### RollingUpdate (default)

Gradually replaces Pods. Zero downtime when configured correctly.

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1         # can be absolute (1) or percent (25%)
    maxUnavailable: 0   # 0 = never reduce below desired count during update
```

**With `maxSurge: 1, maxUnavailable: 0` on 3 replicas:**
```
Start:   [v1] [v1] [v1]
Step 1:  [v1] [v1] [v1] [v2]   ← surge to 4
Step 2:  [v1] [v1] [v2]        ← remove one v1
Step 3:  [v1] [v1] [v2] [v2]   ← surge again
Step 4:  [v1] [v2] [v2]
...
End:     [v2] [v2] [v2]
```

### Recreate

Terminates all existing Pods before creating new ones. Causes downtime.

```yaml
strategy:
  type: Recreate
```

Use for: workloads that cannot have two versions running simultaneously (DB schema migrations requiring single writer).

---

## Triggering a Rollout

```bash
# Update image
kubectl set image deployment/my-app app=my-registry/my-app:1.3.0 -n production

# Edit directly
kubectl edit deployment/my-app -n production

# Apply updated YAML
kubectl apply -f deployment.yaml

# Force rollout (no spec change, e.g. to pick up updated ConfigMap)
kubectl rollout restart deployment/my-app -n production
```

---

## Monitoring Rollouts

```bash
# Watch rollout progress
kubectl rollout status deployment/my-app -n production

# View rollout history
kubectl rollout history deployment/my-app -n production

# View specific revision
kubectl rollout history deployment/my-app -n production --revision=3
```

---

## Rollback

```bash
# Undo to previous version
kubectl rollout undo deployment/my-app -n production

# Undo to a specific revision
kubectl rollout undo deployment/my-app -n production --to-revision=2
```

> Rollback creates a new rollout — it does not revert history. The previous version becomes the current revision.

### Annotating Releases (best practice)

```bash
kubectl annotate deployment/my-app \
  kubernetes.io/change-cause="v1.3.0 — add retry logic for downstream API" \
  -n production
```

This message appears in `kubectl rollout history` output.

---

## Pausing and Resuming

Useful when making multiple changes and wanting to apply them all at once.

```bash
# Pause the rollout
kubectl rollout pause deployment/my-app -n production

# Make multiple changes
kubectl set image deployment/my-app app=my-app:1.3.0 -n production
kubectl set env deployment/my-app LOG_LEVEL=debug -n production

# Resume — applies all changes in one rollout
kubectl rollout resume deployment/my-app -n production
```

---

## progressDeadlineSeconds

If a rollout does not make progress within this window (default: 600s), the Deployment is marked as failed.

```bash
kubectl describe deployment/my-app -n production
# Look for: "Progressing" condition with reason "ProgressDeadlineExceeded"
```

---

## Common Issues

| Symptom | Cause | Fix |
|---|---|---|
| Rollout stuck at X/N Pods | New Pods failing readiness probe | Check logs and probe config |
| Old Pods not terminating | `maxUnavailable: 0` + new Pods not ready | Fix new Pod issue first |
| `ProgressDeadlineExceeded` | Rollout took too long | Check events, increase deadline or fix app |
| Rollback did not help | Issue is in the cluster, not the image | Check node pressure, PVCs, ConfigMaps |

---

## Examples

- [nginx-deployment.yaml](examples/nginx-deployment.yaml)

> For scaling configuration (HPA, VPA), see [scaling.md](scaling.md).
