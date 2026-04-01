# IRSA — IAM Roles for Service Accounts

IRSA allows Pods to assume an **AWS IAM Role** using a Kubernetes ServiceAccount — without storing AWS credentials in Secrets or on nodes.

---

## How IRSA Works

```
Pod (ServiceAccount: my-app)
  → kubelet mounts projected ServiceAccount token (OIDC-signed JWT)
  → App calls AWS SDK
  → SDK presents JWT to AWS STS AssumeRoleWithWebIdentity
  → STS validates JWT signature against EKS OIDC provider
  → STS returns temporary AWS credentials
  → App uses credentials to access AWS services
```

**No static credentials. No node IAM role permissions shared across all Pods.**

---

## Prerequisites

1. EKS cluster with OIDC provider enabled
2. IAM OIDC identity provider associated with the cluster

```bash
# Check if OIDC is enabled
aws eks describe-cluster --name my-cluster \
  --query 'cluster.identity.oidc.issuer' --output text

# Enable OIDC (if not enabled)
eksctl utils associate-iam-oidc-provider \
  --cluster my-cluster \
  --approve

# Get OIDC issuer URL (needed for manual IAM role creation)
OIDC_URL=$(aws eks describe-cluster --name my-cluster \
  --query 'cluster.identity.oidc.issuer' --output text | sed 's|https://||')
```

---

## Step-by-Step: Create IRSA with eksctl

The simplest approach — eksctl handles everything.

```bash
eksctl create iamserviceaccount \
  --name my-app-sa \
  --namespace production \
  --cluster my-cluster \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve \
  --override-existing-serviceaccounts
```

This creates:
1. An IAM role with a trust policy for the ServiceAccount
2. A Kubernetes ServiceAccount annotated with the IAM role ARN

---

## Step-by-Step: Manual Setup

### 1. Create IAM Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/*"
    },
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::my-bucket"
    }
  ]
}
```

```bash
aws iam create-policy \
  --policy-name MyAppS3Policy \
  --policy-document file://policy.json
```

### 2. Create IAM Role with Trust Policy

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
OIDC_PROVIDER=$(aws eks describe-cluster --name my-cluster \
  --query 'cluster.identity.oidc.issuer' --output text | sed 's|https://||')
NAMESPACE=production
SA_NAME=my-app-sa
```

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/<OIDC_PROVIDER>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "<OIDC_PROVIDER>:sub": "system:serviceaccount:<NAMESPACE>:<SA_NAME>",
          "<OIDC_PROVIDER>:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

```bash
aws iam create-role \
  --role-name MyApp-IRSA-Role \
  --assume-role-policy-document file://trust-policy.json

aws iam attach-role-policy \
  --role-name MyApp-IRSA-Role \
  --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/MyAppS3Policy
```

### 3. Create Annotated ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/MyApp-IRSA-Role
automountServiceAccountToken: true
```

### 4. Use in Pod

```yaml
spec:
  serviceAccountName: my-app-sa    # that's it — SDK picks up credentials automatically
  containers:
    - name: app
      image: my-app:1.0
      env:
        - name: AWS_REGION
          value: us-east-1
```

---

## Common IRSA Use Cases

### S3 Access

```bash
eksctl create iamserviceaccount \
  --name app-s3-sa \
  --namespace production \
  --cluster my-cluster \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve
```

### Secrets Manager / SSM Parameter Store

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:us-east-1:<account>:secret:production/*"
    },
    {
      "Effect": "Allow",
      "Action": "ssm:GetParameter",
      "Resource": "arn:aws:ssm:us-east-1:<account>:parameter/production/*"
    }
  ]
}
```

### SQS / SNS

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["sqs:SendMessage", "sqs:ReceiveMessage", "sqs:DeleteMessage"],
      "Resource": "arn:aws:sqs:us-east-1:<account>:my-queue"
    }
  ]
}
```

---

## Verification

```bash
# Verify the ServiceAccount annotation
kubectl get serviceaccount my-app-sa -n production -o yaml
# Should show: eks.amazonaws.com/role-arn: arn:aws:iam::...

# Verify from inside the Pod
kubectl exec -it <pod-name> -n production -- \
  aws sts get-caller-identity
# Should show the IRSA role ARN as the identity

# Verify the projected token is mounted
kubectl exec -it <pod-name> -n production -- \
  ls /var/run/secrets/eks.amazonaws.com/serviceaccount/
```

---

## Troubleshooting IRSA

| Symptom | Cause | Fix |
|---|---|---|
| `NoCredentialProviders` | Token not mounted or SA not annotated | Check annotation and `automountServiceAccountToken: true` |
| `AccessDenied` | IAM policy missing the action | Update IAM policy |
| `InvalidIdentityToken` | OIDC provider not associated | Run `eksctl utils associate-iam-oidc-provider` |
| `AssumeRoleWithWebIdentity` fails | Trust policy `sub` doesn't match | Check namespace and SA name in trust policy condition |

```bash
# Check Pod's token
kubectl exec -it <pod> -n production -- \
  cat /var/run/secrets/eks.amazonaws.com/serviceaccount/token | \
  cut -d. -f2 | base64 -d | python3 -m json.tool
# Verify "sub" field matches: system:serviceaccount:<namespace>:<sa-name>
```
