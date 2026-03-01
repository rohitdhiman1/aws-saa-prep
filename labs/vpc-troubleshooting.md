# Lab: VPC Troubleshooting — Diagnosing Connectivity Failures

*Phase 1 · VPC Basics · Concept: [concepts/vpc-basics.md](../concepts/vpc-basics.md)*

---

## Goal

Work through five intentionally broken VPC scenarios. Each one simulates a question format you'll see on the exam: "A developer can't connect to X. What is the cause?" You break it, you diagnose it, you fix it.

---

## Base Setup (do this once)

- [ ] Create a VPC: `10.0.0.0/16`. No default resources.
- [ ] Create a public subnet: `10.0.1.0/24` in AZ-a.
- [ ] Create a private subnet: `10.0.2.0/24` in AZ-a.
- [ ] Create an Internet Gateway → attach to the VPC.
- [ ] Create a public route table → route `0.0.0.0/0 → IGW` → associate with public subnet.
- [ ] Create a private route table (no IGW route) → associate with private subnet.
- [ ] Create a NAT Gateway in the **public subnet** with a new Elastic IP.
- [ ] Add `0.0.0.0/0 → NAT GW` to the **private route table**.
- [ ] Create two EC2 instances (Amazon Linux, t3.micro):
  - `EC2-Public`: public subnet, auto-assign public IP ON.
  - `EC2-Private`: private subnet, no public IP.
- [ ] Security group for both: allow SSH (22) from your IP, allow all outbound.

Confirm the baseline works:
- [ ] SSH into `EC2-Public` from your machine → should succeed.
- [ ] From `EC2-Public`, ping `EC2-Private` private IP → should succeed (same VPC, same SG allows inbound ICMP? Add it if not).
- [ ] From `EC2-Private`, `curl https://example.com` → should succeed (via NAT GW).

---

## Scenario 1 — EC2 Can't Reach the Internet (Missing IGW Route)

**Break it:**
- [ ] In the public route table, delete the route `0.0.0.0/0 → IGW`.

**Symptom:** SSH to `EC2-Public` times out. Can't load any external URL.

**Diagnose:**
- [ ] Check: Is the instance in a public subnet? (Yes)
- [ ] Check: Does the instance have a public IP? (Yes)
- [ ] Check: Is there an IGW attached to the VPC? (Yes)
- [ ] Check: **Route table** — is `0.0.0.0/0 → IGW` present? ← **Missing. This is the problem.**

**Fix:** Add `0.0.0.0/0 → igw-xxx` back to the public route table.

**Exam lesson:** Public IP + IGW attached is not enough. The subnet's route table must have the IGW route. Missing route = no internet regardless of everything else.

---

## Scenario 2 — EC2 Can't Connect to Another EC2 on Port 80 (Wrong Security Group)

**Break it:**
- [ ] Launch a third EC2 (`EC2-Web`) in the public subnet.
- [ ] Create a new security group `web-sg`: allow inbound port 80 from CIDR `192.168.0.0/16` (wrong CIDR — not your VPC range).
- [ ] Attach `web-sg` to `EC2-Web`. Install `httpd` via user data or manually.
- [ ] Try to `curl http://<EC2-Web private IP>` from `EC2-Public` → connection refused or timeout.

**Diagnose:**
- [ ] Check: Route table — both in same VPC, `local` route handles this. Not a routing issue.
- [ ] Check: **Security group on EC2-Web** — inbound rule for 80 is from `192.168.0.0/16`, not from `10.0.0.0/16` (your VPC). ← **This is the problem.**

**Fix:** Change the inbound rule CIDR to `10.0.0.0/16` (or reference the source security group ID).

**Exam lesson:** Security groups are stateful — if inbound is blocked, it doesn't matter that outbound is open. Referencing a CIDR vs a security group ID: use SG ID when source instances may change IPs.

---

## Scenario 3 — HTTPS Works, but HTTP Doesn't Return a Response (NACL Stateless Trap)

**Break it:**
- [ ] Create a custom NACL for the public subnet. By default it denies everything.
- [ ] Add only these rules to the NACL:
  - Inbound: Rule 100 — Allow TCP port 443 from `0.0.0.0/0`
  - Outbound: Rule 100 — Allow TCP port 443 to `0.0.0.0/0`
  - (Deliberately omit outbound ephemeral ports)

**Symptom:** HTTPS requests to EC2-Public seem to connect but responses never come back. Browser spins forever.

**Diagnose:**
- [ ] Check: Security group — allows 443 inbound and all outbound. Fine.
- [ ] Check: **NACL outbound rules** — only allows port 443 outbound. What about the **response** traffic?
  - Client sent request from a random ephemeral port (e.g. port 54123). Your EC2 must respond to that port.
  - Outbound rule only allows port 443 — the response to port 54123 is **blocked**.
  - NACLs are **stateless** — they don't know this is a response. ← **This is the problem.**

**Fix:** Add NACL outbound rule: Allow TCP ports `1024-65535` to `0.0.0.0/0` (ephemeral port range).

**Exam lesson:** This is the most common NACL trap on the exam. Every time you open an inbound port on a NACL, you must also open outbound ephemeral ports (1024–65535). Security groups don't have this problem — they're stateful.

---

## Scenario 4 — Private EC2 Can't Reach the Internet (NAT Gateway in Wrong Subnet)

**Break it:**
- [ ] Create a second NAT Gateway — but put it in the **private subnet** (wrong!). Give it an EIP.
- [ ] Change the private route table: `0.0.0.0/0 → new-wrong-nat-gw`.
- [ ] From `EC2-Private`, try `curl https://example.com` → times out.

**Diagnose:**
- [ ] Check: Private subnet route table — has `0.0.0.0/0 → nat-xxx`. Looks right.
- [ ] Check: **Which subnet is the NAT GW in?** → It's in the private subnet. Private subnet has no route to IGW. So NAT GW itself can't reach the internet. ← **This is the problem.**

**Fix:** Delete the wrong NAT GW. Create a new one in the **public subnet**. Update the private route table to point to the correct NAT GW.

**Exam lesson:** NAT Gateway must be in a **public subnet** (one that has a route to IGW). The NAT GW itself needs internet access to forward your private EC2's traffic.

---

## Scenario 5 — Two VPCs Peered but EC2s Still Can't Talk

**Break it:**
- [ ] Create a second VPC: `172.16.0.0/16`. Subnet: `172.16.1.0/24`. EC2: `EC2-VPC2`.
- [ ] Create a VPC peering connection between your original VPC (`10.0.0.0/16`) and the new one.
- [ ] Accept the peering connection.
- [ ] Update only ONE route table (in VPC1: `172.16.0.0/16 → pcx-xxx`) but forget the other side.
- [ ] Try to ping `EC2-VPC2` from `EC2-Public` → no response.

**Diagnose:**
- [ ] Check: Peering connection status — Active. Fine.
- [ ] Check: VPC1 route table — has route to `172.16.0.0/16 → pcx-xxx`. Fine.
- [ ] Check: **VPC2 route table** — does it have `10.0.0.0/16 → pcx-xxx`? ← **Missing. This is the problem.**
- [ ] Also check: Security groups on EC2-VPC2 — does it allow ICMP from `10.0.0.0/16`? Add it if not.

**Fix:** Add `10.0.0.0/16 → pcx-xxx` to VPC2's route table. Peering is bidirectional in terms of connection but route tables on **both sides** must be updated manually.

**Exam lesson:** VPC peering requires route table updates on BOTH sides. The peering connection itself is just a link — it doesn't add routes automatically.

---

## Summary: Troubleshooting Checklist

When an EC2 can't connect to something, check in this order:

```
1. Is it in the right subnet type? (public needs IGW route, private needs NAT GW)
2. Does the route table have the correct route? (IGW / NAT GW / PCX)
3. Does the security group allow the traffic? (correct port, correct source)
4. Does the NACL allow the traffic IN AND OUT? (including ephemeral outbound)
5. If VPC peering: are BOTH route tables updated?
6. Does the target have a public IP (if internet-facing) or EIP?
```

---

## Clean Up

- Delete all EC2 instances.
- Delete NAT Gateways (release EIPs too — they cost money when unattached).
- Delete VPC peering connection.
- Delete custom NACLs (restore default).
- Delete both VPCs.
