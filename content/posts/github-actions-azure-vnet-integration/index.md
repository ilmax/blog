---
title: "GitHub Actions Azure Vnet Integration"
description: Securely connect GitHub Actions to Azure PaaS with public access disabled. Follow this guide to seamless CI/CD integration while keeping your resources safe.
date: 2024-07-24T14:26:00+02:00
draft: false
tags: ["github", "azure", "devops", "terraform"]
type: "featured"
---

In today's post, we will look at an interesting challenge, having GitHub actions interact with Azure PaaS services for which we have disabled public access.

## Problem Statement

If you are working on improving your cloud security posture on Azure, one of the first things that you should look into when deploying PaaS services (like for example Azure Storage, Azure Cosmos DB or Azure SQL Server) is to disable the public access.

Most PaaS services nowadays allow you to disable public access, meaning that you can't connect to those services over the Internet anymore. For the applications deployed in Azure, you can take advantage of Virtual Networks, Private Endpoints and Private DNS Zones to enable private connectivity without changing any single line of code.

This is all well and good, but what about those services that aren't deployed in Azure, like for example the CI/CD runners?

Chances are that you interact with your infrastructure in CI/CD pipelines (like for example running `terraform apply`) and, as soon as you close the firewall, some operations will start to fail due to connectivity problems.

At this point, we have several ways of fixing the issue, here's a quick non-exhaustive list off the top of my head:

- Open and close the PaaS Service public access when the pipeline starts and revert the operation at the end
- Allow access to the PaaS Services from within your office network and self-host your runners within your office network
- Self-host your runners in Azure Virtual Machines with network connectivity to those services

All the options in the above list will fix the issue but they all have some disadvantages, so can we do better?
Not long ago, we got a new and better alternative which is to take advantage of GitHub private networking for hosted runners.

{{<figure src="github_private_vnet.webp" alt="GitHub private networking inner workings" caption="*GitHub private networking inner workings, image courtesy of https://docs.github.com/en/organizations/managing-organization-settings/about-azure-private-networking-for-github-hosted-runners-in-your-organization*" nozoom=true >}}

## GitHub private networking

This relatively new feature (Public beta started in November 2023 and went GA in April 2024), allows GitHub-hosted runners to use a network interface card (NIC) created in a subnet under your control. This way we don't have to manage our own hosted runners, GitHub will keep managing the runners for us, while we will still be able to connect to PaaS service using private connectivity.

To set this up we need to configure a few things:

1. Get your GitHub the Enterprise ID or Organization ID (more on this below)
1. Create the Azure VNET that will host the NICs used by the GitHub runners
1. Create a GitHub network configuration in Azure
1. Create a Hosted Computed Network configuration in GitHub
1. Create Runner Group(s) and Runner(s) on GitHub
1. Change your GitHub action `runs-on` to reference the runner

> [!CAUTION]
> This functionality is only supported by **GitHub Enterprise** and **GitHub Team** plans

## Azure private connectivity

Let's now take a look into a few concepts that are necessary for understanding how this can be configured.

{{<figure src="private_endpoint.png" alt="Azure Private Link overview" caption="*Azure Private Link overview, image courtesy of https://azure.microsoft.com/en-us/products/private-link/*" nozoom=true >}}

### Private Endpoints

Private Endpoints can be thought of as read-only Network Interface Cards (NICs) for your PaaS services. Those NICs are created in the subnet you specify and are assigned a private IP from the VNET address space. The connection between the private endpoint and the PaaS service uses a secure private link.

When using Private Endpoints, the traffic never leaves the Microsoft backbone network as opposed to going through the public internet, making it not only more secure but also a faster way to access your PaaS services.

### Private DNS Zones

When a PaaS service enables the Private Endpoints, at the DNS level the service FQDN turns into a CNAME to the private link zone for the related service.
Let's see the changes in resolving an Azure Storage Account hostname with the `dig` command (available in Linux and MacOS, when using Windows you can use `dig` on WSL or `nslookup` on cmd/PowerShell)

No Private Endpoints

```sh
dig example.blob.core.windows.net

;; ANSWER SECTION:
example.blob.core.windows.net. 60 IN CNAME blob.ams08prdstr13c.store.core.windows.net.
blob.ams08prdstr13c.store.core.windows.net. 86400 IN A 1.2.3.4
```

With Private Endpoints

```sh
dig examplepe.blob.core.windows.net

;; ANSWER SECTION:
examplepe.blob.core.windows.net. 60 IN CNAME examplepe.privatelink.blob.core.windows.net.
examplepe.privatelink.blob.core.windows.net. 60 IN CNAME blob.am5prdstr12a.store.core.windows.net.
blob.am5prdstr12a.store.core.windows.net. 60 IN A 1.2.3.5
```

As you can see above, after we turn on Private Endpoints for a particular service, we get another DNS indirection. This zone coincides with the name of the Private DNS Zone in which we have to create our records to enable private connectivity.

> [!TIP]
> You can read more about how this works in the Microsoft [documentation](https://learn.microsoft.com/en-us/azure/storage/common/storage-private-endpoints#dns-changes-for-private-endpoints).
>
> Different services have different DNS Zones and those are mentioned in the documentation [here](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dns).

An A record is then created in the respective Private DNS Zone that resolves to the IP of the NIC that represents your PaaS service.

Private Endpoint can be linked to one or more Private DNS Zones to make sure that whenever the IP of the NIC connected to the Private Endpoint changes, the DNS record(s) are automatically updated by the platform for us.

### VNET Peering

If you need to connect to PaaS services via Private Endpoint from a different VNET than the one where the Private Endpoint NICs live, you can take advantage of network peering and you should be able to communicate without any problems. VNET Peering is very flexible and can peer networks that are in the same regions as well as different regions, subscriptions or even tenants.

To be able to resolve the hostname to the private IP of the NIC created by the Private Endpoints, we need to make sure that the private DNS Zone is linked to all the VNETs that have to connect to the PaaS service.

> [!NOTE]
> Bear in mind that network peering is not transitive, so if you need to traverse several networks, you need to configure a network virtual appliance (NVA) that knows how to route traffic

All three components briefly described above are used to configure private access to PaaS services and GitHub-hosted runners can take advantage of this infrastructure. Let's see how below.

## Configuration

To get this working we need to configure the networking in Azure, create the hosted network configuration in GitHub, create runner groups and runners in GitHub and, as the last step, we can change the workflow's `runs-on` to specify the new runner name, let see how to do this in detail.

### Azure configuration

In Azure, you have to decide in which VNET the GitHub-hosted runner NICs will be created. You can use the same network where the private endpoint for your PaaS services lives or create another VNET to keep things separate, this decision is up to you and depends on your networking configuration/requirements. What's worth noting is that just a subset of Azure regions are supported, so you may be forced to create a VNET in a region different from the VNET that contains your Private Endpoints.

As of today (July 2024) the only supported regions are:

<div style="column-count: 2">

- EastUs
- EastUs2
- WestUs2
- WestUs3
- CentralUs
- NorthCentralUs
- SouthCentralUs
- AustraliaEast
- JapanEast
- FranceCentral
- GermanyWestCentral
- NorthEurope
- NorwayEast
- SwedenCentral
- SwitzerlandNorth
- UkSouth
- SoutheastAsia

</div>

If the VNET that contains the Private Endpoints is not in any of those regions, you have to create a new VNET and use Regional VNET Peerings or create another set of Private Endpoints in the designed VNET. If you use a HUB/Spoke network topology, you may want to create a dedicated spoke that will host the GitHub NICs.

When you have multiple spokes that need to communicate, you can either peer them together, configure traffic routing through an NVA or connect them through a VPN gateway. Please refer to the [documentation](https://learn.microsoft.com/en-us/azure/architecture/networking/guide/spoke-to-spoke-networking) on how to achieve that.

In my case, I went with the easy option to use network peering between the two spoke VNETs.

After the networking part has been taken care of, we need to:

1. Register a new resource provider
1. Create a new resource of type GitHub.Network/networkSettings
1. Copy the tag.GithubId output

Register the resource provider can be done in several ways, via Terraform (see [below](#creating-the-github-network-setting-resource)) or via az cli running the following command:

```sh
az provider register --namespace GitHub.Network
```

In GitHub, depending on whether you have an Enterprise Cloud or Team Plan, you can configure the **Hosted Compute Networking** on the Enterprise level or at the organization level. If you have GitHub Enterprise Cloud, you can still select whether to configure it at the Enterprise or Organization level.

The GitHub network settings need to know about your Enterprise/Organization so, before creating the network settings resource in Azure, we need to get a hold of the Enterprise ID/Organization ID from GitHub. As far as I know, this is not displayed anywhere in the UI so we need to execute a GraphQL API call as shown below:

### GitHub Enterprise ID

We get the organization ID via the following GraphQL call, before the call we also need to generate a personal access token with the required grants.

```sh
curl -H "Authorization: Bearer BEARER_TOKEN" -X POST \
  -d '{ "query": "query($slug: String!) { enterprise (slug: $slug) { slug databaseId } }" ,
        "variables": {
          "slug": "ENTERPRISE_SLUG"
        }
      }' \
https://api.github.com/graphql
```

> [!TIP]
> The documentation for configuring the private networking for GitHub-hosted runners in your **Enterprise** can be found [here](https://docs.github.com/en/enterprise-cloud@latest/admin/configuring-settings/configuring-private-networking-for-hosted-compute-products/configuring-private-networking-for-github-hosted-runners-in-your-enterprise)

### GitHub Organization ID

```sh
curl -H "Authorization: Bearer BEARER_TOKEN" -X POST \
  -d '{ "query": "query($login: String!) { organization (login: $login) { login databaseId } }" ,
        "variables": {
          "login": "ORGANIZATION_LOGIN"
        }
      }' \
https://api.github.com/graphql
```

> [!TIP]
> The documentation for configuring private networking for GitHub-hosted runners in your **Organization** can be found [here](https://docs.github.com/en/organizations/managing-organization-settings/configuring-private-networking-for-github-hosted-runners-in-your-organization)

### Create the GitHub network setting

Here's the Terraform code to create the GitHub network settings resource in Azure:

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = ">3.0.0"
    }

    azapi = {
      source  = "Azure/azapi"
      version = "~> 1.14.0"
    }
  }
}

# Register the GitHub.Network resource provider
resource "azurerm_resource_provider_registration" "github_resource_provider" {
  name = "GitHub.Network"
}

resource "azurerm_resource_group" "resource_group" {
  location = "North Europe"
  name     = "My-Rg"
}

resource "azurerm_virtual_network" "vnet" {
  name                = "My-vnet"
  location            = azurerm_resource_group.resource_group.location
  resource_group_name = azurerm_resource_group.resource_group.name
  address_space       = "10.0.0.0/8"
}

resource "azurerm_subnet" "runner_subnet" {
  name                 = "My-runner-subnet"
  resource_group_name  = azurerm_resource_group.resource_group.name
  virtual_network_name = azurerm_virtual_network.vnet.name

  address_prefixes = "10.0.1.0/24"

  delegation {
    name = "delegation"

    service_delegation {
      name    = "GitHub.Network/networkSettings"
      actions = ["Microsoft.Network/virtualNetworks/subnets/join/action"]
    }
  }
}

# Create the GitHub Network settings
resource "azapi_resource" "github_network_settings" {
  type                      = "GitHub.Network/networkSettings@2024-04-02"
  name                      = "github_network_settings_resource"            # The name of the networksettings
  location                  = "West Europe"                                 # The region in which the networksetting resource will be created
  parent_id                 = azurerm_resource_group.resource_group.id      # Parent Id that should point to the ID of the resource group
  schema_validation_enabled = false
  body = jsonencode({
    properties = {
      businessId = var.github_business_id                                   # GitHub EnterpriseID or Organization ID based on Enterprise vs Organization level configuration
      subnetId   = azurerm_subnet.runner_subnet.id                          # ID of the subnet where the NICs will be injected
    }
  })
  response_export_values = ["tags.GitHubId"]                                # Export the tags.GitHubId

  lifecycle {
    ignore_changes = [tags]
  }
}

output "github_network_settings_id" {
    description = "ID of the GitHub.Network/networkSettings resource"
    value = jsondecode(azapi_resource.github_network_settings.output).gitHubId.value
}
```

> You can find the whole source code at the end of the article in the [references](#references) section

### GitHub configuration

I decided to create the Hosted Compute Networking configurations at the Organization levels because it is where it makes the most sense for my use case, but creating it at the Enterprise level is pretty much the same thing, so you can easily adapt this tutorial to it.

1. Go to your Organization Settings
1. In Hosted Compute Networking, create a new Network Configuration and pick Azure Private Networking
1. Add a name to the configuration and then click on the _Add Azure Virtual Network_ button
1. Paste the ID outputted by Terraform while creating the GitHub Network settings resource
1. Save the configuration

{{<figure src="network_settings.jpg" alt="GitHub network setting configuration dialog" caption="*GitHub network setting configuration dialog, image courtesy of https://github.com/garnertb/github-runner-vnet*" nozoom=true >}}

After the network configuration is created, we have to create a Runner Group that uses the network configuration just created.

1. Go to Organization settings > Actions > Runner Groups
1. Give the Runner Group a name
1. In the network configuration dropdown, select the network configuration created in the previous step
1. Save the Runner group

After the Runner Group has been created, it's now time to create a runner (or more) within the Runner Group

1. Click on the Runner Group just created
1. Click on the New runner > New GitHub-hosted runner
1. Specify name, OS, Image (OS Version) and Specs (Size)
1. Make sure the Runner Group is the previously created Runner Group
1. Save the runner

Now that we have configured everything, we can change the `runs-on` label on a workflow with the name of one of the runners created above and it will use the new runner that has VNET connectivity with our PaaS service.

```yml
name: Sample workflow

on:
  pull_request:

jobs:
  build:
    name: Build
    runs-on: azure-vnet-runner # Use the name of the runner just created
```

## References

- [John Savill Technical training - Private Endpoints](https://www.youtube.com/watch?v=57ZwdztCx2w&ab_channel=JohnSavill%27sTechnicalTraining)
- [GitHub docs - Organization private networking](https://docs.github.com/en/organizations/managing-organization-settings/about-azure-private-networking-for-github-hosted-runners-in-your-organization)
- [GitHub docs - Enterprise private networking](ttps://docs.github.com/en/enterprise-cloud@latest/admin/configuring-settings/configuring-private-networking-for-hosted-compute-products/configuring-private-networking-for-github-hosted-runners-in-your-enterprise)
- [Example code](https://github.com/garnertb/github-runner-vnet)

## Conclusion

Thanks to GitHub private networking for hosted runners, we can ensure our CI/CD pipeline works seamlessly even when we deny public access to the PaaS services we use, allowing us to enhance the security posture of our Azure Subscription.

In this repository, I have a simple Terraform configuration where I deploy an Azure Storage Account, disable Public Access, and create the Private Endpoints for blob and tables within a VNET. In another VNET I have configured the GitHub network settings, the output of such configuration is the token that you can input in GitHub when creating the Hosted Compute Networking configuration.

I hope you enjoyed this article, if you have some questions, don't hesitate to reach out.
Till the next time!
