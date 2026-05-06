---
tags:
  - Terraform
---
Both `count` and `for_each` solve the same core problem: creating multiple resource instances from a single resource block without duplicating code.

Choosing the wrong one can cause unexpected infrastructure changes, especially when items are added or removed later.

---

## Why they exist

Without `count` or `for_each`, you would need to write one resource block per instance.

Both meta-arguments let Terraform manage multiple instances from one block, but they track those instances differently in state.

---

## `count`

`count` tells Terraform how many instances to create using a number.

Terraform tracks each instance by its **index** (0, 1, 2, ...).

Basic example:

```hcl
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-12345678"
  instance_type = "t2.micro"
}
```

This creates three instances tracked as:
- `aws_instance.web[0]`
- `aws_instance.web[1]`
- `aws_instance.web[2]`

Inside the block, you can reference the current index using `count.index`:

```hcl
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  tags = {
    Name = "web-${count.index}"
  }
}
```

---

## When to use `count`

Use `count` when:
- resources are nearly identical
- a fixed number or variable number is enough
- index-based addressing is acceptable
- you want to conditionally create a resource (0 or 1)

---

## Conditional creation with `count`

A very common pattern is using `count` to optionally create a resource:

```hcl
resource "aws_s3_bucket" "logs" {
  count  = var.create_logs_bucket ? 1 : 0
  bucket = "my-logs-bucket"
}
```

This means:
- if `var.create_logs_bucket` is true, create the resource
- if false, do not create it

This is one of the most common `count` patterns in real projects.

---

## `for_each`

`for_each` creates one instance per item in a **map** or **set**.

Terraform tracks each instance by its **key**, not by index.

Example with a set:

```hcl
resource "aws_s3_bucket" "env" {
  for_each = toset(["dev", "stage", "prod"])
  bucket   = "myapp-${each.key}-bucket"
}
```

This creates three buckets tracked as:
- `aws_s3_bucket.env["dev"]`
- `aws_s3_bucket.env["stage"]`
- `aws_s3_bucket.env["prod"]`

Inside the block, you can use:
- `each.key` — the map key or set value
- `each.value` — the map value (only when using a map)

Example with a map:

```hcl
resource "aws_security_group" "app" {
  for_each    = {
    web = "Web Security Group"
    db  = "DB Security Group"
  }
  name        = each.key
  description = each.value
}
```

---

## When to use `for_each`

Use `for_each` when:
- resources have distinct names or configurations
- you want stable resource addressing by key
- items may be added or removed independently
- you are iterating over a map or set of values

---

## `toset()` — why it is needed

`for_each` requires a **map** or a **set of strings**.

A plain list is not accepted.

If you have a list, you must convert it first:

```hcl
for_each = toset(["dev", "stage", "prod"])
```

`toset()` converts a list into a set, which `for_each` can use.

Note: sets do not allow duplicate values. Duplicates are removed automatically.

---

## Key difference: how instances are tracked in state

This is the most important real-world difference.

| | `count` | `for_each` |
|---|---|---|
| Tracked by | Index (number) | Key (string) |
| Reference | `resource[0]`, `resource[1]` | `resource["dev"]`, `resource["prod"]` |
| When item removed | All following indexes shift | Only that key is removed |

---

## The index-shift problem with `count`

Imagine you have three instances created with `count = 3`:
- `web[0]` → server A
- `web[1]` → server B
- `web[2]` → server C

If you remove server B (index 1), Terraform sees:
- `web[0]` → server A (unchanged)
- `web[1]` → now becomes server C (was index 2)
- `web[2]` → destroyed

Terraform may **destroy and recreate** `web[1]` because its index changed, even though you only wanted to remove one server.

With `for_each`, removing `"stage"` only removes that key. Other resources are untouched.

---

## Cannot use both together

You cannot use `count` and `for_each` on the same resource block.

Terraform will throw a validation error if you try.

---

## Plan-time requirement

Both `count` and `for_each` must be based on values that Terraform knows at **plan time**, before any remote actions happen.

If the value depends on the output of another resource that does not exist yet, Terraform cannot plan correctly.

Example of what to avoid:

```hcl
# Bad: count depends on a resource attribute that is unknown at plan time
resource "aws_instance" "web" {
  count = length(aws_autoscaling_group.main.instances)
}
```

Use known values like variables, locals, or static data sources instead.

---

## Collection type rules

- `count` accepts a **number**
- `for_each` accepts a **map** or a **set of strings** only
- `for_each` does **not** accept a plain list directly — use `toset()` to convert

---

## Migration warning

Switching from `count` to `for_each` on an existing resource changes the resource addresses in state.

For example:
- before: `aws_instance.web[0]`
- after: `aws_instance.web["app"]`

Terraform will see the old address as destroyed and the new address as created.

This means a **full replace** of the resource unless you use `terraform state mv` to rename the addresses manually.

Choose the right approach early to avoid this.

---

## Common mistakes

- using `count` when items have different names or configurations
- using a plain list with `for_each` without `toset()`
- removing an item from the middle of a `count`-based list without expecting replacements
- trying to use both `count` and `for_each` on the same block
- basing `count` or `for_each` on values unknown at plan time
- switching from `count` to `for_each` on a live resource without updating state

---

## Quick decision guide

| Situation | Use |
|---|---|
| Fixed number of identical resources | `count` |
| Optional resource (create or not) | `count = var.x ? 1 : 0` |
| Named resources with distinct values | `for_each` |
| Iterating over a map of configs | `for_each` |
| Items may be removed independently | `for_each` |
| Stable resource addresses needed | `for_each` |

---

## Rule of thumb

- Use `count` when **identity does not matter** and resources are interchangeable
- Use `for_each` when **identity matters** and each resource should be tracked by a stable name

Simple version:
- **number → count**
- **name → for_each**

---

## Must memorize

```hcl
# count — fixed number
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  tags = {
    Name = "web-${count.index}"
  }
}
```

```hcl
# count — conditional creation
resource "aws_s3_bucket" "logs" {
  count  = var.create_logs_bucket ? 1 : 0
  bucket = "my-logs-bucket"
}
```

```hcl
# for_each — set of strings
resource "aws_s3_bucket" "env" {
  for_each = toset(["dev", "stage", "prod"])
  bucket   = "myapp-${each.key}-bucket"
}
```

```hcl
# for_each — map
resource "aws_security_group" "app" {
  for_each    = {
    web = "Web SG"
    db  = "DB SG"
  }
  name        = each.key
  description = each.value
}
```

---

## Key ideas

- `count` creates index-based instances tracked as `resource[0]`, `resource[1]`
- `for_each` creates key-based instances tracked as `resource["key"]`
- removing an item from a `count` list can shift indexes and trigger unexpected replacements
- `for_each` gives each instance a stable identity
- `for_each` requires a map or set — use `toset()` for lists
- you cannot use both on the same block
- both values must be known at plan time
- switching between them on existing resources affects state addresses
