# Ingress — HTTP Routing & TLS Termination

Ingress manages **external HTTP/HTTPS access** to Services. It provides routing rules (host-based, path-based), TLS termination, and load balancing — without creating a cloud LB per Service.

---

## Ingress vs Service LoadBalancer

| | Ingress | Service LoadBalancer |
|---|---|---|
| Layer | L7 (HTTP/HTTPS) | L4 (TCP/UDP) |
| Routing | Host + path rules | IP + port only |
| TLS termination | Yes | With NLB SSL passthrough |
| Cost | One LB for all services | One LB per Service |
| Use case | HTTP/HTTPS apps | TCP/UDP, non-HTTP |

---

## Ingress Spec

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  namespace: production
  annotations:
    # Controller-specific (nginx example)
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx    # which Ingress controller handles this

  # TLS configuration
  tls:
    - hosts:
        - app.example.com
      secretName: app-tls-secret    # kubernetes.io/tls Secret

  rules:
    # Host-based routing
    - host: app.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80

    - host: admin.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin-service
                port:
                  number: 80
```

### Path Types

| Type | Behavior |
|---|---|
| `Exact` | Exact match only: `/foo` matches `/foo` but not `/foo/` |
| `Prefix` | Prefix match: `/foo` matches `/foo`, `/foo/`, `/foo/bar` |
| `ImplementationSpecific` | Behavior depends on the Ingress controller |

---

## TLS Configuration

### Using cert-manager (recommended)

cert-manager automatically issues and renews TLS certificates.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.example.com
      secretName: app-tls-cert    # cert-manager creates and manages this Secret
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
```

### Manual TLS Secret

```bash
kubectl create secret tls app-tls-secret \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key \
  -n production
```

---

## Ingress Controllers

An Ingress resource is just a spec — it does nothing without an **Ingress controller** deployed in the cluster.

| Controller | Type | Best For |
|---|---|---|
| AWS Load Balancer Controller | ALB (L7) | EKS — AWS-native, IAM integration |
| ingress-nginx | NGINX | Self-managed, on-prem, any cloud |
| Traefik | Traefik | Dynamic config, middleware ecosystem |
| Kong | Kong Gateway | API gateway features |
| HAProxy | HAProxy | High performance, complex routing |

---

## AWS ALB Ingress (EKS — Recommended)

The AWS Load Balancer Controller creates an ALB per Ingress (or one shared ALB with IngressGroup).

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:region:account:certificate/id
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    # Group multiple Ingresses onto one ALB
    alb.ingress.kubernetes.io/group.name: production
    alb.ingress.kubernetes.io/group.order: "10"
    # Health check
    alb.ingress.kubernetes.io/healthcheck-path: /healthz
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "15"
spec:
  ingressClassName: alb
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
```

### Install AWS Load Balancer Controller

```bash
# Create IAM role via IRSA
eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::<account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# Install via Helm
helm repo add eks https://aws.github.io/eks-charts
helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

---

## ingress-nginx (Self-Managed / On-Prem)

```bash
# Install ingress-nginx
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.replicaCount=2 \
  --set controller.service.type=LoadBalancer
```

Common annotations:

```yaml
annotations:
  nginx.ingress.kubernetes.io/ssl-redirect: "true"
  nginx.ingress.kubernetes.io/proxy-body-size: "50m"
  nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
  nginx.ingress.kubernetes.io/rate-limit: "100"
  nginx.ingress.kubernetes.io/limit-connections: "10"
```

---

## Commands

```bash
# List Ingresses
kubectl get ingress -n production

# Describe (check rules, backend, TLS)
kubectl describe ingress my-app-ingress -n production

# Check Ingress controller logs (nginx)
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx --tail=100

# Check if ALB is provisioned
kubectl get ingress -n production -o jsonpath='{.items[*].status.loadBalancer.ingress}'
```
