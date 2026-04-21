---
tags:
  - Terraform
---
Terraform locals are used to define named values inside your configuration.

They help you avoid repeating the same expressions or values in multiple places.

Locals are useful when you want to:
- simplify repeated expressions
- make configuration easier to read
- build derived values from variables or resources
- keep naming conventions consistent

---

## Main idea

A local value is like a temporary named value inside Terraform configuration.

It is not an input like a variable.

It is not an output shown after apply.

It is simply a reusable internal value.

Simple idea:
- variable = input
- local = internal helper value
- output = result

---

## Why locals are useful

Locals are useful because they help you:
- avoid duplication
- keep code cleaner
- centralize naming rules
- build values from multiple inputs
- improve readability

For example, instead of repeating the same naming pattern in many resources, you can define it once in a local and reuse it.

---

## Basic syntax

Locals are defined in a `locals` block.

Example:

```hcl
locals {
  environment = "dev"
  project     = "myapp"
}
```

You reference a local value with:

```hcl
local.environment
```

or

```hcl
local.project
```

---

## Simple example

```hcl
locals {
  instance_name = "web-dev"
}

resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"

  tags = {
    Name = local.instance_name
  }
}
```

This makes the resource use the local value instead of repeating the string directly.

---

## Locals vs variables

Locals and variables are different.

### Variable
A variable is an input provided from outside or from defaults.

Example:

```hcl
variable "environment" {
  type    = string
  default = "dev"
}
```

### Local
A local is derived or defined inside the Terraform configuration.

Example:

```hcl
locals {
  app_name = "myapp-${var.environment}"
}
```

Simple difference:
- variables are inputs
- locals are internal reusable values

---

## Building values from variables

Locals are often used to build values from variables.

Example:

```hcl
variable "project_name" {
  type    = string
  default = "demo"
}

variable "environment" {
  type    = string
  default = "dev"
}

locals {
  full_name = "${var.project_name}-${var.environment}"
}
```

Now you can reuse:

```hcl
local.full_name
```

This is cleaner than repeating the same expression in many places.

---

## Example with tags

A very common use of locals is shared tagging.

Example:

```hcl
locals {
  common_tags = {
    Project     = "demo"
    Environment = "dev"
    ManagedBy   = "Terraform"
  }
}
```

Then use it in resources:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"

  tags = local.common_tags
}
```

This keeps tagging consistent across resources.

---

## Example with naming convention

Locals are useful for naming standards.

Example:

```hcl
variable "project" {
  type    = string
  default = "demo"
}

variable "environment" {
  type    = string
  default = "dev"
}

locals {
  bucket_name = "${var.project}-${var.environment}-bucket"
}
```

Then use:

```hcl
resource "aws_s3_bucket" "app" {
  bucket = local.bucket_name
}
```

This makes naming easier to maintain.

---

## Multiple locals in one block

You can define multiple local values in one `locals` block.

Example:

```hcl
locals {
  project     = "demo"
  environment = "dev"
  owner       = "mostafa"
}
```

This is the most common pattern.

---

## Multiple locals blocks

Terraform also allows more than one `locals` block.

Example:

```hcl
locals {
  project = "demo"
}

locals {
  environment = "dev"
}
```

Terraform combines them.

But in practice, many people prefer keeping related locals grouped clearly.

---

## Using expressions in locals

Locals can contain expressions, not just fixed values.

Example:

```hcl
locals {
  instance_count = 2
  instance_name  = "web-${var.environment}"
}
```

They can also use:
- variables
- functions
- conditionals
- other expressions

This makes them very flexible.

---

## Locals with conditional expressions

Example:

```hcl
variable "environment" {
  type    = string
  default = "dev"
}

locals {
  instance_type = var.environment == "prod" ? "t3.medium" : "t2.micro"
}
```

Now the instance type changes based on the environment.

This is a good example of using locals to centralize logic.

---

## Referencing locals

Locals use the `local.` prefix.

Example:

```hcl
local.bucket_name
local.common_tags
local.instance_type
```

This is similar to:
- `var.` for variables
- `local.` for locals

---

## Where to put locals

A common convention is to place local values in a file like:

```bash
locals.tf
```

But this is only a convention.

Terraform loads all `.tf` files in the same directory together.

So locals can technically be placed in any `.tf` file.

---

## Common use cases

Locals are commonly used for:
- naming conventions
- common tags
- derived values
- repeated expressions
- conditional logic
- cleaner resource configuration

---

## Common mistakes

- using locals when a variable should be an input
- putting too much complex logic into locals
- creating too many locals with unclear names
- duplicating values that could be centralized in one local
- confusing `local.` with `var.`

---

## Good practices

- use locals for internal reusable values
- use variables for external inputs
- keep local names clear and descriptive
- centralize repeated naming and tagging patterns
- avoid unnecessary complexity in local expressions

---

## Important notes

- Locals are internal values inside Terraform configuration.
- Locals are defined with a `locals` block.
- Locals are referenced using `local.<name>`.
- Locals are useful for derived values and repeated expressions.
- Locals are not inputs like variables.
- Locals are not final results like outputs.

---

## Simple rule of thumb

- use variables for inputs
- use locals for internal reusable values
- use outputs for values you want to expose

---

## Must memorize

```hcl
locals {
  environment = "dev"
  project     = "demo"
}
```

```hcl
local.environment
local.project
```

```hcl
locals {
  full_name = "${var.project_name}-${var.environment}"
}
```

```hcl
locals {
  common_tags = {
    Project     = "demo"
    Environment = "dev"
    ManagedBy   = "Terraform"
  }
}
```

---

## Key ideas

- Locals are internal reusable values in Terraform.
- Locals help reduce duplication and improve readability.
- Locals are referenced with `local.<name>`.
- Locals are often built from variables and expressions.
- Locals are useful for naming, tags, and derived values.
- Variables are inputs, locals are internal helpers, and outputs are results.