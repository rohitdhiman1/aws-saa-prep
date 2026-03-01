# Cheatsheet: Databases

*Fill in as you complete Weeks 5–6. Use for exam-day revision.*

---

## Key Facts

| Concept | Detail |
|---|---|
| RDS Multi-AZ failover | ~60–120 s (DNS update) |
| Aurora failover | <30 s (replica promotion) |
| Aurora Global DB lag | <1 second |
| Aurora max replicas | 15 |
| DynamoDB item max size | 400 KB |
| DynamoDB Streams retention | 24 hours |
| DynamoDB PITR window | 35 days |
| DynamoDB TTL delay | Up to 48 hours (not instant) |
| ElastiCache Redis | Persistence + replication |
| ElastiCache Memcached | No persistence, no replication |

---

## Must-Know Distinctions

- **Multi-AZ** → HA, standby NOT readable | **Read Replica** → scale reads, readable
- **Aurora Backtrack** → MySQL only, rewind without restore | **PITR** → all engines, restore to new cluster
- **GSI** → eventually consistent only | **LSI** → strongly consistent possible
- **LSI** → at creation only | **GSI** → can add later
- **DynamoDB DAX** → microsecond reads; write-through; doesn't help writes
- **Secrets Manager** → auto-rotation | **SSM Parameter Store** → no auto-rotation, cheaper

---

## Exam Traps (Quick List)

- Redshift = OLAP (analytics), not OLTP
- RDS encryption must be enabled at creation
- DynamoDB Global Tables requires Streams
- ElastiCache cluster mode = sharding (write scale); non-cluster = replicas (read scale)
- RDS Proxy: reduces connection overhead for Lambda/serverless
- Neptune = graph DB (social networks, fraud detection)

---

## Related Files

`concepts/databases-relational.md` · `concepts/databases-nosql.md` · `diagrams/databases.md` · `mock-exams/phase3.md`
