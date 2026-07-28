+++
title = "Thread Dumps and Heap Dumps: Reading the Wreckage"
date = 2026-03-15
[taxonomies]
tags = ["java", "jvm", "debugging"]
+++

There's a particular kind of dread that comes with a production incident where the service is "slow" and nobody knows why. CPU is normal. Memory is normal-ish. Logs show nothing. Requests are just... hanging.

This is when you take a thread dump and a heap dump. These are the JVM's equivalent of a crash site investigation, and learning to read them has saved me more weekends than I can count.

## Generating Thread Dumps

A thread dump is a snapshot of every thread in the JVM: what it's doing, what state it's in, and what locks it holds. There are several ways to get one:

```bash
# jcmd (preferred)
jcmd <pid> Thread.print

# jstack
jstack <pid>

# kill signal (Unix): prints to stdout
kill -3 <pid>

# In Kubernetes
kubectl exec <pod> - jcmd 1 Thread.print > thread_dump.txt
```

Always take at least three dumps, 5-10 seconds apart. A single dump is a photograph. Three dumps are a video. Threads that appear in the same state across all three dumps are stuck, not just momentarily busy.

## Reading Thread Dumps

A thread dump entry looks like this:

```
"http-nio-8080-exec-42" #85 daemon prio=5 os_prio=0 tid=0x7f2a3c012800 nid=0x5e
  waiting for monitor entry [0x7f2a1c5f9000]
   java.lang.Thread.State: BLOCKED (on object monitor)
        at com.myapp.service.OrderService.processOrder(OrderService.java:47)
        - waiting to lock <0x00000006c7a8b1d0> (a java.lang.Object)
        at com.myapp.controller.OrderController.createOrder(OrderController.java:31)
        ...
```

The key information:

- **Thread name** (`http-nio-8080-exec-42`): tells you which pool it's from.
- **Thread state**: RUNNABLE, BLOCKED, WAITING, TIMED_WAITING.
- **Stack trace**: what code it's executing.
- **Lock information**: what it's waiting on or holding.

Thread states that matter:

- `RUNNABLE`: executing or ready to execute. Normal unless lots of threads are stuck here doing the same thing.
- `BLOCKED`: waiting to acquire a monitor (synchronized block). The culprit is whatever thread holds the lock.
- `WAITING` / `TIMED_WAITING`: waiting on a condition (Object.wait, LockSupport.park, Thread.sleep). Normal for idle pool threads. Suspicious if your request threads are all here.

## Common Patterns

### The Deadlock

Two threads each hold a lock the other needs. The JVM actually detects this and prints it at the bottom of the thread dump:

```
Found one Java-level deadlock:
=============================
"Thread-1":
  waiting to lock monitor 0x7f2a3c012800 (object 0x6c7a8b1d0, a com.myapp.LockA),
  which is held by "Thread-2"
"Thread-2":
  waiting to lock monitor 0x7f2a3c014000 (object 0x6c7a8b2e0, a com.myapp.LockB),
  which is held by "Thread-1"
```

Classic deadlock. The fix depends on the code, but the diagnosis is instant: the JVM literally tells you which threads and which locks.

### Thread Pool Starvation

This is the most common pattern I see. All threads in a pool are busy or blocked, and new requests queue up indefinitely:

```
"http-nio-8080-exec-1" TIMED_WAITING  - waiting on external HTTP call
"http-nio-8080-exec-2" TIMED_WAITING  - waiting on external HTTP call
... (all 200 threads the same)
```

Every Tomcat thread is stuck waiting for a downstream service that's slow or dead. No threads available to handle new requests. The fix: circuit breakers, timeouts, and bulkheads. But the thread dump is what tells you this is happening.

### The Synchronized Bottleneck

Lots of threads blocked on the same synchronized method:

```
"http-nio-8080-exec-15" BLOCKED
  - waiting to lock <0x00000006c7a8b1d0> (a com.myapp.LegacyCache)
  at com.myapp.LegacyCache.get(LegacyCache.java:22)

"http-nio-8080-exec-23" BLOCKED
  - waiting to lock <0x00000006c7a8b1d0> (a com.myapp.LegacyCache)
  at com.myapp.LegacyCache.get(LegacyCache.java:22)

"http-nio-8080-exec-7" RUNNABLE
  - locked <0x00000006c7a8b1d0> (a com.myapp.LegacyCache)
  at com.myapp.LegacyCache.get(LegacyCache.java:22)
  at java.net.SocketInputStream.read(...)
```

One thread holds the lock and is doing a blocking I/O operation inside the synchronized block. Everyone else waits. I've seen this with legacy cache implementations that do HTTP calls inside synchronized methods. Replace the synchronized block with a `ConcurrentHashMap` or a proper cache library.

### The Thread Leak

Thread count grows over time. The thread dump shows hundreds of threads with names like `pool-47-thread-1`, `pool-48-thread-1`, suggesting someone is creating new thread pools instead of reusing them. Check the stack traces on these threads to find the culprit.

## Generating Heap Dumps

A heap dump is a snapshot of every object on the JVM heap. It's large (roughly equal to your heap size) and takes time to generate, but it's the only way to diagnose memory issues definitively.

```bash
# Generate on demand
jcmd <pid> GC.heap_dump /tmp/heap.hprof

# jmap (older approach)
jmap -dump:format=b,file=/tmp/heap.hprof <pid>

# Auto-generate on OOM
java -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/dumps/ -jar myapp.jar
```

The `-XX:+HeapDumpOnOutOfMemoryError` flag should be on every JVM you run. When you get an OOM in production at 3 AM, you want the heap dump waiting for you, not an unreproducible problem and no data.

In Kubernetes:

```yaml
env:
- name: JAVA_OPTS
  value: >-
    -XX:+HeapDumpOnOutOfMemoryError
    -XX:HeapDumpPath=/var/dumps/heap.hprof
```

Make sure the path is a volume mount, not the container filesystem.

## Analyzing Heap Dumps

### Eclipse Memory Analyzer (MAT)

MAT is the standard tool for heap dump analysis. It's ugly, it's a standalone Eclipse-based application, and it's indispensable.

Open the heap dump and MAT immediately runs a "Leak Suspects" report. This catches the obvious cases: one class holding 80% of the heap. For subtler issues:

**Dominator Tree:** Shows the largest objects and what they retain. If a single `HashMap` retains 4GB, you've found your problem. The dominator tree shows the chain of references keeping it alive.

**Histogram:** Lists all classes by instance count and retained size. Sort by retained size. If `byte[]` dominates, follow the references to see what's holding those arrays.

**OQL (Object Query Language):** SQL-like queries against the heap:

```sql
SELECT * FROM java.util.HashMap WHERE size > 10000
```

This finds suspiciously large HashMaps. I've caught memory leaks this way: caches without eviction policies that grow without bound.

### VisualVM

VisualVM is lighter than MAT and good for live monitoring. Connect to a running JVM and you get real-time thread visualization, heap usage graphs, and basic profiling. It can also open heap dumps, though its analysis is less detailed than MAT.

VisualVM's thread tab is actually my preferred way to visualize thread states over time. It color-codes threads by state (green = running, yellow = waiting, red = blocked) and you can see patterns immediately.

### Common Memory Leak Patterns

**The unbounded cache:**
```java
private static final Map<String, Object> cache = new HashMap<>();
// Someone adds to this cache. Nobody removes from it. Ever.
```

MAT shows a HashMap with millions of entries retaining gigabytes. The fix: use a proper cache (Caffeine, Guava Cache) with size limits and TTL.

**The listener that never unregisters:**
```java
eventBus.register(this);  // In constructor
// No unregister in close()/destroy(). Every new instance adds a listener.
// The event bus holds a reference to every instance ever created.
```

MAT shows a list or set growing over time with references to objects that should have been garbage collected.

**The StringBuilder in a loop:**
```java
StringBuilder sb = new StringBuilder();
for (Record record : millionRecords) {
    sb.append(record.toString()).append("\n");  // 2GB string? Sure, why not.
}
```

This one's less a "leak" and more a "you're building a 2GB string in memory." But you see it in heap dumps as a massive `char[]` or `byte[]`.

**The ClassLoader leak:**
In environments that redeploy without restarting (application servers, OSGi), old ClassLoaders can be retained by static references, keeping the entire old version of the application in memory. This is less common with Spring Boot (which typically restarts the JVM), but it still happens with hot-reload tools in development.

## A Systematic Approach

When something goes wrong, I follow this order:

1. **Thread dump first**: it's instant and non-destructive. Take three, 5 seconds apart.
2. **Look for blocked/waiting patterns**: are all threads stuck on the same thing?
3. **If memory is the issue, take a heap dump**: but be aware it pauses the JVM and creates a large file.
4. **Use MAT for the heap dump**: start with the Leak Suspects report.
5. **Correlate**: thread dump shows what code was running, heap dump shows what data was in memory. Together they tell the story.

None of this is glamorous. It's archaeological work: sifting through data looking for the one thing that's wrong. But it beats staring at logs and guessing. And once you've diagnosed your first deadlock or memory leak from a dump, you'll never want to debug without them again.

Keep `-XX:+HeapDumpOnOutOfMemoryError` on every JVM. Learn to read thread dumps. Know how to use MAT. These aren't advanced skills: they're survival skills.
