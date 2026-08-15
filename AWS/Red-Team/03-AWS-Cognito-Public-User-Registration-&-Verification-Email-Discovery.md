# AWS Cognito Public User Registration & Verification Email Discovery

## Challenge Overview

### Objective

The objective of this challenge was to identify the email address used by Secure Corp to send verification codes through Amazon Cognito.

The attack involved discovering the Cognito Hosted UI, extracting the associated App Client ID, and using the AWS CLI to register a new user. This registration triggered a verification email, allowing the destination email address to be identified.

### Scenario

Secure Corp uses AWS Cognito to secure one of its applications.

The challenge provides a public web application hosted on Amazon S3 Static Website Hosting. The application contains a Job Portal Login button that redirects users to an AWS Cognito Hosted UI.

The Cognito App Client permits public user registration. By identifying the exposed Client ID and using the AWS CLI, a new Cognito user can be registered. The registration process triggers a verification email, and the destination address of that email is the required flag.

### Skills Practiced

* AWS Cognito reconnaissance
* AWS CLI usage
* Cognito App Client enumeration
* Cognito user registration
* Verification email workflow analysis

---

# Environment

## Cloud Provider

AWS

## Initial Access

The challenge initially provided a public S3 static website.

The website contained a Job Portal Login button.

Clicking the login button redirected to an AWS Cognito Hosted UI.

The Cognito authorization URL exposed the following Client ID:

```
6h6b6gvm11k0eis3l4vhkhgi67
```

No valid AWS IAM credentials were required to register a user through the Cognito App Client.

## Available Tools

* Web Browser
* AWS CLI

---

# Attack Methodology

## Initial Hypothesis

The challenge description specifically mentioned AWS CLI and Cognito.

After following the Job Portal Login functionality, the application redirected to the Cognito Hosted UI. The resulting URL contained a client_id parameter.

The important clue was that the Cognito Hosted UI also exposed a Sign up option.

This suggested that the Cognito App Client might allow unauthenticated user registration and that the same functionality could potentially be accessed directly through the AWS CLI.

Expected targets:

* Cognito App Client
* Cognito SignUp functionality
* User registration workflow
* Verification email
* Secure Corp email address

---

# Attack Workflow

The attack was performed by identifying the Cognito App Client and using its public registration functionality to trigger a verification email.

## Step 1 - Initial Access Validation

### Action

The provided challenge URL was accessed through a web browser.

The website contained a Job Portal Login button. Clicking it redirected the browser to an AWS Cognito Hosted UI.

The Cognito login URL was:

```
https://a-ws-security-cog-nito-01.auth.ap-south-1.amazoncognito.com/login?client_id=6h6b6gvm11k0eis3l4vhkhgi67&response_type=code&scope=email+openid&redirect_uri=https%3A%2F%2Finfinity.cyberwarfare.live
```

The resulting URL contained the following Client ID:

```
6h6b6gvm11k0eis3l4vhkhgi67
```

### Observation

The Client ID is not a secret, but it identifies the Cognito App Client used by the application.

The Cognito Hosted UI also provided a Sign up option, indicating that self-service registration was enabled.

---

## Step 2 - Cognito Registration Enumeration

### Goal

The goal was to determine whether a new user could be registered directly through the Cognito API without AWS IAM permissions.

The AWS CLI was used with the discovered Client ID.

### Command

```aws
aws cognito-idp sign-up --client-id '<CLIENT_ID>' --username '<USERNAME>' --password '<PASSWORD>' --user-attributes '[{"Name":"email","Value":"<EMAIL_ADDRESS>"}]' --region ap-south-1
```

The important parameters were:

* client-id: The Cognito App Client ID discovered from the login URL.
* username: A new username selected for the test account.
* password: A password satisfying the Cognito password policy.
* user-attributes: The email address associated with the new account.
* region: ap-south-1.

### Result

The registration request was accepted by Cognito and the user-verification workflow was triggered.

### Findings

* Public registration was enabled.
* The Cognito Client ID was sufficient to initiate registration.
* AWS IAM credentials were not required for the SignUp operation.
* A user-controlled email address could be supplied during registration.
* Cognito generated a verification email as part of the registration process.

---

## Step 3 - Trigger Verification Email

### Goal

The goal was to trigger the Cognito verification-code email and inspect the destination address.

The registration request included an email attribute:

```aws
[{"Name":"email","Value":"<EMAIL_ADDRESS>"}]
```

After the account was registered, Cognito sent a verification email.

### Action

The received verification email was inspected.

The recipient address in the To field was checked.

### Result

The email revealed the address used by Secure Corp to send verification codes.

### Finding

The email address shown in the To field was the required challenge flag.

---

# Attack Chain

```
    Public S3 Website
            |
            v
    Job Portal Login
            |
            v
    AWS Cognito Hosted UI
            |
            v
    Extract Client ID
            |
            v
    Cognito SignUp API
            |
            v
    Register New User
            |
            v
    Trigger Verification Email
            |
            v
    Inspect To Address
            |
            v
    Challenge Flag
```

---

## Impact

An attacker could:

* Discover the Cognito Client ID.
* Register new accounts.
* Trigger verification emails.
* Interact directly with the Cognito authentication workflow.
* Potentially abuse the registration functionality for email or account-related attacks.

In this challenge, the verification email exposed the email address used by Secure Corp, which was the required flag.

---

# Evidence

The Cognito Hosted UI exposed the following Client ID:

```
6h6b6gvm11k0eis3l4vhkhgi67
```

The registration was performed using:

```aws
aws cognito-idp sign-up --client-id '<CLIENT_ID>' --username '<USERNAME>' --password '<PASSWORD>' --user-attributes '[{"Name":"email","Value":"<EMAIL_ADDRESS>"}]' --region ap-south-1
```

The resulting verification email contained the target email address in the To field.

---

# Defensive Recommendations

* Disable self-service registration if it is not required.
* Review Cognito User Pool registration settings.
* Apply appropriate rate limiting to registration and verification workflows.
* Monitor abnormal account-registration activity.
* Monitor repeated verification-email requests.
* Implement email abuse protections.
* Regularly review Cognito App Client configuration.
* Apply least-privilege principles to all Cognito-related application components.

---

# Key Takeaways

* AWS Cognito Client IDs are public identifiers and should not be treated as secrets.
* A public Cognito Hosted UI can reveal useful information about the authentication architecture.
* The presence of a Sign up option is an important clue during Cognito reconnaissance.
* Cognito functionality can often be accessed directly through AWS CLI/API operations.
* AWS CLI enumeration should not be limited to operations requiring IAM credentials.
* When investigating Cognito, always consider the User Pool, App Client, Hosted UI, SignUp, SignIn, and verification workflows.
* Verification emails can disclose information about the application's configured email infrastructure.

---

# References

* AWS Cognito User Pools
* AWS Cognito SignUp API
* AWS CLI Cognito Identity Provider commands
