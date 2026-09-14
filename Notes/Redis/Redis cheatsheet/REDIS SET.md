## Core Concept

A Redis **Set** is an unordered collection of unique string values.

```text
List → ordered + duplicates allowed
Set  → unordered + unique members
```

Redis automatically prevents duplicate members.

---

## Core Commands

### Add

```redis
SADD key member [member ...]
```

Adds one or more members.

Returns the number of members actually added.

```redis
SADD workspace:900:members alice bob charlie
```

Adding `alice` again does not create a duplicate.

---

### Read All Members

```redis
SMEMBERS key
```

Returns all members.

**Important:** Set order is not guaranteed.

---

### Check Membership

```redis
SISMEMBER key member
```

Returns:

```text
1 → member exists
0 → member does not exist
```

Useful for fast membership checks.

---

### Remove

```redis
SREM key member [member ...]
```

Removes specified members.

---

### Count

```redis
SCARD key
```

Returns the number of members in the Set.

```text
LLEN  → List length
SCARD → Set cardinality
```

---

# Set Operations

Sets become especially useful when working with relationships between collections.

Assume:

```text
workspace:900:members
→ alice, bob, charlie, david

workspace:901:members
→ bob, charlie, eve
```

## Intersection

```redis
SINTER workspace:900:members workspace:901:members
```

Returns members present in **both** Sets.

```text
bob
charlie
```

Mental model:

```text
SINTER → What do they have IN COMMON?
```

---

## Union

```redis
SUNION workspace:900:members workspace:901:members
```

Returns all unique members across the Sets.

```text
alice
bob
charlie
david
eve
```

Mental model:

```text
SUNION → What does everyone have COMBINED?
```

---

## Difference

```redis
SDIFF workspace:900:members workspace:901:members
```

Returns members that exist in the first Set but not the second.

```text
alice
david
```

Mental model:

```text
SDIFF → What's in A but NOT in B?
```

---

# Storing Set Operation Results

Instead of only returning the result, Redis can store it.

### Intersection

```redis
SINTERSTORE destination key1 key2
```

### Union

```redis
SUNIONSTORE destination key1 key2
```

### Difference

```redis
SDIFFSTORE destination key1 key2
```

Example:

```redis
SINTERSTORE common:members workspace:900:members workspace:901:members
```

This creates:

```text
common:members
```

containing the intersection.

### Caution

Stored/derived Sets can become stale when the source Sets change.

Use them deliberately rather than creating derived keys everywhere.

---

# Random Operations

## SRANDMEMBER

Returns random members **without removing them**.

```redis
SRANDMEMBER key
```

Mental model:

```text
SRANDMEMBER → peek randomly
```

---

## SPOP

Returns random members **and removes them**.

```redis
SPOP key
```

Mental model:

```text
SPOP → consume randomly
```

Useful for simple random assignment or work distribution.

---

# Iterating Large Sets

Avoid blindly using:

```redis
SMEMBERS huge:set
```

for potentially large Sets because it returns the entire collection.

Use:

```redis
SSCAN key cursor
```

Example:

```redis
SSCAN users 0
```

Continue using the returned cursor until Redis returns cursor `0`.

Mental model:

```text
SMEMBERS → give me everything now
SSCAN    → iterate incrementally
```

Related scan commands:

```text
SCAN  → Keys
HSCAN → Hash
SSCAN → Set
ZSCAN → Sorted Set
```

---

# Real-World Use Cases

Sets are useful when **uniqueness and membership** matter more than order.

Common examples:

```text
Workspace members
Online users
Permissions
Blocked users
Tags
Followers / following
Active sessions
Processed IDs
Feature membership
```

Example:

```redis
SADD workspace:900:members alice bob charlie
SISMEMBER workspace:900:members alice
```

---

# Sets for Relationships

Sets are excellent for representing relationships.

Example:

```text
user:alice:following
→ bob, charlie, david

user:bob:following
→ charlie, david, eve
```

Find people both users follow:

```redis
SINTER user:alice:following user:bob:following
```

Result:

```text
charlie
david
```

This allows Redis to perform set-based relationship queries directly.

---

# Reverse Index Pattern

Sets can also be used as reverse indexes.

Example:

```text
tag:redis:articles
→ article:101
→ article:205
→ article:301
```

Add articles:

```redis
SADD tag:redis:articles article:101 article:205 article:301
```

Then:

```redis
SMEMBERS tag:redis:articles
```

returns all articles associated with the tag.

---

# Sets vs Lists

| Requirement | Structure |
|---|---|
| Ordered collection | List |
| Duplicates allowed | List |
| Unique members | Set |
| Fast membership check | Set |
| Intersection / Union / Difference | Set |
| Queue | List |
| Stack | List |
| Recent/latest N items | List |
| Random member | Set |
| Ordered by score | Sorted Set |

Decision rule:

```text
Need uniqueness?
    ↓
Set

Need uniqueness + ordering by score?
    ↓
Sorted Set

Need ordering / queue behavior?
    ↓
List
```

---

# Important Limitations

## Sets are unordered

Do not depend on the order returned by:

```redis
SMEMBERS key
```

If insertion order matters, use a List.

If ordering by score matters, use a Sorted Set.

---

# Phase 1 Commands to Know

### Must know

```text
SADD
SREM
SMEMBERS
SISMEMBER
SCARD
SINTER
SUNION
SDIFF
```

### Should understand

```text
SINTERSTORE
SUNIONSTORE
SDIFFSTORE
SSCAN
SPOP
SRANDMEMBER
```

---

# Key Mental Model

> **Redis Set = unordered collection optimized for uniqueness, membership checks, and set-based relationships.**

Remember:

```text
SADD       → Add
SREM       → Remove
SMEMBERS   → Read all
SISMEMBER  → Check membership
SCARD      → Count

SINTER     → Common
SUNION     → Combined
SDIFF      → A minus B

SSCAN      → Iterate
SPOP       → Random + remove
SRANDMEMBER → Random + keep
```



# Redis Sorted Sets (ZSET)

## Core Concept

A Redis **Sorted Set** is a collection of **unique members**, where every member has an associated **score**.

Redis automatically maintains ordering based on the score.

```text
Set:
alice
bob
charlie

Sorted Set:
alice   → 100
bob     → 250
charlie → 175
```

Mental model:

```text
Set       → unique members
Sorted Set → unique members + score-based ordering
```

The score is an ordering value. It does not have to represent a literal "score".

---

# Core Commands

## ZADD

Add a member with a score.

```redis
ZADD key score member
```

Example:

```redis
ZADD devsync:leaderboard 100 alice
ZADD devsync:leaderboard 250 bob
ZADD devsync:leaderboard 175 charlie
```

Adding an existing member updates its score.

```redis
ZADD devsync:leaderboard 500 alice
```

Alice's score becomes `500`; a duplicate member is not created.

---

## ZRANGE

Return members ordered from lowest score to highest.

```redis
ZRANGE key start stop
```

Example:

```redis
ZRANGE devsync:leaderboard 0 -1
```

Include scores:

```redis
ZRANGE devsync:leaderboard 0 -1 WITHSCORES
```

---

## ZREVRANGE

Return members from highest score to lowest.

```redis
ZREVRANGE key start stop
```

Example:

```redis
ZREVRANGE devsync:leaderboard 0 -1 WITHSCORES
```

Mental model:

```text
ZRANGE    → low → high
ZREVRANGE → high → low
```

---

## ZSCORE

Get a member's score.

```redis
ZSCORE key member
```

Example:

```redis
ZSCORE devsync:leaderboard alice
```

---

## ZRANK

Get a member's rank from lowest score to highest.

Ranks are zero-based.

```redis
ZRANK key member
```

Example:

```text
alice   → 100 → rank 0
charlie → 175 → rank 1
bob     → 250 → rank 2
```

---

## ZREVRANK

Get a member's rank from highest score to lowest.

```redis
ZREVRANK key member
```

For a leaderboard:

```text
highest score → rank 0
```

---

## ZINCRBY

Increment or decrement a member's score.

```redis
ZINCRBY key increment member
```

Example:

```redis
ZINCRBY devsync:leaderboard 100 alice
```

If Alice had `750`, she becomes `850`.

Decrement:

```redis
ZINCRBY devsync:leaderboard -25 alice
```

Useful for:

```text
points
reputation
scores
ranking counters
```

---

## ZREM

Remove a member and its score.

```redis
ZREM key member
```

---

# Score-Based Queries

## ZRANGEBYSCORE

Retrieve members within a score range.

```redis
ZRANGEBYSCORE key min max
```

Example:

```redis
ZRANGEBYSCORE leaderboard 800 1300
```

Returns members whose scores are between `800` and `1300`.

The score does not need to represent points.

It can represent:

```text
priority
timestamp
price
rating
reputation
ranking value
```

---

## Exclusive Boundaries

Use `(` for an exclusive boundary.

```redis
ZRANGEBYSCORE leaderboard (100 500
```

Means:

```text
score > 100
score <= 500
```

Useful when processing ranges without repeating boundary values.

---

## ZCOUNT

Count members within a score range without returning the members.

```redis
ZCOUNT key min max
```

Example:

```redis
ZCOUNT leaderboard 800 1300
```

Mental model:

```text
ZRANGEBYSCORE → give me members
ZCOUNT         → tell me how many
```

---

# Priority Queue Pattern

A Sorted Set can act as a priority queue.

Example:

```text
job-1 → 10
job-2 → 50
job-3 → 20
```

If lower score means higher priority:

```text
10 → job-1
20 → job-3
50 → job-2
```

Consume the lowest-score job:

```redis
ZPOPMIN jobs
```

`ZPOPMIN` returns the member and score **and removes the member**.

---

## ZPOPMAX

Remove and return the highest-score member.

```redis
ZPOPMAX jobs
```

Mental model:

```text
ZPOPMIN → lowest score + remove
ZPOPMAX → highest score + remove
```

---

# Time-Based Scheduling

A very important ZSET pattern is using the score as a timestamp.

Example:

```text
scheduled:jobs

email:101 → 1723700100
email:102 → 1723700200
email:103 → 1723700500
```

The score represents the scheduled execution time.

A worker can query jobs whose timestamp is due:

```redis
ZRANGEBYSCORE scheduled:jobs -inf <current_timestamp>
```

Architecture:

```text
Producer
   ↓
ZADD scheduled:jobs timestamp job
   ↓
Redis ZSET
   ↓
Scheduler / Worker
   ↓
Query jobs whose score <= current time
```

This is useful for:

```text
delayed jobs
scheduled tasks
retry scheduling
expiration workflows
```

---

# Leaderboard Pattern

```text
devsync:reputation

alice   → 850
bob     → 1200
charlie → 950
eve     → 1500
```

Highest first:

```redis
ZREVRANGE devsync:reputation 0 -1 WITHSCORES
```

Get a user's leaderboard position:

```redis
ZREVRANK devsync:reputation alice
```

Update reputation:

```redis
ZINCRBY devsync:reputation 100 alice
```

---

# ZRANGESTORE

Store the result of a range query in another Sorted Set.

```redis
ZRANGESTORE destination source start stop
```

Example:

```redis
ZRANGESTORE top:users leaderboard 0 9 REV
```

Useful for creating derived/materialized subsets.

### Caution

Derived Sorted Sets can become stale when the source changes.

---

# Iterating Large Sorted Sets

Use:

```redis
ZSCAN key cursor
```

instead of blindly retrieving a huge collection when incremental iteration is appropriate.

Related commands:

```text
SCAN  → Keys
HSCAN → Hash
SSCAN → Set
ZSCAN → Sorted Set
```

---

# Equal Scores

Members in a Sorted Set are unique.

If two members have the same score, Redis uses deterministic member ordering for tie-breaking rather than insertion order.

Do not rely on:

```text
"first added wins"
```

If your application requires a specific tie-breaking rule, design for it explicitly.

---

# Real-World Use Cases

Sorted Sets are useful for:

```text
Leaderboards
Rankings
Priority queues
Scheduled jobs
Delayed jobs
Time-based ordering
Reputation systems
Score ranges
Top-N queries
```

---

# List vs Set vs Sorted Set

| Requirement | Structure |
|---|---|
| Ordered sequence | List |
| Queue / stack | List |
| Recent/latest N items | List |
| Unique membership | Set |
| Fast membership check | Set |
| Intersection / union / difference | Set |
| Unique members + score | Sorted Set |
| Ranking | Sorted Set |
| Priority | Sorted Set |
| Scheduling by timestamp | Sorted Set |
| Score-range queries | Sorted Set |

Decision rule:

```text
Need ordering by position?
    ↓
List

Need uniqueness / membership?
    ↓
Set

Need uniqueness + score-based ordering?
    ↓
Sorted Set
```

---

# Phase 1 Commands to Know

## Core

```text
ZADD
ZRANGE
ZREVRANGE
ZSCORE
ZRANK
ZREVRANK
ZREM
ZINCRBY
```

## Score Queries

```text
ZRANGEBYSCORE
ZCOUNT
```

## Queue / Consumption

```text
ZPOPMIN
ZPOPMAX
```

## Iteration / Advanced

```text
ZSCAN
ZRANGESTORE
```

---

# Key Mental Model

> **Redis Sorted Set = unique members ordered by an associated score.**

Remember:

```text
ZADD          → Add / update member score
ZRANGE        → Low → high
ZREVRANGE     → High → low
ZSCORE        → Get score
ZRANK         → Rank low → high
ZREVRANK      → Rank high → low
ZINCRBY       → Change score
ZREM          → Remove

ZRANGEBYSCORE → Query score range
ZCOUNT        → Count score range

ZPOPMIN       → Remove lowest score
ZPOPMAX       → Remove highest score

ZSCAN         → Incremental iteration
```

## Most Important Patterns

```text
Leaderboard:
ZADD + ZREVRANGE + ZREVRANK + ZINCRBY

Priority queue:
ZADD + ZPOPMIN

Scheduling:
ZADD with timestamp as score
+ ZRANGEBYSCORE for due items

Top N:
ZREVRANGE key 0 N-1 WITHSCORES
```
