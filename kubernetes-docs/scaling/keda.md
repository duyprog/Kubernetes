# KEDA — Event-Driven Autoscaling

KEDA (Kubernetes Event-Driven Autoscaling) extends HPA to support **any event source** as a scaling trigger — SQS queue depth, Kafka consumer lag, cron schedules, Prometheus queries, HTTP request rate, and 60+ more.

**Key capability HPA lacks:** KEDA can scale workloads **to zero** when there are no events, and back up from zero when events arrive.

---

## How KEDA Works

```
External Event Source (SQS, Kafka, Prometheus, ...)
        ↓
KEDA ScaledObject (defines: target workload + triggers + scaling rules)
        ↓
KEDA creates/manages an HPA object
        ↓
HPA scales the Deployment / StatefulSet / Job replicas
```

KEDA sits **on top of** HPA — it creates and manages an HPA resource. You don't need to create the HPA yourself.

---

## Installation

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update

helm upgrade --install keda kedacore/keda \
  --namespace keda \
  --create-namespace \
  --version 2.13.0
```

```bash
# Verify
kubectl get pods -n keda
kubectl get crd | grep keda
```

---

## ScaledObject — Core Resource

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: my-app-scaler
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app

  pollingInterval: 30       # check triggers every 30s (default: 30)
  cooldownPeriod: 300       # wait 5 min of idle before scaling to zero (default: 300)
  idleReplicaCount: 0       # scale to zero when no events (omit to keep minReplicaCount)
  minReplicaCount: 1        # minimum replicas when active (0 = scale to zero)
  maxReplicaCount: 50

  triggers:
    - type: <trigger-type>
      metadata:
        # trigger-specific config

  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 300
          policies:
            - type: Pods
              value: 2
              periodSeconds: 60
```

---

## Scale to Zero

When `minReplicaCount: 0` or `idleReplicaCount: 0`, KEDA scales the workload to zero when idle. It wakes it back up when a new event arrives.

```
No messages in SQS → replicas: 0  (saving cost)
Message arrives    → replicas: 1  (KEDA wakes it up, ~15-30s cold start)
Queue depth grows  → replicas: N
Queue drains       → wait cooldownPeriod → replicas: 0
```

> For latency-sensitive workloads, use `minReplicaCount: 1` to keep at least one warm replica.

---

## Triggers

### AWS SQS Queue

```yaml
triggers:
  - type: aws-sqs-queue
    authenticationRef:
      name: keda-aws-credentials    # TriggerAuthentication with IAM credentials or IRSA
    metadata:
      queueURL: https://sqs.us-east-1.amazonaws.com/123456789/my-queue
      queueLength: "10"             # target messages per Pod
      awsRegion: us-east-1
      identityOwner: operator       # use IRSA (recommended on EKS)
```

**Scaling math:** if queue has 100 messages and `queueLength: 10` → scales to 10 replicas.

#### IRSA for KEDA (EKS)

```bash
eksctl create iamserviceaccount \
  --name keda-operator \
  --namespace keda \
  --cluster my-cluster \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonSQSReadOnlyAccess \
  --approve
```

```yaml
# TriggerAuthentication using IRSA
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-aws-credentials
  namespace: production
spec:
  podIdentity:
    provider: aws-eks    # uses IRSA automatically
```

---

### Apache Kafka — Consumer Lag

```yaml
triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka.kafka.svc.cluster.local:9092
      consumerGroup: my-app-consumer-group
      topic: my-topic
      lagThreshold: "100"           # scale up when lag > 100 messages per Pod
      offsetResetPolicy: latest
```

---

### Prometheus Query

```yaml
triggers:
  - type: prometheus
    metadata:
      serverAddress: http://kube-prometheus-stack-prometheus.monitoring.svc:9090
      metricName: http_requests_per_second
      query: |
        sum(rate(http_requests_total{namespace="production", app="my-app"}[2m]))
      threshold: "500"              # scale up when query result > 500
      activationThreshold: "10"    # wake from zero when result > 10
```

---

### Cron Schedule

Scale up on a schedule — useful for predictable traffic patterns.

```yaml
triggers:
  - type: cron
    metadata:
      timezone: America/New_York
      start: "0 8 * * 1-5"      # scale up at 8 AM on weekdays
      end: "0 20 * * 1-5"       # scale down at 8 PM on weekdays
      desiredReplicas: "10"
```

**Combine cron + metric** for pre-scaling before expected load:

```yaml
triggers:
  - type: cron
    metadata:
      timezone: UTC
      start: "50 9 * * *"       # pre-scale 10 min before daily peak
      end: "0 11 * * *"
      desiredReplicas: "20"

  - type: aws-sqs-queue         # also scale based on actual queue
    metadata:
      queueURL: https://sqs...
      queueLength: "5"
```

KEDA uses the **highest** desired replica count across all triggers.

---

### HTTP Traffic (http-add-on)

Requires the [KEDA HTTP Add-on](https://github.com/kedacore/http-add-on).

```yaml
apiVersion: http.keda.sh/v1alpha1
kind: HTTPScaledObject
metadata:
  name: my-app-http-scaler
  namespace: production
spec:
  hosts:
    - app.example.com
  scaleTargetRef:
    deployment: my-app
    service: my-app
    port: 80
  replicas:
    min: 0
    max: 50
  scaledownPeriod: 300
  targetPendingRequests: 100    # scale when > 100 pending requests per Pod
```

---

### Other Supported Triggers

| Trigger | Use Case |
|---|---|
| `redis` | Redis list length / stream consumer lag |
| `rabbitmq` | RabbitMQ queue depth |
| `azure-servicebus` | Azure Service Bus queue/topic |
| `gcp-pubsub` | GCP Pub/Sub subscription |
| `mysql` / `postgresql` | Query result as metric |
| `elasticsearch` | Query result count |
| `datadog` | Any Datadog metric |
| `github-runner` | Scale GitHub Actions self-hosted runners |
| `metrics-api` | Any HTTP endpoint returning a JSON number |

---

## ScaledJob — Scaling Jobs (not Deployments)

For batch workloads where each Job processes one item from a queue.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: queue-processor
  namespace: production
spec:
  jobTargetRef:
    template:
      spec:
        restartPolicy: Never
        containers:
          - name: processor
            image: my-worker:1.0
            command: ["./process-one-message"]

  pollingInterval: 15
  maxReplicaCount: 50
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 3

  scalingStrategy:
    strategy: accurate          # or default, custom

  triggers:
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.us-east-1.amazonaws.com/123456789/jobs-queue
        queueLength: "1"        # one Job per message
        awsRegion: us-east-1
```

---

## Commands

```bash
# List ScaledObjects and their status
kubectl get scaledobject -n production

# Describe — shows current replica count and trigger status
kubectl describe scaledobject my-app-scaler -n production

# Check the HPA KEDA created
kubectl get hpa -n production | grep keda

# Pause a ScaledObject (stops KEDA from scaling — keeps current replicas)
kubectl annotate scaledobject my-app-scaler \
  autoscaling.keda.sh/paused-replicas="3" \
  -n production

# Resume
kubectl annotate scaledobject my-app-scaler \
  autoscaling.keda.sh/paused-replicas- \
  -n production

# Check KEDA operator logs
kubectl logs -n keda -l app=keda-operator --tail=50
```

---

## KEDA vs HPA

| Feature | HPA | KEDA |
|---|---|---|
| Scale on CPU/memory | Yes | Yes (via Prometheus) |
| Scale on queue depth | No | Yes |
| Scale to zero | No | Yes |
| Custom event sources | Limited | 60+ built-in scalers |
| Creates HPA under the hood | N/A | Yes |
| Combine with HPA | Manual | Automatic |
| Best for | CPU/memory-driven services | Event-driven, batch, queue workers |
