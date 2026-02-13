# Terratest Azure Helpers Reference

This document covers Azure-specific helper functions for validating infrastructure with Terratest.

## Table of Contents

- [Authentication Setup](#authentication-setup)
- [Build Tag Requirement](#build-tag-requirement)
- [Resource Group Helpers](#resource-group-helpers)
- [Virtual Network Helpers](#virtual-network-helpers)
- [VM Helpers](#vm-helpers)
- [Key Vault Helpers](#key-vault-helpers)
- [AKS Helpers](#aks-helpers)
- [Storage Account Helpers](#storage-account-helpers)
- [Common Azure Testing Patterns](#common-azure-testing-patterns)

---

## Authentication Setup

Azure tests authenticate via environment variables. Set these before running tests:

| Environment Variable | Description | Required |
|---|---|---|
| `ARM_SUBSCRIPTION_ID` | Azure subscription ID | Yes |
| `ARM_CLIENT_ID` | Service principal client ID | For SP auth |
| `ARM_CLIENT_SECRET` | Service principal client secret | For SP auth |
| `ARM_TENANT_ID` | Azure AD tenant ID | For SP auth |

When `ARM_CLIENT_ID` is not set, Terratest falls back to Azure CLI authentication (`az login`).

The `subscriptionID` parameter in helper functions can be empty string `""` — Terratest will read `ARM_SUBSCRIPTION_ID` from the environment.

---

## Build Tag Requirement

All Azure test files must include the `azure` build tag:

```go
//go:build azure
// +build azure

package test
```

Run with:

```shell
go test -v -timeout 30m -tags azure
```

Import the Azure module:

```go
import "github.com/gruntwork-io/terratest/modules/azure"
```

---

## Resource Group Helpers

| Function | Signature | Description |
|---|---|---|
| `ResourceGroupExists` | `(t, rgName, subscriptionID) bool` | Check if resource group exists |
| `GetAResourceGroup` | `(t, rgName, subscriptionID) *resources.Group` | Get resource group details |

### Example

```go
exists := azure.ResourceGroupExists(t, "rg-myapp", "")
assert.True(t, exists)

rg := azure.GetAResourceGroup(t, "rg-myapp", "")
assert.Equal(t, "westeurope", *rg.Location)
assert.Equal(t, "Production", rg.Tags["Environment"])
```

---

## Virtual Network Helpers

| Function | Signature | Description |
|---|---|---|
| `VirtualNetworkExists` | `(t, vnetName, rgName, subscriptionID) bool` | Check if VNet exists |
| `SubnetExists` | `(t, subnetName, vnetName, rgName, subscriptionID) bool` | Check if subnet exists |
| `GetVirtualNetworkE` | `(vnetName, rgName, subscriptionID) (*network.VirtualNetwork, error)` | Get VNet details |
| `GetSubnetE` | `(subnetName, vnetName, rgName, subscriptionID) (*network.Subnet, error)` | Get subnet details |
| `GetVirtualNetworkSubnets` | `(t, vnetName, rgName, subscriptionID) map[string]string` | Get all subnets as name-to-ID map |
| `CheckSubnetContainsNsg` | `(t, subnet) bool` | Check if subnet has an NSG attached |
| `GetSubnetIPRange` | `(t, subnetName, vnetName, rgName, subscriptionID) string` | Get subnet CIDR |

### Example

```go
vnetExists := azure.VirtualNetworkExists(t, "vnet-main", "rg-myapp", "")
assert.True(t, vnetExists)

subnets := azure.GetVirtualNetworkSubnets(t, "vnet-main", "rg-myapp", "")
assert.Contains(t, subnets, "snet-app")
assert.Contains(t, subnets, "snet-data")

ipRange := azure.GetSubnetIPRange(t, "snet-app", "vnet-main", "rg-myapp", "")
assert.Equal(t, "10.0.1.0/24", ipRange)
```

---

## VM Helpers

| Function | Signature | Description |
|---|---|---|
| `VirtualMachineExists` | `(t, vmName, rgName, subscriptionID) bool` | Check if VM exists |
| `GetVirtualMachine` | `(t, vmName, rgName, subscriptionID) *compute.VirtualMachine` | Get VM details |
| `GetVirtualMachineNics` | `(t, vmName, rgName, subscriptionID) []network.Interface` | Get VM network interfaces |
| `GetSizeOfVirtualMachine` | `(t, vmName, rgName, subscriptionID) compute.VirtualMachineSizeTypes` | Get VM size |
| `GetTagsForVirtualMachine` | `(t, vmName, rgName, subscriptionID) map[string]*string` | Get VM tags |
| `GetVirtualMachineImage` | `(t, vmName, rgName, subscriptionID) *compute.ImageReference` | Get VM image reference |
| `GetVirtualMachineManagedDisks` | `(t, vmName, rgName, subscriptionID) []compute.ManagedDiskParameters` | Get managed disks |
| `GetVirtualMachineOSDisk` | `(t, vmName, rgName, subscriptionID) *compute.OSDisk` | Get OS disk details |

### Example

```go
exists := azure.VirtualMachineExists(t, "vm-web01", "rg-myapp", "")
assert.True(t, exists)

vmSize := azure.GetSizeOfVirtualMachine(t, "vm-web01", "rg-myapp", "")
assert.Equal(t, compute.VirtualMachineSizeTypesStandardB2s, vmSize)

tags := azure.GetTagsForVirtualMachine(t, "vm-web01", "rg-myapp", "")
assert.Equal(t, "Production", *tags["Environment"])
```

---

## Key Vault Helpers

| Function | Signature | Description |
|---|---|---|
| `KeyVaultExists` | `(t, kvName, rgName, subscriptionID) bool` | Check if Key Vault exists |
| `GetKeyVault` | `(t, kvName, rgName, subscriptionID) *keyvault.Vault` | Get Key Vault details |
| `GetKeyVaultSecretE` | `(t, kvName, secretName, secretVersion) (*keyvault.SecretBundle, error)` | Get a secret value |
| `KeyVaultKeyExists` | `(t, kvName, keyName) bool` | Check if a key exists |
| `KeyVaultSecretExists` | `(t, kvName, secretName) bool` | Check if a secret exists |
| `KeyVaultCertificateExists` | `(t, kvName, certName) bool` | Check if a certificate exists |
| `GetKeyVaultSoftDeleteSetting` | `(t, kvName, rgName, subscriptionID) bool` | Check soft delete status |

### Example

```go
exists := azure.KeyVaultExists(t, "kv-myapp", "rg-myapp", "")
assert.True(t, exists)

secretExists := azure.KeyVaultSecretExists(t, "kv-myapp", "db-password")
assert.True(t, secretExists)

softDelete := azure.GetKeyVaultSoftDeleteSetting(t, "kv-myapp", "rg-myapp", "")
assert.True(t, softDelete)
```

---

## AKS Helpers

| Function | Signature | Description |
|---|---|---|
| `GetManagedCluster` | `(t, clusterName, rgName, subscriptionID) *containerservice.ManagedCluster` | Get AKS cluster details |
| `GetManagedClusterE` | `(t, clusterName, rgName, subscriptionID) (*containerservice.ManagedCluster, error)` | Get AKS cluster (with error) |
| `ListKubernetesNodePools` | `(t, clusterName, rgName, subscriptionID) []containerservice.ManagedClusterAgentPoolProfile` | List node pools |
| `GetManagedClusterClientConfig` | `(t, clusterName, rgName, subscriptionID) *rest.Config` | Get kubeconfig for the cluster |

### Example

```go
cluster := azure.GetManagedCluster(t, "aks-myapp", "rg-myapp", "")
assert.Equal(t, "1.28.3", *cluster.KubernetesVersion)

nodePools := azure.ListKubernetesNodePools(t, "aks-myapp", "rg-myapp", "")
assert.Equal(t, 1, len(nodePools))
assert.Equal(t, int32(3), *nodePools[0].Count)
```

---

## Storage Account Helpers

| Function | Signature | Description |
|---|---|---|
| `StorageAccountExists` | `(t, accountName, rgName, subscriptionID) bool` | Check if storage account exists |
| `GetStorageAccount` | `(t, accountName, rgName, subscriptionID) *storage.Account` | Get storage account details |
| `StorageBlobContainerExists` | `(t, containerName, accountName, rgName, subscriptionID) bool` | Check if blob container exists |
| `GetStorageAccountKind` | `(t, accountName, rgName, subscriptionID) string` | Get storage account kind |
| `GetStorageAccountSkuTier` | `(t, accountName, rgName, subscriptionID) string` | Get storage SKU tier |
| `GetStorageDNSString` | `(t, accountName, rgName, subscriptionID) string` | Get primary blob endpoint |

### Example

```go
exists := azure.StorageAccountExists(t, "stmyapp", "rg-myapp", "")
assert.True(t, exists)

kind := azure.GetStorageAccountKind(t, "stmyapp", "rg-myapp", "")
assert.Equal(t, "StorageV2", kind)

tier := azure.GetStorageAccountSkuTier(t, "stmyapp", "rg-myapp", "")
assert.Equal(t, "Standard", tier)

containerExists := azure.StorageBlobContainerExists(t, "data", "stmyapp", "rg-myapp", "")
assert.True(t, containerExists)
```

---

## Common Azure Testing Patterns

### Subtests for Multiple Resources

```go
func TestAzureInfrastructure(t *testing.T) {
    t.Parallel()

    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../examples/complete",
    })

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    rgName := terraform.Output(t, terraformOptions, "resource_group_name")

    t.Run("ResourceGroupExists", func(t *testing.T) {
        exists := azure.ResourceGroupExists(t, rgName, "")
        assert.True(t, exists)
    })

    t.Run("VNetExists", func(t *testing.T) {
        vnetName := terraform.Output(t, terraformOptions, "vnet_name")
        exists := azure.VirtualNetworkExists(t, vnetName, rgName, "")
        assert.True(t, exists)
    })

    t.Run("StorageAccountExists", func(t *testing.T) {
        saName := terraform.Output(t, terraformOptions, "storage_account_name")
        exists := azure.StorageAccountExists(t, saName, rgName, "")
        assert.True(t, exists)
    })
}
```

### Existence Check Pattern

```go
// Standard pattern for verifying a resource was created
func assertResourceExists(t *testing.T, terraformOptions *terraform.Options) {
    rgName := terraform.Output(t, terraformOptions, "resource_group_name")
    resourceName := terraform.Output(t, terraformOptions, "resource_name")
    subscriptionID := ""

    exists := azure.ResourceGroupExists(t, rgName, subscriptionID)
    assert.True(t, exists, "Resource group should exist")

    kvExists := azure.KeyVaultExists(t, resourceName, rgName, subscriptionID)
    assert.True(t, kvExists, "Key Vault should exist")
}
```
