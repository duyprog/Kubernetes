# ConfigMaps — Non-Sensitive Configuration

A ConfigMap stores non-sensitive configuration data as key-value pairs. It decouples configuration from container images, letting you change app behavior without rebuilding images.

---

## Creating ConfigMaps

### From a YAML manifest

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  # Simple key-value
  APP_ENV: "production"
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"

  # Multi-line file content
  app.properties: |
    server.port=8080
    server.timeout=30
    feature.flag.new-ui=true

  nginx.conf: |
    server {
      listen 80;
      location / {
        proxy_pass http://localhost:8080;
      }
    }
```

### From the CLI

```bash
# From literal values
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=info \
  -n production

# From a file (key = filename, value = file content)
kubectl create configmap nginx-config \
  --from-file=nginx.conf \
  -n production

# From a directory (one key per file in the directory)
kubectl create configmap app-configs \
  --from-file=./configs/ \
  -n production
```

---

## Using ConfigMaps in Pods

### As Environment Variables (single key)

```yaml
env:
  - name: APP_ENV
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: APP_ENV
```

### All Keys as Environment Variables

```yaml
envFrom:
  - configMapRef:
      name: app-config
```

> All keys in the ConfigMap become environment variable names. Watch for naming conflicts with other `envFrom` sources.

### As a Volume (files)

```yaml
# Pod spec volumes
volumes:
  - name: config-vol
    configMap:
      name: app-config

# Container volumeMounts
volumeMounts:
  - name: config-vol
    mountPath: /etc/config
    readOnly: true
```

Each key in the ConfigMap becomes a **file** inside `/etc/config`. For example, `app.properties` becomes `/etc/config/app.properties`.

### Mount a Specific Key as a File

```yaml
volumes:
  - name: nginx-config-vol
    configMap:
      name: app-config
      items:
        - key: nginx.conf
          path: nginx.conf      # filename in mountPath
          mode: 0644            # optional file permission

volumeMounts:
  - name: nginx-config-vol
    mountPath: /etc/nginx/conf.d
    readOnly: true
```

### Mount as a Single File at a Specific Path

```yaml
volumes:
  - name: app-props
    configMap:
      name: app-config
      items:
        - key: app.properties
          path: app.properties

volumeMounts:
  - name: app-props
    mountPath: /app/config/app.properties
    subPath: app.properties     # mount as a file, not a directory
    readOnly: true
```

> Use `subPath` when you want to mount a single file without overwriting the whole directory.

---

## Live Updates

When a ConfigMap is mounted as a **volume**, Kubernetes automatically updates the mounted files when the ConfigMap changes (within ~60 seconds, based on kubelet sync period).

When used as **environment variables** (`env` or `envFrom`), changes **do not** propagate until the Pod restarts. This is a common source of confusion.

| Consumption method | Auto-updates? |
|---|---|
| Volume mount | Yes (within ~60s) |
| `env` / `envFrom` | No — requires Pod restart |

---

## Immutable ConfigMaps

For configuration that should never change (prevent accidental modification):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v1
immutable: true
data:
  API_VERSION: "v1"
```

Immutable ConfigMaps cannot be edited — delete and recreate. This also improves cluster performance by reducing watch load on the API server.

---

## Commands

```bash
# View a ConfigMap
kubectl get configmap app-config -n production -o yaml

# Describe (shows data keys)
kubectl describe configmap app-config -n production

# Edit
kubectl edit configmap app-config -n production

# Update from file
kubectl create configmap app-config \
  --from-file=app.properties \
  --dry-run=client -o yaml | kubectl apply -f -
```

---

## Best Practices

- Use ConfigMaps for non-sensitive config only — use Secrets for passwords, tokens, certs
- Use volume mounts for file-based config; use env vars for simple key-value config
- Name ConfigMaps to indicate version when using immutable (`app-config-v2`)
- Keep ConfigMaps small — the etcd limit per object is 1MB
- Use `kubectl diff` before applying changes to ConfigMaps that affect running Pods
