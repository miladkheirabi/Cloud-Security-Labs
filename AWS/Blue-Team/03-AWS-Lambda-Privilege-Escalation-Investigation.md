# AWS Lambda Privilege Escalation Investigation

## Challenge Overview

### Objective

Identify and determine the User ID associated with a suspicious privilege escalation activity in an AWS environment by analyzing AWS CloudTrail logs.

The goal is to investigate IAM activity, identify the user responsible for privilege escalation, and determine the identity associated with the malicious activity.

### Scenario

A suspicious privilege escalation attempt was detected in an AWS account.

An attacker with access to an IAM user account attempted to increase privileges by creating credentials for a highly privileged IAM user. The investigation required analyzing AWS CloudTrail events through a SIEM platform to identify the responsible user.

### Skills Practiced

* AWS CloudTrail log investigation
* IAM privilege escalation detection
* Access Key abuse analysis
* Identity attribution using CloudTrail
* Timeline-based incident investigation

---

# Environment

## Data Source

AWS CloudTrail logs collected and stored in ELK/SIEM.

CloudTrail records AWS API activity including:

* Event name
* User identity
* Access Key information
* Source IP
* Target resources
* Request parameters

## Available Access

* SIEM / ELK dashboard access
* CloudTrail log search capability
* Analytics → Discover section

---

# Investigation Process

## Initial Hypothesis

The challenge scenario suggested suspicious privilege escalation activity targeting AWS Lambda resources.

Expected attack pattern:

```
Compromised IAM User
        |
        |
        v
Privilege Escalation Attempt
        |
        |
        v
Creation of privileged credentials
        |
        |
        v
Use of elevated permissions
```

Expected indicators:

* Suspicious IAM user activity
* Access Key creation events
* Privileged user targeting
* Follow-up activity using newly created credentials

---

# Investigation Workflow

The investigation was performed by gradually narrowing down relevant AWS CloudTrail events.

---

## Step 1 - Filtering AWS CloudTrail Logs

Query:

```text
event.dataset: aws.cloudtrail
```

Purpose:

Filter the SIEM data source and focus only on AWS CloudTrail events.

Why:

CloudTrail contains detailed records of AWS API operations, including identity information, performed actions, and affected resources. Filtering the dataset reduces unrelated logs and improves investigation accuracy.

---

## Step 2 - Prioritizing IAM User Activity

Query:

```text
event.dataset: aws.cloudtrail AND aws.cloudtrail.user_identity.type: IAMUser
```

Purpose:

Focus the investigation on actions performed by IAM users.

Why:

Privilege escalation activities are commonly performed through compromised IAM users that abuse their permissions to obtain additional access.

---

## Step 3 - Investigating Credential Management Activity

Query:

```text
event.dataset: aws.cloudtrail AND event.action:"CreateAccessKey"
```

Purpose:

Identify suspicious creation of AWS Access Keys.

Why:

Attackers often create Access Keys for privileged IAM accounts to maintain access or escalate their permissions.

Expected behavior:

```
Attacker User
      |
      |
      v
Create Access Key
      |
      |
      v
Privileged IAM User
```

---

## Step 4 - Identifying the Privilege Escalation Actor

The investigation revealed the following CloudTrail event:

```text
Event:
CreateAccessKey

Actor:
EMP066735

Target:
Administrator

Created Access Key:
AKIA4MTWLTQE6DIAGBXH
```

The important fields:

```json
{
  "event.action": "CreateAccessKey",
  "user.name": "EMP066735",
  "requestParameters.userName": "Administrator",
  "responseElements.accessKey.accessKeyId": "AKIA4MTWLTQE6DIAGBXH"
}
```

This confirmed that user `EMP066735` created credentials for the Administrator account.

---

## Step 5 - Tracking the Newly Created Credentials

Query:

```text
aws.cloudtrail.user_identity.access_key_id.keyword:"AKIA4MTWLTQE6DIAGBXH"
```

Purpose:

Track all activity performed using the newly created Access Key.

Why:

After privilege escalation, attacker activity should be associated with the compromised privileged credentials.

The investigation found:

```text
Event:
GetCallerIdentity

User:
Administrator

Access Key:
AKIA4MTWLTQE6DIAGBXH
```

This confirmed that the attacker successfully authenticated using the newly created Administrator credentials.

---

# Findings

The privilege escalation activity was performed by:

```text
IAM User:
EMP066735
```

Associated User ID:

```text
AIDA4MTWLTQEUT45ECXUK
```

The escalation method:

```text
EMP066735
        |
        |
        v
CreateAccessKey
        |
        |
        v
Administrator
        |
        |
        v
AKIA4MTWLTQE6DIAGBXH
```

The attacker then verified the new identity using:

```text
sts:GetCallerIdentity
```

with the Administrator Access Key.

---

# Important Fields

| Field                                      | Description                            |
| ------------------------------------------ | -------------------------------------- |
| event.dataset                              | Identifies the log source              |
| event.action                               | AWS API operation performed            |
| user.name                                  | IAM username performing the action     |
| user.id                                    | Internal IAM principal ID              |
| aws.cloudtrail.user_identity.access_key_id | Access Key used for API request        |
| requestParameters.userName                 | Target IAM user affected by the action |
| responseElements.accessKey.accessKeyId     | Newly generated Access Key             |
| user_identity.type                         | Identity type performing the action    |
| source.ip                                  | Source IP address of the request       |

---

# Investigation Insight

The initial `GetCallerIdentity` event was a useful indicator but was not the privilege escalation action itself.

The actual escalation happened when:

```text
CreateAccessKey
```

was executed against the Administrator IAM user.

`GetCallerIdentity` only confirmed the identity being used after the attacker obtained higher privileges.

---

# Key Takeaways

* AWS Account ID, IAM User Name, and IAM User ID represent different concepts.
* IAM User ID (`AIDA...`) is the internal identifier of an AWS principal.
* Creating Access Keys for privileged users is a common AWS privilege escalation technique.
* CloudTrail Access Key tracking helps reconstruct attacker activity.
* `sts:GetCallerIdentity` is commonly used by attackers to validate obtained permissions.
* Timeline analysis is critical when investigating cloud security incidents.

---

# References

* AWS CloudTrail Documentation
* AWS IAM Documentation
* AWS STS GetCallerIdentity Documentation
* AWS IAM Access Key Management Documentation
