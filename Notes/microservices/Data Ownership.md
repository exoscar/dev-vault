Every microservice needs its own database?

Each service should control its authoritative data and the rules for modifying it. Physical database separation can reinforce that boundary, but it is not the definition of ownership. Shared infrastructure can work if access and schema changes remain independently controlled. Direct dependencies on another service's tables undermine that independence.


#### Shared database with shared tables

Example: both services query and modify the `customers` table.
- Highest risk of coupling through table structures and business rules.
- Changes can require coordination across teams.
- Can be acceptable in a monolith or as a deliberate transitional arrangement.

#### Schema per service
Example: `orders.*` and `customers.*` on one PostgreSQL server.
- Supports logical separation and permissions.
- Lower infrastructure cost than separate database instances.
- Requires discipline to prevent cross-schema dependencies.

#### Database per service
Example: separate Order and Customer databases, potentially on the same server or different servers
- Makes data boundaries more explicit.
- Supports independent schema management and potentially independent scaling.
- Cross-service queries and transactions become more difficult.

