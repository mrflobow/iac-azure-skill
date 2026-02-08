# azurerm Provider Functions and Patterns

## Provider Functions

Requires Terraform 1.8+ and azurerm provider v4.0+.

### normalise_resource_id

Normalizes case-sensitive system segments of an Azure Resource ID (e.g., `resourceGroups`, `providers`, `Microsoft.Network`). User-specified segments (resource names) are not changed.

```hcl
# Fix casing: /Subscriptions/... -> /subscriptions/...
output "normalized" {
  value = provider::azurerm::normalise_resource_id(
    "/Subscriptions/12345678-1234-9876-4563-123456789012/ResourceGroups/resGroup1/PROVIDERS/microsoft.apimanagement/service/service1"
  )
  # Result: /subscriptions/12345678-1234-9876-4563-123456789012/resourceGroups/resGroup1/providers/Microsoft.ApiManagement/service/service1
}
```

Use with import blocks to handle IDs with incorrect casing:

```hcl
import {
  id = provider::azurerm::normalise_resource_id("/Subscriptions/12345678-.../resourcegroups/import-example")
  to = azurerm_resource_group.test
}
```

### parse_resource_id

Parses an Azure Resource ID into its component parts. Returns an object with:

| Field | Type | Description |
|-------|------|-------------|
| `subscription_id` | string | Subscription GUID |
| `resource_group_name` | string | Resource group name |
| `resource_provider` | string | e.g., `Microsoft.ApiManagement` |
| `full_resource_type` | string | e.g., `Microsoft.ApiManagement/service/gateways/hostnameConfigurations` |
| `resource_type` | string | Leaf type, e.g., `hostnameConfigurations` |
| `resource_name` | string | Leaf resource name, e.g., `config1` |
| `parent_resources` | map(string) | Parent resource types to names, e.g., `{ "service" = "service1", "gateways" = "gateway1" }` |
| `resource_scope` | string | Resource scope (null for standard resources) |

```hcl
locals {
  parsed = provider::azurerm::parse_resource_id(
    "/subscriptions/12345678-1234-9876-4563-123456789012/resourceGroups/resGroup1/providers/Microsoft.ApiManagement/service/service1/gateways/gateway1/hostnameConfigurations/config1"
  )
}

output "resource_name" {
  value = local.parsed["resource_name"]  # "config1"
}

output "subscription" {
  value = local.parsed["subscription_id"]  # "12345678-1234-9876-4563-123456789012"
}
```

## Common HCL Patterns

### Referencing Other Resources

```hcl
resource "azurerm_resource_group" "example" {
  name     = "example-resources"
  location = "West Europe"
}

resource "azurerm_virtual_network" "example" {
  name                = "example-vnet"
  resource_group_name = azurerm_resource_group.example.name
  location            = azurerm_resource_group.example.location
  address_space       = ["10.0.0.0/16"]
}
```

### Using for_each

```hcl
variable "resource_groups" {
  default = {
    rg1 = { name = "rg-app", location = "West Europe" }
    rg2 = { name = "rg-data", location = "East US" }
  }
}

resource "azurerm_resource_group" "example" {
  for_each = var.resource_groups
  name     = each.value.name
  location = each.value.location
}
```

### Tagging Conventions

```hcl
resource "azurerm_resource_group" "example" {
  name     = "example"
  location = "West Europe"

  tags = {
    environment = "Production"
    team        = "Platform"
    managed_by  = "Terraform"
  }
}
```

### Using depends_on

```hcl
resource "azurerm_role_assignment" "example" {
  scope                = azurerm_resource_group.example.id
  role_definition_name = "Contributor"
  principal_id         = "00000000-0000-0000-0000-000000000000"
}

resource "azurerm_kubernetes_cluster" "example" {
  # ...
  depends_on = [azurerm_role_assignment.example]
}
```

### Multiple Provider Instances (Cross-Subscription)

```hcl
provider "azurerm" {
  features {}
  subscription_id = "sub-1-id"
}

provider "azurerm" {
  alias           = "sub2"
  features {}
  subscription_id = "sub-2-id"
}

resource "azurerm_resource_group" "primary" {
  name     = "rg-primary"
  location = "West Europe"
}

resource "azurerm_resource_group" "secondary" {
  provider = azurerm.sub2
  name     = "rg-secondary"
  location = "East US"
}
```

## Actions

Actions are a Terraform feature for performing operations on Azure resources. They use the `action` keyword and are triggered via `action_trigger` lifecycle blocks.

### azurerm_virtual_machine_power

Start, stop, or restart a Virtual Machine.

```hcl
action "azurerm_virtual_machine_power" "example" {
  config {
    virtual_machine_id = azurerm_linux_virtual_machine.example.id
    power_action       = "restart"  # "restart", "power_on", "power_off"
  }
}
```

### azurerm_cdn_front_door_cache_purge

Purge the cache on a Front Door endpoint.

```hcl
action "azurerm_cdn_front_door_cache_purge" "example" {
  config {
    front_door_endpoint_id = azurerm_cdn_frontdoor_endpoint.example.id
    content_paths          = ["/images/*"]
    # domains = ["example.contoso.com"]  # Optional: specific custom domains
  }
}
```

### azurerm_data_protection_backup_instance_protect

Change protection state of a backup instance. Actions: `stop_protection`, `resume_protection`, `suspend_backups`, `resume_backups`.

```hcl
action "azurerm_data_protection_backup_instance_protect" "example" {
  config {
    backup_instance_id = azurerm_data_protection_backup_instance_postgresql_flexible_server.example.id
    protect_action     = "stop_protection"
  }
}
```

### azurerm_managed_redis_databases_flush

Flush all keys from a Managed Redis database (and optionally linked databases).

```hcl
action "azurerm_managed_redis_databases_flush" "example" {
  config {
    managed_redis_database_id = azurerm_managed_redis.example.default_database[0].id
    # linked_database_ids = [...]  # Optional
  }
}
```

### azurerm_mssql_execute_job

Execute an Elastic Job.

```hcl
action "azurerm_mssql_execute_job" "example" {
  config {
    job_id              = azurerm_mssql_job.example.id
    wait_for_completion = false  # Optional, defaults to false
  }
}
```

### Triggering Actions

Actions are triggered via `action_trigger` in a `terraform_data` lifecycle block:

```hcl
resource "terraform_data" "trigger" {
  input = azurerm_network_interface.example.private_ip_address

  lifecycle {
    action_trigger {
      events  = [after_update]    # after_create, after_update
      actions = [action.azurerm_virtual_machine_power.example]
    }
  }
}
```

## Ephemeral Resources

Ephemeral resources (Terraform 1.10+) access sensitive data without persisting it to state.

### azurerm_key_vault_secret

```hcl
ephemeral "azurerm_key_vault_secret" "example" {
  name         = "secret-sauce"
  key_vault_id = data.azurerm_key_vault.example.id
  # version    = "optional-version"  # Defaults to current version
}
# Attributes: value, expiration_date, not_before_date
```

### azurerm_key_vault_certificate

```hcl
ephemeral "azurerm_key_vault_certificate" "example" {
  name         = "my-cert"
  key_vault_id = data.azurerm_key_vault.example.id
  # version    = "optional-version"  # Defaults to current version
}
# Attributes: hex, pem, key, expiration_date, not_before_date
```
