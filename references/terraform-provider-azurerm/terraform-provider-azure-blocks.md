# azurerm HCL Block Structures

## Resource Block Structure

Standard pattern for creating Azure resources:

```hcl
resource "azurerm_<type>" "example" {
  name                = "example-name"
  resource_group_name = azurerm_resource_group.example.name
  location            = azurerm_resource_group.example.location

  # ... required and optional arguments

  tags = {
    environment = "Production"
  }
}
```

- `name` - resource name in Azure (usually ForceNew: changing it recreates the resource)
- `resource_group_name` - parent resource group (usually ForceNew)
- `location` - Azure region (usually ForceNew)
- `tags` - key-value metadata map (conventionally last)

Example:

```hcl
resource "azurerm_resource_group" "example" {
  name     = "example-resources"
  location = "West Europe"

  tags = {
    environment = "Production"
  }
}
```

## Data Source Block Structure

Read-only lookup of existing Azure resources:

```hcl
data "azurerm_<type>" "example" {
  name                = "existing-resource-name"
  resource_group_name = "existing-rg-name"
}
```

- Data sources do not create or modify resources
- Use to reference existing infrastructure not managed by the current Terraform config
- Access attributes via `data.azurerm_<type>.example.<attribute>`

Example:

```hcl
data "azurerm_resource_group" "example" {
  name = "existing"
}

output "rg_location" {
  value = data.azurerm_resource_group.example.location
}
```

## Ephemeral Resource Block Structure

Access sensitive data without persisting to state (Terraform 1.10+):

```hcl
ephemeral "azurerm_key_vault_secret" "example" {
  name         = "secret-sauce"
  key_vault_id = data.azurerm_key_vault.example.id
}
```

- Values are not stored in state or plan files
- Available: `azurerm_key_vault_secret`, `azurerm_key_vault_certificate`

## Action Block Structure

Perform operations on existing resources:

```hcl
action "azurerm_virtual_machine_power" "example" {
  config {
    virtual_machine_id = azurerm_linux_virtual_machine.example.id
    power_action       = "restart"
  }
}
```

- Actions are triggered via `action_trigger` in a `terraform_data` lifecycle block:

```hcl
resource "terraform_data" "trigger" {
  input = some_value_that_changes

  lifecycle {
    action_trigger {
      events  = [after_update]
      actions = [action.azurerm_virtual_machine_power.example]
    }
  }
}
```

## Common Resource Arguments

Most azurerm resources share these arguments:

| Argument | Description | Notes |
|----------|-------------|-------|
| `name` | Resource name in Azure | Usually ForceNew |
| `resource_group_name` | Parent resource group name | Usually ForceNew |
| `location` | Azure region (e.g., `West Europe`, `East US`) | Usually ForceNew |
| `tags` | `map(string)` of key-value metadata | Conventionally last in block |

- Not all resources have all of these (e.g., child resources inherit location from parent)
- `ForceNew` means changing the value destroys and recreates the resource

## Nested Block Patterns

### Single-Instance Blocks

Blocks that appear at most once:

```hcl
resource "azurerm_linux_virtual_machine" "example" {
  # ...

  identity {
    type = "SystemAssigned"
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }
}
```

### Multi-Instance Blocks

Blocks that can appear multiple times:

```hcl
resource "azurerm_virtual_network" "example" {
  name                = "example-vnet"
  resource_group_name = azurerm_resource_group.example.name
  location            = azurerm_resource_group.example.location
  address_space       = ["10.0.0.0/16"]

  subnet {
    name             = "subnet1"
    address_prefixes = ["10.0.1.0/24"]
  }

  subnet {
    name             = "subnet2"
    address_prefixes = ["10.0.2.0/24"]
    security_group   = azurerm_network_security_group.example.id
  }
}
```

### Deeply Nested Blocks

Some resources have multiple levels of nesting:

```hcl
resource "azurerm_virtual_network" "example" {
  name                = "example-vnet"
  resource_group_name = azurerm_resource_group.example.name
  location            = azurerm_resource_group.example.location
  address_space       = ["10.0.0.0/16"]

  subnet {
    name             = "delegated-subnet"
    address_prefixes = ["10.0.3.0/24"]

    delegation {
      name = "delegation1"

      service_delegation {
        name    = "Microsoft.DBforPostgreSQL/flexibleServers"
        actions = ["Microsoft.Network/virtualNetworks/subnets/join/action"]
      }
    }
  }
}
```

## Identity Block

A common pattern across many resources for managed identity:

```hcl
identity {
  type = "SystemAssigned"
}
```

```hcl
identity {
  type         = "UserAssigned"
  identity_ids = [azurerm_user_assigned_identity.example.id]
}
```

```hcl
identity {
  type         = "SystemAssigned, UserAssigned"
  identity_ids = [azurerm_user_assigned_identity.example.id]
}
```

- `SystemAssigned` - Azure creates and manages the identity; lifecycle tied to the resource
- `UserAssigned` - you create the identity separately and assign it; can be shared across resources
- `SystemAssigned, UserAssigned` - both types simultaneously
- When using `UserAssigned`, you must provide `identity_ids`
- Exported attributes: `principal_id`, `tenant_id` (for SystemAssigned)

## Timeouts Block

Override default operation timeouts:

```hcl
resource "azurerm_resource_group" "example" {
  name     = "example"
  location = "West Europe"

  timeouts {
    create = "60m"
    read   = "5m"
    update = "60m"
    delete = "60m"
  }
}
```

- Not all operations are available for every resource (e.g., data sources only have `read`)
- Format: duration string like `"30m"`, `"2h"`, `"90s"`
- Default timeouts vary by resource (typically 30m for create/update/delete, 5m for read)

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

- The `id` is always the Azure Resource ID
- Resource ID format: `/subscriptions/{sub}/resourceGroups/{rg}/providers/{provider}/{type}/{name}`
- Each resource's documentation includes the specific import ID format
- Use `provider::azurerm::normalise_resource_id()` to fix casing issues (Terraform 1.8+)

## Inline vs Separate Resources

Some resources support both inline configuration and separate resource definitions:

**Inline (in parent resource):**

```hcl
resource "azurerm_virtual_network" "example" {
  name                = "example-vnet"
  resource_group_name = azurerm_resource_group.example.name
  location            = azurerm_resource_group.example.location
  address_space       = ["10.0.0.0/16"]

  subnet {
    name             = "subnet1"
    address_prefixes = ["10.0.1.0/24"]
  }
}
```

**Separate resource:**

```hcl
resource "azurerm_virtual_network" "example" {
  name                = "example-vnet"
  resource_group_name = azurerm_resource_group.example.name
  location            = azurerm_resource_group.example.location
  address_space       = ["10.0.0.0/16"]
}

resource "azurerm_subnet" "example" {
  name                 = "subnet1"
  resource_group_name  = azurerm_resource_group.example.name
  virtual_network_name = azurerm_virtual_network.example.name
  address_prefixes     = ["10.0.1.0/24"]
}
```

- **You cannot mix both approaches** for the same sub-resource type - doing so causes conflicts
- To remove all inline sub-resources, set the block to an empty list (`subnet = []`)
- Inline is simpler for small configs; separate resources give more control and are better with `for_each`

## Note Conventions in Docs

- `->` (Info) - helpful information or tips
- `~>` (Warning) - important caveats or potential issues
- `!>` (Caution) - critical warnings, potential data loss or breaking changes

## Complete Example

Full working configuration from provider setup through resource creation:

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

resource "azurerm_resource_group" "example" {
  name     = "example-resources"
  location = "West Europe"

  tags = {
    environment = "Production"
  }
}

resource "azurerm_network_security_group" "example" {
  name                = "example-nsg"
  location            = azurerm_resource_group.example.location
  resource_group_name = azurerm_resource_group.example.name
}

resource "azurerm_virtual_network" "example" {
  name                = "example-vnet"
  location            = azurerm_resource_group.example.location
  resource_group_name = azurerm_resource_group.example.name
  address_space       = ["10.0.0.0/16"]
  dns_servers         = ["10.0.0.4", "10.0.0.5"]

  subnet {
    name             = "subnet1"
    address_prefixes = ["10.0.1.0/24"]
  }

  subnet {
    name             = "subnet2"
    address_prefixes = ["10.0.2.0/24"]
    security_group   = azurerm_network_security_group.example.id
  }

  tags = {
    environment = "Production"
  }
}

output "vnet_id" {
  value = azurerm_virtual_network.example.id
}

output "resource_group_id" {
  value = azurerm_resource_group.example.id
}
```
