
> Revision notes from today's Phase 3 learning.
>
> Focus: understanding how Redis handles concurrent changes to shared state, and choosing the simplest correct mechanism.

---

## 1. Core Mental Model

Redis executes individual commands atomically relative to other commands.

```text
Client A ──┐
Client B ──┼──> Redis command execution
Client C ──┘
```

A single command such as `INCR` is not interrupted halfway by another Redis command.

**Important:**

```text
Redis is single-threaded
        ≠
My application is automatically race-condition-free
```

Race conditions can happen when one logical operation is split across multiple Redis commands.

---

## 2. Read → Modify → Write Race Condition

Naive counter update:

```text
GET counter
↓
modify in application
↓
SET counter
```

Example:

```text
Initial counter = 10

Client A → GET 10
Client B → GET 10

A → 10 + 1 → SET 11
B → 10 + 1 → SET 11

Final = 11
Expected = 12
```

This is a **lost update**.

The problem is not that Redis executed a command incorrectly. The problem is that the application's multi-command workflow allowed another client to operate on stale data.

---

## 3. Atomic Commands

When Redis already provides the required state transition, prefer the built-in atomic command.

Common examples:

```redis
INCR counter
INCRBY counter 10
DECR counter
DECRBY counter 5
```

`INCR` combines the required numeric update into one Redis command.

### Why `INCR` is safer

```text
GET + local increment + SET
        ↓
multiple commands
        ↓
possible interleaving

INCR
        ↓
one Redis command
        ↓
atomic state transition
```

Typical uses:

- API request counters
- page views / likes
- login attempts
- usage counters
- inventory quantities
- rate-limiter counters

### Rule

> If Redis already has one atomic command that directly expresses the operation, prefer it over a more complicated transaction/concurrency mechanism.

---

## 4. `SET NX` — Atomic Conditional Creation

```redis
SET payment:abc123 processed NX EX 300
```

Meaning:

- `NX` → set only if the key does not already exist
- `EX 300` → expire after 300 seconds

This is useful for:

- idempotency keys
- duplicate-event protection
- deduplication
- initialize-once state
- basic temporary lock patterns

The important property is that the **check + create** is performed atomically by Redis.

Example:

```text
Client A ── SET key NX ──> success
Client B ── SET key NX ──> failure
```

Only one client gets the successful creation.

---

# 5. Redis Transactions — `MULTI / EXEC / DISCARD`

Redis transactions group multiple commands for execution as one uninterrupted execution block.

### Basic flow

```redis
MULTI
SET a 10
INCR a
GET a
EXEC
```

After `MULTI`, commands are queued:

```text
MULTI
   ↓
command
   ↓
QUEUED
   ↓
command
   ↓
QUEUED
   ↓
EXEC
   ↓
execute queued commands
```

### Commands

| Command | Purpose |
|---|---|
| `MULTI` | Start transaction / begin queueing |
| `EXEC` | Execute queued commands |
| `DISCARD` | Abandon queued commands |

### `DISCARD`

```redis
MULTI
INCR counter
SET status active
DISCARD
```

The queued operations are not executed.

---

## 6. What Redis Transactions Guarantee

During execution of the queued transaction commands, another client's command is not interleaved between them.

Conceptually:

```text
Client A:
A → B → C

Transaction execution:
A → B → C
```

rather than:

```text
A → X → B → Y → C
```

where `X` and `Y` belong to another client.

This makes `MULTI/EXEC` useful for coordinated multi-command updates.

---

## 7. Redis Transactions Are NOT PostgreSQL Transactions

Do not think:

```text
Redis MULTI/EXEC = PostgreSQL BEGIN/COMMIT/ROLLBACK
```

They are different models.

Redis transactions do **not** provide PostgreSQL-style general rollback semantics.

For example:

```redis
MULTI
SET user:name Alice
SET user:age 25
INCR user:name
SET user:status ACTIVE
EXEC
```

If `INCR user:name` produces a runtime type error, Redis does not automatically roll back the earlier successful commands.

The commands are executed in order.

### Key interview statement

> Redis `MULTI/EXEC` provides atomic execution of the queued command sequence, but it is not equivalent to a full ACID database transaction with general rollback semantics.

---

# 8. `WATCH` — Optimistic Concurrency Control

`WATCH` is used when the application needs to:

```text
READ
↓
CHECK / CALCULATE
↓
CONDITIONALLY UPDATE
```

Mental model:

```text
WATCH key
   ↓
GET key
   ↓
application checks current value
   ↓
MULTI
   ↓
queue update
   ↓
EXEC
```

If another client modifies a watched key before `EXEC`, Redis aborts the transaction.

```text
Client A:
WATCH balance
GET balance → 100

Client B:
SET balance 200

Client A:
MULTI
DECRBY balance 30
EXEC

→ (nil)
```

The queued update is not applied.

---

## 9. `WATCH` Is NOT a Lock

This is critical.

```text
WATCH ≠ blocking lock
```

A watched key can still be modified by other clients.

`WATCH` instead says:

> "If this key changes before I commit, my transaction is no longer valid."

So the application can detect the conflict and retry.

---

## 10. Optimistic Concurrency Pattern

Typical application logic:

```text
WATCH
  ↓
READ
  ↓
CHECK / CALCULATE
  ↓
MULTI
  ↓
WRITE
  ↓
EXEC
  ↓
success?
 ├── YES → done
 └── NO  → retry
```

This works well when conflicts are relatively uncommon.

### Contention problem

If a key is extremely hot:

```text
many clients
    ↓
same watched key
    ↓
many conflicts
    ↓
many retries
```

Retries can become expensive and can create retry storms.

In high-contention situations, consider whether a simpler atomic command, Lua script, different data model, or another coordination approach is more appropriate.

---

# 11. `MULTI` vs `WATCH`

These solve different problems.

### `MULTI/EXEC`

> "I already know the commands I want to execute. Execute them together without interleaving."

### `WATCH`

> "The update depends on state I read earlier. Abort if that state changes before I commit."

Together:

```text
WATCH
↓
READ
↓
DECIDE
↓
MULTI
↓
WRITE
↓
EXEC
```

---

# 12. Pipelining

Pipelining is primarily a **performance optimization**, not a concurrency mechanism.

It reduces application ↔ Redis network round trips.

Without pipelining:

```text
SET → response
SET → response
SET → response
```

With pipelining:

```text
SET
SET
SET
  ↓
send as a batch
  ↓
responses
```

### Pipeline ≠ Transaction

| Mechanism | Main purpose |
|---|---|
| Atomic command | Correct single state transition |
| `MULTI/EXEC` | Grouped execution |
| `WATCH` | Detect concurrent modification |
| Pipeline | Reduce network round trips |

A pipeline does **not** provide rollback or transaction semantics.

---

# 13. Lua / Server-Side Atomic Operations

Lua is useful when Redis needs to perform:

```text
READ
↓
CHECK
↓
UPDATE
↓
RETURN RESULT
```

as one atomic server-side operation.

Example inventory logic:

```text
if stock >= requested
    decrease stock
    return success
else
    return failure
```

Conceptually:

```text
Application
    ↓
EVAL script
    ↓
Redis
    ├── read
    ├── check
    ├── update
    └── return
    ↓
Application
```

Basic Redis Lua concepts:

```redis
EVAL "return 42" 0
```

- `KEYS[]` → Redis keys used by the script
- `ARGV[]` → other input values

Example:

```lua
local stock = tonumber(redis.call('GET', KEYS[1]))
local requested = tonumber(ARGV[1])

if stock >= requested then
    redis.call('DECRBY', KEYS[1], requested)
    return 1
else
    return 0
end
```

### Lua vs WATCH

Use `WATCH` when:

- application-side logic is more natural
- conflicts are relatively uncommon
- you want optimistic concurrency

Lua is attractive when:

- the logic is Redis-centric
- several Redis operations must be atomic
- the logic can remain small and efficient

Do not use Lua when a built-in command such as `INCR` already solves the problem.

### Lua caution

Redis executes a Lua script as a server-side operation. A long or expensive script can delay other Redis commands.

Keep Redis scripts:

- small
- deterministic
- efficient
- focused on Redis-local state transitions

---

# 14. Choosing the Right Mechanism

Use this decision model:

```text
Need one simple state transition?
        ↓
Atomic command
```

```text
Need conditional creation?
        ↓
SET NX
```

```text
Need several known commands executed together?
        ↓
MULTI / EXEC
```

```text
Need READ → CHECK → conditional WRITE?
        ↓
WATCH + MULTI/EXEC
or Lua
```

```text
Need many independent commands and network latency is the bottleneck?
        ↓
Pipeline
```

### Example

**Increment counter**

```redis
INCR counter
```

**Idempotency**

```redis
SET payment:abc123 processed NX EX 300
```

**Conditional inventory reservation**

```text
WATCH + MULTI/EXEC
```

or a small atomic Lua script.

**Thousands of independent commands**

```text
Pipeline
```

---

# 15. Inventory Reservation — Complete Mental Model

Requirement:

> Reserve 1 seat only if at least 1 seat remains.

Naive:

```text
GET seats
↓
check seats > 0
↓
DECR seats
```

Race:

```text
A → GET 1
B → GET 1

A → DECR → 0
B → DECR → -1
```

With `WATCH`:

```text
WATCH seats
GET seats
↓
check seats > 0
↓
MULTI
DECR seats
EXEC
```

If another client changes `seats` before `EXEC`:

```text
EXEC → nil
```

The application can retry.

With Lua:

```text
script:
    read seats
    if seats >= 1:
        decrement
        return success
    else:
        return failure
```

Each script execution is atomic, so one request can consume the final seat and another will see the updated state when its script executes.

---

# 16. Key Misconceptions

### ❌ "Redis is single-threaded, so race conditions don't exist."

Correct:

> Individual Redis commands execute atomically, but multi-command application workflows can still interleave.

### ❌ "`WATCH` locks the key."

Correct:

> `WATCH` provides optimistic concurrency control. Other clients can still modify the key.

### ❌ "`MULTI/EXEC` is an ACID transaction like PostgreSQL."

Correct:

> Redis provides atomic execution of the queued commands, but not PostgreSQL-style general rollback semantics.

### ❌ "Always use WATCH for concurrent updates."

Correct:

> Prefer a built-in atomic command when it directly expresses the required operation.

### ❌ "Pipeline makes commands atomic."

Correct:

> Pipelining reduces network round trips; it does not provide transaction semantics.

---

# 17. Phase 3 Mental Model

```text
Single Redis command
        ↓
Atomic execution

Multiple commands
        ↓
Potential interleaving
        ↓
Choose appropriate mechanism
        │
        ├── Atomic command
        │
        ├── SET NX
        │
        ├── MULTI / EXEC
        │
        ├── WATCH + MULTI / EXEC
        │
        ├── Lua
        │
        └── Pipeline
```

The central principle:

> **Use the simplest mechanism that correctly solves the concurrency or performance problem.**

---

# Interview Questions

## 1. Why can a race condition happen even though Redis is single-threaded?

### Answer

Redis does not interleave the execution of an individual command, but a logical application operation may consist of multiple commands.

```text
GET
↓
application modifies value
↓
SET
```

Another client can execute commands between the `GET` and `SET`.

Therefore, Redis's single-threaded command execution does not make multi-command workflows automatically atomic.

### Reasoning

The unit Redis guarantees is the **command**, not arbitrary application workflows.

---

## 2. Why is `INCR` safer than `GET → increment → SET`?

### Answer

`INCR` performs the increment as one Redis command.

```text
INCR counter
```

is executed atomically, so concurrent increments are applied sequentially rather than both operating on the same stale value.

### Reasoning

With:

```text
GET → modify → SET
```

two clients can read the same old value and overwrite each other's updates.

With:

```text
INCR
```

the read-modify-write transition happens inside one atomic Redis command.

---

## 3. Is `WATCH` a lock?

### Answer

No.

`WATCH` is an **optimistic concurrency control mechanism**.

Other clients can still modify the watched key.

If the watched key changes before `EXEC`, Redis aborts the transaction.

### Reasoning

A real blocking lock tries to prevent concurrent access:

```text
lock → work → unlock
```

`WATCH` instead does:

```text
observe → work → detect conflict → retry
```

---

## 4. Are Redis `MULTI/EXEC` transactions ACID like PostgreSQL transactions?

### Answer

No.

Redis `MULTI/EXEC` provides atomic execution of the queued command sequence, but it does not provide PostgreSQL-style general rollback semantics.

For example, if an individual command encounters a runtime error, earlier successful commands are not automatically rolled back.

### Reasoning

Redis transactions are designed around atomic grouped command execution, not the full transactional database model of:

```text
BEGIN
↓
statements
↓
COMMIT / ROLLBACK
```

---

## 5. When would you choose Lua instead of `WATCH`?

### Answer

Use Lua when a small piece of Redis-local logic needs multiple commands and must execute atomically.

Example:

```text
read inventory
↓
check stock
↓
decrement
↓
return success/failure
```

Lua keeps the whole operation inside Redis.

`WATCH` is attractive when application-side logic naturally performs the decision and conflicts are relatively uncommon.

### Reasoning

The choice is not "Lua is better."

Instead:

```text
Simple operation
→ atomic command

Application-side optimistic update
→ WATCH

Redis-local multi-step atomic logic
→ Lua
```

---

# Quick Revision Cheatsheet

```text
INCR
→ atomic single-command state transition

SET ... NX
→ atomic conditional creation

MULTI
→ queue commands

EXEC
→ execute queued commands

DISCARD
→ discard queued commands

WATCH
→ detect changes to watched keys before EXEC

WATCH conflict
→ EXEC aborts / returns null result

Pipeline
→ reduce network round trips

Lua
→ custom Redis-side atomic logic
```

### One-line rules

```text
Need simple atomic update?
→ Atomic command

Need "create only if absent"?
→ SET NX

Need grouped known commands?
→ MULTI/EXEC

Need read → decide → write?
→ WATCH or Lua

Need many independent commands efficiently?
→ Pipeline
```

