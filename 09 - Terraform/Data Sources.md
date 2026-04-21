---
tags:
  - Terraform
---
Terraform data sources are used to read information from existing infrastructure or external systems without creating or modifying those objects.

They are useful when you want Terraform to look up values such as:
- existing VPC IDs
- subnet IDs
- AMI IDs
- availability zones
- account information

Simple idea:
- resource = create or manage something
- data source = read information about something that already exists

---

## Main idea

A data source lets Terraform fetch information from a provider.

This is useful when:
- the infrastructure already exists
- another team created the resource
- you need dynamic lookup instead of hardcoding values
- you want to reference existing objects in your configuration

Data sources are read-only.

Terraform uses them to get information, not to create infrastructure.

---

## What is a data source

A data source is a special Terraform block used to query existing information.

Example:

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true

  owners = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}
```

This does not create an AMI.

It looks up an existing AMI and returns information about it.

---

## Why data sources are useful

Data sources are useful because they help you:
- avoid hardcoding values
- look up dynamic infrastructure details
- integrate with existing environments
- reuse already-created resources
- make configuration more flexible

For example, instead of hardcoding an AMI ID, you can ask Terraform to find the latest matching AMI.

---

## Data source vs resource

This is one of the most important Terraform distinctions.

### Resource
A resource creates, updates, or destroys infrastructure.

Example:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
}
```

### Data source
A data source reads existing information.

Example:

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
}
```

Simple rule:
- `resource` = manage infrastructure
- `data` = read infrastructure information

---

## Basic syntax

A data source uses the `data` block.

Example:

```hcl
data "aws_vpc" "main" {
  default = true
}
```

Here:
- `data` = block type
- `aws_vpc` = data source type
- `main` = local Terraform name

---

## Referencing a data source

You reference a data source using:

```hcl
data.<type>.<name>.<attribute>
```

Example:

```hcl
data.aws_vpc.main.id
```

This means:
- data source type = `aws_vpc`
- local name = `main`
- attribute = `id`

---

## Example: using an existing VPC

```hcl
data "aws_vpc" "main" {
  default = true
}
```

Then use it in a resource:

```hcl
resource "aws_subnet" "app" {
  vpc_id     = data.aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}
```

This means Terraform reads the existing default VPC and then uses its ID when creating a subnet.

---

## Example: looking up an AMI

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true

  owners = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}
```

Then use it in an EC2 instance:

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"
}
```

This is a very common Terraform pattern.

---

## Why dynamic lookups are better than hardcoding

Hardcoding values like AMI IDs can cause problems because:
- they change over time
- they may differ by region
- they become outdated
- they make code less reusable

Using a data source often makes the configuration more flexible and maintainable.

---

## Common use cases

Data sources are commonly used for:
- looking up existing VPCs
- reading subnet IDs
- finding AMIs
- getting availability zones
- reading caller identity or account info
- reading existing DNS zones
- referencing already-created infrastructure

---

## Data sources and existing infrastructure

Data sources are especially useful when you are working in environments where:
- some infrastructure already exists
- not everything should be managed by the same Terraform configuration
- your configuration depends on shared infrastructure created elsewhere

This helps Terraform integrate into real-world environments more easily.

---

## Example: availability zones

```hcl
data "aws_availability_zones" "available" {}
```

Then reference it:

```hcl
data.aws_availability_zones.available.names
```

This can be useful when you want to distribute resources across available zones.

---

## Example: caller identity

```hcl
data "aws_caller_identity" "current" {}
```

Then reference values such as:

```hcl
data.aws_caller_identity.current.account_id
```

This is useful for:
- account-aware naming
- IAM-related logic
- verification
- multi-account setups

---

## Data sources and dependencies

Terraform automatically handles many dependencies when one block references another.

For example, if a resource uses:

```hcl
data.aws_ami.ubuntu.id
```

Terraform understands that the data source must be read before that resource can be fully planned or applied.

---

## Common mistakes

- confusing a data source with a resource
- expecting a data source to create infrastructure
- hardcoding values when a lookup would be better
- using the wrong filter criteria
- assuming the lookup will always return exactly one expected result
- not checking whether an object already exists before deciding between `resource` and `data`

---

## Good practices

- use data sources for existing infrastructure
- use resources for infrastructure you want Terraform to manage
- prefer dynamic lookups when values change often
- use clear names for data sources
- keep filters specific enough to avoid ambiguous results
- avoid unnecessary hardcoding

---

## Important notes

- Data sources are read-only.
- Data sources let Terraform fetch existing information.
- Data sources are declared with the `data` block.
- Resources manage infrastructure, but data sources only read information.
- Data sources are often used for AMIs, VPCs, subnets, and account details.
- Data source references use the form `data.type.name.attribute`.

---

## Simple rule of thumb

- use a resource if Terraform should create or manage it
- use a data source if Terraform only needs to read it
- prefer data lookups when hardcoded values are likely to change

---

## Must memorize

```hcl
data "aws_vpc" "main" {
  default = true
}
```

```hcl
data.aws_vpc.main.id
```

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
}
```

```hcl
ami = data.aws_ami.ubuntu.id
```

```hcl
data "aws_availability_zones" "available" {}
```

```hcl
data "aws_caller_identity" "current" {}
```

---

## Key ideas

- Data sources are used to read existing infrastructure information.
- Data sources do not create or modify infrastructure.
- Resources manage infrastructure, while data sources read it.
- Data sources are useful for dynamic lookups and integration with existing environments.
- Common examples include AMIs, VPCs, subnets, and account data.
- Data references use the form `data.type.name.attribute`.