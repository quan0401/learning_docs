---
title: "JPA Stale Collection Trap: Why a Re-Query Doesn't Refresh an Already-Managed Entity"
date: 2026-04-28
updated: 2026-04-28
tags: [jpa, hibernate, persistence-context, soft-delete, gotcha]
---

# JPA Stale Collection Trap: Why a Re-Query Doesn't Refresh an Already-Managed Entity

**Date:** 2026-04-28 | **Updated:** 2026-04-28
**Tags:** `jpa` `hibernate` `persistence-context` `soft-delete` `gotcha`

## Summary

When you mutate a JPA entity's collection in memory and then call a JPQL query (even one with `JOIN FETCH` and a filter) within the same transaction, the query returns the in-memory object — not the rows the SQL produced. This is JPA's first-level cache identity guarantee: one persistence context, one Java reference per primary key. We hit this in `UpdateEmailSequenceService.executeWithSteps` where soft-deleted steps leaked into the API response despite the query filtering on `enabled = true`.

## Table of Contents

- [Summary](#summary)
- [Symptom](#symptom)
- [What Actually Happens](#what-actually-happens)
- [Why Hibernate Does This](#why-hibernate-does-this)
- [When the Cache Is Populated](#when-the-cache-is-populated)
- [Two Loads with Different Filters in the Same Transaction](#two-loads-with-different-filters-in-the-same-transaction)
- [JPQL vs Native Query — Why This Matters](#jpql-vs-native-query--why-this-matters)
- [How to Verify in the Debugger](#how-to-verify-in-the-debugger)
- [What Doesn't Fix It](#what-doesnt-fix-it)
- [Operations Matrix — What Each Operation Actually Does](#operations-matrix--what-each-operation-actually-does)
- [What Does Fix It](#what-does-fix-it)
- [Recommended Pattern for This Codebase](#recommended-pattern-for-this-codebase)
- [References](#references)
- [Changelog](#changelog)

## Symptom

`PATCH /api/email-sequences/{id}` with a `steps` payload containing only step 1 returned a body like:

```json
{
  "emailSequenceSteps": [
    { "stepNumber": 1, "enabled": true,  "version": 3 },
    { "stepNumber": 2, "enabled": false, "version": 3 },
    { "stepNumber": 3, "enabled": false, "version": 3 }
  ]
}
```

The disabled rows shouldn't be in the response. The repository query is explicitly:

```sql
SELECT DISTINCT es FROM EmailSequence es
LEFT JOIN FETCH es.emailSequenceSteps step
WHERE es.id = :id AND step.enabled = true
```

The SQL is correct. The response is wrong.

## What Actually Happens

The flow inside one `@Transactional` method:

1. `getByIdWithThrowException` loads `EmailSequence A`. The persistence context binds `A` to its primary key. `A.steps = [s1, s2, s3]`, all enabled.
2. The service mutates the collection: `A.steps.clear()`, then `addAll([s1_new_enabled, s2_disabled, s3_disabled])`. With `orphanRemoval = true` this is the soft-delete trick — the rows stay in the collection so they get `UPDATE enabled=false` instead of `DELETE`.
3. `repository.save(A)` flags `A` dirty.
4. `findByIdWithEnabledStep(id)` runs the SQL. The DB returns one row (the new step 1). Hibernate sees a row mapping to a primary key already in the persistence context and **returns the existing reference `A`** with its in-memory collection unchanged. The fetched rows are discarded for entity assembly purposes.

The response is `A`, whose collection still holds the disabled rows.

## Why Hibernate Does This

Two rules in the JPA / Hibernate spec collide:

**Identity guarantee.** "The same managed entity reference will be returned to the caller" no matter how many times the entity is loaded from the persistence context. ([Vlad Mihalcea — JPA/Hibernate First-Level Cache](https://vladmihalcea.com/jpa-hibernate-first-level-cache/))

**One entity per identifier per session.** "In a JPA EntityManager or Hibernate Session, there can be only one and only one entity stored using the same identifier and entity class type." ([Vlad Mihalcea — JPA/Hibernate First-Level Cache](https://vladmihalcea.com/jpa-hibernate-first-level-cache/))

The first-level cache "cannot prevent a JPQL or SQL from loading the latest entity snapshot from the database, only to discard it upon assembling the query result set." ([Vlad Mihalcea — JPA/Hibernate First-Level Cache](https://vladmihalcea.com/jpa-hibernate-first-level-cache/)) The query runs, the rows come back, and then Hibernate throws them away in favor of the already-managed reference. The collection on that reference reflects whatever the application code most recently did to it in memory.

## When the Cache Is Populated

A common follow-up question: "where exactly does the entity get into the cache?" Cache population happens on **read paths that return entity types**, not on writes.

In `UpdateEmailSequenceService.executeWithSteps`, the line that populates the cache is the **first** query — line 37 in the current file:

```java
var matchedEmailSequence = this.emailSequenceService.getByIdWithThrowException(UUID.fromString(id));
```

Trace into the call chain:

1. `EmailSequenceService.getByIdWithThrowException(UUID)` — throwing wrapper:

   ```java
   @NonNull
   public EmailSequence getByIdWithThrowException(UUID id) {
     var matchedEmailSequence = this.emailSequenceRepository.findByIdWithEnabledStep(id);

     if (matchedEmailSequence.isEmpty()) {
       throw new BusinessException("Email sequence is not found", HttpStatus.NOT_FOUND);
     }

     return matchedEmailSequence.get();
   }
   ```

2. → `EmailSequenceRepository.findByIdWithEnabledStep(UUID)` — JPQL with `JOIN FETCH` and `WHERE step.enabled = true`:

   ```java
   @Query("""
       SELECT DISTINCT es FROM EmailSequence es
       LEFT JOIN FETCH es.emailSequenceSteps step
       WHERE es.id = :id AND step.enabled = true
       ORDER BY step.stepNumber ASC
       """)
   Optional<EmailSequence> findByIdWithEnabledStep(@Param("id") UUID id);
   ```

   This is what runs the SQL, materializes managed entities, and registers them in the persistence context.

After this line returns, the persistence context contains:

```text
EntityKey[EmailSequence#<uuid>]       → matchedEmailSequence
EntityKey[EmailSequenceStep#<uuid>]   → s1
EntityKey[EmailSequenceStep#<uuid>]   → s2
EntityKey[EmailSequenceStep#<uuid>]   → s3
```

Those bindings stay until the transaction ends, the entity is detached, or the persistence context is cleared.

### Which operations populate the cache vs. which don't

Cache population (entity gets registered or made managed in the persistence context) happens on:

- `EntityManager.find(class, id)` / `Session.get(class, id)`
- JPQL/HQL queries that select entity types (`SELECT e FROM Entity e ...`)
- Native queries with `@SqlResultSetMapping` mapping to entity types
- Loading associations (lazy initialization or eager fetch)
- `EntityManager.persist(transient)` — registers a brand-new transient entity as managed (a different lifecycle than "load and cache")

What does **not** populate the cache from the DB:

- `repository.save(e)` on a managed entity — only flags dirty; cache binding was already set when `e` was originally loaded
- `repository.saveAndFlush(e)` — same as save plus a flush; still no DB read
- `flush()` — write path only
- `entityManager.contains(e)` — returns a boolean, no side effects

### Verify in the debugger

Set a breakpoint on the load line. Add this watch:

```java
((org.hibernate.engine.spi.SharedSessionContractImplementor)
    entityManager.unwrap(org.hibernate.Session.class))
  .getPersistenceContextInternal()
  .getEntityEntriesByKey()
  .size()
```

- Before the line executes: typically `0` for a fresh transaction
- After the line executes: the count jumps by `1 + N` (one parent plus one per fetched step row)

That jump is the moment of caching. Subsequent queries by primary key for the same entity will reuse those bindings, regardless of `WHERE` filters at the SQL level.

## Two Loads with Different Filters in the Same Transaction

A common variant of this trap appears when a repository method is parametrized so different callers can get different views — for example, the same query can be called with `enabled = true` for read flows and `enabled = null` (no filter) for step-diff flows:

```java
@Query("""
    SELECT DISTINCT es FROM EmailSequence es
    LEFT JOIN FETCH es.emailSequenceSteps step
    WHERE es.id = :id
      AND (:enabled IS NULL OR step.enabled = :enabled)
    """)
Optional<EmailSequence> findByIdWithEnabledStep(@Param("id") UUID id, @Param("enabled") Boolean enabled);
```

The JPQL is correct. The `(:enabled IS NULL OR step.enabled = :enabled)` predicate evaluates to `TRUE` when the parameter is null, so the SQL produces all step rows.

But the cache trap fires anyway when an outer method has *already* loaded the same entity in this transaction with a different filter:

```text
@Transactional execute(payload)
 ├─ load entity with filter=true  → PC binds EmailSequence#X with only enabled steps
 ├─ apply name/metadata mutations
 └─ delegate to executeWithSteps(id, steps)
     └─ load entity with filter=null → SQL returns ALL rows
                                      ↳ Hibernate sees X already in PC
                                      ↳ returns the cached reference (enabled-only collection)
                                      ↳ SQL rows are discarded
```

Symptom: the second method reads `matchedEmailSequence.getEmailSequenceSteps()` and sees only the enabled ones, even though the SQL it just ran was `WHERE :enabled IS NULL OR step.enabled = :enabled` with `:enabled` bound to `null`.

The structural fix is to **load once per transaction** with the right filter chosen up front. See "Recommended Pattern for This Codebase" below.

## JPQL vs Native Query — Why This Matters

`findByIdWithEnabledStep` is a **JPQL** query, not a native one — there's no `nativeQuery = true` on the `@Query`, and it references the entity name `EmailSequence` and the field `es.emailSequenceSteps` rather than table/column names. This distinction matters when reading external articles:

- **Articles about "native queries causing stale persistence context"** (e.g., the Medium piece below) describe a *different* problem: a native `UPDATE` / `DELETE` writes to the DB without going through JPA, leaving managed entities in memory inconsistent with the DB. The fix there is `flush()` before the native write and `clear()` (or `refresh()`) after.
- **Our case** has no native modification. It's a pure JPQL `SELECT` that returns a managed entity already bound to the persistence context. The cache wins on identity grounds, not because of an out-of-band write. ([Medium — Handling outdated data in JPA persistence context after native queries](https://medium.com/@ndhamani2002/handling-outdated-data-in-jpa-persistence-context-after-native-queries-ea1b10ce1d88))

In short: native modification staleness and managed-entity identity-cache staleness are two different traps with overlapping fixes. Our problem is the second.

## How to Verify in the Debugger

When diagnosing a suspected cache hit, three checks confirm the cause without guessing.

### 1. `entityManager.contains(entity)` — is it managed?

Inject `EntityManager` (temporarily) into the service:

```java
@jakarta.persistence.PersistenceContext
private jakarta.persistence.EntityManager entityManager;
```

Set a breakpoint just before the suspect query and add to the IDE's WATCH panel:

```text
entityManager.contains(matchedEmailSequence)
```

If it returns `true`, the entity is bound to the current persistence context and any subsequent JPQL `SELECT` matching its primary key will return this exact reference. The query's `WHERE` clause runs in SQL but does not change which Java object you receive.

### 2. Reference identity — same object returned by the query?

This proof needs no extra injection. Refactor the query call so the result is a named local:

```java
this.emailSequenceRepository.save(matchedEmailSequence);
var fromQuery = this.emailSequenceRepository.findByIdWithEnabledStep(matchedEmailSequence.getId()).get();
return fromQuery;
```

Set a breakpoint on `return fromQuery;` and watch:

```text
fromQuery == matchedEmailSequence
fromQuery.getEmailSequenceSteps().size()
fromQuery.getEmailSequenceSteps().stream().map(s -> s.getStepNumber() + "=" + s.getEnabled()).toList()
```

If the first watch is `true`, identity guarantee is in effect: the SQL ran but Hibernate handed you back the cached reference. The size and step-list watches reveal that the in-memory collection was not refreshed by the query — it still contains whatever you last assigned to it.

### 3. SQL log vs in-memory size — the smoking gun

Enable SQL logging:

```properties
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.orm.jdbc.bind=TRACE
spring.jpa.properties.hibernate.generate_statistics=true
```

Run the same flow. The log will show one execution of the JPQL with `rows: 1`, while the in-memory `fromQuery.getEmailSequenceSteps().size()` is `3`. The mismatch is the clearest possible signal that SQL produced one row but the cached entity's collection was returned untouched.

### Where the cache actually lives

Useful pointers when stepping through Hibernate internals:

- `org.hibernate.engine.spi.PersistenceContext` — the interface. Implementation is `StatefulPersistenceContext`. The map `entitiesByKey` is the cache.
- `StatefulPersistenceContext.getEntity(EntityKey)` — fires for every entity lookup; a good function breakpoint to watch cache hits.
- `org.hibernate.event.internal.DefaultLoadEventListener.loadFromSessionCache` — explicit "is this in the cache?" branch on the load path.

Spring AOP fields you might see while debugging — `TransactionInterceptor.transactionManagerCache`, `JdkDynamicAopProxy$ProxiedInterfacesCache` — are unrelated to the JPA first-level cache. They cache transaction managers and proxied interface lists respectively.

## What Doesn't Fix It

- **`repository.save()` then re-query.** Even after auto-flush pushes the in-memory state to the DB, the next query still returns the cached managed instance. Save doesn't evict.
- **`repository.saveAndFlush()` then re-query.** A common mis-intuition: "if I flush now, the DB has the latest, so the next query will read fresh data." The DB *is* up to date — but the persistence context still holds the same Java reference, and JPA's identity guarantee means the requery hands back that cached object regardless of how recently the rows changed. Spring Data describes `saveAndFlush` as "Saves an entity and flushes changes instantly" ([Spring Data `JpaRepository.saveAndFlush()` Javadoc](https://docs.spring.io/spring-data/jpa/docs/current/api/org/springframework/data/jpa/repository/JpaRepository.html)) — it bundles save + flush, but **flush ≠ evict**. Without `detach` or `clear`, the cached collection state still wins on the next query.
- **`entityManager.refresh(entity)`.** Refresh "synchronizes the current persistence context to the underlying database" and loads the entity by primary key. ([LogicBig — Refreshing Managed Entities](https://www.logicbig.com/tutorials/java-ee-tutorial/jpa/refreshing.html)) But it loads *all* steps — it doesn't apply the `WHERE step.enabled = true` filter from your custom JPQL. So you'd refresh enabled and disabled rows back in and lose the soft-delete view.
- **Adding a parametrized filter to the repository method.** Making the JPQL more flexible (e.g., `(:enabled IS NULL OR step.enabled = :enabled)`) does not help if a caller has *already* loaded the same entity in this transaction with a different filter. Identity guarantee still wins — the SQL runs, but the cached reference is what gets returned. See [Two Loads with Different Filters in the Same Transaction](#two-loads-with-different-filters-in-the-same-transaction).
- **Mutating the managed collection to filter out disabled rows before returning.** Removing items from a collection with `orphanRemoval = true` schedules them for `DELETE` at flush. That defeats the soft-delete.

## Operations Matrix — What Each Operation Actually Does

The recurring source of confusion is conflating "writing to the DB" with "evicting from the cache." They are independent.

| Operation | Marks dirty | Writes SQL | Evicts from PC |
|---|---|---|---|
| `save(e)` | ✓ | only at next flush | ✗ |
| `flush()` | — | ✓ (all queued writes) | ✗ |
| `saveAndFlush(e)` | ✓ | ✓ | ✗ |
| `detach(e)` | — | — | ✓ (this entity) |
| `clear()` | — | — | ✓ (everything in PC) |
| `refresh(e)` | — | — | overwrites in-memory state with DB row, but does not run your custom JPQL filters |

The "Evicts from PC" column is the one that matters for stale-cache bugs. Only `detach` and `clear` set it. No amount of `save` / `flush` / `saveAndFlush` produces eviction; they're orthogonal operations bundled by convenience methods. If your symptom is "I flushed and it still returned old data", the missing piece is almost always eviction.

## What Does Fix It

Two patterns work, with different trade-offs:

**A. Build a transient response object** (no JPA involvement). Construct a new `EmailSequence` instance with the fields you want, including a filtered step list. The managed entity keeps the soft-deleted rows for the upcoming flush; the response object is just a DTO-shaped view.

```java
return EmailSequence.builder()
    .id(matched.getId())
    .name(matched.getName())
    // ... other fields
    .emailSequenceSteps(enabledStepsOnly)
    .build();
```

**B. `saveAndFlush` + detach + re-query.** Push pending changes to the DB, evict the entity from the persistence context, then run the query so it produces a fresh managed instance built from DB rows.

```java
@PersistenceContext
EntityManager entityManager;

repository.saveAndFlush(matched);
entityManager.detach(matched);
return repository.findByIdWithEnabledStep(id).orElseThrow();
```

`saveAndFlush` covers the save + flush in one call. `detach()` removes the entity from the persistence context so the next query has no cached binding to satisfy. ([Baeldung — Refresh and Fetch an Entity After Save in JPA](https://www.baeldung.com/spring-data-jpa-refresh-fetch-entity-after-save)) Both lines are required — neither alone is sufficient. See the [operations matrix](#operations-matrix--what-each-operation-actually-does) for why.

This costs one extra `SELECT` round-trip per call. Pattern A is preferred when that cost matters.

## Recommended Pattern for This Codebase

The structural fix is **a single load per transaction**. If two code paths in the same transaction need different views of the same entity, choose the strictest/widest filter at the entry point and pass the loaded entity down rather than re-querying. Eviction (`detach`) and transient-response builders are tactical patches that work *after* the trap has been set up — single-load-per-transaction prevents it.

Concrete shape:

```java
@Transactional
public EmailSequence execute(String id, UpdateEmailSequenceRequest payload) {
  // Pick the filter that covers the work about to happen.
  // null → all steps (needed for diff/version/soft-delete logic)
  // true → enabled-only (sufficient for read-only response)
  Boolean stepFilter = needsAllSteps(payload) ? null : true;
  var matched = repository.findByIdWithEnabledStep(UUID.fromString(id), stepFilter)
      .orElseThrow(() -> new BusinessException("Email sequence not found", HttpStatus.NOT_FOUND));

  // mutate matched in-memory...

  if (needsAllSteps(payload)) {
    return applyStepDiff(matched, payload.getSteps());   // takes the loaded entity, no re-query
  }
  repository.save(matched);
  return matched;
}
```

For read-after-write inside a method that just produced soft-deleted state and now needs to return the enabled-only view, fall back to Pattern **A** (transient response builder) — no extra query, no PC manipulation. Pattern **B** (`saveAndFlush + detach + requery`) is acceptable when downstream code in the same transaction must interact with the freshly-loaded state.

If a similar trap appears for a read flow with no preceding mutation or load, neither pattern is needed — the cache is empty and the query returns fresh data normally.

## References

- [JPA/Hibernate First-Level Cache — Vlad Mihalcea](https://vladmihalcea.com/jpa-hibernate-first-level-cache/)
- [JPA + Hibernate Refreshing an Entity Instance — LogicBig](https://www.logicbig.com/tutorials/java-ee-tutorial/jpa/refreshing.html)
- [Refresh and Fetch an Entity After Save in JPA — Baeldung](https://www.baeldung.com/spring-data-jpa-refresh-fetch-entity-after-save)
- [Spring Data `JpaRepository.saveAndFlush()` Javadoc](https://docs.spring.io/spring-data/jpa/docs/current/api/org/springframework/data/jpa/repository/JpaRepository.html)
- [Handling outdated data in JPA persistence context after native queries — Medium](https://medium.com/@ndhamani2002/handling-outdated-data-in-jpa-persistence-context-after-native-queries-ea1b10ce1d88)

## Changelog

- 2026-04-28: Added "JPQL vs Native Query" clarification and "How to Verify in the Debugger" section covering `entityManager.contains()`, reference-identity watches, SQL logging, and the relevant Hibernate internals.
- 2026-04-28: Clarified that `saveAndFlush()` is not eviction; added save/flush/detach operations matrix; updated Pattern B to use `saveAndFlush + detach` (matches current applied code).
- 2026-04-28: Added "When the Cache Is Populated" section pinpointing the exact line (`getByIdWithThrowException`) and listing read vs. write operations to clarify which ones cache.
- 2026-04-28: Documented the "two loads with different filters in the same transaction" variant of the cache trap, and recommended single-load-per-transaction as the structural fix.
