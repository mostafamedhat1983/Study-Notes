---
tags:
  - Terraform
---
When creating multiple EC2 instances, the shape of your input variable matters as much as which meta-argument you use.

Two common input shapes are `list(object)` and `map(object)`.

The shape you choose determines how you create the instances, how you reference them later, and what your outputs look like.

Simple rule:
- If the items have **names**, use `map(object)`
- If the items only have **order**, use `list(object)`

---

## What is `list(object)`

A `list(object)` is an ordered collection of objects.

Each item is accessed by its **index** (0, 1, 2, ...).

Example variable definition:

```hcl
variable "instances" {
  type = list(object({
    name          = string
    instance_type = string
    ami           = string
  }))
}
```

Example variable value:

```hcl
instances = [
  { name = "web",  instance_type = "t2.micro",  ami = "ami-12345678" },
  { name = "app",  instance_type = "t2.small",  ami = "ami-12345678" },
  { name = "db",   instance_type = "t2.medium", ami = "ami-12345678" }
]
```

---

## Creating EC2 instances with `list(object)` and `count`

```hcl
resource "aws_instance" "server" {
  count         = length(var.instances)
  ami           = var.instances[count.index].ami
  instance_type = var.instances[count.index].instance_type

  tags = {
    Name = var.instances[count.index].name
  }
}
```

Terraform creates:
- `aws_instance.server[0]` → web
- `aws_instance.server[1]` → app
- `aws_instance.server[2]` → db

---

## Referencing `count`-based instances

Inside the resource block, use `count.index` to access the current item:

```hcl
var.instances[count.index].name
var.instances[count.index].instance_type
```

Outside the resource block, reference a specific instance by index:

```hcl
aws_instance.server[0].id           # web instance ID
aws_instance.server[1].public_ip    # app instance public IP
aws_instance.server[2].private_ip   # db instance private IP
```

Use splat expression to get all values at once:

```hcl
aws_instance.server[*].id           # list of all instance IDs
aws_instance.server[*].public_ip    # list of all public IPs
```

---

## Output for `count`-based instances

```hcl
output "instance_ids" {
  value = aws_instance.server[*].id
}

output "instance_public_ips" {
  value = aws_instance.server[*].public_ip
}
```

Both outputs produce a **list**:

```
instance_ids = [
  "i-0abc123",
  "i-0abc456",
  "i-0abc789"
]
```

---

## What is `map(object)`

A `map(object)` is a collection of objects where each item has a **named key**.

Each item is accessed by its **key** (like "web", "app", "db").

Example variable definition:

```hcl
variable "instances" {
  type = map(object({
    instance_type = string
    ami           = string
  }))
}
```

Example variable value:

```hcl
instances = {
  web = { instance_type = "t2.micro",  ami = "ami-12345678" }
  app = { instance_type = "t2.small",  ami = "ami-12345678" }
  db  = { instance_type = "t2.medium", ami = "ami-12345678" }
}
```

Note: the key itself acts as the name, so you do not need a `name` field inside the object.

---

## Creating EC2 instances with `map(object)` and `for_each`

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

Terraform creates:
- `aws_instance.server["web"]`
- `aws_instance.server["app"]`
- `aws_instance.server["db"]`

---

## Referencing `for_each`-based instances

Inside the resource block, use `each.key` and `each.value`:

```hcl
each.key                      # the map key, e.g. "web"
each.value.instance_type      # the instance type for this item
each.value.ami                # the AMI for this item
```

Outside the resource block, reference a specific instance by key:

```hcl
aws_instance.server["web"].id           # web instance ID
aws_instance.server["app"].public_ip    # app instance public IP
aws_instance.server["db"].private_ip    # db instance private IP
```

---

## Output for `for_each`-based instances

```hcl
output "instance_ids" {
  value = { for name, instance in aws_instance.server : name => instance.id }
}

output "instance_public_ips" {
  value = { for name, instance in aws_instance.server : name => instance.public_ip }
}
```

Both outputs produce a **map**:

```
instance_ids = {
  "web" = "i-0abc123"
  "app" = "i-0abc456"
  "db"  = "i-0abc789"
}
```

This is much easier to read than a plain list because each ID is labeled by name.

---

## Side-by-side comparison

| | `list(object)` + `count` | `map(object)` + `for_each` |
|---|---|---|
| Input shape | Ordered list | Named map |
| Instance tracked by | Index (0, 1, 2) | Key ("web", "app", "db") |
| Reference inside block | `var.instances[count.index].field` | `each.value.field` |
| Reference outside block | `aws_instance.server[0].id` | `aws_instance.server["web"].id` |
| Output shape | List | Map |
| Removing one item | Can shift indexes of others | Only removes that key |
| Best for | Identical or interchangeable instances | Named or distinct instances |

---

## The index-shift problem with `list(object)`

If you remove the `app` instance (index 1) from the list:

Before:
- `server[0]` → web
- `server[1]` → app
- `server[2]` → db

After removing app:
- `server[0]` → web (unchanged)
- `server[1]` → db (was index 2, now shifted to index 1)
- `server[2]` → destroyed

Terraform may **destroy and recreate** `server[1]` (db) because its index changed, even though you only intended to remove app.

With `map(object)`, removing `"app"` only destroys `server["app"]`. The `"web"` and `"db"` instances are completely untouched.

---

## When to use each

Use `list(object)` when:
- instances are interchangeable or nearly identical
- order matters but names do not
- you are comfortable with index-based addressing

Use `map(object)` when:
- each instance has a meaningful name
- instances have different configurations
- you want stable resource addresses
- instances may be added or removed independently

---

## Rule of thumb

- **Items have names → `map(object)` + `for_each`**
- **Items only have order → `list(object)` + `count`**

---

## Must memorize

```hcl
# list(object) variable
variable "instances" {
  type = list(object({
    name          = string
    instance_type = string
    ami           = string
  }))
}

# using count
resource "aws_instance" "server" {
  count         = length(var.instances)
  ami           = var.instances[count.index].ami
  instance_type = var.instances[count.index].instance_type
  tags = {
    Name = var.instances[count.index].name
  }
}

# referencing
aws_instance.server[0].id
aws_instance.server[*].id        # all IDs as list
```

```hcl
# map(object) variable
variable "instances" {
  type = map(object({
    instance_type = string
    ami           = string
  }))
}

# using for_each
resource "aws_instance" "server" {
  for_each      = var.instances
  ami           = each.value.ami
  instance_type = each.value.instance_type
  tags = {
    Name = each.key
  }
}

# referencing
aws_instance.server["web"].id
{ for name, i in aws_instance.server : name => i.id }   # all IDs as map
```

---

## Key ideas

- `list(object)` uses indexes — good for ordered, interchangeable instances
- `map(object)` uses keys — good for named, distinct instances
- `count` pairs naturally with `list(object)`
- `for_each` pairs naturally with `map(object)`
- removing an item from a list can shift indexes and trigger unexpected replacements
- removing a key from a map only affects that one resource
- `list(object)` outputs are lists; `map(object)` outputs are maps
- map-based outputs are easier to read because each value is labeled by name
