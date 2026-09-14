A Redis **List** is an ordered collection of strings where elements can be added or removed efficiently from either end.

```
LEFT                         RIGHT
 ↓                             ↓
[item-1] [item-2] [item-3]
```

### Basic Commands
Add to left/right.
```
LPUSH key value
RPUSH key value
```

Remove from left/right.
```
LPOP key
RPOP key
```

Read a range.
```
LRANGE key start stop
```

Read an element by index.
```
LINDEX key index
```

Get list length.
```
LLEN key
```

Replace an element at an index.
```
LREM key count value
```

Remove matching elements.
```
LTRIM key start stop
```

Keep only the specified range.
```
LINSERT key BEFORE|AFTER pivot value
```


### Queue Pattern — FIFO
```
RPUSH queue job-1
RPUSH queue job-2
RPUSH queue job-3

LPOP queue
```

```
RPUSH → LPOP = FIFO
```
Alternative
```
LPUSH → RPOP = FIFO
```

### Stack Pattern — LIFO
```
LPUSH → LPOP = LIFO
```
The most recently added element is removed first

### Blocking Queues
```
BLPOP queue 0
BRPOP queue 0
```

`B` = blocking.
`0` = wait indefinitely.
Useful for simple producer/consumer workers.
```
Producer
   ↓
Redis List
   ↓
Worker
```

`BLPOP` / `BRPOP` remove the item when consumed.

#### Multiple Queues
```
BLPOP high normal low 0
```
Redis checks the keys in the order supplied.
```
high → normal → low
```
Important: this does **not** make Lists true priority queues. The priority behavior comes from the order of keys passed to `BLPOP`.

## Bounded List Pattern
Useful for keeping only the latest N items:
```
LPUSH recent:event event-1
LTRIM recent:event 0 99
```

Keeps only the latest **100 elements**.
Useful for:
- Recent activity
- Recent searches
- Notifications
- Rolling history

### Lists vs Streams
Lists
Best for:
```
Simple queues
Stacks
Recent items
Bounded collections
```

Streams
Better when you need:
```
Multiple consumers
Consumer groups
Acknowledgements
Consumer progress
Event replay
More reliable event processing
```

Key limitation of Lists:
```
BLPOP
  ↓
item removed
```
If the worker crashes after consuming the item, the item is already gone.


