# AWS CLI & CloudShell

**Phase:** 00 - Foundations
**Status:** ✅ Completed
**Last updated:** 2026-07-26

---

## 1. What it is & when to use it
The AWS CLI is a command-line tool that talks to the same underlying AWS APIs as the console — anything clickable in the console can be scripted through the CLI. This matters beyond convenience: CI/CD pipelines, Terraform, and automation scripts all rely on the same API surface, so CLI fluency now directly transfers to IaC/pipeline work later. CloudShell is a browser-based terminal built into the AWS Console that comes pre-authenticated as whichever IAM identity is currently logged into the console — zero local setup, useful for quick checks from any machine.

## 2. Core Concepts
- **CLI uses Access Keys** (Access Key ID + Secret Access Key) tied to a specific IAM user — long-lived credentials stored locally in `~/.aws/credentials`.
- **CloudShell uses the console session** — no separate credentials to manage, but only available while logged into the browser console.
- **`aws configure`** sets up the local credentials/region/output-format profile used by every subsequent CLI command.
- **`aws sts get-caller-identity`** is the fastest way to confirm exactly which identity a terminal session is currently authenticated as — critical habit before running anything destructive.
- Best practice: create a **dedicated IAM user for CLI use** (e.g. `aayush-cli`) separate from the console-login admin user, so access keys can be rotated/revoked independently without affecting console access.

```
Local machine (Mac M4) AWS Console (browser)
│ │
├── ~/.aws/credentials ├── logged in as aayush-admin
│ (access keys for aayush-cli) │
│ └── CloudShell inherits this session
└── aws sts get-caller-identity │
→ returns aayush-cli └── aws sts get-caller-identity
→ returns aayush-admin

```

## 3. Hands-on Lab
### Console steps
1. IAM → Users → created `aayush-cli` (console access unchecked — CLI-only identity)
2. Attached permissions for CLI use
3. Security credentials tab → Create access key → use case: Command Line Interface → copied Access Key ID + Secret Access Key
4. Opened CloudShell from the console top nav (no setup required, inherits `aayush-admin` session)

### CLI steps
```bash
# Install (Mac M4 / Apple Silicon)
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /
aws --version   # confirmed arm64 build, correct for M4

# Configure with aayush-cli's access keys
aws configure
# AWS Access Key ID: <entered>
# AWS Secret Access Key: <entered>
# Default region: eu-north-1
# Default output format: json

# Verify identity — confirmed aayush-cli, not root or aayush-admin
aws sts get-caller-identity
```

Result — local CLI:
```json
{
    "UserId": "AIDAWCBCPZ2HUXPAXRVXI",
    "Account": "<ACCOUNT_ID>",
    "Arn": "arn:aws:iam::<ACCOUNT_ID>:user/aayush-cli"
}
```

Result — CloudShell (inherits console session):
```json
{
    "UserId": "AIDAWCBCPZ2HQ7A3P6BSF",
    "Account": "<ACCOUNT_ID>",
    "Arn": "arn:aws:iam::<ACCOUNT_ID>:user/aayush-admin"
}
```

Confirmed: two different terminals, two different identities, exactly as expected — local CLI uses stored access keys for `aayush-cli`, CloudShell uses the browser's logged-in session (`aayush-admin`).

## 4. Infrastructure as Code
```hcl
# N/A - this module is tooling setup, not a provisionable resource
```

## 5. Security Best Practices (DevSecOps lens)
- [x] Dedicated CLI-only IAM user (`aayush-cli`) created instead of reusing console admin credentials
- [x] Console access disabled for the CLI user — reduces blast radius if access keys ever leak
- [x] Verified identity with `sts get-caller-identity` before trusting any terminal session
- [ ] Access keys not yet rotated on a schedule (revisit once we cover Secrets Manager / key rotation policy)
- [ ] Consider IAM Identity Center / temporary SSO credentials instead of long-lived access keys (advanced, later)

## 6. Errors I Hit & How I Debugged Them
| Error / Symptom | Root Cause | Fix | Notes |
|---|---|---|---|
| N/A — clean install and configure on Mac M4 | — | — | `aws --version` confirmed `arm64` build, correct architecture for Apple Silicon |

## 7. Cost Gotchas
- AWS CLI and CloudShell are both free to use. CloudShell has a small persistent storage allowance (1GB) included at no cost.

## 8. CLI Cheat Sheet
```bash
# Confirm current identity
aws sts get-caller-identity

# Check current configured region/profile
aws configure list

# Switch between multiple profiles (useful once you have several IAM users configured locally)
aws configure --profile aayush-cli
aws s3 ls --profile aayush-cli
```

## 9. Further Reading
- [AWS CLI Install Guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [AWS CloudShell Docs](https://docs.aws.amazon.com/cloudshell/)