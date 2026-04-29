---
title: "Inspecting the JPA First-Level Cache from a Spring Shared EntityManager Proxy"
date: 2026-04-28
updated: 2026-04-28
tags: [jpa, hibernate, debugging, spring, persistence-context]
---

# Inspecting the JPA First-Level Cache from a Spring Shared EntityManager Proxy

**Date:** 2026-04-28 | **Updated:** 2026-04-28
**Tags:** `jpa` `hibernate` `debugging` `spring` `persistence-context`

## Summary

When you inject `EntityManager` via `@PersistenceContext` in a Spring service, the field is a **shared, thread-safe proxy** — not the EntityManager that holds the cache. Drilling into the proxy fields in a debugger shows transaction/factory metadata, never the persistence context. To see the cache, you must call a method on the proxy (typically `unwrap`) so it resolves the real transaction-bound EntityManager and the underlying Hibernate `Session`. From there, `getPersistenceContextInternal().getEntityEntriesByKey()` is the actual map of cached entities.

## Table of Contents

- [Summary](#summary)
- [Why Drilling Into the Proxy Doesn't Reveal the Cache](#why-drilling-into-the-proxy-doesnt-reveal-the-cache)
- [Where the Cache Actually Lives](#where-the-cache-actually-lives)
- [Watch / Evaluate-Expression Cheatsheet](#watch--evaluate-expression-cheatsheet)
- [Reference Identity vs `contains()`](#reference-identity-vs-contains)
- [What Looks Like a Cache But Isn't](#what-looks-like-a-cache-but-isnt)
- [Source Pointers](#source-pointers)
- [Related](#related)
- [References](#references)

## Why Drilling Into the Proxy Doesn't Reveal the Cache

Spring injects EntityManager through `SharedEntityManagerCreator`. The Spring docs describe its return value as: "A shared EntityManager will behave just like an EntityManager fetched from an application server's JNDI environment, as defined by the JPA specification. **It will delegate all calls to the current transactional EntityManager, if any; otherwise it will fall back to a newly created EntityManager per operation.**" ([Spring `SharedEntityManagerCreator` Javadoc](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/orm/jpa/SharedEntityManagerCreator.html))

Concretely, the field you see in a debugger looks like:

```text
entityManager ($Proxy163 — "Shared EntityManager proxy for target factory ...")
 └─ h (SharedEntityManagerCreator$SharedEntityManagerInvocationHandler)
     ├─ targetFactory       — EntityManagerFactory bean used to resolve the real EM
     ├─ properties = null
     ├─ synchronizedWithTransaction = true
     └─ proxyClassLoader
```

None of these expose the persistence context. The handler's job is to look up — on every method call — the real EntityManager bound to the current thread's transaction, then forward the call. That bound EntityManager is what holds the cache, and it doesn't exist as a field on the proxy.

This matters because clicking through fields in your IDE will never reach `entitiesByKey`. You have to **invoke a method** so the proxy resolves the real EntityManager and you can unwrap to the Hibernate `Session`.

## Where the Cache Actually Lives

The Hibernate `Session` (which extends `EntityManager`) implements `SharedSessionContractImplementor`, which is "the internal contract shared between `Session` and `StatelessSession` as used by other parts of Hibernate." ([Hibernate `SharedSessionContractImplementor` Javadoc](https://docs.hibernate.org/orm/current/javadocs/org/hibernate/engine/spi/SharedSessionContractImplementor.html))

This SPI exposes:

- `getPersistenceContext()` — "Get the persistence context for this session." ([Hibernate Javadoc](https://docs.hibernate.org/orm/current/javadocs/org/hibernate/engine/spi/SharedSessionContractImplementor.html))
- `getPersistenceContextInternal()` — "Similar to `getPersistenceContext()`, with two differences: this version performs better as it allows for inlining and probably better prediction, and it skips some checks of the current state of the session." ([Hibernate Javadoc](https://docs.hibernate.org/orm/current/javadocs/org/hibernate/engine/spi/SharedSessionContractImplementor.html))

The returned `PersistenceContext` represents the state of "stuff" Hibernate is tracking — entities, collections, snapshots, and proxies. It is "often referred to as the first-level cache." ([Baeldung — JPA/Hibernate Persistence Context](https://www.baeldung.com/jpa-hibernate-persistence-context))

Inside, the relevant map is `entitiesByKey` (accessor: `getEntityEntriesByKey()`), keyed by `EntityKey` (entity class + identifier).

## Watch / Evaluate-Expression Cheatsheet

Set a breakpoint inside any service method that has executed at least one query. Then use these in the debugger's WATCH or Evaluate Expression panel.

### One-line "is this cached?"

```java
entityManager.contains(matchedEmailSequence)
```

`true` means the entity is in the persistence context. Any subsequent JPQL `SELECT` for the same primary key will return this exact reference, regardless of `WHERE` clauses applied at the SQL level.

### Get the real Hibernate Session

```java
entityManager.unwrap(org.hibernate.Session.class)
```

Calling `unwrap` forces the shared proxy to resolve the real transactional EntityManager and return it as a `Session`. From here you can drill down properly — `persistenceContext`, `statistics`, `transaction`, etc.

### Get the cache map directly

```java
((org.hibernate.engine.spi.SharedSessionContractImplementor)
    entityManager.unwrap(org.hibernate.Session.class))
  .getPersistenceContextInternal()
  .getEntityEntriesByKey()
```

Returns `Map<EntityKey, EntityEntry>`. Expand it in the debugger to see one entry per managed entity:

```text
EntityKey[com.qode.emailsequence.entities.EmailSequence#<uuid>]
  → EntityEntry { entity=<the cached object>, status=MANAGED, loadedState=[...], ... }
```

`loadedState` is the snapshot Hibernate took at load time; it's how dirty checking works.

### Count of cached entities

```java
entityManager.unwrap(org.hibernate.Session.class).getStatistics().getEntityCount()
```

Useful as a quick numeric watch — see whether a query added new entities or reused cached ones. Requires `spring.jpa.properties.hibernate.generate_statistics=true`.

## Reference Identity vs `contains()`

If you want to avoid injecting `EntityManager` purely for debugging, a no-injection alternative exists. Refactor the query call to a named local:

```java
var fromQuery = repository.findByIdWithEnabledStep(matched.getId()).get();
```

Then watch:

```java
fromQuery == matched
System.identityHashCode(fromQuery) == System.identityHashCode(matched)
```

`true` proves the cache returned the same Java reference. This is functionally equivalent to `entityManager.contains(matched) == true` for the purpose of confirming a cache hit, and works on any local with no extra plumbing.

## What Looks Like a Cache But Isn't

Two fields you may see while drilling around in IDE that **do not** contain JPA entities:

- `org.springframework.transaction.interceptor.TransactionInterceptor.transactionManagerCache` — caches `PlatformTransactionManager` lookups for `@Transactional` beans. AOP infrastructure, not JPA.
- `org.springframework.aop.framework.JdkDynamicAopProxy$ProxiedInterfacesCache` — caches the list of interfaces the AOP proxy implements. Also AOP infrastructure.

If you see either expanding from a service's `emailSequenceRepository` or the `EntityManager` proxy, you're in the wrong layer. Pivot to `unwrap(Session.class)`.

## Source Pointers

For stepping through Hibernate internals:

- `org.hibernate.engine.internal.StatefulPersistenceContext` — concrete implementation of `PersistenceContext`. Field `entitiesByKey` is the primary identity map. Method `getEntity(EntityKey)` is a useful function breakpoint to watch cache hits.
- `org.hibernate.event.internal.DefaultLoadEventListener.loadFromSessionCache` — explicit cache-check branch on the load path.
- `org.springframework.orm.jpa.SharedEntityManagerCreator` — Spring's source for the shared proxy ([GitHub](https://github.com/spring-projects/spring-framework/blob/main/spring-orm/src/main/java/org/springframework/orm/jpa/SharedEntityManagerCreator.java)). The inner `SharedEntityManagerInvocationHandler` is what intercepts every call.

## Related

- [JPA Stale Collection Trap: Why a Re-Query Doesn't Refresh an Already-Managed Entity](jpa-stale-collection-trap.md) — sibling doc on the bug pattern this debugging technique is most often used to confirm.

## References

- [Spring `SharedEntityManagerCreator` Javadoc](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/orm/jpa/SharedEntityManagerCreator.html)
- [Spring Framework Reference — JPA](https://docs.spring.io/spring-framework/reference/data-access/orm/jpa.html)
- [Hibernate `SharedSessionContractImplementor` Javadoc](https://docs.hibernate.org/orm/current/javadocs/org/hibernate/engine/spi/SharedSessionContractImplementor.html)
- [Baeldung — JPA/Hibernate Persistence Context](https://www.baeldung.com/jpa-hibernate-persistence-context)
- [Spring `SharedEntityManagerCreator` source](https://github.com/spring-projects/spring-framework/blob/main/spring-orm/src/main/java/org/springframework/orm/jpa/SharedEntityManagerCreator.java)
