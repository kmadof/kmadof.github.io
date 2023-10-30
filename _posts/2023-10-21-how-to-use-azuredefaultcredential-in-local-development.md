---
layout: post
title: How to use AzureDefaultCredential in local development?
date: 2023-10-21
excerpt_separator:  <!--more-->
featured_image: /assets/images/posts/2023/laptop-with-charts.jpeg
tags: .NET Azure
featured: false
hidden: false
---

The `DefaultAzureCredential` class is a part of the `Azure.Identity` namespace and is included in the [Azure.Identity](https://www.nuget.org/packages/Azure.Identity) package. It provides a default `TokenCredential` authentication flow for applications that will be deployed to Azure.

<!--more-->

### Multiple Authentication Methods

The `DefaultAzureCredential` class uses multiple identities for authentication. When an access token is needed, it requests one using these identities in turn, stopping when one provides a token. The following credential types, if enabled, will be tried in order:

- **EnvironmentCredential**: A service principal configured by environment variables.
- **WorkloadIdentityCredential**: If environment variable configuration is set by the Azure workload identity webhook.
- **ManagedIdentityCredential**: An Azure managed identity.
- **SharedTokenCacheCredential**: On Windows only, a user who has signed in with a Microsoft application, such as Visual Studio. (Deprecate in favor of VisualStudioCredential)
- **VisualStudioCredential**: The identity currently logged in to Visual Studio.
- **VisualStudioCodeCredential**: The identity currently logged in to Visual Studio Code.
- **AzureCliCredential**: The identity currently logged in to the Azure CLI.
- **AzurePowerShellCredential**: The identity currently logged in to Azure PowerShell.
- **AzureDeveloperCliCredential**: The identity currently logged in to the Azure Developer CLI.
- **InteractiveBrowserCredential**: Credentials requiring user interaction are not included by default. Callers must explicitly enable this when constructing the DefaultAzureCredential.

#### Token Acquisition

The `DefaultAzureCredential` class sequentially calls GetToken on all the included credentials in the order mentioned above, returning the first successfully obtained AccessToken. Acquired tokens are cached by the credential instance and token lifetime and refreshing is handled automatically.

#### Usage
The `DefaultAzureCredential` class is appropriate for most scenarios where the application is intended to ultimately be run in Azure. It combines credentials that are commonly used to authenticate when deployed, with credentials that are used to authenticate in a development environment.

This code:
    
```csharp
var credential = new DefaultAzureCredential();

var blobClient = new BlobClient(new Uri("https://myaccount.blob.core.windows.net/mycontainer/myblob"), credential);
```

When this code is deployed to Azure, the `DefaultAzureCredential` class will authenticate using the managed identity of the resource it is deployed to. If you set `AZURE_CLIENT_ID` environment variable, it will authenticate using the managed identity configured by that environment variable. When this code is run locally, the `DefaultAzureCredential` class will authenticate using the developer's Azure Active Directory account. In my case it uses AzurCliCredential. But what happens when you run it in docker? 

#### Running in Docker

You can install Azure CLI on the image, but this means that you will get two types of images. And then you won't be able to use easily the same image in Azure and locally. So what you can do? You can use [EnvironmentCredential](https://learn.microsoft.com/en-us/dotnet/api/azure.identity.environmentcredential?view=azure-dotnet) class. 

```docker
version: '3.8'
services:
  api:
    platform: linux/amd64
    build:
      context: .
      dockerfile: src/Service.Api/Dockerfile
      args:
        - FEED_ACCESSTOKEN=${FEED_ACCESSTOKEN}
    container_name: serviceapi
    env_file:
        - ./development-variables.env
    environment:
        - AZURE_TENANT_ID=${TENANT_ID}
        - AZURE_CLIENT_ID=${AZURE_CLIENT_ID}
        - AZURE_CLIENT_SECRET=${AZURE_CLIENT_SECRET}
    ports:
      - "5187:80"
```

In order to use this credential you need to create service principal and set environment variables. You can do this using Azure CLI:

```bash
az ad sp create-for-rbac -n LocalDevelopment --role Contributor --scopes /subscriptions/SubId
```

Please make sure you set proper as low as possible credentials for this service principal.

### References

- [DefaultAzureCredential Class](https://learn.microsoft.com/en-us/dotnet/api/azure.identity.defaultazurecredential?view=azure-dotnet)
- [https://github.com/Azure-Samples/azure-devops-terraform-oidc-ci-cd](https://github.com/Azure/azure-sdk-for-net/issues/19167)
- [Azure SDK: What’s new in the Azure Identity August 2020 General Availability Release](https://devblogs.microsoft.com/azure-sdk/azure-identity-august-2020-ga/)