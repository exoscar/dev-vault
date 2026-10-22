---
aliases:
- Service Boundaries
- Microservices Boundary Design
status: completed
tags:
- microservices
- architecture
- backend-engineering
title: "Microservices Architecture --- Segment 2: Service Boundaries"
---

# Microservices Architecture --- Segment 2: Service Boundaries

> \[!abstract\] Core question **Where should one service end and another
> begin, and what evidence justifies that boundary?**
>
> A service boundary is an architectural decision, not a direct
> translation of database tables into deployable services.

## 1. Core mental model

A service boundary should be justified by the relationship between
business responsibilities, ownership, coupling, consistency
requirements, and the value of independent operation.

``` text
Business capabilities
        ↓
Responsibilities that belong together
        ↓
Cohesion and domain meaning
        ↓
Data and rule ownership
        ↓
Dependencies and transaction requirements
        ↓
Benefits of independent deployment, scaling, and fault isolation
        ↓
Candidate service boundary
        ↓
Evaluate benefits against distributed-systems costs
```

**Do not use these rules:**

-   One entity = one service.
-   One database table = one service.
-   Different responsibilities automatically require different services.
-   Every service must have its own team.
-   Microservices are automatically more scalable.
-   A good boundary has zero coupling.

Instead, ask: **What does this boundary let us change, operate, scale,
or isolate independently---and what complexity does it introduce?**

## 2. Business capabilities

### What

A **business capability** is something the business needs to be able to
do. It describes a business responsibility, not a class, table, or
deployment unit.

Example e-commerce capabilities:

-   Customer account/profile management
-   Product catalog management
-   Inventory management
-   Order management
-   Payment processing
-   Shipping/fulfillment
-   Reviews
-   Customer communications

A database entity is a thing; a capability describes what the business
does with or through that part of the domain.

  Entity / thing     Possible business capability
  ------------------ ----------------------------------------------------------
  Product            Product catalog management
  Inventory record   Stock management and reservation
  Order              Order lifecycle management
  Payment            Authorization, capture, refunds, reconciliation
  Customer           Account management, customer support, sales relationship

### Why it matters

Starting with tables often produces an arbitrary architecture:

``` text
Customer table → Customer Service
Order table    → Order Service
Product table  → Product Service
```

This assumes that every entity is an independent business
responsibility. It can split closely related behavior and create
unnecessary network communication.

### Practical rule

Identify business capabilities first. Then investigate which
responsibilities naturally belong together. Only after that should you
consider whether a capability or group of capabilities needs an
independent service.

**Classification:** ESSENTIAL.

Related: \[\[Cohesion\]\], \[\[Bounded Context\]\], \[\[Data
Ownership\]\].

## 3. Cohesion

### What

**Cohesion** asks whether the responsibilities inside a module or
service naturally belong together.

High-cohesion Order responsibilities might include:

-   Creating and cancelling orders
-   Managing order items
-   Enforcing order lifecycle rules
-   Calculating order totals according to the order's rules
-   Retrieving order history

Responsibilities that may belong elsewhere include payment
authorization, inventory reservation, customer-profile updates, and
email delivery. Order can coordinate with those capabilities without
owning all of their business rules.

### Why it matters

A service with high cohesion has a clear purpose. Its internal behavior
and rules form a meaningful unit.

``` text
Higher cohesion
Order
├── Create order
├── Cancel order
├── Manage order items
└── Enforce order lifecycle
```

Compare with a module that collects unrelated responsibilities:

``` text
Lower cohesion
Order
├── Create order
├── Update customer profile
├── Send email
├── Resize image
└── Manage warehouse stock
```

Being part of the same user journey does not automatically mean that
every operation belongs to one service.

**Classification:** ESSENTIAL.

## 4. Coupling

### What

**Coupling** describes how much one component depends on another
component's behavior, data, contract, or availability.

Cohesion and coupling are related design concerns, but they are not
opposites:

-   Cohesion: Do these responsibilities belong together?
-   Coupling: How much does this component depend on things outside
    itself?

A healthy candidate boundary generally has high internal cohesion and
manageable external coupling.

### Forms of coupling

  -----------------------------------------------------------------------
  Type                    Example                 Risk
  ----------------------- ----------------------- -----------------------
  Compile-time            Services import a       Model changes propagate
                          shared domain-model     across services
                          library                 

  Runtime / synchronous   Order waits for Product Latency and
                          over HTTP               availability dependency

  Data                    Order directly reads    Ownership and schema
                          Product's database      coupling

  Event / asynchronous    Consumer relies on      Contract, versioning,
                          `OrderCreated`          and delivery concerns
                          semantics               

  Deployment              A change to A always    Services cannot evolve
                          requires deploying B    independently

  Consistency             A, B, and C must commit Distributed transaction
                          atomically              or workflow complexity
  -----------------------------------------------------------------------

### Legitimate versus excessive coupling

A dependency is not automatically bad:

``` text
Order → Inventory: reserve stock
```

This is a meaningful business interaction.

Warning signs include excessive calls for one operation, circular
dependencies, shared database writes, or many services that must all be
available for a basic operation.

**Classification:** ESSENTIAL.

Related: \[\[Synchronous Communication\]\], \[\[Asynchronous
Communication\]\], \[\[Service Boundary Smells\]\].

## 5. Data ownership

### What

A service owns data when it is the authoritative place for changing that
data and enforcing the relevant business rules.

Example:

``` text
Customer Service → owns current customer profile
Product Service  → owns current catalog details
Inventory Service → owns stock and reservations
Order Service    → owns orders and historical order facts
Payment Service  → owns payment records and payment lifecycle
```

Ownership does **not** mean no other service can hold a copy of relevant
information. It means other services should not silently become
alternate authorities for changing the owner's data.

### Why shared database access is risky

``` text
Order Service ──┐
                ├── Same Product tables
Product Service ┘
```

If Order queries or modifies Product tables directly, it depends on
Product's schema and internal storage decisions. Product can no longer
change its schema independently without considering external consumers.

A clearer boundary is:

``` text
Order Service → explicit Product API/event/contract
Product Service → owns Product data
```

### Important rule

**Ownership is about authority to change data, not exclusive permission
to know a copy of it.**

**Classification:** ESSENTIAL.

## 6. Data duplication and historical snapshots

Duplication is not always a design failure. A snapshot can preserve
facts needed by one business capability and reduce runtime dependencies.

### Example: order item

Suppose a customer buys a product for ₹80,000. The catalog price later
changes to ₹75,000.

Order history should still show the amount paid at purchase time.

``` text
OrderItem
---------
productId
productName
unitPriceAtPurchase
quantity
```

Depending on requirements, the snapshot may also include SKU/variant,
discounts, tax details, or other purchase facts.

### Current state versus historical fact

-   `Product.currentPrice` is current catalog state, owned by Product.
-   `OrderItem.unitPriceAtPurchase` is a historical purchase fact, owned
    by Order.
-   `Order.shippingAddress` can preserve the address associated with
    that order even if the customer later changes their profile address.

### Benefits

-   Historical correctness
-   Fewer runtime calls
-   Better availability when the source service is unavailable
-   Clearer ownership of business facts

### Costs

-   Duplicate information
-   Snapshot schema/design decisions
-   Potential staleness if a copy is intended to represent current state
-   Need to distinguish historical data from live data

**Important:** Decide whether each field is a historical snapshot or a
live value. Do not fetch current data automatically when doing so would
rewrite history.

**Classification:** ESSENTIAL.

Related: \[\[Data Ownership\]\], \[\[Eventual Consistency\]\].

## 7. Bounded contexts

### What

A **bounded context** is an area in which a particular business model,
terminology, and rules have a specific meaning.

The same real-world concept does not need one universal model across an
entire company.

Example: `Customer`.

  -----------------------------------------------------------------------
  Context                             What Customer means / what matters
  ----------------------------------- -----------------------------------
  Sales                               Name, sales segment, purchase
                                      history

  Support                             Contact details, communication
                                      preferences, support history

  Finance                             Billing details, tax information,
                                      payment history
  -----------------------------------------------------------------------

All may refer to the same real-world person, but each context has
different questions, rules, and model needs.

``` text
Real-world customer
        |
        +── Sales model
        +── Support model
        └── Finance model
```

### Why one universal model can be harmful

A universal `Customer` model forces unrelated contexts to share fields
and change together. A new Sales-specific field may cause unnecessary
changes in Support and Finance.

Prefer independently owned context-specific models when their meanings
and rules differ.

``` java
class SalesCustomer {
    CustomerId id;
    String name;
    SalesSegment segment;
}

class SupportCustomer {
    CustomerId id;
    ContactInformation contact;
}

class FinanceCustomer {
    CustomerId id;
    BillingInformation billing;
}
```

These are illustrative models, not a requirement to duplicate every
field in every context.

### Shared model warning

A shared `common-domain-model.jar` used by Order, Payment, and Inventory
can create compile-time coupling. Changing the shared business object
can force several services to update and coordinate deployments.

Shared **technical** libraries such as logging, tracing, or security
utilities may be reasonable. They still need versioning discipline, but
they do not inherently force services to share one business model.

**Classification:** Bounded contexts --- ESSENTIAL. Deep DDD theory
beyond practical boundary reasoning --- OPTIONAL at this stage.

Related: \[\[Business Capabilities\]\], \[\[Shared Models\]\], \[\[Data
Ownership\]\].

## 8. Shared models and contracts

Distinguish three things:

1.  **Shared domain model:** Common business objects and rules reused by
    multiple contexts. Can tightly couple their evolution.
2.  **Technical library:** Logging, tracing, authentication utilities,
    or other infrastructure code. Often reasonable to share.
3.  **Integration contract:** An API schema or event schema agreed
    between producer and consumer. Some coupling is necessary, but the
    contract should be explicit and versioned thoughtfully.

A service can be independent at deployment time yet tightly coupled
through shared models or contracts.

**Classification:** ESSENTIAL to recognize the risks; the exact library
strategy is context-dependent.

## 9. Entity relationships across services

In a monolith, ORM relationships can make data access feel local:

``` java
@ManyToOne
private Product product;
```

After separating Product and Order into services, a direct object
relationship across service databases is not the same as a local ORM
relationship.

Order may instead store:

``` text
OrderItem
---------
productId
productNameSnapshot
unitPriceAtPurchase
quantity
```

Use `productId` as a reference when appropriate, and store the
historical facts Order needs. Fetch current Product information only for
use cases that actually need current catalog data.

### Do not assume

-   Every foreign key must become a network call.
-   Every related entity must live in one service.
-   Every cross-service relationship requires a distributed transaction.
-   A direct database join across independently owned service databases
    is a suitable integration strategy.

Alternatives for cross-service queries include batch APIs, a local read
model, or an intentional snapshot.

**Classification:** ESSENTIAL.

## 10. Transaction boundaries

### What

A local database transaction can atomically commit or roll back changes
within its transaction boundary.

In a monolith using one database:

``` sql
BEGIN;

-- Create order
-- Reserve inventory
-- Record required local changes

COMMIT;
```

If these operations share a transaction, atomicity is relatively
straightforward.

With separate services and databases, a single ordinary local ACID
transaction no longer spans all participants.

### Example: Order, Inventory, Payment

Suppose the workflow becomes:

``` text
Order created
    ↓
Inventory reserved
    ↓
Payment fails
```

The system may be left with a pending order and reserved inventory. It
needs a defined recovery policy, such as releasing the reservation and
marking the order rejected or cancelled.

### What this tells us about boundaries

If two responsibilities frequently need to commit atomically, separation
has a consistency cost and deserves scrutiny. It does **not**
automatically prove they must be one service.

Compare:

-   How cohesive the responsibilities are
-   Whether separate ownership/deployment/scaling provides real value
-   Whether intermediate states are acceptable
-   Whether compensation or eventual consistency meets the business
    requirement
-   Whether the workflow would become operationally fragile

### Critical distinction

A cross-service business workflow does not automatically require a
distributed ACID transaction. A workflow may use local transactions,
explicit states, retries, and compensating actions.

If frequent distributed transactions are necessary for basic
correctness, reconsider whether the boundary is appropriate.

**Classification:** ESSENTIAL.

Related: \[\[Saga Pattern\]\], \[\[Eventual Consistency\]\], \[\[Data
Ownership\]\].

## 11. Synchronous dependencies

### What

With synchronous communication, a caller waits for a response.

``` text
Client → Order → Inventory → response
```

A synchronous dependency can introduce:

-   Network latency
-   Timeouts and connection failures
-   Retry and idempotency concerns
-   Availability dependencies
-   Serialization and API compatibility requirements

If Order calls Customer, Product, Inventory, and Payment on every
request, the overall request can become fragile even when each
individual dependency seems reasonable.

### Latency

If sequential calls take 50 ms, 40 ms, and 60 ms, their approximate
combined waiting time is 150 ms, before other overheads.

Parallel calls can reduce elapsed time but complicate partial failures,
cancellation, and aggregation.

### Reduce chatty interactions

Bad pattern: one request per item.

``` text
Order → Product(1)
Order → Product(2)
Order → Product(3)
...
```

Better, where appropriate:

``` text
Order → Product batch API
```

A batch operation can reduce network round trips without merging
services.

**Classification:** ESSENTIAL.

## 12. Asynchronous dependencies

### What

With asynchronous messaging, a producer emits an event and a consumer
processes it separately.

For example, if the requirement is "send an email after an order is
successfully created":

``` text
Order Service
    |
    | OrderCreated
    v
Message Broker
    |
    v
Notification Service
    |
    v
Email provider
```

Order does not need to wait for Notification to send the email.
Notification can recover after temporary unavailability, assuming the
event is durably retained and delivery/retry behavior is designed
correctly.

### What changes

Asynchronous communication reduces immediate runtime coupling, but
introduces other concerns:

-   Event contract and schema evolution
-   Event delivery and retries
-   Duplicate messages and idempotent consumers
-   Ordering requirements
-   Eventual consistency
-   Monitoring and recovery

### Outbox pattern

A typical outbox design commits the business change and an event record
in one **local database transaction**:

``` text
BEGIN DB TRANSACTION
  Create Order
  Insert OrderCreated into Outbox
COMMIT
```

A separate publisher sends the outbox event to the broker and retries
failures.

The application is not normally committing directly to both its database
and Kafka in one ordinary atomic transaction.

**Classification:** Sync vs async reasoning --- ESSENTIAL. Detailed
broker and outbox implementation --- USEFUL, to be studied when
appropriate.

Related: \[[Outbox Pattern](#outbox-pattern)\], \[\[Eventual
Consistency\]\], \[\[Saga Pattern\]\].

## 13. Chatty services and other boundary smells

A **boundary smell** is evidence that the current split or its
integration design may be costly. It is a reason to investigate, not
proof that services must be merged.

### Smell A: Chatty services

``` text
Order → Product × 100
```

Potential problems: accumulated latency, more failure points, and
complex error handling.

Possible remedies:

-   Batch API
-   Local read model
-   Snapshot of the facts needed by Order
-   Reassess whether every remote call is necessary

Do not automatically merge Product and Order.

### Smell B: Shared database

``` text
Order Service ──┐
                ├── Same tables
Product Service ┘
```

Potential problems: schema coupling, unclear authority, and unsafe
independent changes.

Prefer explicit contracts and clear ownership.

### Smell C: Distributed transaction dependency

``` text
A + B + C must always commit or roll back together
```

Potential problem: the architecture has split a consistency boundary
across services.

Investigate whether the split is justified and whether the business can
tolerate explicit workflow states and compensation.

### Smell D: Shared deployment

If changing A always requires deploying B, the services may not be
independently evolvable.

Possible cause: shared domain model, incompatible contracts, or a
workflow that assumes synchronized releases.

### Smell E: Excessive synchronous chain

``` text
A → B → C → D → E
```

A failure or delay deep in the chain can affect the original request.
Investigate whether each dependency is necessary and whether synchronous
communication is appropriate.

**Classification:** ESSENTIAL.

## 14. Organizational boundaries and Conway's Law

Conway's Law describes the tendency for a system's design to reflect the
communication structures of the organization building it.

If Product and Order teams must coordinate on every shared-model change,
their architecture may reflect that coupling.

Clear ownership and well-designed contracts can help teams evolve
independently.

But:

-   One team does not have to equal one service.
-   One team can own multiple related services.
-   Several teams can contribute to a shared platform.
-   Team boundaries do not automatically justify technical boundaries.

The objective is to align ownership and communication with the system's
responsibilities without creating excessive coordination.

**Classification:** USEFUL.

## 15. Independent deployment, scaling, and fault isolation

These are possible benefits of a service boundary, not automatic
outcomes.

### Independent deployment

Useful when a component needs to change and release without coordinating
with unrelated components.

Evidence: different release cadence, clear contracts, and a real cost to
coordinated deployments.

### Independent scaling

Useful when workloads differ enough that scaling them together wastes
resources or cannot meet performance targets.

Example:

``` text
Product:   500 requests/sec
Inventory: 10,000 requests/sec
```

This is evidence to investigate. Also measure CPU, memory, database
bottlenecks, latency, and cost. Request rate alone does not prove a
separate service is necessary.

### Fault isolation

Separate deployment can limit the effect of failures, but only if
resource limits, dependencies, infrastructure, and failure handling
support isolation.

A service that synchronously requires five other services for every
request may not be operationally independent.

**Classification:** USEFUL boundary evidence; all three are important
when relevant to the workload.

## 16. Modular monolith versus microservices

A modular monolith can provide clear internal boundaries without
introducing network calls and distributed transactions.

``` text
One deployable application
├── Product module
├── Inventory module
├── Order module
└── Payment module
```

This is often a good starting point when:

-   One team owns the application.
-   Independent scaling has not been demonstrated as necessary.
-   Responsibilities can be isolated internally.
-   Cross-module consistency is important.
-   The operational cost of distributed services would outweigh current
    benefits.

A modular monolith is not a failed microservices architecture. It is a
legitimate architectural choice and can preserve future extraction
options if module boundaries and data access rules are respected.

**Classification:** ESSENTIAL.

Related: \[\[Modular Monolith\]\], \[\[Microservices Trade-offs\]\].

## 17. Boundary decision checklist

Before proposing a service split, answer:

-   [ ] What business capability or cohesive responsibility does this
    area represent?
-   [ ] Which responsibilities naturally belong together?
-   [ ] Who owns its business rules and authoritative data?
-   [ ] Are its model and language specific to its context?
-   [ ] What API, event, database, and deployment dependencies will
    exist?
-   [ ] Does the workflow require atomic transactions across the
    proposed boundary?
-   [ ] How will partial failures and recovery work?
-   [ ] Are synchronous calls necessary, or would a batch API, snapshot,
    or event be more appropriate?
-   [ ] Is independent deployment, scaling, or fault isolation a
    demonstrated need?
-   [ ] Can the responsible team operate and evolve the boundary without
    excessive coordination?
-   [ ] Do the benefits justify the additional distributed-systems
    complexity?
-   [ ] Would a modular monolith meet the requirements more simply?

This is a reasoning checklist, not a numerical score. One severe
consistency or coupling problem can outweigh several smaller benefits.

## 18. Interview perspective

**Question:** How do you decide where one microservice should end and
another should begin?

### Shallow answer

"I split services by business entities, such as Customer, Order, and
Product."

Problem: entities do not automatically define cohesive, independently
operable boundaries.

### Good answer

"I identify business capabilities, group cohesive responsibilities, and
evaluate coupling, data ownership, transactions, and independent
deployment or scaling needs."

### Strong engineering answer

"I start from business capabilities and domain responsibilities, then
identify ownership and consistency boundaries. I evaluate the
dependencies a split creates---API and event contracts, shared data,
synchronous failure paths, and cross-service transactions. I compare
those costs with measurable benefits such as independent scaling,
deployment, or fault isolation. I may keep the responsibilities in a
modular monolith until there is evidence that separation is worth the
operational cost."

## 19. Key takeaways

1.  A service should represent a cohesive business responsibility or
    bounded context, not simply a database entity.
2.  High cohesion and manageable coupling are more useful goals than
    zero coupling.
3.  Data ownership means authority over changes, not a ban on copying
    relevant information.
4.  Historical snapshots can preserve business facts and reduce runtime
    dependencies.
5.  Different contexts may need different models for the same real-world
    concept.
6.  A legitimate business dependency can still have a badly designed
    communication pattern.
7.  Frequent cross-service atomic transactions are a warning to
    reconsider the boundary.
8.  Asynchronous messaging reduces immediate runtime dependency but adds
    event-delivery and consistency concerns.
9.  Separate teams, databases, deployments, and scaling needs provide
    evidence---not automatic proof---for a split.
10. A modular monolith is often preferable until independent operation
    creates enough value to justify distributed complexity.

## 20. Self-assessment

You should be able to explain, without relying on memorized definitions:

-   Why Product and Inventory might or might not be separate services.
-   Why Order can store a product snapshot without taking ownership of
    the current catalog.
-   Why a shared domain-model library can undermine independent
    evolution.
-   Why a batch API can solve chatty communication without merging
    services.
-   Why a workflow spanning several services does not automatically
    require a distributed ACID transaction.
-   Why a modular monolith may be the better choice for a small team and
    an unproven scaling requirement.

### Related Obsidian notes

Create separate linked notes as you deepen each concept:

-   \[\[Business Capabilities\]\]
-   \[\[Cohesion\]\]
-   \[\[Coupling\]\]
-   \[\[Bounded Context\]\]
-   \[\[Data Ownership\]\]
-   \[\[Shared Models\]\]
-   \[\[Transaction Boundaries\]\]
-   \[\[Synchronous Communication\]\]
-   \[\[Asynchronous Communication\]\]
-   \[[Outbox Pattern](#outbox-pattern)\]
-   \[\[Saga Pattern\]\]
-   \[\[Eventual Consistency\]\]
-   \[\[Service Boundary Smells\]\]
-   \[\[Conway's Law\]\]
-   \[\[Modular Monolith\]\]
-   \[\[Microservices Trade-offs\]\]
