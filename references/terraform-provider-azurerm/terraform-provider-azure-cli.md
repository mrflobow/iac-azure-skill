# Terraform CLI for azurerm Provider

## Provider Setup

- Add `required_providers` block and run `terraform init` to download the provider:

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
  }
}

provider "azurerm" {
  features {}
  subscription_id = "00000000-0000-0000-0000-000000000000"
}
```

- `subscription_id` is **required** in v4.0+ (set in provider block or via `ARM_SUBSCRIPTION_ID`)
- Pin the version with `= 4.1.0` (exact) or `~> 4.0` (minor version flexibility)
- Run `terraform init` to download the provider; run `terraform init -upgrade` to update to a newer version

## Authentication Setup

### Azure CLI (recommended for local development)

```shell
az login
az account set --subscription="SUBSCRIPTION_ID"
export ARM_SUBSCRIPTION_ID="00000000-0000-0000-0000-000000000000"
```

- The provider uses Azure CLI credentials by default (`use_cli = true`)
- To use a specific tenant, set `tenant_id` in the provider block

### Service Principal + Client Secret (recommended for CI/CD)

```shell
export ARM_CLIENT_ID="00000000-0000-0000-0000-000000000000"
export ARM_CLIENT_SECRET="your-client-secret"
export ARM_TENANT_ID="00000000-0000-0000-0000-000000000000"
export ARM_SUBSCRIPTION_ID="00000000-0000-0000-0000-000000000000"
```

- Create a Service Principal: `az ad sp create-for-rbac --role="Contributor" --scopes="/subscriptions/SUBSCRIPTION_ID"`
- `appId` = `ARM_CLIENT_ID`, `password` = `ARM_CLIENT_SECRET`, `tenant` = `ARM_TENANT_ID`

### Service Principal + Client Certificate

```shell
export ARM_CLIENT_ID="00000000-0000-0000-0000-000000000000"
export ARM_CLIENT_CERTIFICATE_PATH="/path/to/certificate.pfx"
export ARM_CLIENT_CERTIFICATE_PASSWORD="Pa55w0rd123"
export ARM_TENANT_ID="00000000-0000-0000-0000-000000000000"
export ARM_SUBSCRIPTION_ID="00000000-0000-0000-0000-000000000000"
```

- Alternatively, pass the base64-encoded certificate via `ARM_CLIENT_CERTIFICATE` instead of the path

### Service Principal + OIDC (recommended for GitHub Actions / Azure DevOps)

```shell
export ARM_CLIENT_ID="00000000-0000-0000-0000-000000000000"
export ARM_TENANT_ID="00000000-0000-0000-0000-000000000000"
export ARM_SUBSCRIPTION_ID="00000000-0000-0000-0000-000000000000"
export ARM_USE_OIDC=true
```

**GitHub Actions** - add permissions and the provider auto-detects `ACTIONS_ID_TOKEN_REQUEST_URL`/`ACTIONS_ID_TOKEN_REQUEST_TOKEN`:

```yaml
permissions:
  id-token: write
  contents: read
```

**Azure DevOps Pipelines** - use the `AzureCLI@2` task:

```yaml
- task: AzureCLI@2
  inputs:
    azureSubscription: $(SERVICE_CONNECTION_ID)
    scriptType: bash
    scriptLocation: "inlineScript"
    inlineScript: |
      terraform plan
  env:
    ARM_USE_OIDC: true
    SYSTEM_ACCESSTOKEN: $(System.AccessToken)
    SYSTEM_OIDCREQUESTURI: $(System.OidcRequestUri)
    ARM_ADO_PIPELINE_SERVICE_CONNECTION_ID: $(SERVICE_CONNECTION_ID)
```

### Managed Identity (for VMs / containers running in Azure)

```shell
export ARM_USE_MSI=true
export ARM_SUBSCRIPTION_ID="00000000-0000-0000-0000-000000000000"
export ARM_TENANT_ID="00000000-0000-0000-0000-000000000000"
# For user-assigned identity:
export ARM_CLIENT_ID="00000000-0000-0000-0000-000000000000"
# For custom MSI endpoint (e.g., Azure Container Apps):
export ARM_MSI_ENDPOINT="$MSI_ENDPOINT"
```

### AKS Workload Identity (for pods in AKS clusters)

```shell
export ARM_USE_AKS_WORKLOAD_IDENTITY=true
export ARM_USE_CLI=false
export ARM_SUBSCRIPTION_ID="00000000-0000-0000-0000-000000000000"
# ARM_CLIENT_ID only needed if service account is not annotated with azure.workload.identity/client-id
```

- `client_id`, `tenant_id`, and `oidc_token_file_path` are auto-detected from the pod environment

## Core Workflow

```shell
terraform plan              # Preview changes
terraform apply             # Apply changes (prompts for confirmation)
terraform apply -auto-approve  # Apply without confirmation prompt
terraform destroy           # Destroy all managed resources
terraform validate          # Validate configuration syntax (does not require subscription_id)
```

## Import

### CLI Import

```shell
terraform import azurerm_resource_group.example /subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/example
```

### Config Block Import (Terraform 1.5+)

```hcl
import {
  to = azurerm_resource_group.example
  id = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/example"
}
```

- The `id` is the Azure Resource ID (found in Azure Portal or via `az resource show`)
- Use `provider::azurerm::normalise_resource_id()` to fix casing issues in import IDs (Terraform 1.8+)

## State Management

```shell
terraform state list                          # List all resources in state
terraform state show azurerm_resource_group.example  # Show details of a specific resource
terraform state rm azurerm_resource_group.example    # Remove a resource from state (does not destroy it)
```

## Resource Migration

To migrate from a deprecated resource to its replacement:

1. Update the Terraform configuration to use the new resource type
2. Get the Azure Resource ID: `echo azurerm_old_resource.example.id | terraform console`
3. Remove the old resource from state: `terraform state rm azurerm_old_resource.example`
4. Import into the new resource: `terraform import azurerm_new_resource.example <azure_resource_id>`
5. Verify: `terraform plan` should show no changes

## Provider Functions

Requires Terraform 1.8+ and azurerm provider v4.0+.

```hcl
# Normalize casing of system segments in an Azure Resource ID
provider::azurerm::normalise_resource_id("/Subscriptions/.../ResourceGroups/...")
# Returns: /subscriptions/.../resourceGroups/...

# Parse an Azure Resource ID into its component parts
provider::azurerm::parse_resource_id("/subscriptions/.../resourceGroups/.../providers/...")
# Returns object with: subscription_id, resource_group_name, resource_provider,
#   full_resource_type, resource_type, resource_name, parent_resources, resource_scope
```

## Debugging

```shell
TF_LOG=DEBUG terraform plan    # Enable debug logging
TF_LOG=TRACE terraform apply   # Maximum verbosity
TF_LOG_PATH=./terraform.log terraform plan  # Write logs to file
```
