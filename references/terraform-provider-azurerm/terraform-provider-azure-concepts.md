# azurerm Provider Concepts

## Provider Configuration

The `provider "azurerm"` block configures how Terraform communicates with Azure.

### Required Arguments

| Argument | Description | Env Variable |
|----------|-------------|-------------|
| `features {}` | Feature behavior toggles (required, can be empty) | - |
| `subscription_id` | Azure Subscription ID | `ARM_SUBSCRIPTION_ID` |

### Optional Arguments

| Argument | Description | Env Variable | Default |
|----------|-------------|-------------|---------|
| `client_id` | Service Principal / App Client ID | `ARM_CLIENT_ID` | - |
| `tenant_id` | Azure AD Tenant ID | `ARM_TENANT_ID` | - |
| `environment` | Cloud environment: `public`, `usgovernment`, `german`, `china` | `ARM_ENVIRONMENT` | `public` |
| `storage_use_azuread` | Use Azure AD for Storage Blob & Queue APIs | `ARM_STORAGE_USE_AZUREAD` | `false` |
| `resource_provider_registrations` | Auto-register resource providers: `core`, `extended`, `all`, `none` | `ARM_RESOURCE_PROVIDER_REGISTRATIONS` | `core` |
| `resource_providers_to_register` | List of specific Azure Resource Providers to register | - | `[]` |
| `metadata_host` | Custom Azure Metadata Service hostname | `ARM_METADATA_HOSTNAME` | - |
| `partner_id` | GUID for Microsoft partner usage attribution | `ARM_PARTNER_ID` | - |
| `auxiliary_tenant_ids` | Up to 3 additional tenant IDs for cross-tenant scenarios | `ARM_AUXILIARY_TENANT_IDS` | `[]` |

## Authentication Methods

| Method | Provider Args | Best For |
|--------|--------------|----------|
| Azure CLI | `use_cli = true` (default) | Local development |
| Service Principal + Client Secret | `client_id`, `client_secret` | CI/CD pipelines |
| Service Principal + Client Certificate | `client_id`, `client_certificate_path` | CI/CD with cert-based auth |
| Service Principal + OIDC | `client_id`, `use_oidc = true` | GitHub Actions, Azure DevOps |
| Managed Identity | `use_msi = true` | VMs, containers running in Azure |
| AKS Workload Identity | `use_aks_workload_identity = true` | Pods in AKS clusters |

- Use Azure CLI for local development; use Service Principal or Managed Identity for non-interactive (CI/CD) execution
- The principal running Terraform needs permissions to register Azure Resource Providers, or set `resource_provider_registrations = "none"`

## Features Block

The `features {}` block is **required** (can be empty for defaults). It controls provider behavior for specific resource types.

### api_management

| Setting | Description | Default |
|---------|-------------|---------|
| `purge_soft_delete_on_destroy` | Permanently purge on destroy | `true` |
| `recover_soft_deleted` | Recover soft-deleted instance on create | `true` |

### app_configuration

| Setting | Description | Default |
|---------|-------------|---------|
| `purge_soft_delete_on_destroy` | Permanently purge on destroy | `true` |
| `recover_soft_deleted` | Recover soft-deleted instance on create | `true` |

### application_insights

| Setting | Description | Default |
|---------|-------------|---------|
| `disable_generated_rule` | Disable auto-generated Alert Rule on create | `false` |

### cognitive_account

| Setting | Description | Default |
|---------|-------------|---------|
| `purge_soft_delete_on_destroy` | Permanently purge on destroy | `true` |

### databricks_workspace

| Setting | Description | Default |
|---------|-------------|---------|
| `force_delete` | Force delete managed resource group with Unity Catalog data | `false` |

### key_vault

| Setting | Description | Default |
|---------|-------------|---------|
| `purge_soft_delete_on_destroy` | Purge Key Vault on destroy | `true` |
| `purge_soft_deleted_certificates_on_destroy` | Purge certificates on destroy | `true` |
| `purge_soft_deleted_keys_on_destroy` | Purge keys on destroy | `true` |
| `purge_soft_deleted_secrets_on_destroy` | Purge secrets on destroy | `true` |
| `purge_soft_deleted_hardware_security_modules_on_destroy` | Purge HSMs on destroy | `true` |
| `purge_soft_deleted_hardware_security_module_keys_on_destroy` | Purge HSM keys on destroy | `true` |
| `recover_soft_deleted_key_vaults` | Recover soft-deleted Key Vault | `true` |
| `recover_soft_deleted_certificates` | Recover soft-deleted certificates | `true` |
| `recover_soft_deleted_keys` | Recover soft-deleted keys | `true` |
| `recover_soft_deleted_secrets` | Recover soft-deleted secrets | `true` |
| `recover_soft_deleted_hardware_security_module_keys` | Recover soft-deleted HSM keys | `true` |

### log_analytics_workspace

| Setting | Description | Default |
|---------|-------------|---------|
| `permanently_delete_on_destroy` | Permanently delete on destroy | `false` |

### machine_learning

| Setting | Description | Default |
|---------|-------------|---------|
| `purge_soft_deleted_workspace_on_destroy` | Purge workspace on destroy | `false` |

### managed_disk

| Setting | Description | Default |
|---------|-------------|---------|
| `expand_without_downtime` | Expand disk without restarting VM | `true` |

### netapp

| Setting | Description | Default |
|---------|-------------|---------|
| `delete_backups_on_backup_vault_destroy` | Delete backups when vault is destroyed | `false` |
| `prevent_volume_destruction` | Protect volumes against deletion | `true` |

### postgresql_flexible_server

| Setting | Description | Default |
|---------|-------------|---------|
| `restart_server_on_configuration_value_change` | Restart after static parameter change | `true` |

### recovery_service

| Setting | Description | Default |
|---------|-------------|---------|
| `vm_backup_stop_protection_and_retain_data_on_destroy` | Stop protection and retain data on destroy | `false` |
| `vm_backup_suspend_protection_and_retain_data_on_destroy` | Suspend protection and retain data on destroy | `false` |
| `purge_protected_items_from_vault_on_destroy` | Purge all protected items when vault is destroyed | `false` |

### recovery_services_vault

| Setting | Description | Default |
|---------|-------------|---------|
| `recover_soft_deleted_backup_protected_vm` | Recover soft-deleted protected VM | `false` |

### resource_group

| Setting | Description | Default |
|---------|-------------|---------|
| `prevent_deletion_if_contains_resources` | Error if resource group contains resources on delete | `true` |

### storage

| Setting | Description | Default |
|---------|-------------|---------|
| `data_plane_available` | Use data plane APIs for storage accounts | `true` |

### subscription

| Setting | Description | Default |
|---------|-------------|---------|
| `prevent_cancellation_on_destroy` | Prevent subscription cancellation on destroy | `false` |

### template_deployment

| Setting | Description | Default |
|---------|-------------|---------|
| `delete_nested_items_during_deletion` | Delete ARM-provisioned resources when template deployment is deleted | `true` |

### virtual_machine

| Setting | Description | Default |
|---------|-------------|---------|
| `delete_os_disk_on_deletion` | Delete OS disk when VM is destroyed | `true` |
| `graceful_shutdown` | Request graceful shutdown on destroy (deprecated) | `false` |
| `skip_shutdown_and_force_delete` | Skip shutdown and force delete VM | `false` |
| `detach_implicit_data_disk_on_deletion` | Detach implicit data disk instead of deleting | `false` |

### virtual_machine_scale_set

| Setting | Description | Default |
|---------|-------------|---------|
| `force_delete` | Force delete VMSS | `false` |
| `roll_instances_when_required` | Auto-roll instances on Sku/Image update | `true` |
| `reimage_on_manual_upgrade` | Auto-reimage on manual upgrade | `true` |
| `scale_to_zero_before_deletion` | Scale to 0 instances before deleting | `true` |

## Resource Provider Registrations

The provider auto-registers Azure Resource Providers before plan/apply. Control this with `resource_provider_registrations`:

| Value | Description |
|-------|-------------|
| `core` | Essential services: Compute, Networking, Storage |
| `extended` | Most common supported resources (default-like coverage) |
| `all` | Every resource provider the azurerm provider supports |
| `none` | No auto-registration (use when principal lacks registration permissions) |

- Combine with `resource_providers_to_register` to add specific providers on top of a set
- The principal running Terraform needs `Microsoft.Resources/subscriptions/providers/register/action` permission to register providers

## Azure Resource IDs

Format: `/subscriptions/{sub-id}/resourceGroups/{rg-name}/providers/{provider}/{type}/{name}`

- Example: `/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/mygroup/providers/Microsoft.Network/virtualNetworks/myvnet`
- Used for: `terraform import`, cross-resource references via `id` attribute, provider functions
- Nested resources extend the path: `.../providers/Microsoft.ApiManagement/service/svc1/gateways/gw1`
- Case-sensitive in system segments (e.g., `resourceGroups` not `resourcegroups`); use `provider::azurerm::normalise_resource_id()` to fix

## Resource Subcategories

The provider covers 116 Azure service categories:

AAD B2C, API Management, Active Directory Domain Services, Advisor, Analysis Services, App Configuration, App Service (Web Apps), Application Insights, Arc Resource Bridge, ArcKubernetes, Attestation, Authorization, Automanage, Automation, Azure Managed Lustre File System, Azure Stack HCI, Azure VMware Solution, Base, Batch, Billing, Blueprints, Bot, CDN, Chaos Studio, Cognitive Services, Communication, Compute, Confidential Ledger, Connections, Consumption, Container, Container Apps, CosmosDB (DocumentDB), Cost Management, Custom Providers, DNS, Dashboard, Data Explorer, Data Factory, Data Share, DataProtection, Database, Database Migration, Databox Edge, Databricks, Datadog, Desktop Virtualization, Dev Center, Dev Test, Digital Twins, Dynatrace, Elastic, Elastic SAN, Extended Location, Fabric, Fluid Relay, Graph Services, HDInsight, Hardware Security Module, Healthcare, Hybrid Compute, IoT Central, IoT Hub, Key Vault, Lighthouse, Load Balancer, Load Test, Log Analytics, Logic App, Machine Learning, Maintenance, Managed Applications, Managed Redis, Management, Maps, Messaging, Mongo Cluster, Monitor, NGINX, NetApp, Network, Network Function, New Relic, Oracle, Orbital, Palo Alto, Policy, Portal, PowerBI, Private DNS, Private DNS Resolver, Purview, Qumulo, Recovery Services, Red Hat OpenShift, Redis, Redis Enterprise, Search, Security Center, Sentinel, Service Fabric, Service Fabric Managed Clusters, Service Networking, Spring Cloud, Storage, Storage Mover, Stream Analytics, Synapse, System Center Virtual Machine Manager, Template, Trusted Signing, Video Indexer, Voice Services, Web PubSub, Workloads

## Version 4.0 Key Changes

- **Mandatory `subscription_id`** - must be set in provider block or via `ARM_SUBSCRIPTION_ID`
- **Improved Resource Provider Registration** - new `resource_provider_registrations` property with `core`/`extended`/`all`/`none` presets
- **Provider Functions** - `normalise_resource_id()` and `parse_resource_id()` (requires Terraform 1.8+)
- **Enhanced Subnet Configuration** - `subnet` block in `azurerm_virtual_network` supports delegations, route tables, and more options inline

## Continuous Validation

Use `check` blocks in Terraform Cloud for health monitoring between applies:

```hcl
data "azurerm_virtual_machine" "example" {
  name                = azurerm_linux_virtual_machine.example.name
  resource_group_name = azurerm_resource_group.example.name
}

check "check_vm_state" {
  assert {
    condition     = data.azurerm_virtual_machine.example.power_state == "running"
    error_message = "VM ${data.azurerm_virtual_machine.example.id} is ${data.azurerm_virtual_machine.example.power_state}, expected running"
  }
}
```

- Checks run between applied runs in Terraform Cloud
- Use data sources to read current state and assert expected conditions
- Common checks: VM power state, certificate expiry, App Service usage limits
