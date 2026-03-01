# Lab: Transit Gateway — Hub-and-Spoke Multi-VPC Networking

*Phase 4 · Advanced Networking · Concept: [concepts/networking-advanced.md](../concepts/networking-advanced.md)*

---

## Goal

Build a Transit Gateway connecting three VPCs in a hub-and-spoke model. Demonstrate transitive routing (which VPC peering cannot do), implement route table isolation to block traffic between specific VPCs, and understand TGW vs peering vs PrivateLink.

---

## Architecture

```
       VPC-A (10.1.0.0/16)            VPC-B (10.2.0.0/16)
       "Production"                    "Development"
       EC2: 10.1.1.x                   EC2: 10.2.1.x
            │                               │
            │ TGW Attachment                 │ TGW Attachment
            │                               │
       ┌────┴───────────────────────────────┴────┐
       │         Transit Gateway (TGW)            │
       │         Regional hub router              │
       └────┬────────────────────────────────────┘
            │
            │ TGW Attachment
            │
       VPC-C (10.3.0.0/16)
       "Shared Services"
       EC2: 10.3.1.x
```

After setup: A↔C works, B↔C works, A↔B is blocked (isolated via TGW route tables).

---

## Part A — Create Three VPCs

**VPC-A (Production)**
- [ ] VPC → Create VPC → VPC only.
- [ ] Name: `vpc-prod`. CIDR: `10.1.0.0/16`.
- [ ] Create subnet: `prod-subnet` → AZ: `us-east-1a` → CIDR: `10.1.1.0/24`.
- [ ] Create and attach an IGW: `igw-prod`.
- [ ] Route table: add `0.0.0.0/0 → igw-prod` (so you can SSH in).

**VPC-B (Development)**
- [ ] Name: `vpc-dev`. CIDR: `10.2.0.0/16`.
- [ ] Subnet: `dev-subnet` → AZ: `us-east-1a` → CIDR: `10.2.1.0/24`.
- [ ] IGW: `igw-dev`. Route: `0.0.0.0/0 → igw-dev`.

**VPC-C (Shared Services)**
- [ ] Name: `vpc-shared`. CIDR: `10.3.0.0/16`.
- [ ] Subnet: `shared-subnet` → AZ: `us-east-1a` → CIDR: `10.3.1.0/24`.
- [ ] IGW: `igw-shared`. Route: `0.0.0.0/0 → igw-shared`.

**Launch one EC2 instance in each VPC:**
- [ ] `t3.micro`, Amazon Linux 2023, public IP enabled, SSH SG (port 22 from your IP).
- [ ] Names: `ec2-prod`, `ec2-dev`, `ec2-shared`.
- [ ] **Important:** Each instance's security group must allow **All ICMP - IPv4** from `10.0.0.0/8` (so ping works across VPCs after TGW is set up).

---

## Part B — Create the Transit Gateway

- [ ] VPC → Transit Gateways → Create Transit Gateway.
- [ ] Name: `lab-tgw`.
- [ ] ASN: default (64512).
- [ ] DNS support: enable. VPN ECMP support: enable.
- [ ] Default route table association: **enable**.
- [ ] Default route table propagation: **enable**.
- [ ] Create. Wait for state: **available** (~2 minutes).

---

## Part C — Attach All Three VPCs

**1. Attach VPC-A**
- [ ] Transit Gateway → Attachments → Create attachment.
- [ ] Transit Gateway: `lab-tgw`.
- [ ] Attachment type: **VPC**.
- [ ] VPC: `vpc-prod`. Subnet: `prod-subnet`.
- [ ] Name: `tgw-attach-prod`.
- [ ] Create.

**2. Attach VPC-B**
- [ ] Same steps → VPC: `vpc-dev` → Subnet: `dev-subnet` → Name: `tgw-attach-dev`.

**3. Attach VPC-C**
- [ ] Same steps → VPC: `vpc-shared` → Subnet: `shared-subnet` → Name: `tgw-attach-shared`.

- [ ] Wait for all three attachments to show **available**.

---

## Part D — Update VPC Route Tables

Each VPC needs a route to the other VPCs via the TGW.

**VPC-A route table:**
- [ ] Add route: `10.0.0.0/8 → Transit Gateway (lab-tgw)`.

**VPC-B route table:**
- [ ] Add route: `10.0.0.0/8 → Transit Gateway (lab-tgw)`.

**VPC-C route table:**
- [ ] Add route: `10.0.0.0/8 → Transit Gateway (lab-tgw)`.

Using `10.0.0.0/8` as a summary route means all three VPCs can reach each other through the TGW with a single route entry.

---

## Part E — Test Transitive Routing

**1. Verify full connectivity (all three VPCs can talk)**
- [ ] SSH into `ec2-prod` (VPC-A):
  ```bash
  # Ping Shared Services (VPC-C)
  ping 10.3.1.x -c 3    # should work

  # Ping Dev (VPC-B)
  ping 10.2.1.x -c 3    # should work
  ```
- [ ] SSH into `ec2-dev` (VPC-B):
  ```bash
  ping 10.3.1.x -c 3    # should work (B → C)
  ping 10.1.1.x -c 3    # should work (B → A)
  ```

**2. This is transitive routing — the key advantage over VPC peering**
- [ ] With VPC peering: A↔B, B↔C works, but A↔C does NOT (non-transitive). You'd need a third peering connection.
- [ ] With Transit Gateway: A↔B, A↔C, B↔C all work through the single TGW hub. This scales to hundreds of VPCs.

---

## Part F — Route Table Isolation (Block Prod ↔ Dev)

Now isolate Production from Development while both can still reach Shared Services.

**1. Create two TGW route tables**
- [ ] Transit Gateway → Route Tables → Create.
- [ ] Name: `rt-prod-shared`. Associate with: `tgw-attach-prod`.
- [ ] Create another: `rt-dev-shared`. Associate with: `tgw-attach-dev`.
- [ ] (You may need to disassociate attachments from the default TGW route table first.)

**2. Configure rt-prod-shared (Prod can reach Shared only)**
- [ ] Create route: `10.3.0.0/16 → tgw-attach-shared`.
- [ ] Do NOT add a route to `10.2.0.0/16` (Dev is unreachable from Prod).
- [ ] Enable propagation from `tgw-attach-shared` (so Shared routes are learned automatically).

**3. Configure rt-dev-shared (Dev can reach Shared only)**
- [ ] Create route: `10.3.0.0/16 → tgw-attach-shared`.
- [ ] Do NOT add a route to `10.1.0.0/16` (Prod is unreachable from Dev).
- [ ] Enable propagation from `tgw-attach-shared`.

**4. Update the default TGW route table for Shared Services**
- [ ] The Shared Services attachment stays on the default route table.
- [ ] Ensure it has routes to both `10.1.0.0/16` and `10.2.0.0/16` (via propagation or static routes).

**5. Test isolation**
- [ ] SSH into `ec2-prod`:
  ```bash
  ping 10.3.1.x -c 3    # works (Prod → Shared)
  ping 10.2.1.x -c 3    # FAILS (Prod → Dev blocked)
  ```
- [ ] SSH into `ec2-dev`:
  ```bash
  ping 10.3.1.x -c 3    # works (Dev → Shared)
  ping 10.1.1.x -c 3    # FAILS (Dev → Prod blocked)
  ```
- [ ] SSH into `ec2-shared`:
  ```bash
  ping 10.1.1.x -c 3    # works (Shared → Prod)
  ping 10.2.1.x -c 3    # works (Shared → Dev)
  ```

This is the classic **shared services isolation pattern** — heavily tested on the exam.

---

## Part G — TGW vs VPC Peering vs PrivateLink

| Feature | VPC Peering | Transit Gateway | PrivateLink |
|---|---|---|---|
| Topology | Point-to-point | Hub-and-spoke | Service-to-consumer |
| Transitive routing | No | Yes | N/A |
| Max connections | 125 per VPC | 5,000 attachments | Per endpoint |
| Cross-region | Yes | Yes (inter-region peering) | Yes (via interface endpoints) |
| Cross-account | Yes | Yes | Yes |
| Cost | Free (data transfer only) | $0.05/hr per attachment + $0.02/GB | $0.01/hr per endpoint + $0.01/GB |
| Route isolation | No | Yes (TGW route tables) | Implicit (expose one service) |
| Best for | Few VPCs, simple connections | Many VPCs, centralized networking | Exposing a single service privately |

**Exam decision tree:**

| Scenario | Answer |
|---|---|
| "Connect 2–3 VPCs, keep it simple" | VPC Peering |
| "Connect 10+ VPCs in a hub-and-spoke" | Transit Gateway |
| "Centralized egress (share one NAT GW)" | Transit Gateway |
| "Isolate prod from dev but share common services" | TGW with route table isolation |
| "On-premises to multiple VPCs" | TGW + VPN or Direct Connect |
| "Expose one microservice to partner VPCs" | PrivateLink |
| "Full mesh connectivity across 50 VPCs" | Transit Gateway (not 50×49/2 peering connections) |

**Exam trap:** TGW is **regional**. To connect VPCs across regions, you create a TGW in each region and peer them (TGW inter-region peering). The peered TGWs share routes you explicitly configure — they don't auto-propagate.

---

## Key Things to Observe

- Transit Gateway acts as a regional cloud router — attach VPCs, VPNs, and Direct Connect gateways to one hub.
- Transitive routing is the key differentiator vs VPC peering. A→TGW→B→TGW→C works automatically.
- TGW route tables enable network segmentation (prod vs dev vs shared) without separate TGWs.
- Each TGW attachment costs ~$0.05/hr (~$36/month). This adds up with many VPCs — know the cost trade-off vs peering (free).
- VPC route tables must point to the TGW for cross-VPC traffic. The TGW route tables determine which attachment to forward to.
- Default route table association + propagation means "everything talks to everything." Custom route tables enable isolation.

---

## Clean Up

Delete in this order (dependencies matter):

1. Disassociate TGW route table associations.
2. Delete custom TGW route tables (`rt-prod-shared`, `rt-dev-shared`).
3. Delete TGW attachments (all three).
4. Delete Transit Gateway `lab-tgw` (wait for it to fully delete).
5. Terminate all 3 EC2 instances.
6. Delete IGWs (detach first, then delete).
7. Delete subnets.
8. Delete all 3 VPCs.
