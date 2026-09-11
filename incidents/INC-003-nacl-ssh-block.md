# INC-003 — SSH Access Blocked by Network ACL Deny Rule

## Environment
- AWS EC2 instance (Ubuntu 22.04 LTS, t3.micro) in a custom VPC, public subnet
- Default Network ACL associated with the subnet (originally allow-all, both directions)
- Security Group and route table already confirmed correctly configured (see INC-001, INC-002)

## Problem
SSH connection to the EC2 instance failed despite the Security Group and route table both being correctly configured.

## Impact
Loss of inbound SSH access to the instance. Unlike INC-002, this was scoped specifically to port 22 rather than all traffic, since the deny rule targeted TCP/22 only.

## Investigation
1. Confirmed the Security Group's inbound rule for port 22 was present and correct.
2. Confirmed the route table still had its default route to the Internet Gateway (already fixed from INC-002).
3. With both the instance-level (Security Group) and routing layers ruled out, checked the subnet-level firewall — the Network ACL:
   ```
   aws ec2 describe-network-acls --network-acl-ids <acl-id> --profile personal
   ```
   Found an explicit `deny` rule for TCP/22 inbound, rule number 90 — numerically lower than the default `allow all` rule (100).
4. Attempted SSH to confirm the signature:
   ```
   ssh -i "mi-primera-ec2.pem" ubuntu@<public-ip>
   ```
   Result: `Connection timed out` — indistinguishable from a Security Group or route table block from the client side alone, which is why isolating the layer requires checking each one directly rather than guessing from the SSH error message.

## Root Cause
A custom NACL rule (rule #90, deny TCP/22 inbound) took precedence over the default allow-all rule (#100), because NACLs evaluate rules in ascending numerical order and stop at the first match. Unlike Security Groups, NACLs are stateless — they inspect every packet independently regardless of whether it belongs to an already-established connection, so this rule blocked the SSH handshake outright.

## Commands Used
```
aws ec2 describe-network-acls --filters "Name=association.subnet-id,Values=<subnet-id>" --query "NetworkAcls[0].NetworkAclId" --output text --profile personal
aws ec2 create-network-acl-entry --network-acl-id <acl-id> --rule-number 90 --protocol tcp --port-range From=22,To=22 --cidr-block 0.0.0.0/0 --rule-action deny --ingress --profile personal
ssh -i "mi-primera-ec2.pem" ubuntu@<public-ip>
aws ec2 describe-network-acls --network-acl-ids <acl-id> --profile personal
aws ec2 delete-network-acl-entry --network-acl-id <acl-id> --rule-number 90 --ingress --profile personal
```

## Resolution
Deleted the deny rule (rule #90) via `delete-network-acl-entry`. Verified restored access with a successful SSH login.

## Prevention
- Because NACLs are stateless and evaluated by rule number, any new custom rule needs its number chosen deliberately relative to existing rules — a rule meant to be an exception (e.g., "deny this one thing") must sit *before* (numerically lower than) any broader allow rule it's meant to override.
- Given Security Groups (stateful, instance-level) already provide fine-grained access control, NACLs are best kept close to default (allow-all) unless there's a specific subnet-wide requirement — layering two overlapping stateless/stateful firewalls with independently managed rules increases the surface for exactly this kind of silent conflict.

## What I'd Do Differently in Production
Document any non-default NACL rule inline (via AWS resource tags or a companion IaC comment) explaining its rule number choice and purpose — a bare rule number in a list of dozens is not self-explanatory months later, unlike a Security Group rule, which is usually self-evident from its single CIDR/port pairing.
