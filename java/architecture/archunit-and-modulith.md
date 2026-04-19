---
title: "ArchUnit and Spring Modulith — Architecture as Tests"
date: 2026-04-19
updated: 2026-04-19
tags: [architecture, archunit, spring-modulith, testing, modularity]
---

# ArchUnit and Spring Modulith — Architecture as Tests

**Date:** 2026-04-19 | **Updated:** 2026-04-19
**Tags:** `architecture` `archunit` `spring-modulith` `testing` `modularity`

## Table of Contents

- [Summary](#summary)
- [Why Architecture Rots Without Tests](#why-architecture-rots-without-tests)
- [ArchUnit — Unit Tests for Your Architecture](#archunit--unit-tests-for-your-architecture)
- [Layer Rules](#layer-rules)
- [Naming and Dependency Rules](#naming-and-dependency-rules)
- [Cyclic Dependency Detection](#cyclic-dependency-detection)
- [Spring Modulith — Modular Monolith with Explicit Boundaries](#spring-modulith--modular-monolith-with-explicit-boundaries)
- [From Modulith to Microservices](#from-modulith-to-microservices)
- [Related](#related)
- [References](#references)

---

## Summary

Architecture documents rot. Diagrams on Confluence go stale within a sprint. The only durable enforcement is code: **tests that fail CI when architectural invariants break**. [ArchUnit](https://www.archunit.org/) lets you write JUnit tests like "no class in `controller` may depend on `repository`" or "no class may use `java.util.Date`", and they run on every PR. [Spring Modulith](https://spring.io/projects/spring-modulith) takes it further for Spring apps: it defines **modules** as top-level packages, enforces boundaries between them, publishes and consumes domain events across modules without coupling, and generates C4 diagrams from the actual code. Together they turn architecture from aspirational into executable — a compiling build guarantees the system still matches the design. This doc covers the ArchUnit rule DSL, the Modulith model, and how to adopt both without a big-bang refactor.

---

## Why Architecture Rots Without Tests

A typical decay sequence:

1. Architect draws a diagram: `web → service → repository`.
2. Junior dev needs a data lookup from a controller, adds `@Autowired Repository` to the controller. Works.
3. PR reviewer doesn't catch it.
4. Next month, five more controllers cheat.
5. Six months later, "the architecture" is a polite fiction; actual coupling is spaghetti.

ArchUnit would have failed step 2 in CI. Every cheat requires an explicit test-suppression, visible in code review.

---

## ArchUnit — Unit Tests for Your Architecture

Dependency:

```gradle
testImplementation 'com.tngtech.archunit:archunit-junit5:1.3.0'
```

Basic example:

```java
@AnalyzeClasses(packages = "com.example.orders", importOptions = DoNotIncludeTests.class)
class ArchTests {

    @ArchTest
    static final ArchRule controllers_dont_access_repositories = noClasses()
        .that().resideInAPackage("..controller..")
        .should().dependOnClassesThat().resideInAPackage("..repository..");

    @ArchTest
    static final ArchRule no_field_injection = noFields()
        .should().beAnnotatedWith(Autowired.class);

    @ArchTest
    static final ArchRule services_use_constructor_injection = classes()
        .that().resideInAPackage("..service..")
        .should().onlyHaveDependentClassesThat().resideInAnyPackage("..controller..", "..service..");
}
```

Rules fail CI as loud as any other test. Violations are reported with class name, offending line, and which rule fired.

---

## Layer Rules

A layered architecture rule in one declaration:

```java
@ArchTest
static final ArchRule layers = layeredArchitecture().consideringAllDependencies()
    .layer("Controller").definedBy("..controller..")
    .layer("Service").definedBy("..service..")
    .layer("Repository").definedBy("..repository..")
    .whereLayer("Controller").mayNotBeAccessedByAnyLayer()
    .whereLayer("Service").mayOnlyBeAccessedByLayers("Controller")
    .whereLayer("Repository").mayOnlyBeAccessedByLayers("Service");
```

Reads as English; enforces: controllers are leaf dependencies, services are only called by controllers, repositories only by services. Violators can't compile a passing test.

For hexagonal / clean architecture:

```java
@ArchTest
static final ArchRule onion = onionArchitecture()
    .domainModels("..domain.model..")
    .domainServices("..domain.service..")
    .applicationServices("..application..")
    .adapter("persistence", "..adapter.persistence..")
    .adapter("web", "..adapter.web..")
    .adapter("messaging", "..adapter.messaging..");
```

---

## Naming and Dependency Rules

Enforce naming conventions:

```java
@ArchTest
static final ArchRule controllers_named = classes()
    .that().areAnnotatedWith(RestController.class)
    .should().haveSimpleNameEndingWith("Controller");

@ArchTest
static final ArchRule services_in_service_package = classes()
    .that().areAnnotatedWith(Service.class)
    .should().resideInAPackage("..service..");
```

Ban specific APIs:

```java
@ArchTest
static final ArchRule no_date_api = noClasses()
    .should().dependOnClassesThat().areAssignableTo(Date.class)
    .orShould().dependOnClassesThat().areAssignableTo(Calendar.class)
    .because("use java.time instead (see date-and-time-api.md)");

@ArchTest
static final ArchRule no_system_out = noClasses()
    .should().callMethod(System.class, "out")
    .because("use SLF4J");
```

Ban `java.util.logging`, `new Date()`, `SimpleDateFormat` — whatever your team has agreed to avoid.

---

## Cyclic Dependency Detection

The most insidious form of coupling — package A calls B calls A:

```java
@ArchTest
static final ArchRule no_cycles = slices()
    .matching("com.example.orders.(*)..")
    .should().beFreeOfCycles();
```

Runs a Tarjan-style SCC check. Fails with the exact cycle. Do this on day one of any project; cycles are much easier to prevent than remove.

---

## Spring Modulith — Modular Monolith with Explicit Boundaries

[Spring Modulith](https://docs.spring.io/spring-modulith/reference/) (formerly Moduliths) treats a top-level Spring Boot package as a **module**. Modules expose a public API package; internal packages are closed to other modules. Cross-module communication uses Spring application events.

Dependency:

```gradle
implementation 'org.springframework.modulith:spring-modulith-starter-core'
implementation 'org.springframework.modulith:spring-modulith-events-jpa'
testImplementation 'org.springframework.modulith:spring-modulith-starter-test'
```

Structure:

```text
com.example.app
├── order/                  # module
│   ├── Order.java          # public
│   ├── OrderService.java   # public
│   └── internal/           # inaccessible to other modules
│       ├── OrderRepository.java
│       └── OrderEntity.java
├── inventory/              # module
│   └── ...
└── notification/           # module
    └── ...
```

Test:

```java
class ModularityTests {

    ApplicationModules modules = ApplicationModules.of(Application.class);

    @Test
    void verifiesModules() {
        modules.verify();                    // fails if any module violates boundary
    }

    @Test
    void writesDocs() {
        new Documenter(modules)
            .writeModulesAsPlantUml()        // C4 diagrams
            .writeIndividualModulesAsPlantUml()
            .writeAggregatingDocument();
    }
}
```

Cross-module communication via events:

```java
// order module publishes
@Service
@RequiredArgsConstructor
public class OrderService {
    private final ApplicationEventPublisher events;

    public Order place(Cmd cmd) {
        Order o = save(cmd);
        events.publishEvent(new OrderPlaced(o.getId(), o.getTotal()));
        return o;
    }
}

// inventory module consumes
@Component
class InventoryListener {
    @ApplicationModuleListener
    void on(OrderPlaced event) {
        reserveStock(event.orderId());
    }
}
```

`@ApplicationModuleListener` is transactional, async, and persistent (events survive restart via the events JPA module). This is the [Outbox pattern](../graphql/multi-database-patterns.md#outbox-pattern) built into Spring.

---

## From Modulith to Microservices

Spring Modulith is an explicit on-ramp: start as a modular monolith, extract modules to services once boundaries are proven. Each module already communicates via events, not direct calls — the only refactor is moving the event transport from in-process to Kafka.

Modulith's [production-ready](https://docs.spring.io/spring-modulith/reference/production-ready.html) features include:

- Event externalization to Kafka / RabbitMQ / AMQP.
- Observability (Micrometer) per module.
- Test slices per module.

Decision guide: start as a modular monolith unless you know you need microservices day one. Modulith is what lets you make that choice later without a rewrite. See [project-structure.md](project-structure.md) for the broader architecture context.

---

## Related

- [Project Structure and Architecture](project-structure.md) — broader architecture context.
- [Event-Driven Patterns](../messaging/event-driven-patterns.md) — event-driven patterns apply inside Modulith.
- [Federated GraphQL with Polyglot Persistence](../graphql/multi-database-patterns.md) — the microservices endpoint of the Modulith road.
- [DDD Tactical Patterns](ddd-tactical-patterns.md) — modules often map to DDD bounded contexts.
- [Spring Boot Testing Fundamentals](../testing/spring-boot-test-basics.md) — architecture tests are part of the test suite.
- [Distributed Systems Primer](distributed-systems-primer.md) — what you're deferring until you need it.

---

## References

- [ArchUnit documentation](https://www.archunit.org/userguide/html/000_Index.html)
- [ArchUnit GitHub](https://github.com/TNG/ArchUnit)
- [Spring Modulith reference](https://docs.spring.io/spring-modulith/reference/)
- [Spring Modulith GitHub](https://github.com/spring-projects/spring-modulith)
- [Oliver Drotbohm — Building Modular Monoliths](https://odrotbohm.de/)
- [Simon Brown — The Modular Monolith](https://simonbrown.je/) / C4 model.
- [Sam Newman — Monolith to Microservices](https://samnewman.io/books/monolith-to-microservices/)
