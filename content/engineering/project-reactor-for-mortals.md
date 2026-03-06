+++
title = "Project Reactor for Mortals"
date = 2026-03-06
[taxonomies]
tags = ["java", "reactive", "spring-boot"]
+++

Let me save you some time: you probably don't need reactive programming. I know that's a weird way to start an article about Project Reactor, but I've spent enough time watching teams adopt WebFlux for CRUD apps that I feel morally obligated to lead with the disclaimer.

That said, when you *do* need it, Reactor is genuinely impressive. Let's talk about what it actually does and when it earns its complexity tax.

## Mono and Flux: The Two Things You Need to Know

Everything in Reactor boils down to two types:

- `Mono<T>` - zero or one element. Think of it as a lazy `Optional` that might also fail.
- `Flux<T>` - zero to N elements. A lazy stream that might also fail.

```java
Mono<User> user = userRepository.findById(id);
Flux<Order> orders = orderRepository.findByUserId(id);
```

Nothing happens until someone subscribes. This is the part that trips up every imperative programmer (including me, the first seventeen times). You can chain operators all day long and not a single byte of work gets done until `.subscribe()` is called or Spring WebFlux wires it into the HTTP response.

```java
// This does absolutely nothing
userRepository.findById(id)
    .map(User::getName)
    .flatMap(name -> emailService.send(name));

// You forgot to subscribe. The email never sends.
// Ask me how I know.
```

## The Threading Model (Where the Confusion Lives)

Reactor doesn't magically make things parallel. By default, everything runs on the thread that triggered the subscription. If you call `.subscribe()` from a Tomcat thread, the whole chain runs on that Tomcat thread.

You control threading with Schedulers:

```java
Mono.fromCallable(() -> blockingDatabaseCall())
    .subscribeOn(Schedulers.boundedElastic())  // run on a bounded elastic thread
    .publishOn(Schedulers.parallel())          // switch downstream to parallel scheduler
    .map(this::transformResult);
```

- `Schedulers.parallel()` - fixed pool, sized to CPU cores. For CPU-bound work.
- `Schedulers.boundedElastic()` - grows as needed (capped at 10x cores). For blocking I/O you can't avoid.
- `Schedulers.single()` - one thread. For work that must be sequential.

The distinction between `subscribeOn` and `publishOn` is one of those things that's simple once you get it and maddening until you do. `subscribeOn` affects where the source emits from. `publishOn` affects where downstream operators run. Most of the time, you want `subscribeOn` for wrapping blocking calls and `publishOn` when you need to hop threads mid-chain.

## Spring WebFlux: The Reactor-Powered Web Stack

WebFlux replaces the traditional Servlet-based stack with a non-blocking one built on Netty. Your controllers return `Mono` or `Flux` instead of plain objects:

```java
@GetMapping("/users/{id}")
public Mono<User> getUser(@PathVariable String id) {
    return userService.findById(id);
}

@GetMapping("/users")
public Flux<User> getAllUsers() {
    return userService.findAll();
}
```

The Netty event loop handles I/O with a small number of threads (typically 2x CPU cores). No thread-per-request model. No thread pool sizing. Sounds great on paper.

The catch: *everything* in the chain must be non-blocking. One blocking call on an event loop thread and you've just frozen your entire server. Not one request - all of them. I've seen this happen in production. It's not fun.

## Reactive WebClient

`RestTemplate` is blocking. `WebClient` is reactive. If you're on WebFlux, you need WebClient:

```java
WebClient client = WebClient.builder()
    .baseUrl("https://api.example.com")
    .build();

Mono<User> user = client.get()
    .uri("/users/{id}", userId)
    .retrieve()
    .bodyToMono(User.class);
```

Where WebClient genuinely shines is fan-out patterns - calling multiple services in parallel and combining results:

```java
Mono<OrderDetails> details = Mono.zip(
    orderClient.getOrder(orderId),
    customerClient.getCustomer(customerId),
    inventoryClient.getStock(productId)
).map(tuple -> new OrderDetails(tuple.getT1(), tuple.getT2(), tuple.getT3()));
```

This fires all three requests concurrently and combines the results. With `RestTemplate`, you'd need explicit thread management to do the same thing. Fair enough - that's a genuine win.

## The RxJava Comparison

If you've used RxJava, Reactor will feel familiar. The types map roughly:

| RxJava 3 | Reactor |
|---|---|
| `Single<T>` | `Mono<T>` |
| `Observable<T>` | `Flux<T>` |
| `Completable` | `Mono<Void>` |
| `Maybe<T>` | `Mono<T>` |

Reactor and RxJava 3 both implement the Reactive Streams spec, so they interop through `Publisher`. The practical difference: Reactor is Spring's first-class citizen. If you're in the Spring ecosystem, Reactor is the path of least resistance. If you're on Android or outside Spring, RxJava still has a larger ecosystem.

Reactor's `Context` for propagating metadata through the chain is more ergonomic than RxJava's approach, and Reactor's test utilities (`StepVerifier`) are excellent. But functionally, they solve the same problem.

## Error Handling: The Part Nobody Talks About Enough

Reactive error handling is where the complexity really bites. Exceptions don't propagate normally - they become error signals in the stream:

```java
userRepository.findById(id)
    .switchIfEmpty(Mono.error(new UserNotFoundException(id)))
    .onErrorResume(TimeoutException.class, e -> fallbackService.getCachedUser(id))
    .onErrorMap(DatabaseException.class, e -> new ServiceUnavailableException("DB down", e))
    .doOnError(e -> log.error("Failed to fetch user {}", id, e));
```

Debugging reactive stack traces is a special kind of suffering. The stack trace shows you Reactor internals, not your code. Reactor 3.5+ added better diagnostics with `Hooks.onOperatorDebug()`, but it has a performance cost. In production, use `ReactorDebugAgent` (a Java agent that instruments at class-load time with less overhead).

```java
// In your main method or test setup
ReactorDebugAgent.init();
```

This gives you assembly-time stack traces that actually point to your code. It's not optional. Install it.

## When Reactive Actually Helps

Here's my honest take after building both reactive and imperative services in production:

**Reactive earns its keep when:**
- You have high concurrency with many concurrent connections (thousands+)
- Your workload is I/O-bound with lots of waiting on external calls
- You need backpressure for streaming large datasets
- You're building an API gateway or proxy layer

**Reactive is overhead when:**
- You have a standard CRUD service with moderate traffic
- Your team isn't experienced with reactive patterns
- You're using JDBC (which is blocking, so you're back to `boundedElastic` anyway)
- You value debuggability and simplicity over theoretical throughput

The dirty secret is that for most Spring Boot services, the traditional Servlet stack with virtual threads (Java 21+) gives you similar throughput benefits with none of the reactive complexity. One line in your config versus rewriting your entire codebase. I've written about virtual threads separately, and honestly, for most teams, that's the better answer.

## The Verdict

Project Reactor is a powerful tool that solves a real problem. But it's a *specialized* tool. The reactive programming model adds significant cognitive overhead - composition, threading, error handling, debugging all become harder. That cost is worth paying when you're building a high-throughput gateway or a streaming data pipeline. It's not worth paying for your average order management service.

Learn it. Understand it. But don't reach for it by default. The boring solution is usually the right one.
