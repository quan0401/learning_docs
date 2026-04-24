---
title: Spring Boot Startup Lifecycle
date: 2026-04-17
updated: 2026-04-17
tags: [spring-boot, lifecycle, startup, runners]
---

# Spring Boot Startup Lifecycle

**Date:** 2026-04-17
**Tags:** `spring-boot` · `lifecycle` · `startup` · `runners`

## Table of Contents

- [Summary](#summary)
- [The Startup Sequence](#the-startup-sequence)
- [`@PostConstruct`](#postconstruct)
- [`InitializingBean.afterPropertiesSet()`](#initializingbeanafterpropertiesset)
- [`@Bean(initMethod = "...")`](#beaninitmethod--)
- [`SmartInitializingSingleton`](#smartinitializingsingleton)
- [`ApplicationRunner` vs `CommandLineRunner`](#applicationrunner-vs-commandlinerunner)
- [`@EventListener(ApplicationReadyEvent.class)`](#eventlistenerapplicationreadyeventclass)
- [`ApplicationContextEvent` Hierarchy](#applicationcontextevent-hierarchy)
- [Decision Table: Which Hook?](#decision-table-which-hook)
- [`SmartLifecycle`](#smartlifecycle)
- [Shutdown: `@PreDestroy`, `DisposableBean`](#shutdown-predestroy-disposablebean)
- [Graceful Shutdown](#graceful-shutdown)
- [Startup Failures](#startup-failures)
- [Startup Performance](#startup-performance)
- [Common Bugs](#common-bugs)
- [Related](#related)
- [References](#references)

---

## Summary

Spring Boot fires a well-defined sequence of events on startup. Hooks to run
code during that sequence include `@PostConstruct`, `InitializingBean`,
`SmartInitializingSingleton`, `ApplicationRunner`, `CommandLineRunner`,
`@EventListener(ApplicationReadyEvent.class)`, and `SmartLifecycle`. Each has a
specific semantic around *when* it fires and *how ordering* is controlled.

Picking the wrong hook is the root cause of classic bugs: slow startup because
work was stuffed into `@PostConstruct`, listeners firing twice because they
listened on `ContextRefreshedEvent` in a hierarchical context, or a
`CommandLineRunner` that throws and leaves the app in a half-ready state.

This document walks the full sequence, shows each hook's code shape, and ends
with a decision table plus the bugs you're likely to hit in production.

---

## The Startup Sequence

```mermaid
flowchart TD
  A[SpringApplication.run] --> B[ApplicationStartingEvent]
  B --> C[EnvironmentPreparedEvent<br/>properties loaded]
  C --> D[ApplicationContextInitializedEvent<br/>context created, not refreshed]
  D --> E[BeanFactoryPostProcessors]
  E --> F[Bean definitions registered]
  F --> G[BeanPostProcessors run on each bean]
  G --> G1[constructor]
  G1 --> G2[@Autowired field / setter injection]
  G2 --> G3[*Aware setters<br/>BeanNameAware, ApplicationContextAware...]
  G3 --> G4[BeanPostProcessor.postProcessBeforeInitialization]
  G4 --> G5[@PostConstruct]
  G5 --> G6[InitializingBean.afterPropertiesSet]
  G6 --> G7["@Bean(initMethod)"]
  G7 --> G8[BeanPostProcessor.postProcessAfterInitialization<br/>proxies created here]
  G8 --> H[SmartInitializingSingleton.afterSingletonsInstantiated]
  H --> I[ContextRefreshedEvent<br/>all singletons created]
  I --> J[SmartLifecycle.start in phase order]
  J --> K[ApplicationStartedEvent]
  K --> L[ApplicationRunner / CommandLineRunner<br/>ordered by @Order]
  L --> M[ApplicationReadyEvent<br/>APP IS READY]
  L -.-> X[ApplicationFailedEvent<br/>on startup error]
```

Key invariants:

- **Singletons are fully built before `ContextRefreshedEvent`.** By the time
  you see that event, every eagerly created bean has run through
  `@PostConstruct`.
- **Runners execute *after* `ApplicationStartedEvent` but *before*
  `ApplicationReadyEvent`.** If a runner throws, the app may fail to report
  "ready".
- **`ApplicationReadyEvent` is the last step.** It's the correct signal for
  "the app is serving traffic".

---

## `@PostConstruct`

The standard answer to "run once, after dependency injection, before anyone
uses this bean".

```java
@Component
@RequiredArgsConstructor
public class CacheWarmer {

    private final ProductRepository repo;
    private final Cache<Long, Product> cache;

    @PostConstruct
    public void warmUp() {
        repo.findTop100().forEach(p -> cache.put(p.id(), p));
    }
}
```

Semantics:

- Runs **after** constructor, `@Autowired` fields, and `*Aware` setters.
- Runs **before** `BeanPostProcessor.postProcessAfterInitialization`, which
  means proxies (transactional, AOP) are *not yet applied* to `this`. Do not
  call `@Transactional` methods on `this` from `@PostConstruct` expecting a
  new transaction — the proxy is not yet in place.
- **Blocks startup.** If `warmUp()` takes 30 seconds, your app takes 30
  extra seconds to become ready. In Kubernetes that often means the readiness
  probe fails and the pod gets killed.

Use it for: small, synchronous, bean-local initialization (populating a local
cache, validating config, opening a resource the bean owns).

Do *not* use it for: long-running work, network calls that may fail, anything
that depends on *other beans* being fully initialized (they might not be, if
this bean was constructed first).

---

## `InitializingBean.afterPropertiesSet()`

Older, framework-coupled alternative.

```java
public class LegacyBean implements InitializingBean {
    @Override
    public void afterPropertiesSet() {
        // same timing as @PostConstruct, ish
    }
}
```

Prefer `@PostConstruct`. It's standard (`jakarta.annotation`), has no Spring
coupling, and reads more clearly. The only reason to use `InitializingBean` is
when you can't add an annotation (e.g., the class is generated).

---

## `@Bean(initMethod = "...")`

For beans you don't own — a third-party class with a non-standard init method
that isn't annotated.

```java
@Configuration
public class ThirdPartyConfig {

    @Bean(initMethod = "connect", destroyMethod = "disconnect")
    public ExternalClient externalClient() {
        return new ExternalClient(endpoint);
    }
}
```

Spring calls `connect()` at the same point `@PostConstruct` would fire on a
bean you owned. Useful when porting legacy code into Spring's lifecycle.

---

## `SmartInitializingSingleton`

Callback that fires **after all non-lazy singletons are instantiated**, not
just "this bean".

```java
@Component
public class CrossBeanWiring implements SmartInitializingSingleton {

    private final ApplicationContext ctx;

    public CrossBeanWiring(ApplicationContext ctx) {
        this.ctx = ctx;
    }

    @Override
    public void afterSingletonsInstantiated() {
        Map<String, Plugin> plugins = ctx.getBeansOfType(Plugin.class);
        plugins.values().forEach(Plugin::register);
    }
}
```

Use when you need to iterate over *all* beans of some type and do setup — at
`@PostConstruct` time some of them might not exist yet.

---

## `ApplicationRunner` vs `CommandLineRunner`

Both run **after** the context is refreshed, **before**
`ApplicationReadyEvent`. Use them for "do this once on boot, with access to
command-line args".

```java
@Component
@Order(1)  // runs first among runners
@RequiredArgsConstructor
public class SeedDataRunner implements ApplicationRunner {

    private final SeedService seedService;

    @Override
    public void run(ApplicationArguments args) {
        if (args.containsOption("seed")) {
            seedService.loadFixtures();
        }
    }
}
```

| Interface           | Signature                                | Notes                                   |
|---------------------|------------------------------------------|-----------------------------------------|
| `CommandLineRunner` | `run(String... args)`                    | Raw argv                                |
| `ApplicationRunner` | `run(ApplicationArguments args)`         | Parsed (option vs non-option split)     |

Prefer `ApplicationRunner`. `ApplicationArguments` gives you
`containsOption("seed")`, `getOptionValues("profile")`, etc. — no manual
parsing.

Ordering:

- `@Order(1)` runs before `@Order(2)`.
- Without `@Order`, ordering is unspecified. Don't depend on discovery order.
- All runners run on the main thread, sequentially. A slow runner delays
  `ApplicationReadyEvent`.

Error behavior:

- By default, an exception thrown from a runner propagates and fails startup.
- The app exits with a non-zero code. Good — you probably want that.

---

## `@EventListener(ApplicationReadyEvent.class)`

The **last** hook in the startup sequence. Runs after every runner.

```java
@Component
@RequiredArgsConstructor
public class ReadinessAnnouncer {

    private final ServiceRegistry registry;

    @EventListener(ApplicationReadyEvent.class)
    public void announce() {
        registry.markHealthy(instanceId());
    }
}
```

Use for:

- Announcing "server up" to a service registry, Slack, PagerDuty, etc.
- Starting background schedulers that shouldn't race with startup.
- Any "everything is ready, now begin steady-state work" signal.

You can make it non-blocking with `@Async` if the listener does I/O you don't
want to block on.

---

## `ApplicationContextEvent` Hierarchy

Four context events you can listen to:

| Event                    | When                                                                 |
|--------------------------|----------------------------------------------------------------------|
| `ContextRefreshedEvent`  | After all singletons created. **Fires multiple times in hierarchical contexts** (parent context + each child). |
| `ContextStartedEvent`    | Only if `context.start()` is called explicitly. Most apps never see it. |
| `ContextStoppedEvent`    | Only if `context.stop()` is called explicitly.                       |
| `ContextClosedEvent`     | On shutdown, before bean destroy callbacks.                          |

The double-firing gotcha: in Spring Boot apps with a parent `ApplicationContext`
(common with Spring Cloud bootstrap context, or actuator's child context),
`ContextRefreshedEvent` fires *twice*. If you bind "run once on startup" logic
to it, that logic runs twice.

→ Prefer `ApplicationReadyEvent`. It fires exactly once, at the end.

Cross-ref: [`events-async/application-events.md`](events-async/application-events.md)

---

## Decision Table: Which Hook?

| You need to...                                                  | Use                                                |
|-----------------------------------------------------------------|----------------------------------------------------|
| Initialize a bean's own state before it's used                  | `@PostConstruct`                                   |
| Wrap a third-party class you can't annotate                     | `@Bean(initMethod = "...")`                        |
| Run something after *all* singletons exist                      | `SmartInitializingSingleton`                       |
| Process command-line arguments                                  | `ApplicationRunner`                                |
| Control ordering between runners                                | `@Order` on runners                                |
| Run as the *very last* startup step, after runners              | `@EventListener(ApplicationReadyEvent.class)`      |
| Run async so startup isn't blocked                              | `@EventListener(ApplicationReadyEvent) + @Async`   |
| Start/stop a scheduler or queue listener with phase control     | `SmartLifecycle`                                   |
| Cleanup on shutdown                                             | `@PreDestroy`                                      |
| Something depends on other beans being *fully* initialized      | `SmartInitializingSingleton` or `ApplicationReadyEvent` |

---

## `SmartLifecycle`

For components that need explicit start/stop beyond the context lifecycle.
Think: a Kafka consumer, a scheduled job pool, a background file watcher.

```java
@Component
public class QueueListener implements SmartLifecycle {

    private volatile boolean running = false;

    @Override
    public void start() {
        // open connection, start threads
        running = true;
    }

    @Override
    public void stop() {
        // drain, close, join
        running = false;
    }

    @Override
    public boolean isRunning() {
        return running;
    }

    @Override
    public int getPhase() {
        // lower phase starts first, stops LAST
        return 100;
    }

    @Override
    public boolean isAutoStartup() {
        return true;
    }
}
```

Phase rules:

- **Lower phase → starts earlier, stops later.** Mirror of "first in, last out".
- Default phase is 0.
- Use phases when you have a strict dependency like "DB connection pool must
  be up before the queue consumer, and must shut down after it".

`SmartLifecycle.stop(Runnable callback)` — the three-arg overload — lets you
report completion asynchronously during shutdown. Use it if your component
drains slowly.

---

## Shutdown: `@PreDestroy`, `DisposableBean`

`@PreDestroy` is the symmetric counterpart to `@PostConstruct`. It runs on
`ContextClosedEvent`, before the bean is removed.

```java
@Component
public class ConnectionHolder {

    private Connection conn;

    @PostConstruct
    public void open() { conn = dataSource.getConnection(); }

    @PreDestroy
    public void close() {
        if (conn != null) conn.close();
    }
}
```

Use for: flushing caches, closing connections, committing pending state, de-
registering from service registry.

`DisposableBean.destroy()` is the older, framework-coupled equivalent. Prefer
`@PreDestroy`.

Critical caveat: `@PreDestroy` **only runs if the JVM gets a chance to shut
down cleanly**. A `kill -9` (SIGKILL) skips it entirely. In Kubernetes, if
your pod takes longer than `terminationGracePeriodSeconds` to shut down, it
gets SIGKILL — and your cleanup doesn't run.

---

## Graceful Shutdown

```properties
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=30s
```

With `server.shutdown=graceful`:

1. On SIGTERM, the HTTP server stops accepting new requests.
2. In-flight requests are allowed to finish, up to
   `timeout-per-shutdown-phase`.
3. Then `SmartLifecycle.stop()` phases run, then `@PreDestroy`.

Match `timeout-per-shutdown-phase` to your Kubernetes
`terminationGracePeriodSeconds` (leave 5-10s headroom).

Cross-ref: [`configurations/docker-and-deployment.md`](configurations/docker-and-deployment.md)

---

## Startup Failures

`ApplicationFailedEvent` fires on any startup exception — thrown from a
`@PostConstruct`, a bean factory method, a `BeanPostProcessor`, or a runner.

```java
@Component
public class FailureNotifier {

    @EventListener(ApplicationFailedEvent.class)
    public void onFail(ApplicationFailedEvent event) {
        log.error("Startup failed", event.getException());
        // phone home, write marker file, etc.
    }
}
```

`SpringApplication.setRegisterShutdownHook(false)` disables the default
shutdown hook. You then have to call `context.close()` yourself. Rarely
needed — leave the default on.

By default, a startup failure logs the full stack trace to STDOUT and exits
non-zero.

---

## Startup Performance

Three tools:

1. **`--debug` flag** — prints auto-configuration report and timing summary.
2. **`ApplicationStartup` API** — Spring Framework 5.3+ records per-step
   timings. Expose via actuator:
   ```properties
   management.endpoints.web.exposure.include=startup
   ```
   Then `GET /actuator/startup` returns the step tree with durations.
3. **`BufferingApplicationStartup`** — wire it in `main()` so the actuator
   endpoint has data:
   ```java
   SpringApplication app = new SpringApplication(App.class);
   app.setApplicationStartup(new BufferingApplicationStartup(2048));
   app.run(args);
   ```

Cross-ref: [`actuator-deep-dive.md`](actuator-deep-dive.md)

---

## Common Bugs

**1. `@PostConstruct` doing long-running work.**
App takes 40 seconds to start. Kubernetes readiness probe fails at 30
seconds. Pod gets killed and restarted in a loop.
→ Move the work to `@EventListener(ApplicationReadyEvent)` with `@Async`, or
to a `SmartLifecycle` component that starts in the background.

**2. `@EventListener(ContextRefreshedEvent)` running twice.**
In hierarchical contexts (Spring Cloud bootstrap, actuator management
context), `ContextRefreshedEvent` fires once per context. Seed data gets
inserted twice. Idempotent code gets lucky; non-idempotent code corrupts.
→ Use `ApplicationReadyEvent`. It fires exactly once.

**3. `ApplicationRunner` that swallows exceptions.**
A `try/catch` around the body turns a fatal config error into a silent
successful startup. App reports "ready" but doesn't actually work.
→ Let exceptions propagate. Startup failure is the correct outcome for
misconfiguration.

**4. Relying on bean initialization order.**
Spring is free to instantiate unrelated beans in any order. `@PostConstruct`
in bean A may run before bean B's `@PostConstruct`, or after. Do not assume.
→ Use `@DependsOn("beanB")` if A genuinely requires B to be ready, or move
the cross-bean wiring to `SmartInitializingSingleton`.

**5. `@Transactional` call from `@PostConstruct`.**
The proxy isn't in place yet when `@PostConstruct` runs. Calling a
`@Transactional` method via `this.foo()` — or any self-call — bypasses the
transactional proxy entirely. Silently no transaction.
→ Move transactional init to a runner, or inject the bean as a dependency
(so you go through the proxy) or use `ApplicationReadyEvent`.

**6. `@PreDestroy` not running in containers.**
App killed with SIGKILL because `terminationGracePeriodSeconds` is too low,
or because `server.shutdown=graceful` wasn't set and requests hung.
→ Set `server.shutdown=graceful`, match
`spring.lifecycle.timeout-per-shutdown-phase` to the pod's grace period.

**7. Slow `SmartLifecycle.stop()` blocking shutdown.**
Phases run sequentially. A slow stop in one phase blocks the next.
→ Use the `stop(Runnable callback)` overload, or tighten per-phase timeout.

**8. Circular startup dependency through events.**
Bean A posts an event from `@PostConstruct` that Bean B listens to, and B
also has `@PostConstruct` work. Whichever was constructed first "wins";
the other silently misses the event.
→ Publish startup events from `ApplicationReadyEvent` listeners, not from
`@PostConstruct`.

---

## Related

- [`spring-fundamentals.md`](spring-fundamentals.md)
- [`events-async/application-events.md`](events-async/application-events.md)
- [`configurations/docker-and-deployment.md`](configurations/docker-and-deployment.md)
- [`actuator-deep-dive.md`](actuator-deep-dive.md)
- [`java-fundamentals/concurrency/concurrency-basics.md`](java-fundamentals/concurrency/concurrency-basics.md)

## References

- Spring Boot reference docs — *Application Events and Listeners*
- `org.springframework.boot.SpringApplication` — javadoc
- `org.springframework.boot.context.event.ApplicationReadyEvent` — javadoc
- `org.springframework.context.SmartLifecycle` — javadoc
- `org.springframework.context.event.ContextRefreshedEvent` — javadoc
- Spring Boot reference docs — *Graceful Shutdown*
- Spring Framework reference docs — *ApplicationStartup / Startup Tracking*
- JSR-250 — `jakarta.annotation.PostConstruct`, `jakarta.annotation.PreDestroy`
