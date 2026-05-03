---
tags:
  - Kubernetes
---
A StatefulSet manages stateful applications that need stable network identity, ordered deployment, and persistent storage per Pod.

---

## Main idea

- Deployments are for stateless apps — any Pod is identical and interchangeable
- StatefulSets are for stateful apps — each Pod has a stable, unique identity
- StatefulSet Pods have: stable DNS names, stable storage, ordered startup and shutdown

---

## StatefulSet vs Deployment

| Feature | Deployment | StatefulSet |
|---|---|---|
| Pod identity | random names (e.g. app-xyz12) | stable ordered names (e.g. db-0, db-1) |
| Storage | shared or none | each Pod gets its own PVC |
| Startup order | all at once | ordered (0, then 1, then 2) |
| Shutdown order | all at once | reverse order (2, then 1, then 0) |
| DNS name | shared Service name | stable per-Pod DNS name |
| Use case | stateless apps | databases, queues, distributed systems |

---

## StatefulSet manifest

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: "postgres"       # must match a Headless Service
  replicas: 3
  selector:
    matchLabels:
      app: postgres
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
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

---

## Headless Service (required for StatefulSets)

StatefulSets require a Headless Service to give each Pod a stable DNS name.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None     # this makes it headless
  selector:
    app: postgres
  ports:
    - port: 5432
```

Each Pod gets a DNS name:
```
postgres-0.postgres.default.svc.cluster.local
postgres-1.postgres.default.svc.cluster.local
postgres-2.postgres.default.svc.cluster.local
```

---

## volumeClaimTemplates

- each Pod gets its own PVC automatically
- PVCs are named `<volume-name>-<pod-name>`
- PVCs are NOT deleted when the StatefulSet is deleted (data protection)
- you must manually delete PVCs to free storage

---

## Common StatefulSet commands

```bash
kubectl get statefulsets
kubectl describe statefulset <name>
kubectl get pvc                      # view per-Pod PVCs
kubectl delete pod postgres-1        # Pod restarts with same identity
```

---

## Common use cases

- databases: PostgreSQL, MySQL, MongoDB
- message queues: Kafka, RabbitMQ
- distributed caches: Redis Cluster
- distributed storage: Elasticsearch, Cassandra
- Zookeeper

---

## Common mistakes

- using a Deployment for a database (Pods have no stable identity or dedicated storage)
- forgetting the Headless Service (StatefulSet Pods will not get stable DNS names)
- deleting a StatefulSet and expecting PVCs to be cleaned up (they are not)
- not using init containers to handle first-run database initialization
- updating a StatefulSet without understanding rolling update behavior for stateful apps

---

## Good practices

- always pair a StatefulSet with a Headless Service
- always use `volumeClaimTemplates` for per-Pod storage
- use `podManagementPolicy: Parallel` only if your app handles parallel startup safely
- back up PVC data before deleting a StatefulSet
- use readiness probes to prevent traffic before the database is ready
- use init containers for one-time database setup tasks

---

## Must memorize

```
StatefulSet Pod names: <name>-0, <name>-1, <name>-2
DNS: <pod>.<service>.<namespace>.svc.cluster.local
PVCs: created per Pod, NOT deleted with the StatefulSet
Requires: Headless Service (clusterIP: None)
```

```bash
kubectl get statefulsets
kubectl get pvc
```

---

## Key ideas

- StatefulSets give Pods stable names, stable DNS, and stable storage.
- Each Pod gets its own PVC via volumeClaimTemplates.
- Pods start in order (0 → 1 → 2) and stop in reverse (2 → 1 → 0).
- A Headless Service is required for stable per-Pod DNS.
- PVCs are not deleted when the StatefulSet is deleted — clean them up manually.
