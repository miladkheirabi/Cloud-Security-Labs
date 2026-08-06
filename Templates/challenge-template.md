# Repository Structure

```text
Cloud-Security-Labs/

├── AWS/
│   ├── Red-Team/
│   └── Blue-Team/
│
├── GCP/
│   ├── Red-Team/
│   └── Blue-Team/
│
├── Azure/
│   ├── Red-Team/
│   └── Blue-Team/
│
└── Templates/
    └── challenge-template.md
```

---

# Blue Team Documentation Template

Use the following structure for each challenge.

```markdown
# [Challenge Name]

## Challenge Overview

### Objective

[Describe the goal of the challenge.]

### Scenario

[Describe the security scenario.]

### Skills Practiced

- [Skill 1]
- [Skill 2]
- [Skill 3]


---

## Environment

### Data Source

[Example: AWS CloudTrail logs / GCP Audit Logs / Azure Activity Logs]

### Available Access

- [Access type]
- [Tools available]


---

## Investigation Process

### Initial Hypothesis

[Explain what type of activity you expect to find and why.]

Expected indicators:

- [Indicator 1]
- [Indicator 2]
- [Indicator 3]


---

## Investigation Workflow

The investigation was performed by gradually narrowing down relevant logs.


### Step 1 - [Investigation Step Name]

Query:

CODE_BLOCK_START
[Query]
CODE_BLOCK_END

Purpose:

[Explain what this query filters.]

Why:

[Explain why this step is useful.]


---

### Step 2 - [Investigation Step Name]

Query:

CODE_BLOCK_START
[Query]
CODE_BLOCK_END

Purpose:

[Explain the purpose.]

Why:

[Explain the security reasoning.]


---

## Findings

[Describe what was discovered.]

Important evidence:

CODE_BLOCK_START
[Evidence]
CODE_BLOCK_END


---

## Important Fields

| Field | Description |
|---|---|
| Field name | Description |
| Field name | Description |


---

## Key Takeaways

- [Lesson learned]
- [Important cloud security concept]
- [Investigation insight]


---

## References

- [Relevant documentation]
```

# Red Team Documentation Template

Use the following structure for each challenge.

```markdown

# [Challenge Name]

## Challenge Overview

### Objective

[Describe the attack objective and what the attacker is trying to achieve.]

### Scenario

[Describe the cloud environment, initial access, and security weakness being investigated.]

### Skills Practiced

* [Skill 1]
* [Skill 2]
* [Skill 3]

---

# Environment

## Cloud Provider

[AWS / Azure / GCP]

## Initial Access

[Describe the access provided at the beginning of the challenge.]

Examples:

* Access Key + Secret Key
* Client ID + Client Secret
* User Credentials
* Vulnerable Application Access

## Available Tools

* [Tool 1]
* [Tool 2]
* [Tool 3]

---

# Attack Methodology

## Initial Hypothesis

[Explain what you expected to find and why.]

Expected targets:

* [Target 1]
* [Target 2]
* [Target 3]

---

# Attack Workflow

The attack was performed by gradually expanding access and identifying privilege escalation paths.

## Step 1 - Initial Access Validation

### Action

[Describe the first action performed.]

Command:

CODE_BLOCK_START
command
CODE_BLOCK_END

Result:

CODE_BLOCK_START
output
CODE_BLOCK_END

Observation:

[Explain what this result means.]

---

## Step 2 - Enumeration

### Goal

[Explain what information you were trying to discover.]

Command:

CODE_BLOCK_START
command
CODE_BLOCK_END

Result:

CODE_BLOCK_START
output
CODE_BLOCK_END

Findings:

* [Finding 1]
* [Finding 2]

---

## Step 3 - Privilege Escalation

### Misconfiguration Identified

[Describe the vulnerability or weakness that allowed privilege escalation.]

Example:

An IAM role trust relationship allowed the compromised identity to assume a higher privileged role.

Command:

CODE_BLOCK_START
command
CODE_BLOCK_END

Result:

CODE_BLOCK_START
output
CODE_BLOCK_END

---

## Step 4 - Access Target Resource

### Target

[Resource name]

Example:

* S3 Bucket
* Secret Store
* Database
* API

Command:

CODE_BLOCK_START
command
CODE_BLOCK_END

Result:

CODE_BLOCK_START
output
CODE_BLOCK_END

---

# Attack Chain

CODE_BLOCK_START
Initial Access
        |
        v
Enumeration
        |
        v
Privilege Escalation
        |
        v
Target Access
        |
        v
Sensitive Data Retrieval
CODE_BLOCK_END

---

# Findings

## Security Weakness

[Describe the root cause of the vulnerability.]

## Impact

[Explain what an attacker could achieve.]

Example:

An attacker with compromised credentials could escalate privileges and access restricted cloud resources.

---

# Evidence

Important commands and outputs:

CODE_BLOCK_START
Evidence
CODE_BLOCK_END

---

# Defensive Recommendations

* Apply least privilege principles.
* Review IAM roles and trust relationships.
* Monitor privilege escalation activities.
* Enable cloud audit logging.
* Regularly review permissions.

---

# Key Takeaways

* [Cloud security concept learned]
* [Attack technique learned]
* [Important investigation insight]

---

# References

* [Relevant documentation]
```
