# NestJS for Spring Boot Developers

**Updated:** 2026-04-24

> Map your Spring mental model to Nest.

If you already think in `@RestController`, `@Service`, `@Configuration`, `SecurityFilterChain`, and `@ControllerAdvice`, NestJS will feel immediately familiar. Both frameworks are opinionated, decorator-driven, and built around dependency injection. NestJS was explicitly inspired by Angular **and Spring**, so the mapping is almost 1:1. For the Java side of the same mental model, keep [Spring Fundamentals](../../java/spring-fundamentals.md) nearby.

---

## 1. Architecture Overview

Both frameworks share the same philosophy: convention over configuration, layered architecture, and inversion of control at the core.

```
Spring Boot                          NestJS
──────────────────────────           ──────────────────────────
@SpringBootApplication               AppModule (root)
  ├── @Configuration                   ├── @Module()
  │     @ComponentScan                 │     imports / providers
  ├── @RestController                  ├── @Controller()
  │     @GetMapping                    │     @Get()
  │     @PostMapping                   │     @Post()
  ├── @Service                         ├── @Injectable()
  ├── @Repository                      ├── @Injectable() (repos)
  ├── SecurityFilterChain              ├── Guards
  ├── HandlerInterceptor               ├── Interceptors
  ├── @ControllerAdvice                ├── Exception Filters
  ├── Filter (Jakarta)                 ├── Middleware
  └── @ConfigurationProperties         └── ConfigModule
```

**Key difference**: Spring Boot auto-scans the classpath and registers beans automatically. NestJS requires explicit declaration inside `@Module()` — nothing is global unless you make it so.

---

## 2. Modules <-> @Configuration

### Spring: @Configuration + @ComponentScan

```java
@Configuration
@ComponentScan(basePackages = "com.example.users")
public class UserConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

Spring discovers beans via classpath scanning. `@ComponentScan` defines the search scope. Any `@Component`, `@Service`, or `@Repository` in that package is registered automatically.

### NestJS: @Module

```typescript
@Module({
  imports: [TypeOrmModule.forFeature([User])],
  controllers: [UserController],
  providers: [UserService, PasswordService],
  exports: [UserService], // expose to other modules
})
export class UserModule {}
```

**Critical distinction**: Providers declared in a NestJS module are scoped to that module. Other modules cannot inject `UserService` unless `UserModule` exports it **and** the consuming module imports `UserModule`. Spring has no equivalent restriction — any bean is globally available by default.

To make a provider truly global in NestJS:

```typescript
@Global()
@Module({
  providers: [LoggerService],
  exports: [LoggerService],
})
export class LoggerModule {}
```

---

## 3. Providers <-> @Service / @Bean

### Constructor Injection (Both Frameworks)

**Spring:**

```java
@Service
public class OrderService {

    private final UserService userService;
    private final OrderRepository orderRepository;

    public OrderService(UserService userService, OrderRepository orderRepository) {
        this.userService = userService;
        this.orderRepository = orderRepository;
    }
}
```

**NestJS:**

```typescript
@Injectable()
export class OrderService {
  constructor(
    private readonly userService: UserService,
    private readonly orderRepository: OrderRepository,
  ) {}
}
```

Both use constructor injection as the default. NestJS resolves dependencies by TypeScript type metadata (via `reflect-metadata`), similar to how Spring resolves by type.

### Custom Providers <-> @Bean Methods

**Spring — factory via @Bean:**

```java
@Configuration
public class AppConfig {

    @Bean
    public RestClient restClient() {
        return RestClient.builder()
            .baseUrl("https://api.example.com")
            .defaultHeader("Authorization", "Bearer " + apiKey)
            .build();
    }
}
```

**NestJS — useFactory:**

```typescript
@Module({
  providers: [
    {
      provide: 'HTTP_CLIENT',
      useFactory: (configService: ConfigService) => {
        return axios.create({
          baseURL: configService.get('API_BASE_URL'),
          headers: { Authorization: `Bearer ${configService.get('API_KEY')}` },
        });
      },
      inject: [ConfigService],
    },
  ],
})
export class AppModule {}
```

Other custom provider types in NestJS:

| NestJS Provider | Spring Equivalent |
|-----------------|-------------------|
| `useClass` | `@Bean` returning `new MyImpl()` |
| `useValue` | `@Bean` returning a literal value |
| `useFactory` | `@Bean` method with logic |
| `useExisting` | Aliasing — `@Qualifier` + `@Bean` delegation |

---

## 4. Controllers <-> @RestController

### Spring: Full CRUD Controller

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping
    public List<UserDto> findAll() {
        return userService.findAll();
    }

    @GetMapping("/{id}")
    public UserDto findById(@PathVariable Long id) {
        return userService.findById(id);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public UserDto create(@Valid @RequestBody CreateUserDto dto) {
        return userService.create(dto);
    }

    @PutMapping("/{id}")
    public UserDto update(@PathVariable Long id, @Valid @RequestBody UpdateUserDto dto) {
        return userService.update(id, dto);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        userService.delete(id);
    }
}
```

### NestJS: Full CRUD Controller

```typescript
@Controller('api/users')
export class UserController {
  constructor(private readonly userService: UserService) {}

  @Get()
  findAll(): Promise<UserDto[]> {
    return this.userService.findAll();
  }

  @Get(':id')
  findById(@Param('id', ParseIntPipe) id: number): Promise<UserDto> {
    return this.userService.findById(id);
  }

  @Post()
  @HttpCode(HttpStatus.CREATED)
  create(@Body() dto: CreateUserDto): Promise<UserDto> {
    return this.userService.create(dto);
  }

  @Put(':id')
  update(
    @Param('id', ParseIntPipe) id: number,
    @Body() dto: UpdateUserDto,
  ): Promise<UserDto> {
    return this.userService.update(id, dto);
  }

  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)
  remove(@Param('id', ParseIntPipe) id: number): Promise<void> {
    return this.userService.delete(id);
  }
}
```

**Decorator mapping:**

| Spring | NestJS |
|--------|--------|
| `@GetMapping` | `@Get()` |
| `@PostMapping` | `@Post()` |
| `@PathVariable` | `@Param()` |
| `@RequestBody` | `@Body()` |
| `@RequestParam` | `@Query()` |
| `@RequestHeader` | `@Headers()` |
| `@ResponseStatus` | `@HttpCode()` |

---

## 5. Pipes <-> @Valid / Bean Validation

### Spring: Jakarta Bean Validation

```java
public class CreateUserDto {

    @NotBlank(message = "Name is required")
    private String name;

    @Email(message = "Must be a valid email")
    @NotBlank
    private String email;

    @Min(value = 18, message = "Must be at least 18")
    private int age;

    // getters, setters
}
```

Controller triggers validation with `@Valid`:

```java
@PostMapping
public UserDto create(@Valid @RequestBody CreateUserDto dto) {
    return userService.create(dto);
}
```

### NestJS: class-validator + ValidationPipe

```typescript
import { IsNotEmpty, IsEmail, Min } from 'class-validator';

export class CreateUserDto {
  @IsNotEmpty({ message: 'Name is required' })
  name: string;

  @IsEmail({}, { message: 'Must be a valid email' })
  @IsNotEmpty()
  email: string;

  @Min(18, { message: 'Must be at least 18' })
  age: number;
}
```

Enable globally in `main.ts`:

```typescript
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,        // strip unknown properties
    forbidNonWhitelisted: true, // throw on unknown properties
    transform: true,        // auto-transform payloads to DTO instances
  }),
);
```

`whitelist: true` has no direct Spring equivalent — Spring accepts unknown JSON fields silently by default (Jackson ignores them). NestJS lets you reject or strip them explicitly.

### Custom Pipes <-> Custom Validators

**Spring — custom ConstraintValidator:**

```java
@Constraint(validatedBy = UniqueEmailValidator.class)
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface UniqueEmail {
    String message() default "Email already exists";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class UniqueEmailValidator implements ConstraintValidator<UniqueEmail, String> {
    @Autowired private UserRepository userRepository;

    @Override
    public boolean isValid(String email, ConstraintValidatorContext ctx) {
        return !userRepository.existsByEmail(email);
    }
}
```

**NestJS — custom pipe:**

```typescript
@Injectable()
export class ParseDatePipe implements PipeTransform<string, Date> {
  transform(value: string, metadata: ArgumentMetadata): Date {
    const date = new Date(value);
    if (isNaN(date.getTime())) {
      throw new BadRequestException(`"${value}" is not a valid date`);
    }
    return date;
  }
}

// Usage
@Get('events')
findByDate(@Query('date', ParseDatePipe) date: Date) { ... }
```

---

## 6. Guards <-> Spring Security Filters

### Spring: SecurityFilterChain + @PreAuthorize

```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(s -> s.sessionCreationPolicy(STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .build();
    }
}
```

Method-level:

```java
@PreAuthorize("hasRole('ADMIN')")
@DeleteMapping("/{id}")
public void delete(@PathVariable Long id) { ... }
```

### NestJS: Guards + Custom Decorators

**JWT Auth Guard:**

```typescript
@Injectable()
export class JwtAuthGuard implements CanActivate {
  constructor(private readonly jwtService: JwtService) {}

  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const token = this.extractToken(request);
    if (!token) throw new UnauthorizedException();

    try {
      request.user = this.jwtService.verify(token);
      return true;
    } catch {
      throw new UnauthorizedException('Invalid token');
    }
  }

  private extractToken(request: Request): string | null {
    const [type, token] = request.headers.authorization?.split(' ') ?? [];
    return type === 'Bearer' ? token : null;
  }
}
```

**Role-based access:**

```typescript
// Custom decorator — like defining a custom annotation in Spring
export const Roles = (...roles: string[]) => SetMetadata('roles', roles);

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<string[]>('roles', [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!requiredRoles) return true;

    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some((role) => user.roles?.includes(role));
  }
}
```

**Usage:**

```typescript
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles('admin')
@Delete(':id')
remove(@Param('id') id: string) { ... }
```

**Key difference**: Spring Security is a comprehensive framework with built-in OAuth2, CSRF, session management, and filter chains. NestJS guards are simpler building blocks — you compose authentication from `@nestjs/passport`, `@nestjs/jwt`, and custom guards. More assembly, more flexibility, less magic.

---

## 7. Interceptors <-> HandlerInterceptor

### Spring: HandlerInterceptor

```java
@Component
public class LoggingInterceptor implements HandlerInterceptor {

    private static final Logger log = LoggerFactory.getLogger(LoggingInterceptor.class);

    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
        req.setAttribute("startTime", System.currentTimeMillis());
        log.info("→ {} {}", req.getMethod(), req.getRequestURI());
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest req, HttpServletResponse res,
                           Object handler, ModelAndView mav) {
        long duration = System.currentTimeMillis() - (long) req.getAttribute("startTime");
        log.info("← {} {} [{}ms]", req.getMethod(), req.getRequestURI(), duration);
    }
}
```

### NestJS: NestInterceptor

```typescript
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  private readonly logger = new Logger(LoggingInterceptor.name);

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const { method, url } = request;
    const start = Date.now();

    this.logger.log(`→ ${method} ${url}`);

    return next.handle().pipe(
      tap(() => {
        this.logger.log(`← ${method} ${url} [${Date.now() - start}ms]`);
      }),
    );
  }
}
```

NestJS interceptors wrap the handler in an RxJS `Observable` pipeline, giving you `tap`, `map`, `catchError`, and `timeout` operators for response transformation. This is more powerful than Spring's split `preHandle`/`postHandle` model — you get a single function with full control over both request and response.

**Common interceptor patterns:**

| Pattern | Spring | NestJS |
|---------|--------|--------|
| Logging / timing | `HandlerInterceptor` | `tap()` in interceptor |
| Response wrapping | `ResponseBodyAdvice` | `map()` in interceptor |
| Caching | `@Cacheable` | `of(cachedValue)` short-circuit |
| Timeout | servlet timeout config | `timeout()` operator |

---

## 8. Exception Filters <-> @ControllerAdvice

### Spring: @ControllerAdvice + @ExceptionHandler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(EntityNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(EntityNotFoundException ex) {
        return new ErrorResponse("NOT_FOUND", ex.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(MethodArgumentNotValidException ex) {
        List<String> errors = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .toList();
        return new ErrorResponse("VALIDATION_ERROR", errors.toString());
    }

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleGeneral(Exception ex) {
        return new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred");
    }
}
```

### NestJS: @Catch + ExceptionFilter

```typescript
@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(GlobalExceptionFilter.name);

  catch(exception: unknown, host: ArgumentsHost): void {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    const { status, body } = this.mapException(exception);

    this.logger.error(`${request.method} ${request.url} → ${status}`, exception);

    response.status(status).json({
      ...body,
      timestamp: new Date().toISOString(),
      path: request.url,
    });
  }

  private mapException(exception: unknown): { status: number; body: Record<string, any> } {
    if (exception instanceof HttpException) {
      return {
        status: exception.getStatus(),
        body: { message: exception.message },
      };
    }
    return {
      status: HttpStatus.INTERNAL_SERVER_ERROR,
      body: { message: 'An unexpected error occurred' },
    };
  }
}
```

Register globally:

```typescript
// main.ts
app.useGlobalFilters(new GlobalExceptionFilter());
```

**NestJS built-in exceptions** map to Spring's `ResponseStatusException`:

| NestJS | Spring | HTTP Status |
|--------|--------|-------------|
| `NotFoundException` | `ResponseStatusException(NOT_FOUND, ...)` | 404 |
| `BadRequestException` | `ResponseStatusException(BAD_REQUEST, ...)` | 400 |
| `UnauthorizedException` | `ResponseStatusException(UNAUTHORIZED, ...)` | 401 |
| `ForbiddenException` | `AccessDeniedException` | 403 |
| `ConflictException` | `ResponseStatusException(CONFLICT, ...)` | 409 |

---

## 9. Middleware <-> Filter (Jakarta)

### Spring: Jakarta @WebFilter

```java
@Component
@Order(1)
public class CorrelationIdFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                     FilterChain chain) throws ServletException, IOException {
        String correlationId = Optional.ofNullable(req.getHeader("X-Correlation-ID"))
            .orElse(UUID.randomUUID().toString());
        res.setHeader("X-Correlation-ID", correlationId);
        MDC.put("correlationId", correlationId);

        try {
            chain.doFilter(req, res);
        } finally {
            MDC.remove("correlationId");
        }
    }
}
```

### NestJS: Middleware

```typescript
@Injectable()
export class CorrelationIdMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction): void {
    const correlationId = (req.headers['x-correlation-id'] as string) ?? randomUUID();
    res.setHeader('X-Correlation-ID', correlationId);
    req['correlationId'] = correlationId;
    next();
  }
}
```

Register in the module:

```typescript
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer): void {
    consumer
      .apply(CorrelationIdMiddleware)
      .forRoutes('*');
  }
}
```

### When to Use What in NestJS

| Layer | Purpose | Spring Equivalent |
|-------|---------|-------------------|
| **Middleware** | Raw request/response manipulation (CORS, compression, correlation IDs) | Jakarta `Filter` |
| **Guards** | Authentication, authorization, access control | Spring Security filters, `@PreAuthorize` |
| **Interceptors** | Logging, response transformation, caching, timing | `HandlerInterceptor`, `ResponseBodyAdvice` |
| **Pipes** | Input validation, transformation | `@Valid`, `@InitBinder`, `Converter` |
| **Exception Filters** | Error handling, error response formatting | `@ControllerAdvice` |

**Execution order**: Middleware -> Guards -> Interceptors (before) -> Pipes -> Handler -> Interceptors (after) -> Exception Filters (on error)

This mirrors Spring's: Filter -> Security Filter Chain -> HandlerInterceptor.preHandle -> Validation -> Handler -> HandlerInterceptor.postHandle -> @ControllerAdvice (on error)

---

## 10. Configuration <-> @ConfigurationProperties

### Spring: application.yml + @ConfigurationProperties

```yaml
# application.yml
app:
  jwt:
    secret: ${JWT_SECRET}
    expiration: 3600
  database:
    pool-size: 10
```

```java
@ConfigurationProperties(prefix = "app.jwt")
@Validated
public record JwtProperties(
    @NotBlank String secret,
    @Min(60) int expiration
) {}
```

### NestJS: ConfigModule + @nestjs/config

```typescript
// app.module.ts
@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      envFilePath: [`.env.${process.env.NODE_ENV}`, '.env'],
      validationSchema: Joi.object({
        JWT_SECRET: Joi.string().required(),
        JWT_EXPIRATION: Joi.number().default(3600),
        DB_POOL_SIZE: Joi.number().default(10),
      }),
    }),
  ],
})
export class AppModule {}
```

**Typed config with namespaces** (closest to `@ConfigurationProperties`):

```typescript
// jwt.config.ts
export const jwtConfig = registerAs('jwt', () => ({
  secret: process.env.JWT_SECRET,
  expiration: parseInt(process.env.JWT_EXPIRATION ?? '3600', 10),
}));

// Inject typed config
@Injectable()
export class AuthService {
  constructor(
    @Inject(jwtConfig.KEY)
    private readonly jwt: ConfigType<typeof jwtConfig>,
  ) {}

  generateToken(userId: string): string {
    return this.jwtService.sign({ sub: userId }, {
      secret: this.jwt.secret,
      expiresIn: this.jwt.expiration,
    });
  }
}
```

| Spring | NestJS |
|--------|--------|
| `application.yml` / `application.properties` | `.env` files + `ConfigModule` |
| `@ConfigurationProperties` | `registerAs()` namespaced config |
| Profile-based config (`application-{profile}.yml`) | `.env.development`, `.env.production` |
| `@Value("${key}")` | `configService.get('key')` |
| `@Validated` on config class | Joi or `class-validator` schema on `ConfigModule` |

---

## 11. Comprehensive Mapping Table

| Concern | Spring Boot | NestJS |
|---------|-------------|--------|
| **DI Container** | Spring IoC (`ApplicationContext`) | NestJS IoC (`ModuleRef`) |
| **Module System** | `@Configuration` + `@ComponentScan` | `@Module()` |
| **Service Layer** | `@Service` / `@Component` | `@Injectable()` |
| **REST Controllers** | `@RestController` | `@Controller()` |
| **Request Validation** | `@Valid` + Jakarta Bean Validation | `ValidationPipe` + `class-validator` |
| **Authentication** | Spring Security + `SecurityFilterChain` | Passport + Guards |
| **Authorization** | `@PreAuthorize`, `@Secured` | Custom Guards + `@SetMetadata` |
| **Interceptors** | `HandlerInterceptor` | `NestInterceptor` (RxJS-based) |
| **Exception Handling** | `@ControllerAdvice` + `@ExceptionHandler` | `@Catch()` + `ExceptionFilter` |
| **Middleware** | Jakarta `Filter` | `NestMiddleware` (Express/Fastify) |
| **Configuration** | `@ConfigurationProperties` | `@nestjs/config` + `registerAs()` |
| **Lifecycle Hooks** | `@PostConstruct`, `@PreDestroy`, `SmartLifecycle` | `OnModuleInit`, `OnModuleDestroy`, `OnApplicationBootstrap`, `OnApplicationShutdown` |
| **Testing (Unit)** | JUnit 5 + Mockito | Jest + `@nestjs/testing` (`Test.createTestingModule`) |
| **Testing (Integration)** | `@SpringBootTest` + `MockMvc` | Supertest + `INestApplication` |
| **Testing (E2E)** | `@SpringBootTest(webEnvironment = RANDOM_PORT)` | `@nestjs/testing` + Supertest |
| **Scheduling** | `@Scheduled` + `@EnableScheduling` | `@Cron()` + `@nestjs/schedule` |
| **Caching** | `@Cacheable` + `@EnableCaching` | `@nestjs/cache-manager` + `@CacheKey()` |
| **Health Checks** | Spring Boot Actuator (`/actuator/health`) | `@nestjs/terminus` (`HealthCheckService`) |
| **Microservices** | Spring Cloud (Eureka, Config Server, Gateway) | `@nestjs/microservices` (TCP, Redis, NATS, gRPC, Kafka) |
| **WebSockets** | Spring WebSocket + STOMP | `@WebSocketGateway` + `@SubscribeMessage` |
| **ORM** | Spring Data JPA / Hibernate | TypeORM, Prisma, MikroORM |
| **API Documentation** | SpringDoc OpenAPI (`springdoc-openapi`) | `@nestjs/swagger` (decorators generate spec) |
| **Event System** | `ApplicationEventPublisher` + `@EventListener` | `@nestjs/event-emitter` |
| **Queues** | Spring AMQP, Spring Kafka | `@nestjs/bull` (Redis-backed) |
| **GraphQL** | Spring for GraphQL | `@nestjs/graphql` (code-first or schema-first) |
| **Serialization** | Jackson (`@JsonProperty`, mixins) | `class-transformer` (`@Exclude`, `@Expose`, `@Transform`) |
| **Logging** | SLF4J + Logback | Built-in `Logger` or Pino / Winston |
| **Graceful Shutdown** | `SpringApplication.exit()` + shutdown hooks | `app.enableShutdownHooks()` + lifecycle interfaces |

---

## Related

- [Spring Fundamentals](../../java/spring-fundamentals.md) — IoC container, bean lifecycle, proxies, and auto-configuration on the Java side.
- [Filters, Interceptors, and the Request Processing Pipeline](../../java/web-layer/filters-and-interceptors.md) — the Spring equivalents of middleware and interceptors.
- [Exception Handling in Spring Boot — MVC and WebFlux](../../java/validation/exception-handling.md) — the `@ControllerAdvice` side of Nest exception filters.
- [Jakarta Validation in Spring Boot](../../java/validation/bean-validation.md) — `@Valid` and constraint annotations compared to `ValidationPipe` and `class-validator`.

## References

- [NestJS Documentation](https://docs.nestjs.com/) — official docs, comprehensive
- [NestJS Fundamentals](https://docs.nestjs.com/fundamentals/custom-providers) — custom providers, async providers, scopes
- [NestJS Security](https://docs.nestjs.com/security/authentication) — authentication, authorization, CORS, CSRF
- [NestJS Techniques](https://docs.nestjs.com/techniques/configuration) — configuration, validation, caching, scheduling
- [NestJS Interceptors](https://docs.nestjs.com/interceptors) — RxJS-based request/response pipeline
- [NestJS Exception Filters](https://docs.nestjs.com/exception-filters) — global and scoped error handling
- [Spring Boot Reference](https://docs.spring.io/spring-boot/reference/) — official Spring Boot docs
- [Spring Security Reference](https://docs.spring.io/spring-security/reference/) — authentication, authorization, filter chains
- [class-validator](https://github.com/typestack/class-validator) — decorator-based validation for TypeScript
- [class-transformer](https://github.com/typestack/class-transformer) — serialization/deserialization decorators
