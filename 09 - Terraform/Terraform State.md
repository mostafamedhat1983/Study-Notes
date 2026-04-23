---
tags:
  - Terraform
---
Terraform state is one of the most important Terraform concepts.

Terraform uses the state file to keep track of the infrastructure it manages.

Without state, Terraform would not reliably know:
- what resources already exist
- what it created before
- what needs to change
- what should be removed

---

## Main idea

Terraform stores information about managed infrastructure in a state file.

By default, this file is usually:

```bash
terraform.tfstate
```

Terraform uses this file to map:
- resources in your code
- real infrastructure in the target platform

Simple idea:
- configuration = desired state
- real cloud resources = actual state
- Terraform state file = Terraform's record of what it manages

---
## What the state file contains

The Terraform state file stores Terraform’s recorded view of the infrastructure it manages.

It commonly contains:
- resource IDs
- resource attributes
- output values
- dependency and metadata information
- provider-related state details
- sometimes sensitive values

This helps Terraform:
- map configuration to real infrastructure
- detect changes
- plan updates correctly
- track what it already manages

Because the state file may contain sensitive information, it should be stored and protected carefully.

---

## Why this matters

The state file is important because Terraform depends on it to understand the current infrastructure.

If the state is lost, corrupted, or exposed:
- Terraform may behave incorrectly
- recovery can become difficult
- sensitive information may be leaked

That is why remote backends, encryption, locking, and controlled access are important in real projects.
## Why Terraform needs state

Terraform is declarative.

You describe the final infrastructure you want, and Terraform figures out what actions are needed.

To do that safely, Terraform must know:
- what resources already exist
- which resources belong to the current configuration
- what attributes were created previously
- what changed since the last apply

That is why state is required.

---

## What state contains

Terraform state usually contains information such as:
- managed resource mappings
- resource metadata
- dependency information
- provider-related details
- output values

The state file helps Terraform connect your configuration to real infrastructure objects.

---

## Default local state

By default, Terraform stores state locally in the current working directory.

Example:

```bash
terraform.tfstate
```

This is fine for:
- small learning projects
- local experiments
- personal testing

But local state becomes risky in real team environments.

---

## Problems with local state

Local state can cause problems such as:
- team members using different copies of state
- accidental state loss
- difficulty collaborating
- no built-in locking
- higher chance of conflicting changes

This is why local state is usually not the best choice for shared environments.

---

## Remote state

A better approach for real projects is to store Terraform state remotely.

A remote backend stores the state in a shared location.

Common examples:
- AWS S3
- Azure Blob Storage
- Google Cloud Storage
- Terraform Cloud

Remote state helps with:
- collaboration
- centralized storage
- durability
- access control
- locking support in many backends

---

## Why remote state is better

Remote state is usually better because it helps:
- keep one shared source of truth
- avoid multiple local copies
- support team workflows
- protect state better
- reduce conflicts

For production or team use, remote state is usually the preferred approach.

---

## State locking

State locking helps prevent multiple Terraform operations from modifying the same state at the same time.

This matters when:
- two team members run `terraform apply` together
- a CI pipeline runs while someone is applying changes manually
- multiple processes try to update the same infrastructure at once

Without locking, state can become inconsistent or corrupted.

---

## Example idea of locking

A common AWS pattern is:
- store state in S3
- use DynamoDB for locking

This allows one Terraform operation to hold the lock while others wait.

The exact backend setup depends on the platform you use.

---

## State versioning

Versioning is useful when the backend supports it.

For example, if you use a storage service with object versioning, you can:
- keep old state versions
- recover from mistakes
- review previous versions

This makes state management safer.

---

## State can contain sensitive data

Terraform state may include sensitive information depending on the resources and providers you use.

That is why state should be handled carefully.

Important precautions:
- do not commit state files to Git
- use access control
- prefer encrypted remote storage
- avoid exposing state carelessly

Even if Terraform marks some values as sensitive in output, state may still contain important details.

---

## Remote backend example

Example of an S3 backend:

```hcl
terraform {
  backend "s3" {
    bucket = "my-terraform-state-bucket"
    key    = "network/dev/terraform.tfstate"
    region = "us-east-1"
  }
}
```

This tells Terraform to store state remotely in S3.

In a real setup, you may also add locking and encryption-related configuration depending on your backend design.

---

## State workflow idea

A simple way to think about Terraform state is:

1. You write Terraform code
2. Terraform checks the current state
3. Terraform compares code with real infrastructure
4. Terraform plans the changes
5. Terraform applies the changes
6. Terraform updates the state file

State is updated as Terraform manages infrastructure.

---

## Useful state commands

### Show current state resources

```bash
terraform state list
```

Lists resources currently tracked in state.

---

### Show details for one resource

```bash
terraform state show aws_instance.web
```

Shows detailed state information for a specific resource.

---

### Pull remote state locally

```bash
terraform state pull
```

Pulls the current state data.

---

### Move a resource in state

```bash
terraform state mv
```

Used when you need to move resource addresses in state.

This is more advanced and should be used carefully.

---

### Remove a resource from state

```bash
terraform state rm
```

Removes a resource from Terraform state without deleting the real infrastructure object.

This is also an advanced command and should be used carefully.

---

## Important warning about state commands

Low-level `terraform state` commands are powerful.

They can be useful for:
- repair
- migration
- refactoring
- advanced troubleshooting

But they should be used carefully because incorrect state operations can make Terraform lose track of resources.

In normal workflows, prefer changing configuration and using standard Terraform commands like:
- `plan`
- `apply`
- `destroy`

---

## Best practices for Terraform state

- prefer remote state for real projects
- enable locking when possible
- enable versioning when possible
- protect access to state files
- do not commit state files to Git
- avoid manual editing of the state file
- treat state as important infrastructure data
- use separate state for separate environments or systems when appropriate

---

## State and environments

It is usually a good idea to separate state between environments such as:
- dev
- staging
- production

This reduces risk because:
- changes in one environment do not affect another
- the blast radius is smaller
- management is cleaner

This separation can be done in different ways depending on your project design.

---

## Common mistakes

- storing only local state in team projects
- committing `terraform.tfstate` to Git
- disabling or ignoring locking
- manually editing the raw state file
- letting multiple environments share state carelessly
- forgetting that state may contain sensitive information

---

## Important notes

- Terraform state is Terraform's record of managed infrastructure.
- By default, state is stored locally in `terraform.tfstate`.
- Remote state is better for collaboration and shared environments.
- State locking helps prevent concurrent modification problems.
- State files may contain sensitive data.
- Versioning and encryption improve safety.
- `terraform state` commands are powerful and should be used carefully.

---

## Simple rule of thumb

- local state is acceptable for learning
- remote state is better for real projects
- never treat the state file like an unimportant temp file
- protect it, lock it, and avoid editing it manually

---

## Must memorize

```bash
terraform.tfstate
terraform state list
terraform state show aws_instance.web
terraform state pull
terraform state mv
terraform state rm
```

```hcl
terraform {
  backend "s3" {
    bucket = "my-terraform-state-bucket"
    key    = "network/dev/terraform.tfstate"
    region = "us-east-1"
  }
}
```

---

## Key ideas

- Terraform state keeps track of managed infrastructure.
- State helps Terraform map code to real resources.
- Local state is simple but weak for team usage.
- Remote state is better for collaboration and reliability.
- State locking helps prevent conflicting updates.
- State may contain sensitive information and must be protected.
- State commands are useful but should be handled carefully.