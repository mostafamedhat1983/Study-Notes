---
tags:
  - Terraform
---
Terraform can manage existing infrastructure, but sometimes that infrastructure was not originally created by Terraform.

In that case, Terraform provides ways to:
- import existing resources into state
- inspect state
- move state entries
- remove state entries
- repair state during refactoring or migration

These operations are powerful, but they should be used carefully.

---

## Main idea

Terraform normally creates and manages infrastructure through configuration and state together.

But real-world environments often already contain resources created manually or by other tools.

Terraform import and state operations help Terraform start tracking or reorganizing those resources.

Simple idea:
- import = bring an existing resource into Terraform state
- state operations = inspect or adjust Terraform's record of resources

---

## What `terraform import` does

`terraform import` tells Terraform to associate an existing real infrastructure object with a resource address in Terraform state.

Example:

```bash
terraform import aws_instance.web i-1234567890abcdef0
```

This means:
- `aws_instance.web` = Terraform resource address
- `i-1234567890abcdef0` = real provider resource ID

After import, Terraform knows that the existing infrastructure object belongs to that resource address.

---

## Important import idea

Import adds the resource to Terraform state.

It does **not** automatically create complete Terraform configuration for you.

That means after import:
- the resource is in state
- but you still need matching Terraform code
- otherwise future plans may show unexpected differences

This is one of the most important things to understand about import.

---

## Typical import workflow

A common import workflow looks like this:

1. write the Terraform resource block
2. run `terraform import`
3. inspect the imported resource
4. adjust the configuration until `terraform plan` looks correct

This is usually an iterative process.

---

## Example import

### Resource block

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
}
```

### Import command

```bash
terraform import aws_instance.web i-1234567890abcdef0
```

After that, Terraform associates the existing EC2 instance with `aws_instance.web` in state.

You may still need to refine the resource block so Terraform does not try to make unwanted changes.

---

## Why import is useful

Import is useful when:
- infrastructure already exists
- you want Terraform to start managing it
- resources were created manually before adopting Terraform
- resources need to be brought under Infrastructure as Code management

This is common in brownfield environments.

---

## What state operations are

Terraform state operations are commands that inspect or modify Terraform's state entries.

These are often used for:
- troubleshooting
- refactoring
- renaming resources
- splitting configurations
- fixing state mapping issues

They act on Terraform's record of infrastructure, not directly on the infrastructure itself.

---

## `terraform state list`

```bash
terraform state list
```

Shows all resource addresses currently tracked in state.

This helps you see what Terraform is managing.

---

## `terraform state show`

```bash
terraform state show aws_instance.web
```

Shows detailed state information for one tracked resource.

This is useful for:
- inspection
- troubleshooting
- confirming imported resource details
- understanding what Terraform currently knows

---

## `terraform state mv`

```bash
terraform state mv aws_instance.web aws_instance.app
```

Moves a state entry from one resource address to another.

This is very useful when:
- renaming resources
- moving resources into modules
- restructuring Terraform code

This changes the state mapping so Terraform does not think the old resource should be destroyed and a new one created unnecessarily.

---

## Example of `state mv`

Suppose you rename:

```hcl
resource "aws_instance" "web" {
}
```

to:

```hcl
resource "aws_instance" "app" {
}
```

Without updating state, Terraform may think:
- `web` should be destroyed
- `app` should be created

To avoid that, you can use:

```bash
terraform state mv aws_instance.web aws_instance.app
```

This keeps the existing resource associated with the new name.

---

## `terraform state rm`

```bash
terraform state rm aws_instance.web
```

Removes a resource from Terraform state without deleting the real infrastructure object.

This means:
- Terraform stops tracking the object
- the actual resource still exists in the provider

This can be useful when:
- you no longer want Terraform to manage a resource
- you want to re-import it differently
- state mapping needs to be cleaned up

Use it carefully.

---

## `terraform state pull`

```bash
terraform state pull
```

Pulls the current state data.

This is more common in advanced troubleshooting or automation scenarios.

---

## `terraform state push`

```bash
terraform state push
```

Pushes state data back.

This is an advanced operation and should be used very carefully.

In normal workflows, you usually do not need to push raw state manually.

---

## Refactoring with state operations

State operations become very useful during Terraform refactoring.

Examples:
- renaming resources
- moving resources into modules
- splitting one configuration into multiple projects
- reorganizing code while preserving real infrastructure

Without careful state handling, Terraform may think existing resources should be destroyed and recreated.

State operations help avoid that.

---

## Import vs create

This is an important difference.

### Create
Terraform creates a brand-new resource with `apply`.

### Import
Terraform starts tracking an already existing resource.

Simple rule:
- `apply` creates new resources
- `import` adopts existing resources into Terraform state

---

## Common mistakes with import

- expecting import to generate full Terraform code automatically
- importing a resource without writing matching configuration
- forgetting to run `terraform plan` after import
- importing to the wrong resource address
- assuming imported resources will immediately be perfectly aligned with code

---

## Common mistakes with state commands

- using `state rm` without understanding the consequences
- moving state to the wrong address
- editing or changing state carelessly in production
- forgetting that state operations affect Terraform tracking logic
- using advanced state commands without a backup or clear plan

---

## Good practices

- write the resource block before importing
- run `terraform plan` after import and adjust configuration
- use `state mv` when refactoring resource addresses
- use `state rm` only when you truly want Terraform to stop tracking something
- treat state operations as advanced tools
- make state changes carefully and intentionally

---

## Important notes

- Import brings existing infrastructure into Terraform state.
- Import does not fully generate Terraform configuration.
- `terraform state list` shows tracked resources.
- `terraform state show` displays detailed state information.
- `terraform state mv` changes resource addresses in state.
- `terraform state rm` removes tracking without deleting the real object.
- State operations are powerful and should be used carefully.

---

## Simple rule of thumb

- use `import` when the infrastructure already exists
- use `state mv` when refactoring resource names or module paths
- use `state rm` when Terraform should stop tracking something
- always verify the result with `terraform plan`

---

## Must memorize

```bash
terraform import aws_instance.web i-1234567890abcdef0
terraform state list
terraform state show aws_instance.web
terraform state mv aws_instance.web aws_instance.app
terraform state rm aws_instance.web
terraform state pull
terraform state push
```

---

## Key ideas

- Terraform import is used to adopt existing infrastructure.
- Import only adds the resource to state.
- Matching Terraform configuration is still required.
- State operations help inspect and reorganize Terraform's record of resources.
- `state mv` is useful during refactoring.
- `state rm` stops Terraform from tracking a resource without deleting it.
- State commands should be used carefully because they affect Terraform's understanding of infrastructure.