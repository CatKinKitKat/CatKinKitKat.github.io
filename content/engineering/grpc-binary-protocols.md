+++
title = "gRPC and Binary Protocols: When JSON Just Isn't Fast Enough"
date = 2025-09-05
[taxonomies]
tags = ["api", "grpc", "microservices"]
+++

I measured the overhead of JSON serialization in one of our services. The service spent 15% of its CPU time parsing JSON requests and serializing JSON responses. The actual business logic? About 5%. The rest was framework overhead, but the serialization stood out because it was the one thing I could fix without rewriting the architecture.

Switching the internal service-to-service communication to gRPC with Protobuf dropped that serialization overhead to under 2%. For external APIs, we kept REST/JSON because browsers and frontend developers exist. For the internal mesh of 30+ microservices talking to each other, binary protocols won.

## Protobuf: The Wire Format

Protocol Buffers (Protobuf) is Google's binary serialization format. You define messages in a `.proto` file and generate code for your language:

```protobuf
syntax = "proto3";

package orders;

option java_package = "com.myapp.order.grpc";
option java_multiple_files = true;

message Order {
  string id = 1;
  string customer_id = 2;
  OrderStatus status = 3;
  repeated OrderItem items = 4;
  Money total_amount = 5;
  google.protobuf.Timestamp created_at = 6;
}

message OrderItem {
  string product_id = 1;
  int32 quantity = 2;
  Money unit_price = 3;
}

message Money {
  int64 amount_micros = 1;  // amount in micros (1,000,000 = 1 unit)
  string currency_code = 2;
}

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  DRAFT = 1;
  SUBMITTED = 2;
  PAID = 3;
  SHIPPED = 4;
  CANCELLED = 5;
}
```

Protobuf messages are compact (typically 3-10x smaller than JSON), fast to serialize/deserialize (10-100x faster depending on the message), and have a well-defined schema with backward/forward compatibility built in.

Field numbers (`= 1`, `= 2`) are the wire format identifiers. You can add new fields (with new numbers) without breaking existing clients. You can remove fields without breaking existing clients (the field number is just skipped). You can't change field types or reuse field numbers. This is better versioning than most REST APIs manage.

## gRPC: The Transport

gRPC uses Protobuf for serialization and HTTP/2 for transport. The service definition lives in the `.proto` file:

```protobuf
service OrderService {
  rpc GetOrder(GetOrderRequest) returns (Order);
  rpc CreateOrder(CreateOrderRequest) returns (Order);
  rpc ListOrders(ListOrdersRequest) returns (ListOrdersResponse);
  rpc StreamOrderUpdates(StreamOrdersRequest) returns (stream OrderUpdate);
}

message GetOrderRequest {
  string id = 1;
}

message CreateOrderRequest {
  string customer_id = 1;
  repeated CreateOrderItemRequest items = 2;
}

message CreateOrderItemRequest {
  string product_id = 1;
  int32 quantity = 2;
}

message ListOrdersRequest {
  string customer_id = 1;
  int32 page_size = 2;
  string page_token = 3;
}

message ListOrdersResponse {
  repeated Order orders = 1;
  string next_page_token = 2;
}
```

The `protoc` compiler generates client stubs and server interfaces. You implement the server interface; clients use the generated stub. No manual HTTP client code, no URL construction, no JSON parsing.

## Spring Boot gRPC

Using `grpc-spring-boot-starter`:

```java
@GrpcService
public class OrderGrpcService extends OrderServiceGrpc.OrderServiceImplBase {

    private final OrderService orderService;
    private final OrderProtoMapper mapper;

    @Override
    public void getOrder(GetOrderRequest request,
                         StreamObserver<Order> responseObserver) {
        try {
            var order = orderService.findById(request.getId());
            responseObserver.onNext(mapper.toProto(order));
            responseObserver.onCompleted();
        } catch (OrderNotFoundException e) {
            responseObserver.onError(Status.NOT_FOUND
                .withDescription("Order not found: " + request.getId())
                .asRuntimeException());
        }
    }

    @Override
    public void createOrder(CreateOrderRequest request,
                            StreamObserver<Order> responseObserver) {
        var command = mapper.toCommand(request);
        var order = orderService.createOrder(command);
        responseObserver.onNext(mapper.toProto(order));
        responseObserver.onCompleted();
    }
}
```

The client side:

```java
@Component
public class OrderGrpcClient {

    private final OrderServiceGrpc.OrderServiceBlockingStub stub;

    public OrderGrpcClient(@GrpcClient("order-service")
                           OrderServiceGrpc.OrderServiceBlockingStub stub) {
        this.stub = stub;
    }

    public Order getOrder(String id) {
        GetOrderRequest request = GetOrderRequest.newBuilder()
            .setId(id)
            .build();
        return stub.getOrder(request);
    }
}
```

Type-safe. No URL strings. No JSON serialization. The compiler catches breaking changes. If the server changes a field type, the client code won't compile.

## Streaming: Where gRPC Shines

gRPC supports four communication patterns:

**Unary**: one request, one response. Like REST.

**Server streaming**: one request, multiple responses. Great for real-time updates:

```java
@Override
public void streamOrderUpdates(StreamOrdersRequest request,
                                StreamObserver<OrderUpdate> responseObserver) {
    orderEventStream.subscribe(event -> {
        if (event.matchesFilter(request)) {
            responseObserver.onNext(mapper.toUpdateProto(event));
        }
    });
    // stream stays open until client disconnects
}
```

**Client streaming**: multiple requests, one response. Good for bulk uploads.

**Bidirectional streaming**: multiple requests, multiple responses. Real-time chat-style communication.

REST can't do streaming natively (WebSocket is a separate protocol). GraphQL subscriptions are server streaming only. gRPC handles all four patterns as first-class citizens on the same connection.

## gRPC vs REST: The Real Comparison

**Performance**: gRPC wins. Binary serialization is faster and smaller. HTTP/2 multiplexing reduces connection overhead. In benchmarks, gRPC is 5-10x faster for serialization-heavy workloads.

**Developer experience**: REST wins for simplicity. You can test REST with curl, a browser, or Postman. gRPC needs specialized tools (grpcurl, BloomRPC, Postman with gRPC support). Debugging gRPC is harder because the wire format is binary.

**Browser support**: REST wins. Browsers speak HTTP/1.1 and JSON natively. gRPC-Web exists but requires a proxy (Envoy) to translate. For public APIs consumed by browsers, REST is the pragmatic choice.

**Schema and type safety**: gRPC wins. The `.proto` file is the contract. Code generation ensures client and server agree. REST APIs have OpenAPI specs, but they're optional and often out of date. The `.proto` file IS the implementation.

**Versioning**: gRPC wins. Protobuf's field numbering scheme handles forward and backward compatibility better than URL versioning or content negotiation.

**Ecosystem**: REST wins. Every language, framework, tool, and tutorial assumes REST/JSON. gRPC has good support in Java, Go, C++, and Python, but the ecosystem is smaller.

## Service Mesh gRPC Support

Istio and Linkerd handle gRPC natively. Because gRPC uses HTTP/2, the mesh's Envoy proxy can route, load balance, and apply policies to gRPC traffic just like HTTP traffic.

One difference: gRPC uses long-lived HTTP/2 connections with multiplexed streams. Kubernetes' default round-robin load balancing at the connection level (kube-proxy) doesn't distribute requests evenly - all streams on one connection go to the same pod.

A service mesh solves this by doing request-level (L7) load balancing. Each gRPC request within the HTTP/2 connection can be routed to a different pod. Without a mesh, you need a client-side load balancer or gRPC's built-in load balancing.

## When Binary Protocols Win

**Service-to-service communication**: when both sides are your code and performance matters, gRPC is the clear winner.

**High-throughput data pipelines**: event streaming, data processing, batch operations. The serialization overhead of JSON adds up at volume.

**Polyglot environments**: a `.proto` file generates correct clients for Java, Go, Python, TypeScript, and more. Better than hoping everyone implements the same REST API correctly.

**When you need streaming**: server-push, bidirectional communication, real-time updates. gRPC handles this natively; REST requires WebSocket bolt-ons.

## When REST Wins

**Public APIs**: browsers, curl, and developer adoption. REST is universal.

**Simple CRUD**: the overhead of `.proto` files and code generation isn't worth it for a straightforward data API.

**Teams without gRPC experience**: the learning curve is real. REST is something every developer already knows.

My approach: REST for external APIs, gRPC for internal service-to-service communication. The gateway translates between them. Clients see REST/JSON, services talk gRPC/Protobuf. Both sides get what they need.
