---
tags:
  - Terraform
---
Filtering in Terraform means narrowing down data or collections to only the items you actually need.

It appears in two main places:
- `filter` block inside **data sources** — queries the cloud API
- `for` expression with `if` — filters a collection inside Terraform itself

---

## `filter` block in data sources

When you use a `data` block to look up existing cloud resources, a `filter` block tells the cloud API which specific resource to return.

Without a filter, the lookup may be too broad and return multiple results, which causes Terraform to fail.

Example — finding a specific AMI:

```hcl
data "aws_ami" "web" {
  most_recent = true
  owners      = ["self"]

  filter {
    name   = "name"
    values = ["my-web-ami-*"]
  }

  filter {
    name   = "tag:Environment"
    values = ["prod"]
  }
}
```

---

## AND vs OR logic in filters

- Multiple `filter` blocks = **AND** — all filters must match
- Multiple `values` inside one `filter` block = **OR** — any value can match

Example:

```hcl
filter {
  name   = "tag:Environment"
  values = ["prod", "staging"]   # OR — matches prod OR staging
}

filter {
  name   = "state"
  values = ["available"]         # AND — must also be available
}
```

---

## Referencing the filtered result

```hcl
data.aws_ami.web.id
data.aws_ami.web.name
```

If the filter returns more than one result, Terraform fails unless you use `most_recent = true`.

---

## `for` expression with `if` — filtering collections

You can filter a list or map inside Terraform using a `for` expression with an `if` condition.

Syntax:

```hcl
[for item in collection : item if condition]
```

Example — keep only `t2.micro` instances from a map:

```hcl
locals {
  small_instances = {
    for name, config in var.instances : name => config
    if config.instance_type == "t2.micro"
  }
}
```

Example — get only prod environment subnet IDs from a list:

```hcl
locals {
  prod_subnets = [
    for subnet in var.subnets : subnet.id
    if subnet.environment == "prod"
  ]
}
```

---

## Side-by-side comparison

| | `filter` block | `for` + `if` |
|---|---|---|
| Where used | `data` source blocks | locals, outputs, variables |
| What it filters | Cloud API results | Terraform collections (lists, maps) |
| Result | Filtered cloud resource | New filtered list or map |
| Common use | Find specific AMI, VPC, subnet | Subset of a variable or local |

---

## Common mistakes

- not adding `most_recent = true` when the filter may return multiple AMIs
- confusing AND and OR logic in filter blocks
- trying to use `filter` blocks outside of data sources
- forgetting that `for + if` produces a new collection, not a modified one

---

## Rule of thumb

- Use `filter` when querying **existing cloud resources** via data sources
- Use `for + if` when filtering **Terraform collections** like variables and locals

---

## Must memorize

```hcl
# filter block in data source
data "aws_ami" "web" {
  most_recent = true
  owners      = ["self"]

  filter {
    name   = "name"
    values = ["my-web-ami-*"]
  }

  filter {
    name   = "tag:Environment"
    values = ["prod"]
  }
}

# reference the result
data.aws_ami.web.id
```

```hcl
# for + if — filter a map
locals {
  small_instances = {
    for name, config in var.instances : name => config
    if config.instance_type == "t2.micro"
  }
}

# for + if — filter a list
locals {
  prod_subnets = [
    for subnet in var.subnets : subnet.id
    if subnet.environment == "prod"
  ]
}
```

---

## Key ideas

- `filter` blocks narrow down cloud API results inside data sources
- multiple `filter` blocks are AND logic — all must match
- multiple `values` inside one `filter` are OR logic — any can match
- `for + if` filters Terraform collections like lists and maps
- filtering with `for + if` produces a new collection, it does not modify the original
- always use `most_recent = true` when a filter may return multiple results