# Error Handling Architecture

> Propagate domain errors to HTTP responses without leaking internals.

Every backend developer eventually faces the same question: a service method
discovers that an order does not exist. Does it throw an exception? Return
`null`? Set a status code on some shared object? The answer shapes the entire
reliability story of the application.

Coming from Java, you know checked exceptions force callers to handle failures
explicitly. TypeScript has no equivalent mechanism. Exceptions are untyped,
invisible in function signatures, and break the type safety guarantees that
drew you to TypeScript in the first place. This document builds an error
handling architecture that restores that discipline.

---

## The Problem with Exceptions

### Exceptions Are Untyped

```typescript
try {
  const order = await orderService.findById(orderId);
} catch (e) {
  // e is `unknown` (since TS 4.4)
  // What type is it? OrderNotFound? DatabaseError? TypeError from a bug?
  // The compiler cannot tell you.
}
```

In Java, `throws OrderNotFoundException` is part of the method signature. The
compiler forces you to handle it. In TypeScript, nothing in the signature of
`findById` reveals what it might throw. The caller has to guess, read the source,
or discover failures at runtime.

### Exceptions Are Invisible Control Flow

Exceptions create a hidden second return path from every function. When you call
`placeOrder(dto)`, the function can return an `Order` or it can throw any of a
dozen different errors. The type system tracks only the happy path.

### Exceptions Break Composition

Functional pipelines fall apart when any step might throw:

```typescript
// This looks clean, but any step can explode
const result = await pipe(
  validateInput(dto),
  enrichWithPricing,
  applyDiscounts,
  persistOrder,
);
```

If `applyDiscounts` throws, you lose context about which step failed and why.
The only way to recover is a try/catch around the entire pipeline with
fragile `instanceof` checks inside.

### When Exceptions Are Appropriate

Exceptions should represent **programmer errors** and **infrastructure failures**:

- `TypeError`: dereferencing `null` where it should be impossible
- `RangeError`: index out of bounds due to a logic bug
- Database connection lost mid-transaction
- File system permissions error

These are situations where the caller cannot reasonably recover inline. They
should crash the request (or the process) and be caught by a top-level handler.

---

## The Result Type Pattern

The core idea: make **success and failure both explicit in the return type**.

### A Minimal Implementation

```typescript
type Result<T, E> = 
  | { readonly ok: true; readonly value: T }
  | { readonly ok: false; readonly error: E };

function ok<T>(value: T): Result<T, never> {
  return { ok: true, value };
}

function err<E>(error: E): Result<never, E> {
  return { ok: false, error };
}
```

The `never` type parameter is key. `ok('hello')` returns `Result<string, never>`,
which is assignable to `Result<string, AnyErrorType>`. This lets you compose
results from functions with different error types.

### Using It

```typescript
function findOrder(id: string): Result<Order, OrderNotFoundError> {
  const order = orders.get(id);
  if (!order) {
    return err({ type: 'ORDER_NOT_FOUND', orderId: id });
  }
  return ok(order);
}

// Caller MUST handle both branches — the compiler enforces it
const result = findOrder('abc-123');
if (!result.ok) {
  // result.error is narrowed to OrderNotFoundError
  console.log(`Order ${result.error.orderId} not found`);
  return;
}
// result.value is narrowed to Order
console.log(result.value.total);
```

### Chaining with map and flatMap

Raw `if/else` blocks get verbose when you chain multiple fallible operations.
Add combinators:

```typescript
function map<T, U, E>(
  result: Result<T, E>,
  fn: (value: T) => U,
): Result<U, E> {
  if (!result.ok) return result;
  return ok(fn(result.value));
}

function flatMap<T, U, E1, E2>(
  result: Result<T, E1>,
  fn: (value: T) => Result<U, E2>,
): Result<U, E1 | E2> {
  if (!result.ok) return result;
  return fn(result.value);
}

function mapErr<T, E1, E2>(
  result: Result<T, E1>,
  fn: (error: E1) => E2,
): Result<T, E2> {
  if (result.ok) return result;
  return err(fn(result.error));
}
```

Now a pipeline reads linearly:

```typescript
const pipeline = flatMap(
  findOrder(orderId),
  (order) => flatMap(
    validateStock(order),
    (validated) => applyDiscount(validated, couponCode),
  ),
);
// pipeline: Result<DiscountedOrder, OrderNotFoundError | InsufficientStockError | InvalidCouponError>
```

The **union of all error types** accumulates automatically through `E1 | E2`.
The compiler tracks every possible failure without you maintaining a manual list.

### Comparison: Java Checked Exceptions and Rust Result

| Mechanism | Forces Handling? | Composable? | Type-Safe Errors? |
|-----------|-----------------|-------------|-------------------|
| Java checked exceptions | Yes (`throws` clause) | Awkward (try/catch nesting) | Partially (catch by class) |
| Rust `Result<T, E>` | Yes (must match or `?`) | Yes (`?` operator, combinators) | Yes (enum variants) |
| TS `Result<T, E>` | Yes (union discrimination) | Yes (map/flatMap) | Yes (discriminated unions) |
| TS exceptions | No | No | No (`unknown`) |

TypeScript's `Result` is closest to Rust's. Java's checked exceptions share the
"force the caller to handle it" property but lack the ergonomic chaining.

---

## Domain Errors as Discriminated Unions

### Defining Error Types

Each domain defines its own error union. The `type` field is the discriminant:

```typescript
type OrderError =
  | { readonly type: 'ORDER_NOT_FOUND'; readonly orderId: string }
  | { readonly type: 'INSUFFICIENT_STOCK'; readonly productId: string; readonly requested: number; readonly available: number }
  | { readonly type: 'MINIMUM_NOT_MET'; readonly minimum: number; readonly actual: number };

type PaymentError =
  | { readonly type: 'CARD_DECLINED'; readonly last4: string }
  | { readonly type: 'PAYMENT_TIMEOUT'; readonly gateway: string }
  | { readonly type: 'FRAUD_SUSPECTED'; readonly riskScore: number };
```

### Exhaustive Handling with never

The `switch` + `never` pattern guarantees you handle every variant. If you add a
new error type to the union and forget to handle it, the compiler complains:

```typescript
function describeOrderError(error: OrderError): string {
  switch (error.type) {
    case 'ORDER_NOT_FOUND':
      return `Order ${error.orderId} does not exist`;
    case 'INSUFFICIENT_STOCK':
      return `Product ${error.productId}: requested ${error.requested}, only ${error.available} available`;
    case 'MINIMUM_NOT_MET':
      return `Order minimum is ${error.minimum}, got ${error.actual}`;
    default:
      // If OrderError gains a new variant, this line will not compile
      const _exhaustive: never = error;
      return _exhaustive;
  }
}
```

### Why Not Error Classes?

You could define `class OrderNotFoundError extends Error { ... }`, but:

1. `instanceof` checks break across module boundaries and serialization
2. Classes carry prototype chains that are irrelevant for data-only errors
3. Discriminated unions give you exhaustiveness checking, classes do not

Plain objects with a `type` discriminant are simpler, more reliable, and play
better with TypeScript's structural type system.

---

## Error Mapping Across Layers

A well-layered application has three error boundaries:

```
Domain Layer          Application Layer         HTTP Layer
─────────────        ──────────────────        ──────────
OrderError      →     ApiError             →    HTTP Response
PaymentError    →     ApiError             →    HTTP Response
```

Each layer only knows its own error types. The domain never imports HTTP status
codes. The HTTP layer never sees raw domain errors.

### Domain Layer

```typescript
// domain/order.service.ts
function placeOrder(dto: PlaceOrderDto): Result<Order, OrderError | PaymentError> {
  const stockCheck = validateStock(dto.items);
  if (!stockCheck.ok) return stockCheck;

  const payment = chargePayment(dto.paymentMethod, dto.total);
  if (!payment.ok) return payment;

  return ok(createOrder(dto, payment.value));
}
```

### Application Layer: Domain Errors to API Errors

```typescript
// application/errors.ts
type ApiError = {
  readonly code: string;
  readonly message: string;
  readonly status: number;
  readonly details?: Record<string, unknown>;
};

function orderErrorToApiError(error: OrderError): ApiError {
  switch (error.type) {
    case 'ORDER_NOT_FOUND':
      return {
        code: 'ORDER_NOT_FOUND',
        message: `Order ${error.orderId} was not found`,
        status: 404,
      };
    case 'INSUFFICIENT_STOCK':
      return {
        code: 'INSUFFICIENT_STOCK',
        message: `Not enough stock for product ${error.productId}`,
        status: 409,
        details: { requested: error.requested, available: error.available },
      };
    case 'MINIMUM_NOT_MET':
      return {
        code: 'MINIMUM_NOT_MET',
        message: `Order total ${error.actual} is below minimum ${error.minimum}`,
        status: 422,
      };
    default:
      const _exhaustive: never = error;
      return _exhaustive;
  }
}

function paymentErrorToApiError(error: PaymentError): ApiError {
  switch (error.type) {
    case 'CARD_DECLINED':
      return {
        code: 'CARD_DECLINED',
        message: 'Payment was declined',
        status: 402,
      };
    case 'PAYMENT_TIMEOUT':
      return {
        code: 'PAYMENT_TIMEOUT',
        message: 'Payment gateway timed out. Please retry.',
        status: 504,
      };
    case 'FRAUD_SUSPECTED':
      // Do NOT leak the risk score to the client
      return {
        code: 'PAYMENT_REJECTED',
        message: 'Payment could not be processed',
        status: 403,
      };
    default:
      const _exhaustive: never = error;
      return _exhaustive;
  }
}
```

Notice the `FRAUD_SUSPECTED` case: the API error deliberately hides the internal
reason. This is the "without leaking internals" part. The domain error carries
a `riskScore` for logging and auditing; the HTTP response says only "rejected."

### Combining Mappers

```typescript
function domainErrorToApiError(error: OrderError | PaymentError): ApiError {
  switch (error.type) {
    case 'ORDER_NOT_FOUND':
    case 'INSUFFICIENT_STOCK':
    case 'MINIMUM_NOT_MET':
      return orderErrorToApiError(error);
    case 'CARD_DECLINED':
    case 'PAYMENT_TIMEOUT':
    case 'FRAUD_SUSPECTED':
      return paymentErrorToApiError(error);
    default:
      const _exhaustive: never = error;
      return _exhaustive;
  }
}
```

### Controller / Route Handler

```typescript
// adapters/http/order.controller.ts
router.post('/orders', async (req, res, next) => {
  const result = await placeOrder(req.body);

  if (!result.ok) {
    const apiError = domainErrorToApiError(result.error);
    return res.status(apiError.status).json({
      code: apiError.code,
      message: apiError.message,
      details: apiError.details,
    });
  }

  return res.status(201).json(result.value);
});
```

---

## Express Error Middleware

### Centralized Error Handler

Express recognizes a four-parameter middleware as an error handler. This is the
safety net for anything that slips through the Result-based flow:

```typescript
// adapters/http/error-handler.ts
import { ErrorRequestHandler } from 'express';

interface ApiErrorWithStatus extends Error {
  status?: number;
  code?: string;
  details?: Record<string, unknown>;
}

const errorHandler: ErrorRequestHandler = (err, _req, res, _next) => {
  // Log the full error for operators
  console.error('[unhandled]', {
    message: err.message,
    stack: err.stack,
    code: err.code,
  });

  // If it is a known ApiError shape, use its status
  if (isApiError(err)) {
    return res.status(err.status).json({
      code: err.code,
      message: err.message,
      details: err.details,
    });
  }

  // Unknown errors get a generic 500 — never leak the real message
  return res.status(500).json({
    code: 'INTERNAL_ERROR',
    message: 'An unexpected error occurred',
  });
};

function isApiError(err: unknown): err is ApiErrorWithStatus {
  return (
    err instanceof Error &&
    typeof (err as ApiErrorWithStatus).status === 'number'
  );
}

export { errorHandler };
```

Register it **last**, after all routes:

```typescript
app.use('/api/orders', orderRouter);
app.use('/api/payments', paymentRouter);
// ... other routes

app.use(errorHandler); // Must be after all route registrations
```

### Async Error Forwarding

Express does not automatically catch promise rejections in route handlers.
Either wrap every handler or use a helper:

```typescript
function asyncHandler(
  fn: (req: Request, res: Response, next: NextFunction) => Promise<void>,
): RequestHandler {
  return (req, res, next) => {
    fn(req, res, next).catch(next);
  };
}

// Usage
router.post('/orders', asyncHandler(async (req, res) => {
  // If this throws, the error handler catches it
  const result = await placeOrder(req.body);
  // ...
}));
```

### Global Process-Level Handlers

For truly unexpected failures (unhandled promise rejections, uncaught
exceptions), register process-level handlers. These are last-resort safety nets:

```typescript
process.on('uncaughtException', (error: Error) => {
  console.error('[FATAL] Uncaught exception:', error);
  // Flush logs, close connections, then exit
  process.exit(1);
});

process.on('unhandledRejection', (reason: unknown) => {
  console.error('[FATAL] Unhandled rejection:', reason);
  // In production, treat this as fatal
  process.exit(1);
});
```

In production, these handlers should:

1. Log the error with full context (structured logging)
2. Flush any buffered logs
3. Close database connections gracefully
4. Exit the process (let the process manager restart it)

Never attempt to "recover" from an uncaught exception. The process state is
unknown and potentially corrupted.

---

## When to Throw vs Return Result

This is the most practical question. Use this decision guide:

| Situation | Mechanism | Reasoning |
|-----------|-----------|-----------|
| User not found by ID | `Result.err()` | Expected business case. Caller needs to decide: 404? Create? |
| Validation failure | `Result.err()` | Expected input. Return all violations to the caller. |
| Insufficient balance | `Result.err()` | Business rule. The domain layer decides, the caller reacts. |
| Duplicate email on signup | `Result.err()` | Expected race condition. Caller maps to 409 Conflict. |
| Database connection lost | `throw` | Infrastructure failure. Cannot recover inline. |
| Null pointer on required field | `throw` | Programmer error. Indicates a bug. |
| JSON.parse on corrupted data | `throw` | Cannot reason about the data. Abort the operation. |
| Config missing at startup | `throw` | Fail fast. Do not serve traffic without config. |

### The Rule of Thumb

Ask: **"Can the immediate caller do something meaningful with this failure?"**

- **Yes** (show a message, try an alternative, return a different response) → `Result`
- **No** (the world is broken, just crash gracefully) → `throw`

### Validation: Always Result

Validation is the clearest case for `Result`. You want to collect _all_ errors,
not bail on the first one:

```typescript
type ValidationError = {
  readonly type: 'VALIDATION_FAILED';
  readonly violations: ReadonlyArray<{
    readonly field: string;
    readonly message: string;
  }>;
};

function validatePlaceOrder(dto: unknown): Result<PlaceOrderDto, ValidationError> {
  const violations: Array<{ field: string; message: string }> = [];

  if (!isString(dto.customerId)) {
    violations.push({ field: 'customerId', message: 'Required string' });
  }
  if (!isPositiveNumber(dto.total)) {
    violations.push({ field: 'total', message: 'Must be a positive number' });
  }

  if (violations.length > 0) {
    return err({ type: 'VALIDATION_FAILED', violations });
  }

  return ok(dto as PlaceOrderDto);
}
```

Throwing on the first validation error forces the user to fix issues one at a
time and resubmit. Returning all violations at once is both better UX and a
cleaner data flow.

---

## Existing Libraries

### neverthrow

The most popular Result library for TypeScript. Provides `ok()`, `err()`,
`Result`, `ResultAsync`, and a full combinator API:

```typescript
import { ok, err, Result, ResultAsync } from 'neverthrow';

function findUser(id: string): ResultAsync<User, UserNotFoundError> {
  return ResultAsync.fromPromise(
    db.users.findUnique({ where: { id } }),
    () => ({ type: 'USER_NOT_FOUND' as const, userId: id }),
  ).andThen((user) =>
    user ? ok(user) : err({ type: 'USER_NOT_FOUND' as const, userId: id }),
  );
}
```

**Strengths:** Well-typed, good async support, active maintenance, small bundle.
**Weakness:** Still a dependency. `ResultAsync` wrapping can feel verbose.

### ts-results

Closer to Rust's API with `Ok`, `Err`, `Some`, `None`:

```typescript
import { Ok, Err, Result } from 'ts-results';

function divide(a: number, b: number): Result<number, string> {
  if (b === 0) return Err('Division by zero');
  return Ok(a / b);
}
```

**Strengths:** Familiar to Rust developers, includes `Option` type.
**Weakness:** Smaller community, less async-focused.

### Effect

A full functional programming runtime. Result handling is a small part of a much
larger system (fibers, scheduling, dependency injection, streaming):

```typescript
import { Effect, pipe } from 'effect';

const program = pipe(
  findOrder(orderId),
  Effect.flatMap(validateStock),
  Effect.flatMap(chargePayment),
  Effect.mapError(domainErrorToApiError),
);
```

**Strengths:** Extremely powerful composition, handles async/sync uniformly.
**Weakness:** Steep learning curve, large conceptual surface area, heavy for
teams that only need error handling.

### When to Roll Your Own

| Project Context | Recommendation |
|----------------|----------------|
| Small service, few fallible operations | Roll your own (30 lines) |
| Medium service, moderate pipelines | `neverthrow` |
| Large application, deep composition needs | `neverthrow` or `Effect` |
| Team already uses functional patterns | `Effect` |

The 30-line implementation shown earlier in this document is sufficient for
many projects. Adopt a library when you find yourself reimplementing
`ResultAsync`, `combine`, or `safeTry`.

---

## Java Parallel

If you already have Java mental models, this table maps them directly:

| Java | TypeScript Equivalent | Notes |
|------|----------------------|-------|
| Checked exceptions (`throws`) | `Result<T, E>` return type | Both force the caller to handle. Java uses the compiler's exception checker; TS uses union narrowing. |
| `@ControllerAdvice` / `@ExceptionHandler` | Express error middleware `(err, req, res, next)` | Both centralize error-to-response mapping. Spring matches by exception class; Express uses a catch-all with manual dispatch. |
| `Optional.empty()` | `err({ type: 'NOT_FOUND', ... })` | Java `Optional` signals absence. TS `Result.err()` signals absence _with a reason_. Result is strictly more informative. |
| `RuntimeException` (unchecked) | `throw new Error(...)` | Both represent programmer errors that propagate unchecked. Neither forces handling. |
| `try-with-resources` | `try/finally` or `using` (Stage 3 proposal) | Resource cleanup. TS has no equivalent of `AutoCloseable` yet, but the `using` keyword with `Symbol.dispose` is arriving. |
| Exception hierarchy (`extends`) | Discriminated union variants | Java organizes errors by class hierarchy. TS organizes by literal `type` field. TS approach is more compositional (unions merge, class hierarchies do not). |

### @ControllerAdvice vs Express Error Middleware

In Spring:

```java
@ControllerAdvice
public class OrderExceptionHandler {
    @ExceptionHandler(OrderNotFoundException.class)
    public ResponseEntity<ErrorResponse> handle(OrderNotFoundException ex) {
        return ResponseEntity.status(404)
            .body(new ErrorResponse("ORDER_NOT_FOUND", ex.getMessage()));
    }
}
```

In Express:

```typescript
const errorHandler: ErrorRequestHandler = (err, _req, res, _next) => {
  if (isApiError(err)) {
    return res.status(err.status).json({
      code: err.code,
      message: err.message,
    });
  }
  return res.status(500).json({ code: 'INTERNAL_ERROR', message: 'Unexpected error' });
};
```

The key difference: Spring dispatches to the correct handler by exception type
automatically. Express gives you one handler where you do the dispatching
yourself. The Result pattern avoids needing this middleware for business errors
entirely, since they never become exceptions.

---

## Putting It All Together

A complete request flow:

```
1. Express route handler receives request
2. Validate input → Result<Dto, ValidationError>
3. Call domain service → Result<Order, OrderError | PaymentError>
4. Map success → 201 with order JSON
5. Map error → domainErrorToApiError → { status, code, message }
6. If anything throws (bug, infra) → Express error middleware → 500
7. If process crashes → process.on('uncaughtException') → log + exit
```

Each layer handles its own concerns:

- **Domain:** Knows business rules, returns `Result` with domain errors
- **Application:** Maps domain errors to API errors, hides internals
- **HTTP:** Sends JSON responses with appropriate status codes
- **Middleware:** Safety net for unexpected failures, generic 500

The compiler enforces handling at every boundary. New error variants cause
compile errors in switch statements. Internal details like risk scores never
reach the client. This is what "propagate domain errors to HTTP responses
without leaking internals" looks like in practice.

---

## References

- [TypeScript Handbook: Narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html) — discriminated unions and type guards
- [TypeScript Handbook: Utility Types](https://www.typescriptlang.org/docs/handbook/utility-types.html) — `Readonly`, `Record`, and friends
- [neverthrow](https://github.com/supermacro/neverthrow) — Type-safe error handling for TypeScript
- [ts-results](https://github.com/vultix/ts-results) — Rust-inspired Result and Option types
- [Effect](https://effect.website/) — Full functional effect system for TypeScript
- [Express Error Handling Guide](https://expressjs.com/en/guide/error-handling.html) — official docs on error middleware
- [TC39 Explicit Resource Management](https://github.com/tc39/proposal-explicit-resource-management) — `using` declarations and `Symbol.dispose`
- [Rust std::result](https://doc.rust-lang.org/std/result/) — Rust's Result type for comparison
- [Spring @ControllerAdvice](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-advice.html) — centralized exception handling in Spring MVC
