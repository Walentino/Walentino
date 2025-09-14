# Problem 1.3 – Enforcing IAM User Password Policies in Your AWS Account

## Problem statement

An internal security policy requires strong passwords for every IAM user in your AWS account.  Passwords must expire after 90 days and must include at least one lowercase letter, one uppercase letter, one number and one symbol.  You need to enforce this password policy across the account.

## Solution

AWS accounts support a global password policy that applies to all IAM users.  You can set the policy via the console or by using the AWS CLI.  Once configured, any new or existing user must comply with the complexity and rotation requirements.

### Setting the password policy using the AWS CLI

Use the `update-account-password-policy` command with the appropriate flags.  The following example (inspired by an AWS CookBook blog post【332081831164251†L46-L59】) enforces a 90‑day rotation and requires a mix of character types:

```bash
aws iam update-account-password-policy \
  --minimum-password-length 14 \
  --require-symbols \
  --require-numbers \
  --require-uppercase-characters \
  --require-lowercase-characters \
  --allow-users-to-change-password \
  --max-password-age 90 \
  --password-reuse-prevention 10
```

**Key parameters:**

* `--minimum-password-length` – set to a value appropriate for your organization (the cookbook example used 32 characters【332081831164251†L46-L59】; a length of 14 or more is common).
* `--require-symbols`, `--require-numbers`, `--require-uppercase-characters`, `--require-lowercase-characters` – enforce complexity.
* `--max-password-age` – number of days after which the password must be changed (90 in this problem).
* `--password-reuse-prevention` – prevents users from reusing the last *n* passwords (set to 10 in the example).

Once the policy is set, any user who changes their password must conform to these rules.  Existing passwords continue to work until they expire.

### Creating and managing users

You can create users and assign them to groups as usual.  The blog post demonstrates how to create a group, attach a read‑only policy and add users【332081831164251†L60-L99】.  For example:

```bash
# create a group
aws iam create-group --group-name Developers

# attach a managed policy to the group
aws iam attach-group-policy \
  --group-name Developers \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess

# create a user
aws iam create-user --user-name dev-user

# add the user to the group
aws iam add-user-to-group \
  --group-name Developers \
  --user-name dev-user

# create a random password using Secrets Manager and assign it to the user
RANDOM_PW=$(aws secretsmanager get-random-password \
  --password-length 20 \
  --require-each-included-type \
  --query RandomPassword --output text)

aws iam create-login-profile \
  --user-name dev-user \
  --password "$RANDOM_PW" \
  --password-reset-required
```

### Why this works and trade‑offs

* **Central enforcement.**  The account password policy applies to all IAM users and cannot be bypassed by individual users.
* **Complexity and rotation.**  Requiring mixed character types and regular rotation reduces the risk of password compromise.
* **Password reuse prevention.**  Preventing reuse of previous passwords discourages predictable patterns and weak secrets.

Trade‑offs include increased administrative overhead (users must change passwords regularly) and potential user frustration.  Pair password policies with multi‑factor authentication (MFA) for stronger security, and consider using AWS IAM Identity Center or federated identity providers to minimize the number of IAM users.

## Verification steps

1. Run `aws iam get-account-password-policy` to confirm that the policy settings match your requirements.  The blog’s example output shows values such as `RequireSymbols: true`, `MaxPasswordAge: 90` and `PasswordReusePrevention: 10`【332081831164251†L112-L127】.
2. Create a test IAM user and attempt to set a password that violates the policy (e.g., too short or missing a symbol).  The AWS CLI will return an error.
3. After setting a compliant password, wait for the rotation period (or manually expire the password) and verify that the user is prompted to change it.
