+++
title = "ZGC and the War on Tail Latency"
date = 2026-03-10
[taxonomies]
tags = ["java", "jvm", "performance"]
+++

Nobody cares about garbage collection until their p99 latency spikes to 500ms and the SRE team starts asking uncomfortable questions. Then, suddenly, everyone cares about garbage collection.

I spent a few months last year going down the GC tuning rabbit hole for a service with strict latency requirements. Here's what I learned, filtered through the lens of a backend developer who'd rather be writing business logic.

## The Problem: GC Pauses

Every garbage collector needs to stop the world (STW) at some point. During a STW pause, your application threads freeze. No requests are processed. No responses are sent. Your service just... stops.

For G1GC (the default since Java 9), STW pauses typically range from 10ms to 200ms, depending on heap size and workload. For most services, this is fine. For services where your SLA says "p99 under 50ms" and your heap is 8GB, G1's pauses are a problem.

## ZGC: The Sub-Millisecond Collector

ZGC (Z Garbage Collector) was designed with one goal: keep pause times under 1ms regardless of heap size. Not 10ms. Not 5ms. Sub-millisecond.

```bash
java -XX:+UseZGC -XX:+ZGenerational -Xmx16g -jar myapp.jar
```

Since Java 21, ZGC is generational by default (the `-XX:+ZGenerational` flag is no longer needed in newer versions). Generational ZGC is strictly better - it collects short-lived objects more efficiently and reduces CPU overhead compared to the original non-generational ZGC.

How does ZGC keep pauses so short? It does almost all its work concurrently - while your application threads are running. The only STW phases are brief root scanning operations that don't scale with heap size. A 16GB heap and a 1TB heap have the same pause time.

The trade-off: ZGC uses more CPU for concurrent GC work. Your application gets slightly less CPU time overall. In practice, I've seen 5-15% throughput reduction compared to G1 on CPU-bound workloads. For I/O-bound services (most of mine), the difference is negligible.

## ZGC vs G1 vs Shenandoah

Here's the comparison nobody gives you straight:

**G1GC:**
- Default collector. Well-understood. Battle-tested.
- Pause times: 10-200ms typically, tunable with `-XX:MaxGCPauseMillis`.
- Best for: general-purpose workloads where occasional pauses are acceptable.
- Throughput: good. The best bang for your CPU buck in most cases.

**ZGC:**
- Sub-millisecond pauses. Constant, predictable.
- Slightly lower throughput than G1 (concurrent work costs CPU).
- Best for: latency-sensitive services, large heaps (16GB+).
- Since Java 21, generational mode makes it practical for a wider range of workloads.

**Shenandoah:**
- Also targets low-pause-time collection. Similar goals to ZGC.
- Available in OpenJDK (not in Oracle JDK historically, though this has changed).
- Pause times: also sub-millisecond in most cases.
- Uses a different approach (Brooks forwarding pointers) vs ZGC (colored pointers).
- In my testing, ZGC and Shenandoah perform similarly for latency. ZGC has better large-heap behavior.

My recommendation: if you're on Java 21+ and need low latency, start with ZGC. It's the better-supported option and generational ZGC closed the throughput gap with G1 significantly.

## Pause Time Targets

G1 has `-XX:MaxGCPauseMillis` which is a *target*, not a guarantee. G1 will try to stay under your target but will exceed it if it has to:

```bash
# G1 with 50ms pause target
java -XX:+UseG1GC -XX:MaxGCPauseMillis=50 -Xmx8g -jar myapp.jar
```

ZGC doesn't have a pause time target because it doesn't need one. Its pauses are consistently under 1ms. What you tune with ZGC is heap size and how aggressively it collects:

```bash
# ZGC with soft max heap hint
java -XX:+UseZGC -Xmx16g -XX:SoftMaxHeapSize=12g -jar myapp.jar
```

`SoftMaxHeapSize` tells ZGC "try to stay under 12GB, but you can go up to 16GB if needed." This lets you balance memory footprint against GC frequency.

## Native Memory Tracking

GC tuning without data is just superstition. Native Memory Tracking (NMT) shows you where the JVM is actually spending memory:

```bash
java -XX:NativeMemoryTracking=summary -jar myapp.jar

# Then at runtime:
jcmd <pid> VM.native_memory summary
```

Output looks something like:

```
Total: reserved=4567MB, committed=2345MB
-                 Java Heap (reserved=2048MB, committed=2048MB)
-                     Class (reserved=1100MB, committed=85MB)
-                    Thread (reserved=200MB, committed=200MB)
-                      Code (reserved=250MB, committed=45MB)
-                        GC (reserved=500MB, committed=120MB)
-                  Internal (reserved=30MB, committed=30MB)
-                    Symbol (reserved=15MB, committed=15MB)
```

The "GC" line tells you how much memory the collector itself uses. ZGC's overhead is proportional to heap size - roughly 3-5% of your max heap. For a 16GB heap, expect 500-800MB of GC overhead. That's memory your application doesn't get to use. Factor this into your container memory limits.

I've seen containers OOM-killed because someone set `-Xmx` equal to the container memory limit without accounting for off-heap memory. Leave headroom. A good rule of thumb: set container memory to 1.5x your Xmx, minimum.

## Class Unloading in Layered Applications

This is a niche topic, but if you run Spring Boot applications with lots of dynamically generated classes (Hibernate proxies, CGLIB proxies, Spring AOP), class metadata can accumulate in the Metaspace.

By default, ZGC performs concurrent class unloading. You can verify it's working:

```bash
java -XX:+UseZGC -Xlog:gc+classunloading=info -jar myapp.jar
```

In layered applications - think Spring Boot with many auto-configured modules - the class count can be surprisingly high. I've seen services with 30,000+ loaded classes. If you're seeing Metaspace pressure, check:

```bash
jcmd <pid> VM.native_memory summary | grep Class
```

And set reasonable Metaspace limits:

```bash
java -XX:MaxMetaspaceSize=256m -jar myapp.jar
```

Without a limit, Metaspace grows unbounded and can eat into your container's memory headroom. With a limit, you'll get an `OutOfMemoryError: Metaspace` that you can diagnose, rather than a mysterious container kill.

## When to Care About GC Tuning

Here's the honest truth: most services don't need GC tuning. The defaults work. If your p99 is fine and your throughput meets requirements, don't touch the GC. Premature GC tuning is the JVM equivalent of premature optimization.

**Care when:**
- Your p99/p999 latency has spikes that correlate with GC pauses (check your GC logs)
- You have a large heap (8GB+) and G1's pauses are too long
- You're running in a memory-constrained container and need to understand the GC's overhead
- You're seeing `OutOfMemoryError` and need to understand where memory is going

**Don't bother when:**
- Your service is meeting its SLAs
- You haven't looked at GC logs yet (look first, tune second)
- You're trying to squeeze 5% more throughput out of a service that's limited by database latency
- You read a blog post about ZGC and want to use it because it sounds cool

Enable GC logging. Always. It's free and invaluable when you need it:

```bash
java -Xlog:gc*:file=gc.log:time,uptime,level,tags -jar myapp.jar
```

Read the logs before changing anything. More often than not, the real problem isn't the GC - it's an allocation pattern in your code that's creating unnecessary garbage. Fix the cause, not the symptom.

## My Setup

For what it's worth, here's what I run on latency-sensitive services (Java 21+):

```bash
java \
  -XX:+UseZGC \
  -Xmx12g \
  -XX:SoftMaxHeapSize=10g \
  -XX:NativeMemoryTracking=summary \
  -Xlog:gc*:file=/var/log/app/gc.log:time,uptime,level,tags:filecount=5,filesize=50m \
  -jar myapp.jar
```

ZGC for sub-millisecond pauses. NMT enabled for diagnostics. GC logs with rotation. Nothing exotic. It works, and that's the point.

Garbage collection is one of those topics where the right answer is usually the boring answer. Use the defaults until they're not enough. Measure before you tune. And if someone in a meeting suggests switching to ZGC because they saw a conference talk, ask them what their current p99 is first.
