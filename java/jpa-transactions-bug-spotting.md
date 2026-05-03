---
title: "JPA & @Transactional — Bug Spotting"
date: 2026-05-03
updated: 2026-05-03
tags: [bug-spotting, jpa, hibernate, transactions, spring, java]
---

# JPA & @Transactional — Bug Spotting

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `bug-spotting` `jpa` `hibernate` `transactions` `spring` `java`

---

## Table of Contents
1. [How to use this doc](#how-to-use-this-doc)
2. [Easy (warm-up traps)](#1-easy-warm-up-traps)
3. [Subtle (review-passers)](#2-subtle-review-passers)
4. [Senior trap (production-only failures)](#3-senior-trap-production-only-failures)
5. [Solutions](#4-solutions)
6. [Related](#related)
7. [References](#references)

## Summary
This doc is an active-recall practice surface for the JPA/Hibernate + Spring `@Transactional` failure surface. It collects 22 broken snippets covering proxy semantics (self-invocation, private methods, `@Async`), entity lifecycle (lazy loading, `LazyInitializationException`, detached merge), fetching (N+1, EAGER Cartesian product, `JOIN FETCH`), propagation (`REQUIRES_NEW`, rollback markers, checked-exception silent commit), persistence-context coherence (first-level cache vs JPQL bulk update, native query flushing), and locking (optimistic version misses, pessimistic deadlock). Try the bug, peek at the one-line hint, then jump to §4 for the full root cause and reference. Treat this as a 30-minute warm-up before reviewing any PR that touches `@Entity` or `@Transactional`.

## How to use this doc
- Try to spot the bug before opening the `<details>` hint.
- The hint is one line; the full root cause and fix are in §4 Solutions, keyed by bug number.
- Skip Easy only if you've already nailed those traps in code review.

## 1. Easy (warm-up traps)

### Bug 1 — Lazy load outside the transaction

```java
@Service
public class OrderService {
    @Autowired private OrderRepository orderRepository;

    @Transactional(readOnly = true)
    public Order findOrder(Long id) {
        return orderRepository.findById(id).orElseThrow();
    }
}

@RestController
class OrderController {
    @Autowired private OrderService orderService;

    @GetMapping("/orders/{id}")
    public List<OrderItem> items(@PathVariable Long id) {
        Order order = orderService.findOrder(id);
        return order.getItems(); // boom at iteration time
    }
}
```
<details><summary>Hint</summary>
Where does the persistence context end?
</details>

### Bug 2 — N+1 in a forEach

```java
@Transactional(readOnly = true)
public List<String> postTitlesWithAuthor() {
    return postRepository.findAll().stream()
        .map(p -> p.getAuthor().getName() + ": " + p.getTitle())
        .toList();
}
// Post.author is @ManyToOne(fetch = LAZY)
```
<details><summary>Hint</summary>
Count the SELECTs Hibernate emits as the stream iterates.
</details>

### Bug 3 — Self-invocation skipping the proxy

```java
@Service
public class UserService {
    public void registerUsers(List<UserDto> dtos) {
        dtos.forEach(this::registerOne); // same-bean call
    }

    @Transactional
    public void registerOne(UserDto dto) {
        userRepository.save(new User(dto.email()));
        auditService.record(dto.email()); // throws → expected rollback
    }
}
```
<details><summary>Hint</summary>
Spring AOP only sees calls that go through the proxy.
</details>

### Bug 4 — `@Transactional` on a private method

```java
@Service
public class PaymentService {
    public void charge(Long orderId) {
        applyCharge(orderId);
    }

    @Transactional
    private void applyCharge(Long orderId) {
        // update balance, insert ledger row
    }
}
```
<details><summary>Hint</summary>
Spring's proxy-mode visibility rules forbid this.
</details>

### Bug 5 — Checked exception that doesn't roll back

```java
@Transactional
public void importInvoice(InvoiceDto dto) throws IOException {
    invoiceRepository.save(new Invoice(dto));
    fileStore.write(dto.pdfBytes()); // throws IOException
}
```
<details><summary>Hint</summary>
What does Spring's default rollback rule actually cover?
</details>

### Bug 6 — Read-only transaction with a write

```java
@Transactional(readOnly = true)
public User touchLogin(String email) {
    User u = userRepository.findByEmail(email).orElseThrow();
    u.setLastLoginAt(Instant.now());
    return u; // returns successfully... sometimes
}
```
<details><summary>Hint</summary>
`readOnly` is a hint, not always an enforcer — but Hibernate may quietly skip the dirty check.
</details>

### Bug 7 — Bidirectional association set on only one side

```java
Post post = new Post("Hello");
PostComment comment = new PostComment("first!");
comment.setPost(post);            // owning side set
postRepository.save(post);        // post.comments still empty in memory
post.getComments().forEach(System.out::println); // prints nothing
```
<details><summary>Hint</summary>
Hibernate doesn't synchronize the inverse side for you.
</details>

## 2. Subtle (review-passers)

### Bug 8 — `REQUIRES_NEW` that still rolls back the outer

```java
@Service
public class EnrollmentService {
    @Transactional
    public void enroll(StudentDto dto) {
        studentRepository.save(new Student(dto));
        try {
            auditLogger.logEnrollment(dto); // @Transactional(REQUIRES_NEW)
        } catch (Exception e) {
            // swallow — audit failure must not break enrollment
        }
        // why does the whole transaction sometimes still roll back?
    }
}
```
<details><summary>Hint</summary>
Spring tracks a rollback-only flag on the outer scope independently of exceptions you catch.
</details>

### Bug 9 — Equals/hashCode using a generated id

```java
@Entity
public class Post {
    @Id @GeneratedValue private Long id;

    @Override public boolean equals(Object o) {
        return o instanceof Post p && Objects.equals(id, p.id);
    }
    @Override public int hashCode() { return Objects.hash(id); }
}

Set<Post> drafts = new HashSet<>();
Post p = new Post("hi");
drafts.add(p);
postRepository.save(p);
boolean stillThere = drafts.contains(p); // false
```
<details><summary>Hint</summary>
The hash code changed between `add` and `contains`.
</details>

### Bug 10 — Detached entity passed to `persist`

```java
public Post update(Long id, String title) {
    Post existing = postRepository.findById(id).orElseThrow();
    em.clear(); // simulate boundary crossing (DTO round-trip in real code)
    existing.setTitle(title);
    em.persist(existing); // intent: "save my changes"
    return existing;
}
```
<details><summary>Hint</summary>
`persist` and `merge` are not interchangeable for detached entities.
</details>

### Bug 11 — `@OneToMany(cascade = ALL)` deleting more than expected

```java
@Entity
public class Author {
    @OneToMany(mappedBy = "author", cascade = CascadeType.ALL)
    private List<Book> books = new ArrayList<>();
}

authorRepository.delete(author); // also wipes shared co-authored books in some schemas
```
<details><summary>Hint</summary>
`CascadeType.ALL` includes `REMOVE`. What if a child is referenced from elsewhere?
</details>

### Bug 12 — JPQL bulk update with stale first-level cache

```java
@Transactional
public void promoteToPremium(Long userId) {
    User u = userRepository.findById(userId).orElseThrow(); // tier=BASIC loaded
    em.createQuery("update User u set u.tier = 'PREMIUM' where u.id = :id")
      .setParameter("id", userId)
      .executeUpdate();
    notify(u.getTier()); // still 'BASIC'
}
```
<details><summary>Hint</summary>
Bulk DML doesn't update the entities already in the persistence context.
</details>

### Bug 13 — Native query that misses pending changes

```java
@Transactional
public BigDecimal totalAfterDiscount(Long orderId, BigDecimal discount) {
    Order o = orderRepository.findById(orderId).orElseThrow();
    o.setDiscount(discount); // dirty, not yet flushed
    return (BigDecimal) em.createNativeQuery(
        "select sum(line_total) - :d from order_line where order_id = :id")
        .setParameter("d", discount)
        .setParameter("id", orderId)
        .getSingleResult(); // may compute against pre-discount state
}
```
<details><summary>Hint</summary>
Hibernate's auto-flush logic doesn't introspect native SQL.
</details>

### Bug 14 — Eager everywhere → Cartesian product

```java
@Entity
public class Order {
    @OneToMany(fetch = FetchType.EAGER) private List<OrderItem> items;
    @OneToMany(fetch = FetchType.EAGER) private List<Shipment> shipments;
    @OneToMany(fetch = FetchType.EAGER) private List<PaymentEvent> events;
}

orderRepository.findAll(); // returns rows × items × shipments × events
```
<details><summary>Hint</summary>
Multiple eager bag-style collections in one SELECT multiply.
</details>

### Bug 15 — `JOIN FETCH` plus `setMaxResults`

```java
List<Post> page = em.createQuery(
    "select p from Post p join fetch p.comments where p.published = true",
    Post.class)
  .setFirstResult(0)
  .setMaxResults(20)
  .getResultList(); // HHH000104 warning — pagination done in memory
```
<details><summary>Hint</summary>
Hibernate has to materialize the whole result before slicing — see HHH000104.
</details>

### Bug 16 — `@Async` losing the transaction

```java
@Service
public class ReportService {
    @Transactional
    public void generate(Long reportId) {
        reportRepository.markRunning(reportId);
        renderAsync(reportId); // returns immediately
    }

    @Async
    public void renderAsync(Long reportId) {
        Report r = reportRepository.findById(reportId).orElseThrow();
        r.getSections().forEach(this::render); // LazyInitializationException
    }
}
```
<details><summary>Hint</summary>
`@Async` switches threads — does the persistence context follow?
</details>

### Bug 17 — Open Session in View hiding the N+1

```yaml
# application.yml
spring:
  jpa:
    open-in-view: true # the Spring Boot default
```
```java
@RestController
class FeedController {
    @GetMapping("/feed")
    public List<PostDto> feed() {
        return postRepository.findAll().stream()
            .map(p -> new PostDto(p.getTitle(), p.getAuthor().getName())) // lazy hit per row
            .toList();
    }
}
```
<details><summary>Hint</summary>
The default Boot setting masks this in dev and ambushes you in prod.
</details>

## 3. Senior trap (production-only failures)

### Bug 18 — Optimistic lock missed on a child collection

```java
@Entity
public class Order {
    @Version private long version;
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderLine> lines = new ArrayList<>();
}

@Transactional
public void appendLine(Long orderId, OrderLine line) {
    Order o = orderRepository.findById(orderId).orElseThrow();
    line.setOrder(o);
    o.getLines().add(line);
    // two concurrent appends both succeed; @Version on Order never bumps
}
```
<details><summary>Hint</summary>
Mutating the children alone doesn't always trigger an owning-side dirty check.
</details>

### Bug 19 — Pessimistic write lock with no timeout

```java
@Transactional
public void debit(Long accountId, BigDecimal amount) {
    Account a = em.find(Account.class, accountId, LockModeType.PESSIMISTIC_WRITE);
    a.subtract(amount);
}
// Two threads call debit(1, ...) and debit(2, ...) in opposite order from another method.
```
<details><summary>Hint</summary>
Locks held until commit + no timeout + inconsistent ordering = classic.
</details>

### Bug 20 — Second-level cache stale after a sibling service writes

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class TaxRate {
    @Id private String region;
    private BigDecimal rate;
}

// Service A (this app)
TaxRate r = em.find(TaxRate.class, "DE"); // populated 2L cache

// Service B (another app pointing at the same DB) updates DE rate via SQL.
// Service A keeps reading the old value across requests.
```
<details><summary>Hint</summary>
Hibernate's L2 cache doesn't know about writers it didn't go through.
</details>

### Bug 21 — `@Transactional` on a `@Configuration` bean

```java
@Configuration
@Transactional
public class TenantBootstrap {
    @Bean
    public TenantSeeder seeder(TenantRepository repo) {
        return new TenantSeeder(repo); // proxied @Configuration + @Transactional → AOP weirdness
    }
}
```
<details><summary>Hint</summary>
`@Configuration` is already proxied by CGLIB for bean-method semantics. Stacking `@Transactional` on top is asking for trouble.
</details>

### Bug 22 — Flush-order surprise on insert-then-delete

```java
@Entity
@Table(uniqueConstraints = @UniqueConstraint(columnNames = "email"))
public class User { @Id @GeneratedValue Long id; @Column String email; }

@Transactional
public void replace(String email) {
    User old = userRepository.findByEmail(email).orElseThrow();
    userRepository.delete(old);
    userRepository.save(new User(email)); // ConstraintViolationException at flush
}
```
<details><summary>Hint</summary>
Hibernate orders DML by entity action type, not by call order.
</details>

## 4. Solutions

### Bug 1 — Lazy load outside the transaction
**Root cause:** The `@Transactional` boundary ends when `findOrder` returns. The persistence context (and its Hibernate `Session`) closes, so accessing `order.getItems()` in the controller triggers `LazyInitializationException`. Open Session in View can mask this in Spring Boot, but turning it off (or running with it off in prod) exposes the bug.
**Fix:**
```java
@Transactional(readOnly = true)
public OrderView findOrder(Long id) {
    Order o = orderRepository.findById(id).orElseThrow();
    return new OrderView(o.getId(), o.getItems().stream().map(OrderItemView::from).toList());
}
```
Project to a DTO inside the transaction, or use an entity graph / `JOIN FETCH`.
**Reference:** Hibernate User Guide §6 "Persistence Context" and §11 "Fetching" — [Hibernate User Guide](https://docs.jboss.org/hibernate/orm/6.6/userguide/html_single/Hibernate_User_Guide.html)

### Bug 2 — N+1 in a forEach
**Root cause:** `findAll()` issues one SELECT for posts. Then each `p.getAuthor().getName()` triggers a SELECT against `author` because the association is `LAZY`. With N posts, that's N+1 queries.
**Fix:**
```java
@Query("select p from Post p join fetch p.author")
List<Post> findAllWithAuthor();
```
Or use `@EntityGraph(attributePaths = "author")` on the repository method.
**Reference:** [Vlad Mihalcea — N+1 Query Problem](https://vladmihalcea.com/n-plus-1-query-problem/)

### Bug 3 — Self-invocation skipping the proxy
**Root cause:** Spring's default `@Transactional` is implemented with a JDK or CGLIB proxy. `this::registerOne` invokes the target instance directly, bypassing the proxy, so no transaction is started and no rollback happens. The `auditService` failure leaves the saved `User` committed.
**Fix:**
```java
@Service
public class UserService {
    @Autowired private UserService self; // self-injected proxy

    public void registerUsers(List<UserDto> dtos) { dtos.forEach(self::registerOne); }

    @Transactional
    public void registerOne(UserDto dto) { /* ... */ }
}
```
Or split the transactional method into a separate `@Service`. AspectJ mode also fixes it without proxying.
**Reference:** [Spring Framework — @Transactional annotations (self-invocation)](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html)

### Bug 4 — `@Transactional` on a private method
**Root cause:** In proxy mode, Spring can only intercept calls dispatched via the proxy. Private methods are invoked directly on the target instance and are never intercepted, so `@Transactional` is silently ignored. The Spring docs explicitly call this "not advised".
**Fix:**
```java
@Transactional
public void applyCharge(Long orderId) { /* ... */ }
```
Make the method at least package-private (and call it through the proxy), or move it to another bean, or switch to AspectJ weaving.
**Reference:** [Spring Framework — Method visibility and @Transactional](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html)

### Bug 5 — Checked exception that doesn't roll back
**Root cause:** Spring's default rollback rule rolls back only on `RuntimeException` and `Error`. A checked `IOException` propagates out but the transaction commits the saved `Invoice` first. You end up with an invoice row pointing at a non-existent file.
**Fix:**
```java
@Transactional(rollbackFor = IOException.class)
public void importInvoice(InvoiceDto dto) throws IOException { /* ... */ }
```
Or wrap in an unchecked exception. Don't rely on "all exceptions roll back" — they don't.
**Reference:** [Spring Framework — Declarative Transaction Management (rollback rules)](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative.html)

### Bug 6 — Read-only transaction with a write
**Root cause:** With `readOnly = true`, Hibernate sets the session's flush mode to `MANUAL` and skips automatic dirty checking, so `setLastLoginAt` is never flushed. Some JDBC drivers also forward `setReadOnly(true)` to the database, which would actively reject writes. Either way, the update is silently lost.
**Fix:**
```java
@Transactional // read-write
public User touchLogin(String email) {
    User u = userRepository.findByEmail(email).orElseThrow();
    u.setLastLoginAt(Instant.now());
    return u;
}
```
**Reference:** [Vlad Mihalcea — Spring @Transactional annotation (read-only)](https://vladmihalcea.com/spring-transactional-annotation/)

### Bug 7 — Bidirectional association set on only one side
**Root cause:** Hibernate persists based on the *owning* side (the side without `mappedBy`). It never reads the inverse side and never auto-syncs the in-memory `post.comments` list. The DB row is correct; the Java object graph is not.
**Fix:**
```java
public class Post {
    public void addComment(PostComment c) {
        comments.add(c);
        c.setPost(this);
    }
}
```
Always provide an `add/remove` helper that maintains both ends.
**Reference:** Hibernate User Guide §2.7 "Bidirectional associations" — [Hibernate User Guide](https://docs.jboss.org/hibernate/orm/6.6/userguide/html_single/Hibernate_User_Guide.html)

### Bug 8 — `REQUIRES_NEW` that still rolls back the outer
**Root cause:** Read again — the inner method must have `Propagation.REQUIRES_NEW`. If it's actually `REQUIRED` (or self-invoked, defaulting to no propagation change), the inner failure marks the *shared* transaction rollback-only. Even after you catch the exception, the outer `enroll` commit becomes `UnexpectedRollbackException`. Confirm the inner method is on a *different* bean and the propagation attribute is set.
**Fix:**
```java
@Service
public class AuditLogger {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logEnrollment(StudentDto dto) { /* ... */ }
}
```
**Reference:** [Spring Framework — Transaction Propagation](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html)

### Bug 9 — Equals/hashCode using a generated id
**Root cause:** Before `save`, `id` is null and `hashCode` returns one value. After `save`, the id is assigned and the hash code changes. The `HashSet` bucket is wrong, so `contains` misses. Any `Set`-based association mapping breaks the same way.
**Fix:**
```java
@Override public int hashCode() { return getClass().hashCode(); }
@Override public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Post other)) return false;
    return id != null && id.equals(other.id);
}
```
Constant `hashCode`; `equals` only matches non-null ids.
**Reference:** [Vlad Mihalcea — Best way to implement equals/hashCode/toString with JPA](https://vladmihalcea.com/the-best-way-to-implement-equals-hashcode-and-tostring-with-jpa-and-hibernate/)

### Bug 10 — Detached entity passed to `persist`
**Root cause:** `persist` requires a *transient* (no id, never persisted) entity. A detached entity (already has an id) makes `persist` throw `EntityExistsException` or `PersistenceException`, depending on provider. Use `merge` (returns a managed copy) — or just modify a managed entity inside the transaction and let dirty checking flush.
**Fix:**
```java
public Post update(Long id, String title) {
    Post managed = postRepository.findById(id).orElseThrow();
    managed.setTitle(title); // dirty check on commit
    return managed;
}
```
**Reference:** Hibernate User Guide §6 "Persistence Context" (entity states) — [Hibernate User Guide](https://docs.jboss.org/hibernate/orm/6.6/userguide/html_single/Hibernate_User_Guide.html)

### Bug 11 — `@OneToMany(cascade = ALL)` deleting more than expected
**Root cause:** `CascadeType.ALL` includes `REMOVE`. Deleting the parent recursively deletes children — fine when the children are exclusively owned, catastrophic when they're shared (many-to-many or referenced by other parents). The fix is to cascade only what business logic actually demands.
**Fix:**
```java
@OneToMany(mappedBy = "author", cascade = {CascadeType.PERSIST, CascadeType.MERGE})
private List<Book> books = new ArrayList<>();
```
Cascade `REMOVE` only for true parent-owned aggregates.
**Reference:** [Vlad Mihalcea — Beginner's Guide to JPA and Hibernate Cascade Types](https://vladmihalcea.com/a-beginners-guide-to-jpa-and-hibernate-cascade-types/)

### Bug 12 — JPQL bulk update with stale first-level cache
**Root cause:** `executeUpdate` runs SQL directly. It bypasses the persistence context, so the already-loaded `User` instance still has `tier=BASIC`. Subsequent reads from that managed entity return stale data within the same transaction.
**Fix:**
```java
em.createQuery("update User u set u.tier = 'PREMIUM' where u.id = :id")
  .setParameter("id", userId)
  .executeUpdate();
em.refresh(u); // or em.clear() and re-load
notify(u.getTier());
```
Prefer dirty checking (`u.setTier(PREMIUM)`) when you have a single entity in hand.
**Reference:** Hibernate User Guide §15 "HQL & JPQL — Bulk update/delete" — [Hibernate User Guide](https://docs.jboss.org/hibernate/orm/6.6/userguide/html_single/Hibernate_User_Guide.html)

### Bug 13 — Native query that misses pending changes
**Root cause:** Hibernate's auto-flush triggers when a JPQL query references entities with pending changes. For native SQL, Hibernate cannot parse table names back to entity types, so it does *not* auto-flush. The native query sees the pre-flush DB state.
**Fix:**
```java
em.flush();
return (BigDecimal) em.createNativeQuery("...").getSingleResult();
```
Or annotate the query method with `@QueryHints` / set `FlushModeType.AUTO` and explicitly tell Hibernate which entities the query reads (`@NamedNativeQuery` with `synchronize`).
**Reference:** Hibernate User Guide §17 "Native SQL queries" (synchronization) — [Hibernate User Guide](https://docs.jboss.org/hibernate/orm/6.6/userguide/html_single/Hibernate_User_Guide.html)

### Bug 14 — Eager everywhere → Cartesian product
**Root cause:** Each eager `@OneToMany` joins another table. With three sibling collections, the result set is `orders × items × shipments × events` rows, which Hibernate then deduplicates in memory. Latency explodes and memory blows up.
**Fix:**
```java
@OneToMany(fetch = FetchType.LAZY) private List<OrderItem> items;
@OneToMany(fetch = FetchType.LAZY) private List<Shipment> shipments;
@OneToMany(fetch = FetchType.LAZY) private List<PaymentEvent> events;
```
Default everything to `LAZY`; load what you actually need with entity graphs or per-call `JOIN FETCH` (and only one bag-collection per query).
**Reference:** Hibernate User Guide §11 "Fetching" — [Hibernate User Guide](https://docs.jboss.org/hibernate/orm/6.6/userguide/html_single/Hibernate_User_Guide.html)

### Bug 15 — `JOIN FETCH` plus `setMaxResults`
**Root cause:** `JOIN FETCH` produces a Cartesian product, so paginating in SQL would slice mid-parent. Hibernate logs `HHH000104: firstResult/maxResults specified with collection fetch; applying in memory` and pulls the entire result set into memory before truncating. Page size lies; memory bites.
**Fix:** Two-phase load — first a paginated query for parent ids, then a `JOIN FETCH` query bound to those ids.
```java
List<Long> ids = em.createQuery("select p.id from Post p where p.published = true", Long.class)
    .setMaxResults(20).getResultList();
List<Post> page = em.createQuery(
    "select distinct p from Post p join fetch p.comments where p.id in :ids", Post.class)
    .setParameter("ids", ids).getResultList();
```
**Reference:** Hibernate User Guide §11.4 "Pagination with collection fetching" — [Hibernate User Guide](https://docs.jboss.org/hibernate/orm/6.6/userguide/html_single/Hibernate_User_Guide.html)

### Bug 16 — `@Async` losing the transaction
**Root cause:** `@Async` is also implemented via a Spring proxy. Calling `renderAsync` from inside `generate` is a self-invocation — the call stays on the calling thread synchronously, *and* the `@Transactional` on `generate` doesn't propagate to the async work because Spring transactions are thread-bound. Even if you fix self-invocation, the async thread has no persistence context.
**Fix:** Move the async method to a different bean, and start a new transaction inside it:
```java
@Service
public class AsyncReportRenderer {
    @Async
    @Transactional
    public void renderAsync(Long reportId) { /* ... */ }
}
```
Pass ids, never managed entities, across thread boundaries.
**Reference:** [Spring Framework — @Transactional annotations (self-invocation, threads)](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html)

### Bug 17 — Open Session in View hiding the N+1
**Root cause:** OSIV keeps the persistence context open through view rendering, so lazy access in the controller silently triggers extra queries. Spring Boot defaults to `open-in-view: true`, which masks N+1 in dev and silently issues hundreds of queries per request in prod, often outside any transaction (auto-commit per statement).
**Fix:**
```yaml
spring:
  jpa:
    open-in-view: false
```
Then load required associations inside `@Transactional` methods or project to DTOs.
**Reference:** [Vlad Mihalcea — The Open Session in View Anti-Pattern](https://vladmihalcea.com/the-open-session-in-view-anti-pattern/)

### Bug 18 — Optimistic lock missed on a child collection
**Root cause:** `@Version` on `Order` is incremented only when Hibernate detects a dirty *owning-side* change. Adding to `o.getLines()` modifies the children, not `Order` itself. Two concurrent appends both pass the version check, both succeed, and a "lost update" never trips.
**Fix:** Force the parent dirty, or use `OPTIMISTIC_FORCE_INCREMENT`:
```java
em.lock(o, LockModeType.OPTIMISTIC_FORCE_INCREMENT);
o.getLines().add(line);
```
Or model the operation as a parent-side mutation that bumps a counter.
**Reference:** [Vlad Mihalcea — Optimistic Locking with @Version](https://vladmihalcea.com/optimistic-locking-version-property-jpa-hibernate/)

### Bug 19 — Pessimistic write lock with no timeout
**Root cause:** `PESSIMISTIC_WRITE` issues `SELECT ... FOR UPDATE`. Locks are held until commit. With two transactions acquiring `account 1` and `account 2` in opposite order (e.g., transfer A→B vs B→A), you get a deadlock; with no timeout, callers hang until the DB's deadlock detector fires (or worse, threads pile up).
**Fix:**
```java
em.find(Account.class, accountId, LockModeType.PESSIMISTIC_WRITE,
    Map.of("jakarta.persistence.lock.timeout", 3000));
// And: always lock accounts in canonical order (e.g., by id ascending).
```
**Reference:** Hibernate User Guide §10 "Locking" — [Hibernate User Guide](https://docs.jboss.org/hibernate/orm/6.6/userguide/html_single/Hibernate_User_Guide.html)

### Bug 20 — Second-level cache stale after a sibling service writes
**Root cause:** Hibernate's L2 cache is a write-through cache only for writes that go through Hibernate. External writers (other services, DBA scripts, replication) don't invalidate cache entries. Reads stay stale until TTL or app restart.
**Fix:** Don't enable L2 cache for entities mutated outside this app. If you must, use a clustered cache provider with database change-data-capture invalidation, or accept eventual consistency with an aggressive TTL. Treat the L2 cache as a single-writer optimization.
**Reference:** Hibernate User Guide §13 "Caching" (Second-level cache) — [Hibernate User Guide](https://docs.jboss.org/hibernate/orm/6.6/userguide/html_single/Hibernate_User_Guide.html)

### Bug 21 — `@Transactional` on a `@Configuration` bean
**Root cause:** `@Configuration` classes are already CGLIB-proxied so Spring can intercept `@Bean` method calls and return the singleton. Adding `@Transactional` at the class level layers a second proxy concern onto the same bean and can cause the `@Bean` interception to lose its semantics, plus you'd start transactions around bean factory methods — which is meaningless and can deadlock startup.
**Fix:** Move transactional logic into a `@Service` or `@Component` and inject it. Keep `@Configuration` strictly about wiring.
```java
@Service
public class TenantSeeder {
    @Transactional
    public void seed() { /* ... */ }
}
```
**Reference:** [Spring Framework — Declarative Transaction Management](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative.html)

### Bug 22 — Flush-order surprise on insert-then-delete
**Root cause:** Hibernate's `ActionQueue` flushes by action type in a fixed order: inserts → updates → deletes. So `save(new User(email))` runs *before* `delete(old)`, and the unique index on `email` rejects the insert with `ConstraintViolationException` because `old` is still in the DB.
**Fix:** Force a flush between the operations, or use a single `update` if you can:
```java
userRepository.delete(old);
userRepository.flush();           // emit DELETE now
userRepository.save(new User(email));
```
Or change the model so identity isn't tied to a unique non-id column.
**Reference:** Hibernate User Guide §6.4 "Flushing" (action queue ordering) — [Hibernate User Guide](https://docs.jboss.org/hibernate/orm/6.6/userguide/html_single/Hibernate_User_Guide.html)

## Related
- [JPA Transactions](jpa-transactions.md)
- [JPA Transaction Propagation](jpa-transaction-propagation.md)
- [JPA Stale Collection Trap](jpa-stale-collection-trap.md)
- [Inspecting JPA Cache (debugger walkthrough)](inspecting-jpa-cache-debugger.md)
- [Reactive Blocking JPA Pattern](reactive-blocking-jpa-pattern.md)

## References
- Hibernate User Guide (6.6): https://docs.jboss.org/hibernate/orm/6.6/userguide/html_single/Hibernate_User_Guide.html
- Spring Framework Reference — Declarative Transaction Management: https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative.html
- Spring Framework Reference — `@Transactional` Annotations (proxy mode, self-invocation, visibility): https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html
- Spring Framework Reference — Transaction Propagation: https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html
- Vlad Mihalcea — N+1 Query Problem: https://vladmihalcea.com/n-plus-1-query-problem/
- Vlad Mihalcea — equals/hashCode/toString with JPA: https://vladmihalcea.com/the-best-way-to-implement-equals-hashcode-and-tostring-with-jpa-and-hibernate/
- Vlad Mihalcea — Open Session in View Anti-Pattern: https://vladmihalcea.com/the-open-session-in-view-anti-pattern/
- Vlad Mihalcea — Spring `@Transactional` annotation: https://vladmihalcea.com/spring-transactional-annotation/
- Vlad Mihalcea — JPA & Hibernate Cascade Types: https://vladmihalcea.com/a-beginners-guide-to-jpa-and-hibernate-cascade-types/
- Vlad Mihalcea — Optimistic Locking with `@Version`: https://vladmihalcea.com/optimistic-locking-version-property-jpa-hibernate/
- Jakarta Persistence 3.2 Specification: https://jakarta.ee/specifications/persistence/3.2/
