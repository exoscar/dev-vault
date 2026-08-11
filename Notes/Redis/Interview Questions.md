Main notes -- [[Redis]]
- Why is Redis single-threaded?
- Why doesn't Redis suffer from race conditions for ordinary commands?
- What is an event loop?
- What is non-blocking I/O?
- What is a context switch?
- Why can multiple threads reduce performance?
- Is Redis completely single-threaded today?
- Why does Redis avoid heavy locking?


1. What is Redis?
```
Redis is an in-memory data structure server that provides multiple data structures and is commonly used for caching, ephemeral state, counters, messaging, and other low-latency workloads.
```

2. What is SDS?
```
SDS is Redis's Simple Dynamic String representation. It stores metadata such as string length alongside the character data, allowing efficient length retrieval and safer binary-data handling compared with traditional null-terminated C strings.
```

3. Why doesn't Redis use normal C strings?
```
Because traditional C strings:
- Depend on null termination.
- Make length retrieval O(n).
- Aren't convenient for arbitrary binary data.
```

4. Is Redis's String only text?
```
No.
It can represent:
- Text
- Integers
- Binary data
- Serialized objects
- Counters
```
5. Is Redis faster than a Java HashMap?
```
Generally **no** for a single JVM.

A Java `HashMap` avoids the network boundary.

Redis's advantage is **shared, centralized, extremely fast state across processes/servers**, not beating local memory access.
```
6. Why does Redis use RDB?
```
Because it provides a point-in-time snapshot of the in-memory dataset that can be persisted to disk and used to recover the dataset after restart.
```

7. Does Redis stop serving requests during RDB snapshotting?
```
After `fork()`, parent and child initially share memory pages. When one process modifies a shared page, the OS creates a private copy for the modifying process.
```

8. What is Copy-on-Write?
```
After `fork()`, parent and child initially share memory pages. When one process modifies a shared page, the OS creates a private copy for the modifying process.
```
9. Why can RDB increase memory usage?
```
Because writes to memory pages shared with the snapshot child can trigger Copy-on-Write, creating additional physical memory pages.
```
10. What is RDB's biggest durability weakness?
```
Writes occurring after the most recent snapshot can be lost if Redis crashes before another snapshot is created.
```


11. Give me one situation where you **wouldn't** want to blindly combine everything into one huge `MGET`.
```
Batching reduces round trips, but excessively large batches can create payload, latency, and memory problems.
```

