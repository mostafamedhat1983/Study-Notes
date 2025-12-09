---
tags:
  - AWS
  - EKS
  - Kubernetes
---


The Amazon VPC CNI (Container Network Interface) plugin is the default networking solution for Amazon EKS that enables Kubernetes pods to receive native VPC IP addresses, integrating pod networking directly with AWS VPC infrastructure [web:70][web:77].

## How It Works

The VPC CNI operates through two key components running as a DaemonSet on each node [web:71][web:74]:

**CNI Binary**: Configures pod network interfaces and enables pod-to-pod communication when pods are added or removed from nodes [web:71].

**ipamd daemon**: Manages Elastic Network Interfaces (ENIs) and maintains a "warm pool" of pre-allocated IP addresses for faster pod startup [web:71][web:80]. When a pod is scheduled, the plugin attaches an ENI to the node and assigns a secondary IP address from the node's subnet directly to the pod [web:74][web:77].

## IP Address Management

The VPC CNI supports two operating modes [web:76]:

- **Secondary IP Mode (default)**: Allocates individual IP addresses from node subnets, suitable for smaller clusters
- **Prefix Mode**: Assigns /28 prefixes to ENIs instead of individual IPs, allowing up to 30 prefixes per ENI for higher pod density [web:80][web:76]

The plugin maintains warm pools by pre-allocating ENIs and IP addresses based on configurable targets like `WARM_IP_TARGET` and `WARM_ENI_TARGET` [web:71][web:83].

## Key Benefits

Pods receive routable VPC IP addresses, enabling direct communication without overlay networking and allowing use of VPC security groups, flow logs, and routing tables [web:74][web:77]. This simplifies network architecture by giving pods the same IP address inside and outside the container, and supports both Linux and Windows containers seamlessly [web:74][web:84].
