---
tags:
  - Terraform
---
Terraform uses a state file to track real infrastructure and map it to your configuration. A backend controls where that state is stored and how Terraform accesses it. 

By default, Terraform uses the local backend, which stores state on the machine running Terraform. Remote backends store state in a shared location, which is better for team use, automation, and safer state management. 

---

## Main idea

A backend is Terraform's state storage configuration.

Simple idea:
- state = Terraform's record of real infrastructure
- backend = where that state lives and how it is managed

If state is only local, collaboration becomes harder and state can be lost with the machine. Remote backends improve sharing, consistency, and operational safety. 

---

## Why remote state matters

Remote state is useful because it helps teams work from the same source of truth. It also reduces the risk of losing state files that only exist on a local laptop or server. 

Good remote backends often support:
- centralized storage
- locking
- encryption
- access control
- versioning

These features help prevent corruption, reduce conflicts, and improve recovery options. 

---

## Local backend

If you do not configure a backend, Terraform uses the local backend. In that setup, the state file is stored locally in the working directory environment rather than in a shared remote system. 

Local state can be acceptable for:
- personal labs
- small experiments
- short-lived learning projects

It is usually a weak choice for team environments, production systems, or CI/CD workflows. 

---

## Remote backends

A remote backend stores Terraform state outside the local machine. HashiCorp's backend documentation describes backends as the mechanism used to control where state is stored, and remote backends make collaboration and shared operations easier. 

Common remote backend options include:
- Amazon S3
- Azure Blob Storage
- Google Cloud Storage
- Terraform Cloud / remote backend

Different backends have different features, but the common goal is centralized and safer state management. 

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

The backend block tells Terraform where to store or read the state file. Backend configuration is part of Terraform settings, not a provider resource. 

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

This setup is widely used because S3 provides centralized storage, while DynamoDB locking helps prevent concurrent Terraform runs from corrupting state. 

---

## Initialize backend

After adding or changing backend configuration, run:

```bash
terraform init
```

Terraform initializes the backend during `terraform init`. HashiCorp's backend documentation notes that changing backend configuration requires reinitialization. 

---

## Partial backend config

Terraform supports partial backend configuration, which means some backend values can be supplied later instead of being fully hardcoded in the configuration.

This is useful when you do not want to commit all backend details directly in source files.

This is commonly used with `-backend-config` during initialization.

### Inline backend config

```bash
terraform init -backend-config="bucket=my-company-tf-state"
```

This lets you pass a backend setting directly in the command.

### Backend config file

```bash
terraform init -backend-config=backend.hcl
```

This tells Terraform to read backend settings from a separate file during initialization.

This is useful when backend values are:
- dynamic
- environment-specific
- sensitive
- kept outside the main Terraform configuration

Using a backend config file can make backend setup cleaner and easier to manage.

---

## Backend migration

Terraform supports changing backend configuration and migrating state from one backend to another. HashiCorp documents backend reconfiguration and migration as part of backend management. 

A common example is:
- start with local state
- create a remote backend
- reinitialize Terraform
- migrate the existing state

This is a normal path when a small project grows into a shared team workflow. 

---
### Migrate state to a new backend

```bash
terraform init -migrate-state
```

This reinitializes Terraform after backend changes and attempts to copy the existing state to the new backend.

Use this when moving state, such as from local storage to an S3 backend.
## Remote state data source

Terraform can also read outputs from another state file by using the `terraform_remote_state` data source. This allows one Terraform configuration to consume outputs produced by another configuration. 

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

This is commonly used when one stack, such as an application stack, needs values from another stack, such as a networking stack. 

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

This allows one Terraform project to reuse important values exposed by another project's outputs. 

---

## Security concerns

Terraform state can contain sensitive information, so backend security matters. Best-practice guidance commonly recommends strong access control, encryption at rest, encryption in transit, and avoiding public exposure of state storage. 

For example, S3 state buckets are commonly configured with:
- versioning
- server-side encryption
- blocked public access

Those controls help reduce risk and improve recoverability. 

---

## Locking

State locking helps prevent two Terraform operations from modifying the same state at the same time. This reduces the risk of corruption and conflicting updates. 

For AWS-based setups, DynamoDB is commonly used with S3 backends for locking. In general, choosing a backend with locking support is a strong operational practice. 

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

Versioning allows you to keep historical copies of the state file. This is useful if you need to inspect or recover from accidental changes. 

For S3-based backends, enabling bucket versioning is a common recommendation because Terraform state is a critical operational asset. 

---

## Environment separation

Best-practice guidance often recommends separate state files per environment or logical system. This reduces blast radius and keeps production, staging, and development more isolated. 

Examples:
- `environments/dev/terraform.tfstate`
- `environments/staging/terraform.tfstate`
- `environments/prod/terraform.tfstate`

This approach makes state easier to organize and safer to manage at scale. 

---

## Common mistakes

- storing important team state only on a local machine
- not enabling locking when the backend supports it
- not enabling encryption or versioning
- giving too many people direct access to state
- keeping all environments in one state file
- exposing state storage publicly
- assuming backend configuration can freely use normal input variables

Backend configuration has special rules, and examples for S3 backends often note that values must be hardcoded or passed through backend-specific configuration rather than normal Terraform variables. 

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

These practices are repeatedly recommended in Terraform backend and state-management guidance because state is one of the most sensitive operational files in a Terraform workflow. 

---

## Important notes

- Terraform state tracks real infrastructure.
- A backend controls where state is stored and accessed.
- Local backend is the default when no backend is configured.
- Remote backends are better for collaboration and automation.
- S3 with DynamoDB locking is a common AWS pattern.
- `terraform init` is used to initialize or reinitialize backend settings.
- `terraform_remote_state` can read outputs from another state file.

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

- Terraform state is critical because it maps configuration to real infrastructure.
- Backends define how and where that state is stored.
- Remote backends improve collaboration, reliability, and safety.
- Locking, encryption, versioning, and access control are important backend features.
- Remote state outputs can be shared across Terraform configurations with `terraform_remote_state`.
