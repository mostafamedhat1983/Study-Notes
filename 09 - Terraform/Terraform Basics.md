---
tags:
  - Terraform
---

Terraform is an Infrastructure as Code (IaC) tool used to define, provision, and manage infrastructure using configuration files.

Instead of creating infrastructure manually from the cloud console, you write code that describes what you want, and Terraform creates or updates it for you.

This makes infrastructure:
- repeatable
- version-controlled
- easier to review
- easier to automate
- more consistent

---

## Main idea

With Terraform, you describe the desired infrastructure in configuration files.

Terraform then compares:
- what you defined in code
- what already exists in real infrastructure

and figures out what changes are needed.

The normal Terraform workflow is:

```bash
terraform init
terraform plan
terraform apply
```

---

## What Terraform is

Terraform is a declarative Infrastructure as Code tool.

Declarative means:
- you describe the final desired state
- Terraform figures out the steps needed to reach that state

For example, instead of writing:
- create VPC
- create subnet
- create instance

you define the infrastructure in configuration files, and Terraform determines what to create, update, or delete.

Terraform can manage:
- cloud infrastructure
- networking
- compute
- storage
- DNS
- SaaS resources
- some on-prem infrastructure

---

## Why Terraform is useful

Terraform is useful because it helps you:
- automate infrastructure creation
- avoid repetitive manual setup
- reduce human error
- reuse infrastructure definitions
- keep infrastructure in Git
- review changes before applying them
- manage infrastructure consistently across environments

This is especially useful for:
- DevOps workflows
- CI/CD pipelines
- cloud infrastructure provisioning
- multi-environment deployments like dev, staging, and production

---

## Important Terraform concepts

### Configuration files

Terraform uses configuration files written in HCL.

HCL stands for **HashiCorp Configuration Language**.

Terraform configuration files usually use the `.tf` extension.

Example:

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "demo" {
  bucket = "my-demo-bucket-12345"
}
```

---

### Provider

A provider is a plugin that lets Terraform interact with a platform or service.

Examples:
- AWS
- Azure
- Google Cloud
- Kubernetes
- GitHub

Without a provider, Terraform cannot manage infrastructure on that platform.

Example:

```hcl
provider "aws" {
  region = "us-east-1"
}
```

---

### Resource

A resource is an infrastructure object managed by Terraform.

Examples:
- EC2 instance
- S3 bucket
- VPC
- DNS record
- Kubernetes namespace

Example:

```hcl
resource "aws_s3_bucket" "demo" {
  bucket = "my-demo-bucket-12345"
}
```

Here:
- `aws_s3_bucket` = resource type
- `demo` = local Terraform name

---

### State

Terraform keeps track of managed infrastructure in a **state file**.

By default, this is stored in:

```bash
terraform.tfstate
```

The state file helps Terraform:
- map real infrastructure to configuration
- track metadata
- know what already exists
- determine what changes are needed

State is very important in Terraform.

---

## Terraform workflow

The core Terraform workflow is:

1. `terraform init`
2. `terraform plan`
3. `terraform apply`

---

### `terraform init`

```bash
terraform init
```

Initializes the working directory.

This usually does things like:
- download required providers
- initialize backend configuration
- prepare the working directory

You normally run this first in a new Terraform project.

---

### `terraform plan`

```bash
terraform plan
```

Shows what Terraform is going to do before it changes anything.

This helps you preview:
- resources to be created
- resources to be changed
- resources to be destroyed

This is one of the most important safety features in Terraform.

---

### `terraform apply`

```bash
terraform apply
```

Applies the planned changes to real infrastructure.

Terraform usually shows the execution plan and asks for confirmation before making changes.

After apply, Terraform updates the state file.

---

### `terraform destroy`

```bash
terraform destroy
```

Removes the infrastructure managed by the current Terraform configuration.

Use this carefully, because it can delete real infrastructure resources.

---

## Simple example

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "demo" {
  bucket = "my-demo-bucket-12345"
}
```

### What this does

- configures the AWS provider
- tells Terraform to create an S3 bucket
- names the resource `demo` inside Terraform

---

## Typical project flow

A normal Terraform workflow often looks like this:

1. Write or update `.tf` files
2. Run `terraform init`
3. Run `terraform plan`
4. Review the changes
5. Run `terraform apply`
6. Terraform updates infrastructure and state

---

## Why state matters

Terraform does not just create resources blindly.

It uses the state file to understand:
- what infrastructure it already manages
- what changed since the last run
- what needs to be created, updated, or deleted

Without state, Terraform would not reliably know how your real infrastructure maps to your code.

---

## Declarative vs imperative

Terraform is declarative.

That means:
- you define the desired end state
- Terraform decides how to get there

This is different from imperative tools, where you specify each step one by one.

### Easy mental model

- imperative = tell the system **how** to do everything
- declarative = tell the system **what** final result you want

Terraform is mostly about describing the final infrastructure state.

---

## Benefits of Terraform

- Infrastructure is stored as code
- Changes can be reviewed before apply
- Configurations can be reused
- Infrastructure can be version-controlled in Git
- Environments can be created consistently
- Automation becomes easier in CI/CD

---

## Important notes

- Terraform uses providers to manage infrastructure platforms.
- Terraform resources represent actual infrastructure objects.
- Terraform is declarative, not imperative.
- Terraform uses a state file to track managed resources.
- `terraform plan` helps you preview changes before applying them.
- `terraform apply` makes real infrastructure changes.
- `terraform destroy` removes managed infrastructure.

---

## Must memorize

```bash
terraform init
terraform plan
terraform apply
terraform destroy
```

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "demo" {
  bucket = "my-demo-bucket-12345"
}
```

---

## Key ideas

- Terraform is an Infrastructure as Code tool.
- You define infrastructure in `.tf` files.
- Terraform uses providers to interact with platforms like AWS.
- Resources are the infrastructure objects Terraform manages.
- Terraform keeps track of infrastructure in a state file.
- The basic workflow is `init`, `plan`, and `apply`.
- Terraform helps make infrastructure repeatable, reviewable, and automated.