# VPC (Virtual Private Cloud)

**Phase:** 01 - Networking
**Status:** ✅ Completed

---

## 1. What it is & when to use it
A VPC is a logically isolated, private slice of AWS's network that you fully control — your own software-defined data center. Nothing inside it is reachable from the internet unless you explicitly build a path to it. Every EC2 instance, RDS database, and most other AWS resources live inside a VPC, making it the literal foundation everything else in AWS networking sits on. Most real-world AWS security incidents trace back to a VPC-level misconfiguration (an accidentally public subnet, an overly permissive route, an open security group) — this is where security gets built in from day one or bolted on badly later.

## 2. Core Concepts

**The hierarchy:**

```
VPC (10.0.0.0/16) ← the whole private network, a CIDR block you own
│
└── Availability Zone (us-east-2a) ← a physical AWS data center
├── Public Subnet (10.0.1.0/24) ← route table points to Internet Gateway
└── Private Subnet (10.0.2.0/24) ← route table points to NAT Gateway (outbound only)
Internet Gateway (IGW) → attached once per VPC, the door to the public internet
Route Table → decides where traffic goes; determines public vs private
NAT Gateway → lets private-subnet resources make outbound calls without allowing inbound
Security Group → instance-level firewall, evaluated last

```

- **CIDR block** — the IP range owned by the VPC (e.g. `10.0.0.0/16` ≈ 65,000 addresses); subnets are carved out of this range.
- **Subnet** — a smaller slice of the VPC's CIDR, pinned to exactly one Availability Zone.
- **Internet Gateway (IGW)** — the literal door to the internet, attached once per VPC.
- **Route Table** — the actual mechanism that makes a subnet "public" or "private." There is no public/private flag anywhere in AWS — a subnet is public purely because its route table sends `0.0.0.0/0` traffic to an IGW. Point it at a NAT Gateway instead and it's private.
- **NAT Gateway** — lives in a public subnet, gives private-subnet resources outbound-only internet access. The internet sees the NAT Gateway's IP, never the private resource's real address, so nothing can ever initiate a connection back in. This operates at the routing layer, before security groups are even evaluated.
- **The "three things have to line up" rule** — a resource is only internet-reachable if all three are true: (1) it's in a subnet whose route table points to an IGW, (2) it has a public IP, (3) its security group allows the traffic. Most accidental exposures happen when someone doesn't realize all three lined up.

## 3. Hands-on Lab
### Console steps
1. Created VPC `learning-vpc` with CIDR `10.0.0.0/16` (chose "VPC only," not the wizard, to build every piece manually)
2. Created `learning-public-subnet` (`10.0.1.0/24`) and `learning-private-subnet` (`10.0.2.0/24`), both in `us-east-2a`
3. Created and attached Internet Gateway `learning-igw` to `learning-vpc`
4. Created route table `learning-public-rt`, added route `0.0.0.0/0 → learning-igw`, associated it with the public subnet
5. Allocated an Elastic IP, created NAT Gateway `learning-nat-gw` in the public subnet using that EIP
6. Created route table `learning-private-rt`, added route `0.0.0.0/0 → learning-nat-gw`, associated it with the private subnet
7. Launched `public-test-ec2` in the public subnet — auto-assign public IP **enabled**, security group SSH (port 22) scoped to **My IP** only (initially left at `0.0.0.0/0` by mistake in the wizard, caught and corrected before launch — see Errors table)
8. Launched `private-test-ec2` in the private subnet — auto-assign public IP **disabled**
9. Lost the original key pair (wasn't present locally) — terminated both instances, created fresh key pair `learning-vpc-key`, relaunched both with correct permissions (`chmod 400`) on the key file
10. Verified connectivity from local machine (see results below)
11. Cleanup: terminated both instances, deleted `learning-nat-gw`, released the associated Elastic IP — confirmed all three fully removed. Left `learning-vpc`, subnets, route tables, and IGW intact (free to keep, reusable for future labs)

### Connectivity proof (the actual payoff of this lab)
```bash
# SSH into the PUBLIC instance — succeeded
ssh -i learning-vpc-key.pem ec2-user@3.145.11.210
# → logged in successfully, confirmed shell access, exited cleanly

# Ping the PRIVATE instance's private IP from local machine — failed as expected
ping 10.0.2.63
# → 100% packet loss, all requests timed out
```
This is the core lesson made concrete: same VPC, same lab, but one subnet's routing allows internet reachability and the other's doesn't — proven on real hardware, not just theory.

### CLI steps
```bash
# (Not yet done via CLI — this lab was done entirely via console to focus on the concepts.
#  Revisit with Terraform in Phase 5 once IaC is covered.)
```

## 4. Infrastructure as Code
```hcl
# Reference only — not yet applied, will build properly in Phase 5 (Terraform)
resource "aws_vpc" "learning_vpc" {
  cidr_block = "10.0.0.0/16"
  tags = { Name = "learning-vpc" }
}

resource "aws_subnet" "public" {
  vpc_id            = aws_vpc.learning_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-2a"
  map_public_ip_on_launch = true
}

resource "aws_subnet" "private" {
  vpc_id            = aws_vpc.learning_vpc.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "us-east-2a"
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.learning_vpc.id
}
```

## 5. Security Best Practices (DevSecOps lens)
- [x] Public and private subnets properly separated by routing, not just naming
- [x] SSH restricted to "My IP" instead of `0.0.0.0/0` — caught and fixed a wizard default before launch
- [x] Private instance confirmed to have zero public IP and zero internet reachability
- [x] NAT Gateway and Elastic IP fully cleaned up immediately after the lab (both are billed hourly even when idle)
- [ ] Multi-AZ subnet spread for high availability — not done in this lab (single AZ used for simplicity), needed for production-grade VPCs
- [ ] Security Groups vs NACLs not yet explored in depth — next topic

## 6. Errors I Hit & How I Debugged Them
| Error / Symptom | Root Cause | Fix | Notes |
|---|---|---|---|
| Launch wizard defaulted SSH security group rule to Source: Anywhere (`0.0.0.0/0`) on port 22 | AWS console wizard default is permissive unless manually changed | Changed Source type dropdown from "Anywhere" to "My IP" before launching | Easy to miss — this is one of the most common real-world misconfigurations, worth double-checking on every instance launch |
| Original key pair not present on local machine when trying to SSH | Key pair was selected during launch but the private key file itself was never saved/downloaded to this machine | Terminated both instances, created a new key pair (`learning-vpc-key`), relaunched with it | Key pairs can't be recovered or re-downloaded after creation — always confirm the `.pem` file is saved before relying on an instance |
| (Anticipated but avoided) `Permissions 0644 for 'key.pem' are too open` | SSH refuses to use a private key with overly open file permissions | Ran `chmod 400 learning-vpc-key.pem` immediately after downloading | Standard step, worth doing automatically every time a new `.pem` is created |
| Terminated instances still showing in EC2 console list | Expected AWS behavior — terminated instances remain visible for roughly an hour before disappearing | No fix needed — confirmed all showed `Terminated` state with no public IP, no cost incurred | Don't confuse "still listed" with "still running" — check the state column, not just presence in the list |

## 7. Cost Gotchas
- VPC, subnets, route tables, and Internet Gateway are all **free** — safe to leave `learning-vpc` intact indefinitely for future labs.
- **NAT Gateway is not free** — charges hourly (~$0.045/hr in most regions) plus per-GB data processing, even sitting idle. Deleted immediately after this lab.
- **Elastic IP is free only while attached to a running resource** — an unattached EIP silently accrues charges. Released immediately after NAT Gateway deletion.
- EC2 test instances used `t3.micro` — free tier eligible, but confirm current free tier status since it can lapse per account.

## 8. CLI Cheat Sheet
```bash
# List VPCs
aws ec2 describe-vpcs

# List subnets in a specific VPC
aws ec2 describe-subnets --filters "Name=vpc-id,Values=<vpc-id>"

# List route tables
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=<vpc-id>"

# Check for any lingering NAT Gateways (cost check)
aws ec2 describe-nat-gateways --filter "Name=state,Values=available,pending"

# Check for unattached Elastic IPs (cost check)
aws ec2 describe-addresses --query "Addresses[?AssociationId==null]"
```

## 9. Further Reading
- [Amazon VPC Docs](https://docs.aws.amazon.com/vpc/)
- [VPC Route Tables Explained](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html)
- [NAT Gateway Docs](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)