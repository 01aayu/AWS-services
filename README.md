# AWS Services — 2026 DevSecOps Learning Journey

Personal, hands-on documentation of learning AWS services for a DevSecOps career path. Every service has its own folder with a README covering concepts, hands-on labs, IaC, security best practices, and — most importantly — **real errors I hit and how I debugged them**.

**Started:** July 2026
**Target completion:** ~50 weeks
**Format:** One service = one folder = one README, following [`_TEMPLATE.md`](_TEMPLATE.md)

---

## How this repo is organized

```
aws-services/
├── 00-foundations/          IAM, CLI, Billing
├── 01-networking/           VPC, ALB/NLB, Route53, SG/NACL
├── 02-compute/              EC2, Lambda, ECS/Fargate, EKS, ECR
├── 03-storage-db/           S3, EBS/EFS, RDS, DynamoDB
├── 04-security/             KMS, Secrets Manager, GuardDuty, Security Hub,
│                             Inspector, WAF/Shield, Macie, Access Analyzer
├── 05-devsecops-pipeline/   CodePipeline, CodeBuild/Deploy, CloudFormation,
│                             Terraform, image scanning, IaC/secrets scanning
├── 06-observability/        CloudWatch, CloudTrail, Config, X-Ray
├── 07-governance-advanced/  Organizations/SCP, Well-Architected Framework
├── 08-capstone-projects/    3 end-to-end projects combining everything
└── scripts/                 Helper scripts (e.g. README template generator)
```

## Progress Tracker

| Phase | Topic | Status | Target |
|---|---|---|---|
| 0 | [Foundations](00-foundations/README.md) | 🔲 Not Started | Week 1–3 |
| 1 | [Networking](01-networking/README.md) | 🔲 Not Started | Week 4–8 |
| 2 | [Compute](02-compute/README.md) | 🔲 Not Started | Week 9–14 |
| 3 | [Storage & Database](03-storage-db/README.md) | 🔲 Not Started | Week 15–18 |
| 4 | [Security](04-security/README.md) | 🔲 Not Started | Week 19–26 |
| 5 | [CI/CD & IaC](05-devsecops-pipeline/README.md) | 🔲 Not Started | Week 27–34 |
| 6 | [Observability & Governance](06-observability/README.md) | 🔲 Not Started | Week 35–40 |
| 7 | [Governance & Advanced](07-governance-advanced/README.md) | 🔲 Not Started | Week 41–46 |
| 8 | [Capstone Projects](08-capstone-projects/README.md) | 🔲 Not Started | Week 47–50 |

Update the emoji as you go: 🔲 Not Started → 🟡 In Progress → ✅ Completed. Also update the `Status` field inside each individual service README.

## Conventions

- **One folder = one service.** Never mix two services in a folder.
- **Every README follows [`_TEMPLATE.md`](_TEMPLATE.md)** — consistency matters more than perfection.
- **Errors are documented, not deleted.** The "Errors I Hit" table in each README is the actual point of this repo.
- **Commit style:** see [`CONTRIBUTING.md`](CONTRIBUTING.md) for branch naming and commit conventions.
- **Diagrams:** store as `.png`/`.svg` inside the relevant service folder, referenced from its README (or use ASCII/mermaid diagrams inline).
- **No secrets, no real account IDs, no real ARNs** committed — see `.gitignore` and always sanitize CLI output before pasting into a README.

## Why this repo exists

Not just a study log — this is meant to double as a portfolio artifact for DevSecOps roles. It should show: breadth across AWS services, depth on security-specific services, comfort with IaC and CI/CD, and — critically — the ability to debug real failures, not just follow tutorials.

---

*Last updated: July 2026*
