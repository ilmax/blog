---
title: "Fix az aks get-credentials Persistence Decryption Error"
date: 2025-06-02T15:46:42+02:00
draft: false
tags: ["Kubernetes", "Azure", "gist"]
---

Today I ran into an interesting error trying to connect to the AKS clusters. Upon running `az aks get-credentials` I was getting the following error:

```sh
Decryption failed: [WinError 87]  App developer may consider this guidance: https://github.com/AzureAD/microsoft-authentication-extensions-for-python/wiki/PersistenceDecryptionError
```

I wasn't able to find much documentation about this error online, so I decided to write down the steps that allowed me to overcome this issue:

```sh
az config set core.encrypt_token_cache=false
az account clear
az config set core.encrypt_token_cache=true
az login
```

That was it. After clearing the local account, I was able to log-in to the AKS cluster once again. Note that without disabling token cache encryption, running `az account clear` was failing with the same error.
