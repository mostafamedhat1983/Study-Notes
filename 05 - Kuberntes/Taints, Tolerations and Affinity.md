# Taints, Tolerations and Affinity

These are Kubernetes scheduling controls that determine **which nodes a Pod can or must run on**.

---

## Taints and Tolerations

### Taints (on Nodes)
A taint marks a node so that Pods **will not be scheduled** there unless they explicitly tolerate it.

```bash
# Add a taint
kubectl taint nodes <node-name> key=value:Effect

# Remove a taint
kubectl taint nodes <node-name> key=value:Effect-

# Example: dedicated GPU node
kubectl taint nodes gpu-node gpu=true:NoSchedule
```

### Taint Effects

| Effect | Behaviour |
|---|---|
| `NoSchedule` | New Pods without toleration won't be scheduled |
| `PreferNoSchedule` | Scheduler tries to avoid the node, but not guaranteed |
| `NoExecute` | New Pods won't schedule AND existing Pods are evicted |

### Tolerations (on Pods)
A toleration allows a Pod to be scheduled on a tainted node.

```yaml
spec:
  tolerations:
    - key: "gpu"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
```

```yaml
# Tolerate any taint with a specific key
spec:
  tolerations:
    - key: "dedicated"
      operator: "Exists"
      effect: "NoSchedule"
```

---

## Node Affinity

Node Affinity is a more expressive replacement for `nodeSelector`. It controls which nodes a Pod **prefers or requires**.

### Types

| Type | Behaviour |
|---|---|
| `requiredDuringSchedulingIgnoredDuringExecution` | Hard rule — Pod won't schedule if no match |
| `preferredDuringSchedulingIgnoredDuringExecution` | Soft rule — scheduler prefers but doesn't require |

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                  - us-east-1a
                  - us-east-1b
```

### Operators
`In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`

---

## Pod Affinity and Anti-Affinity

Controls scheduling relative to **other Pods**, not nodes.

### Pod Affinity — schedule near certain Pods
```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: cache
          topologyKey: kubernetes.io/hostname
```

### Pod Anti-Affinity — spread Pods across nodes
```yaml
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: web
            topologyKey: kubernetes.io/hostname
```

> `topologyKey` defines the scope — `kubernetes.io/hostname` means per node, `topology.kubernetes.io/zone` means per AZ.

---

## nodeSelector (simple version)

```yaml
spec:
  nodeSelector:
    disktype: ssd
```
Simpler than Node Affinity but less expressive — no operators, no preferences.

---

## Comparison

| Feature | nodeSelector | Node Affinity | Taints/Tolerations |
|---|---|---|---|
| Direction | Pod → Node | Pod → Node | Node → Pod |
| Expressiveness | Low | High | Medium |
| Soft rules | ❌ | ✅ | ❌ |
| Repel Pods | ❌ | ❌ | ✅ |

---

## Common Use Cases

- **Dedicated nodes**: Taint GPU/spot nodes, only GPU workloads tolerate them
- **Spread replicas**: Pod anti-affinity to avoid all replicas on same node
- **Zone pinning**: Node affinity to pin Pods to specific AZ
- **System Pods**: `node-role.kubernetes.io/control-plane:NoSchedule` keeps workloads off control plane

---

## Useful Commands

```bash
# View node labels
kubectl get nodes --show-labels

# View node taints
kubectl describe node <node-name> | grep Taints

# Add label to node
kubectl label node <node-name> disktype=ssd

# Check why a Pod is pending
kubectl describe pod <pod-name> | grep -A10 Events
```
