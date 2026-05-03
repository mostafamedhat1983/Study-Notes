# HPA, VPA and Cluster Autoscaler

Kubernetes has three complementary autoscaling mechanisms operating at different levels.

---

## HPA — Horizontal Pod Autoscaler

Scales the **number of Pod replicas** up or down based on observed metrics.

- Default metric: CPU utilization
- Also supports memory, custom metrics (via Metrics Server or Prometheus Adapter)
- Acts on Deployments, StatefulSets, ReplicaSets

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
```

```bash
# Create HPA imperatively
kubectl autoscale deployment web --cpu-percent=60 --min=2 --max=10

# Check HPA status
kubectl get hpa
kubectl describe hpa web-hpa
```

### Requirements
- **Metrics Server** must be installed in the cluster
- Pods must have **resource requests** defined (CPU/memory)

### Behaviour
- Scale-up: fast (default 15s evaluation)
- Scale-down: conservative (default 5 min stabilization window) to avoid flapping

---

## VPA — Vertical Pod Autoscaler

Automatically adjusts **CPU and memory requests/limits** for containers based on actual usage.

- Does NOT change replica count
- Requires Pod restart to apply new resource values (unless using in-place updates — alpha)
- Useful for workloads with unpredictable or varying resource needs

### VPA Modes

| Mode | Behaviour |
|---|---|
| `Off` | Only recommends, no changes applied |
| `Initial` | Sets resources at Pod creation only |
| `Auto` | Applies recommendations, restarts Pods as needed |
| `Recreate` | Same as Auto but always evicts Pods |

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  updatePolicy:
    updateMode: "Auto"
```

> ⚠️ Do NOT use HPA (CPU/memory) and VPA (Auto) together on the same Deployment — they conflict. Use HPA for scaling replicas + VPA in `Off` mode for recommendations only.

---

## Cluster Autoscaler

Scales the **number of nodes** in the cluster up or down based on pending Pods and underutilized nodes.

- Adds nodes when Pods are `Pending` due to insufficient resources
- Removes nodes when they are underutilized for a sustained period (default 10 min)
- Cloud-provider specific: AWS (EKS managed node groups), GKE, AKS all supported

### How it works
1. Pod is `Pending` — no node has enough resources
2. Cluster Autoscaler detects this and triggers node group scale-up
3. New node joins, Pod gets scheduled
4. When nodes are underutilized, CA evicts Pods and removes the node

### Key Annotations/Flags

```yaml
# Prevent a Pod from being evicted during scale-down
annotations:
  cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
```

```yaml
# PodDisruptionBudget — protect minimum replicas during scale-down
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web
```

### On EKS
- Use **Managed Node Groups** or **Karpenter** (preferred modern alternative)
- Karpenter is faster and more flexible than Cluster Autoscaler on AWS

---

## Comparison

| Feature | HPA | VPA | Cluster Autoscaler |
|---|---|---|---|
| Scales | Pod replicas | Pod resources | Nodes |
| Trigger | Metrics (CPU/memory) | Resource usage | Pending Pods / idle nodes |
| Requires restart | ❌ | ✅ (usually) | ❌ |
| Use together | ✅ with CA | ⚠️ not with HPA | ✅ with HPA |

---

## Best Practices

- Always define `requests` on all containers — HPA and CA depend on them
- Use HPA + Cluster Autoscaler together for full horizontal scaling
- Use VPA in `Off` mode to get right-sizing recommendations without restarts
- Set `minReplicas >= 2` on HPA for production availability
- Use PodDisruptionBudgets to protect apps during CA scale-down
- On AWS EKS, evaluate **Karpenter** as a replacement for Cluster Autoscaler
