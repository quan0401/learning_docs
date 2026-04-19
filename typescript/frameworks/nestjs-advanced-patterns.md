# NestJS Advanced Patterns

> Build a multi-tenant API with per-request database selection.

You have read [NestJS for Spring Boot Developers](nestjs-for-spring-devs.md) and can map controllers, providers, modules, guards, and pipes to their Spring equivalents. This doc goes deeper: custom decorators, dynamic modules, request-scoped providers, microservice transports, the CQRS module, lifecycle hooks, testing, and performance. The running example is a SaaS platform where every HTTP request targets a specific tenant's database.

---

## 1. Custom Decorators

### 1.1 Parameter Decorators with `createParamDecorator`

Spring extracts request-level data with `@AuthenticationPrincipal` or custom `HandlerMethodArgumentResolver`. NestJS uses `createParamDecorator`.

```typescript
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (data: string | undefined, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user; // set by AuthGuard earlier in the pipeline
    return data ? user?.[data] : user;
  },
);
```

The optional `data` argument lets callers pick a single property:

```typescript
@Get('profile')
getProfile(@CurrentUser() user: UserPayload) { return user; }

@Get('email')
getEmail(@CurrentUser('email') email: string) { return { email }; }
```

### 1.2 Metadata Decorators with `SetMetadata` + Reflector

Spring pairs custom annotations with `HandlerInterceptor` or `@PreAuthorize`. NestJS pairs `SetMetadata` with `Reflector` inside guards.

```typescript
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

```typescript
@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(ctx: ExecutionContext): boolean {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      ctx.getHandler(),
      ctx.getClass(),
    ]);
    if (isPublic) return true;
    return !!ctx.switchToHttp().getRequest().user;
  }
}
```

Now any endpoint tagged `@Public()` skips authentication.

### 1.3 Composing Decorators

When multiple decorators always appear together, combine them with `applyDecorators`:

```typescript
import { applyDecorators, SetMetadata, UseGuards } from '@nestjs/common';

export const Roles = (...roles: string[]) =>
  applyDecorators(SetMetadata('roles', roles), UseGuards(RolesGuard));
```

Spring equivalent: a meta-annotation like `@AdminOnly` combining `@PreAuthorize("hasRole('ADMIN')")` with other annotations.

---

## 2. Dynamic Modules

### 2.1 The `forRoot` / `forRootAsync` / `register` Pattern

Spring's `@ConditionalOnProperty` and `@ConfigurationProperties` produce different beans based on environment. NestJS achieves the same with dynamic modules — a module whose providers are determined at import time.

**Convention**:
- `forRoot()` — static options object. Root import.
- `forRootAsync()` — factory/`useClass`/`useExisting`. For config depending on other providers.
- `register()` / `forFeature()` — per-feature configuration (like `TypeOrmModule.forFeature([Entity])`).

### 2.2 Building a Configurable `DatabaseModule`

```typescript
export interface DatabaseModuleOptions {
  url: string;
  logging?: boolean;
}

export interface DatabaseModuleAsyncOptions {
  imports?: any[];
  useFactory: (...args: any[]) => Promise<DatabaseModuleOptions> | DatabaseModuleOptions;
  inject?: any[];
}
```

```typescript
@Global()
@Module({})
export class DatabaseModule {
  static forRoot(options: DatabaseModuleOptions): DynamicModule {
    return {
      module: DatabaseModule,
      providers: [
        { provide: DATABASE_OPTIONS, useValue: options },
        PrismaService,
      ],
      exports: [PrismaService],
    };
  }

  static forRootAsync(opts: DatabaseModuleAsyncOptions): DynamicModule {
    return {
      module: DatabaseModule,
      imports: opts.imports ?? [],
      providers: [
        { provide: DATABASE_OPTIONS, useFactory: opts.useFactory, inject: opts.inject ?? [] },
        PrismaService,
      ],
      exports: [PrismaService],
    };
  }
}
```

Consumer side — config-driven wiring:

```typescript
@Module({
  imports: [
    ConfigModule.forRoot(),
    DatabaseModule.forRootAsync({
      imports: [ConfigModule],
      useFactory: (config: ConfigService) => ({
        url: config.getOrThrow<string>('DATABASE_URL'),
        logging: config.get<boolean>('DB_LOGGING', false),
      }),
      inject: [ConfigService],
    }),
  ],
})
export class AppModule {}
```

| Spring | NestJS |
|--------|--------|
| `@ConditionalOnProperty("db.type")` | `forRoot({ type: 'postgres' })` |
| `@ConfigurationProperties(prefix = "db")` | `forRootAsync({ useFactory: ... })` |
| `@EnableJpaRepositories(basePackages = ...)` | `TypeOrmModule.forFeature([...])` |

---

## 3. Request-Scoped Providers (Multi-Tenancy)

### 3.1 When You Need Them

By default every NestJS provider is a **singleton** — created once, shared across all requests (same as Spring's default scope). Request-scoped providers create a **new instance per request**:

- Multi-tenant database selection (our running example)
- Per-request audit context (current user, correlation ID)
- Request-level caching

### 3.2 Performance Implications

Request scope **breaks the singleton chain**. If a singleton service injects a request-scoped provider, NestJS forces the singleton into request scope too.

```
Singleton A  →  injects Request-scoped B
Result: A also becomes request-scoped (new instance per request)
```

Mitigation: keep request-scoped providers at the edge (middleware, controllers). Use `ModuleRef` to access them from singletons without scope contamination.

### 3.3 The Multi-Tenant Database Pattern

Full flow: middleware extracts tenant -> request-scoped Prisma client connects to the right database.

**Step 1: Middleware extracts tenant ID**

```typescript
@Injectable()
export class TenantMiddleware implements NestMiddleware {
  use(req: Request, _res: Response, next: NextFunction) {
    const tenantId = req.headers['x-tenant-id'] as string;
    if (!tenantId) throw new BadRequestException('Missing X-Tenant-Id header');
    req['tenantId'] = tenantId;
    next();
  }
}
```

**Step 2: Request-scoped tenant-aware Prisma client**

```typescript
@Injectable({ scope: Scope.REQUEST })
export class TenantPrismaService extends PrismaClient {
  constructor(@Inject(REQUEST) private readonly request: Request) {
    const tenantId = request['tenantId'];
    super({ datasources: { db: { url: `postgresql://host:5432/tenant_${tenantId}` } } });
  }

  async onModuleInit() { await this.$connect(); }
  async onModuleDestroy() { await this.$disconnect(); }
}
```

**Step 3: Wire into the module**

```typescript
@Module({
  providers: [TenantPrismaService, OrderService],
  exports: [TenantPrismaService],
})
export class TenantModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(TenantMiddleware).forRoutes('*');
  }
}
```

Now any service injecting `TenantPrismaService` automatically queries the correct tenant's database.

> **Connection pooling caveat**: Creating a new `PrismaClient` per request is expensive. In production, use a `Map<string, PrismaClient>` singleton keyed by tenant ID and reuse clients.

---

## 4. Microservices Transport

### 4.1 Hybrid Application

NestJS can run HTTP and microservice transports in the same process. Spring Boot achieves similar with embedded Kafka listeners alongside REST controllers.

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.connectMicroservice<MicroserviceOptions>({
    transport: Transport.REDIS,
    options: { host: 'localhost', port: 6379 },
  });

  await app.startAllMicroservices();
  await app.listen(3000);
}
```

### 4.2 Available Transports

| Transport | Use Case | Spring Equivalent |
|-----------|----------|-------------------|
| TCP | Internal service calls | Spring Cloud OpenFeign |
| Redis | Pub/sub events, simple RPC | Spring Data Redis |
| NATS | High-throughput messaging | — |
| Kafka | Event streaming | Spring Kafka `@KafkaListener` |
| gRPC | Typed service contracts | `grpc-spring-boot-starter` |
| RabbitMQ | Task queues, routing | Spring AMQP `@RabbitListener` |

### 4.3 `@MessagePattern` and `@EventPattern`

- **Request-response** (`@MessagePattern`) — client sends and waits for a reply.
- **Event-based** (`@EventPattern`) — fire-and-forget.

```typescript
@Controller()
export class OrdersMicroserviceController {
  @MessagePattern('order.findById')
  findOrderById(@Payload() id: string) {
    return { id, status: 'shipped' }; // returned to caller
  }

  @EventPattern('order.created')
  handleOrderCreated(@Payload() data: { orderId: string; userId: string }) {
    console.log(`Order ${data.orderId} created`); // no response
  }
}
```

### 4.4 Client-Side: Sending Messages

```typescript
@Injectable()
export class NotificationService {
  constructor(@Inject('ORDER_SERVICE') private readonly client: ClientProxy) {}

  async getOrder(id: string) {
    return firstValueFrom(this.client.send('order.findById', id));
  }

  emitOrderCreated(orderId: string, userId: string) {
    this.client.emit('order.created', { orderId, userId });
  }
}
```

Register the client proxy in a module via `ClientsModule.register()` with the same transport config as the server.

---

## 5. CQRS Module

### 5.1 `@nestjs/cqrs` Overview

The [CQRS & Event Patterns](../patterns/cqrs-and-events.md) doc builds CQRS from scratch. `@nestjs/cqrs` provides the same primitives as first-class NestJS citizens.

| Concept | Manual (patterns doc) | `@nestjs/cqrs` |
|---------|----------------------|-----------------|
| Command dispatch | Custom `CommandBus` class | `CommandBus` injectable |
| Query dispatch | Custom `QueryBus` class | `QueryBus` injectable |
| Event publishing | Custom `EventBus` | `EventBus` injectable |
| Sagas | Manual event listeners | `@Saga()` on observable streams |

### 5.2 Commands

```typescript
export class CreateOrderCommand {
  constructor(
    public readonly userId: string,
    public readonly items: Array<{ productId: string; qty: number }>,
  ) {}
}

@CommandHandler(CreateOrderCommand)
export class CreateOrderHandler implements ICommandHandler<CreateOrderCommand> {
  constructor(private readonly eventBus: EventBus) {}

  async execute(command: CreateOrderCommand) {
    const order = { id: crypto.randomUUID(), ...command, status: 'pending' };
    // persist order...
    this.eventBus.publish(new OrderCreatedEvent(order.id, command.userId));
    return order;
  }
}
```

### 5.3 Queries

```typescript
export class GetOrderQuery {
  constructor(public readonly orderId: string) {}
}

@QueryHandler(GetOrderQuery)
export class GetOrderHandler implements IQueryHandler<GetOrderQuery> {
  async execute(query: GetOrderQuery) {
    return { id: query.orderId, status: 'shipped' }; // read from optimized read model
  }
}
```

### 5.4 Events and Sagas

```typescript
export class OrderCreatedEvent {
  constructor(public readonly orderId: string, public readonly userId: string) {}
}

@EventsHandler(OrderCreatedEvent)
export class OrderCreatedHandler implements IEventHandler<OrderCreatedEvent> {
  handle(event: OrderCreatedEvent) {
    console.log(`Sending confirmation for order ${event.orderId}`);
  }
}
```

Sagas observe event streams and dispatch new commands:

```typescript
@Injectable()
export class OrderSaga {
  @Saga()
  orderCreated = (events$: Observable<any>): Observable<ICommand> =>
    events$.pipe(
      ofType(OrderCreatedEvent),
      map((event) => new SendConfirmationCommand(event.orderId, event.userId)),
    );
}
```

### 5.5 Wiring the Module

```typescript
@Module({
  imports: [CqrsModule],
  providers: [CreateOrderHandler, GetOrderHandler, OrderCreatedHandler, OrderSaga],
})
export class OrderModule {}
```

Spring has no built-in CQRS module. Teams typically use Axon Framework or build their own with `ApplicationEventPublisher`.

---

## 6. Lifecycle Hooks

### 6.1 Hook Comparison

| NestJS Hook | When | Spring Equivalent |
|-------------|------|-------------------|
| `OnModuleInit` | After providers instantiated | `@PostConstruct` |
| `OnApplicationBootstrap` | After all modules initialized | `ApplicationReadyEvent` |
| `OnModuleDestroy` | On `app.close()` | `@PreDestroy` |
| `BeforeApplicationShutdown` | Before connections close (receives signal) | `ContextClosedEvent` |
| `OnApplicationShutdown` | After all connections closed | `SmartLifecycle.stop()` |

### 6.2 Implementation

```typescript
@Injectable()
export class PrismaService
  extends PrismaClient
  implements OnModuleInit, OnApplicationShutdown
{
  async onModuleInit() { await this.$connect(); }

  async onApplicationShutdown(signal?: string) {
    console.log(`Shutting down (signal: ${signal})`);
    await this.$disconnect();
  }
}
```

### 6.3 Graceful Shutdown

NestJS does not listen for OS signals by default — you must opt in:

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.enableShutdownHooks(); // listens for SIGTERM, SIGINT
  await app.listen(3000);
}
```

Spring Boot enables graceful shutdown with `server.shutdown=graceful`. NestJS gives you the hooks but leaves timeout logic to your implementation.

**Note**: `enableShutdownHooks()` consumes memory for each listener. Always enable it on Kubernetes (where the orchestrator sends SIGTERM). Skip it on serverless.

---

## 7. Testing in NestJS

### 7.1 Unit Testing a Service

NestJS provides `Test.createTestingModule()` — a mini IoC container for tests. Spring's equivalent is `@SpringBootTest` with `@MockBean`.

```typescript
describe('OrderService', () => {
  let service: OrderService;

  const mockPrisma = {
    order: {
      findMany: jest.fn().mockResolvedValue([
        { id: '1', status: 'pending' },
        { id: '2', status: 'shipped' },
      ]),
      findUnique: jest.fn().mockResolvedValue({ id: '1', status: 'pending' }),
    },
  };

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        OrderService,
        { provide: TenantPrismaService, useValue: mockPrisma },
      ],
    }).compile();

    service = module.get(OrderService);
  });

  it('should return all orders', async () => {
    const orders = await service.findAll();
    expect(orders).toHaveLength(2);
    expect(mockPrisma.order.findMany).toHaveBeenCalled();
  });
});
```

### 7.2 Overriding Providers

For full module graphs, `.overrideProvider()` replaces a provider after module setup (closer to Spring's `@MockBean`):

```typescript
const module = await Test.createTestingModule({ imports: [OrderModule] })
  .overrideProvider(TenantPrismaService)
  .useValue(mockPrisma)
  .compile();
```

### 7.3 E2E Testing with Supertest

```typescript
describe('Orders (e2e)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleFixture = await Test.createTestingModule({ imports: [AppModule] })
      .overrideProvider(TenantPrismaService)
      .useValue(mockPrisma)
      .compile();

    app = moduleFixture.createNestApplication();
    app.useGlobalPipes(new ValidationPipe({ whitelist: true }));
    await app.init();
  });

  afterAll(() => app.close());

  it('GET /orders returns 200 with tenant header', () =>
    request(app.getHttpServer())
      .get('/orders')
      .set('X-Tenant-Id', 'acme')
      .expect(200)
      .expect((res) => expect(res.body).toBeInstanceOf(Array)));

  it('GET /orders without tenant header returns 400', () =>
    request(app.getHttpServer()).get('/orders').expect(400));
});
```

### 7.4 Spring Testing Comparison

| Spring | NestJS |
|--------|--------|
| `@SpringBootTest` | `Test.createTestingModule({ imports: [AppModule] })` |
| `@MockBean` | `.overrideProvider(X).useValue(mock)` |
| `MockMvc` | `supertest` with `app.getHttpServer()` |
| `@WebMvcTest(Controller.class)` | `Test.createTestingModule({ controllers: [X] })` |

---

## 8. Performance Tips

### 8.1 Fastify Adapter

Express is the default. Fastify handles ~2-3x more requests per second due to schema-based serialization and radix-tree routing.

```typescript
import { FastifyAdapter, NestFastifyApplication } from '@nestjs/platform-fastify';

const app = await NestFactory.create<NestFastifyApplication>(
  AppModule,
  new FastifyAdapter(),
);
await app.listen(3000, '0.0.0.0'); // Fastify requires explicit host
```

**Caveat**: Some Express-specific middleware (like `express-session`) won't work. Check compatibility first.

### 8.2 Lazy-Loaded Modules

Load feature modules on demand instead of at startup:

```typescript
@Controller()
export class AppController {
  constructor(private readonly lazyLoader: LazyModuleLoader) {}

  @Get('reports')
  async getReports() {
    const { ReportsModule } = await import('./reports/reports.module');
    const moduleRef = await this.lazyLoader.load(() => ReportsModule);
    return moduleRef.get(ReportsService).generate();
  }
}
```

Spring has no direct equivalent — all beans load at startup unless individually `@Lazy`.

### 8.3 Caching with `@nestjs/cache-manager`

```typescript
@Module({
  imports: [CacheModule.register({ ttl: 30_000, max: 100 })],
})
export class AppModule {}
```

```typescript
@Controller('products')
@UseInterceptors(CacheInterceptor)
export class ProductsController {
  @Get()
  @CacheTTL(60_000)
  findAll() { return this.productsService.findAll(); }
}
```

For production, swap the in-memory store for Redis via `cache-manager-redis-yet`.

### 8.4 Compression and Security Headers

```typescript
import helmet from '@fastify/helmet';
import compress from '@fastify/compress';

await app.register(helmet);
await app.register(compress, { encodings: ['gzip', 'deflate'] });
```

### 8.5 Performance Checklist

- [ ] Use Fastify adapter if not dependent on Express middleware
- [ ] Lazy-load heavy feature modules not needed on every request
- [ ] Cache expensive queries with Redis-backed `@nestjs/cache-manager`
- [ ] Enable compression (prefer reverse proxy level in production)
- [ ] Set Helmet headers (zero-cost security best practice)
- [ ] Avoid request-scoped providers unless truly necessary
- [ ] Profile with `clinic.js` or `0x` before optimizing

---

## References

- [NestJS Custom Decorators](https://docs.nestjs.com/custom-decorators)
- [NestJS Dynamic Modules](https://docs.nestjs.com/fundamentals/dynamic-modules)
- [NestJS Injection Scopes](https://docs.nestjs.com/fundamentals/injection-scopes)
- [NestJS Microservices](https://docs.nestjs.com/microservices/basics)
- [NestJS CQRS](https://docs.nestjs.com/recipes/cqrs)
- [NestJS Lifecycle Events](https://docs.nestjs.com/fundamentals/lifecycle-events)
- [NestJS Testing](https://docs.nestjs.com/fundamentals/testing)
- [NestJS Performance (Fastify)](https://docs.nestjs.com/techniques/performance)

---

**See also**: [NestJS for Spring Boot Developers](nestjs-for-spring-devs.md) | [CQRS & Event Patterns](../patterns/cqrs-and-events.md)
