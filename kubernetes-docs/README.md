# Kubernetes Documentation

A structured reference for DevOps engineers working with Kubernetes in production, with an AWS/EKS focus.

---

## Cluster Architecture

```
                        ┌─────────────────────────────────┐
                        │         Control Plane            │
                        │                                  │
                        │  kube-apiserver                  │
                        │  etcd (cluster state store)      │
                        │  kube-scheduler                  │
                        │  kube-controller-manager         │
                        │  cloud-controller-manager        │
                        └────────────────┬────────────────┘
                                         │
              ┌──────────────────────────┼──────────────────────────┐
              │                          │                          │
     ┌────────▼────────┐       ┌─────────▼───────┐       ┌─────────▼───────┐
     │     Node 1       │       │     Node 2       │       │     Node 3       │
     │                  │       │                  │       │                  │
     │  kubelet         │       │  kubelet         │       │  kubelet         │
     │  kube-proxy      │       │  kube-proxy      │       │  kube-proxy      │
     │  container rt    │       │  container rt    │       │  container rt    │
     │  ┌────┐ ┌────┐   │       │  ┌────┐ ┌────┐   │       │  ┌────┐ ┌────┐   │
     │  │Pod │ │Pod │   │       │  │Pod │ │Pod │   │       │  │Pod │ │Pod │   │
     │  └────┘ └────┘   │       │  └────┘ └────┘   │       │  └────┘ └────┘   │
     └──────────────────┘       └──────────────────┘       └──────────────────┘
```

### Control Plane Components

| Component | Role |
|---|---|
| `kube-apiserver` | Front door for all API requests — REST endpoint, auth, admission control |
| `etcd` | Distributed key-value store holding all cluster state |
| `kube-scheduler` | Watches for unscheduled Pods and assigns them to nodes |
| `kube-controller-manager` | Runs controllers (Deployment, ReplicaSet, Node, etc.) |
| `cloud-controller-manager` | Integrates with cloud provider APIs (AWS, GCP, Azure) |

### Node Components

| Component | Role |
|---|---|
| `kubelet` | Agent on each node — ensures containers in Pods are running |
| `kube-proxy` | Maintains network rules for Service routing |
| Container runtime | Runs containers (containerd, CRI-O) |

---

## Key Concepts

### Declarative Model

You describe the **desired state** in YAML. Kubernetes controllers continuously reconcile actual state to match desired state.

```
Desired State (YAML) → kube-apiserver → etcd
                                          ↓
                              Controllers reconcile
                                          ↓
                              Actual State (running cluster)
```

### Namespace

Logical isolation within a cluster. Resources are namespace-scoped (Pods, Services, Deployments) or cluster-scoped (Nodes, PersistentVolumes, ClusterRoles).

```bash
kubectl get pods -n <namespace>
kubectl get pods --all-namespaces
```

### Labels and Selectors

Labels are key-value pairs on objects. Selectors are how controllers find their managed objects.

```yaml
metadata:
  labels:
    app: my-service
    env: production
    version: "2.1"
```

### Resource Hierarchy

```
Cluster
└── Namespace
    ├── Deployment
    │   └── ReplicaSet
    │       └── Pod
    │           └── Container(s)
    ├── Service
    ├── ConfigMap / Secret
    └── PersistentVolumeClaim → PersistentVolume
```

---

## Documentation Index

### Workloads

| Resource | Description |
|---|---|
| [Pods](workloads/pods/README.md) | Lifecycle, phases, conditions |
| [Deployments](workloads/deployments/README.md) | Rolling updates, rollback strategy |
| [ReplicaSets](workloads/replicasets/README.md) | Relationship with Deployments |
| [DaemonSets](workloads/daemonsets/README.md) | Node-level agents |
| [StatefulSets](workloads/statefulsets/README.md) | Ordered deployment, stable identity |

### Configuration

| Resource | Description |
|---|---|
| [ConfigMaps](configuration/configmaps/README.md) | Non-sensitive configuration |
| [Secrets](configuration/secrets/README.md) | Sensitive data, encryption at rest |

### Storage

| Resource | Description |
|---|---|
| [Storage Overview](storage/README.md) | PV, PVC, StorageClass lifecycle |
| [PersistentVolumes](storage/persistent-volumes.md) | PV types, reclaim policies |
| [StorageClasses](storage/storage-classes.md) | Dynamic provisioning on AWS |

### Networking

| Resource | Description |
|---|---|
| [Services](networking/services.md) | ClusterIP, NodePort, LoadBalancer |
| [Ingress](networking/ingress.md) | Ingress controllers, TLS |
| [Network Policies](networking/network-policies.md) | Pod-level firewall rules |

### Operations

| Resource | Description |
|---|---|
| [Troubleshooting](operations/troubleshooting.md) | Common failure patterns, debug commands |
| [RBAC](operations/rbac.md) | Roles, ClusterRoles, ServiceAccounts |
| [Resource Quotas](operations/resource-quotas.md) | Namespace-level resource limits |
| [EKS Setup](operations/aws-eks/eks-setup.md) | Cluster provisioning on AWS |
| [IRSA](operations/aws-eks/iam-roles-serviceaccounts.md) | IAM Roles for Service Accounts |
| [Cluster Autoscaler](operations/aws-eks/cluster-autoscaler.md) | Node autoscaling on EKS |

---

## Quick Reference

```bash
# Context and namespace
kubectl config get-contexts
kubectl config use-context <context>
kubectl config set-context --current --namespace=<namespace>

# Resource inspection
kubectl get all -n <namespace>
kubectl describe <resource> <name> -n <namespace>
kubectl explain pod.spec.containers

# Apply / delete
kubectl apply -f <file>.yaml
kubectl delete -f <file>.yaml

# Rollout
kubectl rollout status deployment/<name> -n <namespace>
kubectl rollout undo deployment/<name> -n <namespace>
```
