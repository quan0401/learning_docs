---
title: "Server and HTTP Configuration in Spring WebFlux"
date: 2026-04-15
updated: 2026-04-15
tags: [webflux, reactor-netty, server, http, cors, codecs, configuration]
---

# Server and HTTP Configuration in Spring WebFlux

**Date:** 2026-04-15 | **Updated:** 2026-04-15
**Tags:** `webflux` `reactor-netty` `server` `http` `cors` `codecs` `configuration`

## Table of Contents

- [Summary](#summary)
- [Server Properties](#server-properties)
  - [Port and Address](#port-and-address)
  - [Compression](#compression)
  - [HTTP/2](#http2)
  - [Access Logs](#access-logs)
- [Reactor Netty Server Tuning](#reactor-netty-server-tuning)
  - [Event Loop Sizing](#event-loop-sizing)
  - [Connection and Idle Timeouts](#connection-and-idle-timeouts)
  - [Request Decoder Limits](#request-decoder-limits)
  - [Programmatic Customization](#programmatic-customization)
- [SSL/TLS Configuration](#ssltls-configuration)
- [CORS Configuration](#cors-configuration)
  - [Global CORS via WebFluxConfigurer](#global-cors-via-webfluxconfigurer)
  - [Per-Endpoint CORS with @CrossOrigin](#per-endpoint-cors-with-crossorigin)
  - [CorsWebFilter for Functional Routes](#corswebfilter-for-functional-routes)
- [Server-Side Codecs](#server-side-codecs)
  - [Body Size Limits](#body-size-limits)
  - [Custom Encoders/Decoders](#custom-encodersdecoders)
- [WebFluxConfigurer — The Customization Hub](#webfluxconfigurer--the-customization-hub)
- [Path Matching and Content Negotiation](#path-matching-and-content-negotiation)
- [Static Resources](#static-resources)
- [Graceful Shutdown](#graceful-shutdown)
- [Complete Production Configuration](#complete-production-configuration)
- [Related](#related)
- [References](#references)

---

## Summary

Spring WebFlux runs on [Reactor Netty](https://docs.spring.io/projectreactor/reactor-netty/docs/current-SNAPSHOT/reference/html/http-server.html) by default, with most server behavior controlled via `server.*` and `spring.webflux.*` properties. Production server configuration involves event-loop sizing, connection/idle timeouts, request body limits, SSL/TLS, CORS policies, codec buffer sizes, and graceful shutdown. Custom behavior beyond properties uses [`WebFluxConfigurer`](https://docs.spring.io/spring-framework/reference/web/webflux/config.html) for codecs/CORS/path-matching, and `NettyReactiveWebServerFactory` customization for low-level Netty tuning.

---

## Server Properties

The full reference of [Spring Boot server properties](https://docs.spring.io/spring-boot/appendix/application-properties/index.html) is the authoritative list. Common ones:

### Port and Address

```yaml
server:
  port: 8080                # 0 = random port
  address: 0.0.0.0          # Bind to all interfaces (default)
```

### Compression

Compress responses to reduce network usage:

```yaml
server:
  compression:
    enabled: true
    mime-types:
      - text/html
      - text/css
      - text/javascript
      - application/json
      - application/xml
    min-response-size: 2KB  # Don't compress small responses
```

### HTTP/2

```yaml
server:
  http2:
    enabled: true
  ssl:
    enabled: true   # HTTP/2 generally requires TLS
    # ...SSL config...
```

### Access Logs

Enable Reactor Netty access logging:

```yaml
server:
  netty:
    access-log-enabled: true
```

Or via system property:
```bash
-Dreactor.netty.http.server.accessLogEnabled=true
```

Customize the log format with a custom logger named `reactor.netty.http.server.AccessLog`.

---

## Reactor Netty Server Tuning

### Event Loop Sizing

Reactor Netty creates `availableProcessors() * 2` event-loop threads by default. Override:

```yaml
spring:
  reactor:
    netty:
      ioWorkerCount: 16   # Server I/O threads (override default)
      ioSelectCount: 4
```

Or programmatically (more control):

```java
@Bean
public NettyReactiveWebServerFactory nettyReactiveWebServerFactory() {
    NettyReactiveWebServerFactory factory = new NettyReactiveWebServerFactory();
    factory.addServerCustomizers(httpServer ->
        httpServer.runOn(LoopResources.create("server-loop", 1, 16, true)));
    return factory;
}
```

**Sizing guidance:** Default is fine for most workloads. Increase only if profiling shows event-loop saturation.

### Connection and Idle Timeouts

```yaml
server:
  netty:
    connection-timeout: 30s     # Time to accept the connection
    idle-timeout: 60s           # Close idle connections
```

### Request Decoder Limits

Protect against malformed/oversized requests:

```yaml
server:
  netty:
    max-keep-alive-requests: 100  # -1 = unlimited
    h2c-max-content-length: 8MB
    initial-buffer-size: 128B
    max-initial-line-length: 4KB
    max-header-size: 8KB
    max-chunk-size: 8KB
    validate-headers: true
```

### Programmatic Customization

For deeper control:

```java
@Bean
public NettyServerCustomizer nettyServerCustomizer() {
    return httpServer -> httpServer
        .idleTimeout(Duration.ofSeconds(60))
        .doOnConnection(conn -> conn
            .addHandlerLast(new ReadTimeoutHandler(30, TimeUnit.SECONDS))
            .addHandlerLast(new WriteTimeoutHandler(30, TimeUnit.SECONDS)))
        .compress(true)
        .protocol(HttpProtocol.H2, HttpProtocol.HTTP11);
}
```

---

## SSL/TLS Configuration

```yaml
server:
  port: 8443
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: ${KEYSTORE_PASSWORD}
    key-store-type: PKCS12
    key-alias: my-cert
    protocol: TLS
    enabled-protocols: TLSv1.3,TLSv1.2
    ciphers:
      - TLS_AES_256_GCM_SHA384
      - TLS_CHACHA20_POLY1305_SHA256
```

For mutual TLS (client cert auth):

```yaml
server:
  ssl:
    client-auth: need              # need | want | none
    trust-store: classpath:truststore.p12
    trust-store-password: ${TRUSTSTORE_PASSWORD}
```

For multiple SSL bundles (Spring Boot 3.1+):

```yaml
spring:
  ssl:
    bundle:
      jks:
        my-server-bundle:
          key:
            alias: my-cert
          keystore:
            location: classpath:keystore.p12
            password: ${KEYSTORE_PASSWORD}

server:
  ssl:
    bundle: my-server-bundle
```

---

## CORS Configuration

[Cross-Origin Resource Sharing](https://docs.spring.io/spring-framework/reference/web/webflux-cors.html) controls which browser origins can call your API.

### Global CORS via WebFluxConfigurer

```java
@Configuration
public class WebConfig implements WebFluxConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://app.example.com", "https://admin.example.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowedHeaders("*")
            .exposedHeaders("X-Total-Count", "X-Correlation-Id")
            .allowCredentials(true)
            .maxAge(3600);

        registry.addMapping("/public/**")
            .allowedOriginPatterns("*")  // Use patterns, NOT "*", with credentials
            .allowedMethods("GET");
    }
}
```

**Security warning:** `allowedOrigins("*")` + `allowCredentials(true)` is rejected by browsers. Use `allowedOriginPatterns("*")` if you really need a wildcard, but **prefer specific origins**.

### Per-Endpoint CORS with @CrossOrigin

```java
@RestController
@RequestMapping("/api/movies")
@CrossOrigin(origins = "https://app.example.com")
public class MoviesController {

    @GetMapping("/{id}")
    @CrossOrigin(origins = "*", methods = RequestMethod.GET)  // Override per method
    public Mono<Movie> getMovie(@PathVariable String id) { ... }
}
```

### CorsWebFilter for Functional Routes

For `RouterFunction` based endpoints:

```java
@Bean
public CorsWebFilter corsWebFilter() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://app.example.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
    config.setAllowedHeaders(List.of("*"));
    config.setAllowCredentials(true);
    config.setMaxAge(3600L);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);
    return new CorsWebFilter(source);
}
```

---

## Server-Side Codecs

### Body Size Limits

The default in-memory limit for buffering request bodies is **256 KB**. Adjust for large payloads:

```java
@Configuration
public class WebConfig implements WebFluxConfigurer {
    @Override
    public void configureHttpMessageCodecs(ServerCodecConfigurer configurer) {
        configurer.defaultCodecs().maxInMemorySize(16 * 1024 * 1024);  // 16 MB
    }
}
```

For large file uploads, use streaming (`Flux<DataBuffer>` or `Part` streaming) instead of buffering.

### Custom Encoders/Decoders

```java
@Override
public void configureHttpMessageCodecs(ServerCodecConfigurer configurer) {
    // Override Jackson ObjectMapper used for JSON
    ObjectMapper mapper = new ObjectMapper()
        .registerModule(new JavaTimeModule())
        .setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE);
    
    configurer.defaultCodecs()
        .jackson2JsonEncoder(new Jackson2JsonEncoder(mapper));
    configurer.defaultCodecs()
        .jackson2JsonDecoder(new Jackson2JsonDecoder(mapper));

    // Add custom codec
    configurer.customCodecs().register(new ProtobufEncoder());
    configurer.customCodecs().register(new ProtobufDecoder());
}
```

---

## WebFluxConfigurer — The Customization Hub

[`WebFluxConfigurer`](https://docs.spring.io/spring-framework/reference/web/webflux/config.html) is the central interface for customizing WebFlux behavior:

```java
@Configuration
@EnableWebFlux  // Often optional in Spring Boot — auto-applied
public class WebConfig implements WebFluxConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) { ... }

    @Override
    public void configureHttpMessageCodecs(ServerCodecConfigurer configurer) { ... }

    @Override
    public void configureContentTypeResolver(RequestedContentTypeResolverBuilder builder) { ... }

    @Override
    public void configurePathMatching(PathMatchConfigurer configurer) { ... }

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) { ... }

    @Override
    public void addFormatters(FormatterRegistry registry) { ... }

    @Override
    public Validator getValidator() { ... }
}
```

**Don't use `@EnableWebFlux` unless you know you need it** — it disables Spring Boot's WebFlux auto-configuration. Just implement `WebFluxConfigurer` and Spring Boot will pick it up.

---

## Path Matching and Content Negotiation

```java
@Override
public void configurePathMatching(PathMatchConfigurer configurer) {
    configurer
        .setUseCaseSensitiveMatch(false)
        .addPathPrefix("/api", clazz -> clazz.isAnnotationPresent(RestController.class));
}
```

`addPathPrefix` is great for versioning — all `@RestController` classes get `/api` prepended.

Content negotiation:

```java
@Override
public void configureContentTypeResolver(RequestedContentTypeResolverBuilder builder) {
    builder
        .headerResolver()                              // Use Accept header (default)
        .parameterResolver().parameterName("format");  // ?format=json fallback
}
```

---

## Static Resources

```java
@Override
public void addResourceHandlers(ResourceHandlerRegistry registry) {
    registry.addResourceHandler("/static/**")
        .addResourceLocations("classpath:/static/")
        .setCacheControl(CacheControl.maxAge(Duration.ofDays(30))
            .cachePublic());
}
```

For SPA-style routing where unknown paths serve `index.html`:

```yaml
spring:
  webflux:
    static-path-pattern: /**
  web:
    resources:
      static-locations: classpath:/static/
```

---

## Graceful Shutdown

Stop accepting new requests, drain in-flight requests, then shut down:

```yaml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s   # Max wait for in-flight requests
```

Combined with Kubernetes:

```yaml
# K8s pod spec
terminationGracePeriodSeconds: 35   # Slightly more than Spring's timeout
```

---

## Complete Production Configuration

```yaml
spring:
  application:
    name: movies-service
  reactor:
    context-propagation: auto
  lifecycle:
    timeout-per-shutdown-phase: 30s

server:
  port: ${SERVER_PORT:8080}
  shutdown: graceful
  compression:
    enabled: true
    mime-types: text/html, text/css, application/json
    min-response-size: 2KB
  http2:
    enabled: true
  netty:
    connection-timeout: 30s
    idle-timeout: 60s
    max-keep-alive-requests: 1000
    max-header-size: 8KB
  ssl:
    bundle: server-bundle
    client-auth: none
    enabled-protocols: TLSv1.3,TLSv1.2
```

```java
@Configuration
public class WebConfig implements WebFluxConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins(
                "https://app.example.com",
                "https://admin.example.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE", "PATCH")
            .allowedHeaders("*")
            .exposedHeaders("X-Total-Count", "X-Correlation-Id")
            .allowCredentials(true)
            .maxAge(3600);
    }

    @Override
    public void configureHttpMessageCodecs(ServerCodecConfigurer configurer) {
        configurer.defaultCodecs().maxInMemorySize(16 * 1024 * 1024);  // 16 MB

        ObjectMapper mapper = new ObjectMapper()
            .registerModule(new JavaTimeModule())
            .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
            .setSerializationInclusion(JsonInclude.Include.NON_NULL);

        configurer.defaultCodecs()
            .jackson2JsonEncoder(new Jackson2JsonEncoder(mapper));
        configurer.defaultCodecs()
            .jackson2JsonDecoder(new Jackson2JsonDecoder(mapper));
    }

    @Override
    public void configurePathMatching(PathMatchConfigurer configurer) {
        configurer.addPathPrefix("/api/v1",
            clazz -> clazz.isAnnotationPresent(RestController.class));
    }
}

@Bean
public NettyServerCustomizer nettyServerCustomizer() {
    return httpServer -> httpServer
        .doOnConnection(conn -> conn
            .addHandlerLast(new ReadTimeoutHandler(30, TimeUnit.SECONDS))
            .addHandlerLast(new WriteTimeoutHandler(30, TimeUnit.SECONDS)));
}
```

---

## Related

- [WebClient Configuration](webclient-config.md) — client-side equivalent
- [Externalized Configuration](externalized-config.md) — externalizing server.* and CORS settings
- [Reactive Observability](../reactive-observability.md) — actuator endpoints, health checks for the server
- [Java @Configuration Classes](java-bean-config.md) — WebFluxConfigurer is a typical @Configuration pattern

## References

- [HTTP Server — Reactor Netty Reference Guide](https://docs.spring.io/projectreactor/reactor-netty/docs/current-SNAPSHOT/reference/html/http-server.html) — server binding, SSL, compression, access log, idle timeout
- [CORS — Spring Framework WebFlux](https://docs.spring.io/spring-framework/reference/web/webflux-cors.html) — @CrossOrigin, global CORS via WebFluxConfigurer, CorsWebFilter
- [Common Application Properties — Spring Boot](https://docs.spring.io/spring-boot/appendix/application-properties/index.html) — appendix listing every server.*, spring.webflux.*, and spring.codec.* property
- [WebFlux Config — Spring Framework](https://docs.spring.io/spring-framework/reference/web/webflux/config.html) — codec configuration, content type resolvers, path matching, view resolution
- [ServerProperties Javadoc](https://docs.spring.io/spring-boot/api/java/org/springframework/boot/web/server/autoconfigure/ServerProperties.html) — full server.* configuration class
