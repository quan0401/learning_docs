---
title: "Dependency Inversion Principle (DIP)"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, solid, dip, dependency-injection, testability]
---

# Dependency Inversion Principle (DIP)

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `solid` `dip` `dependency-injection` `testability`

## Summary

The Dependency Inversion Principle says **high-level modules should not depend on low-level modules; both should depend on abstractions**, and **abstractions should not depend on details — details depend on abstractions**. Robert C. Martin formulated it as the foundation of decoupled architecture. DIP is what makes the other SOLID principles operational, and it is the keystone of testability. Inversion of Control (IoC) is the broader pattern; Dependency Injection (DI) — as in Spring or NestJS — is the most common mechanism that realizes it.

## Table of Contents

- [The Two Rules](#the-two-rules)
- [Why "Inversion"](#why-inversion)
- [DIP, IoC, and DI Are Not the Same Thing](#dip-ioc-and-di-are-not-the-same-thing)
- [DIP as the Foundation of Testability](#dip-as-the-foundation-of-testability)
- [Spring DI](#spring-di)
- [NestJS DI](#nestjs-di)
- [Common Mistakes](#common-mistakes)
- [Related](#related)

## The Two Rules

From *Agile Software Development* and *Clean Architecture*:

1. High-level modules should not depend on low-level modules. Both should depend on abstractions.
2. Abstractions should not depend on details. Details should depend on abstractions.

"High-level" here means policy — business rules, use cases, what the application *does*. "Low-level" means mechanism — databases, HTTP clients, file systems, message queues, third-party APIs.

The traditional, naive direction of dependency runs top-down with the call graph: the controller calls the service, the service calls the SQL repository, the SQL repository calls the JDBC driver. Each layer imports the one below it. DIP says: invert that. The service should not import a SQL repository class; it should depend on a `UserRepository` abstraction it owns, and the SQL implementation should plug into that abstraction from below.

## Why "Inversion"

Without DIP:

```
Service  ─►  SqlUserRepository  ─►  JDBC
(policy)     (mechanism)            (detail)
```

The arrow of dependency follows the arrow of *control*. Policy is shackled to mechanism.

With DIP:

```
Service  ─►  UserRepository (interface)  ◄─  SqlUserRepository
(policy)     (abstraction owned          (mechanism plugs in)
              by the policy layer)
```

Control still flows policy → repository → JDBC at runtime. But *source-code dependency* of `Service` is on the abstraction, not the SQL class. The mechanism layer's source-code arrow now points *up* into the abstraction. That's the inversion.

Concrete payoff: you can swap SQL for an in-memory store, a Mongo store, or a fake without touching `Service`. The policy code is decoupled from the mechanism choice.

## DIP, IoC, and DI Are Not the Same Thing

Three terms get conflated. They're related but distinct.

**Dependency Inversion Principle** — a *design principle* about which way dependencies should point. It says nothing about *how* you achieve that inversion.

**Inversion of Control (IoC)** — a *broader pattern* where the framework or container drives the application, instead of the application driving the framework. The Hollywood principle: "don't call us, we'll call you." Examples: event loops, frameworks calling your handlers, lifecycle hooks.

**Dependency Injection (DI)** — a specific *technique* for realizing DIP and IoC. Instead of a class constructing its own collaborators (`new SqlUserRepository()`), the collaborators are *injected* — handed in via constructor parameters, setters, or framework wiring.

You can do DIP without DI (e.g., service locator pattern, factory). You can use DI containers without truly inverting dependencies (if you inject concrete classes, you've moved wiring around but kept the same coupling). DIP is the goal; DI is the most common path; IoC is the family the path belongs to.

```java
// Not inverted: service constructs and depends on a concrete
public class UserService {
    private final SqlUserRepository repo = new SqlUserRepository();
    public User get(long id) { return repo.findById(id); }
}

// Inverted: service depends on an abstraction it owns
public interface UserRepository {
    User findById(long id);
}

public class UserService {
    private final UserRepository repo;
    public UserService(UserRepository repo) { this.repo = repo; }
    public User get(long id) { return repo.findById(id); }
}

public class SqlUserRepository implements UserRepository { /* ... */ }
public class InMemoryUserRepository implements UserRepository { /* ... */ }
```

## DIP as the Foundation of Testability

This is where DIP earns its reputation. Without DIP, unit tests for `UserService` must spin up a database (or a heavy mock framework that intercepts JDBC). With DIP, you instantiate the service with an in-memory fake:

```java
@Test
void getReturnsUserWhenPresent() {
    var repo = new InMemoryUserRepository();
    repo.save(new User(1L, "alice"));
    var service = new UserService(repo);

    var result = service.get(1L);

    assertThat(result.name()).isEqualTo("alice");
}
```

No SQL, no containers, no transaction management. The test runs in milliseconds and exercises only the policy.

The pattern repeats for every external mechanism: time (`Clock` interface), randomness (`RandomSource`), HTTP (`HttpClient` abstraction), filesystem, message queue, payment gateway. Each becomes a seam where tests inject fakes and production injects real implementations. This is also why mocking frameworks feel awkward when used to mock concretes — DIP-shaped code rarely needs them, because the abstraction is already there.

```typescript
interface UserRepository {
    findById(id: number): Promise<User | null>;
    save(user: User): Promise<void>;
}

class UserService {
    constructor(private readonly repo: UserRepository) {}
    async get(id: number): Promise<User | null> {
        return this.repo.findById(id);
    }
}

// Test
class InMemoryUserRepository implements UserRepository {
    private store = new Map<number, User>();
    async findById(id: number) { return this.store.get(id) ?? null; }
    async save(u: User) { this.store.set(u.id, u); }
}

test("get returns user when present", async () => {
    const repo = new InMemoryUserRepository();
    await repo.save({ id: 1, name: "alice" });
    const service = new UserService(repo);

    const result = await service.get(1);

    expect(result?.name).toBe("alice");
});
```

## Spring DI

Spring's IoC container is the textbook DI implementation in the Java world. You declare components and their collaborators; Spring wires them.

```java
public interface UserRepository { User findById(long id); }

@Repository
public class JpaUserRepository implements UserRepository {
    private final EntityManager em;
    public JpaUserRepository(EntityManager em) { this.em = em; }
    public User findById(long id) { return em.find(User.class, id); }
}

@Service
public class UserService {
    private final UserRepository repo;
    public UserService(UserRepository repo) { this.repo = repo; }
    public User get(long id) { return repo.findById(id); }
}

@RestController
@RequestMapping("/users")
public class UserController {
    private final UserService service;
    public UserController(UserService service) { this.service = service; }
    @GetMapping("/{id}") public User get(@PathVariable long id) { return service.get(id); }
}
```

Notes:

- **Constructor injection is preferred.** It produces immutable collaborators, makes tests trivially constructible without Spring, and exposes missing dependencies at compile time. Field injection (`@Autowired` on fields) is discouraged precisely because it bypasses these properties.
- **Depend on the interface, not the implementation.** `UserService` takes `UserRepository`, not `JpaUserRepository`. Swapping JPA for a different store doesn't touch the service.
- **Spring is a means, not the goal.** A well-DIP-shaped codebase can be tested without Spring; Spring just supplies the production wiring.

## NestJS DI

NestJS, heavily inspired by Angular, brings the same model to Node/TypeScript with a module + provider system.

```typescript
// user-repository.interface.ts
export interface UserRepository {
    findById(id: number): Promise<User | null>;
}
export const USER_REPOSITORY = Symbol("USER_REPOSITORY");

// prisma-user.repository.ts
@Injectable()
export class PrismaUserRepository implements UserRepository {
    constructor(private readonly prisma: PrismaService) {}
    findById(id: number) { return this.prisma.user.findUnique({ where: { id } }); }
}

// user.service.ts
@Injectable()
export class UserService {
    constructor(@Inject(USER_REPOSITORY) private readonly repo: UserRepository) {}
    get(id: number) { return this.repo.findById(id); }
}

// user.module.ts
@Module({
    providers: [
        { provide: USER_REPOSITORY, useClass: PrismaUserRepository },
        UserService,
    ],
    exports: [UserService],
})
export class UserModule {}
```

Because TypeScript erases interfaces at runtime, NestJS uses **injection tokens** (typically `Symbol`s or strings) to identify the abstract dependency. The token is the runtime stand-in for the interface. Tests substitute another provider mapped to the same token:

```typescript
const moduleRef = await Test.createTestingModule({
    providers: [
        UserService,
        { provide: USER_REPOSITORY, useValue: new InMemoryUserRepository() },
    ],
}).compile();
```

Same DIP shape; same testability payoff; framework-specific wiring.

## Common Mistakes

**Injecting concretes.** `constructor(repo: SqlUserRepository)` uses DI mechanics but defeats DIP. The arrow of source-code dependency still points at SQL.

**Abstractions that mirror one implementation.** If your `UserRepository` interface has exactly the same shape as `JpaUserRepository` and was extracted reflexively, it's not buying you decoupling — it's a tax on the next change. Real abstractions are stated in the *policy's* vocabulary, not the mechanism's.

**Leaking framework types into the abstraction.** A `UserRepository` whose method returns a Spring `Page<User>` or a Mongoose `Document` has dragged the mechanism back into the abstraction. The abstraction is no longer truly stable.

**Service locator masquerading as DI.** Code that calls `Container.get(UserRepository)` from inside business logic keeps the dependency hidden. Constructor injection makes dependencies explicit; service locators hide them.

**Over-abstracting.** DIP is most valuable across boundaries that genuinely vary or matter for testing. Inverting every internal collaborator produces interface-itis. Start by inverting at the I/O edge — databases, HTTP clients, time, randomness, queues — and work inward only when there's a real reason.

DIP is the principle that decides whether your codebase ages well. SRP, OCP, LSP, and ISP all assume you have abstractions to work with — DIP is the rule that says: own the abstraction, point dependencies at it, and let the details plug in.

## Related

- [Single Responsibility Principle (SRP)](./single-responsibility-principle.md)
- [Open/Closed Principle (OCP)](./open-closed-principle.md)
- [Liskov Substitution Principle (LSP)](./liskov-substitution-principle.md)
- [Interface Segregation Principle (ISP)](./interface-segregation-principle.md)
- [../oop-fundamentals/abstraction.md](../oop-fundamentals/abstraction.md)
- [../design-principles/coupling-and-cohesion.md](../design-principles/coupling-and-cohesion.md)
- [../solid/dependency-inversion-principle.md](../solid/dependency-inversion-principle.md)
- [../solid/dependency-inversion-principle.md](../solid/dependency-inversion-principle.md)
