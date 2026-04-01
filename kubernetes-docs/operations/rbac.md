# RBAC — Roles, ClusterRoles & ServiceAccounts

Role-Based Access Control (RBAC) controls **who can do what** to which Kubernetes resources. It's the primary authorization mechanism in Kubernetes.

---

## RBAC Objects

| Object | Scope | Description |
|---|---|---|
| `Role` | Namespace | Grants permissions within one namespace |
| `ClusterRole` | Cluster | Grants permissions cluster-wide or reusable across namespaces |
| `RoleBinding` | Namespace | Binds a Role or ClusterRole to subjects in a namespace |
| `ClusterRoleBinding` | Cluster | Binds a ClusterRole to subjects cluster-wide |

---

## Role (namespace-scoped)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
  - apiGroups: [""]          # "" = core API group (pods, services, secrets, etc.)
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]

  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "update", "patch"]
```

### Common Verbs

| Verb | HTTP Method | Description |
|---|---|---|
| `get` | GET | Read a specific resource |
| `list` | GET | List resources |
| `watch` | GET (stream) | Watch for changes |
| `create` | POST | Create a resource |
| `update` | PUT | Replace a resource |
| `patch` | PATCH | Partially update a resource |
| `delete` | DELETE | Delete a resource |
| `deletecollection` | DELETE | Delete a collection |

### Common API Groups

| Group | Resources |
|---|---|
| `""` (core) | `pods`, `services`, `secrets`, `configmaps`, `namespaces`, `nodes`, `persistentvolumes` |
| `apps` | `deployments`, `replicasets`, `statefulsets`, `daemonsets` |
| `autoscaling` | `horizontalpodautoscalers` |
| `batch` | `jobs`, `cronjobs` |
| `networking.k8s.io` | `ingresses`, `networkpolicies` |
| `rbac.authorization.k8s.io` | `roles`, `rolebindings`, `clusterroles`, `clusterrolebindings` |
| `storage.k8s.io` | `storageclasses`, `persistentvolumes` |

---

## ClusterRole (cluster-scoped)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: deployment-manager
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  - apiGroups: [""]
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list", "watch"]
```

---

## RoleBinding

Binds a Role (or ClusterRole) to users, groups, or ServiceAccounts **within a namespace**.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deploy-manager-binding
  namespace: production
subjects:
  # Bind to a ServiceAccount
  - kind: ServiceAccount
    name: ci-deploy
    namespace: production

  # Bind to a user
  - kind: User
    name: "jane@example.com"
    apiGroup: rbac.authorization.k8s.io

  # Bind to a group (e.g., from OIDC provider)
  - kind: Group
    name: "devops-team"
    apiGroup: rbac.authorization.k8s.io

roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole          # can reference either Role or ClusterRole
  name: deployment-manager
```

---

## ClusterRoleBinding

Binds a ClusterRole to subjects cluster-wide.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-binding
subjects:
  - kind: User
    name: "admin@example.com"
    apiGroup: rbac.authorization.k8s.io
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
```

---

## ServiceAccounts

A ServiceAccount is an identity for **processes running in Pods**. Every Pod runs as a ServiceAccount (default: `default` in the namespace).

### Create a ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ci-deploy
  namespace: production
automountServiceAccountToken: false   # disable auto-mounting if not needed
```

### Assign to a Pod

```yaml
spec:
  serviceAccountName: ci-deploy
  automountServiceAccountToken: true   # only if the app needs API access
```

### ServiceAccount Token (auto-mounted)

When `automountServiceAccountToken: true`, Kubernetes mounts a token at:
```
/var/run/secrets/kubernetes.io/serviceaccount/token
```

The app uses this token to authenticate to the Kubernetes API.

---

## Built-in ClusterRoles

| ClusterRole | Description |
|---|---|
| `cluster-admin` | Full access to everything — use sparingly |
| `admin` | Full access within a namespace |
| `edit` | Read/write most resources in a namespace, no RBAC changes |
| `view` | Read-only access to most resources in a namespace |

```bash
# Grant a user read-only access to production namespace
kubectl create rolebinding view-production \
  --clusterrole=view \
  --user=jane@example.com \
  --namespace=production
```

---

## CI/CD ServiceAccount Pattern

Minimal permissions for a CI pipeline to deploy to a namespace.

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ci-deploy
  namespace: production

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ci-deploy-role
  namespace: production
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "update", "patch"]
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "create", "update", "patch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-deploy-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: ci-deploy
    namespace: production
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ci-deploy-role
```

---

## RBAC Commands

```bash
# Check what a user can do
kubectl auth can-i list pods --as=jane@example.com -n production
kubectl auth can-i '*' '*' --as=jane@example.com    # check cluster-admin

# Check what a ServiceAccount can do
kubectl auth can-i get secrets \
  --as=system:serviceaccount:production:ci-deploy \
  -n production

# List all permissions in a namespace (verbose)
kubectl auth can-i --list -n production

# Describe a Role
kubectl describe role pod-reader -n production
kubectl describe clusterrole deployment-manager

# List RoleBindings in a namespace
kubectl get rolebindings -n production

# Audit: find who has access to secrets
kubectl get rolebindings,clusterrolebindings --all-namespaces \
  -o json | jq '.items[] | select(.roleRef.name == "admin") | .subjects'
```

---

## Common Mistakes

| Mistake | Problem | Fix |
|---|---|---|
| Using `cluster-admin` for CI/CD | Blast radius if token is compromised | Minimal Role with only needed verbs |
| ClusterRoleBinding instead of RoleBinding | Grants cluster-wide access instead of namespace-scoped | Use RoleBinding with ClusterRole |
| `automountServiceAccountToken: true` when not needed | Token exposed unnecessarily | Set to `false` on ServiceAccount or Pod spec |
| Using `default` ServiceAccount for apps | Default SA may accumulate permissions | Create dedicated ServiceAccount per app |
