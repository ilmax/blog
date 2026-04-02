---
title: "Publish Dacpac to Azure SQL with Entra-only authentication using GitHub Actions"
description: How to use GitHub Actions to publish dacpac database projects to Azure SQL when Entra-Only authentication is enabled
date: 2024-08-20T19:59:36+02:00
tags: [Azure, DevOps, SQL, Terraform]
draft: false
---

In Azure, most services nowadays supports Microsoft Entra based authentication, for example via a system managed identity or a user assigned one.
This authentication method is preferred to the old connection string based one because it gets rids of secrets and with that the need of secret rotation.

Some services then push this concept even further and allow you to completely disable a secret based connection, Azure SQL is one of them. When Entra-only authentication is enabled it's pretty easy to configure a service running in Azure to connect to the service in question but what about the CI/CD pipelines?

This article explains how to connect to Azure SQL when Microsoft Entra-only authentication is enabled.

## Prerequisites

Create a service principal used by GitHub actions to connect to the Azure subscription. This can be created with or without using a shared secret. Using the shared secret option is easier but also less secure and can be achieved with the following:

```sh
az ad sp create-for-rbac -n YourServicePrincipalNameHere --role Owner --scopes /subscriptions/YourSubcriptionIdHere
```

If you want to use the secretless approach, you can refer to my other post here that explains step-by-step how to set up and configure a Microsoft Entra application and configure a federated identity credential on the application:

{{<article link="/posts/github-azure-oidc/">}}

> [!TIP]
> The article above uses an App registration with a service principal, the same can also be achieved via a user-assigned managed identity. You can find more details on the Microsoft [documentation](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure-openid-connect).

## Enable Entra-only authentication

Enable Entra-only authentication has to be done at the Azure SQL Server level and can be achieved in several ways:

### Terraform azurerm_mssql_server

```hcl
resource "azurerm_mssql_server" "sql_server" {
  name                = local.server_name
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  version             = "12.0"
  minimum_tls_version = "1.2"

  azuread_administrator {
    login_username              = var.admin_username
    object_id                   = var.admin_object_id
    azuread_authentication_only = true                  # Add this to enable Entra-only authentication
  }
}
```

### AZ CLI

```sh
az sql server ad-only-auth enable --resource-group mygroup --name myServer
```

### Azure Portal

{{<figure src="azure-portal.png" alt="Enable Entra-only authentication in the portal" caption="*Enable Entra-only authentication in the portal*">}}

## Create a SQL user

### Manual script

Now we need to create the user in the SQL Database that represent the service principal and grant the necessary permissions, we achieve this by running the following script:

```sql
CREATE USER [ServicePrincipalName] FROM EXTERNAL PROVIDER;
GO
ALTER ROLE db_owner ADD MEMBER [ServicePrincipalName];

-- Depending on your requirements you can also use a less privileged role, e.g. db_ddladmin
```

> [!NOTE]
> When creating a user mapped to an Azure service principal (e.g., when using FROM EXTERNAL PROVIDER), you must connect to the database using Microsoft Entra authentication. If you try to run this script using a regular username and password connection, you will get an error message like the following:
>
> `
> Failed to execute query. Error: Principal 'ServicePrincipalName' could not be created.
> Only connections established with Active Directory accounts can create other Active Directory users.
> `

### Terraform mssql_user

The official Azure Terraform provider, azurerm, doesn't support creating Azure SQL users. However there's a [community provider](https://registry.terraform.io/providers/betr-io/mssql/latest) that does support Azure SQL user creation.

If you apply your infrastructure changes manually, configuring this might be worthwhile. However, if you use CI/CD pipelines to apply changes, it creates a chicken-and-egg problem since mssql_user still requires an Entra connection to the Azure SQL database.

## GitHub action workflow

Now that everything is set up, we can examine the action workflow. My workflow is a simplified example that generates the DACPAC, connects to Azure, and publishes it.

In a more realistic setup, the workflow might be divided into multiple jobs. The first job would build the DACPAC and create an artifact, while the subsequent jobs would deploy the same artifact across different environments, with steps like approval, baking time, and so on.

### Building the Dacpac

For this demo, I've used the community built **MSBuild.Sdk.SqlProj**, but the same can be achieved with a regular database project created in Visual Studio. This SDK uses regular csproj to generate a dacpac hence it's very convenient to use because all it takes is just a simple command:

```sh
dotnet build path-to-the-project.csproj --configuration Release
```

### Get the connection string

Since the connection string does not contain secrets anymore, there’s no need to put it in a GitHub Action secret. So for the sake of simplicity in this example a connection string is created in a workflow step, but you can create it in any way you like.

The connection string format should be the following:

`
Server=tcp:{SqlServerName}.database.windows.net,1433; Initial Catalog={DatabaseName}; Authentication=Active Directory Default; Encrypt=True; TrustServerCertificate=False; Connection Timeout=30;"
`

> [!NOTE]
> It would be nice if [az cli](https://learn.microsoft.com/en-us/cli/azure/sql/db?view=azure-cli-latest#az-sql-db-show-connection-string) had support for creating the correct connection string but, as of now, this is not supported.
>
> Make sure you replace **SqlServerName** and **DatabaseName** with the appropriate values.

### Publishing the dacpac

Publish the dacpac can be done with the azure/sql-action, this is an action that wraps sqlcmd and provided by Microsoft. The action requires a connection string, the dacpac file and the action to perform.

Here below, you can see the interesting part of the workflow:

```yml
jobs:
  deploy:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup dotnet SDK
        uses: actions/setup-dotnet@v4

      - name: Create Dacpac
        run: dotnet build Database/Database.csproj --configuration Release -o ./out

      - name: Login to Azure
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Construct connection string
        run: |
          connection_string="Server=tcp:${{ vars.SQL_SERVER }}.database.windows.net,1433; Initial Catalog=${{ vars.SQL_DATABASE }}; Authentication=Active Directory Default; Encrypt=True; TrustServerCertificate=False; Connection Timeout=30;"
          echo "::add-mask::$connection_string"
          echo "connection_string=$connection_string" >> $GITHUB_ENV

      - name: Deploy Database
        uses: azure/sql-action@v2.3
        with:
          connection-string: ${{ env.connection_string }}
          action: 'publish'
          path: ./out/Database.dacpac

      - name: Logout of Azure
        run: az logout
```

And now what's left is just running the action itself!

{{<figure src="action.png" alt="GitHub Action workflow run" caption="*GitHub Action workflow run*">}}

All the code for this blog post can be found here:

{{< github repo="ilmax/azure-sql-entra-only-id" >}}

## Conclusion

Enabling Entra-only authentication can help us improve our Azure SQL security baseline. Additionally, the process of updating GitHub Action workflows to incorporate Entra-only authentication is straightforward and manageable. This means we can seamlessly integrate this security measure into our CI/CD pipelines, taking full advantage of the benefits it provides.

Till the next time!
