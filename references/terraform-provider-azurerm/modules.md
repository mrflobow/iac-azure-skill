# Key Vault Module Reference

## azurerm_key_vault

Manages an Azure Key Vault.

### Arguments

| Argument | Description | Required | Default |
|----------|-------------|----------|---------|
| `name` | Globally unique vault name. Forces new on change. Soft-deleted vaults must be purged before name reuse | Yes | - |
| `location` | Azure region. Forces new on change | Yes | - |
| `resource_group_name` | Resource group name. Forces new on change | Yes | - |
| `sku_name` | `standard` or `premium` | Yes | - |
| `tenant_id` | Azure AD tenant ID for authenticating requests | Yes | - |
| `access_policy` | List of access policy blocks (max 1024). Set to `[]` to remove all | No | - |
| `enabled_for_deployment` | Allow VMs to retrieve certificates stored as secrets | No | `false` |
| `enabled_for_disk_encryption` | Allow Azure Disk Encryption to retrieve secrets and unwrap keys | No | `false` |
| `enabled_for_template_deployment` | Allow ARM to retrieve secrets | No | `false` |
| `rbac_authorization_enabled` | Use RBAC for data plane authorization instead of access policies | No | `false` |
| `network_acls` | Network ACL block | No | - |
| `purge_protection_enabled` | Enable purge protection. **Cannot be disabled once enabled** | No | `false` |
| `public_network_access_enabled` | Allow public network access | No | `true` |
| `soft_delete_retention_days` | Retention days for soft-deleted items (7-90). **Cannot be changed after creation** | No | `90` |
| `tags` | Tags map | No | - |

### access_policy Block

| Argument | Description | Required |
|----------|-------------|----------|
| `tenant_id` | Azure AD tenant ID (must match vault `tenant_id`) | Yes |
| `object_id` | Object ID of user, service principal, or security group (must be unique per policy) | Yes |
| `application_id` | Object ID of an Azure AD Application | No |
| `certificate_permissions` | Values: `Backup`, `Create`, `Delete`, `DeleteIssuers`, `Get`, `GetIssuers`, `Import`, `List`, `ListIssuers`, `ManageContacts`, `ManageIssuers`, `Purge`, `Recover`, `Restore`, `SetIssuers`, `Update` | No |
| `key_permissions` | Values: `Backup`, `Create`, `Decrypt`, `Delete`, `Encrypt`, `Get`, `Import`, `List`, `Purge`, `Recover`, `Restore`, `Sign`, `UnwrapKey`, `Update`, `Verify`, `WrapKey`, `Release`, `Rotate`, `GetRotationPolicy`, `SetRotationPolicy` | No |
| `secret_permissions` | Values: `Backup`, `Delete`, `Get`, `List`, `Purge`, `Recover`, `Restore`, `Set` | No |
| `storage_permissions` | Values: `Backup`, `Delete`, `DeleteSAS`, `Get`, `GetSAS`, `List`, `ListSAS`, `Purge`, `Recover`, `RegenerateKey`, `Restore`, `Set`, `SetSAS`, `Update` | No |

### network_acls Block

| Argument | Description | Required |
|----------|-------------|----------|
| `bypass` | `AzureServices` or `None` | Yes |
| `default_action` | `Allow` or `Deny` | Yes |
| `ip_rules` | List of IP addresses or CIDR blocks | No |
| `virtual_network_subnet_ids` | List of subnet IDs | No |

### Attributes

- `id` - Key Vault resource ID
- `vault_uri` - Vault URI for key/secret operations

### Import

```shell
terraform import azurerm_key_vault.example /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.KeyVault/vaults/{name}
```

### Important Notes

- **Inline vs separate access policies**: Cannot mix `access_policy` blocks inside `azurerm_key_vault` with separate `azurerm_key_vault_access_policy` resources - will cause conflicts
- **Soft delete recovery**: Provider automatically recovers soft-deleted vaults during creation (controlled by `features.key_vault.recover_soft_deleted_key_vaults`)
- **Purge protection**: Once enabled, cannot be disabled. Vault deletion schedules purge after retention period
- **RBAC permission change**: Requires unrestricted `Microsoft.Authorization/roleAssignments/write` permission (`Owner` or `User Access Administrator` role)

---

## azurerm_key_vault_access_policy

Manages a Key Vault Access Policy as a separate resource. Use this **or** inline `access_policy` blocks on `azurerm_key_vault` - never both.

### Arguments

| Argument | Description | Required |
|----------|-------------|----------|
| `key_vault_id` | Key Vault resource ID. Forces new on change | Yes |
| `tenant_id` | Azure AD tenant ID. Forces new on change | Yes |
| `object_id` | Object ID of user, service principal, or security group. Forces new on change | Yes |
| `application_id` | Object ID of an Azure AD Application. Forces new on change | No |
| `certificate_permissions` | Same values as inline access policy | No |
| `key_permissions` | Same values as inline access policy | No |
| `secret_permissions` | Same values as inline access policy | No |
| `storage_permissions` | Same values as inline access policy | No |

### Import

```shell
# With object_id only
terraform import azurerm_key_vault_access_policy.example /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.KeyVault/vaults/{name}/objectId/{object-id}

# With object_id and application_id
terraform import azurerm_key_vault_access_policy.example /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.KeyVault/vaults/{name}/objectId/{object-id}/applicationId/{app-id}
```

---

## azurerm_key_vault_secret

Manages a Key Vault Secret. Secret values are stored in plain-text in Terraform state.

### Arguments

| Argument | Description | Required | Default |
|----------|-------------|----------|---------|
| `name` | Secret name. Forces new on change | Yes | - |
| `key_vault_id` | Key Vault resource ID. Forces new on change | Yes | - |
| `value` | Secret value. Creates new version on change. One of `value` or `value_wo` required | Conditional | - |
| `value_wo` | Write-only secret value (not stored in state). Creates new version on change | Conditional | - |
| `value_wo_version` | Integer to trigger updates for `value_wo`. Increment when updating | No | - |
| `content_type` | Content type hint | No | - |
| `not_before_date` | Not usable before this UTC datetime (`Y-m-dTH:M:SZ`) | No | - |
| `expiration_date` | Expiration UTC datetime (`Y-m-dTH:M:SZ`) | No | - |
| `tags` | Tags map | No | - |

### Attributes

- `id` - Secret ID (versioned URL)
- `resource_id` - Versioned ID (won't auto-rotate in other Azure services)
- `resource_versionless_id` - Versionless ID (enables auto-rotation in other Azure services)
- `version` - Current version
- `versionless_id` - Base ID

### Import

```shell
terraform import azurerm_key_vault_secret.example "https://example-keyvault.vault.azure.net/secrets/example/fdf067c93bbb4b22bff4d8b7a9a56217"
```

### Notes

- Key Vault strips newlines. Preserve with `replace(file("secret"), "/\n/", "\n")` or `base64encode(file("secret"))`
- Recovery of soft-deleted secrets creates a new version (provider re-sets the value to ensure consistency)

---

## azurerm_key_vault_key

Manages a Key Vault cryptographic key.

### Arguments

| Argument | Description | Required | Default |
|----------|-------------|----------|---------|
| `name` | Key name. Forces new on change | Yes | - |
| `key_vault_id` | Key Vault resource ID. Forces new on change | Yes | - |
| `key_type` | `EC`, `EC-HSM`, `RSA`, `RSA-HSM`. Forces new on change | Yes | - |
| `key_size` | RSA key size in bytes (1024, 2048, etc.). Required for `RSA`/`RSA-HSM`. Forces new on change | Conditional | - |
| `curve` | EC curve: `P-256`, `P-256K`, `P-384`, `P-521`. Required for `EC`/`EC-HSM`. Forces new on change | Conditional | `P-256` |
| `key_opts` | Operations list (case-sensitive): `decrypt`, `encrypt`, `sign`, `unwrapKey`, `verify`, `wrapKey` | Yes | - |
| `rotation_policy` | Rotation policy block | No | - |
| `not_before_date` | Not usable before UTC datetime (`Y-m-dTH:M:SZ`) | No | - |
| `expiration_date` | Expiration UTC datetime. Cannot be unset once set. Removal forces new resource | No | - |
| `tags` | Tags map | No | - |

### rotation_policy Block

| Argument | Description | Required |
|----------|-------------|----------|
| `expire_after` | ISO 8601 duration (e.g., `P90D`) | No |
| `notify_before_expiry` | ISO 8601 duration (e.g., `P29D`) | No |
| `automatic` | Automatic rotation block | No |

### automatic Block

| Argument | Description | Required |
|----------|-------------|----------|
| `time_after_creation` | Rotate after ISO 8601 duration from creation | No |
| `time_before_expiry` | Rotate ISO 8601 duration before expiry | No |

### Attributes

- `id` - Key ID (versioned URL)
- `resource_id` - Versioned ID (won't auto-rotate)
- `resource_versionless_id` - Versionless ID (enables auto-rotation)
- `version` - Current version
- `versionless_id` - Base ID
- `n`, `e` - RSA modulus and public exponent
- `x`, `y` - EC X/Y components
- `public_key_pem` - PEM encoded public key
- `public_key_openssh` - OpenSSH encoded public key

### Import

```shell
terraform import azurerm_key_vault_key.example "https://example-keyvault.vault.azure.net/keys/example/fdf067c93bbb4b22bff4d8b7a9a56217"
```

### Required Permissions

- Without rotation policy: `Create`, `Delete`, `Get`, `Purge`, `Recover`, `Update`, `GetRotationPolicy`
- With rotation policy: add `SetRotationPolicy`

---

## azurerm_key_vault_certificate

Manages a Key Vault Certificate. Either import an existing cert (`certificate` block) or generate a new one (`certificate_policy` block).

### Arguments

| Argument | Description | Required | Default |
|----------|-------------|----------|---------|
| `name` | Certificate name. Forces new on change | Yes | - |
| `key_vault_id` | Key Vault resource ID. Forces new on change | Yes | - |
| `certificate` | Import block. Creates new version on change | Conditional | - |
| `certificate_policy` | Generation policy block. Changes (except `lifetime_action`) create new version | Conditional | - |
| `tags` | Tags map | No | - |

### certificate Block (Import)

| Argument | Description | Required |
|----------|-------------|----------|
| `contents` | Base64-encoded certificate contents. PFX: use `filebase64()`. PEM: must include cert + pkcs8 private key | Yes |
| `password` | Certificate password | No |

### certificate_policy Block (Generate)

| Argument | Description | Required |
|----------|-------------|----------|
| `issuer_parameters` | Issuer block | Yes |
| `key_properties` | Key properties block | Yes |
| `secret_properties` | Secret properties block | Yes |
| `lifetime_action` | Lifetime action block | No |
| `x509_certificate_properties` | X509 properties block. Required when not importing | Conditional |

### issuer_parameters Block

| Argument | Description | Required |
|----------|-------------|----------|
| `name` | `Self` for self-signed, `Unknown` for external CA (e.g., Let's Encrypt) | Yes |

### key_properties Block

| Argument | Description | Required |
|----------|-------------|----------|
| `exportable` | Is the key exportable? | Yes |
| `key_type` | `EC`, `EC-HSM`, `RSA`, `RSA-HSM`, `oct` | Yes |
| `key_size` | `2048`, `3072`, `4096` (RSA) or `256`, `384`, `521` (EC). Required for RSA | Conditional |
| `curve` | EC curve: `P-256`, `P-256K`, `P-384`, `P-521` | No |
| `reuse_key` | Reuse the key on renewal? | Yes |

### lifetime_action Block

| Argument | Description | Required |
|----------|-------------|----------|
| `action.action_type` | `AutoRenew` or `EmailContacts` | Yes |
| `trigger.days_before_expiry` | Days before expiry to trigger (conflicts with `lifetime_percentage`) | No |
| `trigger.lifetime_percentage` | Percentage of lifetime to trigger (conflicts with `days_before_expiry`) | No |

### secret_properties Block

| Argument | Description | Required |
|----------|-------------|----------|
| `content_type` | `application/x-pkcs12` (PFX) or `application/x-pem-file` (PEM) | Yes |

### x509_certificate_properties Block

| Argument | Description | Required |
|----------|-------------|----------|
| `subject` | Certificate subject (e.g., `CN=hello-world`) | Yes |
| `validity_in_months` | Validity period in months | Yes |
| `key_usage` | Values (case-sensitive): `cRLSign`, `dataEncipherment`, `decipherOnly`, `digitalSignature`, `encipherOnly`, `keyAgreement`, `keyCertSign`, `keyEncipherment`, `nonRepudiation` | Yes |
| `extended_key_usage` | OID list (e.g., `1.3.6.1.5.5.7.3.1` for Server Auth, `1.3.6.1.5.5.7.3.2` for Client Auth) | No |
| `subject_alternative_names.dns_names` | List of DNS FQDNs | No |
| `subject_alternative_names.emails` | List of email addresses | No |
| `subject_alternative_names.upns` | List of User Principal Names | No |

### Attributes

- `id` - Certificate ID (versioned URL)
- `secret_id` - Associated Secret ID
- `version` - Current version
- `versionless_id` - Base ID
- `versionless_secret_id` - Base Secret ID
- `certificate_data` - Raw cert data (hex string)
- `certificate_data_base64` - Base64-encoded cert data
- `thumbprint` - X509 thumbprint (hex string)
- `resource_manager_id` - Versioned ARM resource ID
- `resource_manager_versionless_id` - Versionless ARM resource ID

### Import

```shell
terraform import azurerm_key_vault_certificate.example "https://example-keyvault.vault.azure.net/certificates/example/fdf067c93bbb4b22bff4d8b7a9a56217"
```

---

## Data Sources

### data.azurerm_key_vault

| Argument | Description |
|----------|-------------|
| `name` | Vault name |
| `resource_group_name` | Resource group name |

**Attributes**: `id`, `vault_uri`, `location`, `tenant_id`, `sku_name`, `access_policy`, `enabled_for_deployment`, `enabled_for_disk_encryption`, `enabled_for_template_deployment`, `enable_rbac_authorization`, `purge_protection_enabled`, `public_network_access_enabled`, `tags`

### data.azurerm_key_vault_secret

| Argument | Description | Required |
|----------|-------------|----------|
| `name` | Secret name | Yes |
| `key_vault_id` | Key Vault ID | Yes |
| `version` | Specific version (defaults to latest) | No |

**Attributes**: `id`, `value`, `content_type`, `resource_id`, `resource_versionless_id`, `versionless_id`, `not_before_date`, `expiration_date`, `tags`

### data.azurerm_key_vault_key

| Argument | Description | Required |
|----------|-------------|----------|
| `name` | Key name | Yes |
| `key_vault_id` | Key Vault ID | Yes |

**Attributes**: `id`, `key_type`, `key_size`, `key_opts`, `curve`, `e`, `n`, `x`, `y`, `public_key_pem`, `public_key_openssh`, `resource_id`, `resource_versionless_id`, `version`, `versionless_id`, `tags`

### data.azurerm_key_vault_certificate

| Argument | Description | Required |
|----------|-------------|----------|
| `name` | Certificate name | Yes |
| `key_vault_id` | Key Vault ID | Yes |
| `version` | Specific version (defaults to latest) | No |

**Attributes**: `id`, `secret_id`, `version`, `versionless_id`, `versionless_secret_id`, `certificate_data`, `certificate_data_base64`, `thumbprint`, `certificate_policy`, `expires`, `not_before`, `resource_manager_id`, `resource_manager_versionless_id`, `tags`

---

## Ephemeral Resources

Ephemeral resources (Terraform 1.10+) read secrets without storing them in state.

### ephemeral.azurerm_key_vault_secret

| Argument | Description | Required |
|----------|-------------|----------|
| `name` | Secret name | Yes |
| `key_vault_id` | Key Vault ID | Yes |
| `version` | Specific version (defaults to latest) | No |

**Attributes**: `value`, `expiration_date`, `not_before_date`

### ephemeral.azurerm_key_vault_certificate

| Argument | Description | Required |
|----------|-------------|----------|
| `name` | Certificate name | Yes |
| `key_vault_id` | Key Vault ID | Yes |
| `version` | Specific version (defaults to latest) | No |

**Attributes**: `hex`, `pem`, `key`, `expiration_date`, `not_before_date`

---

## Provider Features Block (key_vault)

Controls destroy/recovery behavior. See [Concepts](terraform-provider-azure-concepts.md) for full list.

| Setting | Description | Default |
|---------|-------------|---------|
| `purge_soft_delete_on_destroy` | Purge vault on destroy | `true` |
| `purge_soft_deleted_certificates_on_destroy` | Purge certificates on destroy | `true` |
| `purge_soft_deleted_keys_on_destroy` | Purge keys on destroy | `true` |
| `purge_soft_deleted_secrets_on_destroy` | Purge secrets on destroy | `true` |
| `recover_soft_deleted_key_vaults` | Recover soft-deleted vault on create | `true` |
| `recover_soft_deleted_certificates` | Recover soft-deleted certificates | `true` |
| `recover_soft_deleted_keys` | Recover soft-deleted keys | `true` |
| `recover_soft_deleted_secrets` | Recover soft-deleted secrets | `true` |
