# GCP IAM Privilege Escalation Path Enumeration

## Challenge Overview

### Objective

Identify the privilege escalation path within a Google Cloud Platform (GCP) environment by enumerating IAM roles, custom roles, and Service Account permissions. The objective is to determine how an attacker could move from a compromised low-privileged Service Account to a more privileged identity.

### Scenario

An attacker obtains the JSON key of a compromised Service Account (`testing-service-account`) inside a GCP project. The task is to enumerate the cloud environment, inspect IAM configurations, identify custom roles, and discover Service Accounts that possess administrative permissions over other identities.

Although no actual privilege escalation is performed during the challenge, the objective is to identify the attack path that would enable one.

### Skills Practiced

- GCP IAM Enumeration
- Service Account Analysis
- Custom Role Enumeration
- IAM Policy Analysis
- Privilege Escalation Path Discovery

---

# Environment

## Cloud Provider

Google Cloud Platform (GCP)

## Initial Access

A leaked Service Account JSON key:

- Service Account Private Key
- Client Email
- Project ID

This allowed authentication using Google Cloud SDK.

## Available Tools

- Google Cloud SDK (gcloud)
- IAM API
- Cloud Storage API

---

# Attack Methodology

## Initial Hypothesis

The provided Service Account was expected to have limited permissions. The primary objective was to enumerate IAM objects and identify any relationships that could lead to privilege escalation.

Expected targets:

- Custom IAM Roles
- Service Accounts
- IAM Bindings
- Cloud Storage Buckets

---

# Attack Workflow

The attack was performed by gradually expanding knowledge of the environment and identifying privilege escalation paths.

## Step 1 - Initial Access Validation

### Action

Authenticate using the leaked Service Account credentials and verify access.

Command:

```bash
gcloud auth activate-service-account \
    --key-file service-account.json

gcloud auth list
```

Result:

```text
Active Account:
testing-service-account@woven-acolyte-428406-v9.iam.gserviceaccount.com
```

Observation:

The leaked key was valid and provided authenticated access to the target GCP project.

---

## Step 2 - Enumerate IAM Configuration

### Goal

Discover project-level IAM bindings and determine which roles are assigned to the compromised Service Account.

Command:

```bash
gcloud projects get-iam-policy woven-acolyte-428406-v9
```

Findings:

- The Service Account had several built-in viewer roles.
- A custom role named `customViewerRole1` was assigned.
- Multiple additional Service Accounts existed inside the project.

Next, enumerate the custom role:

```bash
gcloud iam roles describe customViewerRole1 \
    --project woven-acolyte-428406-v9
```

Result:

```text
title: Custom Viewer Role1

includedPermissions:
- storage.buckets.list
- storage.objects.get
```

Observation:

The custom role only granted read-only access to Cloud Storage resources.

---

## Step 3 - Enumerate Service Accounts

### Goal

Identify high-value identities inside the project.

Command:

```bash
gcloud iam service-accounts list
```

Result:

```text
testing-service-account
prod-service-account
devops-service-account
resource-mgmt
secret-mgmt-sa
log-reviewer-sa
...
```

Observation:

Several administrative-looking Service Accounts were present, suggesting additional privilege boundaries inside the project.

---

## Step 4 - Identify Privilege Escalation Path

### Misconfiguration Identified

Inspect the IAM policy attached directly to the DevOps Service Account.

Command:

```bash
gcloud iam service-accounts get-iam-policy \
devops-service-account@woven-acolyte-428406-v9.iam.gserviceaccount.com
```

Result:

```yaml
bindings:

- members:
    - serviceAccount:prod-service-account@woven-acolyte-428406-v9.iam.gserviceaccount.com
  role: roles/iam.serviceAccountAdmin

- members:
    - serviceAccount:prod-service-account@woven-acolyte-428406-v9.iam.gserviceaccount.com
  role: roles/iam.serviceAccountKeyAdmin
```

Observation:

The `prod-service-account` possesses administrative permissions over the DevOps Service Account.

This represents a potential privilege escalation path because an attacker who compromises the production Service Account could:

- Manage the DevOps Service Account.
- Create or rotate Service Account keys.
- Potentially impersonate or take control of the DevOps identity.

---

## Step 5 - Enumerate Accessible Storage Resources

### Target

Cloud Storage Buckets

Command:

```bash
gcloud storage buckets list
```

Result:

```text
secret-bucket-woven-acolyte-428406-v9
production-v545965
woven-acolyte-428406-v9_cloudbuild
...
```

Observation:

The compromised identity had read access to enumerate storage buckets, confirming the permissions granted by the custom viewer role.

---

# Attack Chain

```text
Leaked Service Account Key
        |
        v
Authenticate to GCP
        |
        v
Enumerate IAM Policies
        |
        v
Identify Custom Role
        |
        v
Enumerate Service Accounts
        |
        v
Inspect Service Account IAM Policies
        |
        v
Identify Privilege Escalation Relationship
```

---

# Findings

## Security Weakness

A privileged Service Account (`prod-service-account`) was granted administrative control over another Service Account (`devops-service-account`).

Although the initial compromised identity could not directly exploit this relationship, identifying these trust relationships is a critical step during cloud privilege escalation assessments.

## Impact

If the production Service Account were compromised, an attacker could administer the DevOps Service Account, potentially creating new keys or abusing impersonation to expand access within the environment.

This demonstrates how Service Account relationships can become lateral movement opportunities even when the initially compromised identity has limited privileges.

---

# Evidence

Important commands used during the assessment:

```bash
gcloud auth list

gcloud projects get-iam-policy woven-acolyte-428406-v9

gcloud iam roles describe customViewerRole1 \
    --project woven-acolyte-428406-v9

gcloud iam service-accounts list

gcloud iam service-accounts get-iam-policy \
devops-service-account@woven-acolyte-428406-v9.iam.gserviceaccount.com

gcloud storage buckets list
```

---

# Defensive Recommendations

- Apply the principle of least privilege to all Service Accounts.
- Regularly audit Service Account IAM policies.
- Avoid granting `roles/iam.serviceAccountAdmin` unless operationally necessary.
- Restrict the ability to create Service Account keys.
- Periodically review custom IAM roles and remove unnecessary permissions.
- Detect unusual IAM enumeration and Service Account administration activities through Cloud Audit Logs.

---

# Key Takeaways

- Cloud privilege escalation often begins with **enumeration**, not exploitation.
- Service Account IAM policies are as important as project-level IAM bindings.
- Custom roles should always be inspected to understand the actual permissions they grant.
- Mapping trust relationships between Service Accounts is a fundamental Red Team technique in GCP.
- Not every challenge ends with obtaining administrator access; identifying a viable privilege escalation path is often the intended objective.

---

# References

- https://cloud.google.com/iam/docs
- https://cloud.google.com/iam/docs/service-account-overview
- https://cloud.google.com/storage/docs/access-control/iam
- https://cloud.google.com/sdk/gcloud
