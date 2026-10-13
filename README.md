# IAM Workshop for Beginners - Silver Oak University

## Overview

This hands-on lab is designed for students at **Silver Oak University** to gain foundational experience with identity and access management in the cloud using **AWS IAM**.

In this workshop, you will learn how to:
- Create IAM users, groups, and attach permissions
- Understand the structure of IAM policies (JSON documents)
- Write and attach a custom customer-managed policy
- Manage IAM identities using AWS CLI commands
- Create IAM roles and attach them to EC2 instances
- Retrieve temporary role credentials from instance metadata
- Apply least-privilege best practices
- Clean up IAM resources to keep your account secure

---

## Important: IAM is a Global Service

**IMPORTANT:** IAM is a **global** service - users, groups, roles, and policies are not tied to a region. The IAM API endpoint lives in `us-east-1`.

For the EC2 resources used in **Stage 3**, make sure you are working within the **US-EAST-1 (N. Virginia)** region for consistency and service availability.

---

## Access the Lab Environment

Before starting the workshop, access the lab through CloudKida:

1. Navigate to the CloudKida platform: **https://prod.cloudkida.com/**
2. **Sign up** for a new account or **Login** if you already have one
3. Open the IAM lab from your lab list (your instructor will share the direct lab link)
4. Follow the lab instructions to get started with your AWS environment

---

## Workshop Scenario

You are a junior cloud administrator at a university lab team. A group of new students is joining the cloud program, and each one needs AWS console access with **only** the permissions required for their coursework - nothing more.

Your job is to set up these identities properly: create users, group them by role, attach the right policies, and give applications running on EC2 their own permissions through IAM roles instead of hard-coded access keys.

---

## Workshop Structure

This workshop is divided into **3 progressive stages**:

### **Stage 1: IAM Users, Groups, and Policies**
Learn the fundamentals of IAM by creating users and groups, attaching AWS managed policies, and writing your first custom policy through the AWS Management Console.

**[→ Go to Stage 1](./Stage-1-IAM-Users-Groups/README.md)**

---

### **Stage 2: Exploring IAM CLI Commands**
Master AWS CLI commands to manage IAM identities programmatically. Learn to create, inspect, modify, and delete users, groups, policies, and access keys from the command line.

**[→ Go to Stage 2](./Stage-2-IAM-CLI/README.md)**

---

### **Stage 3: IAM Roles and Instance Profiles**
Explore advanced IAM features including roles, trust policies, and instance profiles. Attach a role to an EC2 instance and read temporary credentials from the metadata service.

**[→ Go to Stage 3](./Stage-3-IAM-Roles/README.md)**

---

## Prerequisites

Before starting this workshop, ensure you have:

- AWS Account with administrator (or IAM full access) permissions
- Access to AWS Management Console
- Internet browser (Chrome, Firefox, Safari, or Edge)
- A terminal with AWS CLI v2 installed (for Stage 2)
- An SSH client and a running EC2 instance (for Stage 3)
- A text editor for writing JSON policy documents

---

## Key IAM Concepts

Before you start clicking, get familiar with the vocabulary. Every task in this workshop maps back to one of these.

| Concept | What It Is |
|---------|------------|
| **User** | A long-lived identity for a person or application |
| **Group** | A collection of users that share the same permissions |
| **Role** | An identity assumed temporarily by a user, service, or application |
| **Policy** | A JSON document that allows or denies specific actions |
| **AWS Managed Policy** | A pre-built policy maintained by AWS (e.g. `ReadOnlyAccess`) |
| **Customer Managed Policy** | A policy you write and maintain yourself |
| **Trust Policy** | Attached to a role; defines **who** is allowed to assume it |
| **Instance Profile** | A container that lets an EC2 instance use an IAM role |
| **ARN** | Amazon Resource Name - the unique ID of every AWS resource |

### Anatomy of an IAM Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowListingBuckets",
      "Effect": "Allow",
      "Action": "s3:ListAllMyBuckets",
      "Resource": "*"
    }
  ]
}
```

- **Version** - Always use `2012-10-17` (the current policy language version)
- **Sid** - Optional statement ID for your own readability
- **Effect** - Either `Allow` or `Deny`
- **Action** - The API calls being controlled, in `service:Operation` form
- **Resource** - The ARN(s) the statement applies to

**Remember:** Everything in IAM is **denied by default**. An explicit `Deny` always wins over an `Allow`.

---

## Quick Start: Pre-Lab Setup

### 1. **Set an Account Alias (Optional but Recommended)**

By default, IAM users sign in with your 12-digit account ID. An account alias gives them a friendlier URL.

**Steps:**

1. Navigate to the **IAM Console**: https://console.aws.amazon.com/iam/
2. On the **Dashboard**, find **AWS Account** on the right side
3. Next to **Account Alias**, click **Create**
4. Enter an alias, for example: `silveroak-cloudlab`
5. Click **Create alias**

Your sign-in URL becomes:
```
https://silveroak-cloudlab.signin.aws.amazon.com/console
```

**Save this URL** - your students will need it in Stage 1.

---

### 2. **Review the IAM Dashboard**

Take a minute to look around before creating anything.

**Steps:**

1. In the IAM Console, look at the left navigation pane:
   - **User groups** - collections of users
   - **Users** - individual identities
   - **Roles** - identities assumed temporarily
   - **Policies** - permission documents
   - **Identity providers**, **Account settings**, **Access reports**
2. On the Dashboard, check the **Security recommendations** panel
3. Note the current number of users, groups, roles, and policies in the account

---

### 3. **Verify Your AWS CLI Identity (Needed for Stage 2)**

Confirm your CLI is configured and you know which identity you are using.

```bash
aws --version
```

You should see output like:
```
aws-cli/2.x.x Python/3.x.x Linux/x.x.x
```

Now check who you are:

```bash
aws sts get-caller-identity
```

**Expected Output:**
```json
{
    "UserId": "AIDEXAMPLEUSERID",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/your-admin-user"
}
```

If this command fails, configure your credentials:

```bash
aws configure
```

When prompted, enter:
- **AWS Access Key ID:** Your AWS access key
- **AWS Secret Access Key:** Your AWS secret key
- **Default region name:** `us-east-1`
- **Default output format:** `json`

---

## Security Best Practices

Carry these habits through every stage of the workshop:

- **Never use the root account** for daily work - create an IAM user instead
- **Enable MFA** on the root account and all privileged users
- **Grant least privilege** - start with no permissions and add only what is needed
- **Prefer roles over access keys** for anything running on AWS compute
- **Rotate access keys** regularly and delete unused ones
- **Use groups** to assign permissions, not per-user policy attachments
- **Review Access Advisor** to find and remove unused permissions

---

## Support & Resources

- **AWS IAM Documentation:** https://docs.aws.amazon.com/iam/
- **IAM Policy Reference:** https://docs.aws.amazon.com/service-authorization/latest/reference/
- **AWS CLI IAM Reference:** https://docs.aws.amazon.com/cli/latest/reference/iam/
- **IAM Policy Simulator:** https://policysim.aws.amazon.com/
- **Troubleshooting Access Denied:** Read the error message carefully - it names the action and the ARN that was blocked

---

## Lab Cleanup

After completing the workshop, **remember to clean up resources** to keep your account tidy and secure:

- Delete IAM users created during the lab
- Delete access keys and login profiles
- Delete IAM groups
- Detach and delete customer managed policies
- Remove roles and instance profiles
- Terminate any EC2 instances launched in Stage 3

Each stage includes its own cleanup section with exact steps.

---

**Workshop Created by:** CloudKida Labs  
**Last Updated:** October 2026  
**Target Audience:** Beginner Cloud Engineers at Silver Oak University

---

**Ready to begin? Start with [Stage 1: IAM Users, Groups, and Policies →](./Stage-1-IAM-Users-Groups/README.md)**
