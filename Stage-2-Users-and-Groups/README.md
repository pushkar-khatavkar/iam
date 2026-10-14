# Stage 2: Users, Groups, and Group Policies

## Objective

In this stage you will learn how AWS organizes permissions. You will explore IAM users and groups, understand how a group hands the same permissions to every member, and practice moving a user between groups - the single cleanest way to grant and revoke access at scale.

---

## Prerequisites

Before starting, make sure you have:
- Completed **Stage 1: Getting Started**
- A working console sign-in and configured AWS CLI
- Access keys for CLI commands

**[← Back to Main Workshop](../README.md)**

---

## Why Groups Exist

Imagine 30 students who all need the same access. You could attach the same policy to all 30 users one at a time - and then re-do it every time the permissions change. That does not scale.

A **group** solves this. You attach the policy **once** to the group, and every member inherits it instantly. Add a user to the group, they get the access. Remove them, they lose it. One change, many users.

> **Best practice:** Attach policies to **groups**, not to individual users. Users should get almost all their permissions through group membership.

---

## Step 1: Explore Existing Users

### Instructions (Console):

1. Go to **IAM Console** → **Users**
2. Look at the list of users - these are the identities your instructor created
3. Click any user and note the **Groups** and **Permissions** tabs

### Instructions (CLI):

List all users in the account:

```bash
aws iam list-users --query 'Users[*].[UserName,UserId]' --output table
```

**Command Breakdown:**
- `list-users` - returns every IAM user in the account
- `--query` - filters the output to just the fields we care about
- `--output table` - formats the result as a readable table

**Expected Output:**
A table of user names and their unique IDs.

---

## Step 2: Explore Existing Groups

### Instructions (Console):

1. Go to **IAM Console** → **User groups**
2. Review the groups that exist
3. Click a group and open its **Permissions** tab to see which policies are attached
4. Open its **Users** tab to see who belongs to it

### Instructions (CLI):

List all groups:

```bash
aws iam list-groups --query 'Groups[*].GroupName' --output table
```

Pick a group name from the output and see which policies it has:

```bash
aws iam list-attached-group-policies --group-name <group-name>
```

See who is in that group:

```bash
aws iam get-group --group-name <group-name> --query 'Users[*].UserName' --output table
```

---

## Step 3: Trace a User's Effective Permissions

A user's real permissions = (directly attached policies) + (policies from every group they belong to). Let's trace one.

### Pick a user and run all three checks:

```bash
# 1. Which groups is the user in?
aws iam list-groups-for-user --user-name <user-name> --query 'Groups[*].GroupName' --output table
```

```bash
# 2. Any policies attached directly to the user?
aws iam list-attached-user-policies --user-name <user-name>
```

```bash
# 3. Any inline policies on the user?
aws iam list-user-policies --user-name <user-name>
```

### Then for each group the user is in:

```bash
aws iam list-attached-group-policies --group-name <group-name>
```

### Observe:

- Most permissions should come from **groups**, not direct attachment
- If a user has a lot of directly attached policies, that is usually a sign the setup could be cleaner
- This tracing skill is exactly what you do when debugging "why can this user do X?"

---

## Step 4: Understand a Group's Policy

Let's actually read what a group's policy allows.

### Instructions:

1. In the **IAM Console** → **User groups**, click a group
2. Open the **Permissions** tab
3. Click a policy name to open it
4. Click the **JSON** tab (or **{ } JSON** view)

### What to Look For:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:Get*",
        "s3:List*"
      ],
      "Resource": "*"
    }
  ]
}
```

Read it in plain English:
- **Effect: Allow** - this grants access (rather than blocking it)
- **Action** - the specific API calls permitted (here, S3 read and list operations)
- **Resource: "\*"** - applies to all resources of that type

Every member of this group can do exactly what this document says - no more, no less.

---

## Step 5: Move a User Between Groups

This is the core group skill: changing a user's access by changing their group membership. **Only do this if your instructor says you have permission to modify group membership** - in many labs this is the one write action students are given.

### Add a user to a group:

```bash
aws iam add-user-to-group --user-name <user-name> --group-name <group-name>
```

**Command Breakdown:**
- `add-user-to-group` - makes the user a member of the group
- The user **immediately** inherits every policy attached to that group

**Expected Output:**
No output on success. In IAM, silence means it worked.

### Verify the change:

```bash
aws iam list-groups-for-user --user-name <user-name> --query 'Groups[*].GroupName' --output table
```

### Remove a user from a group:

```bash
aws iam remove-user-from-group --user-name <user-name> --group-name <group-name>
```

### Verify again:

```bash
aws iam list-groups-for-user --user-name <user-name> --query 'Groups[*].GroupName' --output table
```

### Observe:

- The moment you add the user, their permissions change - no restart, no delay beyond IAM's brief consistency window
- The moment you remove them, the access is gone
- This is why group membership is the preferred lever for granting and revoking access

---

## Step 6: Do the Same in the Console (Optional)

If you prefer clicking to typing:

1. Go to **IAM Console** → **User groups** → click a group
2. Open the **Users** tab
3. Click **Add users** to add a member, or select a user and click **Remove** to take them out
4. Switch to **Users** → click the user → **Groups** tab to confirm the change

Console and CLI do exactly the same thing. Use whichever you prefer.

---

## Troubleshooting

### Issue: "AccessDenied" when adding or removing a user from a group

- Your user may not have `iam:AddUserToGroup` / `iam:RemoveUserFromGroup` permission
- Some labs restrict this to **specific groups only** - check the error message, it names the group it blocked
- Ask your instructor which groups you are allowed to modify

### Issue: "NoSuchEntity: The group ... does not exist"

- The group name is misspelled or does not exist - run `aws iam list-groups` to see valid names
- Group names are case sensitive

### Issue: Permissions did not change after moving the user

- IAM is eventually consistent - wait a few seconds and retry
- Confirm you modified the identity you are actually testing with (`aws sts get-caller-identity`)
- Check whether a directly attached policy or another group is also in play

### Issue: Cannot see group members

- You may lack `iam:GetGroup` permission - this is a read restriction, not a problem with your command

---

## Summary

Successfully completed:
- Explored existing IAM users and groups in the console and CLI
- Traced a user's effective permissions across direct policies and group memberships
- Read and understood a group's policy document
- Added and removed a user from a group and watched their access change
- Learned why groups are the preferred way to manage permissions at scale

---

## Lab Cleanup

If you added yourself or others to groups for testing, return membership to its original state:

```bash
aws iam remove-user-from-group --user-name <user-name> --group-name <group-name>
```

Leave groups your instructor pre-created in place.

---

## Next Steps

Now that you understand how groups carry permissions, you're ready to look inside the policies themselves:

**[→ Stage 3: Policies Deep Dive](../Stage-3-Policies-Deep-Dive/README.md)**

Here you'll learn the three types of policies, write your own, and use an explicit `Deny` to build a guardrail.
