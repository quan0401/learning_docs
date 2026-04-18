# Documentation Index — Learning Path

A progressive path from Java fundamentals through Spring to advanced reactive patterns.
Work through the tiers in order — each builds on the previous one.

If you're already comfortable with Java (collections, generics, exceptions, streams), **skip to Tier 1 (Spring Foundation)**.
If you come from TypeScript or another language, start at **Tier 0** below.

---

## Tier 0 — Java Fundamentals (for TypeScript Developers)

Skip this tier if you already write Java daily. Otherwise, these docs give you the mental model shifts that matter most when moving from TS/JS to Java.

1. [Java Type System for TypeScript Developers](java-fundamentals/type-system-for-ts-devs.md) — nominal vs structural types, generics + erasure, nullability, `var`, records, enums _(2026-04-17)_
2. [Optional Deep Dive](java-fundamentals/optional-deep-dive.md) — `Optional<T>` correct usage, `map` vs `flatMap`, `orElse` vs `orElseGet`, when NOT to use _(2026-04-17)_
3. [Collections and Streams](java-fundamentals/collections-and-streams.md) — `List`/`Set`/`Map` hierarchy, Stream API vs JS array methods, Collectors, method references _(2026-04-17)_
4. [Functional Interfaces and Lambdas](java-fundamentals/functional-interfaces-and-lambdas.md) — `Function`/`Predicate`/`Consumer`/`Supplier`, method refs, composition, checked-exception traps _(2026-04-17)_
5. [Exceptions and Error Handling](java-fundamentals/exceptions-and-error-handling.md) — checked vs unchecked, try-with-resources, lambda + checked-exception trap _(2026-04-17)_
6. [Equality and Identity](java-fundamentals/equality-and-identity.md) — `==` vs `.equals()`, `hashCode` contract, `Comparable` vs `Comparator` _(2026-04-17)_
7. [Modern Java Features (17–21)](java-fundamentals/modern-java-features.md) — records, sealed types, pattern matching, `var`, text blocks, virtual threads _(2026-04-17)_
8. [Date and Time API (java.time)](java-fundamentals/date-and-time-api.md) — `Instant`, `LocalDateTime`, `ZonedDateTime`, `Duration`, formatting, timezones, Jackson integration _(2026-04-17)_
9. [Lombok and Boilerplate Reduction](java-fundamentals/lombok-and-boilerplate.md) — `@Data`, `@Builder`, `@RequiredArgsConstructor`, `@Slf4j`, records vs Lombok _(2026-04-17)_
10. [Common Design Patterns in Java](java-fundamentals/common-design-patterns.md) — Builder, Factory, Strategy, Proxy, Template Method, Observer — with Spring examples _(2026-04-17)_
11. [Concurrency Basics](java-fundamentals/concurrency-basics.md) — threads, `ExecutorService`, `CompletableFuture`, virtual threads, `synchronized`/`volatile`/atomics _(2026-04-17)_
12. [Virtual Threads in Java — Project Loom, JEP 444, and the Return of Thread-per-Request](java-fundamentals/virtual-threads.md) — Project Loom, JEP 444, pinning, and when virtual threads beat reactive _(2026-04-17)_
13. [Structured Concurrency in Java](java-fundamentals/structured-concurrency.md) — StructuredTaskScope, ScopedValue, and the modern fork/join model _(2026-04-17)_
14. [Structured Concurrency Before Project Loom](java-fundamentals/structured-concurrency-before-loom.md) — pre-Java-21 approaches, Trio, Kotlin, Swift, Reactor, and what Java borrowed _(2026-04-17)_
15. [Build Tools and JVM](java-fundamentals/build-tools-and-jvm.md) — Gradle/Maven, classpath, JAR packaging, JIT, GC, Docker for Java _(2026-04-17)_

---

## Tier 1 — Spring Foundation

Start here for Spring. Understand how Spring works before touching anything else.

16. [Spring Fundamentals — IoC, DI, AOP, Auto-Configuration, and the Ecosystem](spring-fundamentals.md) — IoC container, dependency injection, bean lifecycle, AOP proxies, the `@SpringBootApplication` breakdown _(2026-04-16)_
17. [Spring AOP Deep Dive](spring-aop-deep-dive.md) — custom aspects, pointcut expressions, advice types, proxy limitations _(2026-04-17)_
18. [Spring Expression Language (SpEL)](spring-expression-language.md) — `#{...}` expressions, `@Value`, `@PreAuthorize`, `@Cacheable`, security SpEL _(2026-04-17)_
19. [Spring Startup Lifecycle](spring-startup-lifecycle.md) — `@PostConstruct`, `ApplicationRunner`, `ApplicationReadyEvent`, `SmartLifecycle`, runners _(2026-04-17)_
20. [Virtual Threads and Spring Boot](spring-virtual-threads.md) — `spring.threads.virtual.enabled`, @Async, @Scheduled, MVC migration, JDBC + virtual threads, pinning _(2026-04-18)_
21. [The javac `-parameters` Flag — Why Local Spring Boot Differs from Deployed](javac-parameters-flag.md) — subtle compiler flag that affects parameter-name reflection _(2026-04-15)_

---

## Tier 2 — Configuration

How to tell Spring what to do without hardcoding anything.

22. [Java @Configuration Classes in Spring Boot 3.x/4.x](configurations/java-bean-config.md) — `@Configuration`, `@Bean` lifecycle, conditional config patterns _(2026-04-15)_
23. [Externalized Configuration in Spring Boot 3.x/4.x](configurations/externalized-config.md) — property sources, YAML, environment variables, `@ConfigurationProperties` _(2026-04-15)_
24. [Spring Profiles and the Environment Abstraction](configurations/environment-and-profiles.md) — `@Profile`, `Environment` API, profile groups, `EnvironmentPostProcessor` _(2026-04-17)_
25. [Conditional Beans and Custom Auto-Configuration in Spring Boot](configurations/custom-auto-configuration.md) — all `@Conditional` annotations, writing your own starters _(2026-04-17)_
26. [Docker and Deployment](configurations/docker-and-deployment.md) — layered JARs, multi-stage Dockerfile, JVM flags for containers, GraalVM native, K8s probes _(2026-04-17)_

---

## Tier 3 — Web Layer (Building REST APIs)

Build HTTP endpoints, handle input, return responses, manage errors.

27. [REST Controller Patterns in Spring Boot — MVC and WebFlux](web-layer/rest-controller-patterns.md) — `@RestController`, functional endpoints, streaming, content negotiation _(2026-04-17)_
28. [Bean Validation in Spring — JSR 380, @Valid, Custom Validators](validation/bean-validation.md) — constraint annotations, validation groups, custom validators _(2026-04-17)_
29. [Exception Handling in Spring Boot — @ControllerAdvice, Problem Details, and Error Patterns](validation/exception-handling.md) — `@ExceptionHandler`, RFC 7807 Problem Details, custom exception hierarchies _(2026-04-17)_
30. [Filters, Interceptors, and the Request Processing Pipeline](web-layer/filters-and-interceptors.md) — Servlet `Filter`, `HandlerInterceptor`, `WebFilter`, `ExchangeFilterFunction` _(2026-04-17)_
31. [Spring MVC Fundamentals](web-layer/spring-mvc-fundamentals.md) — DispatcherServlet, HandlerMapping/Adapter, ViewResolver, servlet stack vs WebFlux _(2026-04-17)_
32. [MVC Controllers, Forms, and Validation](web-layer/mvc-controllers-forms-validation.md) — `@Controller`, `@ModelAttribute`, `BindingResult`, PRG pattern, flash attributes _(2026-04-17)_
33. [Thymeleaf and Server-Side Views](web-layer/thymeleaf-and-views.md) — `th:*` attributes, fragments, layouts, Spring integration, Security dialect _(2026-04-17)_
34. [Session Management](web-layer/session-management.md) — `HttpSession`, `@SessionAttributes`, Spring Session + Redis, sticky vs externalized _(2026-04-17)_
35. [Static Resources and i18n](web-layer/static-resources-and-i18n.md) — resource handlers, cache-busting, `MessageSource`, `LocaleResolver` _(2026-04-17)_

---

## Tier 3.5 — Realtime APIs

Server push, streaming, and bidirectional channels for both MVC and WebFlux.

36. [Server-Sent Events and HTTP Streaming](realtime/sse-and-streaming.md) — SSE in WebFlux + MVC, NDJSON, heartbeats, EventSource, proxy gotchas _(2026-04-18)_
37. [WebSockets in Spring](realtime/websockets.md) — `WebSocketHandler` (WebFlux) + STOMP/`@MessageMapping` (MVC), security, backpressure _(2026-04-18)_

---

## Tier 4 — Data Access & Persistence

Talk to databases, manage transactions, optimize queries.

38. [Spring Data Repository Interfaces — The Data Access Abstraction](data-repositories/repository-interfaces.md) — repository hierarchy, derived queries, projections _(2026-04-17)_
39. [Queries, Pagination, and Advanced Spring Data Features](data-repositories/queries-and-pagination.md) — `@Query`, `Pageable`, Specifications, auditing, `@EntityGraph` _(2026-04-17)_
40. [JPA Relationships](data-repositories/jpa-relationships.md) — `@OneToMany`/`@ManyToOne`/`@ManyToMany`, fetch strategies, N+1 problem, cascade types _(2026-04-17)_
41. [Database Configuration in Spring Boot — MongoDB Reactive, R2DBC, and JPA](configurations/database-config.md) — connection pools, multiple datasources, driver selection _(2026-04-15)_
42. [JPA Transactions in Spring Boot — With Reactive WebFlux Caveats](jpa-transactions.md) — `@Transactional`, propagation, isolation, self-invocation trap _(2026-04-16)_
43. [JPA Transaction Propagation and Isolation](jpa-transaction-propagation.md) — all 7 propagation levels, 5 isolation levels, rollback rules, `TransactionTemplate`, reactive caveat _(2026-04-17)_
44. [Cache Configuration in Spring Boot — Caffeine, Redis, and Reactive Caching](configurations/cache-config.md) — `@Cacheable`, cache managers, eviction strategies _(2026-04-15)_
45. [R2DBC Deep Dive — Reactive SQL in Spring WebFlux](data-repositories/r2dbc-deep-dive.md) — `DatabaseClient`, `R2dbcRepository`, drivers, pooling, migrations _(2026-04-18)_
46. [Building a Reactive Data Layer](data-repositories/reactive-data-layer.md) — mixing Mongo + R2DBC + Redis, pagination, auditing, decision guide _(2026-04-18)_
47. [Reactive Transactions](data-repositories/reactive-transactions.md) — `TransactionalOperator`, `@Transactional` on `Mono`/`Flux`, rollback semantics _(2026-04-18)_

---

## Tier 5 — Security

Protect the application: who is the caller, and what can they do?

48. [Spring Security Architecture — The Filter Chain](security/security-filter-chain.md) — `SecurityFilterChain`, CSRF, CORS, security headers _(2026-04-17)_
49. [Authentication and Authorization in Spring Security](security/authentication-authorization.md) — `UserDetailsService`, `@PreAuthorize`, role hierarchy, session management _(2026-04-17)_
50. [OAuth2 and JWT in Spring Security — Resource Server, Client, and Token Handling](security/oauth2-jwt.md) — JWT validation, OAuth2 Resource Server, Client Credentials, testing _(2026-04-17)_

---

## Tier 6 — Testing

Verify correctness at every layer.

51. [Spring Boot Testing Fundamentals — @SpringBootTest, Test Slices, and Mocking](testing/spring-boot-test-basics.md) — `@SpringBootTest`, `@WebFluxTest`, `@MockBean`, Mockito, StepVerifier _(2026-04-17)_
52. [Web Layer Testing — WebTestClient, MockMvc, and Endpoint Verification](testing/web-layer-testing.md) — `WebTestClient` fluent API, `MockMvc`, streaming tests _(2026-04-17)_
53. [Testcontainers — Integration Testing with Real Infrastructure](testing/testcontainers.md) — Docker containers for MongoDB, PostgreSQL, CI integration _(2026-04-17)_

---

## Tier 7 — Async, Events & Scheduling

Decouple components, run work in the background, schedule recurring tasks.

54. [Spring Application Events — Publishing, Listening, and Event-Driven Patterns](events-async/application-events.md) — `@EventListener`, `@TransactionalEventListener`, event-driven design _(2026-04-17)_
55. [Async Processing in Spring — @Async, Executors, and Background Tasks](events-async/async-processing.md) — `@Async`, custom executors, virtual threads, self-invocation trap _(2026-04-17)_
56. [Task Scheduling in Spring — @Scheduled, Cron Expressions, and Distributed Scheduling](events-async/scheduling.md) — `@Scheduled`, cron syntax, ShedLock, Quartz _(2026-04-17)_

---

## Tier 7.5 — Cross-Cutting Concerns

Patterns that apply everywhere: logging, observability, correlation.

57. [Logging in Java and Spring Boot](logging.md) — SLF4J/Logback architecture, MDC, structured JSON logs, reactive MDC propagation, anti-patterns _(2026-04-17)_
58. [Actuator Deep Dive](actuator-deep-dive.md) — custom `HealthIndicator`, Micrometer metrics, custom `@Endpoint`, K8s probes, securing endpoints _(2026-04-17)_

---

## Tier 7.6 — Messaging & Event-Driven Architecture

Move beyond in-process events to message brokers and distributed event flows.

59. [Event-Driven Patterns](messaging/event-driven-patterns.md) — event sourcing, CQRS, outbox, saga, idempotency, DLQ, broker-agnostic _(2026-04-17)_
60. [Reactive Kafka (reactor-kafka)](messaging/reactive-kafka.md) — `KafkaSender`/`KafkaReceiver`, backpressure, offset management, EOS _(2026-04-17)_
61. [Spring for Apache Kafka](messaging/spring-kafka.md) — `@KafkaListener`, `KafkaTemplate`, `@RetryableTopic`, DLT handling _(2026-04-17)_

---

## Tier 8 — Reactive Programming & WebFlux

Non-blocking, back-pressured, event-loop-based web stack. Learn this last — it reuses everything above but with reactive types.

62. [Reactive Programming in Java with Project Reactor and Spring WebFlux](reactive-programming-java.md) — comprehensive guide to `Mono`, `Flux`, operators, and WebFlux fundamentals _(2026-04-14)_
63. [Reactor Operator Catalog](reactive/operator-catalog.md) — 40+ operators by category: creating, transforming, filtering, combining, aggregating, time-based _(2026-04-17)_
64. [Synchronous vs Asynchronous Transformation in Project Reactor](sync-vs-async-transformation.md) — `map` vs `flatMap`, when each is appropriate _(2026-04-16)_
65. [Reactor Schedulers and Threading](reactive/schedulers-and-threading.md) — `publishOn` vs `subscribeOn`, `parallel`/`boundedElastic`/`single`, virtual threads, BlockHound _(2026-04-17)_
66. [Wrapping Blocking JPA Calls in a Reactive Chain](reactive-blocking-jpa-pattern.md) — how to isolate blocking code inside reactive pipelines _(2026-04-14)_
67. [WebClient Configuration in Spring WebFlux](configurations/webclient-config.md) — reactive HTTP client, timeouts, connection pool, filters _(2026-04-15)_
68. [Server and HTTP Configuration in Spring WebFlux](configurations/server-http-config.md) — Reactor Netty server tuning, compression, HTTP/2 _(2026-04-15)_
69. [Fixing WebClient DNS Resolution Failures — Netty vs JDK Resolver](webclient-netty-dns-resolver-fix.md) — production-grade DNS resolver fix _(2026-04-16)_
70. [Reactive Observability — Tracing, Logging, Metrics, and Health in Spring WebFlux](reactive-observability.md) — Micrometer, distributed tracing, structured logging _(2026-04-15)_
71. [Advanced Reactive Programming in Java — Beyond the Basics](reactive-advanced-topics.md) — backpressure, schedulers, context propagation, error recovery _(2026-04-15)_

---

## Tier 9 — Architecture & Meta

How to structure and reason about Spring Boot codebases.

72. [Project Structure and Architecture](architecture/project-structure.md) — layered vs package-by-feature vs hexagonal, clean architecture, anti-patterns _(2026-04-17)_

---

## Quick Reference by Topic

If you need to look something up rather than learn linearly, here is the same content grouped by topic:

### Java Fundamentals (for TS Developers)
- [Type System for TS Devs](java-fundamentals/type-system-for-ts-devs.md)
- [Optional Deep Dive](java-fundamentals/optional-deep-dive.md)
- [Collections and Streams](java-fundamentals/collections-and-streams.md)
- [Functional Interfaces and Lambdas](java-fundamentals/functional-interfaces-and-lambdas.md)
- [Exceptions and Error Handling](java-fundamentals/exceptions-and-error-handling.md)
- [Equality and Identity](java-fundamentals/equality-and-identity.md)
- [Modern Java Features](java-fundamentals/modern-java-features.md)
- [Date and Time API](java-fundamentals/date-and-time-api.md)
- [Lombok and Boilerplate](java-fundamentals/lombok-and-boilerplate.md)
- [Common Design Patterns](java-fundamentals/common-design-patterns.md)
- [Concurrency Basics](java-fundamentals/concurrency-basics.md)
- [Virtual Threads (Project Loom)](java-fundamentals/virtual-threads.md)
- [Structured Concurrency](java-fundamentals/structured-concurrency.md)
- [Structured Concurrency Before Project Loom](java-fundamentals/structured-concurrency-before-loom.md)
- [Build Tools and JVM](java-fundamentals/build-tools-and-jvm.md)

### Core
- [Spring Fundamentals](spring-fundamentals.md)
- [Spring AOP Deep Dive](spring-aop-deep-dive.md)
- [Spring Expression Language (SpEL)](spring-expression-language.md)
- [Spring Startup Lifecycle](spring-startup-lifecycle.md)
- [Virtual Threads and Spring Boot](spring-virtual-threads.md)
- [javac -parameters flag](javac-parameters-flag.md)

### Configuration
- [Java @Configuration Classes](configurations/java-bean-config.md)
- [Externalized Configuration](configurations/externalized-config.md)
- [Profiles & Environment](configurations/environment-and-profiles.md)
- [Custom Auto-Configuration](configurations/custom-auto-configuration.md)
- [Docker and Deployment](configurations/docker-and-deployment.md)

### Web Layer — REST
- [REST Controller Patterns](web-layer/rest-controller-patterns.md)
- [Filters & Interceptors](web-layer/filters-and-interceptors.md)
- [Bean Validation](validation/bean-validation.md)
- [Exception Handling](validation/exception-handling.md)

### Web Layer — Spring MVC (Servlet Stack)
- [Spring MVC Fundamentals](web-layer/spring-mvc-fundamentals.md)
- [MVC Controllers, Forms, and Validation](web-layer/mvc-controllers-forms-validation.md)
- [Thymeleaf and Server-Side Views](web-layer/thymeleaf-and-views.md)
- [Session Management](web-layer/session-management.md)
- [Static Resources and i18n](web-layer/static-resources-and-i18n.md)

### Data Access
- [Repository Interfaces](data-repositories/repository-interfaces.md)
- [Queries & Pagination](data-repositories/queries-and-pagination.md)
- [JPA Relationships](data-repositories/jpa-relationships.md)
- [Database Configuration](configurations/database-config.md)
- [JPA Transactions](jpa-transactions.md)
- [JPA Transaction Propagation and Isolation](jpa-transaction-propagation.md)
- [Cache Configuration](configurations/cache-config.md)
- [R2DBC Deep Dive](data-repositories/r2dbc-deep-dive.md)
- [Building a Reactive Data Layer](data-repositories/reactive-data-layer.md)
- [Reactive Transactions](data-repositories/reactive-transactions.md)

### Realtime APIs
- [Server-Sent Events and HTTP Streaming](realtime/sse-and-streaming.md)
- [WebSockets in Spring](realtime/websockets.md)

### Security
- [Security Filter Chain](security/security-filter-chain.md)
- [Authentication & Authorization](security/authentication-authorization.md)
- [OAuth2 & JWT](security/oauth2-jwt.md)

### Testing
- [Spring Boot Test Basics](testing/spring-boot-test-basics.md)
- [Web Layer Testing](testing/web-layer-testing.md)
- [Testcontainers](testing/testcontainers.md)

### Async & Events
- [Application Events](events-async/application-events.md)
- [Async Processing](events-async/async-processing.md)
- [Task Scheduling](events-async/scheduling.md)

### Cross-Cutting
- [Logging in Java and Spring Boot](logging.md)
- [Actuator Deep Dive](actuator-deep-dive.md)

### Messaging & Event-Driven
- [Event-Driven Patterns](messaging/event-driven-patterns.md)
- [Reactive Kafka (reactor-kafka)](messaging/reactive-kafka.md)
- [Spring for Apache Kafka](messaging/spring-kafka.md)

### Architecture & Meta
- [Project Structure and Architecture](architecture/project-structure.md)

### Reactive / WebFlux
- [Reactive Programming in Java](reactive-programming-java.md)
- [Reactor Operator Catalog](reactive/operator-catalog.md)
- [Sync vs Async Transformation](sync-vs-async-transformation.md)
- [Reactor Schedulers and Threading](reactive/schedulers-and-threading.md)
- [Blocking JPA in Reactive Chain](reactive-blocking-jpa-pattern.md)
- [WebClient Configuration](configurations/webclient-config.md)
- [Server & HTTP Configuration](configurations/server-http-config.md)
- [WebClient DNS Resolver Fix](webclient-netty-dns-resolver-fix.md)
- [Reactive Observability](reactive-observability.md)
- [Advanced Reactive Topics](reactive-advanced-topics.md)
