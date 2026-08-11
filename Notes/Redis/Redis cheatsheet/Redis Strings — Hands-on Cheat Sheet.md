Main notes -- [[Redis]]
## 1. Basic String Operations

### SET

```redis
SET key value
```

Creates or replaces a String value.

```redis
SET user:1 "Alice"
```

Returns:

```text
OK
```

**Important:** Normal `SET` removes an existing TTL.

---

### GET

```redis
GET key
```

Returns the String value.

```redis
GET user:1
```

If key doesn't exist:

```text
(nil)
```

---

### DEL

```redis
DEL key
```

Deletes the key.

Returns:

```text
(integer) 1   # key existed and was deleted
(integer) 0   # key didn't exist
```

---

### TYPE

```redis
TYPE key
```

Returns the Redis data type.

```redis
TYPE user:1
```

→

```text
string
```

---

## 2. Multiple-Key Operations

### MGET

```redis
MGET key1 key2 key3
```

Gets multiple String values.

```redis
MGET user:1 user:2 user:3
```

Missing keys return `nil` for that position.

---

### MSET

```redis
MSET key1 value1 key2 value2
```

Sets multiple String keys.

```redis
MSET user:1 "Alice" user:2 "Bob"
```

Returns:

```text
OK
```

**Important:**

- Overwrites existing values.
    
- Removes existing TTLs.
    
- Has no `EX`/`PX` expiration option.
    

---

### MSETNX

```redis
MSETNX key1 value1 key2 value2
```

Sets multiple keys **only if ALL target keys don't already exist**.

Returns:

```text
(integer) 1   # success
(integer) 0   # at least one key already exists
```

**Atomic + all-or-nothing.**

If one key exists, none are modified.

---

## 3. Numeric Strings / Counters

Redis Strings can contain numeric values.

```redis
SET counter 10
```

### INCR

```redis
INCR counter
```

Atomically increments by `1`.

```text
10 → 11
```

If key doesn't exist:

```text
INCR counter
```

creates it with:

```text
1
```

---

### INCRBY

```redis
INCRBY counter 10
```

Atomically increments by the specified amount.

```text
10 → 20
```

---

### DECR

```redis
DECR counter
```

Atomically decrements by `1`.

---

### Important Numeric Rule

The String value must be a valid Redis integer representation.

```redis
SET counter 10
INCR counter
```

→ works.

But:

```redis
SET counter "hello"
INCR counter
```

→

```text
ERR value is not an integer or out of range
```

Also, values such as:

```text
"00010"
```

are not accepted by `INCR`.

---

## 4. Expiration / TTL

### EXPIRE

```redis
EXPIRE key seconds
```

Sets expiration in seconds.

```redis
EXPIRE session:1 60
```

---

### SET with EX

```redis
SET key value EX seconds
```

Creates/updates a key with expiration.

```redis
SET session:1 "Alice" EX 60
```

---

### SET with PX

```redis
SET key value PX milliseconds
```

Expiration in milliseconds.

```redis
SET session:1 "Alice" PX 5000
```

---

### TTL

```redis
TTL key
```

Returns remaining TTL in seconds.

Possible results:

```text
> 0   → expires in N seconds
-1    → key exists but has no expiration
-2    → key doesn't exist
```

Example:

```text
(integer) 57
```

---

### Expired Key

Once a key expires:

```text
expired key ≈ nonexistent key
```

Therefore:

```redis
GET key
```

returns `nil`, and:

```redis
TTL key
```

returns:

```text
-2
```

---

### Important SET + TTL Rule

Normal:

```redis
SET key value
```

**removes the existing TTL.**

Example:

```redis
SET session:1 "Alice" EX 60
SET session:1 "Bob"
TTL session:1
```

→

```text
-1
```

The key still exists, but no longer expires.

---

### KEEPTTL

```redis
SET key value KEEPTTL
```

Replaces the value while preserving the existing TTL.

Example:

```redis
SET session:1 "Alice" EX 120
SET session:1 "Bob" KEEPTTL
```

The TTL continues counting down from whatever remains.

It does **not** reset to 120.

---

## 5. Conditional SET

### NX — Create Only If Absent

```redis
SET key value NX
```

Means:

> Set only if the key does NOT exist.

Example:

```redis
SET lock:1 "server-A" NX
```

If key doesn't exist:

```text
OK
```

If key already exists:

```text
(nil)
```

Existing value is untouched.

---

### NX + Expiration

Common lock/token pattern:

```redis
SET lock:1 "server-A" NX EX 30
```

Atomically:

```text
IF key doesn't exist
    create value
    set TTL
ELSE
    do nothing
```

---

### XX — Update Only If Present

```redis
SET key value XX
```

Means:

> Set only if the key already exists.

Example:

```redis
SET profile:1 "Alice" XX
```

If key doesn't exist:

```text
(nil)
```

If key exists:

```text
OK
```

---

### NX vs XX

```text
NX → create-if-absent
XX → update-if-present
```

---

## 6. APPEND

```redis
APPEND key value
```

Appends to the existing String.

```redis
SET msg "Hello"
APPEND msg " World"
```

Result:

```text
"Hello World"
```

Returns the **new String length**.

```text
(integer) 11
```

If the key doesn't exist:

```redis
APPEND msg "Hello"
```

creates:

```text
msg → "Hello"
```

Returns:

```text
5
```

**TTL:** APPEND preserves an existing TTL.

---

## 7. STRLEN

```redis
STRLEN key
```

Returns String length in bytes.

```redis
SET msg "Hello"
STRLEN msg
```

→

```text
5
```

Remember:

> Redis Strings are fundamentally byte sequences.

---

## 8. GETRANGE

```redis
GETRANGE key start end
```

Returns a substring using **zero-based byte indexes**.

```redis
SET text "Hello Redis"
```

```redis
GETRANGE text 0 4
```

→

```text
"Hello"
```

```redis
GETRANGE text 6 10
```

→

```text
"Redis"
```

### Negative indexes

```text
-1 → last byte
-2 → second-last byte
```

Therefore:

```redis
GETRANGE text 0 -1
```

returns the entire String.

---

## 9. SETRANGE

```redis
SETRANGE key offset value
```

Overwrites bytes starting at the specified offset.

```redis
SET text "Hello World"
SETRANGE text 6 "Redis"
```

Result:

```text
"Hello Redis"
```

Returns the new String length.

### Beyond Current Length

If the offset is beyond the current String length:

```redis
SET x "abc"
SETRANGE x 5 "Z"
```

Redis expands the String:

```text
abc\0\0Z
```

Missing positions are filled with zero bytes.

---

## 10. GETSET

```redis
GETSET key new-value
```

Atomically:

1. Returns the old value.
    
2. Replaces it with the new value.
    

Example:

```redis
SET counter 10
GETSET counter 20
```

Returns:

```text
"10"
```

Afterward:

```text
counter → "20"
```

### Important

`GETSET` removes the existing TTL, like normal `SET`.

For a TTL-preserving replacement, use:

```redis
SET key value KEEPTTL
```

---

## 11. GETDEL

```redis
GETDEL key
```

Atomically:

1. Gets the value.
    
2. Deletes the key.
    

Example:

```redis
SET otp:user:1 "482913"
GETDEL otp:user:1
```

→

```text
"482913"
```

Then:

```redis
GET otp:user:1
```

→

```text
(nil)
```

### Why GETDEL matters

Avoid:

```redis
GET key
DEL key
```

because another client can execute operations between those commands.

`GETDEL` performs both atomically.

---

# 12. Important TTL Behavior Summary

|Command|Modifies value?|Existing TTL|
|---|--:|---|
|`SET key value`|Yes|❌ Removed|
|`SET ... EX 60`|Yes|🔄 Set to 60s|
|`SET ... KEEPTTL`|Yes|✅ Preserved|
|`MSET`|Yes|❌ Removed|
|`GETSET`|Yes|❌ Removed|
|`INCR`|Yes|✅ Preserved|
|`INCRBY`|Yes|✅ Preserved|
|`DECR`|Yes|✅ Preserved|
|`APPEND`|Yes|✅ Preserved|
|`SETRANGE`|Yes|✅ Preserved|
|`GET`|No|✅ Preserved|
|`GETDEL`|Deletes|Key removed|

---

# 13. Atomicity Cheat Sheet

### Atomic single-command operations

These are atomic at the Redis command level:

```text
SET
GET
DEL
INCR
INCRBY
DECR
APPEND
SETRANGE
GETSET
GETDEL
MSET
MSETNX
```

### Important distinction

Atomic does **not** mean:

> Multiple commands automatically form one atomic operation.

For example:

```redis
GET counter
INCR counter
```

is **two commands**, not one atomic application-level operation.

Another client can execute commands between them.

---

# 14. Common Backend Patterns

## OTP Creation

```redis
SET user:123:otp 739214 NX EX 300
```

```text
OK   → OTP created
nil  → valid OTP already exists
```

---

## OTP Consumption

```redis
GETDEL user:123:otp
```

```text
"739214" → OTP existed and was consumed
nil      → missing / expired / already consumed
```

---

## Simple Lock

```redis
SET lock:123 server-A NX EX 30
```

```text
OK   → lock acquired
nil  → lock already exists
```

---

## Fixed-Window Counter

Initial creation:

```redis
SET rate:user:42 1 NX EX 60
```

Subsequent requests:

```redis
INCR rate:user:42
```

Then inspect:

```redis
GET rate:user:42
TTL rate:user:42
```

For a production-grade conditional rate limiter, the decision logic may need to be made atomically using Redis scripting or another atomic mechanism.

---

# 15. Mental Models to Remember

### Redis String

```text
key
 ↓
String value
```

The Redis data type is **String**, regardless of whether you use the value as:

```text
text
number
counter
JSON
token
OTP
lock value
binary data
```

---

### Value vs TTL

Think of them as separate state:

```text
key
├── value
└── expiration metadata
```

A command can modify one without necessarily modifying the other.

---

### NX

```text
Does key exist?
    ↓
   NO  → SET
   YES → don't SET
```

### XX

```text
Does key exist?
    ↓
   YES → SET
   NO  → don't SET
```

### TTL

```text
positive → expires
-1       → exists, no expiration
-2       → doesn't exist
```

### Atomicity

A single Redis command executes atomically, but:

```text
COMMAND A
COMMAND B
COMMAND C
```

is **not automatically one atomic unit**.

That distinction is critical for backend development and interviews.