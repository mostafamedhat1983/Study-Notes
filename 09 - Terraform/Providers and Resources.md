---
tags:
  - Terraform
---
Providers and resources are two of the most important Terraform concepts.

A provider allows Terraform to talk to a platform such as AWS, Azure, Google Cloud, Kubernetes, or GitHub.

A resource is the actual infrastructure object Terraform creates or manages, such as:
- a virtual machine
- an S3 bucket
- a VPC
- a DNS record

---

## Main idea

Terraform does not manage infrastructure by itself.

It needs:
- a **provider** to connect to a platform
- a **resource** block to describe what infrastructure object to manage

Simple idea:
- provider = how Terraform talks to the platform
- resource = what Terraform creates or manages on that platform

---

## What is a provider

A provider is a plugin that allows Terraform to interact with an external platform or API.

Examples:
- AWS provider
- AzureRM provider
- Google provider
- Kubernetes provider
- GitHub provider
- local provider
- random provider

Without a provider, Terraform cannot manage infrastructure on that system.

---

## Why providers matter

Providers are important because they define:
- what resource types are available
- how Terraform communicates with the target platform
- what arguments and features are supported

For example:
- the AWS provider gives access to resources like `aws_instance` and `aws_s3_bucket`
- the Azure provider gives access to resources like `azurerm_resource_group`
- the Kubernetes provider gives access to resources like `kubernetes_namespace`

---

## What is a resource

A resource is an infrastructure object managed by Terraform.

Examples:
- EC2 instance
- S3 bucket
- security group
- subnet
- DNS record
- Kubernetes deployment

A resource block usually looks like this:

```hcl
resource "aws_s3_bucket" "demo" {
  bucket = "my-demo-bucket-12345"
}
```

Here:
- `resource` = block type
- `aws_s3_bucket` = resource type
- `demo` = local resource name inside Terraform

---

## Resource type and resource name

A resource block has two labels:

```hcl
resource "aws_instance" "web" {
}
```

These mean:
- `aws_instance` = resource type
- `web` = local Terraform name

The resource type comes from the provider.

The local name is used inside Terraform to reference that resource.

Example reference:

```hcl
aws_instance.web.id
```

This means:
- resource type = `aws_instance`
- resource name = `web`
- attribute = `id`

---

## Provider requirements

Terraform recommends declaring provider requirements inside the top-level `terraform` block.

Example:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

This tells Terraform:
- which provider is needed
- where to find it
- which version to use

This is important for:
- reproducibility
- team consistency
- avoiding unexpected version changes

---

## Provider configuration

After declaring the provider requirement, you usually configure the provider with a `provider` block.

Example:

```hcl
provider "aws" {
  region = "us-east-1"
}
```

This tells Terraform how to use that provider.

For AWS, this often includes things like:
- region
- credentials
- profile
- assume role settings

The exact arguments depend on the provider.

---
## Provider alias

Terraform lets you define multiple configurations for the same provider by using the `alias` argument.

This is useful when you want to work with:
- multiple regions
- multiple accounts
- multiple subscriptions
- different provider settings in one configuration

Example:

```hcl
provider "aws" {
  region = "eu-central-1"
}

provider "aws" {
  alias  = "use1"
  region = "us-east-1"
}
```

In this example:
- the first provider block is the default AWS provider
- the second provider block is an aliased provider named `use1`

You can tell a resource to use the aliased provider like this:

```hcl
resource "aws_s3_bucket" "example" {
  provider = aws.use1
  bucket   = "my-example-bucket"
}
```

If a resource does not specify a provider explicitly, Terraform uses the default provider configuration that matches the resource type.

---

## Provider alias with modules

Aliased providers can also be passed into modules.

Example:

```hcl
module "app" {
  source = "./modules/app"

  providers = {
    aws = aws.use1
  }
}
```

This tells the module to use the aliased provider instead of the default one.

In child modules, provider aliases must be declared with `configuration_aliases` in the `required_providers` block when needed.
## Full example

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "demo" {
  bucket = "my-demo-bucket-12345"
}
```

### What this does

- declares that the configuration needs the AWS provider
- tells Terraform which provider source and version to use
- configures AWS to use the `us-east-1` region
- creates an S3 bucket resource

---

## How Terraform uses providers

When you run:

```bash
terraform init
```

Terraform reads the configuration and downloads the required provider plugins for the project.

That is one reason `terraform init` is needed before `plan` or `apply`.

---

## One provider can manage many resources

A single provider can manage many resource types.

For example, the AWS provider can manage:
- EC2 instances
- S3 buckets
- IAM roles
- VPCs
- subnets
- security groups
- Route 53 records

So one provider often gives access to a very large number of resources.

---

## Different providers, different resource types

Each provider has its own resource types.

Examples:
- AWS: `aws_instance`, `aws_s3_bucket`
- AzureRM: `azurerm_resource_group`
- Kubernetes: `kubernetes_namespace`
- GitHub: `github_repository`

This is why the provider matters so much.

Resource names are not universal across providers.

---

## Local name inside Terraform

The second label in a resource block is the local Terraform name.

Example:

```hcl
resource "aws_instance" "app_server" {
}
```

Here:
- `aws_instance` is the type
- `app_server` is the local name

This name does **not always mean** the real cloud resource will be named the same way.

It is mainly used inside Terraform configuration for references.

---

## Referencing resources

Terraform lets you reference one resource from another.

Example:

```hcl
aws_s3_bucket.demo.id
```

This follows the pattern:

```hcl
resource_type.resource_name.attribute
```

This is very important in Terraform because resources often depend on values from other resources.

---

## Simple AWS example

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
}
```

### What this does

- declares the AWS provider requirement
- configures the AWS provider
- defines one EC2 instance resource named `web`

---

## Common mistakes

- forgetting to run `terraform init`
- using a resource type without the correct provider
- confusing the resource type with the local resource name
- not pinning provider versions
- assuming all providers use the same arguments

---

## Important notes

- Every Terraform resource type comes from a provider.
- Terraform needs provider plugins to manage infrastructure.
- `required_providers` belongs inside the top-level `terraform` block.
- The `provider` block configures how Terraform connects to the platform.
- A resource block describes a real infrastructure object.
- Resource references often use the form `resource_type.resource_name.attribute`.

---

## Simple rule of thumb

- use `required_providers` to declare what provider Terraform needs
- use `provider` blocks to configure access to that provider
- use `resource` blocks to define the infrastructure objects you want Terraform to manage

---

## Must memorize

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

```hcl
provider "aws" {
  region = "us-east-1"
}
```

```hcl
resource "aws_s3_bucket" "demo" {
  bucket = "my-demo-bucket-12345"
}
```

```hcl
aws_s3_bucket.demo.id
```

---

## Key ideas

- A provider is the plugin Terraform uses to interact with a platform.
- A resource is the infrastructure object Terraform manages.
- Resource types come from providers.
- `required_providers` declares which providers Terraform needs.
- `provider` blocks configure those providers.
- Resource references commonly use `resource_type.resource_name.attribute`.
- Terraform cannot manage infrastructure without providers.