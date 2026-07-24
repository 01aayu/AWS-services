# Capstone 1: Secure 3-Tier Web App

**Status:** 🔲 Not Started
**Target week:** Week 47

## Goal
Deploy a secure 3-tier web application (Web → App → DB) on a custom VPC, provisioned entirely via Terraform, with a CI/CD pipeline that runs security scans before every deploy.

## Architecture
```
(Add diagram: Internet -> WAF -> ALB -> ASG (App tier, private subnet) -> RDS (private subnet))
```

## Services Used
- VPC (public/private subnets, NAT, IGW)
- ALB + WAF
- Auto Scaling Group (EC2)
- RDS (Multi-AZ, encrypted)
- IAM roles (least privilege per component)
- Terraform for all provisioning
- CodePipeline + CodeBuild with Checkov/tfsec gate

## Build Log
| Date | What I did | Issues hit | Fix |
|---|---|---|---|

## Outcome / Screenshots
-

## Lessons Learned
-
