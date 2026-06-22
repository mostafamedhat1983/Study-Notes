---
tags:
  - Terraform
---
Terraform data types define what kind of value a variable, local, or output can hold.

Using explicit types helps Terraform:
- validate inputs early
- catch mistakes before `apply`
- make modules easier to understand and reuse

---

## Categories

Terraform has three categories of data types:
- **Primitive** — single values: `string`, `number`, `bool`
- **Collection** — multiple values of the same type: `list`, `set`, `map`
- **Structural** — grouped values with mixed types: `object`, `tuple`

There is also a special type: `any`

---

## Primitive types

### `string`

Stores text.

```hcl
variable "region" {
  type    = string
  default = "us-east-1"
}
```

Used for: region names, AMI IDs, instance names, tags.

---

### `number`

Stores integers or decimals.

```hcl
variable "instance_count" {
  type    = number
  default = 2
}
```

Used for: counts, ports, disk sizes.

---

### `bool`

Stores `true` or `false`.

```hcl
variable "enable_monitoring" {
  type    = bool
  default = true
}
```

Used for: feature flags, toggling resources on or off.

---

## Collection types

All elements in a collection must share the same type.

### `list`

Ordered collection. Duplicates allowed. Access by index.

```hcl
variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b"]
}
```

Access example:

```hcl
var.availability_zones[0]   # "us-east-1a"
```

Used for: subnet IDs, availability zones, CIDR blocks.

---

### `set`

Unordered collection. No duplicates. No index access.

```hcl
variable "environments" {
  type    = set(string)
  default = ["dev", "staging", "prod"]
}
```

Commonly used with `for_each`:

```hcl
resource "aws_s3_bucket" "app" {
  for_each = var.environments
  bucket   = "myapp-${each.key}"
}
```

Simple rule:
- use `list` when order matters
- use `set` when you want unique values and order does not matter

---

### `map`

Key-value pairs. All values must be the same type. Access by key.

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
var.instance_types["dev"]   # "t2.micro"
```

Used for: environment-specific values, tags, label mappings.

---

## Structural types

Structural types allow mixed types inside the same variable.

### `object`

Named attributes, each with its own type.

```hcl
variable "server" {
  type = object({
    instance_type = string
    count         = number
    enable_backup = bool
  })
  default = {
    instance_type = "t2.micro"
    count         = 2
    enable_backup = true
  }
}
```

Access example:

```hcl
var.server.instance_type
var.server.count
```

Used for: complex module inputs, grouping related configuration values together.

---

### `tuple`

Ordered, fixed-length sequence. Each position can have a different type.

```hcl
variable "server_config" {
  type    = tuple([string, number, bool])
  default = ["t2.micro", 2, true]
}
```

Access by index:

```hcl
var.server_config[0]   # "t2.micro"
var.server_config[1]   # 2
var.server_config[2]   # true
```

Simple rule:
- use `list` when all elements are the same type
- use `tuple` when elements have different types and the length is fixed

---

## Special type

### `any`

Disables type enforcement. Terraform accepts any value without validation.

```hcl
variable "flexible_input" {
  type = any
}
```

Simple rule:
- avoid `any` in production
- use specific types for clarity and validation

---

## Types at a glance

| Type | Order | Duplicates | Mixed types | Access |
|------|-------|------------|-------------|--------|
| `list` | ✅ Yes | ✅ Yes | ❌ No | by index `[0]` |
| `set` | ❌ No | ❌ No | ❌ No | with `for_each` |
| `map` | ❌ No | keys unique | ❌ No | by key `["key"]` |
| `object` | N/A | N/A | ✅ Yes | by name `.attr` |
| `tuple` | ✅ Yes | ✅ Yes | ✅ Yes | by index `[0]` |

---

## Common mistakes

- using `any` when a specific type would work
- using `list` when uniqueness matters — use `set` instead
- using `map` when mixed value types are needed — use `object` instead
- using `tuple` when all elements are the same type — use `list` instead
- not defining a type at all, which defaults to `any` silently

---

## Good practices

- always declare a type for every variable
- use `object` for grouped module inputs
- use `map(string)` for tags and environment-specific values
- use `list(string)` for subnet IDs and availability zones
- use `set(string)` when working with `for_each`
- avoid `any` unless flexibility is genuinely needed

---

## Must memorize

```hcl
# Primitive
variable "name"  { type = string }
variable "count" { type = number }
variable "flag"  { type = bool   }

# Collection
variable "zones" { type = list(string) }
variable "tags"  { type = map(string)  }
variable "envs"  { type = set(string)  }

# Structural
variable "config" {
  type = object({
    size    = string
    count   = number
    enabled = bool
  })
}

variable "mixed" {
  type    = tuple([string, number, bool])
  default = ["t2.micro", 2, true]
}
```

---

## Key ideas

- Primitive types hold single values: `string`, `number`, `bool`.
- Collection types hold multiple values of the same type: `list`, `set`, `map`.
- Structural types hold mixed types: `object` (named attributes), `tuple` (indexed positions).
- `list` is ordered with duplicates; `set` is unordered with unique values.
- `map` uses string keys; `object` uses named attributes with different value types.
- `tuple` is like `list` but with fixed length and mixed types.
- Always declare types explicitly — do not rely on `any`.
