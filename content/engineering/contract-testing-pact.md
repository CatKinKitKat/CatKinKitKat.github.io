+++
title = "Contract Testing with Pact, or How to Stop Breaking Each Other's Services"
date = 2025-11-10
[taxonomies]
tags = ["testing", "microservices", "spring-boot"]
+++

You know the drill. Your service works perfectly in isolation. You deploy it. Fifteen minutes later someone from another team messages you: "Hey, your API response changed and our parsing broke." You check. You renamed a field from `orderId` to `order_id` because you were doing a naming convention cleanup. Nobody told you that three other teams depended on that exact field name.

This is what contract testing solves. And Pact is the tool I've had the most success with.

## What Contract Testing Actually Is

Integration tests verify that your service works with its dependencies. Contract tests verify that services *agree* on the shape of their communication. Different problem, different solution.

A contract is the agreed-upon interface between a consumer and a provider. The consumer says "I expect this request to return these fields in this shape." The provider says "I will always return these fields in this shape." If either side breaks the agreement, the test fails.

The key insight: you don't need both services running at the same time to verify this. The consumer generates a contract (a Pact file). The provider verifies it independently. No complex integration environments needed.

## Consumer-Driven Contracts

Pact follows the consumer-driven model. The consumer defines what it needs from the provider, not the other way around. This is the right way to do it, because the consumer knows what fields it actually uses. The provider might return 40 fields, but if the consumer only uses 5, the contract only covers those 5.

Here's a consumer test in a Spring Boot service:

```java
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "order-service", port = "8080")
class OrderClientPactTest {

    @Pact(provider = "order-service", consumer = "shipping-service")
    public V4Pact createPact(PactDslWithProvider builder) {
        return builder
            .given("an order with ID 123 exists")
            .uponReceiving("a request for order 123")
            .path("/orders/123")
            .method("GET")
            .willRespondWith()
            .status(200)
            .body(newJsonBody(body -> {
                body.stringType("orderId", "123");
                body.stringType("status", "SHIPPED");
                body.numberType("amount", 99.99);
            }).build())
            .toPact(V4Pact.class);
    }

    @Test
    @PactTestFor(pactMethod = "createPact")
    void testGetOrder(MockServer mockServer) {
        OrderClient client = new OrderClient(mockServer.getUrl());
        Order order = client.getOrder("123");
        assertThat(order.getStatus()).isEqualTo("SHIPPED");
    }
}
```

This generates a Pact file - a JSON contract that describes the interaction. You publish it to a Pact Broker (or Pactflow if you want the hosted version).

## 7 Reasons to Actually Use This

1. **You catch breaking changes before deployment.** The provider verification runs in CI. If someone renames a field, the build fails.

2. **You stop maintaining shared integration environments.** Those "staging" environments that are always broken because three teams deployed incompatible versions? Gone.

3. **You test exactly what you use.** No over-specifying. If you don't use a field, you don't test it, and the provider is free to change it.

4. **You get documentation for free.** The Pact Broker shows you which services talk to each other and what the contracts look like.

5. **You can deploy independently.** The "can I deploy" check tells you whether your version is compatible with what's currently in production.

6. **You reduce the feedback loop.** Contract tests run in seconds. Integration tests against real services take minutes (if the environment is even available).

7. **You make API evolution explicit.** Want to deprecate a field? Check the broker. If no consumer uses it, remove it. If three consumers use it, talk to them first.

## Provider Verification

On the provider side, you verify against the published contracts:

```java
@Provider("order-service")
@PactBroker(url = "https://your-broker.pactflow.io")
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OrderProviderPactVerificationTest {

    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void verifyPact(PactVerificationContext context) {
        context.verifyInteraction();
    }

    @State("an order with ID 123 exists")
    void setupOrder() {
        orderRepository.save(new Order("123", "SHIPPED", 99.99));
    }
}
```

The `@State` annotations set up the preconditions. When the consumer says "given an order with ID 123 exists," the provider test creates that state. This is where people trip up - your state setup needs to be reliable and fast.

## Contract Testing Event-Driven Systems

REST APIs are straightforward. Kafka messages are where it gets interesting.

Pact supports message-based contracts. The consumer defines what message shape it expects:

```java
@Pact(provider = "order-service", consumer = "notification-service")
public MessagePact orderCreatedPact(MessagePactBuilder builder) {
    return builder
        .expectsToReceive("an order created event")
        .withContent(newJsonBody(body -> {
            body.stringType("orderId", "456");
            body.stringType("customerEmail", "test@example.com");
            body.stringType("eventType", "ORDER_CREATED");
        }).build())
        .toPact();
}
```

The provider verifies it can produce a message matching that contract:

```java
@PactVerifyProvider("an order created event")
public String verifyOrderCreatedEvent() {
    OrderCreatedEvent event = new OrderCreatedEvent("456", "test@example.com");
    return objectMapper.writeValueAsString(event);
}
```

No Kafka broker needed. No test containers. Just "does the shape match?" This is dramatically simpler than running a full Kafka cluster in your test suite.

## Microcks for API Contract Testing

If you're already invested in OpenAPI specs, Microcks is worth looking at. It takes your API specs (OpenAPI, AsyncAPI, gRPC protobuf) and generates mock services and contract tests from them.

The workflow is different from Pact. Instead of consumer-driven contracts, you start with the API spec as the source of truth. Microcks can:

- Mock the API based on the spec (useful for frontend teams)
- Run contract tests against the real implementation to verify it matches the spec
- Handle both sync (REST/gRPC) and async (Kafka/AMQP) APIs

I've used Microcks alongside Pact. Microcks catches drift between your spec and your implementation. Pact catches drift between what the consumer needs and what the provider delivers. They solve different problems.

## The Pact Broker is Non-Negotiable

Don't try to share Pact files through Git repositories or shared drives. Use a Pact Broker. It gives you:

- A central registry of all contracts
- The "can I deploy" feature that checks compatibility
- Webhooks that trigger provider verification when a consumer publishes a new contract
- Versioning and tagging

Without the broker, you're just swapping JSON files around and hoping everyone stays in sync. With the broker, you have an actual system that prevents incompatible deployments.

## When Contract Testing Falls Short

Contract testing verifies structure, not business logic. It tells you "yes, the response has an `amount` field that's a number." It doesn't tell you "the amount is calculated correctly." You still need integration tests and end-to-end tests for that.

It also doesn't help with performance. Your contract might be satisfied, but if the provider starts taking 30 seconds to respond, that's a different kind of breakage.

Contract testing is one layer. A very useful layer. But it's not the only layer. Combine it with Testcontainers for integration testing and Gatling for load testing, and you've got a setup that catches most categories of failure before they reach production.
