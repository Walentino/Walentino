# Problem 1.1 – Creating and Assuming an IAM Role for Developer Access

## Problem statement

Teams often grant developers full administrative access so that they can build and test applications quickly.  While convenient, this approach violates the principle of least privilege and increases the risk of mistakes.  To avoid using long‑lived administrator credentials for day‑to‑day work, you need to create an IAM role that developers can assume.  The role should be trusted by your IAM user or group and include only the permissions developers need for their tasks.

## Solution

The high‑level steps are:

1. **Create a trust policy.**  Define which IAM principal (user, group or role) can assume the developer role.  In JSON this is known as the role’s trust policy.
2. **Create the role and attach a permissions policy.**  Use the AWS CLI or Terraform to create the role with the trust policy and attach a managed or custom policy that grants the required permissions.
3. **Assume the role when working.**  Developers invoke `sts:AssumeRole` to get temporary credentials for the developer role.  Those credentials are used for subsequent API calls and expire automatically.
4. **Verify the role.**  Use `aws sts get‑caller‑identity` and run a few API calls to ensure that the role grants the expected access and nothing more.

### Example using the AWS CLI

1. **Create a trust policy file** (`dev‑role‑trust.json`) specifying the principal allowed to assume the role.  Replace `123456789012` and `Alice` with your account ID and IAM user name.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::123456789012:user/Alice"},
      "Action": "sts:AssumeRole"
    }
  ]
}
```

2. **Create the role** and attach a permissions policy.  In this example we attach the AWS‑managed **PowerUserAccess** policy; substitute your own policy to fit your use case.  The AWS re:Post article shows the `create‑role` and `attach‑role‑policy` commands【854151067034376†L249-L263】.

```bash
# create the role
aws iam create-role \ 
  --role-name dev-role \ 
  --assume-role-policy-document file://dev-role-trust.json

# attach a permissions policy (e.g., PowerUserAccess)
aws iam attach-role-policy \ 
  --role-name dev-role \ 
  --policy-arn arn:aws:iam::aws:policy/PowerUserAccess

# confirm that the policy is attached
aws iam list-attached-role-policies --role-name dev-role
```

3. **Assume the role** when doing development work.  Use `aws sts assume-role` to retrieve temporary credentials【854151067034376†L319-L326】:

```bash
# assume the role
aws sts assume-role \ 
  --role-arn arn:aws:iam::123456789012:role/dev-role \ 
  --role-session-name dev-session 

# export the temporary credentials (Bash example)
export AWS_ACCESS_KEY_ID=<RoleAccessKeyID>
export AWS_SECRET_ACCESS_KEY=<RoleSecretAccessKey>
export AWS_SESSION_TOKEN=<RoleSessionToken>

# verify that you are using the role
aws sts get-caller-identity
```

### Terraform example

If you manage infrastructure as code, Terraform can provision the role.  The snippet below shows a module that creates a role and attaches the **PowerUserAccess** policy.  Pass your principal ARN as a variable for the trust relationship.

```hcl
resource "aws_iam_role" "dev_role" {
  name               = "dev-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = "sts:AssumeRole"
        Principal = {
          AWS = var.principal_arn
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "dev_policy" {
  role       = aws_iam_role.dev_role.name
  policy_arn = "arn:aws:iam::aws:policy/PowerUserAccess"
}
```

## Why this works and trade‑offs

* **Temporary credentials reduce risk.**  Roles issue short‑lived tokens instead of long‑lived IAM user keys.  Developers can log in with their normal accounts and switch roles only when performing work.
* **Principle of least privilege.**  By attaching a scoped policy (such as read‑only access to RDS or specific services), the role limits what developers can do.  You can refine the policy over time based on observed usage.
* **Separation of duties.**  The trust policy defines who can assume the role.  Removing a user or group from the trust relationship immediately revokes access.

Trade‑offs include the operational overhead of creating and managing roles and the need for developers to remember to assume the role before working.  If the attached permissions are too broad, the role may still allow risky actions; conversely, if it is too restrictive, developers may be blocked.

## Verification steps

1. Run `aws sts get-caller-identity` after assuming the role.  The returned ARN should be in the form `arn:aws:sts::<account-id>:assumed-role/dev-role/<session-name>`【854151067034376†L319-L326】.
2. Attempt an allowed action (for example, list S3 buckets or describe EC2 instances) to ensure the role’s policy permits it.
3. Attempt a disallowed action (for example, `aws iam list-users`) to confirm that the role does **not** have unnecessary administrative privileges.
