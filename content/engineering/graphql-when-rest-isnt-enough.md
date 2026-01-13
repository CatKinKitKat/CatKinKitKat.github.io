+++
title = "GraphQL: When REST Isn't Enough (And When It Is)"
date = 2025-08-25
[taxonomies]
tags = ["api", "graphql", "spring-boot"]
+++

I adopted GraphQL for a project because the frontend team was making 6 REST calls to render a single page. The "Backend for Frontend" API was just stitching together responses from other services and the frontend still needed to make multiple round trips. GraphQL solved that problem beautifully. Then I tried to use it for a simple CRUD service and it was like swatting a fly with a sledgehammer.

GraphQL is a tool. A specific tool for a specific problem. That problem is: the client needs flexible data fetching across multiple related resources. If that's your problem, GraphQL is excellent. If it's not, REST is simpler.

## The Query Language

GraphQL lets clients specify exactly what data they need:

```graphql
query {
  order(id: "order-123") {
    id
    status
    createdAt
    customer {
      name
      email
    }
    items {
      product {
        name
        price
      }
      quantity
    }
    totalAmount
  }
}
```

One request. The client gets exactly the fields it asked for, with nested related objects. No over-fetching (getting fields you don't need). No under-fetching (needing a second request for related data).

Compare with REST: `GET /orders/123` returns the order. Then `GET /customers/{customerId}` for the customer. Then `GET /products/{productId}` for each product. Three or more round trips, each returning fields the client might not need.

## Spring Boot GraphQL with Netflix DGS

Netflix DGS is the most complete GraphQL framework for Spring Boot. It gives you schema-first development, code generation, data loaders, and federation support.

Schema first:

```graphql
type Query {
    order(id: ID!): Order
    orders(status: OrderStatus, page: Int, size: Int): OrderConnection!
}

type Order {
    id: ID!
    status: OrderStatus!
    customer: Customer!
    items: [OrderItem!]!
    totalAmount: Money!
    createdAt: DateTime!
}

type Customer {
    id: ID!
    name: String!
    email: String!
}

type OrderItem {
    product: Product!
    quantity: Int!
    lineTotal: Money!
}

type Money {
    amount: BigDecimal!
    currency: String!
}

enum OrderStatus {
    DRAFT
    SUBMITTED
    PAID
    SHIPPED
    CANCELLED
}
```

Data fetcher (DGS resolver):

```java
@DgsComponent
public class OrderDataFetcher {

    private final OrderService orderService;

    @DgsQuery
    public Order order(@InputArgument String id) {
        return orderService.findById(id);
    }

    @DgsQuery
    public Connection<Order> orders(
            @InputArgument OrderStatus status,
            @InputArgument Integer page,
            @InputArgument Integer size) {
        // return paginated results
    }

    @DgsData(parentType = "Order", field = "customer")
    public Customer customer(DgsDataFetchingEnvironment dfe) {
        Order order = dfe.getSource();
        return customerService.findById(order.customerId());
    }
}
```

The schema defines the API contract. The fetchers implement the resolution logic. DGS generates Java types from the schema, so you get type safety without manually writing DTOs.

## The N+1 Problem and DataLoader

Here's where GraphQL bites you if you're not careful. Say a client queries 20 orders with their customers:

```graphql
query {
  orders(page: 0, size: 20) {
    edges {
      node {
        id
        customer {
          name
        }
      }
    }
  }
}
```

Without optimization, this executes: 1 query for 20 orders + 20 queries for 20 customers = 21 database queries. Classic N+1.

DataLoader batches these:

```java
@DgsDataLoader(name = "customers")
public class CustomerDataLoader implements BatchLoader<String, Customer> {

    private final CustomerService customerService;

    @Override
    public CompletionStage<List<Customer>> load(List<String> customerIds) {
        return CompletableFuture.supplyAsync(
            () -> customerService.findByIds(customerIds));
    }
}

@DgsData(parentType = "Order", field = "customer")
public CompletableFuture<Customer> customer(DgsDataFetchingEnvironment dfe) {
    Order order = dfe.getSource();
    DataLoader<String, Customer> loader = dfe.getDataLoader("customers");
    return loader.load(order.customerId());
}
```

DataLoader collects all the individual `load()` calls during a single GraphQL request, batches the IDs, and makes one `findByIds()` call. 21 queries become 2: one for orders, one for all customers.

This is essential. Every nested field that fetches data needs a DataLoader. If you skip this step, your GraphQL API will be slower than the REST API it replaced, which is deeply embarrassing.

## Schema Design

Good schema design makes the API intuitive. Bad schema design makes it confusing.

**Use connections for pagination** (Relay cursor spec):

```graphql
type OrderConnection {
    edges: [OrderEdge!]!
    pageInfo: PageInfo!
    totalCount: Int!
}

type OrderEdge {
    node: Order!
    cursor: String!
}

type PageInfo {
    hasNextPage: Boolean!
    hasPreviousPage: Boolean!
    startCursor: String
    endCursor: String
}
```

**Use input types for mutations**:

```graphql
type Mutation {
    createOrder(input: CreateOrderInput!): CreateOrderPayload!
}

input CreateOrderInput {
    customerId: ID!
    items: [OrderItemInput!]!
    shippingAddress: String!
}

type CreateOrderPayload {
    order: Order
    errors: [Error!]
}
```

Return a payload with both the result and potential errors. This lets the client handle partial failures without relying on HTTP status codes (GraphQL always returns 200).

**Don't expose your database schema.** The GraphQL schema is a client-facing API. Design it for how clients use the data, not how you store it.

## Mutations: Where It Gets Weird

GraphQL mutations are for writes. But they feel awkward compared to REST:

```graphql
mutation {
  createOrder(input: {
    customerId: "cust-123"
    items: [
      { productId: "prod-1", quantity: 2 }
      { productId: "prod-2", quantity: 1 }
    ]
    shippingAddress: "123 Main St"
  }) {
    order {
      id
      status
    }
    errors {
      field
      message
    }
  }
}
```

For simple CRUD, this is more verbose than `POST /orders` with a JSON body. The benefit (client specifies which fields to return) is real but marginal for mutations where you usually want the full created resource.

My rule: if your API is mostly reads with complex data requirements, GraphQL shines. If it's mostly writes, REST might be simpler.

## When GraphQL Is Overkill

**Simple CRUD services.** If your API has a few resources and the client always needs the same fields, REST is simpler. GraphQL adds complexity (schema, resolvers, DataLoaders) without adding value.

**Server-to-server communication.** Internal microservice APIs have known clients with known data needs. REST or gRPC is simpler.

**Real-time event streams.** GraphQL subscriptions exist but WebSockets or Server-Sent Events are more straightforward for real-time data.

**Small teams.** GraphQL requires understanding the schema, DataLoaders, the query execution model, and the N+1 problem. For small teams, REST's simplicity is a feature.

## When GraphQL Wins

**Multiple clients with different data needs.** Mobile wants minimal data. Web wants full data. An admin panel wants everything. One GraphQL schema serves all of them.

**Deeply nested data.** When rendering a page requires traversing multiple relationships (order -> customer -> address, order -> items -> products -> categories), GraphQL eliminates multiple round trips.

**Rapid frontend iteration.** Frontend developers can change what data they fetch without waiting for backend changes. This reduces the coordination overhead between teams.

GraphQL solved a real problem for our BFF layer. Three different frontends, each needing different subsets of the same data. One GraphQL schema replaced three dedicated REST APIs. The reduction in code and coordination was significant.

But for the microservices behind that GraphQL layer? They all expose REST APIs. Sometimes the boring tool is the right tool.
