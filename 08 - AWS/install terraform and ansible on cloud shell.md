---
tags:
  - AWS
  - Terraform
  - Ansible
---
curl -O [https://releases.hashicorp.com/terraform/1.11.3/terraform_1.11.3_linux_amd64.zip](https://releases.hashicorp.com/terraform/1.11.3/terraform_1.11.3_linux_amd64.zip)

unzip terraform_1.11.3_linux_amd64.zip

sudo mv terraform /usr/local/bin/
___

sudo yum install ansible -y