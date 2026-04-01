# Resource Quotas — Namespace-Level Limits

ResourceQuota limits the **total amount of resources** that can be consumed in a namespace. It prevents one team or workload from exhausting cluster capacity.

---

## ResourceQuota Spec

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    # Compute resources
    requests.cpu: "20"          # total CPU requests across all Pods
    requests.memory: "40Gi"     # total memory requests
    limits.cpu: "40"            # total CPU limits
    limits.memory: "80Gi"

    # Object counts
    pods: "100"
    services: "20"
    secrets: "50"
    configmaps: "50"
    persistentvolumeclaims: "30"

    # Service types
    services.loadbalancers: "3"
    services.nodeports: "0"     # disallow NodePort Services
```

---

## Storage Quotas

```yaml
spec:
  hard:
    requests.storage: "500Gi"         # total PVC storage requests
    persistentvolumeclaims: "30"      # number of PVCs

    # Per StorageClass quotas
    gp3.storageclass.storage.k8s.io/requests.storage: "300Gi"
    gp3.storageclass.storage.k8s.io/persistentvolumeclaims: "20"
    efs-sc.storageclass.storage.k8s.io/requests.storage: "200Gi"
```

---

## LimitRange (per-Pod defaults)

LimitRange works alongside ResourceQuota — it sets **defaults and bounds per container**, catching Pods deployed without resource definitions.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: production-limits
  namespace: production
spec:
  limits:
    - type: Container
      # Defaults applied when not specified
      default:
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      # Allowed range
      max:
        cpu: "4"
        memory: "8Gi"
      min:
        cpu: "50m"
        memory: "64Mi"

    - type: Pod
      max:
        cpu: "8"
        memory: "16Gi"

    - type: PersistentVolumeClaim
      max:
        storage: "100Gi"
      min:
        storage: "1Gi"
```

---

## How They Work Together

```
ResourceQuota  →  namespace-level ceiling
LimitRange     →  per-container defaults and bounds

Example flow:
  1. Pod submitted with no resource requests
  2. LimitRange injects default: cpu=100m, memory=128Mi
  3. ResourceQuota checks: does remaining quota allow this?
  4. If yes → Pod admitted; quota used -= 100m CPU, 128Mi memory
  5. If no  → Pod rejected with "exceeded quota" error
```

---

## Scopes

Apply a quota to a specific subset of Pods using `scopes`.

```yaml
spec:
  hard:
    pods: "50"
    requests.cpu: "20"
    requests.memory: "40Gi"
  scopes:
    - NotTerminating    # only apply to non-Job Pods
```

| Scope | Description |
|---|---|
| `Terminating` | Pods with `activeDeadlineSeconds` set (Jobs) |
| `NotTerminating` | Pods without `activeDeadlineSeconds` |
| `BestEffort` | Pods with no requests/limits |
| `NotBestEffort` | Pods with at least one request/limit |
| `PriorityClass` | Pods with a specific PriorityClass |

---

## Priority Classes

PriorityClasses allow critical workloads to be protected from eviction and preempt lower-priority workloads.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "Critical production services"

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 100
globalDefault: false
description: "Batch jobs and background tasks"
```

```yaml
# Assign to a Pod
spec:
  priorityClassName: high-priority
```

### Built-in Priority Classes

| Name | Value | Use Case |
|---|---|---|
| `system-cluster-critical` | 2,000,000,000 | Core cluster components (CoreDNS, kube-proxy) |
| `system-node-critical` | 2,000,001,000 | Node-level critical (kubelet, containerd) |

---

## Commands

```bash
# Check quota usage in a namespace
kubectl describe resourcequota -n production
# Shows: hard limits vs currently used

# Example output:
# Name: production-quota
# Resource         Used    Hard
# --------         ----    ----
# pods             24      100
# requests.cpu     4800m   20
# requests.memory  9Gi     40Gi

# Check LimitRange in a namespace
kubectl describe limitrange -n production

# Check all quotas across namespaces
kubectl get resourcequota --all-namespaces

# Check PriorityClasses
kubectl get priorityclass
```

---

## Multi-Tenant Namespace Strategy

For clusters shared by multiple teams:

```
cluster
├── namespace: team-a (quota: 10 CPU, 20Gi memory)
├── namespace: team-b (quota: 10 CPU, 20Gi memory)
└── namespace: team-c (quota: 5 CPU, 10Gi memory)
```

Each namespace gets:
1. `ResourceQuota` — total resource ceiling
2. `LimitRange` — per-Pod defaults and bounds
3. `NetworkPolicy` — isolate traffic between namespaces
4. Dedicated `ServiceAccount` + `RoleBinding` for CI/CD

```bash
# Apply standard namespace setup
kubectl apply -f resource-quota.yaml -n team-a
kubectl apply -f limit-range.yaml -n team-a
kubectl apply -f network-policy.yaml -n team-a
```
