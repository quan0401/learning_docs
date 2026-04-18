---
title: "Database Configuration in Spring Boot — MongoDB Reactive, R2DBC, and JPA"
date: 2026-04-15
updated: 2026-04-15
tags: [database, mongodb, r2dbc, jpa, hikaricp, configuration, reactive]
---

# Database Configuration in Spring Boot — MongoDB Reactive, R2DBC, and JPA

**Date:** 2026-04-15 | **Updated:** 2026-04-15
**Tags:** `database` `mongodb` `r2dbc` `jpa` `hikaricp` `configuration` `reactive`

## Table of Contents

- [Summary](#summary)
- [Choosing a Database Stack](#choosing-a-database-stack)
- [Reactive MongoDB](#reactive-mongodb)
  - [Setup](#setup)
  - [Connection URI](#connection-uri)
  - [Programmatic Configuration](#programmatic-configuration)
  - [Custom Codecs and Converters](#custom-codecs-and-converters)
  - [Auditing](#auditing)
- [R2DBC — Reactive SQL](#r2dbc--reactive-sql)
  - [Setup-1](#setup-1)
  - [Connection Pool Tuning](#connection-pool-tuning)
  - [Programmatic Configuration-1](#programmatic-configuration-1)
  - [Custom Converters](#custom-converters)
  - [Transactions in R2DBC](#transactions-in-r2dbc)
  - [Schema Initialization](#schema-initialization)
- [JPA + HikariCP — The Blocking Stack](#jpa--hikaricp--the-blocking-stack)
  - [The Reactive Caveat](#the-reactive-caveat)
  - [Setup-2](#setup-2)
  - [HikariCP Configuration](#hikaricp-configuration)
  - [JPA / Hibernate Properties](#jpa--hibernate-properties)
  - [Multiple DataSources](#multiple-datasources)
- [Migrations with Flyway](#migrations-with-flyway)
- [Comparison Table](#comparison-table)
- [Related](#related)
- [References](#references)

---

## Summary

Spring Boot supports three primary database access patterns: [Spring Data MongoDB Reactive](https://docs.spring.io/spring-data/mongodb/reference/mongodb/configuration.html) for reactive document storage, [Spring Data R2DBC](https://docs.spring.io/spring-data/relational/reference/r2dbc.html) for reactive SQL, and traditional [Spring Data JPA](https://docs.spring.io/spring-data/jpa/reference/jpa.html) with [HikariCP](https://github.com/brettwooldridge/HikariCP/wiki/Configuration) for blocking SQL. In a reactive WebFlux application, prefer MongoDB Reactive or R2DBC; JPA is acceptable when wrapped properly with `Mono.fromCallable` + `Schedulers.boundedElastic()`.

---

## Choosing a Database Stack

```mermaid
flowchart TD
    Q1{Document or<br/>Relational?}
    Q1 -->|Document| MR[Reactive MongoDB]
    Q1 -->|Relational| Q2{Need full ORM?<br/>Lazy loading,<br/>complex relations}

    Q2 -->|Yes, blocking OK| JPA[JPA + HikariCP<br/>+ wrap with Schedulers]
    Q2 -->|No, prefer reactive| R2DBC

    style MR fill:#e8f5e9,stroke:#2e7d32
    style R2DBC fill:#e3f2fd,stroke:#1565c0
    style JPA fill:#fff3e0,stroke:#e65100
```

| Stack | Reactive? | Use When |
|-------|-----------|----------|
| **Reactive MongoDB** | Yes | Document data, reactive end-to-end |
| **R2DBC** | Yes | SQL data, reactive end-to-end, no need for full ORM |
| **JPA + HikariCP** | No (blocking) | SQL data, need ORM features (lazy loading, cascading), willing to wrap |

---

## Reactive MongoDB

### Setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb-reactive</artifactId>
</dependency>
```

### Connection URI

The simplest configuration uses a single `uri` property:

```yaml
spring:
  data:
    mongodb:
      uri: mongodb://user:pass@host1:27017,host2:27017/movies?replicaSet=rs0&authSource=admin&retryWrites=true
      auto-index-creation: true
```

Or split into individual properties:

```yaml
spring:
  data:
    mongodb:
      host: localhost
      port: 27017
      database: movies
      username: ${MONGO_USER}
      password: ${MONGO_PASSWORD}
      authentication-database: admin
      replica-set-name: rs0
```

**Connection URI options worth knowing:**

| Option | Default | Purpose |
|--------|---------|---------|
| `replicaSet` | none | Name of the replica set |
| `authSource` | matching DB | Database for credentials |
| `retryWrites` | true | Auto-retry transient writes |
| `w` | 1 | Write concern (`majority` for safety) |
| `readPreference` | primary | `primary`, `secondary`, `nearest` |
| `maxPoolSize` | 100 | Max connections in pool |
| `minPoolSize` | 0 | Min connections to maintain |
| `connectTimeoutMS` | 10000 | Connection timeout |
| `socketTimeoutMS` | 0 (unlimited) | Socket read timeout |
| `serverSelectionTimeoutMS` | 30000 | How long to wait for primary |

Production-ready URI:

```text
mongodb://user:pass@host1:27017,host2:27017,host3:27017/movies?replicaSet=rs0&authSource=admin&retryWrites=true&w=majority&readPreference=secondaryPreferred&maxPoolSize=50&serverSelectionTimeoutMS=5000
```

### Programmatic Configuration

For full control, extend `AbstractReactiveMongoConfiguration`:

```java
@Configuration
@EnableReactiveMongoRepositories(basePackages = "com.reactivespring.repository")
public class MongoConfig extends AbstractReactiveMongoConfiguration {

    @Value("${spring.data.mongodb.uri}")
    private String uri;

    @Value("${spring.data.mongodb.database}")
    private String database;

    @Override
    protected String getDatabaseName() {
        return database;
    }

    @Override
    public MongoClient reactiveMongoClient() {
        ConnectionString connectionString = new ConnectionString(uri);

        MongoClientSettings settings = MongoClientSettings.builder()
            .applyConnectionString(connectionString)
            .applyToConnectionPoolSettings(builder -> builder
                .maxSize(50)
                .minSize(5)
                .maxConnectionIdleTime(60, TimeUnit.SECONDS)
                .maxConnectionLifeTime(30, TimeUnit.MINUTES))
            .applyToSocketSettings(builder -> builder
                .connectTimeout(5, TimeUnit.SECONDS)
                .readTimeout(30, TimeUnit.SECONDS))
            .applyToServerSettings(builder -> builder
                .heartbeatFrequency(10, TimeUnit.SECONDS))
            .build();

        return MongoClients.create(settings);
    }

    @Override
    protected boolean autoIndexCreation() {
        return true;  // Auto-create @Indexed fields (dev only)
    }
}
```

### Custom Codecs and Converters

Register converters for custom types:

```java
@Override
public MongoCustomConversions customConversions() {
    return new MongoCustomConversions(List.of(
        new ZonedDateTimeReadConverter(),
        new ZonedDateTimeWriteConverter()
    ));
}

static class ZonedDateTimeWriteConverter implements Converter<ZonedDateTime, Date> {
    @Override
    public Date convert(ZonedDateTime source) {
        return Date.from(source.toInstant());
    }
}
```

### Auditing

Enable created/modified timestamps:

```java
@EnableReactiveMongoAuditing
@Configuration
public class MongoAuditingConfig {
    @Bean
    public ReactiveAuditorAware<String> auditorProvider() {
        return () -> ReactiveSecurityContextHolder.getContext()
            .map(SecurityContext::getAuthentication)
            .map(Authentication::getName)
            .defaultIfEmpty("system");
    }
}
```

```java
@Document
public class MovieInfo {
    @Id private String id;
    @CreatedDate private Instant createdAt;
    @LastModifiedDate private Instant updatedAt;
    @CreatedBy private String createdBy;
    @LastModifiedBy private String modifiedBy;
    // ...
}
```

---

## R2DBC — Reactive SQL

### Setup-1

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-r2dbc</artifactId>
</dependency>

<!-- Driver (PostgreSQL example) -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>r2dbc-postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

Available drivers:
- `org.postgresql:r2dbc-postgresql`
- `io.asyncer:r2dbc-mysql`
- `io.r2dbc:r2dbc-h2` (testing)
- `com.oracle.database.r2dbc:oracle-r2dbc`
- `io.r2dbc:r2dbc-mssql`

### Connection Pool Tuning

R2DBC uses [`r2dbc-pool`](https://github.com/r2dbc/r2dbc-pool) by default. Configure via properties:

```yaml
spring:
  r2dbc:
    url: r2dbc:postgresql://localhost:5432/movies
    username: ${DB_USER}
    password: ${DB_PASSWORD}
    pool:
      enabled: true
      initial-size: 10
      max-size: 50
      max-idle-time: 30m
      max-create-connection-time: 5s
      max-acquire-time: 5s
      max-life-time: 1h
      validation-query: SELECT 1
      validation-depth: REMOTE
```

| Property | Default | Purpose |
|----------|---------|---------|
| `initial-size` | 10 | Connections created at startup |
| `max-size` | 10 | Maximum pool size |
| `max-idle-time` | 30m | Idle connection timeout |
| `max-create-connection-time` | none | Timeout for new connections |
| `max-acquire-time` | none | Timeout to acquire from pool |
| `max-life-time` | none | Force-rotate connection age |
| `validation-query` | none | SQL to validate connections |

### Programmatic Configuration-1

```java
@Configuration
@EnableR2dbcRepositories
public class R2dbcConfig extends AbstractR2dbcConfiguration {

    @Bean
    @Override
    public ConnectionFactory connectionFactory() {
        ConnectionFactoryOptions options = ConnectionFactoryOptions.builder()
            .option(DRIVER, "postgresql")
            .option(HOST, "localhost")
            .option(PORT, 5432)
            .option(USER, "postgres")
            .option(PASSWORD, "secret")
            .option(DATABASE, "movies")
            .option(SSL, true)
            .build();

        ConnectionFactory base = ConnectionFactories.get(options);

        ConnectionPoolConfiguration poolConfig = ConnectionPoolConfiguration.builder(base)
            .initialSize(10)
            .maxSize(50)
            .maxIdleTime(Duration.ofMinutes(30))
            .maxCreateConnectionTime(Duration.ofSeconds(5))
            .maxAcquireTime(Duration.ofSeconds(5))
            .validationQuery("SELECT 1")
            .build();

        return new ConnectionPool(poolConfig);
    }
}
```

### Custom Converters

```java
@Override
protected List<Object> getCustomConverters() {
    return List.of(
        new JsonNodeReadingConverter(),
        new JsonNodeWritingConverter()
    );
}
```

### Transactions in R2DBC

```java
@Service
public class MovieService {
    private final MovieRepository repo;
    private final ReviewRepository reviewRepo;

    @Transactional
    public Mono<Movie> createMovieWithReviews(Movie movie, List<Review> reviews) {
        return repo.save(movie)
            .flatMap(saved -> reviewRepo.saveAll(reviews
                .stream()
                .map(r -> r.withMovieId(saved.getId()))
                .toList())
                .then(Mono.just(saved)));
    }
}
```

Spring's `TransactionalOperator` for programmatic transactions:

```java
@Service
public class MovieService {
    private final TransactionalOperator txOperator;

    public Mono<Movie> createMovieWithReviews(Movie movie, List<Review> reviews) {
        return repo.save(movie)
            .flatMap(saved -> reviewRepo.saveAll(reviews).then(Mono.just(saved)))
            .as(txOperator::transactional);
    }
}
```

### Schema Initialization

R2DBC does NOT support `ddl-auto`. Use Flyway/Liquibase or explicit SQL scripts:

```yaml
spring:
  sql:
    init:
      mode: always
      schema-locations: classpath:schema.sql
      data-locations: classpath:data.sql
```

For production migrations, use Flyway (see [Migrations with Flyway](#migrations-with-flyway)).

---

## JPA + HikariCP — The Blocking Stack

### The Reactive Caveat

JPA/Hibernate is **fundamentally blocking** — it holds a thread for the entire DB round-trip. In a reactive WebFlux app, you must wrap JPA calls:

```java
public Mono<Movie> getMovie(Long id) {
    return Mono.fromCallable(() -> jpaRepository.findById(id).orElseThrow())
        .subscribeOn(Schedulers.boundedElastic());
}
```

See [Wrapping Blocking JPA Calls in a Reactive Chain](../reactive-blocking-jpa-pattern.md) for the full pattern.

### Setup-2

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

HikariCP is the default JDBC connection pool — included automatically.

### HikariCP Configuration

[HikariCP](https://github.com/brettwooldridge/HikariCP/wiki/Configuration) is the de facto JDBC pool. Tune it carefully:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/movies
    username: ${DB_USER}
    password: ${DB_PASSWORD}
    driver-class-name: org.postgresql.Driver
    hikari:
      pool-name: movies-pool
      maximum-pool-size: 20            # Max connections
      minimum-idle: 5                  # Keep 5 warm
      idle-timeout: 600000             # 10 min — close idle
      max-lifetime: 1800000            # 30 min — force-rotate
      connection-timeout: 5000         # 5s to acquire
      validation-timeout: 5000
      leak-detection-threshold: 60000  # Warn if held > 1 min
      auto-commit: false
      data-source-properties:
        cachePrepStmts: true
        prepStmtCacheSize: 250
        prepStmtCacheSqlLimit: 2048
        useServerPrepStmts: true
```

**Key sizing guidance from HikariCP wiki:**

> A formula often quoted is: connections = ((core_count * 2) + effective_spindle_count). For most workloads, 10-20 connections is a good starting point — even for high-throughput apps. More connections = more lock contention in the DB.

For reactive apps wrapping JPA, size based on the `boundedElastic` scheduler thread pool (default: 10 × CPU cores).

### JPA / Hibernate Properties

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate              # never, validate, update, create, create-drop
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
        jdbc:
          batch_size: 50
          order_inserts: true
          order_updates: true
        order_updates: true
        generate_statistics: false
        query:
          fail_on_pagination_over_collection_fetch: true
          in_clause_parameter_padding: true
    show-sql: false                   # Use logging instead
    open-in-view: false               # CRITICAL — disable in production
```

**`open-in-view: false`** is critical. The default (`true`) keeps the Hibernate session open for the entire HTTP request, encouraging lazy-loading anti-patterns. Disable it.

**`ddl-auto`** values:

| Value | Effect | Use When |
|-------|--------|----------|
| `none` | Do nothing | Production with managed migrations |
| `validate` | Verify schema matches | Production (paired with Flyway) |
| `update` | Add missing columns/tables (no drops) | Local dev only |
| `create` | Drop and recreate at startup | Tests |
| `create-drop` | Recreate + drop on shutdown | Tests with embedded DB |

### Multiple DataSources

For services with multiple databases:

```java
@Configuration
public class MultiDataSourceConfig {

    @Bean
    @Primary
    @ConfigurationProperties("spring.datasource.primary")
    public DataSourceProperties primaryDataSourceProperties() {
        return new DataSourceProperties();
    }

    @Bean
    @Primary
    @ConfigurationProperties("spring.datasource.primary.hikari")
    public DataSource primaryDataSource(
            @Qualifier("primaryDataSourceProperties") DataSourceProperties props) {
        return props.initializeDataSourceBuilder().build();
    }

    @Bean
    @ConfigurationProperties("spring.datasource.audit")
    public DataSourceProperties auditDataSourceProperties() {
        return new DataSourceProperties();
    }

    @Bean
    @ConfigurationProperties("spring.datasource.audit.hikari")
    public DataSource auditDataSource(
            @Qualifier("auditDataSourceProperties") DataSourceProperties props) {
        return props.initializeDataSourceBuilder().build();
    }
}
```

```yaml
spring:
  datasource:
    primary:
      url: jdbc:postgresql://primary:5432/main
      username: ${PRIMARY_DB_USER}
      password: ${PRIMARY_DB_PASS}
      hikari:
        maximum-pool-size: 20
    audit:
      url: jdbc:postgresql://audit:5432/audit
      username: ${AUDIT_DB_USER}
      password: ${AUDIT_DB_PASS}
      hikari:
        maximum-pool-size: 5
```

---

## Migrations with Flyway

For both R2DBC and JPA stacks, use [Flyway](https://flywaydb.org/) for schema migrations:

```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-database-postgresql</artifactId>
</dependency>
```

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
    validate-on-migrate: true
```

Place SQL migrations in `src/main/resources/db/migration/`:

```text
V1__create_movies_table.sql
V2__add_index_on_movie_year.sql
V3__add_reviews_table.sql
```

For R2DBC apps, Flyway uses a separate JDBC connection (Flyway is blocking). Add the JDBC driver as a dependency even if your app uses R2DBC at runtime:

```xml
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

---

## Comparison Table

| Aspect | Reactive MongoDB | R2DBC | JPA + HikariCP |
|--------|-----------------|-------|----------------|
| Blocking | No | No | **Yes** (must wrap) |
| Lazy loading | N/A | No | Yes |
| Schema generation | Auto-index only | No | Yes (`ddl-auto`) |
| Relations | Manual (DBRef or aggregation) | Manual joins | Full ORM |
| Caching | Manual | Manual | First/second-level cache |
| Transactions | Limited (4.0+ replica sets) | Yes | Yes (mature) |
| Maturity | Stable | ~5 years | 20+ years |
| Connection pool | MongoDB driver | r2dbc-pool | HikariCP |
| Spring Data return types | `Mono`/`Flux` | `Mono`/`Flux` | `T` / `Optional` / `List` |
| Best with | Spring WebFlux | Spring WebFlux | Spring MVC |

---

## Related

- [Wrapping Blocking JPA Calls in a Reactive Chain](../reactive-blocking-jpa-pattern.md) — required pattern when using JPA in WebFlux
- [Externalized Configuration](externalized-config.md) — externalizing connection strings and credentials
- [Java @Configuration Classes](java-bean-config.md) — custom DataSource/ConnectionFactory beans
- [Cache Configuration](cache-config.md) — caching DB query results to reduce load
- [Reactive Observability](../reactive-observability.md) — DB metrics auto-instrumented via Micrometer

## References

- [Connecting to MongoDB — Spring Data MongoDB](https://docs.spring.io/spring-data/mongodb/reference/mongodb/configuration.html) — reactive client setup, AbstractReactiveMongoConfiguration
- [R2DBC — Spring Data Relational](https://docs.spring.io/spring-data/relational/reference/r2dbc.html) — R2DBC configuration, DatabaseClient, entity mapping
- [Data Access with R2DBC — Spring Framework](https://docs.spring.io/spring-framework/reference/data-access/r2dbc.html) — framework-level R2DBC support
- [HikariCP Configuration Wiki](https://github.com/brettwooldridge/HikariCP/wiki/Configuration) — all HikariCP pool properties and tuning guidance
- [Data Access How-To — Spring Boot](https://docs.spring.io/spring-boot/how-to/data-access.html) — multiple data sources, HikariCP tuning
- [Spring Data JPA Reference](https://docs.spring.io/spring-data/jpa/reference/jpa.html) — JPA repositories, query methods, auditing
