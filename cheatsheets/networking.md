# Cheatsheet: Networking (Advanced + Edge/DNS)

*Fill in as you complete Weeks 7–8. Use for exam-day revision.*

---

## Key Facts

| Concept | Detail |
|---|---|
| VPN bandwidth per tunnel | ~1.25 Gbps |
| DX dedicated speeds | 1, 10, 100 Gbps |
| DX encryption | NOT encrypted by default |
| DX SLA (redundant) | 99.99% |
| TGW max attachments | 5,000 |
| VPC Peering max per VPC | 125 |
| Gateway endpoint cost | Free |
| Interface endpoint cost | Per-hour + per-GB |
| Route 53 Alias | Free queries; works at zone apex |
| CloudFront OAC | Restricts S3 to CloudFront only |

---

## Must-Know Distinctions

- **VPC Peering** → point-to-point, non-transitive | **TGW** → hub-and-spoke, transitive
- **Gateway endpoint** → S3/DynamoDB, route table, free | **Interface endpoint** → all services, ENI, SG, DNS
- **Site-to-Site VPN** → encrypted, internet | **DX** → private, not encrypted, dedicated
- **DX Gateway** → connects DX to VGWs/TGW | NOT a router between VPCs
- **CloudFront** → caches content | **Global Accelerator** → no cache, static IPs, TCP/UDP
- **Route 53 Failover** → active-passive | **Latency** → lowest latency | **Weighted** → A/B split
- **Geolocation** → exact country/continent | **Geoproximity** → distance + bias dial

---

## Exam Traps (Quick List)

- VPN is encrypted; DX is NOT
- Transit VIF requires TGW (not VGW)
- PrivateLink: no VPC peering needed; CIDRs can overlap
- CloudFront signed URL = CloudFront restriction; S3 pre-signed URL = direct S3 access
- Route 53 Multi-Value ≠ load balancer (use ELB for real load balancing)

---

## Related Files

`concepts/networking-advanced.md` · `concepts/edge-dns.md` · `labs/vpc-peering.md` · `diagrams/networking.md` · `mock-exams/phase4.md`
