# Billing, Budgets & Cost Explorer

**Phase:** 00 - Foundations
**Status:** ✅ Completed
**Last updated:** 2026-07-26

---

## 1. What it is & when to use it
AWS Billing tools let you monitor, forecast, and control spend before it surprises you. Three pieces matter for a DevSecOps workflow: **Budgets** (proactive alerts before you overspend), **Cost Explorer** (visual breakdown of what's actually costing money, by service/time/tag), and **Cost Allocation Tags** (attributing spend to specific projects/teams/environments — critical once you're not the only person touching an account).

## 2. Core Concepts
- **Budgets** — set a spend threshold, get emailed at defined percentage thresholds (e.g. 80%, 100%). Doesn't stop spending by itself, just alerts.
- **Cost Explorer** — visual/API tool to analyze historical and forecasted spend, filterable by service, region, tag, etc.
- **IAM Billing visibility is root-only by default** — IAM users (even with `PowerUserAccess`) can't see Billing/Cost Explorer until root explicitly enables "IAM User and Role Access to Billing Information" in Account Settings. This is a deliberate AWS security boundary, not a bug.
- **Cost Allocation Tags** — key/value tags applied to resources (e.g. `project: aws-learning`) that, once activated in Billing, let Cost Explorer break down spend by tag. New tags take time (up to ~24h) to appear in the activation list after first being applied to a resource.
- **Zero-spend budgets** are the recommended starting point over a dollar-threshold budget — they alert on the very first cent of spend rather than waiting for a percentage of a larger number.

```

Root (only identity with billing access by default)
│
└── Account Settings → "IAM User and Role Access to Billing" → Activate IAM Access
│
└── aayush-admin can now see Billing / Cost Explorer / Budgets

```

## 3. Hands-on Lab
### Console steps
1. Attempted to open Cost Explorer as `aayush-admin` → option wasn't available (billing locked to root by default)
2. Logged in as root → Account Settings → "IAM User and Role Access to Billing Information" → Activate IAM Access
3. Logged back in as `aayush-admin` → Cost Explorer now enabled, data began populating
4. Tagged an EC2 instance (`simple-ec2`) with `project: aws-learning`
5. Checked Billing → Cost Allocation Tags → `project` tag not yet visible (propagation delay, expected)
6. Activated an existing available tag (`Environment`) instead to confirm the activation mechanism works end-to-end
7. Confirmed the original $1 zero-spend budget from the very first lab is still active; also created a second $10 monthly cost budget with an 80% alert threshold

### CLI steps
```bash
# Check current budgets via CLI
aws budgets describe-budgets --account-id <ACCOUNT_ID>

# Get cost and usage for the last 7 days, grouped by service
aws ce get-cost-and-usage \
  --time-period Start=2026-07-19,End=2026-07-26 \
  --granularity DAILY \
  --metrics "UnblendedCost" \
  --group-by Type=DIMENSION,Key=SERVICE
```

## 4. Infrastructure as Code
```hcl
# Example: AWS Budget as Terraform (not yet applied — for reference)
resource "aws_budgets_budget" "zero_spend" {
  name              = "zero-spend-budget"
  budget_type       = "COST"
  limit_amount      = "1"
  limit_unit        = "USD"
  time_unit         = "MONTHLY"

  notification {
    comparison_operator       = "GREATER_THAN"
    threshold                 = 80
    threshold_type            = "PERCENTAGE"
    notification_type         = "ACTUAL"
    subscriber_email_addresses = ["you@example.com"]
  }
}
```

## 5. Security Best Practices (DevSecOps lens)
- [x] Billing visibility deliberately kept root-gated by default, then explicitly opened for `aayush-admin` (understood the boundary rather than fighting it)
- [x] At least one zero-spend / low-threshold budget active as an early tripwire
- [x] Cost allocation tagging in place for attributing spend to specific work (`project: aws-learning`)
- [ ] Consider tightening budget notification to also alert on FORECASTED spend, not just ACTUAL, for earlier warning
- [ ] Multiple budgets (e.g. one tight $1 tripwire + one broader $10 monthly ceiling) recommended rather than relying on a single $10 budget alone

## 6. Errors I Hit & How I Debugged Them
| Error / Symptom | Root Cause | Fix | Notes |
|---|---|---|---|
| "Enable Cost Explorer" option missing while logged in as `aayush-admin` | Billing/Cost Explorer access is root-only by default, regardless of IAM policy (even `PowerUserAccess`) | Logged in as root → Account Settings → activated "IAM User and Role Access to Billing Information" | This is an account-level setting, not delegable purely through IAM policies — a real exception to the usual IAM model worth remembering |
| `project` tag not appearing in Cost Allocation Tags list right after tagging EC2 instance | New cost allocation tags take time (up to ~24h) to propagate into the activation list | No fix needed — waited it out; activated an already-available tag (`Environment`) to confirm the mechanism in the meantime | Don't assume a tag is broken just because it's not immediately visible after first use |

## 7. Cost Gotchas
- Cost Explorer and Budgets are both free to use.
- A $10 monthly budget is a reasonable ceiling, but pair it with a $1 zero-spend budget too — the $10 threshold alone could let real charges accumulate before the first alert fires.
- Forgotten resources (NAT Gateways, unattached EBS volumes, idle RDS instances) are the most common source of surprise bills — Cost Explorer grouped by service is the fastest way to spot these.

## 8. CLI Cheat Sheet
```bash
# List all budgets on the account
aws budgets describe-budgets --account-id <ACCOUNT_ID>

# Get last 30 days of cost grouped by service
aws ce get-cost-and-usage \
  --time-period Start=2026-06-26,End=2026-07-26 \
  --granularity MONTHLY \
  --metrics "UnblendedCost" \
  --group-by Type=DIMENSION,Key=SERVICE

# List cost allocation tags
aws ce list-cost-allocation-tags
```

## 9. Further Reading
- [AWS Budgets Docs](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html)
- [Cost Allocation Tags Docs](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/cost-alloc-tags.html)