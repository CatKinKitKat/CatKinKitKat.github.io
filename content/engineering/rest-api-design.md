+++
title = "REST API Design: Pragmatism Over Purism"
date = 2025-08-15
[taxonomies]
tags = ["api", "rest", "spring-boot"]
+++

I once worked on a team that spent two weeks debating whether updating a resource should return 200 or 204. Two weeks. The API had three endpoints. We shipped nothing during those two weeks, but at least our status codes were theoretically correct.

REST API design is full of religious debates that don't matter in practice. What matters: your API is consistent, well-documented, easy to use correctly, and hard to use incorrectly. Everything else is bike-shedding.

## Versioning: Pick One and Commit

There are three approaches, and people will argue about all of them.

**URL versioning**: `/api/v1/orders`. Simple. Visible. Every developer understands it immediately. The "it's not RESTful" crowd hates it because the resource URI shouldn't change between versions.

**Header versioning**: `Accept: application/vnd.myapp.v1+json`. "Pure" REST. The URI represents the resource, the version is in content negotiation. In practice, it's harder to test (you can't just paste a URL in a browser), harder to cache, and harder to explain to frontend developers.

**Query parameter versioning**: `/api/orders?version=1`. Nobody's favorite but everyone can use it.

I use URL versioning. It's explicit, easy to test, easy to route in API gateways, and nobody has ever been confused by it. Purity is not a feature.

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderControllerV1 {
    // v1 implementation
}

@RestController
@RequestMapping("/api/v2/orders")
public class OrderControllerV2 {
    // v2 with breaking changes
}
```

The real rule: avoid breaking changes. Versioning is for when you absolutely must. Adding fields to responses isn't a breaking change. Removing fields is. Changing the type of a field is. If you design your API to be additive, you rarely need a new version.

## Content Negotiation

Support JSON. That's it for 99% of APIs.

If you genuinely need multiple formats, Spring makes it easy:

```java
@GetMapping(value = "/{id}", produces = {
    MediaType.APPLICATION_JSON_VALUE,
    MediaType.APPLICATION_XML_VALUE
})
public Order getOrder(@PathVariable String id) {
    return orderService.findById(id);
}
```

But be honest: are your clients actually requesting XML? If not, don't support it. Every format you support is a format you have to test and maintain.

## HATEOAS: The Theory Nobody Implements

HATEOAS (Hypermedia as the Engine of Application State) says API responses should include links that tell the client what actions are available:

```json
{
  "id": "order-123",
  "status": "SUBMITTED",
  "_links": {
    "self": { "href": "/api/v1/orders/order-123" },
    "cancel": { "href": "/api/v1/orders/order-123/cancel" },
    "payment": { "href": "/api/v1/orders/order-123/payment" }
  }
}
```

Spring HATEOAS supports this:

```java
@GetMapping("/{id}")
public EntityModel<Order> getOrder(@PathVariable String id) {
    Order order = orderService.findById(id);
    EntityModel<Order> model = EntityModel.of(order);

    model.add(linkTo(methodOn(OrderController.class).getOrder(id)).withSelfRel());

    if (order.canCancel()) {
        model.add(linkTo(methodOn(OrderController.class).cancelOrder(id)).withRel("cancel"));
    }

    return model;
}
```

In theory, the client discovers available actions from the links instead of hardcoding URLs. In practice, your React frontend has hardcoded URLs, the mobile app has hardcoded URLs, and the integration partner has hardcoded URLs. Nobody follows the links.

I've implemented HATEOAS once for a project that required it. The client developers ignored the links and hardcoded the URLs anyway. That tells you everything.

HATEOAS is a good idea if you're building a truly generic API where clients discover capabilities dynamically. For specific, known clients, it's overhead.

## Error Handling: Be Helpful

This is bad:

```json
{ "error": "Something went wrong" }
```

This is better:

```json
{
  "type": "VALIDATION_ERROR",
  "title": "Validation failed",
  "status": 400,
  "detail": "Request body contains invalid fields",
  "errors": [
    {
      "field": "email",
      "message": "must be a valid email address",
      "rejectedValue": "not-an-email"
    },
    {
      "field": "quantity",
      "message": "must be greater than 0",
      "rejectedValue": -1
    }
  ]
}
```

Use RFC 7807 (Problem Details for HTTP APIs). Spring Boot 3 supports it natively:

```java
@RestControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(OrderNotFoundException.class)
    public ProblemDetail handleNotFound(OrderNotFoundException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setTitle("Order not found");
        problem.setProperty("orderId", ex.getOrderId());
        return problem;
    }

    @ExceptionHandler(ConstraintViolationException.class)
    public ProblemDetail handleValidation(ConstraintViolationException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.BAD_REQUEST, "Validation failed");
        problem.setTitle("Validation error");

        var errors = ex.getConstraintViolations().stream()
            .map(v -> Map.of(
                "field", v.getPropertyPath().toString(),
                "message", v.getMessage()))
            .toList();
        problem.setProperty("errors", errors);
        return problem;
    }
}
```

Consistent error structure across all endpoints. The client knows exactly what to parse. Field-level errors for validation so the UI can highlight the right input.

## OpenAPI / Swagger Documentation

Document your API with OpenAPI. Spring Boot + springdoc-openapi makes this nearly automatic:

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.3.0</version>
</dependency>
```

Add annotations where the defaults aren't sufficient:

```java
@Operation(summary = "Create a new order",
           description = "Creates an order and initiates payment processing")
@ApiResponse(responseCode = "201", description = "Order created")
@ApiResponse(responseCode = "400", description = "Invalid request")
@ApiResponse(responseCode = "409", description = "Insufficient stock")
@PostMapping
public ResponseEntity<OrderResponse> create(@RequestBody @Valid OrderRequest request) {
    // ...
}
```

Swagger UI at `/swagger-ui.html` gives you interactive documentation that stays in sync with the code. It's not a substitute for written docs, but it's a baseline that's always accurate.

The rule: if the OpenAPI spec doesn't match the implementation, the spec is a liability. Auto-generation from code ensures consistency.

## Pragmatic REST

Here are my practical conventions:

**Nouns for resources**: `/orders`, `/orders/{id}`, `/orders/{id}/items`. Not `/getOrder` or `/createOrder`.

**HTTP methods for actions**: GET reads, POST creates, PUT replaces, PATCH updates partially, DELETE deletes. Don't use POST for everything.

**Plural nouns**: `/orders`, not `/order`. Consistent across all resources.

**Filtering via query params**: `GET /orders?status=SUBMITTED&customerId=abc123`.

**Pagination**: `GET /orders?page=0&size=20`. Return total count in a header or wrapper object.

**Status codes that make sense**: 200 for success, 201 for creation, 204 for deletion, 400 for bad input, 404 for not found, 409 for conflicts, 500 for server errors. Don't overthink it.

**Idempotency**: PUT and DELETE should be idempotent. POST should be, if possible (use an idempotency key header).

None of this is controversial. None of it is pure REST according to Roy Fielding's dissertation. All of it works in practice, which is the only thing that actually matters.
