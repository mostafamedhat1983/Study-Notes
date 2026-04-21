---
tags:
  - Terraform
---
Terraform works best when the code is clean, predictable, secure, and easy to maintain.

Best practices help reduce:
- human error
- infrastructure drift
- inconsistent environments
- unsafe changes
- hard-to-maintain code

These practices become more important as Terraform projects grow.

---

## Main idea

Good Terraform code should not only work.

It should also be:
- readable
- reusable
- safe
- version-controlled
- easy to review
- easy to scale

Simple idea:
- working code is good
- maintainable and safe code is better

---

## Keep infrastructure as code in Git

Terraform code should be stored in version control.

This helps you:
- track changes
- review pull requests
- roll back mistakes
- collaborate safely
- understand infrastructure history

Infrastructure code should be treated like application code.

---

## Review `terraform plan` before `apply`

Always check the execution plan before applying changes.

This helps you catch:
- accidental deletions
- wrong resource names
- unexpected replacements
- bad variable values
- environment mistakes

A very common safe workflow is:

```bash
terraform fmt
terraform validate
terraform plan
terraform apply
```

---

## Use remote state for real projects

Local state is fine for learning, but real projects should usually use remote state.

Remote state is better for:
- team collaboration
- durability
- access control
- locking
- CI/CD workflows

This helps reduce the risk of conflicting or lost state.

---

## Protect state carefully

Terraform state may contain important or sensitive data.

So you should:
- restrict access to state
- enable encryption where possible
- avoid committing state files to Git
- treat state as critical infrastructure data

State is not just a temporary file.

---

## Separate environments clearly

Keep environments such as:
- dev
- staging
- production

properly separated.

This can be done using:
- separate directories
- separate state files
- separate backend keys
- carefully designed workspaces

The main goal is to reduce risk and avoid environment confusion.

---

## Avoid hardcoding values

Do not hardcode values that are likely to change.

Examples:
- region
- instance type
- environment name
- CIDR blocks
- project names

Use:
- variables
- locals
- data sources

This makes Terraform code more reusable and easier to maintain.

---

## Use variables intentionally

Use variables for values that should be configurable.

Good uses:
- environment-specific values
- reusable module inputs
- values that differ across deployments

Avoid turning every small thing into a variable without a clear reason.

Too many unnecessary variables can make the configuration harder to use.

---

## Use locals for repeated expressions

If the same expression or naming pattern appears in many places, move it into a local value.

This helps:
- reduce duplication
- improve readability
- centralize naming logic
- make updates easier

Example:

```hcl
locals {
  common_tags = {
    Project     = "demo"
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}
```

---

## Use modules for reuse

If you repeat the same infrastructure pattern many times, use modules.

Modules are especially useful for:
- networking
- compute patterns
- security groups
- storage patterns
- shared organizational standards

Good modules improve consistency and reduce duplication.

---

## Keep modules focused

A good module should usually do one clear job.

Examples:
- VPC module
- EC2 instance module
- S3 bucket module
- IAM role module

Avoid giant modules that do too many unrelated things.

Smaller focused modules are easier to understand and maintain.

---

## Pin provider and module versions

Version pinning helps keep Terraform behavior predictable.

Example provider version pinning:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

Example module version pinning:

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
}
```

This helps avoid unexpected behavior from version changes.

---

## Format and validate code

Use Terraform formatting and validation regularly.

Commands:

```bash
terraform fmt
terraform validate
```

These help:
- keep code consistent
- catch obvious mistakes early
- improve team readability

This should be part of your normal workflow.

---

## Prefer simple code over clever code

Terraform should be easy to read.

Avoid:
- overly complex expressions
- confusing conditional logic
- unnecessary abstraction
- deeply nested dynamic logic

If the code is hard to understand quickly, it becomes harder to trust and maintain.

---

## Use clear names

Use clear names for:
- resources
- variables
- outputs
- modules
- locals

Bad names create confusion.

Good names make Terraform code easier to read and review.

Examples:
- better: `aws_instance.web`
- worse: `aws_instance.x`

---

## Minimize manual changes outside Terraform

If Terraform manages a resource, avoid changing that resource manually from the cloud console unless necessary.

Manual changes can cause:
- drift
- confusion
- unexpected plans
- harder troubleshooting

Terraform works best when it is the main source of truth.

---

## Use data sources for existing infrastructure

If infrastructure already exists and Terraform only needs to read it, use data sources instead of pretending Terraform created it.

This helps keep the code cleaner and more accurate.

Simple rule:
- create/manage = resource
- read existing = data source

---

## Be careful with provisioners

Provisioners should usually be a last resort.

Prefer:
- native Terraform resources
- cloud-init
- user data
- image baking
- external configuration tools when appropriate

Provisioners can make Terraform less predictable and harder to debug.

---

## Protect critical resources

For important resources, consider safety controls such as:
- `prevent_destroy`
- environment separation
- careful review workflows
- access restrictions

This helps reduce the chance of accidental destructive changes.

---

## Make small and reviewable changes

Avoid huge Terraform changes when possible.

Smaller changes are:
- easier to review
- safer to apply
- easier to troubleshoot
- easier to roll back mentally

This is especially important in production.

---

## Keep secrets out of code when possible

Avoid putting secrets directly in Terraform files.

Examples:
- passwords
- API tokens
- secret keys

Use safer patterns such as:
- sensitive variables
- secret managers
- environment-based secure injection
- controlled CI/CD secret handling

Even with sensitive variables, remember that state handling still matters.

---

## Common mistakes

- applying without reviewing the plan
- committing state files
- hardcoding environment-specific values
- using local state in team projects
- creating overly complicated modules
- making manual cloud-console changes behind Terraform's back
- using unclear names
- skipping formatting and validation

---

## Good practices summary

- store Terraform code in Git
- review plans before apply
- use remote state in real projects
- protect state carefully
- separate environments clearly
- use variables and locals wisely
- use modules for reuse
- pin versions
- keep code readable
- avoid manual drift
- protect critical resources

---

## Important notes

- Good Terraform practices improve safety, consistency, and maintainability.
- Remote state is usually better for real projects.
- Plan review is one of the most important safety habits.
- Version pinning improves predictability.
- Clear naming and simpler logic improve maintainability.
- Terraform should remain the source of truth for managed infrastructure.

---

## Simple rule of thumb

- keep Terraform code clean
- keep state safe
- review before apply
- avoid unnecessary complexity
- make infrastructure changes predictable

---

## Must memorize

```bash
terraform fmt
terraform validate
terraform plan
terraform apply
```

```hcl
lifecycle {
  prevent_destroy = true
}
```

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

- use remote state for real projects
- avoid committing `terraform.tfstate`
- prefer readable Terraform over clever Terraform

---

## Key ideas

- Terraform best practices are about safety, clarity, and maintainability.
- Infrastructure code should be version-controlled and reviewed.
- State should be protected and usually stored remotely in real projects.
- Variables, locals, and modules should be used intentionally.
- Simpler code and clearer naming improve long-term maintainability.
- Good Terraform habits reduce drift, mistakes, and risky changes.