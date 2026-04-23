---
title: "Connection Pooling & Keep-Alive — Managing Persistent Connections"
date: 2026-04-23
updated: 2026-04-23
tags: [networking, connection-pooling, keep-alive, performance, hikari]
---

# Connection Pooling & Keep-Alive — Managing Persistent Connections

**Date:** 2026-04-23 | **Updated:** 2026-04-23
**Tags:** `networking` `connection-pooling` `keep-alive` `performance` `hikari`

---

## Table of Contents

- [Summary](#summary)
- [1. Why Connection Pooling](#1-why-connection-pooling)
  - [1.1 The Cost of a Fresh Connection](#11-the-cost-of-a-fresh-connection)
  - [1.2 Resource Waste Without Pooling](#12-resource-waste-without-pooling)
  - [1.3 Server and Database Connection Limits](#13-server-and-database-connection-limits)
- [2. HTTP Keep-Alive](#2-http-keep-alive)
  - [2.1 Persistent Connections in HTTP/1.1](#21-persistent-connections-in-http11)
  - [2.2 Max Idle Time and Max Requests](#22-max-idle-time-and-max-requests)
  - [2.3 How HTTP/2 Multiplexing Changes Everything](#23-how-http2-multiplexing-changes-everything)
  - [2.4 HTTP/3 and QUIC Connections](#24-http3-and-quic-connections)
- [3. HTTP Client Connection Pools](#3-http-client-connection-pools)
  - [3.1 Node.js http.Agent](#31-nodejs-httpagent)
  - [3.2 undici Pool](#32-undici-pool)
  - [3.3 Java HttpClient Connection Pool](#33-java-httpclient-connection-pool)
  - [3.4 Spring RestTemplate and WebClient Pool Config](#34-spring-resttemplate-and-webclient-pool-config)
- [4. Database Connection Pools](#4-database-connection-pools)
  - [4.1 Why Databases Limit Connections](#41-why-databases-limit-connections)
  - [4.2 Pool Lifecycle](#42-pool-lifecycle)
  - [4.3 Idle Timeout and Max Lifetime](#43-idle-timeout-and-max-lifetime)
- [5. Pool Sizing with Little's Law](#5-pool-sizing-with-littles-law)
  - [5.1 The Formula: L = lambda * W](#51-the-formula-l--lambda--w)
  - [5.2 Practical Calculation Examples](#52-practical-calculation-examples)
  - [5.3 Why Oversizing Hurts](#53-why-oversizing-hurts)
  - [5.4 HikariCP's Formula Recommendation](#54-hikaricps-formula-recommendation)
- [6. HikariCP Deep Dive](#6-hikaricp-deep-dive)
  - [6.1 Core Configuration Properties](#61-core-configuration-properties)
  - [6.2 Leak Detection](#62-leak-detection)
  - [6.3 Spring Boot Auto-Configuration](#63-spring-boot-auto-configuration)
  - [6.4 Configuration Example](#64-configuration-example)
- [7. Node.js Database Pools](#7-nodejs-database-pools)
  - [7.1 pg (node-postgres) Pool Config](#71-pg-node-postgres-pool-config)
  - [7.2 Prisma Connection Pool](#72-prisma-connection-pool)
  - [7.3 Drizzle ORM Pool](#73-drizzle-orm-pool)
  - [7.4 Connection String Parameters](#74-connection-string-parameters)
- [8. PgBouncer](#8-pgbouncer)
  - [8.1 What PgBouncer Does](#81-what-pgbouncer-does)
  - [8.2 Pooling Modes](#82-pooling-modes)
  - [8.3 When to Use PgBouncer](#83-when-to-use-pgbouncer)
  - [8.4 Interaction with Application Pools](#84-interaction-with-application-pools)
- [9. Connection Pool Monitoring](#9-connection-pool-monitoring)
  - [9.1 Metrics to Watch](#91-metrics-to-watch)
  - [9.2 HikariCP Metrics with Micrometer](#92-hikaricp-metrics-with-micrometer)
  - [9.3 Node.js Pool Events](#93-nodejs-pool-events)
  - [9.4 Alerting Thresholds](#94-alerting-thresholds)
- [10. Common Pitfalls](#10-common-pitfalls)
  - [10.1 Pool Exhaustion (Leaked Connections)](#101-pool-exhaustion-leaked-connections)
  - [10.2 Connection Storms on Startup](#102-connection-storms-on-startup)
  - [10.3 Stale Connections After Failover](#103-stale-connections-after-failover)
  - [10.4 DNS Caching Preventing Failover](#104-dns-caching-preventing-failover)
  - [10.5 Too Many Pools](#105-too-many-pools)
- [Related](#related)
- [References](#references)

---

## Summary

Every HTTP request your backend makes to an external service and every database query it runs requires a network connection. Creating that connection from scratch (TCP handshake + TLS negotiation + authentication) costs real time and real resources. Connection pooling amortizes that cost by keeping a set of reusable connections ready to go. This document covers pooling from both sides: HTTP client pools that reuse outbound connections and database pools that share a limited set of database sessions. It walks through the math (Little's Law) that answers "how many connections do I actually need?", the configuration you need in HikariCP (Java/Spring) and node-postgres/Prisma (Node.js), the role of external poolers like PgBouncer, and the monitoring and pitfalls that catch teams in production.

---

## 1. Why Connection Pooling

### 1.1 The Cost of a Fresh Connection

Opening a new TCP connection to a database or external service is not free. The latency adds up:

| Step | Typical Latency (same region) | Typical Latency (cross-region) |
|------|-------------------------------|-------------------------------|
| TCP 3-way handshake | 0.5 - 1 ms | 30 - 80 ms |
| TLS 1.3 handshake | 1 - 2 ms (1-RTT) | 30 - 80 ms (1-RTT) |
| TLS 1.2 handshake | 2 - 4 ms (2-RTT) | 60 - 160 ms (2-RTT) |
| Database auth (PostgreSQL) | 1 - 5 ms | 1 - 5 ms |
| **Total (TLS 1.3 + PG auth)** | **2.5 - 8 ms** | **61 - 165 ms** |

For a service handling 1,000 requests/second, connecting fresh for each request wastes 2.5 - 8 seconds of cumulative latency per second in the best case. With pooling, this connection setup happens once and is reused hundreds or thousands of times.

### 1.2 Resource Waste Without Pooling

Each TCP connection consumes:

- **File descriptors** — one per socket on the server side
- **Kernel memory** — TCP socket buffers (~8 KB minimum per connection on Linux)
- **Database backend memory** — PostgreSQL forks a new process per connection, each consuming ~5-10 MB of RSS
- **TLS session state** — symmetric key material, session tickets

Without pooling, a burst of 500 concurrent requests means 500 simultaneous connections being opened, handshaked, authenticated, used for one query, then torn down. Most of those resources are wasted on setup and teardown rather than actual work.

### 1.3 Server and Database Connection Limits

Servers impose hard limits:

| Resource | Typical Default | Hard Ceiling |
|----------|----------------|--------------|
| PostgreSQL `max_connections` | 100 | ~500 before performance degrades |
| MySQL `max_connections` | 151 | ~1000 practical |
| Linux file descriptors (per process) | 1024 | 1,048,576 (`ulimit -n`) |
| Ephemeral port range | 16384 ports | 28232 (`net.ipv4.ip_local_port_range`) |

When you have 10 microservices, each with 3 replicas, each using a pool of 20 connections: that is 600 connections to one database. Multiply by environments and you see why connection management matters.

---

## 2. HTTP Keep-Alive

### 2.1 Persistent Connections in HTTP/1.1

Before HTTP/1.1, every request opened a new TCP connection:

```
Request 1:  SYN → SYN-ACK → ACK → GET / → 200 OK → FIN
Request 2:  SYN → SYN-ACK → ACK → GET /api → 200 OK → FIN
Request 3:  SYN → SYN-ACK → ACK → GET /img → 200 OK → FIN
```

HTTP/1.1 made `Connection: keep-alive` the default. The connection stays open after the response:

```
SYN → SYN-ACK → ACK
  → GET /       → 200 OK
  → GET /api    → 200 OK
  → GET /img    → 200 OK
  → (idle timeout reached) → FIN
```

To opt out, the client or server sends `Connection: close`. In practice, nearly everything uses keep-alive today.

### 2.2 Max Idle Time and Max Requests

Servers set boundaries on persistent connections:

- **Keep-Alive timeout**: How long a connection stays open with no requests. Nginx defaults to 75 seconds (`keepalive_timeout`). Apache defaults to 5 seconds.
- **Max requests per connection**: After N requests, the server closes the connection. Nginx: `keepalive_requests 1000` (default). This prevents memory leaks and provides a natural rotation point.

```nginx
# Nginx keep-alive tuning
http {
    keepalive_timeout  65;
    keepalive_requests 1000;
}
```

**Mismatch danger**: If your client's idle timeout is longer than the server's, the client tries to reuse a connection the server already closed. This causes "connection reset" errors. Set the client timeout shorter than the server timeout.

### 2.3 How HTTP/2 Multiplexing Changes Everything

HTTP/1.1 keep-alive reuses the connection but requests are still sequential — one response must finish before the next request starts (head-of-line blocking). Browsers worked around this by opening 6 parallel connections per origin.

HTTP/2 multiplexes many concurrent streams over a single TCP connection:

```
Single TCP connection
  ├── Stream 1: GET /         → 200 OK
  ├── Stream 3: GET /api      → 200 OK  (in parallel)
  ├── Stream 5: GET /style.css → 200 OK  (in parallel)
  └── Stream 7: GET /img.avif  → 200 OK  (in parallel)
```

Impact on connection pooling:
- HTTP/1.1: you need multiple connections for parallelism (6 per origin in browsers, configurable in server-to-server)
- HTTP/2: one connection per origin often suffices for high throughput
- Most HTTP/2 clients maintain a single connection per authority and multiplex over it

### 2.4 HTTP/3 and QUIC Connections

HTTP/3 uses QUIC (over UDP), eliminating TCP-level head-of-line blocking. Each stream is independent at the transport layer. Connection migration (via connection IDs) means connections survive network changes — important for mobile clients and cloud failover.

For backend-to-backend calls, HTTP/2 remains dominant. HTTP/3 matters more for edge-to-client communication.

---

## 3. HTTP Client Connection Pools

### 3.1 Node.js http.Agent

Node.js's built-in `http` / `https` modules use an `Agent` to manage connection pooling:

```typescript
import http from 'node:http';

// Default agent — keep-alive is OFF by default in Node < 19
// Node 19+ defaults to keep-alive: true
const agent = new http.Agent({
  keepAlive: true,           // Reuse connections (always set this)
  maxSockets: 50,            // Max concurrent sockets per host:port
  maxFreeSockets: 10,        // Max idle sockets to keep in pool
  maxTotalSockets: Infinity, // Max sockets across all hosts
  timeout: 60_000,           // Socket idle timeout (ms)
  scheduling: 'lifo',        // LIFO reuses warm connections, reduces total open sockets
});

const response = await fetch('https://api.example.com/data', {
  agent, // Node 18 with node-fetch; native fetch in Node 18+ manages its own pool
});
```

Key points:
- `scheduling: 'lifo'` (last-in, first-out) reuses the most recently returned socket, letting least-used sockets time out. Better than the default `'fifo'` for reducing open connections.
- The global `http.globalAgent` has `keepAlive: false` before Node 19. Always create a custom agent or set `http.globalAgent.keepAlive = true`.
- **Native `fetch()` in Node 18+** uses an internal undici pool. You do not pass an `http.Agent` to it.

### 3.2 undici Pool

`undici` is the high-performance HTTP client that powers Node.js's native `fetch`. You can use it directly for more control:

```typescript
import { Pool } from 'undici';

const pool = new Pool('https://api.example.com', {
  connections: 20,          // Max concurrent connections to this origin
  pipelining: 1,            // Requests per connection (1 = no pipelining)
  keepAliveTimeout: 30_000, // Idle connection timeout (ms)
  keepAliveMaxTimeout: 600_000, // Max time a connection stays alive
  connect: {
    rejectUnauthorized: true,
  },
});

const { statusCode, body } = await pool.request({
  path: '/data',
  method: 'GET',
  headers: { 'Accept': 'application/json' },
});

const data = await body.json();

// Graceful shutdown
await pool.close();
```

For multiple origins, use `undici.Agent` (not to be confused with `http.Agent`):

```typescript
import { Agent, setGlobalDispatcher } from 'undici';

const agent = new Agent({
  connections: 10,          // Per-origin connection limit
  keepAliveTimeout: 30_000,
  keepAliveMaxTimeout: 600_000,
});

setGlobalDispatcher(agent); // All fetch() calls use this pool
```

### 3.3 Java HttpClient Connection Pool

Java 11+ `HttpClient` manages its own internal connection pool:

```java
HttpClient client = HttpClient.newBuilder()
    .version(HttpClient.Version.HTTP_2)   // Prefer HTTP/2
    .connectTimeout(Duration.ofSeconds(5))
    .executor(Executors.newFixedThreadPool(10))  // Thread pool for async
    .build();

// The connection pool is internal — no direct configuration
// Connections are reused per-origin automatically
HttpResponse<String> response = client.send(
    HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/data"))
        .GET()
        .build(),
    HttpResponse.BodyHandlers.ofString()
);
```

Java's `HttpClient` reuses connections by default for HTTP/2 (one connection per origin with multiplexing) and HTTP/1.1 (keep-alive). The pool size is not directly configurable — it scales with the executor thread pool.

### 3.4 Spring RestTemplate and WebClient Pool Config

**RestTemplate with Apache HttpClient 5 (blocking):**

```java
@Bean
public RestTemplate restTemplate() {
    PoolingHttpClientConnectionManager connectionManager =
        PoolingHttpClientConnectionManagerBuilder.create()
            .setMaxConnTotal(200)           // Total connections across all routes
            .setMaxConnPerRoute(50)         // Connections per host:port
            .setDefaultConnectionConfig(
                ConnectionConfig.custom()
                    .setConnectTimeout(Timeout.ofSeconds(5))
                    .setSocketTimeout(Timeout.ofSeconds(30))
                    .setTimeToLive(TimeValue.ofMinutes(10)) // Max connection lifetime
                    .build()
            )
            .build();

    CloseableHttpClient httpClient = HttpClients.custom()
        .setConnectionManager(connectionManager)
        .evictIdleConnections(TimeValue.ofSeconds(30))
        .build();

    HttpComponentsClientHttpRequestFactory factory =
        new HttpComponentsClientHttpRequestFactory(httpClient);

    return new RestTemplate(factory);
}
```

**WebClient with Reactor Netty (non-blocking):**

```java
@Bean
public WebClient webClient() {
    ConnectionProvider provider = ConnectionProvider.builder("custom")
        .maxConnections(200)                 // Total pool size
        .maxIdleTime(Duration.ofSeconds(30)) // Idle connection eviction
        .maxLifeTime(Duration.ofMinutes(10)) // Force connection rotation
        .pendingAcquireTimeout(Duration.ofSeconds(5)) // Wait for available connection
        .evictInBackground(Duration.ofSeconds(30))     // Background eviction interval
        .metrics(true)                       // Enable Micrometer metrics
        .build();

    HttpClient httpClient = HttpClient.create(provider)
        .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)
        .responseTimeout(Duration.ofSeconds(30));

    return WebClient.builder()
        .clientConnector(new ReactorClientHttpConnector(httpClient))
        .build();
}
```

---

## 4. Database Connection Pools

### 4.1 Why Databases Limit Connections

PostgreSQL forks a new backend process for every client connection. Each process:

- Allocates its own memory for query parsing, planning, and execution
- Consumes ~5-10 MB of RSS (resident set size)
- Competes for shared resources: shared_buffers, WAL writer, autovacuum
- Requires OS-level context switching between processes

At 500 connections, that is 2.5-5 GB of RAM just for connection overhead, plus massive context-switch overhead. Beyond ~300 connections, PostgreSQL performance typically degrades even on beefy hardware. The `max_connections` default of 100 is intentionally conservative.

MySQL uses threads instead of processes, so it handles more connections (1000+), but still suffers from contention on internal mutexes at high connection counts.

**The solution**: A small pool of database connections (10-50) shared across your application's concurrent requests, dramatically reducing the load on the database server.

### 4.2 Pool Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Creating : Pool needs connection
    Creating --> Idle : Connection established & validated
    Idle --> Borrowed : Application requests connection
    Borrowed --> Idle : Application returns connection
    Idle --> Validating : Validation check triggered
    Validating --> Idle : Validation passed
    Validating --> Evicted : Validation failed (stale/broken)
    Idle --> Evicted : Max idle time exceeded
    Idle --> Evicted : Max lifetime exceeded
    Borrowed --> Leaked : Not returned within leak detection threshold
    Leaked --> Evicted : Forcibly reclaimed
    Evicted --> [*] : Connection closed
```

The pool lifecycle:

1. **Creating**: A new TCP connection is opened to the database, TLS is negotiated, authentication completes
2. **Idle**: Connection sits in the pool, ready to be borrowed
3. **Borrowed**: Application code uses the connection for queries
4. **Returned**: Connection goes back to the idle pool after use
5. **Validating**: Pool runs a lightweight check (`SELECT 1`) to confirm the connection is still alive
6. **Evicted**: Connection is closed because it exceeded idle timeout, max lifetime, or failed validation

### 4.3 Idle Timeout and Max Lifetime

Two distinct timeouts control connection rotation:

| Setting | Purpose | Typical Value |
|---------|---------|---------------|
| **Idle timeout** | Close connections that have been sitting unused | 10 min (HikariCP default: 600,000 ms) |
| **Max lifetime** | Close connections that have been alive too long, regardless of usage | 30 min (HikariCP default: 1,800,000 ms) |

**Why max lifetime matters**: Connections that stay alive forever can accumulate stale state, consume server-side resources, and — critically — prevent failover. If your database switches to a replica (RDS failover, PgBouncer restart), old connections still point at the old endpoint. Max lifetime forces rotation so connections pick up DNS changes.

**Critical rule**: Set `maxLifetime` in your pool to be at least 30 seconds shorter than any server-side timeout (`wait_timeout` in MySQL, `idle_in_transaction_session_timeout` in PostgreSQL, or load balancer idle timeout). Otherwise the server closes the connection, and the pool hands your code a dead socket.

---

## 5. Pool Sizing with Little's Law

### 5.1 The Formula: L = lambda * W

Little's Law states:

```
L = λ × W
```

Where:
- **L** = average number of items in the system (connections in use)
- **λ** (lambda) = arrival rate (requests per second)
- **W** = average time each item spends in the system (query duration)

For connection pools, this translates to:

```
Pool Size Needed = Requests/sec × Average Query Duration (sec)
```

### 5.2 Practical Calculation Examples

**Example 1: Typical web service**

```
λ = 50 requests/sec
W = 20 ms average query time = 0.020 sec

L = 50 × 0.020 = 1 connection

Minimum pool size: 1 (but add headroom for bursts → 3-5)
```

**Example 2: Higher throughput service**

```
λ = 500 requests/sec
W = 10 ms average query time = 0.010 sec

L = 500 × 0.010 = 5 connections

With 2x headroom for P99 latency spikes: 10 connections
```

**Example 3: Slow queries present**

```
λ = 200 requests/sec
W_fast = 5 ms (90% of queries)  → L_fast = 200 × 0.9 × 0.005 = 0.9
W_slow = 200 ms (10% of queries) → L_slow = 200 × 0.1 × 0.200 = 4.0

Total L = 0.9 + 4.0 = 4.9 → minimum ~5 connections

With headroom: 10 connections
```

The takeaway: you almost always need fewer connections than you think. A pool of 10 connections can handle hundreds of requests per second if queries are fast.

### 5.3 Why Oversizing Hurts

Setting `maximumPoolSize: 100` "just to be safe" causes real problems:

1. **Database overload**: If all 100 connections fire queries simultaneously, the database's CPU, memory, and I/O are overwhelmed. Queries that took 5 ms now take 500 ms due to contention.
2. **Lock contention**: More concurrent connections means more concurrent transactions competing for row-level locks, table locks, and internal database mutexes.
3. **Memory waste**: 100 idle connections × 10 MB each = 1 GB of database memory sitting unused.
4. **Cascading failure**: Under load, all 100 connections may become active simultaneously, overwhelming the database and causing timeouts across all services.

A smaller pool with a queue actually produces **lower latency** under load — requests wait briefly for a connection rather than all hammering the database at once.

### 5.4 HikariCP's Formula Recommendation

The HikariCP wiki recommends this formula as a starting point:

```
connections = ((core_count * 2) + effective_spindle_count)
```

For a 4-core database server with 1 SSD (treat SSD as 1 spindle):

```
connections = (4 * 2) + 1 = 9
```

This is the total pool size for the **database server**, not per application instance. If you have 3 application replicas sharing one database, each gets `9 / 3 = 3` connections.

In practice, start with 10 and benchmark. You can almost always serve high throughput with fewer connections than expected.

---

## 6. HikariCP Deep Dive

HikariCP is the default connection pool in Spring Boot. It is fast (zero-overhead connection proxy using bytecode generation), correct (properly handles connection validation), and well-monitored.

### 6.1 Core Configuration Properties

| Property | Default | Description |
|----------|---------|-------------|
| `minimumIdle` | same as `maximumPoolSize` | Minimum idle connections to maintain. HikariCP recommends leaving this equal to `maximumPoolSize` for best performance (fixed-size pool). |
| `maximumPoolSize` | 10 | Maximum number of connections (active + idle). |
| `connectionTimeout` | 30,000 ms | Max time to wait for a connection from the pool. If exceeded, throws `SQLException`. |
| `idleTimeout` | 600,000 ms (10 min) | Max time a connection can sit idle. Only applies when `minimumIdle` < `maximumPoolSize`. |
| `maxLifetime` | 1,800,000 ms (30 min) | Max lifetime of a connection. Set this 30 seconds shorter than any database or infrastructure timeout. |
| `keepaliveTime` | 0 (disabled) | Interval for keep-alive pings to prevent idle connections from being killed by firewalls. Set to ~30,000 ms in cloud environments. |
| `validationTimeout` | 5,000 ms | Timeout for connection validation (alive check). |
| `leakDetectionThreshold` | 0 (disabled) | If a connection is not returned within this time (ms), log a warning with stack trace. Set to 2,000 ms in development. |

### 6.2 Leak Detection

A connection leak happens when application code borrows a connection and never returns it (e.g., missing `finally` block, exception before `close()`). The pool eventually runs out.

HikariCP's leak detection logs the stack trace of the code that borrowed the connection:

```
[WARN] HikariPool-1 - Connection leak detection triggered for conn0, stack trace follows
java.lang.Exception: Apparent connection leak detected
    at com.example.UserRepository.findByEmail(UserRepository.java:42)
    at com.example.UserService.getUser(UserService.java:18)
```

Enable in development and staging:

```yaml
spring:
  datasource:
    hikari:
      leak-detection-threshold: 2000  # 2 seconds
```

### 6.3 Spring Boot Auto-Configuration

Spring Boot auto-configures HikariCP when `spring-boot-starter-jdbc` or `spring-boot-starter-data-jpa` is on the classpath. All HikariCP properties are configurable via `application.yml`:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 10
      minimum-idle: 10            # Fixed-size pool (recommended)
      connection-timeout: 5000    # Fail fast rather than wait 30s
      idle-timeout: 600000        # 10 minutes
      max-lifetime: 1770000       # 29.5 minutes (shorter than PG default)
      keepalive-time: 30000       # Prevent firewall/LB idle kill
      leak-detection-threshold: 5000  # 5s in non-prod
      pool-name: MyServicePool
      connection-test-query: SELECT 1  # Not needed for JDBC4 drivers (uses isValid())
```

### 6.4 Configuration Example

Full Spring Boot HikariCP configuration for a production PostgreSQL service:

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @ConfigurationProperties("spring.datasource.hikari")
    public HikariConfig hikariConfig() {
        HikariConfig config = new HikariConfig();
        config.setPoolName("ProductionPool");
        config.setMaximumPoolSize(10);
        config.setMinimumIdle(10);
        config.setConnectionTimeout(5_000);
        config.setIdleTimeout(600_000);
        config.setMaxLifetime(1_770_000);   // 29.5 min
        config.setKeepaliveTime(30_000);
        config.setLeakDetectionThreshold(0); // Disabled in prod
        config.addDataSourceProperty("cachePrepStmts", "true");
        config.addDataSourceProperty("prepStmtCacheSize", "250");
        config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");
        return config;
    }

    @Bean
    public DataSource dataSource(HikariConfig hikariConfig) {
        return new HikariDataSource(hikariConfig);
    }
}
```

---

## 7. Node.js Database Pools

### 7.1 pg (node-postgres) Pool Config

`pg` (node-postgres) provides a built-in `Pool` class:

```typescript
import { Pool } from 'pg';

const pool = new Pool({
  host: process.env.DB_HOST,
  port: 5432,
  database: 'mydb',
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  ssl: { rejectUnauthorized: true },

  // Pool configuration
  max: 10,                      // Maximum connections in pool
  min: 2,                       // Minimum idle connections
  idleTimeoutMillis: 30_000,    // Close idle connections after 30s
  connectionTimeoutMillis: 5_000, // Fail if no connection within 5s
  maxUses: 7500,                // Close connection after N uses (rotation)
  allowExitOnIdle: true,        // Let process exit if pool is idle
});

// Usage — always use pool, never hold a client reference
const result = await pool.query('SELECT * FROM users WHERE id = $1', [userId]);

// For transactions, borrow and return explicitly
const client = await pool.connect();
try {
  await client.query('BEGIN');
  await client.query('INSERT INTO orders (user_id) VALUES ($1)', [userId]);
  await client.query('UPDATE inventory SET stock = stock - 1 WHERE id = $1', [itemId]);
  await client.query('COMMIT');
} catch (err) {
  await client.query('ROLLBACK');
  throw err;
} finally {
  client.release(); // ALWAYS release — this is where leaks happen
}

// Listen for pool events
pool.on('error', (err) => {
  console.error('Unexpected pool error:', err);
});

pool.on('connect', () => {
  console.log('New connection created');
});

pool.on('remove', () => {
  console.log('Connection removed from pool');
});
```

### 7.2 Prisma Connection Pool

Prisma manages its own connection pool internally:

```typescript
// prisma/schema.prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```

Pool configuration via the connection string:

```bash
# Connection string with pool parameters
DATABASE_URL="postgresql://user:pass@host:5432/mydb?connection_limit=10&pool_timeout=5&connect_timeout=5"
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `connection_limit` | `num_cpus * 2 + 1` | Max connections |
| `pool_timeout` | 10 (seconds) | Time waiting for a connection |
| `connect_timeout` | 5 (seconds) | Time to establish a TCP connection |

```typescript
import { PrismaClient } from '@prisma/client';

// Create a single PrismaClient instance (singleton)
const prisma = new PrismaClient({
  datasources: {
    db: {
      url: process.env.DATABASE_URL,
    },
  },
  log: ['query', 'warn', 'error'],
});

// Prisma handles connection pooling internally
const user = await prisma.user.findUnique({ where: { id: userId } });

// Graceful shutdown
process.on('SIGTERM', async () => {
  await prisma.$disconnect();
  process.exit(0);
});
```

**Prisma Accelerate**: For serverless environments where connection pooling is impractical (each Lambda invocation is a fresh process), Prisma offers Accelerate — an external connection pooler similar in concept to PgBouncer but managed as a service.

### 7.3 Drizzle ORM Pool

Drizzle ORM uses the underlying driver's pool:

```typescript
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 10,
  idleTimeoutMillis: 30_000,
  connectionTimeoutMillis: 5_000,
});

const db = drizzle(pool);

// Drizzle uses the pg pool transparently
const users = await db.select().from(usersTable).where(eq(usersTable.id, userId));
```

### 7.4 Connection String Parameters

PostgreSQL connection strings accept pool-related parameters:

```
postgresql://user:pass@host:5432/db
  ?sslmode=require
  &connect_timeout=5
  &application_name=my-service
  &options=-c%20statement_timeout%3D30000
```

Note: Pool size is NOT a connection string parameter for PostgreSQL itself. It is a client library parameter (`max` in pg, `connection_limit` in Prisma). The connection string configures the individual connection; the pool wraps multiple connections.

---

## 8. PgBouncer

### 8.1 What PgBouncer Does

PgBouncer sits between your application and PostgreSQL as a lightweight connection proxy. It maintains a small pool of real PostgreSQL connections and multiplexes many client connections onto them:

```
Application (100 connections) → PgBouncer (10 real PG connections) → PostgreSQL
```

PgBouncer itself uses very little memory (~2 KB per client connection vs ~10 MB per real PostgreSQL backend). It is a single-threaded, event-driven process written in C.

### 8.2 Pooling Modes

| Mode | How it works | Pros | Cons |
|------|-------------|------|------|
| **Session** | Client gets a dedicated PG connection for the entire session | Supports all PG features | Lowest multiplexing benefit |
| **Transaction** | Client gets a PG connection only during a transaction; returned to pool between transactions | Good multiplexing ratio | Cannot use SET, LISTEN/NOTIFY, prepared statements across transactions |
| **Statement** | Connection returned after every statement | Maximum multiplexing | Cannot use multi-statement transactions at all |

**Transaction mode** is the most common in production. It provides the best balance of multiplexing and compatibility. You lose:

- `SET` commands (use `SET LOCAL` inside transactions instead)
- Session-level prepared statements (use `DEALLOCATE ALL` or disable server-side prepares)
- `LISTEN`/`NOTIFY` (use session mode for these connections)
- Advisory locks that span transactions

### 8.3 When to Use PgBouncer

PgBouncer is most valuable when:

1. **Serverless / Lambda functions**: Each invocation opens a new connection. Without PgBouncer, 1000 concurrent Lambdas = 1000 PostgreSQL backends. With PgBouncer in transaction mode, those 1000 clients share 20-50 real connections.

2. **Connection limit pressure**: Your database `max_connections` is the bottleneck (many microservices, many replicas).

3. **Short-lived connections**: Applications that connect and disconnect frequently benefit from PgBouncer's fast client handshake (it handles auth itself, no PG round-trip).

4. **Multi-tenant architectures**: Many tenants, each with their own connection credentials, but few simultaneous queries.

### 8.4 Interaction with Application Pools

When PgBouncer is in the stack, you still configure a pool in your application — but with different settings:

```
Application Pool (max: 5) → PgBouncer (max_client_conn: 1000, default_pool_size: 20) → PostgreSQL (max_connections: 100)
```

Guidelines for using application pools with PgBouncer:

- **Reduce application pool size**: Your app connects to PgBouncer, not PostgreSQL. PgBouncer handles the multiplexing, so each app instance needs fewer connections (3-5 instead of 10-20).
- **Disable application-level connection validation**: PgBouncer handles stale connection detection. Running `SELECT 1` as a validation query just adds noise.
- **Keep application pool idle timeout short**: Let PgBouncer manage the long-lived connections. Application-side idle connections to PgBouncer are cheap (~2 KB each).
- **Disable server-side prepared statements** in transaction mode: Set `preparedStatementCacheQueries=0` in your JDBC connection string, or `statement_cache_size=0` in PgBouncer config.

---

## 9. Connection Pool Monitoring

### 9.1 Metrics to Watch

| Metric | What it tells you | Alert when |
|--------|-------------------|------------|
| **Active connections** | Currently borrowed and executing queries | Consistently near `maximumPoolSize` |
| **Idle connections** | Sitting in pool, ready to be borrowed | Consistently 0 (pool is saturated) |
| **Pending requests** | Threads/requests waiting for a connection | Any sustained value > 0 |
| **Connection timeout count** | Failed to get a connection within `connectionTimeout` | Any occurrence |
| **Connection creation time** | Time to establish a new connection | > 1 second (network or auth issue) |
| **Connection usage time** | How long connections are borrowed | P99 > expected query time (possible leak) |
| **Total connections** | Active + idle | Exceeds expected `maximumPoolSize` |

### 9.2 HikariCP Metrics with Micrometer

HikariCP exposes metrics to Micrometer automatically when both are on the classpath:

```yaml
# application.yml
spring:
  datasource:
    hikari:
      pool-name: MyServicePool
      register-mbeans: true       # Also expose via JMX

management:
  metrics:
    enable:
      hikaricp: true
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
```

Available Micrometer metrics:

```
hikaricp_connections_active{pool="MyServicePool"}       # Currently borrowed
hikaricp_connections_idle{pool="MyServicePool"}         # In pool, ready
hikaricp_connections_pending{pool="MyServicePool"}      # Waiting for connection
hikaricp_connections_max{pool="MyServicePool"}          # maximumPoolSize
hikaricp_connections_min{pool="MyServicePool"}          # minimumIdle
hikaricp_connections_timeout_total{pool="MyServicePool"} # Connection timeout count
hikaricp_connections_creation_seconds_max{pool="MyServicePool"}  # Slowest creation
hikaricp_connections_usage_seconds{pool="MyServicePool"}  # Borrowed duration histogram
hikaricp_connections_acquire_seconds{pool="MyServicePool"} # Wait time histogram
```

Grafana dashboard query examples:

```promql
# Pool saturation (approaching 1.0 = trouble)
hikaricp_connections_active{pool="MyServicePool"}
  / hikaricp_connections_max{pool="MyServicePool"}

# Connection wait time P99
histogram_quantile(0.99, rate(hikaricp_connections_acquire_seconds_bucket[5m]))

# Timeout rate
rate(hikaricp_connections_timeout_total{pool="MyServicePool"}[5m])
```

### 9.3 Node.js Pool Events

For `pg` (node-postgres), monitor the pool via events and periodic inspection:

```typescript
import { Pool } from 'pg';

const pool = new Pool({ max: 10 });

// Pool metrics logging
const logPoolMetrics = () => {
  console.log({
    totalCount: pool.totalCount,     // Total connections (active + idle)
    idleCount: pool.idleCount,       // Idle connections
    waitingCount: pool.waitingCount, // Queued requests waiting for connection
  });
};

setInterval(logPoolMetrics, 10_000); // Log every 10 seconds

// Event-based monitoring
pool.on('connect', () => {
  console.log('Pool: new connection created');
});

pool.on('acquire', () => {
  console.log('Pool: connection acquired (borrowed)');
});

pool.on('remove', () => {
  console.log('Pool: connection removed (evicted)');
});

pool.on('error', (err) => {
  console.error('Pool: idle client error', err.message);
});
```

For Prometheus-style metrics, use `prom-client`:

```typescript
import { Gauge } from 'prom-client';

const poolTotal = new Gauge({ name: 'pg_pool_total', help: 'Total connections' });
const poolIdle = new Gauge({ name: 'pg_pool_idle', help: 'Idle connections' });
const poolWaiting = new Gauge({ name: 'pg_pool_waiting', help: 'Waiting requests' });

setInterval(() => {
  poolTotal.set(pool.totalCount);
  poolIdle.set(pool.idleCount);
  poolWaiting.set(pool.waitingCount);
}, 5_000);
```

### 9.4 Alerting Thresholds

| Condition | Severity | Action |
|-----------|----------|--------|
| `pending > 0` for > 30 seconds | **Warning** | Pool may be undersized or queries are slow |
| `timeout_total` increasing | **Critical** | Requests are failing; investigate slow queries or pool size |
| `active / max > 0.8` sustained | **Warning** | Pool nearing saturation |
| `active / max = 1.0` sustained | **Critical** | Pool exhausted; all connections in use |
| `creation_seconds > 2s` | **Warning** | Network issue or database under load |
| `usage_seconds P99 > 5s` | **Warning** | Possible connection leak or very slow queries |

---

## 10. Common Pitfalls

### 10.1 Pool Exhaustion (Leaked Connections)

The most common pool problem: code borrows a connection but never returns it.

**Java — the classic leak**:

```java
// WRONG — connection leaks if query throws
Connection conn = dataSource.getConnection();
PreparedStatement ps = conn.prepareStatement("SELECT ...");
ResultSet rs = ps.executeQuery();
// exception here → conn never closed
conn.close();

// CORRECT — try-with-resources guarantees close
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement("SELECT ...");
     ResultSet rs = ps.executeQuery()) {
    // process results
}
// conn.close() called automatically, returns to pool
```

**Node.js — the classic leak**:

```typescript
// WRONG — client not released if query throws
const client = await pool.connect();
await client.query('BEGIN');
await client.query('INSERT ...'); // throws → client never released
await client.query('COMMIT');
client.release();

// CORRECT — always release in finally
const client = await pool.connect();
try {
  await client.query('BEGIN');
  await client.query('INSERT ...');
  await client.query('COMMIT');
} catch (err) {
  await client.query('ROLLBACK');
  throw err;
} finally {
  client.release();
}

// BEST — use pool.query() for single queries (auto-release)
await pool.query('SELECT * FROM users WHERE id = $1', [userId]);
```

### 10.2 Connection Storms on Startup

When an application starts up (or restarts after a crash), it may try to open all pool connections simultaneously. If 20 instances restart at once (rolling deployment), the database sees 200 simultaneous connection attempts.

**Mitigations**:

- **HikariCP**: Set `initializationFailTimeout` to a positive value and let the pool fill lazily (connections created as needed, not all at startup).
- **Staggered startup**: Add a small random delay before each instance starts connecting.
- **PgBouncer**: Absorbs connection storms by queuing clients while real connections are established.

### 10.3 Stale Connections After Failover

When a database fails over (RDS Multi-AZ failover, PG primary promotion), existing TCP connections are still pointing at the old server. The TCP layer does not immediately know the server is gone — it takes up to the TCP keepalive timeout (often 2+ hours by default) to detect a dead peer.

**Mitigations**:

- **Set `maxLifetime`** to 5-10 minutes so connections naturally rotate to the new endpoint.
- **Enable `keepaliveTime`** in HikariCP (30 seconds) to detect dead connections faster.
- **Set TCP keepalive at the OS or JDBC level**: `?tcpKeepAlive=true` in the connection string.
- **Connection validation on borrow**: HikariCP does this automatically using `Connection.isValid()`.

### 10.4 DNS Caching Preventing Failover

After a database failover, the DNS record for the endpoint changes. But your application may have cached the old IP address:

- **JVM DNS cache**: The JVM caches DNS lookups indefinitely by default (when a security manager is present) or for 30 seconds. Set `networkaddress.cache.ttl=30` in `java.security` or via system property.
- **OS DNS cache**: The OS resolver cache may hold stale entries. Use `systemd-resolved` TTL settings or `dscacheutil -flushcache` (macOS) for debugging.
- **Node.js**: Node.js does not cache DNS by default (each `dns.lookup()` goes to the OS). But connection pools hold open sockets that were resolved before the failover.

**Combined mitigation**: `maxLifetime` + short DNS TTL + TCP keepalive covers all cases.

### 10.5 Too Many Pools

In a microservice architecture, total connection count grows multiplicatively:

```
Total Connections = services × replicas × pool_size

Example:
  15 services × 3 replicas × 20 pool_size = 900 connections

  PostgreSQL max_connections: 200 → PROBLEM
```

**Solutions**:

1. **Right-size each pool**: Use Little's Law per service. Most services need 3-10, not 20.
2. **Use PgBouncer**: External pooler multiplexes 900 client connections onto 50-100 real connections.
3. **Read replicas**: Route read-only traffic to replicas, spreading connection load.
4. **Connection-per-service auditing**: Track which service uses how many connections and enforce budgets.

## Pooling Solutions Comparison

| Feature | HikariCP (Java) | pg Pool (Node.js) | Prisma | PgBouncer |
|---------|-----------------|-------------------|--------|-----------|
| **Type** | In-process | In-process | In-process | External proxy |
| **Protocol** | JDBC | PostgreSQL wire | PostgreSQL wire | PostgreSQL wire |
| **Connection validation** | `isValid()` (JDBC4) | Manual or via events | Automatic | Automatic |
| **Leak detection** | Built-in threshold | Manual (check `waitingCount`) | No | No |
| **Metrics** | Micrometer, JMX | Events, manual polling | Prisma Metrics (preview) | SHOW STATS, SHOW POOLS |
| **Prepared statements** | Supported | Supported | Supported | Session mode only (or disable) |
| **Max pool size** | Configurable | `max` option | `connection_limit` | `default_pool_size` |
| **Best for** | Spring Boot / JVM | Direct Node.js DB access | Prisma ORM users | Multi-service, serverless |

---

## Related

- [Async I/O Models](async-io-models.md) — reactor patterns and event-driven networking that underpin connection pools
- [Network Debugging](network-debugging.md) — diagnosing connection issues with tcpdump, ss, and Wireshark
- [TCP Deep Dive](../transport/tcp-deep-dive.md) — handshake costs, TIME_WAIT, and keepalive mechanics
- [Database Connection Management](../../database/operations/connection-management.md) — PostgreSQL server-side configuration

---

## References

1. **HikariCP GitHub & Wiki** — Configuration reference and pool sizing guide. https://github.com/brettwooldridge/HikariCP/wiki
2. **PostgreSQL Documentation — Connection Pooling** — Server configuration for `max_connections` and related settings. https://www.postgresql.org/docs/current/runtime-config-connection.html
3. **PgBouncer Documentation** — Pooling modes, configuration, and limitations. https://www.pgbouncer.org/config.html
4. **node-postgres (pg) Documentation** — Pool API, events, and configuration. https://node-postgres.com/features/pooling
5. **Prisma Connection Management** — Pool configuration and Prisma Accelerate. https://www.prisma.io/docs/orm/prisma-client/setup-and-configuration/databases-connections/connection-pool
6. **Little's Law (Wikipedia)** — Foundational queuing theory behind pool sizing. https://en.wikipedia.org/wiki/Little%27s_law
7. **Spring Boot Data Source Configuration** — Auto-configuration of HikariCP. https://docs.spring.io/spring-boot/docs/current/reference/html/data.html
8. **undici Documentation** — Node.js high-performance HTTP client pool API. https://undici.nodejs.org/
