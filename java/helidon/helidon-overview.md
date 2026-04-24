---
title: "Helidon Overview — Oracle's Lightweight Microservices Framework and Where It Fits"
date: 2026-04-19
updated: 2026-04-19
tags: [helidon, microservices, java, microprofile, virtual-threads]
---

# Helidon Overview — Oracle's Lightweight Microservices Framework and Where It Fits

**Date:** 2026-04-19 | **Updated:** 2026-04-19
**Tags:** `helidon` `microservices` `java` `microprofile` `virtual-threads`

## Table of Contents

- [Summary](#summary)
- [What Is Helidon](#what-is-helidon)
- [SE vs MP — Two Programming Models](#se-vs-mp--two-programming-models)
- [Helidon 4 and Nima — The Virtual Threads Rewrite](#helidon-4-and-nima--the-virtual-threads-rewrite)
- [Framework Comparison](#framework-comparison)
- [When to Choose Helidon](#when-to-choose-helidon)
- [When NOT to Choose Helidon](#when-not-to-choose-helidon)
- [Getting Started](#getting-started)
- [Related](#related)
- [References](#references)

---

## Summary

[Helidon](https://helidon.io/) is Oracle's open-source, cloud-native Java framework for building microservices. It offers two programming models — SE (functional, no annotations) and MP ([Eclipse MicroProfile](https://microprofile.io/) with JAX-RS and CDI) — both running as plain Java SE applications with no application server. Since version 4.0 (October 2023), Helidon is built on a virtual-thread-native web server called Nima, replacing Netty entirely and requiring Java 21+. This doc positions Helidon in the Java framework landscape alongside Spring Boot, Quarkus, and Micronaut.

---

## What Is Helidon

Helidon (Greek for "swallow" — the bird) was open-sourced by Oracle in 2018. It is designed and maintained by the Oracle Java team, many of whom also contribute to the JDK itself. This gives Helidon unusually early access to new JVM features — it was among the earliest Java microservice frameworks built around virtual threads.

Key characteristics:

- **No application server** — runs as a `java -jar` process, like Spring Boot
- **Cloud-native first** — built-in health checks, metrics, tracing, and fault tolerance; Kubernetes-ready out of the box
- **Two models, one runtime** — SE (functional) and MP (declarative) share the same underlying web server and config system
- **Java 21+ required** — Helidon 4 mandates virtual threads; there is no fallback to platform threads
- **Certified** — Helidon 4.0 is certified for [MicroProfile 6.0](https://microprofile.io/2023/01/10/microprofile-6-0-release/) and [Jakarta EE 10 Core Profile](https://jakarta.ee/specifications/coreprofile/10/); Helidon 4.1 for MicroProfile 6.1

As of April 19, 2026, the Helidon 4.4.x line is current. The official site prominently announces Helidon 4.4.0, while the API docs also publish 4.4.1 artifacts, so this guide avoids pinning to a single patch release here.

---

## SE vs MP — Two Programming Models

| Aspect | Helidon SE | Helidon MP |
|--------|-----------|-----------|
| **Style** | Functional, builder-based, no annotations | Declarative, annotation-driven |
| **DI** | None — explicit wiring | CDI (Jakarta Contexts and Dependency Injection) |
| **REST** | `HttpRouting` builder + `HttpService` interface | JAX-RS (`@Path`, `@GET`, `@POST`) via Jersey |
| **Config** | `Config.create()` + builders | MicroProfile Config (`@ConfigProperty`) |
| **Observability** | Programmatic registration | MicroProfile Health, Metrics, OpenTelemetry |
| **Fault tolerance** | Manual (or bring your own) | MicroProfile Fault Tolerance (`@Retry`, `@CircuitBreaker`) |
| **Footprint** | Smaller — fewer transitive deps | Slightly larger — CDI + Jersey + MP specs |
| **TS analogy** | Express.js — explicit routing, no magic | NestJS — decorators, DI container, conventions |
| **Best for** | Maximum control, small services, performance-critical paths | Teams from Jakarta EE / Spring Boot, spec portability |

MP is built on top of SE — the SE web server, config, and security modules are the foundation that MP layers CDI and MicroProfile APIs onto.

**Choose SE** when you want to see every line of wiring, minimize dependencies, and own the threading model. **Choose MP** when you want annotation-driven productivity, MicroProfile spec compliance, and portable code across runtimes (Open Liberty, Payara Micro, Quarkus).

---

## Helidon 4 and Nima — The Virtual Threads Rewrite

Helidon 4 is a ground-up rewrite. The web server layer, codenamed **Nima** (Greek for "thread"), replaces Netty with a custom server built entirely on Java 21 virtual threads:

- **One virtual thread per request** — blocking I/O is cheap; no reactive operator chains needed
- **No Netty** — the entire Netty dependency is gone, replaced by direct `java.net` socket I/O
- **Blocking sockets outperform NIO** — the Helidon team found that blocking sockets on virtual threads matched or exceeded Netty's throughput
- **HTTP/1.1, HTTP/2, gRPC, WebSocket** — all supported on the same server

In Helidon 3, SE was reactive (Reactive Streams, `Single<T>`, `Multi<T>`). In Helidon 4, those reactive types are gone — SE is now imperative/blocking, and the JVM handles concurrency via virtual thread scheduling. See the [Nima deep dive](nima-virtual-threads-architecture.md) for architecture details.

---

## Framework Comparison

### Startup and Memory (JVM Mode)

| Framework | Startup | Memory (RSS) | Notes |
|-----------|---------|-------------|-------|
| **Helidon SE** | ~0.6s | ~70 MB | Virtual threads, no CDI overhead |
| **Helidon MP** | ~1.2s | ~100 MB | CDI + Jersey add startup cost |
| **Spring Boot 3.x** | ~1.5–3s | ~150–250 MB | Auto-configuration, rich ecosystem |
| **Quarkus** | ~0.8–1.2s | ~80–130 MB | Build-time DI, optimized startup |
| **Micronaut** | ~0.5–0.7s | ~70–100 MB | Compile-time DI, fast startup |

### GraalVM Native Image

| Framework | Startup | Memory | Disk |
|-----------|---------|--------|------|
| **Helidon SE** | ~0.06s | ~34 MB | ~38 MB |
| **Quarkus** | ~0.02–0.05s | ~30–50 MB | ~40–60 MB |
| **Micronaut** | ~0.02–0.05s | ~30–50 MB | ~40–60 MB |
| **Spring Boot Native** | ~0.1s | ~50–80 MB | ~70–100 MB |

*Numbers are approximate and vary by application complexity. Sources: [helidon.io](https://helidon.io/), [Java Code Geeks 2026](https://www.javacodegeeks.com/2026/03/helidon-4-vs-quarkus-3-vs-micronaut-4-which-framework-actually-winswith-virtual-threads.html), [TheLinuxCode 2026](https://thelinuxcode.com/5-best-java-frameworks-for-microservices-2026-spring-boot-quarkus-micronaut-helidon-and-dropwizard/).*

### Ecosystem and Community

| Aspect | Spring Boot | Quarkus | Micronaut | Helidon |
|--------|------------|---------|-----------|---------|
| **GitHub stars** | ~75k | ~14k | ~6k | ~3.5k |
| **Stack Overflow** | Massive | Growing | Moderate | Small |
| **Third-party libs** | Broadest | Good (Quarkiverse) | Moderate | Limited |
| **Job market** | Dominant | Growing | Niche | Niche |
| **Backing** | VMware/Broadcom | Red Hat | Object Computing | Oracle |

### Programming Model

| Concern | Spring Boot | Quarkus | Micronaut | Helidon SE | Helidon MP |
|---------|------------|---------|-----------|-----------|-----------|
| **DI** | Runtime reflection | Build-time (ArC/CDI) | Compile-time | None | Runtime CDI |
| **Config** | `@Value`, `@ConfigurationProperties` | MP Config + SmallRye | `@Value`, `@ConfigurationProperties` | `Config.create()` | MP Config |
| **REST** | `@RestController` | JAX-RS | `@Controller` | `HttpRouting` builder | JAX-RS |
| **Reactive** | WebFlux (Reactor) | Mutiny | Reactor/RxJava | Dropped (was in v3) | Dropped (was in v3) |
| **Virtual threads** | Opt-in (Tomcat/Jetty) | Opt-in | Opt-in | Native (Nima) | Native (Nima) |

---

## When to Choose Helidon

- **Oracle Cloud Infrastructure (OCI)** — first-class integration, Oracle support via Java Verified Portfolio
- **Virtual-thread-native design** — if you want blocking I/O without reactive complexity and prefer a server built for VTs rather than retrofitted
- **MicroProfile compliance** — portable code across MP runtimes (Helidon MP)
- **Minimal footprint** — Helidon SE is one of the leanest JVM microservice stacks
- **GraalVM native images** — Helidon SE's small reflection surface makes native compilation straightforward
- **Small, focused services** — routing DSL + config + observability with no framework magic (SE)

---

## When NOT to Choose Helidon

- **Ecosystem breadth** — Spring Boot's library support dwarfs Helidon's; if you need Spring Data, Spring Security, Spring Batch, Spring Integration, etc., Helidon has no equivalent
- **Team familiarity** — most Java developers know Spring; Helidon has a steep learning curve not because it's complex, but because community resources (blogs, Stack Overflow answers, tutorials) are sparse
- **Complex business logic** — applications heavy on ORM (Hibernate), transaction management, or integration middleware are better served by Spring Boot or Quarkus
- **Hiring** — the Helidon job market is tiny compared to Spring Boot

---

## Getting Started

Scaffold a project with Maven:

```bash
# Replace the version below with the latest Helidon 4.4.x release from the official docs.

# Helidon SE
mvn -U archetype:generate \
  -DinteractiveMode=false \
  -DarchetypeGroupId=io.helidon.archetypes \
  -DarchetypeArtifactId=helidon-quickstart-se \
  -DarchetypeVersion=4.4.0 \
  -DgroupId=com.example \
  -DartifactId=helidon-se-demo \
  -Dpackage=com.example.se

# Helidon MP
mvn -U archetype:generate \
  -DinteractiveMode=false \
  -DarchetypeGroupId=io.helidon.archetypes \
  -DarchetypeArtifactId=helidon-quickstart-mp \
  -DarchetypeVersion=4.4.0 \
  -DgroupId=com.example \
  -DartifactId=helidon-mp-demo \
  -Dpackage=com.example.mp
```

Build and run:

```bash
cd helidon-se-demo
mvn package
java -jar target/helidon-se-demo.jar
```

No special Maven plugins, no wrapper scripts — just `java -jar`.

---

## Related

- [Helidon SE — Functional Microservices](helidon-se.md) — the SE programming model in depth
- [Helidon MP — MicroProfile Runtime](helidon-mp.md) — CDI, JAX-RS, and MicroProfile specs
- [Helidon Nima — Virtual-Thread-Native Architecture](nima-virtual-threads-architecture.md) — how the web server works under the hood
- [Spring Fundamentals](../spring-fundamentals.md) — IoC, DI, AOP — the Spring model Helidon contrasts with
- [Virtual Threads in Java](../java-fundamentals/concurrency/virtual-threads.md) — the JVM feature Helidon 4 is built on
- [GraalVM Native Image](../configurations/graalvm-native-image.md) — native compilation, which Helidon supports well

## References

- [Helidon Project — Official Site](https://helidon.io/) — docs, guides, API reference
- [Helidon GitHub Repository](https://github.com/helidon-io/helidon) — source code, issues, wiki
- [Helidon FAQ — SE vs MP](https://github.com/helidon-io/helidon/wiki/FAQ) — official guidance on choosing a model
- [MicroProfile 6.0 and Jakarta EE 10 Core Profile Certification](https://github.com/helidon-io/helidon/wiki/MicroProfile-6-and-Jakarta-EE-10-Core-Profile-Certification,-Helidon-4) — certification results
- [Helidon 4 Adopts Virtual Threads — InfoQ](https://www.infoq.com/articles/helidon-4-adopts-virtual-threads/) — deep dive on the Nima rewrite
- [Microservices with Oracle Helidon — Baeldung](https://www.baeldung.com/microservices-oracle-helidon) — tutorial walkthrough
- [Helidon 4.4.0 Release — Medium](https://medium.com/helidon/helidon-4-4-0-released-d10be2fb8039) — LangChain4j, MCP, OTel metrics/logs
- [5 Best Java Frameworks for Microservices 2026 — TheLinuxCode](https://thelinuxcode.com/5-best-java-frameworks-for-microservices-2026-spring-boot-quarkus-micronaut-helidon-and-dropwizard/) — framework comparison with benchmarks
