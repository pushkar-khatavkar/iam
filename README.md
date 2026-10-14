# AWS IAM Workshop for Beginners

## Overview

This hands-on workshop takes you from zero to confident with **AWS Identity and Access Management (IAM)**. You start with the simplest possible actions - signing in and asking "who am I?" - and build up stage by stage to roles, temporary credentials, and cross-identity access.

In this workshop, you will learn how to:
- Sign in as an IAM user and inspect your own identity
- Read and understand IAM policy JSON documents
- Organize permissions using groups instead of per-user grants
- Work with AWS managed, customer managed, and inline policies
- Use an explicit `Deny` to build guardrails
- Perform real tasks: start/stop EC2 instances, create S3 buckets and DynamoDB tables
- Create IAM roles and attach them to EC2 instances
- Use AWS STS to assume a role and obtain temporary credentials
- Apply least-privilege thinking to every permission decision
- Clean up resources to keep the account tidy and secure

---

## Who This Is For

Anyone learning AWS IAM for the first time - students, bootcamp participants, new cloud engineers, or teams onboarding to AWS. No prior IAM experience is assumed. Each stage builds on the previous one.

---

## Important: How This Workshop Is Set Up

**Your instructor (or account admin) creates one IAM user per participant, each with a console password and, where needed, access keys.** You do **not** create your own login. You sign in with the credentials you are given and complete the labs from there.

A few resources are **pre-created** by the instructor so you can focus on IAM itself:
- One or more **EC2 instances** already exist. You will **start and stop** them - you will **not** launch new ones.
- An IAM **role** and **group structure** may be pre-staged for the later stages.

Resources you **do** create yourself during the workshop:
- **S3 buckets**
- **DynamoDB tables**
- Custom and inline **IAM policies** (where your permissions allow)

---

## Important: IAM is a Global Service

IAM is a **global** service - users, groups, roles, and policies are not tied to a region. The IAM API endpoint lives in `us-east-1`.

For the EC2, S3, and DynamoDB resources used in the later stages, use a **single consistent region** (the examples use `us-east-1`) so everyone sees the same resources.

---

## Workshop Structure

This workshop is divided into **6 progressive stages**. Do them in order.

### **Stage 1: Getting Started with IAM**
The gentlest possible start. Sign in, look around the IAM console, run your first CLI command, and answer the question "who am I and what can I do?"

**[→ Go to Stage 1](./Stage-1-Getting-Started/README.md)**

---

### **Stage 2: Users, Groups, and Group Policies**
Learn how permissions are organized. Explore users and groups, see how a group grants permissions to every member, and move users between groups.

**[→ Go to Stage 2](./Stage-2-Users-and-Groups/README.md)**

---

### **Stage 3: Policies Deep Dive**
Understand the three kinds of policies - AWS managed, customer managed, and inline - and write your own. Use an explicit `Deny` and watch it override an `Allow`.

**[→ Go to Stage 3](./Stage-3-Policies-Deep-Dive/README.md)**

---

### **Stage 4: Permissions in Action**
Put permissions to work on real services. Start and stop a pre-existing EC2 instance, create an S3 bucket, and create a DynamoDB table - then see what happens when a permission is missing.

**[→ Go to Stage 4](./Stage-4-Permissions-In-Action/README.md)**

---

### **Stage 5: IAM Roles and Instance Profiles**
Give applications permissions without access keys. Create a role, attach it to an EC2 instance through an instance profile, and read the temporary credentials from instance metadata.

**[→ Go to Stage 5](./Stage-5-Roles-and-Instance-Profiles/README.md)**

---

### **Stage 6: STS and Assume Role**
The capstone. Use AWS STS to assume a role as an IAM user, receive temporary credentials, and understand the two-way trust that makes role assumption safe.

**[→ Go to Stage 6](./Stage-6-STS-and-AssumeRole/README.md)**

---

## Prerequisites

Before starting this workshop, ensure you have:

- An IAM **user name and console password** provided by your instructor
- The **account sign-in URL** (e.g. `https://<account-alias>.signin.aws.amazon.com/console`)
- Internet browser (Chrome, Firefox, Safari, or Edge)
- **Access keys** for your user (needed from Stage 2 onward for CLI work)
- A terminal with **AWS CLI v2** installed
- A text editor for writing JSON policy documents

### Install AWS CLI v2 (if needed)

Check whether it is already installed:

```bash
aws --version
```

You should see output like:
```
aws-cli/2.x.x Python/3.x.x Linux/x.x.x
```

If not installed, follow the official guide: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

---

## Key IAM Concepts

Get familiar with the vocabulary before you start. Every task in this workshop maps back to one of these.

| Concept | What It Is |
|---------|------------|
| **User** | A long-lived identity for a person or application |
| **Group** | A collection of users that share the same permissions |
| **Role** | An identity assumed temporarily by a user, service, or application |
| **Policy** | A JSON document that allows or denies specific actions |
| **AWS Managed Policy** | A pre-built policy maintained by AWS (e.g. `ReadOnlyAccess`) |
| **Customer Managed Policy** | A reusable policy you write and maintain yourself |
| **Inline Policy** | A policy embedded directly in a single user, group, or role |
| **Explicit Deny** | A `Deny` statement that overrides any `Allow` |
| **Trust Policy** | Attached to a role; defines **who** is allowed to assume it |
| **Instance Profile** | A container that lets an EC2 instance use an IAM role |
| **STS** | Security Token Service - issues temporary credentials |
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

### The Golden Rules of IAM Evaluation

1. **Everything is denied by default.** A brand-new identity can do nothing.
2. **An `Allow` opens up an action.** Permissions are additive across all attached policies.
3. **An explicit `Deny` always wins.** If any policy denies an action, it is blocked - no matter how many Allows exist.

---

## Security Best Practices

Carry these habits through every stage:

- **Never use the root account** for daily work
- **Enable MFA** on privileged users
- **Grant least privilege** - start with nothing and add only what is needed
- **Prefer roles over access keys** for anything running on AWS compute
- **Use groups** to assign permissions, not per-user attachments
- **Rotate access keys** and delete unused ones
- **Never commit access keys** to source control

---

## Support & Resources

- **AWS IAM Documentation:** https://docs.aws.amazon.com/iam/
- **IAM Policy Reference:** https://docs.aws.amazon.com/service-authorization/latest/reference/
- **AWS CLI IAM Reference:** https://docs.aws.amazon.com/cli/latest/reference/iam/
- **AWS STS Reference:** https://docs.aws.amazon.com/cli/latest/reference/sts/
- **IAM Policy Simulator:** https://policysim.aws.amazon.com/
- **Troubleshooting Access Denied:** Read the error carefully - it names the exact action and ARN that was blocked

---

## Lab Cleanup

After completing the workshop, clean up any resources **you** created to keep the account tidy:

- Delete S3 buckets and DynamoDB tables you created
- Detach and delete customer managed policies
- Remove inline policies you added
- Remove roles and instance profiles you created
- Stop any EC2 instances you started

Each stage includes its own cleanup section with exact steps. Resources your instructor pre-created should be left in place unless told otherwise.

---

**Workshop Type:** Beginner → Intermediate, hands-on  
**Last Updated:** October 2026

---

**Ready to begin? Start with [Stage 1: Getting Started with IAM →](./Stage-1-Getting-Started/README.md)**
