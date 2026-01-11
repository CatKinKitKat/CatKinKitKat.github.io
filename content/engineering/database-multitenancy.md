+++
title = "Database Multitenancy: Three Strategies and Their Real Trade-offs"
date = 2025-10-15
[taxonomies]
tags = ["database", "hibernate", "architecture"]
+++

Multitenancy is one of those architectural decisions that seems simple in a whiteboard session and becomes profoundly complicated in production. I've implemented two of the three main strategies. Here's what I've learned about each, including the things nobody mentions until it's too late.

## The Three Strategies

### 1. Separate Database Per Tenant

Each tenant gets their own database instance (or cluster). Complete physical isolation.

```
tenant-a.mydb.example.com -> database_tenant_a
tenant-b.mydb.example.com -> database_tenant_b
```

**Advantages:**
- Maximum isolation. One tenant's data can never leak to another.
- Independent scaling. A large tenant can have a bigger database.
- Independent backup and restore. You can restore one tenant without affecting others.
- Simple data export. Give the tenant their entire database.

**Disadvantages:**
- Connection pool per tenant. With 100 tenants and 10 connections each, you need 1000 connections.
- Cross-tenant queries are impossible (or require federated queries, which are painful).
- Migrations run N times, once per database.
- Operational overhead scales linearly with tenant count.

This is the right choice when you have a small number of high-value tenants with strict data isolation requirements (healthcare, finance, government). It's the wrong choice when you have 10,000 tenants because the operational overhead becomes unmanageable.

### 2. Schema Per Tenant

All tenants share one database instance, but each gets their own schema (namespace).

```sql
CREATE SCHEMA tenant_a;
CREATE SCHEMA tenant_b;

-- tenant_a.orders, tenant_a.customers
-- tenant_b.orders, tenant_b.customers
```

**Advantages:**
- Good isolation (schema-level security, separate tables).
- One database to operate (one connection pool, one backup, one instance).
- Cross-tenant queries possible (just reference the schema).
- Easier than separate databases for migration management.

**Disadvantages:**
- Schema sprawl. 1000 tenants = 1000 copies of every table. `pg_catalog` gets large.
- Migrations still run N times (ALTER TABLE in each schema).
- Connection pool is shared. One tenant's expensive query affects all tenants.
- PostgreSQL handles many schemas reasonably well. MySQL does not (MySQL schemas are databases).

This is my preferred strategy for 10-500 tenants. It balances isolation with operational simplicity.

### 3. Row-Level Tenancy (Shared Schema)

All tenants share the same tables. A `tenant_id` column discriminates the data.

```sql
CREATE TABLE orders (
    id UUID PRIMARY KEY,
    tenant_id VARCHAR(50) NOT NULL,
    customer_id VARCHAR(255),
    total NUMERIC(10,2),
    - ...
);

CREATE INDEX idx_orders_tenant ON orders (tenant_id);
```

**Advantages:**
- Simplest to operate. One database, one schema, one set of tables.
- Migrations run once.
- Connection pool is shared and simple to configure.
- Scales to thousands of tenants without infrastructure complexity.

**Disadvantages:**
- Data isolation is application-enforced. One bug and tenant A sees tenant B's data.
- Every query must include `WHERE tenant_id = ?`. Miss one and you have a data leak.
- Indexes must include `tenant_id` for performance. Table sizes grow with all tenants combined.
- Backup and restore of a single tenant requires filtering, not simple database-level operations.

This is the right choice for SaaS applications with many tenants (1000+) where the data sensitivity is moderate and you trust your application layer.

## Hibernate Multitenancy Support

Hibernate has built-in support for multitenancy via the `MultiTenantConnectionProvider` and `CurrentTenantIdentifierResolver` interfaces.

### Identifying the Tenant

```java
@Component
public class TenantIdentifierResolver implements CurrentTenantIdentifierResolver<String> {
    @Override
    public String resolveCurrentTenantIdentifier() {
        String tenant = TenantContext.getCurrentTenant();
        return tenant != null ? tenant : "default";
    }

    @Override
    public boolean validateExistingCurrentSessions() {
        return true;
    }
}
```

The tenant typically comes from a request header, JWT claim, or subdomain. Store it in a ThreadLocal at the start of the request:

```java
@Component
public class TenantFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain chain) throws ServletException, IOException {
        String tenant = request.getHeader("X-Tenant-Id");
        if (tenant == null) {
            response.sendError(HttpServletResponse.SC_BAD_REQUEST, "Missing tenant");
            return;
        }
        TenantContext.setCurrentTenant(tenant);
        try {
            chain.doFilter(request, response);
        } finally {
            TenantContext.clear();
        }
    }
}
```

### Schema-Per-Tenant Connection Provider

```java
@Component
public class SchemaMultiTenantConnectionProvider implements MultiTenantConnectionProvider<String> {
    private final DataSource dataSource;

    @Override
    public Connection getConnection(String tenantIdentifier) throws SQLException {
        Connection connection = dataSource.getConnection();
        connection.setSchema(tenantIdentifier);
        return connection;
    }

    @Override
    public void releaseConnection(String tenantIdentifier, Connection connection) throws SQLException {
        connection.setSchema("public"); // Reset to default
        connection.close();
    }
}
```

Configure Hibernate:

```yaml
spring:
  jpa:
    properties:
      hibernate:
        multiTenancy: SCHEMA
```

### Row-Level Tenancy with Hibernate Filters

For shared-schema multitenancy, Hibernate filters automatically add the tenant predicate to every query:

```java
@Entity
@FilterDef(name = "tenantFilter", parameters = @ParamDef(name = "tenantId", type = String.class))
@Filter(name = "tenantFilter", condition = "tenant_id = :tenantId")
public class Order {
    @Column(name = "tenant_id", nullable = false)
    private String tenantId;
}
```

Enable the filter at the start of each request:

```java
Session session = entityManager.unwrap(Session.class);
session.enableFilter("tenantFilter").setParameter("tenantId", currentTenant);
```

The filter adds `AND tenant_id = ?` to every query that touches the Order table. But, and this is the critical caveat, filters only apply to queries, not to `find()` by ID. If someone calls `entityManager.find(Order.class, someId)`, the filter is not applied. You need additional safeguards.

## The Missing Safeguards

### Row-Level Security (PostgreSQL)

For shared-schema tenancy, PostgreSQL's Row-Level Security (RLS) is the nuclear option:

```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.current_tenant'));
```

Set the tenant at the connection level:

```java
connection.createStatement().execute(
    "SET app.current_tenant = '" + tenantId + "'"
);
```

Now the database itself enforces tenant isolation. Even if your application code forgets the WHERE clause, the database won't return another tenant's data. This is defense in depth, and I strongly recommend it for row-level tenancy.

### Integration Tests for Isolation

Write tests that verify tenant isolation:

```java
@Test
void shouldNotLeakDataBetweenTenants() {
    TenantContext.setCurrentTenant("tenant-a");
    orderService.createOrder(new CreateOrderCommand(...));

    TenantContext.setCurrentTenant("tenant-b");
    List<Order> orders = orderService.findAll();

    assertThat(orders).isEmpty(); // tenant-b should see nothing
}
```

Run these tests for every entity, every repository method. It's the only way to catch missing tenant filters.

## Operational Trade-offs

| Concern | Separate DB | Schema-per-tenant | Row-level |
|---|---|---|---|
| Isolation | Physical | Logical | Application |
| Migration complexity | O(N) databases | O(N) schemas | O(1) |
| Connection pooling | N pools | 1 pool | 1 pool |
| Cross-tenant queries | Impossible | Possible | Easy |
| Backup/restore per tenant | Simple | Moderate | Complex |
| Max practical tenants | ~50 | ~500 | Unlimited |

## My Recommendation

For most SaaS applications: start with row-level tenancy plus PostgreSQL RLS. It's the simplest to operate, scales to thousands of tenants, and RLS provides database-level isolation guarantees.

If you have strict compliance requirements (tenant data must be physically separate), go with schema-per-tenant. It's a good middle ground.

If you're building for a small number of enterprise customers who demand complete isolation, separate databases. Accept the operational cost.

And whatever you choose: decide early. Retrofitting multitenancy onto an existing application is one of the most painful architectural migrations I've ever done. The tenant identifier touches every table, every query, every test. Get it right from the start.
