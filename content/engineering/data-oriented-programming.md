+++
title = "Data-Oriented Programming in Java: Records, Sealed Types, and Letting Go of OOP"
date = 2025-08-05
[taxonomies]
tags = ["java", "architecture", "patterns"]
+++

I spent years writing Java the "proper" OOP way. Everything was a class with private fields, getters, setters, behavior methods, and inheritance hierarchies three levels deep. Then Java 21 gave us records, sealed types, and pattern matching, and I started questioning everything I knew about Java design.

Data-Oriented Programming (DOP) is the idea that data and behavior should be separate. Data is plain, transparent, immutable. Behavior operates on data but doesn't live inside it. This is the opposite of classic OOP, where data and behavior are encapsulated together.

Java finally has the language features to make this practical.

## Records: Data as Data

A record is a transparent carrier of data. No hidden state, no behavior surprises:

```java
public record Order(
    String id,
    String customerId,
    List<OrderItem> items,
    OrderStatus status,
    Instant createdAt
) {
    public Order {
        Objects.requireNonNull(id);
        Objects.requireNonNull(customerId);
        items = List.copyOf(items); // defensive copy, immutable
    }

    public Money totalAmount() {
        return items.stream()
            .map(OrderItem::lineTotal)
            .reduce(Money.ZERO, Money::add);
    }
}

public record OrderItem(String productId, int quantity, Money unitPrice) {
    public Money lineTotal() {
        return unitPrice.multiply(quantity);
    }
}

public record Money(BigDecimal amount, Currency currency) {
    public Money add(Money other) {
        if (!currency.equals(other.currency)) {
            throw new IllegalArgumentException("Cannot add different currencies");
        }
        return new Money(amount.add(other.amount), currency);
    }

    public Money multiply(int factor) {
        return new Money(amount.multiply(BigDecimal.valueOf(factor)), currency);
    }

    public static final Money ZERO = new Money(BigDecimal.ZERO, Currency.getInstance("EUR"));
}
```

Records give you `equals()`, `hashCode()`, `toString()`, immutability (if you enforce it in the compact constructor), and transparent access to components. No boilerplate. No Lombok.

The computed methods (`totalAmount()`, `lineTotal()`) are derived from the data. They don't mutate state. They're pure functions that happen to live on the data type. This is a pragmatic middle ground between "no behavior on data" and "everything is a method."

## Sealed Types: Modeling Alternatives

Sealed types restrict which classes can extend a type. Combined with records, they model algebraic data types:

```java
public sealed interface PaymentResult
    permits PaymentSuccess, PaymentFailure, PaymentPending {
}

public record PaymentSuccess(String transactionId, Instant processedAt)
    implements PaymentResult {}

public record PaymentFailure(String errorCode, String reason)
    implements PaymentResult {}

public record PaymentPending(String pendingId, Duration estimatedWait)
    implements PaymentResult {}
```

The compiler knows every possible variant. No surprise subclasses at runtime. This is a closed domain model - the set of payment results is fixed and exhaustive.

## Pattern Matching: Operating on Data

Pattern matching with `switch` lets you operate on sealed types exhaustively:

```java
public String handlePayment(PaymentResult result) {
    return switch (result) {
        case PaymentSuccess s ->
            "Payment processed: " + s.transactionId();
        case PaymentFailure f ->
            "Payment failed: " + f.reason();
        case PaymentPending p ->
            "Payment pending, estimated wait: " + p.estimatedWait();
    };
}
```

No `instanceof` chains. No visitor pattern. The compiler ensures you handle every case. If you add a new variant to `PaymentResult`, every switch that doesn't handle it becomes a compile error. This is better than any runtime check.

Nested pattern matching with record patterns:

```java
public String describeOrder(Order order) {
    return switch (order.status()) {
        case DRAFT -> "Draft order with " + order.items().size() + " items";
        case SUBMITTED -> "Submitted, total: " + order.totalAmount();
        case PAID -> "Paid order " + order.id();
        case SHIPPED -> "Shipped to " + order.customerId();
        case CANCELLED -> "Cancelled order";
    };
}
```

## Separating Data from Behavior

In classic OOP, you'd put business logic inside the `Order` class. In DOP, the `Order` is just data. Business logic lives in separate functions:

```java
public class OrderOperations {

    public static Order addItem(Order order, OrderItem item) {
        var newItems = new ArrayList<>(order.items());
        newItems.add(item);
        return new Order(order.id(), order.customerId(),
            newItems, order.status(), order.createdAt());
    }

    public static Order submit(Order order) {
        if (order.items().isEmpty()) {
            throw new IllegalStateException("Cannot submit empty order");
        }
        return new Order(order.id(), order.customerId(),
            order.items(), OrderStatus.SUBMITTED, order.createdAt());
    }

    public static boolean canCancel(Order order) {
        return order.status() == OrderStatus.DRAFT
            || order.status() == OrderStatus.SUBMITTED;
    }
}
```

Every operation returns a new `Order` instead of mutating the existing one. State changes are explicit. You can see exactly what changed by comparing the old and new objects. Testing is trivial - no mocks, no setup, just create data and assert on results.

## The BCE Architecture

Boundary-Control-Entity (BCE) maps naturally to DOP:

**Entity**: Records and sealed types. Pure data with validation. No framework dependencies.

```java
public record Customer(String id, String name, String email, CustomerTier tier) {}

public sealed interface CustomerTier
    permits StandardTier, PremiumTier, EnterpriseTier {}

public record StandardTier() implements CustomerTier {}
public record PremiumTier(LocalDate since) implements CustomerTier {}
public record EnterpriseTier(String accountManager, BigDecimal creditLimit)
    implements CustomerTier {}
```

**Control**: Services that operate on entities. Business logic and orchestration.

```java
public class PricingControl {

    public Money calculateDiscount(Order order, Customer customer) {
        Money total = order.totalAmount();
        BigDecimal discountRate = switch (customer.tier()) {
            case StandardTier s -> BigDecimal.ZERO;
            case PremiumTier p -> new BigDecimal("0.10");
            case EnterpriseTier e -> new BigDecimal("0.20");
        };
        return new Money(total.amount().multiply(discountRate), total.currency());
    }
}
```

**Boundary**: Adapters for external systems. HTTP controllers, repositories, message consumers.

BCE + DOP gives you a clean architecture where data flows through boundaries, gets processed by controls, and the entities are just data. No inheritance hierarchies. No abstract classes with protected methods. Just data, functions, and clear flow.

## DOP vs OOP: The Honest Comparison

OOP excels at modeling things with complex internal behavior - a connection pool, a thread scheduler, a state machine with many transition rules. Encapsulation is genuinely useful when the internal state is complex and invariants must be maintained.

DOP excels at modeling data that flows through a system - requests, events, domain objects, DTOs. When your objects are mostly data with some derived computations, records are cleaner than classes with getters and setters.

Most Java applications have both: infrastructure objects that benefit from OOP encapsulation, and domain data that benefits from DOP transparency. The mistake is applying one paradigm everywhere.

In my services, the domain model is DOP (records, sealed types, pattern matching). The infrastructure is OOP (Spring beans, repository implementations, connection management). They coexist naturally because Java supports both.

## The Practical Impact

Since adopting DOP for domain modeling, my code has become:

- **More testable**: Records are easy to construct. No mocking needed for data.
- **More readable**: Pattern matching makes branching logic explicit.
- **Safer**: Sealed types catch missing cases at compile time.
- **Simpler**: No getter/setter ceremony. No builder pattern for simple data.

The trade-off: immutability means more object creation. For hot paths, this can matter. For 99% of business logic, it doesn't. The JVM's garbage collector is extraordinarily good at handling short-lived objects.

Java took 30 years to get here, but records, sealed types, and pattern matching make data-oriented programming a first-class approach. It's not replacing OOP. It's complementing it, in the places where OOP was always a poor fit.
