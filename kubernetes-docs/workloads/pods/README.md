# Pods — Lifecycle, Phases & Conditions

A Pod is the smallest deployable unit in Kubernetes — it wraps one or more containers that share a network namespace and storage volumes.

> For spec structure and container configuration, see [pod-basics.md](pod-basics.md).
> For CPU/memory resources, see [resource-limits.md](resource-limits.md).

---

## Pod Phases

The `status.phase` field reflects the high-level state of the Pod.

| Phase | Description |
|---|---|
| `Pending` | Accepted by the cluster but not yet running — image pulling, waiting for scheduling, or insufficient resources |
| `Running` | At least one container is running, starting, or restarting |
| `Succeeded` | All containers exited with code `0` and will not restart |
| `Failed` | All containers terminated; at least one exited with non-zero code |
| `Unknown` | Pod state cannot be determined — usually the node is unreachable |

### Lifecycle Flow

```
[Submitted] → Pending → Running → Succeeded
                               → Failed
                      → (crash) → CrashLoopBackOff
```

> **CrashLoopBackOff** is not a phase — it's a container state indicating the container keeps crashing and Kubernetes is applying exponential back-off before restarting (10s → 20s → 40s → … up to 5 min).

---

## Pod Conditions

More granular than phases — found in `status.conditions`. Each condition has a `True`, `False`, or `Unknown` status.

| Condition | Meaning |
|---|---|
| `PodScheduled` | Pod has been assigned to a node |
| `ContainersReady` | All containers in the Pod are ready |
| `Initialized` | All init containers completed successfully |
| `Ready` | Pod can serve requests (readiness probe passing) |

```bash
kubectl describe pod <pod-name> -n <namespace>
# Look for "Conditions:" section in output
```

---

## Container States

Inside a running Pod, each container has its own state under `status.containerStatuses`.

| State | Description |
|---|---|
| `Waiting` | Not yet running — pulling image, waiting for init containers |
| `Running` | Executing normally |
| `Terminated` | Completed or crashed — includes exit code and reason |

```bash
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.status.containerStatuses[*].state}'
```

---

## Restart Policies

Set on the Pod spec at `spec.restartPolicy`. Applies to all containers in the Pod.

| Policy | Behavior | Use Case |
|---|---|---|
| `Always` | Restart on any exit (default) | Long-running services |
| `OnFailure` | Restart only on non-zero exit code | Batch jobs |
| `Never` | Never restart | One-shot tasks |

---

## Pod Termination Flow

Understanding this is critical for zero-downtime deployments.

```
1. kubectl delete pod  (or controller scales down)
2. Pod status → Terminating
3. SIGTERM sent to containers
4. terminationGracePeriodSeconds countdown starts (default: 30s)
5. preStop lifecycle hook executes (if defined)
6. If containers still running after grace period → SIGKILL
7. Pod removed from API server
```

**In parallel with step 3**, Kubernetes removes the Pod from Service endpoints. There is a race condition — the Pod may still receive traffic for a few seconds after SIGTERM. The `preStop` sleep pattern handles this:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 10"]
```

> Set `terminationGracePeriodSeconds` higher than your preStop hook duration + connection drain time.

---

## Init Containers

Run sequentially before app containers. Each must exit `0` before the next starts.

**Use cases:**
- Wait for a database or external service to be available
- Run database migrations
- Populate a shared volume with config or data
- Set file permissions

```yaml
initContainers:
  - name: wait-for-db
    image: busybox:1.36
    command: ['sh', '-c', 'until nc -z postgres 5432; do sleep 2; done']
  - name: run-migrations
    image: my-app:1.0
    command: ['./migrate', '--up']
```

If an init container fails, the Pod restarts from the **first** init container.

---

## Ephemeral Containers

Injected into a running Pod for debugging. Cannot be removed once added.

```bash
# Debug a distroless or minimal image with no shell
kubectl debug -it <pod-name> \
  --image=busybox \
  --target=<container-name> \
  -n <namespace>
```

---

## Pod Status Quick Checks

```bash
# Full status
kubectl get pod <pod-name> -n <namespace> -o yaml

# Just phase
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.status.phase}'

# All conditions
kubectl get pod <pod-name> -n <namespace> \
  -o jsonpath='{range .status.conditions[*]}{.type}: {.status}{"\n"}{end}'

# Container restart count
kubectl get pod <pod-name> -n <namespace> \
  -o jsonpath='{.status.containerStatuses[0].restartCount}'

# Previous container logs (after crash)
kubectl logs <pod-name> -n <namespace> --previous
```
