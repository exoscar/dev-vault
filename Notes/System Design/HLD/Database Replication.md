## Part 1: PostgreSQL Streaming Replication Architecture

Physical streaming replication works by shipping raw binary changes from the Primary's **Write-Ahead Log (WAL)** over a persistent TCP connection (`walsender` $\rightarrow$ `walreceiver`) to the Replica, which continuously replays those logs against its own data files in read-only mode.

### 1. Infrastructure Setup (`docker-compose.yml`)

Start with two isolated PostgreSQL 16 containers sharing a Docker bridge network.

YAML

```
services:
  postgres-primary:
    image: postgres:16
    container_name: pg-primary
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: rootpassword
      POSTGRES_DB: prototypedb
    volumes:
      - primary_data:/var/lib/postgresql/data

  postgres-replica:
    image: postgres:16
    container_name: pg-replica
    ports:
      - "5433:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: rootpassword
    volumes:
      - replica_data:/var/lib/postgresql/data

volumes:
  primary_data:
  replica_data:
```

### 2. Primary Node Configuration (Step-by-Step)

#### Step A: Create the Replication User & Physical Replication Slot

PowerShell

```
docker exec -it pg-primary psql -U postgres -d prototypedb -c "CREATE ROLE repl_user WITH REPLICATION LOGIN PASSWORD 'repl_password';"
docker exec -it pg-primary psql -U postgres -d prototypedb -c "SELECT pg_create_physical_replication_slot('replica_slot_1');"
```

- **`WITH REPLICATION LOGIN`**: Grants the minimum privilege needed to open a WAL stream without exposing superuser (`postgres`) credentials.
    
- **`pg_create_physical_replication_slot('replica_slot_1')`**: Prevents the Primary from deleting old WAL segments from its disk until the Replica explicitly confirms it has received them. Without a slot, if the Replica goes down for an hour and the Primary recycles its WAL files, the Replica breaks and requires a full wipe and re-clone.
    
- **Caveat to remember:** If you permanently delete a replica, you _must_ drop its replication slot (`SELECT pg_drop_replication_slot('replica_slot_1');`), otherwise the Primary will retain WAL files forever until its disk fills up.
    

#### Step B: Engine Settings (`postgresql.conf`)

Located at `/var/lib/postgresql/data/postgresql.conf`:

PowerShell

```
docker exec -i pg-primary bash -c "cat << 'EOF' >> /var/lib/postgresql/data/postgresql.conf

listen_addresses = '*'
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
hot_standby = on
synchronous_commit = on
EOF"
```

|**Attribute**|**Value**|**Why it is required & How it works**|
|---|---|---|
|`listen_addresses`|`'*'`|Binds Postgres to all network interfaces inside the container so other containers can connect over TCP.|
|`wal_level`|`replica`|Controls how much information is written to the WAL. (`minimal` = crash recovery only; `replica` = adds binary page metadata for physical standby streaming; `logical` = adds table/row metadata for CDC/Debezium).|
|`max_wal_senders`|`10`|Max concurrent `walsender` background processes on the Primary. You need at least 1 per replica + 1 extra for tools like `pg_basebackup`.|
|`max_replication_slots`|`10`|Max number of replication slots the Primary can track concurrently.|
|`hot_standby`|`on`|Crucial on the Replica (and inherited when cloned from Primary). Allows the Standby server to accept read-only `SELECT` queries while simultaneously replaying WAL logs. If `off`, it is a "Warm Standby" (syncs data, but rejects all client queries).|
|`synchronous_commit`|`on`|Ensures the Primary flushes WAL to its _own_ local disk before returning commit success (since `synchronous_standby_names` is empty, replication to the standby remains fast and asynchronous).|

#### Step C: Network Access Control (`pg_hba.conf`)

Located at `/var/lib/postgresql/data/pg_hba.conf`:

PowerShell

```
docker exec -i pg-primary bash -c "printf '\nhost replication repl_user 0.0.0.0/0 scram-sha-256\n' >> /var/lib/postgresql/data/pg_hba.conf"
docker restart pg-primary
```

|**Column**|**Value**|**Meaning**|
|---|---|---|
|**Type**|`host`|Matches TCP/IP network connections (with or without SSL).|
|**Database**|`replication`|A special Postgres keyword—does **not** mean a database named "replication". It authorizes the internal WAL streaming protocol. (The default `all` rule in `pg_hba.conf` explicitly excludes replication connections).|
|**User**|`repl_user`|Restricts streaming permission to our dedicated replication role.|
|**Address**|`0.0.0.0/0`|Allows any IPv4 address (useful inside dynamic Docker bridge subnets).|
|**Method**|`scram-sha-256`|Modern salted challenge-response password hashing (default in Postgres 14+).|

### 3. Replica Node Bootstrapping (`pg_basebackup`)

Before a replica can apply incremental WAL streams, its `/var/lib/postgresql/data` folder must start at the exact same binary LSN (Log Sequence Number) checkpoint as the Primary.

PowerShell

```
docker stop pg-replica

docker compose run --rm -u postgres -e PGPASSWORD=repl_password --entrypoint bash postgres-replica -c "rm -rf /var/lib/postgresql/data/* && pg_basebackup -h pg-primary -p 5432 -U repl_user -D /var/lib/postgresql/data -Fp -Xs -P -R -S replica_slot_1"

docker start pg-replica
```

#### Key `pg_basebackup` Flags & What They Do

- **`-h pg-primary -p 5432 -U repl_user`**: Target Primary host, port, and replication user.
    
- **`-D /var/lib/postgresql/data`**: Target local directory where the binary clone will be written.
    
- **`-Fp` (Format = plain)**: Writes standard PostgreSQL data files and directories directly (as opposed to `-Ft`, which outputs compressed `.tar` archives).
    
- **`-Xs` (Xlog/WAL method = stream)**: Opens a second parallel replication stream _while_ the base backup is copying files, ensuring any writes happening on the Primary during the backup are captured so the backup is consistent upon completion.
    
- **`-P` (Progress)**: Calculates total size first and displays live completion percentages.
    
- **`-S replica_slot_1` (Slot)**: Binds the backup and subsequent streaming configuration to our physical replication slot.
    
- **`-R` (Write recovery config — The Magic Flag)**: Automatically does two things inside `/var/lib/postgresql/data`:
    
    1. Creates an empty file named **`standby.signal`**. When Postgres 12+ boots up and detects this file, it enters Read-Only Standby mode instead of normal Read-Write mode.
        
    2. Writes **`primary_conninfo`** and **`primary_slot_name = 'replica_slot_1'`** into `postgresql.auto.conf`, so the Replica's `walreceiver` process automatically knows where to connect on startup.
        

### 4. Essential Diagnostic SQL Queries (Cheat Sheet)

|**Where to run**|**SQL Query**|**What it tells you**|
|---|---|---|
|**Primary**|`SELECT client_addr, state, sync_state, pg_wal_lsn_diff(sent_lsn, replay_lsn) AS byte_lag, replay_lag FROM pg_stat_replication;`|Confirms `state = streaming`, shows whether mode is `async` or `sync`, and calculates exact byte/time lag.|
|**Primary**|`SELECT slot_name, active, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal FROM pg_replication_slots;`|Checks if the replica slot is `active = t` and how much disk space unconsumed WAL logs are taking.|
|**Primary**|`SELECT * FROM pg_hba_file_rules;`|Inspects parsed `pg_hba.conf` rules in memory and reveals syntax errors without restarting Postgres.|
|**Any Node**|`SELECT pg_is_in_recovery();`|Returns `false` on the Primary (Read-Write) and `true` on the Replica (Standby/Read-Only).|

## Part 2: Spring Boot Read-Write Routing Architecture

To route queries dynamically within a single Spring Boot application without changing your JPA repositories, you need **four components working together**.

### 1. Why Plain `AbstractRoutingDataSource` Fails by Default

In a standard Spring `@Transactional` method call, the execution order is:

1. `TransactionInterceptor` intercepts the method call.
    
2. `JpaTransactionManager.doBegin()` asks the `DataSource` for a JDBC `Connection` **immediately**.
    
3. `AbstractRoutingDataSource.determineCurrentLookupKey()` runs during step 2.
    
4. _After_ the connection is fetched, `TransactionSynchronizationManager.setCurrentTransactionReadOnly(true)` is populated!
    

Because step 2 happens before step 4, `TransactionSynchronizationManager.isCurrentTransactionReadOnly()` always evaluates to `false` when `AbstractRoutingDataSource` is asked for a connection—causing 100% of queries to hit the Primary!

### 2. How `LazyConnectionDataSourceProxy` Solves This

Wrapping `AbstractRoutingDataSource` inside `LazyConnectionDataSourceProxy` intercepts step 2 and returns a lightweight **dynamic proxy** instead of fetching a real physical JDBC connection from HikariCP.

- Spring finishes setting up the transaction context (`readOnly = true` is now properly bound to the thread's `ThreadLocal`).
    
- Only when Hibernate actually prepares the first SQL statement (`connection.prepareStatement(...)`) does the lazy proxy trigger `AbstractRoutingDataSource.determineCurrentLookupKey()`.
    
- At that exact moment, `TransactionSynchronizationManager.isCurrentTransactionReadOnly()` accurately reads `true` and pulls a physical connection from the **Replica** Hikari pool.
    

```
@Transactional(readOnly = true) called
       │
       ▼
JpaTransactionManager starts transaction
       │
       ├──> Asks LazyConnectionDataSourceProxy for Connection
       │    (Returns dummy Proxy immediately; NO physical connection opened yet)
       │
       ├──> Sets ThreadLocal: TransactionSynchronizationManager.setReadOnly(true)
       │
       ▼
Repository executes SQL query (e.g., repository.findAll())
       │
       ├──> Proxy intercepts prepareStatement() -> calls AbstractRoutingDataSource
       ├──> Reads ThreadLocal (isReadOnly == true) -> Selects REPLICA HikariDataSource
       └──> Executes SELECT on Port 5433
```

### 3. Complete Spring Boot Configuration Reference

#### `application.yml`

Note that when defining multiple custom `HikariDataSource` beans via `@ConfigurationProperties`, Hikari expects **`jdbc-url`** instead of `url`.

YAML

```
spring:
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true

  datasource:
    primary:
      jdbc-url: jdbc:postgresql://localhost:5432/prototypedb
      username: postgres
      password: rootpassword
      driver-class-name: org.postgresql.Driver
      pool-name: Primary-HikariPool
      maximum-pool-size: 10
    replica:
      jdbc-url: jdbc:postgresql://localhost:5433/prototypedb
      username: postgres
      password: rootpassword
      driver-class-name: org.postgresql.Driver
      pool-name: Replica-HikariPool
      maximum-pool-size: 10
      read-only: true
```

#### `DataSourceConfig.java` (Complete Routing Setup)

Java

```
package com.example.dbscalingdemo.config;

import com.zaxxer.hikari.HikariDataSource;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.jdbc.DataSourceBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.jdbc.datasource.LazyConnectionDataSourceProxy;
import org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource;
import org.springframework.transaction.support.TransactionSynchronizationManager;

import javax.sql.DataSource;
import java.util.Map;

@Configuration
public class DataSourceConfig {

    public enum DataSourceType {
        PRIMARY, REPLICA
    }

    // 1. Physical Pool for Primary (Port 5432)
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }

    // 2. Physical Pool for Replica (Port 5433)
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.replica")
    public DataSource replicaDataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }

    // 3. Router that inspects ThreadLocal transaction readOnly state
    @Bean
    public DataSource routingDataSource(
            @Qualifier("primaryDataSource") DataSource primary,
            @Qualifier("replicaDataSource") DataSource replica) {

        AbstractRoutingDataSource routingDataSource = new AbstractRoutingDataSource() {
            @Override
            protected Object determineCurrentLookupKey() {
                return TransactionSynchronizationManager.isCurrentTransactionReadOnly()
                        ? DataSourceType.REPLICA
                        : DataSourceType.PRIMARY;
            }
        };

        routingDataSource.setTargetDataSources(Map.of(
                DataSourceType.PRIMARY, primary,
                DataSourceType.REPLICA, replica
        ));
        routingDataSource.setDefaultTargetDataSource(primary);
        return routingDataSource;
    }

    // 4. Primary Bean: Delays connection fetch until after @Transactional context is active
    @Primary
    @Bean
    public DataSource dataSource(@Qualifier("routingDataSource") DataSource routingDataSource) {
        return new LazyConnectionDataSourceProxy(routingDataSource);
    }
}
```

### 4. Production Edge Cases to Keep in Your Notes

1. **Hibernate DDL Initialization (`ddl-auto: update`):** When Spring Boot starts up, Hibernate runs schema inspection and DDL updates outside of a `@Transactional(readOnly = true)` context. Because `routingDataSource.setDefaultTargetDataSource(primary)` points to Primary, `CREATE TABLE` / `ALTER TABLE` statements automatically execute on the Primary and immediately replicate to the Replica via WAL streaming.
    
2. **The Read-After-Write Trap:** If a controller method calls `service.createUser()` (`@Transactional` $\rightarrow$ Primary) and on the very next line calls `service.getUser()` (`@Transactional(readOnly = true)` $\rightarrow$ Replica), there is a tiny window (~1–2 ms) where the replica hasn't replayed the WAL record yet.
    
    - **Fix:** If a business workflow reads data it _just_ mutated in the same HTTP request, keepboth operations inside a single `@Transactional` (read-write) service method so the read also comes from the Primary.
        
3. **Propagation Behavior When Nesting Calls:** If a `@Transactional` (Primary) method calls an internal helper method marked `@Transactional(readOnly = true)`, the outer transaction wins (by default `Propagation.REQUIRED` reuses the existing Primary connection). To force a nested call onto the replica, it would have to be called on a separate bean with `Propagation.REQUIRES_NEW` (rarely needed, as reading from Primary during a write transaction is safer anyway).