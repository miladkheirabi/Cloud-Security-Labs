# Cloud Security Labs

## About This Repository

This repository contains my notes, investigations, and findings while practicing Cloud Security challenges.

The main goal is not only to solve challenges, but to document the investigation process:

* Understanding the security scenario
* Identifying the correct log source
* Building investigation queries
* Analyzing cloud events
* Extracting security-relevant evidence
* Documenting lessons learned

The repository focuses on cloud security monitoring and incident investigation across different cloud providers.

---

# Documentation Principles

## Focus on Investigation, Not Only Answers

The purpose of these notes is not only to record the final answer.

A good investigation should explain:

1. What happened?
2. Where was it recorded?
3. Which logs were useful?
4. Why were those logs selected?
5. What evidence confirmed the activity?

---

## Prefer Investigation Paths Over Random Queries

Instead of only writing:

```text
search keyword → find result
```

Document the reasoning:

```text
Scenario
    ↓
Expected cloud activity
    ↓
Relevant log source
    ↓
Filtering strategy
    ↓
Evidence extraction
```

---

## Keep Queries Reusable

Common queries should also be added to the Notes directory.

Examples:

* CloudTrail investigation queries
* GCP Audit Log queries
* IAM monitoring queries
* Suspicious authentication patterns

---

# Goal

Build a personal Cloud Security investigation knowledge base by documenting real-world security scenarios and the process used to analyze them.
