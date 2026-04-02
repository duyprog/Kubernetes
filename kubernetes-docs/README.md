# Kubernetes Documentation

A structured reference for DevOps engineers working with Kubernetes in production, with an AWS/EKS focus.

---

## Cluster Architecture

```
                        ┌──────────────────────────────────────┐
                        │            Control Plane             │
                        │                                      │
                        │  kube-apiserver                      │
                        │  etcd                                │
                        │  kube-scheduler                      │
                        │  kube-controller-manager             │
                        │  cloud-controller-manager            │
                        └──────────────────┬───────────────────┘
                                           │ watches / reconciles
              ┌────────────────────────────┼────────────────────────────┐
              │                            │                            │
     ┌────────▼──────────┐      ┌──────────▼──────────┐      ┌──────────▼──────────┐
     │       Node 1      │      │       Node 2        │      │       Node 3        │
     │                   │      │                     │      │                     │
     │  kubelet          │      │  kubelet            │      │  kubelet            │
     │  kube-proxy       │      │  kube-proxy         │      │  kube-proxy         │
     │  containerd       │      │  containerd         │      │  containerd         │
     │  ┌─────┐ ┌─────┐  │      │  ┌─────┐ ┌─────┐    │      │  ┌─────┐ ┌─────┐    │
     │  │ Pod │ │ Pod │  │      │  │ Pod │ │ Pod │    │      │  │ Pod │ │ Pod │    │
     │  └─────┘ └─────┘  │      │  └─────┘ └─────┘    │      │  └─────┘ └─────┘    │
     └───────────────────┘      └─────────────────────┘      └─────────────────────┘
```

---

## Control Plane Components

The control plane manages the cluster — it makes global decisions (scheduling, reconciliation) and detects/responds to cluster events. On EKS, AWS fully manages the control plane for you.

### kube-apiserver

The **front door** of the cluster. Every interaction — `kubectl`, controllers, kubelets — goes through the API server.

**What it does:**
- Exposes the Kubernetes REST API over HTTPS
- Authenticates and authorizes every request (via certificates, tokens, OIDC)
- Validates request payloads against API schemas
- Persists accepted state to etcd

**Request flow:**
```
kubectl apply -f pod.yaml
       ↓
kube-apiserver
  → Authentication   (who are you? — cert / bearer token / OIDC)
  → Authorization    (are you allowed? — RBAC check)
  → Admission        (mutate + validate — webhooks, LimitRange, PSA)
  → Persist to etcd
  → Notify watchers  (scheduler, controllers, kubelet)
```

**Admission controllers** (important ones):

| Controller | What it does |
|---|---|
| `LimitRanger` | Injects default requests/limits from LimitRange |
| `ResourceQuota` | Rejects objects that would exceed namespace quota |
| `PodSecurity` | Enforces Pod Security Admission (privileged/baseline/restricted) |
| `MutatingWebhook` | Calls external webhooks to mutate objects (e.g. Istio sidecar injection) |
| `ValidatingWebhook` | Calls external webhooks to validate objects (e.g. OPA/Gatekeeper) |

> On EKS: the API server endpoint can be public, private, or both. Restrict public access to specific CIDRs in production.

---

### etcd

A **distributed key-value store** that holds the entire cluster state — every object, every spec, every status. The API server is the only component that talks to etcd directly.

**What it stores:** every Kubernetes object (`/registry/pods/...`, `/registry/deployments/...`, etc.)

**Key properties:**
- Uses the **Raft consensus algorithm** — requires a quorum of `(n/2)+1` members to accept writes
- Typical production setup: **3 or 5 nodes** (tolerates 1 or 2 failures respectively)
- All data is strongly consistent — every read reflects the latest committed write

**Why it matters operationally:**
- etcd is the **single source of truth** — if etcd is lost without a backup, the cluster state is gone
- High write load (many objects, frequent updates) can cause etcd latency — monitor `etcd_disk_wal_fsync_duration_seconds`
- Secrets are stored here — enable **encryption at rest** (KMS on EKS)
- Default storage limit per object: **1.5 MB** (ConfigMaps, Secrets must stay under this)

> On EKS: AWS manages etcd. You do not have direct access to it.

---

### kube-scheduler

Watches for **unscheduled Pods** (Pods with no `nodeName`) and assigns them to a node.

**Two-phase process for each Pod:**

```
1. Filtering   — eliminate nodes that cannot run the Pod
   • Insufficient CPU / memory
   • Node has a taint the Pod doesn't tolerate
   • Node doesn't match nodeSelector / nodeAffinity
   • Volume zone mismatch
   • Pod affinity / anti-affinity violations
   • Node is unschedulable (cordoned)

2. Scoring     — rank remaining nodes, pick the highest score
   • LeastAllocated  — prefer nodes with most free resources
   • BalancedResource — balance CPU and memory usage
   • NodeAffinity     — prefer nodes matching preferred affinity
   • InterPodAffinity — prefer nodes near / away from other Pods
   • ImageLocality    — prefer nodes that already have the image cached
```

After picking a node, the scheduler writes `nodeName` to the Pod spec — the kubelet on that node then picks it up and starts the containers.

> The scheduler only **assigns** Pods — it does not start them. kubelet does that.

---

### kube-controller-manager

Runs a collection of **controllers** — each controller is a reconciliation loop that watches the cluster state and takes action to move it toward the desired state.

```
Watch desired state (spec) ──→ Compare to actual state (status)
                                        ↓
                              If they differ → take action
                                        ↓
                              Update status / create/delete objects
```

**Built-in controllers:**

| Controller | What it reconciles |
|---|---|
| Deployment controller | Creates / updates ReplicaSets to match Deployment spec |
| ReplicaSet controller | Creates / deletes Pods to match replica count |
| Node controller | Monitors node health; taints or evicts Pods on unhealthy nodes |
| Job controller | Creates Pods for Jobs; tracks completions |
| CronJob controller | Creates Jobs on schedule |
| Endpoints controller | Keeps Service Endpoints in sync with Pod IPs |
| ServiceAccount controller | Creates default ServiceAccounts in new namespaces |
| PersistentVolume controller | Binds PVCs to PVs |
| Namespace controller | Cleans up resources when a namespace is deleted |

All controllers run in a single process but as independent goroutines. They use **leader election** so only one replica of `kube-controller-manager` is active at a time (others are on standby).

---

### cloud-controller-manager

Integrates Kubernetes with the **cloud provider API**. Extracted from `kube-controller-manager` so cloud-specific logic doesn't live in the core.

**On AWS EKS, it handles:**

| Controller | What it does |
|---|---|
| Node controller | Registers new EC2 instances as Nodes; removes terminated instances |
| Route controller | Configures VPC routes for Pod CIDR blocks |
| Service controller | Creates / updates / deletes ELB/NLB when `type: LoadBalancer` Services change |

> On EKS, the AWS cloud provider is built in. The AWS Load Balancer Controller (a separate add-on) handles ALB/NLB creation more precisely than the built-in controller.

---

## Node Components

Every worker node runs these three components. They are responsible for actually **running Pods** and keeping them healthy.

### kubelet

The **primary node agent** — the only component that talks to the container runtime directly.

**What it does:**
- Watches the API server for Pods assigned to its node (`nodeName == this node`)
- Translates the Pod spec into container runtime instructions (via CRI)
- Manages the full Pod lifecycle: pull image → create → start → monitor → stop
- Runs liveness, readiness, and startup probes
- Mounts volumes (calls CSI driver), injects env vars and secrets
- Reports Pod and node status back to the API server
- Enforces resource limits via cgroups
- Evicts Pods when the node is under memory or disk pressure

**kubelet does NOT run in a container** — it runs directly on the OS as a systemd service. This is why it can manage everything else.

```bash
# Check kubelet status on a node (EKS — via SSM or node access)
systemctl status kubelet
journalctl -u kubelet -n 50
```

**CRI (Container Runtime Interface):** kubelet communicates with the container runtime via gRPC over a Unix socket. It doesn't know or care which runtime is used — containerd, CRI-O, or any CRI-compliant runtime.

---

### kube-proxy

Maintains **network rules on each node** to implement Service routing. It watches Services and Endpoints and translates them into iptables or IPVS rules.

**What it does:**
- When you create a `ClusterIP` Service, kube-proxy programs rules so that any packet destined for the ClusterIP gets DNAT'd to one of the backend Pod IPs
- Implements basic load balancing (round-robin) across Pod endpoints
- Handles `NodePort` Services by opening ports on the node

**Modes:**

| Mode | Mechanism | Notes |
|---|---|---|
| `iptables` | Linux iptables DNAT rules | Default on most clusters; O(n) rule scan |
| `ipvs` | Linux IPVS (IP Virtual Server) | Better performance at scale; O(1) lookup |
| `nftables` | Linux nftables | New in k8s 1.29, not yet default |

> On EKS with **Cilium** or **AWS VPC CNI with network policy**, kube-proxy can be replaced entirely by eBPF-based routing, which is faster and more observable.

```bash
# Check kube-proxy mode
kubectl get configmap kube-proxy-config -n kube-system -o yaml | grep mode
```

---

### Container Runtime

The software that **actually runs containers** on the node. Kubernetes uses the **CRI (Container Runtime Interface)** to talk to it — a standard gRPC API so any compliant runtime works.

**On EKS:** `containerd` (default since EKS 1.24; Docker was removed)

**How containerd works:**

```
kubelet
  → CRI gRPC call → containerd
                       → containerd-shim (one per container)
                           → runc (OCI runtime — creates the actual container)
                               → Linux namespaces + cgroups
```

**Key responsibilities:**
- Pull images from registries (ECR, Docker Hub, etc.)
- Unpack and store image layers
- Create container filesystem (overlay FS)
- Start/stop/delete containers
- Stream logs to node filesystem (`/var/log/containers/`)

```bash
# Inspect containers directly on a node (bypasses kubectl)
crictl ps                          # list running containers
crictl images                      # list cached images
crictl logs <container-id>         # read container logs
crictl inspect <container-id>      # full container metadata
```

---

## How the Components Interact — Pod Creation Flow

End-to-end walkthrough of what happens when you run `kubectl apply -f pod.yaml`:

```
1. kubectl apply -f pod.yaml
        ↓
2. kube-apiserver
   • authenticates the request
   • runs admission controllers (LimitRanger injects defaults, PSA validates)
   • writes Pod object to etcd with status.phase = Pending

3. kube-scheduler (watching for Pods with no nodeName)
   • filters nodes (resources, taints, affinity)
   • scores remaining nodes
   • writes nodeName = "node-2" to the Pod spec in etcd

4. kubelet on node-2 (watching for Pods assigned to it)
   • sees the Pod spec
   • calls containerd via CRI: pull image, create container
   • mounts volumes (calls CSI driver), injects env vars
   • starts init containers (sequential), then app containers (parallel)
   • runs startup probe → then liveness + readiness probes
   • updates Pod status.phase = Running in etcd via API server

5. kube-proxy on all nodes (watching Services and Endpoints)
   • Endpoints controller updated the Service's endpoint list
   • kube-proxy programs iptables/IPVS rules to include the new Pod IP

Pod is now Running and receiving traffic.
```

---

## Key Concepts

### Declarative Model

You describe the **desired state** in YAML. Kubernetes controllers continuously reconcile actual state to match desired state.

```
Desired State (YAML) → kube-apiserver → etcd
                                          ↓
                              Controllers watch etcd
                                          ↓
                              Reconcile: actual → desired
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
| [Jobs & CronJobs](workloads/jobs/README.md) | Batch workloads, scheduled tasks |

### Scaling

| Resource | Description |
|---|---|
| [Scaling Overview](scaling/README.md) | Decision guide — when to use HPA vs VPA vs KEDA |
| [HPA](scaling/hpa.md) | Horizontal Pod Autoscaler — replicas based on CPU, memory, custom metrics |
| [VPA](scaling/vpa.md) | Vertical Pod Autoscaler — right-size CPU/memory requests |
| [KEDA](scaling/keda.md) | Event-driven autoscaling — SQS, Kafka, Prometheus, cron, scale-to-zero |

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
| [Helm](operations/helm.md) | Chart management, templating, upgrades |
| [Monitoring](operations/monitoring.md) | Prometheus, Grafana, CloudWatch Container Insights |
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
