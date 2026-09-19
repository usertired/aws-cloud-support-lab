# INC-007 — Validating Self-Healing Infrastructure via Auto Scaling

## Environment
- Custom AMI (`lab-ami-v1`) created from the existing EC2 instance
- Launch Template (`lab-lt`) referencing that AMI, instance type `t3.micro`, existing key pair and Security Group
- Auto Scaling Group (`lab-asg`) — min 1, max 2, desired 1 — in the Week 2 VPC's public subnet

## Problem
Simulated scenario: an EC2 instance fails unexpectedly (hardware failure, unrecoverable crash) and needs to be replaced without manual intervention.

## Impact
Without Auto Scaling, a terminated instance stays terminated — someone has to notice and manually relaunch it, which means downtime until a human responds. This exercise validates whether the infrastructure can recover on its own before that becomes a real incident.

## Investigation / Procedure
1. Created a custom AMI from the running, already-configured instance:
   ```
   aws ec2 create-image --instance-id <original-instance-id> --name "lab-ami-v1" --no-reboot --query "ImageId" --output text --profile personal
   ```
   Waited for `State: available` via `describe-images` before using it.

2. Defined a Launch Template referencing the AMI, instance type, key pair, and Security Group as a JSON file, then created it:
   ```
   aws ec2 create-launch-template --launch-template-name lab-lt --version-description v1 --launch-template-data file://launch-template.json --profile personal
   ```
   Note: the JSON file initially failed to parse (`Expected: '=', received: 'ï'`) because `Out-File -Encoding utf8` in PowerShell prepends a UTF-8 BOM (Byte Order Mark) that the AWS CLI's JSON parser rejects. Fixed by regenerating the file with `-Encoding ascii` instead, since the content had no special characters requiring UTF-8.

3. Created the Auto Scaling Group, tied to the Launch Template and the Week 2 subnet:
   ```
   aws autoscaling create-auto-scaling-group --auto-scaling-group-name lab-asg --launch-template "LaunchTemplateName=lab-lt,Version=1" --min-size 1 --max-size 2 --desired-capacity 1 --vpc-zone-identifier <subnet-id> --profile personal
   ```
   This command returns no output on success — confirmed creation separately via `describe-auto-scaling-groups`, which showed one instance already `InService` and `Healthy`, launched automatically from the template without a manual `run-instances` call.

4. Simulated an instance failure by terminating it directly:
   ```
   aws ec2 terminate-instances --instance-ids <original-asg-instance-id> --profile personal
   ```

5. Waited ~1-2 minutes and re-checked the group:
   ```
   aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names lab-asg --query "AutoScalingGroups[0].Instances" --profile personal
   ```
   A new instance, with a different Instance ID, appeared automatically — the ASG detected actual capacity (0) had dropped below desired capacity (1) and launched a replacement without any manual action.

6. Verified the replacement was actually functional, not just "existing":
   ```
   aws ec2 describe-instances --instance-ids <new-instance-id> --query "Reservations[0].Instances[0].PublicIpAddress" --output text --profile personal
   ssh -i "mi-primera-ec2.pem" ubuntu@<new-ip>
   ```
   Successful SSH login confirmed the replacement instance was fully operational, correctly configured from the Launch Template (same key pair, same Security Group access).

## Root Cause
N/A (planned resilience exercise, not an organic failure) — the only real hiccup was the Launch Template JSON encoding issue in step 2, root-caused to PowerShell's default UTF-8 output including a BOM that AWS CLI's parser doesn't tolerate.

## Resolution
Auto Scaling Group replaced the terminated instance automatically, restoring desired capacity without manual intervention. Verified end-to-end functionality (not just instance existence) via SSH.

## Prevention
- When generating JSON files for AWS CLI `--cli-input-json` or `file://` parameters on Windows/PowerShell, use `-Encoding ascii` (for ASCII-safe content) or explicitly strip the BOM — this is a recurring gotcha specific to PowerShell's `Out-File` defaults, not an AWS CLI issue.
- An Auto Scaling Group's self-healing only works as well as its Launch Template — since the template was built from a manually-configured AMI, any configuration drift in the source instance before the AMI was taken would be baked into every future replacement. This is a good argument for building AMIs (or better, using Terraform/user-data scripts) from a reproducible, version-controlled source instead of an ad hoc console/CLI-configured instance.

## What I'd Do Differently in Production
Attach the Auto Scaling Group to a Load Balancer with health checks, rather than relying solely on EC2/ASG's own health checks — ASG alone only notices an instance is gone (terminated), not that it's unresponsive while still technically "running" (e.g., an app crash that leaves the OS up). An ALB health check would catch that second failure mode, which this exercise didn't cover.
