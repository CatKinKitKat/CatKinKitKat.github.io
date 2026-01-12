+++
title = "Hexagonal Architecture: Ports, Adapters, and Actual Code"
date = 2025-07-25
[taxonomies]
tags = ["architecture", "spring-boot", "patterns"]
+++

I've read too many articles about hexagonal architecture that show a pretty diagram with hexagons and arrows and then stop. No code. No practical guidance. Just "your domain should be pure" and a link to Alistair Cockburn's original paper. Thanks, very helpful.

Let me show you what hexagonal architecture actually looks like in a Spring Boot service, why it matters, and where it breaks down.

## The Core Idea

Hexagonal architecture (ports and adapters) separates your business logic from your infrastructure. The business logic defines interfaces (ports) that describe what it needs. The infrastructure provides implementations (adapters) that fulfill those interfaces.

The direction of dependency is inward. The domain doesn't know about Spring. It doesn't know about JPA. It doesn't know about HTTP. It defines what it needs, and the outer layers provide it.

```
[HTTP Controller] -> [Input Port] -> [Domain Service] -> [Output Port] <- [JPA Repository]
         ^                                                                        ^
      adapter                                                                  adapter
```

The domain service depends on the output port (an interface). The JPA repository implements the output port. If you swap JPA for MongoDB, the domain service doesn't change.

## The Package Structure

```
com.myapp.order/
  domain/
    model/
      Order.java
      OrderStatus.java
      OrderItem.java
    port/
      in/
        CreateOrderUseCase.java
        GetOrderUseCase.java
      out/
        OrderRepository.java
        PaymentGateway.java
        EventPublisher.java
    service/
      OrderService.java
    exception/
      OrderNotFoundException.java
      InsufficientStockException.java
  adapter/
    in/
      web/
        OrderController.java
        OrderRequest.java
        OrderResponse.java
    out/
      persistence/
        OrderJpaRepository.java
        OrderEntity.java
        OrderMapper.java
      payment/
        StripePaymentGateway.java
      messaging/
        KafkaEventPublisher.java
```

The `domain` package has zero framework dependencies. No Spring annotations. No JPA annotations. No Jackson annotations. Pure Java.

## Input Ports: What the Application Can Do

Input ports are interfaces that define use cases:

```java
public interface CreateOrderUseCase {
    Order createOrder(CreateOrderCommand command);
}

public interface GetOrderUseCase {
    Order getOrder(String orderId);
    List<Order> getOrdersByCustomer(String customerId);
}
```

These are your application's API from the domain's perspective. The command object is a domain concept, not an HTTP request:

```java
public record CreateOrderCommand(
    String customerId,
    List<OrderItemCommand> items,
    String shippingAddress
) {
    public CreateOrderCommand {
        if (items == null || items.isEmpty()) {
            throw new IllegalArgumentException("Order must have at least one item");
        }
    }
}
```

Validation lives in the domain. The command validates itself.

## Output Ports: What the Application Needs

Output ports are interfaces that the domain defines and the infrastructure implements:

```java
public interface OrderRepository {
    Order save(Order order);
    Optional<Order> findById(String id);
    List<Order> findByCustomerId(String customerId);
}

public interface PaymentGateway {
    PaymentResult charge(String customerId, Money amount);
}

public interface EventPublisher {
    void publish(DomainEvent event);
}
```

These are SPI (Service Provider Interface) contracts. The domain says "I need something that can save orders" without caring whether it's PostgreSQL, MongoDB, or an in-memory map.

## The Domain Service

The domain service implements the input ports and depends on the output ports:

```java
public class OrderService implements CreateOrderUseCase, GetOrderUseCase {

    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;
    private final EventPublisher eventPublisher;

    public OrderService(OrderRepository orderRepository,
                        PaymentGateway paymentGateway,
                        EventPublisher eventPublisher) {
        this.orderRepository = orderRepository;
        this.paymentGateway = paymentGateway;
        this.eventPublisher = eventPublisher;
    }

    @Override
    public Order createOrder(CreateOrderCommand command) {
        Order order = Order.create(command);

        PaymentResult payment = paymentGateway.charge(
            command.customerId(), order.totalAmount());

        if (!payment.isSuccessful()) {
            throw new PaymentFailedException(payment.reason());
        }

        order.markAsPaid(payment.transactionId());
        Order saved = orderRepository.save(order);

        eventPublisher.publish(new OrderCreatedEvent(saved.id()));

        return saved;
    }

    @Override
    public Order getOrder(String orderId) {
        return orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }
}
```

No `@Service`. No `@Transactional`. No Spring anything. This is a plain Java class that expresses business rules in business language.

## Adapters: The Infrastructure

### Inbound Adapter (HTTP)

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private final CreateOrderUseCase createOrder;
    private final GetOrderUseCase getOrder;

    public OrderController(CreateOrderUseCase createOrder,
                           GetOrderUseCase getOrder) {
        this.createOrder = createOrder;
        this.getOrder = getOrder;
    }

    @PostMapping
    public ResponseEntity<OrderResponse> create(@RequestBody OrderRequest request) {
        CreateOrderCommand command = request.toCommand();
        Order order = createOrder.createOrder(command);
        return ResponseEntity.status(201).body(OrderResponse.from(order));
    }

    @GetMapping("/{id}")
    public OrderResponse get(@PathVariable String id) {
        return OrderResponse.from(getOrder.getOrder(id));
    }
}
```

The controller depends on the use case interface, not the service implementation. It translates HTTP concerns (request/response DTOs, status codes) into domain concepts (commands, domain objects).

### Outbound Adapter (Persistence)

```java
@Repository
public class OrderJpaRepository implements OrderRepository {

    private final SpringDataOrderRepository jpaRepo;
    private final OrderMapper mapper;

    @Override
    public Order save(Order order) {
        OrderEntity entity = mapper.toEntity(order);
        OrderEntity saved = jpaRepo.save(entity);
        return mapper.toDomain(saved);
    }

    @Override
    public Optional<Order> findById(String id) {
        return jpaRepo.findById(id).map(mapper::toDomain);
    }
}
```

The JPA entity is separate from the domain model. The mapper converts between them. The domain model doesn't have `@Entity`, `@Column`, or `@Id` annotations. This means your domain model can evolve independently from your database schema.

## Wiring It Together with Spring

The domain service has no Spring annotations, so you need a configuration class:

```java
@Configuration
public class OrderConfiguration {

    @Bean
    public OrderService orderService(OrderRepository orderRepository,
                                      PaymentGateway paymentGateway,
                                      EventPublisher eventPublisher) {
        return new OrderService(orderRepository, paymentGateway, eventPublisher);
    }
}
```

This is where Spring's DI/IoC connects everything. The configuration class is in the adapter layer - it knows about both the domain and the infrastructure. The domain remains pure.

## Business Exceptions

Don't throw HTTP exceptions from your domain. Throw domain-specific exceptions:

```java
public class OrderNotFoundException extends RuntimeException {
    private final String orderId;

    public OrderNotFoundException(String orderId) {
        super("Order not found: " + orderId);
        this.orderId = orderId;
    }
}

public class InsufficientStockException extends RuntimeException {
    private final String productId;
    private final int requested;
    private final int available;
    // ...
}
```

Translate them to HTTP in an exception handler:

```java
@RestControllerAdvice
public class OrderExceptionHandler {

    @ExceptionHandler(OrderNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(OrderNotFoundException ex) {
        return ResponseEntity.status(404)
            .body(new ErrorResponse("ORDER_NOT_FOUND", ex.getMessage()));
    }

    @ExceptionHandler(InsufficientStockException.class)
    public ResponseEntity<ErrorResponse> handleInsufficientStock(InsufficientStockException ex) {
        return ResponseEntity.status(409)
            .body(new ErrorResponse("INSUFFICIENT_STOCK", ex.getMessage()));
    }
}
```

The domain doesn't know about HTTP status codes. The adapter translates domain exceptions to HTTP responses.

## The Open/Closed Principle in Practice

Need to add a new delivery mechanism? A gRPC adapter? A CLI? Add a new inbound adapter that depends on the same use case interfaces. The domain doesn't change. Open for extension (new adapters), closed for modification (domain unchanged).

Need to switch from Kafka to RabbitMQ for events? Implement a new `EventPublisher` adapter. The domain doesn't change. The Open/Closed Principle isn't an academic exercise - it's the practical result of clean boundaries.

## Where It Breaks Down

For CRUD services with no business logic, hexagonal architecture is overkill. If your service is "receive HTTP request, validate, save to database, return response," the ceremony of ports, adapters, and mappers adds complexity without adding value. Just use a controller and a Spring Data repository.

The mapping between domain models and entities is tedious. You write a lot of mapper code. MapStruct helps but it's another dependency. For simple services, the domain model and the JPA entity can be the same class. Heresy? Maybe. But pragmatism beats purity.

The honest truth: I use full hexagonal architecture for services with complex business logic (order processing, payment workflows, pricing engines). For simple CRUD, I use a layered architecture with Spring's defaults. Knowing when to apply which pattern is more valuable than dogmatic adherence to either.
