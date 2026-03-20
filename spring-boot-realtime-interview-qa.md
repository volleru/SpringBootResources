# Spring Boot — Real-Time Interview Questions & Answers

> Deep, scenario-based questions asked at companies like Amazon, Google, Uber, PayPal, etc.

---

## SECTION 1 — Spring Boot Internals & Startup

---

### Q1. Walk me through exactly what happens when you call `SpringApplication.run()`?

**Answer:**
1. Creates a `SpringApplication` instance — determines application type (SERVLET, REACTIVE, NONE) by checking classpath.
2. Loads `SpringApplicationRunListeners` from `spring.factories` and fires `starting()` event.
3. Prepares `Environment` — loads `application.properties`, environment variables, command-line args in priority order.
4. Prints the banner.
5. Creates the appropriate `ApplicationContext` — `AnnotationConfigServletWebServerApplicationContext` for web apps.
6. Prepares the context — sets environment, applies `ApplicationContextInitializers`.
7. **Refreshes** the context — this is where all the heavy lifting happens:
   - Scans for `@Component`, `@Configuration` classes.
   - Processes `@Bean` definitions.
   - Runs auto-configuration by loading classes from `AutoConfiguration.imports`.
   - Instantiates all singleton beans eagerly.
   - Starts the embedded web server.
8. Calls `ApplicationRunner` and `CommandLineRunner` beans.
9. Fires `ApplicationReadyEvent`.

**Trap:** Many candidates skip step 7 and don't know the embedded server starts during `refresh()`, not after.

---

### Q2. You have two auto-configuration classes. How do you control which one loads first?

**Answer:**
Use `@AutoConfigureBefore` and `@AutoConfigureAfter` on your auto-configuration class:

```java
@AutoConfigureAfter(DataSourceAutoConfiguration.class)
@AutoConfigureBefore(HibernateJpaAutoConfiguration.class)
public class MyCustomAutoConfiguration { ... }
```

Also use `@AutoConfigureOrder` for fine-grained ordering:
```java
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
public class MyAutoConfiguration { ... }
```

**Follow-up:** What if you need a bean from another auto-configuration but it might not exist?
Use `@ConditionalOnBean(DataSource.class)` — your config only loads if that bean is already present.

---

### Q3. I added a `DataSource` bean in my config but Spring Boot's auto-configuration also tries to create one. What happens and why?

**Answer:**
Spring Boot's `DataSourceAutoConfiguration` is annotated with `@ConditionalOnMissingBean(DataSource.class)`. Since you defined your own `DataSource` bean first, the condition evaluates to `false` and auto-configuration backs off — your bean wins.

This is the **"user-defined beans take priority"** principle in Spring Boot.

**Trap question:** What if your `DataSource` bean is in a `@Configuration` class that is processed AFTER auto-configuration? The condition evaluation is done at refresh time, and Spring processes user configuration before auto-configuration, so your bean always wins regardless of ordering in most cases.

---

### Q4. What is the difference between `@Conditional`, `@ConditionalOnClass`, `@ConditionalOnBean`, and `@ConditionalOnProperty`?

**Answer:**
All are used to conditionally register beans, but differ in what they check:

| Annotation | Condition |
|---|---|
| `@Conditional(MyCondition.class)` | Custom `Condition` implementation — most flexible |
| `@ConditionalOnClass(Foo.class)` | Foo.class is on the classpath |
| `@ConditionalOnMissingClass` | Class is NOT on the classpath |
| `@ConditionalOnBean(Foo.class)` | A bean of type Foo exists in context |
| `@ConditionalOnMissingBean` | No bean of that type exists |
| `@ConditionalOnProperty("feature.x.enabled")` | Property is set (optionally to a specific value) |
| `@ConditionalOnWebApplication` | Application is a web application |
| `@ConditionalOnExpression("${x} && ${y}")` | SpEL expression evaluates to true |

**Real interview trap:** `@ConditionalOnBean` checks beans registered so far during context refresh. If the bean you're checking for is defined in a later configuration class, the condition may fail. Always use `@AutoConfigureAfter` to ensure ordering when using `@ConditionalOnBean` in auto-configuration.

---

### Q5. Your Spring Boot app takes 45 seconds to start in production. How do you diagnose and fix it?

**Answer:**
**Diagnosis:**
1. Enable startup logging:
   ```properties
   spring.jmx.enabled=false
   logging.level.org.springframework=DEBUG
   ```
2. Use Spring Boot Actuator's `/actuator/startup` endpoint (Spring Boot 2.4+).
3. Add `-Dspring.output.ansi.enabled=always` and look for slow bean initialization.
4. Use Java Flight Recorder or async-profiler.

**Common causes and fixes:**
- **Component scan too broad** — `@SpringBootApplication` on root package scans everything. Narrow it: `@SpringBootApplication(scanBasePackages = "com.myapp.core")`.
- **Eager initialization of heavy beans** — use `@Lazy` or `spring.main.lazy-initialization=true`.
- **Too many Flyway/Liquibase migrations** — consider baseline migrations.
- **Slow database connection at startup** — configure HikariCP properly, check DNS resolution.
- **Auto-configuration loading unused configs** — use `spring.autoconfigure.exclude` to exclude what you don't need.
- **`@PostConstruct` doing heavy work** — move to async init or `ApplicationRunner`.

---

## SECTION 2 — Bean Lifecycle & Scopes

---

### Q6. Explain the complete lifecycle of a Spring bean.

**Answer:**
1. **Instantiation** — Spring calls the constructor.
2. **Dependency Injection** — Spring injects dependencies (setter/field injection).
3. **BeanNameAware / BeanFactoryAware** — Spring calls aware interfaces if implemented.
4. **BeanPostProcessor.postProcessBeforeInitialization()** — all registered `BeanPostProcessor`s run before init.
5. **`@PostConstruct`** — custom initialization method runs.
6. **`InitializingBean.afterPropertiesSet()`** — if implemented.
7. **Custom `init-method`** — if specified in `@Bean(initMethod = "init")`.
8. **BeanPostProcessor.postProcessAfterInitialization()** — post-processors run after init. **This is where AOP proxies are created.**
9. Bean is ready to use.
10. **`@PreDestroy`** — called on context close.
11. **`DisposableBean.destroy()`** — if implemented.
12. **Custom `destroy-method`** — if specified.

**Real trap:** `@Transactional` works because of step 8 — the original bean is replaced by a proxy. If you inject `this` reference inside the bean and call a method on it, you bypass the proxy and `@Transactional` won't work.

---

### Q7. What is the problem with injecting a prototype-scoped bean into a singleton bean?

**Answer:**
This is a classic Spring trap. When a singleton bean has a prototype dependency injected via constructor or `@Autowired`, Spring injects **one instance of the prototype at creation time**. The prototype bean is never refreshed — effectively behaving like a singleton.

**Fixes:**
1. **`ApplicationContext.getBean()`** — inject `ApplicationContext` and call `getBean()` each time (poor design).
2. **Method injection with `@Lookup`:**
```java
@Component
public class SingletonBean {
    @Lookup
    public PrototypeBean getPrototypeBean() {
        return null; // Spring overrides this at runtime
    }

    public void doWork() {
        PrototypeBean p = getPrototypeBean(); // fresh instance each time
    }
}
```
3. **`ObjectFactory<PrototypeBean>`** or **`Provider<PrototypeBean>`** — inject a factory instead of the bean.
4. **`@Scope(proxyMode = ScopedProxyMode.TARGET_CLASS)`** — makes the prototype bean a scoped proxy.

---

### Q8. What is a `BeanPostProcessor` and when would you use one in production?

**Answer:**
`BeanPostProcessor` is an interface that intercepts bean initialization — you can modify or wrap beans after creation but before they're used.

```java
@Component
public class AuditBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if (bean.getClass().isAnnotationPresent(Auditable.class)) {
            return Proxy.newProxyInstance(...); // wrap with audit proxy
        }
        return bean;
    }
}
```

**Real-world uses:**
- Spring AOP uses `AbstractAutoProxyCreator` (a BPP) to create `@Transactional` and `@Async` proxies.
- Validators that check beans for required configuration after initialization.
- Custom annotation processing (e.g., auto-register beans with an external registry).
- Metrics instrumentation — wrapping service beans with timing proxies.

**Important:** `BeanPostProcessor` beans and their dependencies are initialized very early — they cannot use `@Autowired` for beans that themselves need post-processing. Spring will warn you with "Bean X is not eligible for getting processed by all BeanPostProcessors."

---

## SECTION 3 — Transactions & Database

---

### Q9. You call a `@Transactional` method from within the same class and the transaction doesn't work. Why?

**Answer:**
This is the **self-invocation problem**. Spring's `@Transactional` works via AOP proxies. When you call `this.method()` inside the same class, you're calling the real object, not the proxy — so Spring's transaction interceptor never runs.

```java
@Service
public class OrderService {
    public void placeOrder() {
        this.processPayment(); // PROBLEM: bypasses proxy, no transaction
    }

    @Transactional
    public void processPayment() { ... }
}
```

**Fixes:**
1. **Move `processPayment()` to a separate service** (best approach — follows SRP).
2. **Self-inject the proxy:**
```java
@Service
public class OrderService {
    @Autowired
    private OrderService self; // Spring injects the proxy

    public void placeOrder() {
        self.processPayment(); // goes through proxy
    }
}
```
3. **AopContext.currentProxy():**
```java
((OrderService) AopContext.currentProxy()).processPayment();
// requires @EnableAspectJAutoProxy(exposeProxy = true)
```

---

### Q10. Explain `@Transactional` propagation levels with a real scenario — when would `REQUIRES_NEW` cause a bug?

**Answer:**
Propagation levels define how transactions relate when methods call each other:

| Propagation | Behavior |
|---|---|
| `REQUIRED` (default) | Join existing tx, or create new if none |
| `REQUIRES_NEW` | Always suspend existing tx, create new one |
| `SUPPORTS` | Join if exists, run non-transactionally if not |
| `NOT_SUPPORTED` | Always suspend existing tx, run non-transactionally |
| `MANDATORY` | Must have existing tx, throw if none |
| `NEVER` | Must NOT have existing tx, throw if one exists |
| `NESTED` | Nested tx (savepoint) within existing tx |

**Real bug with `REQUIRES_NEW`:**
```java
@Transactional
public void processOrder(Order order) {
    orderRepo.save(order);
    auditService.logAudit(order); // REQUIRES_NEW — separate tx
    // if this line throws, order is rolled back
    // but audit is already COMMITTED in its own tx
    throw new RuntimeException("something failed");
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void logAudit(Order order) {
    auditRepo.save(new AuditLog(order));
}
```
Result: The order is rolled back, but the audit log is committed — **data inconsistency**. Use `REQUIRES_NEW` only when you explicitly want an independent transaction (e.g., logging that should persist even on failure).

---

### Q11. What is the N+1 problem in JPA and how do you fix it in Spring Boot?

**Answer:**
N+1 occurs when fetching N parent entities results in N additional queries to fetch their children.

```java
// FetchType.LAZY (default for @OneToMany)
List<Department> departments = departmentRepo.findAll(); // 1 query
for (Department d : departments) {
    d.getEmployees().size(); // N queries — one per department!
}
```

**Fixes:**

1. **JPQL JOIN FETCH:**
```java
@Query("SELECT d FROM Department d JOIN FETCH d.employees")
List<Department> findAllWithEmployees();
```

2. **Entity Graph:**
```java
@EntityGraph(attributePaths = {"employees"})
List<Department> findAll();
```

3. **Batch fetching** (Hibernate-specific):
```java
@OneToMany
@BatchSize(size = 20) // fetches 20 collections in one IN query
private List<Employee> employees;
```

4. **DTO projection with JPQL** (most performant — avoids loading entities):
```java
@Query("SELECT new com.app.dto.DeptDTO(d.id, d.name, e.name) FROM Department d JOIN d.employees e")
List<DeptDTO> findDeptSummary();
```

**Detect N+1 in production:**
```properties
spring.jpa.properties.hibernate.generate_statistics=true
logging.level.org.hibernate.stat=DEBUG
```
Or use `p6spy` / Datasource proxy to log all SQL.

---

### Q12. Your application runs fine in dev but gets `HikariPool-1 - Connection is not available, request timed out` in prod. What do you investigate?

**Answer:**
This means HikariCP's connection pool is exhausted. Connections are not being returned.

**Investigation steps:**

1. **Enable HikariCP metrics:**
```properties
management.metrics.export.prometheus.enabled=true
# or
logging.level.com.zaxxer.hikari=DEBUG
```

2. **Common root causes:**
   - **Connection leak** — transaction opened but never closed (missing `@Transactional`, exception before close).
   - **Long-running transactions** — holding a connection while doing non-DB work (HTTP calls, file I/O inside a transaction).
   - **Pool too small for load** — default is 10 connections.
   - **Database is slow** — connections pile up waiting for slow queries.
   - **Thread pool misconfiguration** — more request threads than DB connections.

3. **HikariCP leak detection:**
```properties
spring.datasource.hikari.leak-detection-threshold=30000  # 30 seconds
```
This will log a warning with the stack trace of where the connection was obtained.

4. **Tuning:**
```properties
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000
```

**Formula for pool size:** `pool_size = (core_count * 2) + effective_spindle_count` (from HikariCP docs).

---

## SECTION 4 — Spring Security

---

### Q13. How does Spring Security's filter chain work? Where does JWT validation happen?

**Answer:**
Spring Security is implemented as a chain of `javax.servlet.Filter`s, registered as a single `DelegatingFilterProxy` → `FilterChainProxy`. Each request passes through ~15 filters in order.

Key filters in order:
1. `SecurityContextPersistenceFilter` — loads `SecurityContext` from session.
2. `UsernamePasswordAuthenticationFilter` — processes form login.
3. `BasicAuthenticationFilter` — processes HTTP Basic auth.
4. `BearerTokenAuthenticationFilter` — processes OAuth2 Bearer tokens.
5. `ExceptionTranslationFilter` — catches `AuthenticationException` and `AccessDeniedException`.
6. `FilterSecurityInterceptor` — makes the final authorization decision.

**JWT validation** — you add a custom filter before `UsernamePasswordAuthenticationFilter`:
```java
public class JwtAuthFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain) {
        String token = extractToken(request);
        if (token != null && jwtService.isValid(token)) {
            UsernamePasswordAuthenticationToken auth =
                new UsernamePasswordAuthenticationToken(username, null, authorities);
            SecurityContextHolder.getContext().setAuthentication(auth);
        }
        chain.doFilter(request, response);
    }
}

// Register it
http.addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
```

**Trap:** `SecurityContextHolder` uses `ThreadLocal` by default. In reactive/async code, you must use `ReactiveSecurityContextHolder` or configure `SecurityContextHolder.MODE_INHERITABLETHREADLOCAL`.

---

### Q14. What is the difference between `@PreAuthorize`, `@Secured`, and `@RolesAllowed`? Which one do you prefer and why?

**Answer:**
All three provide method-level security:

| Annotation | Source | SpEL Support | Flexibility |
|---|---|---|---|
| `@Secured("ROLE_ADMIN")` | Spring Security | No | Low |
| `@RolesAllowed("ADMIN")` | JSR-250 | No | Low |
| `@PreAuthorize("hasRole('ADMIN')")` | Spring Security | Yes | High |
| `@PostAuthorize("returnObject.owner == authentication.name")` | Spring Security | Yes | High |

**Prefer `@PreAuthorize`** because:
- Full SpEL support — can check method parameters, return values, custom beans.
- Can express complex rules: `@PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")`
- `@PostAuthorize` can filter return value based on the result.
- `@PreFilter` / `@PostFilter` can filter collections.

Enable with:
```java
@EnableMethodSecurity // Spring Boot 3.x (replaces @EnableGlobalMethodSecurity)
```

**Real trap:** `@PreAuthorize` only works on Spring-managed beans called through the proxy. It won't work on `private` methods or self-invocations.

---

## SECTION 5 — REST API & Performance

---

### Q15. Your REST endpoint returns paginated results but clients complain it's slow on large datasets. What changes do you make?

**Answer:**
**Symptom:** `SELECT * FROM orders` fetches millions of rows, then paginates in memory.

**Fix 1 — Database-level pagination:**
```java
Page<Order> findByUserId(Long userId, Pageable pageable);

// Call with:
orderRepo.findByUserId(userId, PageRequest.of(0, 20, Sort.by("createdAt").descending()));
```
Generates: `SELECT ... LIMIT 20 OFFSET 0`

**Fix 2 — Count query optimization** (Spring Data JPA fires a count query for `Page<T>`):
```java
@Query(value = "SELECT o FROM Order o WHERE o.userId = :id",
       countQuery = "SELECT COUNT(o.id) FROM Order o WHERE o.userId = :id")
Page<Order> findByUserId(@Param("id") Long id, Pageable pageable);
```

**Fix 3 — Use `Slice<T>` instead of `Page<T>`** if you don't need total count (no count query):
```java
Slice<Order> findByUserId(Long userId, Pageable pageable);
```

**Fix 4 — Keyset/cursor pagination** for very large datasets (avoids OFFSET performance degradation):
```java
// Instead of OFFSET, use WHERE id > lastSeenId LIMIT 20
List<Order> findByUserIdAndIdGreaterThanOrderById(Long userId, Long lastId, Pageable pageable);
```

**Fix 5 — Projections** to avoid selecting unused columns:
```java
public interface OrderSummary {
    Long getId();
    String getStatus();
    BigDecimal getTotal();
}
Page<OrderSummary> findByUserId(Long userId, Pageable pageable);
```

---

### Q16. How do you handle concurrent requests that update the same record? What strategies does Spring/JPA offer?

**Answer:**
This is a **lost update** problem. Two users read the same record, modify it, and save — the first save is overwritten.

**Strategy 1 — Optimistic Locking** (preferred for low-contention):
```java
@Entity
public class Account {
    @Version
    private Long version;  // JPA automatically increments this
    private BigDecimal balance;
}
```
When two threads update concurrently, one will get `OptimisticLockException` — you retry the operation.

**Strategy 2 — Pessimistic Locking** (for high-contention):
```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT a FROM Account a WHERE a.id = :id")
Optional<Account> findByIdForUpdate(@Param("id") Long id);
```
Issues `SELECT ... FOR UPDATE` — database-level row lock.

**Strategy 3 — Database-level atomic updates** (best for simple increments):
```java
@Modifying
@Query("UPDATE Account a SET a.balance = a.balance - :amount WHERE a.id = :id AND a.balance >= :amount")
int deductBalance(@Param("id") Long id, @Param("amount") BigDecimal amount);
// Returns 0 if balance insufficient — check the return value
```

**Real scenario question:** Which would you use for a bank transfer?
- Pessimistic locking for transfers (you can't afford retries with money movement).
- Optimistic locking for profile updates (cheap to retry, low contention).

---

### Q17. Your Spring Boot REST API is getting 50K requests/second. What would you do to scale it?

**Answer:**
This is a system design + Spring Boot question:

**Application level:**
1. **Enable async processing** — `@EnableAsync` + `@Async` for non-blocking operations.
2. **Switch to reactive** — `spring-boot-starter-webflux` + `WebClient` replaces blocking Tomcat with Netty (handles 50K on far fewer threads).
3. **Connection pooling** — tune HikariCP, use `r2dbc` for reactive DB.
4. **Caching** — Redis cache with `@Cacheable` for read-heavy endpoints.
5. **Compression** — `server.compression.enabled=true` reduces response size.

**Infrastructure level:**
1. **Horizontal scaling** — multiple instances behind a load balancer.
2. **Session externalization** — `spring-session-data-redis` so any instance handles any request.
3. **Rate limiting** — `spring-cloud-gateway` with `RequestRateLimiter` filter.
4. **Circuit breaker** — Resilience4j to stop cascading failures.

**Database level:**
1. **Read replicas** — route read queries to replicas using `@Transactional(readOnly = true)`.
2. **Database connection pool sizing** — don't over-provision connections.
3. **Query optimization** — indexes, query plans.

---

## SECTION 6 — Microservices & Spring Cloud

---

### Q18. How do you handle distributed transactions across microservices in Spring Boot?

**Answer:**
You **cannot** use `@Transactional` across microservices — there's no distributed transaction manager in practice (2PC is too slow and fragile).

**Patterns used in production:**

**1. SAGA Pattern:**
- **Choreography** — each service publishes events, next service listens and acts. If failure, publish a compensating event.
- **Orchestration** — a central orchestrator (saga manager) calls each service and coordinates compensation.

```
OrderService → publishes OrderCreated
PaymentService → listens, charges card, publishes PaymentProcessed
InventoryService → listens, reserves stock, publishes StockReserved
If InventoryService fails → publishes StockFailed
PaymentService → listens, refunds card (compensating transaction)
```

**2. Outbox Pattern** — avoids dual write (save to DB AND publish to Kafka):
```java
@Transactional
public void placeOrder(Order order) {
    orderRepo.save(order);
    outboxRepo.save(new OutboxEvent("OrderCreated", order)); // same TX
}
// Separate process polls outbox and publishes to Kafka
```

**3. Eventual Consistency** — accept that data will be consistent eventually, not immediately.

**Follow-up:** How do you handle idempotency in Saga?
Each step checks if it was already executed using an idempotency key before processing.

---

### Q19. What happens when a microservice your app calls is down? How do you handle it?

**Answer:**
Without protection, threads pile up waiting on the failed service → your service runs out of threads → your service goes down too (**cascading failure**).

**Fix: Circuit Breaker pattern with Resilience4j:**

```java
@CircuitBreaker(name = "inventoryService", fallbackMethod = "fallbackInventory")
@TimeLimiter(name = "inventoryService")
public CompletableFuture<InventoryResponse> checkInventory(Long productId) {
    return CompletableFuture.supplyAsync(() -> inventoryClient.check(productId));
}

public CompletableFuture<InventoryResponse> fallbackInventory(Long productId, Exception e) {
    return CompletableFuture.completedFuture(new InventoryResponse(false, "Service unavailable"));
}
```

Configuration:
```yaml
resilience4j:
  circuitbreaker:
    instances:
      inventoryService:
        sliding-window-size: 10
        failure-rate-threshold: 50      # open circuit after 50% failures
        wait-duration-in-open-state: 10s
        permitted-calls-in-half-open-state: 3
  timelimiter:
    instances:
      inventoryService:
        timeout-duration: 3s
```

**States:** CLOSED (normal) → OPEN (failing, reject all calls) → HALF_OPEN (probe) → CLOSED/OPEN.

**Additional patterns:** Retry with exponential backoff, Bulkhead (limit concurrent calls), Timeout.

---

## SECTION 7 — Debugging & Production Issues

---

### Q20. You deployed a new version of your Spring Boot app and memory usage keeps growing until OOM. How do you diagnose?

**Answer:**
**Step 1 — Identify what's leaking:**
```bash
# Trigger heap dump before OOM
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heapdump.hprof
# Or via Actuator
curl -X POST http://app:8080/actuator/heapdump -o heapdump.hprof
```
Analyze with Eclipse MAT or VisualVM — look for objects with unexpectedly high retention.

**Step 2 — Common Spring Boot memory leaks:**

1. **Static `Map`/`List` caches growing unboundedly** — use Caffeine/Guava cache with eviction.
2. **ThreadLocal not removed** — `@Async` threads reused in pool, `ThreadLocal` accumulates data.
3. **Event listeners not unregistered** — beans registering for application events but never deregistering.
4. **Hibernate first-level cache in long transactions** — processing 1M records in one transaction keeps all in `EntityManager` cache. Fix: call `entityManager.clear()` periodically or use `StatelessSession`.
5. **Metaspace leak** — classloaders not GC'd (common with CGLIB proxies in hot reload). Fix: increase metaspace or fix classloader references.
6. **Actuator + Micrometer metrics with unbounded tag cardinality** — using user IDs as metric tags creates millions of time series.

---

### Q21. `@Async` annotated method is not running asynchronously — it runs on the main thread. Why?

**Answer:**
Three common reasons:

**Reason 1 — Self-invocation (same as `@Transactional`):**
```java
// Wrong
@Service
public class MyService {
    public void doWork() {
        this.asyncMethod(); // bypasses proxy
    }

    @Async
    public void asyncMethod() { ... }
}
```

**Reason 2 — `@EnableAsync` is missing or on wrong class:**
```java
@SpringBootApplication
@EnableAsync  // must be present
public class Application { ... }
```

**Reason 3 — Method is not public:**
`@Async` (AOP-based) only works on `public` methods.

**Reason 4 — No thread pool configured — using default `SimpleAsyncTaskExecutor`:**
This creates a new thread per call — no pooling. In production, always define a custom executor:
```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {
    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}
```

**Follow-up:** How do you propagate `SecurityContext` to async threads?
```java
SecurityContextHolder.setStrategyName(SecurityContextHolder.MODE_INHERITABLETHREADLOCAL);
// or use DelegatingSecurityContextExecutor
```

---

### Q22. How do you ensure your Spring Boot application is production-ready? What do you configure before going live?

**Answer:**

**1. Health & Observability:**
```properties
management.endpoints.web.exposure.include=health,metrics,info,prometheus
management.endpoint.health.show-details=when_authorized
management.health.db.enabled=true
management.health.diskspace.enabled=true
```

**2. Logging — structured JSON for log aggregation:**
```xml
<!-- logback-spring.xml -->
<appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
</appender>
```

**3. Connection pool sizing:**
```properties
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.leak-detection-threshold=60000
```

**4. Graceful shutdown:**
```properties
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=30s
```

**5. JVM tuning:**
```
-Xms512m -Xmx2g
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:+HeapDumpOnOutOfMemoryError
```

**6. Security hardening:**
```properties
server.error.include-stacktrace=never
server.error.include-message=never  # don't leak internals
```

**7. Retry/timeout on external calls:**
```java
WebClient.builder()
    .baseUrl(url)
    .clientConnector(new ReactorClientHttpConnector(
        HttpClient.create().responseTimeout(Duration.ofSeconds(5))
    )).build();
```

**8. Disable dev-only features in prod:**
```properties
spring.devtools.restart.enabled=false
spring.h2.console.enabled=false
```

---

### Q23. Explain how you would implement distributed tracing in a Spring Boot microservices system.

**Answer:**
Distributed tracing links requests across multiple services using a **trace ID** (one per request chain) and **span ID** (one per service hop).

**Spring Boot 3.x (Micrometer Tracing):**
```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-zipkin</artifactId>
</dependency>
```

```properties
management.tracing.sampling.probability=1.0  # 100% in dev, 10% in prod
management.zipkin.tracing.endpoint=http://zipkin:9411/api/v2/spans
```

Spring Boot automatically:
- Injects `traceId` and `spanId` into MDC for log correlation.
- Propagates trace headers (`traceparent`, `X-B3-TraceId`) via `RestTemplate`, `WebClient`, and Spring Cloud OpenFeign.
- Creates spans for HTTP requests, DB queries (with Datasource proxy), Kafka messages.

**Log correlation:**
```properties
logging.pattern.console=%d{HH:mm:ss} [%X{traceId}/%X{spanId}] %-5level %logger - %msg%n
```

Now you can search logs by `traceId` across all services in Kibana/Loki.

---

### Q24. You have a `@Scheduled` job that runs every minute. Sometimes it takes more than a minute to complete — two instances run simultaneously and corrupt data. How do you fix it?

**Answer:**
By default `@Scheduled` runs on a single-threaded executor, so within one instance this doesn't happen. But with **multiple application instances**, two instances both run the job.

**Fix 1 — ShedLock** (most common in production):
```java
@Scheduled(cron = "0 * * * * *")
@SchedulerLock(name = "myJob", lockAtMostFor = "PT55S", lockAtLeastFor = "PT30S")
public void runJob() {
    // only one instance runs at a time
}
```
```properties
# Uses the database as a lock store
spring.datasource.url=jdbc:postgresql://...
# ShedLock creates a "shedlock" table
```

**Fix 2 — Quartz Scheduler with JDBC store** — clustered scheduler that ensures only one node runs each job.

**Fix 3 — Distribute via Kafka/Redis** — only one instance is the "leader" and processes the event.

**lockAtMostFor** — max duration to hold the lock (guards against dead locks if instance crashes).
**lockAtLeastFor** — minimum duration to hold the lock (prevents re-run if job finishes very fast).

---

### Q25. A senior engineer asks: "Why not just annotate everything with `@Transactional`?" How do you respond?

**Answer:**
Annotating everything with `@Transactional` is harmful for several reasons:

1. **Performance overhead** — every `@Transactional` method acquires a database connection from the pool and holds it until the method returns. Even read-only operations that don't need a transaction hold a connection.

2. **Connection starvation** — if you put `@Transactional` on a method that makes HTTP calls, the DB connection is held while waiting for the HTTP response. Under load, pool exhaustion happens.

3. **Transaction scope too wide** — a large transaction holds locks longer, increasing contention and deadlock probability.

4. **Masking bugs** — code that works only because everything is in one big transaction may fail when refactored.

5. **`@Transactional` on `@Controller`** — transaction spans the full HTTP request including response serialization, holding DB connection unnecessarily long.

**Best practices:**
- Put `@Transactional` only at the **service layer**, not controller or repository (Spring Data repos handle their own transactions).
- Use `@Transactional(readOnly = true)` for reads — enables Hibernate optimizations (no dirty checking, can use read replica).
- Keep transactions **short** — no remote calls, no heavy computation inside a transaction.
- Use `REQUIRES_NEW` only when you explicitly need an independent transaction.

---
