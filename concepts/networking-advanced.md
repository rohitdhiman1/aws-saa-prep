# Networking — Advanced VPC & Hybrid Connectivity

*Phase 4 · Week 7 · [roadmap day-by-day](../docs/roadmap.md)*

---

## What It Is

Advanced networking covers how VPCs connect to each other, to on-premises environments, and how AWS services are accessed privately — without traversing the public internet.

---

## How It Works

### VPC Endpoints

Keep traffic between VPC and AWS services on the AWS network (no IGW, NAT Gateway, or public IP needed).

| Type | Backed by | Supports | Cost |
|---|---|---|---|
| **Gateway endpoint** | Route table entry | S3, DynamoDB | Free |
| **Interface endpoint (PrivateLink)** | ENI in your subnet | 100s of AWS services + custom SaaS | Per-hour + per-GB |

- **Gateway endpoint** — add a route in the route table; traffic stays on AWS backbone. No ENI; no security group.
- **Interface endpoint** — deploys an ENI with a private IP in your subnet. Access the service via a private DNS name (e.g., `ec2.us-east-1.amazonaws.com` resolves to private IP in VPC). Supports security groups.

**Endpoint policies** — IAM resource policy attached to the endpoint to restrict which actions/resources are accessible through it.

---

### VPC Peering vs Transit Gateway

| Feature | VPC Peering | Transit Gateway |
|---|---|---|
| Topology | Point-to-point (one-to-one) | Hub-and-spoke (many-to-many) |
| Transitive routing | Not supported | Supported |
| Cross-account | Yes | Yes |
| Cross-region | Yes | Yes (inter-region peering) |
| Bandwidth | No limit (uses AWS backbone) | Up to 50 Gbps per AZ attachment |
| Complexity | Simple (few VPCs) | Better for 5+ VPCs |
| Cost | Data transfer only | Per attachment + per GB |
| Overlapping CIDRs | Not allowed | Not allowed |

- **Transit Gateway (TGW)** — regional router. Attachments: VPCs, VPNs, Direct Connect Gateways, peering with other TGWs.
- **TGW route table** — separate from VPC route tables; controls routing between attachments. Can have multiple route tables for segmentation.
- **TGW multicast** — distributes one packet to multiple recipients; use for media/streaming.

---

### Site-to-Site VPN

Encrypted IPSec tunnel between your on-premises network and AWS.

```
On-premises ─── Customer Gateway (CGW) ─── Internet ─── Virtual Private Gateway (VGW) ─── VPC
```

- **Customer Gateway (CGW)** — represents your on-prem device in AWS (IP address + optional BGP ASN).
- **Virtual Private Gateway (VGW)** — AWS-side VPN concentrator; attached to a VPC.
- Two tunnels per VPN connection (high availability; different AZs on AWS side).
- Supports **static routing** or **BGP (dynamic routing)**.
- Max bandwidth: ~1.25 Gbps per tunnel.
- **Accelerated VPN** — uses AWS Global Accelerator for improved performance over the public internet.
- If using Transit Gateway: attach VPN to TGW instead of VGW for centralised management.

---

### Direct Connect (DX)

Private, dedicated network connection between your on-premises and AWS.

| Type | Speeds | Setup time |
|---|---|---|
| **Dedicated connection** | 1, 10, or 100 Gbps | Weeks (physical cross-connect) |
| **Hosted connection** (via partner) | 50 Mbps – 10 Gbps | Days to weeks |

**Virtual Interfaces (VIFs):**

| VIF type | Connects to | Traffic |
|---|---|---|
| **Private VIF** | VGW or TGW | VPC resources (private IPs) |
| **Public VIF** | AWS public endpoints | S3, DynamoDB, all public services |
| **Transit VIF** | Transit Gateway | Multiple VPCs via TGW |

- DX is NOT encrypted by default — run IPSec VPN over it for encryption, or use MACsec (layer 2 encryption for dedicated connections).
- **DX Gateway** — associate one DX connection with up to 10 VGWs across regions/accounts. Use with Private VIF.
- **DX + Transit VIF** — associate DX Gateway with a TGW to reach many VPCs from one DX connection.
- **Redundancy:** recommend two DX connections to separate locations; add VPN as backup.
- **DX SLA:** 99.9% (single DX), 99.99% (redundant DX). VPN over internet = best effort.

---

### AWS PrivateLink

Expose a service (your own or SaaS provider's) to other VPCs **without VPC peering or public internet**.

```
Consumer VPC ─── Interface Endpoint ─── PrivateLink ─── NLB ─── Provider VPC (service)
```

- **Provider** deploys a service behind an **NLB**.
- **Consumer** creates an **interface endpoint** pointing to the provider's endpoint service.
- Traffic stays on AWS network; no VPC peering; no overlapping CIDR concerns.
- Used by: AWS service interface endpoints, SaaS vendors (e.g. Datadog, Snowflake), inter-account service sharing.
- PrivateLink endpoints support **endpoint policies** for access control.

---

## Key Configs & Limits

| Item | Limit |
|---|---|
| VPC peerings per VPC | 125 |
| TGW attachments per TGW | 5,000 |
| TGW route tables | 20 |
| VPN tunnels per VGW | Up to 10 VPN connections |
| VPN bandwidth per tunnel | ~1.25 Gbps |
| DX connections per account | No hard limit |
| DX Gateway VGW associations | 10 per DX Gateway |
| Interface endpoints per VPC | 50 (default) |
| BGP ASN range (private) | 64,512 – 65,534 |

---

## When To Use

| Scenario | Solution |
|---|---|
| 2–3 VPCs need to communicate | VPC Peering |
| 10+ VPCs, centralised routing | Transit Gateway |
| VPC → S3/DynamoDB, no NAT cost | Gateway endpoint |
| VPC → any AWS service privately | Interface endpoint (PrivateLink) |
| Encrypt traffic over DX | VPN over DX or MACsec |
| Low-cost on-prem connectivity | Site-to-Site VPN |
| High-throughput, low-latency on-prem | Direct Connect (dedicated) |
| DX + access to multiple VPCs | DX Gateway + Private VIF or Transit VIF |
| Expose internal service to other accounts | PrivateLink (NLB + interface endpoint) |
| DX as primary, VPN as backup | DX + VPN failover (same TGW or VGW) |

---

## Exam Traps

- **VPN is encrypted; DX is not** — DX is private but not encrypted by default.
- **Gateway endpoint = free, no ENI** — only for S3 and DynamoDB; route table change only.
- **Interface endpoint = private DNS** — resolves service DNS to private IP inside VPC; requires `enableDnsSupport` in VPC.
- **DX Gateway ≠ Transit Gateway** — DX Gateway connects DX to VGWs/TGW; it doesn't do routing between VPCs.
- **Transit VIF requires TGW** — you cannot use Transit VIF with a VGW.
- **VPC Peering is non-transitive** — repeat from basics, but tested in advanced context too.
- **Accelerated VPN uses Global Accelerator** — better for geographically distant VPNs; not the same as DX.
- **PrivateLink consumer does not need VPC peering** — this is the whole point; CIDRs can overlap.
- **Two VPN tunnels** — always two tunnels per connection for HA; both are active (ECMP) or one active/standby depending on CGW.
- **DX 99.99% SLA = two DX connections to different locations** — single DX only gets 99.9%.

---

## Related Files

| File | Purpose |
|---|---|
| `labs/vpc-peering.md` | Hands-on: peer two VPCs, update route tables |
| `diagrams/networking.md` | Visual: TGW hub-and-spoke, DX topology, PrivateLink flow |
| `cheatsheets/networking.md` | Quick-reference card |
| `mock-exams/phase4.md` | Networking practice questions |
| `concepts/edge-dns.md` | Next concept file (Week 8) |
