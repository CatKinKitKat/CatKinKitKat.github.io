+++
title = "CompletableFuture and the Art of Async Java"
date = 2026-03-07
[taxonomies]
tags = ["java", "concurrency"]
+++

Before virtual threads, before Project Reactor, before anyone said "reactive" in a meeting and made everyone uncomfortable, there was `CompletableFuture`. It landed in Java 8, and for a lot of us, it was the first time async programming in Java didn't feel like chewing glass.

It's still my go-to for async work that doesn't justify pulling in a reactive framework. Let me walk through what works, what doesn't, and where the sharp edges are.

## The Basics: thenApply vs thenCompose

These two methods are the `map` and `flatMap` of the CompletableFuture world, and confusing them is a rite of passage.

`thenApply` transforms the result. You give it a function, it gives you back a `CompletableFuture` of the transformed type:

```java
CompletableFuture<String> name = getUserAsync(id)
    .thenApply(User::getName);
// CompletableFuture<User> -> CompletableFuture<String>
```

`thenCompose` chains another async operation. If your function returns a `CompletableFuture`, use `thenCompose` to avoid nesting:

```java
// Wrong - gives you CompletableFuture<CompletableFuture<Order>>
getUserAsync(id).thenApply(user -> getOrdersAsync(user.getId()));

// Right - gives you CompletableFuture<Order>
getUserAsync(id).thenCompose(user -> getOrdersAsync(user.getId()));
```

If you've used `Optional.map()` vs `Optional.flatMap()`, or `Stream.map()` vs `Stream.flatMap()`, it's the same idea. When your mapping function returns the wrapper type, flatten it.

## The Async Variants

Every operation has three flavors:

```java
future.thenApply(fn);        // runs on completing thread
future.thenApplyAsync(fn);   // runs on ForkJoinPool.commonPool()
future.thenApplyAsync(fn, executor);  // runs on your executor
```

The non-async version runs on whatever thread completed the previous stage. This is fine most of the time, but it means if the upstream stage completed on some random Netty thread, your transformation runs there too. This has bitten me in production - a "cheap" transformation that turned out to be not-so-cheap, running on an event loop thread it had no business being on.

My rule: if there's any doubt about where it runs, use the explicit executor variant. The default `ForkJoinPool.commonPool()` is shared across your entire JVM and I've seen it become a bottleneck in services that abuse `thenApplyAsync` without thinking about it.

```java
ExecutorService executor = Executors.newFixedThreadPool(10);

CompletableFuture<Result> result = fetchDataAsync()
    .thenApplyAsync(this::heavyTransformation, executor)
    .thenApplyAsync(this::formatResult, executor);
```

## Error Handling

Error handling is where `CompletableFuture` gets real. An exception in any stage propagates downstream and skips subsequent stages until it hits an error handler.

```java
CompletableFuture<User> user = getUserAsync(id)
    .thenApply(this::enrichUser)
    .exceptionally(ex -> {
        log.warn("Failed to get user {}, returning fallback", id, ex);
        return User.anonymous();
    });
```

`exceptionally` only handles errors. If you need to handle both success and failure, use `handle`:

```java
CompletableFuture<Response> response = callExternalService()
    .handle((result, ex) -> {
        if (ex != null) {
            return Response.error(ex.getMessage());
        }
        return Response.success(result);
    });
```

One gotcha: the exception in `exceptionally` and `handle` is always a `CompletionException` wrapping your actual exception. You need to call `getCause()` to get to the real thing. I've seen code that catches `TimeoutException` in an `exceptionally` block and wonders why it never matches.

```java
.exceptionally(ex -> {
    Throwable cause = ex instanceof CompletionException ? ex.getCause() : ex;
    if (cause instanceof TimeoutException) {
        return fallbackValue;
    }
    throw new CompletionException(cause);
})
```

Java 12 added `exceptionallyCompose` which lets you return a new `CompletableFuture` from your error handler - useful for async fallback logic.

## Combining Futures

Combining multiple futures is where `CompletableFuture` starts to earn its keep:

```java
// Wait for both, combine results
CompletableFuture<String> greeting = getUserAsync(id)
    .thenCombine(getPreferencesAsync(id), (user, prefs) ->
        String.format("Hello %s, your theme is %s", user.getName(), prefs.getTheme()));

// Wait for all of them
CompletableFuture<Void> all = CompletableFuture.allOf(
    updateInventory(order),
    sendConfirmationEmail(order),
    notifyWarehouse(order)
);

// Wait for the fastest
CompletableFuture<String> fastest = CompletableFuture.anyOf(
    fetchFromPrimary(),
    fetchFromReplica()
).thenApply(obj -> (String) obj);  // anyOf returns CompletableFuture<Object>, ugh
```

`allOf` returns `CompletableFuture<Void>`, which means you lose the individual results. You have to join each future after allOf completes. It's clunky:

```java
CompletableFuture<User> userFuture = getUserAsync(id);
CompletableFuture<List<Order>> ordersFuture = getOrdersAsync(id);
CompletableFuture<Preferences> prefsFuture = getPreferencesAsync(id);

CompletableFuture<Dashboard> dashboard = CompletableFuture
    .allOf(userFuture, ordersFuture, prefsFuture)
    .thenApply(v -> new Dashboard(
        userFuture.join(),
        ordersFuture.join(),
        prefsFuture.join()
    ));
```

Not beautiful. But it works and it's predictable.

## Converting Future to CompletableFuture

If you're dealing with libraries that return the old `Future` interface (looking at you, every pre-2014 Java library), you need to bridge the gap. There's no built-in conversion, because `Future.get()` is blocking.

The straightforward approach:

```java
public static <T> CompletableFuture<T> toCompletable(Future<T> future, Executor executor) {
    return CompletableFuture.supplyAsync(() -> {
        try {
            return future.get();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new CompletionException(e);
        } catch (ExecutionException e) {
            throw new CompletionException(e.getCause());
        }
    }, executor);
}
```

This works but it burns a thread waiting. If you're doing this a lot, consider whether the library offers a callback-based or `ListenableFuture` alternative. Guava's `ListenableFuture` can be converted without burning a thread.

With virtual threads in Java 21+, the "burns a thread" concern mostly evaporates. Blocking a virtual thread is cheap. This is one of those cases where virtual threads simplify the entire conversion problem.

## Timeouts

Java 9 added `orTimeout` and `completeOnTimeout`:

```java
CompletableFuture<Data> data = fetchSlowService()
    .orTimeout(5, TimeUnit.SECONDS)           // fails with TimeoutException
    .completeOnTimeout(Data.empty(), 5, TimeUnit.SECONDS);  // completes with default
```

Before Java 9, you had to roll your own timeout with a `ScheduledExecutorService`. It was ugly. These two methods should have been there from day one.

## The Kotlin Coroutines Comparison

If your team uses Kotlin, coroutines are the obvious alternative. The comparison isn't really fair - coroutines are a language feature, `CompletableFuture` is a library class - but people ask, so here's the honest take.

Coroutines make async code look synchronous:

```kotlin
suspend fun getDashboard(id: String): Dashboard {
    val user = async { getUser(id) }
    val orders = async { getOrders(id) }
    val prefs = async { getPreferences(id) }
    return Dashboard(user.await(), orders.await(), prefs.await())
}
```

Compare that to the `allOf` + `join` dance in Java. It's cleaner. Structured concurrency in Kotlin (via `coroutineScope`) also gives you automatic cancellation - if one child fails, siblings are cancelled. With `CompletableFuture`, you have to wire up cancellation yourself, and most people don't bother.

But coroutines require Kotlin. If your codebase is Java and you're not planning a language migration, `CompletableFuture` is what you have. And honestly, for most use cases - calling a few external services concurrently, doing some async fire-and-forget work - it's perfectly adequate.

Java 21's `StructuredTaskScope` (still in preview) is the JDK's answer to structured concurrency, and it's worth watching. When it stabilizes, it'll be the best of both worlds - structured concurrency semantics with plain Java.

## My Rules of Thumb

After years of working with `CompletableFuture`:

1. Always provide an explicit executor for async variants. Don't abuse the common pool.
2. Keep chains short. If your chain is 8 stages deep, break it into named methods.
3. Always handle errors. An unhandled exception in a `CompletableFuture` is silently swallowed if nobody calls `join()` or `get()`.
4. Use `orTimeout` everywhere you call an external service. Futures without timeouts are futures waiting to ruin your weekend.
5. Don't mix blocking and async. If you call `.get()` in the middle of a chain, you've defeated the purpose.

`CompletableFuture` isn't sexy. It doesn't have its own conference talks anymore. But it's in the JDK, it has no dependencies, and it works. Sometimes that's enough.
