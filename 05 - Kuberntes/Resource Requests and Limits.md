---
tags:
  - Kubernetes
---
Resource requests and limits tell Kubernetes how much CPU and memory a container needs and how much it is allowed to use.

---

## Main idea

- **request** = the minimum guaranteed amount (used for scheduling)
- **limit** = the maximum allowed amount (enforced at runtime)
- the scheduler uses requests to decide which node to place the Pod on
- if a container exceeds its memory limit, it is OOM killed
- if a container exceeds its CPU limit, it is throttled (not killed)

---

## CPU units

- `1` = 1 full CPU core
- `500m` = 0.5 cores (500 millicores)
- `100m` = 0.1 cores
- CPU is compressible — exceeding the limit causes throttling, not termination

## Memory units

- `Mi` = mebibytes (1 Mi = 1,048,576 bytes)
- `Gi` = gibibytes
- `M` = megabytes
- Memory is not compressible — exceeding the limit causes OOM kill

---

## Setting requests and limits

```yaml
spec:
  containers:
    - name: app
      image: my-app:1.0
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
```

---

## QoS classes

Kubernetes assigns a QoS class to each Pod based on its resource settings.

| QoS Class | Condition | Eviction priority |
|---|---|---|
| `Guaranteed` | requests == limits for all containers | last to be evicted |
| `Burstable` | requests set but less than limits | evicted after BestEffort |
| `BestEffort` | no requests or limits set | first to be evicted |

Simple rule:
- set requests == limits for critical production Pods (Guaranteed)
- set requests < limits for normal apps (Burstable)
- never leave both unset in production (BestEffort)

---

## LimitRange (namespace-level defaults)

Sets default requests and limits for all Pods in a namespace.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:
        cpu: "200m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "2"
        memory: "2Gi"
```

---

## Checking resource usage

```bash
# Node resource usage
kubectl top nodes

# Pod resource usage
kubectl top pods
kubectl top pods -n <namespace>

# Describe a node to see allocatable vs requested
kubectl describe node <node-name>
```

---

## Common mistakes

- not setting resource requests (Pods may land on overloaded nodes)
- not setting memory limits (one Pod can consume all node memory)
- setting limits much higher than requests (node can be massively overcommitted)
- setting CPU limits too low (causes unnecessary throttling and slow apps)
- setting memory requests too low (Pod gets evicted under pressure)
- running production Pods as BestEffort (first to be evicted)

---

## Good practices

- always set both requests and limits
- for production critical Pods, set requests == limits (Guaranteed QoS)
- use LimitRange to set safe defaults per namespace
- use ResourceQuota to cap total usage per namespace
- use `kubectl top` to gather real usage data before setting limits
- review and tune limits based on actual usage, not guesses

---

## Must memorize

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

```bash
kubectl top pods
kubectl top nodes
kubectl describe node <name>
```

```
QoS: Guaranteed > Burstable > BestEffort
CPU over limit = throttled
Memory over limit = OOM killed
```

---

## Key ideas

- Requests are for scheduling; limits are for runtime enforcement.
- CPU is throttled when exceeded; memory causes OOM kill.
- QoS class determines eviction priority under node pressure.
- Always set both requests and limits in production.
- Set requests == limits for critical Pods to get Guaranteed QoS.
