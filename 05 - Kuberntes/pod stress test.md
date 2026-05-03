# Pod Stress Testing

Stress testing Pods helps validate resource limits, autoscaling behavior, and cluster stability before production.

---

## Why Stress Test Pods?

- Verify that CPU/memory `limits` are enforced correctly
- Trigger HPA scale-out and confirm autoscaling works
- Confirm OOMKill behavior when memory limit is exceeded
- Validate PodDisruptionBudgets and rolling updates under load
- Test liveness/readiness probe behavior under pressure

---

## Method 1: stress-ng Image

```bash
# Run a stress pod that consumes 1 CPU core for 60 seconds
kubectl run stress-test \
  --image=alexeiled/stress-ng \
  --restart=Never \
  --rm -it \
  -- --cpu 1 --timeout 60s

# Consume 256MB of memory for 30 seconds
kubectl run stress-mem \
  --image=alexeiled/stress-ng \
  --restart=Never \
  --rm -it \
  -- --vm 1 --vm-bytes 256M --timeout 30s
```

---

## Method 2: busybox CPU Loop

```bash
# Spin up a busybox pod and run a CPU loop inside it
kubectl run cpu-stress \
  --image=busybox \
  --restart=Never \
  --rm -it \
  -- sh -c "while true; do :; done"
```

---

## Method 3: stress Pod Manifest with Limits

Useful for testing that limits are enforced and HPA fires:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: stress-pod
spec:
  containers:
    - name: stress
      image: alexeiled/stress-ng
      args: ["--cpu", "2", "--timeout", "120s"]
      resources:
        requests:
          cpu: "500m"
          memory: "128Mi"
        limits:
          cpu: "1"
          memory: "256Mi"
```

```bash
kubectl apply -f stress-pod.yaml

# Watch resource usage
kubectl top pod stress-pod

# Watch HPA respond
kubectl get hpa -w

# Delete when done
kubectl delete pod stress-pod
```

---

## Method 4: Trigger OOMKill

```bash
# Allocate more memory than the limit to trigger OOMKill
kubectl run oom-test \
  --image=alexeiled/stress-ng \
  --restart=Never \
  --limits='memory=64Mi' \
  -- --vm 1 --vm-bytes 128M

# Check status
kubectl get pod oom-test
# Should show: OOMKilled

kubectl describe pod oom-test | grep -A5 'Last State\|OOMKilled'
```

---

## Monitoring During Stress Test

```bash
# Real-time resource usage
kubectl top pods -w
kubectl top nodes

# Watch HPA
kubectl get hpa -w

# Watch pod count change
kubectl get pods -w

# Check events
kubectl get events --sort-by='.lastTimestamp'
```

---

## Cleanup

```bash
kubectl delete pod stress-test stress-mem cpu-stress oom-test 2>/dev/null
```
