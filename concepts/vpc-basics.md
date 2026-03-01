# VPC Basics — Networking Foundations

*Phase 1 · Week 2 · [roadmap day-by-day](../docs/roadmap.md)*

---

## What It Is

A **Virtual Private Cloud (VPC)** is a logically isolated network you define inside an AWS region.
You control IP ranges, subnets, route tables, gateways, and security rules.

Every AWS account has a **default VPC** per region (CIDR `172.31.0.0/16`, one public subnet per AZ).
You should almost always replace this with a custom VPC for production workloads.

---

## How It Works

### CIDR & Subnets

- VPC CIDR: `/16` to `/28` (e.g. `10.0.0.0/16` → 65,536 IPs)
- Each **subnet** lives in exactly one AZ; subnets cannot span AZs.
- AWS reserves **5 IPs per subnet** (first 4 + last): network, VPC router, DNS, future, broadcast.
- **Public subnet** — has a route `0.0.0.0/0 → IGW`; resources can get public IPs.
- **Private subnet** — no direct IGW route; outbound via NAT Gateway or stays internal.

### Internet Gateway (IGW)

- Horizontally scaled, redundant, HA — no bandwidth bottleneck.
- Attach one per VPC.
- Allows two-way internet traffic for resources with public IPs.
- Required in the route table of a public subnet: `0.0.0.0/0 → igw-xxx`.

### NAT Gateway vs NAT Instance

| Feature | NAT Gateway | NAT Instance |
|---|---|---|
| Managed by | AWS | You |
| Availability | Multi-AZ (deploy one per AZ) | Single EC2 instance |
| Bandwidth | Up to 45 Gbps (auto-scales) | Limited by instance size |
| Failover | Automatic | Manual (scripts/ASG needed) |
| Cost | Hourly + data processed | EC2 + data transfer |
| Security groups | Not supported | Supported |
| Bastion host | No | Can double as one |

NAT Gateway is deployed in a **public subnet** with an Elastic IP. Private subnet routes `0.0.0.0/0 → nat-xxx`.

### Security Groups vs NACLs

| Feature | Security Group | NACL |
|---|---|---|
| Level | Instance (ENI) | Subnet |
| State | **Stateful** — return traffic auto-allowed | **Stateless** — must explicitly allow inbound AND outbound |
| Rules | Allow only (no explicit deny) | Allow AND Deny |
| Rule evaluation | All rules evaluated | Rules evaluated in order (lowest number first); first match wins |
| Default | All outbound allowed, all inbound denied | Default NACL allows all; custom NACL denies all |
| Scope | Per resource | Per subnet |

> **Exam key:** Security groups are stateful — if you allow inbound 443, the response traffic goes out automatically. NACLs are stateless — you must allow ephemeral outbound ports (1024–65535) for return traffic.

### Route Tables

- Every subnet is associated with exactly one route table.
- The **main route table** is assigned by default; create custom ones for granular control.
- Routes are evaluated **most specific first** (longest prefix match).

Example custom route table for a private subnet:
```
10.0.0.0/16   local        (VPC-internal, always present)
0.0.0.0/0     nat-xxxxxxxx (internet via NAT)
```

### VPC Peering

- Private, non-transitive connection between two VPCs (same or different account/region).
- No overlapping CIDRs allowed.
- **Non-transitive:** A↔B and B↔C does NOT mean A↔C. You must explicitly peer A↔C.
- Route tables on both sides must be updated; security groups must allow the traffic.
- No bandwidth bottleneck; uses AWS backbone.

### DNS in VPC

| Feature | Description |
|---|---|
| `enableDnsSupport` | Enables the AWS DNS resolver (169.254.169.253 or VPC+2 address) |
| `enableDnsHostnames` | Assigns public DNS hostnames to instances with public IPs |
| **Route 53 Resolver** | DNS resolution for private hosted zones; can forward to on-prem DNS |
| **Inbound endpoint** | On-prem DNS queries resolved by Route 53 |
| **Outbound endpoint** | VPC DNS queries forwarded to on-prem DNS |
| **Private Hosted Zone** | Associate with VPC; internal domain resolution (e.g. `db.internal`) |

Both `enableDnsSupport` and `enableDnsHostnames` must be `true` to use private hosted zones.

---

## Key Configs & Limits

| Item | Limit |
|---|---|
| VPCs per region (default) | 5 (soft limit; can increase) |
| Subnets per VPC | 200 |
| Route tables per VPC | 200 |
| Security groups per VPC | 2,500 |
| Rules per security group | 60 inbound + 60 outbound (default) |
| NACLs per VPC | 200 |
| NAT Gateways per AZ | 1 recommended (deploy one per AZ for HA) |
| IGW per VPC | 1 |
| Elastic IPs per region | 5 (soft limit) |
| VPC CIDR block size | /16 (65,536) to /28 (16) |
| CIDR blocks per VPC | 5 (primary + 4 secondary) |

---

## When To Use

| Scenario | Solution |
|---|---|
| Web server needs internet access | Public subnet + IGW + public IP (or EIP) |
| App/DB server — no inbound from internet | Private subnet; outbound via NAT Gateway |
| Block specific IP from a subnet | NACL Deny rule |
| Allow traffic between two security groups | SG rule referencing the source SG by ID |
| Share resources between two VPCs, same account | VPC Peering |
| Resolve internal domain names inside VPC | Private Hosted Zone + Route 53 Resolver |
| High-availability NAT | One NAT Gateway per AZ, route tables per AZ |

---

## Exam Traps

- **NAT Gateway goes in a public subnet** — common mistake is placing it in private.
- **NACL is stateless** — always add outbound rules for ephemeral ports (1024–65535) when opening inbound.
- **VPC Peering is non-transitive** — explicit peering required for every pair.
- **Default NACL allows all** — a custom NACL starts with implicit deny all; you must add allow rules.
- **Security group cannot explicitly deny** — to block traffic, use NACLs.
- **Subnets are AZ-scoped** — a subnet cannot span multiple AZs.
- **The default VPC's subnets are all public** — don't deploy sensitive workloads there.
- **Route table evaluation** — most specific route wins, not the first match.
- **DNS flags** — both `enableDnsSupport` AND `enableDnsHostnames` must be enabled for Route 53 private hosted zones to work in the VPC.
- **NAT Gateway charges per hour + per GB** — prefer one NAT GW per AZ to avoid cross-AZ data transfer costs.

---

## Related Files

| File | Purpose |
|---|---|
| `labs/vpc-build.md` | Hands-on: build custom VPC with public/private subnets |
| `diagrams/vpc-basics.md` | Visual: VPC layout, subnet tiers, NAT flow, NACL vs SG |
| `cheatsheets/vpc-basics.md` | Quick-reference for exam |
| `mock-exams/phase1.md` | IAM + VPC practice questions |
| `concepts/networking-advanced.md` | Next networking concept (Week 7) |
