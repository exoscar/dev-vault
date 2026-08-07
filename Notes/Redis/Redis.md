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

