---
tags:
  - AWS
  - Kubernetes
  - EKS
---
# IRSA (OIDC) vs Amazon EKS Pod Identity

Both IRSA and EKS Pod Identity let Kubernetes workloads in Amazon EKS obtain temporary AWS credentials for an IAM role without embedding long-term AWS access keys in container images or manifests.

They use different identity and credential-delivery flows. Neither is universally superior; choose based on cluster support, trust requirements, SDK compatibility, and operational design.

## IRSA — IAM Roles for Service Accounts

- Uses the EKS cluster's OIDC issuer and an IAM OIDC identity provider.
- A Kubernetes ServiceAccount is annotated with an IAM role ARN.
- The Pod receives a projected service-account OIDC token.
- A supported AWS SDK uses the web identity token flow (STS `AssumeRoleWithWebIdentity`) to obtain temporary role credentials.
- IAM role trust policies can constrain the OIDC provider, audience, namespace, and ServiceAccount subject.

IRSA is a well-established option and can be appropriate when OIDC-based trust conditions are important or when workloads/platforms already use it.

## EKS Pod Identity

- Uses an EKS Pod Identity association to map a cluster, namespace, and ServiceAccount to an IAM role.
- Requires the EKS Pod Identity Agent add-on on supported EC2-backed nodes/environments.
- EKS configures the Pod to use the container credential provider endpoint and a Pod Identity token.
- The agent calls the EKS Auth API (`AssumeRoleForPodIdentity`) and supplies temporary credentials to supported AWS SDKs.
- The IAM role trust policy must allow the `pods.eks.amazonaws.com` service principal and the required STS actions, including `sts:AssumeRole` and `sts:TagSession`.
- A role can be associated with ServiceAccounts across multiple clusters without adding a separate OIDC provider for each cluster; use association/session tags and IAM conditions to scope access.

The agent does **not** proxy or intercept the workload's normal AWS service API requests. The SDK obtains credentials from the local credential endpoint and then calls AWS services directly.

## Comparison

| Area | IRSA | EKS Pod Identity |
|---|---|---|
| Identity mapping | ServiceAccount annotation + OIDC trust | EKS Pod Identity association |
| Token/credential flow | Projected OIDC token; STS web identity | Pod Identity token; node agent and EKS Auth API |
| OIDC provider | Required for the cluster | Not required for Pod Identity |
| IAM role trust | OIDC provider and subject/audience conditions | `pods.eks.amazonaws.com` principal and STS session actions; scope with supported conditions |
| Node agent | Not required | Pod Identity Agent required where applicable |
| SDK support | SDK must support web identity token credentials | SDK must support container credential provider |
| Cross-cluster reuse | Trust configuration must account for each cluster issuer | Associations can use a role across clusters; use session tags/conditions for scoping |
| Non-EKS Kubernetes | IRSA is EKS-specific OIDC integration; other platforms have their own federation setup | Not supported outside Amazon EKS |

## Important corrections to common claims

- Pod Identity does not remove the need for an IAM role trust policy. The trust policy still needs the EKS Pod Identity service principal and required STS actions.
- Pod Identity does not make IAM permissions automatically “pod-specific” unless roles, ServiceAccounts, associations, and policies are scoped correctly.
- IRSA is not inherently insecure or obsolete; it remains supported.
- Pod Identity's operational simplicity is not proof that it is always the better choice for every regulated or multi-platform environment.
- Both mechanisms provide temporary credentials to workload processes through supported SDK credential providers. Do not claim that credentials never exist in the Pod.

## Choosing between them

Consider EKS Pod Identity for new EKS workloads when the cluster and add-on support it and the application uses a compatible AWS SDK. Consider retaining IRSA when existing trust conditions, integrations, or operational controls depend on OIDC. Evaluate actual requirements rather than choosing solely because one mechanism is newer.

## Official references

- https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html
- https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html
- https://docs.aws.amazon.com/eks/latest/best-practices/identity-and-access-management.html
