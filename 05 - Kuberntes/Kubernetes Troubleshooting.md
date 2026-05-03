# Kubernetes Troubleshooting

A structured debugging workflow for the most common Kubernetes issues.

---

## General Debugging Approach

1. `kubectl get pods` — find the pod and its status
2. `kubectl describe pod <name>` — read Events section for root cause
3. `kubectl logs <name>` — check application logs
4. `kubectl logs <name> --previous` — check logs of crashed container
5. `kubectl get events --sort-by='.lastTimestamp'` — cluster-wide events

---

## Pod Status Reference

| Status | Meaning |
|---|---|
| `Pending` | Pod not scheduled yet — no node available or resource constraints |
| `ContainerCreating` | Pulling image or mounting volumes |
| `Running` | Container is running |
| `CrashLoopBackOff` | Container keeps crashing and restarting |
| `OOMKilled` | Container exceeded memory limit, killed by kernel |
| `ImagePullBackOff` | Cannot pull the container image |
| `ErrImagePull` | First image pull failure (before backoff) |
| `Terminating` | Pod being deleted (stuck = finalizer issue) |
| `Error` | Container exited with non-zero exit code |
| `Completed` | Container ran and exited successfully (Jobs) |

---

## CrashLoopBackOff

Container starts, crashes, Kubernetes restarts it — repeating cycle.

```bash
# Check current logs
kubectl logs <pod-name>

# Check logs of the previous (crashed) container
kubectl logs <pod-name> --previous

# Check exit code and reason
kubectl describe pod <pod-name> | grep -A10 'Last State'
```

**Common causes:**
- Application error on startup (bad config, missing env var)
- Liveness probe failing too aggressively — container killed before ready
- Missing `startupProbe` for slow-starting apps
- OOMKill (memory limit too low)
- Wrong command/entrypoint in the container

---

## ImagePullBackOff / ErrImagePull

```bash
kubectl describe pod <pod-name> | grep -A5 Events
```

**Common causes and fixes:**

| Cause | Fix |
|---|---|
| Wrong image name or tag | Correct the image reference |
| Private registry, no credentials | Add `imagePullSecrets` to the Pod spec |
| ECR token expired | Refresh ECR credentials / use IRSA |
| Image doesn't exist | Verify image exists in the registry |
| Network issue pulling image | Check node network / proxy settings |

```yaml
# Add image pull secret
spec:
  imagePullSecrets:
    - name: my-registry-secret
```

---

## Pending Pod

```bash
kubectl describe pod <pod-name> | grep -A10 Events
```

**Common causes:**

| Event Message | Cause | Fix |
|---|---|---|
| `Insufficient cpu/memory` | No node has enough resources | Scale cluster or reduce requests |
| `0/3 nodes are available` | Taints, affinity, or no matching nodes | Check node labels and taints |
| `pod has unbound PVC` | PVC not bound to a PV | Check PVC/StorageClass status |
| `node(s) had taint` | Node tainted, pod has no toleration | Add toleration or remove taint |

```bash
# Check node capacity
kubectl describe nodes | grep -A5 'Allocated resources'

# Check PVC status
kubectl get pvc -n <namespace>
```

---

## OOMKilled

The Linux kernel killed the container because it exceeded its memory limit.

```bash
kubectl describe pod <pod-name> | grep -i oom
kubectl describe pod <pod-name> | grep -A5 'Last State'
```

**Fix:**
- Increase memory `limits` in the pod spec
- Use VPA in `Off` mode to get right-sizing recommendations
- Check for memory leaks in the application

---

## Service Not Reachable

```bash
# Check service exists and has correct port
kubectl get svc <service-name> -n <namespace>

# Check endpoints — must not be empty
kubectl get endpoints <service-name> -n <namespace>
# Empty = no pods match the service selector

# Verify pod labels match service selector
kubectl get pods --show-labels -n <namespace>
kubectl describe svc <service-name> -n <namespace> | grep Selector

# Test from inside the cluster
kubectl run test --image=busybox --rm -it -- wget -qO- http://<service>:<port>
```

**Common causes:**
- Pod label doesn't match service `selector`
- Pod not in `Running` state (not added to endpoints)
- Wrong `targetPort` in service spec
- NetworkPolicy blocking traffic

---

## DNS Resolution Failure

```bash
# Test DNS from inside a pod
kubectl run dns-test --image=busybox --rm -it -- nslookup kubernetes.default

# Check CoreDNS pods are running
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns
```

**DNS format inside cluster:**
```
<service>.<namespace>.svc.cluster.local
```

---

## Deployment Not Rolling Out

```bash
# Check rollout status
kubectl rollout status deployment/<name>

# Check replica sets
kubectl get rs -n <namespace>

# Describe deployment
kubectl describe deployment <name>

# Check if new pods are crashing
kubectl get pods -n <namespace> --sort-by='.metadata.creationTimestamp'
```

**Common causes:**
- New pods CrashLoopBackOff or failing readiness probes
- `maxUnavailable: 0` + `maxSurge: 0` (invalid config)
- Resource quota exceeded — can't create new pods
- Image pull failure

---

## Node Issues

```bash
# Check node status
kubectl get nodes

# Describe a NotReady node
kubectl describe node <node-name>

# Check resource pressure
kubectl describe node <node-name> | grep -E 'MemoryPressure|DiskPressure|PIDPressure'
```

---

## Stuck Terminating Pod

A pod stuck in `Terminating` usually has a finalizer blocking deletion.

```bash
# Force delete
kubectl delete pod <pod-name> --force --grace-period=0

# Remove finalizers manually
kubectl patch pod <pod-name> -p '{"metadata":{"finalizers":[]}}' --type=merge
```

---

## Quick Diagnostic Commands

```bash
# All pods across all namespaces with status
kubectl get pods -A

# Pods not in Running/Completed state
kubectl get pods -A | grep -v -E 'Running|Completed'

# Recent events cluster-wide
kubectl get events -A --sort-by='.lastTimestamp' | tail -20

# Resource usage across nodes
kubectl top nodes

# Resource usage across pods
kubectl top pods -A --sort-by=memory

# Check all resources in a namespace
kubectl get all -n <namespace>
```
