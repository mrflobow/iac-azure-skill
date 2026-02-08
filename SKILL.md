## Metadata

Name: IaC Guidelines for Terragrunt

Description: Comprehensive guide and assistant for Infrastructure as Code (IaC) using Terraform, Terragrunt, and the AzureRM provider. Provides best practices for project structure, block generation, and troubleshooting.

This skill provides expert knowledge on creating, modifying, and adapting Terragrunt projects. It includes a deep understanding of the AzureRM Provider API and can assist with any Terraform-related inquiries, from basic configuration to complex deployment scenarios.

## Overview

This skill provides guidelines for creating consistent, professional IaC code. Use these guidelines when researching solutions or implementing changes. The projects use Terragrunt for deployment and state management, wrapping Terraform configuration. The primary provider is the AzureRM provider for Terraform.

The AI assistant should reference these guidelines whenever planning or creating code.

## When to Apply

Apply these guidelines whenever:

- Creating or modifying Terraform files (`.tf`)
- Working on Terragrunt configurations (`terragrunt.hcl`)
- Seeking assistance with Terraform or Terragrunt related questions

## Structure

### Module Structure

```
.
├── main.tf
├── variables.tf
├── outputs.tf
├── versions.tf
├── locals.tf
├── data.tf
├── README.md
└── test
    ├── module_test.go
    ├── env.hcl
    └── fixtures/modulename/
        └── terragrunt.hcl
```

#### Testing Components

- **`test/module_test.go`**: Executes Terraform tests using Terratest.
- **`test/env.hcl`**: Provides the basic setup for the test environment.
- **`test/fixtures/modulename/terragrunt.hcl`**: Overrides `env.hcl` and configures the provider. Use this file to add additional fixed inputs for the test.

### Project Structure

```
.
├── env
│   └── live
│       ├── dev
│       │   ├── env.hcl
│       │   └── myproduct
│       │       └── terragrunt.hcl
│       └── test
│           ├── env.hcl
│           └── myproduct2
│               └── terragrunt.hcl
└── modules
    ├── storageaccount
    └── keyvault
```

`env.hcl` has the basic setup for the environment such as Azure Provider and default Terragrunt state storage which is overridden by each product/component and is based on the folder path in `/live/<env>/product`.

## References

Consult the `references/terragrunt/` folder for comprehensive Terragrunt documentation:

- **[Core Concepts](references/terragrunt/terragrunt-concepts.md)** - Fundamental concepts for working with Terragrunt
- **[HCL Blocks](references/terragrunt/terragrunt-hcl-blocks.md)** - Configuration blocks that define structural configurations
- **[HCL Attributes](references/terragrunt/terragrunt-hcl-attributes.md)** - Attributes for Terragrunt configuration such as inputs and operational settings
- **[Functions](references/terragrunt/terragrunt-functions.md)** - Built-in Terragrunt functions for use in `terragrunt.hcl` files
- **[CLI](references/terragrunt/terragrunt-cli.md)** - CLI commands for managing infrastructure at scale

Read the relevant reference files before generating or modifying Terragrunt configurations.
