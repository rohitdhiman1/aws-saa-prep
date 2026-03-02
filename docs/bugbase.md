# Bug Base — Knowledge Gaps Tracker

_Log anything you got wrong in practice questions or found confusing. Revisit before the exam._

## Format

```
### [Service / Topic]
- **Gap:** what you got wrong or didn't know
- **Correct understanding:** the right answer / concept
- **Source:** where you found the answer (AWS docs, course, etc.)
- **Status:** open | resolved
```

---

<!-- Add entries below as you study -->

### Cost Optimization (Domain 4)
- **Gap:** Domain 4 (cost-optimized architectures) was only 60% covered by labs — weakest exam domain.
- **Correct understanding:** Spot Instances (capacityOptimized strategy), RI vs Savings Plans, S3 Intelligent-Tiering, NAT GW cost avoidance via VPC Gateway Endpoints, and data transfer costs are all testable. Added dedicated `labs/cost-optimization.md`.
- **Source:** Lab gap analysis, this session
- **Status:** open — lab written but not yet completed hands-on

### Cognito (User Pool vs Identity Pool)
- **Gap:** Zero coverage on Cognito — worth ~3-4 exam questions.
- **Correct understanding:** User Pool = authentication (JWT tokens). Identity Pool = authorization (temporary AWS credentials via STS). API Gateway uses Cognito authorizer with the ID token (not access token). For "mobile app uploads directly to S3 after login" → User Pool + Identity Pool.
- **Source:** Lab gap analysis, this session
- **Status:** open — lab written but not yet completed hands-on
