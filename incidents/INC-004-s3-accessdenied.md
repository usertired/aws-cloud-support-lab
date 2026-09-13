# INC-004 — S3 AccessDenied Due to Missing IAM Policy

## Environment
- IAM user `lab-limited`, created with no attached policies (intentionally, for this exercise)
- AWS CLI, separate named profile (`lab-limited`) configured with this user's credentials

## Problem
`aws s3 ls` failed for a newly created IAM user.

## Impact
The user had valid, working credentials but could not perform any S3 read operations — expected behavior for a new IAM user, since AWS denies all actions by default until a policy explicitly allows them.

## Investigation
Ran the command directly to capture the exact error:
```
aws s3 ls --profile lab-limited
```
Result:
```
An error occurred (AccessDenied) when calling the ListBuckets operation: User: arn:aws:iam::<account-id>:user/lab-limited is not authorized to perform: s3:ListAllMyBuckets because no identity-based policy allows the s3:ListAllMyBuckets action
```
The error message itself names the exact missing permission (`s3:ListAllMyBuckets`), which is standard for IAM `AccessDenied` errors — no further digging was needed to identify the gap.

## Root Cause
The IAM user had no identity-based policy attached, so by AWS's default-deny model, every action was implicitly denied.

## Commands Used
```
aws s3 ls --profile lab-limited
aws iam attach-user-policy --user-name lab-limited --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess --profile personal
aws s3 ls --profile lab-limited
```

## Resolution
Attached the AWS-managed policy `AmazonS3ReadOnlyAccess`, granting read-only access to S3 — deliberately scoped to the minimum needed rather than a broad policy like `AdministratorAccess`. Verified the command succeeded afterward with no error.

## Prevention
- New IAM users/roles should always start with zero permissions and have policies added incrementally based on demonstrated need (least privilege), rather than starting broad and trying to restrict later.
- AWS-managed policies (like `AmazonS3ReadOnlyAccess`) are a fast, low-risk way to grant common permission sets without hand-writing a custom JSON policy, and are a reasonable default before reaching for finer-grained custom policies.

## What I'd Do Differently in Production
Use a custom, more narrowly scoped policy instead of the AWS-managed `AmazonS3ReadOnlyAccess` if the user only needs to read from one specific bucket — the managed policy grants read access to *every* bucket in the account, which is broader than true least privilege for most real use cases.
