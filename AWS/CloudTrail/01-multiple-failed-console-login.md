# Multiple Failed AWS Console Login Attempts

## Challenge Overview

### Objective

Detect and investigate multiple failed login activities within an AWS environment using CloudTrail logs.

### Scenario

A security analyst needs to identify suspicious authentication activities and determine the number of failed login attempts.

### Skills Practiced

- AWS CloudTrail investigation
- Authentication monitoring
- IAM security monitoring
- Log analysis with ELK/Kibana


---

## Environment

### Data Source

AWS CloudTrail logs collected in ELK.

### Available Access

- ELK access
- Permission to view CloudTrail events


---

## Investigation Process

### Initial Hypothesis

Failed AWS console login attempts should generate authentication-related CloudTrail events.

Expected indicators:

- Event Name: ConsoleLogin
- Event Source: signin.amazonaws.com
- Login Result: Failure


---

## Investigation Workflow

The investigation was performed by gradually narrowing down CloudTrail events.


### Step 1 - Identify CloudTrail Logs

Query:

CODE_BLOCK_START
event.dataset: aws.cloudtrail
CODE_BLOCK_END

Purpose:

Filter only AWS CloudTrail events from the SIEM.

Why:

CloudTrail logs contain records of AWS API activity, including authentication and resource access events.


---

### Step 2 - Focus on IAM User Activity

Query:

CODE_BLOCK_START
event.dataset: aws.cloudtrail and aws.cloudtrail.user_identity.type: IAMUser
CODE_BLOCK_END

Purpose:

Focus investigation on activities performed by IAM users.

Why:

IAM users represent identities that can authenticate and perform actions inside an AWS account.


---

### Step 3 - Filter Failed Authentication Events

Query:

CODE_BLOCK_START
event.dataset: aws.cloudtrail and aws.cloudtrail.user_identity.type: IAMUser and event.outcome: "failure"
CODE_BLOCK_END

Purpose:

Identify unsuccessful authentication attempts.

Why:

Multiple failed authentication events may indicate brute-force attempts or unauthorized access attempts.


---

### Step 4 - Identify Failed Console Login Attempts

Query:

CODE_BLOCK_START
event.dataset: aws.cloudtrail and aws.cloudtrail.user_identity.type: IAMUser and event.outcome: "failure" and event.action: "ConsoleLogin"
CODE_BLOCK_END

Purpose:

Focus specifically on AWS Console login failures.

Why:

ConsoleLogin events represent AWS Management Console authentication activity.


---

## Findings

The search returned 34 records.

All identified events contained:

CODE_BLOCK_START
ConsoleLogin=Failure
CODE_BLOCK_END


The failed login count associated with the identified failed event:

CODE_BLOCK_START
34
CODE_BLOCK_END


---

## Important Fields

| Field | Description |
|---|---|
| eventName | AWS API event name |
| eventSource | AWS service generating the event |
| sourceIPAddress | Source IP address of login attempt |
| userAgent | Client/application used |
| userIdentity | Identity involved in the event |
| responseElements.ConsoleLogin | Login result (Success/Failure) |


---

## Key Takeaways

- AWS CloudTrail records authentication activities through API events.
- AWS Console login attempts appear as ConsoleLogin events.
- Multiple failed authentication attempts can indicate brute-force activity.
- Monitoring authentication failures helps detect unauthorized access attempts.
- Investigation should start broad and gradually narrow down to suspicious events.


---

## References

- AWS CloudTrail
- AWS IAM Authentication
