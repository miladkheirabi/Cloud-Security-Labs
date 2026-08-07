# GCP IAM Enumeration & Privilege Escalation Discovery

## Challenge Overview

### Objective

Enumerate a Google Cloud Platform (GCP) environment using the credentials of a compromised Service Account, identify IAM misconfigurations, discover privilege escalation paths, and retrieve the information required to construct the challenge flag.

### Scenario

As a member of Secure Corp's Red Team, we are provided with the credentials of a low-privileged Service Account. The objective is not to immediately compromise cloud resources, but to understand the IAM architecture, enumerate custom roles, discover resource-level permissions, identify privilege escalation opportunities, and determine which identities possess administrative control over other Service Accounts.

### Skills Practiced

- GCP IAM Enumeration
- Service Account Enumeration
- IAM Policy Analysis
- Custom Role Enumeration
- Bucket Permission Analysis
- Privilege Escalation Path Discovery
- Cloud CLI Usage

---

# Environment

## Cloud Provider

Google Cloud Platform (GCP)

## Initial Access

A Service Account private key was provided in JSON format.

The first step was authenticating with GCP using the provided credentials.

Example:

```bash
gcloud auth activate-service-account \
    --key-file testing-srvacc-key.json
```

## Available Tools

- Google Cloud SDK (gcloud)
- gsutil
- Cloud IAM APIs

---

# Attack Methodology

## Initial Hypothesis

Since the challenge provides credentials for a Service Account instead of an administrator account, it is reasonable to assume that the account has limited permissions.

The investigation therefore focuses on answering four questions:

- Which project do these credentials belong to?
- Which IAM roles are assigned to this Service Account?
- Which cloud resources can it access?
- Is there any privilege escalation path involving other Service Accounts?

Expected targets:

- IAM Roles
- Service Accounts
- Custom Roles
- Storage Buckets
- IAM Policies

---

# Attack Workflow

The attack consisted primarily of IAM enumeration rather than direct exploitation.

## Step 1 - Authenticate to GCP

### Action

Authenticate using the provided Service Account key.

Command:

```bash
gcloud auth activate-service-account \
    --key-file testing-srvacc-key.json
```

Verify authentication:

```bash
gcloud auth list
```

Observation:

The active identity became:

```
testing-service-account@woven-acolyte-428406-v9.iam.gserviceaccount.com
```

This confirms successful authentication.

---

## Step 2 - Discover the Project

### Goal

Identify the GCP project associated with the compromised credentials.

Command

```bash
gcloud projects list
```

Result

```
woven-acolyte-428406-v9
```

Observation

This project ID is required for nearly every subsequent enumeration command.

---

## Step 3 - Enumerate Service Accounts

### Goal

Identify all Service Accounts present in the project.

Command

```bash
gcloud iam service-accounts list \
    --project woven-acolyte-428406-v9
```

Findings

Several Service Accounts were discovered, including:

- testing-service-account
- devops-service-account
- prod-service-account
- resource-mgmt
- secret-mgmt-sa
- service-mgmt-sa
- log-reviewer-sa
- hd-service-account

This immediately suggests an environment where different Service Accounts manage different operational tasks.

---

## Step 4 - Enumerate IAM Policy Bindings

### Goal

Identify role assignments across the entire project.

Command

```bash
gcloud projects get-iam-policy \
    woven-acolyte-428406-v9
```

Findings

The compromised Service Account possesses several roles:

- roles/viewer
- roles/iam.roleViewer
- roles/iam.securityReviewer
- projects/.../roles/customViewerRole1

Other interesting identities were also discovered, including:

- devops-service-account
- prod-service-account
- resource-mgmt

Each with significantly different privilege levels.

---

## Step 5 - Enumerate Assigned Roles

### Goal

Determine the exact roles assigned to the compromised Service Account.

Command

```bash
gcloud projects get-iam-policy \
    woven-acolyte-428406-v9 \
    --flatten="bindings[].members" \
    --filter="bindings.members:serviceAccount:testing-service-account@woven-acolyte-428406-v9.iam.gserviceaccount.com" \
    --format="table(bindings.role)"
```

Result

```
roles/viewer
roles/iam.roleViewer
roles/iam.securityReviewer
projects/.../roles/customViewerRole1
```

Observation

Although the Service Account is relatively low privileged, it possesses strong visibility into IAM resources.

---

## Step 6 - Enumerate Custom Roles

### Goal

Understand the permissions granted by the project's custom roles.

List available custom roles:

```bash
gcloud iam roles list \
    --project woven-acolyte-428406-v9
```

Describe the role:

```bash
gcloud iam roles describe \
    customViewerRole1 \
    --project woven-acolyte-428406-v9
```

Result

```
storage.buckets.list
storage.objects.get
```

Observation

The custom role grants read access to Cloud Storage objects.

---

## Step 7 - Identify Accessible Resources

### Goal

Determine where the custom role can actually be used.

Command

```bash
gcloud storage buckets list
```

During bucket enumeration an interesting bucket appeared:

```
secret-bucket-woven-acolyte-428406-v9
```

The project IAM policy also contained the following conditional binding:

```
resource.name == "secret-bucket-woven-acolyte-428406-v9"
```

Observation

Although the custom role contains only two permissions, the IAM Condition restricts those permissions to a single storage bucket.

---

## Step 8 - Inspect Bucket Permissions

### Goal

Verify access to the target bucket.

Command

```bash
gcloud storage buckets get-iam-policy \
gs://secret-bucket-woven-acolyte-428406-v9
```

The bucket policy confirms that the compromised Service Account has read access.

---

## Step 9 - Retrieve Bucket Contents

### Goal

Read accessible objects.

List files

```bash
gsutil ls -r \
gs://secret-bucket-woven-acolyte-428406-v9
```

Download file

```bash
gsutil cp \
gs://secret-bucket-woven-acolyte-428406-v9/secret.txt .
```

Observation

This demonstrates how a seemingly harmless custom role can expose sensitive information.

---

## Step 10 - Discover Privilege Escalation Paths

### Goal

Identify relationships between Service Accounts.

Command

```bash
gcloud iam service-accounts get-iam-policy \
devops-service-account@woven-acolyte-428406-v9.iam.gserviceaccount.com
```

Result

```
prod-service-account

roles/iam.serviceAccountAdmin

roles/iam.serviceAccountKeyAdmin
```

Observation

The compromised Service Account cannot directly escalate privileges.

However, IAM enumeration reveals that **prod-service-account** possesses administrative control over the **devops-service-account**.

This represents the privilege escalation path intentionally hidden in the environment.

---

# Attack Chain

```text
Compromised Service Account
        |
        v
Authenticate to GCP
        |
        v
Project Enumeration
        |
        v
Service Account Enumeration
        |
        v
IAM Policy Enumeration
        |
        v
Custom Role Enumeration
        |
        v
Bucket Permission Discovery
        |
        v
Sensitive File Retrieval
        |
        v
Privilege Escalation Path Discovery
```

---

# Findings

## Security Weakness

Several IAM misconfigurations and information disclosure issues were identified.

- Overly informative IAM permissions granted to a low-privileged Service Account.
- Custom role exposing storage objects.
- Resource-level IAM Condition protecting only a specific bucket.
- Administrative control delegated from one Service Account to another.

## Impact

An attacker with the compromised Service Account could:

- Enumerate the project's IAM architecture.
- Discover all Service Accounts.
- Understand custom roles.
- Read sensitive bucket contents.
- Map privilege escalation paths for future attacks.

Although immediate privilege escalation was not possible, the environment leaked valuable information for subsequent attack stages.

---

# Evidence

Important commands used during the assessment:

```bash
gcloud auth activate-service-account \
--key-file testing-srvacc-key.json

gcloud projects list

gcloud iam service-accounts list

gcloud projects get-iam-policy woven-acolyte-428406-v9

gcloud iam roles list

gcloud iam roles describe customViewerRole1

gcloud storage buckets list

gcloud storage buckets get-iam-policy \
gs://secret-bucket-woven-acolyte-428406-v9

gsutil ls -r \
gs://secret-bucket-woven-acolyte-428406-v9

gsutil cp \
gs://secret-bucket-woven-acolyte-428406-v9/secret.txt .

gcloud iam service-accounts get-iam-policy \
devops-service-account@woven-acolyte-428406-v9.iam.gserviceaccount.com
```

---

# Defensive Recommendations

- Apply the Principle of Least Privilege.
- Reduce IAM visibility granted to low-privileged identities.
- Regularly audit custom roles.
- Restrict Storage Bucket access using IAM Conditions where appropriate.
- Review Service Account administration permissions.
- Continuously monitor IAM policy changes using Cloud Audit Logs.

---

# Automation

The challenge can also be automated using Rhino Security Labs' GCP IAM Privilege Escalation Scanner.

Useful scripts include:

```text
enumerate_member_permissions.py

check_for_privesc.py
```

These scripts enumerate permissions and automatically identify known privilege escalation paths.

---

# Key Takeaways

- IAM enumeration is often more valuable than immediate exploitation.
- Custom roles should be reviewed as carefully as predefined roles.
- IAM Conditions can significantly reduce the blast radius of permissions.
- Mapping relationships between Service Accounts is essential during cloud assessments.
- Privilege escalation opportunities frequently arise from delegated administrative permissions rather than overly permissive identities.

---

# References

- Google Cloud IAM Documentation
- Google Cloud Service Accounts Documentation
- Google Cloud Storage IAM Documentation
- Rhino Security Labs - GCP IAM Privilege Escalation Scanner
