# Capstone 3: EKS Secure Cluster

**Status:** 🔲 Not Started
**Target week:** Week 49-50

## Goal
Stand up an EKS cluster with network policies, IRSA (IAM Roles for Service Accounts), and a container image scanning gate in the deployment pipeline.

## Architecture
```
(Add diagram: VPC -> EKS control plane -> node groups -> pods with IRSA -> ECR with image scanning)
```

## Services Used
- EKS
- ECR (with scan on push)
- IAM (IRSA)
- VPC CNI / network policies
- CodePipeline with Trivy scanning stage

## Build Log
| Date | What I did | Issues hit | Fix |
|---|---|---|---|

## Outcome / Screenshots
-

## Lessons Learned
-
