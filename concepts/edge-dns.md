# Edge, DNS & Content Delivery — Route 53, CloudFront, Global Accelerator

*Phase 4 · Week 8 · [roadmap day-by-day](../docs/roadmap.md)*

---

## What It Is

Services that sit at or near the internet edge: **Route 53** (DNS and health-checking), **CloudFront** (CDN), and **Global Accelerator** (network layer acceleration).

---

## How It Works

### Route 53

AWS's managed, globally distributed DNS service. Also does health checking and traffic routing.

#### Record Types

| Record | Purpose |
|---|---|
| **A** | Domain → IPv4 address |
| **AAAA** | Domain → IPv6 address |
| **CNAME** | Domain → another domain (cannot be zone apex) |
| **Alias** | AWS-specific; points to AWS resource (ELB, CloudFront, S3, etc.); works at zone apex; free queries |
| **MX** | Mail exchange |
| **TXT** | Text records (SPF, domain verification) |
| **NS** | Name server records (zone delegation) |
| **SRV** | Service records |
| **PTR** | Reverse DNS (IP → domain) |

> **Alias vs CNAME:** Use Alias for AWS resources (ELB, CloudFront, S3 website endpoint, API Gateway). Alias can be at the zone apex (e.g., `example.com`); CNAME cannot. Alias queries are free.

#### Routing Policies

| Policy | How it works | Use case |
|---|---|---|
| **Simple** | Returns one or more IPs; no health check | Single resource |
| **Weighted** | Split traffic by weight (e.g., 80/20) | A/B testing, gradual migration |
| **Latency** | Routes to region with lowest latency to the client | Multi-region active-active |
| **Failover** | Primary → secondary if primary fails health check | Active-passive DR |
| **Geolocation** | Route based on user's country/continent | Geo-restricted content, localisation |
| **Geoproximity** | Route based on geographic proximity; bias adjustable | Fine-grained traffic shifting by location |
| **Multi-Value** | Returns up to 8 healthy records randomly | Simple load balancing with health checks |
| **IP-based** | Route based on client IP CIDR | ISP-based routing |

- **Health checks** — monitor endpoints (HTTP/HTTPS/TCP), other health checks (calculated), or CloudWatch alarms.
- **Failover policy** requires an active health check on the primary record.
- **Private hosted zones** — DNS resolution within VPC. Associate with VPC; need `enableDnsSupport` and `enableDnsHostnames`.
- **Route 53 Resolver** — default DNS in VPC (VPC+2 address). Use **Resolver Rules** and **endpoints** for hybrid DNS.
  - **Inbound endpoint** — on-prem resolves AWS private domains via this endpoint.
  - **Outbound endpoint** — Route 53 forwards queries matching rules to on-prem DNS.

---

### CloudFront

Global CDN with 400+ edge locations (Points of Presence). Caches content at the edge; reduces latency and origin load.

#### Core Concepts

- **Distribution** — a CloudFront configuration. Has a unique domain (`xxx.cloudfront.net`).
- **Origin** — where CloudFront fetches content from. Can be: S3 bucket, ALB, EC2, HTTP server, or any HTTP endpoint.
- **Cache behaviour** — rules applied per URL path pattern. Each behaviour specifies: allowed methods, cache policy, origin, viewer protocol policy.
- **Edge location** — caches content. Hundreds globally.
- **Regional edge cache** — sits between edge locations and origin; larger cache for less popular content.

#### Access Control

- **OAC (Origin Access Control)** — recommended way to restrict S3 origin to CloudFront only. Replaces OAI.
- **OAI (Origin Access Identity)** — legacy; still works but OAC preferred.
- **Signed URLs / Signed Cookies** — restrict access to content to authenticated users. Signed URL = single file; Signed Cookie = multiple files.
- **Geo-restriction** — allowlist or blocklist by country.

#### Cache Behaviour & Invalidation

- **Cache-Control headers** from origin control TTL. CloudFront also has min/max/default TTL settings per behaviour.
- **Cache invalidation** — `/*` invalidates all; charged per invalidation path (first 1,000 paths/month free).
- **Cache key** — by default: URL path only. Can include headers, query strings, cookies in the cache key (add specificity, reduces hit rate).
- **Cache policy** — defines what's in the cache key and TTLs.
- **Origin request policy** — controls what CloudFront forwards to origin (headers, cookies, query strings) without affecting cache key.

#### Lambda@Edge vs CloudFront Functions

| Feature | CloudFront Functions | Lambda@Edge |
|---|---|---|
| Runtime | JavaScript only | Node.js, Python |
| Max duration | 1 ms | 5 s (viewer), 30 s (origin) |
| Triggers | Viewer request, viewer response | All 4 (+ origin request/response) |
| Memory | 2 MB | 128 MB – 10,240 MB |
| Cost | Cheaper | More expensive |
| Use case | Simple redirects, URL rewrites, header manipulation | Complex auth, A/B testing, dynamic content |

---

### Global Accelerator

Routes traffic through the AWS global network to improve performance and availability for **non-HTTP** workloads (or HTTP with static IP requirements).

- Provides **two static Anycast IP addresses** — consistent IPs regardless of backend changes.
- Traffic enters AWS at the nearest edge location → travels over AWS backbone to the region.
- Supports: EC2, ALB, NLB, Elastic IPs as endpoints.
- **Health checks** — routes around unhealthy endpoints within 30 seconds.
- **Traffic dials** — shift traffic % per endpoint group (useful for gradual migration).

#### CloudFront vs Global Accelerator

| Feature | CloudFront | Global Accelerator |
|---|---|---|
| Caching | Yes | No |
| Protocols | HTTP/HTTPS | HTTP, TCP, UDP, any |
| Static IPs | No (dynamic DNS) | Yes (2 static Anycast IPs) |
| Use case | Static assets, API caching, websites | Gaming, IoT, VoIP, latency-sensitive apps needing static IPs |
| Price | Per request + transfer | Per accelerator + data transfer |

---

### WAF Integration

- **AWS WAF** — web application firewall. Attach to: CloudFront, ALB, API Gateway, AppSync, Cognito.
- Rules: IP match, geo match, rate limiting, SQL injection, XSS, managed rule groups.
- **Web ACL** — collection of rules applied to a distribution.

---

## Key Configs & Limits

| Item | Limit |
|---|---|
| Route 53 hosted zones per account | 500 (default) |
| Records per hosted zone | 10,000 (default) |
| Health checks per account | 200 (default) |
| CloudFront distributions | 200 per account |
| CloudFront origins per distribution | 25 |
| CloudFront cache behaviours | 25 per distribution |
| CloudFront Functions memory | 2 MB |
| Lambda@Edge max duration (origin) | 30 seconds |
| Global Accelerator static IPs | 2 per accelerator |
| WAF Web ACL rules | 1,500 WCU per Web ACL |

---

## When To Use

| Scenario | Solution |
|---|---|
| Active-passive failover between regions | Route 53 Failover policy + health check |
| Distribute traffic 10%/90% during migration | Route 53 Weighted |
| Lowest latency for multi-region users | Route 53 Latency |
| Restrict content by user country | Route 53 Geolocation or CloudFront Geo-restriction |
| Cache static assets globally | CloudFront |
| Serve S3 static site over HTTPS | S3 + CloudFront (OAC) |
| Authenticate before serving content | CloudFront + Lambda@Edge |
| Simple header manipulation at edge | CloudFront Functions |
| Static Anycast IPs for TCP/UDP apps | Global Accelerator |
| DDoS protection at L3/L4 | AWS Shield Standard (free) / Advanced |

---

## Exam Traps

- **Alias vs CNAME** — Alias works at zone apex; CNAME does not. Alias queries to AWS resources are free.
- **Geolocation ≠ Geoproximity** — Geolocation is exact (country/continent); Geoproximity uses distance with a bias dial.
- **Multi-Value ≠ load balancer** — it returns multiple healthy IPs but DNS has no knowledge of load; use ELB for proper load balancing.
- **CloudFront caches; Global Accelerator does not** — GA is about routing, not caching.
- **OAC replaces OAI** — OAI is legacy; new distributions should use OAC for S3.
- **CloudFront signed URL vs S3 pre-signed URL** — CloudFront signed URL restricts access via CloudFront; S3 pre-signed URL goes directly to S3.
- **Lambda@Edge runs in multiple regions** — deployed globally; cannot use all Lambda features (no VPC, no layers on some triggers).
- **CloudFront does NOT support RTMP** — real-time streaming; use other solutions.
- **Route 53 health checks for failover** — must be attached to the primary record; Route 53 won't failover without it.
- **Private hosted zone + VPC** — requires both `enableDnsSupport` and `enableDnsHostnames` = true.

---

## Related Files

| File | Purpose |
|---|---|
| `diagrams/networking.md` | Visual: CloudFront → S3 (OAC), Route 53 routing policies |
| `cheatsheets/networking.md` | Quick-reference card |
| `mock-exams/phase4.md` | Networking practice questions |
| `concepts/messaging.md` | Next concept file (Week 9) |
