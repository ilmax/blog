---
title: "Import Certificate to Azure Key Vault via PowerShell"
date: 2025-08-28T12:46:07+02:00
draft: false
tags: ["Azure", "powershell", "gist"]
---

Ever found yourself scratching your head, trying to get a freshly minted PFX TLS certificate into Azure Key Vault with PowerShell? I did—and after wading through sparse documentation, I decided to share my solution so you don’t have to!

This script grabs an access token from your Windows VM using a user-assigned identity (but you can easily tweak it for managed service identity). It’s quick, practical, and gets the job done.

Ready to see how it works? Let’s dive in:

```ps
function Store-CertificateInKeyvault {
    param(
        [Parameter(Mandatory = $true)][string]$KeyVaultName,
        [Parameter(Mandatory = $true)][string]$CertificateName,
        [Parameter(Mandatory = $true)][string]$PfxFilePath,
        [Parameter(Mandatory = $true)][string]$PfxPassword,
        [Parameter(Mandatory = $true)][string]$UamiClientId
    )

    $tokenResponse = Invoke-RestMethod -Uri "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2019-08-01&resource=https://vault.azure.net&client_id=$UamiClientId" -Method GET -Headers @{ Metadata = "true" }
    $accessToken = $tokenResponse.access_token

    $certContent = Get-Content -Path $PfxFilePath -Raw -Encoding Byte
    $base64Cert = [Convert]::ToBase64String($certContent)

    # Create the proper JSON body structure for Key Vault certificate import
    # for a PEM format, the content type should be different
    $requestBody = @{
        value = $base64Cert
        pwd = $PfxPassword
        policy = @{
            secret_props = @{
                contentType = "application/x-pkcs12"
            }
        }
    } | ConvertTo-Json -Depth 3

    $secretUrl = "https://$KeyVaultName.vault.azure.net/certificates/$($CertificateName)/import?api-version=7.1"  # pragma: allowlist secret
    $secretResponse = Invoke-RestMethod -Uri $secretUrl -Method POST -Headers @{
        Authorization = "Bearer $accessToken"
        "Content-Type" = "application/json"
    } -Body $requestBody

    Write-Host "Stored certificate '$CertificateName' in Key Vault '$KeyVaultName'."
}
```

Here below you can see the invocation of such script:

```ps
$keyVaultName = 'name of the key vault instance'
$certificateName = 'name of the certificate that will be imported'
$certPath = 'path of the pfx file on the file system'
$uamiClientId = 'client id of the user assigned managed identity'


Store-CertificateInKeyvault -KeyVaultName $keyVaultName -CertificateName $certificateName -PfxFilePath $certPath -PfxPassword $pfxPassword -UamiClientId $uamiClientId
```

Till the next time!
