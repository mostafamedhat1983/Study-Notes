---
tags:
  - Terraform
---
Terraform provides three ways to add custom validation logic beyond variable validation:
`precondition`, `postcondition`, and `check`.

Each runs at a different point and serves a different purpose.

---

## Why they exist

Variable `validation` only checks input values before anything runs.

But sometimes you need to validate:
- assumptions about existing infrastructure (before apply)
- results after a resource is created (after apply)
- ongoing health of infrastructure (independently of any resource)

That is what `precondition`, `postcondition`, and `check` are for.

---

## Quick comparison

| | `precondition` | `postcondition` | `check` |
|---|---|---|---|
| Lives inside | `lifecycle` block | `lifecycle` block | top-level `check` block |
| Runs | Before create/update | After create/update | After apply, independently |
| Validates | Assumptions going in | Results coming out | Ongoing assertions |
| On failure | Stops apply | Stops apply | Warning only (does not stop) |
| Can reference | `var`, `data`, other resources | `self` (the resource itself) | anything |

---

## `precondition`

A `precondition` checks an assumption **before** Terraform creates or updates a resource.

It lives inside a `lifecycle` block.

If the condition is false, Terraform stops the apply and shows your error message.

### Syntax

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type

  lifecycle {
    precondition {
      condition     = can(regex("^ami-[a-z0-9]+$", var.ami_id))
      error_message = "ami_id must start with ami- followed by alphanumeric characters."
    }
  }
}
```

### When to use it

Use `precondition` when:
- the resource depends on an assumption about an input or data source
- you want to catch a wrong assumption before any infrastructure changes happen
- the condition cannot be expressed as a variable `validation` because it involves data sources or other resources

### Real example — validating an AMI exists in the right region

```hcl
data "aws_ami" "app" {
  most_recent = true
  owners      = ["self"]

  filter {
    name   = "name"
    values = ["my-app-*"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.app.id
  instance_type = "t3.micro"

  lifecycle {
    precondition {
      condition     = data.aws_ami.app.architecture == "x86_64"
      error_message = "The selected AMI must be x86_64 architecture."
    }
  }
}
```

---

## `postcondition`

A `postcondition` checks the **result** after a resource is created or updated.

It also lives inside a `lifecycle` block.

Inside a `postcondition`, you use `self` to refer to the resource itself.

If the condition is false after the resource is created, Terraform marks the apply as failed.

### Syntax

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type

  lifecycle {
    postcondition {
      condition     = self.public_ip != ""
      error_message = "The EC2 instance must have a public IP address."
    }
  }
}
```

### When to use it

Use `postcondition` when:
- you need to verify what actually got created, not just what was requested
- an attribute of the created resource must meet a requirement
- you want early detection of unexpected infrastructure results

### Real example — ensuring an instance launched in the right AZ

```hcl
resource "aws_instance" "web" {
  ami               = var.ami_id
  instance_type     = "t3.micro"
  availability_zone = var.az

  lifecycle {
    postcondition {
      condition     = contains(["us-east-1a", "us-east-1b"], self.availability_zone)
      error_message = "Instance must be in us-east-1a or us-east-1b."
    }
  }
}
```

---

## `precondition` vs variable `validation`

These look similar but serve different purposes.

| | `validation` | `precondition` |
|---|---|---|
| Lives in | `variable` block | `lifecycle` block inside a resource |
| Checks | Input values only | Any value — inputs, data sources, other resources |
| Runs | Before plan | Before resource create/update |
| Has access to | Only `var.<name>` | `var`, `data`, other resources |

Simple rule:
- use `validation` to check that a variable value is acceptable
- use `precondition` to check that an assumption about the environment is true before touching a resource

---

## `check` block

A `check` block is a **top-level** block that runs assertions independently of any resource.

Unlike `precondition` and `postcondition`, a failed `check` does **not** stop the apply. It only produces a warning.

This makes it useful for ongoing health monitoring and soft assertions.

### Syntax

```hcl
check "instance_has_public_ip" {
  assert {
    condition     = aws_instance.web.public_ip != ""
    error_message = "Web instance should have a public IP."
  }
}
```

### With a data source

A `check` block can include an optional scoped `data` block to fetch data just for that assertion.

```hcl
check "ami_is_available" {
  data "aws_ami" "latest" {
    most_recent = true
    owners      = ["amazon"]

    filter {
      name   = "name"
      values = ["amzn2-ami-hvm-*"]
    }
  }

  assert {
    condition     = data.aws_ami.latest.id != ""
    error_message = "Could not find a valid Amazon Linux 2 AMI."
  }
}
```

### When to use it

Use `check` when:
- you want to assert something about your infrastructure without blocking apply on failure
- you want continuous health assertions that run after every apply
- you want to monitor assumptions about data sources or existing resources

---

## `precondition` and `postcondition` together

You can use both in the same resource.

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type

  lifecycle {
    precondition {
      condition     = can(regex("^ami-", var.ami_id))
      error_message = "ami_id must start with ami-."
    }

    postcondition {
      condition     = self.public_ip != ""
      error_message = "Instance must have a public IP after creation."
    }
  }
}
```

---

## Common mistakes

- using `postcondition` when `precondition` is the right choice — if the input is wrong, catch it before apply, not after
- referencing `self` in a `precondition` — `self` is only available in `postcondition`
- expecting `check` to stop apply on failure — it only warns
- putting complex logic in conditions that should be variable `validation`
- forgetting that `precondition` and `postcondition` both require a clear `error_message`

---

## Key ideas

- `precondition` checks assumptions before a resource is created or updated
- `postcondition` checks results after a resource is created or updated using `self`
- `check` is a top-level block that runs assertions independently and only warns on failure
- `precondition` and `postcondition` live inside `lifecycle` blocks
- `check` does not stop apply — it is for soft ongoing assertions
- use `validation` for variable inputs, `precondition` for environment assumptions, `postcondition` for resource results