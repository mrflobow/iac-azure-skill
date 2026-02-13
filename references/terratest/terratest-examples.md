# Terratest Examples Reference

This document provides complete testing patterns for Terraform, Terragrunt, and Azure infrastructure.

## Table of Contents

- [Basic Terraform Test](#basic-terraform-test)
- [Terraform with Outputs Validation](#terraform-with-outputs-validation)
- [Idempotency Testing](#idempotency-testing)
- [Azure Resource Group Test](#azure-resource-group-test)
- [Azure Resource Validation](#azure-resource-validation)
- [Terragrunt Unit Test](#terragrunt-unit-test)
- [Terragrunt Stack Test](#terragrunt-stack-test)
- [HTTP Validation with Retries](#http-validation-with-retries)
- [Test Stages for Local Iteration](#test-stages-for-local-iteration)
- [Parallel Test with Temp Folder](#parallel-test-with-temp-folder)
- [Common Patterns Summary](#common-patterns-summary)

---

## Basic Terraform Test

The simplest Terratest pattern: apply, validate a basic property, then destroy.

```go
package test

import (
    "testing"

    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestTerraformBasic(t *testing.T) {
    t.Parallel()

    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../examples/basic",
    })

    defer terraform.Destroy(t, terraformOptions)

    terraform.InitAndApply(t, terraformOptions)

    output := terraform.Output(t, terraformOptions, "resource_name")
    assert.Contains(t, output, "expected-prefix")
}
```

---

## Terraform with Outputs Validation

Validate multiple outputs and use variables.

```go
package test

import (
    "fmt"
    "testing"

    "github.com/gruntwork-io/terratest/modules/random"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestTerraformOutputs(t *testing.T) {
    t.Parallel()

    uniqueId := random.UniqueId()
    name := fmt.Sprintf("test-%s", uniqueId)

    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../examples/complete",
        Vars: map[string]interface{}{
            "name":     name,
            "location": "westeurope",
            "tags": map[string]interface{}{
                "Environment": "test",
            },
        },
    })

    defer terraform.Destroy(t, terraformOptions)

    terraform.InitAndApply(t, terraformOptions)

    resourceGroupName := terraform.Output(t, terraformOptions, "resource_group_name")
    resourceId := terraform.Output(t, terraformOptions, "resource_id")
    location := terraform.Output(t, terraformOptions, "location")

    assert.Equal(t, name, resourceGroupName)
    assert.NotEmpty(t, resourceId)
    assert.Equal(t, "westeurope", location)
}
```

---

## Idempotency Testing

Verify that running `apply` twice produces no changes, confirming the configuration is idempotent.

```go
package test

import (
    "testing"

    "github.com/gruntwork-io/terratest/modules/terraform"
)

func TestTerraformIdempotent(t *testing.T) {
    t.Parallel()

    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../examples/basic",
    })

    defer terraform.Destroy(t, terraformOptions)

    // ApplyAndIdempotent runs apply twice and fails if the second apply shows changes
    terraform.ApplyAndIdempotent(t, terraformOptions)
}
```

---

## Azure Resource Group Test

Test Azure resource creation using Azure SDK helpers. Requires `azure` build tag.

```go
//go:build azure
// +build azure

package test

import (
    "fmt"
    "testing"

    "github.com/gruntwork-io/terratest/modules/azure"
    "github.com/gruntwork-io/terratest/modules/random"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestAzureResourceGroup(t *testing.T) {
    t.Parallel()

    uniqueId := random.UniqueId()
    name := fmt.Sprintf("rg-test-%s", uniqueId)
    subscriptionID := ""  // uses ARM_SUBSCRIPTION_ID env var

    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../examples/resource-group",
        Vars: map[string]interface{}{
            "name":     name,
            "location": "westeurope",
        },
    })

    defer terraform.Destroy(t, terraformOptions)

    terraform.InitAndApply(t, terraformOptions)

    resourceGroupName := terraform.Output(t, terraformOptions, "resource_group_name")

    // Validate using Azure SDK
    exists := azure.ResourceGroupExists(t, resourceGroupName, subscriptionID)
    assert.True(t, exists)
}
```

---

## Azure Resource Validation

Validate Azure resources (VNet, Subnet) using Azure helper functions.

```go
//go:build azure
// +build azure

package test

import (
    "fmt"
    "testing"

    "github.com/gruntwork-io/terratest/modules/azure"
    "github.com/gruntwork-io/terratest/modules/random"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestAzureVirtualNetwork(t *testing.T) {
    t.Parallel()

    uniqueId := random.UniqueId()
    name := fmt.Sprintf("test-%s", uniqueId)
    subscriptionID := ""

    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../examples/vnet",
        Vars: map[string]interface{}{
            "prefix":   name,
            "location": "westeurope",
        },
    })

    defer terraform.Destroy(t, terraformOptions)

    terraform.InitAndApply(t, terraformOptions)

    rgName := terraform.Output(t, terraformOptions, "resource_group_name")
    vnetName := terraform.Output(t, terraformOptions, "vnet_name")

    // Validate VNet exists
    vnetExists := azure.VirtualNetworkExists(t, vnetName, rgName, subscriptionID)
    assert.True(t, vnetExists)

    // Validate subnet exists within the VNet
    subnetExists := azure.SubnetExists(t, "default", vnetName, rgName, subscriptionID)
    assert.True(t, subnetExists)

    // Check VNet address space
    vnet, err := azure.GetVirtualNetworkE(vnetName, rgName, subscriptionID)
    assert.NoError(t, err)
    assert.Contains(t, *vnet.VirtualNetworkPropertiesFormat.AddressSpace.AddressPrefixes, "10.0.0.0/16")
}
```

---

## Terragrunt Unit Test

Test a single Terragrunt unit (module). Uses `TerraformBinary: "terragrunt"`.

```go
package test

import (
    "fmt"
    "testing"

    "github.com/gruntwork-io/terratest/modules/random"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestTerragruntUnit(t *testing.T) {
    t.Parallel()

    uniqueId := random.UniqueId()

    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir:    "../test/fixtures/modulename",
        TerraformBinary: "terragrunt",
        Vars: map[string]interface{}{
            "name": fmt.Sprintf("test-%s", uniqueId),
        },
    })

    defer terraform.Destroy(t, terraformOptions)

    terraform.InitAndApply(t, terraformOptions)

    output := terraform.Output(t, terraformOptions, "name")
    assert.Contains(t, output, "test-")
}
```

---

## Terragrunt Stack Test

Test an entire Terragrunt stack (multiple modules) using `run-all` commands.

```go
package test

import (
    "testing"

    "github.com/gruntwork-io/terratest/modules/terraform"
)

func TestTerragruntStack(t *testing.T) {
    t.Parallel()

    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir:    "../test/fixtures/stack",
        TerraformBinary: "terragrunt",
    })

    defer terraform.TgDestroyAll(t, terraformOptions)

    terraform.TgApplyAll(t, terraformOptions)

    // Validate outputs from individual modules within the stack
    // Navigate to specific module directory to read outputs
    moduleOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir:    "../test/fixtures/stack/networking",
        TerraformBinary: "terragrunt",
    })

    vnetName := terraform.Output(t, moduleOptions, "vnet_name")
    assert.NotEmpty(t, vnetName)
}
```

---

## HTTP Validation with Retries

Validate that a deployed web service responds correctly, with retries for startup time.

```go
package test

import (
    "fmt"
    "testing"
    "time"

    http_helper "github.com/gruntwork-io/terratest/modules/http-helper"
    "github.com/gruntwork-io/terratest/modules/terraform"
)

func TestHTTPEndpoint(t *testing.T) {
    t.Parallel()

    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../examples/webapp",
    })

    defer terraform.Destroy(t, terraformOptions)

    terraform.InitAndApply(t, terraformOptions)

    url := terraform.Output(t, terraformOptions, "url")
    endpoint := fmt.Sprintf("https://%s", url)

    // Retry up to 30 times with 10 second intervals (5 min total)
    http_helper.HttpGetWithRetry(t, endpoint, nil, 200, "Hello, World!", 30, 10*time.Second)
}
```

---

## Test Stages for Local Iteration

Use test stages to skip deploy/destroy during iterative development.

```go
package test

import (
    "testing"

    "github.com/gruntwork-io/terratest/modules/terraform"
    test_structure "github.com/gruntwork-io/terratest/modules/test-structure"
    "github.com/stretchr/testify/assert"
)

func TestWithStages(t *testing.T) {
    t.Parallel()

    workingDir := test_structure.CopyTerraformFolderToTemp(t, "../", "examples/basic")

    defer test_structure.RunTestStage(t, "teardown", func() {
        terraformOptions := test_structure.LoadTerraformOptions(t, workingDir)
        terraform.Destroy(t, terraformOptions)
    })

    test_structure.RunTestStage(t, "deploy", func() {
        terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
            TerraformDir: workingDir,
        })
        test_structure.SaveTerraformOptions(t, workingDir, terraformOptions)
        terraform.InitAndApply(t, terraformOptions)
    })

    test_structure.RunTestStage(t, "validate", func() {
        terraformOptions := test_structure.LoadTerraformOptions(t, workingDir)
        output := terraform.Output(t, terraformOptions, "name")
        assert.NotEmpty(t, output)
    })
}
```

Skip stages with environment variables:

```shell
# Deploy once
go test -v -timeout 30m -run TestWithStages

# Iterate on validation (skip deploy and teardown)
SKIP_deploy=true SKIP_teardown=true go test -v -timeout 30m -run TestWithStages

# Clean up when done
SKIP_deploy=true go test -v -timeout 30m -run TestWithStages
```

---

## Parallel Test with Temp Folder

Copy the Terraform folder to a temp directory to avoid state conflicts when running multiple tests in parallel.

```go
package test

import (
    "fmt"
    "testing"

    "github.com/gruntwork-io/terratest/modules/random"
    "github.com/gruntwork-io/terratest/modules/terraform"
    test_structure "github.com/gruntwork-io/terratest/modules/test-structure"
    "github.com/stretchr/testify/assert"
)

func TestParallelA(t *testing.T) {
    t.Parallel()
    runParallelTest(t, "westeurope")
}

func TestParallelB(t *testing.T) {
    t.Parallel()
    runParallelTest(t, "northeurope")
}

func runParallelTest(t *testing.T, location string) {
    // Copy to temp dir so each test has its own state
    tempDir := test_structure.CopyTerraformFolderToTemp(t, "../", "examples/basic")

    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: tempDir,
        Vars: map[string]interface{}{
            "name":     fmt.Sprintf("test-%s", random.UniqueId()),
            "location": location,
        },
    })

    defer terraform.Destroy(t, terraformOptions)

    terraform.InitAndApply(t, terraformOptions)

    output := terraform.Output(t, terraformOptions, "name")
    assert.Contains(t, output, "test-")
}
```

---

## Common Patterns Summary

| Pattern | When to Use | Key Function |
|---|---|---|
| Basic apply/destroy | Simple resource validation | `terraform.InitAndApply` + `terraform.Destroy` |
| Output validation | Verify resource properties | `terraform.Output` |
| Idempotency | Ensure no drift on re-apply | `terraform.ApplyAndIdempotent` |
| Azure helpers | Validate Azure resources via SDK | `azure.ResourceGroupExists`, etc. |
| Terragrunt unit | Test single Terragrunt module | `TerraformBinary: "terragrunt"` |
| Terragrunt stack | Test multiple modules together | `terraform.TgApplyAll` / `terraform.TgDestroyAll` |
| HTTP validation | Test deployed web endpoints | `http_helper.HttpGetWithRetry` |
| Test stages | Iterative local development | `test_structure.RunTestStage` |
| Parallel + temp folder | Avoid state conflicts | `test_structure.CopyTerraformFolderToTemp` |
