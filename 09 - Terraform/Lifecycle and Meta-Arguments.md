---
tags:
  - Terraform
---
Terraform lifecycle settings and meta-arguments help control how resources are created, updated, replaced, and managed.

They are useful when the default Terraform behavior is not enough for your use case.

These features help you:
- control resource replacement behavior
- manage dependencies
- create multiple similar resources
- conditionally create resources
- ignore selected changes
- protect important resources

---

## Main idea

Terraform resources support extra configuration that changes how Terraform handles them.

Two important groups are:
- **meta-arguments**
- **lifecycle settings**

Simple idea:
- normal arguments define **what** the resource should look like
- meta-arguments and lifecycle settings influence **how** Terraform manages that resource

---

## What are meta-arguments

Meta-arguments are special arguments that can be used with many resource blocks.

They are not specific to one provider resource type.

Common Terraform meta-arguments include:
- `count`
- `for_each`
- `depends_on`
- `provider`
- `lifecycle`

These are used to control Terraform behavior across resources.

---

## What is `lifecycle`

`lifecycle` is a special block inside a resource that controls certain behaviors during create, update, and destroy operations.

It is commonly used for:
- safer replacements
- preventing accidental deletion
- ignoring selected changes

---

## `count`

`count` is used to create multiple instances of a resource.

Example:

```hcl
resource "aws_instance" "web" {
  count         = 2
  ami           = "ami-12345678"
  instance_type = "t2.micro"
}
```

This tells Terraform to create two instances.

You can reference them by index:

```hcl
aws_instance.web.id
aws_instance.web.id
```

---

## When `count` is useful

`count` is useful when:
- you want multiple very similar resources
- the number of resources is based on a variable
- index-based access is acceptable

Example:

```hcl
resource "aws_instance" "web" {
  count         = var.instance_count
  ami           = "ami-12345678"
  instance_type = "t2.micro"
}
```

---

## `for_each`

`for_each` is used to create multiple resources based on a map or set of values.

Example:

```hcl
resource "aws_s3_bucket" "app" {
  for_each = toset(["dev", "stage", "prod"])
  bucket   = "myapp-${each.key}-bucket"
}
```

This creates one bucket for each value.

`for_each` is often better than `count` when:
- resources have meaningful names
- you want stable references
- you want to manage items by key instead of index

---

## `count` vs `for_each`

Use `count` when:
- resources are almost identical
- index-based numbering is fine

Use `for_each` when:
- resources have distinct names
- key-based access is better
- you want more stable resource addressing

Simple rule:
- `count` = index-based repetition
- `for_each` = key-based repetition

---

## `depends_on`

Terraform usually detects dependencies automatically when one resource references another.

But sometimes you need to define a dependency explicitly.

Example:

```hcl
resource "aws_instance" "app" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"

  depends_on = [aws_security_group.app_sg]
}
```

This tells Terraform to make sure the security group is handled before the instance.

---

## When `depends_on` is useful

Use `depends_on` when:
- Terraform cannot infer the dependency clearly
- the dependency is behavioral, not directly referenced
- ordering matters but there is no direct attribute reference

Use it carefully.

If Terraform can infer the dependency naturally, that is usually better.

---

## `provider` meta-argument

The `provider` meta-argument lets a resource use a specific provider configuration.

This is useful when you have multiple provider configurations, such as:
- multiple AWS regions
- multiple AWS accounts
- multiple Kubernetes clusters

Example idea:

```hcl
resource "aws_instance" "web" {
  provider      = aws.us_east_1
  ami           = "ami-12345678"
  instance_type = "t2.micro"
}
```

This tells the resource to use a specific configured provider alias.

---

## `lifecycle` block

A `lifecycle` block controls special lifecycle behavior for a resource.

Example:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"

  lifecycle {
    create_before_destroy = true
  }
}
```

---

## `create_before_destroy`

This tells Terraform to create the replacement resource before destroying the old one when replacement is required.

Example:

```hcl
lifecycle {
  create_before_destroy = true
}
```

This can help reduce downtime.

It is useful when:
- replacing a resource safely matters
- temporary overlap is acceptable
- you want better availability during replacements

---

## `prevent_destroy`

This tells Terraform to refuse destruction of a resource.

Example:

```hcl
lifecycle {
  prevent_destroy = true
}
```

This is useful for very important resources such as:
- production databases
- critical storage
- important networking resources

It adds protection against accidental deletion.

---

## `ignore_changes`

This tells Terraform to ignore changes to selected attributes.

Example:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"

  lifecycle {
    ignore_changes = [tags]
  }
}
```

This is useful when:
- some attributes are changed outside Terraform
- a platform automatically updates some fields
- you do not want Terraform to keep trying to revert specific values

Use this carefully because ignored changes reduce Terraform's control over that part of the resource.

---

## Example with lifecycle

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"

  lifecycle {
    create_before_destroy = true
    ignore_changes        = [tags]
  }
}
```

This means:
- create the replacement first if replacement is needed
- ignore changes to tags

---

## Conditional creation with `count`

A common pattern is using `count` to conditionally create a resource.

Example:

```hcl
resource "aws_s3_bucket" "logs" {
  count  = var.create_logs_bucket ? 1 : 0
  bucket = "my-logs-bucket"
}
```

This means:
- create the resource if the condition is true
- do not create it if the condition is false

This is a common Terraform pattern.

---

## Example with `for_each`

```hcl
resource "aws_security_group" "app" {
  for_each = {
    web = "Web SG"
    db  = "DB SG"
  }

  name = each.key
}
```

This creates multiple resources using keys from the map.

---

## Why these features matter

Terraform's default behavior is often enough.

But lifecycle settings and meta-arguments become very useful when you need:
- safer production changes
- repeated resources
- conditional resources
- explicit dependencies
- more control over resource behavior

These features are very common in real Terraform projects.

---

## Common mistakes

- using `depends_on` when Terraform can already infer dependencies
- using `count` when `for_each` would be clearer
- using `ignore_changes` too broadly
- applying `prevent_destroy` without understanding its impact
- forgetting that `count` resources use index-based references
- making lifecycle behavior too complex without a good reason

---

## Good practices

- use `count` for simple repeated resources
- use `for_each` for named or key-based resources
- use `depends_on` only when necessary
- use `prevent_destroy` for very important resources
- use `ignore_changes` carefully and intentionally
- keep lifecycle behavior understandable
- prefer simple resource logic unless more control is truly needed

---

## Important notes

- Meta-arguments influence how Terraform manages resources.
- Common meta-arguments include `count`, `for_each`, `depends_on`, `provider`, and `lifecycle`.
- `count` creates index-based repeated resources.
- `for_each` creates key-based repeated resources.
- `depends_on` defines explicit dependencies.
- `create_before_destroy` helps reduce downtime during replacement.
- `prevent_destroy` adds protection against accidental deletion.
- `ignore_changes` tells Terraform to ignore selected drift or external updates.

---

## Simple rule of thumb

- use `count` for simple repetition
- use `for_each` for named repetition
- use `depends_on` only when needed
- use lifecycle rules to make resource behavior safer and more intentional

---

## Must memorize

```hcl
resource "aws_instance" "web" {
  count         = 2
  ami           = "ami-12345678"
  instance_type = "t2.micro"
}
```

```hcl
resource "aws_s3_bucket" "app" {
  for_each = toset(["dev", "stage", "prod"])
  bucket   = "myapp-${each.key}-bucket"
}
```

```hcl
depends_on = [aws_security_group.app_sg]
```

```hcl
lifecycle {
  create_before_destroy = true
}
```

```hcl
lifecycle {
  prevent_destroy = true
}
```

```hcl
lifecycle {
  ignore_changes = [tags]
}
```

---

## Key ideas

- Meta-arguments change how Terraform manages resources.
- `count` and `for_each` are used for repeated resources.
- `depends_on` defines explicit dependencies when needed.
- The `lifecycle` block controls special management behavior.
- `create_before_destroy` can reduce downtime.
- `prevent_destroy` protects important resources.
- `ignore_changes` can help when selected attributes should not be enforced by Terraform.