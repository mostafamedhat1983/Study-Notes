---
tags:
  - AWS
  - Kubernetes
  - EKS
---
Direct Answer/Solution

OIDC with IRSA (IAM Roles for Service Accounts) and EKS Pod Identity are both mechanisms to grant AWS IAM permissions to pods in EKS without using long-lived credentials, but they differ significantly in how identity mapping, setup, and scalability work. EKS Pod Identity is AWS's newer feature that provides simpler, more scalable pod-to-IAM role mapping by removing the explicit OIDC dependency at the cluster level, while IRSA relies on an OIDC trust relationship for every cluster.

Source of Truth (Official Doc Links)

- AWS EKS Pod Identity overview: https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html
- AWS EKS Identity & Access Management: https://docs.aws.amazon.com/eks/latest/best-practices/identity-and-access-management.html
- OIDC for EKS (IRSA): https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html

Architectural Reasoning (Why this? What are the alternatives?)

### IRSA (OIDC-based)

- Associates a Kubernetes service account with an IAM role by leveraging an OIDC provider configured per EKS cluster.
- The service account is annotated with the IAM role ARN; when a pod uses this service account, the AWS SDK uses a projected OIDC JWT to obtain credentials via AssumeRoleWithWebIdentity.
- Setup requires creating an OIDC provider for every EKS cluster, and updating the IAM role's trust relationship.
- Common limits: AWS has a default global account limit of 100 OIDC providers. Managing multiple clusters can quickly exhaust this.
- Best suited for existing clusters that already use IRSA, and where OIDC-based fine-grained identity controls (e.g., via claims) are in place.

### EKS Pod Identity (2024+)

- Removes the need for OIDC providers: roles are associated with a pod or service account via EKS's pod identity APIs.
- Deploys a Pod Identity agent/add-on to the EKS cluster, which handles the association between pods and assigned IAM roles without modifying the pod's manifest.
- Simplifies setup: no need to manage trust policies with OIDC subject condition, and the feature scales with clusters without hitting global OIDC limits.
- Enables more granular assignment, improved management, and better operational separation (IAM admin vs. EKS admin).
- Recommended for new clusters, large/federated EKS environments, or when onboarding new workloads.

#### Example Differences Table

| Feature                          | IRSA (OIDC)                                        | EKS Pod Identity                      |
|-----------------------------------|----------------------------------------------------|---------------------------------------|
| OIDC Provider per cluster         | Required, counts toward global AWS limit           | Not required                          |
| IAM trust policy updates          | Each role must trust cluster's OIDC provider       | Managed by EKS API, not manual        |
| Service account mapping           | Service account annotation                         | EKS API/console association           |
| Operational complexity            | Higher (OIDC, annotations, trust policy updates)   | Lower (declarative EKS API calls)     |
| OIDC Claim customization          | Full OIDC support for fine-grained trust           | ABAC-style, cluster and SA-based tags |
| Scale for large orgs/clusters     | Painful (OIDC limit, multi-cluster ops)            | Scales smoothly                       |
| Security                         | Mature, but trust policy drift risk                | Simpler, better least-privilege by pod|
| Availability                     | Long-established, well-integrated                  | Newer, rapidly being adopted          |

Trade-offs

- IRSA is mature, OIDC-based, and supports granular trust customization, but brings higher operational overhead and scale issues with many clusters.
- EKS Pod Identity lowers complexity, eliminates the OIDC provider limit, and is easier to maintain, but is newer and may not fit regulated environments that require explicit OIDC claims or non-EKS clusters.

Security & Best Practice Check

- Prefer EKS Pod Identity for new EKS clusters and when multi-cluster scalability or operational simplicity is a priority.
- If you already have a mature IRSA deployment with OIDC-based security controls and audit requirements, IRSA remains viable and compliant.
- In either model: avoid excess privilege, scope IAM roles to the smallest set of actions, use pod-specific roles, and rotate roles as needed.

Reference Guidance

- Official EKS Pod Identity docs: https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html
