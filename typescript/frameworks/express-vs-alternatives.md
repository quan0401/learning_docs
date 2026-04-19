# Express vs Fastify vs Hono vs Elysia

> Should I migrate from Express, and what breaks when I do?

If you have been building Node.js APIs with Express, you have probably noticed its age showing: no built-in TypeScript support, no schema validation, callback-style error handling, and performance numbers that trail every modern alternative. This doc compares Express against three credible replacements — Fastify, Hono, and Elysia — using the **same route** implemented four ways so you can judge the tradeoffs directly.

---

## 1. The Reference Route

Every framework section below implements the same thing:

- `POST /users` that accepts `{ name: string, email: string, age: number }`
- Validates the request body
- Returns `201` with the created user (plus an `id`)
- Returns `400` on validation failure

This keeps comparisons honest. No cherry-picked features.

---

## 2. Express — The Incumbent

### Mental Model

Express uses a **middleware chain**: every request flows through a stack of `(req, res, next)` functions. Middleware is global or route-scoped. There is no built-in validation, no schema, no type inference. You bolt on everything yourself.

### The Route

```typescript
import express, { Request, Response, NextFunction } from "express";

const app = express();
app.use(express.json());

interface CreateUserBody {
  name: string;
  email: string;
  age: number;
}

// Manual validation — Express gives you nothing
function validateCreateUser(
  req: Request,
  res: Response,
  next: NextFunction
): void {
  const { name, email, age } = req.body;
  if (
    typeof name !== "string" ||
    typeof email !== "string" ||
    typeof age !== "number"
  ) {
    res.status(400).json({ error: "Invalid body" });
    return;
  }
  next();
}

app.post("/users", validateCreateUser, (req: Request, res: Response) => {
  const body = req.body as CreateUserBody; // cast — no inference
  const user = { id: crypto.randomUUID(), ...body };
  res.status(201).json(user);
});

// Error handler — must have 4 params or Express ignores it
app.use((err: Error, _req: Request, res: Response, _next: NextFunction) => {
  res.status(500).json({ error: err.message });
});

app.listen(3000);
```

### Where Express Falls Short

| Problem | Detail |
|---------|--------|
| No type safety | `req.body` is `any`. You cast or assert manually. |
| No schema validation | You write validators by hand or add `zod`, `joi`, `express-validator`. |
| Callback error handling | The 4-argument error middleware signature is easy to get wrong. Async errors require `express-async-errors` or manual try/catch. |
| Performance | Slowest of the four frameworks by a wide margin. |
| Aging API | `req`/`res` mutate in place. The API predates Promises, `async/await`, and the Fetch standard. |

---

## 3. Fastify — The Drop-In Upgrade

### Mental Model

Fastify uses a **plugin architecture** with encapsulated scoping. Routes declare JSON Schema (or TypeBox) for validation, and Fastify infers TypeScript types from that schema automatically. A hooks system (`onRequest`, `preValidation`, `preHandler`, `onSend`, etc.) replaces Express's linear middleware chain with explicit lifecycle stages.

### The Route

```typescript
import Fastify from "fastify";
import { Type, Static } from "@sinclair/typebox";

const app = Fastify({ logger: true });

const CreateUserBody = Type.Object({
  name: Type.String({ minLength: 1 }),
  email: Type.String({ format: "email" }),
  age: Type.Integer({ minimum: 0 }),
});

type CreateUserBody = Static<typeof CreateUserBody>;

const CreateUserResponse = Type.Object({
  id: Type.String(),
  name: Type.String(),
  email: Type.String(),
  age: Type.Integer(),
});

app.post(
  "/users",
  {
    schema: {
      body: CreateUserBody,
      response: { 201: CreateUserResponse },
    },
  },
  async (request, reply) => {
    // request.body is fully typed as { name: string; email: string; age: number }
    const user = { id: crypto.randomUUID(), ...request.body };
    return reply.status(201).send(user);
  }
);

app.listen({ port: 3000 });
```

### What You Get Over Express

- **Schema = validation = types.** One declaration drives runtime validation AND TypeScript inference. No manual casting.
- **Automatic serialization.** Fastify uses `fast-json-stringify` to serialize responses based on the response schema — faster than `JSON.stringify`.
- **Plugin encapsulation.** Plugins scope their decorators, hooks, and routes. A plugin registered under `/admin` cannot leak state to `/public`.
- **Async-first.** Route handlers return promises. No `next()`, no 4-argument error handlers.
- **2-3x faster than Express** for JSON responses.

### Hooks Lifecycle

```
Incoming Request
  → onRequest
  → preParsing
  → preValidation
  → preHandler        ← most "middleware" logic goes here
  → handler
  → preSerialization
  → onSend
  → onResponse
```

Compare this to Express's flat `use()` chain where you guess execution order by registration order.

---

## 4. Hono — Ultralight, Multi-Runtime

### Mental Model

Hono is built for the **edge-first world**. It runs on Node, Deno, Bun, Cloudflare Workers, Vercel Edge, AWS Lambda, and more — same codebase, no adapters. The API is Web Standards-based (`Request`/`Response`). Middleware uses `c.next()` (context-based, not `req/res/next`). Hono is tiny: the core is under 14kb.

### The Route

```typescript
import { Hono } from "hono";
import { zValidator } from "@hono/zod-validator";
import { z } from "zod";

const app = new Hono();

const createUserSchema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
  age: z.number().int().nonnegative(),
});

app.post("/users", zValidator("json", createUserSchema), (c) => {
  // c.req.valid("json") is typed as { name: string; email: string; age: number }
  const body = c.req.valid("json");
  const user = { id: crypto.randomUUID(), ...body };
  return c.json(user, 201);
});

export default app;
```

### What You Get Over Express

- **Multi-runtime.** Deploy the same code to Cloudflare Workers, Vercel Edge, Deno Deploy, or a traditional Node server. No framework lock-in to a runtime.
- **Web Standards.** Uses the Fetch API `Request`/`Response` objects — portable, no proprietary API.
- **Zod integration.** First-class `@hono/zod-validator` middleware. Schema validates AND infers types.
- **RPC mode.** Type-safe client generation from your routes — similar to tRPC but without the tRPC lock-in.

### RPC Mode (End-to-End Type Safety)

```typescript
// server.ts
import { Hono } from "hono";
import { zValidator } from "@hono/zod-validator";
import { z } from "zod";

const app = new Hono()
  .post(
    "/users",
    zValidator(
      "json",
      z.object({
        name: z.string(),
        email: z.string().email(),
        age: z.number(),
      })
    ),
    (c) => {
      const body = c.req.valid("json");
      return c.json({ id: crypto.randomUUID(), ...body }, 201);
    }
  );

export type AppType = typeof app;

// client.ts
import { hc } from "hono/client";
import type { AppType } from "./server";

const client = hc<AppType>("http://localhost:3000");

// Fully typed — IDE autocompletes route, method, body, and response
const res = await client.users.$post({
  json: { name: "Alice", email: "alice@example.com", age: 30 },
});
const user = await res.json(); // typed as { id: string; name: string; ... }
```

---

## 5. Elysia — Bun-Native, Best-in-Class Type Safety

### Mental Model

Elysia is **Bun-only**. It does not run on Node.js. If your deployment must be Node, stop here.

Elysia achieves end-to-end type safety through **function composition** — no decorators, no code generation, no separate schema declaration step. The `t.Object()` schema simultaneously validates at runtime AND infers TypeScript types. Eden Treaty generates a fully typed client from the server definition.

### The Route

```typescript
import { Elysia, t } from "elysia";

const app = new Elysia()
  .post(
    "/users",
    ({ body }) => {
      // body is typed as { name: string; email: string; age: number }
      return { id: crypto.randomUUID(), ...body };
    },
    {
      body: t.Object({
        name: t.String({ minLength: 1 }),
        email: t.String({ format: "email" }),
        age: t.Integer({ minimum: 0 }),
      }),
      response: {
        201: t.Object({
          id: t.String(),
          name: t.String(),
          email: t.String(),
          age: t.Integer(),
        }),
      },
    }
  )
  .listen(3000);

export type App = typeof app;
```

### Eden Treaty (Type-Safe Client)

```typescript
import { treaty } from "@elysiajs/eden";
import type { App } from "./server";

const api = treaty<App>("http://localhost:3000");

// Fully typed — validation errors caught at compile time
const { data, error } = await api.users.post({
  name: "Alice",
  email: "alice@example.com",
  age: 30,
});

if (data) {
  console.log(data.id); // string — inferred from server response schema
}
```

### What Sets Elysia Apart

- **Single declaration = runtime validation + TypeScript types.** `t.Object()` is both a validator and a type definition. No `z.infer<>`, no `Static<typeof>`, no separate interface.
- **Function composition over middleware.** Plugins compose via `.use()` chaining. Each plugin's types propagate through the chain.
- **Fastest of the four.** Bun's HTTP server plus Elysia's compile-time route optimization.
- **Trade-off: Bun lock-in.** You cannot deploy Elysia on Node.js. Your CI, your Docker images, your hosting — all must support Bun.

---

## 6. Performance Comparison

Approximate requests/sec for a simple JSON response (`{ "hello": "world" }`), single-threaded:

| Framework | Runtime | Req/sec (approx.) | Relative |
|-----------|---------|-------------------|----------|
| Express | Node | ~15,000 | 1x |
| Fastify | Node | ~45,000 | 3x |
| Hono | Bun | ~60,000 | 4x |
| Hono | Node | ~35,000 | 2.3x |
| Elysia | Bun | ~80,000 | 5.3x |

**Caveats:**
- Benchmarks vary wildly by machine, Node/Bun version, and workload. These are ballpark figures from TechEmpower-style JSON benchmarks.
- Hono on Node is slower than Hono on Bun — the runtime matters as much as the framework.
- Elysia cannot run on Node, so the comparison is cross-runtime by necessity.

### When Raw Performance Matters

It usually does not. If your API does database queries, calls external services, and processes business logic, the framework overhead is a rounding error. **Pick the framework for its developer experience, not its benchmark numbers.**

Performance matters when:
- You serve edge functions with strict cold-start and latency budgets (Cloudflare Workers, Vercel Edge)
- You handle very high throughput with minimal per-request logic (API gateways, proxies, real-time endpoints)
- You run on constrained infrastructure where every millisecond of overhead compounds

---

## 7. Type Safety Comparison

The same typed request across all four frameworks:

### Express — Manual Types

```typescript
// You define the interface yourself. Nothing connects it to runtime validation.
interface CreateUserBody {
  name: string;
  email: string;
  age: number;
}

app.post("/users", (req: Request, res: Response) => {
  const body = req.body as CreateUserBody; // unsafe cast
});
```

### Fastify — Schema-Inferred Types

```typescript
// TypeBox schema generates both JSON Schema (runtime) and TS type (compile time)
const CreateUserBody = Type.Object({
  name: Type.String(),
  email: Type.String({ format: "email" }),
  age: Type.Integer(),
});

// request.body automatically typed from schema — no cast needed
```

### Hono — Validator-Inferred Types

```typescript
// Zod schema validates at runtime. c.req.valid() infers the type.
const schema = z.object({ name: z.string(), email: z.string(), age: z.number() });

app.post("/users", zValidator("json", schema), (c) => {
  const body = c.req.valid("json"); // typed from schema — no cast
});
```

### Elysia — First-Class Types

```typescript
// t.Object() IS the type AND the validator. One object, two jobs.
app.post("/users", ({ body }) => { /* body is typed */ }, {
  body: t.Object({ name: t.String(), email: t.String(), age: t.Integer() }),
});
```

### Summary Table

| Feature | Express | Fastify | Hono | Elysia |
|---------|---------|---------|------|--------|
| Request body typing | Manual cast | Schema-inferred | Validator-inferred | First-class |
| Response typing | None | Schema-inferred | RPC mode | First-class |
| Runtime validation | Manual / third-party | JSON Schema / TypeBox | Zod / custom | Built-in (`t.*`) |
| End-to-end client types | No | No (use OpenAPI) | RPC mode | Eden Treaty |
| Type errors catch bad requests at compile time | No | Partial | Yes (with RPC) | Yes |

---

## 8. Middleware Ecosystem

Express has the largest middleware ecosystem by far. The question is whether the alternatives cover what you actually need.

| Middleware | Express | Fastify | Hono | Elysia |
|-----------|---------|---------|------|--------|
| JSON body parsing | Built-in | Built-in | Built-in | Built-in |
| CORS | `cors` | `@fastify/cors` | `hono/cors` | `@elysiajs/cors` |
| Helmet / security headers | `helmet` | `@fastify/helmet` | `hono/secure-headers` | `@elysiajs/server-timing` + manual |
| Rate limiting | `express-rate-limit` | `@fastify/rate-limit` | Third-party / custom | `elysia-rate-limit` |
| Session | `express-session` | `@fastify/session` | Third-party | Third-party |
| Static files | `express.static` | `@fastify/static` | `hono/serve-static` | `@elysiajs/static` |
| Cookie parsing | `cookie-parser` | `@fastify/cookie` | `hono/cookie` | Built-in |
| JWT auth | `jsonwebtoken` + manual | `@fastify/jwt` | `hono/jwt` | `@elysiajs/jwt` |
| Swagger / OpenAPI | `swagger-ui-express` | `@fastify/swagger` | Third-party | `@elysiajs/swagger` |
| File uploads | `multer` | `@fastify/multipart` | Third-party | Built-in |
| WebSocket | `ws` + manual | `@fastify/websocket` | `hono/ws` (runtime-dependent) | Built-in |

**Key takeaway:** Fastify has official plugins for nearly everything Express has. Hono and Elysia cover the essentials but you will occasionally write custom middleware or find community packages with less maturity.

Fastify also has `@fastify/express` — a compatibility layer that lets you use Express middleware inside Fastify during migration.

---

## 9. Migration from Express to Fastify

Fastify is the most compatible migration target for an existing Express codebase. Here is the step-by-step.

### Step 1: Install Fastify and TypeBox

```bash
npm install fastify @sinclair/typebox
npm install @fastify/cors @fastify/helmet  # replace Express equivalents
```

### Step 2: Replace the App Scaffold

```typescript
// Before (Express)
import express from "express";
const app = express();
app.use(express.json());
app.use(cors());

// After (Fastify)
import Fastify from "fastify";
import cors from "@fastify/cors";
const app = Fastify({ logger: true });
await app.register(cors);
// No body parser needed — Fastify parses JSON by default
```

### Step 3: Convert Routes

```typescript
// Before (Express)
app.get("/users/:id", async (req: Request, res: Response) => {
  const user = await db.findUser(req.params.id);
  if (!user) {
    return res.status(404).json({ error: "Not found" });
  }
  res.json(user);
});

// After (Fastify)
app.get<{ Params: { id: string } }>("/users/:id", async (request, reply) => {
  const user = await db.findUser(request.params.id);
  if (!user) {
    return reply.status(404).send({ error: "Not found" });
  }
  return reply.send(user);
});
```

### Step 4: Convert Error Handling

```typescript
// Before (Express) — 4-argument middleware
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  res.status(500).json({ error: err.message });
});

// After (Fastify) — setErrorHandler
app.setErrorHandler((error, request, reply) => {
  request.log.error(error);
  reply.status(error.statusCode ?? 500).send({ error: error.message });
});
```

### Step 5: Convert Middleware to Hooks or Plugins

```typescript
// Before (Express) — auth middleware
function authMiddleware(req: Request, res: Response, next: NextFunction) {
  const token = req.headers.authorization;
  if (!verifyToken(token)) {
    return res.status(401).json({ error: "Unauthorized" });
  }
  req.user = decodeToken(token);
  next();
}

// After (Fastify) — preHandler hook or decorator
app.addHook("preHandler", async (request, reply) => {
  const token = request.headers.authorization;
  if (!verifyToken(token)) {
    return reply.status(401).send({ error: "Unauthorized" });
  }
  request.user = decodeToken(token); // requires decorateRequest first
});
```

### What Breaks During Migration

| Express Pattern | Fastify Equivalent | Breaking Change |
|----------------|-------------------|-----------------|
| `req.body` available everywhere | Must register body parser or rely on default | Body only parsed for `POST`/`PUT`/`PATCH` with correct `Content-Type` |
| `res.json()` | `reply.send()` | Different method name; return value matters (return the reply to short-circuit) |
| `app.use(middleware)` | `app.addHook()` or `app.register(plugin)` | No direct 1:1 mapping. Global middleware becomes hooks; scoped middleware becomes plugins. |
| 4-argument error handler | `app.setErrorHandler()` | Completely different API |
| `req.query` is always an object | `request.query` typing depends on schema | Add query schema or type assertions |
| `express.Router()` | Fastify plugins with `fastify.register()` | Routers become encapsulated plugins with their own scope |
| `res.locals` for request-scoped data | `request.decorateRequest()` + hooks | Must declare decorators up front |

### Estimated Migration Effort

For a medium-sized Express app (20-50 routes, 5-10 middleware):

| Task | Effort |
|------|--------|
| Scaffold swap and plugin registration | 1-2 hours |
| Route conversion | 1-2 days (mostly mechanical) |
| Middleware to hooks/plugins | 1-2 days (requires understanding Fastify's lifecycle) |
| Error handling conversion | 2-4 hours |
| Adding schemas (optional but recommended) | 2-3 days |
| Testing and edge cases | 1-2 days |
| **Total** | **~1-2 weeks** |

The route conversion is mechanical. The hard part is middleware that relied on Express-specific behavior like `res.locals`, shared mutable state between middleware, or libraries that patch `req`/`res`.

---

## 10. Decision Framework

### Stay on Express If...

- You have a large legacy codebase and no performance problems
- Your team knows Express and has no bandwidth for migration
- You depend on Express-only middleware with no Fastify/Hono equivalent
- The API is in maintenance mode — new features are rare

### Pick Fastify If...

- You want the closest migration path from Express
- You need to stay on Node.js (corporate requirement, existing infrastructure)
- You want schema validation and type inference without changing your deployment
- You value a mature plugin ecosystem and active maintenance

### Pick Hono If...

- You deploy to edge runtimes (Cloudflare Workers, Vercel Edge, Deno Deploy)
- You want one codebase that runs on any JavaScript runtime
- You prefer Zod for validation (already use it elsewhere)
- You want a lightweight framework with minimal abstractions
- You want end-to-end type safety via RPC mode

### Pick Elysia If...

- You can commit to Bun as your runtime (dev, CI, and production)
- Type safety is your top priority and you want it with zero boilerplate
- You want the best raw performance
- You are starting a new project (not migrating an existing Express app)
- You accept a smaller ecosystem in exchange for developer experience

### Quick Reference

| Criterion | Express | Fastify | Hono | Elysia |
|-----------|---------|---------|------|--------|
| Runtime | Node | Node | Any | Bun only |
| Type safety | None | Schema-inferred | Zod + RPC | First-class |
| Performance | Baseline | 3x | 2-4x | 5x |
| Ecosystem size | Largest | Large | Growing | Smallest |
| Migration from Express | N/A | Easy | Moderate | Hard (runtime change) |
| Edge deployment | No | No | Yes | No (Bun only) |
| Learning curve from Express | N/A | Low | Low | Medium |
| Maturity | 14+ years | 7+ years | 3+ years | 2+ years |

---

## 11. Common Gotchas

### Express Habits That Do Not Transfer

1. **Mutating `req`/`res`.** Fastify, Hono, and Elysia discourage or prevent mutation. Use decorators (Fastify), context (Hono), or derive/resolve (Elysia) instead.

2. **Relying on middleware order.** Express middleware runs in registration order. Fastify has an explicit lifecycle. Hono middleware is ordered but uses `c.next()`. Elysia composes plugins functionally.

3. **Synchronous error handling.** Express requires the 4-argument signature and `express-async-errors` for async. All three alternatives handle async errors natively.

4. **`req.body` is always available.** Fastify only parses supported content types. Hono requires explicit body parsing for some runtimes. Elysia parses based on schema declaration.

### Bun vs Node — The Runtime Decision

Choosing Elysia means choosing Bun. Consider:

- **npm compatibility:** Bun supports most npm packages but not all. Native addons (`bcrypt`, `sharp`, `canvas`) may need Bun-specific builds or alternatives.
- **Production readiness:** Bun is stable for HTTP servers but younger than Node for long-running production workloads.
- **Hosting:** Fly.io, Railway, and Docker all support Bun. AWS Lambda and some PaaS providers may not have first-class Bun support yet.
- **Team adoption:** Everyone on the team needs Bun installed locally and in CI.

---

## References

- [Express.js Documentation](https://expressjs.com/)
- [Fastify Documentation](https://fastify.dev/docs/latest/)
- [Fastify TypeBox Integration](https://fastify.dev/docs/latest/Reference/Type-Providers/)
- [Hono Documentation](https://hono.dev/)
- [Hono RPC Client](https://hono.dev/docs/guides/rpc)
- [Elysia Documentation](https://elysiajs.com/)
- [Eden Treaty](https://elysiajs.com/eden/overview)
- [TypeBox (Sinclair)](https://github.com/sinclairzx81/typebox)
- [TechEmpower Framework Benchmarks](https://www.techempower.com/benchmarks/)
- [Bun Runtime](https://bun.sh/)
