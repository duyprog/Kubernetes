# EKS Setup — Cluster Provisioning on AWS

---

## EKS Architecture

```
AWS Account
└── VPC
    ├── Private Subnets (AZ-a, AZ-b, AZ-c)  ← Worker nodes
    ├── Public Subnets  (AZ-a, AZ-b, AZ-c)  ← Load balancers
    └── EKS Control Plane (AWS-managed)
        └── API Server endpoint (public or private)
```

---

## Cluster Provisioning with eksctl (recommended)

### Minimal Cluster Config

```yaml
# cluster.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-cluster
  region: us-east-1
  version: "1.30"

iam:
  withOIDC: true      # required for IRSA (IAM Roles for Service Accounts)

vpc:
  cidr: 10.0.0.0/16
  clusterEndpoints:
    publicAccess: true
    privateAccess: true

managedNodeGroups:
  - name: general
    instanceType: m5.xlarge
    minSize: 2
    maxSize: 10
    desiredCapacity: 3
    privateNetworking: true   # nodes in private subnets
    volumeSize: 100
    volumeType: gp3
    volumeEncrypted: true
    iam:
      attachPolicyARNs:
        - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
        - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
        - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
    tags:
      environment: production
      team: platform

addons:
  - name: vpc-cni
    version: latest
  - name: coredns
    version: latest
  - name: kube-proxy
    version: latest
  - name: aws-ebs-csi-driver
    version: latest
    serviceAccountRoleARN: arn:aws:iam::<account-id>:role/AmazonEKS_EBS_CSI_DriverRole

cloudWatch:
  clusterLogging:
    enableTypes: ["api", "audit", "authenticator", "controllerManager", "scheduler"]
```

```bash
eksctl create cluster -f cluster.yaml
```

---

## Multiple Node Groups

Separate node groups for different workload types:

```yaml
managedNodeGroups:
  # General purpose workloads
  - name: general
    instanceType: m5.xlarge
    minSize: 2
    maxSize: 10
    desiredCapacity: 3
    privateNetworking: true

  # Memory-optimized for caches/databases
  - name: memory-optimized
    instanceType: r5.2xlarge
    minSize: 0
    maxSize: 5
    desiredCapacity: 0
    privateNetworking: true
    taints:
      - key: workload-type
        value: memory-intensive
        effect: NoSchedule
    labels:
      workload-type: memory-intensive

  # Spot instances for batch/non-critical
  - name: spot-workers
    instanceTypes: ["m5.large", "m5.xlarge", "m4.xlarge"]
    spot: true
    minSize: 0
    maxSize: 20
    desiredCapacity: 0
    privateNetworking: true
    taints:
      - key: spot
        value: "true"
        effect: NoSchedule
    labels:
      node-type: spot
```

---

## Cluster Access (kubeconfig)

```bash
# Update kubeconfig
aws eks update-kubeconfig --name my-cluster --region us-east-1

# Verify
kubectl get nodes
kubectl cluster-info
```

### Grant Access to Additional IAM Users/Roles

EKS uses `aws-auth` ConfigMap to map IAM identities to Kubernetes RBAC.

```bash
# Add an IAM user
eksctl create iamidentitymapping \
  --cluster my-cluster \
  --region us-east-1 \
  --arn arn:aws:iam::<account-id>:user/jane \
  --username jane \
  --group system:masters    # cluster-admin; use a more restrictive group for non-admins

# Add an IAM role (common for CI/CD and SSO)
eksctl create iamidentitymapping \
  --cluster my-cluster \
  --region us-east-1 \
  --arn arn:aws:iam::<account-id>:role/DevOpsRole \
  --username devops-role \
  --group devops-group
```

---

## Essential Add-ons

### CoreDNS

Pre-installed. Handles in-cluster DNS.

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

### AWS Load Balancer Controller

Required for ALB Ingress and NLB LoadBalancer Services.

```bash
# Create IAM policy
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam-policy.json

# Create IRSA
eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::<account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# Install via Helm
helm repo add eks https://aws.github.io/eks-charts
helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

### Cluster Autoscaler

See [cluster-autoscaler.md](cluster-autoscaler.md).

### EBS CSI Driver

See [storage-classes.md](../../storage/storage-classes.md).

---

## Cluster Upgrades

```bash
# Check available versions
aws eks describe-addon-versions --query 'addons[].addonVersions[].compatibilities[].clusterVersion' \
  --output text | sort -u

# Upgrade control plane (one minor version at a time)
eksctl upgrade cluster --name my-cluster --version 1.30 --approve

# Upgrade managed node group
eksctl upgrade nodegroup \
  --name general \
  --cluster my-cluster \
  --kubernetes-version 1.30

# Update add-ons after cluster upgrade
aws eks update-addon --cluster-name my-cluster --addon-name vpc-cni --addon-version v1.18.0-eksbuild.1
aws eks update-addon --cluster-name my-cluster --addon-name coredns --addon-version v1.11.1-eksbuild.4
aws eks update-addon --cluster-name my-cluster --addon-name kube-proxy --addon-version v1.30.0-eksbuild.3
```

---

## Security Hardening

```bash
# Restrict API server to specific CIDRs
eksctl utils update-cluster-endpoints \
  --cluster=my-cluster \
  --public-access=true \
  --private-access=true \
  --public-access-cidrs="203.0.113.0/24"

# Enable Secrets encryption with KMS
aws eks associate-encryption-config \
  --cluster-name my-cluster \
  --encryption-config '[{"resources":["secrets"],"provider":{"keyArn":"arn:aws:kms:us-east-1:<account>:key/<key-id>"}}]'

# Enable control plane logging
aws eks update-cluster-config \
  --name my-cluster \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'
```
