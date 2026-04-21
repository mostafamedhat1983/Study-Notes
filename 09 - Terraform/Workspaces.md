---
tags:
  - Terraform
---
Terraform workspaces are used to manage multiple state instances for the same Terraform configuration.

They allow you to use one set of Terraform code while keeping separate state for different workspace names.

This can be useful when you want to separate environments such as:
- dev
- staging
- production

without duplicating the whole configuration.

---

## Main idea

A Terraform workspace is a named state instance for the same configuration.

This means:
- the Terraform code stays the same
- the state changes depending on the selected workspace

Simple idea:
- one configuration
- multiple workspaces
- separate state per workspace

---

## Default workspace

Terraform starts with a default workspace called:

```bash
default
```

If you do not create or select another workspace, Terraform uses the `default` workspace.

---

## Why workspaces are useful

Workspaces are useful because they help you:
- separate state for different environments
- reuse the same Terraform code
- avoid copying the same project folder multiple times
- manage similar deployments with different state

For example:
- `dev` workspace may manage development resources
- `prod` workspace may manage production resources

Both can use the same Terraform configuration but keep separate state.

---

## Important idea

Workspaces separate **state**, not necessarily all configuration logic by themselves.

If you want resources to behave differently per workspace, your code usually needs to use the workspace name in expressions or conditions.

For example:
- different names
- different instance sizes
- different tags

So workspaces do not automatically make the infrastructure different.

They mainly separate state.

---

## List workspaces

```bash
terraform workspace list
```

This shows all available workspaces.

The currently selected workspace usually appears with a marker.

---

## Create a workspace

```bash
terraform workspace new dev
```

This creates a new workspace called `dev`.

Terraform will then use a separate state for that workspace.

---

## Select a workspace

```bash
terraform workspace select dev
```

This switches Terraform to the `dev` workspace.

After that, commands like `plan` and `apply` use the state of that workspace.

---

## Show current workspace

```bash
terraform workspace show
```

This shows the currently selected workspace.

This is useful when you want to confirm which environment you are about to modify.

---

## Delete a workspace

```bash
terraform workspace delete dev
```

Deletes the workspace named `dev`.

Use this carefully.

You usually cannot delete the workspace you are currently using, and deleting a workspace should be done with care because it affects the Terraform state for that workspace.

---

## Workspace-aware expressions

Terraform exposes the current workspace name through:

```hcl
terraform.workspace
```

This allows configuration to behave differently based on the selected workspace.

Example:

```hcl
locals {
  environment = terraform.workspace
}
```

Now the local value changes depending on the selected workspace.

---

## Example naming with workspace

```hcl
resource "aws_s3_bucket" "app" {
  bucket = "myapp-${terraform.workspace}-bucket"
}
```

This can produce names like:
- `myapp-dev-bucket`
- `myapp-prod-bucket`

depending on the selected workspace.

This is a common pattern.

---

## Example conditional logic

```hcl
locals {
  instance_type = terraform.workspace == "prod" ? "t3.medium" : "t2.micro"
}
```

This allows different behavior depending on the workspace.

For example:
- production can use larger resources
- development can use smaller and cheaper ones

---

## How workspaces relate to state

Each workspace has its own state.

That means:
- resources tracked in `dev` state are separate from those in `prod` state
- Terraform plans and applies are based on the selected workspace
- switching workspace changes which state Terraform uses

This is the most important thing to understand about workspaces.

---

## Workspaces and remote state

Workspaces can also work with remote backends.

In those cases, the backend usually stores separate state data per workspace.

The exact behavior depends on the backend configuration.

The main idea still stays the same:
- same configuration
- separate state per workspace

---

## When workspaces are useful

Workspaces are often useful for:
- simple environment separation
- small to medium Terraform projects
- learning Terraform
- managing similar deployments without duplicating the whole project

---

## When workspaces are not enough

Workspaces are not always the best solution for every environment strategy.

In larger or more sensitive systems, teams often prefer stronger separation such as:
- separate directories
- separate repositories
- separate backend keys
- separate CI/CD pipelines

Why:
- clearer isolation
- lower risk
- better control for production environments

So workspaces are useful, but they are not always the only or best environment strategy.

---

## Common mistakes

- thinking workspaces automatically change all infrastructure behavior
- forgetting which workspace is currently selected
- applying production changes while still in the wrong workspace
- relying on workspaces alone for complex environment isolation
- using the same names without workspace-aware naming logic

---

## Good practices

- always check the current workspace before `apply`
- use `terraform.workspace` carefully in naming and logic
- keep workspace usage simple and predictable
- avoid overly complex workspace-based branching
- use stronger separation for critical production systems when needed

---

## Important notes

- Workspaces allow multiple state instances for the same configuration.
- Terraform starts with the `default` workspace.
- Workspaces mainly separate state, not all behavior automatically.
- `terraform.workspace` gives the current workspace name.
- Workspace selection affects which state Terraform reads and updates.
- Workspaces are useful, but they are not always the best environment-isolation strategy for large systems.

---

## Simple rule of thumb

- use workspaces when you want simple state separation with the same configuration
- do not assume workspaces alone solve all environment-isolation needs
- always verify the selected workspace before applying changes

---

## Must memorize

```bash
terraform workspace list
terraform workspace new dev
terraform workspace select dev
terraform workspace show
terraform workspace delete dev
```

```hcl
terraform.workspace
```

```hcl
locals {
  environment = terraform.workspace
}
```

```hcl
locals {
  instance_type = terraform.workspace == "prod" ? "t3.medium" : "t2.micro"
}
```

---

## Key ideas

- Terraform workspaces provide multiple state instances for one configuration.
- The default workspace is called `default`.
- Workspaces are mainly about state separation.
- `terraform.workspace` exposes the current workspace name.
- Workspace-aware logic can change naming and behavior.
- Workspaces are useful for simple environment separation.
- For larger systems, stronger environment isolation may be better.