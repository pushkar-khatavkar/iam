# Stage 3: Policies Deep Dive

## Objective

Policies are the heart of IAM - every permission decision comes down to the JSON in a policy document. In this stage you will learn the **three kinds of policies**, write your own **customer managed** policy, add an **inline** policy, and prove to yourself that an **explicit Deny** always wins.

---

## Prerequisites

Before starting, make sure you have:
- Completed **Stage 1** and **Stage 2**
- A working AWS CLI with your access keys configured
- A text editor for writing JSON files
- Permission to create policies (your instructor will confirm; if not, you can still read along and do the console-based reading steps)

**[← Back to Main Workshop](../README.md)**

---

## The Three Types of Policies

| Type | Who Writes It | Reusable? | Lives Where | Best For |
|------|---------------|-----------|-------------|----------|
| **AWS Managed** | AWS | Yes | AWS account library | Quick, common permission sets |
| **Customer Managed** | You | Yes, attach to many identities | Your account's policy library | Reusable custom permissions |
| **Inline** | You | No, tied to one identity | Embedded in one user/group/role | One-off, tightly-bound rules |

**Rule of thumb:** Reach for AWS managed policies first, write customer managed policies when you need something reusable and specific, and use inline policies only for exceptions that must never be attached anywhere else.

---

## Step 1: Read an AWS Managed Policy

AWS maintains hundreds of ready-made policies. Let's read one.

### Instructions (Console):

1. Go to **IAM Console** → **Policies**
2. In the search box, type `AmazonS3ReadOnlyAccess`
3. Click the policy name, then open the **JSON** tab

### Instructions (CLI):

```bash
aws iam get-policy-version \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --version-id v3 \
  --query 'PolicyVersion.Document'
```

**Command Breakdown:**
- `get-policy-version` - retrieves the actual JSON of a policy
- AWS managed policy ARNs always start with `arn:aws:iam::aws:policy/`
- `--version-id` - managed policies are versioned; use the default version shown in the console if `v3` is not found

### Observe:

The policy allows `s3:Get*` and `s3:List*` actions. It does **not** allow `s3:PutObject` or `s3:DeleteObject`. That is why "read only" means exactly that - you can look but not change.

---

## Step 2: Write a Customer Managed Policy

Now write your own reusable policy. This one grants read access to a single S3 bucket.

### Step 2.1: Create the policy document file

```bash
cat > my-s3-readonly.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadOneBucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-workshop-bucket",
        "arn:aws:s3:::my-workshop-bucket/*"
      ]
    }
  ]
}
EOF
```

### Why Two ARNs?

- `arn:aws:s3:::my-workshop-bucket` - the **bucket itself**, needed for `s3:ListBucket`
- `arn:aws:s3:::my-workshop-bucket/*` - the **objects inside**, needed for `s3:GetObject`

This trips up almost everyone the first time. Bucket-level and object-level actions need different ARNs.

### Step 2.2: Create the policy in AWS

```bash
aws iam create-policy \
  --policy-name MyS3ReadOnly \
  --policy-document file://my-s3-readonly.json \
  --description "Read-only access to my-workshop-bucket"
```

**Command Breakdown:**
- `create-policy` - adds a reusable policy to your account's library
- `file://` - tells the CLI to read the document from disk (the `file://` prefix is required)

**Expected Output:**
JSON containing the new policy's `Arn`, for example:
```
arn:aws:iam::123456789012:policy/MyS3ReadOnly
```

**Save that ARN** - you will attach it next.

---

## Step 3: Attach Your Customer Managed Policy

Attach it to a user (replace the ARN with yours and the user name with your own):

```bash
aws iam attach-user-policy \
  --user-name <your-user-name> \
  --policy-arn arn:aws:iam::123456789012:policy/MyS3ReadOnly
```

Verify:

```bash
aws iam list-attached-user-policies --user-name <your-user-name>
```

The policy `MyS3ReadOnly` should appear in the list.

> **Reminder:** In the real world you would attach this to a **group**, not a user. We attach to a user here only to keep the example short.

---

## Step 4: Add an Inline Policy

An **inline** policy is embedded directly in one identity and cannot be reused. It is perfect for a one-off rule that should never accidentally end up attached elsewhere.

### Step 4.1: Write the inline policy document

```bash
cat > inline-dynamodb-read.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadDynamoDB",
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:Query",
        "dynamodb:Scan"
      ],
      "Resource": "*"
    }
  ]
}
EOF
```

### Step 4.2: Attach it inline to your user

```bash
aws iam put-user-policy \
  --user-name <your-user-name> \
  --policy-name InlineDynamoDBRead \
  --policy-document file://inline-dynamodb-read.json
```

**Command Breakdown:**
- `put-user-policy` - embeds the policy **inside** the user (not in the shared library)
- There is no ARN - inline policies are identified by name within the identity they belong to

### Step 4.3: Confirm it is inline, not managed

```bash
# Inline policies show up here:
aws iam list-user-policies --user-name <your-user-name>
```

```bash
# Managed policies show up here (inline ones do NOT):
aws iam list-attached-user-policies --user-name <your-user-name>
```

### Observe:

- `InlineDynamoDBRead` appears under `list-user-policies` (inline)
- `MyS3ReadOnly` appears under `list-attached-user-policies` (managed)
- This difference is why inline policies are easy to lose track of - they are hidden inside identities

---

## Step 5: The Explicit Deny - IAM's Most Important Rule

Permissions are additive: every `Allow` across every attached policy stacks up. But a single explicit **`Deny`** overrides **all** of them. This is how you build hard guardrails.

### Step 5.1: Write a policy that denies all deletes

```bash
cat > deny-deletes.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyAllDeletes",
      "Effect": "Deny",
      "Action": [
        "s3:DeleteObject",
        "s3:DeleteBucket",
        "dynamodb:DeleteTable",
        "dynamodb:DeleteItem"
      ],
      "Resource": "*"
    }
  ]
}
EOF
```

### Step 5.2: Attach it inline to your user

```bash
aws iam put-user-policy \
  --user-name <your-user-name> \
  --policy-name DenyAllDeletes \
  --policy-document file://deny-deletes.json
```

### What This Means:

Even if another policy on your user says `"Effect": "Allow"` for `s3:DeleteObject`, this `Deny` wins. You now **cannot delete** any S3 object or bucket or DynamoDB table or item - full stop. There is no policy you could add that would override this except removing the Deny itself.

> **This is the mental model to lock in:** `Deny` beats `Allow`, every time. Organizations use exactly this pattern (often via Service Control Policies) to set limits no one can exceed.

You will see this Deny take effect for real in Stage 4 when you try to delete something.

---

## Step 6: Use the Policy Simulator (Optional)

AWS gives you a tool to test permissions without actually running the action.

### Instructions:

1. Go to the **IAM Policy Simulator**: https://policysim.aws.amazon.com/
2. In the left pane, select your user
3. Pick a service (e.g. **S3**) and an action (e.g. **DeleteObject**)
4. Click **Run Simulation**
5. The result shows **denied** with the reason pointing at your `DenyAllDeletes` policy

This is the safest way to answer "would this action work?" before you try it.

---

## Troubleshooting

### Issue: "MalformedPolicyDocument"

- Your JSON has a syntax error - a missing comma, bracket, or quote
- Confirm `"Version": "2012-10-17"` is present and spelled exactly
- Validate the JSON in an editor before submitting

### Issue: "Error parsing parameter --policy-document"

- You most likely forgot the `file://` prefix
- Use `--policy-document file://my-s3-readonly.json`, not a bare path

### Issue: "AccessDenied" when running create-policy or put-user-policy

- Your user lacks `iam:CreatePolicy` or `iam:PutUserPolicy`
- Some labs restrict policy creation - ask your instructor, or do the read-only steps and follow along

### Issue: "EntityAlreadyExists"

- A policy with that name already exists - choose a different name or delete the old one first

### Issue: My Allow policy is not working

- Check for a `Deny` on the same action anywhere in your attached or inline policies - a Deny silently overrides it
- Use the Policy Simulator to see which statement is making the decision

---

## Summary

Successfully completed:
- Learned the three policy types: AWS managed, customer managed, and inline
- Read an AWS managed policy and understood what "read only" really means
- Wrote and attached your own customer managed policy
- Added an inline policy and saw how it differs from a managed one
- Created an explicit `Deny` and understood why it overrides every `Allow`
- (Optional) Used the Policy Simulator to test permissions safely

---

## Lab Cleanup

Remove the policies you added so they do not interfere with later stages. **Keep the `DenyAllDeletes` policy for now** - Stage 4 uses it to demonstrate a blocked action. You will remove it at the end of Stage 4.

```bash
# Detach the customer managed policy
aws iam detach-user-policy \
  --user-name <your-user-name> \
  --policy-arn arn:aws:iam::123456789012:policy/MyS3ReadOnly

# Delete the customer managed policy from the library
aws iam delete-policy --policy-arn arn:aws:iam::123456789012:policy/MyS3ReadOnly

# Remove the inline DynamoDB read policy
aws iam delete-user-policy --user-name <your-user-name> --policy-name InlineDynamoDBRead
```

```bash
# Remove the local files
rm my-s3-readonly.json inline-dynamodb-read.json
```

---

## Next Steps

Now that you can read and write policies, it's time to use them on real services:

**[→ Stage 4: Permissions in Action](../Stage-4-Permissions-In-Action/README.md)**

Here you'll start and stop an EC2 instance, create an S3 bucket and a DynamoDB table, and watch your `Deny` policy block a delete.
