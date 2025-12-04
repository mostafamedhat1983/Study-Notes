---
tags:
  - AWS
  - Kubernetes
  - New-Tech
---
Direct Answer/Solution

AWS Gateway API Controller for Amazon VPC Lattice is a Kubernetes controller that implements the Kubernetes Gateway API and translates Gateway API resources (like `Gateway` and `HTTPRoute`) into Amazon VPC Lattice objects. In practice, it lets you configure VPC Lattice service networks, services, listeners, and routing using standard Kubernetes CRDs instead of calling VPC Lattice APIs directly[web:112][web:127][web:132].

## Source of Truth (Official Doc Links)

- AWS blog – Introducing AWS Gateway API Controller for Amazon VPC Lattice[web:112]  
- AWS Gateway API Controller site (docs & concepts)[web:127]  
- Amazon VPC Lattice overview[web:128]  
- AWS application networking K8s repo (controller implementation)[web:132]

## Architectural Reasoning (What it actually does)

- Implements the **Kubernetes Gateway API**:
  - Provides a `GatewayClass` (typically `amazon-vpc-lattice`), `Gateway`, and `HTTPRoute` resources.
  - When you create these CRDs in your EKS cluster, the controller:
    - Creates/updates a **VPC Lattice service network**.
    - Creates **VPC Lattice services** and **listeners**.
    - Registers your Kubernetes `Service` endpoints as targets in VPC Lattice target groups[web:112][web:127][web:128].

- Purpose:
  - Give you **service-mesh–style, L7 routing and security across VPCs, accounts, and clusters** without running your own mesh (no sidecars required)[web:112][web:127][web:129].  
  - Unify traffic management for:
    - EKS services (pods).
    - EC2-based services.
    - Lambda and other compute registered into VPC Lattice[web:127][web:128].

Typical flow:

1. Platform team installs AWS Gateway API Controller in EKS.
2. They define a `GatewayClass` selecting the Lattice implementation.
3. App teams define `Gateway` and `HTTPRoute` objects describing hostnames, paths, and backends.
4. Controller maps those to:
   - VPC Lattice service network.
   - VPC Lattice services, listeners, and routing rules.
   - Target groups with EKS services/pods as targets[web:112][web:127][web:132].

## Pros & Cons vs “classic” approaches

| Dimension                  | ALB Ingress / NLB + Ingress           | VPC Lattice + Gateway API Controller             |
|---------------------------|----------------------------------------|--------------------------------------------------|
| Scope                     | Per‑cluster, per‑VPC (mostly north‑south) | Cross‑VPC, cross‑account, cross‑cluster (east‑west) |
| API model                 | `Ingress` (limited spec)              | Gateway API (richer, role‑oriented)             |
| Mesh features             | Limited (no global service network)   | Service-network abstraction, auth policies, observability[web:112][web:128] |
| Sidecars                  | Often needed with meshes              | Not needed; data plane is managed by VPC Lattice[web:112][web:127] |
| Targets                   | Primarily K8s Services                 | K8s Services, EC2, Lambda, other Lattice targets[web:127][web:128] |

## Security & Best Practice Check

- VPC Lattice provides:
  - Central auth policies at service or service-network level (fine-grained, IAM-style)[web:127][web:128].  
  - Centralized metrics and logs for each request and response across the service network[web:128].  
- The controller:
  - Lets platform teams control **infrastructure (GatewayClass/Gateway)** while app teams control **routes**, following Gateway API’s role separation model[web:112][web:127].  
  - Reduces the need for custom controllers or ad‑hoc meshes, simplifying security review and operations.

In short: AWS Gateway API Controller for Amazon VPC Lattice is the Kubernetes-native control plane that lets you drive VPC Lattice (a managed L7, multi-VPC/multi-cluster service network) using standard Gateway API CRDs instead of bespoke AWS APIs.
