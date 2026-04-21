---
tags:
  - Terraform
---
Terraform commands are used to initialize a project, preview changes, apply infrastructure, inspect state, and clean up resources.

They are the main way you interact with Terraform from the command line.

The most common day-to-day workflow is:
- `terraform init`
- `terraform plan`
- `terraform apply`

---

## Main idea

Terraform commands help you move from configuration files to real infrastructure.

A typical Terraform workflow looks like this:

1. write or update Terraform code
2. initialize the project
3. preview the changes
4. apply the changes
5. inspect or manage the state if needed

---

## `terraform init`

```bash
terraform init
```

Initializes the working directory.

This is usually the first command you run in a Terraform project.

It is commonly used to:
- download providers
- initialize backend configuration
- prepare modules
- set up the working directory

Run it:
- when starting a new project
- after changing backend settings
- after adding providers or modules

---

## `terraform validate`

```bash
terraform validate
```

Checks whether the Terraform configuration is syntactically valid and internally consistent.

This is useful for catching problems early before planning or applying changes.

It does not create infrastructure.

---

## `terraform fmt`

```bash
terraform fmt
```

Formats Terraform files into a standard style.

This helps:
- improve readability
- keep formatting consistent
- reduce style differences between team members

You can also use:

```bash
terraform fmt -recursive
```

to format Terraform files in subdirectories as well.

---

## `terraform plan`

```bash
terraform plan
```

Shows what Terraform intends to do before changing infrastructure.

This is one of the most important Terraform commands.

It helps you preview:
- resources to be created
- resources to be updated
- resources to be destroyed

This makes changes safer and easier to review.

---

## `terraform apply`

```bash
terraform apply
```

Applies the changes to real infrastructure.

Terraform usually shows the execution plan and asks for confirmation before proceeding.

After apply:
- infrastructure is created, updated, or destroyed as needed
- state is updated

---

## `terraform destroy`

```bash
terraform destroy
```

Destroys the infrastructure managed by the current Terraform configuration.

Use this carefully because it can remove real resources.

This is often useful for:
- labs
- test environments
- temporary infrastructure
- cleanup after experiments

---

## `terraform show`

```bash
terraform show
```

Displays information about the current state or a saved plan.

This can help when you want to inspect what Terraform currently knows.

---

## `terraform output`

```bash
terraform output
```

Shows output values defined in the Terraform configuration.

Example:

```bash
terraform output instance_id
```

This displays one specific output value.

Useful for:
- quickly reading important values
- automation
- troubleshooting
- verifying deployments

---

## `terraform state list`

```bash
terraform state list
```

Lists resources currently tracked in Terraform state.

This is helpful when you want to see what Terraform is managing.

---

## `terraform state show`

```bash
terraform state show aws_instance.web
```

Shows detailed state information for a specific resource.

This is useful for:
- inspection
- troubleshooting
- understanding what Terraform has recorded

---

## `terraform taint`

```bash
terraform taint aws_instance.web
```

Marks a resource for recreation on the next apply.

This tells Terraform that the resource should be destroyed and recreated.

This can be useful when a resource exists but is in a bad or unreliable state.

---

## `terraform untaint`

```bash
terraform untaint aws_instance.web
```

Removes the tainted status from a resource.

This is useful if you marked a resource for recreation by mistake.

---

## `terraform import`

```bash
terraform import aws_instance.web i-1234567890abcdef0
```

Imports an existing real infrastructure object into Terraform state.

This is useful when:
- the resource already exists
- Terraform did not originally create it
- you want Terraform to start managing it

Important note:
- import adds the resource to state
- you still need matching Terraform configuration for proper management

---

## `terraform refresh`

```bash
terraform refresh
```

Updates the Terraform state with real infrastructure information.

This was used to sync state with the actual environment.

In many workflows, refresh behavior is now handled through planning and applying, so this command is less central than before.

---

## `terraform console`

```bash
terraform console
```

Opens an interactive console for testing Terraform expressions.

This is useful for:
- testing functions
- checking expressions
- understanding variable and local behavior

---

## `terraform graph`

```bash
terraform graph
```

Generates a visual dependency graph in DOT format.

This can help you understand resource relationships and dependencies.

It is more useful in advanced troubleshooting or learning scenarios.

---

## `terraform workspace`

Terraform also includes workspace-related commands such as:

```bash
terraform workspace list
terraform workspace new dev
terraform workspace select dev
```

These are used to manage workspaces.

Workspaces are a separate topic, but it is useful to know that Terraform supports them through CLI commands.

---

## Common command workflow

A very common Terraform workflow is:

```bash
terraform fmt
terraform validate
terraform init
terraform plan
terraform apply
```

This gives you a clean and safe flow:
- format the files
- validate the configuration
- initialize the project
- preview changes
- apply changes

---

## Common destroy workflow

For cleanup:

```bash
terraform plan
terraform destroy
```

This is especially useful in learning environments and short-lived infrastructure setups.

---

## Common mistakes

- forgetting to run `terraform init`
- running `apply` without checking `plan`
- using `destroy` carelessly
- assuming `import` creates configuration automatically
- editing infrastructure manually and forgetting Terraform state
- ignoring formatting and validation

---

## Good practices

- run `terraform fmt` regularly
- run `terraform validate` before planning
- review `terraform plan` carefully
- avoid applying changes blindly
- treat `destroy` with caution
- use state-related commands carefully
- use `import` only when you understand how state and configuration should match

---

## Important notes

- `terraform init` prepares the working directory.
- `terraform fmt` formats Terraform files.
- `terraform validate` checks configuration correctness.
- `terraform plan` previews infrastructure changes.
- `terraform apply` makes real changes.
- `terraform destroy` removes managed infrastructure.
- `terraform output` reads output values.
- `terraform state` commands inspect managed resources.
- `terraform import` brings existing resources into Terraform state.

---

## Simple rule of thumb

- use `fmt` to clean code
- use `validate` to catch mistakes
- use `init` to prepare the project
- use `plan` before `apply`
- use `destroy` carefully
- use state commands only when needed

---

## Must memorize

```bash
terraform fmt
terraform fmt -recursive
terraform validate
terraform init
terraform plan
terraform apply
terraform destroy
terraform show
terraform output
terraform state list
terraform state show aws_instance.web
terraform taint aws_instance.web
terraform untaint aws_instance.web
terraform import aws_instance.web i-1234567890abcdef0
terraform console
terraform graph
```

---

## Key ideas

- Terraform commands are the main way to interact with Terraform projects.
- The most common workflow is `init`, `plan`, and `apply`.
- `fmt` and `validate` help improve safety and consistency.
- `destroy` removes infrastructure and should be used carefully.
- State commands help inspect what Terraform manages.
- `import` is used for existing resources not originally created by Terraform.
- Some commands are daily-use commands, while others are advanced troubleshooting or migration tools.