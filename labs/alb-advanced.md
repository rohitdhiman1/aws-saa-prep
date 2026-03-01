# Lab: ALB Advanced — Path Routing, HTTPS & Sticky Sessions

*Phase 2 · Compute · Concept: [concepts/compute.md](../concepts/compute.md)*

---

## Goal

Go beyond a basic ALB. Set up path-based routing to split traffic between two target groups, enforce HTTPS with an HTTP redirect, and observe sticky sessions pinning a client to one instance.

---

## Architecture

```
Internet (HTTPS)
     │
  ALB (:443)
  ├── /api/*  → Target Group: api-servers (EC2-API-1, EC2-API-2)
  └── /*      → Target Group: web-servers (EC2-Web-1, EC2-Web-2)

HTTP (:80) → redirect to HTTPS
```

---

## Part A — Launch EC2 Instances

- [ ] Launch 4 EC2s (Amazon Linux, t3.micro) in two public subnets (2 AZs). No key pair needed — use user data.
  - **EC2-Web-1** user data:
    ```bash
    #!/bin/bash
    yum install -y httpd && systemctl start httpd
    echo "<h1>WEB SERVER: $(hostname -f)</h1>" > /var/www/html/index.html
    ```
  - **EC2-Web-2** — same but the hostname will differ.
  - **EC2-API-1** user data:
    ```bash
    #!/bin/bash
    yum install -y httpd && systemctl start httpd
    mkdir -p /var/www/html/api
    echo "<h1>API SERVER: $(hostname -f)</h1>" > /var/www/html/api/index.html
    ```
  - **EC2-API-2** — same.
- [ ] Security group: allow HTTP (80) from the ALB security group (configure after ALB SG is created).

---

## Part B — Create Target Groups

- [ ] Target Group 1: `web-servers` — protocol HTTP, port 80. Health check path: `/`. Register EC2-Web-1 and EC2-Web-2.
- [ ] Target Group 2: `api-servers` — protocol HTTP, port 80. Health check path: `/api/`. Register EC2-API-1 and EC2-API-2.

---

## Part C — Create the ALB

- [ ] Create internet-facing ALB. Subnets: 2 public subnets (2 AZs).
- [ ] Security group: allow inbound 80 and 443 from `0.0.0.0/0`.
- [ ] Listeners:
  - **HTTP :80** → Action: **Redirect to HTTPS** (301, port 443).
  - **HTTPS :443** → Default action: forward to `web-servers` target group. (You need an ACM certificate for HTTPS; for the lab, request a free public cert from ACM for a domain you own, OR use HTTP only and skip HTTPS — just understand the concept.)
- [ ] If skipping HTTPS for now: HTTP :80 → Default: forward to `web-servers`. Add a rule for `/api/*` → `api-servers`.

---

## Part D — Path-Based Routing Rules

- [ ] On the ALB → Listeners → HTTP :80 → View/Edit rules.
- [ ] Rules are evaluated top to bottom. Add:
  - Rule 1: IF path is `/api/*` → Forward to `api-servers`
  - Default rule (last): Forward to `web-servers`
- [ ] Save.

**Test:**
- [ ] Open `http://<ALB-DNS>/` → web server responds (hostname of EC2-Web-1 or Web-2).
- [ ] Open `http://<ALB-DNS>/api/` → API server responds (hostname of EC2-API-1 or API-2).
- [ ] Refresh `/` several times → hostname alternates between Web-1 and Web-2 (round-robin).
- [ ] Refresh `/api/` several times → alternates between API-1 and API-2.

---

## Part E — Sticky Sessions

- [ ] Go to `web-servers` target group → Attributes → Edit.
- [ ] Enable **Stickiness**. Type: Load balancer generated cookie. Duration: 1 day.
- [ ] Save.

**Test:**
- [ ] Open `http://<ALB-DNS>/` in browser. Note the hostname.
- [ ] Refresh 10 times — **same hostname** every time. You're stuck to one instance.
- [ ] Open the browser dev tools → Application → Cookies. Find the `AWSALB` cookie the ALB set.
- [ ] Delete the cookie → refresh → may land on a different instance → new cookie issued → stuck to that one.

**Contrast with target group 2:**
- [ ] Open `http://<ALB-DNS>/api/` → hostname alternates (stickiness not enabled on `api-servers`).

---

## Part F — Health Check Behaviour

- [ ] Stop the httpd service on EC2-Web-1 via SSM Session Manager:
  ```bash
  sudo systemctl stop httpd
  ```
- [ ] Watch the target group → EC2-Web-1 health check starts failing after 2 checks (default unhealthy threshold).
- [ ] ALB stops sending traffic to EC2-Web-1. All requests now go to EC2-Web-2.
- [ ] Restart httpd → EC2-Web-1 becomes healthy again → traffic resumes to both.

---

## Key Things to Observe

- **Path-based routing** — the ALB evaluates rules top-to-bottom; first match wins.
- **HTTP → HTTPS redirect** is a built-in ALB action, not something your app handles.
- **Sticky sessions** work via an `AWSALB` cookie that maps a client to a specific target. Deleting the cookie breaks the stickiness.
- **Health checks** — ALB silently removes unhealthy targets; clients never see a 502 if at least one healthy target exists.
- **Source IP** — the target EC2 sees the ALB's IP, not the client's. Real client IP is in the `X-Forwarded-For` header.

---

## Clean Up

- Delete ALB and both target groups.
- Terminate all 4 EC2s.
- Delete security groups.
