---
title: "Helidon SE — Functional, No-Magic Microservices on Virtual Threads"
date: 2026-04-19
updated: 2026-04-19
tags: [helidon, helidon-se, microservices, java, virtual-threads]
---

# Helidon SE — Functional, No-Magic Microservices on Virtual Threads

**Date:** 2026-04-19 | **Updated:** 2026-04-19
**Tags:** `helidon` `helidon-se` `microservices` `java` `virtual-threads`

## Table of Contents

- [Summary](#summary)
- [Philosophy — No Magic, Just Java](#philosophy--no-magic-just-java)
- [WebServer and Routing](#webserver-and-routing)
- [Config API](#config-api)
- [JSON Handling](#json-handling)
- [Observability — Health, Metrics, Tracing](#observability--health-metrics-tracing)
- [Security](#security)
- [WebClient — Outbound HTTP](#webclient--outbound-http)
- [Testing](#testing)
- [SE vs Spring Boot — Side by Side](#se-vs-spring-boot--side-by-side)
- [Related](#related)
- [References](#references)

---

## Summary

Helidon SE is the functional, annotation-free programming model in [Helidon](https://helidon.io/). There is no dependency injection container, no component scanning, no annotation processing — you wire everything explicitly using builders and functional interfaces. Since Helidon 4, SE runs on virtual threads via the [Nima](nima-virtual-threads-architecture.md) web server, so blocking I/O is efficient without reactive operator chains. If you come from TypeScript, SE feels closer to Express.js than to Spring Boot: you define routes with lambdas, compose middleware manually, and see every line of startup code.

---

## Philosophy — No Magic, Just Java

SE's design principles:

1. **Explicit over implicit** — no classpath scanning, no auto-configuration, no proxy generation
2. **Builders over annotations** — configure via `WebServer.builder()`, `Config.create()`, `Security.builder()`
3. **Functional composition** — routes are lambdas or `HttpService` implementations composed in a routing builder
4. **No DI container** — if you want DI, bring your own (Dagger, Guice) or just pass references through constructors
5. **Small dependency tree** — the core `helidon-webserver` module has minimal transitive dependencies

This contrasts sharply with Spring Boot where `@SpringBootApplication` triggers component scanning, auto-configuration, proxy creation, and a BeanFactory. In SE, if you didn't write it, it doesn't happen.

---

## WebServer and Routing

### Minimal Server

```java
import io.helidon.webserver.WebServer;
import io.helidon.webserver.http.HttpRouting;

public class Main {
    public static void main(String[] args) {
        WebServer server = WebServer.builder()
                .routing(Main::routing)
                .port(8080)
                .build()
                .start();

        System.out.println("Server started at http://localhost:" + server.port());
    }

    static void routing(HttpRouting.Builder routing) {
        routing.get("/hello", (req, res) -> res.send("Hello, Helidon SE!"))
               .register("/greet", new GreetService())
               .register("/pictures", new PictureService());
    }
}
```

No `@RestController`, no `@GetMapping` — just a builder with lambdas and service registrations.

### HttpService — Grouping Routes

For anything beyond trivial endpoints, implement `HttpService`:

```java
import io.helidon.webserver.http.HttpRules;
import io.helidon.webserver.http.HttpService;
import io.helidon.webserver.http.ServerRequest;
import io.helidon.webserver.http.ServerResponse;

public class GreetService implements HttpService {

    @Override
    public void routing(HttpRules rules) {
        rules.get("/", this::defaultGreeting)
             .get("/{name}", this::namedGreeting)
             .put("/", this::updateGreeting);
    }

    private void defaultGreeting(ServerRequest req, ServerResponse res) {
        res.send("{\"message\": \"Hello, World!\"}");
    }

    private void namedGreeting(ServerRequest req, ServerResponse res) {
        String name = req.path().pathParameters().get("name");
        res.send("{\"message\": \"Hello, " + name + "!\"}");
    }

    private void updateGreeting(ServerRequest req, ServerResponse res) {
        // Read request body, update state, respond
        String body = req.content().as(String.class);
        res.status(204).send();
    }
}
```

Services registered at a path prefix via `routing.register("/greet", new GreetService())` get their routes scoped under that prefix — `/greet/`, `/greet/{name}`, etc.

### Request and Response

- **Path parameters**: `req.path().pathParameters().get("name")`
- **Query parameters**: `req.query().get("page")`
- **Headers**: `req.headers().get(HeaderNames.CONTENT_TYPE)`
- **Body**: `req.content().as(MyPojo.class)` — uses registered media support
- **Response**: `res.status(200).send(entity)` — sends and commits the response

### Filters (Middleware)

```java
routing.addFilter((chain, req, res) -> {
    long start = System.nanoTime();
    chain.proceed();   // call next handler
    long elapsed = System.nanoTime() - start;
    System.out.println(req.path() + " took " + elapsed / 1_000_000 + "ms");
});
```

Filters wrap the handler chain — similar to Express middleware or Servlet `Filter`.

---

## Config API

Helidon SE's [Config API](https://helidon.io/docs/v4/se/config/introduction) loads configuration from multiple sources with a priority hierarchy:

1. System properties (highest)
2. Environment variables
3. `application.yaml` (or `.conf`, `.json`, `.properties`)
4. Default values (lowest)

### application.yaml

```yaml
server:
  port: 8080

app:
  greeting: "Hello"

db:
  url: jdbc:postgresql://localhost:5432/mydb
  username: app
  password: ${DB_PASSWORD}
```

### Reading Config in Code

```java
Config config = Config.create();  // loads from default sources

int port = config.get("server.port").asInt().orElse(8080);
String greeting = config.get("app.greeting").asString().orElse("Hello");

// Map to a POJO
DbConfig dbConfig = config.get("db").as(DbConfig.class).orElseThrow();
```

### Config Profiles

```yaml
# application-dev.yaml
server:
  port: 9090
app:
  greeting: "Dev Hello"
```

Activate with `-Dconfig.profile=dev` or `HELIDON_CONFIG_PROFILE=dev`.

Compared to Spring's `@ConfigurationProperties`, Helidon Config is more explicit — you pull values rather than having them injected. There is no `@Value` annotation in SE.

---

## JSON Handling

Helidon SE supports [JSON-P](https://jakarta.ee/specifications/jsonp/) (Jakarta JSON Processing) and [JSON-B](https://jakarta.ee/specifications/jsonb/) (Jakarta JSON Binding) out of the box. Jackson support is available as an additional module.

### JSON-P (Low-Level)

```java
import jakarta.json.Json;
import jakarta.json.JsonObject;

JsonObject json = Json.createObjectBuilder()
        .add("message", greeting)
        .add("timestamp", Instant.now().toString())
        .build();
res.send(json);
```

### JSON-B (Object Mapping)

```java
// With helidon-media-jsonb on the classpath, POJOs serialize automatically
public record GreetResponse(String message, Instant timestamp) {}

res.send(new GreetResponse("Hello", Instant.now()));
```

Register media support when building the server:

```java
WebServer.builder()
        .routing(Main::routing)
        .addMediaSupport(JsonbSupport.create())  // or JsonpSupport, JacksonSupport
        .build()
        .start();
```

---

## Observability — Health, Metrics, Tracing

Helidon SE bundles observability features that register as server features — no Spring Actuator equivalent needed.

### Configuration-Driven Observability

```yaml
server:
  features:
    observe:
      observers:
        health:
          details: true
        metrics:
          key-performance-indicators:
            extended: true
```

This exposes:

- `GET /observe/health` — liveness, readiness, startup checks
- `GET /observe/metrics` — Prometheus-format metrics
- Distributed tracing via [OpenTelemetry](https://opentelemetry.io/) (OTLP exporter)

### Custom Health Check

```java
HealthObserver.builder()
        .addCheck(HealthChecks.healthChecks())  // built-in (deadlock, disk, heap)
        .addCheck(() -> HealthCheckResponse.builder()
                .status(databaseIsReachable())
                .detail("db", "postgresql")
                .build())
        .build();
```

### Custom Metrics

```java
import io.helidon.metrics.api.MeterRegistry;
import io.helidon.metrics.api.Counter;

MeterRegistry registry = MeterRegistry.create();
Counter requestCounter = registry.getOrCreate(Counter.builder("requests.total"));
requestCounter.increment();
```

### Tracing

Helidon 4.4.x integrates [OpenTelemetry](https://opentelemetry.io/) with OTLP export for traces, metrics, and logs. Configure via `application.yaml`:

```yaml
tracing:
  service: my-service
  protocol: otlp
  host: localhost
  port: 4317
```

---

## Security

Helidon SE has a [built-in security module](https://helidon.io/docs/v4/se/security/introduction) with a provider-based architecture:

```java
Security security = Security.builder()
        .addProvider(HttpBasicAuthProvider.builder()
                .realm("my-realm")
                .userStore(myUserStore))
        .addProvider(OidcProvider.create(config.get("security.oidc")))
        .build();

// Register as a routing filter
routing.register(WebSecurity.create(security));
```

Supported providers:

| Provider | Protocol |
|----------|----------|
| `HttpBasicAuthProvider` | HTTP Basic |
| `HttpDigestAuthProvider` | HTTP Digest |
| `OidcProvider` | OpenID Connect / OAuth 2.0 |
| `JwtProvider` | JWT validation |
| `HeaderAtnProvider` | Header-based (proxy auth) |
| `GoogleTokenProvider` | Google Sign-In |

This is more explicit than Spring Security's filter chain auto-configuration — you choose and wire each provider.

---

## WebClient — Outbound HTTP

Helidon SE includes a blocking HTTP client for service-to-service calls:

```java
import io.helidon.webclient.http1.Http1Client;

Http1Client client = Http1Client.builder()
        .baseUri("http://other-service:8080")
        .build();

String body = client.get("/api/data")
        .requestEntity(String.class);
```

Since Helidon 4 runs on virtual threads, blocking on the HTTP response is efficient — there is no need for reactive `Mono<T>` or `CompletableFuture<T>`. The calling virtual thread is unmounted from its carrier while waiting for the network response.

---

## Testing

Helidon provides test support via `helidon-testing-junit5`:

```java
import io.helidon.testing.junit5.Testing;
import io.helidon.webserver.testing.junit5.ServerTest;
import io.helidon.webserver.testing.junit5.SetUpRoute;
import io.helidon.webclient.http1.Http1Client;

@ServerTest
class GreetServiceTest {

    @SetUpRoute
    static void routing(HttpRouting.Builder routing) {
        routing.register("/greet", new GreetService());
    }

    @Test
    void testDefaultGreeting(Http1Client client) {
        String response = client.get("/greet")
                .requestEntity(String.class);
        assertThat(response, containsString("Hello"));
    }
}
```

`@ServerTest` starts a real Helidon web server on a random port and injects an `Http1Client` pointed at it. Tests hit real HTTP — no mocking layer.

---

## SE vs Spring Boot — Side by Side

### Defining a GET Endpoint

**Spring Boot:**

```java
@RestController
@RequestMapping("/greet")
public class GreetController {

    @GetMapping("/{name}")
    public GreetResponse greet(@PathVariable String name) {
        return new GreetResponse("Hello, " + name);
    }
}
```

**Helidon SE:**

```java
public class GreetService implements HttpService {

    @Override
    public void routing(HttpRules rules) {
        rules.get("/{name}", this::greet);
    }

    private void greet(ServerRequest req, ServerResponse res) {
        String name = req.path().pathParameters().get("name");
        res.send(new GreetResponse("Hello, " + name));
    }
}
```

Spring is more concise for simple cases (annotations eliminate boilerplate). SE is more explicit — you see the routing registration, parameter extraction, and response send.

### Configuration

**Spring Boot:**

```java
@ConfigurationProperties(prefix = "app")
public record AppConfig(String greeting, int maxRetries) {}
```

**Helidon SE:**

```java
Config appConfig = Config.create().get("app");
String greeting = appConfig.get("greeting").asString().orElse("Hello");
int maxRetries = appConfig.get("max-retries").asInt().orElse(3);
```

Spring binds config to typed objects automatically. SE requires explicit extraction (or mapping via `config.as(Pojo.class)`).

### The Tradeoff

| SE wins | Spring Boot wins |
|---------|-----------------|
| Transparency — every line visible | Productivity — less code for common cases |
| Fast startup, small footprint | Massive ecosystem (Data, Security, Batch, etc.) |
| No proxy surprises | Auto-configuration handles 90% of wiring |
| Simple debugging (no AOP stack) | Community, docs, Stack Overflow answers |

---

## Related

- [Helidon Overview](helidon-overview.md) — positioning, framework comparison, when to choose
- [Helidon MP](helidon-mp.md) — the annotation-driven model built on top of SE
- [Helidon Nima Architecture](nima-virtual-threads-architecture.md) — the virtual-thread web server under SE
- [Functional Interfaces and Lambdas](../java-fundamentals/functional-interfaces-and-lambdas.md) — SE relies heavily on `Function`, `Consumer`, builder patterns
- [REST Controller Patterns](../web-layer/rest-controller-patterns.md) — Spring's annotation-driven REST for comparison
- [Distributed Tracing](../observability/distributed-tracing.md) — OpenTelemetry, Micrometer — Helidon's tracing integrates here

## References

- [Helidon SE Documentation](https://helidon.io/docs/v4/se/introduction) — official guides
- [Helidon SE Routing API](https://helidon.io/docs/v4/apidocs/io.helidon.webserver/io/helidon/webserver/http/HttpRouting.html) — Javadoc for `HttpRouting`
- [Helidon Config API](https://helidon.io/docs/v4/se/config/introduction) — config sources, profiles, mapping
- [Helidon Security](https://helidon.io/docs/v4/se/security/introduction) — provider architecture, OIDC, JWT
- [Helidon 4 SE — Redpill Linpro](https://www.redpill-linpro.com/techblog/2024/01/09/helidon.html) — walkthrough with code examples
- [Microservices with Oracle Helidon — Baeldung](https://www.baeldung.com/microservices-oracle-helidon) — tutorial covering SE and MP
- [Helidon GitHub Repository](https://github.com/helidon-io/helidon) — source, examples, issues
