# Kubernetes Best Practices

## Resource Management

- **Always set `requests` and `limits`** on every container — without requests, the scheduler cannot make good decisions and HPA/CA won't work properly
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

- Define **all three probes** on every production container: `livenessProbe`, `readinessProbe`, `startupProbe`
- Use `readinessProbe` to gate traffic — a Pod not ready won't receive Service traffic
- Use `startupProbe` for slow-starting apps to avoid liveness killing them before they're ready
- Prefer HTTP probes over exec probes for performance

---

## High Availability

- Run **minimum 2 replicas** for any production Deployment
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

- **Never run containers as root** — set `runAsNonRoot: true` and a specific `runAsUser`
- Use `readOnlyRootFilesystem: true` wherever possible
- Drop all Linux capabilities and add back only what's needed
- Use **NetworkPolicies** with default-deny and explicit allow rules
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
- Use `ReadWriteOnce` for single-node workloads, `ReadWriteMany` (NFS/EFS) for shared data
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
- [ ] All three probes configured
- [ ] Non-root security context
- [ ] NetworkPolicy applied
- [ ] PodDisruptionBudget set
- [ ] Image tag pinned (no `latest`)
- [ ] RBAC scoped to minimum required
- [ ] Secrets from external secret store
- [ ] Structured logging enabled
- [ ] HPA configured for variable-load services
