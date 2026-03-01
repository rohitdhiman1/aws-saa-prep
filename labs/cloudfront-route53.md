# Lab: CloudFront + Route 53 — CDN with OAC & Routing Policies

*Phase 4 · Week 8 · Sat Apr 25 · Concept: [concepts/edge-dns.md](../concepts/edge-dns.md)*

---

## Goal

Serve an S3-hosted site via CloudFront with OAC (private S3), then explore Route 53 routing policies: weighted and failover.

---

## Part A — CloudFront + S3 with OAC

### 1. Create a Private S3 Bucket (Origin)

- [ ] Create bucket: `my-cf-origin-<random>` in `us-east-1`.
- [ ] **Block all public access: ON** (this is what makes OAC necessary).
- [ ] Upload `index.html`:
  ```html
  <!DOCTYPE html>
  <html><body><h1>Served via CloudFront — Origin: us-east-1</h1></body></html>
  ```
- [ ] Try opening the S3 object URL directly → **Access Denied** (expected — bucket is private).

### 2. Create a CloudFront Distribution

- [ ] CloudFront → Create distribution.
- [ ] Origin domain: select your S3 bucket from the dropdown.
- [ ] **Origin access: Origin access control settings (recommended)** → Create new OAC → Sign requests: yes → Create.
- [ ] Viewer protocol policy: **Redirect HTTP to HTTPS**.
- [ ] Default root object: `index.html`.
- [ ] Create distribution. (Takes ~5–10 min to deploy globally.)
- [ ] **CloudFront will prompt:** "Update S3 bucket policy" → Copy the policy → go to S3 → Permissions → Bucket policy → paste it.
  The policy allows only CloudFront (with your distribution's ARN) to access the bucket.

### 3. Test

- [ ] Once deployed, open the CloudFront domain name (`xxxx.cloudfront.net`) in a browser → your HTML page loads over HTTPS.
- [ ] Try the S3 direct URL again → still **Access Denied** (only CloudFront can access it).
- [ ] In CloudFront → Invalidations → Create: path `/*` → submit. This clears the cache; useful when you update S3 content.

### 4. Update Content and Observe Caching

- [ ] Upload a new `index.html` with `<h1>Version 2</h1>`.
- [ ] Open CloudFront URL → still shows old content (cached).
- [ ] Create an invalidation (`/*`) → wait ~30s → refresh → now shows Version 2.

---

## Part B — Route 53 Routing Policies

*(This part requires a registered domain or a Route 53 hosted zone. If you don't have one, read through the steps and observe the console — the exam tests conceptual knowledge, not necessarily execution.)*

### Weighted Routing (A/B Testing)

- [ ] Route 53 → Hosted zones → select your domain.
- [ ] Create two A records with the **same name** (e.g. `app.yourdomain.com`), same type:
  - Record 1: value = `1.2.3.4` (simulating Region A), weight: `80`, Record ID: `primary`
  - Record 2: value = `5.6.7.8` (simulating Region B), weight: `20`, Record ID: `secondary`
- [ ] DNS will route ~80% of requests to Record 1 and ~20% to Record 2.
- [ ] To completely shift traffic away from one: set its weight to `0`.

### Failover Routing (Active-Passive)

**1. Create a health check**
- [ ] Route 53 → Health checks → Create.
- [ ] Monitor endpoint: specify your CloudFront domain or any reachable endpoint.
- [ ] Protocol: HTTPS. Path: `/`. Interval: 30s.

**2. Create failover records**
- [ ] Record 1: name `failover.yourdomain.com`, type: A (or CNAME), value: primary endpoint.
  - Routing policy: **Failover**. Failover record type: **Primary**. Health check: select the one you created.
- [ ] Record 2: same name `failover.yourdomain.com`, value: secondary endpoint (CloudFront domain or any backup).
  - Routing policy: Failover. Failover record type: **Secondary**. No health check needed on secondary.

**3. Observe**
- [ ] DNS currently resolves to Primary (health check is passing).
- [ ] In the health check, manually set the endpoint to an unreachable IP to simulate failure.
- [ ] Wait ~30–60s for health check to fail → Route 53 automatically starts returning the Secondary.

---

## Key Things to Observe

- **OAC vs direct S3:** Direct S3 URL returns 403; CloudFront URL works. The bucket policy only allows CloudFront's service principal with your distribution ARN.
- **Cache invalidation costs:** First 1,000 paths/month free; then $0.005/path. Use specific paths, not `/*`, in production.
- **Weighted routing = active-active:** Both records serve traffic simultaneously. Weight 0 = remove from rotation without deleting.
- **Failover routing = active-passive:** Secondary only serves if primary health check fails. There's a delay (~30–60s) for health check propagation.
- **Route 53 health check ≠ instant:** Takes 30s (standard) for a failure to register.

---

## Clean Up

- CloudFront → Disable distribution first → wait until deployed → then Delete.
- S3 → Delete objects → Delete bucket.
- Route 53 → Delete test records and health checks.
