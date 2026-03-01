# Lab: DynamoDB — Table Design, GSI & Streams

*Phase 3 · Week 6 · Sat Apr 11 · Concept: [concepts/databases-nosql.md](../concepts/databases-nosql.md)*

---

## Goal

Create a DynamoDB table with a composite key, add a GSI, query using both, enable Streams, and observe TTL deletion.

---

## Architecture

```
DynamoDB Table: Orders
  Primary key: PK (CustomerID) + SK (OrderDate)
  GSI: OrderDate-index (PK = OrderDate, for querying all orders on a date)
  Streams → Lambda (log changes)
```

---

## Tasks

### Part A — Create Table with Composite Key

- [ ] DynamoDB → Create table.
- [ ] Table name: `Orders`.
- [ ] Partition key: `CustomerID` (String).
- [ ] Sort key: `OrderDate` (String).
- [ ] Table settings: Customise → Capacity mode: **On-demand** (simplest for lab).
- [ ] Create.

---

### Part B — Insert Items and Query

**Insert sample data (use CloudShell or DynamoDB console → Explore items → Create item)**
- [ ] Item 1: `CustomerID=alice`, `OrderDate=2024-01-10`, `Amount=99`, `Status=shipped`
- [ ] Item 2: `CustomerID=alice`, `OrderDate=2024-01-15`, `Amount=45`, `Status=pending`
- [ ] Item 3: `CustomerID=bob`, `OrderDate=2024-01-10`, `Amount=200`, `Status=delivered`
- [ ] Item 4: `CustomerID=bob`, `OrderDate=2024-01-20`, `Amount=75`, `Status=pending`

**Query the table (from console → Explore items → Query)**
- [ ] Query: `CustomerID = alice` → returns 2 items (both of Alice's orders, sorted by OrderDate).
- [ ] Query: `CustomerID = alice AND OrderDate begins_with 2024-01-1` → returns only Alice's Jan 10–19 orders.
- [ ] Try Scan (not Query) → returns all 4 items but note: Scan reads every partition (expensive at scale).

---

### Part C — Add a GSI

*(Scenario: "Show me all orders from a specific date, across all customers.")*

- [ ] Table → Indexes tab → Create index.
- [ ] Partition key: `OrderDate` (String). Sort key: `CustomerID` (String).
- [ ] Index name: `OrderDate-index`. Projected attributes: All.
- [ ] Create index. Wait for it to become Active (~1–2 min).

**Query the GSI**
- [ ] Explore items → change Query scope to `OrderDate-index`.
- [ ] Query: `OrderDate = 2024-01-10` → returns Alice's and Bob's orders from Jan 10.
- [ ] Try strongly consistent read on GSI → not available (note the error or greyed-out option).

---

### Part D — DynamoDB Streams + Lambda

**Enable Streams**
- [ ] Table → Exports and streams → DynamoDB stream details → Enable.
- [ ] View type: **New and old images** (see what changed).

**Create a Lambda trigger**
- [ ] Lambda → Create function. Name: `stream-logger`. Runtime: Python 3.12. Execution role: create a role with `AWSLambdaDynamoDBExecutionRole` attached.
- [ ] Paste this code:
  ```python
  def lambda_handler(event, context):
      for record in event['Records']:
          eventName = record['eventName']
          newImage = record.get('dynamodb', {}).get('NewImage', {})
          print(f"Event: {eventName} | Item: {newImage}")
  ```
- [ ] Deploy.
- [ ] In Lambda → Add trigger → DynamoDB → select `Orders` table. Batch size: 1. Save.

**Test**
- [ ] In DynamoDB, update Item 1 (Alice, Jan 10): change `Status` from `shipped` to `returned`.
- [ ] Check Lambda → Monitor → CloudWatch logs → see the event type `MODIFY` and both old + new image.
- [ ] Add a new item → see `INSERT` event. Delete an item → see `REMOVE` event.

---

### Part E — TTL

- [ ] Table → Additional settings → Time to Live (TTL) → Enable.
- [ ] TTL attribute name: `ExpiresAt`.
- [ ] Add `ExpiresAt` to an item. The value must be a **Unix epoch timestamp** (seconds since 1970-01-01).
  - To set expiry to 1 minute from now: use https://www.epochconverter.com or in CloudShell:
    ```bash
    date -d "+1 minute" +%s   # gives Unix timestamp ~1 min from now
    ```
  - Set `ExpiresAt` (Number) to that timestamp value.
- [ ] Within 48 hours (often within minutes in practice), DynamoDB deletes the item automatically.
- [ ] Observe: the Streams trigger fires a `REMOVE` event even for TTL deletes.

---

## Key Things to Observe

- Primary key query returns items sorted by sort key automatically.
- GSI lets you query on a completely different attribute (OrderDate as PK).
- GSI reads are eventually consistent only — no strong consistency option.
- Streams capture every mutation (INSERT/MODIFY/REMOVE) as an ordered event.
- TTL deletion appears in Streams as a REMOVE with `userIdentity.type = "Service"`.

---

## Clean Up

- Delete the Lambda trigger (in Lambda → Triggers).
- Delete the Lambda function.
- Delete the DynamoDB table (deletes GSI and streams automatically).
