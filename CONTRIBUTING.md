# Git Workflow for This Repo

Since this is a solo learning repo, the workflow is intentionally lightweight but disciplined —
mimicking real DevSecOps team conventions so the habit sticks.

## Branching Strategy

- `main` — always stable. Only merged, reviewed (self-reviewed) content lives here.
- `service/<phase>-<service-name>` — one branch per service while you're actively learning it.
  - Example: `service/00-iam`, `service/04-guardduty`, `service/08-capstone-3tier`

## Workflow per service

```bash
# 1. Start a new service
git checkout main
git pull origin main
git checkout -b service/04-kms

# 2. Work through the lab, fill in the README as you go, commit incrementally
git add 04-security/kms/README.md
git commit -m "docs(kms): add core concepts and hands-on lab steps"

git add 04-security/kms/README.md
git commit -m "docs(kms): document key rotation error and fix"

# 3. When the service README is complete, update its Status field to ✅
git add 04-security/kms/README.md
git commit -m "docs(kms): mark KMS as completed"

# 4. Merge back to main
git checkout main
git pull origin main
git merge service/04-kms
git push origin main

# 5. Delete the finished branch
git branch -d service/04-kms
git push origin --delete service/04-kms
```

## Commit Message Convention

Follow a simplified Conventional Commits style:

| Type | Use for |
|---|---|
| `docs:` | README content, notes, diagrams |
| `feat:` | New Terraform/IaC code added |
| `fix:` | Correcting a mistake in a previous doc or code |
| `chore:` | Repo structure, scripts, template changes |

Format: `type(scope): short description`
Example: `fix(vpc): correct NAT gateway routing diagram`

## Updating Progress

Every time you finish a service:
1. Update the `Status` field inside that service's own README.
2. Update the checkbox for that service in its phase-level README (e.g. `04-security/README.md`).
3. Update the status emoji for the overall phase in the root `README.md` progress tracker once ALL services in that phase are done.

## Pull Requests (optional but recommended)

Even solo, open a PR from your `service/*` branch into `main` before merging.
It creates a paper trail of your learning and is good practice for real team workflows.
Use the PR template in `.github/PULL_REQUEST_TEMPLATE.md`.
