# INC-002 — Total Loss of Internet Connectivity After Route Table Default Route Deletion

## Environment
- AWS EC2 instance (Ubuntu 22.04 LTS, t3.micro) in a custom VPC (10.0.0.0/16), public subnet (10.0.1.0/24)
- Custom route table associated with the public subnet, with a default route (0.0.0.0/0) pointing to an Internet Gateway
- Access managed via AWS CLI (no console changes)

## Problem
SSH connection to the EC2 instance failed. Security Group rules for port 22 were unchanged and correct.

## Impact
Complete loss of all inbound/outbound internet connectivity for every resource in the subnet — not limited to SSH. Any service running on the instance would have been equally unreachable, and the instance itself would have lost outbound internet access too.

## Investigation
1. Verified the Security Group still had the correct inbound rule for port 22 — ruled out a Security Group issue (already the root cause of INC-001, checked first based on that prior experience).
2. Inspected the subnet's associated route table:
   ```
   aws ec2 describe-route-tables --route-table-ids <route-table-id> --profile personal
   ```
   Found only one route present: `10.0.0.0/16 -> local` (internal VPC traffic). The default route `0.0.0.0/0 -> igw-...` was missing.
3. Attempted SSH to confirm the failure signature:
   ```
   ssh -i "mi-primera-ec2.pem" ubuntu@<public-ip>
   ```
   Result: `Connection timed out` — same signature as a Security Group block, which is why checking the Security Group first (and ruling it out) mattered before moving to the network layer.

## Root Cause
The route table's default route (0.0.0.0/0 pointing to the Internet Gateway) had been deleted. Without this route, the subnet had no path to send or receive traffic outside the VPC's internal CIDR block — a Security Group can be perfectly configured and still have zero effect if there's no route to reach the resource at all.

## Commands Used
```
aws ec2 describe-route-tables --route-table-ids <route-table-id> --profile personal
aws ec2 delete-route --route-table-id <route-table-id> --destination-cidr-block 0.0.0.0/0 --profile personal
ssh -i "mi-primera-ec2.pem" ubuntu@<public-ip>
aws ec2 create-route --route-table-id <route-table-id> --destination-cidr-block 0.0.0.0/0 --gateway-id <igw-id> --profile personal
```

## Resolution
Re-created the default route (0.0.0.0/0 -> Internet Gateway) via `create-route`. Verified restored access with a successful SSH login.

## Prevention
- When troubleshooting connectivity, check layers in order from most specific to most fundamental: Security Group (instance-level) → NACL (subnet-level) → Route Table (does traffic have a path at all?). A missing route breaks everything downstream, so it's worth confirming early if the symptom is broader than one service or one port.
- Route tables, like Security Groups, should be managed as Infrastructure as Code in a team setting — a manually deleted route has no audit trail explaining who removed it or why.

## What I'd Do Differently in Production
Set up a CloudWatch alarm or AWS Config rule to detect when a route table's default route is removed — this kind of change silently breaks everything with no error message from the resource itself, so proactive detection matters more than for something like a Security Group misconfiguration, which at least fails predictably on the affected port.
