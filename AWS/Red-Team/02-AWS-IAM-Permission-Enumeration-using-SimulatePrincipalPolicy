# AWS IAM Permission Enumeration using SimulatePrincipalPolicy

## Challenge Overview

### Objective

Identify the permissions assigned to the currently authenticated AWS IAM user using `iam:SimulatePrincipalPolicy`.

The challenge focuses on IAM permission enumeration and understanding how AWS evaluates authorization requests when direct IAM policy enumeration is restricted.

### Scenario

The challenge provides AWS credentials for an IAM user. The objective is to determine what permissions are assigned to the authenticated identity.

Direct IAM policy enumeration is not permitted for the compromised user, so the IAM Policy Simulator can be used to test specific actions and identify the policy responsible for allowing them.

### Skills Practiced

* AWS IAM enumeration
* AWS CLI
* IAM Policy Simulation
* IAM policy evaluation
* Managed Policies
* Permission analysis
* Cloud privilege assessment

---

# Environment

## Cloud Provider

AWS

## Initial Access

AWS credentials were provided for the challenge.

The authenticated identity can be verified using:

```powershell
aws sts get-caller-identity
```

The command returned:

```json
{
    "UserId": "AIDAQ3EGUZMEQ5XI7YUUK",
    "Account": "058264439561",
    "Arn": "arn:aws:iam::058264439561:user/DevAppUser"
}
```

The current IAM identity is therefore:

```text
arn:aws:iam::058264439561:user/DevAppUser
```

---

# Attack Methodology

The investigation started by attempting to enumerate the IAM permissions assigned to `DevAppUser`.

The initial approach was to query the user's policies directly.

---

## Step 1 - Attempt Direct IAM Enumeration

### List Inline Policies

```powershell
aws iam list-user-policies --user-name DevAppUser
```

Result:

```text
AccessDenied
```

### List Attached Managed Policies

```powershell
aws iam list-attached-user-policies --user-name DevAppUser
```

Result:

```text
AccessDenied
```

### Observation

The current identity could not directly enumerate its own IAM policies.

This is an important distinction in AWS IAM:

> Having permission to perform an AWS action does not automatically mean that the identity has permission to inspect the IAM policies that grant that access.

Therefore, another method was required to determine the effective permissions.

---

# Step 2 - Explore Available AWS Access

The authenticated user was able to list S3 buckets:

```powershell
aws s3 ls
```

Several buckets were returned, including:

```text
adminerstoragebuk
automation-event-buk
aws-security-cognito-01-s3-static-website-hosting
escalation-lambda-bucket
iamawschallenge01buk
lambdamgmtbuk
...
```

An interesting bucket named:

```text
iamawschallenge01buk
```

was investigated and contained a flag.

However, this was not the answer to the current question. The actual objective was to identify the permission assigned to the authenticated IAM user.

This is a useful reminder that cloud enumeration can reveal multiple interesting resources that are unrelated to the current objective.

---

# Step 3 - IAM Policy Simulation

AWS provides the `simulate-principal-policy` API to evaluate whether a specific principal would be allowed to perform one or more AWS actions.

The simulation does not execute the requested action.

Instead, it asks the IAM authorization engine:

```text
Would this principal be allowed to perform this action?
```

The target principal is the currently authenticated user:

```text
arn:aws:iam::058264439561:user/DevAppUser
```

---
# Step 4 - Enumerate Permissions with IAM Policy Simulation

A list of permissions can be created for testing.

Example `policy.txt`:

```text
ec2:StartInstances
s3:ListBucket
iam:CreateUser
```

The permissions can then be tested individually using PowerShell:

```powershell
$permissions = Get-Content "policy.txt" |
    ForEach-Object { $_.Trim() } |
    Where-Object { $_ -ne "" }

foreach ($permission in $permissions) {

    $result = aws iam simulate-principal-policy `
        --policy-source-arn arn:aws:iam::058264439561:user/DevAppUser `
        --action-names $permission

    Write-Output "Permission: $permission"
    Write-Output $result
    Write-Output "`n"
}
```

For manual testing, the same API can be called directly:

```powershell
aws iam simulate-principal-policy `
    --policy-source-arn arn:aws:iam::058264439561:user/DevAppUser `
    --action-names s3:ListAllMyBuckets s3:GetObject ec2:DescribeInstances iam:ListRoles
```

---

# Step 5 - Analyze the Simulation Result

The simulation returned results similar to:

```json
{
    "EvalActionName": "s3:ListAllMyBuckets",
    "EvalResourceName": "*",
    "EvalDecision": "allowed",
    "MatchedStatements": [
        {
            "SourcePolicyId": "AmazonS3ReadOnlyAccess",
            "SourcePolicyType": "IAM Policy"
        }
    ]
}
```

The same policy was also identified for:

```text
s3:GetObject
```

while unrelated actions such as:

```text
iam:ListRoles
ec2:DescribeInstances
lambda:ListFunctions
```

returned:

```text
implicitDeny
```

---

# Finding

The important field was:

```text
SourcePolicyId: AmazonS3ReadOnlyAccess
```

Therefore, the permission assigned to the authenticated user was:

```text
AmazonS3ReadOnlyAccess
```

This was the answer required by the challenge.

---

# Understanding EvalDecision

`EvalDecision` represents the result of AWS's authorization evaluation.

There are three important outcomes.

## allowed

The requested action is permitted.

Example:

```text
Action:
s3:GetObject

EvalDecision:
allowed
```

This means that the IAM evaluation found an applicable Allow statement.

---

## implicitDeny

No applicable Allow statement was found.

For example:

```text
Action:
ec2:DescribeInstances

EvalDecision:
implicitDeny
```

The user did not have a policy granting the requested EC2 action.

This can be summarized as:

```text
No applicable Allow
        ↓
implicitDeny
```

---

## explicitDeny

An applicable policy explicitly denies the action.

For example:

```json
{
    "Effect": "Deny",
    "Action": "s3:*",
    "Resource": "*"
}
```

An explicit Deny takes precedence over an Allow.

Therefore:

```text
Allow + Explicit Deny
        ↓
explicitDeny
```

---

# Understanding SourcePolicyId

`SourcePolicyId` identifies the policy associated with the statement that produced the simulation result.

In this challenge:

```text
SourcePolicyId:
AmazonS3ReadOnlyAccess
```

This is more useful than simply knowing:

```text
s3:GetObject = allowed
```

because it tells us where the permission came from.

The relationship can be represented as:

```text
DevAppUser
    |
    +---- AmazonS3ReadOnlyAccess
                |
                +---- S3 permissions
```

---

# What is AmazonS3ReadOnlyAccess?

`AmazonS3ReadOnlyAccess` is an AWS Managed Policy.

A managed policy is a reusable permission policy that can be attached to different IAM identities.

Conceptually:

```text
                 Managed Policy
                       |
        +--------------+--------------+
        |              |              |
       User          Group           Role
```

The policy itself is not a user, group, or role.

It is a permission definition that can be attached to them.

---

# IAM User vs Group vs Role

Understanding this distinction is important when analyzing AWS IAM.

## IAM User

An IAM User represents an identity inside an AWS account.

Example:

```text
DevAppUser
```

An IAM User can have policies attached directly to it and can also belong to IAM Groups.

Historically, IAM Users were commonly used for human identities. Modern AWS environments often use centralized identity systems and temporary role sessions instead.

---

## IAM Group

A Group is a collection of IAM Users.

For example:

```text
Developers
    |
    +---- Alice
    +---- Bob
    +---- Charlie
```

A policy attached to the group applies to its members.

Groups are useful when several IAM Users should receive the same permissions.

For example:

```text
Developers Group
        |
        +---- AmazonS3ReadOnlyAccess
```

All users in the group receive the permissions provided by that policy.

---

## IAM Role

A Role is an identity that can be assumed by another principal.

Unlike an IAM User, a Role is not primarily intended to represent a permanent individual identity.

A Role can be assumed by:

* IAM Users
* AWS services
* Applications
* Workloads
* Principals from another AWS account

For example:

```text
Lambda Function
       |
       | AssumeRole
       v
LambdaExecutionRole
       |
       +---- S3 permissions
```

This is why Roles are heavily used for AWS workloads.

---

# Permission Policy vs Trust Policy

A Role has two different security questions associated with it.

### Permission Policy

Answers:

> What can this Role do?

For example:

```text
Role
 |
 +---- s3:GetObject
 +---- s3:PutObject
```

### Trust Policy

Answers:

> Who is allowed to assume this Role?

For example:

```text
Lambda Service
       |
       | allowed to assume
       v
LambdaExecutionRole
```

Therefore:

```text
Permission Policy
    =
What can the identity do?

Trust Policy
    =
Who can become/use this identity?
```

This distinction is critical when analyzing AWS privilege escalation paths.

# Why Does IAM Policy Simulation Exist?

At first glance, IAM Policy Simulation may seem unnecessary.

If a user is allowed to perform:

```text
s3:GetObject
```

they can simply perform the operation and observe the result.

However, simulation serves a different purpose.

It allows administrators and security teams to evaluate authorization decisions without actually executing potentially dangerous operations.

For example, an administrator may want to know:

```text
Can this Role terminate EC2 instances?
```

Executing:

```text
ec2:TerminateInstances
```

just to find out would obviously be inappropriate.

Instead, the administrator can simulate the action.

The simulator answers the authorization question without performing the operation.

---

# Simulation vs Actual Execution

These are fundamentally different operations.

### Simulation

```text
simulate-principal-policy
        |
        v
IAM authorization evaluation
        |
        v
allowed / implicitDeny / explicitDeny
```

### Actual AWS Request

```text
AWS API request
        |
        v
IAM authorization evaluation
        |
        v
Action is actually performed
```

Simulation does not grant the permission and does not bypass IAM restrictions.

It only exposes the result of the authorization evaluation for the requested test.

---

# Why Can't a User Always See Its Own Permissions?

This can initially seem strange.

If a user has:

```text
s3:GetObject
```

why should that user not be able to inspect the policy that grants it?

The reason is that IAM permissions are themselves permissions.

For example, these are separate capabilities:

```text
s3:GetObject
```

and:

```text
iam:ListAttachedUserPolicies
```

Having the first does not imply having the second.

This follows the principle of least privilege.

A workload that only needs to read objects from S3 does not necessarily need permission to enumerate IAM roles, policies, users, or other identities in the account.

This can reduce unnecessary information disclosure during a compromise.

However, this does not mean that permission visibility is inherently bad.

If an administrator or security tool needs to inspect IAM configuration, the appropriate IAM permissions can be granted explicitly.

---

# Why Use Groups if Roles Exist?

Groups and Roles solve different problems.

A Group answers:

> Which IAM Users should receive this common set of permissions?

A Role answers:

> Which principal is allowed to assume this identity and temporarily operate with its permissions?

For example:

```text
IAM Users
    |
    v
Developers Group
    |
    v
Common Permissions
```

versus:

```text
Principal
    |
    | AssumeRole
    v
DeveloperRole
    |
    v
Role Permissions
```

Groups are therefore useful for organizing permissions for traditional IAM Users, while Roles are particularly important for workload identities, cross-account access, and temporary access.

Modern AWS environments often use centralized identity management and Role-based access instead of creating long-lived IAM Users for every human employee.

---

# Security Impact

The challenge demonstrates an important cloud security principle:

> An attacker does not necessarily need permission to modify IAM policies in order to learn useful information about an identity's effective permissions.

If an identity has access to `iam:SimulatePrincipalPolicy`, it may be possible to test a large set of AWS actions and determine which operations are allowed.

For an attacker with compromised credentials, this can make post-compromise reconnaissance easier.

However, it is important to distinguish this from privilege escalation.

The ability to simulate a permission does not mean that the permission can actually be used.

For example:

```text
simulate ec2:TerminateInstances
        |
        v
allowed
```

means that the IAM evaluation says the action would be allowed.

The simulator itself did not terminate anything.

---

# Attack Chain

```text
Provided AWS Credentials
            |
            v
Identify Current IAM User
            |
            v
Attempt Direct IAM Policy Enumeration
            |
            v
          Denied
            |
            v
Use simulate-principal-policy
            |
            v
Test AWS Actions
            |
            v
Analyze EvalDecision
            |
            v
Identify SourcePolicyId
            |
            v
AmazonS3ReadOnlyAccess
```

---

# Findings

## Finding 1 - IAM Policy Enumeration Restriction

The current user could not directly enumerate its own IAM policies:

```text
iam:ListUserPolicies
iam:ListAttachedUserPolicies
```

returned `AccessDenied`.

---

## Finding 2 - Permission Discovery Through Simulation

The user could use IAM Policy Simulation to evaluate selected AWS actions.

The simulation revealed:

```text
SourcePolicyId:
AmazonS3ReadOnlyAccess
```

---

## Finding 3 - Effective S3 Access

The simulation showed that actions such as:

```text
s3:ListAllMyBuckets
s3:GetObject
```

were allowed.

Other tested actions such as:

```text
ec2:DescribeInstances
iam:ListRoles
lambda:ListFunctions
```

returned:

```text
implicitDeny
```

---

# Defensive Recommendations

* Apply the principle of least privilege.
* Grant `iam:SimulatePrincipalPolicy` only where required.
* Regularly review IAM policies and permission boundaries.
* Monitor IAM-related API calls using CloudTrail.
* Review identities that can enumerate or modify IAM configuration.
* Prefer temporary credentials and centralized identity management for human access where appropriate.
* Avoid unnecessary long-lived IAM credentials.
* Regularly test effective permissions rather than relying only on policy documents.

---

# Key Takeaways

* `simulate-principal-policy` evaluates IAM authorization without executing the requested action.
* `EvalDecision` represents the result of the IAM authorization evaluation.
* `allowed` means an applicable Allow was found.
* `implicitDeny` means no applicable Allow was found.
* `explicitDeny` means an explicit Deny overrides an Allow.
* `SourcePolicyId` identifies the policy responsible for a matching decision.
* `AmazonS3ReadOnlyAccess` is a Managed Policy, not a User, Group, or Role.
* IAM Groups organize permissions for multiple IAM Users.
* IAM Roles are assumable identities and are heavily used for workloads and temporary access.
* Role Permission Policies define what a Role can do.
* Role Trust Policies define who can assume the Role.
* Permission enumeration is not the same as privilege escalation.
* The presence of `iam:SimulatePrincipalPolicy` does not itself grant the permissions being simulated.

---

# Conclusion

The challenge was solved by using AWS IAM Policy Simulation to enumerate the effective permissions of the authenticated `DevAppUser`.

Direct IAM policy enumeration was blocked, but simulation revealed that the identity had permissions provided by:

```text
AmazonS3ReadOnlyAccess
```

The most important lesson from this challenge is not the flag itself, but understanding how AWS separates:

```text
Identity
    |
    +---- Permissions
    |
    +---- Permission Evaluation
    |
    +---- Trust Relationships
```

This distinction is essential when performing AWS Cloud Security assessments and analyzing potential privilege escalation paths.

---

# References

* AWS IAM Policy Evaluation Logic
* AWS IAM Policy Simulator
* AWS IAM `simulate-principal-policy`
* AWS IAM Managed Policies
* AWS IAM Roles and Trust Policies
