# Security & Compliance — KMS, Secrets Manager, WAF, Shield, GuardDuty

*Phase 5 · Week 10 · [roadmap day-by-day](../docs/roadmap.md)*

---

## What It Is

AWS security services covering: key management (KMS), secrets rotation (Secrets Manager), network/app protection (WAF, Shield), threat detection (GuardDuty), vulnerability assessment (Inspector), data classification (Macie), and posture management (Security Hub).

---

## How It Works

### KMS — Key Management Service

- Create and control **cryptographic keys** used to encrypt AWS services and your data.
- **FIPS 140-2** validated. Audit all key usage via **CloudTrail**.

#### Key Types

| Type | Managed by | Cost | Rotation |
|---|---|---|---|
| **AWS-managed keys** (`aws/s3`, `aws/ebs`, etc.) | AWS | Free | Auto-rotated every year |
| **Customer-managed CMK** | You | $1/month + API calls | Optional auto-rotation (annual); or manual |
| **AWS-owned keys** | AWS (shared) | Free | Managed by AWS |

- **Symmetric CMK** — same key for encrypt/decrypt (AES-256). Used by most AWS services.
- **Asymmetric CMK** — RSA or ECC key pair (public/private). Public key can be shared externally. Used for: digital signing, encryption outside AWS.
- **HMAC keys** — for message authentication codes.

#### Key Policies & Grants

- Every CMK has a **key policy** (resource-based policy). Must explicitly allow IAM to delegate access; unlike IAM policies alone, key policy is required.
- **Grants** — delegate key usage to AWS services (e.g., EC2 to use the key for EBS). Temporary and programmatic.
- **ViaService condition** — restrict key usage to a specific AWS service only.

#### Envelope Encryption

AWS services use envelope encryption:
1. KMS generates a **data key** (plaintext + encrypted copy).
2. Data is encrypted with the plaintext data key (locally, fast).
3. Encrypted data key is stored alongside encrypted data.
4. On decrypt: call KMS to decrypt the data key → decrypt data locally.

This avoids sending all data through KMS (bandwidth/latency); KMS only handles the small data key.

#### Key Policies & Access

- **`kms:Decrypt`** — required to decrypt data.
- **`kms:GenerateDataKey`** — required for services that encrypt data.
- **`kms:CreateGrant`** — allow services to create grants on your behalf.

#### KMS Multi-Region Keys

- Replicate key to other regions; same key material, different ARN.
- Use for: cross-region encryption/decryption without re-encrypting. Used with DynamoDB Global Tables, S3 CRR with SSE-KMS.

---

### Secrets Manager

Store, rotate, and retrieve secrets (DB passwords, API keys, OAuth tokens).

- **Auto-rotation** — uses Lambda to rotate secrets automatically (built-in templates for RDS, Redshift, DocumentDB, and custom Lambda for others). Rotation schedule: daily to annually.
- **Versioning** — `AWSCURRENT` (active), `AWSPENDING` (during rotation), `AWSPREVIOUS` (previous version).
- Integrated with RDS, Aurora, Redshift, DocumentDB — auto-updates the DB password AND the secret.
- **Cross-account** — share secrets via resource-based policy.
- Cost: $0.40/secret/month + $0.05 per 10,000 API calls.

#### Secrets Manager vs SSM Parameter Store

| Feature | Secrets Manager | SSM Parameter Store |
|---|---|---|
| Auto-rotation | Yes (built-in) | No (manual or custom Lambda) |
| Cost | $0.40/secret/month | Free (Standard) / $0.05/advanced param/month |
| Secret size | Up to 64 KB | 4 KB (Standard), 8 KB (Advanced) |
| Cross-account | Yes | Limited |
| Versioning | Yes | Yes |
| Use case | DB passwords, API keys needing rotation | Config params, non-sensitive or less sensitive values |
| Encryption | KMS (required) | KMS (optional for SecureString) |

> **Rule of thumb:** If it needs auto-rotation → Secrets Manager. If it's config or non-sensitive → SSM Parameter Store.

---

### WAF — Web Application Firewall

- Protects web applications from common exploits (SQLi, XSS, bad bots).
- Attach to: **CloudFront, ALB, API Gateway, AppSync, Cognito User Pool**.
- **Web ACL** — collection of rules evaluated in order.
- **Rules:**
  - **Managed rule groups** — AWS or marketplace; pre-built rules for OWASP Top 10, bots, IPs.
  - **Custom rules** — IP match, geo match, string match, regex, size constraint, rate-based.
- **Rate-based rules** — block IP if it exceeds N requests in 5 minutes.
- **Rule actions:** Allow, Block, Count (monitoring without blocking), CAPTCHA, Challenge.
- WAF **Capacity Units (WCU)** — each rule consumes WCU; max 1,500 per Web ACL.

---

### Shield

DDoS protection.

| Tier | Cost | Protection |
|---|---|---|
| **Standard** | Free (automatic) | L3/L4 protection on all AWS resources |
| **Advanced** | $3,000/month (org-level) | L3/L4/L7 protection, DDoS Response Team (DRT), cost protection, real-time metrics |

- Shield Advanced is applied to: CloudFront, ALB, NLB, EIP, Route 53.
- **Firewall Manager** — centrally manage WAF rules and Shield Advanced across accounts/OUs in an Organisation.

---

### GuardDuty

Managed **threat detection** — continuously analyses:
- **VPC Flow Logs** — network anomalies.
- **CloudTrail logs** — unusual API calls.
- **DNS logs** — malware communication.
- **S3 data events** — unusual S3 access.
- **EKS audit logs, Lambda network activity, RDS login events** — extended protection.

- No agents; no infrastructure; fully managed. Enable with one click.
- **Findings** — categorised by severity (Low/Medium/High). Examples: unusual port scanning, known bad IP, crypto-mining, credential exfiltration.
- **Automated response** — EventBridge rule on GuardDuty findings → Lambda to remediate.
- **Multi-account** — designate one account as GuardDuty administrator; aggregate findings from member accounts.

---

### Inspector

**Automated vulnerability assessment** for:
- **EC2 instances** — OS vulnerabilities, unintended network exposure. Uses SSM Agent.
- **Lambda functions** — package vulnerabilities.
- **Container images in ECR** — scan on push or continuously.

- Continuously assesses; integrates with Security Hub.
- Generates findings with CVE details and severity (CVSS score).

---

### Macie

**ML-powered sensitive data discovery** in S3.

- Identifies: PII (names, addresses, credit card numbers, SSNs), credentials, financial data.
- Generates findings in Security Hub and EventBridge.
- Use for: GDPR compliance, data governance, accidental exposure detection.

---

### Security Hub

**Central security posture management** — aggregates findings from GuardDuty, Inspector, Macie, IAM Access Analyzer, Firewall Manager, and third-party tools.

- Runs **automated security checks** against AWS security best practices and compliance frameworks (CIS, PCI-DSS, SOC2).
- **Security score** — percentage of passed checks.
- Multi-account: designate administrator; members send findings.
- **Integrated response** — EventBridge rules on Security Hub findings → Lambda/Step Functions for automated remediation.

---

## Key Configs & Limits

| Item | Limit |
|---|---|
| KMS CMK cost | $1/month/key + API calls |
| KMS request rate default | 5,500–30,000 requests/s (varies by region) |
| Secrets Manager max secret size | 64 KB |
| Secrets Manager rotation | Lambda function; schedule configurable |
| WAF Web ACL max WCU | 1,500 |
| WAF rate-based rule window | 5 minutes |
| Shield Advanced cost | $3,000/month per organisation |
| GuardDuty findings retention | 90 days |

---

## When To Use

| Scenario | Solution |
|---|---|
| Encrypt EBS/S3/RDS | KMS CMK (SSE-KMS) |
| Auto-rotate DB password | Secrets Manager |
| Store non-sensitive config | SSM Parameter Store |
| Block SQLi/XSS at ALB | WAF |
| DDoS protection + DRT access | Shield Advanced |
| Detect crypto-mining on EC2 | GuardDuty |
| Find unpatched CVEs on EC2 | Inspector |
| Find PII in S3 buckets | Macie |
| Centralise all security findings | Security Hub |
| Enforce WAF rules across org | Firewall Manager |

---

## Exam Traps

- **KMS key policy is always required** — IAM policy alone is not sufficient; key policy must also allow.
- **AWS-managed keys auto-rotate annually** — you cannot change this. Customer-managed CMKs: optional.
- **Secrets Manager ≠ SSM Parameter Store** — Secrets Manager auto-rotates; Parameter Store does not.
- **GuardDuty does not block traffic** — it detects threats and generates findings. Use Lambda/Network Firewall to respond.
- **Inspector uses SSM Agent for EC2** — if agent isn't running, Inspector cannot assess the instance.
- **Macie only covers S3** — it does not scan RDS, DynamoDB, or other data stores.
- **Shield Standard is automatic and free** — always on for all AWS customers; no setup.
- **WAF on CloudFront = global scope** — WAF attached to regional services (ALB, API GW) uses regional scope.
- **Firewall Manager requires AWS Organizations** — and a designated Firewall Manager administrator account.
- **Envelope encryption** — AWS services never send your data to KMS. Only the data key is exchanged with KMS.

---

## Related Files

| File | Purpose |
|---|---|
| `diagrams/security.md` | Visual: envelope encryption flow, GuardDuty → EventBridge → Lambda |
| `cheatsheets/security.md` | Quick-reference card |
| `mock-exams/phase5.md` | Security + analytics practice questions |
| `concepts/iam.md` | Foundation: IAM policies, STS, identity |
