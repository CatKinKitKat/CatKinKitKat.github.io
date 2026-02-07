+++
title = "Spring Boot Auto-Configuration: How the Magic Works"
date = 2026-02-07
[taxonomies]
tags = ["spring-boot", "java"]
+++

Spring Boot's auto-configuration is the most misunderstood feature of the framework. People either treat it as impenetrable magic ("it just works, don't touch it") or blame it for every unexpected behavior ("auto-configuration did something weird again"). Neither attitude is productive. Let's open the hood.

## The Core Mechanism

When your Spring Boot application starts, it loads `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` from every jar on the classpath. This file lists auto-configuration classes - each one is a `@Configuration` class that *might* create beans, depending on conditions.

That "depending on conditions" part is the key. Auto-configuration doesn't blindly create beans. It checks whether the conditions are met first.

```java
@AutoConfiguration
@ConditionalOnClass(DataSource.class)
@ConditionalOnProperty(prefix = "spring.datasource", name = "url")
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public DataSource dataSource(DataSourceProperties properties) {
        return properties.initializeDataSourceBuilder().build();
    }
}
```

This reads as: "If `DataSource` is on the classpath AND `spring.datasource.url` is configured, AND nobody has already defined a `DataSource` bean, create one." That's not magic. That's a series of reasonable defaults with escape hatches.

## The @Conditional Family

The `@Conditional` annotations are the building blocks. Here are the ones you'll encounter most:

**@ConditionalOnClass / @ConditionalOnMissingClass** - Check if a class is on the classpath. This is how Spring Boot knows which features you've opted into via dependencies.

```java
@ConditionalOnClass(RedisConnectionFactory.class)
// Only activates if you have spring-data-redis on your classpath
```

**@ConditionalOnBean / @ConditionalOnMissingBean** - Check if a bean of a certain type already exists. This is the "don't override what the user defined" mechanism.

```java
@Bean
@ConditionalOnMissingBean(ObjectMapper.class)
public ObjectMapper objectMapper() {
    // Only created if you haven't defined your own ObjectMapper
    return new ObjectMapper();
}
```

**@ConditionalOnProperty** - Check if a configuration property has a specific value.

```java
@ConditionalOnProperty(
    prefix = "app.feature",
    name = "enabled",
    havingValue = "true",
    matchIfMissing = false
)
```

**@ConditionalOnWebApplication / @ConditionalOnNotWebApplication** - Detect the application type.

**@ConditionalOnExpression** - SpEL expression for complex conditions. Use sparingly - it's hard to read.

```java
@ConditionalOnExpression(
    "${app.cache.enabled:true} and '${app.cache.type}' == 'redis'"
)
```

## The Ordering Game

Auto-configuration classes are loaded in a specific order. You can control this with `@AutoConfigureBefore`, `@AutoConfigureAfter`, and `@AutoConfigureOrder`.

```java
@AutoConfiguration(after = DataSourceAutoConfiguration.class)
@ConditionalOnBean(DataSource.class)
public class JpaAutoConfiguration {
    // Runs after DataSource is configured
}
```

This matters because conditions are evaluated at the time the auto-configuration class is processed. If your auto-configuration depends on a bean created by another auto-configuration, you need to ensure yours runs after.

## Debugging Auto-Configuration

When auto-configuration does something unexpected, the debug report is your best friend.

```
java -jar myapp.jar --debug
```

Or set `debug=true` in `application.properties`. This prints a comprehensive report showing:

- **Positive matches**: Auto-configuration classes that activated and why
- **Negative matches**: Auto-configuration classes that didn't activate and why
- **Exclusions**: Auto-configuration classes explicitly excluded

The output looks like:

```
Positive matches:
-----------------
DataSourceAutoConfiguration matched:
 - @ConditionalOnClass found required classes 'javax.sql.DataSource' (OnClassCondition)
 - @ConditionalOnProperty (spring.datasource.url) matched (OnPropertyCondition)

Negative matches:
-----------------
MongoAutoConfiguration:
 - @ConditionalOnClass did not find required class 'com.mongodb.client.MongoClient' (OnClassCondition)
```

This is infinitely more useful than guessing. If a bean isn't being created, the negative matches tell you exactly which condition failed.

## Writing Your Own Auto-Configuration

This is where the knowledge pays off. If you're writing a shared library for your organization, auto-configuration lets you provide sensible defaults that users can override.

Step 1: Create the auto-configuration class.

```java
@AutoConfiguration
@ConditionalOnClass(AuditService.class)
@EnableConfigurationProperties(AuditProperties.class)
public class AuditAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public AuditService auditService(AuditProperties properties) {
        return new AuditService(properties.getEndpoint(), properties.getApiKey());
    }

    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnProperty(prefix = "audit", name = "async", havingValue = "true")
    public AsyncAuditDecorator asyncAuditDecorator(AuditService auditService) {
        return new AsyncAuditDecorator(auditService);
    }
}
```

Step 2: The properties class.

```java
@ConfigurationProperties(prefix = "audit")
public class AuditProperties {
    private String endpoint = "http://audit-service:8080";
    private String apiKey;
    private boolean async = false;

    // getters and setters
}
```

Step 3: Register the auto-configuration. Create `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`:

```
com.example.audit.AuditAutoConfiguration
```

Step 4: Package it as a Spring Boot starter. By convention, starters are named `my-company-spring-boot-starter` and include the auto-configuration module plus all necessary dependencies.

```xml
<!-- my-company-audit-spring-boot-starter pom.xml -->
<dependencies>
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>audit-autoconfigure</artifactId>
    </dependency>
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>audit-client</artifactId>
    </dependency>
</dependencies>
```

Now any project that adds your starter gets automatic audit configuration that works out of the box but can be customized through properties or by defining their own beans.

## Custom Starter Libraries: Lessons Learned

I've built several internal starters at different consultancy clients. Some lessons:

**Always use @ConditionalOnMissingBean.** Let users override your defaults. Nothing is more frustrating than a library that insists on its own configuration.

**Provide good defaults but make everything configurable.** If your default timeout is 5 seconds, great. But let me change it through properties without forking your starter.

**Test your conditions.** Spring Boot provides `ApplicationContextRunner` for testing auto-configuration in isolation.

```java
@Test
void autoConfiguresAuditServiceWhenClassPresent() {
    new ApplicationContextRunner()
        .withConfiguration(AutoConfigurations.of(AuditAutoConfiguration.class))
        .withPropertyValues("audit.api-key=test-key")
        .run(context -> {
            assertThat(context).hasSingleBean(AuditService.class);
        });
}

@Test
void doesNotConfigureWhenUserDefinesOwnBean() {
    new ApplicationContextRunner()
        .withConfiguration(AutoConfigurations.of(AuditAutoConfiguration.class))
        .withBean(AuditService.class, () -> new CustomAuditService())
        .run(context -> {
            assertThat(context).hasSingleBean(AuditService.class);
            assertThat(context.getBean(AuditService.class))
                .isInstanceOf(CustomAuditService.class);
        });
}
```

**Document the properties.** Generate metadata using `spring-boot-configuration-processor`. This gives IDE autocompletion for your custom properties, which makes your starter feel polished.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

## The Exclusion Escape Hatch

Sometimes auto-configuration does something you genuinely don't want. You can exclude specific auto-configuration classes:

```java
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,
    SecurityAutoConfiguration.class
})
public class MyApplication { }
```

Or via properties:

```yaml
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

I use this most often when a dependency pulls in an auto-configuration I don't need (looking at you, security auto-configuration in a service that handles auth externally).

## The Takeaway

Auto-configuration isn't magic. It's a series of conditional bean definitions that provide sensible defaults. When you understand the conditions, you can predict the behavior. When you can predict the behavior, you can control it.

Run with `--debug` when something isn't working. Check the condition evaluation report. And when you build shared libraries, use auto-configuration to give your users the same "it just works" experience that Spring Boot provides.
