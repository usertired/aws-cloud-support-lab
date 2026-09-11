# AWS Cloud Support Lab

Hands-on AWS lab work focused on operating, troubleshooting, and securing cloud infrastructure — built as part of my transition into a Cloud Support Engineer role.

The focus here isn't "I completed a course." It's: build something real, break it on purpose, diagnose it like an incident, fix it, and document exactly how.

## Structure

- `incidents/` — documented incidents following a Problem → Impact → Investigation → Root Cause → Resolution → Prevention format, the way an on-call engineer would write a postmortem.

## Incidents

| ID | Title | Summary |
|---|---|---|
| [INC-001](incidents/INC-001-ssh-access-loss.md) | SSH Access Loss After Security Group Rule Revocation | Diagnosed and resolved a self-induced loss of SSH access by tracing it to a revoked Security Group ingress rule, distinguishing a firewall-level block from an application-level failure. |

## Stack

AWS (EC2, VPC, Security Groups, IAM), AWS CLI, Linux (Ubuntu), PowerShell

## Notes

All infrastructure in this repo is built and torn down within the AWS Free Tier. Cost controls (AWS Budgets) are configured before any resource is created.
