# Stage 1: Getting Started with IAM

## Objective

This is the gentlest possible introduction. By the end of this stage you will have signed in as your IAM user, looked around the IAM console, run your first AWS CLI command, and answered the most important question in all of IAM: **"Who am I, and what am I allowed to do?"**

No resources are created in this stage. Just explore and observe.

---

## Prerequisites

Before starting, make sure you have:
- An IAM **user name** and **console password** from your instructor
- The **account sign-in URL** (looks like `https://<something>.signin.aws.amazon.com/console`)
- A web browser

**[← Back to Main Workshop](../README.md)**

---

## Step 1: Sign In to the AWS Console

Your instructor created an IAM user just for you. Use it to sign in.

### Instructions:

1. Open your browser and go to the **account sign-in URL** your instructor gave you:
   ```
   https://<account-alias>.signin.aws.amazon.com/console
   ```

2. On the sign-in page, enter:
   - **IAM user name:** the name you were given (e.g. `student-01`)
   - **Password:** the password you were given

3. Click **Sign in**

4. If prompted to **set a new password**, choose a strong one and save it somewhere safe

### What You Should See:

The AWS Management Console home page. In the **top-right corner**, you should see your user name and the account number, like:

```
student-01 @ 1234-5678-9012
```

> **Tip:** The part after the `@` is your 12-digit AWS account ID. You will see it again in the CLI output later.

---

## Step 2: Open the IAM Console and Look Around

Before changing anything, get your bearings.

### Instructions:

1. In the top search bar, type **IAM** and click the **IAM** service

2. Look at the left navigation pane. These are the building blocks you will use throughout the workshop:
   - **User groups** - collections of users
   - **Users** - individual identities (you are one of them)
   - **Roles** - identities assumed temporarily
   - **Policies** - permission documents

3. Click **Users** in the left pane

4. Find your own user name in the list and click it

5. Explore the tabs on your user page:
   - **Permissions** - what you are allowed to do
   - **Groups** - which groups you belong to
   - **Security credentials** - your password and access keys

### Observe:

- Note which **groups** you belong to (there may be none yet)
- Note any **policies** listed under the Permissions tab
- This is the single most useful habit in IAM: before troubleshooting "why can't I do X?", look here first

---

## Step 3: Find Your Access Keys

For CLI work in later stages, you need **access keys**. Your instructor may have given them to you already, or you may need to view them here.

### Instructions:

1. On your user page, open the **Security credentials** tab

2. Scroll to the **Access keys** section

3. You should see an access key ID that starts with `AKIA...`

> **Important:** If you were given access keys as a file or message, keep them safe. The **secret access key** is shown only once at creation time and cannot be retrieved later. If you do not have one, ask your instructor.

**You do not need to do anything else here in Stage 1** - just confirm you know where your credentials are.

---

## Step 4: Configure the AWS CLI

Now set up the CLI so you can run commands from your terminal.

### Instructions:

1. Open a terminal on your computer

2. Confirm the CLI is installed:
   ```bash
   aws --version
   ```
   You should see something like `aws-cli/2.x.x ...`

3. Configure your credentials:
   ```bash
   aws configure
   ```

4. When prompted, enter:
   - **AWS Access Key ID:** your access key (starts with `AKIA`)
   - **AWS Secret Access Key:** your secret key
   - **Default region name:** `us-east-1`
   - **Default output format:** `json`

### Expected Result:

The command finishes without errors. Your credentials are now saved in `~/.aws/credentials` and your settings in `~/.aws/config`.

---

## Step 5: Run Your First CLI Command - "Who Am I?"

This is the single most useful IAM command. It tells you exactly which identity your CLI is using.

### Command to Run:

```bash
aws sts get-caller-identity
```

### Expected Output:

```json
{
    "UserId": "AIDAEXAMPLEUSERID",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/student-01"
}
```

### Command Breakdown:

- `sts get-caller-identity` - asks AWS "who is making this request?"
- `UserId` - the internal unique ID of your identity
- `Account` - your 12-digit AWS account ID (matches what you saw in the console)
- `Arn` - your full identity. The `user/student-01` part confirms you are signed in as your user

> **Get in the habit:** Run this command any time you are unsure which identity your CLI is using. It never changes anything, so it is always safe.

---

## Step 6: Inspect Your Own Permissions from the CLI

Now ask the CLI what you are allowed to do.

### List the groups you belong to:

```bash
aws iam list-groups-for-user --user-name student-01
```

Replace `student-01` with your own user name.

### List policies attached directly to you:

```bash
aws iam list-attached-user-policies --user-name student-01
```

### List any inline policies on you:

```bash
aws iam list-user-policies --user-name student-01
```

### Observe:

- These are read-only commands - they change nothing
- Together they show you the complete picture of where your permissions come from: groups, attached policies, and inline policies
- If a command returns `AccessDenied`, that is itself useful information - it means your user is not allowed to read that detail (which is common and expected in locked-down labs)

---

## Understanding What Just Happened

You did not create anything, but you learned the two things that matter most:

1. **Identity** - `aws sts get-caller-identity` tells you who you are
2. **Permissions** - the console Permissions tab and the `list-*` commands tell you what you can do

Every "Access Denied" error you will ever hit comes down to a mismatch between these two: the identity making the request does not have a policy allowing the action. Keep that mental model.

---

## Troubleshooting

### Issue: Cannot sign in to the console

- Confirm you are using the **account-specific sign-in URL**, not the root sign-in page
- User names and passwords are **case sensitive**
- If you were asked to reset your password and did not finish, ask your instructor to reset it

### Issue: `aws configure` saved but commands fail with "InvalidClientTokenId"

- Your access key or secret key was typed incorrectly - run `aws configure` again
- Make sure there are no extra spaces when pasting the keys

### Issue: "Unable to locate credentials"

- You have not run `aws configure` yet, or it did not save
- Check the file exists: `cat ~/.aws/credentials`

### Issue: `get-caller-identity` works but `list-attached-user-policies` says AccessDenied

- This is normal in some lab setups - your user may not be allowed to read IAM details
- You can still complete the console-based steps; note the limitation and move on

---

## Summary

Successfully completed:
- Signed in to the AWS Console as your IAM user
- Explored the IAM console and found your user, groups, and permissions
- Located your access keys
- Configured the AWS CLI
- Ran `aws sts get-caller-identity` to confirm your identity
- Inspected your own permissions from the command line

You now have a working console session and a working CLI. Everything else builds on this.

---

## Lab Cleanup

Nothing to clean up - this stage only reads and observes.

---

## Next Steps

Now that you can sign in and see who you are, you're ready for:

**[→ Stage 2: Users, Groups, and Group Policies](../Stage-2-Users-and-Groups/README.md)**

Here you'll learn how permissions are organized and how groups grant access to many users at once.
