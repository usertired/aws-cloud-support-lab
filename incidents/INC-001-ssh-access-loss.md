# INC-001 — SSH Access Loss After Security Group Rule Revocation

## Environment
- AWS EC2 instance (Ubuntu 22.04 LTS, t3.micro)
- Custom Security Group with a single inbound rule for SSH (TCP/22) restricted to a specific source IP (/32)
- Access managed via AWS CLI (no console changes)

## Problem
SSH connection to the EC2 instance failed after previously working without issue.

## Impact
Complete loss of terminal/administrative access to the instance. The AWS Console (web UI) remained accessible, so the instance itself was healthy and running — only SSH access was affected.

## Investigation
1. Confirmed the instance state was `running` via:
   ```
   aws ec2 describe-instances --instance-ids i-02f59db42aeef082e --query "Reservations[0].Instances[0].[State.Name,PublicIpAddress]" --output table --profile personal
   ```
   Ruled out an instance-level failure (stopped/terminated instance, crashed OS, etc.).

2. Inspected the Security Group's inbound rules and found the rule permitting TCP/22 from the authorized source IP was missing.

3. Attempted the SSH connection directly to confirm the exact failure signature:
   ```
   ssh -i "mi-primera-ec2.pem" ubuntu@<public-ip>
   ```
   Result: `Connection timed out` — not `Connection refused`. This distinction mattered: a refused connection would point to the SSH daemon itself (not running, wrong port); a timeout points to traffic being silently dropped upstream, consistent with a firewall/Security Group blocking the port rather than an application-level failure.

## Root Cause
The Security Group's inbound rule allowing SSH (port 22) from the authorized IP had been revoked, leaving no path for inbound SSH traffic to reach the instance.

## Commands Used
```
aws ec2 revoke-security-group-ingress --group-id sg-0f13daedb63caaebc --protocol tcp --port 22 --cidr <ip>/32 --profile personal
ssh -i "mi-primera-ec2.pem" ubuntu@<public-ip>
aws ec2 authorize-security-group-ingress --group-id sg-0f13daedb63caaebc --protocol tcp --port 22 --cidr <ip>/32 --profile personal
```

## Resolution
Re-added the inbound rule (TCP/22, source IP restricted to /32) via `authorize-security-group-ingress`. Verified restored access with a successful SSH login.

## Prevention
- Before modifying Security Group rules in a live environment, capture the current rule set (`describe-security-groups`) so changes can be reverted quickly if something breaks.
- In a team setting, Security Group rules should be managed as Infrastructure as Code (e.g., Terraform) rather than through ad hoc CLI/console changes, so the desired state is versioned and any drift is easy to detect and roll back.
- A public Elastic IP was not in use here, which also means the instance's public IP changes on every stop/start — worth keeping in mind when diagnosing "can't connect" issues, since a stale IP looks identical to a real access problem at first glance.

## What I'd Do Differently in Production
Use a bastion host or AWS Systems Manager Session Manager instead of direct SSH exposure on port 22, even restricted to a single IP — removes the need to manage inbound SSH rules entirely.
