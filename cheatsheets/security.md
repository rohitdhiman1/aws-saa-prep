# Cheatsheet: Security & Compliance

*Fill in as you complete Week 10. Use for exam-day revision.*

---

## Key Facts

| Concept | Detail |
|---|---|
| KMS CMK cost | $1/month + API calls |
| KMS AWS-managed rotation | Annual (automatic) |
| Secrets Manager cost | $0.40/secret/month |
| Secrets Manager max size | 64 KB |
| SSM Parameter Store (Standard) | Free, 4 KB |
| WAF Web ACL max WCU | 1,500 |
| Shield Standard | Free, automatic, L3/L4 |
| Shield Advanced | $3,000/month/org |
| GuardDuty finding retention | 90 days |

---

## Must-Know Distinctions

- **Secrets Manager** → auto-rotation, $0.40/secret | **SSM Parameter Store** → no auto-rotation, free/cheap
- **GuardDuty** → threat detection (finds threats) | **Inspector** → vulnerability assessment (finds CVEs)
- **Macie** → PII/sensitive data in S3 | **GuardDuty** → network/API threats
- **Shield Standard** → automatic, free | **Shield Advanced** → paid, DRT access, L7 protection
- **WAF on CloudFront** → global scope | **WAF on ALB/API GW** → regional scope
- **KMS key policy** → required (IAM alone is not enough) | **Grants** → for AWS services

---

## Exam Traps (Quick List)

- GuardDuty does NOT block traffic (detection only; use Lambda/NACL to respond)
- Inspector needs SSM Agent on EC2
- Macie only covers S3 (not RDS, DynamoDB)
- Envelope encryption: data never goes to KMS; only data key does
- KMS multi-region keys: same key material, different ARNs per region
- Firewall Manager requires AWS Organizations

---

## Related Files

`concepts/security.md` · `concepts/iam.md` · `diagrams/security.md` · `mock-exams/phase5.md`
