# Terragrunt Core Concepts Reference

This document covers the fundamental concepts for working with Terragrunt.

## Table of Contents

- [Units](#units)
- [Stacks](#stacks)
- [Dependencies](#dependencies)
- [Includes](#includes)
- [Run Queue and DAG](#run-queue-and-dag)

---

## Units

A **unit** is the smallest deployable entity in Terragrunt - a directory containing a `terragrunt.hcl` file.

### Key Characteristics

- **Atomic**: Each unit represents one infrastructure deployment
- **Self-contained**: All configuration for a deployment is in one file
- **Immutable**: Uses versioned modules for reproducibility

### Directory Structure

```
live/
├── prod/
│   ├── app/
│   │   └── terragrunt.hcl
│   ├── mysql/
│   │   └── terragrunt.hcl
│   └── vpc/
│       └── terragrunt.hcl
└── stage/
    ├── app/
    │   └── terragrunt.hcl
    ├── mysql/
    │   └── terragrunt.hcl
    └── vpc/
        └── terragrunt.hcl
```

### Basic Unit Configuration

```hcl
# live/prod/app/terragrunt.hcl

terraform {
  source = "git::git@github.com:org/modules.git//app?ref=v1.0.0"
}

inputs = {
  instance_count = 10
  instance_type  = "m5.large"
}
```

### Working with Units

```bash
# Deploy a single unit
cd live/prod/app
terragrunt apply

# Local development with source override
terragrunt apply --source ../../../modules//app
```

### Remote Module Sources

Terragrunt downloads modules to `.terragrunt-cache/`:

```hcl
# Git with ref
terraform {
  source = "git::git@github.com:org/repo.git//path/to/module?ref=v1.0.0"
}

# Terraform Registry
terraform {
  source = "tfr:///terraform-aws-modules/vpc/aws?version=3.0.0"
}

# Local path
terraform {
  source = "../../../modules//vpc"
}
```

### Lock Files

Lock files (`.terraform.lock.hcl`) are generated next to `terragrunt.hcl` and should be committed to version control.

---

## Stacks

A **stack** is a collection of units that can be managed together.

### Implicit Stacks

Created by organizing units in directories. Any directory with multiple units is an implicit stack.

```
root/                          # Implicit stack
├── backend-app/
│   └── terragrunt.hcl
├── mysql/
│   └── terragrunt.hcl
└── vpc/
    └── terragrunt.hcl
```

Working with implicit stacks:
```bash
# Run on all units in current directory tree
cd root
terragrunt run --all plan
terragrunt run --all apply
terragrunt run --all destroy
```

### Explicit Stacks

Defined using `terragrunt.stack.hcl` files - blueprints that generate units.

```hcl
# terragrunt.stack.hcl

unit "vpc" {
  source = "git::git@github.com:org/catalog.git//units/vpc?ref=v1.0.0"
  path   = "vpc"
  values = {
    cidr = "10.0.0.0/16"
  }
}

unit "database" {
  source = "git::git@github.com:org/catalog.git//units/database?ref=v1.0.0"
  path   = "database"
  values = {
    engine   = "postgres"
    vpc_path = "../vpc"
  }
}
```

Working with explicit stacks:
```bash
# Generate units from stack definition
terragrunt stack generate

# Run on generated stack
terragrunt stack run plan
terragrunt stack run apply

# Get aggregated outputs
terragrunt stack output
terragrunt stack output --format json

# Clean generated files
terragrunt stack clean
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

### Nested Stacks

Stacks can reference other stacks for reusable patterns:

```hcl
# terragrunt.stack.hcl

stack "dev" {
  source = "./stacks/environment"
  path   = "dev"
  values = {
    environment = "development"
  }
}

stack "prod" {
  source = "./stacks/environment"
  path   = "prod"
  values = {
    environment = "production"
  }
}
```

### When to Use Each

**Use Implicit Stacks when:**
- Small number of unique units
- Getting started with Terragrunt
- Maximum transparency needed

**Use Explicit Stacks when:**
- Multiple similar environments
- Reusable infrastructure patterns
- Version-controlled collections of units

---

## Dependencies

### dependency Block

Access outputs from other units:

```hcl
# mysql/terragrunt.hcl

dependency "vpc" {
  config_path = "../vpc"
}

inputs = {
  vpc_id     = dependency.vpc.outputs.vpc_id
  subnet_ids = dependency.vpc.outputs.private_subnet_ids
}
```

### Multiple Dependencies

```hcl
# backend-app/terragrunt.hcl

dependency "mysql" {
  config_path = "../mysql"
}

dependency "redis" {
  config_path = "../redis"
}

inputs = {
  mysql_url = dependency.mysql.outputs.endpoint
  redis_url = dependency.redis.outputs.endpoint
}
```

### Mock Outputs

Handle unapplied dependencies for planning:

```hcl
dependency "vpc" {
  config_path = "../vpc"

  mock_outputs = {
    vpc_id     = "vpc-mock"
    subnet_ids = ["subnet-1", "subnet-2"]
  }

  # Only use mocks for these commands
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}
```

### Merge Strategies

Combine mock and real outputs:

```hcl
dependency "vpc" {
  config_path = "../vpc"

  mock_outputs = {
    vpc_id     = "default"
    new_output = "default"
  }

  # Options: no_merge, shallow, deep_map_only
  mock_outputs_merge_strategy_with_state = "shallow"
}
```

### dependencies Block

Specify ordering without accessing outputs:

```hcl
dependencies {
  paths = ["../vpc", "../security-groups"]
}
```

### Dependencies in Explicit Stacks

Pass dependency paths as values:

```hcl
# terragrunt.stack.hcl
unit "app" {
  source = "./units/app"
  path   = "app"
  values = {
    vpc_path   = "../vpc"
    mysql_path = "../mysql"
  }
}
```

```hcl
# units/app/terragrunt.hcl
dependency "vpc" {
  config_path = values.vpc_path
}

dependency "mysql" {
  config_path = values.mysql_path
}
```

---

## Includes

Share configuration across units using `include` blocks.

### Basic Include

```hcl
# root.hcl
remote_state {
  backend = "s3"
  config = {
    bucket = "my-state"
    key    = "${path_relative_to_include()}/terraform.tfstate"
    region = "us-east-1"
  }
}

# prod/app/terragrunt.hcl
include "root" {
  path = find_in_parent_folders("root.hcl")
}
```

### Multiple Includes

```hcl
# prod/app/terragrunt.hcl
include "root" {
  path = find_in_parent_folders("root.hcl")
}

include "env" {
  path = "${get_terragrunt_dir()}/../../_env/app.hcl"
}

inputs = {
  env = "prod"
}
```

### Exposed Includes

Access data from included configs:

```hcl
include "env" {
  path   = find_in_parent_folders("env.hcl")
  expose = true
}

terraform {
  source = "${include.env.locals.source_base_url}?ref=v2.0.0"
}

inputs = {
  region = include.env.locals.region
}
```

### Merge Strategies

```hcl
include "root" {
  path           = find_in_parent_folders("root.hcl")
  merge_strategy = "deep"  # or "shallow", "no_merge"
}
```

Deep merge behavior:
- Simple types: child overrides parent
- Lists: concatenated
- Maps: recursively merged
- Blocks: merged by label

### Dynamic Configuration with read_terragrunt_config

Load context based on directory structure:

```hcl
# _env/app.hcl
locals {
  env_vars = read_terragrunt_config(find_in_parent_folders("env.hcl"))
  env      = local.env_vars.locals.env
}

inputs = {
  env      = local.env
  basename = "app-${local.env}"
}
```

```hcl
# prod/env.hcl
locals {
  env = "prod"
}

# stage/env.hcl
locals {
  env = "stage"
}
```

### Recommended Structure

```
live/
├── root.hcl              # Global config (remote state, providers)
├── _env/                 # Shared component configs
│   ├── app.hcl
│   ├── mysql.hcl
│   └── vpc.hcl
├── prod/
│   ├── env.hcl           # Environment-specific values
│   ├── app/
│   │   └── terragrunt.hcl
│   └── mysql/
│       └── terragrunt.hcl
└── stage/
    ├── env.hcl
    ├── app/
    │   └── terragrunt.hcl
    └── mysql/
        └── terragrunt.hcl
```

---

## Run Queue and DAG

Terragrunt uses a Directed Acyclic Graph (DAG) to manage execution order.

### How It Works

1. **Discovery**: Find all units in current directory
2. **DAG Construction**: Build dependency graph from `dependency`/`dependencies` blocks
3. **Execution**: Run units respecting dependency order

### Execution Order

**For apply/plan**: Dependencies run before dependents
```
vpc → mysql → app
      redis ↗
```

**For destroy**: Dependents run before dependencies
```
app → mysql → vpc
    ↘ redis
```

### Visualizing the DAG

```bash
# Generate DOT format
terragrunt list --format dot --dependencies --external

# Render to image
terragrunt dag graph | dot -Tsvg > graph.svg

# List in DAG order
terragrunt list --dag

# List as if running specific command
terragrunt list --queue-construct-as destroy
```

### Controlling Execution

```bash
# Limit parallelism
terragrunt run --all --parallelism 4 apply

# Ignore dependency order (dangerous for apply/destroy)
terragrunt run --all --queue-ignore-dag-order plan

# Continue on errors
terragrunt run --all --queue-ignore-errors plan

# Stop on first failure
terragrunt run --all --fail-fast plan
```

### Filtering the Queue

```bash
# Include only specific paths
terragrunt run --all --filter './prod/**' -- plan

# Exclude patterns
terragrunt run --all --filter '!./test/**' -- plan

# Run affected components only
terragrunt run --all --filter-affected -- plan
```

### Saving Plans

```bash
# Save plans to directory (maintains structure)
terragrunt run --all --out-dir /tmp/plans plan

# Save as JSON
terragrunt run --all --json-out-dir /tmp/plans plan

# Apply saved plans
terragrunt run --all --out-dir /tmp/plans apply
```

### Nested Stacks

Control blast radius with directory structure:

```
root/                      # Full stack
├── us-east-1/             # Regional stack
│   ├── app/
│   └── db/
└── us-west-2/             # Regional stack
    ├── app/
    └── db/
```

```bash
# Deploy everything
cd root
terragrunt run --all apply

# Deploy single region
cd root/us-east-1
terragrunt run --all apply
```

### Dependency Optimization

Terragrunt optimizes dependency fetching when:
- Remote state uses `remote_state` blocks
- `disable_dependency_optimization = false` (default)
- `remote_state` doesn't depend on `dependency` outputs
- No custom hooks on `terraform init`

This avoids recursive parsing of dependency chains.
