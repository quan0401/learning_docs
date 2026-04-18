---
title: "Java @Configuration Classes in Spring Boot 3.x/4.x"
date: 2026-04-15
updated: 2026-04-15
tags: [spring, configuration, beans, dependency-injection, auto-configuration]
---

# Java @Configuration Classes in Spring Boot 3.x/4.x

**Date:** 2026-04-15 | **Updated:** 2026-04-15
**Tags:** `spring` `configuration` `beans` `dependency-injection` `auto-configuration`

## Table of Contents

- [Summary](#summary)
- [@Configuration Basics](#configuration-basics)
  - [Full Mode vs Lite Mode](#full-mode-vs-lite-mode)
- [Bean Definitions with @Bean](#bean-definitions-with-bean)
  - [Inter-Bean References](#inter-bean-references)
  - [Bean Naming](#bean-naming)
  - [Bean Scopes](#bean-scopes)
- [Bean Lifecycle](#bean-lifecycle)
  - [Initialization and Destruction](#initialization-and-destruction)
  - [InitializingBean and DisposableBean](#initializingbean-and-disposablebean)
  - [Lifecycle Order](#lifecycle-order)
- [Conditional Configuration](#conditional-configuration)
  - [@ConditionalOnProperty](#conditionalonproperty)
  - [@ConditionalOnClass / @ConditionalOnMissingClass](#conditionalonclass--conditionalonmissingclass)
  - [@ConditionalOnBean / @ConditionalOnMissingBean](#conditionalonbean--conditionalonmissingbean)
  - [@Profile](#profile)
  - [Custom @Conditional](#custom-conditional)
- [Configuration Composition](#configuration-composition)
  - [@Import](#import)
  - [@ComponentScan](#componentscan)
  - [@EnableXxx Patterns](#enablexxx-patterns)
- [Auto-Configuration](#auto-configuration)
  - [Creating an Auto-Configuration](#creating-an-auto-configuration)
  - [Auto-Configuration Order](#auto-configuration-order)
  - [Disabling Auto-Configuration](#disabling-auto-configuration)
- [Common Patterns in Reactive Apps](#common-patterns-in-reactive-apps)
  - [WebClient Bean](#webclient-bean)
  - [ObjectMapper Customization](#objectmapper-customization)
  - [Scheduler Beans](#scheduler-beans)
- [Related](#related)
- [References](#references)

---

## Summary

Spring's [`@Configuration`](https://docs.spring.io/spring-framework/reference/core/beans/java/configuration-annotation.html) annotation marks a class as a source of bean definitions, replacing XML configuration. Combined with `@Bean` methods, conditional annotations like `@ConditionalOnProperty`, and Spring Boot's auto-configuration mechanism, `@Configuration` classes are how you declaratively wire up your application's dependencies, customize framework defaults, and create environment-specific behavior.

---

## @Configuration Basics

A `@Configuration` class is a Java class that contains `@Bean` methods. Spring processes these methods at startup and registers each returned object as a bean in the application context.

```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient moviesWebClient() {
        return WebClient.builder()
            .baseUrl("https://api.movies.example.com")
            .defaultHeader("Accept", MediaType.APPLICATION_JSON_VALUE)
            .build();
    }
}
```

The bean is now available for injection anywhere:

```java
@Service
public class MoviesService {
    private final WebClient webClient;

    public MoviesService(WebClient moviesWebClient) {  // Injected by name
        this.webClient = moviesWebClient;
    }
}
```

### Full Mode vs Lite Mode

[Full mode](https://docs.spring.io/spring-framework/reference/core/beans/java/configuration-annotation.html) (`@Configuration` proxyBeanMethods=true, **default**):
- Spring proxies the configuration class with CGLIB
- `@Bean` methods are intercepted to return the **singleton** instance
- Calling `beanA()` from `beanB()` returns the **same instance** managed by Spring

```java
@Configuration  // proxyBeanMethods = true (default)
public class AppConfig {
    @Bean
    public ServiceA serviceA() {
        return new ServiceA();
    }

    @Bean
    public ServiceB serviceB() {
        return new ServiceB(serviceA());  // Returns the SAME ServiceA singleton
    }
}
```

**Lite mode** (`@Configuration(proxyBeanMethods = false)` or `@Component` with `@Bean` methods):
- No CGLIB proxy — faster startup, smaller footprint
- `@Bean` methods behave like **plain Java methods**: a direct call creates a new instance
- Safe only when `@Bean` methods are **self-contained** — no method in the class calls another `@Bean` method directly

```java
@Configuration(proxyBeanMethods = false)
public class AppConfig {
    @Bean
    public WebClient webClient() {
        return WebClient.builder().build();  // Self-contained, no inter-bean refs
    }
}
```

#### The Lite-Mode Trap — Why "methods don't call each other" matters

In full mode, the CGLIB proxy intercepts every inter-bean method call and returns the cached singleton from the container. In lite mode there is no proxy, so a method call is just a Java call — it runs the method body again and creates a new object:

```java
@Configuration(proxyBeanMethods = false)  // ⚠️ Lite mode
public class AppConfig {
    @Bean
    public ServiceA serviceA() {
        return new ServiceA();
    }

    @Bean
    public ServiceB serviceB() {
        return new ServiceB(serviceA());  // ⚠️ Plain Java call — creates a NEW ServiceA
    }
}
```

What actually happens at startup:
- Spring calls `serviceA()` once and registers the result as the container's singleton
- Spring calls `serviceB()` → inside it, `serviceA()` runs again → a **second, unmanaged** `ServiceA` instance is constructed
- `ServiceB` now holds a `ServiceA` that is **not** the one the container manages
- Anyone else injecting `ServiceA` gets the container's copy — a different object from the one inside `ServiceB`

You end up with two `ServiceA` instances when you expected one singleton, and the bug is silent — nothing fails at startup.

#### The Safe Pattern in Lite Mode — Inject via Method Parameters

If you need bean-to-bean references in lite mode, declare the dependency as a **method parameter**. Spring resolves it from the container (the same singleton everyone else sees), with no proxy needed:

```java
@Configuration(proxyBeanMethods = false)
public class AppConfig {
    @Bean
    public ServiceA serviceA() {
        return new ServiceA();
    }

    @Bean
    public ServiceB serviceB(ServiceA serviceA) {  // ✅ Injected from container
        return new ServiceB(serviceA);             // Same singleton, no proxy required
    }
}
```

This pattern works identically in both modes, so it's a good habit even in full mode — it makes the dependency explicit and makes the class safe to flip to lite mode later.

**Recommendation:** Default to `proxyBeanMethods = true` (the `@Configuration` default). Only switch to lite mode when (a) startup time is a measured concern and (b) your `@Bean` methods either are self-contained or use method-parameter injection for their dependencies. Auto-configuration classes shipped inside Spring Boot commonly use lite mode for exactly this reason — it saves startup time across hundreds of configuration classes.

---

## Bean Definitions with @Bean

[`@Bean`](https://docs.spring.io/spring-framework/reference/core/beans/java/bean-annotation.html) marks a method whose return value should be registered as a Spring bean.

### Inter-Bean References

In full mode, simply call the other `@Bean` method:

```java
@Configuration
public class AppConfig {
    @Bean
    public ConnectionFactory connectionFactory() {
        return ConnectionFactories.get("r2dbc:postgresql://localhost/mydb");
    }

    @Bean
    public R2dbcEntityTemplate entityTemplate() {
        return new R2dbcEntityTemplate(connectionFactory());  // Same singleton
    }
}
```

Or accept it as a method parameter (works in both modes):

```java
@Configuration(proxyBeanMethods = false)
public class AppConfig {
    @Bean
    public ConnectionFactory connectionFactory() {
        return ConnectionFactories.get("r2dbc:postgresql://localhost/mydb");
    }

    @Bean
    public R2dbcEntityTemplate entityTemplate(ConnectionFactory connectionFactory) {
        return new R2dbcEntityTemplate(connectionFactory);
    }
}
```

### Bean Naming

By default, the bean name is the method name:

```java
@Bean
public WebClient moviesWebClient() { ... }  // Bean name: "moviesWebClient"

@Bean(name = "movieApi")
public WebClient moviesWebClient() { ... }  // Bean name: "movieApi"

@Bean({"primary", "default"})
public WebClient moviesWebClient() { ... }  // Two aliases
```

When multiple beans of the same type exist, use `@Qualifier`:

```java
@Configuration
public class WebClientConfig {
    @Bean
    public WebClient moviesWebClient() { ... }

    @Bean
    public WebClient reviewsWebClient() { ... }
}

@Service
public class AggregatorService {
    public AggregatorService(
        @Qualifier("moviesWebClient") WebClient movies,
        @Qualifier("reviewsWebClient") WebClient reviews
    ) { ... }
}
```

Or mark one as `@Primary`:

```java
@Bean
@Primary
public WebClient moviesWebClient() { ... }  // Default if no @Qualifier
```

### Bean Scopes

| Scope | Lifetime |
|-------|----------|
| `singleton` (default) | One instance per Spring container |
| `prototype` | New instance every injection / lookup |
| `request` | One per HTTP request (web only) |
| `session` | One per HTTP session (web only) |

```java
@Bean
@Scope("prototype")
public RequestProcessor requestProcessor() {
    return new RequestProcessor();
}
```

In WebFlux, prefer **stateless singletons** — `request` and `session` scopes don't fit reactive's thread-hopping model.

---

## Bean Lifecycle

### Initialization and Destruction

`@Bean` provides `initMethod` and `destroyMethod` attributes:

```java
@Bean(initMethod = "start", destroyMethod = "shutdown")
public ConnectionPool connectionPool() {
    return new ConnectionPool();
}
```

Or use `@PostConstruct` / `@PreDestroy` on the method:

```java
@Component
public class CacheManager {
    @PostConstruct
    public void init() {
        log.info("Cache initialized");
    }

    @PreDestroy
    public void cleanup() {
        log.info("Cache cleaned up");
    }
}
```

### InitializingBean and DisposableBean

Implement these interfaces for the same effect (less preferred than annotations):

```java
@Component
public class CacheManager implements InitializingBean, DisposableBean {
    @Override
    public void afterPropertiesSet() { /* init */ }

    @Override
    public void destroy() { /* cleanup */ }
}
```

### Lifecycle Order

For a bean implementing all options, [Spring calls them](https://docs.spring.io/spring-framework/reference/core/beans/factory-nature.html) in this order:

1. Constructor
2. `BeanNameAware.setBeanName()`, `BeanFactoryAware.setBeanFactory()`, etc.
3. `@PostConstruct` methods
4. `InitializingBean.afterPropertiesSet()`
5. `@Bean(initMethod)` method
6. *(Bean is in use)*
7. `@PreDestroy` methods
8. `DisposableBean.destroy()`
9. `@Bean(destroyMethod)` method

**Pick one** — using all three for the same bean is wasteful and confusing.

---

## Conditional Configuration

Conditional annotations register a bean only when a condition is met. This is the foundation of Spring Boot's auto-configuration.

### @ConditionalOnProperty

[`@ConditionalOnProperty`](https://docs.spring.io/spring-boot/api/java/org/springframework/boot/autoconfigure/condition/ConditionalOnProperty.html) is the most common — toggle config via properties:

```java
@Configuration
@ConditionalOnProperty(prefix = "feature", name = "new-pricing", havingValue = "true")
public class NewPricingConfig {
    @Bean
    public PricingService pricingService() {
        return new V2PricingService();
    }
}

@Configuration
@ConditionalOnProperty(prefix = "feature", name = "new-pricing", havingValue = "true",
                       matchIfMissing = false)  // Default: don't activate if absent
public class LegacyPricingConfig {
    @Bean
    public PricingService pricingService() {
        return new V1PricingService();
    }
}
```

```yaml
feature:
  new-pricing: true  # Activates V2, disables V1
```

### @ConditionalOnClass / @ConditionalOnMissingClass

Activate only when a class is on the classpath:

```java
@Configuration
@ConditionalOnClass(name = "io.lettuce.core.RedisClient")
public class RedisConfig {
    @Bean
    public ReactiveRedisTemplate<String, String> redisTemplate(...) { ... }
}
```

### @ConditionalOnBean / @ConditionalOnMissingBean

Activate only when another bean exists (or doesn't):

```java
@Configuration
public class FallbackCacheConfig {
    @Bean
    @ConditionalOnMissingBean(CacheManager.class)
    public CacheManager defaultCacheManager() {
        return new ConcurrentMapCacheManager();
    }
}
```

This pattern lets users override defaults by simply defining their own bean.

### @Profile

Activate based on Spring profile:

```java
@Configuration
@Profile("prod")
public class ProdConfig {
    @Bean
    public DataSource dataSource() {
        return new HikariDataSource();  // Real DB in prod
    }
}

@Configuration
@Profile({"local", "test"})
public class LocalConfig {
    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder().build();  // H2 in local/test
    }
}

@Configuration
@Profile("!prod")  // Active for any profile EXCEPT prod
public class NonProdConfig { ... }
```

### Custom @Conditional

Combine multiple conditions or write custom logic:

```java
public class OnLinuxCondition implements Condition {
    @Override
    public boolean matches(ConditionContext ctx, AnnotatedTypeMetadata md) {
        return System.getProperty("os.name").toLowerCase().contains("linux");
    }
}

@Configuration
@Conditional(OnLinuxCondition.class)
public class LinuxOnlyConfig { ... }
```

---

## Configuration Composition

### @Import

Import other configuration classes:

```java
@Configuration
@Import({WebClientConfig.class, CacheConfig.class, SecurityConfig.class})
public class AppConfig { }
```

Useful for organizing large applications without relying on `@ComponentScan`.

### @ComponentScan

`@SpringBootApplication` includes `@ComponentScan` for the package containing the main class. Customize when needed:

```java
@SpringBootApplication
@ComponentScan(basePackages = {
    "com.reactivespring.movies",
    "com.reactivespring.shared"
})
public class MoviesApplication { }
```

Exclude specific classes:

```java
@ComponentScan(
    basePackages = "com.reactivespring",
    excludeFilters = @ComponentScan.Filter(
        type = FilterType.ASSIGNABLE_TYPE,
        classes = LegacyService.class
    )
)
```

### @EnableXxx Patterns

Many Spring features expose `@EnableXxx` annotations that import a `@Configuration` internally:

```java
@SpringBootApplication
@EnableScheduling      // Enable @Scheduled
@EnableAsync           // Enable @Async
@EnableCaching         // Enable @Cacheable, @CacheEvict
@EnableConfigurationProperties(MoviesProperties.class)
@EnableWebFluxSecurity // (Spring Security WebFlux)
public class Application { }
```

Each of these triggers an internal `@Import` of feature-specific configuration.

---

## Auto-Configuration

[Auto-configuration](https://docs.spring.io/spring-boot/reference/features/developing-auto-configuration.html) is Spring Boot's "convention over configuration" mechanism. Including a starter dependency triggers automatic bean registration based on classpath and properties.

### Creating an Auto-Configuration

```java
@AutoConfiguration
@ConditionalOnClass(MovieClient.class)
@ConditionalOnProperty(prefix = "movies", name = "enabled", matchIfMissing = true)
@EnableConfigurationProperties(MoviesProperties.class)
public class MoviesAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public MovieClient movieClient(MoviesProperties props, WebClient.Builder builder) {
        return new MovieClient(builder.baseUrl(props.url()).build());
    }
}
```

Register it in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`:

```text
com.example.movies.autoconfigure.MoviesAutoConfiguration
```

Now any application that adds your starter gets a `MovieClient` bean automatically.

### Auto-Configuration Order

Control ordering with `@AutoConfigureBefore`, `@AutoConfigureAfter`, `@AutoConfigureOrder`:

```java
@AutoConfiguration(after = WebClientAutoConfiguration.class)
public class MoviesAutoConfiguration { ... }
```

### Disabling Auto-Configuration

Exclude specific auto-configurations:

```java
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,
    SecurityAutoConfiguration.class
})
public class Application { }
```

Or via properties:

```yaml
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

---

## Common Patterns in Reactive Apps

### WebClient Bean

```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient moviesWebClient(MoviesApiProperties props,
                                      WebClient.Builder builder) {
        return builder
            .baseUrl(props.url())
            .defaultHeader(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_VALUE)
            .filter(loggingFilter())
            .build();
    }

    private ExchangeFilterFunction loggingFilter() {
        return (request, next) -> {
            log.debug("Calling: {} {}", request.method(), request.url());
            return next.exchange(request);
        };
    }
}
```

See [WebClient Configuration](webclient-config.md) for deeper coverage.

### ObjectMapper Customization

Jackson's `ObjectMapper` is the workhorse for JSON serialization in Spring Boot. Spring Boot auto-configures one, but you often need custom behavior (Java time types, snake_case field names, tolerant deserialization, etc.). There are **three** patterns, each with a different scope.

#### Pattern 1 — Customize the Auto-Configured ObjectMapper

Use `Jackson2ObjectMapperBuilderCustomizer` when you want Spring's existing `ObjectMapper` — used by controllers and HTTP message converters — to behave differently, without replacing it:

```java
@Configuration
public class JacksonConfig {

    @Bean
    public Jackson2ObjectMapperBuilderCustomizer jacksonCustomizer() {
        return builder -> builder
            .serializationInclusion(JsonInclude.Include.NON_NULL)        // skip null fields
            .featuresToDisable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS,
                               DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES)
            .modules(new JavaTimeModule());                              // Instant, LocalDate, etc.
    }
}
```

This is the **preferred approach** for most apps — you keep Spring's `ObjectMapper` as the single source of truth and only tweak it. The customization is picked up automatically by `@RestController` serialization, `WebClient` body conversion, and anywhere else Spring uses the mapper.

#### Pattern 2 — Provide a Fully Custom ObjectMapper Bean

When you need complete control (e.g., a specific module set, non-default visibility, etc.), declare your own `@Bean ObjectMapper` — this **replaces** the auto-configured one because of `@ConditionalOnMissingBean` on the auto-config:

```java
@Configuration
public class JacksonConfig {

    @Bean
    public ObjectMapper objectMapper() {
        return JsonMapper.builder()
            .addModule(new JavaTimeModule())
            .propertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE)
            .serializationInclusion(JsonInclude.Include.NON_NULL)
            .configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false)
            .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
            .configure(MapperFeature.ACCEPT_CASE_INSENSITIVE_ENUMS, true)
            .build();
    }
}
```

Now anywhere in the app that injects `ObjectMapper` gets this instance, and Spring's HTTP message converters use it too.

Inject it like any other bean:

```java
@Service
public class AuditService {
    private final ObjectMapper objectMapper;

    public AuditService(ObjectMapper objectMapper) {   // Your custom mapper
        this.objectMapper = objectMapper;
    }

    public String serialize(Event event) throws JsonProcessingException {
        return objectMapper.writeValueAsString(event);
    }
}
```

#### Pattern 3 — A Scoped, Named ObjectMapper for a Specific Use Case

Sometimes one mapper for the whole app isn't enough — e.g., your public API uses camelCase but an internal integration uses snake_case. Register a second mapper with a distinct name and keep the default for everyone else:

```java
@Configuration
public class JacksonConfig {

    @Bean
    @Qualifier("snakeCaseMapper")
    public ObjectMapper snakeCaseMapper() {
        return JsonMapper.builder()
            .addModule(new JavaTimeModule())
            .propertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE)
            .build();
    }
}
```

Inject by qualifier:

```java
@Service
public class LegacyIntegrationClient {
    private final ObjectMapper snakeCaseMapper;

    public LegacyIntegrationClient(@Qualifier("snakeCaseMapper") ObjectMapper snakeCaseMapper) {
        this.snakeCaseMapper = snakeCaseMapper;
    }
}
```

Spring's auto-configured `ObjectMapper` remains unchanged for the rest of the app.

#### Which Pattern to Choose

| Situation | Pattern |
|-----------|---------|
| Tweak defaults (JavaTime, ignore unknown fields, exclude nulls) | Pattern 1 — `Jackson2ObjectMapperBuilderCustomizer` |
| Full control over the single app-wide mapper | Pattern 2 — replace with `@Bean ObjectMapper` |
| One app needs two mappers with different conventions | Pattern 3 — named bean + `@Qualifier` |

**Common pitfalls:**
- **`FAIL_ON_UNKNOWN_PROPERTIES = true` (the Jackson default)** breaks API evolution — when the server adds a new field, old clients throw. Disable this on client-side deserialization.
- **`ObjectMapper` is thread-safe after configuration** — never mutate it at runtime. Configure it once at bean creation and reuse.
- **Don't `new ObjectMapper()` inside services** — you lose the Spring-configured modules (JavaTime, etc.) and end up with inconsistent JSON across the app. Always inject the bean.

### Scheduler Beans

Define custom Reactor Schedulers as beans for reuse:

```java
@Configuration
public class SchedulerConfig {

    @Bean(destroyMethod = "dispose")
    public Scheduler ioScheduler() {
        return Schedulers.newBoundedElastic(
            50,                       // Thread cap
            Integer.MAX_VALUE,        // Queue size
            "io-scheduler",           // Thread name prefix
            60                        // Idle TTL seconds
        );
    }

    @Bean(destroyMethod = "dispose")
    public Scheduler cpuScheduler() {
        return Schedulers.newParallel("cpu-scheduler",
            Runtime.getRuntime().availableProcessors());
    }
}
```

Inject and use:

```java
@Service
public class DataService {
    private final Scheduler ioScheduler;

    public Mono<List<Item>> loadItems() {
        return Mono.fromCallable(() -> blockingDbQuery())
            .subscribeOn(ioScheduler);
    }
}
```

`destroyMethod = "dispose"` ensures the scheduler's threads are released cleanly on shutdown.

---

## Related

- [Externalized Configuration](externalized-config.md) — feeds @ConfigurationProperties into your @Configuration beans
- [WebClient Configuration](webclient-config.md) — full WebClient bean setup with timeouts, codecs, filters
- [Database Configuration](database-config.md) — @Configuration patterns for MongoDB, R2DBC, JPA
- [Cache Configuration](cache-config.md) — CacheManager bean configuration

## References

- [Using @Configuration — Spring Framework](https://docs.spring.io/spring-framework/reference/core/beans/java/configuration-annotation.html) — full vs lite mode, proxying semantics
- [Using @Bean — Spring Framework](https://docs.spring.io/spring-framework/reference/core/beans/java/bean-annotation.html) — @Bean method declarations, inter-bean references, scoping
- [@Configuration Javadoc](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/annotation/Configuration.html) — full attribute reference
- [Customizing the Nature of a Bean — Spring Framework](https://docs.spring.io/spring-framework/reference/core/beans/factory-nature.html) — lifecycle callbacks
- [Creating Your Own Auto-configuration — Spring Boot](https://docs.spring.io/spring-boot/reference/features/developing-auto-configuration.html) — @AutoConfiguration, condition annotations
- [@ConditionalOnProperty Javadoc](https://docs.spring.io/spring-boot/api/java/org/springframework/boot/autoconfigure/condition/ConditionalOnProperty.html) — havingValue, matchIfMissing semantics
