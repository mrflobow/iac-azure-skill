# Terratest CLI Reference

This document covers how to execute Terratest tests and related tooling.

## Table of Contents

- [Running Tests](#running-tests)
- [Important Flags](#important-flags)
- [Go Module Setup](#go-module-setup)
- [Build Tags for Azure](#build-tags-for-azure)
- [Test Stage Skipping](#test-stage-skipping)
- [Log Parser](#log-parser)

---

## Running Tests

Terratest tests are standard Go tests run with `go test`.

```shell
# Run all tests in current directory (verbose, 30 min timeout)
go test -v -timeout 30m

# Run a specific test function
go test -v -timeout 30m -run TestTerraformBasic

# Run tests matching a pattern
go test -v -timeout 30m -run "TestTerraform.*"

# Run tests in a specific directory
go test -v -timeout 30m ./test/

# Run tests serially (no parallel execution)
go test -v -timeout 30m -p 1

# Disable test caching
go test -v -timeout 30m -count=1

# Run Azure-tagged tests
go test -v -timeout 30m -tags azure

# Combine common flags
go test -v -timeout 30m -count=1 -p 1 -run TestResourceGroup -tags azure ./test/
```

---

## Important Flags

| Flag | Description | Default | Recommended |
|---|---|---|---|
| `-v` | Verbose output — show `t.Log` output | Off | Always use |
| `-timeout` | Max test duration before forced kill | `10m` | `30m` for infrastructure tests |
| `-run <regex>` | Run only tests matching pattern | All tests | Use for targeted runs |
| `-p <n>` | Max parallel test packages | `GOMAXPROCS` | `1` for infrastructure tests |
| `-count <n>` | Run each test n times; `1` disables caching | Cached | `1` to avoid stale results |
| `-tags <tags>` | Build tags (space-separated) | None | `azure` for Azure tests |
| `-short` | Skip long-running tests (if implemented) | Off | For quick local checks |
| `-parallel <n>` | Max parallel tests within a package | `GOMAXPROCS` | Control with `t.Parallel()` |

---

## Go Module Setup

Initialize a Go module in your test directory:

```shell
# Navigate to the test directory
cd test/

# Initialize module (use your module path)
go mod init github.com/org/module/test

# Add Terratest dependency
go get github.com/gruntwork-io/terratest/modules/terraform
go get github.com/gruntwork-io/terratest/modules/azure

# Download and tidy dependencies
go mod tidy
```

### Minimum `go.mod`

```
module github.com/org/module/test

go 1.21.1

require (
    github.com/gruntwork-io/terratest v0.47.2
    github.com/stretchr/testify v1.9.0
)
```

---

## Build Tags for Azure

Azure-specific tests require a build tag to avoid running in environments without Azure credentials.

Add at the top of your test file:

```go
//go:build azure
// +build azure

package test
```

Run with the tag:

```shell
go test -v -timeout 30m -tags azure
```

Without `-tags azure`, Go skips files with the `azure` build tag.

---

## Test Stage Skipping

Terratest supports test stages that can be individually skipped via environment variables. This enables iterative local development — deploy once, then re-run validation without redeploying.

### Environment Variables

Set `SKIP_<stage_name>=true` to skip a stage:

```shell
# Skip destroy to inspect infrastructure after test
SKIP_teardown=true go test -v -timeout 30m -run TestExample

# Skip deploy to re-validate already-deployed infrastructure
SKIP_deploy=true go test -v -timeout 30m -run TestExample

# Skip both deploy and destroy (validate only)
SKIP_deploy=true SKIP_teardown=true go test -v -timeout 30m -run TestExample
```

### Stage Implementation

```go
func TestExample(t *testing.T) {
    t.Parallel()

    terraformDir := test_structure.CopyTerraformFolderToTemp(t, "../", "examples/basic")

    defer test_structure.RunTestStage(t, "teardown", func() {
        opts := test_structure.LoadTerraformOptions(t, terraformDir)
        terraform.Destroy(t, opts)
    })

    test_structure.RunTestStage(t, "deploy", func() {
        opts := &terraform.Options{
            TerraformDir: terraformDir,
        }
        test_structure.SaveTerraformOptions(t, terraformDir, opts)
        terraform.InitAndApply(t, opts)
    })

    test_structure.RunTestStage(t, "validate", func() {
        opts := test_structure.LoadTerraformOptions(t, terraformDir)
        output := terraform.Output(t, opts, "name")
        assert.NotEmpty(t, output)
    })
}
```

---

## Log Parser

Terratest includes `terratest_log_parser` to convert verbose Go test output into structured results.

### Install

```shell
go install github.com/gruntwork-io/terratest/cmd/terratest_log_parser@latest
```

### Usage

```shell
# Pipe test output to the parser
go test -v -timeout 30m 2>&1 | terratest_log_parser -outputdir /tmp/test-output

# Parse an existing log file
terratest_log_parser -testlog test_output.log -outputdir /tmp/test-output
```

### Output Structure

```
/tmp/test-output/
├── summary.log          # Pass/fail summary for all tests
├── TestFoo.log          # Full log for TestFoo
└── TestBar.log          # Full log for TestBar
```
