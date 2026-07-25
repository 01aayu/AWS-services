# IAM (Identity and Access Management)

**Phase:** 00 - Foundations
**Status:** ✅ Completed
**Last updated:** 2026-07-26

---

## 1. What it is & when to use it
IAM is AWS's identity and permission system — it answers two questions for every single action taken in your account: **"Who is this?"** (authentication) and **"Are they allowed to do this?"** (authorization). Every API call, console click, and CLI command gets checked against IAM before it's allowed to happen. You use it constantly — there's no AWS service that exists outside IAM's reach.

## 2. Core Concepts

**Identities (who):**
- **Root user** — the account owner, full unrestricted power, MFA-protected, never used day-to-day.
- **IAM User** — a person or app with long-term credentials (password and/or access keys). Best for human users.
- **IAM Role** — a temporary identity with no permanent credentials. Anyone/anything can "assume" a role and get short-lived credentials. This is how EC2 instances, Lambda functions, and other AWS services get permissions — never by embedding access keys in code.
- **IAM Group** — just a bucket of users that share the same policies. Groups can't be assumed or logged into; they're purely an org tool.

**Permissions (what they can do):**
- **Policy** — a JSON document listing allowed/denied actions on specific resources. Example: "allow `s3:GetObject` on bucket `my-bucket/*`".
- **Identity-based policy** — attached to a user/role/group ("this user can do X").
- **Resource-based policy** — attached to the resource itself, e.g. an S3 bucket policy ("this bucket can be accessed by X").
- **Policy evaluation logic** (critical, gets asked in every interview): explicit **Deny always wins** > explicit **Allow** > implicit **Deny** (default — if nothing says yes, the answer is no).

**The golden rule — Least Privilege:** Give only the exact permissions needed, nothing more. `AdministratorAccess` is fine for day-1 bootstrapping (what you did) but is a liability long-term — it's the #1 root cause behind AWS security incidents you'll read about (leaked keys with admin access = total account takeover).

**Groups (organizational shortcut):** A Group is just a named bundle of policies attached to multiple users at once. Groups cannot log in, cannot be assumed, and have no permissions of their own outside of what's attached to them — they exist purely so you don't manually re-attach the same policy to every new hire. At scale, real teams manage permissions at the group level (`ec2-operators`, `read-only-auditors`, `admins`), not per-user.

**Roles in practice (the most important pattern for DevSecOps):** A role attached to an EC2 instance (or Lambda, ECS task, etc.) hands out short-lived, auto-rotating credentials behind the scenes — the instance never stores a long-term access key. Confirmed hands-on: an EC2 instance with `ec2-s3-readonly-role` attached could run `aws s3 ls` successfully with zero credentials configured locally. This is why hardcoded access keys in code/config are considered a security smell — roles remove the need entirely.

**PowerUserAccess vs AdministratorAccess (permission boundaries in practice):** `PowerUserAccess` grants access to almost everything except IAM user/group/role management and account settings — a good default for a trusted engineer who shouldn't be able to create new identities or change permissions. Confirmed hands-on: after swapping `aayush-admin` from `AdministratorAccess` to `PowerUserAccess`, `iam:CreateUser` was correctly denied. Recovered IAM management ability by logging in as root and attaching `IAMFullAccess` separately — root remains the only account-recovery lever and should never be disabled, deleted, or used for daily work.

```
Root (locked away, MFA'd, emergency-only)
│
├── IAM User: aayush-admin ──attached to──> Policy (currently AdministratorAccess)
│
└── IAM Role ──assumed by──> EC2 / Lambda / other AWS service (no stored credentials)
```

## 3. Hands-on Lab
### Console steps
1. Checked IAM Dashboard → Security recommendations (root MFA already enabled from prior EC2 usage)
2. IAM → Users → Create user → `aayush-admin` → attached `AdministratorAccess` directly (temporary)
3. Enabled console access for `aayush-admin` → generated password → saved sign-in URL
4. Logged into incognito window as `aayush-admin`, confirmed top-right shows the IAM user, not root
5. Created custom IAM policy `ec2-operator-policy` scoped to `ec2:StartInstances` / `StopInstances` / `DescribeInstances` / `DescribeInstanceStatus` only
6. Created second user `aayush-ec2-operator`, attached the custom policy directly (temporarily)
7. Verified: start/stop worked; `s3:ListAllMyBuckets` and `ec2:CreateKeyPair` were both denied as expected
8. Created group `ec2-operators`, attached `ec2-operator-policy` to the group instead
9. Removed the direct policy attachment from `aayush-ec2-operator`, added the user to `ec2-operators` group instead — re-verified start/stop still worked via group-inherited permission
10. Swapped `aayush-admin` from `AdministratorAccess` to `PowerUserAccess` → confirmed `iam:CreateUser` was denied (expected, PowerUserAccess excludes IAM management)
11. Logged in as root, added `IAMFullAccess` to `aayush-admin` alongside `PowerUserAccess` to restore IAM management ability without going back to full admin
12. Re-tested as `aayush-admin`: created and deleted a `test-user` successfully
13. Created IAM Role `ec2-s3-readonly-role` (trusted entity: EC2, policy: `AmazonS3ReadOnlyAccess`), attached it to a running EC2 instance
14. Installed AWS CLI on the instance (apt package unavailable, used official installer instead) and ran `aws s3 ls` — successfully listed buckets with **zero credentials configured on the instance**, proving role-based temporary credentials work

### CLI steps
```bash
# (Pending — CLI not yet configured on local machine, will add once AWS CLI lab is done)
```

## 4. Infrastructure as Code
```hcl
# Pending — will add Terraform for IAM user/policy once we cover least-privilege lab
```

## 5. Security Best Practices (DevSecOps lens)
- [x] Root MFA enabled
- [x] Root not used for daily work — IAM user created instead
- [x] Least privilege applied — demonstrated with `ec2-operator-policy` on `aayush-ec2-operator`
- [x] Permissions managed via groups, not per-user direct attachments
- [x] Access keys avoided in favor of roles — proven with `ec2-s3-readonly-role` on EC2, zero stored credentials
- [x] Admin user (`aayush-admin`) moved off blanket `AdministratorAccess` to `PowerUserAccess` + scoped `IAMFullAccess`
- [x] Root confirmed as the correct recovery lever for a self-inflicted permissions lockout
- [ ] IAM Access Analyzer enabled to catch overly-broad policies (later phase — Security services)
- [ ] MFA enforced for non-root IAM users too (not yet done for `aayush-admin` / `aayush-ec2-operator`)

## 6. Errors I Hit & How I Debugged Them
> This is the most valuable section. Document real errors, root cause, and fix.

| Error / Symptom | Root Cause | Fix | Notes |
|---|---|---|---|
| N/A — root MFA was already configured from earlier EC2 usage | — | — | Confirmed via IAM Dashboard → Security recommendations before proceeding |
| N/A — created IAM user `aayush-admin` with AdministratorAccess attached directly | — | — | Temporary — will be tightened to least-privilege in a later lab once daily workflow is understood |
| N/A — enabled console access + logged in as `aayush-admin` via incognito window | — | — | Confirmed top-right shows `aayush-admin` instead of root email; root should now be locked away and unused for daily work |
| `AccessDenied` on `s3:ListAllMyBuckets` for `aayush-ec2-operator` | Custom `ec2-operator-policy` doesn't grant any S3 permissions | Expected — proves least privilege is enforced | First real proof of a Deny-by-default outcome |
| `UnauthorizedOperation` on `ec2:CreateKeyPair` for `aayush-ec2-operator` | Policy only allows Start/Stop/Describe on EC2, nothing else | Expected — confirms scope is exactly as written, no more | Encoded authorization failure message included full account ID — redact before sharing publicly |
| `iam:CreateUser` denied for `aayush-admin` after swapping to `PowerUserAccess` | `PowerUserAccess` deliberately excludes IAM management | Logged in as root, attached `IAMFullAccess` to `aayush-admin` in addition to `PowerUserAccess` | Real lockout-recovery drill — confirms root is the correct emergency lever, never delete/disable it |
| `Command 'aws' not found` on EC2, then `Package 'awscli' has no installation candidate` via apt | Ubuntu's apt repo doesn't reliably ship a current awscli package | Installed via AWS's official method: downloaded `awscli-exe-linux-x86_64.zip`, unzipped, ran `sudo ./aws/install` | apt install is not a reliable way to get AWS CLI on newer Ubuntu AMIs — use the official installer script instead |

## 7. Cost Gotchas
- IAM itself is completely free — users, groups, roles, policies cost $0 regardless of free tier status.

## 8. CLI Cheat Sheet
```bash
# Confirm which identity you're currently using
aws sts get-caller-identity

# List all IAM users in the account
aws iam list-users

# List policies attached to a specific user
aws iam list-attached-user-policies --user-name aayush-admin

# List all roles
aws iam list-roles
```

## 9. Further Reading
- [AWS IAM Docs](https://docs.aws.amazon.com/iam/)
- [IAM Policy Evaluation Logic](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html)