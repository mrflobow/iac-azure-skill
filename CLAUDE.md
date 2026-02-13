# CLAUDE.md — iac-azure Skill

## Project Purpose

This is a Claude Code skill that provides Infrastructure as Code (IaC) guidelines for Terraform, Terragrunt, and the AzureRM provider. It is loaded as context when users work on Azure infrastructure projects, giving Claude knowledge of resource definitions, best practices, and practical patterns.

## File Structure

```
.
├── SKILL.md                          # Skill entry point (metadata + overview + references)
├── CLAUDE.md                         # Development guidelines (this file)
└── references/
    ├── terragrunt/                   # Terragrunt documentation (concepts, blocks, attributes, functions, CLI)
    ├── terraform-provider-azurerm/   # AzureRM provider documentation
    │   ├── terraform-provider-azure-concepts.md   # Provider config, auth, core concepts
    │   ├── terraform-provider-azure-blocks.md      # Resource/data source block patterns
    │   ├── terraform-provider-azure-functions.md   # Provider functions
    │   ├── terraform-provider-azure-cli.md         # Provider setup and CLI usage
    │   ├── modules.md                              # Resource definitions and arguments
    │   └── examples.md                             # Practical usage patterns
    └── terratest/                    # Terratest documentation (testing IaC)
        ├── terratest-concepts.md                    # Core concepts, lifecycle, options structs
        ├── terratest-cli.md                         # Test execution commands and flags
        ├── terratest-examples.md                    # Complete testing patterns
        └── terratest-azure-helpers.md               # Azure-specific validation helpers
```

## How to Add New Modules

### Adding to `modules.md`

1. Add a new `## Section Name` under the appropriate top-level category
2. For each resource, document:
   - Resource name as `## azurerm_resource_name` (or `###` if under a category heading)
   - Brief description
   - **Arguments** table: `| Argument | Description | Required | Default |`
   - **Sub-blocks** as separate subsections with their own argument tables
   - **Attributes** list (exported read-only values)
   - **Import** command in a shell code block
   - **Important Notes** for gotchas, constraints, or best practices
3. Include **Data Sources** with their arguments and attributes
4. Use `---` horizontal rules to separate major resource sections

### Adding to `examples.md`

1. Add examples under a `## Category Patterns` heading
2. Each example gets a `## Descriptive Title` with:
   - Brief description of the pattern and when to use it
   - Complete HCL code block (resource group through target resource)
   - Code should use `var.prefix` for naming and reference related resources
3. Add a **Common Patterns Summary** table at the end of each category section
4. Use `---` horizontal rules between examples

## Conventions

- Markdown tables for arguments: `| Argument | Description | Required | Default |`
- HCL code blocks: ` ```hcl `
- Shell code blocks for import commands: ` ```shell `
- `---` separators between resource sections and between examples
- Alphabetical ordering of categories (e.g., Key Vault before Linux Web App)
- Keep descriptions concise — focus on what's essential for code generation
- Include prerequisite resources in examples (resource groups, service plans, etc.)
