---
tags:
  - Terraform
---

Terraform modules are used to organize and reuse infrastructure code.

A module is a container for Terraform configuration.

It helps you avoid repeating the same resource definitions again and again.

Modules are useful when you want to:
- reuse infrastructure patterns
- keep code cleaner
- separate responsibilities
- standardize deployments
- make large Terraform projects easier to manage

---

## Main idea

A module is a reusable Terraform configuration.

Instead of writing the same resources many times, you can put them into a module and call that module whenever you need it.

Simple idea:
- resources = individual infrastructure objects
- module = reusable group of Terraform configuration

---

## What is a module

In Terraform, a module is a set of `.tf` files in a directory.

That means:
- every Terraform project is technically a module
- the main working directory is called the **root module**
- reusable modules called from the root module are called **child modules**

So:
- root module = the main Terraform configuration you run
- child module = a module used by another module

---

## Why modules are useful

Modules are useful because they help you:
- reduce duplication
- reuse infrastructure patterns
- make code more organized
- keep projects easier to maintain
- standardize resource creation
- separate environments and components more cleanly

For example, instead of writing EC2 + security group + IAM role repeatedly, you can create one module and reuse it.

---

## Root module

The Terraform configuration in the current working directory is the root module.

This is the module Terraform runs when you execute commands like:

```bash
terraform plan
terraform apply
```

The root module can:
- define resources directly
- call child modules
- pass variables to child modules
- use outputs from child modules

---

## Child module

A child module is a module called from another Terraform configuration.

Example:

```hcl
module "network" {
  source = "./modules/network"
}
```

This tells Terraform to load a module from the local path:

```bash
./modules/network
```

That directory contains reusable Terraform code.

---

## Module block syntax

A module is used with a `module` block.

Example:

```hcl
module "network" {
  source = "./modules/network"
}
```

Here:
- `module` = block type
- `network` = local name of the module inside Terraform
- `source` = where Terraform should load the module from

---

## Module source

The `source` argument tells Terraform where the module is located.

Common source types:
- local path
- Terraform Registry
- Git repository

### Local path example

```hcl
module "network" {
  source = "./modules/network"
}
```

### Registry example

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
}
```

### Git example

```hcl
module "app" {
  source = "git::https://github.com/example/terraform-app-module.git"
}
```

---

## Basic local module structure

A common local module structure looks like this:

```bash
project/
├── main.tf
├── variables.tf
├── outputs.tf
└── modules/
    └── network/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

Here:
- the top directory is the root module
- `modules/network` is a child module

---

## Passing variables into a module

Modules often accept input variables.

Example child module variable:

```hcl
variable "vpc_cidr" {
  type = string
}
```

Example root module call:

```hcl
module "network" {
  source   = "./modules/network"
  vpc_cidr = "10.0.0.0/16"
}
```

This passes the value into the child module.

---

## Using variables inside the module

Inside the child module, the variable can be used like this:

```hcl
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
}
```

So the root module provides the input, and the child module uses it.

---

## Module outputs

Modules can also return values using outputs.

Example child module output:

```hcl
output "vpc_id" {
  value = aws_vpc.main.id
}
```

Then the root module can use it like this:

```hcl
module.network.vpc_id
```

This is one of the most important module patterns.

Simple idea:
- input goes into the module using variables
- result comes out of the module using outputs

---

## Full small example

### Root module

```hcl
module "network" {
  source   = "./modules/network"
  vpc_cidr = "10.0.0.0/16"
}

output "network_vpc_id" {
  value = module.network.vpc_id
}
```

### Child module variable

```hcl
variable "vpc_cidr" {
  type = string
}
```

### Child module resource

```hcl
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
}
```

### Child module output

```hcl
output "vpc_id" {
  value = aws_vpc.main.id
}
```

---

## Module naming inside Terraform

In this example:

```hcl
module "network" {
  source = "./modules/network"
}
```

- `network` is the local Terraform name of the module
- it is used for references like `module.network.vpc_id`

This is similar to how resources have local names.

---

## Reusability example

A module becomes very useful when you want to reuse the same infrastructure pattern many times.

Examples:
- one module for VPC creation
- one module for EC2 app servers
- one module for security groups
- one module for S3 buckets with standard settings

This makes infrastructure more consistent.

---

## Module versioning

When using remote modules, especially from a registry, it is a good idea to pin a version.

Example:

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
}
```

This helps:
- keep builds predictable
- avoid unexpected module changes
- improve reproducibility

---

## `terraform init` and modules

When a configuration uses modules, `terraform init` also prepares those modules.

This may include:
- downloading remote modules
- setting up local module references
- preparing the working directory

That is why `terraform init` should be re-run when you add or change module sources.

---

## Best uses for modules

Modules are especially useful for:
- repeated infrastructure patterns
- shared organization standards
- large Terraform codebases
- multi-environment reuse
- separating networking, compute, storage, and security concerns

---

## When not to overuse modules

Modules are useful, but not everything needs to be a module.

If a configuration is very small or used only once, creating a separate module may add unnecessary complexity.

A good rule is:
- use modules when reuse or separation gives clear value
- avoid creating tiny modules for everything without a good reason

---

## Common mistakes

- creating modules too early for very small code
- not defining clear input variables
- not exposing useful outputs
- making modules too tightly coupled to one environment
- hardcoding values that should be variables
- forgetting to pin versions for remote modules

---

## Good practices

- keep modules focused on one responsibility
- use variables for module inputs
- use outputs for module results
- keep module names clear
- pin remote module versions
- document expected inputs and outputs
- avoid unnecessary complexity

---

## Important notes

- A module is a reusable Terraform configuration.
- The main working directory is the root module.
- Reusable called modules are child modules.
- Modules are called using a `module` block.
- `source` tells Terraform where the module comes from.
- Modules usually accept inputs through variables and return values through outputs.
- `module.<name>.<output>` is the common way to reference module outputs.

---

## Simple rule of thumb

- use resources for individual objects
- use modules for reusable infrastructure patterns
- use variables to pass values into modules
- use outputs to return values from modules

---

## Must memorize

```hcl
module "network" {
  source = "./modules/network"
}
```

```hcl
module "network" {
  source   = "./modules/network"
  vpc_cidr = "10.0.0.0/16"
}
```

```hcl
module.network.vpc_id
```

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
}
```

---

## Key ideas

- Modules help organize and reuse Terraform code.
- The current working directory is the root module.
- Called reusable modules are child modules.
- Modules take inputs with variables and return values with outputs.
- `source` defines where the module comes from.
- Modules are useful for repeated infrastructure patterns.
- Good modules improve maintainability and consistency.