
Redis -> in-memory data structure server
Redis supports rich data structures, atomic operations, pub/sub, streams, scripting, replication, and clustering


Why Redis?
Because Computers are slow
```
CPU Registers

↓

CPU Cache

↓

RAM

↓

SSD

↓

Network Storage
```
each step is slower than the previous one

|Storage|Approx. Latency|
|---|--:|
|CPU Register|<1 ns|
|L1 Cache|~1 ns|
|L2 Cache|~4 ns|
|L3 Cache|~10–20 ns|
|RAM|~100 ns|
|NVMe SSD|~100 µs|
|HDD|~5–10 ms|
|Network|~0.5–100 ms|
PostgresSql stores data on disk
A PostgreSQL table eventually lives on persistent storage.
Even if PostgreSQL caches frequently accessed pages in memory, its design centers on durability.
```
Application
↓
PostgreSQL
↓
Shared Buffers
↓
Disk
```
Every durable write must eventually reach persistent storage.
That introduces latency.

Redis Stores Data in RAM
Redis keeps its primary dataset in memory
no disk lookup is required for normal reads


Why not store everything in RAM?
RAM has tradeoff 
- Expensive (32GB RAM much more expensive than 32GB SSD)
- Volatile - if power is lost (RAM -> GONE) 
	- it addresses this with persistence mechanisms(RDB,AOF)
- Limited Capacity - servers typically have (64 GB Ram vs 4TB SSD) you cant store everything economically


So Why Redis?
Which data is valuable enough to justify keeping in RAM?
Examples include:
- Active user sessions
- Authentication tokens
- Shopping carts
- Frequently accessed product data
- Leaderboards
- Rate-limiter counters
- Live notifications
- Short-lived analytics
- Queue state
- Distributed locks

Java Collections Comparision

|Java|Redis|
|---|---|
|`HashMap`|Hash|
|`ArrayList`|List|
|`HashSet`|Set|
|`TreeSet`|Sorted Set|
|`AtomicLong`|Counter (`INCR`)|
|`BlockingQueue`|List/Stream patterns|
Java collections live inside JVM. While Redis data lives in a separate server that multiple applications can share

PostgreSQL Comparison

|PostgreSQL|Redis|
|---|---|
|Disk-first design|Memory-first design|
|SQL|Key-based operations|
|ACID transactions|Limited transactional model|
|Rich relational queries|Optimized data structure operations|
|Joins|No joins|
|Long-term storage|Fast-access storage|
|Durable by default|Durability is configurable|
Redis complements PostgresSQL rather than replacing it

Big-O perspective
Accessing data in Redis often maps to efficient in-memory structures.
Typical examples:
- `GET` → **O(1)**
- `SET` → **O(1)**
- Hash field lookup → **O(1)** average
Fast algorithms plus RAM-resident data are what make Redis performant.

# Memory Tradeoffs
Advantages:
- Extremely low latency
- High throughput
- Efficient atomic operations
- Simple concurrency model
Costs:
- Higher memory cost per GB
- Dataset size constrained by RAM
- Requires eviction or persistence strategies
- Restart and recovery considerations


#### Why is Redis Single-Threaded?
Most of the tasks in Redis are performed immediately. such there will be very minimal wait for the requests.

If a task finishes almost immediately, creating more threads may reduce performance.
because
Each thread requires
- Memory for its stack
- Scheduling by the operating system
- Context switches  -->  unnecessary overhead
- Synchronization when accessing shared data --> CPU sync caches -- costs time


### Redis's Radical Decision

What if we avoid these costs entirely?
it just use `One Command Execution Thread`

ASCII:
```
           Client 1
               │
           Client 2
               │
           Client 3
               │
           Client 4
               │
      ┌─────────────────┐
      │   TCP Socket     │
      └─────────────────┘
               │
               ▼
      ┌─────────────────┐
      │ Event Loop      │
      └─────────────────┘
               │
               ▼
      Execute Command
               │
               ▼
         Return Response
```

Only one command executed at a time

that means:
- No command-level locks
- No race conditions inside Redis's core data structures
- No synchronization for ordinary operations

It doesn't mean clients will wait --> commands are queued..

#### Event Loop
```
while (true) {
    accept new connections
    read incoming commands
    execute one command
    send responses
}
```

#### Non-blocking I/O
Redis non-blocking I/O uses a **single thread** and a **multiplexing event loop** (`epoll`) to monitor thousands of network sockets simultaneously

Instead of waiting for slow data transfers, Redis only processes connections that are **instantly ready**, ensuring maximum speed without wasting CPU cycles.

### Why Redis is so Fast
its fast because
- Data is in memory
- Efficient data structures
- minimal allocations
- commands are simple
- event-driven architecture
- little lock contention
- very few context switches

Time Complexities

|Command|Complexity|
|---|---|
|GET|O(1) average|
|SET|O(1) average|
|INCR|O(1)|
|HGET|O(1) average|
|LPUSH|O(1)|
|SADD|O(1) average|


#### When Single-threaded becomes a problem
Imagine:
`Get Key`
Microseconds

now imagine
`Sort 20 million elements`
milliseconds or longer

while redis is doing that `Everyone waits`
This is why some Redis commands are marked as potentially expensive and should be used carefully in production.

### Redis 6 and Beyond
Redis introduced **I/O threads** for tasks like reading requests from sockets and writing responses.
However, **command execution remains single-threaded** in the general case.
```
Read Network
(Thread Pool)
↓
Main Thread
Execute Command
↓
Write Network
(Thread Pool)
```


### Redis Imternal flow
```
          TCP Connections
                │
                ▼
        Kernel Socket Buffers
                │
        epoll()/kqueue()
                │
       Ready File Descriptors
                │
                ▼
      Redis Event Loop Iteration
                │
     Read Request from Socket
                │
         Parse RESP Protocol
                │
        Execute Command
                │
        Write Response Buffer
                │
          Send to Client
```


We know `GET user:1` time complexity is  O(1)
even for 1 million entries its O(1)?

Redis stores keys in a hash table(internally called `dict`)
redis doesnt not look through every key.
instead
- Compute the hash
- jump direclty to the bucket
- compare only a small number of entries in that bucket
- return the value
Average case O(1)
Worse Case O(n) --> when during poor hashfunction 
Fortunately:
- Good hash functions
- Proper resizing (rehashing)
- Keeping the load factor under control
make this situation extremely rare.

this hash table is internal hash called as `dict` - not accessible for users. which is separate from the another data structure `HASH` -- DATA TYPE


Flow 

### Step 1 -- Spring Boot
Your application serializes the command.
```
GET user:1
```
Redis doesn't understand Java objects.
It understands a protocol called **RESP (Redis Serialization Protocol)**.
So the client library converts the command into RESP.
conceptually:
```
GET user:1
↓
RESP
↓
Bytes
```
```
*2
$3
GET
$6
user:1
```

### Step 2 -- TCP
Those bytes are sent over a TCP connection.
```
Spring Boot
↓
TCP Socket
↓
Network
```

### Step 3 -- Operating System
The packet reaches the server.
```
Network Card
↓
Linux Kernel
↓
TCP Stack
↓
Socket Buffer
```
At this point, Redis hasn't done anything yet.
The data is sitting in the kernel's socket buffer.

### Step 4 -- epoll
`epoll` is the Linux kernel system call that acts as the **high-speed notification engine** for the Redis event loop
it tells Redis exactly which network sockets have data ready to be processed, allowing a single thread to manage millions of concurrent connections efficiently. 

```
Linux
↓
epoll
↓
Socket 42 has data.
```
The OS notifies Redis that this socket is ready.
This is why Redis scales to thousands of connections.

### Step 5 -- Event Loop
The Redis event loop wakes up.
```
Event Loop
↓
Read Socket
↓
Receive Bytes
```

### Step 6 -- Parse RESP
Redis now sees:
```
GET user:1
```
It parses the protocol.
```
Bytes
↓
RESP Parser
↓
Command Object
```
Now Redis knows:
- Command = GET
- Key = user:1

### Step 7 -- Hash Function
Redis computes the hash of:
```
"user:1"
```

```
"user:1"
↓
Hash Function
↓
Hash Value
```

### Step 8 — Hash Table Lookup
Redis jumps directly to the correct bucket.
```
Bucket 5132
↓
user:1
↓
Value
```
No scanning of the database.
This is the average **O(1)** lookup you described earlier.
### Step 9 — Redis Object
Redis finds the value.
For example:
```
"user:1"
↓
Redis Object
↓
String
↓
"John"
```

`"John"` is not stored as a plain C string—it uses Redis's internal object model.

---

### Step 10 — Serialize Response
Redis converts the value back into RESP.
```
John
↓
RESP
↓
Bytes
```
---
### Step 11 — Write to Socket

```
Bytes
↓
Socket
↓
Kernel
↓
TCP
↓
Spring Boot
```

---

### Step 12 — Client Library
The Redis Java client receives the bytes and converts them into a Java object.
```
RESP
↓
Java String
↓
"John"
```
Your application gets:
```
String name = redisTemplate.opsForValue().get("user:1");
```
Done.

## Entire Flow
```
Spring Boot
      │
      ▼
Redis Client (Lettuce/Jedis)
      │
      ▼
RESP Encoding
      │
      ▼
TCP Socket
      │
      ▼
Linux Kernel
      │
      ▼
Socket Buffer
      │
      ▼
epoll
      │
      ▼
Redis Event Loop
      │
      ▼
RESP Parser
      │
      ▼
Hash Function
      │
      ▼
Hash Table Lookup
      │
      ▼
Redis Object
      │
      ▼
RESP Encoding
      │
      ▼
TCP Socket
      │
      ▼
Spring Boot
```


### The Redis Object Model
Redis Has Keys and Values
```
Hash Table / Dictionary

        key
         ↓
     "user:1"
         ↓
       value
         ↓
    Redis Object
```

Redis Supports Multiple Data Types
```
String
Hash
List
Set
Sorted Set
Stream
Bitmap
HyperLogLog
Geospatial
```

So Redis needs some way to represent these different kinds of values internally.
That's where the **Redis object model** comes in.

Simplified mental model
```
Redis Dictionary
       │
       ├── "user:1"
       │       │
       │       ▼
       │   Redis Object
       │       │
       │       ▼
       │     String
       │       │
       │       ▼
       │      "John"
       │
       ├── "user:2"
       │       │
       │       ▼
       │   Redis Object
       │       │
       │       ▼
       │      Hash
       │
       └── "users"
               │
               ▼
           Redis Object
               │
               ▼
              Set
```

>The top-level dictionary maps keys to Redis values, while each value can use an appropriate internal representation.

Data Structure Determines Complexity
Different structures give you different operations.
String -> Avg O(1)
Hash -> Avg O(1)
List -> Usually O(1)
Set  -> Avg O(1)
SortedSet
Complexity depends on the operation and range size.
This is why **choosing the Redis data structure is an architectural decision**.

## Java Comparision
Suppose you're implementing DevSync in Java.
You might use:
```
HashMap<String, User>
```
for key-value lookup.
```
ArrayList<Notification>
```
for ordered items.
```
HashSet<UUID>
```
for unique members.
```
TreeSet<User>
```
for sorted data.
Redis essentially gives you these kinds of capabilities as **server-side shared data structures**.
That's the powerful part.
Instead of:
```
Spring Boot Instance 1
        ↓
    Java HashMap
```
and:
```
Spring Boot Instance 2
        ↓
    Different HashMap
```
you have:
```
Spring Boot 1 ──┐
                │
Spring Boot 2 ──┼──→ Redis
                │
Spring Boot 3 ──┘
```
All instances can access the same data structure.


### There is a important Cost
Redis being external means every operation crosses a network boundary.
Java:
```
map.get("user:1");
```
is happening inside your JVM.
Redis:
```
Application
   ↓
Network
   ↓
Redis
   ↓
Network
   ↓
Application
```
So even though Redis is extremely fast, it is **not as fast as an in-process Java `HashMap`**.

```
CPU cache
   ↓
Java object / local memory
   ↓
Redis
   ↓
PostgreSQL
   ↓
Remote service
```

### Why Redis uses its own string representation
C has strings like:
`char *name = "John";`
Why didn't Redis simply use ordinary C strings everywhere?
C string terminated by '\0' so system hast ot find the end by scanning 
getting the length can require O(n).
Redis frequently needs to know:
How long is this String

so Redis developed
SDS — Simple Dynamic String
```
┌────────────────────────────────────┐
│ length │ capacity │ flags │ data   │
└────────────────────────────────────┘
```

"John" 
```
length = 4
capacity = ...
data = J o h n
```

SDS also handles binary data.
Because SDS stores the length explicitly, Redis strings can contain arbitrary binary data.
For example:
```
10101010 00000000 11110001 ...
```
A zero byte doesn't necessarily mean:

> "The string ends here."

Because Redis already knows the length.
This makes Redis strings useful for things beyond human-readable text.

#### Redis String Is More Powerful Than It Sounds
Redis String
could represent:
```
text
integer
serialized JSON
binary data
counter
token
compressed data
```
Redis can treat the stored string representation as an integer for numeric operations.
That's why the Redis String data type is extremely useful.




```
                 REDIS
                   │
       ┌───────────┴────────────┐
       │                        │
 Architecture              Data Model
       │                        │
       ▼                        ▼
 Event Loop                Key → Value
       │                        │
 Non-blocking I/O        Multiple Data Types
       │                        │
 Single command            String
 execution                 Hash
       │                    List
       │                    Set
       │                    ZSet
       │
       ▼
Fast in-memory operations
       │
       ▼
Persistence
       │
   ┌───┴───┐
   RDB     AOF
```


always distinguish your data in redis
Pure cache --> can be rebuilt from source or truth(DB) --> redis persistence not required
Ephemeral State --> like rate limiters --> Redis state matters temporarily but losing it isn't catastrophic. --> persistence may/may not be needed as per Business requirement

Important State --> Redis contains information that affects application behavior and isn't trivially reconstructable.
```
Queue
Stream
Pending work
```

System of Record
If you're using Redis itself as the authoritative store:
```
Application
    ↓
Redis
    ↓
No PostgreSQL copy
```
Then durability requirements become much stronger.
At that point you need to think seriously about:
- RDB
- AOF
- Replication
- Sentinel
- Cluster
- Backups
- Recovery


## Redis Persistence
### RDB -(Redis Database)
How can Redis, whose working dataset lives in RAM, survive a process crash or restart?
>Take a snapshot of the in-memory dataset and save it to disk.

How can Redis take that snapshot without freezing the entire server?
Thats where `fork()` and Copy-on-Write enter

What Problem does RDB Solve?
Without Persistence:
```
Redis Process
     │
     X
   Crash
     │
     ▼
RAM disappears
     │
     ▼
Everything gone
```
RDB creates
```
Redis RAM
    │
    │ Snapshot
    ▼
dump.rdb
```
So after restart
```
Redis crashes
     │
     ▼
Redis starts
     │
     ▼
Load dump.rdb
     │
     ▼
Reconstruct RAM
```


RDB is a point-in-time snapshot of the Redis dataset
```
RDB represents a snapshot at a particular point in time, not a continuously updated copy.
```

Why fork()?
Now there is a problem --> assume the size of redis is around 100 GB. while copying that bulk data.. redis could experience a massive latency.

Redis Uses linux kernal `fork()`
```
Before fork:

             Redis Process
                  │
                  ▼
                 RAM
```
After fork
```
                 Redis
                   │
          ┌────────┴────────┐
          │                 │
        Parent            Child
          │                 │
    handles requests    creates RDB
```

#### Wait -- Did Redis Copy 100GB
This is where **Copy-on-Write (COW)** comes in.
When `fork()` happens, the parent and child initially share the same physical memory pages.

```
Parent Redis
     │
     ├──────────────┐
     │              │
     ▼              ▼
  Page A           Page B
     ▲              ▲
     │              │
     └──────┬───────┘
            │
         Child
```
Both processes can initially reference the same physical memory.

#### Copy-on-Write
Now imagine Redis modifies:
```
user:1
```
That data lives on a memory page.
Before modification:
```
Parent ──────┐
             ▼
          Page X
             ▲
             │
Child ───────┘
```
Parent wants to modify Page X.
The operating system says:
> "This page is shared. You can't modify the shared version."

So it creates a copy.
```
Parent ──────→ Page X'

Child  ──────→ Page X
```
The parent modifies:
```
Page X'
```
The child continues seeing:
```
Page X
```
That's **Copy-on-Write**.

The child is generating a snapshot of the old state.
The parent continues modifying the live dataset.
Therefore:
```
                fork()
                  │
        ┌─────────┴─────────┐
        │                   │
      Parent              Child
        │                   │
Live Redis             RDB Snapshot
        │                   │
Changes memory         Reads memory
        │                   │
        └──── COW ──────────┘
```
The child sees the state as it existed when the snapshot started.
The parent continues serving requests.

##### in RDB there is a big memory tradeOff
Suppose Redis has 100GB and the snapshot is running

if application modifies a large portion of the dataset
many pages need to be copied.
So physical memory usage can increase substantially.
```
Before fork:
100 GB
After fork:
100 GB shared

During heavy writes:
100 GB shared
+
30 GB copied pages

≈ 130 GB
```
The exact overhead depends on the workload and memory-page behavior.
This is why RDB snapshots can cause **memory pressure** during heavy write workloads.

RDB utilizes more memory and higher cpu utilization

#### RDB's Major advantage
RDB files are compact and efficient for backups.
```
RAM Dataset
     │
     ▼
  Snapshot
     │
     ▼
  RDB file
```
This makes RDB useful for:
- Backups
- Disaster recovery
- Faster dataset transfer
- Replica initialization
- Periodic persistence

#### RDB's Major Disadvantage
if snapshots happen every 5 minutes.
Exmple
```
10:00 ───── RDB
10:01 ───── Write
10:02 ───── Write
10:03 ───── Write
10:04 ───── Write
10:05 ───── RDB
```
if redis crash at `10:04:58`
the lastest snapshot may be from 10:00. all data after 10:00 is lost

>RDB gives you point-in-time snapshots, not every individual write.

#### RDB vs PostgreSQL WAL
postgres commonly uses

```
Transaction
    │
    ▼
WAL
    │
    ▼
Durable storage
```

Redis RDB::
```
Current Dataset
      │
      ▼
Snapshot
      │
      ▼
RDB file
```

Very different model.
PostgreSQL is designed around durable transactional storage.
Redis is designed around an in-memory dataset with optional persistence.



"At what point does Redis consider a write safe, and how much data can I lose if the process or machine fails?"


### Append On File


Three AOF Policies
1. appendfsync always
```
Write
 ↓
AOF
 ↓
fsync
 ↓
Return
```
The system waits for the sync operation.
Tradeoff:
```
Durability ↑
Latency ↑
Throughput ↓
```
2. appendfsync everysec
```
Write
 ↓
AOF
 ↓
Return

      ...

~1 second

      ↓

fsync
```
Tradeoff:
Potentially around one second of recent writes can be lost during a catastrophic failure.
```
Durability
    ↑
Performance
    ↑↑
```

3. appendfsync no
```
Redis leaves synchronization largely to the operating system.

Write
 ↓
AOF
 ↓
OS decides when to flush
```
Redis has less control over persistence timing
Tradeoff
```
Performance ↑
Durability predictability ↓
```

|Policy|Durability|Performance|Risk|
|---|---|---|---|
|`always`|Highest|Lowest|Lowest write-loss window|
|`everysec`|High|High|Roughly ~1 sec potential loss|
|`no`|Lowest/predictability|Highest|OS-controlled delay|


## Operations
SET flow 
```
SET user:123:name Alice
        │
        ▼
Parse command
        │
        ├── operation = SET
        ├── key = user:123:name
        └── value = Alice
        │
        ▼
Hash key
        │
        ▼
Redis dictionary
        │
        ▼
Find existing entry?
      /       \
    yes        no
     │          │
   update     create
     │          │
     └────┬─────┘
          ▼
      Redis object
          │
          ▼
   internal representation
```

O(1) refers primarily to the average key lookup, not to the total amount of work involved in retrieving/transmitting an arbitrarily large value.

### `SET` and `GET`

`SET user:123:name Alice` 
`GET user:123:name`

when we want multiple keys at a time then. 

`MGET user:123:name user:123:email user:123:status`
```
Java
  │
  │ one request
  ▼
Redis
  │
  ├── lookup key 1
  ├── lookup key 2
  └── lookup key 3
  │
  ▼
one response
  │
  ▼
Java
```

```
MGET N keys
    ↓
N key lookups
    ↓
O(N)
```

Dont use MGET everywhere. check the below line
"What is the right amount of data to retrieve in one operation for this workload?"

```
SET verification:123 849201 EX 300
```
key   = verification:123
value = 849201
TTL   = 300 seconds

Redis Strings become much more useful when combined with Redis's other capabilities, rather than viewed as isolated commands.

### INCR / DECR
it will create the key if it doesnt exist and initialize it with zero then then
increment/DECR by one 
```
INCR page_views
DECR counter
```
conceptually
```
Client A
   │
   ▼
INCR
   │
   ▼
Redis
   │
   ├── read current value
   ├── increment
   └── store new value
```

### INCRBY/DECRBY
increment by n
```
INCRBY inventory:123 50
DECRBY counter 10
```


### Counters + Integer Semantics
```
SET counter 100
```

even though 100 looks like an integer, at the Redis data-model level this is still a **String**.

then INCR counter tells redis that
>Interpret the current String value as an integer, increment it, and store the result.

```
"100"
  │
  │ INCR
  ▼
"101"
```

```
Redis data type
      ↓
String

Operation semantics
      ↓
numeric increment
```
That is why Redis doesn't need a separate user-facing `Integer` data type just to support counters.

#### What if the value isn't numeric?
Suppose:
```
SET counter hello
```
Then:
```
INCR counter
```
cannot interpret `"hello"` as a valid integer, so Redis returns an error.
This gives you another important design rule:
> **The operation you choose must be compatible with the representation of the value you're storing.**



Count API requests made by each workspace during a time window, then automatically remove the counter.

INCR workspace:123:requests:06:52
EXPIRE workspace:123:requests:06:52 60
```
workspace:123:requests:06:52
        │
        ├── value → 847
        │
        └── TTL   → 60 seconds
```


#### `SETNX`
`SETNX` lets Redis perform "check whether key exists + create if absent" as one atomic operation.
```
SETNX payment:request:abc123 processed
```

This makes conditional writes useful for things like:
- idempotency markers
- "initialize once" state
- basic locking patterns
- duplicate-event protection

To set multi keys
#### MSETNX


Redis lock patterns generally involve **conditional setting + expiration + ownership validation**, rather than blindly using `SETNX`.

Modern `SET` is even more useful
```
SET payment:request:abc123 processed NX EX 300

NX -- if key abset set
XX -- only if key present update
```
Conceptually
```
SET value
  +
NX → only if key doesn't exist
  +
EX 300 → expire after 300 seconds
```
This is particularly useful because you can combine the conditional write and expiration into one command.


#### The String decision
Consider a DevSync issue:
```
Issue #123

title      = "Fix login bug"
status     = "OPEN"
priority   = "HIGH"
assignee   = "alice"
```

#### Option A - JSON String
```
issue:123
    │
    ▼
{
  "title": "Fix login bug",
  "status": "OPEN",
  "priority": "HIGH",
  "assignee": "alice"
}
```
your application does:
```
GET issue:123
      ↓
deserialize JSON
      ↓
Java object
```
if status changes
```
GET
 ↓
deserialize
 ↓
change status
 ↓
serialize
 ↓
SET
```

#### Option B — Hash
```
issue:123
    │
    ├── title    → Fix login bug
    ├── status   → OPEN
    ├── priority → HIGH
    └── assignee → alice
```
Now you can do:
```
HGET issue:123 status
```
or:
```
HSET issue:123 status DONE
```
without retrieving and rewriting the entire object.


Don't directly conclude "Hash is better"

if application mostly or all the time need the entire object then a serialzied String can be perfectly reasonable
```
Redis
 ↓
one value
 ↓
deserialize
```
But if your application frequently needs:
```
just status
just priority
increment commentCount
change assignee
```
a Hash becomes more attractive

So the decision should be driven by **access patterns**, not by the statement:
> "Hashes are structured, therefore they're better."

## The four questions to ask yourself 
When deciding between a String and Hash, ask:
1. Do I usually read the whole object?
```
Yes → String becomes attractive
No  → Hash becomes attractive
```
 2. Do I frequently update individual fields?
```
Yes → Hash becomes attractive
No  → String remains reasonable
```
 3. Do I need application-level serialization?
String:
```
Java object
 ↓
JSON
 ↓
Redis
```
Hash:
```
field → value
```
So Hash can reduce some serialization/deserialization work for field-level access.

 4. What are my memory/access-pattern requirements?
Neither representation automatically wins.
You need to consider:
```
number of objects
number of fields
field sizes
read/write patterns
network payload
serialization cost
memory overhead
```
This is exactly where Redis data modeling becomes engineering rather than command memorization.

```
Redis String
    ↓
redisObject / internal representation
    ↓
SET / GET
    ↓
MGET / MSET
    ↓
INCR / DECR
    ↓
atomic counters
    ↓
TTL
    ↓
SETNX / NX
    ↓
conditional writes
    ↓
String vs Hash
```

### TYPE key
returns the type

### DEL key
returns int --> 0 no key found 1 - 1 key deleted

### TTL key
get the ttl

|   TTL | Meaning                                     |
| ----: | ------------------------------------------- |
| `> 0` | Key exists and expires in that many seconds |
|  `-1` | Key exists but has **no expiration**        |
|  `-2` | Key **doesn't exist**                       |

### GETSET key value
it will return the existing value and set the value


### APPEND
it will append the text to existing key. it adds to exising value it doesnt modify it
preserves TTL 
for non existing key - it create a key and set the value
```
SET msg "Hello"  --> OK

APPEND msg " World" --> length of new string

GET msg --> "Hello World"

STRLEN msg --> length of message
```

### GETRANGE key start end
returns the substring  -- both start and end inclusive
```
SET text "Hello Redis"

GETRANGE text 0 4  -- 'Hello'

GETRANGE text 6 10 --> 'REDIS'

GETRANGE text 0 -1  --> 'Hello REDIS' --> neg indexing
```


#### SETRANGE
`SETRANGE` modifies the String starting at the specified offset; it doesn't replace the entire value.
```
SET text "Hello World"   -- ok
SETRANGE text 6 "Redis"  -- 11
GET text  -- "hello redis"
STRLEN text  -- "11"

x → "abc"
SETRANGE x 5 "Z"
Redis needs to reach offset `5`, so it creates the missing space with **zero bytes (`\0`)**.

a b c \0 \0 Z
0 1 2  3  4 5

STRLEN x -- 6
GET X -- abc\0\0Z

```

if offset is within the string then it will update and appedn the extras char.
if offset is outside the string range -- then i will fill with zero bytes until the offset then it will add the value

```
offset < current length
    → overwrite existing bytes

offset > current length
    → extend String
    → missing bytes become zero bytes
```

### GETDEL
it will get the value and delete the value from the memory