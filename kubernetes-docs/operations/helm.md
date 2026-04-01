# Helm — Kubernetes Package Manager

Helm is the standard package manager for Kubernetes. It templates Kubernetes manifests into reusable, versioned **charts** and manages their lifecycle (install, upgrade, rollback, delete).

---

## Core Concepts

| Term | Description |
|---|---|
| **Chart** | A package of Kubernetes manifests + templates + default values |
| **Release** | A deployed instance of a chart in a cluster |
| **Repository** | A collection of charts (like apt/yum repos) |
| **Values** | Configuration injected into chart templates at render time |
| **Revision** | Each install/upgrade creates a new revision — rollback reverts to a previous one |

---

## Installation

```bash
# macOS
brew install helm

# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify
helm version
```

---

## Repository Management

```bash
# Add a repository
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add eks https://aws.github.io/eks-charts
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# Update all repos
helm repo update

# List repos
helm repo list

# Search for a chart
helm search repo nginx
helm search repo postgres --versions
```

---

## Installing Charts

```bash
# Install with defaults
helm install my-release bitnami/postgresql -n production

# Install with custom values inline
helm install my-release bitnami/postgresql \
  -n production \
  --set auth.postgresPassword=secret \
  --set primary.persistence.size=50Gi

# Install with a values file (preferred)
helm install my-release bitnami/postgresql \
  -n production \
  -f values.yaml

# Install with multiple values files (later files override earlier)
helm install my-release bitnami/postgresql \
  -n production \
  -f values-base.yaml \
  -f values-production.yaml

# Dry-run (render templates without applying)
helm install my-release bitnami/postgresql \
  -n production \
  -f values.yaml \
  --dry-run

# Install a specific chart version
helm install my-release bitnami/postgresql \
  -n production \
  --version 13.2.0
```

---

## Upgrading Releases

```bash
# Upgrade with new values
helm upgrade my-release bitnami/postgresql \
  -n production \
  -f values.yaml

# Install if not exists, upgrade if exists
helm upgrade --install my-release bitnami/postgresql \
  -n production \
  -f values.yaml

# Preview diff before upgrading (requires helm-diff plugin)
helm diff upgrade my-release bitnami/postgresql \
  -n production \
  -f values.yaml
```

---

## Rollback

```bash
# View release history
helm history my-release -n production

# Rollback to previous revision
helm rollback my-release -n production

# Rollback to a specific revision
helm rollback my-release 2 -n production
```

---

## Inspecting Releases

```bash
# List all releases in a namespace
helm list -n production
helm list --all-namespaces

# Get release status
helm status my-release -n production

# Get current values (what was applied)
helm get values my-release -n production

# Get all values (including defaults)
helm get values my-release -n production --all

# Get rendered manifests for a release
helm get manifest my-release -n production

# Show default values for a chart
helm show values bitnami/postgresql
helm show values bitnami/postgresql --version 13.2.0
```

---

## Uninstalling

```bash
# Uninstall (keeps history for rollback)
helm uninstall my-release -n production

# Uninstall and delete history
helm uninstall my-release -n production --keep-history
```

---

## Creating Charts

### Scaffold a New Chart

```bash
helm create my-app
```

Generated structure:

```
my-app/
├── Chart.yaml           # chart metadata
├── values.yaml          # default values
├── charts/              # chart dependencies
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── hpa.yaml
    ├── serviceaccount.yaml
    ├── _helpers.tpl     # reusable template functions
    └── NOTES.txt        # post-install message
```

### Chart.yaml

```yaml
apiVersion: v2
name: my-app
description: My application Helm chart
type: application
version: 1.2.0          # chart version
appVersion: "2.5.1"     # application version (informational)

dependencies:
  - name: postgresql
    version: "13.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
```

### values.yaml

```yaml
replicaCount: 2

image:
  repository: my-registry/my-app
  tag: "1.0.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  host: app.example.com

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

postgresql:
  enabled: true
  auth:
    database: mydb
```

### Template Example (deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 8080
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          {{- if .Values.env }}
          env:
            {{- toYaml .Values.env | nindent 12 }}
          {{- end }}
```

---

## Environment-Specific Values

Structure for managing multiple environments:

```
charts/my-app/
├── Chart.yaml
├── values.yaml              # base defaults
├── values-dev.yaml          # dev overrides
├── values-staging.yaml
└── values-production.yaml
```

```bash
# Deploy to production
helm upgrade --install my-app ./charts/my-app \
  -n production \
  -f charts/my-app/values.yaml \
  -f charts/my-app/values-production.yaml

# Deploy to dev
helm upgrade --install my-app ./charts/my-app \
  -n dev \
  -f charts/my-app/values.yaml \
  -f charts/my-app/values-dev.yaml
```

---

## Chart Dependencies

```bash
# Update dependencies (download charts/)
helm dependency update ./my-app

# Build dependencies (from lockfile)
helm dependency build ./my-app
```

---

## Useful Plugins

```bash
# Install helm-diff — preview changes before upgrade
helm plugin install https://github.com/databus23/helm-diff

# Install helm-secrets — encrypt values with SOPS/Vault
helm plugin install https://github.com/jkroepke/helm-secrets

# Usage
helm diff upgrade my-release ./my-app -n production -f values.yaml
helm secrets upgrade my-release ./my-app -n production -f secrets.yaml
```

---

## Linting and Testing

```bash
# Lint a chart
helm lint ./my-app
helm lint ./my-app -f values-production.yaml

# Render templates locally (without a cluster)
helm template my-release ./my-app -f values.yaml

# Run chart tests (Pods defined in templates/tests/)
helm test my-release -n production
```

---

## OCI Registries (Helm 3.8+)

Store charts in container registries (ECR, GHCR, etc.) instead of traditional repos.

```bash
# Push to ECR
aws ecr create-repository --repository-name helm-charts/my-app
helm package ./my-app
helm push my-app-1.2.0.tgz oci://<account>.dkr.ecr.us-east-1.amazonaws.com/helm-charts

# Install from ECR
aws ecr get-login-password --region us-east-1 | \
  helm registry login --username AWS --password-stdin \
  <account>.dkr.ecr.us-east-1.amazonaws.com

helm upgrade --install my-app \
  oci://<account>.dkr.ecr.us-east-1.amazonaws.com/helm-charts/my-app \
  --version 1.2.0 \
  -n production \
  -f values-production.yaml
```

---

## Common Helm Commands Cheatsheet

```bash
# Search
helm search repo <keyword>
helm show values <chart>

# Install / upgrade
helm upgrade --install <release> <chart> -n <ns> -f values.yaml
helm upgrade --install <release> <chart> -n <ns> --set key=value

# Inspect
helm list -n <ns>
helm status <release> -n <ns>
helm get values <release> -n <ns>
helm history <release> -n <ns>

# Rollback
helm rollback <release> [revision] -n <ns>

# Uninstall
helm uninstall <release> -n <ns>

# Debug
helm template <release> <chart> -f values.yaml
helm lint <chart>
helm diff upgrade <release> <chart> -f values.yaml
```
