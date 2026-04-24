# Node.js in Kubernetes

**Updated:** 2026-04-24

> **Problem**: Your pod gets OOMKilled at 3 AM. The container memory limit is 512Mi, but `--max-old-space-size` was never set. V8 happily allocates beyond the container ceiling, the kernel kills the process, and Kubernetes restarts it in a crash loop. This doc covers every operational concern for running Node.js in Kubernetes reliably. For the Spring/JVM equivalent patterns, see [Kubernetes for Spring Boot](../../java/configurations/kubernetes-spring-boot.md).

---

## Graceful Shutdown

Kubernetes sends `SIGTERM` when it wants a pod gone. Node.js does **nothing** with that signal by default -- the process just dies, dropping every in-flight request.

### The Problem

1. Kubernetes removes the pod from the Service endpoints (async, via the API server).
2. Kubernetes sends `SIGTERM` to the container.
3. The pod has `terminationGracePeriodSeconds` (default 30s) to exit before `SIGKILL`.

Steps 1 and 2 happen **concurrently**. The load balancer might still route traffic to the pod for a few seconds after `SIGTERM` arrives. This is the source of most "connection reset" errors during deploys.

### Complete Shutdown Handler

```typescript
import { Server } from "node:http";

function createGracefulShutdown(server: Server, timeoutMs = 25_000) {
  let isShuttingDown = false;

  return (signal: string) => {
    if (isShuttingDown) return;
    isShuttingDown = true;

    console.log(`Received ${signal}. Starting graceful shutdown...`);

    // 1. Stop accepting new connections
    server.close((err) => {
      if (err) {
        console.error("Error closing server:", err);
        process.exit(1);
      }
      console.log("All connections drained. Exiting.");
      process.exit(0);
    });

    // 2. Force exit after timeout (leave 5s buffer before SIGKILL)
    setTimeout(() => {
      console.error("Shutdown timed out. Forcing exit.");
      process.exit(1);
    }, timeoutMs).unref();
  };
}

// Usage
const server = app.listen(3000);
const shutdown = createGracefulShutdown(server);

process.on("SIGTERM", () => shutdown("SIGTERM"));
process.on("SIGINT", () => shutdown("SIGINT"));
```

### Fastify Built-In Support

Fastify handles this natively with `app.close()`, which drains connections and runs `onClose` hooks. The same `SIGTERM`/`SIGINT` handler pattern applies -- call `await app.close()` then `process.exit(0)`.

### Pre-Stop Hook for Load Balancer Deregistration

Because endpoint removal and `SIGTERM` race each other, add a `preStop` hook that sleeps long enough for the load balancer to catch up:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 5"]
```

The sequence becomes:
1. Kubernetes marks pod as `Terminating`.
2. `preStop` runs -- pod sleeps 5s while endpoints update propagates.
3. `SIGTERM` arrives. The shutdown handler stops new connections and drains in-flight.
4. Pod exits cleanly.

Set `terminationGracePeriodSeconds` to at least `preStop sleep + shutdown timeout + buffer`:

```yaml
spec:
  terminationGracePeriodSeconds: 35
  containers:
    - name: api
      lifecycle:
        preStop:
          exec:
            command: ["sh", "-c", "sleep 5"]
```

---

## Health Checks

Kubernetes uses three probe types to decide what to do with your pod. Getting these wrong causes restart loops or traffic sent to unready pods.

### Liveness Probe

**Question it answers**: Is the process alive and not deadlocked?

**What to check**: Almost nothing. A simple 200 response proves the event loop is not blocked. Do NOT check database connectivity here -- a DB outage would restart every pod, making recovery harder.

```typescript
app.get("/healthz", (_req, res) => {
  res.status(200).json({ status: "ok" });
});
```

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 3000
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 3  # 3 failures = restart
```

### Readiness Probe

**Question it answers**: Can this pod handle traffic right now?

**What to check**: Dependencies that must be available for the pod to serve requests. Database connection pool, cache connection, required external services.

```typescript
import { Pool } from "pg";

const pool = new Pool();

app.get("/ready", async (_req, res) => {
  try {
    await pool.query("SELECT 1");
    res.status(200).json({ status: "ready" });
  } catch (err) {
    res.status(503).json({
      status: "not ready",
      reason: "database unavailable",
    });
  }
});
```

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 2  # 2 failures = stop sending traffic
```

When readiness fails, the pod stays running but is removed from Service endpoints. Traffic stops flowing to it until it passes again.

### Startup Probe

**Question it answers**: Has the app finished its slow initialization?

Use when your app needs to load large datasets, warm caches, or run migrations before serving. Without a startup probe, the liveness probe might kill the pod before it finishes starting.

```typescript
let startupComplete = false;

async function initialize() {
  await runMigrations();
  await warmCache();
  startupComplete = true;
}

app.get("/startup", (_req, res) => {
  if (startupComplete) {
    res.status(200).json({ status: "started" });
  } else {
    res.status(503).json({ status: "starting" });
  }
});
```

```yaml
startupProbe:
  httpGet:
    path: /startup
    port: 3000
  periodSeconds: 5
  failureThreshold: 30  # 30 * 5s = 150s max startup time
```

### Probe Summary

| Probe | Failure Action | Check Scope | Danger If Wrong |
|-------|---------------|-------------|-----------------|
| Liveness | Restart pod | Process health only | Restart storm during DB outage |
| Readiness | Remove from endpoints | Dependencies | Traffic to broken pods |
| Startup | Kill pod (hasn't started) | Init complete | Pod never starts |

---

## Resource Limits and OOMKill

This is where most teams get burned. The interaction between V8's heap management and Linux cgroup memory limits is not intuitive.

### How OOMKill Happens

V8 manages its own heap with `--max-old-space-size`. The Linux OOM killer watches **RSS** (Resident Set Size) -- the total physical memory the process uses. RSS includes:

- V8 heap (old space + new space)
- Native memory (Buffers, C++ objects, libuv)
- Thread stacks
- Shared libraries

V8 heap is only **part** of RSS. If `--max-old-space-size` equals the container limit, native memory pushes RSS over and the kernel kills the process.

### The 75% Rule

Set `--max-old-space-size` to roughly 75% of the container memory limit:

```yaml
resources:
  requests:
    memory: "384Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "1000m"
```

```dockerfile
CMD ["node", "--max-old-space-size=384", "dist/server.js"]
```

| Container Limit | `--max-old-space-size` | Headroom for Native |
|----------------|----------------------|-------------------|
| 256Mi | 192 | 64Mi |
| 512Mi | 384 | 128Mi |
| 1Gi | 768 | 256Mi |
| 2Gi | 1536 | 512Mi |

### Request vs Limit Sizing

- **Requests**: What the scheduler guarantees. Set to typical steady-state usage.
- **Limits**: The ceiling before OOMKill. Set to peak usage + buffer.

A large gap between request and limit means overcommitment. The pod might get scheduled on a node that cannot actually handle peak usage of all its pods simultaneously.

```yaml
# Good: tight ratio, predictable scheduling
resources:
  requests:
    memory: "384Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "1000m"

# Risky: 4x overcommit on memory
resources:
  requests:
    memory: "128Mi"
  limits:
    memory: "512Mi"
```

### Detecting OOMKill

```bash
# Check if last restart was OOMKill
kubectl describe pod <pod-name> | grep -A 5 "Last State"

# Look for OOMKilled reason
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
```

### V8 Default Behavior

Without `--max-old-space-size`, V8 uses a heuristic based on available system memory. In a container, "available memory" might be the **node's** total RAM (on older kernels) or the **cgroup limit** (on newer kernels with `--detect-cgroup-limits` or Node 18+). Do not rely on defaults -- always set the flag explicitly.

---

## Horizontal Scaling

Node.js runs on a single thread. Vertical scaling hits diminishing returns quickly. Horizontal scaling via Kubernetes HPA is the natural fit.

### Stateless Design Requirements

Every piece of mutable state must live outside the pod:

| State Type | Wrong (In-Process) | Right (External) |
|-----------|-------------------|-----------------|
| Sessions | `express-session` memory store | Redis, DynamoDB |
| File uploads | Local `/tmp` | S3, GCS |
| Cache | In-memory Map | Redis, Memcached |
| Scheduled jobs | `setInterval` | External scheduler (CronJob, Temporal) |
| WebSocket state | Connection map in memory | Redis pub/sub + sticky sessions |

### Session Management with Redis

```typescript
import RedisStore from "connect-redis";
import session from "express-session";
import { createClient } from "redis";

const redisClient = createClient({
  url: process.env.REDIS_URL,
});
await redisClient.connect();

app.use(
  session({
    store: new RedisStore({ client: redisClient }),
    secret: process.env.SESSION_SECRET!,
    resave: false,
    saveUninitialized: false,
    cookie: { secure: true, maxAge: 86400000 },
  })
);
```

### HPA Configuration

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Pods
      pods:
        metric:
          name: nodejs_eventloop_lag_p99_seconds
        target:
          type: AverageValue
          averageValue: "0.1"  # Scale if p99 event loop lag > 100ms
```

### Pod Disruption Budget

Prevent deployments from killing all replicas at once:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: api
```

---

## 12-Factor Alignment

The Twelve-Factor App methodology maps well to Node.js on Kubernetes, but some factors need deliberate effort.

### What Node Does Well Naturally

| Factor | How |
|--------|-----|
| **II. Dependencies** | `package.json` + `package-lock.json` declare everything explicitly |
| **VI. Processes** | Single-threaded, stateless by nature (if you avoid in-memory state) |
| **VII. Port Binding** | `http.createServer().listen(PORT)` -- self-contained, no app server wrapper |
| **XI. Logs** | `console.log` goes to stdout. Structured loggers (pino) do the same |

### What Needs Effort

| Factor | Challenge | Solution |
|--------|-----------|----------|
| **III. Config** | Developers hardcode values or use `.env` files | Read from env vars, validate at startup with `zod` or `envalid` |
| **IV. Backing Services** | Connection strings baked into code | Inject via env vars or K8s Secrets |
| **X. Dev/Prod Parity** | Different OS, different Node version | Use the same Docker image locally and in prod |
| **XII. Admin Processes** | One-off scripts run differently | K8s Jobs with the same image |

### Config Validation at Startup

```typescript
import { z } from "zod";

const envSchema = z.object({
  NODE_ENV: z.enum(["development", "production", "test"]),
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  LOG_LEVEL: z.enum(["debug", "info", "warn", "error"]).default("info"),
});

export const config = envSchema.parse(process.env);
// Throws at startup if any required var is missing or invalid
```

---

## Container Best Practices

### Multi-Stage Dockerfile

```dockerfile
# ---- Build Stage ----
FROM node:20-slim AS builder

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY tsconfig.json ./
COPY src/ ./src/
RUN npm run build

# ---- Production Stage ----
FROM node:20-slim AS runtime

RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci --omit=dev && npm cache clean --force

COPY --from=builder /app/dist ./dist

ENV NODE_ENV=production

USER appuser

EXPOSE 3000

CMD ["node", "--max-old-space-size=384", "dist/server.js"]
```

### Base Image Choices

| Image | Size | Pros | Cons |
|-------|------|------|------|
| `node:20` | ~350MB | Full Debian, everything works | Large attack surface |
| `node:20-slim` | ~80MB | Minimal Debian, good balance | Missing some native build tools |
| `node:20-alpine` | ~50MB | Smallest | musl libc -- some native modules break |
| Distroless (`gcr.io/distroless/nodejs20`) | ~40MB | No shell, minimal attack surface | Cannot exec into container to debug |

**Recommendation**: Start with `node:20-slim`. Switch to Alpine only if you have verified all native dependencies compile against musl. Avoid distroless until you have solid observability -- you lose the ability to shell in.

### .dockerignore

```
node_modules
.git
.env*
*.md
tests/
coverage/
.github/
```

### Security Hardening

```dockerfile
# Non-root user (shown above)
USER appuser

# Read-only filesystem in K8s
# (mount writable tmpfs for /tmp if needed)
```

```yaml
securityContext:
  runAsNonRoot: true
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
```

---

## Java Parallel

If you already run Spring Boot in Kubernetes, most of these concepts have direct equivalents.

### Health Checks

| Concern | Spring Boot | Node.js |
|---------|-------------|---------|
| Liveness | Actuator `/actuator/health/liveness` (auto) | Manual `GET /healthz` endpoint |
| Readiness | Actuator `/actuator/health/readiness` (auto) | Manual `GET /ready` with DB check |
| Startup | Actuator startup probe support | Manual `GET /startup` with flag |
| Custom checks | `HealthIndicator` interface | Custom middleware functions |

Spring Boot Actuator gives you health checks out of the box with `management.endpoint.health.probes.enabled=true`. In Node.js, you wire these yourself, but the logic is simpler.

### Memory Configuration

| Concern | JVM (Spring Boot) | Node.js |
|---------|-------------------|---------|
| Heap limit | `-Xmx` / `-XX:MaxRAMPercentage` | `--max-old-space-size` |
| Typical ratio | 75% of container limit | 75% of container limit |
| Off-heap memory | Metaspace, thread stacks, NIO buffers | Native addons, Buffers, libuv |
| OOM behavior | `OutOfMemoryError` (catchable, then exit) | `SIGKILL` from kernel (no chance to handle) |
| Auto-detection | `-XX:+UseContainerSupport` (default since JDK 10) | Partial (Node 18+ reads cgroup, but set explicitly) |

### Graceful Shutdown

| Concern | Spring Boot | Node.js |
|---------|-------------|---------|
| Configuration | `server.shutdown=graceful` in properties | Manual `SIGTERM` handler |
| Timeout | `spring.lifecycle.timeout-per-shutdown-phase=30s` | Manual `setTimeout` |
| Connection drain | Built-in with Tomcat/Netty | `server.close()` stops new, waits for in-flight |
| Complexity | One-line config | ~20 lines of code |

---

## Related

- [Kubernetes for Spring Boot](../../java/configurations/kubernetes-spring-boot.md) — the Spring/Actuator version of probes, graceful shutdown, and HPA wiring.
- [Spring Boot Actuator Deep Dive](../../java/actuator-deep-dive.md) — the Java side of health endpoints and probe groups.
- [Observability for Node Services](observability.md) — metrics, traces, and logs that make K8s incidents debuggable.

## References

- [Kubernetes Documentation -- Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Kubernetes Documentation -- Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Node.js Documentation -- `--max-old-space-size`](https://nodejs.org/api/cli.html#--max-old-space-sizesize-in-mbs)
- [12factor.net](https://12factor.net/)
- [Google Cloud -- Best practices for building containers](https://cloud.google.com/architecture/best-practices-for-building-containers)
- [Fastify Documentation -- Lifecycle and Hooks](https://fastify.dev/docs/latest/Reference/Lifecycle/)
- [Spring Boot Reference -- Graceful Shutdown](https://docs.spring.io/spring-boot/docs/current/reference/html/web.html#web.graceful-shutdown)
- [Spring Boot Reference -- Kubernetes Probes](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html#actuator.endpoints.kubernetes-probes)
