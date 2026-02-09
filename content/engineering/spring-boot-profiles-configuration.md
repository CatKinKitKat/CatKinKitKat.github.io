+++
title = "Spring Boot Profiles and Externalized Configuration Done Right"
date = 2026-02-09
[taxonomies]
tags = ["spring-boot", "java"]
+++

I've seen Spring Boot configuration go wrong in every possible way. Properties scattered across ten files. Secrets committed to Git. Profile-specific logic buried in beans. Environment-specific URLs hardcoded in Java classes behind `@Profile` annotations. It doesn't have to be like this.

## @ConfigurationProperties: The Right Way

Stop injecting individual `@Value` properties. Use `@ConfigurationProperties` instead. It gives you type safety, validation, and a single place to see all configuration for a feature.

```java
@ConfigurationProperties(prefix = "app.payment")
@Validated
public class PaymentProperties {

    @NotBlank
    private String gatewayUrl;

    @NotBlank
    private String apiKey;

    @Min(1) @Max(30)
    private int timeoutSeconds = 5;

    @Min(1) @Max(5)
    private int maxRetries = 3;

    private boolean sandboxMode = false;

    // getters and setters
}
```

```yaml
app:
  payment:
    gateway-url: https://payments.example.com/api
    api-key: ${PAYMENT_API_KEY}
    timeout-seconds: 10
    max-retries: 3
    sandbox-mode: false
```

```java
@Service
public class PaymentService {

    private final PaymentProperties props;

    public PaymentService(PaymentProperties props) {
        this.props = props;
    }

    public PaymentResult charge(PaymentRequest request) {
        // All config in one place, validated at startup
        return client.post(props.getGatewayUrl())
            .timeout(Duration.ofSeconds(props.getTimeoutSeconds()))
            .body(request)
            .execute();
    }
}
```

If the properties fail validation, the application refuses to start. That's a feature. Finding out that your payment gateway URL is blank at startup is infinitely better than finding out at 2 AM when the first customer tries to pay.

## Profiles: Use Sparingly

Profiles are useful but overused. I see teams with profiles like `dev`, `local`, `staging`, `uat`, `preprod`, `prod`, `performance`, and `demo`. Each profile has its own property file, and nobody can tell which values actually apply because of the precedence rules.

My approach: keep profiles to a minimum.

```
application.yml              # Defaults that work everywhere
application-local.yml         # Local development only
application-test.yml          # Test-specific overrides
```

That's it. Everything else comes from environment variables or an external config server. The fewer profile-specific files you have, the less mental overhead for the team.

```yaml
# application.yml - sensible defaults
spring:
  datasource:
    url: ${DATABASE_URL:jdbc:postgresql://localhost:5432/myapp}
    username: ${DATABASE_USER:myapp}
    password: ${DATABASE_PASSWORD:localdev}
  kafka:
    bootstrap-servers: ${KAFKA_BROKERS:localhost:9092}

app:
  payment:
    gateway-url: ${PAYMENT_GATEWAY_URL:https://sandbox.payments.example.com}
    api-key: ${PAYMENT_API_KEY:sandbox-key}
```

Default values point to local development services. In deployed environments, environment variables override them. No profile switching needed.

## The 12-Factor Perspective

The [Twelve-Factor App](https://12factor.net/) says: store config in the environment. Spring Boot supports this beautifully. Every property can be overridden by an environment variable.

The mapping rule is straightforward: replace dots with underscores, make everything uppercase. `spring.datasource.url` becomes `SPRING_DATASOURCE_URL`. `app.payment.gateway-url` becomes `APP_PAYMENT_GATEWAY_URL`.

```yaml
# Kubernetes deployment
spec:
  containers:
    - name: order-service
      env:
        - name: SPRING_DATASOURCE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        - name: SPRING_DATASOURCE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        - name: APP_PAYMENT_GATEWAY_URL
          value: "https://payments.example.com/api"
```

This is the correct way to handle environment-specific config. The application artifact is the same across all environments. The configuration differs. This is a fundamental principle that many teams violate.

## Spring Cloud Config

For larger deployments, Spring Cloud Config gives you a centralized configuration server backed by Git, Vault, or a database.

```yaml
# Config server
spring:
  cloud:
    config:
      server:
        git:
          uri: https://git.example.com/config-repo
          search-paths: "{application}/{profile}"
          default-label: main
```

```yaml
# Client service
spring:
  config:
    import: "configserver:http://config-server:8888"
  cloud:
    config:
      fail-fast: true
      retry:
        max-attempts: 5
        initial-interval: 1000
```

The config server serves property files from a Git repository. Services pull their configuration on startup. Changes can be pushed by refreshing the application context (via Actuator's `/refresh` endpoint or Spring Cloud Bus).

I have a love-hate relationship with Spring Cloud Config. It centralizes configuration, which is good. But it adds a runtime dependency on the config server, which means your services can't start if the config server is down. The `fail-fast: true` with retries helps, but it's still a single point of failure unless you cluster it.

For Kubernetes deployments, I increasingly prefer ConfigMaps and Secrets over Spring Cloud Config. They're native to the platform, don't require a separate server, and can be mounted as files or environment variables.

## Securing Configuration

Never commit secrets to your Git repository. Not even in "private" repos. Not even "temporarily." I've personally witnessed API keys in Git that were committed "just for testing" and were still there two years later.

```yaml
# DO NOT DO THIS
app:
  payment:
    api-key: sk_live_abc123realkey

# DO THIS
app:
  payment:
    api-key: ${PAYMENT_API_KEY}
```

For local development, use a `.env` file (not committed to Git) or environment-specific secrets management. For production, use your platform's secrets management:

- **Kubernetes Secrets** (base64-encoded, not encrypted by default - use Sealed Secrets or External Secrets Operator)
- **HashiCorp Vault** (proper secrets management with rotation, auditing, and dynamic secrets)
- **Cloud provider secrets** (AWS Secrets Manager, Azure Key Vault, GCP Secret Manager)

Spring Cloud Vault integrates directly:

```yaml
spring:
  cloud:
    vault:
      uri: https://vault.example.com
      authentication: kubernetes
      kubernetes:
        role: order-service
      kv:
        enabled: true
        backend: secret
        default-context: order-service
```

This pulls secrets from Vault at startup. The application never sees plaintext credentials in config files or environment variables.

## Property Precedence

Spring Boot has 17 levels of property precedence. You don't need to memorize all of them, but you should know the important ones (highest to lowest):

1. Command line arguments (`--server.port=9090`)
2. OS environment variables (`SERVER_PORT=9090`)
3. Profile-specific properties (`application-prod.yml`)
4. Application properties (`application.yml`)
5. Default properties (`@PropertySource` defaults)

This means environment variables override property files, which is exactly what you want for 12-factor deployments. But it also means debugging can be tricky when a property is being overridden at a level you didn't expect.

The Actuator `/env` endpoint shows all properties and their sources:

```
GET /actuator/env/spring.datasource.url

{
  "property": {
    "source": "systemEnvironment",
    "value": "jdbc:postgresql://prod-db:5432/myapp"
  }
}
```

## @Profile: The Annotation to Avoid

I said it. `@Profile` on beans is almost always wrong. It scatters environment-specific logic across your codebase.

```java
// DON'T do this
@Bean
@Profile("prod")
public PaymentGateway realPaymentGateway() {
    return new StripeGateway();
}

@Bean
@Profile("!prod")
public PaymentGateway fakePaymentGateway() {
    return new FakeGateway();
}
```

Instead, use properties:

```java
// DO this
@Bean
public PaymentGateway paymentGateway(PaymentProperties props) {
    if (props.isSandboxMode()) {
        return new FakeGateway();
    }
    return new StripeGateway(props.getGatewayUrl(), props.getApiKey());
}
```

The difference is subtle but important. The `@Profile` version ties the behavior to a Spring profile name, which is an environment concept. The property version ties it to a business flag, which can be toggled independently of the environment. You can run sandbox mode in production for testing. You can run real mode locally for integration tests. The configuration is decoupled from the environment identity.

## My Configuration Checklist

1. Use `@ConfigurationProperties` with validation for all application config.
2. Keep profile-specific files to a minimum (local, test, and that's usually it).
3. Use environment variables for environment-specific values.
4. Never commit secrets. Use Vault, Kubernetes Secrets, or your cloud provider's secrets manager.
5. Use the Actuator `/env` endpoint to debug property resolution.
6. Avoid `@Profile` on beans - use feature flags through properties instead.
7. Make sure your application fails fast on invalid configuration.

Configuration is boring. It should be boring. The exciting version - where you discover in production that a property is wrong - is the kind of excitement nobody needs.
