# Lab: Kinesis — Data Streams, Firehose & Real-Time Architecture

*Phase 5 · Messaging & Streaming · Concept: [concepts/messaging.md](../concepts/messaging.md)*

---

## Goal

Create a Kinesis Data Stream, produce and consume records, then set up a Kinesis Data Firehose delivery stream to S3. Understand the difference between Streams vs Firehose, and when to use each vs SQS/SNS.

---

## Architecture

```
Producer (CLI) → Kinesis Data Stream (2 shards)
                        │
                        ├── Consumer: Lambda (real-time processing)
                        │
                        └── Kinesis Data Firehose → S3 bucket (archive)
                                                    (buffered, near-real-time)
```

---

## Part A — Create a Kinesis Data Stream

- [ ] Kinesis → Data Streams → Create data stream.
- [ ] Stream name: `lab-stream`.
- [ ] Capacity mode: **Provisioned** (so you can see shard mechanics).
- [ ] Number of shards: **2**.
- [ ] Create.

**Understand shard capacity:**

| Direction | Per Shard | 2 Shards Total |
|---|---|---|
| Write | 1 MB/s or 1,000 records/s | 2 MB/s or 2,000 records/s |
| Read | 2 MB/s (shared) or 2 MB/s per consumer (enhanced) | 4 MB/s |

- [ ] Note: each record has a **partition key** — Kinesis hashes it to assign the record to a shard. Records with the same partition key go to the same shard (ordering guarantee within a shard).

---

## Part B — Produce Records to the Stream

- [ ] In CloudShell, put records into the stream:
  ```bash
  # Put a single record
  aws kinesis put-record \
    --stream-name lab-stream \
    --partition-key user-123 \
    --data $(echo '{"userId":"user-123","action":"login","ts":"2026-03-01T10:00:00Z"}' | base64)

  # Put several records
  for i in $(seq 1 10); do
    aws kinesis put-record \
      --stream-name lab-stream \
      --partition-key "user-$((i % 3))" \
      --data $(echo "{\"userId\":\"user-$((i % 3))\",\"action\":\"click\",\"item\":$i}" | base64)
  done
  ```
- [ ] Note the response: `ShardId` tells you which shard the record went to, `SequenceNumber` is the ordering key.

---

## Part C — Consume Records with the CLI

**1. Get a shard iterator**
- [ ] ```bash
  # List shards
  aws kinesis list-shards --stream-name lab-stream

  # Get iterator for shard-000000000000 (from the beginning)
  SHARD_ITERATOR=$(aws kinesis get-shard-iterator \
    --stream-name lab-stream \
    --shard-id shardId-000000000000 \
    --shard-iterator-type TRIM_HORIZON \
    --query 'ShardIterator' --output text)

  # Read records
  aws kinesis get-records --shard-iterator $SHARD_ITERATOR
  ```
- [ ] Decode the data (it's base64-encoded):
  ```bash
  echo "<base64-data-from-response>" | base64 --decode
  ```
- [ ] You'll see only the records that were assigned to this shard (based on partition key hash).

**2. Understand iterator types**

| Iterator Type | Starts Reading From |
|---|---|
| `TRIM_HORIZON` | Beginning of the stream (oldest record) |
| `LATEST` | Only new records arriving after this point |
| `AT_TIMESTAMP` | A specific timestamp |
| `AT_SEQUENCE_NUMBER` | A specific sequence number |

**Exam trap:** Kinesis retains data for **24 hours** by default (extendable to **365 days**). SQS retains for **4 days** (max 14 days). This retention difference matters for replay scenarios.

---

## Part D — Lambda Consumer (Real-Time Processing)

**1. Create a Lambda function**
- [ ] Lambda → Create function → `kinesis-processor`.
- [ ] Runtime: Python 3.12.
- [ ] Code:
  ```python
  import json
  import base64

  def lambda_handler(event, context):
      for record in event['Records']:
          payload = base64.b64decode(record['kinesis']['data']).decode('utf-8')
          data = json.loads(payload)
          print(f"Shard: {record['eventID']} | Partition: {record['kinesis']['partitionKey']} | Data: {data}")
      return {'statusCode': 200, 'body': f"Processed {len(event['Records'])} records"}
  ```
- [ ] Deploy.

**2. Add Kinesis trigger**
- [ ] Lambda → `kinesis-processor` → Add trigger → Kinesis.
- [ ] Stream: `lab-stream`.
- [ ] Batch size: 10. Starting position: **TRIM_HORIZON** (read from beginning).
- [ ] Add.
- [ ] Lambda execution role needs `kinesis:GetRecords`, `kinesis:GetShardIterator`, `kinesis:DescribeStream`, `kinesis:ListShards` — add the `AWSLambdaKinesisExecutionRole` managed policy.

**3. Produce more records and check Lambda logs**
- [ ] Put a few more records (same commands from Part B).
- [ ] CloudWatch → Log groups → `/aws/lambda/kinesis-processor`.
- [ ] You should see the decoded records logged — Lambda is consuming in near-real-time.

**Key observation:** Lambda creates one concurrent invocation **per shard**. With 2 shards, you get 2 parallel Lambda executions. This is different from SQS, where Lambda scales based on queue depth.

---

## Part E — Kinesis Data Firehose to S3

**1. Create an S3 bucket for delivery**
- [ ] Create bucket: `kinesis-archive-<random>`.

**2. Create a Firehose delivery stream**
- [ ] Kinesis → Data Firehose → Create delivery stream.
- [ ] Source: **Amazon Kinesis Data Streams** → select `lab-stream`.
- [ ] Destination: **Amazon S3** → select `kinesis-archive-<random>`.
- [ ] Buffer conditions:
  - Buffer size: **1 MB** (delivers when buffer reaches 1 MB).
  - Buffer interval: **60 seconds** (or delivers every 60 seconds, whichever comes first).
- [ ] Delivery stream name: `lab-firehose-to-s3`.
- [ ] Create.

**3. Produce records and check S3 delivery**
- [ ] Put 20+ records into `lab-stream` (use the loop from Part B).
- [ ] Wait ~60–90 seconds (buffer interval).
- [ ] Check S3 bucket → Firehose creates a folder structure: `YYYY/MM/DD/HH/`.
- [ ] Download a file — it contains the raw records (newline-delimited JSON after base64 decode).

**4. Observe the key differences**

| Feature | Kinesis Data Streams | Kinesis Data Firehose |
|---|---|---|
| Consumer model | You write consumers (Lambda, KCL app) | Fully managed delivery (no consumer code) |
| Latency | ~200ms (real-time) | 60s–900s (near-real-time, buffered) |
| Destinations | Anything your consumer code writes to | S3, Redshift, OpenSearch, Splunk, HTTP |
| Data retention | 24h–365 days (replay possible) | No retention (deliver and forget) |
| Scaling | Manual (add/remove shards) | Auto-scales |
| Transformation | Consumer code | Built-in Lambda transform (optional) |
| Cost | Per shard-hour + per PUT | Per GB ingested |

---

## Part F — Kinesis vs SQS vs SNS (Exam Decision Tree)

| Scenario | Service |
|---|---|
| Real-time analytics dashboard | Kinesis Data Streams |
| Clickstream ingestion from millions of users | Kinesis Data Streams |
| Log aggregation to S3 (near-real-time) | Kinesis Data Firehose |
| Decouple microservices (request/response) | SQS |
| Fan-out notifications to multiple subscribers | SNS (or SNS → SQS) |
| Ordered processing per user/session | Kinesis (partition key = user ID) |
| IoT sensor data at scale | Kinesis Data Streams |
| ETL pipeline: stream → transform → S3/Redshift | Firehose with Lambda transform |
| Replay old messages for reprocessing | Kinesis (has retention). SQS messages are deleted after processing. |
| Simple task queue with DLQ | SQS |

**Exam trap:** "Multiple consumers need to read the same data independently" → Kinesis (fan-out with enhanced consumers) or SNS → SQS fan-out. SQS alone is single-consumer (one message = one processor).

---

## Key Things to Observe

- Kinesis Data Streams = real-time, you manage shards and write consumers.
- Kinesis Data Firehose = near-real-time managed delivery to S3/Redshift/OpenSearch. No consumer code needed.
- Partition key determines shard assignment. Same partition key = same shard = ordered within that key.
- Lambda processes one batch per shard concurrently. More shards = more parallelism.
- Data Streams retains data (24h default, up to 365 days). SQS deletes after processing. Firehose doesn't retain.
- Enhanced fan-out gives each consumer 2 MB/s per shard (dedicated). Without it, all consumers share 2 MB/s per shard.
- On-demand capacity mode (alternative to provisioned) auto-scales shards — good for unpredictable throughput.

---

## Clean Up

- Lambda → remove Kinesis trigger → delete `kinesis-processor`.
- Kinesis → delete Firehose delivery stream `lab-firehose-to-s3`.
- Kinesis → delete Data Stream `lab-stream`.
- S3 → empty and delete `kinesis-archive-<random>`.
