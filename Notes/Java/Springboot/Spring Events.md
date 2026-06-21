### Spring Event
A Spring event is a mechanism that allows one component to publish an occurence and other interested components to react to it without direct dependencies between them

`Publisher → Event → Listener`

Purpose → reduce coupling between components

Instead of
```
IssueService
    ↓
NotificationService
    ↓
EmailService
    ↓
AuditService
```

Use:
```
IssueService
    ↓
IssueCreatedEvent
    ↓
NotificationListener
AuditListener
EmailListener
```


### ApplicationEventPublisher
`ApplicationEventPublisher` is a Spring interface used to publish events into Spring's event system.

its responsibility is to publish the event. but not processing  event

Example
```
publisher.publishEvent(
    new IssueCreatedEvent(...)
);
```


### @EventListener
`@EventListener` is an annotation used to mark a method as an event consumer.

When an event of the specified type is published, Spring automatically invokes the method.
```
@EventListener
public void handle(IssueCreatedEvent event) {
}
```

### @TransactionalEventListener
`@TransactionalEventListener` is a specialized event listener that executes relative to a transaction lifecycle.

```
@TransactionalEventListener(
    phase = TransactionPhase.AFTER_COMMIT
)
```

##### Transaction Phases
- BEFORE_COMMIT
```
Transaction Open
    ↓
Listener Executes
    ↓
Commit
```

Runs Before commit -- rarely used.

- AFTER_COMMIT
```
Transaction Open
    ↓
Commit
    ↓
Listener Executes
```
Most common -> perfect for email, Notifications, analytics, activity feed and etc

- AFTER_ROLLBACK
```
Transaction Open
    ↓
Rollback
    ↓
Listener Executes
```
Useful for cleanup

- AFTER_COMPLETION
```
Commit OR Rollback
        ↓
Listener Executes
```
Runs regardless of outcome

#### Synchronous Event Handling
Publisher waits for listener completion
```
Publish Event
    ↓
Listener Executes
    ↓
Return Control
```
Default Spring behaviour
implemented using `@EventListener`

#### Asynchronous Event Handling
publisher does not wait.
```
Publish Event
    ↓
Return Immediately

Listener Executes Later
```
Implemented using
```
@Async
@EventListener
```


*** Why synchronous Spring Events become dangerous in production? ***
```
Imagine Devsync

Issue Created
    ↓
Send Email (2 sec)
    ↓
Send Notification (1 sec)
    ↓
Update Search Index (500ms)


API Response time becomes -> 3.5 sec 
even though issue creation itself took only 50ms
```
This is where asynchronous event handling(`@Async`) enters the picture/

***Question: If EmailListener takes 5 seconds to complete, what happens to the HTTP request thread when using a normal synchronous `@EventListener`? What do you think the user experiences?

This would cause unwanted latency --> even though issue created in 50ms. waiting for 3.5s is wasted latency.

***Resource utilization problem***
Spring Boot servers have a limited request thread pool.
Imagine 100 users request for creating issue simultaneously.
Each request take save issue 50ms and Email - 5sec
so each request threads stay occupied for 5 seconds.
casues 
`Thread pool exhaustion`

***Timeout Risk***
example
API Gateway Timeout = 10 sec
Listeners
```
Email = 5 sec
Webhook = 4 sec
Notification = 3 sec
```
Total 12 secs.
actually issue got saved. but client receives timeout error
User may retry and accidentally create duplicates

### Asynchronous Event Handling
Publisher does not wait for listener completion.
```
Publisher
    ↓
Submit task to another thread
    ↓
Return immediately
```

#### Spring Implementaion
Enable async
```
@EnableAsync
@Configuration
public class AsyncConfig {
}
```

Listener:
```
@Async
@EventListener
public void sendEmail(IssueCreatedEvent event) {}
```

New flow
```
Issue Created
    ↓
Publish Event
    ↓
Spring creates async task
    ↓
HTTP Response Returned
    ↓
Email executes in background
```

***Question***
Suppose this async listener throws:
```
throw new RuntimeException("SMTP Failure");
```
after the HTTP response has already been returned.
Who receives that exception?
- The controller?
- The service?
- The user?
- Nobody?

Ans: At this point:
- Controller is already finished
- Service is already finished
- Transaction is already committed
- User already received response
So the exception cannot propagate back. and in Aync, exception occurs in completely different thread.

#### Fire-and-Forget Processing
Many async event listeners are:
```
Fire Event    ↓Continue Processing
```
The publisher does not wait for a result.
The publisher does not receive failures.
This is called **fire-and-forget** execution.

Thats why large systems move to message brokers.
This is one limitation of spring events

Spring Events:
```
In-memory
Fire-and-Forget 
No durability
No Retry
```

Kafka/RabbitMQ later provides
```
Persistence
Retries
Dead Letter Queues
Delivery Guarantees
```

***Question: Why not use Spring Events for everything?****
```
Spring Events are in-process and non-durable.
If the application crashes after publishing the event,
the event is lost.
For cross-service communication or reliable delivery,
Kafka/RabbitMQ are better choices.
```


### Asynchronous Event Listener
An event listener that executes in a separate thread without blocking the publisher thread.

Purpose
```
Reduce request latency
Improve throughput
Avoid blocking HTTP request threads
```
Trafeoff:
```
Exceptions do not propagate back to the publisher.
```


### Event Payload
An **event payload** is the data carried by an event from the publisher to the listeners.
```
public record IssueCreatedEvent(
    Long issueId,
    String title,
    Long assigneeId
) {}
```
payload contains the info listeners need.

Why publishing JPA Entities is bad idea

1. Tight Coupling
2. Lazy loading exceptions
   Example
```
@Entity
class Issue {

    @ManyToOne(fetch = FetchType.LAZY)
    private User assignee;
}
```
Listener
```
@Async
@TransactionalEventListener(
    phase = AFTER_COMMIT
)
public void handle(Issue issue) {

    issue.getAssignee().getEmail();
}
```
Transaction is already over
Persistence context is closed.
Results -> `LazyInitializationException`

3. Entity State Changes  
   Listeners may observe state you didn't intend to publish.
   Events should represent a business fact at a specific moment in time.
4. Large Payloads -- Passing the whole entity can drag a large object graph around.