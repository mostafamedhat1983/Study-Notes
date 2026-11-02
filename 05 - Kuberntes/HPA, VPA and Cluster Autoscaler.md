# HPA, VPA and Cluster Autoscaler

Kubernetes autoscaling operates at different layers: HPA adjusts workload replica counts, VPA recommends or applies container resource settings, and Cluster Autoscaler adjusts node-group capacity when unschedulable Pods can be placed on added nodes.

## HPA — Horizontal Pod Autoscaler

HPA adjusts the replica count of a scalable workload based on observed metrics.

- Resource metrics include CPU and memory utilization or average values.
- Custom and external metrics can also be used when the corresponding metrics APIs/adapters are installed.
- For CPU or memory **utilization percentage**, the relevant container resource requests must be defined; utilization is calculated relative to requests.
- Metrics Server commonly supplies resource metrics, but it is not the provider for every custom/external metric.
- HPA targets a resource with a scale subresource, commonly a Deployment or StatefulSet.

Example:

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

Commands:

```bash
kubectl autoscale deployment web --cpu-percent=60 --min=2 --max=10
kubectl get hpa
kubectl describe hpa web-hpa
```

HPA behavior and timing are controlled by controller-manager configuration and the HPA behavior fields. Do not treat a single evaluation interval or stabilization window as a universal fixed value for every cluster.

## VPA — Vertical Pod Autoscaler

VPA analyzes historical and current resource usage and can recommend or apply CPU and memory requests (and, depending on configuration/version, limits).

- VPA does not change the replica count.
- Applying changed resource values may require Pod eviction/recreation, depending on the VPA update mode and Kubernetes/VPA version.
- VPA is a separate project/add-on; it is not enabled by simply creating a Kubernetes Deployment.

VPA update modes and feature availability can change across VPA releases. Check the installed VPA version's official documentation before copying a mode value into a manifest. In particular, do not use `Auto` as a generic current mode without checking the version; older notes and examples may use mode names that have changed.

Example structure:

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
    updateMode: "Off"
```

`Off` is useful when you want recommendations without VPA applying resource changes.

### HPA and VPA together

Using HPA on CPU/memory utilization while VPA changes the same CPU/memory requests can create feedback or competing adjustments because HPA utilization is calculated relative to requests. They can be combined with careful design (for example, different metrics or recommendation-only VPA), but do not assume every combination is safe.

## Cluster Autoscaler

Cluster Autoscaler changes the size of configured node groups.

It attempts to scale up when Pods are unschedulable and adding a node from a configured node group would make those Pods schedulable. The scheduler's constraints are broader than CPU and memory: they include Pod count capacity, node selectors, affinity, taints/tolerations, topology, and storage constraints.

It can scale down nodes that remain unneeded when their workloads can be safely moved elsewhere, subject to PodDisruptionBudgets, eviction rules, and autoscaler configuration.

### Common reasons a scale-up does not happen

- Cluster Autoscaler is not installed, healthy, or configured to discover the node group.
- The node group is already at its configured maximum.
- A new node of that group would still not satisfy the Pod's scheduling constraints.
- The Pod is not eligible for scale-up under the autoscaler's version/configuration.
- Cloud-provider permissions or node-group configuration prevent scaling.

### EKS considerations

Cluster Autoscaler and Karpenter are separate node-provisioning approaches. EKS managed node groups can be used with Cluster Autoscaler when configured correctly; Karpenter is another option with different provisioning behavior. Do not label one universally “preferred” without considering the architecture, version, and operational requirements.

## Comparison

| Feature | HPA | VPA | Cluster Autoscaler |
|---|---|---|---|
| Changes | Pod replica count | Pod resource recommendations/requests | Node-group size |
| Trigger | Metrics | Observed resource usage | Unschedulable Pods / unneeded nodes |
| Requires Pod restart | No, for replica changes | May, depending on update mode | No, for node count itself |
| Common pairing | HPA + Cluster Autoscaler | Carefully designed with HPA | Supports HPA by adding node capacity |

## Good practices

- Set realistic resource requests; they influence scheduling and HPA utilization calculations.
- Use HPA with Cluster Autoscaler or another node provisioner when workloads need both Pod and node scaling.
- Start with VPA recommendation mode before allowing it to evict/recreate Pods.
- Configure PodDisruptionBudgets thoughtfully; they can affect safe node scale-down.
- Inspect Pending Pod Events and autoscaler logs instead of assuming a scale-up trigger is based only on CPU/memory.
