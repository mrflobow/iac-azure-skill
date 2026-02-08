## Metadata

Name: IaC Guidelines for Terragrunt

Description: Comprehensive guide and assistant for Infrastructure as Code (IaC) using Terraform, Terragrunt, and the AzureRM provider. Provides best practices for project structure, block generation, and troubleshooting.

This skill provides expert knowledge on creating, modifying, and adapting Terragrunt projects. It includes a deep understanding of the AzureRM Provider API and can assist with any Terraform-related inquiries, from basic configuration to complex deployment scenarios.

## Overview

This skill provides Florian's guidelines for creating consistent, professional IaC code. Use these guidelines when researching solutions or implementing changes. The projects use Terragrunt for deployment and state management, wrapping Terraform configuration. The primary provider is the AzureRM provider for Terraform.

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
├── providers.tf
├── README.md
└── test
    ├── module_test.go
    ├── env.hcl
    └── fixtures
        └── terragrunt.hcl
```

#### Testing Components

- **`test/module_test.go`**: Executes Terraform tests using the Terragrunt provider.
- **`test/env.hcl`**: Provides the basic setup for the test environment.
- **`test/fixtures/terragrunt.hcl`**: Overrides `env.hcl` and configures the provider. Use this file to add additional fixed inputs for the test.

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

## Resources

See the resources folder for logo files and font downloads.
