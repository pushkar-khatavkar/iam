# Stage 3: IAM Roles and Instance Profiles

## Objective

In this stage, you will explore advanced IAM features including **roles**, **trust policies**, and **instance profiles**. You'll attach a role to an EC2 instance, read temporary credentials from the instance metadata service, and assume a role as an IAM user.

---

## Prerequisites

Before starting, ensure you have:
- Completed **Stage 1: IAM Users, Groups, and Policies**
- Completed **Stage 2: Exploring IAM CLI Commands**
- AWS CLI configured and working
- A running **EC2 instance** in `us-east-1` with SSH access
- Understanding of IAM policies and ARNs

**[← Back to Main Workshop](../README.md)**

---

## Understanding IAM Roles

### What is a Role?

An **IAM role** is an identity with permissions, but **no long-term credentials**. Nobody signs in as a role. Instead, a trusted principal *assumes* it and receives temporary credentials that expire automatically.

Roles are the right answer when:
- An application on EC2, Lambda, or ECS needs to call AWS APIs
- A user needs elevated permissions only occasionally
- Another AWS account needs limited access to your resources
- An external identity provider (SAML, OIDC) federates into AWS

### Key Points:

- Roles issue **temporary credentials** that rotate automatically
- There are **no access keys to leak, commit, or rotate** manually
- Every role has **two** policies: a **trust policy** (who can assume it) and **permissions policies** (what it can do)
- Credentials delivered to EC2 refresh before they expire, with no application changes needed

### Role vs User

| Aspect | IAM User | IAM Role |
|--------|----------|----------|
| Credentials | Long-lived access keys / password | Temporary, auto-expiring |
| Who uses it | A specific person or app | Any trusted principal |
| Sign-in | Yes, via console | No, assumed instead |
| Rotation | Manual | Automatic |
| Best for | Human operators | Applications and cross-account access |

---

## Understanding Trust Policies

A **trust policy** answers one question: *who is allowed to assume this role?*

Example trust policy for an EC2 instance:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

- **Principal** - The entity being trusted. Here it is the EC2 service itself
- **Action** - Always `sts:AssumeRole` for standard roles
- The permissions the role *grants* live in separate policies, not here

Other common principals:

| Use Case | Principal |
|----------|-----------|
| EC2 instance | `{"Service": "ec2.amazonaws.com"}` |
| Lambda function | `{"Service": "lambda.amazonaws.com"}` |
| An IAM user in your account | `{"AWS": "arn:aws:iam::123456789012:user/student-01"}` |
| Another AWS account | `{"AWS": "arn:aws:iam::999988887777:root"}` |

---

## Understanding Instance Profiles

An **instance profile** is the wrapper that lets EC2 hand a role to an instance. You cannot attach a role to an instance directly - it goes through the profile.

- When you create a role in the **console** with the EC2 use case, AWS creates the instance profile automatically
- When you create a role via **CLI** or **CloudFormation**, you must create the instance profile yourself
- An instance profile holds exactly **one** role

```
EC2 Instance → Instance Profile → IAM Role → Permissions Policies
```

---

## Lab 1: Create an EC2 Role via the Console

In this lab, you'll create a role that lets an EC2 instance read from S3, then attach it to a running instance.

### Step 1: Create the Role

1. Go to **IAM Console** → **Roles** → **Create role**

2. **Select trusted entity:**
   - **Trusted entity type:** AWS service
   - **Use case:** EC2
   - Click **Next**

3. **Add permissions** - search for and select:
   - `AmazonS3ReadOnlyAccess`
   - Click **Next**

4. **Name, review, and create:**
   - **Role name:** `ec2-s3-readonly-role`
   - **Description:** `Allows EC2 instances to read from S3`

5. Review the **Trust policy** section and confirm the principal is `ec2.amazonaws.com`

6. Click **Create role**

### Step 2: Attach the Role to Your EC2 Instance

1. Go to **EC2 Console** → **Instances**

2. Select your running instance

3. Click **Actions** → **Security** → **Modify IAM role**

4. From the dropdown, choose `ec2-s3-readonly-role`

5. Click **Update IAM role**

6. Wait about **30 seconds** for the credentials to propagate to the instance

### Step 3: Verify the Role Works

**Step 3.1: Connect to your instance**
```bash
ssh -i ~/Downloads/ssh-web-key.pem ec2-user@<your-public-ip>
```

**Step 3.2: Confirm the instance is now using the role**
```bash
aws sts get-caller-identity
```

**Expected Output:**
```json
{
    "UserId": "AROAEXAMPLEID:i-0123456789abcdef0",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/ec2-s3-readonly-role/i-0123456789abcdef0"
}
```

Notice the ARN says `assumed-role`, and the session name is your instance ID. No access keys were configured anywhere on this machine.

**Step 3.3: Test an allowed action**
```bash
aws s3 ls
```

The bucket list should return successfully.

**Step 3.4: Test a blocked action**
```bash
aws s3 mb s3://silveroak-should-fail-2026
```

This fails with `AccessDenied` - the role grants read-only access.

---

## Lab 2: Read Role Credentials from Instance Metadata

The credentials behind that role are delivered through the EC2 metadata service. In this lab you'll look at them directly.

### Step 1: Connect to Your Instance

```bash
ssh -i ~/Downloads/ssh-web-key.pem ec2-user@<your-public-ip>
```

### Step 2: Query the Metadata Service

Modern EC2 instances use **IMDSv2**, which requires a session token.

**First, obtain a session token:**
```bash
TOKEN=`curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"`
```

**Get the attached role name:**
```bash
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

**Expected Output:**
```
ec2-s3-readonly-role
```

**Get the temporary credentials for that role:**
```bash
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/ec2-s3-readonly-role
```

**Expected Output:**
```json
{
  "Code": "Success",
  "LastUpdated": "2026-10-13T10:00:00Z",
  "Type": "AWS-HMAC",
  "AccessKeyId": "ASIAEXAMPLEKEYID",
  "SecretAccessKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
  "Token": "IQoJb3JpZ2luX2VjEExample...",
  "Expiration": "2026-10-13T16:00:00Z"
}
```

**Get the instance profile ARN:**
```bash
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/info
```

### What You Just Saw:

- `AccessKeyId` starts with `ASIA` instead of `AKIA` - the `ASIA` prefix marks a temporary key
- A `Token` (session token) is required alongside the key and secret
- `Expiration` is only a few hours out; the service refreshes the credentials before then
- The AWS CLI and every AWS SDK pick these up automatically with zero configuration

**Note:** The metadata token is valid for 6 hours (21600 seconds). If it expires, generate a new one.

**WARNING:** Never expose the metadata service externally and never log these credentials. Anything that can reach `169.254.169.254` from your instance can act as the role.

---

## Lab 3: Create a Role and Instance Profile via CLI

Now build the same thing from the command line, where the instance profile is your responsibility.

### Step 1: Write the Trust Policy

```bash
cat > ec2-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
```

### Step 2: Create the Role

```bash
aws iam create-role \
  --role-name ec2-cli-describe-role \
  --assume-role-policy-document file://ec2-trust-policy.json \
  --description "CLI-created role allowing EC2 describe calls"
```

**Expected Output:**
Returns JSON with the role's `RoleName`, `RoleId`, `Arn`, and the trust policy you supplied.

### Step 3: Attach a Permissions Policy

```bash
aws iam attach-role-policy \
  --role-name ec2-cli-describe-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess
```

### Step 4: Create the Instance Profile

```bash
aws iam create-instance-profile --instance-profile-name ec2-cli-describe-profile
```

### Step 5: Add the Role to the Instance Profile

```bash
aws iam add-role-to-instance-profile \
  --instance-profile-name ec2-cli-describe-profile \
  --role-name ec2-cli-describe-role
```

### Step 6: Associate the Profile with an Instance

```bash
aws ec2 associate-iam-instance-profile \
  --instance-id i-YOUR_INSTANCE_ID \
  --iam-instance-profile Name=ec2-cli-describe-profile
```

**Note:** An instance can hold only one instance profile. If one is already attached, replace it instead:

```bash
# Find the existing association ID
aws ec2 describe-iam-instance-profile-associations \
  --filters Name=instance-id,Values=i-YOUR_INSTANCE_ID
```

```bash
# Replace it
aws ec2 replace-iam-instance-profile-association \
  --association-id iip-assoc-YOUR_ASSOCIATION_ID \
  --iam-instance-profile Name=ec2-cli-describe-profile
```

### Step 7: Verify from Inside the Instance

```bash
aws sts get-caller-identity
```

```bash
aws ec2 describe-instances --query 'Reservations[*].Instances[*].[InstanceId,State.Name]' --output table
```

The describe call succeeds. Now confirm the boundary holds:

```bash
aws ec2 terminate-instances --instance-ids i-YOUR_INSTANCE_ID
```

This fails with `UnauthorizedOperation` - read-only means read-only.

---

## Lab 4: Assume a Role as an IAM User

Roles are not just for services. A user can step into elevated permissions temporarily.

### Step 1: Create a Role Trusted by a User

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

Replace `123456789012` with your account ID and `student-01` with your Stage 1 user.

```bash
aws iam create-role \
  --role-name student-escalation-role \
  --assume-role-policy-document file://user-trust-policy.json
```

```bash
aws iam attach-role-policy \
  --role-name student-escalation-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
```

### Step 2: Grant the User Permission to Assume It

The trust policy alone is not enough. The user also needs `sts:AssumeRole` permission on their side.

```bash
cat > allow-assume.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::123456789012:role/student-escalation-role"
    }
  ]
}
EOF
```

```bash
aws iam put-user-policy \
  --user-name student-01 \
  --policy-name AllowAssumeEscalationRole \
  --policy-document file://allow-assume.json
```

**Both sides must agree.** This two-way handshake is what makes role assumption safe.

### Step 3: Assume the Role

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/student-escalation-role \
  --role-session-name student-01-session
```

**Expected Output:**
```json
{
    "Credentials": {
        "AccessKeyId": "ASIAEXAMPLEKEYID",
        "SecretAccessKey": "wJalrXUtnFEMI/K7MDENGEXAMPLEKEY",
        "SessionToken": "IQoJb3JpZ2luX2VjEExample...",
        "Expiration": "2026-10-13T11:00:00+00:00"
    },
    "AssumedRoleUser": {
        "AssumedRoleId": "AROAEXAMPLEID:student-01-session",
        "Arn": "arn:aws:sts::123456789012:assumed-role/student-escalation-role/student-01-session"
    }
}
```

### Step 4: Use the Temporary Credentials

Export the three values from the response:

```bash
export AWS_ACCESS_KEY_ID=ASIAEXAMPLEKEYID
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENGEXAMPLEKEY
export AWS_SESSION_TOKEN=IQoJb3JpZ2luX2VjEExample...
```

```bash
aws sts get-caller-identity
```

The ARN now shows `assumed-role/student-escalation-role/student-01-session`.

Return to your original identity by clearing the variables:

```bash
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
```

---

## Role Best Practices

- **Use roles instead of access keys** for anything running on AWS compute
- **Scope trust policies narrowly** - name the exact principal, never `"AWS": "*"`
- **Add an ExternalId** for third-party cross-account roles
- **Set `--max-session-duration`** to the shortest workable window
- **Never bake credentials into AMIs, userdata, or source code**
- **Audit with Access Advisor** to find permissions a role has never used
- **Monitor `sts:AssumeRole` calls** in CloudTrail for unexpected escalation

**WARNING:** A role with `AdministratorAccess` and a wide-open trust policy is a full account takeover waiting to happen. Always pin the principal.

---

## Comparison: Access Keys vs IAM Roles

| Task | Access Keys (Stage 2) | IAM Roles (Stage 3) |
|------|----------------------|---------------------|
| Credential lifetime | Until manually deleted | Hours, auto-expiring |
| Stored on disk | Yes, in `~/.aws/credentials` | No |
| Rotation | Manual | Automatic |
| Leak impact | Valid until revoked | Expires on its own |
| Setup effort | Low | Moderate |
| Git-commit risk | High | None |
| Best for | Local developer machines | Applications on AWS |

---

## Troubleshooting

### Issue: "An error occurred (AccessDenied) when calling the AssumeRole operation"

- Check **both** sides: the role's trust policy and the user's `sts:AssumeRole` permission
- Verify the role ARN is spelled exactly right, including the account ID
- Confirm the principal in the trust policy matches your actual caller ARN

### Issue: Instance still shows the old identity after attaching a role

- Wait 30 to 60 seconds for credentials to propagate
- Clear any conflicting local config: check `~/.aws/credentials` and unset `AWS_ACCESS_KEY_ID`
- Environment variables and credential files take precedence over the instance role

### Issue: "No role found" from the metadata service

- Confirm a role is attached: **EC2 Console** → instance → **Security** tab
- The trailing slash matters: `.../iam/security-credentials/` lists the role name
- If the endpoint hangs, IMDS may be disabled on the instance

### Issue: "Cannot delete entity, must remove roles from instance profile first"

- Remove the role from the instance profile before deleting either one
- Also disassociate the profile from any instance still using it

### Issue: "Invalid principal in policy"

- The principal ARN does not exist, or the service name is misspelled
- Service principals end in `.amazonaws.com`, for example `ec2.amazonaws.com`
- A newly created user may take a moment to be referencable in a trust policy

---

## Summary

Successfully completed:
- Learned how IAM roles deliver temporary, auto-rotating credentials
- Created an EC2 service role through the console and attached it to an instance
- Read role credentials directly from the IMDSv2 metadata service
- Built a role and instance profile from the CLI, including the trust policy
- Associated and replaced instance profiles on a running instance
- Assumed a role as an IAM user and used the temporary session credentials
- Compared long-lived access keys against role-based access

---

## Lab Cleanup

Once you've completed this stage, clean up resources to keep your account secure.

**Detach the role from your instance:**

1. Go to **EC2 Console** → **Instances**
2. Select your instance
3. Click **Actions** → **Security** → **Modify IAM role**
4. Choose **No IAM role** and confirm

**Delete the CLI-created role and instance profile (in this order):**

```bash
# 1. Disassociate the instance profile
aws ec2 describe-iam-instance-profile-associations \
  --filters Name=instance-id,Values=i-YOUR_INSTANCE_ID

aws ec2 disassociate-iam-instance-profile --association-id iip-assoc-YOUR_ASSOCIATION_ID
```

```bash
# 2. Remove the role from the instance profile
aws iam remove-role-from-instance-profile \
  --instance-profile-name ec2-cli-describe-profile \
  --role-name ec2-cli-describe-role

# 3. Delete the instance profile
aws iam delete-instance-profile --instance-profile-name ec2-cli-describe-profile
```

```bash
# 4. Detach policies from the role
aws iam detach-role-policy \
  --role-name ec2-cli-describe-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess

# 5. Delete the role
aws iam delete-role --role-name ec2-cli-describe-role
```

**Delete the escalation role from Lab 4:**

```bash
aws iam delete-user-policy --user-name student-01 --policy-name AllowAssumeEscalationRole

aws iam detach-role-policy \
  --role-name student-escalation-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

aws iam delete-role --role-name student-escalation-role
```

**Delete the console-created role:**

1. Go to **IAM Console** → **Roles**
2. Select `ec2-s3-readonly-role`
3. Click **Delete** and confirm by typing the role name

**Remove local policy files:**

```bash
rm ec2-trust-policy.json user-trust-policy.json allow-assume.json
```

**Terminate your EC2 instance (if no longer needed):**

1. Go to **EC2 Console** → **Instances**
2. Select your instance
3. Click **Instance State** → **Terminate Instance**

---

## Next Steps

Congratulations! You've completed all three stages of the IAM Workshop:

- **Stage 1:** IAM Users, Groups, and Policies
- **Stage 2:** Exploring IAM CLI Commands
- **Stage 3:** IAM Roles and Instance Profiles

### What You've Learned:

- Creating and organizing IAM users, groups, and policies
- Reading and writing IAM policy JSON, including explicit denies
- Managing identities programmatically with the AWS CLI
- Handling access keys safely, including rotation and deletion
- Granting permissions to applications through roles instead of keys
- Retrieving temporary credentials from instance metadata
- Applying least-privilege thinking to every permission decision

### Next Learning Paths:

- **MFA and Access Analyzer:** Harden accounts and surface unintended access
- **Permission Boundaries and SCPs:** Set hard limits across accounts
- **Cross-Account Access:** Share resources safely between AWS accounts
- **Identity Federation:** Connect SAML or OIDC providers and IAM Identity Center
- **Infrastructure as Code:** Define IAM resources in Terraform or CloudFormation
- **CloudTrail Auditing:** Trace every API call back to an identity

---

**Workshop Created by:** CloudKida Labs  
**For:** Silver Oak University - IAM Beginners Workshop  
**Completed:** You are now an IAM Beginner!

---

**[← Back to Main Workshop](../README.md)**
