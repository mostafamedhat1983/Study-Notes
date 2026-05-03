---
tags:
  - Kubernetes
---
Namespaces provide a way to divide a single Kubernetes cluster into multiple virtual clusters.

---

## Main idea

- namespaces are a logical boundary, not a physical one
- resources in different namespaces are isolated by name but not by network (by default)
- most Kubernetes resources are namespace-scoped
- some resources are cluster-scoped (nodes, PersistentVolumes, ClusterRoles, StorageClasses)

---

## Default namespaces

| Namespace | Purpose |
|---|---|
| `default` | where resources go if no namespace is specified |
| `kube-system` | Kubernetes system components (coredns, kube-proxy, metrics-server) |
| `kube-public` | publicly readable, rarely used |
| `kube-node-lease` | node heartbeat lease objects |

---

## Creating a namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
```

```bash
kubectl create namespace staging
```

---

## Using namespaces

```bash
# List all namespaces
kubectl get namespaces

# List pods in a specific namespace
kubectl get pods -n production

# List pods across all namespaces
kubectl get pods -A

# Apply a manifest to a specific namespace
kubectl apply -f deployment.yaml -n staging

# Delete a namespace (and everything inside it)
kubectl delete namespace staging
```

---

## Setting default namespace for kubectl context

```bash
kubectl config set-context --current --namespace=production
```

After this, all kubectl commands default to the `production` namespace.

---

## Resource Quotas

Limit total resource usage within a namespace.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
spec:
  hard:
    pods: "20"
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
```

---

## LimitRange

Sets default and max resource requests/limits for Pods in a namespace.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: development
spec:
  limits:
    - type: Container
      default:
        cpu: "200m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
```

---

## Common namespace patterns in production

```
namespaces/
  default         → avoid using in production
  kube-system     → cluster components
  monitoring      → Prometheus, Grafana
  logging         → Fluentd, Loki
  ingress-nginx   → Ingress controller
  cert-manager    → TLS cert automation
  production      → production workloads
  staging         → staging workloads
  development     → dev workloads
```

---

## Cross-namespace communication

Pods in different namespaces can communicate using the full DNS name:

```
<service-name>.<namespace>.svc.cluster.local
```

Example:
```
db.production.svc.cluster.local
```

---

## Common mistakes

- running everything in the `default` namespace in production
- not setting ResourceQuotas (one team can consume all cluster resources)
- confusing namespace isolation with network isolation (not the same by default)
- deleting a namespace without realizing it deletes everything inside it
- forgetting `-n <namespace>` flag in kubectl commands

---

## Good practices

- never use `default` namespace for production workloads
- use separate namespaces per environment (dev, staging, prod)
- use separate namespaces per team or application
- always set ResourceQuotas and LimitRanges per namespace
- use RBAC to limit who can access each namespace
- use Network Policies to restrict cross-namespace traffic

---

## Must memorize

```bash
kubectl get pods -n <namespace>
kubectl get pods -A
kubectl config set-context --current --namespace=<namespace>
kubectl apply -f file.yaml -n <namespace>
```

```
Cross-namespace DNS: <service>.<namespace>.svc.cluster.local
```

---

## Key ideas

- Namespaces are logical boundaries inside one cluster.
- They isolate names, RBAC, and quotas — but not network traffic by default.
- Use ResourceQuotas to prevent one team from consuming all cluster resources.
- Use LimitRange to set default resource limits per namespace.
- Use Network Policies to add network isolation between namespaces.
