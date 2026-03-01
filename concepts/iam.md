# Identity and Access Management (IAM)

*Phase 1 · Week 1 · [roadmap day-by-day](../docs/roadmap.md)*

---

## What It Is

IAM is the global, free AWS service that controls **who** can do **what** on **which** resources.
It has no concept of a region — identities and policies are global.

Key primitives:

| Primitive | What it represents |
|---|---|
| **User** | A long-term identity for a person or application (has permanent credentials) |
| **Group** | A collection of users; policies attached here apply to all members |
| **Role** | A short-term identity assumed by users, services, or cross-account principals |
| **Policy** | A JSON document that grants or denies permissions |

---

## How It Works

### Users vs Groups vs Roles

- **Users** — created for humans or apps needing permanent credentials (access key + secret, or console password). Avoid sharing users across people.
- **Groups** — organisational convenience only; cannot be nested. Attach policies here instead of to individual users.
- **Roles** — meant to be *assumed*, not logged in to. No permanent credentials — they issue **temporary security credentials** via STS. Use roles for: EC2 instance profiles, Lambda, cross-account access, federation.

### Policy Types

| Type | Where attached | Evaluated |
|---|---|---|
| Identity-based | User / Group / Role | Yes |
| Resource-based | S3 bucket, SQS queue, KMS key, etc. | Yes |
| Permission boundary | User or Role | Limits max permissions (AND with identity policy) |
| SCP (Service Control Policy) | AWS Organizations OU / Account | Sets ceiling for entire account |
| Session policy | AssumeRole call | Limits session below role's permissions |
| ACL | S3, VPC | Legacy; prefer bucket policies |

### Policy JSON Structure

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",          // or "Deny"
      "Action": "s3:GetObject",   // or ["s3:*"]  or "*"
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {              // optional
        "StringEquals": { "aws:RequestedRegion": "us-east-1" }
      }
    }
  ]
}
```

### Policy Evaluation Logic (MEMORISE THIS)

1. **Explicit Deny** — if any policy (any type) says Deny → request denied, full stop.
2. **SCP check** — if Organizations SCP does not allow the action → denied.
3. **Permission boundary check** — if boundary does not include the action → denied.
4. **Resource-based policy** — if present and grants access, may allow even without identity policy (same-account scenario).
5. **Identity-based policy** — must Allow the action.
6. **Session policy** — if present, must also Allow.
7. **Default implicit Deny** — anything not explicitly allowed is denied.

> **Rule of thumb:** Explicit Deny always wins. Everything else must explicitly allow at every applicable layer.

### Security Token Service (STS)

STS issues **temporary credentials** (access key + secret + session token, valid 15 min – 36 hours).

Key API calls:

| API | Use case |
|---|---|
| `AssumeRole` | Switch to a different role (same or cross-account) |
| `AssumeRoleWithSAML` | Federation via SAML 2.0 IdP |
| `AssumeRoleWithWebIdentity` | Federation via OIDC (Cognito, Google, etc.) |
| `GetSessionToken` | MFA-protected API access for IAM user |
| `GetFederationToken` | Broker-based federation (legacy) |

**Cross-account role assumption flow:**
1. Account A creates a role with a **trust policy** naming Account B as principal.
2. User in Account B calls `sts:AssumeRole` targeting the Account A role ARN.
3. STS returns temporary credentials scoped to that role.
4. User uses those credentials to access Account A resources.

**Session policies** — passed in the `AssumeRole` call to further restrict what the session can do (cannot grant more than the role allows).

### Identity Federation

Goal: let identities from an **external IdP** (corporate AD, Google, GitHub, Cognito) access AWS without creating IAM users.

| Method | Protocol | Use case |
|---|---|---|
| **SAML 2.0** | XML assertion | Corporate IdP (Active Directory, Okta, ADFS) → AWS console or API |
| **Web Identity (OIDC)** | JWT token | Mobile/web app users (Cognito, Google, Facebook) → AWS APIs |
| **IAM Identity Center (SSO)** | SAML / OIDC under the hood | Centralised SSO for AWS accounts + SAML apps; recommended for workforce |
| **AWS Cognito** | OIDC | User pools (auth) + identity pools (AWS credentials for end-users) |

**SAML 2.0 federation flow (console):**
1. User authenticates to corporate IdP.
2. IdP issues a SAML assertion naming the user and their groups.
3. Browser posts assertion to `https://signin.aws.amazon.com/saml`.
4. AWS STS calls `AssumeRoleWithSAML`, returns temp credentials.
5. User lands in the AWS console assuming the mapped role.

**IAM Identity Center** (formerly AWS SSO) — preferred for multi-account orgs:
- Connects to Active Directory or built-in identity store.
- Uses **Permission Sets** (collections of policies) mapped to accounts/OUs.
- Single place to manage who accesses which account with what permissions.

### AWS Organizations

Manage multiple AWS accounts under one umbrella.

```
Root
└── Management Account (payer)
    ├── OU: Production
    │   ├── Account: prod-web
    │   └── Account: prod-db
    └── OU: Development
        └── Account: dev
```

| Concept | Details |
|---|---|
| **Management account** | Payer and policy administrator; cannot be restricted by SCPs |
| **Member account** | Subject to SCPs; joined by invitation or created via Organizations |
| **OU (Org Unit)** | Logical grouping; SCPs attach here and cascade down |
| **SCP** | JSON policy that sets the **maximum** permissions for an account or OU. Does NOT grant permissions — it restricts them. Affects all principals in the account including root. |
| **Consolidated billing** | One bill for all accounts; volume discounts pooled across accounts |

**SCP example — deny leaving the org:**
```json
{
  "Effect": "Deny",
  "Action": "organizations:LeaveOrganization",
  "Resource": "*"
}
```

**Important:** SCPs do NOT affect the management account. Service-linked roles are not restricted by SCPs.

---

## Key Configs & Limits

| Item | Limit |
|---|---|
| IAM users per account | 5,000 |
| IAM groups per account | 300 |
| IAM roles per account | 1,000 |
| Managed policies per user/role/group | 10 |
| Inline policy size | 2,048 characters |
| Managed policy size | 6,144 characters |
| Role session duration | 1 hour default; up to 12 hours (or 36 hours with `--duration-seconds` on some roles) |
| STS temp credentials minimum | 15 minutes |
| STS temp credentials maximum | 36 hours (for `GetFederationToken`) |
| Password policy enforcement | Per account, configurable |

---

## When To Use

| Scenario | Use this |
|---|---|
| EC2 needs to call S3 | IAM Role → attach as instance profile |
| Lambda needs DynamoDB access | IAM Role → Lambda execution role |
| Corp users need AWS console access | IAM Identity Center + SAML federation |
| Mobile app users need S3 access | Cognito Identity Pool → `AssumeRoleWithWebIdentity` |
| Restrict all accounts from leaving the org | SCP on Root |
| Restrict a dev OU from launching expensive instances | SCP on OU |
| Delegate admin to a team but cap their max perms | Permission boundary on their role |
| API caller needs MFA before destructive actions | `GetSessionToken` + `aws:MultiFactorAuthPresent` condition |

---

## Exam Traps

- **SCPs do not grant permissions** — even with `Allow *` SCP, the identity-based policy must also allow.
- **SCPs do not affect the management account** — don't try to lock down the payer with an SCP.
- **Groups cannot be principals** — you cannot reference a group in a resource-based policy.
- **Role session duration** — max 12 hours for most roles; `sts:SetMaxSessionDuration` must be set on the role first.
- **Permission boundary limits, does not grant** — a boundary of `s3:*` doesn't give S3 access; the identity policy must also have `s3:*`.
- **Inline vs managed policies** — inline policies are deleted with the identity; managed policies are reusable.
- **`aws:PrincipalOrgID` condition** — can restrict resource-based policies to only principals inside the org.
- **Trust policy ≠ permission policy** — trust policy says who can assume the role; permission policy says what they can do once assumed.
- **Confused deputy** — when a service (not a user) assumes a role, use `aws:SourceArn` or `aws:SourceAccount` conditions to prevent confused deputy attacks.

---

## Related Files

| File | Purpose |
|---|---|
| `labs/iam-sandbox.md` | Hands-on: create users, groups, roles, test policy evaluation |
| `diagrams/iam.md` | Visual: cross-account flow, federation flow, SCP hierarchy |
| `cheatsheets/iam.md` | Quick-reference card for exam day |
| `mock-exams/phase1.md` | Practice questions covering IAM + VPC |
| `concepts/vpc-basics.md` | Next concept file (Week 2) |
