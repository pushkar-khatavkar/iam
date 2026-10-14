# Stage 4: Permissions in Action

## Objective

Enough theory - time to use permissions on real AWS services. In this stage you will **start and stop a pre-existing EC2 instance**, **create an S3 bucket**, and **create a DynamoDB table**. Then you will watch the explicit `Deny` from Stage 3 block a delete, proving the rules work in practice.

> **Note:** You will **not launch** new EC2 instances - your instructor already created those. You only start and stop them. You **will** create your own S3 buckets and DynamoDB tables.

---

## Prerequisites

Before starting, make sure you have:
- Completed **Stages 1-3**
- The `DenyAllDeletes` inline policy from Stage 3 still attached to your user
- A working AWS CLI configured for region `us-east-1`
- From your instructor: the **instance ID** of a pre-created EC2 instance (looks like `i-0abc123...`)
- Permission to run EC2 start/stop, S3 create, and DynamoDB create actions

**[← Back to Main Workshop](../README.md)**

---

## Part A: Start and Stop an EC2 Instance

Your instructor pre-created at least one EC2 instance. You will control its power state - a common real-world task for saving costs (stop at night, start in the morning).

### Step 1: Find the Instance

List instances and their current state:

```bash
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name,Tags[?Key==`Name`].Value|[0]]' \
  --output table
```

**Command Breakdown:**
- `describe-instances` - lists EC2 instances and their details
- The `--query` pulls out the instance ID, power state, and Name tag
- Note the **InstanceId** of the instance you were told to use

> If this returns `UnauthorizedOperation`, your user cannot describe instances - ask your instructor for the exact instance ID and skip to Step 2.

### Step 2: Stop the Instance

```bash
aws ec2 stop-instances --instance-ids i-YOUR_INSTANCE_ID
```

**Expected Output:**
JSON showing a `CurrentState` of `stopping` and a `PreviousState` of `running`.

**Command Breakdown:**
- `stop-instances` - gracefully shuts down the instance (it is **not** deleted, just powered off)
- Data on the root volume is preserved

### Step 3: Confirm It Stopped

```bash
aws ec2 describe-instances \
  --instance-ids i-YOUR_INSTANCE_ID \
  --query 'Reservations[0].Instances[0].State.Name' \
  --output text
```

Wait ~30-60 seconds and run it again until it shows `stopped`.

### Step 4: Start the Instance Again

```bash
aws ec2 start-instances --instance-ids i-YOUR_INSTANCE_ID
```

**Expected Output:**
JSON showing `CurrentState` of `pending`, moving to `running`.

### Step 5: Confirm It Started

```bash
aws ec2 describe-instances \
  --instance-ids i-YOUR_INSTANCE_ID \
  --query 'Reservations[0].Instances[0].State.Name' \
  --output text
```

It should return `running` within a minute or two.

### Observe:

You controlled an instance you did not create. Your permission to `ec2:StartInstances` and `ec2:StopInstances` is what allowed this. If you tried `aws ec2 terminate-instances`, it would likely fail - start/stop is a far narrower grant than terminate, which is exactly the point of least privilege.

---

## Part B: Create an S3 Bucket

Now create a resource of your own. S3 bucket names must be **globally unique** across all of AWS, so include something personal.

### Step 1: Pick a Unique Bucket Name

Use a pattern like `iam-workshop-<yourname>-<random-number>`, for example:
```
iam-workshop-alex-20261013
```

### Step 2: Create the Bucket

```bash
aws s3 mb s3://iam-workshop-alex-20261013 --region us-east-1
```

**Command Breakdown:**
- `s3 mb` - "make bucket"
- The name must be globally unique, all lowercase, no underscores

**Expected Output:**
```
make_bucket: iam-workshop-alex-20261013
```

### Step 3: Upload a Test File

```bash
echo "Hello from the IAM workshop" > hello.txt
aws s3 cp hello.txt s3://iam-workshop-alex-20261013/
```

**Expected Output:**
```
upload: ./hello.txt to s3://iam-workshop-alex-20261013/hello.txt
```

### Step 4: List the Bucket Contents

```bash
aws s3 ls s3://iam-workshop-alex-20261013/
```

You should see `hello.txt` listed.

### Observe:

You created a bucket and put an object in it. This required `s3:CreateBucket` and `s3:PutObject` permissions. A read-only policy (like the one you read in Stage 3) would have blocked the create and the upload.

---

## Part C: Create a DynamoDB Table

DynamoDB is AWS's managed NoSQL database. Creating a table is a great example of a "create" permission in action.

### Step 1: Create the Table

```bash
aws dynamodb create-table \
  --table-name iam-workshop-alex \
  --attribute-definitions AttributeName=id,AttributeType=S \
  --key-schema AttributeName=id,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

**Command Breakdown:**
- `create-table` - creates a new DynamoDB table
- `--attribute-definitions` - declares an attribute `id` of type String (`S`)
- `--key-schema` - makes `id` the partition (HASH) key
- `--billing-mode PAY_PER_REQUEST` - no capacity planning, pay only for what you use (ideal for labs)

**Expected Output:**
JSON showing `TableStatus: "CREATING"`.

### Step 2: Wait Until the Table Is Active

```bash
aws dynamodb describe-table \
  --table-name iam-workshop-alex \
  --query 'Table.TableStatus' \
  --output text
```

Run it until it returns `ACTIVE` (usually under a minute).

### Step 3: Write an Item

```bash
aws dynamodb put-item \
  --table-name iam-workshop-alex \
  --item '{"id": {"S": "item-1"}, "message": {"S": "my first item"}}'
```

**Expected Output:**
No output on success.

### Step 4: Read the Item Back

```bash
aws dynamodb get-item \
  --table-name iam-workshop-alex \
  --key '{"id": {"S": "item-1"}}'
```

**Expected Output:**
JSON showing the item you just wrote.

### Observe:

Creating the table needed `dynamodb:CreateTable`. Writing and reading needed `dynamodb:PutItem` and `dynamodb:GetItem`. Each action maps to a specific permission - that is the core idea of IAM.

---

## Part D: Watch the Explicit Deny Block You

Remember the `DenyAllDeletes` policy from Stage 3? Let's see it in action.

### Try to Delete the DynamoDB Table

```bash
aws dynamodb delete-table --table-name iam-workshop-alex
```

**Expected Output:**
```
An error occurred (AccessDeniedException) ... not authorized to perform:
dynamodb:DeleteTable ... with an explicit deny in an identity-based policy
```

### Try to Delete the S3 Object

```bash
aws s3 rm s3://iam-workshop-alex-20261013/hello.txt
```

**Expected Output:**
An `AccessDenied` error, blocked by the same `Deny` policy.

### Why This Happens:

You almost certainly have `Allow` permissions for these delete actions somewhere - but the explicit `Deny` from Stage 3 **overrides** them. This is the golden rule made real: **Deny always wins.**

The error message even tells you: *"with an explicit deny in an identity-based policy."* AWS is pointing straight at the cause.

---

## Troubleshooting

### Issue: "UnauthorizedOperation" on EC2 start/stop

- Your user lacks `ec2:StartInstances` / `ec2:StopInstances`
- Confirm you are in the **same region** as the instance (`--region us-east-1`)
- Ask your instructor to confirm the instance ID and your permissions

### Issue: "BucketAlreadyExists" or "BucketAlreadyOwnedByYou"

- S3 bucket names are **globally unique** - someone already took that name
- Add more randomness: `iam-workshop-<name>-<date>-<random>`

### Issue: "Invalid bucket name"

- Bucket names must be lowercase, 3-63 characters, no underscores or uppercase letters

### Issue: DynamoDB "ResourceInUseException"

- A table with that name already exists - pick a different name or reuse the existing one

### Issue: My delete actually worked (no Deny error)

- The `DenyAllDeletes` policy from Stage 3 is not attached - re-attach it:
  ```bash
  aws iam put-user-policy --user-name <your-user-name> \
    --policy-name DenyAllDeletes --policy-document file://deny-deletes.json
  ```

---

## Summary

Successfully completed:
- Started and stopped a pre-existing EC2 instance (without creating one)
- Created an S3 bucket and uploaded an object to it
- Created a DynamoDB table, wrote an item, and read it back
- Watched an explicit `Deny` block delete actions you otherwise had permission for
- Connected each successful action to the specific IAM permission that allowed it

---

## Lab Cleanup

First, **remove the Deny policy** so you can clean up the resources you created:

```bash
aws iam delete-user-policy --user-name <your-user-name> --policy-name DenyAllDeletes
```

Now delete your resources:

```bash
# Empty and delete the S3 bucket
aws s3 rm s3://iam-workshop-alex-20261013/hello.txt
aws s3 rb s3://iam-workshop-alex-20261013

# Delete the DynamoDB table
aws dynamodb delete-table --table-name iam-workshop-alex

# Remove local files
rm hello.txt deny-deletes.json
```

**Leave the EC2 instance running or stopped** per your instructor's guidance - do not terminate it, since you did not create it.

---

## Next Steps

You've used permissions directly as a user. Next, learn how to give those permissions to an **application** without handing it any keys:

**[→ Stage 5: IAM Roles and Instance Profiles](../Stage-5-Roles-and-Instance-Profiles/README.md)**

Here you'll create a role, attach it to an EC2 instance, and read temporary credentials from the metadata service.
