+++
title = "Connection Pooling: The Silent Killer"
date = 2026-03-08
[taxonomies]
tags = ["database", "java", "spring-boot", "performance"]
+++

Every Java developer uses connection pools. Very few understand them. And the ones who don't understand them end up debugging "connection timeout" errors in production at the worst possible time.

## Why It Matters

Opening a database connection is expensive. TCP handshake, TLS negotiation (if you're doing it right), authentication, session setup - it adds 20-50ms per connection on a good day. If your service handles 100 requests per second and each one opens a fresh connection, you're spending 2-5 seconds per second just on connection overhead. That math doesn't work.

Connection pools keep a set of connections open and ready. Your code borrows one, uses it, returns it. The overhead drops to near zero.

## HikariCP: The Default (For Good Reason)

Spring Boot uses HikariCP by default, and it's the right choice. It's fast, lightweight, and has sane defaults. But those defaults might not be sane for your workload.

The key settings:

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
```

### `maximum-pool-size`

This is the one that matters most, and it's the one most people get wrong. The instinct is "more connections = more throughput." It's the opposite.

PostgreSQL's sweet spot for a given machine is roughly:

```
connections = (2 * number_of_cores) + number_of_disks
```

For a typical 4-core database server with SSDs, that's around 10. Not 50. Not 100. Ten.

Every connection beyond the optimal point adds context-switching overhead to the database. I've seen services go from 500ms average response time to 50ms by *reducing* the pool size from 50 to 10.

### `connection-timeout`

How long a thread waits to get a connection from the pool before giving up. The default is 30 seconds, which is usually too long. If your pool is exhausted for 30 seconds, something is fundamentally wrong, and making requests wait that long just creates a cascading failure.

I set this to 5 seconds. If a connection isn't available in 5 seconds, fail fast and let the caller handle it.

### `max-lifetime`

How long a connection lives before being recycled. Must be shorter than the database's connection timeout (PostgreSQL's `idle_in_transaction_session_timeout` or the firewall/load balancer timeout, whichever is shorter). If a connection exceeds the database's timeout and the pool doesn't know, the next query on that connection fails.

30 minutes is a reasonable default. If you're behind an Azure load balancer, their idle timeout is 4 minutes by default - set `max-lifetime` to 3 minutes or you'll get "connection reset" errors that take forever to diagnose. Ask me how I know.

## The Connection Leak

The most common pool problem isn't configuration - it's leaks. A connection gets borrowed and never returned. The pool slowly drains until `connection-timeout` fires on every request.

The usual suspect:

```java
// DON'T DO THIS
Connection conn = dataSource.getConnection();
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery("SELECT ...");
// exception here = connection never returned
process(rs);
conn.close(); // never reached
```

Use try-with-resources:

```java
try (Connection conn = dataSource.getConnection();
     PreparedStatement stmt = conn.prepareStatement("SELECT ...")) {
    ResultSet rs = stmt.executeQuery();
    process(rs);
} // connection automatically returned to pool
```

If you're using Spring and JPA (which you should be), Spring manages this for you. But if you're doing raw JDBC calls alongside JPA - and in legacy migrations, you will be - watch for leaks.

HikariCP has a leak detection setting:

```yaml
spring:
  datasource:
    hikari:
      leak-detection-threshold: 60000 # log a warning if connection held > 60s
```

This has saved me hours of debugging. Turn it on in all environments.

## Monitoring

You can't tune what you can't see. Expose HikariCP metrics:

```yaml
management:
  metrics:
    enable:
      hikaricp: true
```

The metrics that matter:
- `hikaricp.connections.active` - how many connections are in use right now
- `hikaricp.connections.pending` - how many threads are waiting for a connection (this should be 0)
- `hikaricp.connections.timeout` - how many times the pool failed to provide a connection

If `pending` is regularly above 0, your pool is too small or your queries are too slow. If `timeout` is above 0, you have an active problem.

## The Lesson

Connection pool tuning isn't exciting. But a misconfigured pool is the kind of problem that manifests as "the app is slow sometimes" with no obvious cause. It wastes hours of investigation because the symptoms look like everything except a connection pool issue.

Set the pool size small. Set the timeouts tight. Turn on leak detection. Monitor the metrics. It's twenty minutes of configuration that saves you from a category of production incidents entirely.
