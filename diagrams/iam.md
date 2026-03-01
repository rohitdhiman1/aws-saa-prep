# Diagrams: IAM

*Draw these on paper or in draw.io as part of Week 1 study.*

---

## Diagram 1: Cross-Account Role Assumption

```
Account A (Trusted)                 Account B (Trusting)
┌─────────────────┐                ┌──────────────────────────┐
│  IAM User: Bob  │                │  IAM Role: CrossAcctRole │
│                 │ ──AssumeRole──►│  Trust Policy:           │
│                 │                │    Principal: Account A  │
│                 │◄──Temp Creds── │  Permission Policy:      │
│                 │                │    s3:GetObject           │
│  Uses temp      │                │    on bucket-B/*         │
│  creds to       │                └──────────────────────────┘
│  access S3-B    │                         │
└─────────────────┘                         ▼
                                   ┌──────────────────┐
                                   │  S3 Bucket-B     │
                                   └──────────────────┘
```

**Key points:**
- Trust policy (Account B's role): says WHO can assume.
- Permission policy (Account B's role): says WHAT they can do once assumed.
- Account A user must also have `sts:AssumeRole` permission.

---

## Diagram 2: Policy Evaluation Flow

```
Request Received
       │
       ▼
  Explicit DENY in any policy? ──YES──► DENY
       │ NO
       ▼
  SCP allows it? ──NO──► DENY
       │ YES
       ▼
  Permission Boundary allows it? ──NO──► DENY
       │ YES
       ▼
  Identity Policy allows it? ──NO──► Resource policy allows same-account?
       │                                    │ YES ──► ALLOW
       │                                    │ NO  ──► DENY
       │ YES
       ▼
  Session policy (if any) allows it? ──NO──► DENY
       │ YES
       ▼
      ALLOW
```

---

## Diagram 3: SAML Federation Flow (Console)

```
User ──(1. authenticate)──► Corporate IdP (AD/Okta)
     ◄──(2. SAML assertion)──
User ──(3. POST assertion)──► https://signin.aws.amazon.com/saml
                                          │
                               (4. sts:AssumeRoleWithSAML)
                                          │
                                         STS ──► Temp Credentials
                                          │
                               (5. Redirect to console)
                                          │
                                    AWS Console
                            (assumed the role mapped in assertion)
```

---

## Diagram 4: AWS Organizations & SCP Hierarchy

```
Root
  │  ← SCP: Deny leaving org (applied to Root = all accounts)
  ├── OU: Production
  │     │  ← SCP: Deny disabling CloudTrail
  │     ├── Account: prod-web   (inherits Root + Production SCPs)
  │     └── Account: prod-db    (inherits Root + Production SCPs)
  │
  └── OU: Development
        │  ← SCP: Deny launching instances > t3.large
        └── Account: dev        (inherits Root + Development SCPs)

Management Account: NOT subject to SCPs
```

**Remember:** SCPs limit max permissions. An Allow SCP does NOT grant permissions.
