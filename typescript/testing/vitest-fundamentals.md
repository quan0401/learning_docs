---
title: "Vitest Fundamentals"
date: 2026-04-23
updated: 2026-04-24
tags: [typescript, testing, vitest, coverage]
---

# Vitest Fundamentals

**Date:** 2026-04-23 | **Updated:** 2026-04-24
**Tags:** `typescript` `testing` `vitest` `coverage`

## Table of Contents

- [Summary](#summary)
- [Why Vitest](#why-vitest)
- [Setup from Zero](#setup-from-zero)
- [Test Structure](#test-structure)
- [Matchers](#matchers)
- [Lifecycle Hooks](#lifecycle-hooks)
- [Test Isolation](#test-isolation)
- [Coverage](#coverage)
- [Running Tests](#running-tests)
- [JUnit 5 to Vitest Mapping](#junit-5-to-vitest-mapping)
- [Related](#related)
- [References](#references)

---

## Summary

You know JUnit 5 inside out. You have `@BeforeAll`, `@ParameterizedTest`, Mockito, and JaCoCo wired into every Spring Boot project. Now you are staring at a bare Express + Prisma codebase with zero tests. This doc takes you from `npm install` to a working test suite with coverage enforcement, mapping every JUnit concept you already know to its Vitest equivalent.

Vitest is the modern default test runner for TypeScript projects. It is Vite-powered, understands ESM and TypeScript natively (no `ts-jest` config gymnastics), ships with a Jest-compatible API, and includes built-in coverage. If you are starting fresh in the TS ecosystem, this is the tool to learn. For the Spring/JUnit side of the same testing ladder, pair this with [Spring Boot Testing Fundamentals](../../java/testing/spring-boot-test-basics.md).

**Why this matters for architecture:** Clean [hexagonal architecture](../patterns/hexagonal-architecture.md) with port interfaces makes every layer independently testable. The patterns in this doc assume that separation. If your Express app mixes HTTP handling with business logic in one file, fix the architecture first — tests will fight you otherwise.

---

## Why Vitest

### The Problem with Jest in TypeScript Projects

Jest was built for JavaScript. Running it with TypeScript requires either:

- **`ts-jest`** — a transformer that compiles `.ts` files before Jest sees them. Configuration is fragile, compilation is slow, and ESM support is experimental.
- **SWC/esbuild transformers** — faster, but still bolt-on. You maintain a parallel compilation pipeline that does not match your production `tsc` output.

Neither path gives you native ESM. Both require you to maintain test-specific compilation config alongside your [tsconfig.json](../tooling/tsconfig-demystified.md).

### What Vitest Does Differently

Vitest uses Vite's dev server pipeline under the hood. Vite already understands TypeScript and ESM natively (via esbuild for transforms). Vitest inherits this:

- **Zero TypeScript config** — it reads your existing `tsconfig.json` for paths and settings. No separate transformer setup.
- **Native ESM** — `import`/`export` works without flags or hacks.
- **Watch mode with HMR** — only re-runs tests affected by your change, using Vite's module graph. JUnit has no equivalent; the closest is IntelliJ's continuous testing, but it still recompiles entire modules.
- **Jest-compatible API** — `describe`, `it`, `expect`, `vi.fn()`, `vi.mock()`. If you read a Jest tutorial, the syntax works in Vitest unchanged.
- **Built-in coverage** — no separate `jest --coverage` plugin dance.

### Comparison Table

| Feature | Vitest | Jest | Mocha + Chai |
|---|---|---|---|
| TypeScript support | Native (esbuild) | Requires `ts-jest` or SWC | Requires `ts-mocha` + config |
| ESM support | Native | Experimental, flag-gated | Requires `--loader` flags |
| Watch mode | HMR-based, instant | File-watcher, re-runs suites | Requires `--watch` flag, slower |
| API style | Jest-compatible | Jest | BDD (`describe`/`it`) + Chai assertions |
| Mocking | Built-in `vi` | Built-in `jest` | Requires Sinon |
| Coverage | Built-in (`@vitest/coverage-v8`) | Built-in (`--coverage`) | Requires `nyc` / `c8` |
| Snapshot testing | Built-in | Built-in | Requires plugin |
| Config format | `vitest.config.ts` (Vite) | `jest.config.ts` | `.mocharc.yml` + separate assertion lib |
| Speed (cold start) | Fast (esbuild) | Moderate | Fast (no transform by default) |
| Community momentum (2026) | Growing fast, default for new TS projects | Mature, still dominant in legacy | Declining |

**Bottom line:** For a new TypeScript project in 2026, Vitest is the default choice. Jest remains fine for existing projects with working config. Mocha is legacy unless you have a specific reason.

---

## Setup from Zero

### Scenario: Express + Prisma Project

You have this structure:

```
my-api/
├── src/
│   ├── app.ts              # Express app setup
│   ├── server.ts           # HTTP server entry point
│   ├── routes/
│   │   └── user.routes.ts
│   ├── services/
│   │   └── user.service.ts
│   ├── repositories/
│   │   └── user.repository.ts
│   └── types/
│       └── user.ts
├── prisma/
│   └── schema.prisma
├── tsconfig.json
└── package.json
```

### Step 1 — Install

```bash
npm install -D vitest @vitest/coverage-v8
```

That is it. Two packages. Compare this to the Jest equivalent:

```bash
# Jest equivalent (do NOT do this — shown for comparison)
npm install -D jest ts-jest @types/jest
npx ts-jest config:init  # generates jest.config.ts with transform settings
```

### Step 2 — Create `vitest.config.ts`

```typescript
// vitest.config.ts
import { defineConfig } from "vitest/config";
import path from "path";

export default defineConfig({
  test: {
    // Use 'forks' pool for better isolation (default is 'forks' in Vitest 2+)
    pool: "forks",

    // Global test timeout (in ms)
    testTimeout: 10_000,

    // Coverage configuration
    coverage: {
      provider: "v8",
      reporter: ["text", "html", "lcov"],
      include: ["src/**/*.ts"],
      exclude: ["src/server.ts", "src/**/*.d.ts", "src/types/**"],
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 80,
        statements: 80,
      },
    },

    // Resolve path aliases from tsconfig
    alias: {
      "@/": path.resolve(__dirname, "src") + "/",
    },
  },
});
```

**tsconfig integration:** Vitest uses esbuild for TypeScript compilation, so it does not type-check your tests (just like `tsc --noEmit` is a separate step). It does respect `paths` from your [tsconfig.json](../tooling/tsconfig-demystified.md), but you must mirror them in the `alias` config above. If you use `tsconfig-paths` or Vite's `resolve.alias`, those also work.

### Step 3 — Add Scripts to `package.json`

```json
{
  "scripts": {
    "test": "vitest",
    "test:run": "vitest run",
    "test:coverage": "vitest run --coverage",
    "test:ui": "vitest --ui"
  }
}
```

| Script | Purpose | JUnit parallel |
|---|---|---|
| `npm test` | Watch mode — reruns on file change | IntelliJ continuous testing |
| `npm run test:run` | Single run, then exit | `mvn test` |
| `npm run test:coverage` | Single run + coverage report | `mvn test jacoco:report` |
| `npm run test:ui` | Browser-based test dashboard | IntelliJ test runner UI |

### Step 4 — Create Your First Test

```
my-api/
├── src/
│   └── services/
│       ├── user.service.ts
│       └── __tests__/
│           └── user.service.test.ts   <-- test file
```

Vitest discovers test files by default matching:

- `**/*.test.ts`
- `**/*.spec.ts`
- `**/__tests__/**/*.ts`

No configuration needed — these patterns work out of the box.

### Minimal vs Full Config

For many projects, you need no config file at all. Vitest works with zero config:

```bash
# This works with no vitest.config.ts
npx vitest
```

Add a config file when you need: path aliases, coverage thresholds, custom pool settings, or global setup/teardown scripts.

---

## Test Structure

### Basic Shape

```typescript
// src/services/__tests__/user.service.test.ts
import { describe, it, expect } from "vitest";
import { UserService } from "../user.service";

describe("UserService", () => {
  describe("createUser", () => {
    it("should create a user with a hashed password", () => {
      // Arrange
      const service = new UserService(mockRepo);
      const input = { email: "test@example.com", password: "secret123" };

      // Act
      const user = service.createUser(input);

      // Assert
      expect(user.email).toBe("test@example.com");
      expect(user.password).not.toBe("secret123");
    });

    it("should throw when email is already taken", () => {
      // Arrange
      const service = new UserService(mockRepoWithExistingUser);

      // Act & Assert
      expect(() => service.createUser({ email: "taken@example.com", password: "x" }))
        .toThrow("Email already exists");
    });
  });
});
```

**AAA pattern (Arrange-Act-Assert)** — the same pattern you use in JUnit. Vitest does not enforce it, but every test should follow it. A test that mixes setup, execution, and verification in random order is unreadable in any language.

### `describe` for Grouping

Nest `describe` blocks to create a hierarchy. This maps directly to JUnit 5's `@Nested`:

```java
// JUnit 5
@Nested
@DisplayName("createUser")
class CreateUser {
    @Test
    @DisplayName("should create a user with a hashed password")
    void shouldCreateUser() { ... }
}
```

```typescript
// Vitest equivalent
describe("UserService", () => {
  describe("createUser", () => {
    it("should create a user with a hashed password", () => { ... });
  });
});
```

### `it` vs `test`

They are identical. `it` reads better in BDD style ("it should..."), `test` reads better for direct statements ("test that..."). Pick one and be consistent.

### Parameterized Tests with `it.each`

JUnit 5 has `@ParameterizedTest` with `@ValueSource`, `@CsvSource`, `@MethodSource`. Vitest has `it.each`:

```java
// JUnit 5
@ParameterizedTest
@CsvSource({
    "admin, true",
    "editor, true",
    "viewer, false"
})
void shouldCheckEditPermission(String role, boolean expected) {
    assertEquals(expected, service.canEdit(role));
}
```

```typescript
// Vitest equivalent
it.each([
  { role: "admin", expected: true },
  { role: "editor", expected: true },
  { role: "viewer", expected: false },
])("should return $expected for role $role", ({ role, expected }) => {
  expect(service.canEdit(role)).toBe(expected);
});
```

You can also use a tagged template literal for tabular data:

```typescript
it.each`
  role        | expected
  ${"admin"}  | ${true}
  ${"editor"} | ${true}
  ${"viewer"} | ${false}
`("should return $expected for role $role", ({ role, expected }) => {
  expect(service.canEdit(role)).toBe(expected);
});
```

### Test Modifiers

| Vitest | JUnit 5 | Purpose |
|---|---|---|
| `it.todo("description")` | No direct equivalent | Placeholder for a planned test |
| `it.skip(...)` | `@Disabled` | Skip a test temporarily |
| `it.only(...)` | No direct equivalent (IDE-based) | Run only this test (for debugging) |
| `describe.skip(...)` | `@Disabled` on class | Skip an entire group |
| `describe.only(...)` | — | Run only this group |

**Warning:** Never commit `it.only` or `describe.only`. They silently skip every other test in the file. Use them for local debugging only.

---

## Matchers

### Core Matchers You Will Use Daily

```typescript
// Primitive equality — uses Object.is()
expect(1 + 1).toBe(2);
expect(result).toBe(true);

// Deep equality — recursively compares objects
expect(user).toEqual({ id: 1, name: "Alice" });

// Strict deep equality — also checks undefined properties and object prototypes
expect(user).toStrictEqual({ id: 1, name: "Alice" });
// toStrictEqual({ a: 1 }) FAILS for { a: 1, b: undefined } — toEqual passes!

// Partial match — object contains at least these properties
expect(response.body).toMatchObject({ status: "ok" });

// Property existence
expect(user).toHaveProperty("email");
expect(user).toHaveProperty("address.city", "Tokyo");

// Truthiness
expect(value).toBeTruthy();
expect(value).toBeFalsy();
expect(value).toBeNull();
expect(value).toBeUndefined();
expect(value).toBeDefined();

// Numbers
expect(result).toBeGreaterThan(10);
expect(result).toBeCloseTo(0.3, 5); // floating point comparison

// Strings
expect(message).toContain("error");
expect(message).toMatch(/^Error: .+/);

// Arrays
expect(list).toContain("alice");
expect(list).toHaveLength(3);
expect(list).toEqual(expect.arrayContaining(["alice", "bob"]));

// Exceptions
expect(() => divide(1, 0)).toThrow();
expect(() => divide(1, 0)).toThrow("Division by zero");
expect(() => divide(1, 0)).toThrow(ArithmeticError);

// Async exceptions
await expect(asyncFn()).rejects.toThrow("Not found");
await expect(asyncFn()).resolves.toEqual({ ok: true });
```

### The `toBe` vs `toEqual` vs `toStrictEqual` Trap

This catches every Java developer:

```typescript
// toBe — reference equality (Object.is)
expect({ a: 1 }).toBe({ a: 1 });           // FAILS — different objects
expect(1).toBe(1);                           // PASSES — same primitive

// toEqual — deep value equality (ignores undefined properties)
expect({ a: 1 }).toEqual({ a: 1 });         // PASSES
expect({ a: 1, b: undefined }).toEqual({ a: 1 }); // PASSES (!)

// toStrictEqual — deep equality with strict checks
expect({ a: 1, b: undefined }).toStrictEqual({ a: 1 }); // FAILS
```

**Java mapping:** `toBe` is `==` for objects (reference check). `toEqual` is like a lenient `equals()`. `toStrictEqual` is the strict `equals()` you'd write by hand. **Default to `toStrictEqual` for objects** — it catches bugs that `toEqual` silently misses.

### Custom Matchers

```typescript
// In a setup file or at the top of your test
import { expect } from "vitest";

expect.extend({
  toBeValidEmail(received: string) {
    const pass = /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(received);
    return {
      pass,
      message: () => `expected ${received} ${pass ? "not " : ""}to be a valid email`,
    };
  },
});

// Usage
expect("alice@example.com").toBeValidEmail();
```

**AssertJ comparison:** In Java, you chain assertions fluently: `assertThat(user).hasFieldOrProperty("email").isNotNull()`. Vitest uses a different style — one `expect` per assertion, each with a specific matcher. The expressiveness is similar; the syntax differs.

---

## Lifecycle Hooks

### The Four Hooks

```typescript
import { describe, it, expect, beforeAll, afterAll, beforeEach, afterEach } from "vitest";

describe("UserRepository", () => {
  let db: DatabaseConnection;

  beforeAll(async () => {
    // Run ONCE before all tests in this describe block
    // Java equivalent: @BeforeAll (must be static in JUnit)
    db = await createTestDatabase();
  });

  afterAll(async () => {
    // Run ONCE after all tests in this describe block
    // Java equivalent: @AfterAll
    await db.close();
  });

  beforeEach(async () => {
    // Run before EACH test
    // Java equivalent: @BeforeEach
    await db.beginTransaction();
  });

  afterEach(async () => {
    // Run after EACH test
    // Java equivalent: @AfterEach
    await db.rollbackTransaction(); // undo all changes — each test starts clean
  });

  it("should insert a user", async () => {
    const user = await db.users.create({ name: "Alice" });
    expect(user.id).toBeDefined();
  });

  it("should not see the user from the previous test", async () => {
    const users = await db.users.findAll();
    expect(users).toHaveLength(0); // transaction was rolled back
  });
});
```

### Scoping

Hooks are scoped to their `describe` block. Nested `describe` blocks inherit outer hooks:

```typescript
describe("outer", () => {
  beforeEach(() => console.log("outer beforeEach"));

  describe("inner", () => {
    beforeEach(() => console.log("inner beforeEach"));

    it("runs both", () => {
      // Output:
      // outer beforeEach
      // inner beforeEach
    });
  });
});
```

This is identical to JUnit 5's behavior with `@Nested` classes — outer `@BeforeEach` runs before inner `@BeforeEach`.

### Key Difference from JUnit

In JUnit 5, `@BeforeAll` methods must be `static` (or the class must use `@TestInstance(Lifecycle.PER_CLASS)`). In Vitest, `beforeAll` is just a function — it closes over any variables in scope. No static method gymnastics.

---

## Test Isolation

### How Vitest Isolates Tests

Each **test file** runs in its own worker. This means:

- Module-level state in `user.service.ts` is fresh for each test file.
- Global variables do not leak between files.
- One file crashing does not kill other test files.

Within a single file, tests run **sequentially** by default and share the same module scope. This is why `beforeEach` cleanup matters — tests within a file can leak state to each other.

### Pool Options

Vitest offers different isolation strategies via the `--pool` flag:

| Pool | Mechanism | Isolation | Speed | When to use |
|---|---|---|---|---|
| `forks` (default) | Child processes | Strong — separate V8 instances | Moderate | Default for most projects |
| `threads` | Worker threads | Moderate — shared memory possible | Faster | CPU-bound tests, no global state pollution |
| `vmThreads` | VM contexts in threads | Strong isolation, thread speed | Fast | Best of both when it works |

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    pool: "forks", // safest default
  },
});
```

**Java comparison:** JUnit 5 with `junit.jupiter.execution.parallel.enabled = true` runs tests in parallel within the same JVM. The JVM's class loading provides some isolation, but shared static state is a constant source of flaky tests. Vitest's process-per-file model is more aggressive — closer to running each JUnit test class in its own JVM.

### Concurrent Tests Within a File

By default, tests within a file run sequentially. To run them in parallel:

```typescript
describe("independent operations", () => {
  it.concurrent("should fetch user A", async () => { ... });
  it.concurrent("should fetch user B", async () => { ... });
  it.concurrent("should fetch user C", async () => { ... });
});
```

Use `concurrent` only when tests are truly independent — no shared state, no shared database rows, no order dependency.

---

## Coverage

### Setup

The `@vitest/coverage-v8` package uses V8's built-in code coverage instrumentation. This is the same engine Chrome DevTools uses when you click "Coverage" in the browser.

```bash
npm install -D @vitest/coverage-v8
```

### Configuration

```typescript
// vitest.config.ts (coverage section)
export default defineConfig({
  test: {
    coverage: {
      provider: "v8",

      // Output formats
      reporter: [
        "text",          // console summary table
        "html",          // browsable HTML report (open coverage/index.html)
        "lcov",          // for CI tools (Codecov, Coveralls)
      ],

      // What to measure
      include: ["src/**/*.ts"],
      exclude: [
        "src/server.ts",        // entry point — not unit-testable
        "src/**/*.d.ts",        // type declarations
        "src/types/**",         // type-only files
        "src/**/__tests__/**",  // test files themselves
      ],

      // Fail the run if coverage drops below thresholds
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 80,
        statements: 80,
      },
    },
  },
});
```

### Running Coverage

```bash
# Generate coverage report
npx vitest run --coverage

# Output:
# ----------|---------|----------|---------|---------|-------------------
# File      | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s
# ----------|---------|----------|---------|---------|-------------------
# All files |   85.71 |    77.78 |   90.00 |   85.71 |
#  user.service.ts | 100 | 100 | 100 | 100 |
#  user.repository.ts | 71.43 | 55.56 | 80.00 | 71.43 | 23-31,45
# ----------|---------|----------|---------|---------|-------------------
```

When thresholds are configured, `vitest run --coverage` **exits with a non-zero code** if any threshold is not met. This is how you enforce coverage in CI:

```yaml
# .github/workflows/test.yml
- name: Run tests with coverage
  run: npx vitest run --coverage
  # This step FAILS if coverage < 80%
```

### JaCoCo Comparison

| Aspect | Vitest + V8 | JaCoCo |
|---|---|---|
| Instrumentation | V8 engine native | Bytecode instrumentation |
| Config location | `vitest.config.ts` | `pom.xml` or `build.gradle` |
| Report formats | text, html, lcov, json | html, xml, csv |
| Threshold enforcement | `thresholds` in config | `<rules>` in Maven/Gradle |
| Speed overhead | Minimal (V8 native) | Moderate (bytecode rewrite) |
| Branch coverage | V8-level branch tracking | JVM bytecode branch tracking |

---

## Running Tests

### Everyday Commands

```bash
# Watch mode — the default. Reruns on file change.
npx vitest

# Single run — for CI or pre-commit hooks.
npx vitest run

# Run only tests related to a specific source file.
# Vitest traces the import graph to find which test files import user.service.ts.
npx vitest related src/services/user.service.ts

# Filter by test name (substring or regex).
npx vitest -t "should create user"

# Run a specific test file.
npx vitest src/services/__tests__/user.service.test.ts

# Run with coverage.
npx vitest run --coverage

# Run with the browser UI dashboard.
npx vitest --ui
```

### Watch Mode Intelligence

When you save a file, Vitest does not re-run your entire test suite. It uses Vite's module dependency graph to determine which test files import (directly or transitively) the changed module, and re-runs only those.

This is dramatically faster than Jest's file-watcher approach, which at best uses `--changedSince` with git. Vitest knows the actual import graph.

### VSCode Extension

Install the [Vitest VSCode extension](https://marketplace.visualstudio.com/items?itemName=vitest.explorer). It gives you:

- Inline test status (green check / red X) next to each `it()` block.
- Click-to-run individual tests.
- Test Explorer sidebar with tree view.
- Debug individual tests with breakpoints.

This is similar to IntelliJ's built-in JUnit runner — the green/red gutter icons, the test tree, the click-to-debug.

---

## JUnit 5 to Vitest Mapping

The complete mental model translation. Print this table and keep it until the mapping is automatic.

| JUnit 5 | Vitest | Notes |
|---|---|---|
| `@Test` | `it()` / `test()` | Identical purpose |
| `@DisplayName("...")` | `it("description", ...)` | Description is the first argument, not an annotation |
| `@Nested class Foo` | `describe("Foo", () => { })` | Nesting via functions, not classes |
| `@ParameterizedTest` + `@CsvSource` | `it.each([...])` | Array of cases instead of CSV strings |
| `@BeforeAll` | `beforeAll()` | No `static` requirement |
| `@AfterAll` | `afterAll()` | No `static` requirement |
| `@BeforeEach` | `beforeEach()` | Identical semantics |
| `@AfterEach` | `afterEach()` | Identical semantics |
| `@Disabled` | `it.skip()` / `describe.skip()` | Same purpose |
| `Assertions.assertEquals(expected, actual)` | `expect(actual).toBe(expected)` | Argument order is reversed! |
| `Assertions.assertThrows(Ex.class, () -> ...)` | `expect(() => ...).toThrow(...)` | Inline, no class reference needed |
| `Assertions.assertTrue(cond)` | `expect(cond).toBe(true)` | Or `expect(cond).toBeTruthy()` |
| `assertThat(x).isEqualTo(y)` (AssertJ) | `expect(x).toEqual(y)` | Fluent chain vs method call |
| JaCoCo | `@vitest/coverage-v8` | Config in `vitest.config.ts` |
| `@SpringBootTest` | No direct equivalent | See [NestJS testing](../frameworks/nestjs-for-spring-devs.md) for the NestJS equivalent |
| `@MockBean` | `vi.mock()` | Covered in [Mocking & Test Doubles](mocking-and-test-doubles.md) |
| `@DataJpaTest` (test slice) | No framework-level equivalent | Express has no DI container; see [NestJS Advanced Patterns](../frameworks/nestjs-advanced-patterns.md) for `Test.createTestingModule()` |

**Critical difference — argument order:** JUnit puts `expected` first: `assertEquals(expected, actual)`. Vitest puts `actual` first: `expect(actual).toBe(expected)`. This catches every Java developer at least once.

### Full Side-by-Side Example

```java
// JUnit 5
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @InjectMocks
    private UserService userService;

    @BeforeEach
    void setUp() {
        // Mockito resets mocks before each test
    }

    @Test
    @DisplayName("should find user by email")
    void shouldFindUserByEmail() {
        // Arrange
        when(userRepository.findByEmail("alice@example.com"))
            .thenReturn(Optional.of(new User(1L, "alice@example.com")));

        // Act
        User user = userService.findByEmail("alice@example.com");

        // Assert
        assertThat(user.getEmail()).isEqualTo("alice@example.com");
        verify(userRepository).findByEmail("alice@example.com");
    }
}
```

```typescript
// Vitest
import { describe, it, expect, beforeEach, vi } from "vitest";
import { UserService } from "../user.service";
import type { UserRepository } from "../user.repository";

describe("UserService", () => {
  let mockRepo: UserRepository;
  let service: UserService;

  beforeEach(() => {
    // Create a mock that satisfies the UserRepository interface
    mockRepo = {
      findByEmail: vi.fn(),
      create: vi.fn(),
      findAll: vi.fn(),
    };
    service = new UserService(mockRepo);
  });

  it("should find user by email", async () => {
    // Arrange
    vi.mocked(mockRepo.findByEmail).mockResolvedValue({
      id: 1,
      email: "alice@example.com",
    });

    // Act
    const user = await service.findByEmail("alice@example.com");

    // Assert
    expect(user.email).toBe("alice@example.com");
    expect(mockRepo.findByEmail).toHaveBeenCalledWith("alice@example.com");
  });
});
```

Notice: no `@InjectMocks` magic. You construct the service manually and pass the mock repository. This is explicit dependency injection — the same principle behind [hexagonal architecture](../patterns/hexagonal-architecture.md) ports. The service depends on an interface, and in tests you provide a mock implementation.

---

## Related

### Next in this series

- [Mocking & Test Doubles](mocking-and-test-doubles.md) — `vi.fn()`, `vi.mock()`, `vi.spyOn()`, manual mocks, mocking Prisma client, mocking external APIs
- [Integration & API Testing](integration-and-api-testing.md) — Supertest for Express routes, Testcontainers for real databases, contract testing

### Architecture that enables testing

- [Hexagonal Architecture](../patterns/hexagonal-architecture.md) — port interfaces make every layer independently testable without mocks leaking across boundaries
- [NestJS for Spring Boot Developers](../frameworks/nestjs-for-spring-devs.md) — NestJS testing module compared to Spring Boot test slices
- [NestJS Advanced Patterns](../frameworks/nestjs-advanced-patterns.md) — `Test.createTestingModule()` for DI-aware testing in NestJS

### Tooling

- [tsconfig Demystified](../tooling/tsconfig-demystified.md) — TypeScript compiler options that affect test compilation and path resolution
- [Spring Boot Testing Fundamentals](../../java/testing/spring-boot-test-basics.md) — `@SpringBootTest`, test slices, and the Java side of the comparison
- [Web Layer Testing](../../java/testing/web-layer-testing.md) — `WebTestClient` / `MockMvc` patterns parallel to route-level test clients

---

## References

- [Vitest Official Documentation](https://vitest.dev/) — primary reference for all Vitest APIs and configuration
- [Vitest API Reference](https://vitest.dev/api/) — complete list of `expect` matchers, lifecycle hooks, and test utilities
- [Vitest Configuration](https://vitest.dev/config/) — all `vitest.config.ts` options
- [Vitest Coverage Guide](https://vitest.dev/guide/coverage) — setup and configuration for V8 and Istanbul providers
- [`@vitest/coverage-v8` on npm](https://www.npmjs.com/package/@vitest/coverage-v8) — the V8 coverage provider package
- [Vitest VSCode Extension](https://marketplace.visualstudio.com/items?itemName=vitest.explorer) — IDE integration
- [Vite Documentation](https://vitejs.dev/) — understanding the underlying build tool
- [JUnit 5 User Guide](https://junit.org/junit5/docs/current/user-guide/) — for cross-referencing JUnit concepts mentioned in this doc
