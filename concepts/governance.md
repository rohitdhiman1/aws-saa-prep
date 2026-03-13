# Governance & Compliance — AWS Config, Control Tower, Organizations

*Phase 5 · Week 10 · [roadmap day-by-day](../docs/roadmap.md)*

---

## What It Is

AWS governance services that enforce compliance, manage multi-account environments, and detect configuration drift. Organizations provides the account structure, Control Tower automates best-practice setup, and AWS Config continuously evaluates resource configurations.

---

## How It Works

### AWS Organizations (Deep Dive)

Covered briefly in `concepts/iam.md`. This section goes deeper.

#### Structure

```
Management Account (root)
  └── Root (OU)
        ├── Security OU
        │     ├── Audit account
        │     └── Log Archive account
        ├── Production OU
        │     ├── Prod-App1
        │     └── Prod-App2
        ├── Development OU
        │     ├── Dev-App1
        │     └── Dev-Sandbox
        └── Sandbox OU
              └── Sandbox-01
```

#### Service Control Policies (SCPs)

- **Deny-list strategy** (recommended) — FullAWSAccess SCP attached by default; add deny rules for what you want to block.
- **Allow-list strategy** — remove FullAWSAccess; explicitly allow only permitted actions.
- SCPs do **not** apply to the management account.
- SCPs do **not** grant permissions — they set maximum permission boundaries.
- SCP + IAM policy intersection = effective permission. Both must allow.

#### Key Features

| Feature | Detail |
|---|---|
| **Consolidated billing** | Single payment, volume discounts (S3, EC2 RI sharing) |
| **RI/Savings Plan sharing** | RIs and Savings Plans are shared across all accounts (can be disabled) |
| **Tag policies** | Enforce consistent tagging across the org |
| **Backup policies** | Centrally define backup plans |
| **AI opt-out policies** | Opt out of AWS AI service data usage |
| **Delegated administrator** | Delegate service admin to a member account (GuardDuty, Security Hub, etc.) |

---

### Control Tower

Automated landing zone setup for multi-account AWS environments. Built on top of Organizations, CloudFormation, and IAM Identity Center.

#### Key Concepts

- **Landing zone** — a pre-configured multi-account environment following AWS best practices.
- **Account Factory** — self-service provisioning of new AWS accounts with pre-approved configurations.
- **Guardrails (Controls)** — governance rules applied to OUs:

| Guardrail Type | Behaviour | Example |
|---|---|---|
| **Preventive** | SCP-based; blocks non-compliant actions | "Disallow changes to CloudTrail" |
| **Detective** | AWS Config rule; detects non-compliance | "Detect S3 buckets without encryption" |
| **Proactive** | CloudFormation hooks; block before deployment | "Block EC2 without IMDSv2" |

| Guardrail Level | Meaning |
|---|---|
| **Mandatory** | Always on; cannot be disabled (e.g., disallow root access keys) |
| **Strongly recommended** | AWS best practices; can be opted out |
| **Elective** | Optional; enterprise-specific |

#### What Control Tower Sets Up Automatically

1. **Shared accounts:** Management account, Audit account, Log Archive account.
2. **CloudTrail** — organisation trail to Log Archive S3 bucket.
3. **AWS Config** — enabled in all enrolled accounts.
4. **IAM Identity Center** — SSO for all accounts.
5. **Service Control Policies** — guardrails applied to OUs.
6. **VPC** — default VPC configuration for new accounts.

#### Control Tower vs Manual Organizations Setup

| Capability | Control Tower | Manual Org |
|---|---|---|
| Multi-account setup | Automated | Manual |
| Guardrails (SCPs + Config rules) | Pre-packaged, one-click | Write your own |
| Account vending | Account Factory (self-service) | Manual `CreateAccount` API |
| SSO | Integrated (IAM Identity Center) | Set up separately |
| Baseline compliance | Automatic | Configure individually |
| Customisation | Limited (Customizations for Control Tower — CfCT) | Full flexibility |

---

### AWS Config

Continuously **records and evaluates** resource configurations against desired settings.

#### How It Works

1. **Configuration Recorder** — captures resource configuration changes (every change or periodic snapshots).
2. **Configuration Items** — point-in-time record of a resource's configuration, relationships, and metadata.
3. **Config Rules** — evaluate whether resources comply with desired configuration.
4. **Conformance Packs** — collection of Config rules + remediation actions as a single unit. Templates for PCI-DSS, HIPAA, CIS, etc.

#### Config Rules

| Type | How | Example |
|---|---|---|
| **AWS Managed Rules** | Pre-built by AWS (300+) | `s3-bucket-versioning-enabled`, `ec2-instance-no-public-ip`, `rds-instance-public-access-check` |
| **Custom rules** | Lambda function you write | Check if EC2 tags include "CostCenter" |
| **Trigger types** | Configuration change or periodic (1h, 3h, 6h, 12h, 24h) | On SG change → evaluate immediately |

#### Remediation

- **Manual remediation** — view non-compliant resources; fix manually.
- **Automatic remediation** — Config invokes **SSM Automation** document to fix. Examples:
  - Non-compliant S3 bucket → auto-enable versioning
  - Non-compliant SG → auto-remove rule allowing `0.0.0.0/0` on port 22
- **Remediation retries** — configurable retry count.

#### Config Aggregator

- Aggregate Config data from **multiple accounts and regions** into one aggregator account.
- View compliance across the entire organisation in one place.
- Does not push rules — each account has its own rules; the aggregator just collects results.

#### Config vs CloudTrail

| Feature | AWS Config | CloudTrail |
|---|---|---|
| Focus | Resource **configuration state** | API **call history** |
| Question answered | "Is this resource compliant?" | "Who did what and when?" |
| Data | Current + historical config snapshots | API event logs |
| Rules/Evaluation | Yes (Config Rules) | No (just logs) |
| Use case | Compliance, drift, baseline | Audit, forensics, security |

---

### Service Catalog

Pre-approved CloudFormation templates that users can self-service deploy.

- **Portfolio** — collection of products (templates), shared with specific IAM users/roles/groups.
- **Product** — a CloudFormation template with versioning.
- **Constraint** — rules on how products can be launched (e.g., allowed instance types, required tags).
- **Launch constraint** — IAM role that Service Catalog assumes to launch the product (users don't need direct resource permissions).

> **Exam tip:** Service Catalog appears when the question mentions "self-service provisioning with governance" or "limit what resources developers can create."

---

### Trusted Advisor

Automated best-practice checks across 5 categories:

| Category | Example checks |
|---|---|
| **Cost Optimisation** | Idle RDS instances, underutilised EC2, unassociated EIPs |
| **Performance** | High-utilisation EC2, CloudFront config |
| **Security** | Open SG ports (0.0.0.0/0 on 22/3389), MFA on root, IAM access keys |
| **Fault Tolerance** | Multi-AZ RDS, ASG across AZs, EBS snapshots |
| **Service Limits** | VPCs per region, EIPs, IAM roles |

- **Basic/Developer support:** 7 core checks (S3 bucket permissions, SG ports, IAM, MFA, EBS snapshots, RDS snapshots, service limits).
- **Business/Enterprise support:** All checks + API access + CloudWatch integration.
- **Programmatic access** — `support:DescribeTrustedAdvisorChecks` API (Business+ support required).

---

## Key Configs & Limits

| Item | Limit |
|---|---|
| Max accounts per organisation | Unlimited (soft limit starts at 10) |
| Max SCPs per account | 5 (direct attach) |
| SCP max size | 5,120 characters |
| Max Config rules per account per region | 400 (soft limit) |
| Config rule evaluation throttle | 25 evaluations/second |
| Control Tower max OUs (nested) | 5 levels |
| Trusted Advisor full checks | Business or Enterprise support plan required |
| Service Catalog max products per portfolio | 100 |

---

## When To Use

| Scenario | Solution |
|---|---|
| Manage 50+ AWS accounts centrally | AWS Organizations |
| Block all accounts from disabling CloudTrail | SCP (preventive guardrail) |
| Auto-provision new accounts with baselines | Control Tower Account Factory |
| Detect S3 buckets without encryption | AWS Config rule (`s3-bucket-server-side-encryption-enabled`) |
| Auto-fix non-compliant security groups | Config rule + SSM Automation remediation |
| Compliance dashboard across all accounts | Config Aggregator |
| Let devs deploy only approved architectures | Service Catalog |
| Find underutilised EC2 instances | Trusted Advisor (Business+ support) |
| Enforce consistent tagging | Organizations tag policies + Config rules |
| Audit resource config history | AWS Config (configuration timeline) |

---

## Exam Traps

- **SCPs do NOT grant permissions** — they only limit what IAM policies can grant. An SCP Allow + no IAM policy = no access.
- **SCPs do NOT apply to the management account** — the management account always has full permissions regardless of SCPs.
- **Config records configuration, not API calls** — "Who changed it?" = CloudTrail. "What is the current state?" = Config.
- **Config Aggregator does NOT push rules** — it only collects compliance data. Each account manages its own rules.
- **Control Tower ≠ Organisations** — Control Tower is built ON TOP of Organisations. You can use Organisations without Control Tower, but not the reverse.
- **Control Tower guardrails are either preventive (SCP) or detective (Config)** — know which is which. "Prevent" = SCP. "Detect" = Config rule.
- **Trusted Advisor full checks require Business or Enterprise support** — Free/Developer tier only gets 7 core checks.
- **Service Catalog launch constraint** — users deploy resources without needing direct IAM permissions to those resources. The launch role has the permissions.
- **Config is NOT free** — $0.003 per configuration item recorded + rule evaluations. Can be significant at scale.
- **Drift ≠ non-compliance** — CloudFormation drift = template divergence. Config non-compliance = rule violation. Different concepts, different tools.

---

## Related Files

| File | Purpose |
|---|---|
| `concepts/iam.md` | IAM policies, SCPs, Organizations basics |
| `concepts/security.md` | Security Hub aggregates Config findings |
| `concepts/infrastructure-as-code.md` | CloudFormation (Control Tower uses StackSets) |
| `concepts/monitoring-logging.md` | CloudTrail vs Config distinction |
| `cheatsheets/security.md` | Quick-reference for compliance services |
| `labs/iam-sandbox.md` | SCPs in practice |
