---
title: "Open/Closed Principle (OCP)"
date: 2026-05-02
updated: 2026-05-02
tags: [low-level-design, solid, ocp, polymorphism, design-principles]
---

# Open/Closed Principle (OCP)

**Date:** 2026-05-02 | **Updated:** 2026-05-02
**Tags:** `low-level-design` `solid` `ocp` `polymorphism` `design-principles`

## Summary

The Open/Closed Principle says software entities should be **open for extension but closed for modification** — you should be able to add new behavior without editing existing, working code. Bertrand Meyer coined the principle in *Object-Oriented Software Construction* (1988); Robert C. Martin reframed it around polymorphism and abstraction. In practice, OCP is achieved through inheritance, the strategy pattern, plug-in architectures, and dependency inversion.

## Table of Contents

- [Bertrand Meyer's Original Formulation](#bertrand-meyers-original-formulation)
- [Martin's Polymorphic Reframe](#martins-polymorphic-reframe)
- [The Switch-Statement Smell](#the-switch-statement-smell)
- [Extension via Strategy](#extension-via-strategy)
- [Extension via Inheritance](#extension-via-inheritance)
- [Plug-in Architectures](#plug-in-architectures)
- [Limits of OCP](#limits-of-ocp)
- [Related](#related)

## Bertrand Meyer's Original Formulation

Meyer introduced OCP in 1988. His original mechanism was **inheritance**: a class that ships and gets used should not be edited; new behavior should arrive via subclasses. In Eiffel, where Meyer designed the idea, this fit naturally with class invariants and Design by Contract.

The early reading was strict: once a class was published, you literally never modified it. New requirements meant new subclasses.

## Martin's Polymorphic Reframe

Modern OCP, as in *Clean Architecture* and *Agile Software Development*, is broader. The mechanism shifts from inheritance to **abstraction + polymorphism**:

- Identify what is likely to vary.
- Hide that variation behind an abstraction (interface, abstract class).
- Depend on the abstraction. Add new implementations to add new behavior.

The "closed" half means: existing modules that depend on the abstraction shouldn't need recompilation, retesting, or redeployment when a new implementation lands.

OCP is a goal, not a guarantee. You aim for the *axes of change you actually expect*. Trying to be open to *every* possible change is over-engineering — and ironically often violates YAGNI and the Single Responsibility Principle.

## The Switch-Statement Smell

The textbook OCP violation:

```java
// Violates OCP: every new shape edits this class
public class AreaCalculator {
    public double area(Object shape) {
        if (shape instanceof Rectangle r) {
            return r.width * r.height;
        } else if (shape instanceof Circle c) {
            return Math.PI * c.radius * c.radius;
        } else if (shape instanceof Triangle t) {
            return 0.5 * t.base * t.height;
        }
        throw new IllegalArgumentException("Unknown shape");
    }
}
```

Every new shape forces a code change in `AreaCalculator`, plus retesting and redeployment of every consumer. Multiply by 20 calculators all switching on shape type, and adding a hexagon means surgery in 20 places.

## Extension via Strategy

Push the varying logic behind an interface and make each shape responsible for its own area.

```java
public interface Shape {
    double area();
}

public final class Rectangle implements Shape {
    private final double width, height;
    public Rectangle(double w, double h) { this.width = w; this.height = h; }
    @Override public double area() { return width * height; }
}

public final class Circle implements Shape {
    private final double radius;
    public Circle(double r) { this.radius = r; }
    @Override public double area() { return Math.PI * radius * radius; }
}

public class AreaCalculator {
    public double totalArea(List<Shape> shapes) {
        return shapes.stream().mapToDouble(Shape::area).sum();
    }
}
```

Adding a `Hexagon` means adding one new class. `AreaCalculator` is closed.

The TypeScript version uses the same shape:

```typescript
interface Shape {
    area(): number;
}

class Rectangle implements Shape {
    constructor(private readonly width: number, private readonly height: number) {}
    area(): number { return this.width * this.height; }
}

class Circle implements Shape {
    constructor(private readonly radius: number) {}
    area(): number { return Math.PI * this.radius ** 2; }
}

class AreaCalculator {
    totalArea(shapes: readonly Shape[]): number {
        return shapes.reduce((sum, s) => sum + s.area(), 0);
    }
}
```

## Extension via Inheritance

Inheritance is still a valid OCP mechanism, especially when subclasses customize a few hooks while inheriting the bulk of behavior. The Template Method pattern is the canonical example:

```java
public abstract class ReportGenerator {
    public final String generate(ReportData data) {
        var header = renderHeader(data);
        var body = renderBody(data);
        var footer = renderFooter(data);
        return header + body + footer;
    }

    protected abstract String renderBody(ReportData data);

    protected String renderHeader(ReportData data) { return "=== " + data.title() + " ==="; }
    protected String renderFooter(ReportData data) { return "--- generated " + Instant.now() + " ---"; }
}

public class SalesReportGenerator extends ReportGenerator {
    @Override
    protected String renderBody(ReportData data) {
        // sales-specific layout
        return data.rows().stream().map(this::formatRow).collect(joining("\n"));
    }
    private String formatRow(Row r) { /* ... */ return ""; }
}
```

New report types extend the base class; the orchestration in `generate()` stays closed.

## Plug-in Architectures

At the system scale, OCP becomes a plug-in or extension architecture. The host process discovers implementations at runtime — via service loaders, DI containers, or dynamic imports — and never edits its own dispatch logic to add a new one.

Java's `ServiceLoader`:

```java
public interface PaymentProvider {
    String name();
    PaymentResult charge(ChargeRequest req);
}

// In a host service
ServiceLoader<PaymentProvider> providers = ServiceLoader.load(PaymentProvider.class);
Map<String, PaymentProvider> byName = providers.stream()
    .map(ServiceLoader.Provider::get)
    .collect(toMap(PaymentProvider::name, p -> p));
```

A new payment method ships as a JAR with a `META-INF/services` entry. The host code is closed.

Spring's component scanning achieves the same effect via DI: any `@Component` implementing `PaymentProvider` is auto-registered.

```typescript
// NestJS / TypeScript: providers register themselves
export interface PaymentProvider {
    readonly name: string;
    charge(req: ChargeRequest): Promise<PaymentResult>;
}

@Injectable()
export class StripeProvider implements PaymentProvider {
    readonly name = "stripe";
    async charge(req: ChargeRequest): Promise<PaymentResult> { /* ... */ return { ok: true }; }
}

@Injectable()
export class PaymentService {
    private readonly registry: Map<string, PaymentProvider>;
    constructor(@Inject("PROVIDERS") providers: PaymentProvider[]) {
        this.registry = new Map(providers.map(p => [p.name, p]));
    }
    charge(name: string, req: ChargeRequest) {
        const p = this.registry.get(name);
        if (!p) throw new Error(`unknown provider ${name}`);
        return p.charge(req);
    }
}
```

## Limits of OCP

OCP is aspirational. You cannot anticipate every axis of change, and over-abstracting "just in case" produces a maze of interfaces with one implementation each. Practical guidance:

- **Wait for the second case.** The first time, hard-code it. The second time, abstract it. The third time, you understand the variation.
- **Be open along the axes you've seen vary.** Payment providers? Yes — those change. The number of dimensions on a vector? Probably no — that's stable.
- **Closed doesn't mean immutable.** Bug fixes still happen. Closed means *new requirements* don't force edits to working code.
- **OCP rests on DIP.** You can only be closed if the high-level module depends on an abstraction, not on a concrete class. See the Dependency Inversion doc.

OCP, well applied, makes a codebase feel like a tree where new branches grow without disturbing the trunk. Poorly applied, it produces an interface for every class and a configuration file for every constant. The judgment is in seeing which axes truly vary.

## Related

- [Single Responsibility Principle (SRP)](./single-responsibility-principle.md)
- [Liskov Substitution Principle (LSP)](./liskov-substitution-principle.md)
- [Interface Segregation Principle (ISP)](./interface-segregation-principle.md)
- [Dependency Inversion Principle (DIP)](./dependency-inversion-principle.md)
- [../oop-fundamentals/polymorphism.md](../oop-fundamentals/polymorphism.md)
- [../oop-fundamentals/abstraction.md](../oop-fundamentals/abstraction.md)
- [../design-principles/composing-objects-principle.md](../design-principles/composing-objects-principle.md)
