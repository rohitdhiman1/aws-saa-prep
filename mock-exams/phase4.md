# Practice Questions: Phase 4 — Networking Deep Dive

*20 questions · Target score: ≥ 75% · Complete end of Week 8*

---

## Instructions

Answer each question before reading the answer. Write your choice, then compare.
Log wrong answers in `docs/bugbase.md` with the correct reasoning.

> **Draw first:** Before scoring, sketch a hybrid multi-VPC architecture from memory (VPC peering / TGW / Direct Connect / VPN). Check against `diagrams/networking.md`.

---

**Q1.**
A company has three VPCs: VPC-A, VPC-B, and VPC-C. VPC-A is peered with VPC-B, and VPC-B is peered with VPC-C. An instance in VPC-A tries to communicate with an instance in VPC-C. What happens?

A) Traffic flows through VPC-B as a transit hub
B) The communication fails — VPC Peering is non-transitive
C) Traffic is automatically routed through the peering connections
D) VPC-A needs a peering connection to VPC-C via the VPC-B gateway

**Answer: B** — VPC Peering is non-transitive. Even though A↔B and B↔C are peered, there is no path from A to C through B. Each pair of VPCs that needs to communicate must have a direct peering connection (A↔C). For hub-and-spoke transit routing, use AWS Transit Gateway instead.

---

**Q2.**
A company has 20 VPCs across multiple regions and multiple on-premises data centers connected via Direct Connect. Managing individual VPC peering connections is becoming complex. Which service simplifies this architecture?

A) VPC Peering with a central hub VPC
B) AWS Transit Gateway
C) AWS PrivateLink
D) AWS Direct Connect Gateway

**Answer: B** — Transit Gateway (TGW) is a managed regional transit hub. VPCs and on-premises networks attach to the TGW; routing is managed centrally. You don't need a full mesh of peering connections. PrivateLink (C) is for exposing specific services privately, not general VPC connectivity. Direct Connect Gateway (D) allows a single Direct Connect to reach multiple VPCs but doesn't solve VPC-to-VPC routing.

---

**Q3.**
A company wants private EC2 instances to access S3 without going through the internet or a NAT Gateway. What is the most cost-effective solution?

A) VPC Interface Endpoint for S3
B) VPC Gateway Endpoint for S3
C) NAT Gateway in a public subnet
D) Direct Connect to S3

**Answer: B** — Gateway endpoints for S3 are free. They work by injecting a route into your route table that directs S3-bound traffic to the endpoint, keeping it within the AWS network. Interface endpoints (A) for S3 also work but cost money per AZ per hour plus data. NAT Gateway (C) works but routes through the internet and costs money for data processing. Direct Connect (D) is for on-premises connectivity.

---

**Q4.**
A company uses Route 53 to route traffic to their application. They want 10% of traffic to go to a new version of the application for canary testing. Which Route 53 routing policy achieves this?

A) Latency-based routing
B) Failover routing
C) Weighted routing
D) Geolocation routing

**Answer: C** — Weighted routing lets you assign numeric weights to records. Setting 90 on the old version and 10 on the new version routes ~10% of traffic to the canary. Latency (A) routes to the lowest-latency endpoint. Failover (B) routes to a secondary only when the primary is unhealthy. Geolocation (D) routes based on the user's location.

---

**Q5.**
A company has an on-premises data center connected to AWS via a Site-to-Site VPN. The VPN is over the public internet. They need a connection with consistent bandwidth and low latency for a critical database workload. What should they provision instead?

A) Another Site-to-Site VPN for redundancy
B) AWS Direct Connect
C) AWS Global Accelerator
D) CloudFront with the on-premises origin

**Answer: B** — Direct Connect provides a dedicated private network connection between on-premises and AWS with consistent bandwidth and low, predictable latency. VPN (A) runs over the public internet — bandwidth and latency vary. Global Accelerator (C) improves internet routing to AWS but still uses the internet. CloudFront (D) is a CDN for content delivery, not a private connectivity solution.

---

**Q6.**
A company's CloudFront distribution serves an S3 bucket. They want to ensure that users can only access the S3 content through CloudFront and not directly via the S3 URL. Which configuration achieves this?

A) Enable S3 Transfer Acceleration
B) Configure an Origin Access Control (OAC) on the CloudFront distribution and update the S3 bucket policy to only allow the OAC principal
C) Enable S3 server-side encryption
D) Set the S3 bucket to public and restrict via CloudFront WAF rules

**Answer: B** — OAC (the successor to OAI) restricts S3 access to only the CloudFront distribution. The S3 bucket policy denies all other access. Users who hit the S3 URL directly get `Access Denied`. Encryption (C) protects data at rest but doesn't restrict access. Making the bucket public and using WAF (D) is backwards — the S3 URL would remain accessible to anyone.

---

**Q7.**
A company has a CloudFront distribution caching API responses. After deploying a new API version, users are still seeing stale responses. How should they immediately clear the cache?

A) Disable the CloudFront distribution and re-enable it
B) Create a cache invalidation for the affected paths (e.g., `/api/*`)
C) Change the CloudFront TTL to 0 for all behaviors
D) Create a new CloudFront distribution pointing to the new API

**Answer: B** — Cache invalidation lets you remove specific objects (or path patterns) from CloudFront's edge caches immediately. The first 1,000 invalidation paths per month are free. Disabling (A) and re-enabling the distribution interrupts service. Setting TTL to 0 (C) affects future caching behavior but doesn't clear existing cached content. A new distribution (D) has a different DNS name and requires client updates.

---

**Q8.**
An application running in a private subnet needs to call the AWS SSM API to retrieve parameters from Parameter Store. There is no NAT Gateway. What should the solutions architect create?

A) A NAT Gateway in a public subnet with a route from the private subnet
B) Three Interface Endpoints: `ssm`, `ssmmessages`, `ec2messages`
C) A Gateway Endpoint for SSM
D) A public route table entry for SSM's IP range

**Answer: B** — SSM Session Manager and Parameter Store require three Interface Endpoints (PrivateLink): `ssm`, `ssmmessages`, and `ec2messages`. These create ENIs in your subnet with private IPs, allowing private connectivity to SSM without internet. There is no Gateway Endpoint for SSM (C) — Gateway endpoints only exist for S3 and DynamoDB. Hard-coding SSM IP ranges (D) is not feasible as they change.

---

**Q9.**
A company wants to expose an internal microservice (running on EC2 behind an NLB) to other AWS accounts without making it publicly accessible. Which solution achieves this?

A) VPC Peering between the consumer VPC and the provider VPC
B) Create a VPC Endpoint Service (PrivateLink) fronted by the NLB; consumers create Interface Endpoints to connect
C) Make the NLB internet-facing and share the DNS name
D) Set up a Transit Gateway between all accounts

**Answer: B** — PrivateLink/VPC Endpoint Services let you expose a service privately to other VPCs and accounts without peering, public internet, or overlapping CIDR issues. The provider creates an Endpoint Service backed by an NLB. Consumers create an Interface Endpoint to connect. VPC Peering (A) exposes the entire VPC, not just one service, and has CIDR overlap limitations. TGW (D) allows full VPC-to-VPC routing, not service-level isolation.

---

**Q10.**
A Route 53 health check is monitoring an ALB endpoint in us-east-1. The health check fails. The Route 53 failover record has a secondary record pointing to an ALB in us-west-2. What happens?

A) Route 53 automatically redirects all DNS queries to the us-west-2 record
B) Route 53 continues to return the us-east-1 record until an operator intervenes
C) Route 53 returns both records and lets clients choose
D) Route 53 disables the us-east-1 ALB

**Answer: A** — Route 53 failover routing automatically serves the secondary record when the primary's health check fails. DNS TTL determines how quickly clients pick up the change. Route 53 does not control the ALBs themselves (D) — it only changes which DNS record is served.

---

**Q11.**
A company uses Direct Connect for connectivity between on-premises and AWS. They need redundancy in case the Direct Connect circuit fails. What should they add?

A) A second Direct Connect connection from a different Direct Connect location
B) A Site-to-Site VPN as a backup path over the internet
C) A second Direct Connect from the same location
D) Either A or B — both provide redundancy, with different cost/latency trade-offs

**Answer: D** — AWS recommends either a second Direct Connect from a different location (A) for maximum resilience, or a Site-to-Site VPN as a lower-cost backup (B). A second DX from the same location (C) doesn't protect against location-level failures. The exam accepts both A and B as valid answers depending on the specific requirement (cost vs maximum resilience).

---

**Q12.**
A company's EC2 instances in a VPC need to access an on-premises LDAP server. The on-premises data center is connected via Direct Connect. The LDAP server IP is `192.168.1.10`. What must be in place for the EC2 instances to reach this IP?

A) A public IP on the EC2 instances
B) A route in the VPC route table for the on-premises CIDR pointing to the Virtual Private Gateway (VGW) or Transit Gateway
C) A NAT Gateway to translate EC2 private IPs for the on-premises network
D) VPC Peering between the VPC and the on-premises network

**Answer: B** — For Direct Connect (or VPN) connectivity, the VPC route table must have a route for the on-premises CIDR pointing to the VGW (if using VGW) or TGW (if using TGW as the attachment point). The on-premises router must also have a return route for the VPC CIDR. EC2 instances don't need public IPs for private connectivity. NAT (C) is for internet access. Peering (D) is for VPC-to-VPC, not on-premises.

---

**Q13.**
A company wants users in the US to be routed to servers in us-east-1, users in Europe to eu-west-1, and all other users to us-east-1 as a default. Which Route 53 policy achieves this?

A) Latency-based routing
B) Weighted routing with a 50/50 split
C) Geolocation routing with US → us-east-1, Europe → eu-west-1, and a default record
D) Multivalue answer routing

**Answer: C** — Geolocation routing routes based on the user's geographic location. You define records for specific geographies and a default record for unmatched locations. Latency (A) routes to the lowest-latency endpoint — users in South America might go to us-east-1 but not always. Weighted (B) splits traffic by percentage, not location.

---

**Q14.**
An NLB is configured with a target group of EC2 instances. A client connects to the NLB on port 443. The EC2 instances need to see the original client IP. What is the correct statement?

A) NLBs do not support seeing the client IP — use an ALB instead
B) NLBs preserve the client source IP by default for TCP traffic — the EC2 instance sees the client IP directly
C) The client IP is available in the `X-Forwarded-For` header
D) The client IP is only available via VPC Flow Logs

**Answer: B** — Unlike ALBs (which use a proxy model), NLBs operate at Layer 4 and preserve the client's source IP address by default. EC2 instances see the real client IP in the packet's source address. `X-Forwarded-For` (C) is an HTTP header — not applicable at Layer 4. ALBs use `X-Forwarded-For` because they terminate and re-initiate the TCP connection.

---

**Q15.**
A company wants to improve the performance of their globally distributed application by routing users to the nearest AWS edge location and providing static anycast IP addresses. Which service should they use?

A) CloudFront
B) Route 53 latency-based routing
C) AWS Global Accelerator
D) Elastic Load Balancing

**Answer: C** — Global Accelerator provides static anycast IP addresses at AWS edge locations and routes user traffic over the AWS global backbone to the nearest regional endpoint, reducing latency. It also provides fast failover (within seconds) vs DNS-based failover. CloudFront (A) is a CDN for cacheable content — it doesn't provide static IPs. Route 53 (B) still uses DNS-based routing with TTL delays. ELB (D) operates within a single region.

---

**Q16.**
A company's VPC has both a VPC Gateway Endpoint for S3 and a NAT Gateway. An EC2 instance in a private subnet makes an S3 API call. Which path does the traffic take?

A) NAT Gateway → IGW → public S3 endpoint
B) The Gateway Endpoint route — it's more specific than the `0.0.0.0/0 → NAT GW` default route
C) Both paths simultaneously for redundancy
D) The path depends on the S3 bucket's region

**Answer: B** — The Gateway Endpoint injects a route into the route table with a specific destination prefix list (e.g., `pl-xxxxx`). Route tables use longest-prefix matching — the more specific prefix list entry wins over the `0.0.0.0/0` default route to NAT GW. All S3 traffic automatically uses the endpoint without any configuration change needed in the application.

---

**Q17.**
A company runs a multi-tier application. The web tier is in a public subnet, the app tier in a private subnet, and the database tier in an isolated subnet (no internet route, no NAT). The app tier needs to call an AWS service API. How can this be done without adding internet access to the app tier subnet?

A) Move the app tier to a public subnet
B) Add a NAT Gateway to the app tier subnet
C) Create Interface Endpoints (PrivateLink) for the required AWS services in the private subnet
D) Create a VPC Peering connection to the AWS service endpoint

**Answer: C** — Interface Endpoints (PrivateLink) create ENIs with private IPs in your subnet for accessing AWS service APIs privately. No internet route or NAT GW needed. This is the standard pattern for isolated subnet AWS API access. Moving to a public subnet (A) violates the security design. NAT GW (B) requires internet access in the public subnet and routes traffic externally.

---

**Q18.**
A company recently migrated from an on-premises network (`10.0.0.0/8`) to AWS. Their VPC uses `10.1.0.0/16`. They attempt to create a VPC peering connection with a partner company's VPC that also uses `10.1.0.0/16`. What happens?

A) The peering connection is created successfully — routing handles the overlap
B) The peering connection cannot be created because the CIDR blocks overlap
C) The peering connection is created but traffic is automatically NATed to avoid conflicts
D) One VPC must be re-addressed before peering is possible

**Answer: B** — VPC Peering requires non-overlapping CIDR blocks. Two VPCs with identical or overlapping CIDRs cannot be peered. AWS rejects the peering request. PrivateLink is an alternative that doesn't require non-overlapping CIDRs — it exposes a specific service endpoint. Re-addressing (D) is the solution if peering is required, but it's complex and disruptive.

---

**Q19.**
A company's CloudFront distribution has a default TTL of 24 hours. An object in S3 is updated. A user requests the object 2 hours after the update. What does the user receive?

A) The updated object — CloudFront always fetches fresh content from S3
B) The cached (stale) version — the TTL has not expired, so CloudFront serves from cache
C) A 404 error — CloudFront detects the object was updated
D) It depends on whether the S3 object's ETag changed

**Answer: B** — CloudFront caches objects for the duration of the TTL. With a 24-hour TTL, the cached version is served for 24 hours from the time of the first cache fill, regardless of changes at the origin. To serve the update immediately, you must either create a cache invalidation or use versioned URLs (e.g., `image-v2.jpg`). ETags (D) are checked when CloudFront forwards a conditional request to the origin, but only when the TTL has expired.

---

**Q20.**
A company uses AWS Direct Connect. The connection is established but traffic between on-premises and AWS is not flowing. Which is the most likely configuration missing on the AWS side?

A) VPC Flow Logs are not enabled
B) A Virtual Interface (VIF) has not been created, or the BGP session on the VIF is down
C) The Direct Connect gateway is not attached to a Transit Gateway
D) The S3 Gateway Endpoint is not configured

**Answer: B** — Direct Connect physical connectivity is separate from logical connectivity. After the physical circuit is established, you must create a Virtual Interface (private VIF for VPC, public VIF for AWS public services, transit VIF for TGW) and establish a BGP session over it. If the VIF doesn't exist or BGP isn't up, no traffic flows. Flow Logs (A) are monitoring, not connectivity. TGW attachment (C) is only needed if using a transit VIF. S3 endpoints (D) are unrelated to Direct Connect connectivity.

---

## Score Log

| Date | Score | Notes |
|---|---|---|
| | /20 | |
