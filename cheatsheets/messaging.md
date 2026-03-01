# Cheatsheet: Messaging & Streaming

*Fill in as you complete Week 9. Use for exam-day revision.*

---

## Key Facts

| Concept | Detail |
|---|---|
| SQS max message size | 256 KB |
| SQS max retention | 14 days |
| SQS visibility timeout max | 12 hours |
| SQS FIFO throughput | 300/s (3,000 with batching) |
| Kinesis shard write | 1 MB/s or 1,000 records/s |
| Kinesis shard read | 2 MB/s |
| Kinesis Firehose min buffer | 60 seconds |
| Lambda max duration | 15 minutes |
| Step Functions Standard max | 1 year |
| Step Functions Express | at-least-once |

---

## Must-Know Distinctions

- **SQS Standard** → unlimited throughput, at-least-once | **FIFO** → 300/s, exactly-once
- **SNS** → push, no retention | **SQS** → pull, retention up to 14 days
- **Kinesis Streams** → replay, ordered, real-time | **SQS** → no replay, one consumer
- **Kinesis Firehose** → managed delivery (near-real-time) | **Streams** → real-time, you manage consumers
- **EventBridge** → richer filtering, archive/replay | **SNS** → simpler, mobile push
- **Step Functions Standard** → exactly-once, long-running | **Express** → at-least-once, high-volume

---

## Exam Traps (Quick List)

- SQS DLQ must match queue type (Standard→Standard, FIFO→FIFO)
- Kinesis Firehose = near-real-time (~60s delay), not real-time
- SNS loses messages if subscriber is down (except SQS subscriber)
- SQS visibility timeout ≠ message expiry
- Long polling (`WaitTimeSeconds=20`) reduces cost and empty responses

---

## Related Files

`concepts/messaging.md` · `diagrams/messaging.md` · `mock-exams/phase5.md`
