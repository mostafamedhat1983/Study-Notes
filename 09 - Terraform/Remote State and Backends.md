---
tags:
  - Terraform
---
Terraform uses a state file to track real infrastructure and map it to your configuration. A backend controls where that state is stored and how Terraform accesses it. [web:653][web:654]

By default, Terraform uses the local backend, which stores state on the machine running Terraform. Remote backends store state in a shared location, which is better for team use, automation, and safer state management. [web:653][web:654][web:599]

---

## Main idea

A backend is Terraform’s state storage configuration.

Simple idea:
- state = Terraform’s record of real infrastructure
- backend = where that state lives and how it is managed

If state is only local, collaboration becomes harder and state can be lost with the machine. Remote backends improve sharing, consistency, and operational safety. [web:653][web:654][web:598]

---

## Why remote state matters

Remote state is useful because it helps teams work from the same source of truth. It also reduces the risk of losing state files that only exist on a local laptop or server. [web:654][web:598][web:601]

Good remote backends often support:
- centralized storage
- locking
- encryption
- access control
- versioning

These features help prevent corruption, reduce conflicts, and improve recovery options. [web:599][web:601]

---

## Local backend

If you do not configure a backend, Terraform uses the local backend. In that setup, the state file is stored locally in the working directory environment rather than in a shared remote system. [web:653][web:654]

Local state can be acceptable for:
- personal labs
- small experiments
- short-lived learning projects

It is usually a weak choice for team environments, production systems, or CI/CD workflows. [web:654][web:598]

---

## Remote backends

A remote backend stores Terraform state outside the local machine. HashiCorp’s backend documentation describes backends as the mechanism used to control where state is stored, and remote backends make collaboration and shared operations easier. [web:653][web:715]

Common remote backend options include:
- Amazon S3
- Azure Blob Storage
- Google Cloud Storage
- Terraform Cloud / remote backend

Different backends have different features, but the common goal is centralized and safer state management. [web:598][web:653]

---

## Backend block syntax

Terraform configures the backend inside the `terraform` block.

Example:

```hcl
terraform {
  backend "s3" {
    bucket = "my-terraform-state-bucket"
    key    = "network/terraform.tfstate"
    region = "us-east-1"
  }
}
```

The backend block tells Terraform where to store or read the state file. Backend configuration is part of Terraform settings, not a provider resource. [web:653][web:717]

---

## Common S3 backend example

A very common AWS pattern is:
- S3 bucket for state storage
- DynamoDB table for state locking
- encryption enabled

Example:

```hcl
terraform {
  backend "s3" {
    bucket         = "my-company-terraform-state-prod"
    key            = "environments/production/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-locks"
    encrypt        = true
  }
}
```

This setup is widely used because S3 provides centralized storage, while DynamoDB locking helps prevent concurrent Terraform runs from corrupting state. [web:717][web:599]

---

## Initialize backend

After adding or changing backend configuration, run:

```bash
terraform init
```

Terraform initializes the backend during `terraform init`. HashiCorp’s backend documentation notes that changing backend configuration requires reinitialization. [web:653]

---

## Partial backend config

Terraform supports partial backend configuration, which means some backend values can be supplied later instead of being fully hardcoded in the configuration. This is useful when you do not want to commit all backend details directly in source files. [web:653]

This is commonly used with `-backend-config` during initialization.

Example:

```bash
terraform init -backend-config="bucket=my-company-tf-state"
```

That pattern helps keep some backend details outside the main Terraform configuration. [web:653]

---

## Backend migration

Terraform supports changing backend configuration and migrating state from one backend to another. HashiCorp documents backend reconfiguration and migration as part of backend management. [web:653]

A common example is:
- start with local state
- create a remote backend
- reinitialize Terraform
- migrate the existing state

This is a normal path when a small project grows into a shared team workflow. [web:653][web:716]

---

## Remote state data source

Terraform can also read outputs from another state file by using the `terraform_remote_state` data source. This allows one Terraform configuration to consume outputs produced by another configuration. [web:717]

Example:

```hcl
data "terraform_remote_state" "network" {
  backend = "s3"

  config = {
    bucket = "my-company-terraform-state-prod"
    key    = "environments/production/network/terraform.tfstate"
    region = "us-east-1"
  }
}
```

This is commonly used when one stack, such as an application stack, needs values from another stack, such as a networking stack. [web:717]

---

## Example using remote outputs

Once remote state is connected, you can read output values from it.

Example:

```hcl
resource "aws_instance" "web" {
  ami                    = "ami-0c55b159cbfafe1f0"
  instance_type          = "t3.micro"
  subnet_id              = data.terraform_remote_state.network.outputs.public_subnet_ids
  vpc_security_group_ids = [data.terraform_remote_state.network.outputs.web_security_group_id]
}
```

This allows one Terraform project to reuse important values exposed by another project’s outputs. [web:717]

---

## Security concerns

Terraform state can contain sensitive information, so backend security matters. Best-practice guidance commonly recommends strong access control, encryption at rest, encryption in transit, and avoiding public exposure of state storage. [web:599][web:598][web:601]

For example, S3 state buckets are commonly configured with:
- versioning
- server-side encryption
- blocked public access

Those controls help reduce risk and improve recoverability. [web:717][web:601]

---

## Locking

State locking helps prevent two Terraform operations from modifying the same state at the same time. This reduces the risk of corruption and conflicting updates. [web:599][web:601]

For AWS-based setups, DynamoDB is commonly used with S3 backends for locking. In general, choosing a backend with locking support is a strong operational practice. [web:717][web:599]

## Force unlock

Sometimes a state lock may remain in place if Terraform fails during an operation.

In that case, you can use:

```bash
terraform force-unlock LOCK_ID
```

This manually removes the lock so Terraform can use the state again.

Use this very carefully and only when you are sure no other Terraform operation is currently using the same state.

---

## Versioning

Versioning allows you to keep historical copies of the state file. This is useful if you need to inspect or recover from accidental changes. [web:717][web:601]

For S3-based backends, enabling bucket versioning is a common recommendation because Terraform state is a critical operational asset. [web:717]

---

## Environment separation

Best-practice guidance often recommends separate state files per environment or logical system. This reduces blast radius and keeps production, staging, and development more isolated. [web:599]

Examples:
- `environments/dev/terraform.tfstate`
- `environments/staging/terraform.tfstate`
- `environments/prod/terraform.tfstate`

This approach makes state easier to organize and safer to manage at scale. [web:717][web:599]

---

## Common mistakes

- storing important team state only on a local machine
- not enabling locking when the backend supports it
- not enabling encryption or versioning
- giving too many people direct access to state
- keeping all environments in one state file
- exposing state storage publicly
- assuming backend configuration can freely use normal input variables

Backend configuration has special rules, and examples for S3 backends often note that values must be hardcoded or passed through backend-specific configuration rather than normal Terraform variables. [web:717][web:653]

---

## Good practices

- use a remote backend for team or production work
- enable locking
- enable encryption
- enable versioning where supported
- separate environments and systems into distinct state files
- restrict access to state
- review backend changes carefully
- use `terraform init` after backend changes

These practices are repeatedly recommended in Terraform backend and state-management guidance because state is one of the most sensitive operational files in a Terraform workflow. [web:653][web:599][web:601]

---

## Important notes

- Terraform state tracks real infrastructure. [web:654]
- A backend controls where state is stored and accessed. [web:653]
- Local backend is the default when no backend is configured. [web:653][web:654]
- Remote backends are better for collaboration and automation. [web:654][web:598]
- S3 with DynamoDB locking is a common AWS pattern. [web:717]
- `terraform init` is used to initialize or reinitialize backend settings. [web:653]
- `terraform_remote_state` can read outputs from another state file. [web:717]

---

## Simple rule of thumb

- local backend for small personal labs
- remote backend for real teamwork and production use
- treat the state file as sensitive
- enable locking and protection features whenever possible

---

## Must memorize

```hcl
terraform {
  backend "s3" {
    bucket         = "my-company-terraform-state-prod"
    key            = "environments/production/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-locks"
    encrypt        = true
  }
}
```

```bash
terraform init
terraform init -backend-config="bucket=my-company-tf-state"
```

```hcl
data "terraform_remote_state" "network" {
  backend = "s3"

  config = {
    bucket = "my-company-terraform-state-prod"
    key    = "environments/production/network/terraform.tfstate"
    region = "us-east-1"
  }
}
```

---

## Key ideas

- Terraform state is critical because it maps configuration to real infrastructure. [web:654]
- Backends define how and where that state is stored. [web:653]
- Remote backends improve collaboration, reliability, and safety. [web:654][web:598]
- Locking, encryption, versioning, and access control are important backend features. [web:599][web:601]
- Remote state outputs can be shared across Terraform configurations with `terraform_remote_state`. [web:717]