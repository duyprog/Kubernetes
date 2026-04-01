# Pod Basics — Spec Structure & Container Configuration

---

## Pod Spec Structure

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <name>
  namespace: <namespace>
  labels: {}
  annotations: {}
spec:
  # Scheduling
  nodeSelector: {}
  affinity: {}
  tolerations: []
  priorityClassName: ""

  # Lifecycle
  restartPolicy: Always
  terminationGracePeriodSeconds: 30

  # Containers
  initContainers: []
  containers: []          # required — at least one

  # Storage
  volumes: []

  # Security
  securityContext: {}

  # Service account
  serviceAccountName: default
  automountServiceAccountToken: false
```

---

## Container Fields

Each item in `spec.containers` supports:

```yaml
containers:
  - name: app                         # required, unique within Pod
    image: my-registry/my-app:1.0     # required
    imagePullPolicy: IfNotPresent     # Always | IfNotPresent | Never

    command: ["./server"]             # overrides Dockerfile ENTRYPOINT
    args: ["--port=8080"]             # overrides Dockerfile CMD

    ports:
      - name: http
        containerPort: 8080
        protocol: TCP

    env: []
    envFrom: []

    resources: {}
    volumeMounts: []
    securityContext: {}

    livenessProbe: {}
    readinessProbe: {}
    startupProbe: {}

    lifecycle: {}
```

### `imagePullPolicy`

| Value | Behavior |
|---|---|
| `Always` | Always pull from registry — guarantees latest image |
| `IfNotPresent` | Pull only if not cached locally — faster, use for tagged releases |
| `Never` | Never pull — image must exist on the node |

> Default is `Always` when tag is `latest` or not specified. Use explicit version tags in production.

---

## Environment Variables

### Direct Value

```yaml
env:
  - name: APP_ENV
    value: "production"
  - name: LOG_LEVEL
    value: "info"
```

### From ConfigMap (single key)

```yaml
env:
  - name: DB_HOST
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: db_host
```

### From Secret (single key)

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: app-secrets
        key: db_password
```

### All Keys from ConfigMap or Secret

```yaml
envFrom:
  - configMapRef:
      name: app-config
  - secretRef:
      name: app-secrets
```

> All keys in the ConfigMap/Secret become environment variable names. Use carefully — name collisions are silently overwritten.

### Downward API (Pod metadata as env vars)

```yaml
env:
  - name: POD_NAME
    valueFrom:
      fieldRef:
        fieldPath: metadata.name
  - name: POD_NAMESPACE
    valueFrom:
      fieldRef:
        fieldPath: metadata.namespace
  - name: NODE_NAME
    valueFrom:
      fieldRef:
        fieldPath: spec.nodeName
  - name: POD_IP
    valueFrom:
      fieldRef:
        fieldPath: status.podIP
  - name: CPU_REQUEST
    valueFrom:
      resourceFieldRef:
        containerName: app
        resource: requests.cpu
```

---

## Volume Mounts

Volumes are defined at the Pod level; containers reference them by name.

```yaml
# Pod spec
volumes:
  - name: config-vol
    configMap:
      name: app-config
  - name: secret-vol
    secret:
      secretName: app-secrets
  - name: data-vol
    emptyDir: {}
  - name: pvc-vol
    persistentVolumeClaim:
      claimName: my-pvc

# Container spec
volumeMounts:
  - name: config-vol
    mountPath: /etc/config
    readOnly: true
  - name: secret-vol
    mountPath: /etc/secrets
    readOnly: true
  - name: data-vol
    mountPath: /data
  - name: pvc-vol
    mountPath: /var/data
```

### Mount a Single Key from ConfigMap

```yaml
volumes:
  - name: config-file
    configMap:
      name: app-config
      items:
        - key: app.properties
          path: app.properties   # filename inside mountPath
```

---

## Security Context

Defined at Pod level (applies to all containers) or container level (overrides Pod level).

```yaml
# Pod-level
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000              # group that owns mounted volumes

  containers:
    - name: app
      securityContext:
        readOnlyRootFilesystem: true
        allowPrivilegeEscalation: false
        capabilities:
          drop: ["ALL"]
          add: ["NET_BIND_SERVICE"]  # only if binding to port < 1024
```

| Setting | Recommendation |
|---|---|
| `runAsNonRoot: true` | Always — never run as root |
| `readOnlyRootFilesystem: true` | Enable — requires writable volume mounts for `/tmp` etc. |
| `allowPrivilegeEscalation: false` | Always |
| `capabilities: drop: ["ALL"]` | Always — add back only what's needed |

---

## Probes

See [Health Checks section in lifecycle README](README.md) for full details.

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 10

livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

---

## Multi-Container Pod Patterns

### Sidecar

Extends the main container — shares network and volumes.

```yaml
containers:
  - name: app
    image: my-app:1.0
    volumeMounts:
      - name: log-vol
        mountPath: /var/log/app

  - name: log-shipper
    image: fluentd:v1.16
    volumeMounts:
      - name: log-vol
        mountPath: /var/log/app
        readOnly: true
```

### Ambassador

Proxy between app and external world — app connects to `localhost`.

```yaml
containers:
  - name: app
    image: my-app:1.0
    env:
      - name: DB_HOST
        value: "localhost"
      - name: DB_PORT
        value: "5432"

  - name: db-proxy
    image: cloud-sql-proxy:latest
    args: ["--port=5432", "project:region:instance"]
```

### Adapter

Transforms app output to a standard format.

```yaml
containers:
  - name: app
    image: legacy-app:1.0

  - name: metrics-adapter
    image: metrics-exporter:1.0
    ports:
      - containerPort: 9090
```

---

## Examples

- [simple-pod.yaml](examples/simple-pod.yaml)
- [multi-container-pod.yaml](examples/multi-container-pod.yaml)
