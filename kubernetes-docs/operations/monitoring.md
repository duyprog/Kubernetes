# Monitoring & Observability

---

## Observability Stack Overview

```
Metrics  → Prometheus (scrape) → Grafana (visualize) → Alertmanager (alert)
Logs     → Fluentd/Promtail → Elasticsearch/Loki → Kibana/Grafana
Traces   → OpenTelemetry → Tempo/Jaeger → Grafana

On EKS:  → CloudWatch Container Insights (AWS-native, no setup)
```

---

## metrics-server (required for kubectl top and HPA)

`metrics-server` collects CPU/memory from each node's kubelet and exposes them via the Metrics API. **Required** for `kubectl top` and HPA.

```bash
# Install on EKS
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Install via Helm
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm upgrade --install metrics-server metrics-server/metrics-server \
  -n kube-system

# Verify
kubectl top nodes
kubectl top pods -n production
kubectl top pods -n production --containers
```

---

## Prometheus Stack (kube-prometheus-stack)

The most common monitoring setup — deploys Prometheus, Grafana, Alertmanager, and a full set of recording rules and dashboards in one Helm chart.

### Install

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --create-namespace \
  -f monitoring-values.yaml
```

### monitoring-values.yaml

```yaml
grafana:
  enabled: true
  adminPassword: "CHANGE_ME"
  persistence:
    enabled: true
    storageClassName: gp3
    size: 10Gi
  ingress:
    enabled: true
    ingressClassName: alb
    annotations:
      alb.ingress.kubernetes.io/scheme: internal
    hosts:
      - grafana.internal.example.com

prometheus:
  prometheusSpec:
    retention: 15d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi
    # Scrape all ServiceMonitors across all namespaces
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          resources:
            requests:
              storage: 5Gi

# Node exporter — scrapes node-level metrics (CPU, disk, network)
nodeExporter:
  enabled: true

# kube-state-metrics — Deployment/Pod/Node object-level metrics
kubeStateMetrics:
  enabled: true
```

---

## Exposing App Metrics to Prometheus

### ServiceMonitor (recommended)

Prometheus discovers scrape targets via `ServiceMonitor` CRDs.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  namespace: production
  labels:
    release: kube-prometheus-stack    # must match Prometheus selector
spec:
  selector:
    matchLabels:
      app: my-app
  namespaceSelector:
    matchNames:
      - production
  endpoints:
    - port: http           # named port on the Service
      path: /metrics
      interval: 30s
      scrapeTimeout: 10s
```

The app must expose a `/metrics` endpoint in Prometheus format. Use a client library:
- Go: `prometheus/client_golang`
- Python: `prometheus_client`
- Java: `micrometer` (Spring Boot Actuator)
- Node.js: `prom-client`

### PodMonitor (scrape Pods directly, no Service needed)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: my-app
  namespace: production
spec:
  selector:
    matchLabels:
      app: my-app
  podMetricsEndpoints:
    - port: metrics
      path: /metrics
      interval: 30s
```

---

## Key Metrics to Monitor

### Pod / Container Metrics

```promql
# CPU usage (cores)
rate(container_cpu_usage_seconds_total{container!=""}[5m])

# CPU throttling ratio (high = limits too low)
rate(container_cpu_cfs_throttled_seconds_total[5m])
  / rate(container_cpu_cfs_periods_total[5m])

# Memory usage
container_memory_working_set_bytes{container!=""}

# Memory usage vs limit (%)
container_memory_working_set_bytes
  / container_spec_memory_limit_bytes * 100

# Container restart count (alert if > 0 in last hour)
increase(kube_pod_container_status_restarts_total[1h])

# OOMKilled containers
kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}
```

### Deployment / ReplicaSet Metrics

```promql
# Unavailable replicas (should be 0)
kube_deployment_status_replicas_unavailable

# Rollout progress (desired vs available)
kube_deployment_status_replicas_available
  / kube_deployment_spec_replicas

# Pods not ready
kube_pod_status_ready{condition="false"}
```

### Node Metrics

```promql
# Node CPU utilization
1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (node)

# Node memory available
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes

# Node disk usage
1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)
```

---

## Alertmanager Configuration

```yaml
# alerts.yaml — PrometheusRule
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: my-app-alerts
  namespace: production
  labels:
    release: kube-prometheus-stack
spec:
  groups:
    - name: my-app
      interval: 30s
      rules:
        - alert: HighCPUThrottling
          expr: |
            rate(container_cpu_cfs_throttled_seconds_total{namespace="production"}[5m])
            / rate(container_cpu_cfs_periods_total{namespace="production"}[5m]) > 0.5
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High CPU throttling on {{ $labels.pod }}"
            description: "Container {{ $labels.container }} in pod {{ $labels.pod }} is throttled {{ $value | humanizePercentage }}."

        - alert: PodCrashLooping
          expr: increase(kube_pod_container_status_restarts_total{namespace="production"}[1h]) > 3
          for: 0m
          labels:
            severity: critical
          annotations:
            summary: "Pod crash looping: {{ $labels.pod }}"

        - alert: DeploymentReplicasMismatch
          expr: |
            kube_deployment_spec_replicas{namespace="production"}
            != kube_deployment_status_replicas_available{namespace="production"}
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Deployment {{ $labels.deployment }} has unavailable replicas"
```

### Alertmanager Routing (Slack + PagerDuty)

```yaml
# In kube-prometheus-stack values
alertmanager:
  config:
    global:
      slack_api_url: "https://hooks.slack.com/services/xxx"

    route:
      group_by: ['alertname', 'namespace']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      receiver: slack-warnings
      routes:
        - matchers:
            - severity = critical
          receiver: pagerduty-critical
          continue: true
        - matchers:
            - severity = critical
          receiver: slack-critical

    receivers:
      - name: slack-warnings
        slack_configs:
          - channel: '#k8s-alerts'
            title: '{{ .GroupLabels.alertname }}'
            text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

      - name: slack-critical
        slack_configs:
          - channel: '#k8s-critical'
            title: ':fire: CRITICAL: {{ .GroupLabels.alertname }}'

      - name: pagerduty-critical
        pagerduty_configs:
          - service_key: "YOUR_PAGERDUTY_KEY"
```

---

## CloudWatch Container Insights (EKS — AWS-native)

Sends metrics and logs directly to CloudWatch without running Prometheus.

```bash
# Install CloudWatch agent + Fluentd as DaemonSets
ClusterName=my-cluster
RegionName=us-east-1
FluentBitHttpPort='2020'
FluentBitReadFromHead='Off'

curl https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluent-bit-quickstart.yaml | \
  sed "s/{{cluster_name}}/${ClusterName}/;s/{{region_name}}/${RegionName}/;s/{{http_server_toggle}}/On/;s/{{http_server_port}}/${FluentBitHttpPort}/;s/{{read_from_head}}/${FluentBitReadFromHead}/;s/{{read_from_tail}}/On/" | \
  kubectl apply -f -
```

**Requires** the node IAM role to have `CloudWatchAgentServerPolicy` attached.

```bash
# Check metrics in CloudWatch
aws cloudwatch list-metrics --namespace ContainerInsights --dimensions Name=ClusterName,Value=my-cluster
```

---

## Grafana Dashboards

Import pre-built dashboards from [grafana.com/dashboards](https://grafana.com/grafana/dashboards/):

| Dashboard ID | Name |
|---|---|
| 315 | Kubernetes cluster monitoring |
| 6417 | Kubernetes pod metrics |
| 13770 | 1 Kubernetes all-in-one cluster monitoring |
| 15757 | Kubernetes views — global |
| 11074 | Node Exporter for Prometheus |

```bash
# Access Grafana locally
kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 -n monitoring
# Open: http://localhost:3000  (admin / CHANGE_ME)
```

---

## Quick Debugging with Metrics

```bash
# CPU throttling across all Pods
kubectl top pods --all-namespaces --sort-by=cpu | head -20

# Memory usage across all Pods
kubectl top pods --all-namespaces --sort-by=memory | head -20

# Node pressure
kubectl top nodes

# Check HPA target metrics
kubectl get hpa -n production
kubectl describe hpa my-app -n production
# Look for: "Metrics:" section showing current vs target
```
