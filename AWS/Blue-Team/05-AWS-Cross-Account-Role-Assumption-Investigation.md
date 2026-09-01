# Cross-Account Role Assumption Investigation

This challenge simulates a real-world AWS threat scenario in which an attacker gains access to AWS credentials, attempts to obtain higher privileges through **Cross-Account Role Assumption**, and subsequently accesses data stored in an S3 bucket.

The investigation focuses on correlating the attacker's activities through **AWS CloudTrail logs** collected in the SIEM.

The main investigation phases are:

* Identifying suspicious `AssumeRole` activity
* Detecting potential privilege escalation
* Investigating activity performed using the assumed role
* Identifying the targeted S3 bucket
* Investigating potential data access or exfiltration

> **Note:** The original challenge description incorrectly refers to Azure in several places. The actual investigation and logs are based on **AWS CloudTrail** and AWS services.

---

## 1. Targeted Investigation of CloudTrail Logs

The initial investigation starts by filtering **AWS CloudTrail** events in the SIEM.

```kql
event.dataset: "aws.cloudtrail"
```

CloudTrail records detailed information about AWS API activity, including:

* Event time
* Event name/action
* Source IP
* User identity
* User agent
* AWS account
* Requested resources
* Request parameters
* Error codes
* Authentication and authorization information

These logs can be correlated to reconstruct an attacker's movement across AWS services.

---

## 2. Prioritize Enumeration Activity Related to AssumeRole

During a Cross-Account Role Assumption attack, an attacker may enumerate available roles and attempt to assume roles that provide higher privileges.

Repeated `AssumeRole` requests resulting in `AccessDenied`, especially when associated with automation-oriented user agents such as **Boto3**, can be an indicator of role enumeration or brute-force attempts.

The following query filters for this activity:

```kql
event.dataset: "aws.cloudtrail"
and aws.cloudtrail.error_code: "AccessDenied"
and user_agent.name: "Boto3"
and event.action: "AssumeRole"
```

The results can be used to determine whether an attacker was attempting to identify a role that could successfully be assumed.

---

## 3. Identify the Suspicious AssumeRole Activity

After identifying the role-enumeration activity, the next step is to locate a successful or otherwise suspicious `AssumeRole` operation.

A useful filter is:

```kql
event.dataset: "aws.cloudtrail"
and event.action: "AssumeRole"
and aws.cloudtrail.user_identity.type: "IAMUser"
```

The investigation should focus on identifying:

* The IAM user performing the operation
* The source AWS account
* The target AWS account
* The assumed role
* The source IP address
* The user agent
* The timestamp of the operation

The **Role ARN** returned in the event is particularly important because it identifies the role that the attacker attempted to assume.

For example:

```text
arn:aws:iam::<ACCOUNT_ID>:role/DbAdmin
```

The extracted role name can then be used to investigate subsequent activity performed through the assumed session.

---

## 4. Investigate Actions Performed Using the Assumed Role

Once the assumed role has been identified, the next step is to trace activity performed using that role.

For example, if the identified role is:

```text
DbAdmin
```

the following query can be used to locate successful events associated with that identity:

```kql
event.dataset: "aws.cloudtrail"
and user.name: "DbAdmin"
and event.outcome: success
```

This allows the analyst to investigate what actions were performed after the role was assumed.

The investigation should particularly focus on access to sensitive AWS resources, especially **Amazon S3**.

---

# 5. Identify the Targeted S3 Bucket

After following the assumed-role activity, the next step is to identify which S3 bucket was accessed.

Instead of manually inspecting every CloudTrail event, the bucket name can be extracted from the request parameters.

The relevant field in the SIEM is:

```text
aws.cloudtrail.flattened.request_parameters.bucketName.keyword
```

Filtering/aggregating this field reveals the S3 buckets referenced by the CloudTrail events.

In this challenge, only two bucket names were observed:

```text
securecorpdatastore
```

and

```text
cloudtrail
```

The `cloudtrail` bucket is associated with CloudTrail logging itself and is therefore not the likely target of the attacker's data-access activity.

The relevant target bucket is:

```text
securecorpdatastore
```

---

# 6. Investigate S3 Data Access

The investigation can then be narrowed to events involving the identified bucket:

```kql
event.dataset: "aws.cloudtrail"
and aws.cloudtrail.flattened.request_parameters.bucketName.keyword: "securecorpdatastore"
```

The analyst should examine the resulting events for S3 operations such as:

* `ListObjects`
* `ListObjectsV2`
* `GetObject`
* `HeadObject`
* Other object-level access operations

Of particular interest are successful `GetObject` operations, because they indicate that objects stored in the bucket were actually retrieved.

---

# Investigation Summary

The attack path can therefore be reconstructed as:

```text
Compromised AWS Credentials
          │
          ▼
Role Enumeration
(AssumeRole + AccessDenied)
          │
          ▼
Successful AssumeRole
          │
          ▼
Higher-Privilege Role
          │
          ▼
Actions Performed Using Assumed Role
          │
          ▼
S3 Bucket Access
          │
          ▼
securecorpdatastore
          │
          ▼
Object Retrieval / Potential Data Exfiltration
```

### Key Investigation Indicators

| Investigation Stage        | Important Indicator                                              |
| -------------------------- | ---------------------------------------------------------------- |
| CloudTrail investigation   | `event.dataset: "aws.cloudtrail"`                                |
| Role enumeration           | `AssumeRole` + `AccessDenied` + `Boto3`                          |
| Successful role assumption | `event.action: "AssumeRole"`                                     |
| Identity type              | `aws.cloudtrail.user_identity.type: "IAMUser"`                   |
| Assumed role               | Role ARN / role name                                             |
| Post-assumption activity   | `user.name: "<ROLE_NAME>"`                                       |
| Target bucket              | `aws.cloudtrail.flattened.request_parameters.bucketName.keyword` |
| Identified bucket          | `securecorpdatastore`                                            |
| Data access                | S3 object-level events such as `GetObject`                       |

# Final Finding

The suspicious activity follows a typical AWS privilege-escalation and data-access chain:

**IAM credential compromise → Cross-Account Role Assumption → privileged role activity → S3 access → `securecorpdatastore` data access.**

The relevant S3 bucket identified during the investigation is:

```text
securecorpdatastore
```
