+++
title = "Flyway vs Liquibase - A Pragmatist's Comparison"
date = 2025-10-01
[taxonomies]
tags = ["database", "devops", "spring-boot"]
+++

I've used both. Extensively. On production systems with hundreds of migrations. My preference is clear, but I'll try to be fair. Try.

## The Core Difference

**Flyway** is SQL-first. You write SQL migration files, Flyway executes them in order. That's basically it.

**Liquibase** is abstraction-first. You describe changes in XML, YAML, JSON, or SQL. Liquibase generates the appropriate SQL for your database.

This difference in philosophy drives every other difference between them.

## Flyway: The Simple Path

### How It Works

You create SQL files with a naming convention:

```
db/migration/
  V1__create_orders_table.sql
  V2__add_customer_id_column.sql
  V3__create_index_on_status.sql
```

The `V` prefix means "versioned." The number is the version. The double underscore separates the version from the description. Flyway tracks which versions have been applied in a `flyway_schema_history` table and runs the unapplied ones in order.

```sql
-- V1__create_orders_table.sql
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id VARCHAR(255) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'PENDING',
    total NUMERIC(10,2) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

That's real SQL. You can test it, review it, and run it manually. No abstraction layer, no translation.

### Spring Boot Integration

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
```

Flyway runs automatically at application startup, before Spring initializes the data source for JPA. Migrations are applied, then the application starts. If a migration fails, the application doesn't start. This is the correct behavior - if your schema is wrong, running the application would cause more damage.

### Repeatable Migrations

Flyway supports repeatable migrations (prefix `R__`) that run every time their checksum changes:

```
R__create_views.sql
R__refresh_materialized_views.sql
```

I use these for views, functions, and stored procedures - things that should be recreated from scratch rather than versioned incrementally.

## Liquibase: The Abstract Path

### How It Works

Changes are described in "changelogs" using XML, YAML, JSON, or SQL:

```yaml
databaseChangeLog:
  - changeSet:
      id: 1
      author: goncalo
      changes:
        - createTable:
            tableName: orders
            columns:
              - column:
                  name: id
                  type: uuid
                  constraints:
                    primaryKey: true
              - column:
                  name: status
                  type: varchar(50)
                  defaultValue: PENDING
```

Liquibase translates this into the appropriate SQL for your target database. In theory, the same changelog works on PostgreSQL, MySQL, Oracle, and SQL Server.

### The Abstraction Trade-off

The database abstraction is Liquibase's selling point and its biggest liability.

**When it helps:** If you genuinely need to support multiple database vendors. I've seen this in commercial software that ships to customers running different databases.

**When it hurts:** In every other case. You lose access to database-specific features. You can't use PostgreSQL's `JSONB` operators, MySQL's `FULLTEXT` indexes, or any vendor-specific syntax without dropping down to raw SQL changelogs - at which point you've lost the abstraction benefit.

In my experience, most teams target one database. If you're running PostgreSQL, write PostgreSQL SQL. You'll be more productive, your migrations will be more readable, and you won't fight the abstraction layer.

### Rollback Support

Liquibase can generate rollback SQL for many operations (drop the table that was created, remove the column that was added). Flyway's community edition doesn't support rollback - the paid Teams edition does.

In practice, I've never used automatic rollback in production. Database rollbacks are dangerous because they can lose data. If V5 adds a column and populates it, rolling back V5 drops that column and the data. The correct approach for production is forward-only migrations: if V5 was wrong, write V6 to fix it.

## Spring Boot Integration Comparison

Both integrate seamlessly with Spring Boot.

**Flyway:**
```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
```

**Liquibase:**
```yaml
spring:
  liquibase:
    enabled: true
    change-log: classpath:db/changelog/db.changelog-master.yaml
```

Both run at startup, both track applied changes, both fail the application on migration errors. The developer experience is nearly identical.

## Kubernetes and Continuous Deployment

Running migrations at application startup works for simple deployments. In Kubernetes with rolling updates, it gets complicated.

The problem: during a rolling update, old pods (running the old code) and new pods (running migrations and new code) coexist. If the migration changes the schema in a way that breaks the old code, the old pods crash before the new pods are ready.

### The Solution: Separate Migration from Deployment

Run migrations as a Kubernetes Job or init container before the application starts:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: myapp:latest
          command: ["java", "-cp", "app.jar", "org.flywaydb.core.Flyway", "migrate"]
      restartPolicy: OnFailure
```

Or use a Helm pre-upgrade hook:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    "helm.sh/hook": pre-upgrade
```

### Backward-Compatible Migrations

The golden rule: every migration must be backward-compatible with the previous version of the application.

- **Adding a column?** Give it a default value or make it nullable. Old code ignores it.
- **Removing a column?** First deploy code that doesn't use the column, then remove it in a later migration.
- **Renaming a column?** Add the new column, migrate data, deploy code that uses the new column, then drop the old column. Three steps minimum.

This approach works with both Flyway and Liquibase. The tool doesn't matter. The migration strategy does.

## Schema Validation

Both tools offer schema validation:

**Flyway:** Validates checksums of applied migrations. If you modify an already-applied migration file, Flyway detects the checksum mismatch and refuses to start. This is good - it prevents someone from silently changing history.

**Liquibase:** Similar checksum validation, plus the ability to diff the current database state against the changelog to detect drift. This is useful if someone manually modified the schema outside of the migration tool (it happens, especially in legacy environments).

For teams that use Hibernate's `hibernate.ddl-auto=validate`, the application itself validates that the schema matches the entity model. This catches mismatches regardless of which migration tool you use.

## My Recommendation

**Use Flyway** if:
- You target one database vendor (most teams)
- You prefer writing real SQL
- You want simplicity over features
- Your team has SQL skills

**Use Liquibase** if:
- You genuinely need multi-database support
- You want built-in rollback generation
- Your organization mandates XML-based change management (it happens)
- You need the diff/drift detection features

For most Spring Boot projects targeting PostgreSQL, Flyway is the right choice. It's simpler, the migrations are readable SQL, and there's no abstraction layer to fight. I've used it on projects with 400+ migrations without issues.

Liquibase is more powerful, but that power comes with complexity I've rarely needed. Use power tools when you need power. Use simple tools when you need simplicity.
