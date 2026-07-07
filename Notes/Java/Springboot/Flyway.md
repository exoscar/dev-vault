# Flyway Quick Notes

## What is Flyway?

Flyway is a **database migration tool** used to version, track, and automate database schema changes.

Think of it as **Git for your database structure**.

---

# Why Use Flyway?

Without Flyway:

- Developers manually run SQL scripts
    
- Different environments become inconsistent
    
- Deployment becomes risky
    
- No history of schema changes
    

With Flyway:

- Database changes are versioned
    
- Changes run automatically on startup
    
- Every environment stays synchronized
    
- Migration history is tracked
    

---

# Core Concepts

## Migration

A migration is a SQL file that contains schema changes.

Example:

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL
);
```

---

## Versioned Migrations

Format:

```text
V<version>__<description>.sql
```

Examples:

```text
V1__create_user_table.sql
V2__create_workspace_table.sql
V3__add_indexes.sql
V4__create_notification_table.sql
```

Rules:

- Must start with `V`
    
- Version must be unique
    
- Use double underscore `__`
    
- Executed in ascending order
    

---

## Schema History Table

Flyway automatically creates:

```text
flyway_schema_history
```

Stores:

- Migration version
    
- Description
    
- Checksum
    
- Execution time
    
- Success/Failure status
    

Example:

|Version|Description|Success|
|---|---|---|
|1|create_user_table|TRUE|
|2|create_workspace_table|TRUE|

---

# Migration Lifecycle

Application Startup:

```text
1. Check flyway_schema_history
2. Find pending migrations
3. Execute in order
4. Record execution
5. Start application
```

---

# Common Commands

## migrate

Applies pending migrations.

```text
V1 -> V2 -> V3 -> V4
```

Most frequently used command.

---

## validate

Checks:

- Missing migrations
    
- Modified migrations
    
- Checksum mismatches
    

Prevents accidental changes.

---

## repair

Fixes Flyway metadata issues.

Common use:

```text
Checksum mismatch
```

Recalculates checksums in history table.

---

## baseline

Used when Flyway is introduced into an existing database.

Marks current database state as starting point.

```yaml
spring:
  flyway:
    baseline-on-migrate: true
```

---

# Spring Boot Configuration

```yaml
spring:
  flyway:
    enabled: true
    baseline-on-migrate: true
```

Migration Location:

```text
src/main/resources/db/migration
```

---

# Naming Convention

Recommended:

```text
V1__initial_schema.sql
V2__create_workspace_table.sql
V3__create_project_table.sql
V4__create_issue_table.sql
V5__create_comment_table.sql
```

Avoid:

```text
V1.sql
test.sql
new_table.sql
```

---

# Best Practices

## 1. Never Edit Executed Migrations

Bad:

```text
Modify V2 after it already ran
```

Reason:

```text
Checksum mismatch
```

Instead:

```text
Create V3
Create V4
Create V5
```

---

## 2. One Logical Change Per Migration

Good:

```text
V6__create_attachment_table.sql
```

Bad:

```text
V6__create_everything.sql
```

---

## 3. Keep Migrations Small

Easier to:

- Review
    
- Debug
    
- Rollback manually
    

---

## 4. Use SQL Instead of Auto-DDL

Production preference:

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate
```

Avoid:

```yaml
ddl-auto: update
```

Flyway should manage schema changes.

---

## 5. Add Indexes Through Migrations

Example:

```sql
CREATE INDEX idx_issue_assignee
ON issue(assignee_id);
```

---

# Flyway + JPA Workflow

### Entity Change

```java
private String phoneNumber;
```

### Create Migration

```sql
ALTER TABLE users
ADD COLUMN phone_number VARCHAR(50);
```

### Start Application

```text
Flyway runs migration
Database updated
JPA validates schema
Application starts
```

---

# Common Errors

## Migration Not Running

Check:

```text
Correct file name?
Correct folder?
Flyway enabled?
```

---

## Checksum Mismatch

Cause:

```text
Edited an already executed migration
```

Solution:

```text
Create new migration
OR
Use repair (carefully)
```

---

## Table Not Created

Check:

```text
Migration executed?
flyway_schema_history updated?
SQL syntax valid?
```

Enable logs:

```yaml
logging:
  level:
    org.flywaydb: DEBUG
```

---

# Useful SQL

See migration history:

```sql
SELECT *
FROM flyway_schema_history
ORDER BY installed_rank;
```

---

# DevSync Recommendation

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate

  flyway:
    enabled: true
    baseline-on-migrate: true
```

Folder:

```text
src/main/resources/db/migration
```

Migration Pattern:

```text
V1__init.sql
V2__workspace.sql
V3__project.sql
V4__issue.sql
V5__comment.sql
V6__activity.sql
V7__watchers.sql
V8__labels.sql
V9__notifications.sql
V10__attachments.sql
```

---

# Interview One-Liner

> Flyway is a database version control tool that manages schema changes through ordered migration scripts, ensuring consistent database structure across all environments while maintaining a complete history of changes.