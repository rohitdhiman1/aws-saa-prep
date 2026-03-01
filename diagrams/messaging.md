# Diagrams: Messaging & Streaming

*Draw these on paper or in draw.io as part of Week 9 study.*

---

## Diagram 1: SNS Fan-Out Pattern

```
                    ┌─────────────────┐
                    │   SNS Topic     │
                    │  (order-events) │
                    └────┬────┬───────┘
                         │    │    │
           ┌─────────────┘    │    └───────────────┐
           ▼                  ▼                    ▼
  SQS Queue               Lambda               SQS Queue
  (order-processing)   (send-receipt)       (inventory-update)
  (async; decoupled)   (real-time)          (async; decoupled)
```

**Why fan-out via SNS→SQS?** SQS provides buffering and retry; multiple consumers get independent copies.

---

## Diagram 2: Kinesis Data Stream Pipeline

```
Producers                     Kinesis Data Streams            Consumers
(IoT devices,     ──PutRecord──►  Shard-0 │                  ┌── Lambda (enhanced fan-out)
 clickstreams,                    Shard-1 │ ──GetRecords──►  ├── Kinesis Firehose → S3
 app logs)                        Shard-2 │                  └── KDA (Flink) → alerts

Each shard: 1 MB/s write, 2 MB/s read
Partition key determines shard assignment
Retention: 24h (default) → up to 365 days
```

---

## Diagram 3: SQS Visibility Timeout Flow

```
Message arrives in queue
        │
Consumer polls ──► Message received (invisible for timeout period)
        │                    │
        │           Consumer processes
        │                    │
        │          Success: DeleteMessage ──► Message gone
        │          Failure (timeout expires): Message reappears in queue
        │                    │
        │           Reprocessed... after maxReceiveCount
        │                    │
        │                   DLQ (Dead Letter Queue)
```

---

## Diagram 4: EventBridge Event Flow

```
Event Source                 EventBridge Bus              Targets
────────────                 ───────────────              ───────
AWS service event ─────────► Default Bus ──► Rule ──────► Lambda
  (EC2 stopped)              │              (pattern)    │
Custom app event ───────────► Custom Bus                  ├── SQS
  (order.placed)                                          ├── Step Functions
SaaS event ────────────────► Partner Bus                  └── SNS
  (Zendesk ticket)           │
                             │
                         Archive (replay later)
```

---

## Diagram 5: Step Functions State Machine

```
START
  │
  ▼
[Task: ValidateOrder] ──error──► [Catch: Send to DLQ]
  │ success
  ▼
[Task: ChargePayment] ──error──► [Retry 3x] ──still fail──► [Fail state]
  │ success
  ▼
[Parallel:
  ├── [Task: SendConfirmationEmail]
  └── [Task: UpdateInventory]
]
  │ both complete
  ▼
[Task: ShipOrder]
  │
  ▼
[Succeed]
```
