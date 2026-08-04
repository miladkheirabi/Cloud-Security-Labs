# Repository Structure

```text
Cloud-Security-Labs/

├── AWS/
│   ├── CloudTrail/
│   ├── IAM/
│   ├── S3/
│   └── EC2/
│
├── GCP/
│   ├── Cloud-Audit-Logs/
│   ├── IAM/
│   ├── Storage/
│   └── VPC/
│
├── Azure/
│
├── Notes/
│   ├── AWS-CloudTrail-Queries.md
│   ├── GCP-Audit-Logs-Queries.md
│   └── Cloud-Security-Concepts.md
│
└── Templates/
    └── challenge-template.md
```

---

# Investigation Documentation Template

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
