---
tags:
  - Terraform
---
Creating a module is as simple as creating a directory with `.tf` files inside it.
That does not mean the module is well designed.

A well-designed module requires deliberate decisions about structure, interface, scope, and long-term maintainability.

---

## When to build a module

Build a module when:

- a useful **abstraction** of your infrastructure can be identified — a group of resources that logically belong together and can be seen as a whole
- certain resources **always need to be created together** and strongly depend on each other — this is a technical boundary worth extracting into a module
- you want to **hide infrastructure details** from users — exposing only the necessary interface leads to better developer experience and easier long-term maintenance

Do not build a module just because you can. If the configuration is small or used only once, a module may add unnecessary complexity.

---

## Use `object` type variables

If multiple related values belong together, group them under an `object` type variable instead of separate flat variables.

Instead of:

```hcl
variable "instance_type" { type = string }
variable "ami"           { type = string }
variable "name"          { type = string }
```

Use:

```hcl
variable "instance_config" {
  type = object({
    instance_type = string
    ami           = string
    name          = string
  })
}
```

This makes the interface cleaner, easier to extend, and easier to understand which values belong to which service.

---

## Separate long-lived from short-lived infrastructure

Resources that rarely change should not be grouped with resources that change often.

Example — a database (stable) and an EC2 instance (changes often) should be in separate modules:

```hcl
module "database" {
  source = "./modules/database"
  # rarely changes
}

module "app_server" {
  source     = "./modules/ec2"
  db_address = module.database.endpoint
  # changes often
}
```

This prevents Terraform from touching stable infrastructure every time you update frequently changing parts.

---

## Do not cover every edge case

Modules should be **reusable blocks of infrastructure**.

Catering to edge cases goes against reusability because you are designing the module around rare scenarios instead of the common case. Trying to incorporate every edge case quickly leads to:
- highly complex modules
- difficult to configure interfaces
- hard to maintain code

Keep the module focused on the common use case.

---

## Do not expose every config option as a variable

Only expose the configuration options that are truly necessary.

A good module provides:
- enough variables to be configurable for its intended use cases
- a layer of abstraction that hides unnecessary internals from users

If you expose every single option, the module is no longer an abstraction — it is just a thin wrapper with no added value.

---

## Output as much information as possible

Even if you do not see an immediate use case, output resource attributes proactively.

- For **public modules**, output as much as possible because you do not know who will use it or what they will need
- For **internal modules**, align with your team's needs, but it is still better to output more than less

A missing output is a breaking change to add later. An unused output costs almost nothing.

---

## Define a stable input and output interface

Every variable and output you add creates **coupling** to the module.

- Renaming a variable is a breaking change for anyone using the old name
- Renaming an output is a breaking change for anyone referencing it
- The more coupling, the harder it is to evolve the module without breaking users

Design the interface carefully upfront.

Use **semantic versioning** when publishing modules:
- breaking interface changes → major version bump (`v2.0.0`)
- new features, backwards compatible → minor version bump (`v1.1.0`)
- bug fixes → patch version bump (`v1.0.1`)

---

## Extensively document variables and outputs

Every variable and output should have a clear `description`.

```hcl
variable "instance_config" {
  type = object({
    instance_type = string
    ami           = string
    name          = string
  })
  description = "Configuration for the EC2 instance including type, AMI, and name tag."
}

output "instance_id" {
  value       = aws_instance.web.id
  description = "The ID of the created EC2 instance."
}
```

Good documentation helps module users understand the interface without reading all the internals.

---

## Flat and composable module structure

Avoid deeply nested modules — modules that call modules that call modules.

Deeply nested structures are hard to trace, debug, and maintain.

Instead, keep a **flat structure** and compose modules by referencing one another from the root module:

```hcl
# root module composes everything
module "network" {
  source = "./modules/network"
}

module "app_server" {
  source    = "./modules/ec2"
  vpc_id    = module.network.vpc_id
  subnet_id = module.network.subnet_id
}
```

Each module does one job. The root module wires them together.

---

## Use `for_each` instead of `count` inside modules

`count` inside modules causes index-shift problems when items are added or removed from the middle of a list.

`for_each` with a `map` is more stable because each resource is tracked by key, not position.

```hcl
resource "aws_instance" "server" {
  for_each      = var.instances
  ami           = each.value.ami
  instance_type = each.value.instance_type
  tags = {
    Name = each.key
  }
}
```

---

## Use `dynamic` blocks for optional repeated nested blocks

Some resource arguments are nested blocks that may need to repeat or may be optional. `dynamic` blocks let you generate these blocks from a variable.

### Without `dynamic` — hardcoded, not reusable

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"

  ebs_block_device {
    device_name = "/dev/sdb"
    volume_size = 20
  }

  ebs_block_device {
    device_name = "/dev/sdc"
    volume_size = 40
  }
}
```

### With `dynamic` — flexible and reusable

```hcl
variable "ebs_volumes" {
  type = list(object({
    device_name = string
    volume_size = number
  }))
  default = []
}

resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"

  dynamic "ebs_block_device" {
    for_each = var.ebs_volumes
    content {
      device_name = ebs_block_device.value.device_name
      volume_size = ebs_block_device.value.volume_size
    }
  }
}
```

### How the caller controls it

```hcl
# attach two volumes
ebs_volumes = [
  { device_name = "/dev/sdb", volume_size = 20 },
  { device_name = "/dev/sdc", volume_size = 40 }
]

# attach no extra volumes at all
ebs_volumes = []
```

- If the list has 2 items → Terraform generates 2 `ebs_block_device` blocks
- If the list is empty → Terraform generates 0 blocks — no extra volumes attached
- The EC2 instance itself is still created either way

### How `dynamic` works

- `dynamic "block_name"` — the name must match the nested block you want to repeat
- `for_each` — the collection to iterate over
- `content` — defines what each generated block looks like
- inside `content`, use `block_name.value.field` to access the current item

`dynamic` + `default = []` together make a nested block **completely optional**.

---

## Validate inputs with custom conditions

Do not trust users to always pass valid values. Validate inputs explicitly.

```hcl
variable "instance_type" {
  type = string

  validation {
    condition     = contains(["t2.micro", "t2.small", "t3.micro"], var.instance_type)
    error_message = "instance_type must be one of: t2.micro, t2.small, t3.micro."
  }
}
```

Also validate outputs where possible to ensure the infrastructure the module creates meets its requirements.

---

## Make dependencies explicit via input variables

Avoid using `data` sources inside a module to fetch dependencies.

Data sources inside a module create **implicit dependencies** — they are not visible in the module's interface and are harder to trace.

Instead, require the information as an **input variable**:

```hcl
# avoid this inside a module
data "aws_vpc" "main" {
  default = true
}

# prefer this — explicit and visible in the interface
variable "vpc_id" {
  type        = string
  description = "The VPC ID where the instance will be deployed."
}
```

---

## Use `sensitive = true` for secret outputs

If a module outputs passwords, tokens, or keys, mark the output as sensitive.

```hcl
output "db_password" {
  value     = aws_db_instance.main.password
  sensitive = true
}
```

This prevents the value from being displayed in terminal logs or plan output.

---

## Use `try()` for safe optional attribute access

When accessing optional or deeply nested attributes in object variables, use `try()` to avoid errors if the attribute is missing.

```hcl
locals {
  volume_size = try(var.instance_config.volume_size, 20)
}
```

If `volume_size` is not provided, it defaults to `20` instead of throwing an error.

---

## Test your modules

Do not assume a module works just because `terraform plan` succeeds.

Options:
- `terraform test` — native testing introduced in Terraform 1.6
- Terratest — Go-based testing framework for Terraform modules

Testing confirms that the module actually creates the expected infrastructure, not just that the configuration is syntactically valid.

---

## Keep module scope narrow

Do not try to solve every problem with a single module. A module that does too many unrelated things is harder to understand, reuse, and maintain.

Keep each module focused on one clear responsibility and use the flat composable structure to connect modules from the root module.

---

## Module file structure

A well-structured module directory looks like this:
modules/ec2/  
├── main.tf # resources  
├── variables.tf # input variables  
├── outputs.tf # outputs  
└── README.md # documentation and usage examples


Always include a `README.md` with:
- what the module does
- input variable descriptions
- output descriptions
- at least one usage example

---

## Key ideas

- build modules when a clear abstraction or technical boundary exists
- group related inputs under `object` type variables
- separate stable infrastructure from frequently changing infrastructure
- do not cover every edge case — keep modules reusable
- expose only necessary variables — preserve the abstraction layer
- output as much information as possible, especially for public modules
- design a stable interface — renaming variables or outputs is a breaking change
- use semantic versioning for published modules
- document every variable and output with a `description`
- prefer flat and composable module structure over deeply nested modules
- use `for_each` over `count` inside modules for stable resource addressing
- use `dynamic` blocks for optional or repeated nested configurations
- validate all inputs — never trust users to always pass valid values
- make dependencies explicit through input variables, not hidden data sources
- mark sensitive outputs with `sensitive = true`
- use `try()` for safe access to optional attributes
- test modules to confirm they create the expected infrastructure
- keep each module focused on one narrow responsibility
- always include a `README.md` with usage examples