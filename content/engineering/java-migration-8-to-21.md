+++
title = "Java Migration Survival Guide: 8 to 21"
date = 2026-03-29
[taxonomies]
tags = ["java", "migration"]
+++

I've migrated three production systems from Java 8 to Java 21. Each time I told myself "it'll be straightforward this time." Each time I was wrong. Here's the field guide I wish I'd had.

## The Big Picture

Java 8 to 21 is not one migration. It's at least four: 8 to 11, 11 to 17, 17 to 21, and the javax-to-jakarta jump if you're on Spring Boot 3. Each has its own landmines.

## Java 8 to 11: The Module System Ruins Your Day

### Strong Encapsulation

Java 9 introduced the module system (Project Jigsaw). In practice, this means internal JDK APIs that your code (or your dependencies) relied on are now inaccessible. The biggest offenders:

- `sun.misc.Unsafe`: used by serialization libraries, off-heap memory allocators, and half of Netty
- `com.sun.xml.*`: if you're doing XML processing with internal APIs
- `javax.annotation.*`: removed from the JDK entirely (moved to Jakarta)

The short-term fix is `--add-opens` and `--add-exports` flags. The long-term fix is updating your dependencies. In my experience, the dependency updates took longer than the actual code changes.

### Removed Packages

Java 11 removed several packages that were deprecated in Java 9:

```
java.xml.ws (JAX-WS)
java.xml.bind (JAXB)
java.activation
java.corba
java.transaction
```

If your code uses JAXB (and if you're coming from Java 8 enterprise code, it probably does), you need to add the Jakarta equivalents as explicit dependencies:

```xml
<dependency>
    <groupId>jakarta.xml.bind</groupId>
    <artifactId>jakarta.xml.bind-api</artifactId>
    <version>4.0.0</version>
</dependency>
<dependency>
    <groupId>org.glassfish.jaxb</groupId>
    <artifactId>jaxb-runtime</artifactId>
    <version>4.0.3</version>
</dependency>
```

### The var Keyword (Java 10)

`var` for local variables. It's not dynamic typing; it's local type inference. The type is still checked at compile time.

```java
var orders = orderRepository.findAll();  // List<Order>
var total = BigDecimal.ZERO;             // BigDecimal
var map = new HashMap<String, List<Order>>();  // HashMap<String, List<Order>>
```

My rule: use `var` when the type is obvious from the right side. Don't use it when the type matters for understanding the code. `var x = service.process(input)` tells me nothing. `ProcessingResult x = service.process(input)` tells me what I'm working with.

## Java 11 to 17: The Good Stuff

This is the most pleasant jump. Mostly new features, fewer breaking changes.

### Text Blocks (Java 15)

```java
String sql = """
    SELECT o.id, o.total
    FROM orders o
    WHERE o.status = 'ACTIVE'
    AND o.created_at > :since
    """;
```

No more string concatenation for multi-line SQL, JSON, or XML. The indentation is handled by the compiler; trailing whitespace is stripped based on the closing `"""` position.

### Sealed Classes (Java 17)

Covered in another post, but the migration impact is minimal. It's purely additive; no existing code breaks.

### Records (Java 16)

Also additive, no breaking changes. Start using them for DTOs and value objects.

### Helpful NullPointerExceptions (Java 14)

```
// Before: NullPointerException
// After:  Cannot invoke "String.length()" because "user.address().city()" is null
```

This alone is worth the upgrade. The number of hours I've spent debugging nested NPEs with no context is embarrassing.

### Breaking Change: Removed Nashorn

If you're using JavaScript scripting via Nashorn, it's gone in Java 15. Use GraalJS or rethink why you're running JavaScript inside the JVM (please rethink this).

## Java 17 to 21: Virtual Threads and the Future

### Virtual Threads (Java 21)

Covered in detail in my virtual threads post. The migration impact: if you use `synchronized` blocks around I/O operations, you need to convert them to `ReentrantLock`. Otherwise, it's opt-in via configuration.

### Pattern Matching for switch (Java 21)

Finalized. You can start using it immediately. Covered in the sealed classes post.

### Sequenced Collections (Java 21)

New interfaces: `SequencedCollection`, `SequencedSet`, and `SequencedMap`. They add `getFirst()`, `getLast()`, and `reversed()` to ordered collections. Mostly additive, but if you have classes that implement both `List` and `Deque`, you might hit method resolution issues.

### Breaking Change: Deprecated and Removed Security Managers

If your application relies on `SecurityManager`, you have a problem. It's deprecated for removal in Java 17 and effectively non-functional in Java 21. Most applications don't use it directly, but some legacy frameworks do.

## The javax to jakarta Migration

This deserves its own section because it's the single most annoying part of the whole journey. Spring Boot 3 requires Jakarta EE 9+, which means every `javax.*` import becomes `jakarta.*`.

```java
// Before
import javax.persistence.Entity;
import javax.validation.constraints.NotNull;
import javax.servlet.http.HttpServletRequest;

// After
import jakarta.persistence.Entity;
import jakarta.validation.constraints.NotNull;
import jakarta.servlet.http.HttpServletRequest;
```

It's not just your code. It's every library that touches these namespaces: JPA, Bean Validation, Servlet API, JMS, and JSON-B.

Tools that help:
- **OpenRewrite**: automated refactoring. Run `org.openrewrite.java.migrate.jakarta.JavaxMigrationToJakarta` and it handles most of the import changes.
- **IntelliJ's migration tool**: decent for simple cases, misses some edge cases.
- **Eclipse Transformer**: can transform JAR files from javax to jakarta at the bytecode level.

Don't try to do this manually. I made that mistake on the first migration. Hundreds of files, each with multiple imports. OpenRewrite did it in 30 seconds.

## Preview Features Policy

My policy: don't use preview features in production code. They can change between releases. They require `--enable-preview` at compile and runtime. If you forget the flag in one environment, things break in confusing ways.

Use preview features in side projects and experiments. Wait for finalization before adopting them in production. Java's release cadence is fast enough (every 6 months) that you won't wait long.

## The Migration Checklist

1. **Update your build tool first.** Maven 3.9+ or Gradle 8+ for Java 21 support.
2. **Update dependencies before changing the Java version.** Many libraries need newer versions for Java 11+.
3. **Run with `--illegal-access=warn` first** (Java 11-15) to find reflection issues without breaking anything.
4. **Fix deprecation warnings.** Most removals were deprecated for 2+ releases before being removed.
5. **Run your full test suite at each step.** Don't jump 8 to 21 in one shot. Go 8 to 11, stabilize, then 11 to 17, stabilize, then 17 to 21.
6. **Check your Docker base images.** Eclipse Temurin is the standard now. Oracle's licensing changed.
7. **Update your CI/CD pipeline.** The Java version in CI should match production. Obviously.

## The Honest Truth

Java 8 to 21 is a significant migration. Budget weeks, not days. The code changes are usually the easy part; it's the dependency hell, the obscure reflection warnings, and the one library that hasn't released a Java 17-compatible version that burns your time.

But Java 21 is genuinely worth it. The language features, the performance improvements, the GC improvements (ZGC, Shenandoah), and virtual threads make it the best version of Java ever shipped. The migration pain is temporary. The benefits are permanent.
