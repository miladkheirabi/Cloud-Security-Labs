# Azure App Registration API Permission Enumeration

## Challenge Overview

**Objective:**

Identify the API Permission assigned to an Azure App Registration and determine potential security risks caused by excessive application privileges.

The challenge simulates a Red Team scenario where an attacker has obtained Application Credentials:

* Client ID
* Client Secret
* Organization Domain

The goal is to authenticate against Azure, enumerate the application's permissions, and identify the assigned API Permission.

---

## Scenario

Secure Corp has configured applications, Service Principals, API Permissions, and RBAC roles in Azure.

A security audit identified possible permission misconfigurations in App Registrations.

As a Red Team operator, the task is to investigate the application and identify the permission assigned to it.

---

## Initial Access

Provided:

```
Client ID
Client Secret
Organization Domain
```

Missing:

```
Tenant ID
```

The first step is discovering the Azure Tenant ID.

---

# Step 1 - Discover Tenant ID

Azure authentication requires:

```
Tenant ID
Client ID
Client Secret
```

The tenant ID can be obtained from the organization's domain information.

Example:

```
secure-corp.org
```

Result:

```
Tenant ID:
f2a33211-e46a-4c92-b84d-aff06c2cd13f
```

---

# Step 2 - Obtain Azure Access Token

Authenticate using the Client Credentials flow.

Request:

```bash
curl -X POST https://login.microsoftonline.com/TENANT_ID/oauth2/v2.0/token -H "Content-Type: application/x-www-form-urlencoded" -d "client_id=CLIENT_ID&client_secret=CLIENT_SECRET&grant_type=client_credentials&scope=https://graph.microsoft.com/.default"
```

Response:

```json
{
  "token_type": "Bearer",
  "access_token": "ACCESS_TOKEN"
}
```

The access token will be used to communicate with Microsoft Graph API.

---

# Step 3 - Analyze JWT Token

Decode the received JWT token.

Important fields:

```json
{
  "app_displayname": "dev-app",
  "appid": "caaa28c5-b8da-4d29-b42e-95b1aba6b81c",
  "roles": [
    "Application.Read.All"
  ]
}
```

The `roles` claim reveals the granted Application Permission.

---

# Step 4 - Enumerate Service Principal

Find the Service Principal associated with the application.

Request:

```bash
curl -X GET "https://graph.microsoft.com/v1.0/servicePrincipals?$filter=appId eq 'APPLICATION_ID'" -H "Authorization: Bearer ACCESS_TOKEN"
```

Result:

```json
{
  "id": "d6bc58ed-11f6-47cd-a168-e4d606c5b22a",
  "displayName": "dev-app",
  "appId": "caaa28c5-b8da-4d29-b42e-95b1aba6b81c"
}
```

---

# Step 5 - Enumerate App Role Assignments

Query the assigned API permissions:

```bash
curl -X GET https://graph.microsoft.com/v1.0/servicePrincipals/SERVICE_PRINCIPAL_ID/appRoleAssignments -H "Authorization: Bearer ACCESS_TOKEN"
```

Output:

```json
{
  "value": [
    {
      "appRoleId": "9a5d68dd-52b0-4cc2-bd40-abcf44ac3a30",
      "resourceDisplayName": "Microsoft Graph",
      "principalDisplayName": "dev-app"
    }
  ]
}
```

The application has a Microsoft Graph Application Permission assigned.

---

# Step 6 - Resolve API Permission Name

Find the matching permission from Microsoft Graph service principal roles.

The identified role ID:

```
9a5d68dd-52b0-4cc2-bd40-abcf44ac3a30
```

maps to:

```
Application.Read.All
```

---

# Flag

```
Application.Read.All
```

---

# Attack Flow Summary

```
Application Credentials
          |
          v
Discover Tenant ID
          |
          v
Request OAuth Token
          |
          v
Microsoft Graph API Access
          |
          v
Enumerate Service Principal
          |
          v
Check App Role Assignments
          |
          v
Identify API Permission
```

---

# Security Impact

`Application.Read.All` is an Application Permission that allows an application to read application objects in Microsoft Entra ID.

Granting powerful application permissions increases the impact of credential compromise because an attacker with the Client Secret can authenticate as the application and access permitted resources without user interaction.

Recommended mitigations:

* Follow least privilege principles.
* Remove unused API permissions.
* Regularly review App Registrations.
* Monitor Service Principal activity.
* Rotate application secrets.
* Prefer certificate-based authentication over long-lived client secrets where possible.

---

# Lessons Learned

* Application credentials can provide direct access to Azure resources.
* JWT tokens contain valuable information about granted permissions.
* Service Principals are the identity objects used by applications inside a tenant.
* API permissions can be enumerated through Microsoft Graph.
* Excessive Application Permissions can create significant security risks.
