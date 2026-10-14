# Stage 5: IAM Roles and Instance Profiles

## Objective

So far every permission belonged to a **user** with long-lived access keys. But what about an application running on an EC2 instance - how does it get permissions without you hard-coding keys into it? The answer is **IAM roles**. In this stage you will create a role, attach it to an EC2 instance through an **instance profile**, and read the temporary credentials straight from the instance metadata service.

---

## Prerequisites

Before starting, make sure you have:
- Completed **Stages 1-4**
- A working AWS CLI configured for `us-east-1`
- SSH access to a pre-created EC2 instance (your instructor provides the key and public IP)
- Permission to create roles and modify an instance's IAM role (your instructor will confirm)

**[← Back to Main Workshop](../README.md)**

---

## Understanding IAM Roles

### What is a Role?

An **IAM role** is an identity with permissions but **no long-term credentials**. Nobody signs in as a role. Instead, a trusted entity *assumes* it and receives **temporary credentials** that expire automatically.

Roles are the right answer when:
- An application on EC2, Lambda, or ECS needs to call AWS APIs
- A user needs elevated permissions only occasionally
- Another AWS account needs limited access to your resources

### Role vs User

| Aspect | IAM User | IAM Role |
|--------|----------|----------|
| Credentials | Long-lived access keys / password | Temporary, auto-expiring |
| Who uses it | A specific person or app | Any trusted entity |
| Sign-in | Yes, via console | No, assumed instead |
| Rotation | Manual | Automatic |
| Best for | Human operators | Applications and cross-account access |

### Every Role Has Two Policy Parts

1. **Trust policy** - answers *"who is allowed to assume this role?"*
2. **Permissions policies** - answer *"what can this role do once assumed?"*

---

## Understanding Instance Profiles

An **instance profile** is the wrapper that lets EC2 hand a role to an instance. You cannot attach a role to an instance directly - it goes through the profile.

```
EC2 Instance → Instance Profile → IAM Role → Permissions Policies
```

- When you create a role in the **console** with the EC2 use case, AWS creates the instance profile **automatically**
- When you create a role via **CLI**, you must create the instance profile **yourself**
- An instance profile holds exactly **one** role

---

## Lab 1: Create an EC2 Role via the Console

The easy path first. The console does the instance-profile plumbing for you.

### Step 1: Create the Role

1. Go to **IAM Console** → **Roles** → **Create role**

2. **Select trusted entity:**
   - **Trusted entity type:** AWS service
   - **Use case:** EC2
   - Click **Next**

3. **Add permissions** - search for and select:
   - `AmazonS3ReadOnlyAccess`
   - Click **Next**

4. **Name and create:**
   - **Role name:** `ec2-s3-readonly-role`
   - **Description:** `Lets EC2 instances read from S3`
   - Click **Create role**

### Step 2: Look at the Trust Policy

1. Click the role you just created
2. Open the **Trust relationships** tab

You should see:

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

Read it: the **EC2 service** (`ec2.amazonaws.com`) is allowed to assume this role. That is what lets an instance use it.

### Step 3: Attach the Role to Your EC2 Instance

1. Go to **EC2 Console** → **Instances**
2. Select the pre-created instance
3. Click **Actions** → **Security** → **Modify IAM role**
4. Choose `ec2-s3-readonly-role` from the dropdown
5. Click **Update IAM role**
6. Wait about **30 seconds** for the credentials to propagate

---

## Lab 2: Verify the Role from Inside the Instance

### Step 1: Connect to the Instance

```bash
ssh -i /path/to/your-key.pem ec2-user@<instance-public-ip>
```

### Step 2: Confirm the Instance Is Using the Role

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

Notice the ARN says `assumed-role`, and the session name is the **instance ID**. No access keys were configured anywhere on this machine - the instance got its identity from the role.

### Step 3: Test an Allowed Action

```bash
aws s3 ls
```

The bucket list returns successfully - the role grants S3 read.

### Step 4: Test a Blocked Action

```bash
aws s3 mb s3://this-should-be-denied-2026
```

This fails with `AccessDenied` - the role is read-only, so creating a bucket is not permitted.

---

## Lab 3: Read Role Credentials from Instance Metadata

The temporary credentials behind the role are delivered through the EC2 metadata service. Let's look at them directly. Modern instances use **IMDSv2**, which requires a session token.

### Step 1: Get a Metadata Session Token

```bash
TOKEN=`curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"`
```

### Step 2: Find the Attached Role Name

```bash
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

**Expected Output:**
```
ec2-s3-readonly-role
```

### Step 3: Read the Temporary Credentials

```bash
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/ec2-s3-readonly-role
```

**Expected Output:**
```json
{
  "Code": "Success",
  "Type": "AWS-HMAC",
  "AccessKeyId": "ASIAEXAMPLEKEYID",
  "SecretAccessKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
  "Token": "IQoJb3JpZ2luX2VjEExample...",
  "Expiration": "2026-10-13T16:00:00Z"
}
```

### What You Just Saw:

- `AccessKeyId` starts with **`ASIA`** instead of `AKIA` - the `ASIA` prefix marks a **temporary** key
- A `Token` (session token) is required alongside the key and secret
- `Expiration` is only a few hours out; the service refreshes the credentials automatically before then
- The AWS CLI and every AWS SDK pick these up automatically with **zero configuration**

**Note:** The metadata token is valid for 6 hours (21600 seconds). If it expires, generate a new one.

**WARNING:** Never expose the metadata service externally and never log these credentials. Anything that can reach `169.254.169.254` from the instance can act as the role.

---

## Lab 4: Create a Role and Instance Profile via CLI (Optional, Advanced)

Now build the same thing from the command line, where creating the instance profile is **your** responsibility.

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
  --description "CLI-created role allowing EC2 and DynamoDB read"
```

### Step 3: Attach Permissions Policies

```bash
aws iam attach-role-policy \
  --role-name ec2-cli-describe-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess
```

### Step 4: Create the Instance Profile

```bash
aws iam create-instance-profile --instance-profile-name ec2-cli-describe-profile
```

### Step 5: Put the Role Inside the Instance Profile

```bash
aws iam add-role-to-instance-profile \
  --instance-profile-name ec2-cli-describe-profile \
  --role-name ec2-cli-describe-role
```

### Step 6: Associate the Profile with the Instance

```bash
aws ec2 associate-iam-instance-profile \
  --instance-id i-YOUR_INSTANCE_ID \
  --iam-instance-profile Name=ec2-cli-describe-profile
```

**Note:** An instance can hold only one instance profile. If one is already attached (from the console lab), replace it instead:

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

---

## Role Best Practices

- **Use roles instead of access keys** for anything running on AWS compute
- **Scope trust policies narrowly** - name the exact principal, never `"AWS": "*"`
- **Grant least privilege** in the permissions policies
- **Never bake credentials into AMIs, userdata, or source code**
- **Audit with Access Advisor** to find permissions a role has never used

---

## Comparison: Access Keys vs IAM Roles

| Aspect | Access Keys (user) | IAM Role (instance) |
|--------|--------------------|---------------------|
| Credential lifetime | Until manually deleted | Hours, auto-expiring |
| Stored on disk | Yes, in `~/.aws/credentials` | No |
| Rotation | Manual | Automatic |
| Leak impact | Valid until revoked | Expires on its own |
| Git-commit risk | High | None |
| Best for | Local developer machines | Applications on AWS |

---

## Troubleshooting

### Issue: Instance still shows the old identity after attaching a role

- Wait 30-60 seconds for credentials to propagate
- Environment variables and `~/.aws/credentials` on the instance take precedence over the role - unset `AWS_ACCESS_KEY_ID` and check `~/.aws/credentials`

### Issue: "No role found" from the metadata service

- Confirm a role is attached: **EC2 Console** → instance → **Security** tab
- The trailing slash matters: `.../iam/security-credentials/` lists the role name

### Issue: "Cannot delete entity, must remove roles from instance profile first"

- Remove the role from the instance profile before deleting either one
- Also disassociate the profile from any instance still using it

### Issue: "Invalid principal in policy"

- The service name is misspelled - EC2's service principal is exactly `ec2.amazonaws.com`

### Issue: Metadata endpoint hangs or times out

- You must query from **inside** the instance, not from your laptop
- IMDS may be configured to require IMDSv2 only - make sure you included the token header

---

## Summary

Successfully completed:
- Learned how IAM roles deliver temporary, auto-rotating credentials
- Created an EC2 service role through the console and read its trust policy
- Attached the role to an EC2 instance and verified the new identity
- Read role credentials directly from the IMDSv2 metadata service
- Saw the `ASIA` temporary-key prefix and the required session token
- (Optional) Built a role and instance profile from the CLI

---

## Lab Cleanup

**Detach the role from the instance:**

1. Go to **EC2 Console** → **Instances**
2. Select the instance → **Actions** → **Security** → **Modify IAM role**
3. Choose **No IAM role** and confirm

**Delete the CLI-created role and instance profile (in this order):**

```bash
# 1. Disassociate the instance profile (if associated)
aws ec2 describe-iam-instance-profile-associations \
  --filters Name=instance-id,Values=i-YOUR_INSTANCE_ID
aws ec2 disassociate-iam-instance-profile --association-id iip-assoc-YOUR_ASSOCIATION_ID

# 2. Remove the role from the instance profile
aws iam remove-role-from-instance-profile \
  --instance-profile-name ec2-cli-describe-profile \
  --role-name ec2-cli-describe-role

# 3. Delete the instance profile
aws iam delete-instance-profile --instance-profile-name ec2-cli-describe-profile

# 4. Detach the policy and delete the role
aws iam detach-role-policy \
  --role-name ec2-cli-describe-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess
aws iam delete-role --role-name ec2-cli-describe-role
```

**Delete the console-created role:**

1. Go to **IAM Console** → **Roles** → select `ec2-s3-readonly-role` → **Delete**

**Remove local files:**

```bash
rm ec2-trust-policy.json
```

---

## Next Steps

You've given a role to a service. In the final stage, you'll assume a role yourself as a user:

**[→ Stage 6: STS and Assume Role](../Stage-6-STS-and-AssumeRole/README.md)**

Here you'll use AWS STS to step into a role, receive temporary credentials, and understand the two-way trust that makes it all safe.
