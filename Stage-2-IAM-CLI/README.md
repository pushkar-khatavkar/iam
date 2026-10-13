# Stage 2: Exploring IAM CLI Commands

## Objective

In this stage, you will master AWS CLI commands to manage IAM identities programmatically. Learn to create, inspect, modify, and delete users, groups, policies, and access keys from the command line without using the AWS Management Console.

---

## Prerequisites

Before starting, ensure you have:
- Completed **Stage 1: IAM Users, Groups, and Policies**
- AWS CLI v2 installed and configured
- An identity with IAM write permissions
- Basic understanding of terminal commands and JSON

**[← Back to Main Workshop](../README.md)**

---

## Verify AWS CLI Installation

Amazon Linux 2023 comes with AWS CLI v2 pre-installed. If you are on your own machine, confirm it is available:

```bash
aws --version
```

You should see output like:
```
aws-cli/2.x.x Python/3.x.x Linux/x.x.x
```

### Confirm Your Identity

Always know which principal you are acting as before running IAM commands:

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

**Note:** IAM is a global service. You do not need `--region` for IAM commands, though passing one causes no harm.

---

## IAM CLI Commands Reference

### 1. **Create a New IAM User**

```bash
aws iam create-user --user-name student-cli
```

**Command Breakdown:**
- `create-user` - Creates a new IAM user with **no** permissions
- `--user-name` - The name of the user (must be unique in the account)

**Expected Output:**
Returns JSON with user details including:
- `UserName` - The name you provided
- `UserId` - A unique internal ID (e.g. `AIDAEXAMPLEID`)
- `Arn` - Full ARN (e.g. `arn:aws:iam::123456789012:user/student-cli`)
- `CreateDate` - Timestamp of creation

**Note:** A brand-new user can do nothing at all until you attach a policy.

---

### 2. **Create a Group and Attach a Policy**

```bash
aws iam create-group --group-name cloudlab-cli-students
```

```bash
aws iam attach-group-policy \
  --group-name cloudlab-cli-students \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

**Command Breakdown:**
- `create-group` - Creates an empty group
- `attach-group-policy` - Attaches a policy to every member of the group
- `--policy-arn` - The ARN of the policy; AWS managed policies always start with `arn:aws:iam::aws:policy/`

**Expected Output:**
`attach-group-policy` returns no output on success. Silence means it worked.

---

### 3. **Add a User to a Group**

```bash
aws iam add-user-to-group \
  --user-name student-cli \
  --group-name cloudlab-cli-students
```

**Command Breakdown:**
- `add-user-to-group` - Makes the user a member of the group
- The user immediately inherits all policies attached to that group

**Expected Output:**
No output on success.

---

### 4. **Enable Console Access for a User**

```bash
aws iam create-login-profile \
  --user-name student-cli \
  --password 'ChangeMe#2026!' \
  --password-reset-required
```

**Command Breakdown:**
- `create-login-profile` - Grants AWS Management Console sign-in access
- `--password` - The initial password (wrap it in single quotes so your shell does not interpret special characters)
- `--password-reset-required` - Forces the user to set their own password at first sign-in

⚠️ **WARNING:** Passwords typed on the command line land in your shell history. For real accounts, use `--no-cli-pager` with a generated password or set passwords through the console.

---

### 5. **Create Access Keys for Programmatic Access**

```bash
aws iam create-access-key --user-name student-cli
```

**Command Breakdown:**
- `create-access-key` - Generates an access key ID and secret access key
- A user can hold a maximum of **2** access keys at a time

**Expected Output:**
```json
{
    "AccessKey": {
        "UserName": "student-cli",
        "AccessKeyId": "AKIAIOSFODNN7EXAMPLE",
        "Status": "Active",
        "SecretAccessKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
        "CreateDate": "2026-10-13T10:00:00+00:00"
    }
}
```

⚠️ **WARNING:** The `SecretAccessKey` is shown **only once**. Save it securely or delete the key and create a new one. Never commit access keys to Git.

---

### 6. **Delete an IAM User**

```bash
aws iam delete-user --user-name student-cli
```

**Command Breakdown:**
- `delete-user` - Permanently removes the user
- The call **fails** if the user still has attached policies, group memberships, access keys, or a login profile

⚠️ **WARNING:** Deleting a user is **permanent** and cannot be undone.

See the [Lab Cleanup](#lab-cleanup) section for the correct deletion order.

---

## Practical Exercises

### Exercise 1: List All Your IAM Users

```bash
aws iam list-users --query 'Users[*].[UserName,UserId,CreateDate]' --output table
```

This command displays:
- User Name
- User ID
- Creation Date

---

### Exercise 2: Create a User and Group from Scratch

```bash
aws iam create-user --user-name lab-auditor
```

```bash
aws iam create-group --group-name lab-auditors
```

```bash
aws iam attach-group-policy \
  --group-name lab-auditors \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
```

```bash
aws iam add-user-to-group --user-name lab-auditor --group-name lab-auditors
```

**What This Does:**
- Creates a user named `lab-auditor`
- Creates a group named `lab-auditors` with account-wide read-only access
- Places the user in the group so it inherits those permissions

**Save the user name for the next exercises**

---

### Exercise 3: Inspect the User's Effective Permissions

Check group membership:

```bash
aws iam list-groups-for-user --user-name lab-auditor
```

Check policies attached directly to the user:

```bash
aws iam list-attached-user-policies --user-name lab-auditor
```

Check policies attached to the group:

```bash
aws iam list-attached-group-policies --group-name lab-auditors
```

Check the output for:
- The `lab-auditors` group in the first command
- An **empty** `AttachedPolicies` list in the second command
- `ReadOnlyAccess` in the third command

This confirms the permissions arrive through the group, not the user.

---

### Exercise 4: Create a Custom Managed Policy from a File

**Step 4.1: Write the policy document**

```bash
cat > lab-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowDescribeEC2",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeImages",
        "ec2:DescribeTags"
      ],
      "Resource": "*"
    }
  ]
}
EOF
```

**Step 4.2: Create the policy**

```bash
aws iam create-policy \
  --policy-name SilverOakEC2DescribeOnly \
  --policy-document file://lab-policy.json \
  --description "Allows describing EC2 instances, images, and tags"
```

**Step 4.3: Save the returned policy ARN**

```bash
aws iam list-policies --scope Local \
  --query 'Policies[?PolicyName==`SilverOakEC2DescribeOnly`].Arn' \
  --output text
```

**Note:** `file://` tells the CLI to read the document from disk. The leading `file://` is required - a plain path will not work.

---

### Exercise 5: Attach and Detach the Custom Policy

Attach it to the user (replace `123456789012` with your account ID):

```bash
aws iam attach-user-policy \
  --user-name lab-auditor \
  --policy-arn arn:aws:iam::123456789012:policy/SilverOakEC2DescribeOnly
```

Verify the attachment:

```bash
aws iam list-attached-user-policies --user-name lab-auditor
```

Detach it again:

```bash
aws iam detach-user-policy \
  --user-name lab-auditor \
  --policy-arn arn:aws:iam::123456789012:policy/SilverOakEC2DescribeOnly
```

---

### Exercise 6: Rotate and Delete Access Keys

List the user's keys:

```bash
aws iam list-access-keys --user-name lab-auditor
```

Create a key:

```bash
aws iam create-access-key --user-name lab-auditor
```

Deactivate it without deleting (useful during rotation):

```bash
aws iam update-access-key \
  --user-name lab-auditor \
  --access-key-id AKIAIOSFODNN7EXAMPLE \
  --status Inactive
```

Delete it permanently:

```bash
aws iam delete-access-key \
  --user-name lab-auditor \
  --access-key-id AKIAIOSFODNN7EXAMPLE
```

⚠️ Deleting an access key immediately breaks any application still using it.

---

## Additional Useful Commands

### Get Details About a Specific User

```bash
aws iam get-user --user-name lab-auditor
```

### View a Policy Document

```bash
aws iam get-policy-version \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --version-id v2
```

### List Only Customer Managed Policies

```bash
aws iam list-policies --scope Local --query 'Policies[*].[PolicyName,Arn]' --output table
```

### List All Roles in the Account

```bash
aws iam list-roles --query 'Roles[*].[RoleName,Arn]' --output table
```

### Remove a User from a Group

```bash
aws iam remove-user-from-group --user-name lab-auditor --group-name lab-auditors
```

### Check When a Password Was Last Used

```bash
aws iam get-user --user-name lab-auditor --query 'User.PasswordLastUsed'
```

### Generate an Account-Wide Credential Report

```bash
aws iam generate-credential-report
```

```bash
aws iam get-credential-report --query 'Content' --output text | base64 --decode
```

### Tag a User

```bash
aws iam tag-user --user-name lab-auditor --tags Key=Department,Value=CloudLab
```

---

## Common Parameters

| Parameter | Purpose | Example |
|-----------|---------|---------|
| `--user-name` | Specify which user to manage | `--user-name student-cli` |
| `--group-name` | Specify which group to manage | `--group-name lab-auditors` |
| `--role-name` | Specify which role to manage | `--role-name ec2-s3-role` |
| `--policy-arn` | Identify the policy to attach or detach | `--policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess` |
| `--policy-document` | Supply a JSON policy | `--policy-document file://lab-policy.json` |
| `--scope` | Filter policies by ownership | `--scope Local`, `--scope AWS` |
| `--query` | Filter output | `--query 'Users[*].UserName'` |
| `--output` | Output format | `--output table`, `json`, `text`, `yaml` |

---

## Frequently Used AWS Managed Policy ARNs

| Policy | ARN |
|--------|-----|
| ReadOnlyAccess | `arn:aws:iam::aws:policy/ReadOnlyAccess` |
| AmazonS3ReadOnlyAccess | `arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess` |
| AmazonS3FullAccess | `arn:aws:iam::aws:policy/AmazonS3FullAccess` |
| AmazonEC2ReadOnlyAccess | `arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess` |
| IAMReadOnlyAccess | `arn:aws:iam::aws:policy/IAMReadOnlyAccess` |
| IAMUserChangePassword | `arn:aws:iam::aws:policy/IAMUserChangePassword` |
| AdministratorAccess | `arn:aws:iam::aws:policy/AdministratorAccess` |

---

## Best Practices

- **Always run `aws sts get-caller-identity`** before IAM changes so you know which identity you are using
- **Attach policies to groups**, not individual users
- **Store policy documents in files** and keep them in version control
- **Delete access keys** you are not actively using
- **Tag every identity** with an owner or department for auditing
- **Use `--dry-run` where supported** to test without executing
- **Verify after every change** - IAM commands often succeed silently:
```bash
aws iam list-attached-group-policies --group-name lab-auditors
```

---

## Troubleshooting

### Issue: "EntityAlreadyExists"

- A user, group, or policy with that name already exists in the account
- IAM names are unique account-wide; list existing entities first: `aws iam list-users`

### Issue: "AccessDenied" or "User is not authorized to perform iam:CreateUser"

- Your current identity lacks IAM write permissions
- Check who you are: `aws sts get-caller-identity`
- Ask your instructor or account admin for `IAMFullAccess`

### Issue: "DeleteConflict: Cannot delete entity, must detach all policies first"

- IAM blocks deletion while dependencies remain
- Detach policies, remove group memberships, delete access keys and login profiles, then delete the user

### Issue: "MalformedPolicyDocument"

- Validate your JSON before submitting it
- Confirm `"Version": "2012-10-17"` is present
- Verify every `Action` uses `service:Operation` format and the resource ARN is well formed

### Issue: "Error parsing parameter --policy-document"

- You most likely forgot the `file://` prefix
- Use `--policy-document file://lab-policy.json`, not `--policy-document lab-policy.json`

### Issue: Changes do not seem to take effect

- IAM is eventually consistent; wait a few seconds and retry
- Confirm you changed the identity you are actually testing with

---

## Summary

Successfully completed:
- Verified AWS CLI v2 and confirmed your caller identity
- Created IAM users and groups from the command line
- Attached AWS managed policies to groups and users
- Enabled console access and generated access keys
- Authored a custom managed policy from a JSON file
- Practiced rotating, deactivating, and deleting access keys
- Generated a credential report for auditing
- Understood best practices for CLI-based identity management

---

## Lab Cleanup

IAM enforces a strict deletion order. Work through dependencies first, then the identity.

**Delete the user created in the exercises:**

```bash
# 1. Remove from all groups
aws iam remove-user-from-group --user-name lab-auditor --group-name lab-auditors

# 2. Detach any directly attached policies
aws iam detach-user-policy \
  --user-name lab-auditor \
  --policy-arn arn:aws:iam::123456789012:policy/SilverOakEC2DescribeOnly

# 3. Delete access keys (repeat for each key ID)
aws iam delete-access-key --user-name lab-auditor --access-key-id AKIAIOSFODNN7EXAMPLE

# 4. Delete the login profile, if one exists
aws iam delete-login-profile --user-name lab-auditor

# 5. Finally delete the user
aws iam delete-user --user-name lab-auditor
```

**Delete the groups and custom policy:**

```bash
# Detach policies from the group first
aws iam detach-group-policy \
  --group-name lab-auditors \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess

aws iam delete-group --group-name lab-auditors
```

```bash
# Delete the customer managed policy
aws iam delete-policy --policy-arn arn:aws:iam::123456789012:policy/SilverOakEC2DescribeOnly
```

**Remove the local policy file:**

```bash
rm lab-policy.json
```

---

## Next Steps

Now that you've mastered IAM CLI commands, you're ready for:

**[→ Stage 3: IAM Roles and Instance Profiles](../Stage-3-IAM-Roles/README.md)**

Here you'll explore advanced IAM features to grant permissions to applications without ever handling an access key.

---

**Workshop Created by:** CloudKida Labs  
**For:** Silver Oak University - IAM Beginners Workshop
