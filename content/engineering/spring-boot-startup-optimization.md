+++
title = "Spring Boot Startup Optimization: From 30 Seconds to Sub-Second"
date = 2026-02-12
[taxonomies]
tags = ["spring-boot", "performance", "java"]
+++

Spring Boot applications are not known for fast startups. A typical enterprise application with JPA, security, Kafka, and a handful of other starters takes 15-30 seconds to start. In a Kubernetes world where pods scale up and down constantly, that's an eternity.

I've spent an unreasonable amount of time optimizing Spring Boot startup. Here's what actually works, ordered from "easy wins" to "fundamental changes."

## Measure First

Before optimizing anything, measure your baseline. Spring Boot provides startup timing since version 2.4:

```java
@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(MyApplication.class);
        app.setApplicationStartup(new BufferingApplicationStartup(2048));
        app.run(args);
    }
}
```

Then hit `/actuator/startup` to see exactly where time is spent. In my experience, the top offenders are usually:
- Component scanning (especially with large classpaths)
- JPA entity scanning and Hibernate initialization
- Embedded server startup
- Kafka consumer initialization
- Connection pool initialization

## Lazy Initialization

The cheapest optimization. Instead of creating all beans at startup, create them on first use.

```yaml
spring:
  main:
    lazy-initialization: true
```

This can cut startup time dramatically - I've seen 40-60% reductions. But there's a significant trade-off: the first request that triggers bean creation will be slow. And configuration errors that would have been caught at startup now surface at runtime.

My compromise: use lazy initialization in development, not in production. In production, I want to fail fast.

```yaml
# application-local.yml
spring:
  main:
    lazy-initialization: true
```

For more granular control, `@Lazy` on individual beans:

```java
@Bean
@Lazy
public ReportGenerator reportGenerator() {
    // Only created when first needed
    return new ReportGenerator(heavyDependency);
}
```

## AppCDS (Application Class Data Sharing)

AppCDS pre-processes your application classes into a shared archive that the JVM can memory-map at startup, skipping the class loading and verification overhead.

```bash
# Step 1: Create the class list
java -XX:DumpLoadedClassList=classes.lst -jar myapp.jar

# Step 2: Create the archive
java -Xshare:dump -XX:SharedClassListFile=classes.lst \
     -XX:SharedArchiveFile=app-cds.jsa -jar myapp.jar

# Step 3: Use the archive
java -Xshare:on -XX:SharedArchiveFile=app-cds.jsa -jar myapp.jar
```

In practice, AppCDS shaves 1-3 seconds off startup for typical Spring Boot applications. Not transformative on its own, but it's free performance with no behavioral changes. Spring Boot 3.3+ makes this easier with built-in support:

```bash
# Spring Boot 3.3+
java -Dspring.context.exit=onRefresh -jar myapp.jar
java -XX:SharedArchiveFile=app-cds.jsa -jar myapp.jar
```

## CRaC (Coordinated Restore at Checkpoint)

CRaC is the most exciting thing happening in Java startup optimization right now. The idea: start your application, warm it up, take a snapshot of the entire JVM state (memory, threads, class metadata), and restore from that snapshot on subsequent starts.

```java
@SpringBootApplication
public class MyApplication implements Resource {

    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }

    @Override
    public void beforeCheckpoint(Context<? extends Resource> context) {
        // Close connections, release resources before snapshot
        log.info("Preparing for checkpoint...");
    }

    @Override
    public void afterRestore(Context<? extends Resource> context) {
        // Reconnect, refresh state after restore
        log.info("Restored from checkpoint, reconnecting...");
    }
}
```

```bash
# Start and checkpoint
java -XX:CRaCCheckpointTo=checkpoint-dir -jar myapp.jar
# (trigger checkpoint via jcmd or API)

# Restore (sub-second startup)
java -XX:CRaCRestoreFrom=checkpoint-dir
```

Restore times are typically under 100ms. That's not a typo. The JVM doesn't bootstrap. It just restores state from disk.

The catch: you need to handle resource cleanup properly. Open file handles, network connections, and random number generators all need to be refreshed after restore. Spring Boot 3.2+ has built-in CRaC support that handles most of this automatically, but custom resources (database connection pools, Kafka consumers) need attention.

## Project Leyden

Project Leyden is the JVM team's long-term answer to startup time. It's an umbrella for ahead-of-time compilation features that will eventually be integrated into standard OpenJDK.

The key insight: a lot of work the JVM does at startup is the same every time. Class loading, verification, annotation processing, initialization - all deterministic, all repeatable. Leyden moves this work to build time.

As of early 2026, Leyden's "premain" optimization is available in early-access JDK builds. It records startup decisions (which classes to load, which methods to compile) and replays them on subsequent starts. Early benchmarks show 30-50% startup improvement without code changes.

This is complementary to CRaC. CRaC snapshots the entire application state. Leyden optimizes the JVM's own startup overhead. Combining both should yield the fastest possible startup times.

## GraalVM Native Image

The nuclear option. Compile your entire Spring Boot application to a native binary. No JVM startup, no class loading, no JIT compilation.

```bash
mvn -Pnative native:compile
./target/myapp
# Started in 0.08 seconds
```

The numbers are impressive. 80ms startup. 50MB RSS memory. It feels like a different technology entirely.

The trade-offs are real:

**Build time.** Native compilation takes 3-10 minutes and needs significant memory (8GB+ recommended). Your CI pipeline will feel this.

**Reflection limitations.** GraalVM native image does ahead-of-time compilation, which means it needs to know all reflective access at build time. Spring Boot 3.x generates the necessary metadata automatically for most cases, but some libraries still break.

**No JIT optimization.** The JIT compiler makes Java fast at runtime by optimizing hot paths. Native images use ahead-of-time compilation, which can't specialize for your workload. For long-running services with predictable hot paths, JIT-compiled Java often outperforms native images in throughput.

**Debugging.** Stack traces are less informative. Profiling tools have limited support. When something goes wrong, it's harder to diagnose.

My guidance: use native images for serverless functions, CLI tools, and short-lived processes where startup time is critical. For long-running microservices, the JIT compiler's throughput advantage usually outweighs the startup benefit. Consider CRaC instead: you get fast startup without losing JIT optimization.

## Component Scanning Optimization

A few things that collectively shave seconds off startup:

```yaml
spring:
  jpa:
    defer-datasource-initialization: true
    open-in-view: false
    properties:
      hibernate:
        boot:
          allow_jdbc_metadata_access: false
        temp:
          use_jdbc_metadata_defaults: false

  jmx:
    enabled: false

logging:
  level:
    org.hibernate.SQL: WARN
```

Disabling JMX saves about 200ms. Deferring datasource initialization lets other beans initialize in parallel. Preventing Hibernate from querying JDBC metadata avoids a blocking database call during startup.

Also: check your component scan. If your base package is too broad, Spring scans classes it doesn't need to.

```java
// Too broad: scans everything
@SpringBootApplication
public class MyApplication { }

// Better: explicit base packages
@SpringBootApplication(scanBasePackages = "com.example.myapp")
public class MyApplication { }
```

## My Startup Optimization Playbook

1. **Measure** with Actuator startup endpoint. Know where time goes.
2. **AppCDS** for free 1-3 second savings.
3. **Trim unnecessary starters.** Every dependency adds to component scanning and auto-configuration.
4. **JPA metadata** settings to avoid blocking database calls.
5. **Disable JMX** if you're not using it.
6. **CRaC** for sub-second startup in production without sacrificing throughput.
7. **GraalVM native** only if you genuinely need it (serverless, CLI).

The goal isn't to chase the lowest possible number. It's to make your startup time reasonable for your deployment model. If you're deploying once a day, 20 seconds is fine. If Kubernetes is scaling pods every minute, you need sub-5-seconds at minimum.
.
