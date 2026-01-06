+++
title = "Serverless Java: The Cold Start Problem and What to Do About It"
date = 2025-06-25
[taxonomies]
tags = ["java", "serverless", "cloud"]
+++

Serverless and Java have a reputation problem. The conversation usually goes: "We're going serverless." "What language?" "Java." "Good luck with cold starts."

And honestly, that reputation was earned. A Spring Boot function on AWS Lambda takes 8-15 seconds for a cold start. By the time your function responds, the user has given up, closed the tab, and filed a support ticket. But the landscape has changed significantly, and Java on serverless is no longer the punchline it used to be.

## The Cold Start Problem

Cold starts happen when the serverless platform needs to provision a new instance of your function. It downloads your deployment package, starts a runtime, initializes your framework, and then handles the request. For Java, this means starting a JVM, loading classes, running dependency injection, and initializing connection pools.

Python and Node.js cold starts: 100-500ms. Java with Spring Boot: 8-15 seconds. Java with nothing: 1-3 seconds. The JVM startup isn't the main culprit - it's the framework initialization.

## AWS Lambda with Java

The basic setup:

```java
public class OrderHandler implements RequestHandler<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> {

    @Override
    public APIGatewayProxyResponseEvent handleRequest(
            APIGatewayProxyRequestEvent input, Context context) {
        String orderId = input.getPathParameters().get("id");
        // do stuff
        return new APIGatewayProxyResponseEvent()
                .withStatusCode(200)
                .withBody("{\"orderId\": \"" + orderId + "\"}");
    }
}
```

Without any framework, this cold starts in about 2 seconds. The moment you add Spring Boot, it jumps to 10+. The question is: how much framework do you actually need in a Lambda function?

### SnapStart

AWS Lambda SnapStart is the biggest improvement for Java Lambda functions. It takes a snapshot of the initialized JVM (after your static initialization completes) and restores from that snapshot on cold start. Instead of initializing Spring Boot from scratch, it restores the already-initialized state.

```yaml
Resources:
  OrderFunction:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: java21
      SnapStart:
        ApplyOn: PublishedVersions
```

SnapStart drops cold starts from 10+ seconds to under 1 second for most Spring Boot functions. The catch: you need to handle the fact that your function might restore from a snapshot with stale state. Database connections, temporary credentials, random number generators - anything that shouldn't survive a freeze/thaw cycle needs to be re-initialized.

```java
import org.crac.Context;
import org.crac.Core;
import org.crac.Resource;

public class OrderHandler implements RequestHandler<...>, Resource {

    public OrderHandler() {
        Core.getGlobalContext().register(this);
    }

    @Override
    public void beforeCheckpoint(Context<? extends Resource> context) {
        // close connections before snapshot
        dataSource.close();
    }

    @Override
    public void afterRestore(Context<? extends Resource> context) {
        // reinitialize after restore
        dataSource = createDataSource();
    }
}
```

## Azure Functions with Java

Azure Functions has a different model. The Java worker runs as a long-lived process, so cold starts are mostly about the first invocation after a scale event or idle timeout.

```java
public class OrderFunction {

    @FunctionName("GetOrder")
    public HttpResponseMessage run(
            @HttpTrigger(name = "req",
                         methods = {HttpMethod.GET},
                         authLevel = AuthorizationLevel.ANONYMOUS,
                         route = "orders/{id}")
            HttpRequestMessage<Optional<String>> request,
            @BindingName("id") String orderId,
            ExecutionContext context) {

        return request.createResponseBuilder(HttpStatus.OK)
                .body(orderService.findById(orderId))
                .build();
    }
}
```

Azure's premium plan keeps instances warm. The consumption plan has cold starts, but they're less severe than Lambda because the JVM stays running between invocations. The tradeoff is cost - premium plan means you're paying for always-on instances.

## Quarkus Funqy

Quarkus was built for this. Funqy provides a cloud-agnostic function API that deploys to Lambda, Azure Functions, or Knative:

```java
public class OrderFunction {

    @Inject
    OrderService orderService;

    @Funq
    public Order getOrder(String orderId) {
        return orderService.findById(orderId);
    }
}
```

Quarkus cold starts in about 1-2 seconds on Lambda without SnapStart. With native image compilation (GraalVM), cold starts drop to 200-500ms. At that point, you're competitive with Node.js.

The native image tradeoff: longer build times (minutes instead of seconds), reduced peak throughput (no JIT optimization), and some Java features don't work (reflection needs configuration, dynamic proxies need hints). For short-lived functions, the cold start improvement usually outweighs the throughput reduction.

## Spring Cloud Function

Spring's approach to serverless:

```java
@Bean
public Function<String, Order> getOrder(OrderService orderService) {
    return orderId -> orderService.findById(orderId);
}
```

Spring Cloud Function provides adapters for Lambda, Azure Functions, and Google Cloud Functions. You write a `Function<Input, Output>` bean, and the adapter handles the platform-specific integration.

The cold start problem remains - Spring's dependency injection and autoconfiguration are the bottleneck. Spring Native (GraalVM native image for Spring Boot) helps, but it's not as mature or fast as Quarkus native.

## The Zero-Dependency Approach

Sometimes the answer is: don't use a framework at all.

```java
public class OrderHandler implements RequestHandler<Map<String, Object>, Map<String, Object>> {

    private static final ObjectMapper mapper = new ObjectMapper();
    private static final DynamoDbClient dynamoDb = DynamoDbClient.create();

    @Override
    public Map<String, Object> handleRequest(Map<String, Object> input, Context context) {
        String orderId = extractOrderId(input);

        GetItemResponse response = dynamoDb.getItem(GetItemRequest.builder()
                .tableName("orders")
                .key(Map.of("id", AttributeValue.fromS(orderId)))
                .build());

        return Map.of(
            "statusCode", 200,
            "body", mapper.writeValueAsString(toOrder(response.item()))
        );
    }
}
```

No Spring. No CDI. No framework. Just the AWS SDK and Jackson. Cold start: ~1.5 seconds. Combined with SnapStart: under 500ms.

The downside is obvious: no dependency injection, no configuration management, no middleware pipeline. For simple CRUD functions, this is fine. For anything complex, you'll end up reinventing a framework badly.

## My Decision Framework

**If your function is simple (CRUD, event processing)**: Zero-dependency or Quarkus Funqy with native image. Cold starts under 500ms.

**If your function needs a real framework**: Quarkus with SnapStart. Best balance of developer experience and cold start performance.

**If your team is already Spring Boot**: Spring Cloud Function with SnapStart. Cold starts are acceptable (~1 second), and the team doesn't need to learn a new framework.

**If cold starts don't matter (async processing, scheduled jobs)**: Use whatever you want. Spring Boot on Lambda with SnapStart is fine.

The honest truth: if your function is called frequently enough that cold starts are rare (more than once per 5-15 minutes on Lambda), the cold start problem mostly disappears. It only matters for spiky, infrequent workloads. And for those workloads, maybe Java isn't the right choice - and that's okay to admit.
