+++
title = "GraalVM Native Images - Worth the Trade-Off?"
date = 2026-03-17
[taxonomies]
tags = ["java", "graalvm", "performance"]
+++

Every few months, someone on my team rediscovers GraalVM native images and gets excited about the startup time numbers. "50 milliseconds! A 20MB binary! It's like Go but it's Java!" And then they try to build their Spring Boot service as a native image and things get... educational.

Native images are genuinely impressive technology. They're also genuinely painful in ways the marketing materials don't emphasize. Let me give you the unfiltered version.

## What Native Images Are

GraalVM's `native-image` tool performs ahead-of-time (AOT) compilation. Instead of shipping bytecode and a JVM, you compile your Java application into a standalone native executable. No JVM required at runtime. The result is a platform-specific binary that starts fast and uses less memory.

```bash
# Build a native image
native-image -jar myapp.jar -o myapp

# Or with Spring Boot 3.x
./mvnw -Pnative native:compile
```

The output is a single binary. No JRE, no classpath, no `-Xmx` flags. It runs like any native process.

## The Build Time Problem

Native image compilation is *slow*. The compiler performs a closed-world analysis - it traces every reachable code path from your main method, resolves all dependencies, and compiles everything ahead of time. This means:

- A trivial "Hello World" takes 30-60 seconds.
- A medium Spring Boot service takes 3-8 minutes.
- A large Spring Boot service with many dependencies can take 10-20 minutes.
- The build process consumes enormous memory - 8-16GB of RAM for a typical Spring Boot app.

CI/CD implications are real. If your build pipeline runs on small runners (2 CPU, 4GB RAM), native image compilation will fail or take forever. You need beefy build machines.

```yaml
# GitHub Actions example - you need a big runner
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: graalvm/setup-graalvm@v1
      with:
        java-version: '21'
        distribution: 'graalvm'
    - run: ./mvnw -Pnative native:compile
      env:
        MAVEN_OPTS: "-Xmx8g"
```

Compare this to a regular JAR build that takes 30 seconds and runs on any machine. The build time cost is the first trade-off, and for fast iteration during development, it's significant.

## The Reflection Configuration Pain

This is where the real suffering lives. The closed-world analysis means the compiler must know at build time about all classes that are accessed via reflection, JNI, dynamic proxies, or resource loading. Java frameworks use reflection *everywhere*.

If the compiler can't see a code path, it won't include it. Your application compiles, starts, and then throws `ClassNotFoundException` or `NoSuchMethodException` when it hits a reflective call the compiler missed.

Spring Boot 3.x has first-class GraalVM support with AOT processing that generates the necessary hints automatically. For most Spring features, it just works. But "most" is doing a lot of heavy lifting in that sentence.

Things that still cause pain:

- **Third-party libraries** that use reflection without providing GraalVM metadata. You'll need to create `reflect-config.json` entries manually.
- **Jackson serialization** of complex types. Generic types, polymorphic deserialization, and custom serializers often need explicit hints.
- **JDBC drivers** that load classes dynamically.
- **Anything using `Class.forName()`** or `ServiceLoader` without native image metadata.

The GraalVM reachability metadata repository helps - it's a community-maintained collection of configuration for popular libraries. But you'll still hit gaps, especially with less popular dependencies.

```json
// reflect-config.json - manual configuration for a library
[
  {
    "name": "com.thirdparty.SomeClass",
    "allDeclaredConstructors": true,
    "allDeclaredMethods": true,
    "allDeclaredFields": true
  }
]
```

The GraalVM tracing agent can auto-generate these configs by running your app and recording reflective access:

```bash
java -agentlib:native-image-agent=config-output-dir=native-config/ -jar myapp.jar
# Exercise all code paths, then stop
# Use the generated configs in your native image build
```

This helps, but it only captures the code paths you exercise. Miss a path and you get a runtime error in production. Fun.

## Startup Speed Gains

This is the headline feature and it delivers. Numbers from a real Spring Boot service (REST API with database access):

| Metric | JVM (Java 21) | Native Image |
|---|---|---|
| Startup time | 2.8 seconds | 0.08 seconds |
| Memory at idle | 280 MB | 65 MB |
| Docker image size | 350 MB (with JRE) | 85 MB |

35x faster startup. 4x less memory. Significantly smaller container. These numbers are real and they matter for specific use cases.

## Runtime Performance Differences

Here's the part that gets less airtime: runtime performance (throughput, peak latency) after warmup is typically *worse* with native images than with the JVM.

The JVM's JIT compiler (C2) performs profile-guided optimization at runtime - it watches what your code actually does and optimizes accordingly. Speculative optimizations, inlining based on observed call patterns, loop unrolling. The JIT compiler is incredibly good at this.

Native images use AOT compilation without runtime profiling. The compiler does its best at build time, but it can't speculate on runtime behavior. In practice, I've seen 10-30% lower throughput on compute-heavy workloads with native images compared to a warmed-up JVM.

GraalVM's Profile-Guided Optimization (PGO) for native images narrows the gap. You run the app, collect a profile, and rebuild with the profile data. But it adds another step to your build process and the profile may not match production workloads perfectly.

For I/O-bound services (most web apps), the throughput difference is negligible because you're waiting on network and database, not CPU. But for compute-heavy workloads, the JIT wins.

## When Native Makes Sense

**Serverless / FaaS:**
This is the killer use case. Lambda cold starts with a JVM are painful (5-15 seconds for a Spring Boot app). With a native image, cold start is under 200ms. If you're running on AWS Lambda, Azure Functions, or Google Cloud Run, native images are transformative.

**CLI tools:**
If you're building a command-line tool in Java (using Picocli, for example), native images are perfect. Sub-100ms startup, small binary, no JRE dependency. It feels like a Go binary.

**Sidecar processes:**
Small utility processes that run alongside your main application - health checkers, config reloaders, metrics exporters. Low memory footprint and instant startup are exactly what you want.

**Kubernetes with aggressive scaling:**
If you scale from 0 and need pods ready in milliseconds, native images deliver. Combined with GraalVM's low memory footprint, you can pack more instances per node.

## When Native Doesn't Make Sense

**Long-running services with stable traffic:**
If your service starts once and runs for weeks, startup time is irrelevant. The JIT's runtime optimizations give you better throughput. You're trading runtime performance for a one-time startup benefit you only see during deployments.

**Services with heavy reflection:**
Hibernate, some serialization libraries, and anything that generates classes at runtime will fight you. The configuration burden isn't worth it for a service that starts in 3 seconds on the JVM anyway.

**Rapid development iteration:**
Native image builds take minutes. JVM builds take seconds. During development, the feedback loop matters. You can mitigate this with the JVM during development and native only for deployment, but it means you're testing in a different runtime than production. That's always risky.

**Debugging production issues:**
You lose some debugging tools. JFR support is limited (though improving). jcmd, jmap, and other JVM diagnostic tools don't apply. If you rely on these for production debugging (I do), native images make your incident response harder.

## The Practical Middle Ground

For most of my services - long-running Spring Boot apps on Kubernetes - I don't use native images. The JVM with CRaC or AppCDS gives me fast-enough startup (under 1 second) without sacrificing runtime performance or debugging capability.

Where I do use native images: a few Lambda functions and CLI tools where startup time and binary size genuinely matter.

If you want to try native images with Spring Boot 3.x, the experience is genuinely good for standard CRUD applications. The Spring team has done excellent work on AOT processing. Just budget extra time for build infrastructure, be prepared for third-party library surprises, and test thoroughly - native image behavior can differ from JVM behavior in subtle ways.

The technology is impressive. The trade-offs are real. Make sure the trade-offs align with your actual requirements, not the excitement of seeing a Java app start in 50 milliseconds.
