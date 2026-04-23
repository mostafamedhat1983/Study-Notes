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

### Upgrade providers and modules

```bash
terraform init -upgrade
```

This tells Terraform to upgrade providers and modules to newer allowed versions based on your version constraints.

Use this when:
- you want newer allowed provider versions
- you want newer allowed module versions
- you intentionally want to refresh dependencies

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

## Save a plan to a file

```bash
terraform plan -out=tfplan
```

This saves the generated execution plan to a file instead of only printing it to the terminal.

This is useful when:
- you want to review the exact saved plan later
- you want to apply the exact reviewed plan
- you use CI/CD or approval-based workflows

A saved plan is often more controlled than generating a fresh plan again later.

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

## Apply a saved plan file

```bash
terraform apply tfplan
```

This applies the exact saved plan file instead of recalculating a new plan.

This is useful when:
- the reviewed plan must exactly match what gets applied
- you want a more controlled workflow
- you use automation or approval-based pipelines

When using a saved plan file, Terraform applies the actions already stored in that file.

---

## Apply without confirmation

```bash
terraform apply -auto-approve
```

This applies changes without asking for interactive confirmation.

This is useful in:
- automation
- CI/CD pipelines
- quick lab environments

Use it carefully because it removes the manual confirmation step.

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

## Preview destroy changes

```bash
terraform plan -destroy
```

Creates a destroy plan preview showing everything Terraform intends to remove without deleting resources yet.
This is useful when you want to review destructive changes before running `terraform destroy`.

---

## `terraform show`

```bash
terraform show
```

Displays information about the current state or a saved plan.

This can help when you want to inspect what Terraform currently knows.

---

## Show a saved plan file

```bash
terraform show tfplan
```

This displays the contents of a saved plan file in a human-readable format.

This is useful when:
- you want to inspect the exact plan saved earlier
- you want to review changes before applying that saved plan
- you want to debug or confirm what Terraform is about to do

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

### Show raw output

```bash
terraform output -raw instance_ip
```

This prints a single output value as plain text without extra formatting.

This is useful for:
- shell scripts
- piping output into other commands
- reading a clean value quickly

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

## `terraform providers`

```bash
terraform providers
```

Shows which providers the current Terraform configuration depends on.

This is useful when:
- inspecting project dependencies
- troubleshooting provider-related issues
- understanding module provider usage

---

## `terraform workspace`

Terraform also includes workspace-related commands such as:

```bash
terraform workspace list
terraform workspace new dev
terraform workspace select dev
terraform workspace show
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

## Saved plan workflow

A common controlled workflow is:

```bash
terraform plan -out=tfplan
terraform show tfplan
terraform apply tfplan
```

This is useful when:
- you want the reviewed plan and the applied plan to be exactly the same
- you need a more controlled change process
- you use CI/CD or approval-based deployment workflows

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
- reviewing one plan but later applying a newly recalculated plan instead of the saved one when exact consistency matters

---

## Good practices

- run `terraform fmt` regularly
- run `terraform validate` before planning
- review `terraform plan` carefully
- use saved plan files when you need stricter control
- avoid applying changes blindly
- treat `destroy` with caution
- use state-related commands carefully
- use `import` only when you understand how state and configuration should match

---

## Important notes

- `terraform init` prepares the working directory.
- `terraform init -upgrade` refreshes allowed provider and module versions.
- `terraform fmt` formats Terraform files.
- `terraform validate` checks configuration correctness.
- `terraform plan` previews infrastructure changes.
- `terraform plan -out=tfplan` saves a plan to a file.
- `terraform apply` makes real changes.
- `terraform apply tfplan` applies a previously saved plan file.
- `terraform apply -auto-approve` skips interactive approval.
- `terraform show` can inspect state or a saved plan.
- `terraform show tfplan` displays a saved plan file.
- `terraform destroy` removes managed infrastructure.
- `terraform output` reads output values.
- `terraform output -raw instance_ip` prints a plain-text output value.
- `terraform state` commands inspect managed resources.
- `terraform import` brings existing resources into Terraform state.
- `terraform providers` shows provider dependencies.
- `terraform workspace show` displays the current workspace.

---

## Simple rule of thumb

- use `fmt` to clean code
- use `validate` to catch mistakes
- use `init` to prepare the project
- use `plan` before `apply`
- use `plan -out` when exact reviewed execution matters
- use `destroy` carefully
- use state commands only when needed

---

## Must memorize

```bash
terraform fmt
terraform fmt -recursive
terraform validate
terraform init
terraform init -upgrade
terraform plan
terraform plan -out=tfplan
terraform apply
terraform apply tfplan
terraform apply -auto-approve
terraform destroy
terraform show
terraform show tfplan
terraform output
terraform output instance_id
terraform output -raw instance_ip
terraform state list
terraform state show aws_instance.web
terraform taint aws_instance.web
terraform untaint aws_instance.web
terraform import aws_instance.web i-1234567890abcdef0
terraform console
terraform graph
terraform providers
terraform workspace list
terraform workspace new dev
terraform workspace select dev
terraform workspace show
```

---

## Key ideas

- Terraform commands are the main way to interact with Terraform projects.
- The most common workflow is `init`, `plan`, and `apply`.
- `fmt` and `validate` help improve safety and consistency.
- Saved plan files help make review and execution more controlled.
- `destroy` removes infrastructure and should be used carefully.
- State commands help inspect what Terraform manages.
- `import` is used for existing resources not originally created by Terraform.
- Some commands are daily-use commands, while others are advanced troubleshooting or migration tools.