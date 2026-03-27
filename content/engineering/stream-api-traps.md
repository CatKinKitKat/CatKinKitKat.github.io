+++
title = "Stream API Traps and How to Not Shoot Yourself in the Foot"
date = 2026-03-27
[taxonomies]
tags = ["java", "performance"]
+++

The Stream API is one of the best things that happened to Java. It's also one of the most misused. After years of reviewing code that uses streams for everything from simple transformations to full-blown business logic, I have opinions. Strong ones.

## Lazy Evaluation: The Trap Nobody Warns You About

Streams are lazy. Nothing happens until a terminal operation is called. This is a feature, not a bug, except when you forget about it.

```java
Stream<Order> expensiveOrders = orders.stream()
    .filter(o -> o.total().compareTo(threshold) > 0)
    .peek(o -> log.info("Found expensive order: {}", o.id()));

// Nothing has been logged yet. The stream hasn't executed.
```

I've seen this in production code where someone used `peek` for side effects and then wondered why nothing was happening. The stream was never consumed. No terminal operation, no execution.

Worse: streams are single-use. If you try to consume the same stream twice, you get an `IllegalStateException`. This surprises people who think of streams like collections.

```java
Stream<String> names = people.stream().map(Person::name);
long count = names.count();         // Works
List<String> list = names.toList(); // IllegalStateException
```

## Parallel Streams: Almost Never Worth It

I'm going to say something controversial: I've never seen a parallel stream improve performance in a real Spring Boot application. Not once.

The problem is that `parallelStream()` uses the common ForkJoinPool, which has a fixed number of threads equal to `Runtime.getRuntime().availableProcessors() - 1`. In a web application, you already have hundreds of requests running concurrently. Adding parallel streams means those requests are now competing for the same small ForkJoinPool.

```java
// This looks fast in a microbenchmark
list.parallelStream()
    .map(this::expensiveComputation)
    .toList();

// In production, it's sharing a thread pool with every other parallel stream
// in every other request. You've just created contention.
```

The overhead of splitting, thread handoff, and merging means parallel streams only help with large datasets (10,000+ elements) doing CPU-intensive work. If you're doing I/O in a parallel stream, you're blocking ForkJoinPool threads. That's a great way to stall your entire application.

When I want parallelism, I use a dedicated ExecutorService with virtual threads and `CompletableFuture`. Explicit, controllable, doesn't pollute the common pool.

## mapMulti: The Stream Operation You're Not Using

Added in Java 16, `mapMulti` is `flatMap` without the intermediate stream creation:

```java
// flatMap creates a stream for each element
orders.stream()
    .flatMap(o -> o.lineItems().stream())
    .toList();

// mapMulti avoids the intermediate stream
orders.stream()
    .<LineItem>mapMulti((order, consumer) -> {
        for (LineItem item : order.lineItems()) {
            consumer.accept(item);
        }
    })
    .toList();
```

Is it more verbose? Yes. Is it faster? For small inner collections, measurably so: no Stream object allocation per element. I use it when the inner collection is small (1-3 elements) and the outer collection is large. For everything else, `flatMap` is more readable.

`mapMulti` also lets you filter and transform in one step:

```java
objects.stream()
    .<String>mapMulti((obj, consumer) -> {
        if (obj instanceof String s && s.length() > 3) {
            consumer.accept(s.toUpperCase());
        }
    })
    .toList();
```

## Gatherers: Custom Intermediate Operations (Java 24+)

Gatherers are the biggest addition to the Stream API since its introduction. They let you write custom intermediate operations - something that was impossible before without collecting and re-streaming.

```java
// Windowing: group elements into fixed-size lists
Stream.of(1, 2, 3, 4, 5, 6, 7)
    .gather(Gatherers.windowFixed(3))
    .toList();
// [[1, 2, 3], [4, 5, 6], [7]]

// Sliding window
Stream.of(1, 2, 3, 4, 5)
    .gather(Gatherers.windowSliding(3))
    .toList();
// [[1, 2, 3], [2, 3, 4], [3, 4, 5]]
```

You can write custom gatherers for things like deduplication with a window, rate limiting elements, or accumulating state across the stream. Before this, you'd break out of the stream pipeline, use a loop, and come back. Now the pipeline stays clean.

## Performance vs. Readability: The Real Trade-off

Here's my rule of thumb:

**Use streams when they make the code more readable than a loop.** That's the only criterion that matters for 99% of application code.

```java
// Stream: clear intent, easy to follow
var activeEmails = users.stream()
    .filter(User::isActive)
    .map(User::email)
    .toList();

// Loop: same result, more ceremony
var activeEmails = new ArrayList<String>();
for (User user : users) {
    if (user.isActive()) {
        activeEmails.add(user.email());
    }
}
```

The stream version wins on readability. The loop version is marginally faster (no lambda allocation, no stream object), but we're talking nanoseconds for typical collection sizes.

Where streams lose on readability:

```java
// This is unreadable. Use a loop.
var result = orders.stream()
    .collect(Collectors.groupingBy(
        Order::customerId,
        Collectors.collectingAndThen(
            Collectors.reducing(
                BigDecimal.ZERO,
                Order::total,
                BigDecimal::add),
            total -> total.compareTo(threshold) > 0)));
```

If your stream pipeline has more than 3-4 operations, or nested collectors that require squinting to parse, break it into named steps or use a loop. Nobody is impressed by a single expression that takes five minutes to understand.

## Collectors.toUnmodifiableList() vs .toList()

Since Java 16, `Stream.toList()` returns an unmodifiable list. Use it. It's shorter than `collect(Collectors.toList())` and the result is immutable.

```java
var names = people.stream().map(Person::name).toList(); // Unmodifiable
names.add("nope"); // UnsupportedOperationException
```

One gotcha: `Collectors.toList()` returns a mutable `ArrayList`. If downstream code relies on mutating the result, switching to `.toList()` will break it. Ask me how I know.

## Don't Stream Everything

Not everything needs to be a stream. Single-element operations, simple conditionals, and early returns are better expressed with plain code.

```java
// Don't do this
Optional<User> user = users.stream()
    .filter(u -> u.id().equals(targetId))
    .findFirst();

// If you have a Map, just use it
User user = usersById.get(targetId);
```

Streams are a tool. A great tool. But when all you have is a stream, everything looks like a pipeline. Sometimes a for loop is the right answer, and that's fine.
