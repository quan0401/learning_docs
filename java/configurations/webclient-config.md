---
title: "WebClient Configuration in Spring WebFlux"
date: 2026-04-15
updated: 2026-04-15
tags: [webclient, webflux, reactor-netty, http-client, configuration, timeouts]
---

# WebClient Configuration in Spring WebFlux

**Date:** 2026-04-15 | **Updated:** 2026-04-15
**Tags:** `webclient` `webflux` `reactor-netty` `http-client` `configuration` `timeouts`

## Table of Contents

- [Summary](#summary)
- [The Default WebClient](#the-default-webclient)
- [Timeouts — The Most Important Config](#timeouts--the-most-important-config)
  - [Connection Timeout](#connection-timeout)
  - [Response Timeout](#response-timeout)
  - [Read and Write Timeouts](#read-and-write-timeouts)
  - [Per-Request Timeout](#per-request-timeout)
- [Connection Pool Tuning](#connection-pool-tuning)
  - [Pool Settings](#pool-settings)
  - [Custom ConnectionProvider](#custom-connectionprovider)
  - [Per-Client Pools](#per-client-pools)
- [Codecs and Body Size Limits](#codecs-and-body-size-limits)
  - [In-Memory Buffer Size](#in-memory-buffer-size)
  - [Custom Codecs](#custom-codecs)
- [Default Headers and Cookies](#default-headers-and-cookies)
- [ExchangeFilterFunction — Cross-Cutting Concerns](#exchangefilterfunction--cross-cutting-concerns)
  - [Logging Filter](#logging-filter)
  - [Authentication Filter](#authentication-filter)
  - [Retry Filter](#retry-filter)
  - [Correlation ID Propagation](#correlation-id-propagation)
- [SSL/TLS Configuration](#ssltls-configuration)
- [Proxy Configuration](#proxy-configuration)
- [Multiple WebClient Beans](#multiple-webclient-beans)
- [Complete Production-Ready Example](#complete-production-ready-example)
- [Common Pitfalls](#common-pitfalls)
- [Related](#related)
- [References](#references)

---

## Summary

[Spring WebClient](https://docs.spring.io/spring-framework/reference/web/webflux-webclient/client-builder.html) is the reactive, non-blocking HTTP client for Spring WebFlux applications. Production WebClient configuration requires explicit timeouts (connection, response, read, write), tuned [Reactor Netty connection pools](https://projectreactor.io/docs/netty/release/reference/http-client.html), appropriate codec buffer sizes, and `ExchangeFilterFunction`s for cross-cutting concerns like logging, authentication, retries, and trace propagation.

---

## The Default WebClient

Spring Boot auto-configures a `WebClient.Builder` bean. Use it to create per-target `WebClient` instances:

```java
@Configuration
public class WebClientConfig {
    @Bean
    public WebClient moviesWebClient(WebClient.Builder builder) {
        return builder.baseUrl("https://api.movies.example.com").build();
    }
}
```

**Why use the auto-configured builder?** It comes pre-configured with:
- Codec auto-configuration (Jackson, Reactor Netty)
- Observability hooks for tracing/metrics
- `WebClientCustomizer` beans applied automatically

**Don't** create a raw `WebClient.create()` — you'll lose all of the above.

---

## Timeouts — The Most Important Config

**Default WebClient has NO timeouts.** A hung server can hold reactive subscriptions forever, exhausting connections and causing cascading failures. Configure timeouts explicitly.

### Connection Timeout

How long to wait for a TCP connection to establish:

```java
HttpClient httpClient = HttpClient.create()
    .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000);  // 5 seconds

WebClient client = WebClient.builder()
    .clientConnector(new ReactorClientHttpConnector(httpClient))
    .build();
```

### Response Timeout

Total time from request sent to response fully received:

```java
HttpClient httpClient = HttpClient.create()
    .responseTimeout(Duration.ofSeconds(10));
```

### Read and Write Timeouts

Lower-level timeouts for individual TCP read/write operations:

```java
HttpClient httpClient = HttpClient.create()
    .doOnConnected(connection -> connection
        .addHandlerLast(new ReadTimeoutHandler(10, TimeUnit.SECONDS))
        .addHandlerLast(new WriteTimeoutHandler(10, TimeUnit.SECONDS)));
```

### Per-Request Timeout

Use Reactor's `.timeout()` for per-call overrides:

```java
webClient.get().uri("/slow-endpoint")
    .retrieve()
    .bodyToMono(String.class)
    .timeout(Duration.ofSeconds(3));  // Override default
```

**Recommended starting timeouts:**

| Timeout | Internal services | External APIs |
|---------|------------------|---------------|
| Connection | 1-3s | 5-10s |
| Response | 5-10s | 30-60s |
| Read/Write | 5-10s | 30-60s |

Tune based on your SLAs and downstream behavior.

---

## Connection Pool Tuning

[Reactor Netty's `HttpClient`](https://projectreactor.io/docs/netty/release/reference/http-client.html) maintains a connection pool by default. The defaults are conservative — tune for your workload.

### Pool Settings

| Setting | Default | What It Controls |
|---------|---------|------------------|
| `maxConnections` | 500 | Total open connections |
| `pendingAcquireMaxCount` | 1000 | Max requests waiting for a connection |
| `pendingAcquireTimeout` | 45s | How long a request waits before throwing `PoolAcquireTimeoutException` |
| `maxIdleTime` | unbounded | Idle connection lifetime |
| `maxLifeTime` | unbounded | Total connection lifetime (force-rotate) |

### Custom ConnectionProvider

```java
ConnectionProvider provider = ConnectionProvider.builder("movies-pool")
    .maxConnections(50)                              // Limit per-client
    .pendingAcquireMaxCount(100)
    .pendingAcquireTimeout(Duration.ofSeconds(5))    // Fail fast under load
    .maxIdleTime(Duration.ofSeconds(20))             // Recycle idle conns
    .maxLifeTime(Duration.ofMinutes(5))              // Force rotation
    .evictInBackground(Duration.ofSeconds(120))      // Background eviction
    .build();

HttpClient httpClient = HttpClient.create(provider)
    .responseTimeout(Duration.ofSeconds(10));

WebClient client = WebClient.builder()
    .clientConnector(new ReactorClientHttpConnector(httpClient))
    .build();
```

### Per-Client Pools

**Use separate pools for separate downstreams.** A slow service shouldn't drain the pool used by a fast service:

```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient moviesWebClient() {
        return clientWithPool("movies-pool", 50, "https://movies.example.com");
    }

    @Bean
    public WebClient reviewsWebClient() {
        return clientWithPool("reviews-pool", 30, "https://reviews.example.com");
    }

    private WebClient clientWithPool(String name, int maxConn, String baseUrl) {
        ConnectionProvider provider = ConnectionProvider.builder(name)
            .maxConnections(maxConn)
            .pendingAcquireTimeout(Duration.ofSeconds(5))
            .build();

        HttpClient httpClient = HttpClient.create(provider)
            .responseTimeout(Duration.ofSeconds(10));

        return WebClient.builder()
            .clientConnector(new ReactorClientHttpConnector(httpClient))
            .baseUrl(baseUrl)
            .build();
    }
}
```

---

## Codecs and Body Size Limits

### In-Memory Buffer Size

WebClient's default in-memory buffer is **256 KB**. Large responses throw `DataBufferLimitException`:

```java
WebClient client = WebClient.builder()
    .codecs(configurer -> configurer
        .defaultCodecs()
        .maxInMemorySize(16 * 1024 * 1024))  // 16 MB
    .build();
```

For streaming large payloads, use `bodyToFlux()` + chunked processing instead of increasing the buffer indefinitely.

### Custom Codecs

Register custom codecs (e.g., for protobuf, custom JSON config):

```java
WebClient client = WebClient.builder()
    .codecs(configurer -> {
        configurer.customCodecs().register(new ProtobufEncoder());
        configurer.customCodecs().register(new ProtobufDecoder());
    })
    .build();
```

Customize the auto-configured Jackson `ObjectMapper`:

```java
@Bean
public WebClient jsonWebClient(WebClient.Builder builder) {
    ObjectMapper mapper = new ObjectMapper()
        .registerModule(new JavaTimeModule())
        .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
        .setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE);

    return builder
        .codecs(configurer -> {
            configurer.defaultCodecs().jackson2JsonEncoder(
                new Jackson2JsonEncoder(mapper));
            configurer.defaultCodecs().jackson2JsonDecoder(
                new Jackson2JsonDecoder(mapper));
        })
        .build();
}
```

---

## Default Headers and Cookies

```java
WebClient client = WebClient.builder()
    .baseUrl("https://api.example.com")
    .defaultHeader(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_VALUE)
    .defaultHeader(HttpHeaders.USER_AGENT, "movies-service/1.0")
    .defaultHeaders(headers -> {
        headers.setBasicAuth("user", "pass");
        headers.set("X-Source", "movies-service");
    })
    .defaultCookie("session-id", sessionId)
    .build();
```

For headers that change per request, use a filter (next section).

---

## ExchangeFilterFunction — Cross-Cutting Concerns

[`ExchangeFilterFunction`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/reactive/function/client/ExchangeFilterFunction.html) intercepts requests and responses. Use it for [cross-cutting concerns](https://docs.spring.io/spring-framework/reference/web/webflux-webclient/client-filter.html):

### Logging Filter

```java
public static ExchangeFilterFunction logRequest() {
    return ExchangeFilterFunction.ofRequestProcessor(request -> {
        log.info("Request: {} {}", request.method(), request.url());
        request.headers().forEach((name, values) ->
            values.forEach(value -> log.debug("  {}: {}", name, value)));
        return Mono.just(request);
    });
}

public static ExchangeFilterFunction logResponse() {
    return ExchangeFilterFunction.ofResponseProcessor(response -> {
        log.info("Response: {}", response.statusCode());
        return Mono.just(response);
    });
}

WebClient client = WebClient.builder()
    .filter(logRequest())
    .filter(logResponse())
    .build();
```

### Authentication Filter

```java
public static ExchangeFilterFunction bearerToken(Supplier<String> tokenSupplier) {
    return ExchangeFilterFunction.ofRequestProcessor(request -> {
        ClientRequest authorized = ClientRequest.from(request)
            .header(HttpHeaders.AUTHORIZATION, "Bearer " + tokenSupplier.get())
            .build();
        return Mono.just(authorized);
    });
}

WebClient client = WebClient.builder()
    .filter(bearerToken(() -> tokenService.getCurrentToken()))
    .build();
```

For OAuth2, use `ServerOAuth2AuthorizedClientExchangeFilterFunction` from Spring Security.

### Retry Filter

Combine filter with Reactor's retry — but prefer per-call retry for fine-grained control:

```java
@Bean
public WebClient resilientWebClient(WebClient.Builder builder) {
    return builder
        .filter((request, next) -> next.exchange(request)
            .retryWhen(Retry.fixedDelay(3, Duration.ofSeconds(1))
                .filter(throwable -> throwable instanceof IOException)))
        .build();
}
```

### Correlation ID Propagation

Propagate trace/correlation IDs from Reactor Context to outgoing requests:

```java
public static ExchangeFilterFunction correlationIdFilter() {
    return (request, next) -> Mono.deferContextual(ctx -> {
        String correlationId = ctx.getOrDefault("correlationId", "");
        if (correlationId.isEmpty()) {
            return next.exchange(request);
        }
        ClientRequest withId = ClientRequest.from(request)
            .header("X-Correlation-Id", correlationId)
            .build();
        return next.exchange(withId);
    });
}
```

For automatic trace ID propagation, use Micrometer Tracing (see [Reactive Observability](../reactive-observability.md)).

---

## SSL/TLS Configuration

For self-signed certs in dev (NEVER in production):

```java
SslContext sslContext = SslContextBuilder.forClient()
    .trustManager(InsecureTrustManagerFactory.INSTANCE)
    .build();

HttpClient httpClient = HttpClient.create()
    .secure(spec -> spec.sslContext(sslContext));

WebClient client = WebClient.builder()
    .clientConnector(new ReactorClientHttpConnector(httpClient))
    .build();
```

For production with mutual TLS:

```java
SslContext sslContext = SslContextBuilder.forClient()
    .keyManager(clientCertFile, clientKeyFile)
    .trustManager(caCertFile)
    .build();

HttpClient httpClient = HttpClient.create()
    .secure(spec -> spec.sslContext(sslContext));
```

---

## Proxy Configuration

```java
HttpClient httpClient = HttpClient.create()
    .proxy(spec -> spec
        .type(ProxyProvider.Proxy.HTTP)
        .host("proxy.example.com")
        .port(8080)
        .username("proxy-user")
        .password(user -> "proxy-pass"));
```

Honor system proxy settings:

```java
HttpClient httpClient = HttpClient.create()
    .proxyWithSystemProperties();
```

---

## Multiple WebClient Beans

Define one WebClient per downstream service, not a single shared one:

```java
@Configuration
@EnableConfigurationProperties({MoviesApiProps.class, ReviewsApiProps.class})
public class WebClientConfig {

    @Bean
    public WebClient moviesWebClient(MoviesApiProps props,
                                      WebClient.Builder builder) {
        return buildClient(builder, "movies", props.url(),
            props.timeout(), props.maxConnections());
    }

    @Bean
    public WebClient reviewsWebClient(ReviewsApiProps props,
                                       WebClient.Builder builder) {
        return buildClient(builder, "reviews", props.url(),
            props.timeout(), props.maxConnections());
    }

    private WebClient buildClient(WebClient.Builder builder, String name,
                                   String baseUrl, Duration timeout, int maxConn) {
        ConnectionProvider provider = ConnectionProvider.builder(name + "-pool")
            .maxConnections(maxConn)
            .pendingAcquireTimeout(Duration.ofSeconds(5))
            .build();

        HttpClient httpClient = HttpClient.create(provider)
            .responseTimeout(timeout)
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000);

        return builder
            .baseUrl(baseUrl)
            .clientConnector(new ReactorClientHttpConnector(httpClient))
            .build();
    }
}
```

Inject by qualifier:

```java
@Service
public class AggregatorService {
    public AggregatorService(
        @Qualifier("moviesWebClient") WebClient movies,
        @Qualifier("reviewsWebClient") WebClient reviews
    ) { ... }
}
```

---

## Complete Production-Ready Example

```java
@Configuration
@EnableConfigurationProperties(MoviesApiProperties.class)
public class MoviesWebClientConfig {

    @Bean
    public WebClient moviesWebClient(MoviesApiProperties props,
                                      WebClient.Builder builder,
                                      ObjectProvider<ExchangeFilterFunction> filters) {
        // 1. Connection pool
        ConnectionProvider provider = ConnectionProvider.builder("movies-pool")
            .maxConnections(props.maxConnections())
            .pendingAcquireMaxCount(props.maxConnections() * 2)
            .pendingAcquireTimeout(Duration.ofSeconds(5))
            .maxIdleTime(Duration.ofSeconds(30))
            .maxLifeTime(Duration.ofMinutes(5))
            .evictInBackground(Duration.ofSeconds(60))
            .build();

        // 2. HttpClient with timeouts
        HttpClient httpClient = HttpClient.create(provider)
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, (int) props.connectTimeout().toMillis())
            .responseTimeout(props.responseTimeout())
            .doOnConnected(conn -> conn
                .addHandlerLast(new ReadTimeoutHandler(props.readTimeout().toSeconds(), TimeUnit.SECONDS))
                .addHandlerLast(new WriteTimeoutHandler(props.writeTimeout().toSeconds(), TimeUnit.SECONDS)));

        // 3. WebClient with codecs, headers, filters
        WebClient.Builder b = builder
            .baseUrl(props.url())
            .defaultHeader(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_VALUE)
            .defaultHeader(HttpHeaders.USER_AGENT, "movies-service/1.0")
            .clientConnector(new ReactorClientHttpConnector(httpClient))
            .codecs(c -> c.defaultCodecs().maxInMemorySize(props.maxInMemorySize()))
            .filter(logRequest())
            .filter(logResponse())
            .filter(correlationIdFilter());

        // 4. Apply user-defined filters from context
        filters.orderedStream().forEach(b::filter);

        return b.build();
    }
}
```

```yaml
# application.yml
movies:
  api:
    url: https://api.movies.example.com
    max-connections: 50
    connect-timeout: 3s
    response-timeout: 10s
    read-timeout: 10s
    write-timeout: 10s
    max-in-memory-size: 16777216  # 16 MB
```

```java
@ConfigurationProperties(prefix = "movies.api")
public record MoviesApiProperties(
    @NotBlank String url,
    @Min(1) int maxConnections,
    @NotNull Duration connectTimeout,
    @NotNull Duration responseTimeout,
    @NotNull Duration readTimeout,
    @NotNull Duration writeTimeout,
    @Min(1024) int maxInMemorySize
) {}
```

---

## Common Pitfalls

| Pitfall | Why It Hurts | Fix |
|---------|--------------|-----|
| No timeouts | A hung server holds connections forever | Set `responseTimeout` + `connectTimeout` |
| One shared WebClient for all downstreams | Slow downstream blocks fast ones | Per-target WebClient with per-target pool |
| Default 256 KB buffer | Large responses throw `DataBufferLimitException` | Increase `maxInMemorySize` or stream with `bodyToFlux` |
| `WebClient.create()` instead of builder | Loses observability, codec auto-config | Always inject `WebClient.Builder` |
| Calling `.block()` on result | Defeats reactive — blocks event-loop thread | Stay reactive with `flatMap` |
| Retry on 4xx errors | Wastes calls — 4xx won't recover | Filter retry: `.filter(ex -> ex instanceof ServerException)` |

---

## Related

- [Server / HTTP Configuration](server-http-config.md) — server-side equivalent (Reactor Netty server tuning)
- [Reactive Observability](../reactive-observability.md) — automatic trace propagation through WebClient
- [Wrapping Blocking JPA Calls in a Reactive Chain](../reactive-blocking-jpa-pattern.md) — why never use `.block()` on WebClient results

## References

- [WebClient Builder Configuration — Spring Framework](https://docs.spring.io/spring-framework/reference/web/webflux-webclient/client-builder.html) — codecs, timeouts, default headers, connector customization
- [Reactor Netty HttpClient — Reference Guide](https://projectreactor.io/docs/netty/release/reference/http-client.html) — connection pools, timeouts, SSL, proxy, metrics
- [ConnectionPoolSpec API — Reactor Netty](https://projectreactor.io/docs/netty/snapshot/api/reactor/netty/resources/ConnectionProvider.ConnectionPoolSpec.html) — full pool sizing, idle time, lifetime options
- [WebClient Filters — Spring Framework](https://docs.spring.io/spring-framework/reference/web/webflux-webclient/client-filter.html) — ExchangeFilterFunction reference
- [ExchangeFilterFunction Javadoc](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/reactive/function/client/ExchangeFilterFunction.html) — filter functional interface
- [HTTP Clients How-To — Spring Boot](https://docs.spring.io/spring-boot/how-to/http-clients.html) — auto-configured builders for WebClient and RestClient
