---
title: "Configuring SSO and RBAC with Azure Entra ID on ArgoCD via Terraform"
date: 2025-04-16T15:29:39+02:00
draft: true
tags: [azure, kubernetes, terraform]
---

ArgoCD is a fantastic tool to manage your AKS cluster using a GitOps approach. Configuring SSO and RBAC has been a bit more tricky than I initially anticipated, so I'm writing this short blog post to point out some of the issues I ran into and their solutions.

## Installation

{{<note>}}
I use Terraform to install ArgoCD in the AKS cluster, but as of recently (April 2025 at the time of writing), Azure offers a managed ArgoCD cluster extension.
Currently, it's available as a private preview. You can read about it in the announcement [blog post](https://techcommunity.microsoft.com/blog/azurearcblog/announcing-private-preview-argocd-through-microsoft-gitops/4399747).
{{</note>}}

You can install ArgoCD using the community maintained Helm Chart defined [here](https://github.com/argoproj/argo-helm).

Let's look at the terraform code to install the helm chart:

```terraform
locals {
  argocd_domain = "your-domain-here"
  az_tenant_id  = "your-tenant-id-here"
  argocd_admins = [
    "List of UPNs of users to be added to the ArgoCD Admin group."
  ]
}

resource "kubernetes_namespace_v1" "argocd" {
  metadata {
    name = "argocd-system"
  }
}

## Install the argocd Helm Release
resource "helm_release" "argocd" {
  depends_on       = [kubernetes_namespace_v1.argocd]
  chart            = "argo-cd"
  name             = "argocd-release"
  namespace        = kubernetes_namespace_v1.argocd.metadata[0].name
  repository       = "https://argoproj.github.io/argo-helm"
  version          = "7.8.26" # Latest version at the time of writing, make sure you pick the latest version here
  create_namespace = false

  values = [
    templatefile("templates/argocd-values.yaml", {
      domain           = local.argocd_domain
      tenant           = local.az_tenant_id
      clientId         = azuread_application_registration.argocd_app_registration.client_id
      clientSecret     = azuread_application_password.argocd_client_secret.value
      adminGroupId     = azuread_group.argocd_admin_group.object_id
    })
  ]
}

data "azuread_application_published_app_ids" "well_known" {}

data "azuread_service_principal" "msgraph" {
  client_id = data.azuread_application_published_app_ids.well_known.result["MicrosoftGraph"]
}

## Create the ArgoCD Entra ID Application Registration
resource "azuread_application_registration" "argocd_app_registration" {
  display_name            = "argocd-app-registration"
  sign_in_audience        = "AzureADMyOrg"
  homepage_url            = "https://${local.argocd_domain}"
  description             = "ArgoCD App Registration used to configure SSO"
  group_membership_claims = ["ApplicationGroup"]
}

## Configure the ArgoCD Entra ID Application MSGraph API Access
resource "azuread_application_api_access" "argocd_required_access_msgraph" {
  application_id = azuread_application_registration.argocd_app_registration.id
  api_client_id  = data.azuread_application_published_app_ids.well_known.result["MicrosoftGraph"]

  scope_ids = [
    data.azuread_service_principal.msgraph.oauth2_permission_scope_ids["User.Read"],
    data.azuread_service_principal.msgraph.oauth2_permission_scope_ids["profile"],
    data.azuread_service_principal.msgraph.oauth2_permission_scope_ids["email"],
  ]
}

## Configure the ArgoCD Entra ID Application custom claims, used for RBAC and the username
resource "azuread_application_optional_claims" "argocd_group_id_claim" {
  application_id = azuread_application_registration.argocd_app_registration.id

  id_token {
    name      = "groups"
    essential = true
  }

  id_token {
    name      = "email"
    essential = true
  }
}

## Configure the ArgoCD Entra ID Application redirect URIs for the web UI
resource "azuread_application_redirect_uris" "argocd_web_ui_redirect_uris" {
  application_id = azuread_application_registration.argocd_app_registration.id
  type           = "Web"

  redirect_uris = [
    "https://${local.argocd_domain}/auth/callback",
  ]
}

## Configure the ArgoCD Entra ID Application redirect URIs for the CLI
resource "azuread_application_redirect_uris" "argocd_cli_redirect_uris" {
  application_id = azuread_application_registration.argocd_app_registration.id
  type           = "PublicClient"

  redirect_uris = [
    "http://localhost:8085/auth/callback",
  ]
}

## Generate the ArgoCD Entra ID Application client secret
resource "azuread_application_password" "argocd_client_secret" {
  application_id = azuread_application_registration.argocd_app_registration.id
  display_name   = "ArgoCD client secret, used for the web UI Azure SSO authentication"
}

## Create the ArgoCD Entra ID Service Principal and make sure only the groups assigned to the application can login to ArgoCD.
resource "azuread_service_principal" "argocd_service_principal" {
  client_id                    = azuread_application_registration.argocd_app_registration.client_id
  app_role_assignment_required = true
}

## Create the Entra ID group for ArgoCD Admins
resource "azuread_group" "argocd_admin_group" {
  display_name     = "ArgoCD Admins"
  description      = "Members of this group have Admin privileges in ArgoCD"
  security_enabled = true
}

## Read all the allowed members of the ArgoCD Admin group
data "azuread_user" "argocd_admins" {
  user_principal_name = each.key
  for_each            = local.argocd_admins
}

## Assign users to the ArgoCD Admin group
resource "azuread_group_member" "argocd_admin_members" {
  group_object_id  = azuread_group.argocd_admin_group.object_id
  member_object_id = each.value.object_id

  for_each = data.azuread_user.argocd_admins
}

## Assign the ArgoCD Admin group to the ArgoCD Service Principal
resource "azuread_app_role_assignment" "argocd_application_group_assignment" {
  app_role_id         = "00000000-0000-0000-0000-000000000000"
  principal_object_id = azuread_group.argocd_admin_group.object_id
  resource_object_id  = azuread_service_principal.argocd_service_principal.object_id
}
```

I added some comments to the code to explain what it does. Here we're creating an Entra ID App Registration and configuring it according to the ArgoCD documentation that you can find [here](https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/microsoft/#entra-id-app-registration-auth-using-oidc).

The locals need to be configured with the following values:

- **argocd_domain** The domain where ArgoCD will be exposed, e.g. argocd.mycompany.com
- **az_tenant_id** The tenant of your Azure account. You can get it via the CLI using `az account show --query tenantId` after logging into the az cli.
- **argocd_admins** The list of user principal names that will be added granted administrative permissions in ArgoCD.

{{<tip>}}
You can of course add multiple groups, just repeat the same process used for the admin one and update the `policy.csv` accordingly.
{{</tip>}}

Let's now look at the values file template:

```yaml
global:
  domain: ${domain}

configs:
  cm:
    url: https://${domain}
    admin.enabled: false
    users.session.duration: "8h"
    oidc.config : |
      name: Azure
      issuer: https://login.microsoftonline.com/${tenant}/v2.0
      clientID: ${clientId}
      clientSecret: ${clientSecret}
      requestedIDTokenClaims:
        groups:
          essential: true
      requestedScopes:
        - openid
        - profile
        - email
  rbac:
    policy.default: role:readonly
    policy.csv: |
      p, role:authenticated, projects, get, *, allow
      g, "${adminGroupId}", role:admin
    scopes: '[groups]'

server:
  ingress:
    enabled: true
    tls: true
    annotations:
      kubernetes.io/ingress.class: azure/application-gateway
      cert-manager.io/cluster-issuer: letsencrypt-production
      cert-manager.io/acme-challenge-type: dns01
      appgw.ingress.kubernetes.io/use-private-ip: true
      appgw.ingress.kubernetes.io/cookie-based-affinity: true
      appgw.ingress.kubernetes.io/ssl-redirect: true
      external-dns.alpha.kubernetes.io/internal-hostname: ${domain}
```

Here we configure the Helm chart to do a few different things. Let's have a look at each one of them:

## Configuration

```yaml
oidc.config : |
  name: Azure
  issuer: https://login.microsoftonline.com/${tenant}/v2.0
  clientID: ${clientId}
  clientSecret: ${clientSecret}
  requestedIDTokenClaims:
    groups:
      essential: true
  requestedScopes:
  - openid
  - profile
  - email
```

This snippet configures the Azure SSO using the generated App Registration Client ID and Client Secret; moreover, we specify which OIDC scopes the application should request.

```yaml
admin.enabled: false
```

Disable local user authentication so that only login via Azure will be available.

```yaml
rbac:
  policy.default: role:readonly
  policy.csv: |
    p, role:authenticated, projects, get, *, allow
    g, "${adminGroupId}", role:admin
  scopes: '[groups]'
```

Here we define that every user who can log in and is not part of the admin group will be assigned the built-in readonly role, while the user members of the group `${adminGroupId}` will be part of the built-in ArgoCD admin group and have full access to all ArgoCD features.

{{<tip>}}
You can define multiple groups and assign users to them following
{{</tip>}}

```yaml
server:
  ingress:
    enabled: true
    tls: true
    annotations:
      kubernetes.io/ingress.class: azure/application-gateway
      cert-manager.io/cluster-issuer: letsencrypt-production
      cert-manager.io/acme-challenge-type: dns01
      appgw.ingress.kubernetes.io/use-private-ip: true
      appgw.ingress.kubernetes.io/cookie-based-affinity: true
      appgw.ingress.kubernetes.io/ssl-redirect: true
      external-dns.alpha.kubernetes.io/internal-hostname: ${domain}
```

Here we configure the ingress to use application gateway ingress (you may use a different ingress class) and configure `external-dns` and `cert-manager` to automatically get a TLS certificate and update the DNS record.

{{<note>}}
The configuration of `external-dns` and `cert-manager` is outside the scope of this article, so it's a prerequisite. If you don't use either of these two tools, you can skip all the relative annotations.
{{</note>}}

## Troubleshooting

As stated at the beginning of the article, I ran into some issues and I want to document the solutions I found.

### Empty Initiated By

When you synchronize an application, ArgoCD adds an entry in the application history, this can be viewed using the `HISTORY AND ROLLBACK` button from within the application UI. If there's no value in the `Initiated By`, it's most likely because the generated id_token doesn't contain the email claim.
You can check if your user details by clicking the `User Info` section, make sure the `Username` field is populated.

To fix this, you can:

- Make sure you have added the `email` claim in the optional claims configuration for the App Registration, i.e.

  ```terraform
  id_token {
    name      = "email"
    essential = true
  }
  ```

- Make sure you grant the MSGraph `email` permission scope to the App Registration, i.e.

  ```terraform
  scope_ids = [
    data.azuread_service_principal.msgraph.oauth2_permission_scope_ids["User.Read"],
    data.azuread_service_principal.msgraph.oauth2_permission_scope_ids["profile"],
    data.azuread_service_principal.msgraph.oauth2_permission_scope_ids["email"], # <-- This grants the permission to read the email
  ]
  ```

- Make sure the email field is actually filled in in Entra ID.

You can debug this using a tool like Postman or Insomnia to request a token from Entra ID and then you can inspect the content of the `id_token` with your favorite tool, or via jwt.io.

It's worth pointing out also that ArgoCD, using OIDC is not super flexible, and the claim names ArgoCD expects to find in the `id_token` are hardcoded (as you can see from the ArgoClaims [here](https://github.com/argoproj/argo-cd/blob/master/util/claims/claims.go))

I also opened an enhancement [proposal](https://github.com/argoproj/argo-cd/issues/22688) to allow mapping incoming claims (the one from the token issued by Entra Id) to argo claims.

### Empty UI when logging in

Due to an error I had in the `policy.csv` file, despite logging in with an administrator account, I had an empty UI and every operation I tried resulted in a permission error.

This was ultimately fixed by making sure the `policy.default` maps to either a built-in role or an existing one defined in the `policy.csv`. This took me a bit to realize because it's very difficult to figure out what role has been assigned to you in the ArgoCD UI (if you know a way, please let me know)

Something I found helpful was bumping the logging level of ArgoCD via the following change:

```yaml
global:
  domain: ${domain}
  logging:
    level: debug  
```

Subsequently, I was looking at the argocd-server pod logs and found these two messages helpful to debug and resolve the issue:

```log
time="2025-04-16T09:47:03Z" level=debug msg="enforce failed" claims="<redatcted>" groups="<redacted>" project=dev rval="<redacted>" scopes="[groups]" subject=<redacted>
time="2025-04-16T09:47:03Z" level=warning msg="finished unary call with code PermissionDenied" error="rpc error: code = PermissionDenied desc = permission denied" grpc.code=PermissionDenied grpc.method=ListApps grpc.service=repository.RepositoryService grpc.start_time="2025-04-16T09:47:03Z" grpc.time_ms=3.037 span.kind=server system=grpc
```

## Conclusion

ArgoCD SSO OIDC configuration with Azure is quite easy to set up via Terraform. I'm curious to look into the managed version that's been announced recently (see link at the top of the post).

A possible improvement over the current configuration is to use workload identities to avoid a shared secret, but that will be the subject of a future post, I guess.

That's all for now.
