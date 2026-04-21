---
tags:
  - Terraform
---
 Terraform outputs are used to display useful values after Terraform creates or updates infrastructure.

They help you expose important information from your Terraform configuration, such as:
- instance public IP
- VPC ID
- bucket name
- load balancer DNS name
- database endpoint

Outputs make it easier to:
- read important values after `apply`
- pass values between modules
- reference infrastructure details clearly

---

## Main idea

An output is a named value that Terraform prints after apply.

It lets you expose useful information from your configuration.

Simple idea:
- variable = input to Terraform
- output = value produced by Terraform

---

## Why outputs are useful

Outputs are useful because they help you:
- quickly see important infrastructure values
- avoid searching manually in the cloud console
- pass values from one module to another
- make Terraform configurations easier to use
- expose IDs, names, endpoints, and addresses

For example, after creating an EC2 instance, you may want Terraform to show:
- the instance ID
- the public IP
- the private IP

---

## Basic output syntax

An output is defined with an `output` block.

Example:

```hcl
output "instance_id" {
  value = aws_instance.web.id
}
```

This tells Terraform to display the ID of the `aws_instance.web` resource.

---

## Simple example

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
}

output "instance_id" {
  value = aws_instance.web.id
}
```

After `terraform apply`, Terraform can show the value of `instance_id`.

---

## Output block parts

A basic output block usually includes:
- output name
- value expression

Example:

```hcl
output "bucket_name" {
  value = aws_s3_bucket.demo.bucket
}
```

Here:
- `bucket_name` = output name
- `aws_s3_bucket.demo.bucket` = value expression

---

## Referencing resource attributes in outputs

Outputs often reference resource attributes.

Example:

```hcl
output "public_ip" {
  value = aws_instance.web.public_ip
}
```

This follows the same Terraform reference pattern:

```hcl
resource_type.resource_name.attribute
```

Outputs are often built from:
- resource attributes
- variables
- locals
- expressions

---

## Output from variables

Outputs can also display variable values.

Example:

```hcl
variable "region" {
  type    = string
  default = "us-east-1"
}

output "selected_region" {
  value = var.region
}
```

This can be useful for debugging or confirming configuration values.

---

## Output from expressions

Outputs are not limited to simple direct references.

They can also use expressions.

Example:

```hcl
output "instance_summary" {
  value = "Instance ID: ${aws_instance.web.id}"
}
```

This lets you format or combine values.

---

## Sensitive outputs

Some outputs may contain sensitive information.

Terraform allows outputs to be marked as sensitive.

Example:

```hcl
output "db_password" {
  value     = var.db_password
  sensitive = true
}
```

This helps reduce accidental exposure in normal CLI output.

Use this carefully for things like:
- passwords
- tokens
- secrets
- private connection details

---

## Description in outputs

You can also add a description to make the output easier to understand.

Example:

```hcl
output "instance_public_ip" {
  description = "Public IP address of the web server"
  value       = aws_instance.web.public_ip
}
```

This is useful in larger projects.

---

## Example with multiple outputs

```hcl
output "instance_id" {
  value = aws_instance.web.id
}

output "instance_public_ip" {
  value = aws_instance.web.public_ip
}

output "instance_private_ip" {
  value = aws_instance.web.private_ip
}
```

This is common when you want to expose several useful details after deployment.

---

## Output values after apply

After running:

```bash
terraform apply
```

Terraform usually shows output values at the end if outputs are defined.

This makes it easy to immediately see important details without manually checking the provider console.

---

## Viewing outputs later

You can also inspect outputs later without re-reading the whole state manually.

### Show all outputs

```bash
terraform output
```

This lists all defined output values.

---

### Show one output

```bash
terraform output instance_id
```

This shows the value of a specific output.

---

## JSON output

Terraform can also return outputs in JSON format.

Example:

```bash
terraform output -json
```

This is useful for:
- automation
- scripting
- CI/CD pipelines
- integration with other tools

---

## Output values in modules

Outputs are especially important when working with modules.

A child module can define outputs, and the parent module can use them.

Example idea:
- child module creates a VPC
- child module outputs the VPC ID
- parent module uses that VPC ID elsewhere

This makes modules reusable and composable.

---

## Common output examples

Common things to expose as outputs:
- instance IDs
- public IPs
- private IPs
- VPC IDs
- subnet IDs
- bucket names
- database endpoints
- load balancer DNS names

Outputs should usually expose values that are useful to:
- the operator
- another module
- automation scripts
- deployment workflows

---

## Common mistakes

- creating outputs for unimportant values
- exposing secrets without using `sensitive = true`
- forgetting that outputs often depend on resource attributes
- using unclear output names
- adding too many outputs that provide little value

---

## Good practices

- use clear output names
- expose only useful values
- use descriptions when helpful
- mark secret values as sensitive
- keep outputs meaningful and practical
- use outputs to share values from child modules

---

## Important notes

- Outputs are values Terraform displays after apply.
- Outputs help expose useful infrastructure information.
- Outputs can reference resources, variables, locals, and expressions.
- `terraform output` shows defined outputs.
- `terraform output -json` is useful for automation.
- Sensitive outputs should be marked with `sensitive = true`.

---

## Simple rule of thumb

- use variables for inputs
- use outputs for results
- expose only what is useful
- protect sensitive values

---

## Must memorize

```hcl
output "instance_id" {
  value = aws_instance.web.id
}
```

```hcl
output "db_password" {
  value     = var.db_password
  sensitive = true
}
```

```bash
terraform output
terraform output instance_id
terraform output -json
```

---

## Key ideas

- Outputs are named values Terraform exposes after apply.
- Outputs help display useful infrastructure details.
- Outputs can reference resource attributes and expressions.
- Sensitive outputs should be protected.
- Outputs are especially useful when working with modules.
- `terraform output` is the main command for reading output values.