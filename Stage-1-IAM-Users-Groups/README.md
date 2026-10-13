# Stage 1: IAM Users, Groups, and Policies

## Objective

In this stage, you will use the AWS Management Console to create an IAM group, attach permissions to it, create IAM users inside that group, and verify what those users can and cannot do. You will also write your own custom policy.

---

## Prerequisites

Before starting, ensure you have completed:
- Logged into the **AWS Management Console** with admin permissions
- Noted your **account alias** or 12-digit account ID (for the sign-in URL)
- Reviewed the **IAM policy anatomy** in the main README

**[← Back to Main Workshop](../README.md)**

---

## Step 1: Create a User Group

Groups are the recommended way to assign permissions. You attach policies once to the group, and every member inherits them.

### Instructions:

1. Go to **IAM Console** → **User groups** → **Create group**

2. **Configure Group Details:**
   - **User group name:** `cloudlab-students`

3. **Attach permissions policies** - search for and select:
   - `AmazonS3ReadOnlyAccess`
   - `AmazonEC2ReadOnlyAccess`

4. Leave the **Add users to group** section empty for now

5. Click **Create group**

6. Confirm `cloudlab-students` now appears in the group list with **2** attached policies

---

## Step 2: Create an IAM User

Now create a user and place them directly into the group you just made.

### Instructions:

1. Go to **IAM Console** → **Users** → **Create user**

2. **Specify user details:**
   - **User name:** `student-01`
   - Check **Provide user access to the AWS Management Console**
   - Select **I want to create an IAM user**
   - **Console password:** Choose **Custom password** and enter a strong password
   - Keep **Users must create a new password at next sign-in** checked

3. Click **Next**

4. **Set permissions:**
   - Choose **Add user to group**
   - Select the checkbox for `cloudlab-students`

5. Click **Next**, review the summary, then click **Create user**

6. On the confirmation screen, **download the .csv file** or copy the sign-in details

> **Important:** The console sign-in URL, username, and password are shown only once. Save them now.

### Expected Result:

- `student-01` appears in the **Users** list
- The user's **Groups** column shows `cloudlab-students`
- The user has **no** directly attached policies (permissions come from the group)

---

## Step 3: Verify the User's Permissions

Time to test least privilege in action. Sign in as the new user and see what they can reach.

### Instructions:

1. Open a **new private/incognito browser window** (so you stay logged in as admin in your main window)

2. Navigate to your sign-in URL:
   ```
   https://<your-account-alias>.signin.aws.amazon.com/console
   ```
   
   **Example:**
   ```
   https://silveroak-cloudlab.signin.aws.amazon.com/console
   ```

3. Sign in as `student-01` with the password you set

4. Set a new password when prompted

5. **Test what works** - navigate to the **S3 Console** and list buckets:
   - You should see the bucket list load successfully

6. **Test what is blocked** - still in S3, try to click **Create bucket**:
   - You should receive an **Access Denied** error

7. **Test another blocked action** - navigate to the **IAM Console**:
   - You should receive an **Access Denied** error, because the group has no IAM permissions

### Why This Happens:

`AmazonS3ReadOnlyAccess` grants `s3:Get*` and `s3:List*` actions only. Creating a bucket requires `s3:CreateBucket`, which was never allowed. Since IAM denies everything by default, the request fails.

---

## Step 4: Create a Custom Managed Policy

AWS managed policies are convenient but broad. Now write a tightly scoped policy of your own.

### Instructions:

1. Return to your **admin browser window**

2. Go to **IAM Console** → **Policies** → **Create policy**

3. Select the **JSON** tab and replace the contents with:

   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "AllowReadOnlyOnLabBucket",
         "Effect": "Allow",
         "Action": [
           "s3:GetObject",
           "s3:ListBucket"
         ],
         "Resource": [
           "arn:aws:s3:::silveroak-cloudlab-data",
           "arn:aws:s3:::silveroak-cloudlab-data/*"
         ]
       },
       {
         "Sid": "DenyDeletesEverywhere",
         "Effect": "Deny",
         "Action": [
           "s3:DeleteObject",
           "s3:DeleteBucket"
         ],
         "Resource": "*"
       }
     ]
   }
   ```

4. Click **Next**

5. **Policy details:**
   - **Policy name:** `SilverOakLabBucketReadOnly`
   - **Description:** `Read-only access to the lab data bucket, deletes explicitly denied`

6. Click **Create policy**

### Command Breakdown:

- **Statement 1** allows reading objects and listing contents, but **only** for the bucket named `silveroak-cloudlab-data`
- Two resource ARNs are needed: the bucket itself (for `ListBucket`) and its objects (for `GetObject`)
- **Statement 2** explicitly denies deletes on every resource
- An explicit `Deny` beats any `Allow`, even an admin-level one - this is how you build guardrails

---

## Step 5: Attach the Custom Policy to the Group

### Instructions:

1. Go to **IAM Console** → **User groups** → `cloudlab-students`

2. Open the **Permissions** tab

3. Click **Add permissions** → **Attach policies**

4. Filter by **Customer managed** and select `SilverOakLabBucketReadOnly`

5. Click **Attach policies**

6. Confirm the group now lists **3** attached policies

### Verify the Deny Takes Effect:

1. Switch back to your incognito window signed in as `student-01`
2. Refresh the S3 console
3. Try to delete any object
4. The action fails - the explicit `Deny` overrides the read permissions granted elsewhere

---

## Step 6: Add a Second User (Optional)

Practice the pattern you just learned by adding another student.

### Instructions:

1. Go to **IAM Console** → **Users** → **Create user**
2. **User name:** `student-02`
3. Enable console access with a custom password
4. Add the user to `cloudlab-students`
5. Click **Create user**

Notice that `student-02` inherits all three policies instantly. This is why groups beat per-user policy attachments: one change updates everyone.

---

## Troubleshooting

### Issue: "Access Denied" when signing in as the new user

- Confirm you enabled **console access** when creating the user
- Verify you are using the **account-specific sign-in URL**, not the root sign-in page
- Check that the password was typed correctly (it is case sensitive)

### Issue: User can see more than expected

- Check the **Permissions** tab for **directly attached** policies in addition to group policies
- Look for other groups the user belongs to
- Use the **IAM Policy Simulator** to test a specific action: https://policysim.aws.amazon.com/

### Issue: Custom policy will not save

- Validate your JSON - a missing comma or bracket blocks creation
- Confirm `"Version": "2012-10-17"` is present and spelled exactly
- Make sure every `Action` uses the `service:Operation` format

### Issue: Cannot see the bucket from the custom policy

- The bucket `silveroak-cloudlab-data` is an example name and may not exist in your account
- Replace it with a real bucket ARN from your account, or create a bucket with that name

---

## Summary

Successfully completed:
- Created an IAM user group and attached AWS managed policies
- Created IAM users with console access inside that group
- Verified least-privilege behavior by testing allowed and denied actions
- Authored a custom managed policy with both `Allow` and `Deny` statements
- Observed that an explicit `Deny` overrides any `Allow`
- Understood why group-based permissions scale better than per-user ones

---

## Lab Cleanup

Keep the resources from this stage - Stage 2 reuses the same naming conventions. If you need to clean up now:

1. Go to **IAM Console** → **Users**, select `student-01` and `student-02`, click **Delete**
2. Go to **User groups**, select `cloudlab-students`, click **Delete**
3. Go to **Policies**, filter by **Customer managed**, select `SilverOakLabBucketReadOnly`, click **Delete**

---

## Next Steps

Once you've completed Stage 1, you're ready for:

**[→ Stage 2: Exploring IAM CLI Commands](../Stage-2-IAM-CLI/README.md)**

Here you'll learn to manage IAM identities programmatically using AWS CLI commands.

---

**Workshop Created by:** CloudKida Labs  
**For:** Silver Oak University - IAM Beginners Workshop
