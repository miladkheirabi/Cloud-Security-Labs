# GCP Storage Bucket Policy Modification

## Challenge Overview

### Objective

Detect and investigate suspicious bucket policy modification activity within a GCP environment using audit logs.

### Scenario

A security analyst needs to identify unauthorized changes to GCP Storage bucket permissions and determine the identity involved in the policy modification.

### Skills Practiced

* GCP Cloud Audit Logs investigation
* Google Cloud Storage security monitoring
* IAM policy modification detection
* Log analysis with ELK/Kibana

---

## Environment

### Data Source

GCP Audit Logs collected in ELK.

### Available Access

* ELK access
* Permission to view GCP audit events

---

## Investigation Process

### Initial Hypothesis

Unauthorized bucket access can occur when an attacker modifies IAM policies associated with a Cloud Storage bucket.

Expected indicators:

* Audit log dataset: gcp.audit
* Service: storage.googleapis.com
* Action related to IAM policy modification
* Service Account or user identity performing the action

---

## Investigation Workflow

The investigation was performed by gradually narrowing down GCP Audit Logs.

### Step 1 - Identify GCP Audit Logs

Query:

```
event.dataset: gcp.audit
```

Purpose:

Filter only GCP audit events from the SIEM.

Why:

GCP Audit Logs contain records of actions performed by users, administrators, and services interacting with GCP resources.

---

### Step 2 - Focus on Cloud Storage Activity

Query:

```
event.dataset: gcp.audit and gcp.audit.service_name: storage.googleapis.com
```

Purpose:

Focus investigation on Google Cloud Storage related activities.

Why:

Bucket permission changes and bucket-related security events are recorded by the Storage service.

---

### Step 3 - Identify Bucket IAM Policy Modification

Query:

```
event.dataset: gcp.audit and gcp.audit.service_name: storage.googleapis.com and event.action: storage.setIamPermissions
```

Purpose:

Identify events where bucket IAM permissions were modified.

Why:

Attackers may modify bucket IAM policies to gain unauthorized access or create overly permissive permissions.

---

## Findings

The investigation identified a suspicious Storage IAM policy modification event.

The related Service Account involved in the activity can be identified through:

```
json.protoPayload.serviceData.policyDelta.bindingDeltas.member
```

The identified identity format:

```
[service-account-name@project-id.iam.gserviceaccount.com](mailto:service-account-name@project-id.iam.gserviceaccount.com)
```

---

## Important Fields

| Field                                         | Description                              |
| --------------------------------------------- | ---------------------------------------- |
| gcp.audit.service_name                        | GCP service generating the event         |
| event.action                                  | Action performed in the event            |
| gcp.audit.authentication_info.principal_email | Identity performing the action           |
| resourceName                                  | Target resource                          |
| policyDelta.bindingDeltas.member              | Identity added or modified in IAM policy |

---

## Key Takeaways

* GCP Audit Logs are used to investigate control plane activities.
* Bucket IAM policy changes are not detected through VPC logs.
* Storage IAM modifications should be investigated through audit events.
* Service Accounts can perform actions similar to human users.
* Policy modification events can reveal unauthorized privilege changes.

---

## References

* GCP Cloud Audit Logs
* Google Cloud Storage IAM
* GCP Service Accounts
