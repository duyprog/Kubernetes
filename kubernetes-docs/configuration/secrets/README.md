# Secrets — Sensitive Data & Encryption at Rest

Secrets store sensitive data (passwords, API keys, TLS certificates, tokens). They are similar to ConfigMaps but designed for confidential information.

> **Important:** By default, Secrets are stored as **base64-encoded** in etcd — this is encoding, not encryption. Enable etcd encryption at rest for true security.

---

## Secret Types

| Type | Description |
|---|---|
| `Opaque` | Arbitrary user-defined data (default) |
| `kubernetes.io/tls` | TLS certificate and key |
| `kubernetes.io/dockerconfigjson` | Private container registry credentials |
| `kubernetes.io/service-account-token` | Service account token (auto-created) |
| `kubernetes.io/ssh-auth` | SSH private key |
| `kubernetes.io/basic-auth` | Username and password |

---

## Creating Secrets

### Opaque Secret (YAML)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
type: Opaque
stringData:           # use stringData — Kubernetes base64-encodes it automatically
  db_password: "super-secret-password"
  api_key: "abc123xyz"
```

> Use `stringData` (plain text) instead of `data` (manual base64). Both result in the same stored value.

### From CLI

```bash
# Opaque secret
kubectl create secret generic app-secrets \
  --from-literal=db_password=super-secret-password \
  --from-literal=api_key=abc123xyz \
  -n production

# From a file (key = filename)
kubectl create secret generic tls-secret \
  --from-file=tls.crt=./cert.pem \
  --from-file=tls.key=./key.pem \
  -n production
```

### TLS Secret

```bash
kubectl create secret tls my-tls-secret \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key \
  -n production
```

### Docker Registry Secret

```bash
kubectl create secret docker-registry my-registry-secret \
  --docker-server=my-registry.example.com \
  --docker-username=my-user \
  --docker-password=my-password \
  --docker-email=user@example.com \
  -n production
```

---

## Using Secrets in Pods

### As Environment Variable (single key)

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: app-secrets
        key: db_password
```

### All Keys as Environment Variables

```yaml
envFrom:
  - secretRef:
      name: app-secrets
```

### As Volume (files)

```yaml
volumes:
  - name: secrets-vol
    secret:
      secretName: app-secrets
      defaultMode: 0400     # read-only for owner

volumeMounts:
  - name: secrets-vol
    mountPath: /etc/secrets
    readOnly: true
```

Each key becomes a file: `/etc/secrets/db_password`, `/etc/secrets/api_key`.

### Mount a Specific Key

```yaml
volumes:
  - name: tls-vol
    secret:
      secretName: my-tls-secret
      items:
        - key: tls.crt
          path: tls.crt
        - key: tls.key
          path: tls.key
          mode: 0400

volumeMounts:
  - name: tls-vol
    mountPath: /etc/tls
    readOnly: true
```

---

## Private Registry Pull Secret

```yaml
spec:
  imagePullSecrets:
    - name: my-registry-secret
  containers:
    - name: app
      image: my-private-registry.com/my-app:1.0
```

Attach pull secrets to a ServiceAccount to avoid specifying per-Pod:

```bash
kubectl patch serviceaccount default \
  -p '{"imagePullSecrets": [{"name": "my-registry-secret"}]}' \
  -n production
```

---

## Encryption at Rest

By default, Secrets in etcd are **only base64-encoded** — any user with etcd access can read them. Enable encryption at rest on the API server.

### Check if Encryption is Enabled (EKS)

On EKS, enable envelope encryption via KMS at cluster creation or update:

```bash
aws eks describe-cluster --name my-cluster \
  --query 'cluster.encryptionConfig'
```

Enable via AWS Console: Cluster → Configuration → Secrets encryption → Enable with a KMS key.

### Encryption at Rest (self-managed)

```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-32-byte-key>
      - identity: {}    # fallback — allows reading pre-existing unencrypted secrets
```

---

## External Secret Management (Production Recommendation)

Storing secrets in Kubernetes Secrets (even encrypted at rest) has limitations. For production, use an external secret manager:

### AWS Secrets Manager + External Secrets Operator

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secrets
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: ClusterSecretStore
  target:
    name: app-secrets        # creates a Kubernetes Secret with this name
    creationPolicy: Owner
  data:
    - secretKey: db_password
      remoteRef:
        key: production/app/secrets
        property: db_password
```

### AWS Secrets Manager + ASCP (AWS Secrets and Config Provider)

Mounts secrets directly from AWS Secrets Manager as files, without storing them in etcd at all.

---

## Immutable Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets-v1
immutable: true
type: Opaque
stringData:
  api_key: "abc123"
```

---

## Security Best Practices

- Enable etcd encryption at rest (KMS on EKS)
- Use RBAC to restrict `get`/`list` on Secrets to only the ServiceAccounts that need them
- Prefer volume mounts over env vars — env vars can be leaked via process listings and crash dumps
- Use immutable Secrets for values that should not change
- Rotate secrets regularly; consider External Secrets Operator with short `refreshInterval`
- Never commit Secret YAML with real values to git — use Sealed Secrets or external managers
- Audit Secret access: enable Kubernetes audit logging

```bash
# List who has access to secrets in a namespace
kubectl auth can-i get secrets --as=system:serviceaccount:production:my-app -n production
```
