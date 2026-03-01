# Diagrams: Networking

*Draw these on paper or in draw.io as part of Week 7–8 study.*

---

## Diagram 1: Transit Gateway Hub-and-Spoke

```
                    ┌─────────────────────────────────────┐
                    │         Transit Gateway              │
                    │  (TGW Route Table: all attachments) │
                    └──┬───────┬──────────┬───────────────┘
                       │       │          │
               ┌───────┘   ┌───┘      ┌───┘
               ▼           ▼          ▼
            VPC-A        VPC-B     VPC-C        ◄── All can talk to each other
         (10.0.0.0/16) (10.1.0.0/16) (10.2.0.0/16)      via TGW
                             │
                          VPN Attachment
                             │
                      On-Premises Network
```

**Route table control:** Segment VPCs with separate TGW route tables (e.g., Prod TGW RT, Dev TGW RT).

---

## Diagram 2: Direct Connect Topology

```
On-Premises DC
      │
  Customer Router
      │  (private fibre cross-connect)
  DX Location (colocation facility)
      │  (AWS backbone)
  AWS Direct Connect Router
      │
  ┌───┴─────────────────────────────────────┐
  │    Private VIF ──► VGW ──► VPC          │
  │    Public VIF ──► AWS Public Services   │
  │    Transit VIF ──► Transit Gateway      │
  └─────────────────────────────────────────┘

For multi-VPC/region:
On-Prem ──DX──► DX Gateway ──Private VIF──► VGW (VPC-A, VPC-B, VPC-C)
                            └─Transit VIF──► TGW (hundreds of VPCs)
```

---

## Diagram 3: CloudFront + S3 with OAC

```
User (browser)
      │  HTTPS
      ▼
CloudFront Distribution
      │  Cache miss: fetches from origin
      │  Cache hit: serves from edge
      ▼
S3 Bucket (private, Block Public Access = ON)
  └── Bucket Policy: Allow cloudfront.amazonaws.com
                     with aws:SourceArn = distribution ARN (OAC)

S3 does NOT have a public URL — only accessible via CloudFront.
```

---

## Diagram 4: Route 53 Failover Routing

```
Route 53 Hosted Zone: api.example.com

                    Health Check ──► Primary (us-east-1 ALB)
                    ● PASS: route to Primary
                    ● FAIL: route to Secondary

Primary Record:     api.example.com A ALIAS → alb.us-east-1.amazonaws.com  [PRIMARY]
Secondary Record:   api.example.com A ALIAS → alb.eu-west-1.amazonaws.com  [SECONDARY]
```

---

## Diagram 5: VPC Endpoints

```
GATEWAY ENDPOINT (S3 / DynamoDB — Free):
EC2 (private subnet) ──route table entry──► Gateway Endpoint ──► S3
(no IGW, no NAT, no public IP needed)

INTERFACE ENDPOINT (any AWS service):
EC2 (private subnet) ──private DNS──► Interface Endpoint (ENI) ──► Service
                                       10.0.x.x (private IP)
(security group on endpoint; private DNS resolves service hostname to private IP)
```
