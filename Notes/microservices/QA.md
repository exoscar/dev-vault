1. "What is the difference between a monolith and a modular monolith?"
-  Both are deployed as a single application, but a modular monolith deliberately enforces strong boundaries between business modules. The goal is to reduce internal coupling and improve maintainability while retaining the operational simplicity of a monolith

2. Why not directly move to microservices?
- Microservices introduce distributed-system complexity such as network failures, latency, service discovery, observability and data-consistency challenges. A modular monolith can provide strong domain boundaries without paying those costs. If a module later needs independent deployment or scaling, it can potentially be extracted into a service.

3. What is a distributed monolith?
- A distributed monolith is a system split into multiple services or processes that communicate over the network but remain tightly coupled. It retains many of the coupling characteristics of a monolith while introducing the operational and failure complexity of distributed systems.

4. How is it different from microservices?
- The distinction is not simply the number of services. Microservices aim for meaningful service boundaries and a degree of independent deployment, scaling, ownership, and evolution. A distributed monolith has multiple processes but strong runtime, deployment, data, or organizational coupling.
