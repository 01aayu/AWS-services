# Security Groups & NACLs

**Phase:** 01 - Networking
**Status:** ✅ Completed

---

## 1. What it is & when to use it
AWS gives you two separate firewall layers for controlling traffic in and out of a VPC: **Security Groups** (instance-level) and **Network ACLs / NACLs** (subnet-level). They look similar on the surface — both are just lists of allow/deny rules by port and IP — but they behave fundamentally differently, and most real-world confusion in AWS networking comes from treating them as interchangeable. Used together, they form a "defense in depth" model: traffic has to clear both checkpoints to reach an instance.

## 2. Core Concepts

**The hierarchy:**

```
Internet
│
▼
[NACL] ← subnet-level checkpoint, STATELESS, evaluated in rule-number order, first match wins
│
▼
[Subnet: 10.0.1.0/24]
│
▼
[Security Group] ← instance-level checkpoint, STATEFUL, all rules evaluated together
│
▼
[EC2 Instance]

```

**The critical distinction:**

| | Security Group | NACL |
|---|---|---|
| Applies to | Individual instance/ENI | Entire subnet |
| State | **Stateful** — allow inbound, response is auto-allowed out | **Stateless** — inbound and outbound evaluated fully independently |
| Rule type | Allow only, implicit deny on no-match | Allow AND explicit Deny |
| Evaluation | All rules considered together | **Numbered order, first match wins, then stops** |
| Default | Denies everything until rules added | Default NACL allows all; custom NACL denies all until rules added |

**Why stateful vs stateless is the part that actually matters:** in a Security Group, allowing inbound port 80 automatically permits the response traffic back out — no extra rule needed. In a NACL, allowing inbound port 80 does nothing for the response unless you *also* explicitly allow outbound traffic on **ephemeral ports** (the random high-numbered port, typically 1024–65535, that the client's OS picked for the return leg). This is the single most common NACL debugging trap — "inbound looks fine, but the site still doesn't load" is almost always a missing or malformed outbound ephemeral-port rule.

## 3. Hands-on Lab
### Console steps
1. Launched `sg-nacl-test` EC2 in `learning-public-subnet` (reused from the VPC lab)
2. Created a Security Group allowing SSH (22) from My IP and HTTP (80) from `0.0.0.0/0`
3. Installed Apache via user data script on boot, confirmed `http://<public-ip>` loaded correctly — Security Group behaved exactly as configured throughout, no issues
4. Added a NACL inbound **Deny** rule (Rule 50, HTTP/80, `0.0.0.0/0`) on the subnet's NACL → confirmed the site immediately stopped loading, proving the NACL is checked independently of the Security Group and can override it
5. Removed the Deny rule → confirmed site loaded again
6. Removed the NACL's default outbound allow-all rule, replaced with a narrow HTTP-only (port 80) outbound Allow → site broke again, this time with inbound fully open — proved the stateless outbound gotcha
7. Attempted to fix by adding an outbound rule for the ephemeral port range — hit a real bug (see Errors table) before landing on the actual fix
8. Confirmed site restored after fixing the malformed rule
9. Terminated `sg-nacl-test`, reset the NACL back to its clean default-allow-all state

### CLI steps
```bash
# Useful commands for inspecting SG/NACL state without opening the console
aws ec2 describe-security-groups --group-ids <sg-id>
aws ec2 describe-network-acls --network-acl-ids <nacl-id>
```

## 4. Infrastructure as Code
```hcl
# Reference only — not yet applied, will build properly in Phase 5 (Terraform)
resource "aws_security_group" "web_sg" {
  name   = "sg-nacl-test-sg"
  vpc_id = aws_vpc.learning_vpc.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["<my-ip>/32"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

## 5. Security Best Practices (DevSecOps lens)
- [x] Understood and demonstrated the difference between instance-level (SG) and subnet-level (NACL) filtering
- [x] Confirmed both layers must independently allow traffic — a misconfigured NACL can silently break a correctly configured Security Group
- [x] Practiced restricting SSH to a specific IP rather than `0.0.0.0/0`, carried over from the VPC lab
- [x] Debugged a real stateless-outbound gotcha rather than just reading about it
- [ ] NACLs are rarely customized in most real setups (default-allow is common practice, real filtering done at SG level) — worth remembering NACLs are best used for coarse subnet-wide blocks (e.g. blocking a known bad IP range) rather than fine-grained rules

## 6. Errors I Hit & How I Debugged Them
| Error / Symptom | Root Cause | Fix | Notes |
|---|---|---|---|
| Website unreachable after adding NACL inbound Deny rule (Rule 50) on port 80 | Deliberate test — NACL evaluated before Security Group, and NACL rules are independent of SG rules | Removed/disabled the Deny rule | Confirms NACL can override an otherwise-correct Security Group; first real proof of the "two locks" model |
| Website unreachable again after narrowing NACL outbound to HTTP-only, despite inbound and SG both correct | NACL is stateless — response traffic uses a random ephemeral port that wasn't covered by any outbound Allow rule | Added outbound Allow rule for ephemeral port range | Classic NACL gotcha — inbound being correct means nothing if the return path isn't separately allowed |
| Added ephemeral port outbound rule but site still didn't load; `curl -v` hung at "Trying..." with no response | Port range field only captured a single port `0` instead of the full `1024-65535` range — a UI/data-entry issue, not a logic error | Re-edited the rule, explicitly set From port `1024` and To port `65535` as a real range | `curl -v` hanging (vs. an immediate "Connection refused") was the key diagnostic signal — a hang means the packet is being silently dropped somewhere stateless, not actively rejected; pointed straight at the NACL rather than the Security Group |

## 7. Cost Gotchas
- Security Groups and NACLs are both completely free — no cost implications either way, only correctness/security ones.

## 8. CLI Cheat Sheet
```bash
# List all security groups in a VPC
aws ec2 describe-security-groups --filters "Name=vpc-id,Values=<vpc-id>"

# List all NACLs in a VPC
aws ec2 describe-network-acls --filters "Name=vpc-id,Values=<vpc-id>"

# Check what a specific NACL's rules currently are
aws ec2 describe-network-acls --network-acl-ids <nacl-id> --query "NetworkAcls[0].Entries"

# Check inbound/outbound rules on a specific security group
aws ec2 describe-security-groups --group-ids <sg-id> --query "SecurityGroups[0].[IpPermissions,IpPermissionsEgress]"
```

## 9. Further Reading
- [Security Groups Docs](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html)
- [Network ACLs Docs](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html)
- [Ephemeral Ports Reference](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html#nacl-ephemeral-ports)