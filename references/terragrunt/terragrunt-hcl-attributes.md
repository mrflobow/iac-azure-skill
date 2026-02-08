# Terragrunt HCL Attributes Reference

Terragrunt HCL attributes define values for Terragrunt configuration, such as inputs to pass to OpenTofu/Terraform or operational settings.

## Table of Contents

- [inputs](#inputs)
- [download_dir](#download_dir)
- [prevent_destroy](#prevent_destroy)
- [iam_role](#iam_role)
- [iam_assume_role_duration](#iam_assume_role_duration)
- [iam_assume_role_session_name](#iam_assume_role_session_name)
- [iam_web_identity_token](#iam_web_identity_token)
- [terraform_binary](#terraform_binary)
- [terraform_version_constraint](#terraform_version_constraint)
- [terragrunt_version_constraint](#terragrunt_version_constraint)

---

## inputs

A map of input variables to pass to OpenTofu/Terraform via `TF_VAR_` environment variables.

### Basic Usage

```hcl
inputs = {
  string      = "string"
  number      = 42
  bool        = true
  list_string = ["a", "b", "c"]
  list_number = [1, 2, 3]

  map_string = {
    foo = "bar"
  }

  object = {
    str  = "string"
    num  = 42
    list = [1, 2, 3]
    map = {
      foo = "bar"
    }
  }

  from_env = get_env("FROM_ENV", "default")
}
```

### Equivalent TF_VAR Usage

Setting this:
```hcl
inputs = {
  instance_type  = "t2.micro"
  instance_count = 10
  tags = {
    Name = "example-app"
  }
}
```

Is equivalent to running:
```bash
TF_VAR_instance_type="t2.micro" \
TF_VAR_instance_count=10 \
TF_VAR_tags='{"Name":"example-app"}' \
tofu apply
```

### Variable Precedence (Lowest to Highest)

1. `inputs` in `terragrunt.hcl`
2. `TF_VAR_` environment variables
3. `terraform.tfvars` files
4. `terraform.tfvars.json` files
5. `*.auto.tfvars` or `*.auto.tfvars.json` files (lexical order)
6. `-var` and `-var-file` CLI options

### Type Constraints

Since values are passed via environment variables as JSON, type information is lost. Ensure proper type constraints in your OpenTofu/Terraform variables:

```hcl
# variables.tf
variable "tags" {
  type = map(string)
}

variable "subnet_ids" {
  type = list(string)
}
```

---

## download_dir

Override the default directory where Terragrunt downloads remote configurations (default: `.terragrunt-cache`).

```hcl
download_dir = "${get_parent_terragrunt_dir()}/.cache"
```

### Precedence (Highest to Lowest)

1. `--download-dir` CLI flag
2. `TG_DOWNLOAD_DIR` environment variable
3. `download_dir` in module's `terragrunt.hcl`
4. `download_dir` in included `terragrunt.hcl`

---

## prevent_destroy

Protect the module from `destroy` and `run --all destroy` commands.

```hcl
terraform {
  source = "git::git@github.com:org/modules.git//database?ref=v1.0.0"
}

prevent_destroy = true
```

When `prevent_destroy = true`, Terragrunt will exit with an error if a destroy is attempted on this unit.

---

## iam_role

Specify an IAM role for Terragrunt to assume before invoking OpenTofu/Terraform.

```hcl
iam_role = "arn:aws:iam::123456789012:role/TerraformRole"
```

### With Local Variables

```hcl
locals {
  account_id = "123456789012"
}

iam_role = "arn:aws:iam::${local.account_id}:role/TerraformRole"
```

### Precedence (Highest to Lowest)

1. `--iam-assume-role` CLI flag
2. `TG_IAM_ASSUME_ROLE` environment variable
3. `iam_role` in module's `terragrunt.hcl`
4. `iam_role` in included `terragrunt.hcl`

---

## iam_assume_role_duration

STS session duration in seconds for the assumed IAM role.

```hcl
iam_role                 = "arn:aws:iam::123456789012:role/TerraformRole"
iam_assume_role_duration = 14400  # 4 hours
```

### Precedence (Highest to Lowest)

1. `--iam-assume-role-duration` CLI flag
2. `TG_IAM_ASSUME_ROLE_DURATION` environment variable
3. `iam_assume_role_duration` in module's `terragrunt.hcl`
4. `iam_assume_role_duration` in included `terragrunt.hcl`

---

## iam_assume_role_session_name

STS session name for the assumed IAM role.

```hcl
iam_role                      = "arn:aws:iam::123456789012:role/TerraformRole"
iam_assume_role_session_name  = "terragrunt-deploy"
```

### Precedence (Highest to Lowest)

1. `--iam-assume-role-session-name` CLI flag
2. `TG_IAM_ASSUME_ROLE_SESSION_NAME` environment variable
3. `iam_assume_role_session_name` in module's `terragrunt.hcl`
4. `iam_assume_role_session_name` in included `terragrunt.hcl`

---

## iam_web_identity_token

Use AssumeRoleWithWebIdentity instead of AssumeRole. Enables credential-less CI/CD pipelines.

### Token from Environment Variable

```hcl
iam_role               = "arn:aws:iam::123456789012:role/TerraformRole"
iam_web_identity_token = get_env("OIDC_TOKEN")
```

### Token from File

```hcl
iam_role               = "arn:aws:iam::123456789012:role/TerraformRole"
iam_web_identity_token = "/var/run/secrets/token"
```

### Precedence (Highest to Lowest)

1. `--iam-assume-role-web-identity-token` CLI flag
2. `TG_IAM_ASSUME_ROLE_WEB_IDENTITY_TOKEN` environment variable
3. `iam_web_identity_token` in module's `terragrunt.hcl`
4. `iam_web_identity_token` in included `terragrunt.hcl`

### Provider Configuration

Configure your CI/CD OIDC provider in AWS:
- **GitHub**: [Configuring OpenID Connect in AWS](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
- **GitLab**: [Configure OpenID Connect in AWS](https://docs.gitlab.com/ee/ci/cloud_services/aws/)
- **CircleCI**: [Using OpenID Connect tokens](https://circleci.com/docs/openid-connect-tokens/)

---

## terraform_binary

Override the default binary Terragrunt invokes (default: `tofu`).

```hcl
terraform_binary = "terraform"
```

### Precedence (Highest to Lowest)

1. `--tf-path` CLI flag
2. `TG_TF_PATH` environment variable
3. `terraform_binary` in module's `terragrunt.hcl`
4. `terraform_binary` in included `terragrunt.hcl`

---

## terraform_version_constraint

Override the minimum supported version of OpenTofu/Terraform.

```hcl
terraform_version_constraint = ">= 0.11"
```

```hcl
terraform_version_constraint = ">= 1.0, < 2.0"
```

---

## terragrunt_version_constraint

Restrict which versions of Terragrunt CLI can be used with this configuration.

```hcl
terragrunt_version_constraint = ">= 0.23"
```

```hcl
terragrunt_version_constraint = ">= 0.50, < 1.0"
```

If the running Terragrunt version doesn't match, Terragrunt exits with an error.
