---
tags:
  - My_CV_Project
  - AWS
  - EKS
  - Kubernetes
---
# Investigating EKS Pod-Density and Cluster Autoscaler

This note separates two different questions:

1. Why did a Pod fail to schedule on the existing EKS nodes?
2. Why did the node autoscaler or managed node group not add capacity?

Do not assume that reaching a node's Pod limit prevents Cluster Autoscaler from scaling. Cluster Autoscaler reacts to **unschedulable Pods** when adding a node from a configured node group would make the Pod schedulable. Its scheduling simulation includes node constraints, not only CPU and memory.

## Pod density on EKS nodes

With the Amazon VPC CNI, Pods generally receive VPC IP addresses from ENI/IP capacity (or prefix delegation, if configured). The maximum Pod count depends on:

- EC2 instance type network limits (maximum ENIs and addresses/slots per ENI)
- VPC CNI configuration
- Whether prefix delegation is enabled
- The node's configured `maxPods`
- Host-network Pods and other implementation details

For some common instance types, EKS publishes a recommended maximum Pod count. Do not derive a node's actual Pod capacity from a simplified ENI calculation without checking the instance type, CNI mode, and node bootstrap configuration.

Useful checks:

```bash
# Node allocatable capacity and current Pod count
kubectl describe node <node-name>

# Inspect the node's capacity/allocatable fields
kubectl get node <node-name> -o jsonpath='{.status.capacity.pods}{" / "}{.status.allocatable.pods}{"\n"}'

# Check scheduling failure reason
kubectl describe pod <pending-pod> -n <namespace>

# Inspect VPC CNI configuration
kubectl get daemonset aws-node -n kube-system -o yaml
kubectl get configmap amazon-vpc-cni -n kube-system -o yaml
```

## Does Cluster Autoscaler scale for Pod-count limits?

Yes, it can. If the scheduler marks a Pod unschedulable because a node has reached its Pod capacity, Cluster Autoscaler can consider scaling a configured node group **if a new node of that group would make the Pod schedulable**.

A scale-up may still not happen when, for example:

- Cluster Autoscaler is not installed, healthy, or configured to discover the node group.
- The node group has already reached its configured maximum size.
- No configured node-group template satisfies the Pod's resource requests, selectors, affinity, taints/tolerations, topology, or storage constraints.
- The Pod is not considered eligible for scale-up by the installed autoscaler version/configuration.
- The node group's AWS Auto Scaling configuration or permissions prevent the requested change.

The statement “Cluster Autoscaler does not scale for Pod limits because it only checks CPU/memory” is incorrect.

## Troubleshooting sequence

1. Check the Pending Pod's Events:
   ```bash
   kubectl describe pod <pending-pod> -n <namespace>
   ```
2. Confirm the scheduler reason (for example, `Too many pods`, insufficient CPU/memory, taint, affinity, or volume topology).
3. Check each node's Pod capacity and current Pod count:
   ```bash
   kubectl describe node <node-name>
   ```
4. Check Cluster Autoscaler logs and confirm that the node group is discovered and below its maximum size.
5. Verify that adding a node of that node group would satisfy the Pod's scheduling constraints.
6. Check the EKS VPC CNI mode and the instance type's network limits before changing `maxPods` or instance type.

## Important distinction

Increasing a managed node group's `max_size` only raises its allowed upper bound; it does not itself add nodes. A working autoscaler must be configured and must decide that an eligible unschedulable Pod can be scheduled on a new node.

## Project-specific facts must be verified

The old version of this note stated that the cluster hit an 11-Pod limit on `t3.small`, that Cluster Autoscaler would not scale for Pod limits, and that manual scaling to four nodes was the fix. Keep those statements only if supported by project evidence such as:

- the actual EC2 instance type and node `maxPods` value;
- Pod scheduling Events showing `Too many pods`;
- Cluster Autoscaler logs/configuration at the time;
- node-group minimum/maximum values and actual scaling history.

Without those records, describe this as a troubleshooting hypothesis, not a confirmed root cause or verified fix.

## Key takeaway

A Pod-capacity limit can make a Pod unschedulable, and Cluster Autoscaler can scale for that condition when a configured node group can provide a node that makes the Pod schedulable. Diagnose the scheduler Events and autoscaler logs instead of assuming CPU/memory-only behavior.
