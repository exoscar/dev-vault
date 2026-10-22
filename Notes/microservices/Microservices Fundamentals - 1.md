>The number of services does not determine whether an architecture is microservices. The quality of boundaries and degree of service autonomy matter more than service count.



#### Monolith
A monolith application is an application where all major business capabilites are packaged and deployed as a single application unit.

Why monolith?
it is the simplest way to build and operate an application
- mostly in-process function calls
Do not need to deal with
- Network latency
- HTTP failures
- service discovery
- request serialization
- retries
- distributed tracing


##### Where does the problem start with monolith
assume there is application with users, orders, inventory..............etc
 this may introduce several forms of coupling

- Deployment coupling
	- modify once module but need to deploy entire application
- Scaling couping
	- if only some parts of application need more resources. but in monolith need to scale the entire application
- Organisation coupling
	- many teams working on same large application and increasingly interfere with eachother changes
- Code coupling

There are Two Different Problems
1. Everything is deployed together
2. Everything is tightly coupled

microservices solve the first problem by introducing independently deployable service, but good architecture also requires addressing the second

Trade-offs
Strengths
- Simple deployment
- Simple local development
- In-process communication
- Easier debugging
- Easier transactions
- Lower infrastructure complexity

Weaknesses
- Deployment coupling
- Scaling coupling
- Large codebase can become difficult to maintain
- Modules can become tightly coupled
- Large teams can have coordination problems


### Modular Monolith
A **modular monolith** is a monolithic application that is internally divided into well-defined modules with explicit boundaries.

```
                ONE APPLICATION
                       │
       ┌───────────────┼───────────────┐
       ↓               ↓               ↓
    User Module    Order Module    Payment Module
       │               │               │
       └───────────────┼───────────────┘
                       ↓
                   Database
```
still
```
ONE JVM
ONE APPLICATION
ONE DEPLOYMENT
```
But internally:
```
User Module
     X
     │
     │ cannot freely access
     ↓
Order internals
```

#### Why Does It Exist?
It tries to solve:
>Internal coupling and lack of boundaries, without introducing distributed-system complexity.

#### Trade-offs
##### Advantages
- Stronger maintainability
- Clear business boundaries
- Easier testing of modules
- Easier team ownership
- No network communication between modules
- Simple deployment
- Easier transactions
#### Limitations
The modules still share the same runtime. it is a modular monolith. 

### Distributed Monolith
A **distributed monolith** is a system composed of multiple independently running services, but those services remain **so tightly coupled that they effectively behave like one application**.

>A distributed monolith changes the deployment boundary without adequately changing the coupling boundary.



```
        ┌──────────────┐
        │ Order Service│
        └──────┬───────┘
               │ HTTP
               ↓
        ┌──────────────┐
        │Payment Service│
        └──────┬───────┘
               │ HTTP
               ↓
        ┌──────────────┐
        │ User Service │
        └──────────────┘
```
and still **not have achieved the architectural independence we're looking for from microservices**.
Why Does This Problem Exist?
Usually because teams take a monolith and split it mechanically.
They've changed the **deployment structure** without properly changing the **coupling structure**.

What Problem Does a Distributed Monolith Create?

system has both
- Monolith-style coupling
- Distributed-system complexity

finally system end up like 
```
Monolith coupling
       +
Distributed-system complexity
       =
Distributed Monolith
```


Example
In Spring Boot, you could have:
```
order-service/
payment-service/
user-service/
```
each might have :
`@SpringBootApplication`
and run independently
```
Order Service   → :8081
Payment Service → :8082
User Service    → :8083
```

Communication
```
Order
  ↓
RestClient / WebClient / OpenFeign
  ↓
Payment
```


### Failure Modes
Distributed monoliths have a particularly nasty failure characteristic.
suppose
`A → B → C → D`
if D is slow
then C waits, B waits, and A waits

Eventually
```
D slow
 ↓
C slow
 ↓
B slow
 ↓
A slow
 ↓
entire request path degraded
```

this is one reason distributed systems requires concepts such as:
- Timeouts
- Retries
- Circuit breakers
- Bulkheads
- Asynchronous messaging
- Observability

#### Shared Database problem
Another common characteristic is:
```
Order Service ──┐
Payment Service ├──→ SAME DATABASE
User Service ───┘
```
this can create strong coupling
for example
```
Order Service
    ↓
payment table
```

if order service depends on payment's database schema, then payment cannot freely change its schema

even though application is separated but the data ownership isn't

It doesn't mean **any shared database automatically equals distributed monolith**. Architecture depends on the broader coupling. But shared database schemas and direct cross-service data access are common sources of tight coupling.


#### Distributed Monolith vs Microservices
Distributed monolith
```
Multiple processes
        +
Strong coupling
        +
Limited independence
```

Microservices
```
Multiple processes
        +
Well-defined boundaries
        +
Meaningful independence
```

> A service should have enough autonomy that it can be developed, deployed, scaled, and changed without requiring the entire system to behave like one giant application.


## Microservices
Microservices is an architectural style where an application is composed of independently deployable services organized around meaningful business capabilities. Each service owns a well-defined responsibility and has enough autonomy to be developed, deployed, scaled, and evolved independently, at the cost of distributed-system complexity.

### Microservice
A microservice is an independently deployable application component that owns a **specific business capability**.

```
                         E-Commerce System
                                │
             ┌──────────────────┼──────────────────┐
             ↓                  ↓                  ↓
       User Service       Order Service       Payment Service
             │                  │                  │
             ↓                  ↓                  ↓
          User DB           Order DB           Payment DB
```

Each service:
- has a focused responsibility
- has a defined boundary
- can be deployed independently
- can potentially be scaled independently
- owns its implementation
- ideally owns its data
The entire application is therefore not one executable unit.

### Why Does Microservices Architecture Exist?
Microservices emerged largely because large systems encountered problems that became difficult to handle with a single deployment unit.
These areas may have very different:
- scaling requirements
- release schedules
- technical requirements
- business ownership
- reliability requirements

A single application forces them to share the same deployment boundary.
Microservices allow those boundaries to become explicit.

### Business Boundaries
boundaries should generally correspond to **business capabilities or domains**. rather than techinal layers
```
E-Commerce
│
├── Catalog
├── Ordering
├── Payments
├── Shipping
└── Customer Management
```

This is why microservices are strongly connected with **domain-driven design**.

### Why "Database Per Service" Matters

Database-per-service is a principle, not necessarily "one physical database server per service."

Having a database per service provides greater autonomy but it creates a new problem
>What happens when one business operation needs data owned by several services?

this is where distributed systems become genuinely difficult.

#### What Microservices Give You
1. Independent deployment
2. Independent scaling
3. Fault isolation
	- a failure in one service can potentially be contained.
	-  but it depends on how the system is designed
4. Team autonomy
	-  Different teams can own different services:
	-  Again, this only works if the service boundaries and organizational ownership align.


### What Microservices Cost

Gained:
```
independent deployment
independent scaling
service ownership
```
But introduced
```
network communication
       ↓
latency
       ↓
timeouts
       ↓
partial failures
       ↓
retries
       ↓
data consistency problems
       ↓
distributed transactions
       ↓
observability requirements
```

Now need to thinkg about
```
Logs
Metrics
Tracing
Timeouts
Retries
Circuit breakers
Authentication
Service discovery
Configuration
Deployment
Monitoring
```

### Segment 1 Architecture Model
```
                        APPLICATION
                            │
        ┌───────────────────┴───────────────────┐
        │                                       │
    MONOLITH                              DISTRIBUTED
        │                                      │
        │                                      │
   one deployment                       multiple processes
        │                                      │
        ↓                                      ↓
MODULAR MONOLITH                      DISTRIBUTED MONOLITH
        │                                      │
 strong boundaries                    multiple services
        │                                      │
 same deployment                       tight coupling
        │                                      │
        └──────────────────┬───────────────────┘
                           │
                  meaningful boundaries
                  + reasonable autonomy
                           ↓
                      MICROSERVICES
```

Mental Model
```
			Monolith
			    │
			    │ single deployment
			    ↓
			Modular Monolith
			    │
			    │ strong internal boundaries
			    │ but still one deployment
			    ↓
 ┌──────────────────────────────┐
 │                              │
 │ Bad extraction               │ Good extraction
 ↓                              ↓
Distributed Monolith       Microservices
 │                              │
 │ multiple processes           │ multiple processes
 │ tight coupling               │ meaningful boundaries
 │ distributed complexity       │ useful autonomy
 │                              │
 └──────────────┐      ┌────────┘
                ↓      ↓
             Distributed
              System
             Complexity
```

