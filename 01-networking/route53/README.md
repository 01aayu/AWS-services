# Route 53

**Phase:** 01 - Networking
**Status:** ✅ Completed

---

## 1. What it is & when to use it
Route 53 is AWS's DNS (Domain Name System) service — it translates human-readable domain names into IP addresses or AWS resource endpoints, and can also act as a domain registrar. It matters beyond convenience: IPs change (EC2 restarts get a new public IP unless using an Elastic IP; ALBs never expose a stable IP at all, only a DNS name), so DNS is the stable pointer that survives underlying infrastructure changes. It also enables real production resilience through health-check-driven failover routing, and is a genuine security surface — dangling DNS records and subdomain takeovers are real, exploited vulnerability classes.

## 2. Core Concepts

```
User types: myapp.com
│
▼
[Route 53] ← looks up the Hosted Zone for myapp.com
│
▼
Record found: A record → EC2 IP (or ALIAS → ALB DNS name)
│
▼
Browser connects directly to that IP

```


- **Hosted Zone** — a container for all DNS records belonging to one domain. Public (internet-facing) or Private (only resolves inside a VPC). Creating one does **not** make a domain live on the internet by itself — it's authoritative only once a domain registrar is pointed at it.
- **Record types**: **A record** maps a name to an IPv4 address directly; **CNAME** maps a name to another name (not an IP); **ALIAS** (AWS-specific, free) works like CNAME but for AWS resources like ALBs/S3/CloudFront — this is what's used to point a domain at an ALB, since ALBs never have a stable IP.
- **NS (Name Server) records** — every hosted zone auto-generates 4 unique AWS nameserver addresses. A domain only becomes globally resolvable once its registrar's nameserver settings are updated to point at these 4 — that's the actual link between "AWS knows about this domain" and "the internet knows about this domain."
- **Routing policies** (for interviews/awareness): Simple, Weighted (traffic-split canary deploys), Failover (active/passive with health checks), Latency-based (nearest region).
- **DNS resolvers** — a `dig`/`nslookup` query doesn't talk to AWS or the domain directly; it goes to a configured resolver (often the ISP's) which does the actual multi-hop lookup (root → TLD → domain nameservers) and returns the result. The "SERVER" field in `dig` output is just that resolver, not the domain's own infrastructure.
- **Security implication — subdomain takeover**: if a domain's NS records point at AWS and a hosted-zone-referenced resource is later deleted without removing the corresponding DNS record, an attacker can sometimes claim that same resource name and hijack traffic intended for the original domain.

## 3. Hands-on Lab
### Console steps
1. Created a public Hosted Zone `learning-aayush.com` (not an owned/registered domain — created specifically to explore the mechanics without spending money)
2. Reviewed the auto-generated **NS** and **SOA** records
3. Created a custom **A record** pointing the zone's root at an existing EC2 instance's public IP (`web-instance-1`, `52.14.99.184`)
4. Noted the 4 auto-generated NS values: `ns-1479.awsdns-56.org`, `ns-790.awsdns-34.net`, `ns-503.awsdns-62.com`, `ns-1709.awsdns-21.co.uk`
5. Deleted the hosted zone after the lab (avoids the ongoing $0.50/month charge)

### CLI / Terminal steps
```bash
# Query the hosted zone's NS records directly from the terminal
dig NS learning-aayush.com
# → returned NXDOMAIN: expected, since the domain was never registered with a
#   real registrar, so the global DNS system has no path to AWS's nameservers

# Force the query through a specific public resolver (e.g. Google) instead of the default
dig NS learning-aayush.com @8.8.8.8
```

## 4. Infrastructure as Code
```hcl
# Reference only — not yet applied
resource "aws_route53_zone" "learning_zone" {
  name = "learning-aayush.com"
}

resource "aws_route53_record" "root_a" {
  zone_id = aws_route53_zone.learning_zone.zone_id
  name    = "learning-aayush.com"
  type    = "A"
  ttl     = 300
  records = ["52.14.99.184"]
}
```

## 5. Security Best Practices (DevSecOps lens)
- [x] Understood the registrar → NS record → Hosted Zone chain that makes a domain actually resolvable
- [x] Understood the subdomain takeover risk pattern (dangling DNS record + deleted underlying resource)
- [ ] Never left a real registered domain's NS records pointed at an AWS hosted zone that was later deleted without cleanup (no real domain used in this lab, so not directly applicable yet — worth remembering for future real deployments)
- [ ] Health checks and failover routing not yet explored hands-on (would need multiple live endpoints to test properly)

## 6. Errors I Hit & How I Debugged Them
| Error / Symptom | Root Cause | Fix | Notes |
|---|---|---|---|
| `dig NS learning-aayush.com` returned `NXDOMAIN` | Domain was never registered with a real registrar — AWS's Hosted Zone existing internally doesn't make a domain globally resolvable | Not a bug — expected result; understood that a registrar's nameserver settings must be pointed at AWS's 4 NS values to close the gap | Good hands-on proof that "creating a Hosted Zone" and "domain being live on the internet" are two separate, disconnected steps |
| Confusion over the `SERVER: 103.17.159.50` field in `dig` output | Assumed it was AWS or the domain's own server | Clarified it's just the local DNS resolver (likely ISP-provided) that handled the lookup request, not related to AWS at all | Re-ran with `@8.8.8.8` to see the same NXDOMAIN result through a different, known resolver, confirming the field just reflects whichever resolver was queried |

## 7. Cost Gotchas
- A Hosted Zone costs **$0.50/month** per zone if left running — deleted immediately after this lab since no real domain was involved.
- Actually registering a domain through Route 53 (or any registrar) has separate annual costs — not required to learn the DNS concepts, as this lab demonstrated.
- DNS queries themselves are billed per million queries but are effectively free at this scale.

## 8. CLI Cheat Sheet
```bash
# List hosted zones
aws route53 list-hosted-zones

# List records in a specific zone
aws route53 list-resource-record-sets --hosted-zone-id <zone-id>

# Query NS records for a domain
dig NS <domain>

# Query through a specific resolver
dig NS <domain> @8.8.8.8

# Reverse lookup an IP
nslookup <ip-address>
```

## 9. Further Reading
- [Amazon Route 53 Docs](https://docs.aws.amazon.com/route53/)
- [Route 53 Routing Policies](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy.html)
- [Subdomain Takeover Overview](https://owasp.org/www-community/attacks/Subdomain_takeover)