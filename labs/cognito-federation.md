# Lab: Cognito — User Pools, Identity Pools & API Gateway Authorizer

*Phase 5 · Security · Concept: [concepts/security.md](../concepts/security.md)*

---

## Goal

Create a Cognito User Pool with a hosted UI, sign up a user, inspect the JWT tokens, then protect an API Gateway endpoint with a Cognito authorizer. Optionally explore Identity Pools for temporary AWS credentials.

---

## Architecture

```
Browser → Cognito Hosted UI → JWT (id + access tokens)
                                     │
                                     ▼
Browser → API Gateway ──► Cognito Authorizer (validates JWT)
               │
               ▼
          Lambda (returns user claims)
```

---

## Part A — Create a Cognito User Pool

- [ ] Cognito → User Pools → Create user pool.
- [ ] Sign-in: **Email** only (simplest for the lab).
- [ ] Password policy: minimum 8 characters, no special requirements (keeps lab quick).
- [ ] MFA: **No MFA** (lab only — always enable in production).
- [ ] Self-registration: **Allow** (so you can sign up via hosted UI).
- [ ] Attributes: keep defaults (email required).
- [ ] Email: **Send email with Cognito** (free tier, no SES setup needed).
- [ ] User pool name: `lab-user-pool`.
- [ ] App client name: `lab-app-client`.
- [ ] App type: **Public client** (no client secret — browser-based flow).
- [ ] Callback URL: `https://example.com/callback` (placeholder — you'll read tokens from the URL).
- [ ] OAuth flows: **Authorization code grant** (most secure).
- [ ] Scopes: `openid`, `email`, `profile`.
- [ ] Create user pool.

**Record these values:**
- [ ] User Pool ID (format: `us-east-1_xxxxxxx`)
- [ ] App Client ID (32-char string)
- [ ] Cognito domain (set one up: Actions → Create Cognito domain → e.g. `lab-pool-<random>`)

---

## Part B — Sign Up and Inspect JWT Tokens

**1. Open the hosted UI**
- [ ] Cognito → User pool → App integration → App client → View Hosted UI. Or construct the URL:
  ```
  https://<your-domain>.auth.<region>.amazoncognito.com/login?
    client_id=<app-client-id>&
    response_type=code&
    scope=openid+email+profile&
    redirect_uri=https://example.com/callback
  ```
- [ ] Click **Sign up** → enter email + password → check your email for the verification code → confirm.

**2. Sign in and capture the authorization code**
- [ ] Sign in with your new credentials.
- [ ] You'll be redirected to `https://example.com/callback?code=<auth-code>`.
- [ ] Copy the `code` value from the URL bar.

**3. Exchange code for tokens**
- [ ] In CloudShell:
  ```bash
  curl -X POST https://<your-domain>.auth.<region>.amazoncognito.com/oauth2/token \
    -H "Content-Type: application/x-www-form-urlencoded" \
    -d "grant_type=authorization_code" \
    -d "client_id=<app-client-id>" \
    -d "code=<auth-code>" \
    -d "redirect_uri=https://example.com/callback"
  ```
- [ ] Response contains: `id_token`, `access_token`, `refresh_token`.

**4. Decode the ID token (JWT)**
- [ ] Copy the `id_token` value. Go to [jwt.io](https://jwt.io) and paste it.
- [ ] Observe the payload:
  - `sub` — unique user ID
  - `email` — the email you signed up with
  - `iss` — issuer = your Cognito User Pool URL
  - `aud` — audience = your App Client ID
  - `token_use` — `id` (vs `access` for the access token)
  - `exp` — expiry (default 1 hour)

**Exam trap:** The **ID token** contains user identity claims (email, name). The **access token** controls API access (scopes). Don't confuse them.

---

## Part C — Protect an API Gateway Endpoint with Cognito

**1. Create a Lambda function**
- [ ] Lambda → Create function → `cognito-test-handler`.
- [ ] Runtime: Python 3.12.
- [ ] Code:
  ```python
  import json

  def lambda_handler(event, context):
      claims = event.get('requestContext', {}).get('authorizer', {}).get('claims', {})
      return {
          'statusCode': 200,
          'body': json.dumps({
              'message': 'Authenticated!',
              'email': claims.get('email', 'unknown'),
              'sub': claims.get('sub', 'unknown')
          })
      }
  ```
- [ ] Deploy.

**2. Create a REST API**
- [ ] API Gateway → Create API → REST API (not HTTP API for this lab).
- [ ] API name: `cognito-protected-api`.
- [ ] Create a resource: `/me`. Create a method: `GET` → Lambda → select `cognito-test-handler`.

**3. Create the Cognito authorizer**
- [ ] API Gateway → Authorizers → Create → **Cognito**.
- [ ] Name: `CognitoAuth`.
- [ ] User Pool: select `lab-user-pool`.
- [ ] Token source: `Authorization` (header name).
- [ ] Create.

**4. Attach the authorizer to the method**
- [ ] Go to `/me` GET → Method Request → Authorization → select `CognitoAuth`.
- [ ] Deploy API → Stage: `dev`.

**5. Test without a token**
- [ ] In CloudShell:
  ```bash
  curl https://<api-id>.execute-api.<region>.amazonaws.com/dev/me
  ```
  → `{"message": "Unauthorized"}` (401).

**6. Test with a valid token**
- [ ] Use the `id_token` from Part B:
  ```bash
  curl -H "Authorization: <id_token>" \
    https://<api-id>.execute-api.<region>.amazonaws.com/dev/me
  ```
  → `{"message": "Authenticated!", "email": "you@example.com", ...}`.

- [ ] Try with an expired or garbage token → `401 Unauthorized`.

---

## Part D — User Pool vs Identity Pool (Conceptual + Quick Demo)

**User Pool = authentication (who are you?)**
- Issues JWT tokens after sign-in.
- Used for API Gateway authorizers.

**Identity Pool = authorization (what can you access?)**
- Exchanges a User Pool token (or social login token) for temporary AWS credentials (STS).
- Used when browser/mobile app needs direct AWS access (e.g., upload to S3).

**Quick demo:**
- [ ] Cognito → Identity Pools → Create identity pool.
- [ ] Name: `lab-identity-pool`.
- [ ] Authentication providers → Cognito → enter your User Pool ID + App Client ID.
- [ ] Create. Note the two IAM roles created: **Authenticated role** and **Unauthenticated role**.
- [ ] Observe the Authenticated role → it has an `sts:AssumeRoleWithWebIdentity` trust policy for `cognito-identity.amazonaws.com`.
- [ ] This role is what your users get when they trade a JWT for AWS credentials.

**Exam pattern:** "Mobile app needs to upload directly to S3 after login" → Cognito User Pool (login) + Identity Pool (get temp S3 credentials).

---

## Part E — Federation with External IdP (Conceptual)

You won't set up a full SAML IdP in this lab, but understand the exam patterns:

**SAML 2.0 Federation (enterprise SSO):**
```
Corporate AD → SAML assertion → Cognito User Pool (or STS directly)
                                       → JWT or temp AWS credentials
```
- Cognito User Pool can act as a SAML Service Provider.
- Alternatively, for console access: STS `AssumeRoleWithSAML` directly.

**Social Login (Google, Facebook, Apple):**
- Cognito User Pool → Federation → Add Google as an identity provider.
- Users sign in with Google → Cognito maps Google token to a User Pool user → issues JWT.

**Exam decision tree:**
| Scenario | Solution |
|---|---|
| App login with email/password | Cognito User Pool |
| App login with Google/Facebook | Cognito User Pool + social IdP |
| Enterprise SSO to custom app | Cognito User Pool + SAML IdP |
| Enterprise SSO to AWS Console | STS AssumeRoleWithSAML (no Cognito) |
| Mobile app needs direct S3 access | User Pool + Identity Pool |
| Millions of users (not IAM users) | Cognito (IAM users max 5000) |

---

## Key Things to Observe

- User Pool = authentication → JWT tokens. Identity Pool = authorization → AWS credentials. They're complementary, not interchangeable.
- Cognito authorizer on API Gateway validates the JWT signature and expiry — no Lambda custom code needed.
- The ID token has user claims (email, sub). The access token has scopes. API Gateway Cognito authorizer uses the **ID token**.
- Federation (SAML, social) flows through the User Pool — Cognito normalizes external tokens into its own JWT format.
- Identity Pool trust policy uses `sts:AssumeRoleWithWebIdentity` — this is how temporary credentials are issued.

---

## Clean Up

- API Gateway → delete `cognito-protected-api`.
- Lambda → delete `cognito-test-handler`.
- Cognito → delete Identity Pool → delete User Pool (this also removes the app client and domain).
- IAM → delete the two Identity Pool roles (Authenticated/Unauthenticated).
