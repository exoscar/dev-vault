 # Redis Phase 2 — Expiration & Eviction

> Goal: understand how Redis manages **key lifetime** and **memory pressure**, and how to design TTL/eviction behavior in real backend systems.

---

# 1. EXPIRE / PEXPIRE

Redis can associate an expiration time with a key.

```redis
SET session:123 abc
EXPIRE session:123 60
```

The key now has:

```text
value → "abc"
expiration → ~60 seconds from now
```

### Commands

```redis
EXPIRE key seconds
PEXPIRE key milliseconds
```

Example:

```redis
SET otp:user123 849201
EXPIRE otp:user123 300
```

The OTP is intended to be valid for 5 minutes.

### Important

Expiration is **key-level metadata**. It is not a separate Redis data type.

```text
String + TTL
Hash + TTL
List + TTL
Set + TTL
Sorted Set + TTL
Stream + TTL
```

All can have key-level expiration.

Also:

> Expiration time being reached does not guarantee that the key is physically deleted at that exact instant.

---

# 2. TTL / PTTL

Use these to inspect remaining lifetime.

```redis
TTL key
PTTL key
```

- `TTL` → remaining lifetime in seconds
- `PTTL` → remaining lifetime in milliseconds

Example:

```redis
SET session:123 abc EX 60
TTL session:123
```

Possible result:

```text
(integer) 57
```

because a few seconds may already have passed.

### Special return values

```text
TTL > 0  → key exists and has expiration
TTL = -1 → key exists but has no expiration
TTL = -2 → key does not exist
```

The same meanings apply to `PTTL`.

### Mental model

```text
expiration timestamp
        -
current time
        ↓
remaining lifetime
        ↓
TTL / PTTL
```

TTL is **not** a precise physical-deletion countdown.

---

# 3. PERSIST

`PERSIST` removes a key's expiration without deleting the key.

```redis
SET session:123 abc EX 300
PERSIST session:123
```

Before:

```text
session:123
 ├── value → abc
 └── expiration → timestamp
```

After:

```text
session:123
 └── value → abc
```

Check:

```redis
TTL session:123
```

Result:

```text
(integer) -1
```

The value still exists:

```redis
GET session:123
```

```text
"abc"
```

### Key distinction

```text
PERSIST → remove expiration
DEL     → remove key/value
```

---

# 4. Expiration Metadata

Conceptually, think of Redis as tracking:

```text
Main key/value data
session:123 → "abc"

Expiration information
session:123 → expiration timestamp
```

This explains several commands:

```text
EXPIRE
→ creates/changes expiration metadata

TTL
→ reads remaining lifetime

PERSIST
→ removes expiration metadata
```

The value itself does not need to contain the expiration information.

---

# 5. Lazy Expiration

**Lazy expiration** means Redis checks whether a key has expired when it encounters/accesses that key.

Example:

```redis
SET session:123 abc EX 5
```

After the deadline:

```redis
GET session:123
```

Conceptually:

```text
GET key
   ↓
Check expiration
   ↓
Expired?
 ┌─┴──┐
No   Yes
│     │
Use   Remove/treat as nonexistent
key
```

So an expired key is not available as valid data.

### Important distinction

```text
Expiration deadline reached
        ≠
Immediate physical deletion
```

Lazy expiration is one way Redis discovers expired keys.

---

# 6. Active Expiration

Lazy expiration alone has a problem.

Imagine:

```text
1,000,000 keys expire
        ↓
nobody accesses them
```

If Redis only checked expiration during access, those keys could remain in memory temporarily.

Redis therefore also performs **active expiration processing**.

Conceptually:

```text
Expiration processing
       ↓
Sample expiring keys
       ↓
Check expiration
       ↓
Expired?
       ↓
Remove
```

Redis uses sampling rather than continuously scanning every expiring key.

### Why?

Redis must balance:

```text
Cleanup work
     ↕
CPU usage
     ↕
Expired memory
```

So:

```text
Lazy expiration
→ discovered during access

Active expiration
→ discovered by Redis's expiration processing
```

---

# 7. Expiration and Memory

Expiration is not identical to memory reclamation.

```text
Expiration deadline reached
        ↓
Key becomes expired
        ↓
Redis discovers it
        ↓
Key is removed
        ↓
Memory becomes available
```

Therefore, for a short period:

```text
key = expired
memory = may still be occupied
```

Large numbers of keys expiring at once can create:

- expiration-related CPU work
- temporary memory pressure
- a large amount of cleanup activity

This matters for systems with millions of short-lived keys.

---

# 8. TTL Design in Real Systems

Do not start with:

> "What TTL should Redis use?"

Start with:

> **"How long should this data remain valid or useful?"**

Common examples:

| Data | Why TTL exists |
|---|---|
| Session | Authentication lifetime / inactivity |
| OTP | Security validity window |
| Password reset token | Security validity window |
| Rate-limit counter | Counting window |
| Cache | Maximum acceptable staleness |
| Temporary state | Process/business lifetime |

### Design questions

1. How long should the data remain valid?
2. What happens when it expires?
3. Can it be reconstructed?
4. How stale can it safely be?
5. Is expiration a business rule or just a memory-management mechanism?

Not every Redis key needs a TTL.

---

# 9. TTL + Cache Design

A common **cache-aside** pattern:

```text
Request
   ↓
Redis GET
   │
   ├── HIT → return cached value
   │
   └── MISS
          ↓
      Database
          ↓
      Redis SET + TTL
          ↓
        return
```

Example:

```redis
SET issue:123 "<data>" EX 300
```

The TTL limits how long the cached representation can remain.

### But TTL is NOT cache invalidation

Suppose:

```text
Database changes at 10:01
Cache TTL expires at 10:10
```

The cache can contain stale data between those times.

Therefore:

```text
TTL
→ maximum intended cache lifetime

Invalidation
→ remove/update cache when source data changes
```

A stronger pattern is:

```text
Database update
      ↓
Invalidate Redis key
      +
TTL as safety fallback
```

Example:

```redis
DEL issue:123
```

If invalidation fails, TTL eventually limits how long stale data survives.

---

# 10. maxmemory

Redis is memory-oriented, so memory must be managed deliberately.

Inspect:

```redis
CONFIG GET maxmemory
CONFIG GET maxmemory-policy
```

Conceptually:

```text
Redis memory usage
        ↓
approaches maxmemory
        ↓
memory pressure
        ↓
maxmemory-policy determines behavior
```

`maxmemory` is a configured Redis memory-management boundary. It is not simply the amount of physical RAM installed in the machine.

---

# 11. maxmemory-policy

This determines what Redis does when memory pressure requires a decision.

Main policies:

```text
noeviction

allkeys-lru
volatile-lru

allkeys-lfu
volatile-lfu

allkeys-random
volatile-random

volatile-ttl
```

A useful way to decode them:

```text
allkeys   → who can be considered?
volatile  → who can be considered?

lru       → choose using recency
lfu       → choose using frequency
random    → choose randomly
ttl       → prefer shorter remaining TTL
```

---

# 12. noeviction

```text
maxmemory-policy noeviction
```

When Redis cannot accommodate a memory-requiring write:

```text
Memory pressure
      ↓
Don't evict existing keys
      ↓
Write can fail with OOM
```

Existing keys are not automatically sacrificed to make room.

### Important

`noeviction` does **not** disable normal expiration.

```text
Expiration → still works
Eviction   → not used to solve memory pressure
```

### When it can make sense

Useful when existing Redis data is important and automatic removal is unacceptable.

Potentially problematic for a pure cache:

```text
Cache full
   ↓
No eviction
   ↓
New cache writes fail
```

For a rebuildable cache, eviction is often more useful.

---

# 13. LRU Eviction

LRU = **Least Recently Used**

Question:

> **When was this key used most recently?**

General assumption:

```text
Recently used
→ likely useful

Not recently used
→ better eviction candidate
```

Policies:

```text
allkeys-lru
volatile-lru
```

### Redis LRU is approximate

Redis does not maintain a perfect global ordering of every key.

Conceptually:

```text
Many keys
   ↓
Sample candidates
   ↓
Compare recency
   ↓
Choose a good candidate
   ↓
Evict
```

This avoids the overhead of maintaining perfect LRU state for every key.

### Best mental model

```text
LRU = recency
```

---

# 14. LFU Eviction

LFU = **Least Frequently Used**

Question:

> **How frequently is this key accessed?**

General assumption:

```text
Frequently accessed
→ likely valuable

Rarely accessed
→ better eviction candidate
```

Policies:

```text
allkeys-lfu
volatile-lfu
```

### LRU vs LFU

```text
LRU → "When was it last used?"
LFU → "How often is it used?"
```

Example:

```text
Key A → 10,000 accesses, last used 1 hour ago
Key B → 2 accesses, last used 1 second ago
```

LRU favors recent usage.

LFU favors high frequency.

LFU can be attractive when a stable subset of keys is consistently popular.

Redis's LFU implementation is approximate and allows old popularity to decay, rather than treating lifetime access count as an exact permanent score.

---

# 15. allkeys vs volatile

This is one of the most important policy distinctions.

## `allkeys-*`

Any key can be an eviction candidate.

```text
TTL key     → candidate
No-TTL key  → candidate
```

Example:

```text
cache:1 → TTL
cache:2 → TTL
user:1  → no TTL
```

With:

```text
allkeys-lru
```

all three can be considered.

---

## `volatile-*`

Only keys that have an expiration are candidates.

```text
TTL key     → candidate
No-TTL key  → not a candidate
```

With:

```text
volatile-lru
```

only `cache:1` and `cache:2` are candidates.

### Critical implication

If you use:

```text
volatile-lru
```

but most keys have no TTL:

```text
No TTL keys
      ↓
Not eligible
      ↓
May be no candidates to evict
      ↓
Write can fail
```

So `volatile-*` requires consistent TTL usage.

---

# 16. Other Eviction Policies

## `allkeys-random`

```text
Any key
   ↓
random candidate
   ↓
evict
```

Useful when access patterns are not important and simple candidate selection is preferred.

## `volatile-random`

```text
Only TTL keys
   ↓
random candidate
   ↓
evict
```

## `volatile-ttl`

```text
Only TTL keys
      ↓
prefer shorter remaining TTL
      ↓
eviction candidate
```

Important:

```text
volatile-ttl
```

is an **eviction policy**, not the normal expiration mechanism.

---

# 17. Expiration vs Eviction ⭐

This distinction should become automatic.

## Expiration

```text
Trigger → time
Purpose → key/data lifecycle
Commands → EXPIRE / PEXPIRE
```

Meaning:

> "This data should stop being valid after its lifetime ends."

## Eviction

```text
Trigger → memory pressure
Purpose → free memory
Controlled by → maxmemory-policy
```

Meaning:

> "Redis needs memory and must decide what data can be removed."

### Three ways a key can disappear

```text
1. Expiration
2. Eviction
3. Explicit deletion (DEL)
```

---

# 18. TTL Does NOT Guarantee Residency

This is a critical rule.

Suppose:

```redis
SET cache:123 abc EX 300
```

You might have:

```text
TTL ≈ 300 seconds
```

That does **not** mean:

> "Redis guarantees this key will stay for 300 seconds."

It means:

> "The key has a 300-second expiration lifetime, assuming it is not removed earlier."

It can disappear earlier through:

```text
Expiration
Eviction
DEL
```

For example:

```text
TTL = 290 seconds
       ↓
maxmemory pressure
       ↓
allkeys-lru
       ↓
key selected
       ↓
key evicted
```

Therefore:

> **TTL is not a guaranteed minimum residency time.**

---

# 19. Can a Key Without TTL Be Evicted?

Yes, depending on the policy.

```text
No TTL
   │
   ├── allkeys-* → can be candidate
   │
   └── volatile-* → not a candidate
```

Example:

```text
user:123 → no TTL
```

With:

```text
allkeys-lru
```

it can be evicted.

With:

```text
volatile-lru
```

it is not an eligible candidate.

---

# 20. Memory Pressure Scenario

Suppose:

```text
maxmemory = 100 MB
current usage = 99 MB
incoming write = 5 MB
```

The write would require approximately:

```text
99 + 5 = 104 MB
```

which exceeds the configured limit.

### `noeviction`

```text
Don't evict
→ write can fail
```

### `allkeys-lru`

```text
All keys eligible
→ choose suitable LRU candidate
→ evict
→ make room
→ write can proceed
```

### `volatile-lru` with TTL keys

```text
Only TTL keys eligible
→ choose suitable LRU candidate
→ evict
→ make room
```

### `volatile-lru` with no TTL keys

```text
No eligible candidates
→ cannot solve pressure through eviction
→ write can fail
```

---

# 21. Quick Policy Map

| Policy | Candidates | Selection |
|---|---|---|
| `noeviction` | None | Reject memory-requiring writes |
| `allkeys-lru` | All keys | Least recently used |
| `volatile-lru` | TTL keys | Least recently used |
| `allkeys-lfu` | All keys | Least frequently used |
| `volatile-lfu` | TTL keys | Least frequently used |
| `allkeys-random` | All keys | Random |
| `volatile-random` | TTL keys | Random |
| `volatile-ttl` | TTL keys | Shorter TTL preferred |

---

# 22. Core Mental Model

```text
                       Redis
                         │
              ┌──────────┴──────────┐
              │                     │
         Key Lifetime          Memory Pressure
              │                     │
           EXPIRE               maxmemory
              │                     │
         TTL / PTTL             maxmemory-policy
              │                     │
       Key becomes expired       Eviction decision
              │                     │
       Lazy / Active         ┌───────┼────────┐
          expiration         │       │        │
              │             LRU     LFU     Random/TTL
              │
          Key removed
```

---

# Final Rules to Remember

```text
EXPIRE  → set expiration in seconds
PEXPIRE → set expiration in milliseconds

TTL     → remaining seconds
PTTL    → remaining milliseconds

TTL = -1 → exists, no expiration
TTL = -2 → key doesn't exist

PERSIST → remove expiration, keep value

Lazy expiration  → expiration discovered during access
Active expiration → Redis periodically processes expiring keys

maxmemory → configured memory-management limit
noeviction → don't evict; memory-requiring writes can fail

LRU → recency
LFU → frequency

allkeys-* → all keys can be candidates
volatile-* → only TTL keys can be candidates

Expiration → time-based
Eviction → memory-pressure-based

TTL ≠ cache invalidation
TTL ≠ guaranteed survival until expiration
```

---

# Phase 2 Complete

```text
✅ EXPIRE / PEXPIRE
✅ TTL / PTTL
✅ PERSIST
✅ Expiration metadata
✅ Lazy expiration
✅ Active expiration
✅ Expiration + memory
✅ TTL design
✅ TTL + cache design
✅ maxmemory
✅ noeviction
✅ LRU
✅ LFU
✅ allkeys vs volatile
✅ Random / volatile-ttl
✅ Expiration vs eviction
✅ Memory-pressure scenarios

➡️ Next: Hands-on Docker/Redis eviction labs
```
