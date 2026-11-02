---
tags:
  - AWS
  - EKS
  - Kubernetes
---

The Amazon VPC CNI (Container Network Interface) plugin is the default networking solution for Amazon EKS that enables Kubernetes pods to receive native VPC IP addresses, integrating pod networking directly with AWS VPC infrastructure.

## How It Works

The VPC CNI operates through two key components running as a DaemonSet on each node:

**CNI Binary**: Configures pod network interfaces and enables pod-to-pod communication when pods are added or removed from nodes.

**ipamd daemon**: Manages Elastic Network Interfaces (ENIs) and maintains a "warm pool" of pre-allocated IP addresses for faster pod startup. When a pod is scheduled, the plugin attaches an ENI to the node and assigns a secondary IP address from the node's subnet directly to the pod.

## IP Address Management

The VPC CNI supports two operating modes:

- **Secondary IP Mode (default)**: Allocates individual IP addresses from node subnets, suitable for smaller clusters
- **Prefix mode (IPv4)**: Assigns `/28` prefixes (16 IPv4 addresses each) to ENIs instead of individual secondary IPs. Each prefix consumes one ENI address slot, so the number of prefixes per ENI depends on the EC2 instance type's IP-slot limit and other assigned addresses. For example, a `c5.large` ENI supports up to 9 prefixes when no secondary IPv4 addresses are assigned; there is no universal 30-prefix limit.

The plugin maintains warm pools by pre-allocating ENIs and IP addresses based on configurable targets like `WARM_IP_TARGET` and `WARM_ENI_TARGET`.

## Key Benefits

Pods receive routable VPC IP addresses, enabling direct communication without overlay networking and allowing use of VPC security groups, flow logs, and routing tables. This simplifies network architecture by giving pods the same IP address inside and outside the container, and supports both Linux and Windows containers seamlessly.
