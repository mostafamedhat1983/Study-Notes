---
tags:
  - Kubernetes
---
Resource requests and limits describe the CPU and memory resources a container requests for scheduling and the limits enforced at runtime.

## Requests and limits

- **Request:** used by the scheduler when selecting a node. It is the amount Kubernetes accounts for when placing the Pod; it is not a hard runtime cap.
- **Limit:** an upper bound enforced by the runtime/kernel where supported.
- A container can use more than its request if resources are available, up to its limit.
- If a container exceeds its memory limit, it may be terminated by the OOM killer. The exact observed Pod/container status depends on the circumstances.
- CPU is compressible: exceeding a CPU limit results in throttling rather than an OOM kill.

## CPU and memory units

- `1` CPU = one logical CPU core.
- `500m` CPU = 0.5 CPU.
- `100m` CPU = 0.1 CPU.
- `Mi` and `Gi` are binary memory units; `M` and `G` are decimal units.

Example:

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

## QoS classes

Kubernetes assigns a Pod a QoS class based on the CPU and memory requests and limits of its containers.

| Class | Simplified condition |
|---|---|
| `Guaranteed` | Every container has CPU and memory requests and limits, and each request equals its corresponding limit. |
| `Burstable` | The Pod does not qualify as Guaranteed, but at least one container has a CPU or memory request/limit. |
| `BestEffort` | No container in the Pod has CPU or memory requests or limits. |

QoS class influences eviction decisions under node pressure, but it is not an absolute promise that every Guaranteed Pod is always protected or that eviction order is determined by QoS alone. Kubernetes also considers whether usage exceeds requests and Pod priority.

## LimitRange

A LimitRange can provide default requests/limits and enforce minimums or maximums for containers or Pods in a namespace.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: container-defaults
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

## Inspect resource use and scheduling

```bash
kubectl top nodes
kubectl top pods
kubectl top pods -n <namespace>
kubectl describe node <node-name>
```

`kubectl top` requires the resource metrics pipeline (commonly Metrics Server). It reports observed usage, not the scheduler's full accounting. Use `kubectl describe node` and Pod specs to inspect allocatable resources and requests.

## Practical guidance

- Set realistic requests based on observed workload behavior; they affect scheduling and capacity planning.
- Set limits when you need to bound resource consumption, while understanding the trade-offs (especially CPU throttling).
- Avoid arbitrarily setting requests equal to limits for every workload. Guaranteed QoS has specific criteria and may reduce flexibility.
- Use LimitRange for namespace defaults and ResourceQuota for aggregate namespace constraints.
- Monitor usage over time before tuning resource values.

## Quick reference

- Requests: scheduling and resource accounting.
- Limits: runtime ceilings.
- CPU limit exceeded: throttling.
- Memory limit exceeded: possible OOM termination.
- QoS: Guaranteed, Burstable, BestEffort; eviction behavior also depends on Pod priority and usage relative to requests.
