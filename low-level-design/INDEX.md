# Low-Level Design Documentation Index — Learning Path

A progressive path from OOP fundamentals through SOLID, design patterns, UML, and the canonical LLD design problems. This is the **class-level / object-oriented design** layer — what you draw on a whiteboard *after* the [system-design path](../system-design/INDEX.md) has decided the high-level architecture.

LLD answers: how do I model entities, what classes do I create, what relationships and patterns hold the design together, how do I keep the code testable and extensible. It's the Software Engineering 2 of OO design — the bridge between data structures and architecture.

Cross-references to the [System Design learning path](../system-design/INDEX.md) (HLD), the [Java learning path](../java/INDEX.md), the [TypeScript learning path](../typescript/INDEX.md), the [Database learning path](../database/INDEX.md), the [Networking learning path](../networking/INDEX.md), and the [Kubernetes learning path](../kubernetes/INDEX.md) where topics overlap.

If you already write production OO code, **skim Tiers 1–3 and dive into Tiers 4–6 (principles, SOLID, UML)**. The interview-style case studies in Tier 12 exercise everything from Tiers 1–11.

> Note: Some Tier 12 problems (URL Shortener, Rate Limiter, Search Autocomplete, Chat, Notifications) appear in both the [system-design Tier 10 case studies](../system-design/INDEX.md) and here. The system-design version is **HLD** (capacity, sharding, replication, infra topology); the LLD version below is **class-level OOD** (entities, relationships, patterns, interfaces). Different deliverables for the same problem.

---

## Tier 1 — LLD Foundations

What LLD is, where it sits relative to HLD, and how interviews exercise it.

1. [What Is Low-Level Design?](foundations/what-is-lld.md) — class-level OOD vs system-level architecture, deliverables, audience, when LLD matters _(2026-05-02)_
2. [LLD vs HLD — Where the Boundary Sits](foundations/lld-vs-hld.md) — what each design pass produces, hand-off contracts, common confusion, typical interview split _(2026-05-02)_
3. [Types of LLD Interviews — OOD, Machine Coding, Live Pairing](foundations/types-of-lld-interviews.md) — OOD whiteboard, machine-coding (60–120 min), live pairing, what each evaluates _(2026-05-02)_

---

## Tier 2 — OOP Fundamentals

The vocabulary every LLD discussion assumes. Skim if you're senior; come back when a junior asks "what's encapsulation actually for?"

4. [Classes and Objects](oop-fundamentals/classes-and-objects.md) — class definition, instantiation, state vs behavior, identity vs equality, value vs reference semantics _(2026-05-02)_
5. [Enums — Bounded Sets and Type-Safe Constants](oop-fundamentals/enums.md) — enum vs constant, behavior on enums, sealed enum hierarchies, Java vs TS vs C# differences _(2026-05-02)_
6. [Interfaces — Contracts Without Implementation](oop-fundamentals/interfaces.md) — interface as a contract, default methods, multiple interface inheritance, interface segregation preview _(2026-05-02)_
7. [Encapsulation — Hiding the Right Things](oop-fundamentals/encapsulation.md) — visibility modifiers, getters/setters reality, immutability, encapsulation vs naked records _(2026-05-02)_
8. [Abstraction — Modeling at the Right Level](oop-fundamentals/abstraction.md) — abstract classes, abstract data types, picking the abstraction line, leaky abstractions _(2026-05-02)_
9. [Inheritance — When and When Not](oop-fundamentals/inheritance.md) — is-a relationships, single vs multiple inheritance, the diamond problem, composition over inheritance preview _(2026-05-02)_
10. [Polymorphism — Subtype, Parametric, Ad-Hoc](oop-fundamentals/polymorphism.md) — subtype polymorphism, generics, method overloading, dynamic dispatch, polymorphism in design patterns _(2026-05-02)_

---

## Tier 3 — Class Relationships

The arrows between boxes on a UML class diagram. Naming the relationship is half the design discussion.

11. [Association — A Knows About B](class-relationships/association.md) — bidirectional vs unidirectional, navigability, multiplicity, the basic "uses" relationship _(2026-05-02)_
12. [Aggregation — A Has B (Loose)](class-relationships/aggregation.md) — shared lifetime, parts can outlive whole, "has-a" without ownership _(2026-05-02)_
13. [Composition — A Owns B (Strong)](class-relationships/composition.md) — coincident lifetime, parts die with whole, exclusive ownership _(2026-05-02)_
14. [Dependency — A Uses B Briefly](class-relationships/dependency.md) — transient coupling via parameter/return/local, weakest relationship, easy to invert _(2026-05-02)_
15. [Realization — A Implements Interface B](class-relationships/realization.md) — implements/realizes, interface vs class hierarchy, in UML notation _(2026-05-02)_

---

## Tier 4 — Design Principles

The principles that predate SOLID — the broader hygiene rules every clean codebase follows.

16. [DRY — Don't Repeat Yourself](design-principles/dry-principle.md) — what counts as duplication, when DRY makes things worse (premature abstraction), rule of three _(2026-05-02)_
17. [KISS — Keep It Simple, Stupid](design-principles/kiss-principle.md) — simplicity as a feature, complexity budget, "smart" code as a code smell _(2026-05-02)_
18. [YAGNI — You Aren't Gonna Need It](design-principles/yagni-principle.md) — speculative generality, hooks for nothing, when YAGNI is wrong (irreversible decisions) _(2026-05-02)_
19. [Law of Demeter — Talk Only to Friends](design-principles/law-of-demeter.md) — train-wreck calls, what counts as a violation, when LoD makes the API worse _(2026-05-02)_
20. [Separation of Concerns](design-principles/separation-of-concerns.md) — each module one concern, cross-cutting concerns (logging, security), how SoC drives layering _(2026-05-02)_
21. [Coupling and Cohesion](design-principles/coupling-and-cohesion.md) — afferent/efferent coupling, cohesion types (logical, temporal, sequential, functional), the high-cohesion/low-coupling rule _(2026-05-02)_
22. [Composing Objects Principle — Composition Over Inheritance](design-principles/composing-objects-principle.md) — why "is-a" via inheritance gets brittle, composition + interfaces as the default _(2026-05-02)_
22a. [Code Smells and Refactoring Triggers](design-principles/code-smells-and-refactoring-triggers.md) — long method, large class, feature envy, shotgun surgery, primitive obsession, data clumps; Fowler's catalog as a design checklist _(2026-05-02)_
22b. [Anti-Patterns in OO Design](design-principles/anti-patterns-in-oo-design.md) — god object, anemic domain model, spaghetti, lasagna, golden hammer, premature optimization, copy-paste programming _(2026-05-02)_

---

## Tier 5 — SOLID Principles

The five canonical OO design principles. Memorize, then internalize.

23. [Single Responsibility Principle (SRP)](solid/single-responsibility-principle.md) — one reason to change, "actor" interpretation, splitting god classes, SRP at module vs class level _(2026-05-02)_
24. [Open/Closed Principle (OCP)](solid/open-closed-principle.md) — open for extension, closed for modification, extension via inheritance vs composition vs strategy _(2026-05-02)_
25. [Liskov Substitution Principle (LSP)](solid/liskov-substitution-principle.md) — subtype contracts, preconditions/postconditions/invariants, the rectangle/square classic, why LSP matters _(2026-05-02)_
26. [Interface Segregation Principle (ISP)](solid/interface-segregation-principle.md) — fat interfaces, role interfaces, splitting by client need _(2026-05-02)_
27. [Dependency Inversion Principle (DIP)](solid/dependency-inversion-principle.md) — depend on abstractions, IoC vs DI, DIP as the foundation of testability _(2026-05-02)_

---

## Tier 6 — UML Diagrams

Just enough UML to communicate. Don't draw a diagram in 14 styles — draw the right diagram in one style.

28. [Class Diagram](uml/class-diagram.md) — boxes, lines, multiplicity, relationships, when to draw, how to keep it useful _(2026-05-02)_
29. [Use Case Diagram](uml/use-case-diagram.md) — actors, use cases, boundary, includes/extends, when use cases beat user stories _(2026-05-02)_
30. [Sequence Diagram](uml/sequence-diagram.md) — lifelines, messages, activations, alt/loop fragments, sequence vs flowchart _(2026-05-02)_
31. [Activity Diagram](uml/activity-diagram.md) — flow modeling, swim lanes, decisions, parallel flows, when activity beats sequence _(2026-05-02)_
32. [State Machine Diagram](uml/state-machine-diagram.md) — states, transitions, guards, hierarchical states, when to draw a state machine instead of writing if-statements _(2026-05-02)_

---

## Tier 7 — Creational Design Patterns

Object creation patterns. Pick one when "new ConcreteClass()" doesn't fit.

33. [Singleton — One Instance, Globally Accessible](design-patterns/creational/singleton.md) — thread-safe singletons, double-checked locking, enum singleton (Java), when singleton is wrong (testability) _(2026-05-02)_
34. [Builder — Step-by-Step Object Construction](design-patterns/creational/builder.md) — fluent builders, immutable result, vs telescoping constructors, Lombok @Builder, TS literal builders _(2026-05-02)_
35. [Factory Method — Subclass Decides the Concrete Type](design-patterns/creational/factory-method.md) — vs static factory methods, when factory method beats new(), framework hooks _(2026-05-02)_
36. [Abstract Factory — Families of Related Products](design-patterns/creational/abstract-factory.md) — factory of factories, when "skin" or "platform" varies, GUI toolkit example _(2026-05-02)_
37. [Prototype — Clone Existing Instances](design-patterns/creational/prototype.md) — shallow vs deep clone, registry of prototypes, when cloning beats constructing _(2026-05-02)_

---

## Tier 8 — Structural Design Patterns

How classes and objects compose into larger structures.

38. [Adapter — Make Incompatible Interfaces Talk](design-patterns/structural/adapter.md) — class adapter (inheritance) vs object adapter (composition), legacy integration, the ports-and-adapters tie-in _(2026-05-02)_
39. [Facade — One Door Into a Subsystem](design-patterns/structural/facade.md) — simplifying a complex API, vs the BFF pattern, what facade is NOT (god class) _(2026-05-02)_
40. [Decorator — Add Behavior Without Subclassing](design-patterns/structural/decorator.md) — dynamic composition of behavior, java.io stream wrappers, vs inheritance explosion _(2026-05-02)_
41. [Composite — Trees of Uniform Nodes](design-patterns/structural/composite.md) — leaf vs composite, recursive operations, file system / DOM examples _(2026-05-02)_
42. [Proxy — Stand-In With Extra Behavior](design-patterns/structural/proxy.md) — virtual / protection / remote / smart proxies, vs decorator, lazy loading, ORM proxies _(2026-05-02)_
43. [Bridge — Decouple Abstraction From Implementation](design-patterns/structural/bridge.md) — orthogonal hierarchies, drivers, when "two dimensions of variation" appear _(2026-05-02)_
44. [Flyweight — Share Intrinsic State Across Many Objects](design-patterns/structural/flyweight.md) — intrinsic vs extrinsic state, glyph / particle / icon caches, memory wins _(2026-05-02)_

---

## Tier 9 — Behavioral Design Patterns

How objects collaborate and distribute responsibilities.

45. [Strategy — Swap Algorithms at Runtime](design-patterns/behavioral/strategy.md) — strategy interface + context, vs if/else chains, lambdas as lightweight strategies _(2026-05-02)_
46. [Iterator — Traverse a Collection Without Exposing It](design-patterns/behavioral/iterator.md) — internal vs external iterators, language built-ins (Iterable, IEnumerable), generators _(2026-05-02)_
47. [Observer — Many Subscribers React to State Change](design-patterns/behavioral/observer.md) — pub/sub at object level, push vs pull, leaks via dangling subscriptions, RxJS / Reactor link _(2026-05-02)_
48. [Command — Encapsulate a Request as an Object](design-patterns/behavioral/command.md) — undo/redo, queues of commands, request logging, command handlers in CQRS _(2026-05-02)_
49. [State — Behavior Varies With Internal State](design-patterns/behavioral/state.md) — state classes vs enum-driven if/else, transitions, vs strategy (state owns transitions) _(2026-05-02)_
50. [Template Method — Skeleton With Hookable Steps](design-patterns/behavioral/template-method.md) — abstract base + hook methods, when inheritance is the right tool, vs strategy via composition _(2026-05-02)_
51. [Chain of Responsibility — Pass Request Down a Chain](design-patterns/behavioral/chain-of-responsibility.md) — middleware pipelines, servlet filters, Express middleware, when chain beats explicit dispatch _(2026-05-02)_
52. [Visitor — Add Operations Without Modifying Classes](design-patterns/behavioral/visitor.md) — double dispatch, when expression trees / AST traversal benefit, the schema-evolution cost _(2026-05-02)_
53. [Mediator — Centralize Inter-Object Communication](design-patterns/behavioral/mediator.md) — replacing N×N coupling with N×1 to mediator, chat rooms, UI dialogs, vs event bus _(2026-05-02)_
54. [Memento — Capture and Restore Internal State](design-patterns/behavioral/memento.md) — undo/redo with originator/caretaker/memento, snapshotting, vs serializing all state _(2026-05-02)_

---

## Tier 10 — Additional Patterns

Patterns that predate or postdate the GoF book but show up daily in real codebases.

55. [Null Object Pattern](design-patterns/additional/null-object-pattern.md) — replace null with a benign no-op object, removing if-null branches, no-op logger / no-op auth _(2026-05-02)_
56. [Repository Pattern](design-patterns/additional/repository-pattern.md) — collection-style abstraction over data access, decoupling domain from persistence, when it pays off vs ORM-direct _(2026-05-02)_
57. [MVC Pattern — Model, View, Controller](design-patterns/additional/mvc-pattern.md) — classic MVC, vs MVP, vs MVVM, where business logic actually lives _(2026-05-02)_
58. [Dependency Injection Pattern](design-patterns/additional/dependency-injection-pattern.md) — constructor / setter / interface injection, DI containers (Spring, Guice, NestJS), when DI is overkill _(2026-05-02)_
59. [Specification Pattern](design-patterns/additional/specification-pattern.md) — composable query/business-rule predicates, AND/OR/NOT combinators, evaluator vs query-translator forms _(2026-05-02)_
60. [Game Loop Pattern](design-patterns/additional/game-loop-pattern.md) — fixed timestep, variable timestep, accumulator pattern, decoupling update rate from render rate _(2026-05-02)_
61. [Thread Pool Pattern](design-patterns/additional/thread-pool-pattern.md) — bounded worker pool, queue, work stealing, sizing (Little's Law), vs reactive / virtual threads _(2026-05-02)_
62. [Producer-Consumer Pattern](design-patterns/additional/producer-consumer-pattern.md) — bounded buffer, blocking queue, backpressure, classic concurrency primitive _(2026-05-02)_

---

## Tier 11 — LLD Interview Method

How to actually run an LLD interview — useful even outside interviews because it mirrors a small design review.

63. [How to Approach OOD Interviews](interview-method/approach-ood-interviews.md) — the canonical 5-step structure (clarify → entities → relationships → behaviors → diagram), time allocation, common stumbles _(2026-05-02)_
64. [How to Approach Machine-Coding Interviews](interview-method/approach-machine-coding-interviews.md) — 60-90 minute codable design problems, scaffolding strategy, what to skip vs polish _(2026-05-02)_
65. [How to Identify Entities and Model Relationships](interview-method/identify-entities-and-model-relationships.md) — noun extraction, role identification, multiplicity decisions, validating with sample flows _(2026-05-02)_
66. [How to Write Clean Code in an Interview](interview-method/write-clean-code-in-interview.md) — naming, function size, comments, error handling, where to invest reviewer time _(2026-05-02)_
67. [How to Choose Design Patterns Without Forcing Them](interview-method/choose-design-patterns.md) — pattern intent vs pattern shape, justifying choices, the "have I named the trade-off" rule _(2026-05-02)_
68. [How to Handle Concurrency Scenarios](interview-method/handle-concurrency-scenarios.md) — shared state, locking strategies, immutability, async patterns, what concurrency the interviewer expects vs what they don't _(2026-05-02)_
68a. [Test Doubles — Mocks, Stubs, Fakes, Spies](interview-method/test-doubles-in-ood.md) — Meszaros's taxonomy, when to use which, test-doubles as a design-quality signal, DIP making them possible _(2026-05-02)_

---

## Tier 12 — Design Problems (Case Studies)

The canonical LLD problem set. Each doc follows the same template: requirements → entities → class diagram → key methods → patterns used → concurrency → trade-offs.

### 12.A Games & Puzzles

69. [Design Tic-Tac-Toe](case-studies/games/design-tic-tac-toe.md) — board model, win-detection algorithm, player abstraction, state pattern for game phases _(2026-05-02, easy)_
70. [Design Snake and Ladder](case-studies/games/design-snake-and-ladder.md) — board with jumps, dice abstraction, player turn order, configurable rules _(2026-05-02, easy)_
71. [Design Minesweeper](case-studies/games/design-minesweeper.md) — grid + cell state machine, flood-fill reveal, mine placement, observer pattern for UI _(2026-05-02, medium)_
72. [Design Chess](case-studies/games/design-chess.md) — piece hierarchy with strategy per move type, board, move validation, check/checkmate detection _(2026-05-02, hard)_

### 12.B Data Structures & Search

73. [Design LRU Cache](case-studies/data-structures/design-lru-cache.md) — hash map + doubly-linked list, O(1) get/put, eviction; thread-safe variant _(2026-05-02, easy)_
74. [Design Bloom Filter](case-studies/data-structures/design-bloom-filter.md) — bit array, multiple hash functions, false-positive math, scalable / counting bloom variants _(2026-05-02, easy)_
75. [Design Search Autocomplete](case-studies/data-structures/design-search-autocomplete.md) — trie + ranking, top-K per prefix, async fetch, debouncing _(2026-05-02, easy)_
76. [Design Simple Search Engine](case-studies/data-structures/design-simple-search-engine.md) — inverted index, tokenizer, ranking, query parsing, document store _(2026-05-02, medium)_

### 12.C State-Machine Systems

77. [Design ATM](case-studies/state-machines/design-atm.md) — card/PIN/transaction state machine, account/cash dispense components, withdrawal flow _(2026-05-02, medium)_
78. [Design Vending Machine](case-studies/state-machines/design-vending-machine.md) — state pattern (idle → coin → product → dispense), inventory, change-making _(2026-05-02, medium)_
79. [Design Elevator System](case-studies/state-machines/design-elevator-system.md) — multi-elevator scheduling (SCAN/LOOK), request queues, direction state, observer for UI _(2026-05-02, medium)_
80. [Design Traffic Control System](case-studies/state-machines/design-traffic-control-system.md) — intersection state machine, signal phases, sensor integration, emergency override _(2026-05-02, medium)_
81. [Design Coffee Vending Machine](case-studies/state-machines/design-coffee-vending-machine.md) — recipe builder, ingredient inventory, payment + state machine, beverage variants _(2026-05-02, hard)_

### 12.D Management Systems

82. [Design Parking Lot](case-studies/management/design-parking-lot.md) — multi-level / multi-spot-size, ticket lifecycle, payment, near-fit allocation strategy _(2026-05-02, easy)_
83. [Design Task Management System](case-studies/management/design-task-management-system.md) — task entities, priorities, assignment, dependencies, observer for notifications _(2026-05-02, easy)_
84. [Design Inventory Management System](case-studies/management/design-inventory-management-system.md) — SKU, stock movements, reorder triggers, multi-warehouse, audit log _(2026-05-02, medium)_
85. [Design Library Management System](case-studies/management/design-library-management-system.md) — book / member / loan, due-date logic, reservations, fines, search _(2026-05-02, medium)_
86. [Design Restaurant Management System](case-studies/management/design-restaurant-management-system.md) — menu, table, order, kitchen, reservation, billing _(2026-05-02, hard)_

### 12.E Social & Content Platforms

87. [Design Stack Overflow](case-studies/social-content/design-stack-overflow.md) — question/answer/comment hierarchy, voting, reputation, tags, search _(2026-05-02, medium)_
88. [Design a Social Network](case-studies/social-content/design-social-network.md) — user graph, post timeline, follow / friendship semantics, feed generation _(2026-05-02, medium)_
89. [Design Learning Platform](case-studies/social-content/design-learning-platform.md) — course / lesson / progress, quizzes, certificates, instructor + student roles _(2026-05-02, medium)_
90. [Design Cricinfo](case-studies/social-content/design-cricinfo.md) — match / innings / over / ball entities, live scoreboard, statistics, commentary stream _(2026-05-02, hard)_
91. [Design LinkedIn](case-studies/social-content/design-linkedin.md) — profile, connections, jobs, messages, feed, recommendations _(2026-05-02, hard)_
92. [Design Spotify](case-studies/social-content/design-spotify.md) — track / album / artist / playlist, library, search, recommendation, streaming session _(2026-05-02, hard)_

### 12.F Communication & Messaging

93. [Design Notification System (LLD)](case-studies/communication/design-notification-system-lld.md) — channels, templates, preferences, delivery state machine, observer + strategy _(2026-05-02, easy)_
94. [Design Pub/Sub System (LLD)](case-studies/communication/design-pub-sub-system-lld.md) — topic, subscription, dispatcher, in-process broker, thread-safe delivery _(2026-05-02, medium)_
95. [Design Chat Application (LLD)](case-studies/communication/design-chat-application-lld.md) — conversation, participant, message, delivery state, group chat semantics _(2026-05-02, medium)_

### 12.G Financial & Payment Systems

96. [Design Splitwise](case-studies/financial/design-splitwise.md) — group, expense, share strategies (equal/exact/percentage), simplify-debts algorithm, balance ledger _(2026-05-02, medium)_
97. [Design Payment Gateway](case-studies/financial/design-payment-gateway.md) — payment intent state machine, processor adapters (Stripe/PayPal/etc.), idempotency keys, refunds _(2026-05-02, medium)_
98. [Design Online Stock Exchange](case-studies/financial/design-online-stock-exchange.md) — order book, matching engine, price-time priority, market vs limit orders _(2026-05-02, hard)_

### 12.H E-Commerce & Booking Systems

99. [Design Amazon Locker](case-studies/e-commerce/design-amazon-locker.md) — locker location, size buckets, reservation, OTP / pickup state machine _(2026-05-02, medium)_
100. [Design Shopping Cart](case-studies/e-commerce/design-shopping-cart.md) — cart, line items, discount strategies, persistence across sessions, checkout handoff _(2026-05-02, medium)_
101. [Design Amazon (Catalog + Order)](case-studies/e-commerce/design-amazon.md) — product, catalog, cart, order, payment, fulfillment — the OOD survey, not the HLD _(2026-05-02, hard)_
102. [Design Movie Booking System](case-studies/e-commerce/design-movie-booking-system.md) — show / screen / seat, seat-hold concurrency, payment integration, refund flow _(2026-05-02, hard)_
103. [Design Car Rental System](case-studies/e-commerce/design-car-rental-system.md) — vehicle inventory, location, reservation, return condition, pricing strategy _(2026-05-02, hard)_
104. [Design Meeting Scheduler](case-studies/e-commerce/design-meeting-scheduler.md) — calendar, free-busy, conflict detection, recurrence, time-zone handling _(2026-05-02, hard)_
105. [Design Online Auction System](case-studies/e-commerce/design-online-auction-system.md) — auction lifecycle, bidder, bid history, sniping protection, observer for live updates _(2026-05-02, hard)_
106. [Design Online Food Delivery Service](case-studies/e-commerce/design-online-food-delivery-service.md) — restaurant, menu, order, delivery agent, dispatch strategy, order state machine _(2026-05-02, hard)_
107. [Design Ride-Hailing Service](case-studies/e-commerce/design-ride-hailing-service.md) — rider / driver, trip state machine, dispatch matching, surge, ratings _(2026-05-02, hard)_

### 12.I Developer Tools & Infrastructure

108. [Design URL Shortener (LLD)](case-studies/developer-tools/design-url-shortener-lld.md) — short-code generation strategies, repository, redirect controller, custom alias, hit tracking _(2026-05-02, medium)_
109. [Design Logging Framework](case-studies/developer-tools/design-logging-framework.md) — logger / appender / formatter, level hierarchy, async appender, MDC, configuration _(2026-05-02, medium)_
110. [Design Rate Limiter (LLD)](case-studies/developer-tools/design-rate-limiter-lld.md) — algorithm strategies (token bucket / sliding window), key resolver, in-process vs distributed, integration as middleware _(2026-05-02, medium)_
111. [Design In-Memory File System](case-studies/developer-tools/design-in-memory-file-system.md) — directory + file composite tree, path resolver, permissions, locking _(2026-05-02, hard)_
112. [Design Version Control System](case-studies/developer-tools/design-version-control-system.md) — commit / branch / object store, content-addressed storage, diff/merge, refs _(2026-05-02, hard)_
113. [Design Task Scheduler](case-studies/developer-tools/design-task-scheduler.md) — job, trigger (one-shot / cron / interval), executor pool, retry, persistence _(2026-05-02, hard)_

---

## Quick Reference by Topic

All entries link to the planned doc paths above. Items marked _(2026-05-02)_ have not been written yet.

### Foundations

- [What Is Low-Level Design?](foundations/what-is-lld.md)
- [LLD vs HLD — Where the Boundary Sits](foundations/lld-vs-hld.md)
- [Types of LLD Interviews](foundations/types-of-lld-interviews.md)

### OOP Fundamentals

- [Classes and Objects](oop-fundamentals/classes-and-objects.md)
- [Enums](oop-fundamentals/enums.md)
- [Interfaces](oop-fundamentals/interfaces.md)
- [Encapsulation](oop-fundamentals/encapsulation.md)
- [Abstraction](oop-fundamentals/abstraction.md)
- [Inheritance](oop-fundamentals/inheritance.md)
- [Polymorphism](oop-fundamentals/polymorphism.md)

### Class Relationships

- [Association](class-relationships/association.md)
- [Aggregation](class-relationships/aggregation.md)
- [Composition](class-relationships/composition.md)
- [Dependency](class-relationships/dependency.md)
- [Realization](class-relationships/realization.md)

### Design Principles

- [DRY](design-principles/dry-principle.md)
- [KISS](design-principles/kiss-principle.md)
- [YAGNI](design-principles/yagni-principle.md)
- [Law of Demeter](design-principles/law-of-demeter.md)
- [Separation of Concerns](design-principles/separation-of-concerns.md)
- [Coupling and Cohesion](design-principles/coupling-and-cohesion.md)
- [Composing Objects Principle](design-principles/composing-objects-principle.md)
- [Code Smells and Refactoring Triggers](design-principles/code-smells-and-refactoring-triggers.md)
- [Anti-Patterns in OO Design](design-principles/anti-patterns-in-oo-design.md)

### SOLID

- [Single Responsibility (SRP)](solid/single-responsibility-principle.md)
- [Open/Closed (OCP)](solid/open-closed-principle.md)
- [Liskov Substitution (LSP)](solid/liskov-substitution-principle.md)
- [Interface Segregation (ISP)](solid/interface-segregation-principle.md)
- [Dependency Inversion (DIP)](solid/dependency-inversion-principle.md)

### UML

- [Class Diagram](uml/class-diagram.md)
- [Use Case Diagram](uml/use-case-diagram.md)
- [Sequence Diagram](uml/sequence-diagram.md)
- [Activity Diagram](uml/activity-diagram.md)
- [State Machine Diagram](uml/state-machine-diagram.md)

### Creational Patterns

- [Singleton](design-patterns/creational/singleton.md)
- [Builder](design-patterns/creational/builder.md)
- [Factory Method](design-patterns/creational/factory-method.md)
- [Abstract Factory](design-patterns/creational/abstract-factory.md)
- [Prototype](design-patterns/creational/prototype.md)

### Structural Patterns

- [Adapter](design-patterns/structural/adapter.md)
- [Facade](design-patterns/structural/facade.md)
- [Decorator](design-patterns/structural/decorator.md)
- [Composite](design-patterns/structural/composite.md)
- [Proxy](design-patterns/structural/proxy.md)
- [Bridge](design-patterns/structural/bridge.md)
- [Flyweight](design-patterns/structural/flyweight.md)

### Behavioral Patterns

- [Strategy](design-patterns/behavioral/strategy.md)
- [Iterator](design-patterns/behavioral/iterator.md)
- [Observer](design-patterns/behavioral/observer.md)
- [Command](design-patterns/behavioral/command.md)
- [State](design-patterns/behavioral/state.md)
- [Template Method](design-patterns/behavioral/template-method.md)
- [Chain of Responsibility](design-patterns/behavioral/chain-of-responsibility.md)
- [Visitor](design-patterns/behavioral/visitor.md)
- [Mediator](design-patterns/behavioral/mediator.md)
- [Memento](design-patterns/behavioral/memento.md)

### Additional Patterns

- [Null Object](design-patterns/additional/null-object-pattern.md)
- [Repository](design-patterns/additional/repository-pattern.md)
- [MVC](design-patterns/additional/mvc-pattern.md)
- [Dependency Injection](design-patterns/additional/dependency-injection-pattern.md)
- [Specification](design-patterns/additional/specification-pattern.md)
- [Game Loop](design-patterns/additional/game-loop-pattern.md)
- [Thread Pool](design-patterns/additional/thread-pool-pattern.md)
- [Producer-Consumer](design-patterns/additional/producer-consumer-pattern.md)

### Interview Method

- [Approach OOD Interviews](interview-method/approach-ood-interviews.md)
- [Approach Machine-Coding Interviews](interview-method/approach-machine-coding-interviews.md)
- [Identify Entities and Model Relationships](interview-method/identify-entities-and-model-relationships.md)
- [Clean Code in an Interview](interview-method/write-clean-code-in-interview.md)
- [Choose Design Patterns Without Forcing Them](interview-method/choose-design-patterns.md)
- [Handle Concurrency Scenarios](interview-method/handle-concurrency-scenarios.md)
- [Test Doubles — Mocks, Stubs, Fakes, Spies](interview-method/test-doubles-in-ood.md)

### Design Problems — Games

- [Tic-Tac-Toe](case-studies/games/design-tic-tac-toe.md) _(easy)_
- [Snake and Ladder](case-studies/games/design-snake-and-ladder.md) _(easy)_
- [Minesweeper](case-studies/games/design-minesweeper.md) _(medium)_
- [Chess](case-studies/games/design-chess.md) _(hard)_

### Design Problems — Data Structures & Search

- [LRU Cache](case-studies/data-structures/design-lru-cache.md) _(easy)_
- [Bloom Filter](case-studies/data-structures/design-bloom-filter.md) _(easy)_
- [Search Autocomplete](case-studies/data-structures/design-search-autocomplete.md) _(easy)_
- [Simple Search Engine](case-studies/data-structures/design-simple-search-engine.md) _(medium)_

### Design Problems — State Machines

- [ATM](case-studies/state-machines/design-atm.md) _(medium)_
- [Vending Machine](case-studies/state-machines/design-vending-machine.md) _(medium)_
- [Elevator System](case-studies/state-machines/design-elevator-system.md) _(medium)_
- [Traffic Control System](case-studies/state-machines/design-traffic-control-system.md) _(medium)_
- [Coffee Vending Machine](case-studies/state-machines/design-coffee-vending-machine.md) _(hard)_

### Design Problems — Management

- [Parking Lot](case-studies/management/design-parking-lot.md) _(easy)_
- [Task Management System](case-studies/management/design-task-management-system.md) _(easy)_
- [Inventory Management System](case-studies/management/design-inventory-management-system.md) _(medium)_
- [Library Management System](case-studies/management/design-library-management-system.md) _(medium)_
- [Restaurant Management System](case-studies/management/design-restaurant-management-system.md) _(hard)_

### Design Problems — Social & Content

- [Stack Overflow](case-studies/social-content/design-stack-overflow.md) _(medium)_
- [Social Network](case-studies/social-content/design-social-network.md) _(medium)_
- [Learning Platform](case-studies/social-content/design-learning-platform.md) _(medium)_
- [Cricinfo](case-studies/social-content/design-cricinfo.md) _(hard)_
- [LinkedIn](case-studies/social-content/design-linkedin.md) _(hard)_
- [Spotify](case-studies/social-content/design-spotify.md) _(hard)_

### Design Problems — Communication

- [Notification System (LLD)](case-studies/communication/design-notification-system-lld.md) _(easy)_
- [Pub/Sub System (LLD)](case-studies/communication/design-pub-sub-system-lld.md) _(medium)_
- [Chat Application (LLD)](case-studies/communication/design-chat-application-lld.md) _(medium)_

### Design Problems — Financial

- [Splitwise](case-studies/financial/design-splitwise.md) _(medium)_
- [Payment Gateway](case-studies/financial/design-payment-gateway.md) _(medium)_
- [Online Stock Exchange](case-studies/financial/design-online-stock-exchange.md) _(hard)_

### Design Problems — E-Commerce & Booking

- [Amazon Locker](case-studies/e-commerce/design-amazon-locker.md) _(medium)_
- [Shopping Cart](case-studies/e-commerce/design-shopping-cart.md) _(medium)_
- [Amazon (Catalog + Order)](case-studies/e-commerce/design-amazon.md) _(hard)_
- [Movie Booking System](case-studies/e-commerce/design-movie-booking-system.md) _(hard)_
- [Car Rental System](case-studies/e-commerce/design-car-rental-system.md) _(hard)_
- [Meeting Scheduler](case-studies/e-commerce/design-meeting-scheduler.md) _(hard)_
- [Online Auction System](case-studies/e-commerce/design-online-auction-system.md) _(hard)_
- [Online Food Delivery Service](case-studies/e-commerce/design-online-food-delivery-service.md) _(hard)_
- [Ride-Hailing Service](case-studies/e-commerce/design-ride-hailing-service.md) _(hard)_

### Design Problems — Developer Tools & Infrastructure

- [URL Shortener (LLD)](case-studies/developer-tools/design-url-shortener-lld.md) _(medium)_
- [Logging Framework](case-studies/developer-tools/design-logging-framework.md) _(medium)_
- [Rate Limiter (LLD)](case-studies/developer-tools/design-rate-limiter-lld.md) _(medium)_
- [In-Memory File System](case-studies/developer-tools/design-in-memory-file-system.md) _(hard)_
- [Version Control System](case-studies/developer-tools/design-version-control-system.md) _(hard)_
- [Task Scheduler](case-studies/developer-tools/design-task-scheduler.md) _(hard)_

---

## Cross-Path Reading

- [System Design Path (HLD)](../system-design/INDEX.md) — what comes before LLD: capacity, sharding, replication, infrastructure topology. Some Tier 12 problems here have an HLD twin in `system-design/case-studies/`.
- [Java Path](../java/INDEX.md) — JVM-flavored realizations of the patterns; concurrency primitives; Spring DI/AOP.
- [TypeScript Path](../typescript/INDEX.md) — TS realizations; structural typing's effect on patterns; class-based OO in a duck-typed world.
- [Database Path](../database/INDEX.md) — repository pattern, ORM / JPA realities, query optimization underneath the domain model.
- [Networking Path](../networking/INDEX.md) — protocol-level detail behind chat / pub-sub / notification LLDs.
- [Kubernetes Path](../kubernetes/INDEX.md) — platform context for thread pools, schedulers, and rate limiters that scale beyond one process.
