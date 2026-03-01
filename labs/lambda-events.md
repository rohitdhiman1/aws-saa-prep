# Lab: Lambda — Event Triggers & Concurrency

*Phase 2 · Week 4 · Sun Mar 22 · Concept: [concepts/compute.md](../concepts/compute.md)*

---

## Goal

Trigger a Lambda function from an S3 event and from API Gateway. Observe async vs sync invocation. Test concurrency limits.

---

## Architecture

```
S3 (file upload) ──async──► Lambda (process-file)  ──► CloudWatch Logs
API Gateway       ──sync───► Lambda (hello-api)     ──► response
```

---

## Tasks

### Part A — Lambda via S3 Event (Async)

**1. Create the Lambda function**
- [ ] Go to Lambda → Create function → Author from scratch.
- [ ] Name: `process-file`. Runtime: Python 3.12. Execution role: Create new basic role.
- [ ] Paste this code:
  ```python
  import json, urllib.parse, boto3

  def lambda_handler(event, context):
      bucket = event['Records'][0]['s3']['bucket']['name']
      key = urllib.parse.unquote_plus(event['Records'][0]['s3']['object']['key'])
      print(f"New file uploaded: s3://{bucket}/{key}")
      return {'statusCode': 200}
  ```
- [ ] Deploy.

**2. Create an S3 bucket and add trigger**
- [ ] Create an S3 bucket (any unique name).
- [ ] In Lambda → Configuration → Triggers → Add trigger → S3.
- [ ] Select your bucket. Event type: `PUT`. Save.

**3. Test**
- [ ] Upload any file to the S3 bucket.
- [ ] Go to Lambda → Monitor → View CloudWatch logs.
- [ ] Find the log stream — see the "New file uploaded" message.

**4. Observe async behaviour**
- [ ] Notice: S3 did not wait for Lambda to finish — S3 PUT returned immediately.
- [ ] Check: Lambda retry on error — introduce a bug (e.g. `raise Exception("error")`), re-upload → check CloudWatch to see 3 attempts (initial + 2 retries).

---

### Part B — Lambda via API Gateway (Sync)

**1. Create a new Lambda function**
- [ ] Name: `hello-api`. Runtime: Python 3.12. Same execution role.
- [ ] Paste this code:
  ```python
  def lambda_handler(event, context):
      name = event.get('queryStringParameters', {}).get('name', 'World')
      return {
          'statusCode': 200,
          'body': f'Hello, {name}!'
      }
  ```
- [ ] Deploy.

**2. Add API Gateway trigger**
- [ ] In Lambda → Triggers → Add trigger → API Gateway.
- [ ] Create a new HTTP API. Security: Open. Save.

**3. Test**
- [ ] Copy the API endpoint URL from the trigger details.
- [ ] Open in browser: `<URL>?name=Rohit` → should return `Hello, Rohit!`
- [ ] This is **synchronous** — API Gateway waits for Lambda response before returning to browser.

---

### Part C — Concurrency (Observe)

**1. Check reserved concurrency**
- [ ] Go to `hello-api` → Configuration → Concurrency.
- [ ] Set Reserved concurrency to `0` → Save.
- [ ] Hit the API URL → should get a throttle error (429).
- [ ] Set Reserved concurrency back to `Unreserved`. Notice how this acts as an on/off switch.

**2. Observe cold start (optional)**
- [ ] Invoke the function a few times quickly (same URL, rapid refresh).
- [ ] Check CloudWatch logs — look for `INIT_START` entries. Each new execution environment initialisation = cold start.

---

## Key Things to Observe

- **Async (S3):** Lambda retries automatically on failure; caller (S3) doesn't wait.
- **Sync (API Gateway):** Caller waits; Lambda must respond within timeout.
- **Reserved concurrency = 0** completely throttles the function.
- `INIT_START` in logs = cold start; `START` without `INIT` = warm invocation.

---

## Clean Up

- Delete both Lambda functions.
- Delete the S3 bucket (empty it first).
- Delete the API Gateway (via API Gateway console).
