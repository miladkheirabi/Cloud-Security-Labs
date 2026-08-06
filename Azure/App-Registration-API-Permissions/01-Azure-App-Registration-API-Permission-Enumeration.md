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

Request:

```
https://login.microsoftonline.com/<DOMAIN>/.well-known/openid-configuration
```

Response:

```
{
  "token_endpoint":"https://login.microsoftonline.com/f2a33211-e46a-4c92-b84d-aff06c2cd13f/oauth2/token",
  "token_endpoint_auth_methods_supported":["client_secret_post","private_key_jwt","client_secret_basic"],
  "jwks_uri":"https://login.microsoftonline.com/common/discovery/keys",
  "response_modes_supported":["query","fragment","form_post"],
  "subject_types_supported":["pairwise"],"id_token_signing_alg_values_supported":["RS256"],
  "response_types_supported":["code","id_token","code id_token","token id_token","token"],
  "scopes_supported":["openid"],
  "issuer":"https://sts.windows.net/f2a33211-e46a-4c92-b84d-aff06c2cd13f/",
  "microsoft_multi_refresh_token":true,
  "authorization_endpoint":"https://login.microsoftonline.com/f2a33211-e46a-4c92-b84d-aff06c2cd13f/oauth2/authorize",
  "device_authorization_endpoint":"https://login.microsoftonline.com/f2a33211-e46a-4c92-b84d-aff06c2cd13f/oauth2/devicecode",
  "http_logout_supported":true,"frontchannel_logout_supported":true,
  "end_session_endpoint":"https://login.microsoftonline.com/f2a33211-e46a-4c92-b84d-aff06c2cd13f/oauth2/logout",
  "claims_supported":["sub","iss","cloud_instance_name","cloud_instance_host_name","cloud_graph_host_name","msgraph_host","aud","exp","iat","auth_time","acr","amr","nonce","email","given_name","family_name","nickname"],
  "check_session_iframe":"https://login.microsoftonline.com/f2a33211-e46a-4c92-b84d-aff06c2cd13f/oauth2/checksession",
  "userinfo_endpoint":"https://login.microsoftonline.com/f2a33211-e46a-4c92-b84d-aff06c2cd13f/openid/userinfo",
  "kerberos_endpoint":"https://login.microsoftonline.com/f2a33211-e46a-4c92-b84d-aff06c2cd13f/kerberos",
  "tenant_region_scope":"AS",
  "cloud_instance_name":"microsoftonline.com",
  "cloud_graph_host_name":"graph.windows.net",
  "msgraph_host":"graph.microsoft.com",
  "rbac_url":"https://pas.windows.net"}
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
