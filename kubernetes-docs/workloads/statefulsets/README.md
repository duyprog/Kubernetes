# StatefulSets — Ordered Deployment & Stable Identity

A StatefulSet manages stateful applications — workloads that need **stable network identities**, **persistent storage**, and **ordered deployment/scaling**.

---

## StatefulSet vs Deployment

| Feature | StatefulSet | Deployment |
|---|---|---|
| Pod names | Stable, predictable (`pod-0`, `pod-1`) | Random suffix (`pod-7d9f8-abc`) |
| Network identity | Stable hostname per Pod | Changes on reschedule |
| Storage | Dedicated PVC per Pod (retained on delete) | Shared or no persistent storage |
| Deployment order | Sequential (0, 1, 2, ...) | Parallel |
| Scale-down order | Reverse sequential (N, N-1, ...) | Random |
| Rolling update | One Pod at a time, ordered | Configurable via strategy |

---

## When to Use StatefulSets

- **Databases:** PostgreSQL, MySQL, MongoDB
- **Distributed coordination:** ZooKeeper, etcd, Consul
- **Message brokers:** Kafka, RabbitMQ
- **Search engines:** Elasticsearch
- **Caches with persistence:** Redis Sentinel, Redis Cluster

---

## StatefulSet Spec

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: "postgres"    # must match a Headless Service
  replicas: 3
  podManagementPolicy: OrderedReady   # or Parallel

  selector:
    matchLabels:
      app: postgres

  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0    # only update Pods with index >= partition (canary updates)

  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              value: "mydb"
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: username
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            - name: PGDATA
              value: "/var/lib/postgresql/data/pgdata"
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "1"
              memory: "1Gi"
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data

  # PVC template — creates a dedicated PVC for each Pod
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp3
        resources:
          requests:
            storage: 20Gi
```

---

## Headless Service

StatefulSets **require a Headless Service** — a Service with `clusterIP: None`. This creates stable DNS records for each Pod.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: production
  labels:
    app: postgres
spec:
  clusterIP: None       # headless
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

### DNS Records Created

```
# Pod DNS pattern
<pod-name>.<service-name>.<namespace>.svc.cluster.local

# For a 3-replica StatefulSet named "postgres" in "production":
postgres-0.postgres.production.svc.cluster.local
postgres-1.postgres.production.svc.cluster.local
postgres-2.postgres.production.svc.cluster.local
```

This stable DNS allows replicas to find each other for cluster formation (Kafka broker list, Postgres replication, etc.).

---

## Pod Management Policies

### OrderedReady (default)

- Scale up: 0 → 1 → 2 (each waits for previous to be Ready)
- Scale down: 2 → 1 → 0 (reverse order)
- Updates: one at a time, highest index first

### Parallel

- All Pods created/deleted simultaneously
- Faster but no ordering guarantee
- Suitable for: independent replicas (sharded systems)

```yaml
podManagementPolicy: Parallel
```

---

## Canary Updates with `partition`

Update only Pods with index >= partition. Useful for gradual rollouts.

```yaml
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    partition: 2    # only pod-2 gets new version; pod-0 and pod-1 stay on old version
```

```bash
# Validate pod-2 with new version, then set partition: 1, then 0
kubectl patch statefulset postgres \
  -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":1}}}}' \
  -n production
```

---

## Commands

```bash
# View StatefulSet status
kubectl get statefulset -n production
kubectl describe statefulset postgres -n production

# View Pods with their stable names
kubectl get pods -n production -l app=postgres

# View PVCs created by volumeClaimTemplates
kubectl get pvc -n production -l app=postgres

# Scale
kubectl scale statefulset postgres --replicas=5 -n production

# Rollout status
kubectl rollout status statefulset/postgres -n production
```

---

## Examples

- [postgres-statefulset.yaml](examples/postgres-statefulset.yaml)

> For persistent storage details, see [persistent-storage.md](persistent-storage.md).
