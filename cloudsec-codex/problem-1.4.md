# Problem\u00a01.4\u00a0\u2013 Testing IAM Policies with the IAM Policy Simulator

## Problem statement

When you write or modify IAM policies, it’s important to confirm that they grant only the intended permissions before deploying them.  AWS provides an IAM Policy Simulator that can evaluate identity‑based policies, resource‑based policies and permission boundaries against specific actions.  You want to test whether a user or role can perform certain API actions without affecting your production environment.

## Solution

The IAM Policy Simulator is available through both the AWS Management Console and the AWS CLI.  For repeatable testing and automation, the CLI is preferable.  Use the `simulate-principal-policy` command to evaluate how a set of policies apply to a specific principal for given actions and resources.

### Using the AWS CLI

Follow the steps from AWS re:Post to simulate a policy【137759928141066†L221-L229】:

1. **Identify the principal.**  Determine the ARN of the IAM user or role whose permissions you want to test.
2. **Choose the actions and resources.**  Select the API operations and resource ARNs you want to evaluate.
3. **Run `simulate-principal-policy`.**  The command returns evaluation results indicating whether the action is allowed, implicitly denied or explicitly denied.

#### Example: testing a user’s S3 and EC2 permissions

```bash
# test whether user 'dev-user' can put an object into an S3 bucket and describe EC2 instances
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:user/dev-user \
  --action-names "s3:PutObject" "ec2:DescribeInstances"
```

The output includes `EvaluationResults` for each action with an `EvalDecision` of `allowed`, `implicitDeny` or `explicitDeny`.

#### Example: including a resource‑based policy

To test a resource policy together with the principal’s policies, define the policy in a file and pass it with `--resource-policy` and `--resource-arns`【137759928141066†L243-L260】:

```json
// resource-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "arn:aws:iam::123456789012:user/dev-user",
      "Action": "s3:PutObject",
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ]
    }
  ]
}
```

```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:user/dev-user \
  --action-names "s3:PutObject" \
  --resource-policy file://resource-policy.json \
  --resource-arns arn:aws:s3:::my-bucket
```

#### Testing permissions boundaries

The simulator can also include permissions boundaries in the evaluation.  Create a JSON array of boundary policies and pass it via `--permissions-boundary-policy-input-list`【137759928141066†L292-L304】:

```bash
cat > boundary-policy.json <<'POLICY'
[{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":["ec2:*","iam:*","s3:*"],"Resource":"*"}]}]
POLICY

aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:user/dev-user \
  --action-names "s3:PutObject" "ec2:DescribeInstances" \
  --permissions-boundary-policy-input-list file://boundary-policy.json
```

### Why this works and trade‑offs

* **No risk to production.**  The simulator evaluates policies without executing the actions, so you can test deny scenarios safely.
* **Fine‑grained evaluation.**  You can test multiple actions and resources simultaneously, include resource policies and boundaries, and supply context keys to simulate conditions.
* **Useful for debugging.**  The output shows which statements were matched and whether an explicit deny is the cause of a failure.

Trade‑offs:

* The simulator does not evaluate service control policies (SCPs) and cannot test cross‑account access【137759928141066†L160-L163】.
* Some service features (for example, wildcard conditions or dynamic values) may behave differently in real operations.  Always test critical changes in a non‑production environment as well.

## Verification steps

1. Define a set of actions and resources that correspond to your application’s workflow.
2. Run `simulate-principal-policy` and confirm that allowed actions return `allowed` and that unneeded actions return `implicitDeny` or `explicitDeny`.
3. If testing a new policy, attach it to a test user or role and rerun the simulation to ensure that the result matches expectations.  Iterate on the policy until the simulation reflects the intended access pattern.
