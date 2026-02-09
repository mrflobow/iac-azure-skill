# Key Vault Example Patterns

## Basic Key Vault with Access Policy

Minimal vault with inline access policy using access policy authorization model.

```hcl
data "azurerm_client_config" "current" {}

resource "azurerm_key_vault" "example" {
  name                       = "${var.prefix}-kv"
  location                   = azurerm_resource_group.example.location
  resource_group_name        = azurerm_resource_group.example.name
  tenant_id                  = data.azurerm_client_config.current.tenant_id
  sku_name                   = "standard"
  soft_delete_retention_days = 7
  purge_protection_enabled   = false

  access_policy {
    tenant_id = data.azurerm_client_config.current.tenant_id
    object_id = data.azurerm_client_config.current.object_id

    secret_permissions = ["Get", "Set", "Delete", "Purge", "Recover"]
    key_permissions    = ["Get", "Create"]
  }
}
```

---

## Key Vault with Secret

Vault + secret storage pattern. Requires `Set`, `Get`, `Delete`, `Purge`, `Recover` secret permissions.

```hcl
resource "azurerm_key_vault" "example" {
  name                       = "${var.prefix}-kv"
  location                   = azurerm_resource_group.example.location
  resource_group_name        = azurerm_resource_group.example.name
  tenant_id                  = data.azurerm_client_config.current.tenant_id
  sku_name                   = "premium"
  soft_delete_retention_days = 7

  access_policy {
    tenant_id          = data.azurerm_client_config.current.tenant_id
    object_id          = data.azurerm_client_config.current.object_id
    secret_permissions = ["Set", "Get", "Delete", "Purge", "Recover"]
    key_permissions    = ["Create", "Get"]
  }
}

resource "azurerm_key_vault_secret" "example" {
  name         = "secret-sauce"
  value        = "szechuan"
  key_vault_id = azurerm_key_vault.example.id
}
```

---

## Key Vault with Key and Rotation Policy

Vault + cryptographic RSA key with automatic rotation. Requires `premium` SKU for HSM keys. Key permissions must include `GetRotationPolicy` and `SetRotationPolicy` when using rotation policy.

```hcl
resource "azurerm_key_vault" "example" {
  name                       = "${var.prefix}-kv"
  location                   = azurerm_resource_group.example.location
  resource_group_name        = azurerm_resource_group.example.name
  tenant_id                  = data.azurerm_client_config.current.tenant_id
  sku_name                   = "premium"
  soft_delete_retention_days = 7

  access_policy {
    tenant_id = data.azurerm_client_config.current.tenant_id
    object_id = data.azurerm_client_config.current.object_id

    key_permissions = [
      "Create", "Delete", "Get", "Purge", "Recover", "Update",
      "GetRotationPolicy", "SetRotationPolicy",
    ]
    secret_permissions = ["Set"]
  }
}

resource "azurerm_key_vault_key" "example" {
  name         = "${var.prefix}-key"
  key_vault_id = azurerm_key_vault.example.id
  key_type     = "RSA"
  key_size     = 2048

  key_opts = ["decrypt", "encrypt", "sign", "unwrapKey", "verify", "wrapKey"]

  rotation_policy {
    automatic {
      time_before_expiry = "P30D"
    }
    expire_after         = "P90D"
    notify_before_expiry = "P29D"
  }
}
```

---

## Key Vault with Self-Signed Certificate

Generate a self-signed certificate via `certificate_policy`. Common for Service Fabric, App Gateway, and internal TLS.

```hcl
resource "azurerm_key_vault" "example" {
  name                   = "${var.prefix}-kv"
  location               = azurerm_resource_group.example.location
  resource_group_name    = azurerm_resource_group.example.name
  tenant_id              = data.azurerm_client_config.current.tenant_id
  sku_name               = "standard"
  enabled_for_deployment = true

  access_policy {
    tenant_id = data.azurerm_client_config.current.tenant_id
    object_id = data.azurerm_client_config.current.object_id

    certificate_permissions = ["Create", "Delete", "Get", "List", "Update"]
    key_permissions         = ["Create"]
    secret_permissions      = ["Set"]
  }
}

resource "azurerm_key_vault_certificate" "example" {
  name         = "${var.prefix}-cert"
  key_vault_id = azurerm_key_vault.example.id

  certificate_policy {
    issuer_parameters {
      name = "Self"
    }

    key_properties {
      exportable = true
      key_size   = 2048
      key_type   = "RSA"
      reuse_key  = true
    }

    lifetime_action {
      action {
        action_type = "AutoRenew"
      }
      trigger {
        days_before_expiry = 30
      }
    }

    secret_properties {
      content_type = "application/x-pkcs12"
    }

    x509_certificate_properties {
      extended_key_usage = ["1.3.6.1.5.5.7.3.1", "1.3.6.1.5.5.7.3.2"]

      key_usage = [
        "cRLSign", "dataEncipherment", "digitalSignature",
        "keyAgreement", "keyCertSign", "keyEncipherment",
      ]

      subject            = "CN=${var.prefix}.${var.location}.cloudapp.azure.com"
      validity_in_months = 12
    }
  }
}
```

---

## Key Vault with Imported Certificate

Import a PFX certificate. Use separate `azurerm_key_vault_access_policy` resources when multiple principals need access. The `certificate_policy` block with `issuer_parameters.name = "Unknown"` is required for imported certs.

```hcl
resource "azurerm_key_vault" "example" {
  name                = "${var.prefix}-kv"
  location            = azurerm_resource_group.example.location
  resource_group_name = azurerm_resource_group.example.name
  tenant_id           = data.azurerm_client_config.current.tenant_id
  sku_name            = "standard"
}

resource "azurerm_key_vault_access_policy" "current_user" {
  key_vault_id            = azurerm_key_vault.example.id
  tenant_id               = azurerm_key_vault.example.tenant_id
  object_id               = data.azurerm_client_config.current.object_id
  certificate_permissions = ["Get", "Import"]
}

resource "azurerm_key_vault_certificate" "example" {
  name         = "${var.prefix}-cert"
  key_vault_id = azurerm_key_vault.example.id

  certificate {
    contents = filebase64("certificate.pfx")
    password = "terraform"
  }

  certificate_policy {
    issuer_parameters {
      name = "Unknown"
    }
    key_properties {
      exportable = true
      key_size   = 2048
      key_type   = "RSA"
      reuse_key  = false
    }
    secret_properties {
      content_type = "application/x-pkcs12"
    }
  }

  depends_on = [azurerm_key_vault_access_policy.current_user]
}

# Use secret_id to reference cert from other resources (e.g., App Service Certificate)
resource "azurerm_app_service_certificate" "example" {
  name                = "${var.prefix}-cert"
  resource_group_name = azurerm_resource_group.example.name
  location            = azurerm_resource_group.example.location
  key_vault_secret_id = azurerm_key_vault_certificate.example.secret_id
}
```

---

## Customer-Managed Key (CMK) Pattern - Cosmos DB

Key Vault provides encryption keys for other Azure services. Pattern requires:
- `premium` SKU (for HSM-backed keys)
- `purge_protection_enabled = true`
- Separate access policy for the service principal of the target service

```hcl
data "azuread_service_principal" "cosmosdb" {
  display_name = "Azure Cosmos DB"
}

resource "azurerm_key_vault" "example" {
  name                     = "${var.prefix}kv"
  location                 = azurerm_resource_group.example.location
  resource_group_name      = azurerm_resource_group.example.name
  tenant_id                = data.azurerm_client_config.current.tenant_id
  sku_name                 = "premium"
  purge_protection_enabled = true

  # Deployer access - full key management
  access_policy {
    tenant_id       = data.azurerm_client_config.current.tenant_id
    object_id       = data.azurerm_client_config.current.object_id
    key_permissions = ["List", "Create", "Delete", "Get", "Update"]
  }

  # Service access - wrap/unwrap only
  access_policy {
    tenant_id       = data.azurerm_client_config.current.tenant_id
    object_id       = data.azuread_service_principal.cosmosdb.id
    key_permissions = ["Get", "UnwrapKey", "WrapKey"]
  }
}

resource "azurerm_key_vault_key" "example" {
  name         = "${var.prefix}key"
  key_vault_id = azurerm_key_vault.example.id
  key_type     = "RSA"
  key_size     = 3072
  key_opts     = ["decrypt", "encrypt", "wrapKey", "unwrapKey"]
}

resource "azurerm_cosmosdb_account" "example" {
  name                = "${var.prefix}-cosmosdb"
  location            = azurerm_resource_group.example.location
  resource_group_name = azurerm_resource_group.example.name
  offer_type          = "Standard"
  kind                = "MongoDB"
  key_vault_key_id    = azurerm_key_vault_key.example.id

  consistency_policy {
    consistency_level = "Strong"
  }
  geo_location {
    location          = azurerm_resource_group.example.location
    failover_priority = 0
  }
}
```

---

## Customer-Managed Key (CMK) Pattern - Databricks

Databricks CMK uses `azurerm_databricks_workspace_root_dbfs_customer_managed_key`. The workspace managed identity needs Key Vault access, which requires creating the workspace first.

```hcl
resource "azurerm_databricks_workspace" "example" {
  name                         = "${var.prefix}-DBW"
  resource_group_name          = azurerm_resource_group.example.name
  location                     = azurerm_resource_group.example.location
  sku                          = "premium"
  managed_resource_group_name  = "${var.prefix}-DBW-managed-dbfs"
  customer_managed_key_enabled = true
}

resource "azurerm_key_vault" "example" {
  name                       = "${var.prefix}-keyvault"
  location                   = azurerm_resource_group.example.location
  resource_group_name        = azurerm_resource_group.example.name
  tenant_id                  = data.azurerm_client_config.current.tenant_id
  sku_name                   = "premium"
  purge_protection_enabled   = true
  soft_delete_retention_days = 7
}

# Deployer access
resource "azurerm_key_vault_access_policy" "terraform" {
  key_vault_id = azurerm_key_vault.example.id
  tenant_id    = azurerm_key_vault.example.tenant_id
  object_id    = data.azurerm_client_config.current.object_id

  key_permissions = [
    "Get", "List", "Create", "Decrypt", "Encrypt", "Sign",
    "UnwrapKey", "Verify", "WrapKey", "Delete", "Restore",
    "Recover", "Update", "Purge", "GetRotationPolicy", "SetRotationPolicy",
  ]
}

# Databricks managed identity access (created after workspace)
resource "azurerm_key_vault_access_policy" "databricks" {
  depends_on   = [azurerm_databricks_workspace.example]
  key_vault_id = azurerm_key_vault.example.id
  tenant_id    = azurerm_databricks_workspace.example.storage_account_identity.0.tenant_id
  object_id    = azurerm_databricks_workspace.example.storage_account_identity.0.principal_id
  key_permissions = ["Get", "UnwrapKey", "WrapKey"]
}

resource "azurerm_key_vault_key" "example" {
  depends_on   = [azurerm_key_vault_access_policy.terraform]
  name         = "${var.prefix}-certificate"
  key_vault_id = azurerm_key_vault.example.id
  key_type     = "RSA"
  key_size     = 2048
  key_opts     = ["Decrypt", "Encrypt", "Sign", "UnwrapKey", "Verify", "WrapKey"]
}

resource "azurerm_databricks_workspace_root_dbfs_customer_managed_key" "example" {
  depends_on       = [azurerm_key_vault_access_policy.databricks]
  workspace_id     = azurerm_databricks_workspace.example.id
  key_vault_key_id = azurerm_key_vault_key.example.id
}
```

---

## Disk Encryption Set

Key Vault provides encryption keys for managed disks via `azurerm_disk_encryption_set`. The disk encryption set's managed identity needs Key Vault access plus `Reader` role on the vault.

```hcl
resource "azurerm_key_vault" "example" {
  name                        = "${var.prefix}kv"
  location                    = azurerm_resource_group.example.location
  resource_group_name         = azurerm_resource_group.example.name
  tenant_id                   = data.azurerm_client_config.current.tenant_id
  sku_name                    = "premium"
  enabled_for_disk_encryption = true
  purge_protection_enabled    = true
}

# Deployer access
resource "azurerm_key_vault_access_policy" "deployer" {
  key_vault_id       = azurerm_key_vault.example.id
  tenant_id          = data.azurerm_client_config.current.tenant_id
  object_id          = data.azurerm_client_config.current.object_id
  key_permissions    = ["Create", "Delete", "Get", "Update"]
  secret_permissions = ["Delete", "Get", "Set"]
}

resource "azurerm_key_vault_key" "example" {
  name         = "examplekey"
  key_vault_id = azurerm_key_vault.example.id
  key_type     = "RSA"
  key_size     = 2048
  key_opts     = ["decrypt", "encrypt", "sign", "unwrapKey", "verify", "wrapKey"]
  depends_on   = [azurerm_key_vault_access_policy.deployer]
}

resource "azurerm_disk_encryption_set" "example" {
  name                = "${var.prefix}-des"
  resource_group_name = azurerm_resource_group.example.name
  location            = azurerm_resource_group.example.location
  key_vault_key_id    = azurerm_key_vault_key.example.id

  identity {
    type = "SystemAssigned"
  }
}

# Disk encryption set managed identity access
resource "azurerm_key_vault_access_policy" "disk_encryption" {
  key_vault_id    = azurerm_key_vault.example.id
  tenant_id       = azurerm_disk_encryption_set.example.identity.0.tenant_id
  object_id       = azurerm_disk_encryption_set.example.identity.0.principal_id
  key_permissions = ["Get", "UnwrapKey", "WrapKey"]
}

resource "azurerm_role_assignment" "disk_encryption_read_keyvault" {
  scope                = azurerm_key_vault.example.id
  role_definition_name = "Reader"
  principal_id         = azurerm_disk_encryption_set.example.identity.0.principal_id
}

resource "azurerm_managed_disk" "example" {
  name                   = "${var.prefix}-disk"
  location               = azurerm_resource_group.example.location
  resource_group_name    = azurerm_resource_group.example.name
  storage_account_type   = "Standard_LRS"
  create_option          = "Empty"
  disk_size_gb           = 10
  disk_encryption_set_id = azurerm_disk_encryption_set.example.id

  depends_on = [
    azurerm_role_assignment.disk_encryption_read_keyvault,
    azurerm_key_vault_access_policy.disk_encryption,
  ]
}
```

---

## Common Patterns Summary

| Pattern | SKU | Purge Protection | Key Permissions for Service |
|---------|-----|------------------|-----------------------------|
| Secret storage | `standard` | Optional | N/A (secret permissions) |
| Self-signed cert | `standard` | Optional | N/A (certificate permissions) |
| Imported cert | `standard` | Optional | N/A (certificate permissions) |
| CMK (Cosmos DB, Databricks) | `premium` | **Required** | `Get`, `UnwrapKey`, `WrapKey` |
| Disk encryption | `premium` | **Required** | `Get`, `UnwrapKey`, `WrapKey` + `Reader` role |
