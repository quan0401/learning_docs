---
title: "Interface Segregation Principle (ISP)"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, solid, isp, interfaces, design-principles]
---

# Interface Segregation Principle (ISP)

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `solid` `isp` `interfaces` `design-principles`

## Summary

The Interface Segregation Principle says clients should not be forced to depend on methods they do not use. Robert C. Martin's rule, sharpened in *Agile Software Development*, attacks "fat" interfaces by splitting them into many small, **role-based** interfaces — each shaped to one client's actual needs. ISP reduces accidental coupling, keeps recompilation surfaces small, and makes test doubles easier to write.

## Table of Contents

- [The Rule and the Pain It Solves](#the-rule-and-the-pain-it-solves)
- [Fat Interfaces: The Smell](#fat-interfaces-the-smell)
- [Role Interfaces](#role-interfaces)
- [Splitting by Client Need](#splitting-by-client-need)
- [Worked Example: A Multifunction Printer](#worked-example-a-multifunction-printer)
- [ISP in Practice](#isp-in-practice)
- [Related](#related)

## The Rule and the Pain It Solves

Martin's phrasing:

> Clients should not be forced to depend on methods they do not use.

When `ClientA` only needs methods 1 and 2 of `BigInterface`, but `BigInterface` also exposes methods 3, 4, and 5 used by `ClientB`, then `ClientA` is *transitively* coupled to changes in 3, 4, and 5. A signature change to method 4 forces `ClientA` to recompile, retest, and possibly redeploy — even though `ClientA` never calls method 4.

ISP is the principle that says: design interfaces from the **client's** perspective. The interface a client depends on should be no wider than what the client uses.

## Fat Interfaces: The Smell

A fat interface accumulates methods because someone, somewhere, needs each one. Symptoms:

- Implementations contain methods that throw `UnsupportedOperationException` (also an LSP smell).
- Test doubles for the interface are pages long, with most methods stubbed to defaults.
- A change in one method's signature ripples to dozens of unrelated callers.
- The interface mixes unrelated *concerns*: serialization, persistence, validation, logging.

```java
// Fat: every client must depend on every method
public interface UserService {
    User findById(long id);
    List<User> findAll();
    void save(User user);
    void delete(long id);
    void sendWelcomeEmail(User user);
    void exportToCsv(OutputStream out);
    byte[] generateAvatar(User user);
    AuditLog auditTrail(long id);
}
```

A controller that just needs `findById` is now coupled to email, CSV, avatars, and audit. If the avatar method's signature changes, the controller's tests must be re-checked.

## Role Interfaces

Martin Fowler distinguishes **header interfaces** (one big interface mirroring a class's public API) from **role interfaces** (small interfaces shaped around the role a collaborator plays for a specific client).

ISP is essentially a vote for role interfaces. The role tells you both *what the contract is* and *who needs it*.

```java
public interface UserFinder {
    User findById(long id);
    List<User> findAll();
}

public interface UserWriter {
    void save(User user);
    void delete(long id);
}

public interface WelcomeMailer {
    void sendWelcomeEmail(User user);
}

public interface UserExporter {
    void exportToCsv(OutputStream out);
}
```

A single class can implement several role interfaces:

```java
public class UserServiceImpl implements UserFinder, UserWriter {
    // ...
}
```

But each *client* only depends on the role it actually plays against. `UserController` depends on `UserFinder`. The signup flow depends on `UserWriter` and `WelcomeMailer`. Nobody depends on the union.

## Splitting by Client Need

The right axis to split along is **the client's vocabulary**, not arbitrary categorization. Two heuristics:

1. **Group methods that are always called together by the same caller.** That clustering reveals a coherent role.
2. **Watch what your test doubles look like.** If `MockBigInterface` has 90% no-op methods, the real client only used 10%, and that 10% is the role interface struggling to escape.

```typescript
// TypeScript: structural typing makes ISP feel natural
interface UserFinder {
    findById(id: number): Promise<User | null>;
    findAll(): Promise<User[]>;
}

interface UserWriter {
    save(user: User): Promise<void>;
    delete(id: number): Promise<void>;
}

class UserController {
    constructor(private readonly users: UserFinder) {}
    async show(id: number) {
        const u = await this.users.findById(id);
        return u;
    }
}

class SignupHandler {
    constructor(
        private readonly writer: UserWriter,
        private readonly mailer: { sendWelcomeEmail(u: User): Promise<void> }
    ) {}
    async signup(u: User) {
        await this.writer.save(u);
        await this.mailer.sendWelcomeEmail(u);
    }
}
```

In TypeScript, structural typing means the same concrete object can be passed where any compatible role is expected — no implements clause needed. ISP becomes the natural style.

## Worked Example: A Multifunction Printer

Martin's canonical example. A `Printer` interface starts simple, then management asks for fax, scan, and staple. Tempting fat version:

```java
public interface MultiFunctionDevice {
    void print(Document doc);
    void scan(Document doc);
    void fax(Document doc);
    void staple(Document doc);
}
```

Now you sell a low-end model that only prints. Its implementation must still implement `scan`, `fax`, and `staple` — typically by throwing or doing nothing. Worse, a client that only prints is now forced to know the device might fax.

ISP fix:

```java
public interface Printer { void print(Document doc); }
public interface Scanner { void scan(Document doc); }
public interface Fax     { void fax(Document doc); }
public interface Stapler { void staple(Document doc); }

public class BasicPrinter implements Printer { /* ... */ }

public class OfficeMachine implements Printer, Scanner, Fax, Stapler { /* ... */ }
```

```typescript
interface Printer { print(doc: Document): void; }
interface Scanner { scan(doc: Document): void; }
interface Fax     { fax(doc: Document): void; }
interface Stapler { staple(doc: Document): void; }

class BasicPrinter implements Printer {
    print(doc: Document): void { /* ... */ }
}

class OfficeMachine implements Printer, Scanner, Fax, Stapler {
    print(doc: Document): void { /* ... */ }
    scan(doc: Document): void { /* ... */ }
    fax(doc: Document): void { /* ... */ }
    staple(doc: Document): void { /* ... */ }
}
```

A reception-desk client that only prints types its dependency as `Printer`. It cannot accidentally call `fax`, and signature changes to `Fax` don't touch it.

## ISP in Practice

**Repository interfaces in Spring Data and similar.** Spring Data exposes `CrudRepository`, `PagingAndSortingRepository`, and `JpaRepository` — each progressively wider. Pick the narrowest your service truly needs.

**Web framework request handlers.** Express, Koa, NestJS controllers should depend on small service interfaces, not on a giant `BackendApi` blob. NestJS DI specifically encourages tokenized provider interfaces, which dovetails with ISP.

**Frontend stores and hooks.** A React component should consume the slice of state it actually reads, not the entire app store. Selectors are role interfaces in disguise.

**Database access.** A read-only service depends on a `Reader` interface, not on the full `Repository`. Tests can supply a fixture-backed `Reader` without touching write logic.

**Don't go feral.** ISP doesn't mean every method gets its own interface. Group methods that genuinely cohere around a single role and a single client conversation. One-method interfaces everywhere is just FunctionalAbuse with extra ceremony.

A reasonable rule of thumb: if removing any method from the interface would force the same client to drag in a *second* interface anyway, the interface is probably right-sized. If multiple clients use disjoint subsets, it's almost certainly two interfaces glued together.

## Related

- [Single Responsibility Principle (SRP)](./single-responsibility-principle.md)
- [Open/Closed Principle (OCP)](./open-closed-principle.md)
- [Liskov Substitution Principle (LSP)](./liskov-substitution-principle.md)
- [Dependency Inversion Principle (DIP)](./dependency-inversion-principle.md)
- [../oop-fundamentals/abstraction.md](../oop-fundamentals/abstraction.md)
- [../design-principles/coupling-and-cohesion.md](../design-principles/coupling-and-cohesion.md)
- [../solid/dependency-inversion-principle.md](../solid/dependency-inversion-principle.md)
