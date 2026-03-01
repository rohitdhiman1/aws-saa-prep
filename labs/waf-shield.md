# Lab: WAF & Shield — Web ACL, Rate Limiting & Managed Rules

*Phase 5 · Security · Concept: [concepts/security.md](../concepts/security.md)*

---

## Goal

Attach a WAF Web ACL to an ALB, block a specific IP, apply rate-based limiting, enable AWS Managed Rules (OWASP top 10), and test that a SQL injection attempt is blocked. Understand the difference between Shield Standard (free, always on) and Shield Advanced (paid).

---

## Architecture

```
Internet → ALB → EC2 (simple web server)
               ↑
           WAF Web ACL
           ├── Rule 1: Block my test IP (IP set rule)
           ├── Rule 2: Rate limit — 100 req/5min per IP
           └── Rule 3: AWS Managed Rules (SQLi, XSS, common threats)
```

---

## Part A — Set Up a Simple Web Server Behind an ALB

- [ ] Launch an EC2 (Amazon Linux, t3.micro, public subnet). User data:
  ```bash
  #!/bin/bash
  yum install -y httpd && systemctl start httpd
  echo "<h1>WAF Test Server: $(hostname -f)</h1>" > /var/www/html/index.html
  ```
- [ ] Create a target group (Instance type, HTTP :80, health check `/`). Register the EC2.
- [ ] Create an internet-facing ALB. Listener HTTP :80 → forward to target group.
- [ ] Test: `http://<ALB-DNS>/` → returns the page.

---

## Part B — Create a WAF Web ACL

- [ ] WAF & Shield → Web ACLs → Create.
- [ ] Resource type: **Regional** (for ALB). Region: your region.
- [ ] Name: `lab-web-acl`. Default action: **Allow**.

**Add Rule 1 — IP Set Block:**
- [ ] Rules → Add rules → Add my own rules → IP set rule.
  - First create an IP set: WAF → IP sets → Create.
    - Name: `blocked-ips`. IP version: IPv4. Add your current public IP (find it at checkip.amazonaws.com) in CIDR: `<your-ip>/32`.
  - Back in Web ACL: Rule type: IP set. Select `blocked-ips`. Action: **Block**.

**Add Rule 2 — Rate-Based Rule:**
- [ ] Add rules → Rate-based rule.
- [ ] Name: `rate-limit-100`. Rate limit: **100** (requests per 5-minute window). Evaluation: IP address. Action: **Block**.

**Add Rule 3 — AWS Managed Rules:**
- [ ] Add rules → Add managed rule groups → AWS managed rule groups.
- [ ] Enable: **AWSManagedRulesCommonRuleSet** (covers XSS, SQLi, bad bots, common CVEs). Action: **Block**.
- [ ] Save rule group.

- [ ] Review priority order:
  - 1 — IP set block (most specific, evaluated first)
  - 2 — Rate limit
  - 3 — AWS Managed Rules
  - Default: Allow

- [ ] Create Web ACL.

---

## Part C — Associate Web ACL with the ALB

- [ ] Web ACL → Associated AWS resources → Add association.
- [ ] Resource type: Application Load Balancer. Select your ALB. Save.

---

## Part D — Test Each Rule

**Test 1: IP Block**
- [ ] From your machine (the IP you added to `blocked-ips`), open `http://<ALB-DNS>/`.
- [ ] You get `403 Forbidden` immediately — WAF blocked before the request reached EC2.
- [ ] Remove your IP from the IP set → page loads again.

**Test 2: SQL Injection Block (AWS Managed Rules)**
- [ ] Try a SQL injection in a query string:
  ```
  http://<ALB-DNS>/?id=1' OR '1'='1
  ```
  → WAF returns 403. The `AWSManagedRulesCommonRuleSet` matched the SQLi pattern.

- [ ] Try a benign request:
  ```
  http://<ALB-DNS>/?id=123
  ```
  → 200 OK. Passes through.

**Test 3: Rate Limiting**
- [ ] Run a quick burst from your machine:
  ```bash
  for i in {1..110}; do curl -s -o /dev/null -w "%{http_code}\n" http://<ALB-DNS>/; done
  ```
  → First ~100 requests return `200`. After 100, you get `403` — rate limit triggered.
  → Wait 5 minutes → resets, `200` again.

---

## Part E — Review WAF Logs and Metrics

**CloudWatch Metrics:**
- [ ] WAF → Web ACL → Overview tab → see request counts by rule.
- [ ] CloudWatch → Metrics → WAFV2 → `BlockedRequests`, `AllowedRequests`, `CountedRequests`.

**Enable WAF Logging (optional, costs ~$0.10/GB):**
- [ ] Web ACL → Logging and metrics → Enable logging.
- [ ] Destination: S3 bucket or CloudWatch Logs.
- [ ] After a few requests, check logs — each entry shows which rule matched, the request headers, and the action taken.

---

## Part F — Understand Shield Standard vs Advanced

| | Shield Standard | Shield Advanced |
|---|---|---|
| Cost | **Free** — always on | $3,000/month per org |
| Coverage | L3/L4 DDoS (SYN flood, UDP flood) | L3/L4 + L7 application DDoS |
| Automatic mitigation | Yes, basic | Yes, with AWS DDoS Response Team |
| WAF integration | No | Yes — WAF rules auto-deployed during attack |
| Financial protection | No | Yes — credits for scaling costs during attacks |
| Applies to | All AWS customers automatically | Opt-in |

**Exam scenarios where Shield Advanced is the answer:**
- Customer needs DDoS protection for EC2, ELB, CloudFront, Route 53 at L7.
- Customer wants AWS DRT (DDoS Response Team) support during an active attack.
- "Cost protection" against AWS bill spikes from DDoS scaling.

Shield Standard is always on — you never need to enable it. If an exam question says "enable DDoS protection", they usually mean Shield Advanced.

---

## Key Things to Observe

- WAF rules are evaluated in priority order (lowest number first). First match wins — execution stops.
- Default action (Allow or Block) applies only when **no rule matches**.
- WAF `Count` action is useful for testing rules without blocking — logs the match, passes the request.
- AWS Managed Rules cover OWASP Top 10 without you writing any regex — use them in production.
- WAF can be attached to: ALB, CloudFront, API Gateway, AppSync, Cognito User Pools.
- WAF cannot be attached directly to an EC2 or NLB.
- Shield Advanced wraps WAF for automatic L7 attack mitigation — without it you write WAF rules manually.

---

## Clean Up

- WAF → Web ACL → Associated resources → remove ALB association.
- Delete Web ACL.
- Delete IP set.
- Delete ALB, target group.
- Terminate EC2.
