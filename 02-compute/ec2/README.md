# EC2 (Elastic Compute Cloud)

**Phase:** 02 - Compute
**Status:** ✅ Completed

---

## 1. What it is & when to use it
EC2 ("Elastic Compute Cloud") is AWS's rentable virtual machine service — instead of buying physical servers, you rent compute capacity by the second/hour, choosing OS and hardware specs, while AWS manages the underlying physical infrastructure. "Elastic" refers to the ability to resize, add, or remove capacity on demand. EC2 has been used throughout earlier labs already, but this module covers it properly as its own topic: AMIs, instance types, EBS, lifecycle states, and — critically for DevSecOps — building a repeatable "golden image" pipeline.

## 2. Core Concepts

- **AMI (Amazon Machine Image)** — the template EC2 boots from: OS + pre-installed software + configuration, frozen at a point in time. A **custom AMI** built from a hardened, pre-configured instance is a core DevSecOps pattern — bake security patches, monitoring agents, and required software into a "golden image" once, then launch identical, pre-secured instances from it repeatedly instead of manually configuring each one.
- **Instance type naming** — e.g. `t3.micro` = `t`-family (burstable, cheap, general purpose) + generation `3` + `.micro` size. Other families: `m` (balanced), `c` (compute-optimized), `r` (memory-optimized).
- **EBS (Elastic Block Store)** — the disk attached to an instance, architecturally separate from the instance itself. Can persist after instance termination (if not set to delete-on-termination), can be snapshotted for backup, and can even be attached to a different instance later. This decoupling of compute and storage is a core AWS architectural idea.
- **Instance lifecycle**: `pending → running → stopping → stopped → terminated`. **Stopped** still incurs EBS storage charges; **terminated** is the only state that fully stops all charges and is irreversible.
- **User data** — a script that runs once, only on first boot, as root. Good for one-time bootstrapping, not for anything needing to run repeatedly.
- **AMI capture is a point-in-time EBS snapshot** — it captures whatever is on disk at the moment "Create Image" is triggered. If a service was only just installed/started moments before capture, there can be a real consistency gap between "service confirmed running via curl" and "state fully persisted to the snapshot" — see Errors table for a concrete instance of this.

```

Base instance (configured, hardened, tested)
│
▼
Create Image → Custom AMI → EBS Snapshot
│
▼
Launch N identical instances from this AMI
│
▼
Each new instance boots pre-configured — zero manual setup, zero user data needed

```


## 3. Hands-on Lab

### Detour: an unrelated but important VPC-level lesson
Before this lab could even start, a fresh EC2 launch in the old `learning-vpc` (built manually in the VPC module) hit a persistent, unresolved connectivity timeout — both HTTP and SSH — despite every individual setting (security group, NACL, route table, IGW) checking out correct in isolation. Ruled out local network entirely via a hotspot test (still failed). Rather than continuing to debug an unknown root cause in a heavily manually-edited VPC, rebuilt a fresh VPC (`test-vpc-2`) using the **VPC wizard** ("VPC and more") instead of manual configuration — connectivity worked immediately, first try, with otherwise identical instance/SG setup. This strongly suggests `learning-vpc` had some accumulated misconfiguration from repeated manual edits across multiple sessions that was never precisely identified. **Lesson: when manually-built infrastructure develops an unexplained persistent issue, rebuilding via a known-good path (the wizard) is a legitimate and often faster resolution than continuing to debug blind — the wizard is also useful as a "known-good baseline" to diff against.**

### Console + CLI steps
1. Created `test-vpc-2` via the VPC wizard (2 AZs, 2 public subnets, no NAT, no endpoints) as a clean baseline
2. Launched `clean-test` in `test-vpc-2` with a fresh single-purpose security group (`clean-test-sg`: HTTP 80 from anywhere, SSH 22 from My IP) — confirmed SSH and HTTP both reachable immediately, "Connection refused" (not timeout) confirmed the network path is healthy before a web server was even installed
3. SSH'd in, installed and started Apache manually:
```bash
sudo yum update -y
sudo yum install -y httpd
sudo systemctl enable httpd
sudo systemctl start httpd
echo "<h1>Golden Image Base</h1>" | sudo tee /var/www/html/index.html
```
4. Verified locally (`curl http://localhost`) and externally (`curl -v http://<ip>`) — both returned the HTML correctly
5. Created a custom AMI (`learning-golden-ami`) from `clean-test` via **Actions → Image and templates → Create image**
6. Launched a new instance (`golden-image-clone`) from this AMI, in the same VPC/subnet/SG, deliberately with **zero user data**
7. Test failed — `curl` returned "Connection refused" (network fine) but SSH into the clone showed `systemctl status httpd` → "Unit httpd.service could not be found" — Apache wasn't captured in the AMI at all
8. Diagnosed: confirmed the AMI's source Instance ID matched `clean-test` exactly, and confirmed `clean-test` itself still had httpd running solidly — ruled out both a wrong-instance mix-up and an actual instance problem, isolating the issue to the AMI capture step itself (likely a timing/consistency gap between service start and snapshot capture)
9. Deregistered the broken AMI and its orphaned snapshot, recreated a fresh image (`learning-golden-ami-v2`) from the same, now long-stable `clean-test` instance
10. Launched a new clone (`golden-image-clone-v2`) from the v2 AMI, again with zero user data — `curl` immediately returned `<h1>Golden Image Base</h1>`, confirming the golden image pattern works correctly once given a stable source state
11. Cleaned up: terminated all test instances, deregistered both AMIs and their associated snapshots

## 4. Infrastructure as Code
```hcl
# Reference only — not yet applied
resource "aws_instance" "golden_base" {
  ami           = "ami-xxxxxxxx"
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.public.id
  key_name      = "learning-vpc-key"
}

# In practice, custom AMI creation is typically automated via tools like
# HashiCorp Packer rather than manual console clicks — worth exploring later.
```

## 5. Security Best Practices (DevSecOps lens)
- [x] Practiced the golden-image pattern — the real-world mechanism for shipping hardened, consistent server configurations at scale instead of manual per-instance setup
- [x] Confirmed AMI capture is not instantaneous or guaranteed-consistent — validate a freshly created AMI before relying on it in a pipeline (e.g. launch a test instance from it and verify, rather than assuming success)
- [x] Deregistered unused AMIs and deleted their snapshots promptly — unused AMIs are a real, easy-to-forget storage cost
- [ ] Not yet explored: automated AMI pipelines (Packer, EC2 Image Builder) — manual console creation was used here for learning purposes only
- [ ] Golden image should also include OS-level hardening (CIS benchmarks, disabling unused services) in a production context — this lab focused on the mechanism, not a hardened baseline

## 6. Errors I Hit & How I Debugged Them
| Error / Symptom | Root Cause | Fix | Notes |
|---|---|---|---|
| Fresh EC2 in `learning-vpc` timed out on both SSH and HTTP, despite all individual network settings checking out correct | Suspected accumulated misconfiguration in `learning-vpc` from repeated manual edits across sessions — exact setting never pinpointed | Rebuilt a fresh VPC via the wizard (`test-vpc-2`) instead of continuing to debug manually; connectivity worked immediately | When manually-built infra develops an unexplained persistent fault, rebuilding via a known-good path can be faster and more productive than continued blind debugging |
| Clone launched from `learning-golden-ami` had no httpd installed at all (`Unit httpd.service could not be found`), despite curl showing it working on the source instance minutes earlier | AMI/EBS snapshot captured before the httpd installation had fully and consistently persisted to disk — a timing/consistency gap between "service confirmed running" and "image capture," not a configuration mistake | Deregistered the broken AMI and its snapshot, waited for the source instance's state to be long-stable, recreated the AMI fresh — worked correctly on the second attempt | Real lesson: don't trust an AMI blindly right after creation — launch and verify a test instance from any new golden image before relying on it, especially in an automated pipeline |

## 7. Cost Gotchas
- Stopped (not terminated) instances still incur EBS volume storage charges — always terminate test instances when fully done, not just stop them.
- Custom AMIs are backed by EBS snapshots and **cost money to store indefinitely** even if no instance ever launches from them again — deregister the AMI and separately delete its snapshot when done (deregistering the AMI alone does not delete the underlying snapshot).
- `t3.micro` instances used throughout — confirm current free-tier eligibility for your account, as it can vary/lapse.

## 8. CLI Cheat Sheet
```bash
# List your AMIs
aws ec2 describe-images --owners self

# Deregister an AMI
aws ec2 deregister-image --image-id <ami-id>

# List snapshots (to find and clean up orphaned ones after deregistering an AMI)
aws ec2 describe-snapshots --owner-ids self

# Delete a snapshot
aws ec2 delete-snapshot --snapshot-id <snapshot-id>

# Check instance status
aws ec2 describe-instance-status --instance-ids <instance-id>
```

## 9. Further Reading
- [EC2 AMI Docs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html)
- [EC2 Instance Types](https://aws.amazon.com/ec2/instance-types/)
- [EBS Docs](https://docs.aws.amazon.com/ebs/)
- [HashiCorp Packer (automated AMI building, for later)](https://developer.hashicorp.com/packer)

