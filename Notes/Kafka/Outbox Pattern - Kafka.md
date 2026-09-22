The problem we're solving
current flow:
```
DB transaction
    ↓
COMMIT ✅
    ↓
Kafka publish
    ↓
❌ Kafka unavailable
```

Now the database says the operation happened, but Kafka never received the event.
The reverse ordering has its own failure:
```
Kafka publish ✅
    ↓
DB transaction ❌
```
Now Kafka contains an event for a database operation that never committed.

This is the **dual-write problem**.


### Phase 1 - Understand the outbox
instead of writing to two independent systems:
we can make postgresSql the source of truth for the initial transaction
```
                 PostgreSQL
                     │
              ONE transaction
                ┌────┴────┐
                ↓         ↓
          Business DB   Outbox
                         │
                         ↓
                   Outbox Publisher
                         │
                         ↓
                       Kafka
```

For DevSync
```
IssueService
    │
    │ transaction
    ├──────────────→ Issue changes
    │
    └──────────────→ Outbox event
                         │
                         │ later
                         ↓
                    Kafka Producer
                         │
                         ↓
                devsync.issue-events
```

> The business change and the outbox record are committed atomically in the same PostgresSQL transaction.

