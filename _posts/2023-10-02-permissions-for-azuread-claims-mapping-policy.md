---
layout: post
title: Permissions needed to create azuread_claims_mapping_policy
date: 2023-10-01
excerpt_separator:  <!--more-->
featured_image: /assets/images/posts/2023/azure-active-directory-ai-image.jpeg
tags: DevOps Azure Terraform
featured: true
hidden: true
---

If you try to create `azuread_claims_mapping_policy` using Terraform, you might encounter an error like this:

```
│ Error: retrieving Claims Mapping Policy with object ID: "<GUID HERE>"
│ 
│   with azuread_claims_mapping_policy.integration_claims_mapping[0],
│   on active_directory.tf line 54, in resource "azuread_claims_mapping_policy" "integration_claims_mapping":
│   54: resource "azuread_claims_mapping_policy" "integration_claims_mapping" {
│ 
│ ClaimsMappingPolicyClient.BaseClient.Get(): unexpected status 403 with
│ OData error: Authorization_RequestDenied: Insufficient privileges to
│ complete the operation.
```

<!--more-->

The error message is quite clear. You don't have enough permissions to create `azuread_claims_mapping_policy`. But what permissions are needed? I found in [documentation](https://registry.terraform.io/providers/hashicorp/azuread/latest/docs/resources/claims_mapping_policy) that:

<div class="note-box">
  <p>
    When authenticated with a service principal, this resource requires the following application roles: Policy.ReadWrite.ApplicationConfiguration and Policy.Read.All
    <br>
    When authenticated with a user principal, this resource requires one of the following directory roles: Application Administrator or Global Administrator
  </p>
</div>

Which of roles you need depends on how you authenticate. If you use service principal, you need to have `Policy.ReadWrite.ApplicationConfiguration` and `Policy.Read.All` permissions. If you use user principal, you need to have `Application Administrator` or `Global Administrator` role. I use managed identity, so I need to have `Policy.ReadWrite.ApplicationConfiguration` and `Policy.Read.All` permissions.

But how to assign these permissions?

### Assigning permissions

For regular Service Principal it can be done via Portal, but not for Managed Idenity. There is no button to do this.

![Permissions tab of Managed Identity on Microsoft Entra](/assets/images/posts/2023/ms-entra-managed-identity-permissions.png)

We can do this using [Azure PowerShell](https://techcommunity.microsoft.com/t5/azure-integration-services-blog/grant-graph-api-permission-to-managed-identity-object/ba-p/2792127). I used this script:

```powershell
$TenantID = "provide the tenant ID"
$GraphAppId = "00000003-0000-0000-c000-000000000000" # Microsoft Graph App ID
$DisplayNameOfMSI = "Managed identity name"
$PermissionNames = @("Policy.ReadWrite.ApplicationConfiguration", "Policy.Read.All") 

# Install the module
Install-Module AzureAD

Connect-AzureAD -TenantId $TenantID

$MSI = (Get-AzureADServicePrincipal -Filter "displayName eq '$DisplayNameOfMSI'")
Start-Sleep -Seconds 10

$GraphServicePrincipal = Get-AzureADServicePrincipal -Filter "appId eq '$GraphAppId'"

foreach ($PermissionName in $PermissionNames) {
    $AppRole = $GraphServicePrincipal.AppRoles | Where-Object { $_.Value -eq $PermissionName -and $_.AllowedMemberTypes -contains "Application" }
    New-AzureAdServiceAppRoleAssignment -ObjectId $MSI.ObjectId -PrincipalId $MSI.ObjectId -ResourceId $GraphServicePrincipal.ObjectId -Id $AppRole.Id
}

Write-Output "Permissions added successfully."
```

Please make sure you use account with enough permissions to assign roles. I used Global Administrator account.

But even then I got the same error. After some [digging](https://github.com/hashicorp/terraform-provider-azuread/issues/804) I found that we miss one more permission `Application.ReadWrite.All`. So the final list of permissions is:

```powershell
$PermissionNames = @("Application.ReadWrite.All", "Policy.ReadWrite.ApplicationConfiguration", "Policy.Read.All") 
```

![Permissions tab of Managed Identity on Microsoft Entra](/assets/images/posts/2023/ms-entra-managed-identity-permissions-added.png)

After adding this permission, I was able to create `azuread_claims_mapping_policy` using Terraform.

### Summary

To create `azuread_claims_mapping_policy` using Terraform, you need to have `Policy.ReadWrite.ApplicationConfiguration`, `Policy.Read.All` and `Application.ReadWrite.All` permissions. You can assign these permissions using Azure PowerShell.