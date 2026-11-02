# Kubernetes Best Practices

## Resource Management

- Set realistic resource requests for workloads where scheduling and capacity planning matter. Limits are workload-dependent; do not assume every container needs both CPU and memory limits. HPA utilization targets require relevant resource requests.
- Use `LimitRange` objects to enforce defaults at namespace level
- Use `ResourceQuota` to cap total CPU/memory per namespace
- Set memory `requests == limits` to avoid OOMKill surprises on memory-constrained nodes

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - default:
        cpu: 500m
        memory: 256Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      type: Container
```

---

## Health Probes

- Configure the probes that match the application's behavior. Readiness is useful for traffic gating; liveness is useful for detecting unrecoverable process states; startup probes help slow-starting applications. Not every container needs all three.
- Use `readinessProbe` to gate traffic — a Pod not ready won't receive Service traffic
- Use `startupProbe` for slow-starting apps to avoid liveness killing them before they're ready
- Prefer HTTP probes over exec probes for performance

---

## High Availability

- Choose replica count based on availability, workload, cost, and failure-domain requirements; two replicas alone do not guarantee high availability.
- Use **PodDisruptionBudgets** to maintain availability during node drain/upgrades
- Use **Pod Anti-Affinity** to spread replicas across nodes and AZs
- Use **topology spread constraints** for fine-grained distribution control

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: web
```

---

## Security

- Prefer non-root containers and set `runAsNonRoot: true` when the image supports it. Some workloads require specific privileges; document and constrain exceptions.
- Use `readOnlyRootFilesystem: true` wherever possible
- Drop all Linux capabilities and add back only what's needed
- Use NetworkPolicies where the CNI/network implementation enforces them; validate DNS, control-plane, monitoring, and application flows before applying default-deny.
- Use **RBAC** with least privilege — avoid ClusterAdmin for workloads
- Store secrets in external systems (AWS Secrets Manager, Vault) rather than Kubernetes Secrets alone
- Scan images for vulnerabilities before deploying

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
```

---

## Deployments and Rollouts

- Use **RollingUpdate** strategy with sensible `maxSurge` and `maxUnavailable`
- Set `revisionHistoryLimit` to keep rollback history manageable (default is 10)
- Use **image tags** — never use `latest` in production
- Pin image digests for fully reproducible deployments
- Use `kubectl rollout status` to confirm a deployment succeeds before marking CI green

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

---

## Namespaces and Multi-tenancy

- Separate environments into different namespaces (or clusters for strict isolation)
- Apply `ResourceQuota` and `LimitRange` per namespace
- Apply `NetworkPolicy` default-deny per namespace
- Use RBAC to scope team permissions to their own namespaces

---

## Observability

- Ship **structured logs** (JSON) from all containers
- Export metrics to **Prometheus** — at minimum: CPU, memory, request rate, error rate, latency
- Use **liveness/readiness probe failures** as alerting signals
- Enable **audit logging** on the API server in production
- Use `kubectl top pods/nodes` for quick resource checks

---

## Storage

- Use **PersistentVolumeClaims** — never hardcode hostPath volumes in production
- Use **StorageClasses** with appropriate reclaim policies (`Retain` for production data)
- Choose access modes based on the storage driver's documented capabilities. `ReadWriteOnce` is node-scoped, not necessarily single-Pod; `ReadWriteMany` requires backend support.
- Back up persistent volumes regularly with tools like Velero

---

## Labels and Annotations

Always apply consistent labels for selection, monitoring, and filtering:

```yaml
labels:
  app: web
  version: v1.2.3
  environment: production
  team: platform
  managed-by: helm
```

---

## General Checklist

- [ ] Resource requests and limits set
- [ ] Appropriate health probes configured
- [ ] Non-root security context where supported
- [ ] NetworkPolicy applied
- [ ] PodDisruptionBudget set
- [ ] Image tag pinned (no `latest`)
- [ ] RBAC scoped to minimum required
- [ ] Secrets from external secret store
- [ ] Structured logging enabled
- [ ] HPA configured for variable-load services
