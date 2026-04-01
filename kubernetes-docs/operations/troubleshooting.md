# Troubleshooting — Common Failure Patterns & Debug Commands

---

## Diagnostic Workflow

```
1. kubectl get pod → identify phase and restarts
2. kubectl describe pod → read Events section
3. kubectl logs → read application output
4. kubectl exec / kubectl debug → inspect container
5. kubectl get events → cluster-wide timeline
6. kubectl top → resource usage
```

---

## Pod States and Causes

### Pending (stuck)

Pod accepted by cluster but not running on any node.

| Cause | Check | Fix |
|---|---|---|
| Insufficient CPU/memory | `kubectl describe pod` → Events: `Insufficient cpu` | Add nodes, reduce requests, or scale down other workloads |
| No node matches nodeSelector/affinity | `kubectl describe pod` → `FailedScheduling` | Check node labels: `kubectl get nodes --show-labels` |
| All nodes tainted, no toleration | `kubectl describe pod` → `node(s) had taints` | Add toleration or untaint a node |
| PVC not bound | `kubectl get pvc` → `Pending` | See PVC troubleshooting below |
| Too many Pods on all nodes | `kubectl describe pod` → `Too many pods` | Scale node group |

```bash
kubectl describe pod <pod-name> -n <namespace>
# Look for "Events:" at the bottom — this is the most useful section
```

### CrashLoopBackOff

Container is crashing repeatedly; Kubernetes keeps restarting with back-off.

```bash
# Read current container logs
kubectl logs <pod-name> -n <namespace>

# Read logs from the previous (crashed) container
kubectl logs <pod-name> -n <namespace> --previous

# Get exit code and reason
kubectl get pod <pod-name> -n <namespace> \
  -o jsonpath='{.status.containerStatuses[0].lastState.terminated}'
```

| Cause | Check |
|---|---|
| App startup crash | Check logs for error on exit |
| OOM kill | `reason: OOMKilled` in container status |
| Liveness probe fails too early | Increase `initialDelaySeconds` |
| Missing env var or secret | App logs show missing config |
| Wrong image CMD/entrypoint | Image doesn't exist or command wrong |

### ImagePullBackOff / ErrImagePull

```bash
kubectl describe pod <pod-name> -n <namespace>
# Events: "Failed to pull image ... 404 Not Found"
```

| Cause | Fix |
|---|---|
| Wrong image name or tag | Verify image exists in registry |
| Private registry, no pull secret | Add `imagePullSecrets` to Pod spec |
| Pull secret in wrong namespace | Create secret in Pod's namespace |
| Rate limit (Docker Hub) | Use authenticated pulls or mirror |

```bash
# Verify image pull secret
kubectl get secret <secret-name> -n <namespace> -o yaml

# Test pull manually from node
ssh <node-ip>
crictl pull <image>
```

### OOMKilled

Container exceeded its memory limit and was killed by the kernel.

```bash
kubectl get pod <pod-name> -n <namespace> \
  -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
# Output: OOMKilled
```

**Fix options:**
1. Increase memory limit: edit `resources.limits.memory`
2. Fix memory leak in application
3. Use VPA in `Off` mode to observe actual usage before setting limits

### Evicted

Node ran out of memory or disk space and evicted lower-priority Pods.

```bash
kubectl get pods -n <namespace> | grep Evicted

# Clean up evicted Pods
kubectl get pods -n <namespace> | grep Evicted | \
  awk '{print $1}' | xargs kubectl delete pod -n <namespace>
```

**Prevent:** Set `requests == limits` (Guaranteed QoS) for critical workloads.

---

## Node Issues

```bash
# Check node status
kubectl get nodes

# Describe a NotReady node
kubectl describe node <node-name>
# Look for: Conditions, Events, Allocated resources

# Check node resource usage
kubectl top nodes

# Check kubelet logs on the node (EKS)
journalctl -u kubelet -n 100

# Check node taints
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

### Node NotReady

| Cause | Check |
|---|---|
| kubelet crashed | `journalctl -u kubelet` on the node |
| Disk pressure | `kubectl describe node` → `DiskPressure: True` |
| Memory pressure | `kubectl describe node` → `MemoryPressure: True` |
| Network issue | Node unreachable from control plane |

---

## Service / Networking Issues

```bash
# Check if Service has endpoints (Pods matching selector)
kubectl get endpoints <service-name> -n <namespace>
# If ADDRESS column is empty → no Pods match the selector

# Check selector matches Pod labels
kubectl get svc <service-name> -n <namespace> -o yaml | grep selector
kubectl get pods -n <namespace> --show-labels | grep <label>

# Test DNS from inside a Pod
kubectl exec -it <pod-name> -n <namespace> -- \
  nslookup <service-name>.<namespace>.svc.cluster.local

# Test connectivity
kubectl exec -it <pod-name> -n <namespace> -- \
  curl -v http://<service-name>.<namespace>.svc.cluster.local/healthz
```

---

## PVC / Storage Issues

```bash
# Check PVC status
kubectl get pvc -n <namespace>
kubectl describe pvc <pvc-name> -n <namespace>

# PVC stuck in Pending
# → Check StorageClass exists
kubectl get storageclass
# → Check provisioner logs
kubectl logs -n kube-system -l app=ebs-csi-controller

# Pod stuck due to volume attach failure
kubectl describe pod <pod-name> -n <namespace>
# Events: "AttachVolume.Attach failed"
# → EBS volume may be in wrong AZ — check node and PV AZ
kubectl get node <node-name> -o jsonpath='{.metadata.labels.topology\.kubernetes\.io/zone}'
kubectl get pv <pv-name> -o jsonpath='{.spec.nodeAffinity}'
```

---

## Resource / Quota Issues

```bash
# Check ResourceQuota usage
kubectl describe resourcequota -n <namespace>

# Check LimitRange
kubectl describe limitrange -n <namespace>

# Find Pods with no resource requests
kubectl get pods -n <namespace> -o json | \
  jq '.items[] | select(.spec.containers[].resources.requests == null) | .metadata.name'
```

---

## Debug Commands Reference

```bash
# === Pod inspection ===
kubectl get pod <pod> -n <ns> -o yaml
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns>
kubectl logs <pod> -n <ns> --previous
kubectl logs <pod> -n <ns> -c <container>
kubectl logs <pod> -n <ns> -f --tail=100

# === Shell access ===
kubectl exec -it <pod> -n <ns> -- /bin/sh
kubectl exec -it <pod> -n <ns> -- /bin/bash

# === Debug distroless / no-shell images ===
kubectl debug -it <pod> --image=busybox --target=<container> -n <ns>

# === Copy files ===
kubectl cp <pod>:/path/to/file ./local-file -n <ns>

# === Events ===
kubectl get events -n <ns> --sort-by='.lastTimestamp'
kubectl get events -n <ns> --field-selector involvedObject.name=<pod-name>

# === Resource usage ===
kubectl top pod -n <ns>
kubectl top pod -n <ns> --containers
kubectl top node

# === Force delete stuck Pod ===
kubectl delete pod <pod> -n <ns> --grace-period=0 --force
```

---

## Useful One-Liners

```bash
# All non-running Pods across cluster
kubectl get pods --all-namespaces | grep -v Running | grep -v Completed

# Pods with high restart counts
kubectl get pods --all-namespaces \
  --sort-by='.status.containerStatuses[0].restartCount' | tail -20

# Events sorted by time for a namespace
kubectl get events -n production \
  --sort-by='.lastTimestamp' | tail -30

# Which node is each Pod on
kubectl get pods -n production -o wide

# Resource usage by namespace
kubectl top pods --all-namespaces --sort-by=memory | head -20
```
