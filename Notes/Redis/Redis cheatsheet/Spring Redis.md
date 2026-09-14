### Spring Data Redis
Spring Data Redis is the **Spring abstraction layer for Redis**.

It gives your application convenient APIs for Redis operations without forcing your business code to deal with low-level client/network details.

It provides things such as:
- `RedisTemplate`
- `StringRedisTemplate`
- Redis repositories
- Spring Cache integration
- transaction/pipeline support
- serializers
- connection abstractions

### RedisTemplate

`RedisTemplate` is the **general-purpose programming interface you use to perform Redis operations from Java**.
```
opsForValue()  → Strings
opsForHash()   → Hashes
opsForList()   → Lists
opsForSet()    → Sets
opsForZSet()   → Sorted Sets
opsForStream() → Streams
```
Eg
```
redisTemplate.opsForHash()
             .put("workspace:123", "name", "DevSync");
redisTemplate.opsForValue()
             .set("issue:42", issue);
```

### Lettuce ★★★
Lettuce is the actual java Redis client
Lettuce knows how to communicate with a Redis server.

It handles things such as:
- Redis connections
- sending commands
- receiving responses
- synchronous operations
- asynchronous operations
- reactive operations
- connection reuse/multiplexing


>Spring Data Redis gives you the Spring programming model. Lettuce provides the underlying Redis communication.



connections are **reused** according to the client/API/connection configuration.
```
              ┌── Request 1 ──┐
              │               ↓
Application ──┼── Request 2 → Redis connection/client
              │               ↑
              └── Request 3 ──┘
```

The exact behavior depends on which Spring Data Redis API you're using and how connections are configured.

#### Thread Safety
Lettuce's API is designed to support concurrent use of its connections in appropriate modes.

Lettuce can support concurrent operations through its connection architecture.

#### Multiplexing
Instead of creating a separate connection for every operation, multiple commands can use the same underlying connection in appropriate Lettuce usage.

```
                ┌── GET issue:1
                ├── GET issue:2
Lettuce         ├── GET issue:3
connection ─────┼── GET issue:4
                └── GET issue:5
                         ↓
                       Redis
```

This is **multiplexing**.
The client manages multiple logical operations over shared connection infrastructure.
This is particularly useful for asynchronous/reactive workloads.

#### Synchronous API
```
String value =
    redisTemplate.opsForValue().get("issue:42");
```

```
Application
    ↓
send GET
    ↓
wait
    ↓
Redis response
    ↓
return value
```
Your Java code waits for the operation to complete.
For ordinary Spring MVC applications, this programming model is very common.

#### Asynchronous API
With asynchronous programming, you don't necessarily block waiting for the response.
```
Application
    ↓
send GET
    ↓
continue doing other work
    ↓
Redis response arrives
    ↓
callback / future completes
```

```
Synchronous
→ send request
→ wait for result

Asynchronous
→ send request
→ don't immediately wait
→ result arrives later
```


Mental model
```
┌─────────────────────────────┐
│       Spring Boot App       │
│                             │
│   Service / Repository      │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│      Spring Data Redis      │
│                             │
│      RedisTemplate          │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│           Lettuce           │
│                             │
│  Redis Java client          │
│  Connections                │
│  Network communication      │
│  Sync / Async / Reactive    │
└──────────────┬──────────────┘
               ↓
             TCP
               ↓
┌─────────────────────────────┐
│           Redis             │
│                             │
│ GET / SET / HGET / etc.     │
└─────────────────────────────┘
```

RedisConnectionFactory
>RedisConnectionFactory is Spring Data Redis's abstraction for obtaining Redis connections. The concrete implementation uses the configured Redis client, such as Lettuce, and connection reuse/pooling behavior depends on the client and configuration.



#### StringRedisTemplate
`StringRedisTemplate` is a specialized form of `RedisTemplate` designed for the common case where your **Redis keys and values are strings**.

```
RedisTemplate<K, V>
        ↓
general-purpose Redis operations

StringRedisTemplate
        ↓
String keys + String values
```



Redis stores data as bytes, so Java objects must be serialized before being written and deserialized when read. The serialization strategy affects payload size, CPU cost, latency, interoperability, security, and compatibility when application models evolve. For application caches, JSON-based serialization is often a practical choice because it is portable and debuggable, while the exact choice should depend on the data and access pattern.