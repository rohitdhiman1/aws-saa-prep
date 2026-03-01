# Cheatsheet: VPC Basics

*Fill in as you complete Week 2. Use for exam-day revision.*

---

## Key Facts

| Concept | Detail |
|---|---|
| AWS reserved IPs per subnet | 5 (first 4 + broadcast) |
| Public subnet | Has route `0.0.0.0/0 → IGW` |
| NAT Gateway location | Public subnet with Elastic IP |
| Security group | Stateful, Allow only, instance level |
| NACL | Stateless, Allow + Deny, subnet level |
| Default NACL | Allows all; custom NACL starts with deny all |
| VPC peering | Non-transitive; no overlapping CIDRs |
| DNS flags | Both `enableDnsSupport` + `enableDnsHostnames` for private hosted zones |

---

## Must-Know Distinctions

- **SG stateful** → return traffic auto-allowed
- **NACL stateless** → must allow ephemeral outbound ports (1024–65535)
- **NAT GW** → managed, HA, no SG | **NAT Instance** → self-managed, SG supported
- **IGW** → two-way internet for public IPs | **NAT GW** → outbound only for private IPs
- **VPC Peering** → non-transitive; both route tables must be updated

---

## Exam Traps (Quick List)

- NAT GW goes in public subnet (not private)
- Route table: most specific route wins (longest prefix)
- Custom NACL denies all by default
- SG cannot explicitly deny (only allow)
- Subnets are AZ-scoped, not multi-AZ

---

## Related Files

`concepts/vpc-basics.md` · `labs/vpc-build.md` · `diagrams/vpc-basics.md` · `mock-exams/phase1.md`
