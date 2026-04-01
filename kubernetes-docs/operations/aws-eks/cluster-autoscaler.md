# Cluster Autoscaler — Node Autoscaling on EKS

The Cluster Autoscaler (CA) automatically adjusts the number of nodes in your cluster when:
- Pods cannot be scheduled due to insufficient resources (**scale up**)
- Nodes are underutilized and their Pods can be rescheduled elsewhere (**scale down**)

---

## How It Works

```
Scale Up:
  Pod in Pending state (Insufficient CPU/memory)
  → CA evaluates: can adding a node fix this?
  → Yes → CA increases ASG desired count → new node joins → Pod scheduled

Scale Down:
  Node utilization < 50% for 10+ minutes
  → CA checks: can all Pods move to other nodes?
  → Yes → CA drains node (cordon + evict) → decreases ASG count
```

---

## Prerequisites

1. EKS managed node groups with autoscaling enabled
2. IAM permissions for CA to modify Auto Scaling Groups
3. Node group tags required by CA

---

## Setup

### 1. Tag Node Groups (required for CA to discover them)

```bash
# These tags must be on the Auto Scaling Group
aws autoscaling create-or-update-tags \
  --tags \
    "ResourceId=<asg-name>,ResourceType=auto-scaling-group,Key=k8s.io/cluster-autoscaler/<cluster-name>,Value=owned,PropagateAtLaunch=true" \
    "ResourceId=<asg-name>,ResourceType=auto-scaling-group,Key=k8s.io/cluster-autoscaler/enabled,Value=true,PropagateAtLaunch=true"
```

With `eksctl`, tags are applied automatically when you set `minSize`/`maxSize` on a managed node group.

### 2. Create IAM Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DescribeScalingActivities",
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup",
        "ec2:DescribeImages",
        "ec2:DescribeInstanceTypes",
        "ec2:DescribeLaunchTemplateVersions",
        "ec2:GetInstanceTypesFromInstanceRequirements",
        "eks:DescribeNodegroup"
      ],
      "Resource": "*"
    }
  ]
}
```

```bash
aws iam create-policy \
  --policy-name ClusterAutoscalerPolicy \
  --policy-document file://cluster-autoscaler-policy.json
```

### 3. Create IRSA for Cluster Autoscaler

```bash
eksctl create iamserviceaccount \
  --name cluster-autoscaler \
  --namespace kube-system \
  --cluster my-cluster \
  --attach-policy-arn arn:aws:iam::<account-id>:policy/ClusterAutoscalerPolicy \
  --approve \
  --override-existing-serviceaccounts
```

### 4. Deploy Cluster Autoscaler

```bash
helm repo add autoscaler https://kubernetes.github.io/autoscaler

helm upgrade --install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=my-cluster \
  --set awsRegion=us-east-1 \
  --set rbac.serviceAccount.create=false \
  --set rbac.serviceAccount.name=cluster-autoscaler \
  --set extraArgs.balance-similar-node-groups=true \
  --set extraArgs.skip-nodes-with-system-pods=false \
  --set extraArgs.scale-down-delay-after-add=5m \
  --set extraArgs.scale-down-unneeded-time=10m \
  --set extraArgs.scale-down-utilization-threshold=0.5
```

---

## Key Configuration Parameters

| Parameter | Default | Description |
|---|---|---|
| `scale-down-delay-after-add` | 10m | Wait after scale-up before considering scale-down |
| `scale-down-unneeded-time` | 10m | Node must be underutilized for this long before scale-down |
| `scale-down-utilization-threshold` | 0.5 | Node CPU+memory < 50% triggers scale-down consideration |
| `max-node-provision-time` | 15m | Give up if a new node doesn't appear within this time |
| `balance-similar-node-groups` | false | Keep similar node groups at similar sizes (good for multi-AZ) |
| `skip-nodes-with-system-pods` | true | Don't scale down nodes with kube-system Pods |
| `skip-nodes-with-local-storage` | true | Don't scale down nodes with Pods using local volumes |

---

## Preventing Scale-Down for Specific Pods

### Annotation on Pod

```yaml
metadata:
  annotations:
    cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
```

Use for: Pods with local state, long-running batch jobs that shouldn't be interrupted.

### PodDisruptionBudget (PDB)

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
  namespace: production
spec:
  minAvailable: 2         # always keep at least 2 Pods running
  selector:
    matchLabels:
      app: my-app
```

CA respects PDBs — it will not evict a Pod if doing so would violate the PDB.

---

## Overprovisioning (Warm Nodes)

Cluster Autoscaler only adds nodes **after** a Pod is pending. This means there's a delay (usually 2–5 minutes) while the node boots.

To maintain warm spare capacity, deploy **low-priority pause Pods** that consume placeholder capacity. Real Pods preempt them immediately.

```yaml
# PriorityClass for placeholder Pods
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: overprovisioning
value: -1            # negative = below all normal Pods
globalDefault: false
preemptionPolicy: Never

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: overprovisioning
  namespace: kube-system
spec:
  replicas: 2            # adjust to desired warm capacity
  selector:
    matchLabels:
      app: overprovisioning
  template:
    metadata:
      labels:
        app: overprovisioning
    spec:
      priorityClassName: overprovisioning
      containers:
        - name: pause
          image: k8s.gcr.io/pause:3.9
          resources:
            requests:
              cpu: "1"
              memory: "1Gi"
```

---

## Monitoring

```bash
# Check Cluster Autoscaler logs
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-cluster-autoscaler --tail=100

# Check scaling events
kubectl get events -n kube-system | grep cluster-autoscaler

# Check node group status
kubectl describe configmap cluster-autoscaler-status -n kube-system

# Watch node count
watch kubectl get nodes
```

---

## Karpenter (AWS-Native Alternative)

Karpenter is an AWS-developed autoscaler that provisions nodes **faster** (30–60s vs 2–5min) and with more flexibility than Cluster Autoscaler.

| Feature | Cluster Autoscaler | Karpenter |
|---|---|---|
| Node provision time | 2–5 min | 30–60s |
| Instance selection | Predefined node groups | Dynamic from instance family |
| Spot handling | Per node group | Automatic diversification |
| Setup complexity | Moderate | Moderate |
| AWS integration | ASG-based | Direct EC2 Fleet API |

> For new EKS clusters on AWS, Karpenter is generally recommended over Cluster Autoscaler.

```bash
# Install Karpenter (brief overview)
helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --namespace karpenter \
  --create-namespace \
  --set settings.clusterName=my-cluster \
  --set settings.interruptionQueue=my-cluster \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi
```
