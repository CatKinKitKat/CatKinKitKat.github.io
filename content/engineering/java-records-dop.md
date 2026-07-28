+++
title = "Java Records and the Data-Oriented Programming Shift"
date = 2026-03-25
[taxonomies]
tags = ["java", "language"]
+++

I spent years writing Java classes that existed solely to hold data. Getters, setters, equals, hashCode, toString - the whole ceremony. Lombok helped, but it was always a band-aid over the fact that Java had no first-class concept of "this is just data." Records changed that.

## The Basics You Already Know

```java
public record OrderSummary(String orderId, BigDecimal total, Instant createdAt) {}
```

That's it. You get the constructor, accessors, equals, hashCode, and toString for free. They're final, immutable, and transparent. The compiler generates everything based on the component list in the header.

What I didn't appreciate at first: records aren't just "less boilerplate." They signal intent. When I see a record, I know it's data. No hidden state, no mutable surprises, no lifecycle weirdness. It's a value.

## Record Design That Doesn't Annoy Your Team

### Compact Constructors for Validation

```java
public record Email(String value) {
    public Email {
        if (value == null || !value.contains("@")) {
            throw new IllegalArgumentException("Invalid email: " + value);
        }
        value = value.trim().toLowerCase();
    }
}
```

The compact constructor has no parameter list - it implicitly takes the record components. You can validate and normalize, and the assignment to the fields happens automatically at the end. This is where I put all my domain validation now. The record becomes a validated value type that can't exist in an invalid state.

### Records as Local Types

This is underrated. You can declare a record inside a method:

```java
public List<String> topSpenders(List<Order> orders) {
    record CustomerTotal(String customerId, BigDecimal total) {}

    return orders.stream()
        .collect(Collectors.groupingBy(
            Order::customerId,
            Collectors.reducing(BigDecimal.ZERO, Order::total, BigDecimal::add)))
        .entrySet().stream()
        .map(e -> new CustomerTotal(e.getKey(), e.getValue()))
        .sorted(Comparator.comparing(CustomerTotal::total).reversed())
        .limit(10)
        .map(CustomerTotal::customerId)
        .toList();
}
```

No more anonymous Pair or Tuple classes polluting your codebase. Define the shape of the data where you need it.

## Sealed Hierarchies: Records Get Interesting

Records shine when combined with sealed interfaces. This is where Data-Oriented Programming starts to click.

```java
public sealed interface PaymentResult {
    record Success(String transactionId, Instant processedAt) implements PaymentResult {}
    record Declined(String reason, String code) implements PaymentResult {}
    record GatewayError(Exception cause, boolean retryable) implements PaymentResult {}
}
```

The `sealed` keyword means only these three records can implement `PaymentResult`. The compiler knows the full set. No rogue subclass in some other package can break your assumptions.

## Pattern Matching With Records

Here's where it all comes together. Deconstruction patterns let you pull records apart in switch expressions:

```java
public String describeResult(PaymentResult result) {
    return switch (result) {
        case Success(var txId, var time) ->
            "Payment %s processed at %s".formatted(txId, time);
        case Declined(var reason, _) ->
            "Payment declined: " + reason;
        case GatewayError(var ex, var retryable) ->
            retryable ? "Transient error, retrying" : "Fatal: " + ex.getMessage();
    };
}
```

The compiler enforces exhaustiveness. If you add a new variant to `PaymentResult`, every switch that handles it will fail to compile until you handle the new case. Compare that to the old approach: an abstract class with `instanceof` checks scattered across the codebase, and good luck finding them all when you add a new subtype.

## Data-Oriented Programming: The Paradigm

DOP isn't some academic exercise. The idea is simple: model your domain as plain immutable data (records), define the possible shapes with sealed types, and use pattern matching to process them. Behavior lives in functions that operate on data, not in methods attached to the data itself.

The OOP instinct is to put behavior inside the data class. DOP says: keep the data clean, put the behavior where it's used.

```java
// OOP instinct
public record Order(String id, List<LineItem> items) {
    public BigDecimal total() { /* logic here */ }
}

// DOP approach
public class Pricing {
    public static BigDecimal total(Order order) { /* logic here */ }
    public static BigDecimal totalWithDiscount(Order order, Discount d) { /* logic here */ }
}
```

The DOP version is easier to test, easier to extend, and doesn't force pricing logic into the Order type. When you have five different ways to calculate a total depending on context, the OOP approach collapses into a mess of method overloads or strategy objects. The DOP approach is just... functions.

## When Records Are Wrong
Records are not entities. They don't have identity beyond their component values. Two records with the same components are equal. Period. If you need mutable state, lifecycle hooks, or JPA identity: records are not your tool.

## The Design Philosophy
I've seen people try to use records as JPA entities. Don't. Hibernate needs a no-arg constructor, mutable fields for dirty checking, and proxy support. Records provide none of that. Use records for DTOs, value objects, API responses, and event payloads. Use regular classes for entities.

## The Shift

Before records and sealed types, modeling a domain in Java meant choosing between "proper OOP" (class hierarchies, visitor pattern, abstract methods) and "just use maps and strings." Both were bad in different ways.

Records, sealed interfaces, and pattern matching give you a third option that's actually good. Your data model becomes a type-safe algebraic data type. The compiler checks your work. And you wrote about 70% less code to get there.

I've been slowly converting DTOs, event types, and command objects to records across our services. Every time I do, the code gets shorter and clearer. No Lombok, no boilerplate, no ceremony. Just data.
