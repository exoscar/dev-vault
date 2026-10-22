Chatty Communication : when two services constantly communicate with each other synchronously. like for every operation which include many synchronous calls between them.


If every change requires both teams to coordinate, review each others work, and release together, what kind of independence do they really have?

This brings us to conway's law 

 #### 1. The problem it addresses
 ```
                  Engineering
                      |
             +--------+--------+
             |                 |
        Product Team       Order Team
             |                 |
        Product Code       Order Code
 ```
This arrangement can support clear ownership, but only if the architecture permits the teams to work independently.

now imagine this implementation instead
```
Product Team ──┐
               ├── Shared business model
Order Team ────┘
               |
        Coordinated releases
```
a change to shared model might require both teams to coordinate, even if their business responsibilities are otherwise distinct.

The organizational boundary and technical boundary are now in tension

### Conway's Law
The basic idea is that system tend to reflect the communication structures of the organizations that build them.

in practical terms
- Teams that need to coordinate constantly often produce software with strong dependencies between their areas.
- Clear ownership can support clearer interfaces and independent evolution.
- Poorly designed team boundaries can create handoffs and communication bottlenecks.

A team might own several closely related services. Several teams might contribute to a platform. What matters is whether ownership and communication responsibilities are clear.


## What makes a good boundary?
## Boundary evaluation

Use these questions when considering a candidate service split.

1. Business responsibility: Does this area perform a distinct business capability?
    
2. Cohesion: Do its responsibilities naturally belong together?
    
3. Ownership: Is there a clear owner for its data and business rules?
    
4. Coupling: What APIs, events, shared models, and data dependencies would exist?
    
5. Transactions: Would separating it create costly cross-service consistency requirements?
    
6. Independence: Would separate deployment, scaling, or fault isolation provide meaningful benefits?
    
7. Organization: Can the responsible team evolve and operate the boundary without excessive coordination?


|Cost|Why it appears|
|---|---|
|Network communication|Calls that were previously in-process become remote calls.|
|Distributed consistency|Operations involving both services may need coordination.|
|Failure handling|Timeouts, retries, partial failures, and recovery must be addressed.|
|Operational overhead|Independent deployments, monitoring, security, and capacity management.|
|Data coordination|Services must exchange information through explicit contracts.|

