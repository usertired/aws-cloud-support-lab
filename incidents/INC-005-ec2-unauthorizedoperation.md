# INC-005 — EC2 UnauthorizedOperation Due to Missing IAM Policy

## Environment
- IAM user `lab-limited`, already granted `AmazonS3ReadOnlyAccess` (see INC-004), no EC2 permissions attached
- AWS CLI, named profile `lab-limited`

## Problem
`aws ec2 describe-instances` failed for the same IAM user that had just been granted working S3 access.

## Impact
The user could read S3 but could not view any EC2 resources — isolated to EC2 only, since S3 access remained functional throughout.

## Investigation
1. Ran the command directly:
   ```
   aws ec2 describe-instances --profile lab-limited
   ```
   Result:
   ```
   An error occurred (UnauthorizedOperation) when calling the DescribeInstances operation: You are not authorized to perform this operation. User: arn:aws:iam::<account-id>:user/lab-limited is not authorized to perform: ec2:DescribeInstances because no identity-based policy allows the ec2:DescribeInstances action
   ```
   Notably, EC2 returns `UnauthorizedOperation` for this condition, while S3/IAM return `AccessDenied` for the equivalent case (no policy grants the action) — same underlying cause, different error name depending on which AWS service API is called.
2. Confirmed by listing the user's attached policies:
   ```
   aws iam list-attached-user-policies --user-name lab-limited --profile personal
   ```
   Result showed only `AmazonS3ReadOnlyAccess` attached — no EC2-related policy present, confirming the gap directly rather than inferring it from the error message alone.

## Root Cause
No IAM policy granting any `ec2:*` permissions was attached to the user. S3 access existed independently and had no bearing on EC2 permissions — each AWS service's permissions are managed independently under IAM.

## Commands Used
```
aws ec2 describe-instances --profile lab-limited
aws iam list-attached-user-policies --user-name lab-limited --profile personal
aws iam attach-user-policy --user-name lab-limited --policy-arn arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess --profile personal
aws iam list-attached-user-policies --user-name lab-limited --profile personal
aws ec2 describe-instances --profile lab-limited
```

## Resolution
Attached the AWS-managed policy `AmazonEC2ReadOnlyAccess`. Verified both the policy attachment (via `list-attached-user-policies`) and functional access (via a successful `describe-instances` call).

## Prevention
- Don't assume permissions "carry over" between AWS services for the same IAM identity — each service's actions are governed by their own permission namespace (`s3:*`, `ec2:*`, etc.), and a policy for one grants nothing for another.
- When diagnosing a permissions issue, confirm the actual attached policies directly (`list-attached-user-policies`) rather than relying solely on the error message — the error tells you what's missing, but checking the policy list confirms *why* and rules out other causes like a misconfigured policy or propagation delay.

## What I'd Do Differently in Production
For a real support/ops role, I'd bundle related read-only permissions (S3 + EC2 + CloudWatch, for example) into a single custom IAM policy or group, rather than attaching individual AWS-managed policies one at a time — easier to audit and assign consistently across multiple users with the same job function.
