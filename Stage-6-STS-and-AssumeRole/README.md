# Stage 6: STS and Assume Role

## Objective

This is the capstone. Roles are not just for services like EC2 - a **user** can step into a role too, temporarily gaining a different set of permissions. The mechanism is **AWS STS (Security Token Service)** and the `sts:AssumeRole` action. By the end of this stage you will create a role, assume it as your user, receive temporary credentials, and understand the **two-way trust** that makes role assumption secure.

---

## Prerequisites

Before starting, make sure you have:
- Completed **Stages 1-5**
- A working AWS CLI configured with your user's access keys
- Permission to create roles and attach user policies (your instructor will confirm)
- Your **account ID** (run `aws sts get-caller-identity --query Account --output text`)

**[← Back to Main Workshop](../README.md)**

---

## What is STS?

**AWS Security Token Service (STS)** is the service that hands out **temporary credentials**. Every time a role is assumed - by EC2 in Stage 5, or by a user here - it is STS doing the work behind the scenes.

Temporary credentials from STS always come as a **set of three**:
- An **Access Key ID** (starts with `ASIA`)
- A **Secret Access Key**
- A **Session Token**

They expire after a set duration (minutes to hours), which is exactly what makes them safer than long-lived keys.

---

## The Two-Way Trust

Role assumption is a **handshake**. Both sides must agree, or it fails:

1. **The role's trust policy** must say *"this user is allowed to assume me."*
2. **The user's own permissions** must include `sts:AssumeRole` for that role.

If either side is missing, you get `AccessDenied`. This two-way requirement is the whole security model - no single policy can unilaterally grant escalation.

```
   User                                   Role
   ----                                   ----
   "I have sts:AssumeRole      <----->    "My trust policy allows
    permission for this role"              this user to assume me"

                  Both agree → STS issues temporary credentials
```

---

## Lab 1: Create a Role That a User Can Assume

Instead of trusting the EC2 service (as in Stage 5), this role trusts a **specific IAM user**.

### Step 1: Get Your Account ID

```bash
aws sts get-caller-identity --query Account --output text
```

Note this 12-digit number - you will use it in the ARNs below. The examples use `123456789012`.

### Step 2: Write the Trust Policy

This says "the user `student-01` is allowed to assume this role." Replace the account ID and user name with yours.

```bash
cat > user-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:user/student-01"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
```

### Step 3: Create the Role

```bash
aws iam create-role \
  --role-name s3-power-user-role \
  --assume-role-policy-document file://user-trust-policy.json \
  --description "Elevated S3 access, assumable by a specific user"
```

### Step 4: Give the Role Some Permissions

```bash
aws iam attach-role-policy \
  --role-name s3-power-user-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
```

Now the role **can** do a lot with S3 - but nobody can use it yet, because the user side of the handshake is still missing.

---

## Lab 2: Grant the User Permission to Assume the Role

The trust policy alone is not enough. The user also needs `sts:AssumeRole` permission on **their** side.

### Step 1: Write the User Permission

```bash
cat > allow-assume.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::123456789012:role/s3-power-user-role"
    }
  ]
}
EOF
```

### Step 2: Attach It to the User

```bash
aws iam put-user-policy \
  --user-name student-01 \
  --policy-name AllowAssumeS3PowerRole \
  --policy-document file://allow-assume.json
```

**Now both sides agree.** The role trusts the user, and the user is allowed to assume the role. The handshake is complete.

---

## Lab 3: Assume the Role

### Step 1: Call AssumeRole

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/s3-power-user-role \
  --role-session-name my-assume-session
```

**Command Breakdown:**
- `assume-role` - asks STS for temporary credentials for the role
- `--role-arn` - the full ARN of the role to assume
- `--role-session-name` - a label for this session (shows up in CloudTrail logs)

**Expected Output:**
```json
{
    "Credentials": {
        "AccessKeyId": "ASIAEXAMPLEKEYID",
        "SecretAccessKey": "wJalrXUtnFEMI/K7MDENGEXAMPLEKEY",
        "SessionToken": "IQoJb3JpZ2luX2VjEExample...",
        "Expiration": "2026-10-13T17:00:00+00:00"
    },
    "AssumedRoleUser": {
        "AssumedRoleId": "AROAEXAMPLEID:my-assume-session",
        "Arn": "arn:aws:sts::123456789012:assumed-role/s3-power-user-role/my-assume-session"
    }
}
```

There are your three temporary credentials, plus an `Expiration` timestamp.

### Step 2: Switch to the Temporary Credentials

Copy the three values from the output into environment variables:

```bash
export AWS_ACCESS_KEY_ID=ASIAEXAMPLEKEYID
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENGEXAMPLEKEY
export AWS_SESSION_TOKEN=IQoJb3JpZ2luX2VjEExample...
```

**Note:** When these three environment variables are set, the CLI uses them instead of your `~/.aws/credentials` user keys.

### Step 3: Confirm You Are Now the Role

```bash
aws sts get-caller-identity
```

**Expected Output:**
```json
{
    "UserId": "AROAEXAMPLEID:my-assume-session",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/s3-power-user-role/my-assume-session"
}
```

The ARN now shows `assumed-role/s3-power-user-role` - you are temporarily operating **as the role**, not as your user. You now have `AmazonS3FullAccess` even if your user never did.

### Step 4: Use the Elevated Permissions

```bash
aws s3 ls
```

This works using the role's permissions, not your user's.

### Step 5: Return to Your Normal Identity

Clear the environment variables to drop the temporary credentials:

```bash
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
```

Confirm you are back to your user:

```bash
aws sts get-caller-identity
```

The ARN shows `user/student-01` again. The temporary session is gone - and it would have expired on its own anyway.

---

## Lab 4: See the Handshake Fail (Optional)

To really understand the two-way trust, break one side and watch it fail.

### Remove the user's permission:

```bash
aws iam delete-user-policy --user-name student-01 --policy-name AllowAssumeS3PowerRole
```

### Try to assume the role again:

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/s3-power-user-role \
  --role-session-name should-fail
```

**Expected Output:**
```
An error occurred (AccessDenied) when calling the AssumeRole operation:
User: arn:aws:iam::123456789012:user/student-01 is not authorized to perform:
sts:AssumeRole on resource: ...
```

Even though the role's trust policy still allows your user, the **user side** of the handshake is now missing - so the assumption fails. Both halves are required, every time.

Re-add the permission if you want to continue experimenting:

```bash
aws iam put-user-policy \
  --user-name student-01 \
  --policy-name AllowAssumeS3PowerRole \
  --policy-document file://allow-assume.json
```

---

## Common STS Use Cases

| Scenario | How STS Helps |
|----------|---------------|
| **Temporary elevation** | A user assumes an admin role only when needed, then drops it |
| **Cross-account access** | A role in account B trusts a user in account A |
| **Federated sign-in** | SAML/OIDC identities exchange tokens for AWS credentials |
| **EC2/Lambda/ECS** | Compute assumes a role automatically (Stage 5) |
| **Break-glass access** | Emergency roles that log every assumption to CloudTrail |

---

## STS and Role Best Practices

- **Keep sessions short** - use `--duration-seconds` to limit how long credentials live
- **Name sessions meaningfully** - `--role-session-name` shows up in CloudTrail for auditing
- **Scope trust policies to exact principals** - never trust `"AWS": "*"`
- **Add an ExternalId** for third-party cross-account roles
- **Monitor `sts:AssumeRole` calls** in CloudTrail to catch unexpected escalation
- **Prefer assuming roles over creating more access keys** - temporary beats permanent

**WARNING:** A role with `AdministratorAccess` and a wide-open trust policy is a full account takeover waiting to happen. Always pin the trusted principal.

---

## Troubleshooting

### Issue: "AccessDenied" when calling AssumeRole

- Check **both** sides: the role's trust policy **and** the user's `sts:AssumeRole` permission
- Verify the role ARN is spelled exactly right, including the correct account ID
- Confirm the principal in the trust policy matches your actual caller ARN

### Issue: "The security token included in the request is invalid"

- One of the three environment variables is wrong or incomplete - re-copy all three
- The `AWS_SESSION_TOKEN` is long; make sure you pasted the whole thing

### Issue: Still showing my user after exporting the variables

- A typo in a variable name means the CLI ignored it - the names must be exact
- Run `env | grep AWS` to confirm all three are set

### Issue: "Invalid principal in policy" when creating the role

- The user named in the trust policy does not exist, or the account ID is wrong
- A newly created user can take a moment before it is referencable

### Issue: Credentials stopped working after a while

- STS temporary credentials expire - check the `Expiration` timestamp
- Run `assume-role` again to get a fresh set

---

## Summary

Successfully completed:
- Learned what AWS STS is and how it issues temporary credentials
- Understood the two-way trust: role trust policy **and** user `sts:AssumeRole` permission
- Created a role trusted by a specific IAM user
- Assumed the role and received temporary `ASIA` credentials with a session token
- Operated as the role, then returned to your normal identity
- (Optional) Broke the handshake and watched the assumption fail

---

## Lab Cleanup

Clear any temporary credentials still in your shell:

```bash
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
```

Remove the role, user policy, and local files:

```bash
# Remove the user's assume permission
aws iam delete-user-policy --user-name student-01 --policy-name AllowAssumeS3PowerRole

# Detach the policy from the role and delete the role
aws iam detach-role-policy \
  --role-name s3-power-user-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
aws iam delete-role --role-name s3-power-user-role

# Remove local files
rm user-trust-policy.json allow-assume.json
```

---

## Congratulations - You've Completed the Workshop!

You've worked through all six stages:

- **Stage 1:** Getting Started with IAM
- **Stage 2:** Users, Groups, and Group Policies
- **Stage 3:** Policies Deep Dive
- **Stage 4:** Permissions in Action
- **Stage 5:** IAM Roles and Instance Profiles
- **Stage 6:** STS and Assume Role

### What You've Learned:

- Signing in, inspecting your identity, and reading your own permissions
- Organizing permissions with users and groups
- Writing AWS managed, customer managed, and inline policies
- Using explicit `Deny` to build guardrails that cannot be overridden
- Performing real tasks: starting/stopping EC2, creating S3 buckets and DynamoDB tables
- Granting permissions to applications via roles and instance profiles
- Using STS to assume roles and work with temporary credentials

### Where to Go Next:

- **MFA and IAM Access Analyzer:** Harden accounts and surface unintended access
- **Permission Boundaries and SCPs:** Set hard limits across many accounts
- **Cross-Account Access:** Share resources safely between AWS accounts
- **Identity Federation and IAM Identity Center:** Connect SAML/OIDC providers
- **Infrastructure as Code:** Define IAM in Terraform or CloudFormation
- **CloudTrail Auditing:** Trace every API call back to an identity

---

**[← Back to Main Workshop](../README.md)**
