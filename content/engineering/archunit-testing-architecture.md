+++
title = "ArchUnit, or How to Stop Architectural Rules from Being Suggestions"
date = 2025-12-10
[taxonomies]
tags = ["testing", "architecture", "java"]
+++

Every team has architectural rules. "Controllers shouldn't access repositories directly." "Domain objects must not depend on Spring." "No circular dependencies between packages." These rules live in a wiki page that was last updated eighteen months ago, or in the head of the senior developer who's now on parental leave.

The problem isn't defining the rules. The problem is enforcing them. Code reviews catch some violations, but reviewers are human and they miss things, especially in large pull requests. By the time someone notices, the violation has been in the codebase for months and removing it means refactoring half the module.

ArchUnit makes architectural rules executable. You write them as tests. They run in CI. They fail the build if someone breaks the rules. No wiki required.

## The Basics

ArchUnit is a Java library that inspects your compiled bytecode and lets you write assertions about it. Add the dependency:

```xml
<dependency>
    <groupId>com.tngtech.archunit</groupId>
    <artifactId>archunit-junit5</artifactId>
    <version>1.2.1</version>
    <scope>test</scope>
</dependency>
```

Your first test:

```java
@AnalyzeClasses(packages = "com.example.orderservice")
class ArchitectureTest {

    @ArchTest
    static final ArchRule controllers_should_not_access_repositories =
        noClasses()
            .that().resideInAPackage("..controller..")
            .should().accessClassesThat()
            .resideInAPackage("..repository..");
}
```

If any class in a `controller` package directly accesses a class in a `repository` package, this test fails. Simple. Clear. Automated.

## Enforcing Layer Dependencies

The classic layered architecture: controllers call services, services call repositories. Nothing else.

```java
@ArchTest
static final ArchRule layer_dependencies = layeredArchitecture()
    .consideringAllDependencies()
    .layer("Controller").definedBy("..controller..")
    .layer("Service").definedBy("..service..")
    .layer("Repository").definedBy("..repository..")
    .layer("Domain").definedBy("..domain..")
    .whereLayer("Controller").mayNotBeAccessedByAnyLayer()
    .whereLayer("Service").mayOnlyBeAccessedByLayers("Controller")
    .whereLayer("Repository").mayOnlyBeAccessedByLayers("Service")
    .whereLayer("Domain").mayOnlyBeAccessedByLayers("Service", "Repository", "Controller");
```

This single test replaces paragraphs of documentation. And it never forgets to check.

## Testing Verticals in Spring Boot

If you've moved beyond layers to a vertical slice architecture (which I increasingly prefer), you need different rules. Each feature or bounded context is a vertical that should be self-contained:

```java
@ArchTest
static final ArchRule slices_should_not_depend_on_each_other =
    slices()
        .matching("com.example.orderservice.(*)..")
        .should().notDependOnEachOther();
```

This says: each top-level package under `orderservice` is a slice, and slices shouldn't depend on each other. If the `shipping` package imports something from the `billing` package, the test fails. If they need to communicate, they should go through a shared contract or event.

You can also be more selective:

```java
@ArchTest
static final ArchRule order_should_not_depend_on_shipping =
    noClasses()
        .that().resideInAPackage("..order..")
        .should().dependOnClassesThat()
        .resideInAPackage("..shipping..");
```

## Module Boundaries

In a multi-module Maven or Gradle project, dependencies between modules are controlled by the build file. But within a module, packages can reach into each other freely. ArchUnit lets you enforce boundaries within a module.

The pattern I use for domain-driven design:

```java
@ArchTest
static final ArchRule domain_should_not_depend_on_infrastructure =
    noClasses()
        .that().resideInAPackage("..domain..")
        .should().dependOnClassesThat()
        .resideInAnyPackage("..infrastructure..", "..adapter..", "..controller..");

@ArchTest
static final ArchRule domain_should_not_use_spring =
    noClasses()
        .that().resideInAPackage("..domain..")
        .should().dependOnClassesThat()
        .resideInAPackage("org.springframework..");

@ArchTest
static final ArchRule domain_should_not_use_jpa_annotations =
    noClasses()
        .that().resideInAPackage("..domain..")
        .should().beAnnotatedWith("jakarta.persistence.Entity")
        .orShould().beAnnotatedWith("jakarta.persistence.Table");
```

That last one is contentious. Some people put JPA annotations on domain entities and call it pragmatic. I used to be in that camp until I had to swap Postgres for MongoDB on one module and realized my "domain" was coupled to JPA. Now the domain is clean and the infrastructure layer handles the mapping.

## Naming and Annotation Conventions

Beyond dependencies, ArchUnit enforces conventions:

```java
@ArchTest
static final ArchRule controllers_should_be_suffixed =
    classes()
        .that().areAnnotatedWith(RestController.class)
        .should().haveSimpleNameEndingWith("Controller");

@ArchTest
static final ArchRule services_should_be_annotated =
    classes()
        .that().resideInAPackage("..service..")
        .and().areNotInterfaces()
        .should().beAnnotatedWith(Service.class)
        .orShould().beAnnotatedWith(Component.class);

@ArchTest
static final ArchRule no_field_injection =
    noFields()
        .should().beAnnotatedWith(Autowired.class)
        .because("Use constructor injection instead of field injection");
```

The last one is my favorite. Field injection is a code smell that makes testing harder. Instead of catching it in code review every time, the test catches it every time.

## Preventing Architectural Drift

Architectural drift is slow. It doesn't happen in one PR. It happens over months, one small violation at a time. "Just this once, I'll access the repository from the controller because it's a simple query." Then someone copies that pattern. Then it's everywhere.

ArchUnit prevents drift by making violations immediate and visible. But there's a practical concern: when you add ArchUnit to an existing codebase, you'll have hundreds of violations on day one. You can't fix them all at once.

The freeze feature handles this:

```java
@ArchTest
static final ArchRule no_cycles =
    FreezingArchRule.freeze(
        slices().matching("com.example.orderservice.(*)..").should().beFreeOfCycles()
    );
```

`FreezingArchRule` records existing violations in a file (`archunit_store/` by default). Those violations are ignored. New violations fail the build. This lets you adopt ArchUnit incrementally: freeze what exists, prevent new violations, fix the frozen ones when you have time.

## My Standard Test Suite

Here's the test class I start with on every new Spring Boot project:

```java
@AnalyzeClasses(packages = "com.example", importOptions = ImportOption.DoNotIncludeTests.class)
class ArchitectureTest {

    @ArchTest
    static final ArchRule no_field_injection =
        noFields().should().beAnnotatedWith(Autowired.class);

    @ArchTest
    static final ArchRule no_jodatime =
        noClasses().should().dependOnClassesThat()
            .resideInAPackage("org.joda.time..");

    @ArchTest
    static final ArchRule no_java_util_logging =
        noClasses().should().dependOnClassesThat()
            .haveFullyQualifiedName("java.util.logging.Logger");

    @ArchTest
    static final ArchRule exceptions_should_live_in_domain =
        classes().that().areAssignableTo(RuntimeException.class)
            .should().resideInAPackage("..domain..")
            .orShould().resideInAPackage("..exception..");

    @ArchTest
    static final ArchRule no_generic_exceptions =
        noClasses().should().throwGenericExceptions();
}
```

No field injection, no Joda-Time (use `java.time`), no `java.util.logging` (use SLF4J), exceptions in the right place, no throwing raw `Exception` or `RuntimeException`.

These aren't opinions. They're standards. And now they're enforced by tests, not by memory.

## The Real Value

ArchUnit isn't about being an architecture police. It's about making the implicit explicit. Every team has rules about how code should be structured. Most of those rules are communicated verbally, documented poorly, and enforced inconsistently.

Writing them as tests costs you an hour. That hour saves you dozens of hours of code review discussions, refactoring sessions, and "wait, why does this controller talk to the database directly?" conversations.

If you're not testing your architecture, your architecture is just a suggestion.
