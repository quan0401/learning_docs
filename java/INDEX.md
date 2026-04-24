# Documentation Index — Learning Path

A progressive path from Java fundamentals through Spring to advanced reactive patterns.
Work through the tiers in order — each builds on the previous one. Cross-references to the [Networking learning path](../networking/INDEX.md) for protocol and infrastructure topics, and the [Kubernetes learning path](../kubernetes/INDEX.md) for platform-level K8s understanding.

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
    - [Creational Patterns Deep Dive](design-patterns/creational-patterns.md) — Step Builder, Abstract Factory, Prototype, Object Pool _(2026-04-19)_
    - [Structural Patterns Deep Dive](design-patterns/structural-patterns.md) — Decorator chains, Composite, Facade, Flyweight _(2026-04-19)_
    - [Behavioral Patterns Deep Dive](design-patterns/behavioral-patterns.md) — Command, State machine, Visitor (modern Java), Mediator, Iterator, Memento _(2026-04-19)_
    - [Enterprise Patterns Deep Dive](design-patterns/enterprise-patterns.md) — Service Layer, Specification, DTO Assembler, Gateway, Unit of Work _(2026-04-19)_
11. [Concurrency Basics](java-fundamentals/concurrency/concurrency-basics.md) — threads, `ExecutorService`, `CompletableFuture`, virtual threads, `synchronized`/`volatile`/atomics _(2026-04-17)_
11a. [Java Multithreading Deep Dive — Memory Model, Locks, Synchronizers, and Thread Pools](java-fundamentals/concurrency/multithreading-deep-dive.md) — JMM happens-before, `ReentrantLock`/`StampedLock`, `CountDownLatch`/`Semaphore`/`Phaser`, `ThreadPoolExecutor` internals, deadlock diagnosis _(2026-04-18)_
11b. [CompletableFuture Deep Dive — Composition, Errors, Timeouts, Interop](java-fundamentals/concurrency/completablefuture-deep-dive.md) — `thenApply`/`thenCompose`/`thenCombine`, `allOf`/`anyOf`, exception handling, `orTimeout`, Reactor interop _(2026-04-24)_
11c. [Interruption and Cancellation in Java — The InterruptedException Idiom](java-fundamentals/concurrency/interruption-and-cancellation.md) — cooperative cancellation, `Thread.interrupt`, `ExecutorService.shutdown`, `Future.cancel`, poison pills _(2026-04-24)_
11d. [Concurrent Collections — ConcurrentHashMap, LongAdder, CAS, ABA Problem](java-fundamentals/concurrency/concurrent-collections.md) — `computeIfAbsent`/`merge`, `LongAdder`, `VarHandle`, `AtomicStampedReference`, memory-ordering modes _(2026-04-24)_
11e. [ThreadLocal and Context Propagation — Leaks, ScopedValue, Reactor Context](java-fundamentals/concurrency/threadlocal-and-context.md) — TL leaks, `InheritableThreadLocal`, `ScopedValue` (JDK 25), Reactor `Context`, OpenTelemetry `Context` _(2026-04-24)_
11f. [ForkJoinPool and Parallel Streams — Work Stealing, RecursiveTask, commonPool](java-fundamentals/concurrency/forkjoinpool-and-parallel-streams.md) — work-stealing deques, `RecursiveTask`, `ManagedBlocker`, parallel-stream pitfalls _(2026-04-24)_
11g. [Concurrency Debugging Playbook — Thread Dumps, JFR, async-profiler, jcstress](java-fundamentals/concurrency/concurrency-debugging.md) — `jstack`/`jcmd`, deadlock detection, lock-contention flame graphs, JMM correctness testing _(2026-04-24)_
11h. [Producer-Consumer Patterns — BlockingQueue, Backpressure, Disruptor](java-fundamentals/concurrency/producer-consumer-patterns.md) — queue variants, backpressure, graceful shutdown, Disruptor ring buffer, reactive comparison _(2026-04-24)_
12. [Virtual Threads in Java — Project Loom, JEP 444, and the Return of Thread-per-Request](java-fundamentals/concurrency/virtual-threads.md) — Project Loom, JEP 444, pinning, and when virtual threads beat reactive _(2026-04-17)_
13. [Structured Concurrency in Java](java-fundamentals/concurrency/structured-concurrency.md) — StructuredTaskScope, ScopedValue, and the modern fork/join model _(2026-04-17)_
14. [Structured Concurrency Before Project Loom](java-fundamentals/concurrency/structured-concurrency-before-loom.md) — pre-Java-21 approaches, Trio, Kotlin, Swift, Reactor, and what Java borrowed _(2026-04-17)_
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
36. [Scaling Spring MVC Before Virtual Threads — Async Servlet, Tuning, and Resilience](web-layer/mvc-high-throughput.md) — `Callable`/`DeferredResult`/`CompletableFuture`, Tomcat tuning, HikariCP sizing, Resilience4j bulkheads _(2026-04-18)_

---

## Tier 3.5 — Realtime APIs

Server push, streaming, and bidirectional channels for both MVC and WebFlux.

37. [Server-Sent Events and HTTP Streaming](realtime/sse-and-streaming.md) — SSE in WebFlux + MVC, NDJSON, heartbeats, EventSource, proxy gotchas _(2026-04-18)_
38. [WebSockets in Spring](realtime/websockets.md) — `WebSocketHandler` (WebFlux) + STOMP/`@MessageMapping` (MVC), security, backpressure _(2026-04-18)_

---

## Tier 4 — Data Access & Persistence

Talk to databases, manage transactions, optimize queries.

39. [Spring Data Repository Interfaces — The Data Access Abstraction](data-repositories/repository-interfaces.md) — repository hierarchy, derived queries, projections _(2026-04-17)_
40. [Queries, Pagination, and Advanced Spring Data Features](data-repositories/queries-and-pagination.md) — `@Query`, `Pageable`, Specifications, auditing, `@EntityGraph` _(2026-04-17)_
41. [JPA Relationships](data-repositories/jpa-relationships.md) — `@OneToMany`/`@ManyToOne`/`@ManyToMany`, fetch strategies, N+1 problem, cascade types _(2026-04-17)_
42. [Database Configuration in Spring Boot — MongoDB Reactive, R2DBC, and JPA](configurations/database-config.md) — connection pools, multiple datasources, driver selection _(2026-04-15)_
43. [JPA Transactions in Spring Boot — With Reactive WebFlux Caveats](jpa-transactions.md) — `@Transactional`, propagation, isolation, self-invocation trap _(2026-04-16)_
44. [JPA Transaction Propagation and Isolation](jpa-transaction-propagation.md) — all 7 propagation levels, 5 isolation levels, rollback rules, `TransactionTemplate`, reactive caveat _(2026-04-17)_
45. [Cache Configuration in Spring Boot — Caffeine, Redis, and Reactive Caching](configurations/cache-config.md) — `@Cacheable`, cache managers, eviction strategies _(2026-04-15)_
46. [R2DBC Deep Dive — Reactive SQL in Spring WebFlux](data-repositories/r2dbc-deep-dive.md) — `DatabaseClient`, `R2dbcRepository`, drivers, pooling, migrations _(2026-04-18)_
47. [Building a Reactive Data Layer](data-repositories/reactive-data-layer.md) — mixing Mongo + R2DBC + Redis, pagination, auditing, decision guide _(2026-04-18)_
48. [Reactive Transactions](data-repositories/reactive-transactions.md) — `TransactionalOperator`, `@Transactional` on `Mono`/`Flux`, rollback semantics _(2026-04-18)_

---

## Tier 5 — Security

Protect the application: who is the caller, and what can they do?

49. [Spring Security Architecture — The Filter Chain](security/security-filter-chain.md) — `SecurityFilterChain`, CSRF, CORS, security headers _(2026-04-17)_
50. [Authentication and Authorization in Spring Security](security/authentication-authorization.md) — `UserDetailsService`, `@PreAuthorize`, role hierarchy, session management _(2026-04-17)_
51. [OAuth2 and JWT in Spring Security — Resource Server, Client, and Token Handling](security/oauth2-jwt.md) — JWT validation, OAuth2 Resource Server, Client Credentials, testing _(2026-04-17)_

---

## Tier 6 — Testing

Verify correctness at every layer.

52. [Spring Boot Testing Fundamentals — @SpringBootTest, Test Slices, and Mocking](testing/spring-boot-test-basics.md) — `@SpringBootTest`, `@WebFluxTest`, `@MockBean`, Mockito, StepVerifier _(2026-04-17)_
53. [Web Layer Testing — WebTestClient, MockMvc, and Endpoint Verification](testing/web-layer-testing.md) — `WebTestClient` fluent API, `MockMvc`, streaming tests _(2026-04-17)_
54. [Testcontainers — Integration Testing with Real Infrastructure](testing/testcontainers.md) — Docker containers for MongoDB, PostgreSQL, CI integration _(2026-04-17)_

---

## Tier 7 — Async, Events & Scheduling

Decouple components, run work in the background, schedule recurring tasks.

55. [Spring Application Events — Publishing, Listening, and Event-Driven Patterns](events-async/application-events.md) — `@EventListener`, `@TransactionalEventListener`, event-driven design _(2026-04-17)_
56. [Async Processing in Spring — @Async, Executors, and Background Tasks](events-async/async-processing.md) — `@Async`, custom executors, virtual threads, self-invocation trap _(2026-04-17)_
57. [Task Scheduling in Spring — @Scheduled, Cron Expressions, and Distributed Scheduling](events-async/scheduling.md) — `@Scheduled`, cron syntax, ShedLock, Quartz _(2026-04-17)_

---

## Tier 7.5 — Cross-Cutting Concerns

Patterns that apply everywhere: logging, observability, correlation.

58. [Logging in Java and Spring Boot](logging.md) — SLF4J/Logback architecture, MDC, structured JSON logs, reactive MDC propagation, anti-patterns _(2026-04-17)_
59. [Actuator Deep Dive](actuator-deep-dive.md) — custom `HealthIndicator`, Micrometer metrics, custom `@Endpoint`, K8s probes, securing endpoints _(2026-04-17)_

---

## Tier 7.6 — Messaging & Event-Driven Architecture

Move beyond in-process events to message brokers and distributed event flows.

60. [Event-Driven Patterns](messaging/event-driven-patterns.md) — event sourcing, CQRS, outbox, saga, idempotency, DLQ, broker-agnostic _(2026-04-17)_
61. [Reactive Kafka (reactor-kafka)](messaging/reactive-kafka.md) — `KafkaSender`/`KafkaReceiver`, backpressure, offset management, EOS _(2026-04-17)_
62. [Spring for Apache Kafka](messaging/spring-kafka.md) — `@KafkaListener`, `KafkaTemplate`, `@RetryableTopic`, DLT handling _(2026-04-17)_

---

## Tier 8 — Reactive Programming & WebFlux

Non-blocking, back-pressured, event-loop-based web stack. Learn this last — it reuses everything above but with reactive types.

63. [Reactive Programming in Java with Project Reactor and Spring WebFlux](reactive-programming-java.md) — comprehensive guide to `Mono`, `Flux`, operators, and WebFlux fundamentals _(2026-04-14)_
64. [Reactor Operator Catalog](reactive/operator-catalog.md) — 40+ operators by category: creating, transforming, filtering, combining, aggregating, time-based _(2026-04-17)_
65. [Synchronous vs Asynchronous Transformation in Project Reactor](sync-vs-async-transformation.md) — `map` vs `flatMap`, when each is appropriate _(2026-04-16)_
66. [Reactor Schedulers and Threading](reactive/schedulers-and-threading.md) — `publishOn` vs `subscribeOn`, `parallel`/`boundedElastic`/`single`, virtual threads, BlockHound _(2026-04-17)_
67. [Wrapping Blocking JPA Calls in a Reactive Chain](reactive-blocking-jpa-pattern.md) — how to isolate blocking code inside reactive pipelines _(2026-04-14)_
68. [WebClient Configuration in Spring WebFlux](configurations/webclient-config.md) — reactive HTTP client, timeouts, connection pool, filters _(2026-04-15)_
69. [Server and HTTP Configuration in Spring WebFlux](configurations/server-http-config.md) — Reactor Netty server tuning, compression, HTTP/2 _(2026-04-15)_
70. [Fixing WebClient DNS Resolution Failures — Netty vs JDK Resolver](webclient-netty-dns-resolver-fix.md) — production-grade DNS resolver fix _(2026-04-16)_
71. [Reactive Observability — Tracing, Logging, Metrics, and Health in Spring WebFlux](reactive-observability.md) — Micrometer, distributed tracing, structured logging _(2026-04-15)_
72. [Advanced Reactive Programming in Java — Beyond the Basics](reactive-advanced-topics.md) — backpressure, schedulers, context propagation, error recovery _(2026-04-15)_

---

## Tier 9 — Architecture & Meta

How to structure and reason about Spring Boot codebases.

73. [Project Structure and Architecture](architecture/project-structure.md) — layered vs package-by-feature vs hexagonal, clean architecture, anti-patterns _(2026-04-17)_

---

## Tier 10 — JVM Performance & Garbage Collection

Advanced performance topics. Read after you're comfortable with everything above and need to tune pauses, throughput, or memory in production.

74. [JVM Garbage Collection — Concepts and Mental Model](jvm-gc/concepts.md) — heap layout, generational hypothesis, marking, safepoints, barriers _(2026-04-18)_
75. [JVM Garbage Collectors — Serial, Parallel, CMS, G1, ZGC, Shenandoah](jvm-gc/collectors.md) — which is default per JDK, decision tree, concrete flag recipes _(2026-04-18)_
76. [GC Pause Diagnosis Playbook](jvm-gc/pause-diagnosis.md) — `-Xlog:gc*`, JFR, async-profiler, heap dumps, pathology cookbook _(2026-04-18)_
77. [GC Impact on Reactive, Virtual Threads, and Streaming](jvm-gc/reactive-impact.md) — Reactor allocation, SSE p99, Netty direct memory, VT + Generational ZGC _(2026-04-18)_

---

## Tier 11 — GraphQL & Federated APIs

Build schema-driven APIs that scale across teams, services, and databases. Read after you're comfortable with REST + WebFlux and know why you need federation.

78. [GraphQL Federation Concepts — Subgraphs, Gateway, and the Apollo Spec](graphql/federation-concepts.md) — `@key`, `@external`, `@requires`, `@provides`, v1 vs v2, composition, router, registry _(2026-04-18)_
79. [Netflix DGS vs Spring for GraphQL — Building Java Subgraphs](graphql/dgs-and-spring-graphql.md) — library comparison, resolvers, entity fetchers, DataLoader, federation setup, testing _(2026-04-18)_
80. [Federated GraphQL with Polyglot Persistence — DB-per-Subgraph, Saga, Outbox, Query Planning](graphql/multi-database-patterns.md) — cross-subgraph writes, caching, N+1, tracing, deployment _(2026-04-18)_

---

## Tier 12 — Production Practices (Observability, K8s, Gateway, Perf, Cloud)

After the core stack, these are what separate hobby projects from production systems.

81. [Distributed Tracing and Metrics Beyond Logs](observability/distributed-tracing.md) — OpenTelemetry, Micrometer, Prometheus, RED/USE _(2026-04-19)_
82. [Kubernetes for Spring Boot](configurations/kubernetes-spring-boot.md) — probes, ConfigMap/Secret, HPA, graceful shutdown, PDB _(2026-04-19)_
83. [API Gateway Patterns with Spring Cloud Gateway](web-layer/api-gateway-patterns.md) — routing, rate limiting, BFF, vs Envoy/Kong _(2026-04-19)_
84. [Distributed Systems Primer](architecture/distributed-systems-primer.md) — CAP, PACELC, consistency, Raft, idempotency, exactly-once myths _(2026-04-19)_
85. [Performance Testing with Gatling and k6](testing/performance-testing.md) — load scenarios, SLO gates in CI, flame graphs under load _(2026-04-19)_
86. [GraalVM Native Image for Spring Boot](configurations/graalvm-native-image.md) — AOT, reachability, Lambda cold starts, CRaC _(2026-04-19)_
87. [gRPC in Java](grpc/grpc-java.md) — protobuf, four RPC modes, interceptors, Reactor-gRPC bridge _(2026-04-19)_
88. [OIDC and Modern Auth Flows](security/oidc-and-modern-auth.md) — PKCE, refresh rotation, device flow, WebAuthn, MFA _(2026-04-19)_
89. [Secrets Management](security/secrets-management.md) — Vault, Spring Cloud Vault, k8s secrets, envelope encryption, KMS _(2026-04-19)_
90. [ArchUnit and Spring Modulith](architecture/archunit-and-modulith.md) — architecture as tests, modular monolith boundaries _(2026-04-19)_
91. [Feature Flags](configurations/feature-flags.md) — LaunchDarkly/Unleash/Togglz, progressive rollout, experiment-as-code _(2026-04-19)_
92. [Caching Deep Dive](data-repositories/caching-deep-dive.md) — invalidation, stampede, Redis patterns, Caffeine tuning, CDN _(2026-04-19)_
93. [Spring Boot on AWS and GCP](cloud/spring-boot-aws-gcp.md) — Spring Cloud AWS, Lambda SnapStart, Cloud Run, IRSA _(2026-04-19)_
94. [Event Sourcing and CQRS](architecture/event-sourcing-cqrs.md) — event store, aggregates, projections, snapshots, Axon _(2026-04-19)_
95. [DDD Tactical Patterns in Java](architecture/ddd-tactical-patterns.md) — aggregates, value objects, bounded contexts, domain events _(2026-04-19)_

---

## Tier 13 — Alternative JVM Frameworks

Beyond Spring Boot: other approaches to microservices on the JVM. These docs assume you know Spring and compare against it.

96. [Helidon Overview — Oracle's Lightweight Microservices Framework](helidon/helidon-overview.md) — SE vs MP, Nima, framework comparison, when to choose Helidon _(2026-04-19)_
97. [Helidon SE — Functional, No-Magic Microservices on Virtual Threads](helidon/helidon-se.md) — WebServer API, Config, routing DSL, no CDI, Express.js parallels _(2026-04-19)_
98. [Helidon MP — MicroProfile and Jakarta EE on a Lightweight Runtime](helidon/helidon-mp.md) — JAX-RS, CDI, MicroProfile specs, portability, Spring equivalences _(2026-04-19)_
99. [Helidon Nima — Virtual-Thread-Native Web Server Architecture](helidon/nima-virtual-threads-architecture.md) — from-scratch VT server, blocking sockets, Tomcat/Netty comparison, reactive exit _(2026-04-19)_
100. [Scheduling Beyond Spring — db-scheduler, Helidon Scheduling, and Quarkus Quartz](scheduling-beyond-spring.md) — db-scheduler with Helidon/Quarkus, Quarkus Quartz clustering, decision guide _(2026-04-19)_
101. [Quarkus Overview — Supersonic Subatomic Java for Cloud-Native](quarkus/quarkus-overview.md) — build-time DI, ArC/Jandex, dev mode, extensions, framework comparison _(2026-04-19)_
102. [Quarkus Reactive with Mutiny — Uni, Multi, and RESTEasy Reactive](quarkus/quarkus-reactive-mutiny.md) — Mutiny vs Reactor, Vert.x architecture, reactive data access, messaging _(2026-04-19)_
103. [Quarkus Native Image — GraalVM, Mandrel, and Serverless Deployment](quarkus/quarkus-native-image.md) — native build, limitations, testing, Lambda/Cloud Run/K8s deployment _(2026-04-19)_
104. [Quarkus Extensions — Panache, REST Client, Qute, and the Quarkiverse](quarkus/quarkus-extensions.md) — Panache ORM, REST Client, SmallRye, Spring compat layer _(2026-04-19)_
105. [Quarkus Virtual Threads — @RunOnVirtualThread and the Three Concurrency Models](quarkus/quarkus-virtual-threads.md) — event loop vs worker pool vs VT, pinning, Mutiny complement _(2026-04-19)_

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
- [Creational Patterns Deep Dive](design-patterns/creational-patterns.md)
- [Structural Patterns Deep Dive](design-patterns/structural-patterns.md)
- [Behavioral Patterns Deep Dive](design-patterns/behavioral-patterns.md)
- [Enterprise Patterns Deep Dive](design-patterns/enterprise-patterns.md)
- [Concurrency Basics](java-fundamentals/concurrency/concurrency-basics.md)
- [Multithreading Deep Dive (JMM, locks, synchronizers, pools)](java-fundamentals/concurrency/multithreading-deep-dive.md)
- [CompletableFuture Deep Dive](java-fundamentals/concurrency/completablefuture-deep-dive.md)
- [Interruption and Cancellation](java-fundamentals/concurrency/interruption-and-cancellation.md)
- [Concurrent Collections (ConcurrentHashMap, LongAdder, CAS)](java-fundamentals/concurrency/concurrent-collections.md)
- [ThreadLocal and Context Propagation](java-fundamentals/concurrency/threadlocal-and-context.md)
- [ForkJoinPool and Parallel Streams](java-fundamentals/concurrency/forkjoinpool-and-parallel-streams.md)
- [Concurrency Debugging Playbook](java-fundamentals/concurrency/concurrency-debugging.md)
- [Producer-Consumer Patterns](java-fundamentals/concurrency/producer-consumer-patterns.md)
- [Virtual Threads (Project Loom)](java-fundamentals/concurrency/virtual-threads.md)
- [Structured Concurrency](java-fundamentals/concurrency/structured-concurrency.md)
- [Structured Concurrency Before Project Loom](java-fundamentals/concurrency/structured-concurrency-before-loom.md)
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
- [Scaling MVC Before Virtual Threads](web-layer/mvc-high-throughput.md)

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

### JVM Performance & GC

- [GC Concepts and Mental Model](jvm-gc/concepts.md)
- [JVM Collectors (Serial/Parallel/CMS/G1/ZGC/Shenandoah)](jvm-gc/collectors.md)
- [GC Pause Diagnosis Playbook](jvm-gc/pause-diagnosis.md)
- [GC Impact on Reactive, VTs, and Streaming](jvm-gc/reactive-impact.md)

### GraphQL & Federation

- [GraphQL Federation Concepts](graphql/federation-concepts.md)
- [Netflix DGS vs Spring for GraphQL](graphql/dgs-and-spring-graphql.md)
- [Multi-Database Federation Patterns](graphql/multi-database-patterns.md)

### Observability

- [Distributed Tracing and Metrics Beyond Logs](observability/distributed-tracing.md)

### gRPC

- [gRPC in Java](grpc/grpc-java.md)

### Cloud

- [Spring Boot on AWS and GCP](cloud/spring-boot-aws-gcp.md)

### Production Practices

- [Kubernetes for Spring Boot](configurations/kubernetes-spring-boot.md)
- [API Gateway Patterns](web-layer/api-gateway-patterns.md)
- [Performance Testing](testing/performance-testing.md)
- [GraalVM Native Image](configurations/graalvm-native-image.md)
- [Feature Flags](configurations/feature-flags.md)
- [Secrets Management](security/secrets-management.md)
- [OIDC and Modern Auth Flows](security/oidc-and-modern-auth.md)
- [Caching Deep Dive](data-repositories/caching-deep-dive.md)

### Advanced Architecture

- [Distributed Systems Primer](architecture/distributed-systems-primer.md)
- [ArchUnit and Spring Modulith](architecture/archunit-and-modulith.md)
- [Event Sourcing and CQRS](architecture/event-sourcing-cqrs.md)
- [DDD Tactical Patterns in Java](architecture/ddd-tactical-patterns.md)

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

### Helidon

- [Helidon Overview](helidon/helidon-overview.md)
- [Helidon SE](helidon/helidon-se.md)
- [Helidon MP](helidon/helidon-mp.md)
- [Helidon Nima Architecture](helidon/nima-virtual-threads-architecture.md)

### Quarkus

- [Quarkus Overview](quarkus/quarkus-overview.md)
- [Quarkus Reactive with Mutiny](quarkus/quarkus-reactive-mutiny.md)
- [Quarkus Native Image](quarkus/quarkus-native-image.md)
- [Quarkus Extensions](quarkus/quarkus-extensions.md)
- [Quarkus Virtual Threads](quarkus/quarkus-virtual-threads.md)

### Cross-Framework

- [Scheduling Beyond Spring](scheduling-beyond-spring.md)
