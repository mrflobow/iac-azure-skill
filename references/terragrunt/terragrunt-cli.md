# Terragrunt CLI Reference

Terragrunt extends OpenTofu/Terraform with additional commands for managing infrastructure at scale.

## Table of Contents

- [Usage](#usage)
- [Main Commands](#main-commands)
  - [run](#run)
  - [exec](#exec)
- [Stack Commands](#stack-commands)
  - [stack generate](#stack-generate)
  - [stack run](#stack-run)
  - [stack output](#stack-output)
  - [stack clean](#stack-clean)
- [Backend Commands](#backend-commands)
  - [backend bootstrap](#backend-bootstrap)
  - [backend migrate](#backend-migrate)
  - [backend delete](#backend-delete)
- [Discovery Commands](#discovery-commands)
  - [find](#find)
  - [list](#list)
- [Configuration Commands](#configuration-commands)
  - [render](#render)
  - [hcl fmt](#hcl-fmt)
  - [hcl validate](#hcl-validate)
- [Catalog Commands](#catalog-commands)
  - [catalog](#catalog)
  - [scaffold](#scaffold)
- [Global Flags](#global-flags)

---

## Usage

Replace `tofu`/`terraform` with `terragrunt` for most commands:

```bash
# These are equivalent
terragrunt plan
terragrunt run plan

# Pass arguments to terraform
terragrunt plan -no-color
terragrunt run -- plan -no-color
```

Terragrunt automatically runs `init` when needed (Auto-Init).

---

## Main Commands

### run

Run OpenTofu/Terraform commands with Terragrunt orchestration.

```bash
# Basic usage
terragrunt run plan
terragrunt run apply
terragrunt run destroy

# With terraform arguments (use -- separator)
terragrunt run -- plan -no-color
terragrunt run -- output -json
```

#### Run on Multiple Units

```bash
# Run on all units in current directory tree
terragrunt run --all plan
terragrunt run --all apply
terragrunt run --all destroy

# Run on dependency graph of current unit
terragrunt run --graph apply
```

#### Filtering Units

```bash
# Filter by path glob
terragrunt run --all --filter 'prod/**' -- plan

# Filter by name
terragrunt run --all --filter 'app*' -- apply

# Filter by type
terragrunt run --all --filter 'type=unit' -- plan

# Exclude patterns
terragrunt run --all --filter '!./test/**' -- plan

# Combine filters (intersection)
terragrunt run --all --filter './prod/** | type=unit' -- apply

# Multiple filters (OR logic)
terragrunt run --all --filter 'app1' --filter 'app2' -- plan

# Run on affected components (git changes)
terragrunt run --all --filter-affected -- plan
```

#### Common Flags

| Flag | Description |
|------|-------------|
| `--all` | Run on all units in current directory |
| `--graph` | Run on dependency graph of current unit |
| `--parallelism N` | Limit concurrent executions |
| `--source PATH` | Override terraform source |
| `--source-update` | Force re-download of sources |
| `--out-dir DIR` | Save plan files to directory |
| `--json-out-dir DIR` | Save plan JSON to directory |
| `--no-auto-approve` | Disable auto-approve for apply/destroy |
| `--no-auto-init` | Disable automatic init |
| `--queue-ignore-errors` | Continue on unit failures |
| `--queue-ignore-dag-order` | Ignore dependency order |
| `--fail-fast` | Stop immediately on first failure |

---

### exec

Execute arbitrary commands wrapped by Terragrunt.

```bash
# Execute any command
terragrunt exec -- echo "Hello"

# Execute in download directory (where terraform runs)
terragrunt exec --in-download-dir -- cat main.tf

# Access inputs as environment variables
terragrunt exec -- env | grep TF_VAR_
```

Useful for custom scripts that need Terragrunt's context (inputs, IAM role, etc.).

---

## Stack Commands

Commands for working with `terragrunt.stack.hcl` files.

### stack generate

Generate units from a `terragrunt.stack.hcl` file.

```bash
# Generate stack
terragrunt stack generate

# With parallelism control
terragrunt stack generate --parallelism 4

# Skip validation
terragrunt stack generate --no-stack-validate

# Force source refresh
terragrunt stack generate --source-update
```

Generated output goes to `.terragrunt-stack/` directory.

---

### stack run

Run commands against all units in a stack.

```bash
# Plan all units
terragrunt stack run plan

# Apply all units
terragrunt stack run apply

# Destroy all units
terragrunt stack run destroy

# Skip automatic generation
terragrunt stack run --no-stack-generate plan

# Force source refresh
terragrunt stack run plan --source-update
```

Automatically generates the stack before running if needed.

---

### stack output

Get aggregated outputs from all units in a stack.

```bash
# Get all outputs
terragrunt stack output

# Get outputs in JSON
terragrunt stack output --format json

# Get specific unit output
terragrunt stack output vpc

# Get specific value
terragrunt stack output vpc.vpc_id

# Get raw value (for scripts)
terragrunt stack output --format raw app.domain
```

Output formats: `default` (HCL), `json`, `raw`

---

### stack clean

Remove generated `.terragrunt-stack` directories.

```bash
terragrunt stack clean
```

---

## Backend Commands

Commands for managing OpenTofu/Terraform state backends.

### backend bootstrap

Provision backend infrastructure (S3 buckets, DynamoDB tables, etc.).

```bash
# Bootstrap current unit's backend
terragrunt backend bootstrap

# Bootstrap all units
terragrunt backend bootstrap --all
```

Provisions resources defined in `remote_state` block:
- S3: bucket, DynamoDB table, access logging bucket
- GCS: bucket with labels and versioning

Idempotent - safe to run multiple times.

---

### backend migrate

Migrate state between units (useful when renaming).

```bash
# Migrate state
terragrunt backend migrate old-unit-name new-unit-name

# Force migration (even without versioning)
terragrunt backend migrate --force old-unit new-unit
```

Workflow for renaming a unit:
```bash
cp -R old-unit new-unit
terragrunt backend migrate old-unit new-unit
rm -rf old-unit
```

---

### backend delete

Delete backend state for a unit.

```bash
# Delete state (with confirmation)
terragrunt backend delete

# Delete without confirmation
terragrunt backend delete --non-interactive

# Force delete (without versioning check)
terragrunt backend delete --force

# Delete for all units
terragrunt backend delete --all
```

---

## Discovery Commands

Commands to explore Terragrunt configurations.

### find

Find Terragrunt configurations (units and stacks).

```bash
# Find all configurations
terragrunt find

# JSON output
terragrunt find --format json
terragrunt find --json

# Sort by dependencies (DAG order)
terragrunt find --dag

# Sort as if running a command
terragrunt find --queue-construct-as plan
terragrunt find --queue-construct-as destroy

# Include dependency info
terragrunt find --dependencies --format json

# Include external dependencies
terragrunt find --dependencies --external --format json

# Include hidden directories
terragrunt find --hidden

# Show files read by each unit
terragrunt find --reading --format json
```

---

### list

List configurations with various output formats.

```bash
# Default (columns)
terragrunt list

# Long format (table)
terragrunt list -l
terragrunt list --long

# Tree format
terragrunt list -T
terragrunt list --tree

# With dependencies
terragrunt list -l --dependencies

# DAG order
terragrunt list --dag

# DOT format for graphviz
terragrunt list --format dot --dependencies
terragrunt list --format dot --dependencies | dot -Tsvg > graph.svg

# Sort by command
terragrunt list --queue-construct-as plan
terragrunt list --queue-construct-as destroy
```

---

## Configuration Commands

### render

Render fully resolved Terragrunt configuration.

```bash
# Render to stdout (HCL)
terragrunt render

# Render to JSON
terragrunt render --format json
terragrunt render --json

# Write to file
terragrunt render --json --write

# Render all configurations
terragrunt render --all --json --write
```

Output shows all includes merged, functions evaluated, and variables resolved.

---

### hcl fmt

Format Terragrunt HCL files.

```bash
# Format all files
terragrunt hcl fmt

# Check only (exit 1 if changes needed)
terragrunt hcl fmt --check

# Show diff
terragrunt hcl fmt --diff

# Exclude directories
terragrunt hcl fmt --exclude-dir .terragrunt-cache

# Format single file
terragrunt hcl fmt --file terragrunt.hcl
```

---

### hcl validate

Validate Terragrunt HCL configurations.

```bash
# Validate all configurations
terragrunt hcl validate

# JSON output
terragrunt hcl validate --json

# Show config paths in output
terragrunt hcl validate --show-config-path

# Validate inputs against terraform variables
terragrunt hcl validate --inputs

# Strict mode
terragrunt hcl validate --strict
```

---

## Catalog Commands

### catalog

Browse and use OpenTofu/Terraform modules via TUI.

```bash
# Launch catalog TUI
terragrunt catalog

# Specify catalog URL
terragrunt catalog https://github.com/org/modules

# Custom root file name
terragrunt catalog --root-file-name root.hcl
```

---

### scaffold

Generate Terragrunt configuration from a module.

```bash
# Scaffold from GitHub
terragrunt scaffold github.com/org/modules//vpc

# With custom variables
terragrunt scaffold github.com/org/modules//vpc \
  --var region=us-east-1 \
  --var environment=prod

# With variable file
terragrunt scaffold github.com/org/modules//vpc \
  --var-file common.tfvars

# Custom root file name
terragrunt scaffold github.com/org/modules//vpc \
  --root-file-name root.hcl
```

---

## Global Flags

Available for all commands:

| Flag | Env Var | Description |
|------|---------|-------------|
| `--working-dir DIR` | `TG_WORKING_DIR` | Change working directory |
| `--config FILE` | `TG_CONFIG` | Path to terragrunt.hcl |
| `--log-level LEVEL` | `TG_LOG_LEVEL` | Log level (debug, info, warn, error) |
| `--log-format FORMAT` | `TG_LOG_FORMAT` | Log format (text, json) |
| `--log-disable` | `TG_LOG_DISABLE` | Disable logging |
| `--no-color` | `TG_NO_COLOR` | Disable color output |
| `--non-interactive` | `TG_NON_INTERACTIVE` | Disable interactive prompts |
| `--strict-mode` | `TG_STRICT_MODE` | Enable strict mode |
| `--experiment NAME` | `TG_EXPERIMENT` | Enable experimental features |
| `--help` | | Show help |
| `--version` | | Show version |

---

## Common Workflows

### Deploy Everything

```bash
# Plan all
terragrunt run --all plan

# Apply all (respects DAG order)
terragrunt run --all apply
```

### Deploy Single Unit with Dependencies

```bash
cd units/app
terragrunt run --graph apply
```

### Save Plans for Review

```bash
# Save binary plans
terragrunt run --all --out-dir /tmp/plans plan

# Save JSON plans
terragrunt run --all --json-out-dir /tmp/plans plan

# Apply saved plans
terragrunt run --all --out-dir /tmp/plans apply
```

### Destroy in Correct Order

```bash
# Preview destroy order
terragrunt list --queue-construct-as destroy

# Destroy
terragrunt run --all destroy
```

### Visualize Dependencies

```bash
# Generate graph
terragrunt list --format dot --dependencies --external | dot -Tsvg > deps.svg

# Or use dag command
terragrunt dag graph | dot -Tpng > deps.png
```

### Local Development

```bash
# Override source for testing
terragrunt run --source ../local-modules//vpc plan

# Or set environment variable
export TG_SOURCE=../local-modules
terragrunt plan
```

### CI/CD: Run Affected Units

```bash
# Run only changed components
terragrunt run --all --filter-affected -- plan
terragrunt run --all --filter-affected -- apply
```
