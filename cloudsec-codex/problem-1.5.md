# Problem\u00a01.5\u00a0\u2013 Delegating IAM Administrative Capabilities Using Permissions Boundaries

## Problem statement

In many organizations, developers or platform engineers need the ability to create and manage IAM roles, policies and service integrations.  Granting full IAM administration rights is risky because it allows them to grant themselves or others any permission.  You need to delegate a subset of IAM administrative tasks while ensuring that delegated administrators cannot exceed predefined limits.  Permissions boundaries provide a mechanism to cap the maximum permissions a user or role can assume, acting as a guardrail.

## Solution

The strategy is to define a **permissions boundary policy** that allows routine development actions and explicitly denies high‑risk IAM and account‑level operations.  You then attach this boundary to any role or user who will be delegated administrative capabilities.  When a policy evaluation occurs, the permissions boundary acts as an additional filter – an identity’s permissions cannot exceed what the boundary allows.【173543730236958†L160-L197】

### Step 1 – Create a permissions boundary policy

Define a JSON policy that allows specific services and denies dangerous actions.  The Policy as Code guide provides an example boundary that denies IAM and Organizations administration and allows common development services【173543730236958†L160-L197】:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyHighRiskActions",
      "Effect": "Deny",
      "Action": [
        "iam:CreateRole",
        "iam:DeleteRole",
        "iam:AttachRolePolicy",
        "iam:DetachRolePolicy",
        "organizations:*",
        "account:*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowDevelopmentActions",
      "Effect": "Allow",
      "Action": [
        "ec2:*",
        "s3:*",
        "lambda:*",
        "logs:*",
        "cloudformation:*"
      ],
      "Resource": "*"
    }
  ]
}
```

Save this document as `permissions-boundary.json` and create a managed policy:

```bash
aws iam create-policy \
  --policy-name DevPermissionsBoundary \
  --policy-document file://permissions-boundary.json
```

Take note of the resulting policy ARN; you will use it as the boundary.

### Step 2 – Create a delegated admin role

Create an IAM role that grants selected IAM actions (such as creating roles, attaching policies, etc.).  Attach the permissions boundary from step 1 so that the role’s effective permissions are capped.  The Policy as Code guide shows how to attach a boundary in Terraform by setting the `permissions_boundary` argument on the role resource【173543730236958†L470-L490】; the same concept applies with the AWS CLI.

```bash
# Trust policy allowing a specific group to assume the delegated admin role
cat > delegate-admin-trust.json <<'POLICY'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::123456789012:group/AdminDelegates"},
      "Action": "sts:AssumeRole"
    }
  ]
}
POLICY

# Create the role
aws iam create-role \
  --role-name delegated-admin-role \
  --assume-role-policy-document file://delegate-admin-trust.json \
  --permissions-boundary arn:aws:iam::123456789012:policy/DevPermissionsBoundary

# Create an inline policy that allows selected IAM actions (scoped to a path)
cat > delegated-admin-policy.json <<'POLICY'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "iam:CreateRole",
        "iam:PutRolePolicy",
        "iam:DeleteRole",
        "iam:TagRole"
      ],
      "Resource": "arn:aws:iam::123456789012:role/developer/*"
    }
  ]
}
POLICY

# Attach the inline policy
aws iam put-role-policy \
  --role-name delegated-admin-role \
  --policy-name DelegateLimitedIAM \
  --policy-document file://delegated-admin-policy.json
```

Developers or platform engineers who are members of the `AdminDelegates` group can now assume `delegated-admin-role` and perform only the allowed actions.  The permissions boundary ensures that even if a broader policy is attached later, the role cannot exceed the limits defined in `DevPermissionsBoundary`.

### Why this works and trade‑offs

* **Permissions boundaries act as a guardrail.**  The IAM evaluation logic applies explicit denies in the boundary before any allow statements【173543730236958†L221-L233】.  Even if an attached policy allows a dangerous action, the boundary will block it.
* **Separation of duties.**  You can grant users the ability to manage roles within a specific path or service without giving them full IAM administration.
* **Composable.**  The same boundary can be reused for multiple delegated roles or users.

Trade‑offs:

* **Complexity.**  Permissions boundaries introduce another layer of policy evaluation.  Misconfigured boundaries may inadvertently block necessary actions.
* **Not a grant.**  A boundary cannot grant permissions; it only limits them.  You must still attach policies granting the desired actions.

## Verification steps

1. Assume the delegated admin role and attempt to create a role within the allowed path.  The operation should succeed.  Attempt to attach an AWS managed policy not permitted by the boundary (e.g., `AdministratorAccess`) and observe that it is denied.
2. Use `aws iam simulate-principal-policy` (see problem 1.4) with the permissions boundary included to test the delegated role’s permissions.
3. Review CloudTrail logs to ensure that delegated administrators cannot perform actions outside the boundary.
