# Terragrunt HCL Blocks Reference

Terragrunt HCL configuration uses configuration blocks to define structural configurations. Blocks control different systems used by Terragrunt.

## Table of Contents

- [terraform](#terraform)
- [remote_state](#remote_state)
- [include](#include)
- [locals](#locals)
- [dependency](#dependency)
- [dependencies](#dependencies)
- [generate](#generate)
- [feature](#feature)
- [exclude](#exclude)
- [errors](#errors)
- [unit](#unit)
- [stack](#stack)

---

## terraform

The `terraform` block configures how Terragrunt interacts with OpenTofu/Terraform, including where to find configuration files, extra arguments, and hooks.

### Attributes

- **`source`**: Where to find OpenTofu/Terraform configuration files. Supports:
  - Local file paths: `source = "../modules/vpc"`
  - Git URLs: `source = "git::git@github.com:org/repo.git//path?ref=v1.0.0"`
  - Terraform Registry (tfr protocol): `source = "tfr:///terraform-aws-modules/vpc/aws?version=3.3.0"`

- **`include_in_copy`**: Glob patterns of files to always copy into the working directory (e.g., `[".python-version", "*.yaml"]`)

- **`exclude_from_copy`**: Glob patterns of files to exclude from copying

- **`copy_terraform_lock_file`**: Boolean to control copying `.terraform.lock.hcl` (default: `true`)

### Nested Blocks

#### extra_arguments

Pass extra CLI arguments to terraform commands:

```hcl
terraform {
  extra_arguments "retry_lock" {
    commands  = get_terraform_commands_that_need_locking()
    arguments = ["-lock-timeout=20m"]
  }

  extra_arguments "custom_vars" {
    commands           = ["apply", "plan"]
    required_var_files = ["${get_parent_terragrunt_dir()}/common.tfvars"]
    optional_var_files = ["${get_terragrunt_dir()}/override.tfvars"]
    env_vars = {
      TF_VAR_region = "us-east-1"
    }
  }
}
```

#### before_hook / after_hook

Run commands before or after terraform operations:

```hcl
terraform {
  before_hook "before_plan" {
    commands     = ["apply", "plan"]
    execute      = ["echo", "Running Terraform"]
    run_on_error = false
  }

  after_hook "after_apply" {
    commands     = ["apply"]
    execute      = ["./notify.sh"]
    run_on_error = true
  }

  # Special hooks
  after_hook "init_from_module" {
    commands = ["init-from-module"]
    execute  = ["cp", "${get_parent_terragrunt_dir()}/foo.tf", "."]
  }
}
```

Hook attributes:
- `commands`: List of terraform commands to trigger the hook
- `execute`: Command and arguments to run
- `working_dir`: Optional working directory
- `run_on_error`: Run even if previous hooks or terraform failed
- `suppress_stdout`: Suppress command output
- `if`: Conditional execution

Special hook commands:
- `read-config`: Runs after terragrunt loads config (after_hook only)
- `init-from-module`: Runs when downloading remote configurations
- `init`: Runs during terraform init for Auto-Init

#### error_hook

Handle specific errors:

```hcl
terraform {
  error_hook "handle_errors" {
    commands  = ["apply", "plan"]
    execute   = ["echo", "Error occurred"]
    on_errors = [".*Error: transient.*"]
  }
}
```

### Complete Example

```hcl
terraform {
  source = "git::git@github.com:acme/modules.git//vpc?ref=v0.0.1"

  extra_arguments "retry_lock" {
    commands  = get_terraform_commands_that_need_locking()
    arguments = ["-lock-timeout=20m"]
  }

  before_hook "pre_apply" {
    commands = ["apply", "plan"]
    execute  = ["echo", "Starting..."]
  }

  after_hook "post_apply" {
    commands     = ["apply"]
    execute      = ["./cleanup.sh"]
    run_on_error = true
  }
}
```

---

## remote_state

Configures remote state backend with automatic initialization support.

### Attributes

- **`backend`**: Backend type (`s3`, `gcs`, `azurerm`, etc.)
- **`disable_init`**: Skip automatic backend initialization
- **`disable_dependency_optimization`**: Disable optimized dependency fetching
- **`generate`**: Auto-generate backend configuration file
- **`config`**: Backend configuration map
- **`encryption`**: State encryption configuration (OpenTofu)

### S3 Backend Example

```hcl
remote_state {
  backend = "s3"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
  config = {
    bucket         = "my-terraform-state"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

### S3 Backend Additional Options

```hcl
remote_state {
  backend = "s3"
  config = {
    bucket         = "my-state"
    key            = "terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true

    # Locking options
    dynamodb_table = "tf-locks"           # DynamoDB locking
    use_lockfile   = true                 # Native S3 locking (OpenTofu >= 1.10)

    # Bucket creation options
    skip_bucket_versioning            = false
    skip_bucket_ssencryption          = false
    skip_bucket_root_access           = false
    skip_bucket_enforced_tls          = false
    skip_bucket_public_access_blocking = false

    # DynamoDB options
    enable_lock_table_ssencryption = true

    # Access logging
    accesslogging_bucket_name = "my-state-logs"
    accesslogging_target_prefix = "logs/"

    # Tags
    s3_bucket_tags = {
      Environment = "production"
    }
    dynamodb_table_tags = {
      Environment = "production"
    }
  }
}
```

### GCS Backend Example

```hcl
remote_state {
  backend = "gcs"
  config = {
    project  = "my-gcp-project"
    location = "us"
    bucket   = "my-terraform-state"
    prefix   = "${path_relative_to_include()}/terraform.tfstate"

    skip_bucket_creation   = false
    skip_bucket_versioning = false
    enable_bucket_policy_only = true

    gcs_bucket_labels = {
      owner = "terraform"
    }
  }
}
```

### State Encryption (OpenTofu)

```hcl
remote_state {
  backend = "s3"
  config = {
    bucket = "mybucket"
    key    = "terraform.tfstate"
    region = "us-east-1"
  }
  encryption = {
    key_provider = "pbkdf2"
    passphrase   = get_env("PBKDF2_PASSPHRASE")
  }
}
```

### Dynamic remote_state

```hcl
locals {
  common = read_terragrunt_config(find_in_parent_folders("common.hcl"))
}

remote_state = local.common.remote_state
```

---

## include

Inherit configuration from parent files. Supports multiple includes with labels.

### Attributes

- **`name`** (label): Unique identifier for the include
- **`path`**: Path to parent configuration file
- **`expose`**: Make included config available as `include.<name>` variable
- **`merge_strategy`**: How to merge configs (`no_merge`, `shallow`, `deep`)

### Single Include

```hcl
include "root" {
  path = find_in_parent_folders("root.hcl")
}
```

### Multiple Includes with Expose

```hcl
include "root" {
  path   = find_in_parent_folders("root.hcl")
  expose = true
}

include "env" {
  path           = find_in_parent_folders("env.hcl")
  expose         = true
  merge_strategy = "no_merge"
}

inputs = {
  remote_state_config = include.root.remote_state
  environment         = include.env.locals.env
}
```

### Deep Merge

```hcl
include "root" {
  path           = find_in_parent_folders("root.hcl")
  merge_strategy = "deep"
}
```

Deep merge behavior:
- Simple types: child overrides parent
- Lists: concatenated
- Maps: recursively merged
- Blocks: merged by label, or appended

**Limitations**: `remote_state` and `generate` blocks use shallow merge only.

---

## locals

Define local variables for reuse within the configuration.

```hcl
locals {
  aws_region = "us-east-1"

  regions = ["us-east-1", "us-west-2"]

  region_config = {
    us-east-1 = { instance_type = "t3.micro" }
    us-west-2 = { instance_type = "t3.small" }
  }

  # Read from other files
  env_vars = read_terragrunt_config(find_in_parent_folders("env.hcl"))
  env_name = local.env_vars.locals.env

  # Read YAML/JSON
  config = yamldecode(file("${get_terragrunt_dir()}/config.yaml"))
}

inputs = {
  region        = local.aws_region
  instance_type = local.region_config[local.aws_region].instance_type
}
```

---

## dependency

Access outputs from other Terragrunt units.

### Attributes

- **`name`** (label): Reference name for the dependency
- **`config_path`**: Path to the dependency unit
- **`enabled`**: Enable/disable the dependency
- **`skip_outputs`**: Skip fetching outputs (use with mock_outputs)
- **`mock_outputs`**: Fallback outputs when real outputs unavailable
- **`mock_outputs_allowed_terraform_commands`**: Commands that can use mocks
- **`mock_outputs_merge_strategy_with_state`**: Merge strategy (`no_merge`, `shallow`, `deep_map_only`)

### Basic Usage

```hcl
dependency "vpc" {
  config_path = "../vpc"
}

dependency "rds" {
  config_path = "../rds"
}

inputs = {
  vpc_id = dependency.vpc.outputs.vpc_id
  db_url = dependency.rds.outputs.endpoint
}
```

### With Mock Outputs

```hcl
dependency "vpc" {
  config_path = "../vpc"

  mock_outputs = {
    vpc_id     = "mock-vpc-id"
    subnet_ids = ["subnet-1", "subnet-2"]
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}
```

### Merge Mocks with State

```hcl
dependency "vpc" {
  config_path = "../vpc"

  mock_outputs = {
    vpc_id     = "default-vpc"
    new_output = "default-value"
  }
  mock_outputs_merge_strategy_with_state = "shallow"
}
```

---

## dependencies

Specify ordering dependencies without accessing outputs. Used for `run --all` commands.

```hcl
dependencies {
  paths = ["../vpc", "../rds"]
}
```

---

## generate

Generate files in the terraform working directory before runs.

### Attributes

- **`name`** (label): Unique identifier
- **`path`**: Output file path
- **`if_exists`**: Action if file exists (`overwrite`, `overwrite_terragrunt`, `skip`, `error`)
- **`if_disabled`**: Action if disabled (`remove`, `remove_terragrunt`, `skip`)
- **`contents`**: File contents
- **`comment_prefix`**: Comment prefix for signature (default: `#`)
- **`disable_signature`**: Disable terragrunt signature
- **`disable`**: Disable generation

### Provider Generation

```hcl
generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
provider "aws" {
  region = "us-east-1"
  assume_role {
    role_arn = "arn:aws:iam::123456789:role/terraform"
  }
}
EOF
}
```

### Backend Generation

```hcl
generate "backend" {
  path      = "backend.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
terraform {
  backend "s3" {
    bucket = "my-state"
    key    = "terraform.tfstate"
    region = "us-east-1"
  }
}
EOF
}
```

---

## feature

Define feature flags that can be overridden via CLI.

```hcl
feature "environment" {
  default = "dev"
}

feature "enable_monitoring" {
  default = false
}

# Dynamic default from external source
feature "version" {
  default = run_cmd("--terragrunt-quiet", "get-latest-version.sh")
}

inputs = {
  environment = feature.environment.value
  monitoring  = feature.enable_monitoring.value
}
```

Override via CLI:
```bash
terragrunt --feature environment=prod --feature enable_monitoring=true apply
```

Override via environment:
```bash
export TG_FEATURE=environment=prod,enable_monitoring=true
terragrunt apply
```

---

## exclude

Control when units are excluded from execution.

### Attributes

- **`if`**: Boolean condition for exclusion
- **`actions`**: Actions to exclude (`plan`, `apply`, `destroy`, `all`, `all_except_output`)
- **`exclude_dependencies`**: Also exclude dependencies
- **`no_run`**: Prevent unit from running entirely when conditions match

```hcl
feature "skip_in_dev" {
  default = false
}

exclude {
  if                   = feature.skip_in_dev.value
  actions              = ["plan", "apply"]
  exclude_dependencies = false
}
```

### Exclude All Except Output

```hcl
exclude {
  if      = true
  actions = ["all_except_output"]
}
```

### Prevent Unit from Running

```hcl
exclude {
  if      = true
  no_run  = true
  actions = ["plan"]
}
```

---

## errors

Configure error handling with retry and ignore rules.

### Retry Configuration

```hcl
errors {
  retry "transient_errors" {
    retryable_errors   = [".*Error: transient network issue.*"]
    max_attempts       = 5
    sleep_interval_sec = 10
  }
}
```

### Ignore Configuration

```hcl
errors {
  ignore "known_safe_errors" {
    ignorable_errors = [
      ".*Error: safe warning.*",
      "!.*Error: critical.*"  # Never ignore critical errors
    ]
    message = "Ignoring safe warning"
    signals = {
      safe_to_revert = true
    }
  }
}
```

### Combined Example

```hcl
errors {
  retry "network_errors" {
    retryable_errors   = get_default_retryable_errors()
    max_attempts       = 3
    sleep_interval_sec = 5
  }

  retry "custom_errors" {
    retryable_errors   = [".*timeout.*"]
    max_attempts       = 5
    sleep_interval_sec = 10
  }

  ignore "warnings" {
    ignorable_errors = [".*Warning:.*"]
    message          = "Ignoring warnings"
  }
}
```

---

## unit

Define deployment units within a `terragrunt.stack.hcl` file.

### Attributes

- **`name`** (label): Unique identifier
- **`source`**: Source of the unit configuration
- **`path`**: Where to generate the unit
- **`values`**: Values to pass to the unit
- **`no_dot_terragrunt_stack`**: Generate outside `.terragrunt-stack` directory
- **`no_validation`**: Skip validation checks

```hcl
# terragrunt.stack.hcl

unit "vpc" {
  source = "git::git@github.com:org/modules.git//vpc?ref=v1.0.0"
  path   = "vpc"
  values = {
    cidr     = "10.0.0.0/16"
    vpc_name = "main"
  }
}

unit "database" {
  source = "git::git@github.com:org/modules.git//rds?ref=v1.0.0"
  path   = "database"
  values = {
    engine   = "postgres"
    vpc_path = "../vpc"
  }
}
```

Generated structure:
```
.terragrunt-stack/
├── vpc/
│   ├── terragrunt.hcl
│   └── terragrunt.values.hcl
└── database/
    ├── terragrunt.hcl
    └── terragrunt.values.hcl
```

Access values in generated unit:
```hcl
# .terragrunt-stack/vpc/terragrunt.hcl
inputs = {
  cidr     = values.cidr
  vpc_name = values.vpc_name
}
```

---

## stack

Define nested stacks within a `terragrunt.stack.hcl` file for reusable infrastructure patterns.

### Attributes

- **`name`** (label): Unique identifier
- **`source`**: Source of the stack configuration
- **`path`**: Where to generate the stack
- **`values`**: Values to pass to the stack
- **`no_dot_terragrunt_stack`**: Generate outside `.terragrunt-stack` directory
- **`no_validation`**: Skip validation checks

```hcl
# terragrunt.stack.hcl

stack "dev" {
  source = "git::git@github.com:org/stacks.git//environment?ref=v1.0.0"
  path   = "dev"
  values = {
    environment = "development"
    cidr        = "10.0.0.0/16"
  }
}

stack "prod" {
  source = "git::git@github.com:org/stacks.git//environment?ref=v1.0.0"
  path   = "prod"
  values = {
    environment = "production"
    cidr        = "10.1.0.0/16"
  }
}
```

Referenced stack:
```hcl
# stacks/environment/terragrunt.stack.hcl

unit "vpc" {
  source = "../units/vpc"
  path   = "vpc"
  values = {
    cidr = values.cidr
  }
}

unit "app" {
  source = "../units/app"
  path   = "app"
  values = {
    environment = values.environment
    vpc_path    = "../vpc"
  }
}
```

Generated structure:
```
.terragrunt-stack/
├── dev/
│   └── .terragrunt-stack/
│       ├── vpc/
│       └── app/
└── prod/
    └── .terragrunt-stack/
        ├── vpc/
        └── app/
```
