# Problem 1.2 – Generating a Least‑Privilege IAM Policy Based on Access Patterns

## Problem statement

Granting broad permissions to users or roles can expose your AWS account to accidental or malicious misuse.  To follow the principle of least privilege you need to scope down permissions so that an IAM principal can perform only the actions it actually uses.  AWS Access Analyzer can examine CloudTrail logs and generate a policy that reflects real usage.  Your task is to generate such a policy for a user or role in your account.

## Solution

The overall approach is:

1. **Enable CloudTrail and gather usage data.**  Access Analyzer reads CloudTrail logs, so ensure your account has a trail capturing management events for the period you want to analyze.
2. **Start a policy‑generation job.**  Use the `aws accessanalyzer start-policy-generation` command to request a least‑privilege policy for a specific principal.  AWS will analyze the last 90 days of CloudTrail events for that principal.
3. **Poll for completion and retrieve the policy.**  Check the job status until it is `SUCCEEDED`, then retrieve the generated policy document.
4. **Review and refine the policy.**  Inspect the generated policy and adjust it manually if necessary (for example, remove unused services or tighten resource scopes).
5. **Create and attach the policy.**  Save the policy as a JSON file, create a managed policy, and attach it to the principal.

### Example using the AWS CLI

1. **Start policy generation** for a role.  The Policy as Code guide provides a script that uses `aws accessanalyzer start‑policy‑generation` to begin the analysis【173543730236958†L668-L703】.  Replace the principal ARN with the user or role you are analyzing.

```bash
# Start the job and capture the job ID
ANALYSIS_ID=$(aws accessanalyzer start-policy-generation \
  --policy-generation-details '{
    "principalArn": "arn:aws:iam::123456789012:role/MyApplicationRole"
  }' \
  --query 'jobId' --output text)

echo "Started policy generation job: $ANALYSIS_ID"
```

2. **Wait for completion.**  Poll the job status until it succeeds【173543730236958†L668-L703】:

```bash
while true; do
  STATUS=$(aws accessanalyzer get-generated-policy \
    --job-id "$ANALYSIS_ID" \
    --query 'jobDetails.status' --output text)
  echo "Status: $STATUS"
  if [ "$STATUS" = "SUCCEEDED" ]; then
    break
  elif [ "$STATUS" = "FAILED" ]; then
    echo "Policy generation failed"; exit 1
  fi
  sleep 30
done
```

3. **Retrieve the generated policy.**  When the job completes, request the generated policy and save it:

```bash
# Save the least‑privilege policy to a file
aws accessanalyzer get-generated-policy \
  --job-id "$ANALYSIS_ID" \
  --query 'generatedPolicyResult.generatedPolicies[0].policy' \
  --output text > least-privilege-policy.json

echo "Generated policy saved to least-privilege-policy.json"
```

4. **Create and attach the policy** to your principal.  You can turn the JSON into a customer managed policy and attach it:

```bash
# Create a customer managed policy
aws iam create-policy \
  --policy-name MyLeastPrivilegePolicy \
  --policy-document file://least-privilege-policy.json

# Attach the policy to a role
aws iam attach-role-policy \
  --role-name MyApplicationRole \
  --policy-arn arn:aws:iam::123456789012:policy/MyLeastPrivilegePolicy
```

### Why this works and trade‑offs

* **Data‑driven scoping.**  Access Analyzer reviews CloudTrail events to determine which services, actions and resources the principal actually used.  The generated policy contains only those permissions, providing a strong starting point for least privilege.
* **Iterative refinement.**  You can regenerate the policy periodically as usage patterns change.  The policy does not grant anything that was not observed, so infrequently used actions may need to be added manually.
* **Automation friendly.**  The job and retrieval commands are fully scriptable; you can integrate them into CI/CD pipelines or periodically run them to keep policies up to date.

Trade‑offs:

* **Requires CloudTrail logs.**  Without recent log data, Access Analyzer cannot determine what was used.  New principals may need broader permissions initially until enough activity is recorded.
* **May miss rare operations.**  If a critical operation is performed infrequently, it may not appear in the analysis window and will be omitted from the generated policy.  Always review and adjust the output.

## Verification steps

1. Review the generated JSON file manually to ensure it includes all necessary actions and restricts resource ARNs appropriately.
2. Attach the policy and run your typical workflows.  If any operations fail due to `AccessDenied`, add the missing actions and regenerate or edit the policy.
3. Use the IAM policy simulator (see problem 1.4) to test specific actions against the new policy before deploying it broadly.
