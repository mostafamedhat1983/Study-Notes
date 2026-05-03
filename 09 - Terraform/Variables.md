---
tags:
  - Terraform
---
Terraform variables are used to make configuration more flexible and reusable.

Instead of hardcoding values like:
- region
- instance type
- project name
- environment name

you can define variables and give them values later.

This makes Terraform code:
- cleaner
- easier to reuse
- easier to maintain
- better for multiple environments

---

## Main idea

Variables let you replace fixed values with reusable inputs.

Instead of writing the same value directly in many places, you define a variable once and reference it where needed.

Simple idea:
- variable = input to Terraform configuration
- hardcoded value = fixed value written directly in the code

---

## What is a variable

A Terraform input variable is a named value that can be used in configuration.

You define it with a `variable` block.

Example:

```hcl
variable "region" {
  type    = string
  default = "us-east-1"
}
```

You can then use it like this:

```hcl
provider "aws" {
  region = var.region
}
```

---

## Why variables are useful

Variables are useful because they help you:
- avoid hardcoding values
- reuse the same configuration in different environments
- make code easier to understand
- separate logic from environment-specific values
- pass custom values at runtime

For example:
- dev may use one instance size
- prod may use a larger instance size
- both can still use the same Terraform code

---

## Basic variable syntax

A variable is defined with a `variable` block.

Example:

```hcl
variable "instance_type" {
  type    = string
  default = "t2.micro"
}
```

This defines:
- variable name = `instance_type`
- type = `string`
- default value = `t2.micro`

---

## Referencing variables

To use a variable, Terraform uses the `var.` prefix.

Example:

```hcl
var.instance_type
```

Example inside a resource:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = var.instance_type
}
```

---

## Common variable arguments

### `type`

Specifies the expected data type.

Example:

```hcl
variable "instance_count" {
  type = number
}
```

Using types helps Terraform validate input values.

If `type` is not specified, Terraform defaults the variable to `any`. This means Terraform will accept any value without type validation. This works, but it is better to always specify a type for clarity and safety.

---

### `default`

Sets a default value.

Example:

```hcl
variable "region" {
  type    = string
  default = "us-east-1"
}
```

If a variable has a default value, it becomes optional.

If a variable has no default value, Terraform expects a value to be provided.

---

### `description`

Adds a description for the variable.

Example:

```hcl
variable "region" {
  type        = string
  description = "AWS region for deployment"
}
```

This makes the configuration easier to understand.

---

### `sensitive`

Marks the variable as sensitive.

Example:

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}
```

This helps Terraform avoid showing the value directly in normal CLI output.

---

### `validation`

Defines custom validation rules.

Example:

```hcl
variable "instance_count" {
  type = number

  validation {
    condition     = var.instance_count > 0
    error_message = "instance_count must be greater than 0."
  }
}
```

This helps catch invalid values early.

## Validation condition examples

The `condition` inside a `validation` block can use many built-in functions.

### `contains`

Checks if a value exists in a list.

```hcl
variable "environment" {
  type = string

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be dev, staging, or prod."
  }
}
```

---

### `startswith`

Checks if a string starts with a specific prefix.

```hcl
variable "bucket_name" {
  type = string

  validation {
    condition     = startswith(var.bucket_name, "myapp-")
    error_message = "bucket_name must start with myapp-."
  }
}
```

---

### `endswith`

Checks if a string ends with a specific suffix.

```hcl
variable "region" {
  type = string

  validation {
    condition     = endswith(var.region, "-1")
    error_message = "region must end with -1."
  }
}
```

---

### `length`

Checks the length of a string or list.

```hcl
variable "project_name" {
  type = string

  validation {
    condition     = length(var.project_name) > 0
    error_message = "project_name must not be empty."
  }
}
```

---

### `can` and `regex`

`can` tests if an expression runs without error.
`regex` matches a string against a pattern.

Used together to validate string format.

```hcl
variable "ami_id" {
  type = string

  validation {
    condition     = can(regex("^ami-[a-z0-9]+$", var.ami_id))
    error_message = "ami_id must start with ami- followed by alphanumeric characters."
  }
}
```

Simple idea:
- `regex` does the pattern match
- `can` catches the error if the match fails, so Terraform returns your `error_message` instead of crashing

---

### Combining conditions

You can combine conditions using `&&` (and) or `||` (or).

```hcl
variable "instance_count" {
  type = number

  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 10
    error_message = "instance_count must be between 1 and 10."
  }
}
```
---

## Common variable types

### `string`

Stores text values.

Example:

```hcl
variable "region" {
  type    = string
  default = "us-east-1"
}
```

---

### `number`

Stores numeric values.

Example:

```hcl
variable "instance_count" {
  type    = number
  default = 2
}
```

---

### `bool`

Stores `true` or `false`.

Example:

```hcl
variable "create_bucket" {
  type    = bool
  default = true
}
```

---

### `list`

Stores an ordered list of values.

Example:

```hcl
variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b"]
}
```

Access example:

```hcl
var.availability_zones
```

---

### `map`

Stores key-value pairs.

Example:

```hcl
variable "instance_types" {
  type = map(string)
  default = {
    dev  = "t2.micro"
    prod = "t3.medium"
  }
}
```

Access example:

```hcl
var.instance_types["dev"]
```

---

## Example variable file

A common convention is to define variables in `variables.tf`.

Example:

```hcl
variable "region" {
  type        = string
  description = "AWS region"
  default     = "us-east-1"
}

variable "instance_type" {
  type        = string
  description = "EC2 instance type"
  default     = "t2.micro"
}
```

---

## Example usage in resources

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = var.instance_type
}
```

```hcl
provider "aws" {
  region = var.region
}
```

This means the resource and provider use values from variables instead of fixed values.

---

## Ways to assign variable values

Terraform variables can get values in several ways.

Common ways include:
- default values in the `variable` block
- `terraform.tfvars`
- `.auto.tfvars`
- command-line flags
- environment variables

---

## Using `terraform.tfvars`

A common way to assign values is with a `terraform.tfvars` file.

Example:

```hcl
region        = "us-west-2"
instance_type = "t3.micro"
```

Terraform automatically loads `terraform.tfvars` if it exists in the project directory.

This is useful for project-specific values.

---

## Using a custom `.tfvars` file

You can also use your own variable file.

Example:

```bash
terraform plan -var-file="dev.tfvars"
```

This is useful when you have different environments such as:
- dev
- staging
- production

---

## Using command-line variables

You can pass values directly on the command line.

Example:

```bash
terraform plan -var="region=us-east-1"
```

This is useful for quick testing, but not ideal for large real projects.

---

## Using environment variables

Terraform can also read environment variables using this format:

```bash
TF_VAR_region=us-east-1
```

Example:

```bash
export TF_VAR_region=us-east-1
terraform plan
```

This can be useful in automation and CI/CD pipelines.

---

## Required vs optional variables

A variable is usually:
- **optional** if it has a `default`
- **required** if it does not have a `default`

Example of a required variable:

```hcl
variable "project_name" {
  type = string
}
```

Terraform will ask for a value if none is provided from another source.

---

## Example with validation and sensitive value

```hcl
variable "db_password" {
  type        = string
  description = "Database password"
  sensitive   = true
}

variable "instance_count" {
  type    = number
  default = 1

  validation {
    condition     = var.instance_count > 0
    error_message = "instance_count must be greater than 0."
  }
}
```

---

## Common mistakes

- hardcoding values that should be variables
- forgetting to use `var.` when referencing a variable
- not defining a type when clarity matters
- storing sensitive values carelessly
- mixing environment-specific values directly into main configuration
- using too many CLI `-var` arguments instead of structured `.tfvars` files
- not specifying a type and relying on `any` without realizing it

---

## Good practices

- keep variable definitions in `variables.tf`
- keep environment-specific values in `.tfvars` files
- use `description` for clarity
- use `type` for validation and readability
- use `validation` for important rules
- mark secrets as `sensitive`
- avoid hardcoding values that may change between environments

---

## Important notes

- Variables make Terraform configurations reusable.
- Variables are referenced with `var.<name>`.
- Variables can have types, defaults, descriptions, validation rules, and sensitivity settings.
- Variables without defaults usually require a value.
- `terraform.tfvars` is a common way to assign variable values.
- Environment variables can be passed using the `TF_VAR_<name>` format.

---

## Simple rule of thumb

- use variables for values that may change
- use defaults for common values
- use `.tfvars` files for environment-specific values
- use `sensitive = true` for secrets
- use validation when bad input could cause problems

---

## Must memorize

```hcl
variable "region" {
  type    = string
  default = "us-east-1"
}
```

```hcl
var.region
```

```bash
terraform plan -var="region=us-east-1"
terraform plan -var-file="dev.tfvars"
```

```bash
export TF_VAR_region=us-east-1
```

```hcl
region = "us-west-2"
```

---

## Key ideas

- Variables make Terraform code flexible and reusable.
- A variable is defined with a `variable` block.
- Variables are referenced using `var.<name>`.
- Variables can be required or optional.
- Values can come from defaults, `.tfvars` files, CLI flags, or environment variables.
- Types, validation, and descriptions improve clarity and safety.
- Sensitive variables help reduce accidental exposure in output.