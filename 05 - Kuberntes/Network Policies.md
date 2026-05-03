# Network Policies

## What is a Network Policy?
A NetworkPolicy is a Kubernetes resource that controls traffic flow at the IP/port level between Pods, namespaces, or external endpoints. By default, all Pods can communicate with each other freely — NetworkPolicies restrict that.

- Enforced by the **CNI plugin** (e.g., Calico, Cilium, Weave) — not by Kubernetes itself
- Without a CNI that supports NetworkPolicy, rules are ignored
- Policies are **additive** — multiple policies combine with OR logic
- A Pod with no NetworkPolicy applied has **no restrictions**
- A Pod with at least one NetworkPolicy is restricted to only what's allowed

---

## Policy Selectors

| Selector | Purpose |
|---|---|
| `podSelector` | Target Pods within the same namespace |
| `namespaceSelector` | Allow traffic from/to specific namespaces |
| `ipBlock` | Allow traffic from specific CIDR ranges |

---

## Ingress vs Egress

- **Ingress**: controls incoming traffic TO the selected pods
- **Egress**: controls outgoing traffic FROM the selected pods
- You must explicitly specify `policyTypes` or both are implied by what you define

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

---

## Common Patterns

### Deny All Ingress (default deny)
```yaml
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

### Deny All Egress
```yaml
spec:
  podSelector: {}
  policyTypes:
    - Egress
```

### Allow only from same namespace
```yaml
ingress:
  - from:
      - podSelector: {}
```

### Allow from specific namespace
```yaml
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: monitoring
```

### Allow DNS egress (required for most apps)
```yaml
egress:
  - ports:
      - protocol: UDP
        port: 53
      - protocol: TCP
        port: 53
```

---

## Best Practices

- Start with **default deny all** in each namespace, then open only what's needed
- Always allow **DNS egress** (port 53) or apps will break
- Label namespaces with `kubernetes.io/metadata.name` to use `namespaceSelector`
- Use Calico or Cilium for full NetworkPolicy support
- Test policies with `kubectl exec` + `curl` or `nc` to verify traffic is blocked/allowed
- Keep policies in version control alongside your app manifests

---

## Troubleshooting

```bash
# Check if a NetworkPolicy exists in a namespace
kubectl get networkpolicy -n <namespace>

# Describe a policy
kubectl describe networkpolicy <name> -n <namespace>

# Test connectivity from a pod
kubectl exec -it <pod> -- curl http://<target-service>:<port>

# Test blocked connection
kubectl exec -it <pod> -- nc -zv <target-ip> <port>
```
