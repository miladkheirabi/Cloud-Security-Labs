# AWS IAM Enumeration and Role Discovery

## Challenge Overview

### Objective

Determine the role associated with the `backend-developer` IAM user and extract additional information from the related IAM policies.

The final flag required:

1. The role name associated with `backend-developer`
2. The `DefaultVersionId` of `BackendAssumeManagerRolePolicy`
3. The inline policy name attached to `ManagerRole`

The three values were concatenated and Base64 encoded.

### Scenario

Initial access was provided through an AWS Access Key ID and Secret Access Key belonging to the `frontend-developer` IAM user.

The goal was to enumerate the IAM environment and identify the relationship between the `backend-developer` user and an IAM role.

### Skills Practiced

* AWS IAM enumeration
* IAM User / Group / Role / Policy analysis
* Managed vs Inline policies
* `sts:AssumeRole` analysis
* AWS CLI
* Base64 encoding

---

# Environment

## Cloud Provider

AWS

## Initial Access

Access was provided through:

* AWS Access Key ID
* AWS Secret Access Key

The credentials authenticated as:

```
arn:aws:iam::058264439561:user/frontend-developer
```

## Available Tools

* AWS CLI
* AWS IAM
* AWS STS

---

# Attack Methodology

## Initial Hypothesis

The provided `frontend-developer` credentials were expected to have limited permissions but potentially enough IAM permissions to enumerate the AWS environment.

The main targets were:

* IAM Users
* IAM Groups
* IAM Roles
* IAM Policies
* Role/User relationships

---

# Attack Workflow

## Step 1 - Initial Access Validation

### Action

Configure the AWS CLI and verify the authenticated identity.

Command:

```
    aws configure

    aws sts get-caller-identity
```
Result:

    {
        "UserId": "AIDAQ3EGUZME2FNJH2PVX",
        "Account": "058264439561",
        "Arn": "arn:aws:iam::058264439561:user/frontend-developer"
    }

### Observation

The provided credentials belong to the `frontend-developer` IAM user.

---

## Step 2 - Enumerate Current User Policies

### Goal

Determine which permissions are directly assigned to the current user.

Command:

    aws iam list-attached-user-policies --user-name frontend-developer

Result:

    {
        "AttachedPolicies": []
    }

No managed policies were directly attached.

Enumerate inline policies:

    aws iam list-user-policies --user-name frontend-developer

Result:

    {
        "PolicyNames": [
            "FrontendReadOnlyPolicy"
        ]
    }

Retrieve the policy:

    aws iam get-user-policy --user-name frontend-developer --policy-name FrontendReadOnlyPolicy

Important permissions included:

    iam:ListUsers
    iam:ListGroups
    iam:ListRoles
    iam:ListPolicies
    iam:GetRole
    iam:GetRolePolicy
    iam:ListRolePolicies
    iam:GetPolicyVersion
    iam:ListAttachedUserPolicies
    iam:ListAttachedGroupPolicies
    iam:ListAttachedRolePolicies
    iam:ListGroupsForUser

### Finding

The current user had extensive IAM enumeration capabilities despite not having administrative access.

---

## Step 3 - Enumerate Groups

### Goal

Determine whether the current user received additional permissions through group membership.

Command:

    aws iam list-groups-for-user --user-name frontend-developer

Result:

    Developers

Enumerate group policies:

    aws iam list-attached-group-policies --group-name Developers

Result:

    AmazonS3ReadOnlyAccess
    S3ListBucketPolicy

No inline group policies were present:

    aws iam list-group-policies --group-name Developers

Result:

    {
        "PolicyNames": []
    }

---

## Step 4 - Enumerate IAM Roles

### Goal

Identify potentially relevant roles in the AWS account.

Command:

    aws iam list-roles --query "Roles[].RoleName" --output table

Among the discovered roles:

    ManagerRole
    DBAdmin
    cloudcorp-web-prod-role
    privesc-lambda-role
    lowpriv-lambda-role
    LambdaExecutionRole

`ManagerRole` was identified as a potentially relevant role.

---

## Step 5 - Enumerate IAM Policies

### Goal

Identify policies that could reveal relationships between users and roles.

Command:

    aws iam list-policies --scope Local --query "Policies[].PolicyName" --output table

Important policies included:

    BackendS3ReadOnlyPolicy
    BackendAssumeManagerRolePolicy
    Manager_Access_S3
    ManagerListUsersAndPolicies
    S3ListBucketPolicy
    PermissionBoundaryPolicy

The most interesting policy was:

    BackendAssumeManagerRolePolicy

---

## Step 6 - Identify the Backend Developer Role

### Goal

Determine which role the `backend-developer` user could assume.

Enumerate policies attached to the user:

    aws iam list-attached-user-policies --user-name backend-developer

Result:

    BackendS3ReadOnlyPolicy
    BackendAssumeManagerRolePolicy

Retrieve the policy metadata:

    aws iam get-policy --policy-arn arn:aws:iam::058264439561:policy/BackendAssumeManagerRolePolicy

Important result:

    PolicyName: BackendAssumeManagerRolePolicy
    DefaultVersionId: v1
    Description: Allow backend developer to assume the manager role

Retrieve the policy document:

    aws iam get-policy-version --policy-arn arn:aws:iam::058264439561:policy/BackendAssumeManagerRolePolicy --version-id v1

Relevant policy statement:

    Action: sts:AssumeRole
    Effect: Allow
    Resource: arn:aws:iam::058264439561:role/ManagerRole

### Finding

The policy explicitly allows:

    backend-developer
            |
            | sts:AssumeRole
            v
        ManagerRole

Therefore:

    Flag 1 = ManagerRole

---

## Step 7 - Determine the Default Policy Version

The `BackendAssumeManagerRolePolicy` metadata showed:

    DefaultVersionId = v1

Therefore:

    Flag 2 = v1

---

## Step 8 - Identify ManagerRole Inline Policy

### Goal

Determine the name of the inline policy attached to `ManagerRole`.

Command:

    aws iam list-role-policies --role-name ManagerRole

Result:

    {
        "PolicyNames": [
            "ManagerRole-inline-policy"
        ]
    }

Therefore:

    Flag 3 = ManagerRole-inline-policy

The role had no managed policies directly attached:

    aws iam list-attached-role-policies --role-name ManagerRole

Result:

    {
        "AttachedPolicies": []
    }

---

# Attack Chain

    AWS Credentials
          |
          v
    frontend-developer
          |
          v
    IAM Enumeration
          |
          v
    Enumerate Users / Roles / Policies
          |
          v
    backend-developer
          |
          v
    BackendAssumeManagerRolePolicy
          |
          | sts:AssumeRole
          v
    ManagerRole
          |
          v
    ManagerRole-inline-policy

---

# Findings

## Security Weakness

The `frontend-developer` account had broad IAM enumeration permissions.

Additionally, the `backend-developer` user had an explicit `sts:AssumeRole` permission for `ManagerRole`.

This exposed an important privilege relationship within the AWS environment.

## Impact

An attacker obtaining the `frontend-developer` credentials could enumerate a significant portion of the IAM environment and discover:

* IAM users
* IAM groups
* IAM roles
* IAM policies
* Role relationships
* Potential privilege escalation paths

The `backend-developer` permissions also revealed that it could request access to `ManagerRole`.

---

# Flag Generation

The three values were:

    Flag1 = ManagerRole
    Flag2 = v1
    Flag3 = ManagerRole-inline-policy

Concatenate them:

    ManagerRole+v1+ManagerRole-inline-policy

Base64 encoded value:

    TWFuYWdlclJvbGUrdjErTWFuYWdlclJvbGUtaW5saW5lLXBvbGljeQ==

Final flag:

    CWL{TWFuYWdlclJvbGUrdjErTWFuYWdlclJvbGUtaW5saW5lLXBvbGljeQ==}

---

# Defensive Recommendations

* Apply least privilege to IAM users.
* Avoid granting unnecessary IAM enumeration permissions.
* Regularly review `sts:AssumeRole` permissions.
* Review IAM role trust policies.
* Monitor `AssumeRole` activity through CloudTrail.
* Regularly audit inline and managed policies.
* Remove unused IAM credentials and policies.
* Use temporary credentials where possible.
* Use IAM Access Analyzer to identify unintended access paths.

---

# Key Takeaways

* `list-user-policies` enumerates inline policies attached directly to a user.
* `list-attached-user-policies` enumerates managed policies attached directly to a user.
* `list-role-policies` enumerates inline policies attached to a role.
* `list-attached-role-policies` enumerates managed policies attached to a role.
* IAM enumeration can reveal useful attack paths without administrative privileges.
* `sts:AssumeRole` permissions can reveal relationships between IAM users and roles.
* When analyzing `AssumeRole`, both the identity policy and the target role's trust policy should be considered.
* IAM enumeration is one of the first things to perform after obtaining AWS credentials.

---

# References

* AWS IAM
* AWS STS
* AWS CLI IAM Commands
* Pacu
* enumerate-iam
