# Deploying an Azure Function with Terraform: A Step-by-Step Guide

## Overview

Azure Functions are a common choice for lightweight, event-driven services — things like sending notifications, processing queue messages, or running scheduled jobs. Deploying them by hand through the Azure Portal works fine for a one-off experiment, but it doesn't scale: you lose repeatability, you can't easily recreate the same environment twice, and there's no record of *why* a resource is configured the way it is.

This guide walks through provisioning an Azure Function App using Terraform (Infrastructure as Code), and wiring it up with a secure, keyless authentication method — Azure's Default Credential flow — instead of managing connection strings or secrets by hand.

By the end, you'll have:
- A Terraform configuration that provisions an Azure Function App from scratch
- Keyless authentication configured via Azure Default Credentials
- A repeatable deployment process you can version, review, and reuse

## Prerequisites

Before starting, make sure you have:
- An active Azure subscription
- [Terraform](https://developer.hashicorp.com/terraform/install) installed locally (v1.5+)
- The [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) installed and authenticated (`az login`)
- Basic familiarity with Azure resource groups and storage accounts

## Step 1: Set Up Your Terraform Project Structure

Create a new directory for your Terraform configuration:

```
azure-function-deploy/
├── main.tf
├── variables.tf
├── outputs.tf
└── function-app/
    └── (your function code)
```

Keeping the Terraform configuration separate from your function's application code makes it easier to reason about infrastructure changes independently from application logic changes.

## Step 2: Configure the Azure Provider

In `main.tf`, declare the Azure provider and a resource group to contain everything you'll create:

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "function_rg" {
  name     = "rg-function-notification-service"
  location = "East US"
}
```

Grouping related resources under a single resource group makes cleanup and lifecycle management straightforward — deleting the group removes everything provisioned inside it.

## Step 3: Provision Storage and the Function App

Azure Functions require a storage account for internal state and logging. Define that first, then the Function App itself:

```hcl
resource "azurerm_storage_account" "function_storage" {
  name                     = "stfuncnotifysvc"
  resource_group_name      = azurerm_resource_group.function_rg.name
  location                 = azurerm_resource_group.function_rg.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
}

resource "azurerm_service_plan" "function_plan" {
  name                = "asp-function-notification-service"
  resource_group_name = azurerm_resource_group.function_rg.name
  location            = azurerm_resource_group.function_rg.location
  os_type             = "Linux"
  sku_name            = "Y1"  # Consumption plan
}

resource "azurerm_linux_function_app" "notification_function" {
  name                       = "func-notification-service"
  resource_group_name       = azurerm_resource_group.function_rg.name
  location                  = azurerm_resource_group.function_rg.location
  storage_account_name      = azurerm_storage_account.function_storage.name
  storage_account_access_key = azurerm_storage_account.function_storage.primary_access_key
  service_plan_id           = azurerm_service_plan.function_plan.id

  identity {
    type = "SystemAssigned"
  }

  site_config {
    application_stack {
      java_version = "17"
    }
  }
}
```

The `identity { type = "SystemAssigned" }` block is doing important work here — it tells Azure to generate a managed identity for the Function App. This is the foundation of the keyless authentication approach covered next.

## Step 4: Configure Keyless Authentication with Azure Default Credentials

Rather than embedding connection strings or API keys in your application configuration, Azure's Default Credential flow lets your Function App authenticate to other Azure services (like Key Vault or a database) using its own managed identity — no secrets to rotate, store, or accidentally leak.

Grant the Function App's managed identity access to the resources it needs. For example, to allow it to read secrets from an Azure Key Vault:

```hcl
resource "azurerm_key_vault_access_policy" "function_access" {
  key_vault_id = azurerm_key_vault.notification_secrets.id
  tenant_id    = azurerm_linux_function_app.notification_function.identity[0].tenant_id
  object_id    = azurerm_linux_function_app.notification_function.identity[0].principal_id

  secret_permissions = ["Get", "List"]
}
```

In your application code, you'd then use the `DefaultAzureCredential` class (available in Azure SDKs for Java, .NET, Python, and others) to authenticate — it automatically detects and uses the managed identity when running in Azure, with no credentials hardcoded anywhere:

```java
DefaultAzureCredential credential = new DefaultAzureCredentialBuilder().build();
SecretClient secretClient = new SecretClientBuilder()
    .vaultUrl("https://your-vault.vault.azure.net/")
    .credential(credential)
    .buildClient();
```

## Step 5: Deploy

Initialize and apply your Terraform configuration:

```bash
terraform init
terraform plan
terraform apply
```

Review the plan output carefully before applying — Terraform will show you exactly what it intends to create, modify, or destroy, which is one of the biggest advantages over manual portal configuration: nothing happens silently.

## Step 6: Verify

Once the apply completes, confirm the Function App is running and can authenticate correctly:

```bash
az functionapp show --name func-notification-service --resource-group rg-function-notification-service
```

Check the Function App's logs (via Application Insights, if configured) to confirm it successfully authenticated to any dependent services using its managed identity, without any connection strings in the application settings.

## Why This Pattern Matters

Two things make this approach worth the extra setup compared to portal-based deployment:

1. **Repeatability.** The entire environment is defined in version-controlled files. If you need to recreate this in a different subscription or region, it's a config change and a re-apply, not a manual checklist.
2. **No secrets to manage.** The managed identity + Default Credential pattern eliminates an entire category of security risk — there's no connection string sitting in an environment variable or config file that could be accidentally committed or exposed.

For a production service, you'd typically extend this with remote state storage (so Terraform state isn't sitting on a single developer's machine), and CI/CD integration to run `terraform plan` on pull requests before merging.
