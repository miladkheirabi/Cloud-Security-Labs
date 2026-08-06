# GCP - Service Account Granted with Privileged Access

## Scenario

Detect suspicious activities where a **Google Cloud Service Account** is granted privileged permissions and identify the Autonomous System (AS) organization associated with the activity.

---

## Objective

Identify the value of:

```
source.as.organization.name
```

for the service account activity.

---

## Investigation Process

### Step 1 - Limit the investigation to GCP Audit Logs

```kql
event.dataset: gcp.audit
```

This filters the investigation to Google Cloud Audit Logs, which record administrative and API activities.

---

### Step 2 - Find IAM Policy Changes

Granting privileges is performed through IAM policy modifications.

Search for:

```kql
event.dataset: gcp.audit and event.action: SetIamPolicy
```

This reveals events where IAM permissions were modified.

---

### Step 3 - Identify the Service Account

Inspect the returned event and note the service account email.

Example:

```text
auto-bot-sa@internal-428507.iam.gserviceaccount.com
```

---

### Step 4 - Review All Activities of the Service Account

```kql
event.dataset: gcp.audit and user.email:"auto-bot-sa@internal-428507.iam.gserviceaccount.com"
```

This shows every recorded action performed by that service account.

---

### Step 5 - Identify the Source AS Organization

Open the relevant event and inspect the field:

```text
source.as.organization.name
```

This field contains the Autonomous System (AS) organization from which the request originated.

---

## Useful Fields

| Field | Purpose |
|--------|---------|
| `event.dataset` | Restrict logs to GCP Audit Logs |
| `event.action` | Identify the API operation performed |
| `user.email` | Service account or user responsible for the action |
| `source.ip` | Source IP address |
| `source.as.organization.name` | Autonomous System organization |
| `source.geo.*` | Geographic information |

---

## Investigation Logic

```
GCP Audit Logs
        │
        ▼
SetIamPolicy
        │
        ▼
Identify Service Account
        │
        ▼
Review Service Account Activities
        │
        ▼
Inspect source.as.organization.name
```

---

## Key Takeaways

- `SetIamPolicy` is one of the most important events when investigating IAM privilege changes.
- A service account often appears in `user.email`.
- After identifying the service account, pivot on `user.email` to review all of its activities.
- Network attribution can be obtained from `source.ip`, `source.as.organization.name`, and `source.geo.*`.
