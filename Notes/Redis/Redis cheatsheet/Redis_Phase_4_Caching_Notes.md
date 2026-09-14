# Redis Phase 4 — Caching Architecture & Production Patterns

> **Core idea:** Caching is not simply putting data into Redis. It is an architectural trade-off between **performance, freshness, consistency, memory, complexity, and failure handling**.

---

## 1. Why Do We Need a Cache?

Without caching:

```text
Client
  ↓
Spring Boot
  ↓
PostgreSQL
  ↓
Response
```

Repeated requests can repeatedly consume:

- Database query/CPU resources
- Network latency
- Connection-pool capacity
- Memory/I/O
- Application-side mapping/serialization work

With a cache:

```text
Client
  ↓
Spring Boot
  ↓
Redis
  ↓
PostgreSQL only when necessary
```

Frequently requested data can be served without repeatedly querying PostgreSQL.

### The fundamental trade-off

```text
Lower latency
+
Lower database load
+
Less repeated work
        ↓
        BUT
        ↓
Potential stale data
+
Extra memory
+
Invalidation complexity
+
Failure handling
```

> **Caching trades some combination of freshness, consistency, memory, and complexity for performance.**

Do not think: **"Redis is faster than PostgreSQL, therefore cache everything."**

The real question is whether the performance benefit justifies the added complexity and memory cost.

---

# 2. What Should Be Cached?

Good candidates tend to have:

- High read frequency
- Relatively low/moderate change frequency
- Expensive queries or computation
- Shared results across requests
- Tolerable staleness

Examples:

```text
User profile
Issue details
Workspace metadata
Dashboard statistics
Expensive search results
Configuration
```

Poor candidates often include:

- Highly volatile data
- Data requiring strict consistency
- Rarely accessed data
- Extremely large objects
- Data that is already cheap to retrieve
- Sensitive data without a clear security model

### Cache-candidate mental model

```text
Read frequency
      +
Retrieval/computation cost
      +
Acceptable staleness
      +
Memory cost
      +
Invalidation complexity
      ↓
Should this be cached?
```

There is no universal rule.

For example, a frequently changing dashboard may still be worth caching if its aggregation is expensive and the business allows 30 seconds of staleness.

---

# 3. Cache Hit and Cache Miss

## Cache Hit

```text
Request
  ↓
Redis GET
  ↓
Value exists
  ↓
HIT
  ↓
Return value
```

## Cache Miss

```text
Request
  ↓
Redis GET
  ↓
MISS
  ↓
Database
  ↓
Populate cache
  ↓
Return value
```

### Cache Hit Ratio

```text
Hit Ratio = Cache Hits / Total Cache Requests
```

Example:

```text
100,000 requests
90,000 hits
10,000 misses

Hit ratio = 90%
```

A high hit ratio is useful, but **does not prove the cache is valuable**.

You also need to consider:

- Redis latency
- Serialization cost
- Database query cost
- Database load reduction
- Application latency

A cache that eliminates a 500 ms database operation is much more valuable than one that eliminates a 1 ms operation.

---

# 4. Cache-Aside ⭐

Cache-Aside is the most important caching pattern for typical Spring Boot + Redis applications.

### Read

```text
Application
    ↓
Redis GET
    ↓
 HIT ─────────→ Return
    │
   MISS
    ↓
PostgreSQL
    ↓
Redis SET
    ↓
Return
```

The defining characteristic:

> **The application manages the cache.**

Redis does not automatically know how to retrieve the missing object from PostgreSQL.

### Why Cache-Aside is popular

- Simple mental model
- Application has explicit control
- PostgreSQL remains the source of truth
- Easy to adopt incrementally
- Redis can be treated as a disposable copy

---

# 5. Cache-Aside Write Flow

A common approach is:

```text
PostgreSQL UPDATE
       ↓
Redis DELETE
```

Example:

```text
DB:
OPEN → CLOSED

Redis:
DELETE issue:123
```

Next read:

```text
Redis MISS
   ↓
PostgreSQL → CLOSED
   ↓
Redis SET → CLOSED
```

### Why delete instead of updating Redis?

Updating both systems creates a dual-write problem:

```text
DB UPDATE
   +
Redis SET
```

Possible failures:

```text
DB succeeds
Redis fails

Redis succeeds
DB fails

Application crashes between operations
```

With deletion:

```text
DB = source of truth
Redis = disposable copy
```

The cache can always be rebuilt from PostgreSQL.

### Important limitation

`DB UPDATE → Redis DELETE` does **not** guarantee strong consistency.

A concurrent reader can still do:

```text
A → DB UPDATE → CLOSED

B → Redis GET → OPEN

A → Redis DELETE
```

B has already obtained stale data before the cache was deleted.

---

# 6. TTL and Stale Data

TTL is useful as a cache safety mechanism.

Example:

```text
DB:
status = OPEN

Redis:
status = OPEN
TTL = 5 minutes
```

Then:

```text
DB:
OPEN → CLOSED
```

Redis may still contain:

```text
OPEN
```

until invalidation or expiration.

### Critical distinction

> **TTL limits the lifetime of an uncorrected cache entry; it does not guarantee consistency.**

A value can become stale immediately after being cached:

```text
10:00:00 → Redis stores OPEN
10:00:01 → DB changes to CLOSED
```

The cache can now remain stale for most of its remaining TTL.

Therefore:

```text
TTL ≠ consistency
```

TTL and explicit invalidation are often used together:

```text
DB UPDATE
   ↓
Redis DELETE
   ↓
TTL as safety net if deletion fails
```

TTL should be chosen based on the application's **freshness requirement**, not because a particular number is considered standard.

---

# 7. Cache Invalidation

The fundamental problem:

```text
Database changes
        ↓
Cache still contains old value
```

Common approaches:

### Explicit invalidation

```text
DB UPDATE
   ↓
DELETE cache
```

### Cache update

```text
DB UPDATE
   ↓
SET new value in cache
```

### TTL-only

```text
DB UPDATE
   ↓
wait for expiration
```

### Event-driven invalidation

```text
DB change
   ↓
event
   ↓
cache invalidation
```

TTL-only is simple but can produce a large stale-data window.

Explicit invalidation reduces the stale-data window but can still fail or race with concurrent reads.

---

# 8. Read/Write Ordering and Failure Scenarios

## DB succeeds, Redis delete fails

```text
DB = CLOSED
Redis = OPEN
```

Redis contains stale data.

If the stale entry eventually expires:

```text
Redis MISS
   ↓
DB → CLOSED
   ↓
Redis SET
```

the cache repairs itself.

## Redis update succeeds, DB update fails

```text
DB = OPEN
Redis = CLOSED
```

Now the cache contains a value that never became authoritative.

This is one reason ordinary Cache-Aside prefers:

```text
DB UPDATE → Redis DELETE
```

rather than treating Redis as another authoritative write target.

## Application crashes between operations

```text
DB UPDATE succeeds
        ↓
Application crashes
        ↓
Redis DELETE never happens
```

Again:

```text
DB = correct
Redis = potentially stale
```

### Key principle

Two operations across two independent systems are **not one atomic transaction**.

Redis `MULTI/EXEC` cannot make a PostgreSQL update and Redis operation one distributed transaction.

---

# 9. Cache Stampede ⭐

A cache stampede occurs when many requests simultaneously miss the **same cache entry** and all try to regenerate it.

```text
Popular key
    ↓
expires
    ↓
1,000 requests
    ↓
1,000 Redis MISS
    ↓
1,000 PostgreSQL queries
```

This can create a sudden database load spike and high latency.

### Why TTL can contribute

A popular key can expire at exactly the wrong time:

```text
Cache expires
     ↓
large request burst
     ↓
all requests miss
```

---

# 10. Stampede Mitigation

## Request Coalescing / Single-Flight

Only one request regenerates the value:

```text
1,000 requests
      ↓
Redis MISS
      ↓
1 request → DB
999 requests → wait/share result
      ↓
Redis SET
      ↓
1,000 responses
```

This directly prevents duplicate database work.

In a multi-instance application, in-process coordination only coordinates requests within one instance; cross-instance coordination is a separate concern.

## Lock-Based Regeneration

```text
Redis MISS
   ↓
acquire regeneration lock
   ↓
one request → DB
others → wait/retry/fallback
   ↓
Redis SET
   ↓
release lock
```

Locks are not automatically required. They introduce their own failure and timeout concerns.

## Randomized TTL / TTL Jitter

Instead of:

```text
TTL = 60s for everything
```

use:

```text
55s
61s
64s
68s
...
```

This spreads expiration across time.

**Important:** TTL jitter helps spread expiration across many keys; it does not guarantee that only one request regenerates one hot key.

## Stale-While-Revalidate

Serve a slightly stale value while refreshing asynchronously:

```text
stale value
   ↓
serve immediately

background
   ↓
DB
   ↓
Redis refresh
```

Useful when stale data is acceptable and latency is important.

## Early Refresh / Refresh-Ahead

Refresh a popular entry before it expires:

```text
TTL nearing expiry
       ↓
background refresh
       ↓
new value + new TTL
```

Useful for predictable hot data.

---

# 11. Cache Penetration

Cache penetration occurs when requests repeatedly ask for data that **does not exist**.

```text
GET user:999999
      ↓
Redis MISS
      ↓
PostgreSQL → NOT FOUND
```

Repeated requests repeatedly hit PostgreSQL.

### Negative caching

Cache the absence:

```text
user:999999 → NOT_FOUND
TTL = short
```

Then:

```text
Request
  ↓
Redis
  ↓
NOT_FOUND
  ↓
return 404
```

When the user is later created:

```text
DB INSERT
   ↓
DELETE negative cache
```

A short TTL remains a fallback if deletion fails.

Other protection includes:

- Input validation
- Bloom filters conceptually

### Key distinction

```text
Penetration → requested data does not exist
Stampede    → existing data expires and many requests regenerate it
```

---

# 12. Cache Avalanche

Avalanche is a large-scale expiration/invalidation problem where many cached keys become unavailable around the same time.

```text
100,000 keys
     ↓
similar TTLs
     ↓
expire together
     ↓
huge DB traffic spike
```

Typical mitigations:

- TTL jitter
- Staggered expiration
- Cache warming
- Capacity planning
- Graceful degradation

### Distinction

```text
Penetration → nonexistent data
Stampede    → same popular key
Avalanche   → many keys
```

---

# 13. Hot Keys

A hot key is a single key receiving disproportionate traffic.

Example:

```text
issue:123
    ↓
100,000 requests/sec
```

The problem is not that the value is missing.

The problem is **concentrated traffic around one key**.

Possible approaches:

- Local application caching
- Request coalescing
- Read distribution/replication
- Key splitting or replication where appropriate

Hot-key detection should be part of production observability.

---

# 14. Big Keys

A big key contains a very large amount of data.

Example:

```text
workspace:17:all-issues
    ↓
hundreds of thousands of issues
```

Problems:

- High memory usage
- Large network payloads
- Serialization/deserialization cost
- Slow operations
- Difficult deletion
- Greater operational risk

Instead of one huge representation:

```text
workspace:17:all-issues
```

you may prefer smaller cache units:

```text
issue:100
issue:101
issue:102
...
```

or carefully designed aggregated representations.

> **Redis data-structure choice is also a cache-design decision.**

---

# 15. Negative Caching

Negative caching stores the fact that an object does not exist.

```text
user:999 → NOT_FOUND
TTL = short
```

Benefits:

- Protects PostgreSQL from repeated invalid requests
- Useful against cache penetration
- Short TTL limits the impact of newly-created data

Failure scenario:

```text
Negative cache exists
      ↓
User gets created
      ↓
Redis DELETE fails
      ↓
NOT_FOUND remains temporarily
      ↓
short TTL expires
```

Therefore negative-cache TTLs should generally be much shorter than normal object-cache TTLs when freshness matters.

---

# 16. Cache Strategies

## Cache-Aside

```text
Application → Cache
                  ↓ miss
               Database
```

Application manages cache reads/misses/invalidation.

**Common default for Spring Boot + Redis.**

## Read-Through

```text
Application
    ↓
Cache
    ↓
Database on miss
```

The cache layer handles loading missing data.

## Write-Through

```text
Application
    ↓
Cache
    ↓
Database
```

Cache participates in synchronous writes.

Benefit: cache can remain fresh.

Cost: more complex write/failure behavior.

## Write-Behind / Write-Back

```text
Application
    ↓
Cache
    ↓
async persistence
    ↓
Database
```

Very fast writes, but introduces significant durability and consistency risks.

## Refresh-Ahead

```text
Entry nearing expiration
        ↓
refresh before expiry
```

Useful for predictable hot data.

### Comparison

| Strategy | Main responsibility | Main benefit | Main risk/cost |
|---|---|---|---|
| Cache-Aside | Application | Simple/flexible | Invalidation complexity |
| Read-Through | Cache | Simplified reads | More cache-layer complexity |
| Write-Through | Cache + DB | Fresh cache | Complex writes/failures |
| Write-Behind | Cache + async DB | Very fast writes | Data-loss/consistency risk |
| Refresh-Ahead | Cache/application | Avoid expiry misses | Extra background work |

Do not assume the more complex pattern is automatically better.

---

# 17. Cache Consistency Models

Caching must match the business requirement.

## Strong consistency

The application should effectively see the latest authoritative state.

Examples may include:

- Critical authorization decisions
- Certain financial/accounting state

Do not casually cache such data.

## Bounded staleness

Data can be stale, but only within an acceptable window.

Example:

```text
Dashboard
Maximum acceptable staleness = 30 sec
```

## Eventual consistency

Temporary divergence is acceptable as long as the cache eventually converges.

Example:

```text
DB → CLOSED
Redis → OPEN
    ↓
invalidation/expiration
    ↓
Redis → CLOSED
```

## Best effort

Cache is purely an optimization; if unavailable, another behavior is used.

> **Consistency is a business requirement, not a Redis property.**

---

# 18. Redis + PostgreSQL Architecture

For a normal cache:

```text
              Spring Boot
               /       \
              /         \
           Redis     PostgreSQL
          Cache      Source of Truth
```

The important principle:

> **If Redis is only a cache, losing Redis should not destroy authoritative application state.**

If Redis disappears:

```text
Redis ❌
PostgreSQL ✅
```

the application may be able to fall back to PostgreSQL and rebuild the cache.

---

# 19. Redis Failure Strategies

## Fail Open

```text
Redis unavailable
      ↓
skip cache
      ↓
PostgreSQL
```

Suitable when Redis is merely an optimization.

Risk:

```text
Redis outage
    ↓
DB traffic spike
    ↓
DB overload
```

## Fail Closed

```text
Redis unavailable
      ↓
reject request
```

Can be appropriate when Redis is essential to correctness or enforcement.

## Degraded Mode

```text
Redis unavailable
      ↓
core functionality continues
optional/expensive functionality disabled
```

Different Redis roles can require different behavior:

```text
Issue cache      → fail open may be reasonable
Rate limiter     → may require fail closed/degraded behavior
Session-only data → DB fallback may not exist
```

---

# 20. Cache Warming

After a restart or flush:

```text
PostgreSQL
    ↓
Redis = empty
```

### Lazy warming

```text
Request
  ↓
MISS
  ↓
DB
  ↓
Redis SET
```

Simple, but a cold cache can cause a large DB spike.

### Prewarming

```text
Redis restart
    ↓
load important/hot data
    ↓
accept normal traffic
```

Trade-off:

```text
Lazy warming → simple, slower recovery
Prewarming   → higher startup cost, faster warm state
```

---

# 21. Cache Value Design

For a Java object:

```text
Java Issue
   ↓
Jackson
   ↓
JSON
   ↓
Redis String
```

Example:

```json
{
  "id": 900,
  "title": "Fix login bug",
  "status": "OPEN",
  "priority": "HIGH"
}
```

### String + JSON

Good when most requests need the **entire object**.

Advantages:

- Simple
- One GET
- Natural DTO/object representation

Costs:

- Full serialization/deserialization
- Full object transfer
- Updating one field means replacing the representation

### Redis Hash

```text
issue:900
 ├── title
 ├── status
 ├── priority
 └── assignee
```

Useful when field-level access/update is genuinely important.

Don't choose Hash simply because it is more granular.

> **Choose the representation based on the application's dominant access pattern.**

For DevSync issue caching where most requests need the complete issue, serialized JSON in a String is a reasonable starting design.

---

# 22. Serialization and Schema Evolution

Cache hits still have work:

```text
Redis GET
   ↓
network
   ↓
JSON deserialization
   ↓
Java object creation
```

Serialization cost matters for large/high-volume objects.

Schema changes create another concern.

For example:

```text
v1:
{id, title}

v2:
{id, title, priority}
```

Old cached representations may not match new expectations.

Possible approaches:

- Backward-compatible schemas
- Explicit invalidation during deployments
- Versioned keys

Example:

```text
issue:v1:900
issue:v2:900
```

Don't optimize serialization before measuring its actual impact.

---

# 23. Spring Cache Abstraction — Conceptual

Spring provides:

```java
@Cacheable
@CachePut
@CacheEvict
```

Conceptually:

### `@Cacheable`

```text
Method call
   ↓
Generate key
   ↓
Check cache
   ↓
HIT → return cached value

MISS
   ↓
Execute method
   ↓
Cache result
   ↓
Return
```

### `@CachePut`

Execute the method and update the cache with the result.

### `@CacheEvict`

Remove the corresponding cache entry.

These annotations abstract caching behavior; they do not eliminate the underlying concerns of:

- TTL
- consistency
- invalidation
- failure handling
- key design
- serialization

Detailed Spring Data Redis implementation belongs in the implementation-focused phase.

---

# 24. Cache Monitoring & Observability

A cache should be measured, not assumed to help.

Important metrics:

### Cache

```text
Hit ratio
Miss ratio
Redis latency
Redis errors
Memory usage
Evictions
```

### Database

```text
Queries/sec
CPU
Connection-pool usage
Query latency
```

### Application

```text
p50 latency
p95 latency
p99 latency
Error rate
```

### Key-level

```text
Hot keys
Big keys
Unexpected key cardinality
```

A useful cache dashboard might show:

```text
Hit ratio       94%
Redis p95       2 ms
Redis p99       5 ms
DB QPS          1,200
DB CPU           42%
Redis memory     3.1 / 4 GB
Evictions       120/sec
Redis errors      0/sec
```

### Most important principle

Don't optimize for:

> "95% cache hit ratio."

Optimize for:

> **Lower application latency + lower expensive DB work + acceptable consistency + controlled memory/operational cost.**

Always establish a baseline before introducing caching.

```text
BEFORE:
DB latency
DB QPS
DB CPU
Application latency

AFTER:
Redis latency
Hit ratio
DB latency
DB QPS
DB CPU
Application latency
```

---

# 25. Production Cache Mental Model ⭐⭐⭐⭐⭐

When evaluating a caching requirement, follow this sequence:

```text
Business requirement
        ↓
What data is read?
        ↓
How frequently?
        ↓
How expensive is retrieval?
        ↓
How stale can it be?
        ↓
Should it be cached?
        ↓
Cache key design
        ↓
Value representation
        ↓
Caching strategy
        ↓
TTL
        ↓
Invalidation
        ↓
Stampede / penetration / avalanche protection
        ↓
Redis failure behavior
        ↓
Monitoring
```

The central architecture:

```text
              CACHE
                │
       ┌────────┼────────┐
       ↓        ↓        ↓
 Performance Consistency Memory
       │        │        │
       └────────┼────────┘
                ↓
         Failure handling
                ↓
        Production design
```

Caching is **not**:

```text
Database
   ↓
Copy to Redis
   ↓
Done
```

It is:

```text
Requirement
    ↓
Freshness
    ↓
Performance
    ↓
Cache strategy
    ↓
Key/value design
    ↓
TTL
    ↓
Invalidation
    ↓
Failure handling
    ↓
Observability
```

---

# 26. Interview Quick Reference

### What is caching?

Caching stores frequently needed data in a faster-access layer to reduce repeated expensive work, at the cost of memory and additional consistency/operational complexity.

### What is Cache-Aside?

The application checks the cache first. On a miss it reads the source of truth, populates the cache, and returns the result. On writes, it separately handles cache invalidation or refresh.

### Why is Cache-Aside common?

It is simple, flexible, and keeps the database as the source of truth while Redis remains a disposable performance layer.

### Does TTL solve cache invalidation?

No.

TTL bounds how long an uncorrected cache entry can remain, but the value can become stale immediately after being cached.

### What is cache stampede?

Many concurrent requests miss the same cached value and independently regenerate it, potentially overwhelming the database.

### Stampede vs avalanche?

```text
Stampede  → same popular key
Avalanche  → many keys
```

### What is cache penetration?

Repeated requests for data that doesn't exist bypass the cache and repeatedly hit the database.

### How can penetration be mitigated?

Negative caching, validation, and Bloom filters conceptually.

### What is a hot key?

A single cache key receiving disproportionate traffic.

### What is a big key?

A single cache key containing an unusually large amount of data.

### What happens if Redis goes down?

It depends on Redis's architectural role. A pure cache can often fail open to PostgreSQL; other roles such as rate limiting or session state may require different behavior.

### Why delete cache after a DB update?

It keeps PostgreSQL authoritative and allows the cache to be rebuilt rather than requiring two independent systems to be updated consistently.

### Does DB update → cache delete guarantee strong consistency?

No. Concurrent readers and operation failures can still produce stale reads.

### How do you prevent stampede?

Depending on the workload:

- Request coalescing
- Lock-based regeneration
- TTL jitter
- Stale-while-revalidate
- Early refresh

### How do you choose TTL?

Based on business freshness requirements, access/update patterns, regeneration cost, and failure tolerance—not an arbitrary standard value.

### What should you monitor?

Hit ratio, Redis latency/errors, memory, evictions, DB load/latency, application latency, and abnormal hot/big keys.

---

# 27. Phase 4 Mental Checklist

Before saying **"I know caching"**, you should be able to reason through:

```text
□ Why should this data be cached?
□ What happens on a cache hit?
□ What happens on a miss?
□ Is Cache-Aside appropriate?
□ What is the key?
□ What is the cached representation?
□ How stale can it be?
□ What TTL makes sense?
□ When is it invalidated?
□ What if DB succeeds but Redis fails?
□ What if Redis is unavailable?
□ What if the key becomes hot?
□ What if many keys expire together?
□ What if the requested data doesn't exist?
□ What if the cached object is huge?
□ How will stampede be prevented?
□ How will cache effectiveness be measured?
```

If you can answer those questions, you're no longer treating Redis as merely a key-value store. You're reasoning about it as a **production caching architecture**.
