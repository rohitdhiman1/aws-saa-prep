# Diagrams: VPC Basics

*Draw these on paper or in draw.io as part of Week 2 study.*

---

## Diagram 1: VPC Architecture — Public + Private Subnets

```
Region: us-east-1
┌──────────────────────────────────────────────────────┐
│  VPC: 10.0.0.0/16                                    │
│                                                      │
│  ┌──────────────────────┐  ┌──────────────────────┐  │
│  │  AZ: us-east-1a      │  │  AZ: us-east-1b      │  │
│  │                      │  │                      │  │
│  │  Public Subnet       │  │  Public Subnet       │  │
│  │  10.0.1.0/24         │  │  10.0.2.0/24         │  │
│  │  ┌──────────────┐    │  │  ┌──────────────┐    │  │
│  │  │ NAT Gateway  │    │  │  │ NAT Gateway  │    │  │
│  │  │ (with EIP)   │    │  │  │ (with EIP)   │    │  │
│  │  └──────────────┘    │  │  └──────────────┘    │  │
│  │                      │  │                      │  │
│  │  Private Subnet      │  │  Private Subnet      │  │
│  │  10.0.3.0/24         │  │  10.0.4.0/24         │  │
│  │  ┌──────────────┐    │  │  ┌──────────────┐    │  │
│  │  │  App Server  │    │  │  │  App Server  │    │  │
│  │  └──────────────┘    │  │  └──────────────┘    │  │
│  └──────────────────────┘  └──────────────────────┘  │
│                                                      │
│         Internet Gateway (IGW)                       │
└──────────────────────────────┬───────────────────────┘
                               │
                           Internet
```

**Route table - Public subnet:**
```
10.0.0.0/16  → local
0.0.0.0/0    → igw-xxx
```

**Route table - Private subnet (AZ-a):**
```
10.0.0.0/16  → local
0.0.0.0/0    → nat-xxx (NAT GW in AZ-a)
```

---

## Diagram 2: Security Group vs NACL

```
┌─────────────────────────────────────────┐
│  Subnet                                 │
│  ┌──────────────────────────────────┐   │
│  │  NACL (stateless)                │   │
│  │  Inbound: Allow 443              │   │
│  │  Outbound: Allow 1024-65535      │   ◄──── Internet traffic
│  │                                  │   │
│  │  ┌───────────────────────────┐   │   │
│  │  │  Security Group (stateful)│   │   │
│  │  │  Inbound: Allow 443       │   │   │
│  │  │  Outbound: All (default)  │   │   │
│  │  │                           │   │   │
│  │  │  ┌─────────────────────┐  │   │   │
│  │  │  │   EC2 Instance      │  │   │   │
│  │  │  └─────────────────────┘  │   │   │
│  │  └───────────────────────────┘   │   │
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

**Stateful (SG):** Allow inbound 443 → return traffic (outbound) auto-allowed.
**Stateless (NACL):** Must explicitly allow BOTH inbound 443 AND outbound ephemeral ports.

---

## Diagram 3: VPC Peering (Non-Transitive)

```
VPC-A ◄──────── Peering ────────► VPC-B
  │                                  │
  │                           Peering│
  │                                  │
  │                               VPC-C

A cannot reach C via B. Must create separate A↔C peering.
```

---

## Diagram 4: NAT Gateway Flow

```
EC2 (Private)                NAT GW (Public)         Internet
10.0.3.10          ──────►  Subnet: 10.0.1.x    ──────►  8.8.8.8
                            with Elastic IP
                            (SNAT: replaces src IP)
                   ◄──────  returns response    ◄──────
```

NAT GW does **Source NAT** — replaces private IP with its Elastic IP for outbound.
Return traffic is handled transparently. Inbound connections from internet are NOT possible.
