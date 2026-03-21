+++
title = "Thread Pools and Executor Internals - The Devil's in the Queue"
date = 2026-03-21
[taxonomies]
tags = ["java", "concurrency"]
+++

I have a confession: for the first three years of my career, I used `Executors.newFixedThreadPool(10)` for everything and never once thought about why 10. It worked. Things ran concurrently. What was there to think about?

Then I ran into a production issue where a thread pool was silently queuing thousands of tasks, response times were climbing, and the monitoring showed zero errors. The pool was doing exactly what I told it to - accepting every task into an unbounded queue and processing them at its own pace. The problem wasn't the pool. The problem was me not understanding what I'd configured.

## ThreadPoolExecutor: The Constructor From Hell

Every thread pool in `java.util.concurrent` is built on `ThreadPoolExecutor`. The factory methods in `Executors` are just convenience wrappers. Understanding the raw constructor is understanding thread pools:

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    5,                     // corePoolSize
    20,                    // maximumPoolSize
    60, TimeUnit.SECONDS,  // keepAliveTime for excess threads
    new ArrayBlockingQueue<>(100),  // workQueue
    new ThreadPoolExecutor.CallerRunsPolicy()  // rejectionHandler
);
```

The lifecycle of a submitted task:

1. If fewer than `corePoolSize` threads exist, create a new thread.
2. If `corePoolSize` threads exist, put the task in the queue.
3. If the queue is full, create a new thread (up to `maximumPoolSize`).
4. If the pool is at `maximumPoolSize` and the queue is full, invoke the rejection handler.

This ordering surprises people. The pool doesn't scale up to `maximumPoolSize` first and then start queuing. It fills the core, fills the queue, and *then* adds more threads. With an unbounded queue (like `LinkedBlockingQueue` with no capacity), you'll never go above `corePoolSize` because the queue never fills.

## Why Executors.newFixedThreadPool Is Dangerous

```java
// What you write
ExecutorService pool = Executors.newFixedThreadPool(10);

// What you get
new ThreadPoolExecutor(10, 10, 0L, TimeUnit.MILLISECONDS,
    new LinkedBlockingQueue<Runnable>());
```

That `LinkedBlockingQueue` has no capacity limit. If your 10 threads are busy and tasks keep arriving, the queue grows without bound. I've seen queues accumulate millions of pending tasks, consuming gigabytes of memory, with no indication anything is wrong until the JVM runs out of heap.

Always use a bounded queue in production:

```java
ExecutorService pool = new ThreadPoolExecutor(
    10, 10, 0L, TimeUnit.MILLISECONDS,
    new ArrayBlockingQueue<>(1000),  // bounded!
    new ThreadPoolExecutor.CallerRunsPolicy()
);
```

## Rejection Policies

When the pool can't accept a task (max threads reached, queue full), the rejection handler kicks in. The JDK provides four:

- `AbortPolicy` (default) - throws `RejectedExecutionException`. Task is lost unless you catch it.
- `CallerRunsPolicy` - the submitting thread executes the task. This provides natural backpressure - if the pool is overwhelmed, the submitter slows down. My default choice.
- `DiscardPolicy` - silently drops the task. Terrible for anything you care about.
- `DiscardOldestPolicy` - drops the oldest queued task and retries. Useful if newer tasks supersede older ones.

`CallerRunsPolicy` deserves special attention. It's elegant: when the pool is saturated, the thread that submitted the task (often a Tomcat request thread) runs it directly. This automatically throttles the submission rate without losing work. But be careful - if the caller is an event loop thread (Netty), running a blocking task on it is catastrophic.

## Work-Stealing Pools

`Executors.newWorkStealingPool()` creates a `ForkJoinPool` where idle threads "steal" tasks from busy threads' local queues:

```java
ExecutorService pool = Executors.newWorkStealingPool();
// or with explicit parallelism
ExecutorService pool = Executors.newWorkStealingPool(8);
```

Work-stealing shines when tasks are uneven - some finish quickly, others take longer. In a regular fixed pool, if thread 1 has 100 quick tasks queued and thread 2 has nothing, thread 2 sits idle. In a work-stealing pool, thread 2 steals from thread 1's queue.

The `ForkJoinPool` also supports recursive task decomposition:

```java
class SumTask extends RecursiveTask<Long> {
    private final long[] array;
    private final int start, end;

    @Override
    protected Long compute() {
        if (end - start < 1000) {
            return sequentialSum(array, start, end);
        }
        int mid = (start + end) / 2;
        SumTask left = new SumTask(array, start, mid);
        SumTask right = new SumTask(array, mid, end);
        left.fork();
        return right.compute() + left.join();
    }
}
```

In practice, I rarely use `ForkJoinPool` directly. Its main role in most applications is as the backing pool for parallel streams and `CompletableFuture.supplyAsync()`. But if you have genuinely divisible, CPU-bound work, it's the right tool.

## Blocking Queue Behavior

The queue type matters more than most people realize:

**`ArrayBlockingQueue`** - fixed capacity, backed by an array. Fair ordering available (FIFO with lock fairness). My go-to for bounded queues.

**`LinkedBlockingQueue`** - optionally bounded, backed by linked nodes. Higher throughput than `ArrayBlockingQueue` under contention because it uses separate locks for put and take operations. Use with explicit capacity.

**`SynchronousQueue`** - zero capacity. Every put blocks until a take, and vice versa. Used by `Executors.newCachedThreadPool()`. Tasks are handed directly to threads without queuing. This forces the pool to create new threads when all existing threads are busy. Good for short-lived tasks; dangerous for long-lived ones (thread count can explode).

**`PriorityBlockingQueue`** - unbounded priority queue. Tasks are dequeued by priority, not FIFO. Useful for task schedulers, but unbounded means you're back to the "grows forever" problem.

```java
// High-throughput bounded pool
ThreadPoolExecutor pool = new ThreadPoolExecutor(
    coreSize, maxSize, 60, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(queueCapacity),  // separate put/take locks
    namedThreadFactory("worker"),
    new CallerRunsPolicy()
);
```

## Sizing for CPU-Bound vs I/O-Bound

The classic formula:

- **CPU-bound:** threads = number of CPU cores (or cores + 1)
- **I/O-bound:** threads = cores * (1 + wait_time / compute_time)

For a web service where each task waits 90% of the time on I/O and computes 10%: threads = 8 * (1 + 9) = 80. This is why Tomcat defaults to 200 threads - web requests are mostly waiting.

But these formulas are starting points, not answers. Real workloads are mixed. Some requests are CPU-heavy, some are I/O-heavy. The only reliable approach:

1. Start with a reasonable guess.
2. Load test.
3. Monitor thread pool utilization, queue depth, and rejection rate.
4. Adjust.

Spring Boot exposes thread pool metrics through Micrometer. Use them:

```java
@Bean
public ExecutorService myPool(MeterRegistry registry) {
    ThreadPoolExecutor executor = new ThreadPoolExecutor(
        10, 50, 60, TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(200)
    );
    return ExecutorServiceMetrics.monitor(registry, executor, "my-pool");
}
```

Now you get `my-pool.pool.size`, `my-pool.pool.active`, `my-pool.queue.size` in your metrics dashboard. When `queue.size` is consistently near capacity, you need more threads or your tasks need to be faster.

## Deterministic Concurrent Testing

Testing concurrent code is notoriously hard. Here's what actually works for me:

**Controlled executors:** In tests, replace async executors with synchronous ones:

```java
// In production
@Bean
public ExecutorService taskPool() {
    return Executors.newFixedThreadPool(10);
}

// In tests
@Bean
public ExecutorService taskPool() {
    return MoreExecutors.directExecutor();  // Guava, runs on caller thread
}
```

This makes async code deterministic for functional testing. You're not testing concurrency here - you're testing logic.

**CountDownLatch for coordination:**

```java
@Test
void testConcurrentAccess() throws Exception {
    int threadCount = 50;
    CountDownLatch startGate = new CountDownLatch(1);
    CountDownLatch endGate = new CountDownLatch(threadCount);

    for (int i = 0; i < threadCount; i++) {
        new Thread(() -> {
            try {
                startGate.await();  // All threads wait here
                service.doSomething();
            } finally {
                endGate.countDown();
            }
        }).start();
    }

    startGate.countDown();     // Release all threads at once
    endGate.await(10, TimeUnit.SECONDS);  // Wait for completion

    assertThat(service.getCount()).isEqualTo(threadCount);
}
```

**CyclicBarrier for phased testing:**

```java
CyclicBarrier barrier = new CyclicBarrier(threadCount + 1);

// Each thread does work, then hits the barrier
// The test thread also hits the barrier to synchronize
// After the barrier, you can assert intermediate state
```

**Awaitility for async assertions:**

```java
@Test
void testEventualConsistency() {
    service.submitAsync(task);

    Awaitility.await()
        .atMost(5, TimeUnit.SECONDS)
        .pollInterval(100, TimeUnit.MILLISECONDS)
        .untilAsserted(() -> assertThat(service.getResult()).isNotNull());
}
```

For stress testing, tools like jcstress (from the OpenJDK project) can detect subtle concurrency bugs that regular testing misses. But for day-to-day work, controlled executors plus Awaitility covers 90% of my needs.

## Virtual Threads Change the Math

With virtual threads (Java 21+), the thread pool sizing question changes fundamentally for I/O-bound work. You don't need a pool - create a virtual thread per task:

```java
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
```

No sizing, no queue, no rejection policy. Each task gets a virtual thread and the JVM handles the rest.

But for CPU-bound work, thread pools still matter. Virtual threads don't magically give you more CPU. A work-stealing pool with core-count parallelism is still the right answer for CPU-bound tasks.

The new normal: virtual threads for I/O-bound work, platform thread pools for CPU-bound work. Know which is which in your application, and configure accordingly.
