## 1. What is a Redis Hash?
A Redis Hash is a Redis data type that stores multiple **field-value pairs** under a single Redis key.

```text
Redis Key
    ↓
  Hash
    ├── field → value
    ├── field → value
    └── field → value
```

Example:

```text
workspace:123
    ├── name        → DevSync
    ├── owner       → alice
    ├── status      → ACTIVE
    └── memberCount → 42
```

The core mental model is:

```text
String:
key → one value

Hash:
key → { field → value }
```

A Hash is useful when the **fields themselves are meaningful units of access or update**.

---

# 2. Why do Redis Hashes exist?

The same workspace could be stored using multiple independent Strings:

```text
workspace:123:name        → DevSync
workspace:123:owner       → alice
workspace:123:status      → ACTIVE
workspace:123:memberCount → 42
```

Or as one Hash:

```text
workspace:123
    ├── name        → DevSync
    ├── owner       → alice
    ├── status      → ACTIVE
    └── memberCount → 42
```

The Hash groups related fields under one Redis key and gives direct field-level operations.

This is particularly useful when the application frequently needs to:

- read individual fields
- update individual fields
- increment numeric fields
- retrieve several selected fields

---

# 3. Hash vs String/JSON

Consider:

```text
issue:123
```

### String / JSON

```text
issue:123
    ↓
{
  "title": "Fix login bug",
  "status": "OPEN",
  "priority": "HIGH",
  "assignee": "alice"
}
```

Typical update flow:

```text
GET entire JSON
    ↓
deserialize
    ↓
modify field
    ↓
serialize
    ↓
SET entire JSON
```

### Hash

```text
issue:123
    ├── title    → Fix login bug
    ├── status   → OPEN
    ├── priority → HIGH
    └── assignee → alice
```

Field-level operations:

```text
HGET issue:123 status
HSET issue:123 status DONE
```

No need to retrieve and rewrite the entire object.

## Decision rule

Do not use:

```text
Hash = complex object
String = simple object
```

as the decision rule.

Instead ask:

> **What access and update pattern does the application require?**

### String/JSON is attractive when:

- the application usually reads the complete object
- the application usually writes the complete object
- the object is treated as one opaque value
- field-level access is relatively uncommon

### Hash is attractive when:

- fields are frequently accessed independently
- fields are frequently updated independently
- numeric fields need atomic field-level operations
- the application does not always need the complete object

Neither representation is universally better.

---

# 4. Redis Hash vs Java Map

A Redis Hash can be mentally compared to:

```java
Map<String, String>
```

But they are architecturally different.

### Java Map

```text
Application JVM
      ↓
Heap memory
      ↓
Map
```

It is local to the process.

### Redis Hash

```text
Service A ──┐
Service B ──┼── Redis Hash
Service C ──┘
```

Multiple application instances can access the same data through Redis.

Key differences:

| Aspect | Redis Hash | Java Map |
|---|---|---|
| Location | Redis server | JVM heap |
| Scope | Shared across services | Process-local |
| Network | Yes | No |
| Persistence | Depends on Redis configuration | Not inherently persistent |
| Concurrency | Redis command semantics | JVM/thread semantics |
| Access | Remote | Local |
| Sharing | Multiple applications | One process |

The logical operations may look similar, but the architecture is very different.

---

# 5. Redis Hash vs PostgreSQL Row

A Hash can look similar to a PostgreSQL row:

```text
PostgreSQL

workspace
--------------------------------
id | name | owner | status | ...
```

Redis:

```text
workspace:123
    ├── name
    ├── owner
    └── status
```

But they serve different purposes.

PostgreSQL provides capabilities such as:

- relational queries
- joins
- constraints
- durable source-of-truth storage
- transactions across relational data
- indexes
- complex filtering

A Redis Hash is primarily a fast key-based data structure.

Redis Hashes should not automatically be treated as replacements for relational rows.

A common architecture can be:

```text
PostgreSQL
    ↓
Source of truth
    ↓
Redis Hash
    ↓
Fast application access
```

Whether this architecture is appropriate depends on the workload.

---

# 6. Hash Operations

The core Hash operations learned in Phase 1 are:

```text
HSET
HGET
HMGET
HGETALL
HDEL
HEXISTS
HINCRBY
```

---

# 7. HSET

## Purpose

Set a field to a value.

```bash
HSET workspace:123 status ACTIVE
```

Conceptually:

```text
workspace:123
    ↓
  Hash
    ↓
status → ACTIVE
```

If the field does not exist:

```text
status → ACTIVE
```

is created.

If the field already exists:

```text
status → ACTIVE
```

and we execute:

```bash
HSET workspace:123 status SUSPENDED
```

the existing field is updated:

```text
status → SUSPENDED
```

Therefore:

> `HSET` is effectively an insert-or-update operation for a Hash field.

---

# 8. HGET

Retrieve one field:

```bash
HGET workspace:123 status
```

Conceptually:

```text
workspace:123
    ↓
find Hash
    ↓
find field "status"
    ↓
return value
```

For normal Hash field lookup:

```text
Average complexity: O(1)
```

The lookup is expected to remain constant-time on average as the number of fields grows, under normal hash-table behavior.

---

# 9. HMGET

Retrieve multiple fields from one Hash:

```bash
HMGET workspace:123 name owner status
```

Instead of:

```bash
HGET workspace:123 name
HGET workspace:123 owner
HGET workspace:123 status
```

`HMGET` can combine the field retrieval into one Redis request.

Conceptually:

```text
Application
     │
     │ one request
     ▼
Redis
     ├── lookup name
     ├── lookup owner
     └── lookup status
     │
     ▼
one response
```

Complexity is generally:

```text
O(N)
```

where `N` is the number of requested fields.

The important production distinction is:

```text
Redis work:
O(N)

Network interaction:
one request instead of N separate requests
```

Therefore, Big-O complexity and network round trips are separate performance considerations.

---

# 10. HGETALL

Retrieve every field and value in a Hash:

```bash
HGETALL workspace:123
```

If the Hash contains:

```text
name        → DevSync
owner       → alice
status      → ACTIVE
memberCount → 42
```

all fields and values are returned.

Complexity:

```text
O(N)
```

where `N` is the number of fields returned.

## Production warning

Do not assume:

```text
HGETALL
```

is cheap simply because individual `HGET` operations are O(1).

If a Hash contains a huge number of fields:

```text
Hash
 ↓
500,000 fields
```

then:

```bash
HGETALL large-hash
```

can result in:

```text
large Redis-side work
+
large response
+
network transfer
+
application memory usage
+
client-side processing
```

Prefer targeted operations such as:

```bash
HMGET key field1 field2
```

when only selected fields are required.

---

# 11. HDEL

Delete one or more fields from a Hash:

```bash
HDEL workspace:123 status
```

This removes only the field:

```text
workspace:123
    ├── name
    ├── owner
    └── memberCount
```

It does **not** delete the entire Hash.

Compare:

```text
HDEL workspace:123 status
```

with:

```text
DEL workspace:123
```

`HDEL` removes a field.

`DEL` removes the entire Redis key and therefore the complete Hash.

---

# 12. HEXISTS

Check whether a field exists:

```bash
HEXISTS workspace:123 owner
```

Conceptually:

```text
field exists?
    │
 ┌──┴──┐
yes    no
 │      │
 1      0
```

Complexity:

```text
O(1) average
```

`HEXISTS` directly expresses the intent:

> "Does this field exist?"

This is preferable to reconstructing the answer indirectly when existence itself is what the application needs to know.

---

# 13. HINCRBY

`HINCRBY` performs an atomic numeric update on a Hash field.

Example:

```bash
HINCRBY workspace:123 memberCount 1
```

If:

```text
memberCount → 42
```

then:

```text
memberCount → 43
```

You can also decrement:

```bash
HINCRBY workspace:123 memberCount -1
```

## Why not HGET + Java + HSET?

Unsafe pattern:

```text
HGET
    ↓
Java + 1
    ↓
HSET
```

Two application instances can read the same old value and overwrite each other's updates.

Example:

```text
Initial = 42

Server A: HGET → 42
Server B: HGET → 42

A: +1 → 43
B: +1 → 43

A: HSET → 43
B: HSET → 43
```

Final:

```text
43
```

Expected:

```text
44
```

With:

```bash
HINCRBY workspace:123 memberCount 1
```

the read-modify-write operation occurs as one atomic Redis command.

Conceptually:

```text
42
 ↓ HINCRBY 1
43
```

Concurrent increments are not lost in the same way.

---

# 14. HINCRBY on a Missing Field

If the field doesn't exist:

```bash
HINCRBY workspace:123 memberCount 1
```

Redis can initialize the missing numeric field and increment it.

Conceptually:

```text
missing
   ↓
0
   ↓
+1
   ↓
1
```

You do not need to manually perform:

```text
HEXISTS
HSET memberCount 0
HINCRBY
```

The atomic numeric operation handles the missing-field case.

---

# 15. Hash Data Model — Two Levels of Lookup

For:

```bash
HGET workspace:123 status
```

think conceptually:

```text
Redis key lookup
       ↓
workspace:123
       ↓
Hash lookup
       ↓
status
       ↓
ACTIVE
```

This differs from a String:

```text
GET user:123:name
       ↓
Redis key lookup
       ↓
Alice
```

A Hash introduces a second logical level:

```text
Redis key
    ↓
Hash
    ↓
field
    ↓
value
```

---

# 16. Hash Internal Representation

The logical Redis data type is:

```text
Hash
```

But the logical type does not dictate one fixed physical memory representation.

Conceptually:

```text
Redis Hash
     ↓
internal encoding
     ↓
memory representation
```

For small/simple Hashes, Redis can use a compact representation.

A relevant compact representation is:

```text
listpack
```

As the Hash grows or becomes more complex, Redis can use a more general hash-table-based representation.

The exact thresholds and implementation details are not the primary concern for a 2-YOE backend engineer.

The important mental model is:

```text
Small/simple Hash
      ↓
compact representation
      ↓
memory efficient

Larger/more complex Hash
      ↓
different representation
      ↓
general-purpose operations
```

Therefore:

> A Redis data type is not necessarily represented internally by one fixed data structure.

This is the same logical-vs-physical distinction used with Redis Strings.

---

# 17. Hash Memory Considerations

Consider two designs.

## Design A — One Hash

```text
user:123
    ├── name
    ├── email
    ├── status
    └── plan
```

## Design B — Multiple Strings

```text
user:123:name
user:123:email
user:123:status
user:123:plan
```

They represent similar logical information, but their memory footprints can differ.

Design B creates multiple Redis keys and therefore multiple key/dictionary/object overheads.

A Hash can group several related fields under one Redis key and may benefit from compact Hash representations for suitable workloads.

Therefore:

> Do not assume that multiple independent Strings use less memory simply because each value is small.

Memory depends on:

- key sizes
- field sizes
- number of entries
- Redis object overhead
- dictionary overhead
- internal encoding
- allocator behavior
- access pattern

---

# 18. Hash vs Multiple Strings

### Multiple Strings

```text
user:123:name
user:123:email
user:123:status
user:123:plan
```

Advantages:

- independent Redis keys
- direct key-based access
- simple individual values

Potential disadvantages:

- more Redis keys
- repeated key/object overhead
- related fields are not grouped as one logical object
- retrieving multiple fields can require multiple lookups/requests unless batched

### Hash

```text
user:123
    ├── name
    ├── email
    ├── status
    └── plan
```

Advantages:

- related fields grouped under one key
- field-level access
- field-level updates
- atomic field counters
- potentially lower memory overhead for suitable workloads

Potential disadvantages:

- one large Hash can become problematic
- `HGETALL` can become expensive
- not suitable for every access pattern
- does not provide relational database semantics

---

# 19. Hash vs String/JSON — Decision Framework

Ask:

### 1. Do I usually read the whole object?

```text
Yes → String/JSON becomes attractive
No  → Hash becomes attractive
```

### 2. Do I frequently update individual fields?

```text
Yes → Hash becomes attractive
No  → String remains reasonable
```

### 3. Do individual fields need atomic operations?

For example:

```bash
HINCRBY issue:123 commentCount 1
```

If yes, Hash can be useful.

### 4. Do I need application-level serialization?

String/JSON:

```text
Java object
 ↓
JSON
 ↓
Redis String
```

Hash:

```text
field → value
```

### 5. What are the memory requirements?

Consider:

```text
number of objects
number of fields
field sizes
key sizes
encoding
access patterns
```

### 6. Is Redis even the right system?

If the data is authoritative relational business data, PostgreSQL may remain the source of truth.

---

# 20. Atomicity vs Consistency

This is an important production distinction.

Suppose:

```text
PostgreSQL
    ↓
authoritative membership data

Redis
    ↓
memberCount cache
```

If PostgreSQL contains:

```text
43 members
```

but Redis contains:

```text
memberCount = 42
```

then Redis is stale.

`HINCRBY` can guarantee that Redis increments are atomic:

```text
HINCRBY workspace:123 memberCount 1
```

But it does **not** guarantee that Redis remains consistent with PostgreSQL.

Therefore:

```text
Atomicity
    ≠
Consistency with another database
```

This distinction is critical in production Redis design.

---

# 21. Common Mistakes

## Mistake 1 — "Hash is always better than JSON"

Wrong.

Choose based on access patterns.

---

## Mistake 2 — "Hash is just a Java Map"

Wrong.

A Redis Hash is remote, shared server-side state.

---

## Mistake 3 — "O(1) means always fast"

Wrong.

Actual latency also depends on:

- value size
- network
- encoding
- CPU
- memory behavior
- client-side processing

---

## Mistake 4 — Blindly using HGETALL

A huge Hash can produce a huge response.

Prefer targeted field reads when possible.

---

## Mistake 5 — Using HGET → Java → HSET for counters

This introduces a read-modify-write race.

Use:

```bash
HINCRBY
```

when the requirement is an atomic numeric field update.

---

## Mistake 6 — Assuming Redis is the source of truth

Redis may be:

- cache
- derived state
- temporary state
- coordination state

The architecture must define which system is authoritative.

---

# 22. Production Use Cases

Redis Hashes are useful for:

### Cached entities

```text
user:123
    ├── name
    ├── email
    ├── status
    └── plan
```

### Workspace metadata

```text
workspace:123
    ├── name
    ├── owner
    ├── status
    └── memberCount
```

### Session-like state

```text
session:abc123
    ├── userId
    ├── role
    └── lastActivity
```

### Frequently updated counters

```text
issue:123
    ├── commentCount
    └── reactionCount
```

with:

```bash
HINCRBY issue:123 commentCount 1
```

---

# HSCAN
`HSCAN` is used to **incrementally iterate through the fields of a Hash**.
```bash

HSCAN key cursor [MATCH pattern] [COUNT count]
```

Example:

```
HSCAN workspace:123 0
```

Redis returns:
```
cursor
field
value
field
value
...
```

The returned cursor is then used for the next iteration:
```
HSCAN workspace:123 <returned-cursor>
```
Continue until Redis returns cursor:
```
0
```

## Why HSCAN exists

Avoid retrieving a potentially huge Hash all at once with:
```
HGETALL workspace:123
```

Instead
```
HSCAN
  ↓
small batch
  ↓
HSCAN
  ↓
small batch
  ↓
...
  ↓
cursor = 0
```

This is useful when a Hash contains many fields and you want to iterate through it incrementally.

## Important
`COUNT` is a **hint**, not an exact number of elements returned.
For example:

```
HSCAN workspace:123 0 COUNT 100
```

does not guarantee exactly 100 fields.

Also, `HSCAN` does **not** provide a consistent snapshot of the Hash while it is being modified.

## HGETALL vs HSCAN

```
HGETALL
    ↓
retrieve entire Hash
    ↓
simple
    ↓
potentially huge response
```

```
HSCAN
    ↓
incremental iteration
    ↓
smaller batches
    ↓
better for large Hashes
```
### Mental model

> **Use `HGETALL` when the Hash is reasonably small and you genuinely need everything. Use `HSCAN` when you need to iterate through a potentially large Hash without requesting the entire dataset in one operation.**

### Complexity

`HSCAN` is designed for incremental iteration rather than retrieving everything in one operation. The overall work to scan the entire Hash is proportional to the amount of hash-table state traversed; individual calls process only part of the iteration.

Do not treat:

HSCAN

as a guaranteed "O(1) query for N fields."

It is an **iteration mechanism** designed to avoid one giant retrieval.



# 23. When NOT to Use a Hash

Avoid using a Hash simply because the data "looks like an object."

Consider another structure when:

- you need ordered elements → List
- you need uniqueness/membership → Set
- you need ranking/order by score → Sorted Set
- you need a single opaque serialized object → String may be simpler
- you need relational querying/constraints → PostgreSQL may be appropriate
- you need durable messaging semantics → consider Redis Streams or a messaging system
- the Hash would become an enormous unrelated collection

The correct question is always:

```text
Requirement
    ↓
Access pattern
    ↓
Data structure
```

not:

```text
Object
    ↓
Hash
```

---

# 24. Complexity Summary

| Operation | Typical Complexity | Purpose |
|---|---:|---|
| `HSET` | O(1) average per field | Create/update field |
| `HGET` | O(1) average | Read one field |
| `HMGET` | O(N) | Read N fields |
| `HGETALL` | O(N) | Read all N fields |
| `HDEL` | O(1) average per field | Delete field |
| `HEXISTS` | O(1) average | Check field existence |
| `HINCRBY` | O(1) average | Atomic numeric field update |

Remember:

> Complexity describes Redis-side algorithmic work. It does not include the complete application request latency.

For multi-field operations, also consider:

```text
Redis work
+
network round trips
+
response size
+
application processing
```

---

# 25. Interview Mental Models

### "Why use a Hash?"

> When related fields need independent field-level access or updates under one logical Redis key.

### "Why not JSON String?"

> If the application frequently accesses or updates individual fields, a Hash avoids repeatedly transferring and serializing/deserializing the entire object.

### "Why is HGET O(1)?"

> Because Hash field lookup uses a hash-based lookup mechanism in the general representation, giving average constant-time field lookup.

### "Why can HGETALL be expensive?"

> Because the operation must retrieve all fields and values, so work and response size grow with the number of fields returned.

### "Why use HINCRBY?"

> It performs an atomic numeric update on a Hash field, avoiding the lost-update race of HGET → application modification → HSET.

### "Can Redis Hash replace PostgreSQL?"

> Not generally. A Hash is a key-based data structure, while PostgreSQL provides relational querying, constraints, durable source-of-truth semantics, and transactional capabilities.

---

# 26. Core Mental Model

If you remember only one diagram:

```text
                    Redis Hash
                        │
                        ▼
                 Redis Key
                 workspace:123
                        │
                        ▼
              ┌─────────────────┐
              │      Hash       │
              ├─────────────────┤
              │ name → DevSync  │
              │ owner → alice   │
              │ status → ACTIVE │
              │ count → 42      │
              └─────────────────┘
                   │   │   │
                   ▼   ▼   ▼
                 HGET HSET HINCRBY
```

And the data-modeling rule:

```text
Whole object as one value
        ↓
String / JSON may be appropriate

Fields independently accessed/updated
        ↓
Hash may be appropriate
```

The final decision should always consider:

```text
Access pattern
+
Memory
+
Network
+
Atomicity
+
Consistency
+
Failure behavior
+
Whether Redis is needed at all
```

---
