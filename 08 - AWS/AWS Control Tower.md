---
tags:
  - AWS
---

AWS Control Tower is an AWS service that automates the setup and ongoing governance of a secure, multi-account AWS environment (a “landing zone”) using AWS best practices for security, compliance, and operations. It sits on top of AWS Organizations and related services to give you centralized guardrails, account provisioning, and a governance dashboard across all your AWS accounts.

## What Control Tower Does
- Sets up a well-architected multi-account “landing zone” with predefined organizational units (OUs), shared accounts (management, log archive, audit), identity, and logging baselines.
- Applies governance “controls” (guardrails) for security, operations, and compliance using Service Control Policies (SCPs), AWS Config rules, and related mechanisms.
- Provides Account Factory so teams can self-service new AWS accounts with standardized, pre-approved configurations while remaining under central governance.

## Key Concepts
- **Landing Zone**: A preconfigured, multi-account AWS environment following AWS multi-account and security best practices (OUs, centralized logging, baseline security controls).
- **Controls / Guardrails**: High-level rules that can be preventive (block actions), detective (detect and flag drift), or proactive, with categories like mandatory or strongly recommended.
- **Account Factory**: A templated way to create accounts with standard settings (regions, guardrails, network patterns, IAM baselines) using AWS Service Catalog under the hood.

## Why Use It vs. Rolling Your Own

**Pros of AWS Control Tower**
- Rapid, opinionated multi-account setup following AWS best-practice guidance without building all automation yourself.
- Centralized governance and visibility via a dashboard showing account inventory, enabled controls, and non-compliant resources by OU/account.
- Native integration with AWS Organizations, IAM Identity Center, AWS Config, and CloudTrail for identity, logging, and compliance at scale.

**Cons / Trade-offs**
- Opinionated: you are constrained by Control Tower’s model for OUs, shared accounts, and how guardrails are implemented; highly custom orgs may outgrow it.
- Not every advanced pattern is supported; very bespoke landing zones may require direct use of AWS Organizations, CloudFormation/Terraform, and custom governance pipelines instead.

## Comparison: Control Tower vs DIY Organizations

| Aspect | AWS Control Tower | DIY with AWS Organizations & Custom IaC |
|---|---|---|
| Setup speed | Fast; landing zone in under ~1 hour with guided workflow. | Slower; must design and implement org structure, logging, and guardrails yourself. |
| Governance model | Predefined guardrails and structure aligned to AWS best practices. | Fully custom; you design SCPs, Config rules, and processes. |
| Account provisioning | Built-in Account Factory with templates and self-service. | Must build your own account vending (APIs, Service Catalog, or external tooling). |
| Flexibility | Limited by Control Tower’s opinionated patterns. | Maximum flexibility; any org/OU layout or policy combination. |
| Operational overhead | Lower ongoing maintenance; AWS manages much orchestration. | Higher; you own all automation, updates, and drift management. |
| Best-practice alignment | Directly aligned with AWS multi-account strategy guidance. | Depends entirely on your implementation quality. |

## Security & Best Practice Check
From a principal DevOps/Security perspective, Control Tower is usually the default starting point for greenfield, multi-account AWS setups because it bakes in centralized logging, baseline security controls, and consistent account structure by design. For highly regulated or complex environments, it is still advisable to extend Control Tower with additional SCPs, Config rules, and CI/CD-based policy-as-code, rather than replacing it, unless your requirements fundamentally conflict with its opinionated model.

## Source of Truth (Official Doc Links)
- AWS Control Tower – “What is AWS Control Tower?”: https://docs.aws.amazon.com/controltower/latest/userguide/what-is-control-tower.html
- AWS Control Tower product page: https://aws.amazon.com/controltower/
- AWS Control Tower main documentation index: https://docs.aws.amazon.com/controltower/
- AWS Organizations integration: https://docs.aws.amazon.com/organizations/latest/userguide/services-that-can-integrate-CTower.html
