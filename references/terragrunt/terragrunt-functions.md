# Terragrunt Functions Reference

Terragrunt provides built-in functions for use in `terragrunt.hcl` files, plus access to all OpenTofu/Terraform built-in functions.

## Table of Contents

- [OpenTofu/Terraform Functions](#opentofuterraform-functions)
- [Path Functions](#path-functions)
  - [find_in_parent_folders](#find_in_parent_folders)
  - [path_relative_to_include](#path_relative_to_include)
  - [path_relative_from_include](#path_relative_from_include)
  - [get_terragrunt_dir](#get_terragrunt_dir)
  - [get_working_dir](#get_working_dir)
  - [get_parent_terragrunt_dir](#get_parent_terragrunt_dir)
  - [get_original_terragrunt_dir](#get_original_terragrunt_dir)
  - [get_repo_root](#get_repo_root)
  - [get_path_from_repo_root](#get_path_from_repo_root)
  - [get_path_to_repo_root](#get_path_to_repo_root)
- [Environment Functions](#environment-functions)
  - [get_env](#get_env)
  - [get_platform](#get_platform)
- [AWS Functions](#aws-functions)
  - [get_aws_account_id](#get_aws_account_id)
  - [get_aws_account_alias](#get_aws_account_alias)
  - [get_aws_caller_identity_arn](#get_aws_caller_identity_arn)
  - [get_aws_caller_identity_user_id](#get_aws_caller_identity_user_id)
- [Terraform Command Functions](#terraform-command-functions)
  - [get_terraform_command](#get_terraform_command)
  - [get_terraform_cli_args](#get_terraform_cli_args)
  - [get_terraform_commands_that_need_vars](#get_terraform_commands_that_need_vars)
  - [get_terraform_commands_that_need_input](#get_terraform_commands_that_need_input)
  - [get_terraform_commands_that_need_locking](#get_terraform_commands_that_need_locking)
  - [get_terraform_commands_that_need_parallelism](#get_terraform_commands_that_need_parallelism)
- [Configuration Functions](#configuration-functions)
  - [read_terragrunt_config](#read_terragrunt_config)
  - [read_tfvars_file](#read_tfvars_file)
  - [sops_decrypt_file](#sops_decrypt_file)
  - [get_terragrunt_source_cli_flag](#get_terragrunt_source_cli_flag)
- [Utility Functions](#utility-functions)
  - [run_cmd](#run_cmd)
  - [get_default_retryable_errors](#get_default_retryable_errors)
  - [mark_as_read](#mark_as_read)
  - [constraint_check](#constraint_check)

---

## OpenTofu/Terraform Functions

All [OpenTofu/Terraform built-in functions](https://opentofu.org/docs/language/functions/) are available:

```hcl
terraform {
  source = "../modules/${basename(get_terragrunt_dir())}"
}

remote_state {
  backend = "s3"
  config = {
    bucket = trimspace("   my-bucket     ")
    region = join("-", ["us", "east", "1"])
    key    = format("%s/terraform.tfstate", path_relative_to_include())
  }
}
```

File functions (`file`, `fileexists`, `filebase64`, etc.) are relative to the `terragrunt.hcl` file:

```hcl
locals {
  content = file("assets/config.txt")
}
```

---

## Path Functions

### find_in_parent_folders

Search up the directory tree for a file or folder.

```hcl
# Find file in parent folders
include "root" {
  path = find_in_parent_folders("root.hcl")
}

# With fallback value
include "root" {
  path = find_in_parent_folders("root.hcl", "fallback.hcl")
}

# Find a directory
locals {
  modules_dir = find_in_parent_folders("modules")
}
```

Searches from the child `terragrunt.hcl` when called from an included config.

---

### path_relative_to_include

Returns the relative path from the current config to the included config.

```hcl
# root.hcl
remote_state {
  backend = "s3"
  config = {
    bucket = "my-bucket"
    key    = "${path_relative_to_include()}/terraform.tfstate"
    # Results in: "prod/mysql/terraform.tfstate" for prod/mysql/terragrunt.hcl
  }
}
```

With multiple includes, specify which one:
```hcl
terraform {
  source = "../modules/${path_relative_to_include("root")}"
}
```

---

### path_relative_from_include

Returns the relative path from the included config to the current config (counterpart of `path_relative_to_include`).

```hcl
# root.hcl
terraform {
  source = "${path_relative_from_include()}/../sources//${path_relative_to_include()}"
  # For mysql module: "../../sources//mysql"
}

terraform {
  extra_arguments "common_var" {
    commands = ["apply", "plan"]
    arguments = [
      "-var-file=${get_terragrunt_dir()}/${path_relative_from_include()}/common.tfvars"
    ]
  }
}
```

---

### get_terragrunt_dir

Returns the directory containing the current `terragrunt.hcl` file.

```hcl
terraform {
  source = "git::git@github.com:org/modules.git//app?ref=v1.0.0"

  extra_arguments "custom_vars" {
    commands = ["apply", "plan"]
    arguments = [
      "-var-file=${get_terragrunt_dir()}/../common.tfvars"
    ]
  }
}
```

Useful for relative paths that work correctly when terraform runs in `.terragrunt-cache`.

---

### get_working_dir

Returns the absolute path where Terragrunt runs OpenTofu/Terraform commands (the `.terragrunt-cache` directory).

```hcl
locals {
  working_dir = get_working_dir()
}
```

---

### get_parent_terragrunt_dir

Returns the directory of the parent (included) configuration file.

```hcl
# root.hcl
terraform {
  extra_arguments "common_vars" {
    commands = ["apply", "plan"]
    arguments = [
      "-var-file=${get_parent_terragrunt_dir()}/common.tfvars"
    ]
  }
}
```

With multiple includes:
```hcl
terraform {
  source = "${get_parent_terragrunt_dir("root")}/modules/vpc"
}
```

---

### get_original_terragrunt_dir

Returns the directory of the original `terragrunt.hcl` that started the run. Useful when configs are read from other configs via `read_terragrunt_config`.

```hcl
# If /terraform/terragrunt.hcl calls read_terragrunt_config("/foo/bar.hcl"),
# get_original_terragrunt_dir() returns "/terraform" when called within bar.hcl
```

---

### get_repo_root

Returns the absolute path to the Git repository root.

```hcl
inputs = {
  config_path = "${get_repo_root()}/config/settings.json"
}
```

---

### get_path_from_repo_root

Returns the path from the Git repository root to the current directory.

```hcl
remote_state {
  backend = "s3"
  config = {
    key = "${get_path_from_repo_root()}/terraform.tfstate"
  }
}
```

---

### get_path_to_repo_root

Returns the relative path to the Git repository root.

```hcl
terraform {
  source = "${get_path_to_repo_root()}//modules/vpc"
}
```

---

## Environment Functions

### get_env

Get environment variable value with optional default.

```hcl
# Required - errors if not set
locals {
  api_key = get_env("API_KEY")
}

# With default value
locals {
  region = get_env("AWS_REGION", "us-east-1")
}

inputs = {
  bucket = get_env("BUCKET", "default-bucket")
}
```

Share variables between Terragrunt and Terraform using `TF_VAR_` prefix:
```bash
export TF_VAR_region=us-west-2
```
```hcl
locals {
  region = get_env("TF_VAR_region", "us-east-1")
}
```

---

### get_platform

Returns the current operating system.

```hcl
inputs = {
  platform = get_platform()
}

locals {
  script = get_platform() == "windows" ? "setup.ps1" : "setup.sh"
}
```

Possible values: `darwin`, `linux`, `windows`, `freebsd`

---

## AWS Functions

### get_aws_account_id

Returns the current AWS account ID.

```hcl
remote_state {
  backend = "s3"
  config = {
    bucket = "terraform-state-${get_aws_account_id()}"
  }
}
```

---

### get_aws_account_alias

Returns the AWS account alias (or empty string if none).

```hcl
inputs = {
  account_alias = get_aws_account_alias()
}
```

---

### get_aws_caller_identity_arn

Returns the ARN of the current AWS identity.

```hcl
inputs = {
  caller_arn = get_aws_caller_identity_arn()
}
```

---

### get_aws_caller_identity_user_id

Returns the UserId of the current AWS identity.

```hcl
inputs = {
  caller_user_id = get_aws_caller_identity_user_id()
}

# Account-specific tfvars
terraform {
  extra_arguments "account_vars" {
    commands  = get_terraform_commands_that_need_vars()
    arguments = ["-var-file=${get_aws_account_id()}.tfvars"]
  }
}
```

**Note**: AWS identity values can change during config parsing, especially after `iam_role` evaluation.

---

## Terraform Command Functions

### get_terraform_command

Returns the current terraform command being executed.

```hcl
inputs = {
  current_command = get_terraform_command()
}
```

---

### get_terraform_cli_args

Returns the CLI arguments for the current terraform command.

```hcl
inputs = {
  cli_args = get_terraform_cli_args()
}
```

---

### get_terraform_commands_that_need_vars

Returns commands that accept `-var` and `-var-file` parameters.

```hcl
terraform {
  extra_arguments "common_vars" {
    commands  = get_terraform_commands_that_need_vars()
    arguments = ["-var-file=common.tfvars"]
  }
}
```

---

### get_terraform_commands_that_need_input

Returns commands that accept `-input` parameter.

```hcl
terraform {
  extra_arguments "disable_input" {
    commands  = get_terraform_commands_that_need_input()
    arguments = ["-input=false"]
  }
}
```

---

### get_terraform_commands_that_need_locking

Returns commands that accept `-lock-timeout` parameter.

```hcl
terraform {
  extra_arguments "retry_lock" {
    commands  = get_terraform_commands_that_need_locking()
    arguments = ["-lock-timeout=20m"]
  }
}
```

---

### get_terraform_commands_that_need_parallelism

Returns commands that accept `-parallelism` parameter.

```hcl
terraform {
  extra_arguments "parallelism" {
    commands  = get_terraform_commands_that_need_parallelism()
    arguments = ["-parallelism=5"]
  }
}
```

---

## Configuration Functions

### read_terragrunt_config

Parse and return another terragrunt config file.

```hcl
locals {
  common = read_terragrunt_config(find_in_parent_folders("common.hcl"))
  env    = read_terragrunt_config(find_in_parent_folders("env.hcl"))
}

inputs = merge(
  local.common.inputs,
  local.env.inputs,
  {
    # Additional inputs
  }
)
```

With fallback:
```hcl
locals {
  optional = read_terragrunt_config(
    find_in_parent_folders("optional.hcl", "optional.hcl"),
    { inputs = {} }
  )
}
```

Access dependency outputs from read configs:
```hcl
# common_deps.hcl
dependency "vpc" {
  config_path = "${get_terragrunt_dir()}/../vpc"
}

# terragrunt.hcl
locals {
  deps = read_terragrunt_config(find_in_parent_folders("common_deps.hcl"))
}

inputs = {
  vpc_id = local.deps.dependency.vpc.outputs.vpc_id
}
```

Works with `terragrunt.stack.hcl` and `terragrunt.values.hcl` files.

---

### read_tfvars_file

Read a `.tfvars` or `.tfvars.json` file.

```hcl
locals {
  vars = read_tfvars_file("common.tfvars")
}

inputs = merge(
  local.vars,
  {
    # Additional inputs
  }
)
```

```hcl
locals {
  backend = read_tfvars_file("backend.tfvars")
}

remote_state {
  backend = "s3"
  config = {
    bucket = local.backend.bucket
    region = local.backend.region
  }
}
```

---

### sops_decrypt_file

Decrypt a file encrypted with [SOPS](https://github.com/getsops/sops).

```hcl
locals {
  secrets = yamldecode(sops_decrypt_file(find_in_parent_folders("secrets.yaml")))
}

inputs = merge(
  local.secrets,
  {
    # Additional inputs
  }
)
```

With fallback:
```hcl
locals {
  secrets = try(
    jsondecode(sops_decrypt_file("secrets.json")),
    {}
  )
}
```

Supports: YAML, JSON, ENV, INI, and raw text files.

---

### get_terragrunt_source_cli_flag

Returns the value of `--source` CLI flag or `TG_SOURCE` env var (empty string if not set).

```hcl
locals {
  is_local_dev = get_terragrunt_source_cli_flag() != ""
}

# Conditional behavior for local development
terraform {
  before_hook "debug" {
    commands = ["plan"]
    execute  = local.is_local_dev ? ["echo", "Local dev mode"] : ["echo", "Remote mode"]
  }
}
```

---

## Utility Functions

### run_cmd

Execute a shell command and return stdout.

```hcl
remote_state {
  backend = "s3"
  config = {
    bucket         = run_cmd("./get_bucket_name.sh")
    dynamodb_table = run_cmd("./get_table_name.sh")
  }
}
```

### Special Parameters

| Parameter | Description |
|-----------|-------------|
| `--terragrunt-quiet` | Suppress stdout in logs (still returns value) |
| `--terragrunt-global-cache` | Cache results globally (directory-independent) |
| `--terragrunt-no-cache` | Disable caching, run every time |

```hcl
locals {
  # Suppress sensitive output
  secret = run_cmd("--terragrunt-quiet", "./decrypt.sh", "secret-key")

  # Global cache for account ID
  account_id = run_cmd(
    "--terragrunt-global-cache",
    "aws", "sts", "get-caller-identity",
    "--query", "Account", "--output", "text"
  )

  # Fresh value each time
  timestamp = run_cmd("--terragrunt-no-cache", "date", "+%s")

  # Combine flags
  session_token = run_cmd(
    "--terragrunt-no-cache",
    "--terragrunt-quiet",
    "./generate-token.sh"
  )
}
```

### Caching Behavior

By default, `run_cmd` caches results by directory + command. Each invocation with different arguments runs separately:

```hcl
locals {
  # Different cache keys due to uuid()
  uuid1 = run_cmd("echo", uuid())
  uuid2 = run_cmd("echo", uuid())

  # Same cache key, runs once
  potato1 = run_cmd("echo", "potato")
  potato2 = run_cmd("echo", "potato")  # Uses cached result
}
```

---

### get_default_retryable_errors

Returns the default list of retryable error patterns.

```hcl
errors {
  retry "default" {
    retryable_errors   = get_default_retryable_errors()
    max_attempts       = 3
    sleep_interval_sec = 5
  }

  retry "custom" {
    retryable_errors   = [".*my custom error.*"]
    max_attempts       = 5
    sleep_interval_sec = 10
  }
}
```

---

### mark_as_read

Mark a file as read for `--queue-include-units-reading` flag support.

```hcl
locals {
  # Mark file read by external tools
  config_file = mark_as_read("/path/to/config-read-by-terraform.txt")

  # Mark multiple files
  yaml_files = [
    for f in fileset("./config", "*.yaml") :
    file(mark_as_read(abspath("${get_terragrunt_dir()}/config/${f}")))
  ]
}

inputs = {
  config_path = local.config_file
}
```

Use with:
```bash
terragrunt run --all --queue-include-units-reading config.txt plan
```

**Note**: Must use absolute paths. Must be in `locals` block for proper queue population.

---

### constraint_check

Check if a version satisfies a constraint.

```hcl
feature "module_version" {
  default = "1.2.3"
}

locals {
  version            = feature.module_version.value
  is_v2              = constraint_check(local.version, ">= 2.0.0")
  needs_legacy_input = constraint_check(local.version, "< 1.5.0")
}

terraform {
  source = "github.com/org/module.git//?ref=v${local.version}"
}

# Adjust inputs based on version
inputs = local.is_v2 ? {
  new_input_name = "value"
} : {
  old_input_name = "value"
}
```

Supports same constraint syntax as `terragrunt_version_constraint` and `terraform_version_constraint`.
