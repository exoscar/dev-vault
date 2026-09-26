### Kafka Observability
How do i know kafka is healthy in production?
```
Kafka Observability
│
├── 1. Consumer lag
├── 2. Processing latency
├── 3. Consumer failures/retries
├── 4. DLT activity
└── 5. Producer failures
```

Consumer Lag --> The difference of committed offset and produced offset.
> Lag over time is more informative than a single lag measurement

For backend systems, **p95/p99** are often much more useful than average latency.

Retry Metrics
`How many messages are entering retry?`

DLT Metrics
This is one of the strongest operational signals.
A DLT record should be treated as an operational event:
```
Message failed
      ↓
Retries exhausted
      ↓
DLT
      ↓
Alert / investigation
```
You don't want a DLT silently accumulating thousands of records.


### Spring Boot Actuator
Actuator gives us application-level observability endpoints.
```dependency
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

``` YAML
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
```

now you can expose
```
/actuator/health
/actuator/metrics
/actuator/prometheus
```

### Micrometer
Spring Boot uses **Micrometer** for metrics instrumentation.

```
Application
    │
    ▼
Micrometer
    │
    ├── counters
    ├── timers
    └── gauges
    │
    ▼
Prometheus
    │
    ▼
Grafana
```

> **Kafka transactions solve atomicity inside Kafka workflows.**

> **Transactional Outbox solves atomicity between your database and event publication**


```
Kafka transactions can provide exactly-once semantics for supported Kafka read-process-write workflows, when the application and consumers are configured appropriately.
```


```
Metric              What it tells us

Outbox pending      DB → Kafka publishing backlog
Consumer lag        Kafka → consumer backlog
Processing latency  Consumer processing performance
Retry count         Failure frequency
DLT count           Exhausted failures
Producer failures   Kafka publishing problems
```


Kafka Final Architecture - DevSync
```
                         DEVSync
                            │
                     IssueService
                            │
                 ┌──────────┴──────────┐
                 │                     │
            Issue update          Outbox event
                 │                     │
                 └──────────┬──────────┘
                            │
                         COMMIT
                            │
                            ▼
                    Outbox Publisher
                            │
                      Kafka ACK
                            │
                            ▼
                   devsync.issue-events
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
          Partition 0   Partition 1   Partition 2
              │             │             │
              ▼             ▼             ▼
             C1            C2            C3
                            │
                       processing
                            │
                    ┌───────┴───────┐
                    │               │
                 success          failure
                    │               │
                 commit         retry-1000
                                    │
                                retry-2000
                                    │
                                retry-4000
                                    │
                                    ▼
                                   DLT
                                    │
                               controlled
                                 replay
                                    │
                                    ▼
                              Main topic
```

along side
```
Producer:
  idempotence
  acks=-1

Consumer:
  manual Spring-managed commits
  idempotent processing
  concurrency=3

Reliability:
  transactional outbox
  retry topics
  DLT
  stable eventId

Operations:
  consumer lag
  processing latency
  retry visibility
  DLT visibility
  outbox backlog
  Micrometer
```