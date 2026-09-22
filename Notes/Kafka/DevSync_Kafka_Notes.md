# Kafka — DevSync Notes

> **Project:** DevSync  
> **Purpose:** Learn Kafka by integrating it into a real Spring Boot backend.  
> **Current use case:** `IssueAssignedEvent` → Kafka → notification processing.

---

## 1. Why Kafka in DevSync?

Kafka is being used to decouple event producers from asynchronous consumers.

```text
IssueService
    ↓
IssueAssignedEvent
    ↓
Kafka Adapter
    ↓
devsync.issue-events
    ↓
Kafka Consumer
    ↓
IssueAssignmentNotificationService
    ↓
PostgreSQL Notification
```

**Architecture rule:** `IssueService` should not directly depend on Kafka.

The application publishes a domain event. A Kafka adapter translates it into a Kafka-specific event and publishes it.

---

## 2. Kafka Event vs Kafka DTO

The domain event remains independent of Kafka.

Kafka receives a dedicated DTO:

```java
public record IssueAssignedKafkaEvent(
        UUID eventId,
        UUID issueId,
        String title,
        String description,
        UUID actorId,
        UUID assigneeId,
        UUID workspaceId,
        String workspaceName,
        UUID projectId,
        String projectName
) {}
```

Architecture:

```text
Domain
  ↓
Domain Event
  ↓
Kafka Adapter
  ↓
Kafka DTO
```

This keeps infrastructure concerns out of the domain layer.

---

## 3. Kafka Topics

Main topic:

```text
devsync.issue-events
```

Dead Letter Topic:

```text
devsync.issue-events.DLT
```

The main topic currently has 3 partitions.

---

## 4. Kafka Message Keys and Ordering

The producer uses:

```java
event.issueId().toString()
```

as the Kafka key.

Events for the same issue therefore go to the same partition.

Example:

```text
Issue A → assigned to B
Issue A → assigned to C
Issue A → assigned to D
```

All use the same key:

```text
Issue A
   ↓
same partition
```

Kafka guarantees ordering **within a partition**, not globally across all partitions.

Therefore, events for the same issue can maintain ordering while different issues can be processed in parallel.

---

## 5. Producer Reliability

Current producer configuration includes:

```text
acks = -1
enable.idempotence = true
```

### `acks=-1`

The producer waits for acknowledgement from all in-sync replicas.

### `enable.idempotence=true`

Kafka producer idempotence prevents certain duplicate records caused by producer retries.

Important:

> Producer idempotence does **not** make the consumer business operation idempotent.

Consumer-side idempotency is still required.

---

## 6. Consumer Groups

Current consumer group:

```text
devsync-notification-service
```

A consumer group allows multiple consumers to share partitions.

Example:

```text
Topic: 3 partitions

Consumer A → Partition 0
Consumer B → Partition 1
Consumer C → Partition 2
```

Within one consumer group, a partition is assigned to only one consumer at a time.

Different consumer groups each receive their own copy of the event stream:

```text
devsync.issue-events
       │
       ├── notification-service
       ├── analytics-service
       └── audit-service
```

---

## 7. Offsets

Kafka records are identified by offsets:

```text
Partition 0

offset 0
offset 1
offset 2
offset 3
```

The consumer group stores its progress through committed offsets.

Current configuration:

```yaml
enable.auto.commit: false
```

This means the application/framework controls when offsets are committed.

If processing fails before the offset is committed, Kafka can deliver the message again.

---

## 8. `auto.offset.reset=earliest`

Current configuration:

```yaml
auto.offset.reset: earliest
```

If the consumer group has no committed offset for a partition, Kafka starts from the earliest available record.

It does **not** mean every restart replays the entire topic.

If a committed offset exists, Kafka resumes from that committed position.

---

## 9. At-Least-Once Delivery

Kafka consumers commonly operate with **at-least-once semantics**.

Meaning:

> A message can be delivered more than once.

Example:

```text
Kafka
  ↓
Event A
  ↓
Consumer
  ↓
DB transaction succeeds
  ↓
Application fails before offset commit
  ↓
Kafka delivers Event A again
```

Therefore:

```text
delivery count ≠ business operation count
```

A reliable consumer must safely handle duplicate delivery.

---

## 10. Retry Handling

Spring Kafka's:

```java
DefaultErrorHandler
```

was configured with:

```java
FixedBackOff backOff = new FixedBackOff(
        1000L,
        2L
);
```

Meaning:

```text
Initial attempt
     ↓
Retry #1
     ↓
Retry #2
```

Total attempts: **3**

There is a 1-second delay between attempts.

Important:

> `enable.auto.commit=false` does not itself define retry behavior. The Spring Kafka error handler controls what happens after listener failure.

---

## 11. Retry and Offsets

When a listener throws:

```text
Consumer receives offset N
        ↓
processing fails
        ↓
retry same record
        ↓
offset N
```

The same topic/partition/offset can therefore appear multiple times during retries.

---

## 12. Dead Letter Topic (DLT)

After retries are exhausted, the record can be recovered to a DLT.

Configured with:

```java
DeadLetterPublishingRecoverer
```

Flow:

```text
devsync.issue-events
        ↓
failure
        ↓
retry × 2
        ↓
DLT
        ↓
devsync.issue-events.DLT
```

Important configuration detail:

```java
new DefaultErrorHandler(
        recoverer,
        backOff
);
```

Creating a `DeadLetterPublishingRecoverer` bean alone is not enough. It must be supplied to the error handler.

---

## 13. DLT Verification

The DLT was verified with:

```cmd
docker exec -it devsync-kafka /opt/kafka/bin/kafka-get-offsets.sh --bootstrap-server localhost:9092 --topic devsync.issue-events.DLT
```

Result:

```text
devsync.issue-events.DLT:0:2
```

This confirmed that failed records had been published to the DLT.

The DLT consumer showed the original event payloads.

---

## 14. Idempotent Consumer

Problem:

```text
Kafka delivers Event A
        ↓
notification created
        ↓
consumer crashes before offset commit
        ↓
Kafka delivers Event A again
        ↓
notification created again
```

Without idempotency:

```text
Duplicate business effect
```

---

## 15. Why `issueId` Is Not the Idempotency Key

An issue can generate multiple legitimate events:

```text
Issue A → User B
Issue A → User C
Issue A → User D
```

Therefore:

```text
issueId ≠ event identity
```

We introduced:

```java
UUID eventId
```

Each event gets its own unique identity.

---

## 16. `processed_kafka_events`

A durable database table was created:

```sql
CREATE TABLE processed_kafka_events (
    event_id UUID PRIMARY KEY,
    processed_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

The primary key provides the database-level uniqueness guarantee.

Conceptually:

```text
eventId
   ↓
processed_kafka_events
   ↓
already processed?
```

---

## 17. Idempotency Flow

The notification service checks:

```java
if (processedKafkaEventRepository.existsById(event.eventId())) {
    return;
}
```

If the event is new:

```text
1. Create notification
2. Store eventId as processed
```

If the event was already processed:

```text
Skip business operation
```

---

## 18. Transactional Idempotency

The notification processing method is transactional:

```java
@Transactional
public void createNotification(IssueAssignedKafkaEvent event) {
    ...
}
```

Desired transaction:

```text
BEGIN TRANSACTION

    check eventId

    create notification

    save processed event

COMMIT
```

If notification creation fails:

```text
ROLLBACK
```

The processed-event record is also rolled back.

This prevents a failed event from being incorrectly marked as successfully processed.

---

## 19. Idempotency Test

The notification-processing method was called twice with the same event.

Result:

```text
First call
    ↓
notification created
    ↓
eventId stored

Second call
    ↓
eventId already exists
    ↓
skip
```

Business effect:

```text
1 notification
```

not:

```text
2 notifications
```

---

## 20. Redelivery Test

We simulated a failure **after the database transaction completed**.

Flow:

```text
Kafka delivers event
        ↓
createNotification()
        ↓
DB transaction commits
        ↓
business effect occurs once
        ↓
consumer throws exception
        ↓
Kafka retries
        ↓
same event delivered again
        ↓
idempotency check
        ↓
event already processed
        ↓
business operation skipped
```

This demonstrated:

> **At-least-once delivery + idempotent consumer = safe duplicate delivery.**

Kafka can deliver the same message multiple times while the database-backed idempotency mechanism ensures the business operation occurs only once.

---

## 21. Kafka Delivery vs Business Effect

Do not confuse:

```text
Kafka delivery semantics
```

with:

```text
business operation semantics
```

Kafka may deliver:

```text
Event A
Event A
Event A
```

while the application can guarantee:

```text
Notification created: once
```

---

## 22. Current DevSync Kafka Architecture

```text
                         DevSync
                            │
                            ▼
                    IssueService
                            │
                    Domain Event
                            │
                            ▼
                 Kafka Event Adapter
                            │
                            ▼
                 devsync.issue-events
                            │
                            ▼
                    Consumer Group
                            │
                            ▼
          IssueAssignmentNotificationService
                            │
                    ┌───────┴────────┐
                    │                │
              PostgreSQL          Failure
                    │                │
                    ▼                ▼
        processed_kafka_events    Retry
                    │                │
                    ▼                ▼
              Notification          DLT
                                  │
                                  ▼
                       devsync.issue-events.DLT
```

---

## 23. What We Have Covered

### Kafka fundamentals

- Topics
- Partitions
- Records
- Keys
- Offsets
- Consumer groups
- Partition ordering

### Producer reliability

- `acks=-1`
- Producer idempotence
- Key-based partitioning

### Consumer reliability

- Manual offset management
- `enable.auto.commit=false`
- Retry
- Backoff
- DLT
- Idempotent consumers
- At-least-once delivery

### DevSync implementation

- Kafka event DTO
- Kafka producer
- Kafka consumer
- Notification migration to Kafka
- `DefaultErrorHandler`
- `DeadLetterPublishingRecoverer`
- DLT
- `eventId`
- `processed_kafka_events`
- Transactional notification processing

---

## 24. Mental Model

Kafka is not a database and Kafka delivery is not the same thing as exactly-once business execution.

Kafka provides durable event storage and delivery semantics.

Consumers must be designed to tolerate:

- retries
- redelivery
- duplicate delivery
- processing failures

Therefore:

```text
Reliable Kafka Consumer
        =
At-least-once delivery
        +
Idempotent business processing
        +
Retry strategy
        +
Dead Letter handling
```

---

# 25. Remaining Problem: Dual Write

There is still a producer-side reliability gap.

Current DevSync flow is approximately:

```text
DB transaction
      ↓
COMMIT
      ↓
Kafka publish
```

Failure window:

```text
DB COMMIT ✅
      ↓
Kafka publish ❌
```

Now PostgreSQL contains the state change, but Kafka does not contain the corresponding event.

This is known as the:

> **Dual-write problem**

The next concept to learn and implement is:

# Transactional Outbox Pattern

Expected architecture:

```text
Application
     │
     ▼
PostgreSQL Transaction
     │
     ├── Business data
     │
     └── Outbox event
              │
              ▼
        Outbox Publisher
              │
              ▼
            Kafka
```

The goal is to make the database state change and event publication reliable without relying on an unsafe two-system write.

---
# 26. Transactional Outbox Pattern

The dual-write problem was identified:

```text
DB transaction
      ↓
   COMMIT
      ↓
Kafka publish
```

Failure window:

```text
DB COMMIT ✅
      ↓
Kafka publish ❌
```

The database contains the state change, but Kafka does not contain the corresponding event.

This can cause the application state and event stream to become inconsistent.

The solution implemented in DevSync is the:

> **Transactional Outbox Pattern**

---

# 27. Outbox Architecture

The new flow is:

```text
IssueService
      │
      ▼
PostgreSQL Transaction
      │
      ├── Update business data
      │
      ├── Create activity
      │
      └── Insert outbox event
              │
              ▼
           COMMIT
              │
              ▼
       Outbox Publisher
              │
              ▼
            Kafka
```

The important property is:

```text
Business change + Outbox event
            ↓
       Same DB transaction
```

If the transaction commits, both are persisted.

If the transaction rolls back, both are rolled back.

---

# 28. Outbox Table

Migration:

```sql
CREATE TABLE outbox_events (
    id UUID PRIMARY KEY,
    aggregate_type VARCHAR(100) NOT NULL,
    aggregate_id UUID NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    published_at TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_outbox_events_unpublished
    ON outbox_events (created_at)
    WHERE published_at IS NULL;
```

Important fields:

```text
id
    → unique outbox event identity

aggregate_type
    → type of business entity

aggregate_id
    → ID of the affected entity

event_type
    → type of event

payload
    → serialized event data

created_at
    → event creation time

published_at
    → when successfully published to Kafka
```

---

# 29. Outbox Event Entity

The entity represents a pending event:

```java
@Entity
@Table(name = "outbox_events")
@Getter
@NoArgsConstructor
public class OutboxEvent {

    @Id
    private UUID id;

    @Column(nullable = false, length = 100)
    private String aggregateType;

    @Column(nullable = false)
    private UUID aggregateId;

    @Column(nullable = false, length = 100)
    private String eventType;

    @Column(nullable = false, columnDefinition = "jsonb")
    private String payload;

    @Column(nullable = false)
    private Instant createdAt;

    private Instant publishedAt;

    public OutboxEvent(
            UUID id,
            String aggregateType,
            UUID aggregateId,
            String eventType,
            String payload
    ) {
        this.id = id;
        this.aggregateType = aggregateType;
        this.aggregateId = aggregateId;
        this.eventType = eventType;
        this.payload = payload;
        this.createdAt = Instant.now();
    }

    public void markPublished() {
        this.publishedAt = Instant.now();
    }
}
```

---

# 30. Writing the Outbox Event

The outbox service serializes the domain event:

```java
public void save(
        String aggregateType,
        UUID aggregateId,
        String eventType,
        Object event
) {
    try {
        String payload =
                objectMapper.writeValueAsString(event);

        OutboxEvent outboxEvent =
                new OutboxEvent(
                        UUID.randomUUID(),
                        aggregateType,
                        aggregateId,
                        eventType,
                        payload
                );

        outboxEventRepository.save(outboxEvent);

    } catch (JsonProcessingException e) {
        throw new IllegalStateException(
                "Failed to serialize outbox event",
                e
        );
    }
}
```

This method executes inside the existing business transaction.

---

# 31. DevSync Assignment Flow After Outbox

Issue assignment now effectively performs:

```text
BEGIN
    │
    ├── UPDATE issue
    │
    ├── INSERT issue_activity
    │
    └── INSERT outbox_events
    │
COMMIT
```

All three database operations belong to the same transaction.

Therefore:

```text
Business change succeeds
        +
Outbox event persisted
        =
consistent database state
```

---

# 32. Why `@TransactionalEventListener(AFTER_COMMIT)` Was Not Enough

The old approach published Kafka events after the database transaction committed.

Conceptually:

```text
DB transaction
      ↓
COMMIT
      ↓
AFTER_COMMIT listener
      ↓
Kafka publish
```

The problem is:

```text
DB COMMIT ✅
      ↓
Application crashes
      ↓
Kafka publish never happens
```

The database change cannot be rolled back anymore.

The Outbox pattern removes this failure gap by persisting the event before the transaction commits.

---

# 33. Outbox Publisher

A scheduled publisher periodically reads unpublished events.

```java
@Scheduled(fixedDelay = 5000)
public void publishPendingEvents() {
    outboxPublisher.publishPendingEvents();
}
```

The publisher:

```text
Find unpublished events
        ↓
Convert payload
        ↓
Create Kafka event
        ↓
Publish to Kafka
        ↓
Mark outbox event as published
```

---

# 34. Selecting Unpublished Events

The repository uses:

```sql
SELECT *
FROM outbox_events
WHERE published_at IS NULL
ORDER BY created_at
LIMIT 100
FOR UPDATE SKIP LOCKED
```

This provides two useful properties:

### `FOR UPDATE`

Rows selected by the transaction are locked.

### `SKIP LOCKED`

Another publisher can skip rows already locked by another publisher.

This allows multiple publisher workers to process different outbox rows without waiting on each other.

---

# 35. Outbox Publisher Transaction

The publisher uses a transaction around the publishing operation.

Conceptually:

```text
BEGIN
    │
    ├── SELECT unpublished events
    │
    ├── Publish event to Kafka
    │
    └── mark published_at
    │
COMMIT
```

If Kafka publishing fails:

```text
BEGIN
    │
    ├── Kafka publish ❌
    │
ROLLBACK
```

The outbox event remains unpublished.

It can therefore be attempted again during the next scheduler execution.

---

# 36. Synchronous Kafka Publishing for Outbox

The outbox publisher uses synchronous Kafka publishing:

```java
kafkaTemplate
        .send(topic, key, event)
        .get();
```

This allows the publisher to know whether Kafka accepted the record before marking the outbox row as published.

Conceptually:

```text
send()
   ↓
wait for Kafka result
   ↓
success?
 ┌─┴─┐
YES  NO
 ↓    ↓
mark  rollback
published
```

---

# 37. Stable Event Identity

The Outbox event ID is used as the Kafka event ID.

```java
new IssueAssignedKafkaEvent(
        event.getId(),
        ...
);
```

Therefore:

```text
Outbox ID
    ↓
Kafka eventId
    ↓
Idempotency record
```

The same logical event retains the same identity even if it is published multiple times.

This is critical for duplicate detection.

---

# 38. Outbox Crash Window

There is still a small crash window:

```text
Outbox event
      ↓
Publish to Kafka ✅
      ↓
💥 Application crashes
      ↓
markPublished() never completes
```

The Kafka record exists, but:

```text
published_at = NULL
```

Therefore the scheduler publishes the event again.

This is expected.

---

# 39. Why Duplicate Outbox Publishing Is Safe

Suppose:

```text
Outbox Event E1
      ↓
Kafka publish ✅
      ↓
Application crashes
```

Next scheduler run:

```text
E1 still unpublished
      ↓
Kafka publish again
```

Kafka may therefore contain:

```text
E1
E1
```

The consumer receives the event twice.

The idempotency mechanism handles this:

```text
eventId
   ↓
processed_kafka_events
   ↓
already exists
   ↓
skip business operation
```

Therefore:

```text
Transactional Outbox
        +
At-least-once Kafka delivery
        +
Idempotent consumer
        =
Reliable event processing
```

---

# 40. Outbox Crash Test

The crash scenario was tested.

Observed behavior:

```text
Kafka publish succeeds
        ↓
Application crashes before markPublished()
        ↓
published_at remains NULL
        ↓
Scheduler runs again
        ↓
Event published again
        ↓
Consumer receives duplicate
        ↓
Idempotency prevents duplicate business effect
```

After disabling the simulated crash:

```text
published_at
     ↓
updated successfully
```

This confirmed the expected Outbox behavior.

---

# 41. Current Producer Reliability

Kafka producer configuration includes:

```text
acks = -1
enable.idempotence = true
```

Therefore the producer waits for acknowledgement from all in-sync replicas and uses Kafka producer idempotence.

The producer also uses:

```text
key = issueId
```

for issue events.

---

# 42. Consumer Concurrency

The DevSync topic currently has:

```text
3 partitions
```

The notification consumer was configured with:

```java
@KafkaListener(
        topics = "devsync.issue-events",
        groupId = "devsync-notification-service",
        concurrency = "3"
)
```

This creates three Kafka consumers within the same application instance.

Architecture:

```text
devsync.issue-events

Partition 0 → Consumer 1
Partition 1 → Consumer 2
Partition 2 → Consumer 3
```

---

# 43. Consumer Concurrency and Partitions

Consumer parallelism is limited by the number of partitions.

Rule:

```text
Active consumers =
min(total consumer threads, number of partitions)
```

Example:

```text
3 partitions
6 consumers
```

Result:

```text
Consumer 1 → P0
Consumer 2 → P1
Consumer 3 → P2
Consumer 4 → idle
Consumer 5 → idle
Consumer 6 → idle
```

Increasing consumers beyond the number of partitions does not increase partition-level parallelism.

---

# 44. DevSync Concurrency Verification

The application logs confirmed three consumers joined the group:

```text
consumer-devsync-notification-service-1
consumer-devsync-notification-service-2
consumer-devsync-notification-service-3
```

Kafka assigned:

```text
Consumer 1 → devsync.issue-events-0
Consumer 2 → devsync.issue-events-1
Consumer 3 → devsync.issue-events-2
```

Therefore:

```text
3 partitions
+
3 consumer threads
=
3 active consumers
```

The concurrency configuration is working as expected.

---

# 45. Offset Management

Current configuration:

```yaml
spring:
  kafka:
    consumer:
      enable.auto.commit: false
```

This disables Kafka consumer auto-commit.

The application/framework controls when offsets are committed.

Important distinction:

```text
enable.auto.commit=false
```

does **not** mean:

```text
No offset commits
```

It means:

```text
Kafka's automatic consumer offset commit mechanism
is disabled.
```

Spring Kafka can still manage commits.

---

# 46. Offset Processing Model

The desired processing model is:

```text
Kafka
  ↓
Receive record
  ↓
Process business operation
  ↓
SUCCESS
  ↓
Offset committed
```

If processing fails:

```text
Kafka
  ↓
Receive record
  ↓
Processing fails
  ↓
Offset not successfully committed
  ↓
Error handler
  ↓
Retry
```

This prevents committing a record before its business operation succeeds.

---

# 47. Dangerous Offset Ordering

Unsafe pattern:

```text
Receive record
      ↓
Commit offset
      ↓
Process business logic
      ↓
💥 crash
```

Kafka believes the record was successfully processed.

The record may not be delivered again.

This can result in:

```text
Message acknowledged
+
Business operation never completed
=
Potential data/business-effect loss
```

Therefore:

> Process first, commit after successful processing.

---

# 48. Spring Kafka `AckMode`

Spring Kafka provides acknowledgment modes that control when offsets are committed.

Important modes include:

```text
RECORD
BATCH
MANUAL
MANUAL_IMMEDIATE
```

### `RECORD`

Offset is committed after successful processing of a record.

Conceptually:

```text
Record
  ↓
Process
  ↓
Commit
```

### `BATCH`

Records returned by a poll are processed as a batch and offsets are committed after successful batch processing.

### `MANUAL`

The listener explicitly acknowledges the record:

```java
acknowledgment.acknowledge();
```

### `MANUAL_IMMEDIATE`

Similar to manual acknowledgment but requests immediate commitment when possible.

---

# 49. Manual Acknowledgment

Manual acknowledgment can be useful when the application needs explicit control over when a record is considered successfully processed.

Example:

```java
@KafkaListener(...)
public void consume(
        IssueAssignedKafkaEvent event,
        Acknowledgment acknowledgment
) {
    try {
        notificationService.createNotification(event);

        acknowledgment.acknowledge();

    } catch (Exception e) {
        throw e;
    }
}
```

The important rule is:

```text
SUCCESS → acknowledge

FAILURE → do not acknowledge
```

If acknowledgment is forgotten or placed incorrectly, offset management can become incorrect.

For the current DevSync notification consumer, manual acknowledgment is not necessary. Spring Kafka's normal acknowledgment lifecycle together with the error handler is sufficient.

---

# 50. `AckMode` vs `DefaultErrorHandler`

These solve different problems.

### AckMode

Answers:

```text
WHEN should the offset be committed?
```

### DefaultErrorHandler

Answers:

```text
WHAT should happen when processing fails?
```

DevSync uses:

```text
Ack/offset management
        +
DefaultErrorHandler
        +
FixedBackOff
        +
DeadLetterPublishingRecoverer
```

Together they provide the consumer failure-handling mechanism.

---

# 51. Consumer Crash and Rebalance

A consumer group continuously tracks its members.

Example:

```text
3 partitions
3 consumers

P0 → C1
P1 → C2
P2 → C3
```

If C1 crashes:

```text
C1 💥
```

Kafka detects that the consumer has left the group.

A rebalance occurs.

The partitions are reassigned among the remaining consumers.

Conceptually:

```text
Before:

P0 → C1
P1 → C2
P2 → C3


C1 crashes


After rebalance:

P0 → C2/C3
P1 → C2/C3
P2 → C2/C3
```

The exact assignment depends on the partition assignment strategy.

The important concept is:

> A partition previously owned by a failed consumer can be reassigned to another consumer in the same group.

---

# 52. Consumer Crash Before Offset Commit

Consider:

```text
Consumer C1
      ↓
Event E42
      ↓
DB notification created
      ↓
💥 C1 crashes
      ↓
Offset 42 not committed
```

After rebalance:

```text
Consumer C2
      ↓
receives E42 again
```

This is expected with at-least-once processing.

---

# 53. DevSync Handling of Redelivery After Rebalance

C2 receives E42 again:

```text
E42
 ↓
processedKafkaEventRepository.existsById(E42.eventId())
 ↓
true
 ↓
return
```

The notification is not created again.

Therefore:

```text
Kafka delivery:
E42
E42

Business effect:
1 notification
```

This demonstrates:

```text
At-least-once delivery
        +
Idempotent consumer
        =
Safe duplicate delivery
```

---

# 54. Consumer Crash After Offset Commit

Another scenario:

```text
C1
 ↓
E42
 ↓
business operation succeeds
 ↓
offset committed
 ↓
💥 C1 crashes
```

After rebalance, the new consumer resumes after the committed offset.

The event does not need to be processed again.

---

# 55. Consumer Crash Before Business Processing

If the consumer crashes before processing:

```text
C1
 ↓
E42 received
 ↓
💥 crash
```

No business operation occurred.

The offset was not successfully committed.

After reassignment:

```text
C2
 ↓
E42
 ↓
process normally
```

The event is successfully processed.

---

# 56. Consumer Group Rebalance

A rebalance can happen when group membership changes.

Examples include:

```text
Consumer joins
Consumer leaves
Consumer crashes
Consumer becomes unavailable
Partition assignment changes
```

Kafka redistributes partitions among the active consumers.

The consumer then resumes processing from its committed offset.

---

# 57. `session.timeout.ms` vs `max.poll.interval.ms`

Kafka has multiple mechanisms related to consumer liveness.

### `session.timeout.ms`

Conceptually asks:

```text
Is the consumer still alive?
```

If heartbeats stop for too long, Kafka can consider the consumer dead.

### `max.poll.interval.ms`

Conceptually asks:

```text
Is the consumer taking too long between poll() calls?
```

If processing takes too long and the consumer does not poll within the configured interval, Kafka can consider it unresponsive and trigger a rebalance.

Simplified mental model:

```text
session.timeout.ms
    ↓
consumer liveness / heartbeat

max.poll.interval.ms
    ↓
maximum time between successful polls
```

---

# 58. Partition Ordering

Kafka guarantees ordering **within a partition**.

It does not provide global ordering across all partitions.

DevSync currently has:

```text
P0
P1
P2
```

Each partition has its own ordered sequence of offsets.

Example:

```text
P0:

offset 100 → Event A
offset 101 → Event B
offset 102 → Event C
```

A consumer reads these records in partition order.

---

# 59. Same Issue Ordering

DevSync uses:

```text
issueId
```

as the Kafka message key.

Therefore events for the same issue are routed to the same partition according to the producer's partitioning strategy.

Example:

```text
Issue I100

I100 → Assigned
I100 → IN_PROGRESS
I100 → DONE
```

Conceptually:

```text
I100
 ↓
same Kafka key
 ↓
same partition
 ↓
ordered offsets
```

Therefore:

```text
Assigned
   ↓
IN_PROGRESS
   ↓
DONE
```

can preserve ordering for that issue.

---

# 60. Different Issues Can Be Processed Concurrently

Suppose:

```text
I100 → P0
I200 → P1
I300 → P2
```

With:

```text
C1 → P0
C2 → P1
C3 → P2
```

the events can be processed concurrently:

```text
C1 → I100
C2 → I200
C3 → I300
```

This gives DevSync:

```text
Same issue
    ↓
Ordered processing

Different issues
    ↓
Potential parallel processing
```

This is the main reason for choosing `issueId` as the Kafka key.

---

# 61. No Global Ordering

Consider:

```text
P0:
I100-E1
I100-E2

P1:
I200-E1
I200-E2
```

Kafka guarantees:

```text
I100-E1 → I100-E2
```

and:

```text
I200-E1 → I200-E2
```

But Kafka does not guarantee an ordering such as:

```text
I100-E1
I200-E1
I100-E2
I200-E2
```

across the two partitions.

Therefore:

> Kafka ordering is a partition-level guarantee, not a topic-wide global guarantee.

---

# 62. Why the Kafka Key Matters

The key determines which partition a record is routed to.

DevSync:

```text
key = issueId
```

Therefore:

```text
same issue
    ↓
same key
    ↓
same partition
    ↓
partition ordering
```

while:

```text
different issues
    ↓
potentially different partitions
    ↓
parallel processing
```

This gives the desired balance between ordering and scalability.

---

# 63. Important Ordering Limitation

Partition ordering should not be interpreted as permanent global identity for a key.

If the topic's partition count changes, the partition selected for a key can change according to the producer's partitioning behavior.

Therefore the important application-level rule is:

```text
Events for the same logical key should be
consistently partitioned when ordering is required.
```

Kafka's ordering guarantee itself remains:

```text
ORDERING = WITHIN A PARTITION
```

---

# 64. Current DevSync Kafka Architecture

The architecture now looks like:

```text
                         DevSync
                            │
                            ▼
                       IssueService
                            │
                            ▼
                     Database Transaction
                       │           │
                       │           └── Outbox Event
                       │
                       └── Business Data
                            │
                            ▼
                       PostgreSQL
                            │
                            ▼
                     Outbox Publisher
                            │
                            ▼
                   devsync.issue-events
                            │
              ┌─────────────┼─────────────┐
              │             │             │
             P0            P1            P2
              │             │             │
             C1            C2            C3
              │             │             │
              └─────────────┼─────────────┘
                            │
                            ▼
              IssueAssignmentNotificationService
                            │
                            ▼
                  Idempotency Check
                            │
                   ┌────────┴────────┐
                   │                 │
                 NEW             ALREADY PROCESSED
                   │                 │
                   ▼                 ▼
             Create DB          Skip operation
             notification
                   │
                   ▼
             Commit offset

Failure:
    ↓
DefaultErrorHandler
    ↓
Retry
    ↓
Retries exhausted
    ↓
DLT
```

---

# 65. Current Kafka Reliability Model

DevSync now uses multiple reliability mechanisms.

```text
Producer
    │
    ├── acks=-1
    └── idempotent producer
    │
    ▼
Transactional Outbox
    │
    ├── Business data
    └── Outbox event
          ↓
      same DB transaction
          │
          ▼
    Outbox Publisher
          │
          ▼
        Kafka
          │
          ▼
    Consumer Group
          │
          ├── Partition ordering
          ├── Consumer concurrency
          └── Offset management
          │
          ▼
    Business Processing
          │
          ├── Idempotency
          ├── Retry
          └── DLT
```

---

# 66. Reliability Mental Model

The complete model is:

```text
Transactional Outbox
        ↓
prevents DB/Kafka dual-write inconsistency

At-least-once delivery
        ↓
allows safe redelivery

Idempotent consumer
        ↓
prevents duplicate business effects

Offset management
        ↓
tracks successful processing

Retry
        ↓
handles temporary failures

DLT
        ↓
isolates permanently failing messages

Consumer groups
        ↓
provide scalable consumption

Partitions
        ↓
provide parallelism

Kafka keys
        ↓
provide partition affinity and ordering
```

---

# 67. What We Have Covered So Far

### Kafka Fundamentals

- Topics
- Partitions
- Records
- Keys
- Offsets
- Consumer groups
- Partition ordering

### Producer Reliability

- `acks=-1`
- Producer idempotence
- Key-based partitioning
- Kafka-specific event DTOs

### Consumer Reliability

- `enable.auto.commit=false`
- Offset management
- Ack modes
- At-least-once delivery
- Idempotent consumers
- Retry
- Backoff
- DLT
- Consumer crashes
- Rebalancing

### Transactional Outbox

- Dual-write problem
- Outbox table
- Outbox entity
- Same DB transaction for business data + event
- Outbox publisher
- `FOR UPDATE SKIP LOCKED`
- Synchronous Kafka publishing
- Stable event IDs
- Crash-window handling
- Duplicate outbox publishing

### Kafka Scalability

- Consumer concurrency
- Partition-to-consumer assignment
- Maximum useful concurrency
- Parallel processing
- Same-key ordering
- Partition-level ordering
- No global topic ordering

---

# 68. DevSync Kafka Progress

Completed:

```text
[✓] Kafka infrastructure
[✓] Producer
[✓] Consumer
[✓] Kafka DTO
[✓] Topic + partitions
[✓] Keys
[✓] Consumer groups
[✓] Retry
[✓] DLT
[✓] Idempotent consumer
[✓] At-least-once model
[✓] Transactional Outbox
[✓] Outbox crash-window test
[✓] Consumer concurrency
[✓] Offset management concepts
[✓] Consumer crash + rebalance
[✓] Partition ordering
```

---

# 69. Next Topic

## Advanced Consumer Failure Handling

The next problem is what happens when processing itself is slow or repeatedly fails.

Topics to cover:

1. Blocking retries
2. Non-blocking retries
3. `@RetryableTopic`
4. Retry topics
5. Why blocking one partition can affect later records
6. Retry topic architecture
7. When to use blocking vs non-blocking retry
8. Ordering implications of retry topics
9. Choosing the correct retry strategy for DevSync

The key problem:

```text
Partition 0

Event A
Event B
Event C
```

Suppose:

```text
Event A → takes 30 seconds / repeatedly fails
```

If the consumer blocks on A:

```text
A ❌
 ↓
retry
 ↓
retry
 ↓
retry
 ↓
B waits
 ↓
C waits
```

This introduces a new scalability and ordering problem.

The next step is to understand how Kafka retry strategies deal with this.
