---
tags:
  - Terraform
---
Terraform provisioners are used to execute scripts, commands, or file-copy actions during resource creation or destruction.

They are sometimes used for:
- bootstrapping servers
- copying files
- running local scripts
- running remote commands

However, provisioners should be used carefully.

They are generally considered a **last resort** in Terraform.

---

## Main idea

Provisioners let Terraform perform extra actions after a resource is created, or before it is destroyed.

Simple idea:
- Terraform resources manage infrastructure
- provisioners run extra imperative actions around that infrastructure

Examples:
- copy a file to a VM
- run a shell command locally
- run setup commands remotely on a server

---

## Why provisioners should be used carefully

Provisioners are powerful, but they are not usually the preferred Terraform approach.

They are considered a last resort because:
- they add imperative behavior to a declarative tool
- they can be hard to debug
- they depend on network and OS access
- they are less predictable than native Terraform resources
- they can leave resources in a partially configured state if they fail

In general:
- prefer native provider resources when possible
- prefer cloud-init, user data, or image baking when appropriate
- use provisioners only when other cleaner options are not suitable

---

## Common provisioner types

Terraform commonly uses these provisioner types:
- `local-exec`
- `remote-exec`
- `file`

---

## `local-exec`

`local-exec` runs a command on the machine where Terraform itself is running.

Example:

```hcl
resource "null_resource" "example" {
  provisioner "local-exec" {
    command = "echo Terraform ran this command locally"
  }
}
```

This runs on:
- your laptop
- your workstation
- your CI runner

It does **not** run on the created resource.

---

## When `local-exec` is useful

`local-exec` can be useful for:
- calling local scripts
- sending notifications
- triggering external automation
- running helper commands after resource creation

But it should be used carefully because it depends on the local environment where Terraform runs.

---

## `remote-exec`

`remote-exec` runs commands on the remote resource after it is created.

Example:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"

  provisioner "remote-exec" {
    inline = [
      "sudo apt update",
      "sudo apt install -y nginx"
    ]
  }
}
```

This runs commands on the remote machine, not on the Terraform machine.

---

## `remote-exec` connection

`remote-exec` usually needs a `connection` block so Terraform knows how to connect to the remote machine.

Example:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"

  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = file("mykey.pem")
    host        = self.public_ip
  }

  provisioner "remote-exec" {
    inline = [
      "sudo apt update",
      "sudo apt install -y nginx"
    ]
  }
}
```

The connection block commonly includes:
- connection type
- username
- private key or password
- host address

---

## `file` provisioner

The `file` provisioner copies a file or directory from the Terraform machine to the remote resource.

Example:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"

  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = file("mykey.pem")
    host        = self.public_ip
  }

  provisioner "file" {
    source      = "app.conf"
    destination = "/tmp/app.conf"
  }
}
```

This is useful when you want to upload:
- config files
- scripts
- templates
- bootstrap assets

---

## Creation-time and destroy-time provisioners

Provisioners usually run during resource creation.

They can also be configured for destroy time.

Example:

```hcl
provisioner "local-exec" {
  when    = destroy
  command = "echo Resource is being destroyed"
}
```

This makes the provisioner run during destruction instead of creation.

---

## Failure behavior

If a creation-time provisioner fails, Terraform can mark the resource as tainted.

This means Terraform may recreate the resource on the next apply.

That happens because Terraform cannot safely assume the resource is fully configured after a failed provisioner.

This is one reason provisioners can be risky.

---

## Multiple provisioners

A resource can have more than one provisioner.

Terraform runs them in the order they are defined.

Example:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"

  provisioner "local-exec" {
    command = "echo Starting"
  }

  provisioner "local-exec" {
    command = "echo Finished"
  }
}
```

Provisioner order can matter when actions depend on each other.

---

## Example with `remote-exec`

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"

  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = file("mykey.pem")
    host        = self.public_ip
  }

  provisioner "remote-exec" {
    inline = [
      "echo Hello from Terraform",
      "sudo apt update",
      "sudo apt install -y nginx"
    ]
  }
}
```

This is a common beginner example of remote bootstrapping.

---

## Example with `file` and `remote-exec`

A common pattern is:
1. copy a script with `file`
2. execute it with `remote-exec`

Example:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"

  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = file("mykey.pem")
    host        = self.public_ip
  }

  provisioner "file" {
    source      = "setup.sh"
    destination = "/tmp/setup.sh"
  }

  provisioner "remote-exec" {
    inline = [
      "chmod +x /tmp/setup.sh",
      "sudo /tmp/setup.sh"
    ]
  }
}
```

This works, but should still be used carefully.

---

## Better alternatives to provisioners

In many cases, better alternatives include:
- cloud-init
- user data
- custom machine images
- Packer
- configuration management tools like Ansible

These are often better because they are:
- more predictable
- easier to test
- better suited for OS and application configuration
- less dependent on Terraform runtime connectivity

---

## When provisioners may still be acceptable

Provisioners may still be acceptable for:
- lightweight bootstrapping
- one-time setup tasks
- simple file copy actions
- integration with legacy workflows
- special cases where native Terraform resources are not enough

The key is to keep them small and intentional.

---

## Common mistakes

- using provisioners for full configuration management
- relying on SSH-based setup when a cleaner bootstrap method exists
- putting long complex scripts directly into `inline`
- assuming provisioners are always reliable
- forgetting that `local-exec` runs locally, not on the remote resource
- not understanding failure and taint behavior

---

## Good practices

- treat provisioners as a last resort
- keep provisioner logic minimal
- prefer cloud-init or user data for instance bootstrapping
- use `file` only when copying files is truly needed
- avoid complex long-running setup steps in Terraform
- understand that connectivity problems can break provisioners

---

## Important notes

- Provisioners let Terraform run extra imperative actions.
- Common provisioners are `local-exec`, `remote-exec`, and `file`.
- `local-exec` runs on the machine where Terraform is executed.
- `remote-exec` runs on the created remote resource.
- `file` copies files from the Terraform machine to the remote resource.
- Provisioners are generally considered a last resort.
- Failed creation-time provisioners can taint resources.

---

## Simple rule of thumb

- use native Terraform resources first
- use cloud-init or user data before using provisioners
- use provisioners only when simpler and more declarative options are not enough

---

## Must memorize

```hcl
provisioner "local-exec" {
  command = "echo Hello"
}
```

```hcl
provisioner "remote-exec" {
  inline = [
    "sudo apt update",
    "sudo apt install -y nginx"
  ]
}
```

```hcl
provisioner "file" {
  source      = "setup.sh"
  destination = "/tmp/setup.sh"
}
```

```hcl
connection {
  type        = "ssh"
  user        = "ubuntu"
  private_key = file("mykey.pem")
  host        = self.public_ip
}
```

```hcl
when = destroy
```

---

## Key ideas

- Provisioners run commands or copy files during Terraform resource operations.
- They are useful but should be used carefully.
- `local-exec` runs locally, `remote-exec` runs remotely, and `file` copies files.
- Provisioners are usually less ideal than native Terraform approaches.
- Failed provisioners can leave resources in a bad state.
- In most cases, provisioners should be a last resort.