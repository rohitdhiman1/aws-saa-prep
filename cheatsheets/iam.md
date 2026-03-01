# Cheatsheet: IAM

*Fill in as you complete Week 1. Use for exam-day revision.*

---

## Key Facts

| Concept | Detail |
|---|---|
| Policy evaluation order | Explicit Deny → SCP → Boundary → Resource policy → Identity policy → Session policy |
| SCP scope | Limits all principals in account (except management account) |
| STS temp credentials | Key + Secret + **Session Token** (all 3 required) |
| AssumeRole max session | 12 hours (default 1 hour; set `sts:SetMaxSessionDuration`) |
| Permission boundary | Limits max; does not grant |
| IAM is | Global (no region) |

---

## Must-Know Distinctions

- **Multi-AZ vs Read Replica** — HA vs Scale (IAM equivalent: Groups vs Roles)
- **Trust policy** → WHO can assume | **Permission policy** → WHAT they can do
- **SCP** → cannot exceed; does NOT grant | **Identity policy** → must also allow
- **SAML 2.0** → enterprise federation | **OIDC/Web Identity** → mobile/web app users
- **IAM Identity Center** → recommended for workforce (multi-account SSO)

---

## Exam Traps (Quick List)

- SCPs don't affect management account
- Groups cannot be resource-based policy principals
- Inline policies deleted with identity; managed policies are reusable
- `aws:PrincipalOrgID` restricts to org members in resource policies
- Confused deputy → use `aws:SourceArn` / `aws:SourceAccount` conditions

---

## Related Files

`concepts/iam.md` · `labs/iam-sandbox.md` · `diagrams/iam.md` · `mock-exams/phase1.md`
