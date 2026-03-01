# Lab: SQS + SNS — Fan-Out, DLQ & Visibility Timeout

*Phase 5 · Week 9 · Sat May 2 · Concept: [concepts/messaging.md](../concepts/messaging.md)*

---

## Goal

Build an SNS → SQS fan-out pattern. Observe visibility timeout, DLQ behaviour, and message filtering.

---

## Architecture

```
                    ┌── SQS Queue A (orders-processing)
SNS Topic ──────────┤
(order-events)      └── SQS Queue B (orders-audit)

SQS Queue A ── DLQ (orders-processing-dlq) [after 3 failed receives]
```

---

## Tasks

### Part A — Create SQS Queues

**1. Create DLQ first**
- [ ] SQS → Create queue → Standard queue.
- [ ] Name: `orders-processing-dlq`. All defaults. Create.

**2. Create Queue A (with DLQ)**
- [ ] Create queue → Standard → Name: `orders-processing`.
- [ ] Scroll to Dead-letter queue → Enable → select `orders-processing-dlq` → Max receives: `3`.
- [ ] Create.

**3. Create Queue B (audit)**
- [ ] Create queue → Standard → Name: `orders-audit`. All defaults. Create.

---

### Part B — Create SNS Topic and Subscribe Queues

**1. Create topic**
- [ ] SNS → Topics → Create topic → Standard.
- [ ] Name: `order-events`. Create.

**2. Subscribe Queue A**
- [ ] Topic → Create subscription → Protocol: **SQS** → Endpoint: ARN of `orders-processing`.
- [ ] Create subscription.
- [ ] **Allow SNS to write to SQS:** Go to `orders-processing` → Access Policy → Edit:
  Add a statement allowing `sns:SendMessage` from your SNS topic ARN:
  ```json
  {
    "Effect": "Allow",
    "Principal": {"Service": "sns.amazonaws.com"},
    "Action": "sqs:SendMessage",
    "Resource": "<orders-processing ARN>",
    "Condition": {"ArnEquals": {"aws:SourceArn": "<topic ARN>"}}
  }
  ```

**3. Subscribe Queue B**
- [ ] Same steps for `orders-audit`.

---

### Part C — Test Fan-Out

- [ ] SNS → Topics → `order-events` → Publish message.
  ```json
  {
    "orderId": "ORD-001",
    "customer": "alice",
    "amount": 99.99
  }
  ```
- [ ] Check `orders-processing` (SQS → Poll for messages) → message appears.
- [ ] Check `orders-audit` (Poll for messages) → same message appears independently.
- [ ] Both queues received a copy — this is fan-out.

---

### Part D — Visibility Timeout

- [ ] In `orders-processing` → Poll for messages → receive the message. It disappears from the queue (invisible).
- [ ] Default visibility timeout: 30 seconds. Wait 30 seconds without deleting the message.
- [ ] Poll again → message reappears (visibility timeout expired; someone else can pick it up now).
- [ ] This simulates a consumer crashing without acknowledging — DynamoDB/Lambda equivalent.

**Extend visibility timeout:**
- [ ] When you receive a message in the console, note the Receipt Handle.
- [ ] In a real app, a consumer can call `ChangeMessageVisibility` to extend the timeout if processing takes longer.

---

### Part E — Trigger the DLQ

- [ ] Receive the message from `orders-processing` 3 times without deleting it (each receive = 1 count).
  (Poll → receive → wait for timeout → poll again → repeat 3 times.)
- [ ] After 3 receives, the message stops appearing in `orders-processing`.
- [ ] Check `orders-processing-dlq` → Poll → the message appears here.
- [ ] This is the DLQ in action — message moved here after `maxReceiveCount` (3) failures.

---

### Part F — SNS Message Filtering (Stretch)

- [ ] Modify the subscription for `orders-audit` to only receive orders > $50:
  - Subscription → Edit → Subscription filter policy:
    ```json
    {
      "amount": [{"numeric": [">", 50]}]
    }
    ```
- [ ] Publish a message with `amount: 25` → only `orders-processing` receives it (audit filtered out).
- [ ] Publish a message with `amount: 99` → both queues receive it.

---

## Key Things to Observe

- SNS publishes once; both SQS queues receive independent copies (fan-out).
- SQS visibility timeout ≠ message deletion — it's just hidden temporarily.
- DLQ receives messages only after `maxReceiveCount` failures — useful for debugging poison-pill messages.
- SNS message filtering happens at the subscription level — before the message reaches the queue.
- FIFO queues require FIFO SNS topics for fan-out (Standard ↔ Standard only).

---

## Clean Up

- Delete subscriptions (in SNS).
- Delete SNS topic.
- Delete all 3 SQS queues (`orders-processing`, `orders-audit`, `orders-processing-dlq`).
