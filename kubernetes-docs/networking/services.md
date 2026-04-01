# Services — ClusterIP, NodePort & LoadBalancer

A Service provides a **stable network endpoint** for a set of Pods selected by label. Since Pod IPs change when Pods restart, Services act as a permanent, load-balanced address.

---

## Service Types

### ClusterIP (default)

Internal-only IP accessible within the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: my-app          # routes to Pods with this label
  ports:
    - name: http
      port: 80           # Service port (what clients connect to)
      targetPort: 8080   # container port (where app listens)
      protocol: TCP
```

**DNS:** `my-app.production.svc.cluster.local`

**Use for:** Internal service-to-service communication.

### NodePort

Exposes the Service on a static port on **every node's IP**.

```yaml
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080    # 30000–32767; omit for auto-assignment
```

**Access:** `<node-ip>:30080`

**Use for:** Development, external access without a cloud load balancer. Avoid in production — exposes node IPs directly.

### LoadBalancer

Provisions an **external cloud load balancer** (AWS ALB/NLB) and assigns an external IP or hostname.

```yaml
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 443
      targetPort: 8443
  annotations:
    # AWS NLB (recommended over classic ELB)
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
```

**Use for:** Production internet-facing services. Prefer Ingress + ALB for HTTP/HTTPS traffic.

### ExternalName

Maps a Service to an external DNS name. No proxying — returns a CNAME.

```yaml
spec:
  type: ExternalName
  externalName: my-database.us-east-1.rds.amazonaws.com
```

**Use for:** Accessing external services (RDS, ElastiCache) via a Kubernetes-internal name.

### Headless Service

No ClusterIP — returns individual Pod IPs directly. Used by StatefulSets.

```yaml
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
```

---

## Service Discovery

### DNS (recommended)

Kubernetes creates DNS records for every Service.

```
# Full form
<service-name>.<namespace>.svc.cluster.local

# Within same namespace — short form works
<service-name>

# Port-specific (SRV records)
_<port-name>._<protocol>.<service-name>.<namespace>.svc.cluster.local
```

### Environment Variables (legacy)

Kubernetes injects `<SERVICE_NAME>_SERVICE_HOST` and `<SERVICE_NAME>_SERVICE_PORT` into every Pod in the same namespace. Not recommended — order-dependent and clutters env.

---

## Session Affinity

Route requests from the same client to the same Pod.

```yaml
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800   # 3 hours
```

---

## Traffic Routing Details

### `selector` — Label Matching

The Service routes to all Pods matching the selector labels. kube-proxy maintains iptables/IPVS rules to load balance across them.

### `targetPort` — Named Ports

Reference container port by name instead of number — useful when different containers use different port numbers.

```yaml
# In Pod spec
ports:
  - name: http
    containerPort: 8080

# In Service spec
ports:
  - port: 80
    targetPort: http      # name reference
```

### Service Without Selector (manual endpoints)

Route to IPs outside the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  ports:
    - port: 5432

---
apiVersion: v1
kind: Endpoints
metadata:
  name: external-db      # must match Service name
subsets:
  - addresses:
      - ip: 10.0.1.100   # external DB IP
    ports:
      - port: 5432
```

---

## AWS Load Balancer Controller Annotations

For LoadBalancer type Services on EKS:

```yaml
annotations:
  # NLB with IP target mode (recommended)
  service.beta.kubernetes.io/aws-load-balancer-type: "external"
  service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"

  # Internet-facing vs internal
  service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"   # or "internal"

  # Specific subnets
  service.beta.kubernetes.io/aws-load-balancer-subnets: "subnet-abc,subnet-def"

  # TLS termination at NLB
  service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:region:account:certificate/id"
  service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443"

  # Health check
  service.beta.kubernetes.io/aws-load-balancer-healthcheck-path: "/healthz"
  service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval: "10"
```

---

## Commands

```bash
# List Services
kubectl get svc -n production

# Describe a Service (check endpoints)
kubectl describe svc my-app -n production

# Check Endpoints (actual Pod IPs backing the Service)
kubectl get endpoints my-app -n production

# Test DNS resolution from inside a Pod
kubectl exec -it <pod> -n production -- nslookup my-app.production.svc.cluster.local

# Port-forward Service locally for testing
kubectl port-forward svc/my-app 8080:80 -n production
```
