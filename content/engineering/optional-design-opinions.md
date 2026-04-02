+++
title = "Optional: Design, Usage, and My Very Strong Opinions"
date = 2026-04-02
[taxonomies]
tags = ["java", "language", "opinion"]
+++

Few things in Java generate more heated arguments than `Optional`. I've seen code reviews devolve into philosophical debates about its use. I've been on both sides. I have opinions now, and they're firm.

## What Optional Is

`Optional<T>` is a container that may or may not hold a value. It was introduced in Java 8, primarily as a return type for methods that might not have a result.

```java
public Optional<User> findByEmail(String email) {
    return Optional.ofNullable(userRepository.findByEmail(email));
}
```

That's the intended use case. A method that might not find something tells you so in its signature. The caller is forced to acknowledge the possibility of absence.

## When to Use It: Return Types

This is the primary and most defensible use of Optional. If a method can legitimately return "nothing," make the return type `Optional<T>`.

```java
public Optional<Order> findById(String id) { ... }
public Optional<BigDecimal> calculateDiscount(Customer customer) { ... }
```

The caller then decides how to handle absence:

```java
Order order = orderService.findById(id)
    .orElseThrow(() -> new OrderNotFoundException(id));

BigDecimal discount = pricingService.calculateDiscount(customer)
    .orElse(BigDecimal.ZERO);
```

This is clean, explicit, and self-documenting. The method signature tells you that absence is a normal case, not an exceptional one.

## When Not to Use It: Parameters

Don't use Optional as a method parameter. I will die on this hill.

```java
// Don't do this
public List<Order> findOrders(Optional<String> customerId, Optional<Status> status) {
    // Now every caller has to wrap their arguments
}

// Do this instead
public List<Order> findOrders(@Nullable String customerId, @Nullable Status status) {
    // Or better: use a query object
}
```

Optional parameters make the calling code ugly. Every caller needs `Optional.of()` or `Optional.empty()`. It adds noise without adding safety. The method already knows which parameters are optional, that's part of its design, not its callers' problem.

Use `@Nullable` annotations, method overloads, or a builder/query object pattern instead.

## When Not to Use It: Fields

Don't use Optional as a field type. It's not serializable (intentionally), it wastes memory (extra object allocation per field), and it fights with every serialization framework.

```java
// Don't do this
public class User {
    private Optional<String> middleName; // Why
}

// Do this
public class User {
    @Nullable
    private String middleName;
}
```

JPA won't map Optional fields. Jackson can handle them with extra configuration, but why make your life harder? A null field with a `@Nullable` annotation conveys the same information with less overhead.

## When Not to Use It: Collections

If your method returns a collection, never wrap it in Optional. An empty collection already represents "no results." `Optional<List<T>>` is redundant and confusing.

```java
// Don't do this
public Optional<List<Order>> findOrders(String customerId) { ... }

// Do this
public List<Order> findOrders(String customerId) {
    // Return empty list if no results
    return Collections.emptyList();
}
```

Same goes for maps, sets, and arrays. Empty collections are the absence indicator. No need for a second layer.

## The "Just Return Null" Crowd

There's a school of thought that says: "We survived 20 years with null returns. Just check for null. Optional is unnecessary complexity."

I understand this position. I disagree with it.

The problem with null returns isn't that they're hard to check. It's that nothing in the type system forces you to check. You call a method, get a reference back, use it, and get a `NullPointerException` at 3 AM. The method signature said `User` - it didn't say "User, but sometimes actually null, and you should check."

Optional makes absence visible in the type system. You can't call `.getName()` on an `Optional<User>` without first unwrapping it. The compiler forces you to acknowledge the possibility.

Is it perfect? No. You can still call `.get()` without checking, and you'll get a `NoSuchElementException` instead of a `NullPointerException`. But the bar for doing the wrong thing is higher.

## The Anti-Patterns

### Optional.get() Without isPresent()

```java
// This defeats the entire purpose of Optional
Optional<User> user = findById(id);
String name = user.get().getName(); // NoSuchElementException if empty
```

Never call `.get()` directly. Use `.orElse()`, `.orElseThrow()`, `.ifPresent()`, or `.map()`.

### Optional for Control Flow

```java
// This is just an if-else with extra steps
Optional.ofNullable(value)
    .map(this::processA)
    .or(() -> Optional.of(defaultValue))
    .map(this::processB)
    .ifPresent(this::save);
```

If your Optional chain is longer than 2-3 operations, you're writing functional code in a language that isn't designed for it. Use an if-else. It's more readable.

### Optional.of() with a Nullable Value

```java
Optional.of(possiblyNull); // NullPointerException if null. Ironic.
Optional.ofNullable(possiblyNull); // This is what you want
```

`Optional.of()` is for when you know the value is non-null and want to lift it into an Optional context. If there's any chance of null, use `ofNullable`.

## The Patterns I Actually Use

### orElseThrow for Required Values

```java
User user = userRepository.findById(id)
    .orElseThrow(() -> new NotFoundException("User not found: " + id));
```

This is my most common pattern. If the value should exist, throw a meaningful exception when it doesn't.

### map for Transformations

```java
Optional<String> displayName = userRepository.findById(id)
    .map(user -> user.firstName() + " " + user.lastName());
```

Clean, no null checks, no intermediate variables.

### flatMap for Chaining Optionals

```java
Optional<Address> address = userRepository.findById(id)
    .flatMap(User::address);  // address() returns Optional<Address>
```

Without `flatMap`, you'd get `Optional<Optional<Address>>`. Nesting Optionals is a code smell - `flatMap` flattens it.

### or() for Fallback Lookups

```java
Optional<Config> config = localConfig.find(key)
    .or(() -> remoteConfig.find(key))
    .or(() -> defaultConfig.find(key));
```

Try local first, then remote, then default. Each lookup is only executed if the previous one was empty.

## Stream Integration

Since Java 9, Optional has a `.stream()` method that returns a zero-or-one element stream:

```java
List<Address> addresses = userIds.stream()
    .map(userRepository::findById)     // Stream<Optional<User>>
    .flatMap(Optional::stream)          // Stream<User>, empties removed
    .map(User::address)
    .flatMap(Optional::stream)          // Stream<Address>, empties removed
    .toList();
```

This is the cleanest way to filter out absent values from a stream of Optionals.

## My Verdict

Use Optional for return types when absence is a normal outcome. Don't use it for parameters, fields, or collections. Keep Optional chains short and readable. Prefer `orElseThrow` and `map` over `isPresent` and `get`.

And if someone tells you Optional is "just like null but with more steps", they're wrong, but don't waste your energy arguing. Just write code that's clear about its intentions and move on.
