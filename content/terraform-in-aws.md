+++
title = "Terraform in AWS"
description = "Tools and best practices. that makes your terraform life easier on AWS."
date = 2022-04-17

[taxonomies]
tags = ["AWS", "terraform"]

[extra]
author = "Bill Hegazy"
+++
<link rel="canonical" href="https://billhegazy.medium.com/terraform-in-aws-9793e3c01173"/>
  
# Tools and best practices, that makes your terraform life easier on AWS.
![AWS SA Pro note](/images/terraform-aws.jpeg)

<i> Originally published at <a href="https://billhegazy.medium.com/terraform-in-aws-9793e3c01173">https://medium.com</a> on 19 June 2021</i>

## 1) aws-vault
Although it's not exactly specific for Terraform aws-vault is a must-use tool for Terraform in AWS, especially when you have multiple AWS accounts.

[aws-vault](https://github.com/99designs/aws-vault) stores IAM credentials in your Os's secure Keystore and generates temporary credentials to be used in shell.

Using aws-vault with Terraform to easily switch between AWS accounts and avoid hard-coding AWS profile in Terraform backend state code.

### Install aws-vault
```bash
brew install --cask aws-vault
```


### Usage Example
```bash
# Run simple aws command
aws-vault exec aws_example_account -- aws s3 ls
# Login to aws console using temporary credentials
aws-vault login aws_example_account
# Use with terraform
aws-vault exec aws_example_account -- terraform apply
```

## 2) tfenv
tfenv is Terraform version manager similar to rbenv.

### Install tfenv
```bash
brew install tfenv
```

### Usage Example
```bash
# List tf remote versions
tfenv list-remote
# Install tf version
tfenv install 0.11.15
# Use tf version
tfenv use 0.11.15
```

### tfenv automatic version switching
Add `.terraform-version` file to automatically switch between different Terraform versions and control versions between accounts.

Example:
```
├── .
├── aws_prodution_account
    ├── resource_1
    │   ├── main.tf
    |   ├── variables.tf
    |   └── ...
    ├── resource_2
    │   ├── main.tf
    |   ├── variables.tf
    |   └── ...
    ├── .terraform-version
├── aws_staging_account
    ├── resource_1
    │   ├── main.tf
    |   ├── variables.tf
    |   └── ...
    ├── resource_2
    │   ├── main.tf
    |   ├── variables.tf
    |   └── ...
    ├── .terraform-version
├── README.md
└── ...
```

## 3) pre-commit
Using [pre-commit framework](http://pre-commit.com/) with terraform repository, will help your code to be kept clean, formated, updated document and checked for tf security issues (optional with [tfsec](https://github.com/tfsec/tfsec)) before committing and pushing the code to git source.

### Install precommit and related tools

```bash
brew install pre-commit gawk terraform-docs tflint coreutils checkov terrascan
```

### Install the pre-commit hook globally
```bash
DIR=~/.git-template
git config --global init.templateDir ${DIR}
pre-commit init-templatedir -t pre-commit ${DIR}
```
### Initialize git repo with terraform hooks
```bash
cd your_terraform_git_repo
git init # if new repo
cat <<EOF > .pre-commit-config.yaml
repos:
- repo: git://github.com/antonbabenko/pre-commit-terraform
  rev: <VERSION> # Get the latest from: <https://github.com/antonbabenko/pre-commit-terraform/releases>
  hooks:
    - id: terraform_fmt
    - id: terraform_docs
EOF
pre-commit install
# Test pre commit
pre-commit run --all-
```

> Now, whenever you run git commit on terraform repo, pre-commit will run the hooks
### Auto generate Terraform docs with pre-commit
> Using [terraform-docs](https://github.com/terraform-docs/terraform-docs) with terraform modules

```bash
cd terraform_example_module
cat <<EOF > README.md
<!-- BEGINNING OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
lines here will be replaced by terraform_docs when pre-commit runs
<!-- END OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
EOF
pre-commit run --all-files
```

## 4) tfsec
Want static analysis for your terraform code to help spot potential security issues? then all you need is [tfsec](https://github.com/tfsec/tfsec).

### Install tfsec
```bash
brew install tfsec
```

### Usage Example
```bash
cd terraform_folder
tfsec .
```
### Add tfsec to your pre-commit config
Add `terraform_tfsec` hook to `.pre-commit-config.yaml`
**Example**:
```yaml
repos:
- repo: git://github.com/antonbabenko/pre-commit-terraform
  hooks:
    - ...
    - id: terraform_tfsec
```
### Ignoring some tfsec rules
You may wish to ignore some warnings from tfsec. you can simply add a comment containing `tfsec:ignore:<RULE>` to the offending line in your templates.

**For example, to ignore an open security group rule:**

```hcl
resource "aws_security_group_rule" "my-rule" {
    type = "ingress"
    #tfsec:ignore:AWS006
    cidr_blocks = ["0.0.0.0/0"]
}
```

## Other best practices:
- Use [official Terraform public module](https://registry.terraform.io/providers/hashicorp/aws/latest) for AWS, official public module are well written and tested ( don't re-invent the wheel).
- Limit access to Terraform state S3bucket access, encrypt it and enable versioning.
- Avoid storing secrets when creating a resource as Terraform state will store secrets plain-text, at least create a temporary password and change it after the resource is created.

```hcl
# Simple Example
resource "random_password" "db-password" {
  length  = 16
  special = false
}
resource "aws_db_instance" "default" {
  engine           = "mysql"
  engine_version = "5.7"
  instance_class       = "db.t3.micro"
  name                 = "mydb"
  username             = "foo"
  password             = random_password.db-password.result
  ...
}
```