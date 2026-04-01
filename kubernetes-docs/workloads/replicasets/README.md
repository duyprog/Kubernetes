# ReplicaSets — Relationship with Deployments

---

## What Is a ReplicaSet?

A ReplicaSet ensures a specified number of Pod replicas are running at any given time. It continuously reconciles actual Pod count to the desired count.

**You almost never create ReplicaSets directly.** Deployments create and manage ReplicaSets for you — and add rollout, rollback, and update strategy on top.

---

## ReplicaSet vs Deployment

| Feature | ReplicaSet | Deployment |
|---|---|---|
| Maintains N replicas | Yes | Yes (via ReplicaSet) |
| Rolling updates | No | Yes |
| Rollback | No | Yes |
| Revision history | No | Yes |
| Pause/resume | No | Yes |
| **When to use** | Never directly | Always |

---

## How Deployments Use ReplicaSets

Each Deployment update creates a **new ReplicaSet**. Old ReplicaSets are kept (scaled to 0) for rollback purposes, up to `revisionHistoryLimit`.

```bash
# See the ReplicaSets owned by a Deployment
kubectl get replicasets -n <namespace> -l app=my-app

# Example output:
# NAME                   DESIRED   CURRENT   READY   AGE
# my-app-7d9f8b6c5f      3         3         3       2d    ← current
# my-app-5c8d7f4b3a      0         0         0       5d    ← previous (kept for rollback)
# my-app-2b7c6e3d1f      0         0         0       10d   ← older revision
```

---

## ReplicaSet Spec (for reference)

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-app-replicaset
  namespace: default
spec:
  replicas: 3

  selector:
    matchLabels:
      app: my-app
      version: v1      # must match template labels

  template:
    metadata:
      labels:
        app: my-app
        version: v1
    spec:
      containers:
        - name: app
          image: my-app:1.0
          ports:
            - containerPort: 8080
```

> The `selector` field is **immutable** after creation. If you need to change it, delete and recreate.

---

## Pod Adoption and Orphaning

ReplicaSets use label selectors to find their Pods. This means:

- If you **manually add matching labels** to an existing Pod, the ReplicaSet will adopt it (and may scale down other Pods to compensate)
- If you **remove matching labels** from a Pod, the ReplicaSet orphans it and creates a new Pod to replace it

This behavior is useful for debugging — remove the label from a crashing Pod to isolate it without the ReplicaSet replacing it immediately.

```bash
# Isolate a Pod for debugging (remove it from the ReplicaSet's selector)
kubectl label pod <pod-name> app=my-app-debug --overwrite -n <namespace>
```

---

## Standalone ReplicaSet Use Cases

The only scenario where a standalone ReplicaSet makes sense is when you **never need rolling updates** — for example, a fixed infrastructure component that is always fully replaced. Even then, a Deployment with `strategy: Recreate` is usually cleaner.

---

## Commands

```bash
# List ReplicaSets
kubectl get replicasets -n <namespace>

# Describe a ReplicaSet
kubectl describe replicaset <rs-name> -n <namespace>

# See which Deployment owns a ReplicaSet
kubectl get replicaset <rs-name> -n <namespace> \
  -o jsonpath='{.metadata.ownerReferences[0].name}'
```
