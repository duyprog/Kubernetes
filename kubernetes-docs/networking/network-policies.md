# Network Policies — Pod-Level Firewall Rules

A NetworkPolicy acts as a **firewall** for Pods — it controls which Pods can communicate with which other Pods and external endpoints. Without NetworkPolicies, all Pods in a cluster can communicate freely.

> NetworkPolicies are enforced by the **CNI plugin**, not Kubernetes itself. Your CNI must support them (Calico, Cilium, Weave). Flannel does **not** enforce NetworkPolicies.

---

## Default Behavior

| Scenario | Default |
|---|---|
| No NetworkPolicy on a Pod | All ingress and egress allowed |
| A NetworkPolicy selects a Pod | Only traffic matching the rules is allowed; everything else is denied |

Once a NetworkPolicy selects a Pod, it switches from allow-all to **deny-all + explicit allows**.

---

## NetworkPolicy Spec

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: my-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: my-app       # applies to Pods with this label; {} = all Pods

  policyTypes:
    - Ingress           # control incoming traffic
    - Egress            # control outgoing traffic

  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
        - namespaceSelector:
            matchLabels:
              environment: production
      ports:
        - protocol: TCP
          port: 8080

  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
```

---

## Selectors in Rules

### `podSelector`

Select Pods **within the same namespace**.

```yaml
from:
  - podSelector:
      matchLabels:
        app: frontend
```

### `namespaceSelector`

Select all Pods from matching **namespaces**.

```yaml
from:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: monitoring
```

### Combined `podSelector` + `namespaceSelector` (AND condition)

Both conditions must match — specific Pods in a specific namespace.

```yaml
from:
  - namespaceSelector:
      matchLabels:
        environment: production
    podSelector:
      matchLabels:
        app: frontend
```

> Note: When `namespaceSelector` and `podSelector` are in the **same list item** (not separate `-` items), they are AND'd. Separate `-` items are OR'd.

### `ipBlock`

Allow traffic from specific IP CIDR ranges.

```yaml
from:
  - ipBlock:
      cidr: 10.0.0.0/8
      except:
        - 10.0.1.0/24    # exclude this subnet
```

---

## Common Policy Patterns

### Default Deny All (baseline for namespaces)

Apply this first, then add explicit allow policies.

```yaml
# Deny all ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}         # all Pods
  policyTypes:
    - Ingress

---
# Deny all egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
```

### Allow Only Within Namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector: {}   # any Pod in the same namespace
```

### Allow Specific App to Reach Database

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: my-app
      ports:
        - protocol: TCP
          port: 5432
```

### Allow Egress to DNS and External APIs

Always allow DNS — otherwise Pod DNS resolution breaks.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-dns-and-external
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
    - Egress
  egress:
    # Allow DNS
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53

    # Allow HTTPS to external services
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
      ports:
        - protocol: TCP
          port: 443
```

### Allow Monitoring (Prometheus scraping)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: production
spec:
  podSelector: {}    # all Pods
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
          podSelector:
            matchLabels:
              app: prometheus
      ports:
        - protocol: TCP
          port: 9090
```

---

## Namespace Labels for Policies

For `namespaceSelector` to work, namespaces need labels. Kubernetes automatically adds `kubernetes.io/metadata.name` as of v1.21.

```bash
# Label a namespace for policy targeting
kubectl label namespace monitoring environment=monitoring

# Verify
kubectl get namespace monitoring --show-labels
```

---

## Testing Network Policies

```bash
# Test connectivity from one Pod to another
kubectl exec -it <source-pod> -n production -- \
  nc -zv <target-pod-ip> 5432

# Test DNS
kubectl exec -it <pod> -n production -- \
  nslookup postgres.production.svc.cluster.local

# Test with curl
kubectl exec -it <pod> -n production -- \
  curl -v http://my-service.production.svc.cluster.local/healthz
```

---

## Commands

```bash
# List NetworkPolicies
kubectl get networkpolicy -n production

# Describe a policy (show rules)
kubectl describe networkpolicy allow-app-to-db -n production

# Check if CNI enforces policies (Calico)
kubectl get pods -n kube-system | grep calico
```
