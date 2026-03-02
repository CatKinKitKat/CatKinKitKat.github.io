+++
title = "DDD and Event Storming: From Sticky Notes to Bounded Contexts"
date = 2026-03-02
[taxonomies]
tags = ["ddd", "architecture", "patterns"]
+++

I used to think Domain-Driven Design was something consultants talked about to justify expensive engagements. Then I spent six months on a project where five teams were building microservices without agreeing on what "order" meant, and each team had a different Order class with different fields and different lifecycle states. That's when I understood: DDD isn't about fancy patterns. It's about making sure everyone is talking about the same thing. And event storming is how you get there.

## Event Storming: The Workshop That Changes Everything

Event storming was invented by Alberto Brandolini, and the premise is deceptively simple: get domain experts and developers in the same room, hand out sticky notes, and map out everything that happens in the business process.

Here's how I run these sessions:

### Phase 1: Chaotic Exploration

Everyone writes domain events on orange sticky notes. Events are past-tense facts: "Order Placed," "Payment Received," "Shipment Dispatched," "Invoice Generated." No filtering, no ordering, no debating. Just get everything on the wall.

This phase is intentionally messy. The point is to extract knowledge from people's heads. The domain expert who's been handling customer complaints for ten years knows edge cases that no developer has considered. The junior developer asks "what happens if the payment fails?" and suddenly everyone realizes there's no defined process for that.

Give it 30 minutes. You'll have a wall covered in sticky notes. It will look chaotic. That's correct.

### Phase 2: Timeline Ordering

Now arrange the events in chronological order. Left to right, from the first event (customer starts the process) to the last (process complete).

This is where the arguments start, and that's the point. When the warehouse team says "Shipment Dispatched" comes before "Invoice Generated" and the finance team says it's the other way around, you've discovered a domain conflict that would have become a production bug.

Gaps appear too. "Wait, what happens between Payment Received and Order Confirmed? Does someone approve it?" These gaps are where the undocumented business logic lives.

### Phase 3: Commands, Actors, and Aggregates

Introduce more sticky notes:
- **Blue:** Commands - what triggers the event. "Place Order," "Process Payment."
- **Yellow:** Actors - who or what issues the command. A customer, an admin, a scheduled job.
- **Light yellow/pale:** Aggregates - the entity that handles the command and produces the event. The Order, the Payment, the Shipment.

```
[Customer]  -->  [Place Order]  -->  [Order]  -->  [Order Placed]
   Actor          Command          Aggregate        Event
```

This is where DDD concepts start emerging naturally. You don't need to lecture people about aggregates - they discover them by asking "what thing is responsible for this decision?"

### Phase 4: Bounded Contexts

Look at the wall. You'll see clusters of related events and aggregates. There's a cluster around ordering, a cluster around payment, a cluster around shipping, a cluster around inventory. The boundaries between these clusters are your bounded contexts.

This is the money step. These boundaries tell you where your microservice boundaries should be. Not based on technical concerns ("the orders microservice, the users microservice") but based on domain boundaries ("the ordering context, the fulfillment context").

## Reliable Domain Events

Event storming gives you the domain events. Now you need to implement them reliably. "Reliably" means: if the business state changes, the event gets published. Always. No exceptions.

The naive approach fails:

```java
@Transactional
public void placeOrder(PlaceOrderCommand cmd) {
    Order order = Order.place(cmd);
    orderRepository.save(order);
    eventPublisher.publish(new OrderPlacedEvent(order)); // what if this fails?
}
```

If the event publisher fails (broker down, network issue), the order exists but nobody knows about it. The fulfillment context never starts processing. The customer waits forever.

The solution is to make domain events part of the aggregate's state:

```java
public class Order {
    private String id;
    private OrderStatus status;
    private List<DomainEvent> pendingEvents = new ArrayList<>();

    public static Order place(PlaceOrderCommand cmd) {
        Order order = new Order();
        order.id = UUID.randomUUID().toString();
        order.status = OrderStatus.PLACED;
        order.pendingEvents.add(new OrderPlacedEvent(order.id, cmd.items()));
        return order;
    }

    public List<DomainEvent> getPendingEvents() {
        return Collections.unmodifiableList(pendingEvents);
    }

    public void clearPendingEvents() {
        pendingEvents.clear();
    }
}
```

Then use the outbox pattern to ensure events are persisted in the same transaction as the state change:

```java
@Transactional
public void placeOrder(PlaceOrderCommand cmd) {
    Order order = Order.place(cmd);
    orderRepository.save(order);

    for (DomainEvent event : order.getPendingEvents()) {
        outboxRepository.save(new OutboxEntry(event));
    }
    order.clearPendingEvents();
}
```

A separate process (polling or CDC) picks up outbox entries and publishes them to the broker. The domain event is guaranteed to be published because it's written in the same transaction as the business data.

## Bounded Contexts: Where the Real Value Lives

A bounded context is a linguistic boundary. Within a context, every term has exactly one meaning. "Order" in the ordering context means "a customer's intent to purchase." "Order" in the fulfillment context means "a set of items to be picked, packed, and shipped." Same word, different models, different responsibilities.

This is hard for developers who've been taught to eliminate duplication. Having an `Order` class in two different services feels wrong. It's not. It's correct, because these are different concepts that happen to share a name.

The anti-pattern I see constantly: a shared "Order" library used by all services. Every time the ordering context needs a new field, the library gets updated, every service gets redeployed, and the "microservices" are actually a distributed monolith with extra network hops.

### Translating Between Contexts

Contexts communicate through published events, and each context translates incoming events into its own model:

```java
// In the Fulfillment context
@EventHandler
public void on(OrderPlacedEvent event) {
    // Translate from Ordering context's model to Fulfillment context's model
    FulfillmentOrder fulfillment = new FulfillmentOrder();
    fulfillment.setReference(event.getOrderId());
    fulfillment.setItems(event.getItems().stream()
        .map(item -> new FulfillmentItem(item.getSku(), item.getQuantity()))
        .toList());
    fulfillment.setStatus(FulfillmentStatus.PENDING_PICK);
    fulfillmentRepository.save(fulfillment);
}
```

The fulfillment context doesn't care about the customer name, payment method, or order total. It needs SKUs, quantities, and a reference ID. The translation at the boundary keeps each context clean and focused.

This translation layer is what DDD calls an "Anti-Corruption Layer." It protects your domain model from being polluted by another context's model. Without it, every context slowly converges into the same bloated model, which defeats the purpose of having separate contexts.

## How DDD Maps to Microservice Boundaries

The mapping is straightforward but not 1:1.

**One bounded context = one microservice** is the default. Each context has its own deployment, its own database, its own team. This gives you independent deployability and clear ownership.

**One bounded context = multiple microservices** is fine for scaling. If the "Ordering" context has a write-heavy command processing component and a read-heavy query component, split them. But they share the same ubiquitous language and the same team owns both.

**Multiple bounded contexts = one microservice** is a red flag. If one service handles ordering, fulfillment, and billing, you have a monolith wearing a microservice costume. Either the contexts aren't really separate (merge them intentionally) or you haven't finished decomposing (split them).

The event storming wall gives you the map. The clusters of events and aggregates are your bounded contexts. The events that cross cluster boundaries are your integration events - the contracts between services.

## What I Got Wrong

When I first adopted DDD, I made some mistakes that cost time:

**Over-designing aggregates.** I made aggregates too large, trying to enforce consistency across things that didn't need to be consistent within the same transaction. A `Customer` aggregate that included their orders, payments, and preferences. It should have been four separate aggregates communicating through events.

**Ignoring the ubiquitous language.** I used technical names (`PaymentProcessingEntity`) instead of domain names (`Payment`). The domain experts couldn't read the code, which meant they couldn't validate the model. The whole point of DDD is shared language.

**Premature bounded context splitting.** I split contexts too early, before the domain was well understood. This created integration overhead between contexts that should have been one. It's easier to split a monolith than to merge microservices.

**Skipping event storming.** I tried to identify bounded contexts from code analysis and architecture diagrams. It doesn't work. The knowledge lives in people's heads, not in code. Event storming extracts that knowledge. There's no substitute for getting everyone in a room with sticky notes.

## The Bottom Line

DDD and event storming are the most valuable architecture tools I've used - not because they produce clean code (though they help), but because they produce shared understanding. When developers and domain experts agree on what the system does, what the entities are, and where the boundaries lie, everything downstream gets easier. The code, the APIs, the service boundaries, the team structure - they all follow naturally from a well-understood domain model.

The investment is a few days of workshops and some sticky notes. The return is months of avoided miscommunication. That's a trade I'll make every time.
