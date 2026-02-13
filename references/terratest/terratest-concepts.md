# Terratest Core Concepts Reference

This document covers the fundamental concepts for testing Infrastructure as Code with Terratest.

## Table of Contents

- [What is Terratest](#what-is-terratest)
- [Test Lifecycle](#test-lifecycle)
- [Requirements](#requirements)
- [Project Structure](#project-structure)
- [Terraform Options](#terraform-options)
- [Terragrunt Options](#terragrunt-options)
- [Key Packages](#key-packages)
- [Error Handling Pattern](#error-handling-pattern)
- [Terragrunt 0.85+ Compatibility](#terragrunt-085-compatibility)

---

## What is Terratest

Terratest is a Go library by Gruntwork that provides patterns and helper functions for testing infrastructure code. It deploys real infrastructure (Terraform, Terragrunt, Packer, Docker, etc.), validates it works correctly, then tears it down.

Import path: `github.com/gruntwork-io/terratest/modules/<package>`

---

## Test Lifecycle

Every Terratest test follows the **Deploy, Validate, Undeploy** pattern:

1. **Deploy** — Run `terraform apply` or `terragrunt apply` on real infrastructure
2. **Validate** — Make HTTP requests, API calls, or SDK queries to verify the infrastructure behaves correctly
3. **Undeploy** — Run `terraform destroy` or `terragrunt destroy` to clean up

The `defer` keyword ensures cleanup runs even if the test fails:

```go
defer terraform.Destroy(t, terraformOptions)
terraform.InitAndApply(t, terraformOptions)
// ... validate ...
```

---

## Requirements

| Requirement | Version | Notes |
|---|---|---|
| Go | 1.21.1+ | Required for module support |
| Terraform | Any supported | Must be on `PATH` |
| Terragrunt | < 0.85 recommended | See [compatibility warning](#terragrunt-085-compatibility) |
| Azure CLI | Latest | For Azure tests; must be authenticated |

---

## Project Structure

### Standard module with tests

```
module/
├── main.tf
├── variables.tf
├── outputs.tf
└── test/
    ├── module_test.go
    ├── go.mod
    ├── go.sum
    └── fixtures/
        └── modulename/
            └── terragrunt.hcl
```

### Conventions

- Test files end in `_test.go`
- Test functions start with `Test` (e.g., `TestTerraformBasic`)
- Fixtures contain Terraform/Terragrunt configs used by tests
- The `go.mod` file lives in the `test/` directory

---

## Terraform Options

The `terraform.Options` struct configures how Terratest runs Terraform commands.

| Field | Type | Description | Required |
|---|---|---|---|
| `TerraformDir` | `string` | Path to Terraform configuration directory | Yes |
| `TerraformBinary` | `string` | Path to binary; set to `"terragrunt"` for Terragrunt | No |
| `Vars` | `map[string]interface{}` | Variables passed via `-var` flags | No |
| `VarFiles` | `[]string` | Paths to `.tfvars` files | No |
| `EnvVars` | `map[string]string` | Environment variables for the command | No |
| `BackendConfig` | `map[string]interface{}` | Backend configuration values | No |
| `RetryableTerraformErrors` | `map[string]string` | Error-to-message map for auto-retry | No |
| `MaxRetries` | `int` | Max retries for retryable errors | No |
| `TimeBetweenRetries` | `time.Duration` | Wait between retries | No |
| `Upgrade` | `bool` | Run `init -upgrade` | No |
| `NoColor` | `bool` | Disable color in output | No |
| `Parallelism` | `int` | Terraform parallelism flag value | No |
| `PlanFilePath` | `string` | Path to save/load plan file | No |
| `Lock` | `bool` | Enable/disable state locking | No |
| `LockTimeout` | `string` | State lock timeout duration | No |
| `Targets` | `[]string` | Resource targets for `-target` flags | No |
| `MigrateState` | `bool` | Enable state migration on init | No |
| `Reconfigure` | `bool` | Reconfigure backend on init | No |

### Example

```go
terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
    TerraformDir: "../examples/basic",
    Vars: map[string]interface{}{
        "name":     fmt.Sprintf("test-%s", random.UniqueId()),
        "location": "westeurope",
    },
    EnvVars: map[string]string{
        "ARM_SUBSCRIPTION_ID": os.Getenv("ARM_SUBSCRIPTION_ID"),
    },
})
```

---

## Terragrunt Options

Use `terraform.Options` with `TerraformBinary` set to `"terragrunt"` for Terragrunt tests. Terragrunt-specific functions are in the `terragrunt` package.

| Field | Type | Description |
|---|---|---|
| `TerraformBinary` | `string` | Set to `"terragrunt"` |
| `TerraformDir` | `string` | Path to directory containing `terragrunt.hcl` |

### Terragrunt-Specific Functions

| Function | Description |
|---|---|
| `terragrunt.Apply(t, options)` | Run `terragrunt apply` on a single unit |
| `terragrunt.Destroy(t, options)` | Run `terragrunt destroy` on a single unit |
| `terragrunt.Output(t, options, key)` | Get output value from a unit |
| `terragrunt.ApplyAll(t, options)` | Run `terragrunt run-all apply` on a stack |
| `terragrunt.DestroyAll(t, options)` | Run `terragrunt run-all destroy` on a stack |
| `terragrunt.Plan(t, options)` | Run `terragrunt plan` on a single unit |

---

## Key Packages

| Package | Import Path | Purpose |
|---|---|---|
| `terraform` | `modules/terraform` | Run Terraform/Terragrunt commands, read outputs |
| `terragrunt` | `modules/terraform` | Terragrunt-specific helpers (same package as terraform) |
| `azure` | `modules/azure` | Azure SDK helpers for resource validation |
| `http-helper` | `modules/http-helper` | HTTP request helpers with retries |
| `random` | `modules/random` | Generate unique IDs and random strings |
| `retry` | `modules/retry` | Retry logic with configurable backoff |
| `logger` | `modules/logger` | Structured test logging |
| `shell` | `modules/shell` | Run arbitrary shell commands |
| `files` | `modules/files` | File and directory utilities |
| `test-structure` | `modules/test-structure` | Test stages for iterative development |
| `environment` | `modules/environment` | Environment variable helpers |

---

## Error Handling Pattern

Most Terratest functions come in pairs:

| Pattern | Behavior |
|---|---|
| `Foo(t, ...)` | Calls `FooE(t, ...)`, fails the test on error via `t.Fatal()` |
| `FooE(t, ...)` | Returns `(result, error)`, does **not** fail the test |

Use the `E` variant when you want to handle errors yourself:

```go
// Fails test immediately on error
output := terraform.Output(t, opts, "id")

// Returns error for custom handling
output, err := terraform.OutputE(t, opts, "id")
if err != nil {
    // custom error handling
}
```

---

## Terragrunt 0.85+ Compatibility

> **WARNING: Terratest is not fully compatible with Terragrunt 0.85 and above.**
>
> The `--terragrunt-non-interactive` flag was renamed to `--non-interactive` in the Terragrunt CLI redesign (v0.85+). Terratest hardcodes the old flag name in `modules/terraform/cmd.go` (line 59), causing failures with newer Terragrunt versions.
>
> **Status:** Fix proposed in [PR #1588](https://github.com/gruntwork-io/terratest/pull/1588)
>
> **Workarounds:**
> - Use Terragrunt < 0.85
> - Apply the patch from PR #1588 manually
> - Pin Terratest to a version with the fix once merged
