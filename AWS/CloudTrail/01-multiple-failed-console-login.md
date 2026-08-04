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

- Event Name: `ConsoleLogin`
- Event Source: `signin.amazonaws.com`
- Login Result: `Failure`


---

## Query

```text
consolelogin
```


---

## Findings

The search returned **34 records**.

All identified events contained:

```text
ConsoleLogin=Failure
```


---

## Analysis

The events indicate multiple failed AWS Console authentication attempts.

The failed login count associated with the identified failed event:

```text
34
```


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
- AWS Console login attempts appear as `ConsoleLogin` events.
- Multiple failed authentication attempts can indicate brute-force activity.
- Monitoring authentication failures helps detect unauthorized access attempts.


---

## References

- AWS CloudTrail
- AWS IAM Authentication
