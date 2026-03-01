# Lab: VPC Peering

*Phase 4 · Week 7 · Concept: [concepts/networking-advanced.md](../concepts/networking-advanced.md)*

---

## Goal

Peer two custom VPCs in the same region; verify private communication between EC2 instances in each VPC.

---

## Architecture

```
VPC-A (10.0.0.0/16)          VPC-B (172.16.0.0/16)
  Subnet-A (10.0.1.0/24)  ↔  Subnet-B (172.16.1.0/24)
  EC2-A                       EC2-B
```

---

## Tasks

### 1. Create VPC-A
- [ ] CIDR: `10.0.0.0/16`. No default VPC.
- [ ] Public subnet: `10.0.1.0/24` in AZ-a.
- [ ] Internet Gateway + route table (`0.0.0.0/0 → IGW`).
- [ ] Launch EC2-A (Amazon Linux, t3.micro) in public subnet. Security group: SSH from your IP.

### 2. Create VPC-B
- [ ] CIDR: `172.16.0.0/16`.
- [ ] Public subnet: `172.16.1.0/24` in AZ-a.
- [ ] Internet Gateway + route table.
- [ ] Launch EC2-B in public subnet. Security group: ICMP + SSH from VPC-A CIDR (`10.0.0.0/16`).

### 3. Create VPC Peering Connection
- [ ] Go to VPC → Peering Connections → Create.
- [ ] Requester: VPC-A. Accepter: VPC-B (same account, same region).
- [ ] Accept the peering connection from VPC-B side.

### 4. Update Route Tables
- [ ] VPC-A route table: add `172.16.0.0/16 → pcx-xxxxxx`.
- [ ] VPC-B route table: add `10.0.0.0/16 → pcx-xxxxxx`.

### 5. Test
- [ ] SSH into EC2-A.
- [ ] From EC2-A: `ping <EC2-B private IP>` → should succeed.
- [ ] From EC2-A: `curl http://<EC2-B private IP>` (if httpd installed on B).

### 6. Verify Non-Transitivity (Stretch)
- [ ] Create VPC-C (`192.168.0.0/16`); peer with VPC-B.
- [ ] Try to ping EC2-C from EC2-A → should fail (no direct peering between A and C).

---

## Key Things to Observe

- Both route tables must be updated for bidirectional routing.
- Security groups must explicitly allow traffic from the peered CIDR.
- Peering is non-transitive — you cannot reach VPC-C via VPC-B from VPC-A.

---

## Clean Up

- Delete EC2 instances.
- Delete peering connections.
- Delete VPCs (deletes associated subnets, route tables, IGWs).
