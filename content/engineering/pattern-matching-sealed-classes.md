+++
title = "Pattern Matching and Sealed Classes: The Visitor Pattern Is Dead"
date = 2026-03-26
[taxonomies]
tags = ["java", "language"]
+++

I wrote my last Visitor pattern implementation in 2023. I'm not going back. Java's pattern matching and sealed classes killed it, and I'm here to celebrate the funeral.

## The Old Pain

The Visitor pattern exists because Java (historically) had no good way to dispatch behavior based on the runtime type of an object without a chain of `instanceof` checks. So you'd create an interface with an `accept` method, a visitor interface with a `visit` overload for every type, and wire them together with double dispatch.

It worked. It was also 200 lines of boilerplate for what amounts to "do different things depending on the type." Every new type required changes in the visitor interface, every implementation of that interface, and the accept method. It was maintainable in the same way a house of cards is stable - technically true until someone sneezes.

## instanceof Patterns

Java 16 gave us the first piece: pattern matching for `instanceof`.

```java
// Before
if (obj instanceof String) {
    String s = (String) obj;
    System.out.println(s.length());
}

// After
if (obj instanceof String s) {
    System.out.println(s.length());
}
```

Small improvement, but it set the stage. The variable `s` is scoped to the `if` block, and the compiler knows it's a `String`. No cast needed.

The flow scoping is smarter than you'd expect:

```java
if (!(obj instanceof String s)) {
    return;
}
// s is in scope here - the compiler knows the early return means obj must be a String
s.toLowerCase();
```

## Switch Expressions with Patterns

This is where the real power is. Java 21 finalized pattern matching for switch:

```java
public double calculateShippingCost(Package pkg) {
    return switch (pkg) {
        case Letter l -> l.weight() * 0.50;
        case Parcel p when p.weight() > 30 -> p.weight() * 2.00 + 15.00;
        case Parcel p -> p.weight() * 1.50;
        case OversizedItem o -> throw new UnsupportedOperationException("Call freight");
    };
}
```

Guards (`when` clauses) let you add conditions without nested `if` statements. The ordering matters - more specific patterns must come before general ones, just like catch blocks.

## Sealed Interfaces: Closing the Hierarchy

A sealed interface declares exactly which classes can implement it:

```java
public sealed interface Shape
    permits Circle, Rectangle, Triangle {}

public record Circle(double radius) implements Shape {}
public record Rectangle(double width, double height) implements Shape {}
public record Triangle(double base, double height) implements Shape {}
```

The `permits` clause is the contract. No class outside this list can implement `Shape`. The compiler knows the full universe of subtypes.

If all the permitted subtypes are in the same file, you can omit `permits` and the compiler infers it:

```java
public sealed interface Shape {
    record Circle(double radius) implements Shape {}
    record Rectangle(double width, double height) implements Shape {}
    record Triangle(double base, double height) implements Shape {}
}
```

I prefer this style. Everything in one place, no hunting across files.

## Exhaustiveness: The Compiler Is Your Reviewer

The real payoff of sealed types is exhaustiveness checking in switch expressions:

```java
public double area(Shape shape) {
    return switch (shape) {
        case Circle c -> Math.PI * c.radius() * c.radius();
        case Rectangle r -> r.width() * r.height();
        // Compiler error: 'switch' expression does not cover all possible input values
        // Missing: Triangle
    };
}
```

Add a new subtype to the sealed hierarchy? Every switch expression that handles it will fail to compile. This is infinitely better than the Visitor pattern's runtime failures or the `default` branch that silently swallows new types.

This is why I say the Visitor pattern is dead. The compiler does what the Visitor pattern did - ensure you handle every type - but without any of the ceremony.

## Nested Patterns and Record Deconstruction

You can pattern match into nested records:

```java
sealed interface Expr {
    record Num(double value) implements Expr {}
    record Add(Expr left, Expr right) implements Expr {}
    record Mul(Expr left, Expr right) implements Expr {}
}

public double eval(Expr expr) {
    return switch (expr) {
        case Num(var v) -> v;
        case Add(var l, var r) -> eval(l) + eval(r);
        case Mul(var l, var r) -> eval(l) * eval(r);
    };
}
```

This is a textbook expression evaluator in about 10 lines. Try doing this with the Visitor pattern. I'll wait.

You can go deeper with nested deconstruction:

```java
case Add(Num(var l), Num(var r)) -> l + r;  // Optimize: direct addition for two numbers
case Add(var l, var r) -> eval(l) + eval(r); // General case
```

## The Unnamed Pattern

Java 22 introduced the unnamed pattern (`_`) for components you don't care about:

```java
case Declined(var reason, _) -> "Declined: " + reason;
case GatewayError(_, var retryable) -> retryable ? "Will retry" : "Giving up";
```

Small thing, but it makes intent clear: "I know this component exists, I don't need it here."

## Real-World Example: Command Handling

Here's how I actually use this in production. A command handler for an API:

```java
public sealed interface Command {
    record CreateOrder(String customerId, List<LineItem> items) implements Command {}
    record CancelOrder(String orderId, String reason) implements Command {}
    record UpdateShipping(String orderId, Address newAddress) implements Command {}
}

public CommandResult handle(Command command) {
    return switch (command) {
        case CreateOrder cmd -> orderService.create(cmd.customerId(), cmd.items());
        case CancelOrder cmd -> orderService.cancel(cmd.orderId(), cmd.reason());
        case UpdateShipping cmd -> shippingService.updateAddress(cmd.orderId(), cmd.newAddress());
    };
}
```

Clean, exhaustive, type-safe. When the product team adds a new command type, the compiler tells me exactly where to handle it.

## What's Left of the Visitor?

Honestly, nothing. Every use case I had for the Visitor pattern is better served by sealed types + pattern matching:

- **Type-safe dispatch** - switch expressions with exhaustiveness
- **Separation of data and behavior** - records for data, external functions for behavior
- **Extensibility** - add a new sealed subtype and the compiler guides you

The Visitor pattern was a workaround for a language limitation. That limitation is gone. Let it go.

## Migration Tip

If you have existing Visitor implementations, migrate bottom-up. First convert the visited types to a sealed hierarchy. Then replace each Visitor implementation with a switch expression. Delete the accept methods last. Each step compiles and works independently.

Your IDE's "find usages" on the Visitor interface will show you every place that needs conversion. It's mechanical, it's boring, and it makes the code dramatically better.
