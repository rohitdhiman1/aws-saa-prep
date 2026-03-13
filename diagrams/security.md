# Diagrams: Security

*Draw these on paper or in draw.io as part of Week 10 study.*

---

## Diagram 1: Envelope Encryption (KMS)

```
Application wants to encrypt data
            │
            ▼
   Call KMS: GenerateDataKey (CMK-ID)
            │
            ▼
   KMS returns:
     ├── Plaintext data key (used to encrypt data locally)
     └── Encrypted data key (encrypted under CMK)
            │
            ▼
   Encrypt data locally with plaintext data key (AES-256, fast)
            │
            ▼
   Discard plaintext data key
            │
            ▼
   Store: [Encrypted Data] + [Encrypted Data Key]

TO DECRYPT:
   Call KMS: Decrypt(Encrypted Data Key) → Plaintext data key
   Decrypt data locally with plaintext data key
```

---

## Diagram 2: GuardDuty → Automated Remediation

```
Data Sources:
  VPC Flow Logs │
  CloudTrail    │──► GuardDuty (ML analysis) ──► Finding (High severity)
  DNS Logs      │                                        │
  S3 Events     │                                        ▼
                                                  EventBridge Rule
                                                  (on finding type)
                                                        │
                                                        ▼
                                                   Lambda Function
                                                   - Isolate EC2 (SG change)
                                                   - Revoke IAM credentials
                                                   - Notify Security Hub
```

---

## Diagram 3: Secrets Manager Rotation Flow

```
App calls Secrets Manager: GetSecretValue("db/password")
        │
        ▼
  Secrets Manager returns current secret (AWSCURRENT)
        │
ROTATION TRIGGERED (scheduled):
        │
        ▼
  Lambda (rotation function):
    1. createSecret → generate new password (AWSPENDING)
    2. setSecret → update DB with new password
    3. testSecret → verify new password works
    4. finishSecret → promote AWSPENDING → AWSCURRENT
                                        AWSCURRENT → AWSPREVIOUS
        │
        ▼
App next call → gets new password transparently
```

---

## Diagram 4: WAF + Shield Layers

```
Internet traffic
      │
      ▼
Shield Standard (always on, L3/L4 DDoS, free)
      │
      ▼
CloudFront (edge)
      │
WAF Web ACL:
  Rule 1: Block IPs in IP Set
  Rule 2: Rate limit > 100 req/5min
  Rule 3: AWS Managed Rules (OWASP Top 10)
  Rule 4: Geo-block specific countries
      │
      ▼
ALB / API Gateway
      │
      ▼
Application (EC2 / Lambda)
```

---

## Diagram 5: Cognito — User Pools vs Identity Pools

```
                        AUTHENTICATION (Who are you?)
                        ──────────────────────────────
User (browser/mobile)
      │
      ▼
Cognito User Pool
  - Sign-up / Sign-in (hosted UI or custom)
  - MFA, password policies
  - Social IdP (Google, Facebook, Apple)
  - SAML / OIDC federation (corporate AD)
      │
      ▼
Returns JWT tokens:
  ├── ID Token     (user attributes: email, name, groups)
  ├── Access Token (scopes: API permissions)
  └── Refresh Token (get new tokens without re-login)
      │
      ├──► API Gateway (Cognito Authorizer validates JWT)
      │         │
      │         ▼
      │    Lambda / backend
      │
      ▼
                        AUTHORISATION (What can you access?)
                        ────────────────────────────────────
Cognito Identity Pool (Federated Identities)
  - Exchanges JWT (from User Pool) or IdP token
    for temporary AWS credentials (STS)
  - Maps to IAM Role:
      ├── Authenticated role   → scoped AWS access
      └── Unauthenticated role → guest access (optional)
      │
      ▼
Temporary AWS Credentials (AccessKeyId, SecretKey, SessionToken)
      │
      ▼
Direct AWS access: S3, DynamoDB, etc. (from mobile/browser)
```

### Quick Decision

```
"Users need to sign in"
  └── Cognito User Pool (authentication → JWT)

"App needs AWS credentials (S3, DynamoDB from client)"
  └── Cognito Identity Pool (authorisation → STS temp creds)

"Both sign-in AND AWS access"
  └── User Pool → Identity Pool (JWT → temp creds)

"Corporate SSO (SAML) to AWS Console"
  └── IAM Identity Center (not Cognito)
```
