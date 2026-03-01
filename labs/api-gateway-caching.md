# Lab: API Gateway — REST API, Caching, Throttling & Usage Plans

*Phase 2 · Compute · Concept: [concepts/compute.md](../concepts/compute.md)*

---

## Goal

Build a REST API with Lambda integration, enable API caching, set up throttling via usage plans and API keys, and observe how each feature affects latency and availability.

---

## Architecture

```
Client → API Gateway (REST) → Lambda
              │
              ├── Stage: dev (no cache)
              └── Stage: prod (cache enabled, 0.5 GB, 60s TTL)

Usage Plan → API Key → throttle: 5 req/sec, burst 10, quota 100/day
```

---

## Part A — Create the Lambda Backend

- [ ] Lambda → Create function → `api-backend`.
- [ ] Runtime: Python 3.12.
- [ ] Code:
  ```python
  import json
  import time

  def lambda_handler(event, context):
      # Simulate a slow database call
      time.sleep(1)
      return {
          'statusCode': 200,
          'body': json.dumps({
              'message': 'Hello from Lambda',
              'timestamp': int(time.time()),
              'path': event.get('path', '/')
          })
      }
  ```
- [ ] Deploy. This deliberately sleeps 1 second so you can observe caching.

---

## Part B — Create a REST API with Two Stages

**1. Create the API**
- [ ] API Gateway → Create API → **REST API** (not HTTP API).
- [ ] API name: `lab-api`. Endpoint type: **Regional**.

**2. Create resources and methods**
- [ ] Create resource: `/items`.
- [ ] Create method: `GET` on `/items` → Integration type: Lambda → select `api-backend`.
- [ ] Create resource: `/items/{id}` (with path parameter).
- [ ] Create method: `GET` on `/items/{id}` → Lambda → `api-backend`.

**3. Deploy to two stages**
- [ ] Deploy API → New stage: `dev`. Deploy.
- [ ] Deploy API → New stage: `prod`. Deploy.

**4. Test baseline latency**
- [ ] In CloudShell:
  ```bash
  # dev stage — no cache
  time curl -s https://<api-id>.execute-api.<region>.amazonaws.com/dev/items
  ```
  → ~1.1–1.3 seconds (Lambda cold start + 1s sleep).
- [ ] Call it a few more times → ~1.0–1.1s (warm Lambda, but sleep is always there).

---

## Part C — Enable API Caching

**1. Turn on caching for the prod stage**
- [ ] API Gateway → Stages → `prod` → Stage editor → Cache settings.
- [ ] Enable API cache: **Yes**.
- [ ] Cache capacity: **0.5 GB** (smallest — note: caching costs money, ~$0.02/hr for 0.5 GB).
- [ ] Cache TTL: **60** seconds.
- [ ] Save. Wait ~3–4 minutes for the cache to provision.

**2. Test cached responses**
- [ ] First call (cache miss):
  ```bash
  time curl -s https://<api-id>.execute-api.<region>.amazonaws.com/prod/items
  ```
  → ~1.0s (backend called, response cached).

- [ ] Second call (cache hit):
  ```bash
  time curl -s https://<api-id>.execute-api.<region>.amazonaws.com/prod/items
  ```
  → ~50–100ms (response served from cache, Lambda NOT invoked).

- [ ] Verify: both responses have the **same timestamp** (cached response is a replay).

**3. Cache key includes the path**
- [ ] Call `/prod/items/123` → ~1.0s (different cache key, miss).
- [ ] Call `/prod/items/123` again → ~50ms (hit on that specific path).
- [ ] Call `/prod/items/456` → ~1.0s (different path parameter, different cache key, miss).

**4. Cache invalidation**
- [ ] Pass `Cache-Control: max-age=0` header:
  ```bash
  curl -s -H "Cache-Control: max-age=0" \
    https://<api-id>.execute-api.<region>.amazonaws.com/prod/items
  ```
  → Fresh response (new timestamp). Note: by default, any client can invalidate. In production, restrict this with `Require authorization` in cache settings.

---

## Part D — Throttling with Usage Plans and API Keys

**1. Create an API Key**
- [ ] API Gateway → API Keys → Create API key.
- [ ] Name: `lab-key`. Auto generate: Yes. Save.
- [ ] Copy the API key value.

**2. Require the API key on the method**
- [ ] Go to `/items` GET → Method Request → API Key Required: **true**.
- [ ] Redeploy to `prod`.

**3. Test without key**
- [ ] ```bash
  curl -s https://<api-id>.execute-api.<region>.amazonaws.com/prod/items
  ```
  → `{"message": "Forbidden"}` (403).

**4. Test with key**
- [ ] ```bash
  curl -s -H "x-api-key: <your-api-key>" \
    https://<api-id>.execute-api.<region>.amazonaws.com/prod/items
  ```
  → `200 OK` with Lambda response.

**5. Create a Usage Plan with throttling**
- [ ] API Gateway → Usage Plans → Create.
- [ ] Name: `basic-plan`.
- [ ] Throttle: Rate = **5 requests/second**, Burst = **10**.
- [ ] Quota: **100 requests per day**.
- [ ] Add API Stage: select `lab-api` → `prod`.
- [ ] Add API Key: select `lab-key`.
- [ ] Save.

**6. Test throttling**
- [ ] Rapid-fire requests:
  ```bash
  for i in $(seq 1 20); do
    curl -s -o /dev/null -w "%{http_code}\n" \
      -H "x-api-key: <your-api-key>" \
      https://<api-id>.execute-api.<region>.amazonaws.com/prod/items &
  done
  wait
  ```
  → Most return `200`, but some return `429 Too Many Requests` (rate exceeded).

---

## Part E — Stage Variables (Stretch)

Stage variables let you point different stages at different Lambda aliases:

- [ ] Lambda → `api-backend` → create alias `live` pointing to `$LATEST`.
- [ ] API Gateway → Stages → `prod` → Stage Variables → Add: `lambdaAlias = live`.
- [ ] Update the Lambda integration to use `api-backend:${stageVariables.lambdaAlias}`.
- [ ] Add Lambda permission for API Gateway to invoke the alias:
  ```bash
  aws lambda add-permission \
    --function-name api-backend:live \
    --statement-id apigateway-prod \
    --action lambda:InvokeFunction \
    --principal apigateway.amazonaws.com
  ```
- [ ] Now `dev` can point to a `dev` alias, `prod` to `live` — same API, different Lambda versions per stage.

---

## Key Things to Observe

- API Gateway caching is **per-stage** and **per-path** (including path parameters and query strings as cache keys).
- Cached responses skip Lambda entirely — no invocation, no cost, much lower latency.
- Cache TTL default is 300s (5 min). Max 3600s. Setting TTL to 0 disables caching.
- Usage plans control throttling (rate/burst) and quotas (daily/monthly). The API key identifies the caller.
- API keys are for **throttling/metering**, NOT for authentication. Use Cognito or Lambda authorizers for auth.
- REST API vs HTTP API: REST API supports caching, usage plans, and API keys. HTTP API is cheaper and simpler but lacks caching.
- Stage variables allow environment-specific configuration without duplicating APIs.

---

## Clean Up

- API Gateway → delete `lab-api` (deletes stages, resources, authorizers).
- Delete API key and usage plan.
- Lambda → delete `api-backend`.
- Note: API Gateway cache is billed hourly — delete the API promptly.
