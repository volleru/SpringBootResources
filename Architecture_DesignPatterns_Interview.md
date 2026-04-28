# Architecture & Design Patterns — 20 Easy Scenario Interview Questions & Answers

> Covers SOLID, DDD, Hexagonal Architecture, Scalability, Caching, Observability, Fault Tolerance, and Creational/Structural/Behavioural Design Patterns.

---

## Q1. Your codebase has a `UserService` that handles registration, email sending, password reset, and reporting. A teammate says it violates SOLID. Which principle and how do you fix it?

**Answer:**
It violates the **Single Responsibility Principle (SRP)** — a class should have only one reason to change. `UserService` changes when registration logic changes, when email templates change, when reporting requirements change, etc.

```
Before (BAD):
UserService
 ├── registerUser()
 ├── sendWelcomeEmail()
 ├── resetPassword()
 └── generateUserReport()

After (GOOD — each class has one responsibility):
UserRegistrationService  → registerUser()
EmailService             → sendWelcomeEmail()
PasswordResetService     → resetPassword()
UserReportService        → generateUserReport()
```

```java
// BAD — one class doing everything
@Service
public class UserService {
    public void registerUser(User u) { ... }
    public void sendWelcomeEmail(User u) { ... }
    public void resetPassword(String email) { ... }
    public byte[] generateReport() { ... }
}

// GOOD — each class owns one concern
@Service public class UserRegistrationService { public void register(User u) { ... } }
@Service public class EmailService            { public void sendWelcome(User u) { ... } }
@Service public class PasswordResetService    { public void reset(String email) { ... } }
@Service public class UserReportService       { public byte[] generate() { ... } }
```

---

## Q2. You need to add a new payment method (Crypto) to your app without changing existing PaymentService code. Which SOLID principle applies and how do you implement it?

**Answer:**
**Open/Closed Principle (OCP)** — classes should be open for extension, closed for modification. Define a `PaymentProcessor` interface and add new payment types without touching existing code.

```java
// Interface — the contract
public interface PaymentProcessor {
    void process(Payment payment);
}

// Existing implementations — untouched when adding new ones
@Service("creditCard")
public class CreditCardProcessor implements PaymentProcessor {
    public void process(Payment p) { /* credit card logic */ }
}

@Service("paypal")
public class PayPalProcessor implements PaymentProcessor {
    public void process(Payment p) { /* paypal logic */ }
}

// NEW — just add without changing anything above
@Service("crypto")
public class CryptoProcessor implements PaymentProcessor {
    public void process(Payment p) { /* crypto logic */ }
}

// PaymentService uses the interface — never changes
@Service
public class PaymentService {
    private final Map<String, PaymentProcessor> processors;

    public PaymentService(Map<String, PaymentProcessor> processors) {
        this.processors = processors;
    }

    public void pay(String method, Payment payment) {
        processors.get(method).process(payment); // just add new key to map
    }
}
```

---

## Q3. You use a `Bird` base class with a `fly()` method. Now `Penguin` extends `Bird` but throws `UnsupportedOperationException` on `fly()`. What principle is broken?

**Answer:**
**Liskov Substitution Principle (LSP)** — a subclass must be substitutable for its parent without breaking the program. Fix by restructuring the hierarchy so `fly()` is only on birds that can actually fly.

```java
// BAD — substituting Penguin for Bird breaks the program
public class Bird { public void fly() { ... } }
public class Penguin extends Bird {
    @Override
    public void fly() { throw new UnsupportedOperationException("Penguins can't fly!"); }
}

// GOOD — separate flyable behavior
public abstract class Bird { public abstract void eat(); }

public interface Flyable { void fly(); }

public class Eagle extends Bird implements Flyable {
    public void eat() { ... }
    public void fly() { /* soar */ }
}

public class Penguin extends Bird {
    public void eat() { ... }
    // no fly() — correct!
}
```

---

## Q4. A junior dev asks why your `OrderService` depends on `EmailService` directly. What principle is violated and how do you fix it?

**Answer:**
**Dependency Inversion Principle (DIP)** — high-level modules should not depend on low-level modules; both should depend on abstractions. Inject an interface, not the concrete class.

```java
// BAD — OrderService tightly coupled to EmailService
@Service
public class OrderService {
    private final EmailService emailService = new EmailService(); // hard dependency

    public void placeOrder(Order order) {
        orderRepo.save(order);
        emailService.sendConfirmation(order); // can't swap this out
    }
}

// GOOD — depend on abstraction
public interface NotificationService {
    void sendOrderConfirmation(Order order);
}

@Service
public class EmailNotificationService implements NotificationService {
    public void sendOrderConfirmation(Order order) { /* email */ }
}

@Service
public class OrderService {
    private final NotificationService notificationService; // abstraction

    public OrderService(NotificationService notificationService) {
        this.notificationService = notificationService;
    }

    public void placeOrder(Order order) {
        orderRepo.save(order);
        notificationService.sendOrderConfirmation(order); // swap SMS/Slack anytime
    }
}
```

---

## Q5. You're designing an e-commerce system. How would you apply Domain-Driven Design (DDD) to model the Order domain?

**Answer:**
DDD organizes code around the business domain using **Entities**, **Value Objects**, **Aggregates**, **Repositories**, and **Domain Services**.

```
Order Bounded Context:
├── Aggregate Root: Order
│   ├── Entity: OrderItem
│   └── Value Object: Money, Address
├── Repository: OrderRepository (interface in domain, impl in infra)
├── Domain Service: PricingService (logic that spans entities)
└── Domain Event: OrderPlaced, OrderShipped
```

```java
// Value Object — immutable, no identity
public record Money(BigDecimal amount, String currency) {
    public Money add(Money other) {
        return new Money(this.amount.add(other.amount), this.currency);
    }
}

// Entity — has identity
public class OrderItem {
    private Long id;
    private String productId;
    private int quantity;
    private Money unitPrice;
}

// Aggregate Root — controls consistency boundary
public class Order {
    private Long id;
    private List<OrderItem> items = new ArrayList<>();
    private OrderStatus status;

    public void addItem(OrderItem item) {
        if (status != OrderStatus.PENDING) throw new IllegalStateException("Order is closed");
        items.add(item);
    }

    public Money totalAmount() {
        return items.stream().map(i -> i.getUnitPrice()).reduce(Money::add).orElseThrow();
    }
}

// Repository interface lives in domain layer
public interface OrderRepository {
    Order findById(Long id);
    void save(Order order);
}
```

---

## Q6. What is Hexagonal Architecture and how does it differ from layered architecture?

**Answer:**
Hexagonal (Ports & Adapters) architecture puts the **domain at the center**, surrounded by **ports** (interfaces) and **adapters** (implementations). Nothing in the domain knows about HTTP, databases, or frameworks.

```
Traditional Layered:                  Hexagonal:
Controller                            [ REST Adapter ]   [ DB Adapter ]
    ↓                                        ↓                  ↓
Service                              [ Port (interface) ] [ Port (interface) ]
    ↓                                        ↓                  ↓
Repository                           [       Domain Core        ]
    ↓                                        ↑                  ↑
Database                             [ Port (interface) ] [ Port (interface) ]
                                             ↑                  ↑
                                      [ Kafka Adapter ]  [ CLI Adapter ]
```

```java
// Domain — pure Java, zero framework dependencies
public class OrderService {
    private final OrderRepository orderRepo;       // Port (interface)
    private final PaymentGateway paymentGateway;  // Port (interface)

    public void placeOrder(Order order) {
        paymentGateway.charge(order.getTotal());
        orderRepo.save(order);
    }
}

// Adapter — implements port, knows about JPA
@Repository
public class JpaOrderRepository implements OrderRepository {
    @Autowired JpaOrderJpaRepo jpaRepo;
    public void save(Order order) { jpaRepo.save(OrderEntity.from(order)); }
}

// Adapter — implements port, knows about Stripe
@Component
public class StripePaymentGateway implements PaymentGateway {
    public void charge(Money amount) { stripeClient.charge(amount.getAmount()); }
}
```

---

## Q7. Your app is slow under heavy read traffic. How would you use caching to fix it and what are the trade-offs?

**Answer:**
Add a **cache-aside** pattern with Redis. Read from cache first; on miss, load from DB and populate cache.

```java
// Spring Cache with Redis
@Service
public class ProductService {

    @Cacheable(value = "products", key = "#id")
    public Product getProduct(Long id) {
        return productRepo.findById(id).orElseThrow(); // DB hit only on cache miss
    }

    @CacheEvict(value = "products", key = "#product.id")
    public Product updateProduct(Product product) {
        return productRepo.save(product); // evict stale cache on write
    }
}
```

```yaml
# application.yml
spring:
  cache:
    type: redis
  redis:
    host: localhost
    port: 6379
```

**Trade-offs:**

| Pro | Con |
|---|---|
| Dramatically reduces DB load | Stale data if cache not invalidated |
| Faster response times | Cache invalidation is hard to get right |
| Scales reads horizontally | Extra infra (Redis) to operate |
| Reduces DB connection pressure | Cache stampede on cold start |

> **Cache stampede fix:** Use cache-lock or probabilistic early expiration to prevent all threads hitting DB at once on a cold cache miss.

---

## Q8. Your monolith database is becoming a bottleneck. How would you apply sharding to scale it?

**Answer:**
**Sharding** splits data across multiple DB instances (shards) based on a shard key. Each shard holds a subset of the data.

```
Without sharding:            With sharding (by user_id):
                             Shard 0: users 0–999,999
  All users → DB             Shard 1: users 1M–1.9M
                             Shard 2: users 2M–2.9M

Shard key formula:
shard_id = user_id % total_shards
```

```java
public class ShardRouter {

    private final List<DataSource> shards;

    public DataSource getShardFor(Long userId) {
        int shardIndex = (int) (userId % shards.size());
        return shards.get(shardIndex);
    }
}

// Consistent hashing — better for adding/removing shards without reshuffling all data
public class ConsistentHashRouter {
    private final TreeMap<Integer, DataSource> ring = new TreeMap<>();

    public DataSource getShardFor(String key) {
        int hash = key.hashCode();
        Map.Entry<Integer, DataSource> entry = ring.ceilingEntry(hash);
        return entry != null ? entry.getValue() : ring.firstEntry().getValue();
    }
}
```

**Trade-offs:**

| Strategy | Good for | Drawback |
|---|---|---|
| Range sharding | Range queries | Hot partitions possible |
| Hash sharding | Even distribution | Range queries are expensive |
| Directory sharding | Flexible routing | Lookup table is a bottleneck |

---

## Q9. How would you design an observability stack for a Spring Boot microservice in production?

**Answer:**
Observability = **Metrics + Logs + Traces** (the three pillars).

```
Spring Boot App
    ↓ metrics (Micrometer)    → Prometheus → Grafana dashboards
    ↓ logs (Logback + JSON)   → Elasticsearch → Kibana (ELK)
    ↓ traces (OpenTelemetry)  → Jaeger / Zipkin (distributed tracing)
    ↓ alerts                  → Alertmanager → PagerDuty / Slack
```

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
```

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  tracing:
    sampling:
      probability: 1.0   # 100% in dev, ~0.1 in prod
```

```java
// Custom business metric
@Service
public class OrderService {

    private final Counter orderCounter;

    public OrderService(MeterRegistry registry) {
        this.orderCounter = Counter.builder("orders.placed")
            .description("Total orders placed")
            .register(registry);
    }

    public void placeOrder(Order order) {
        orderRepo.save(order);
        orderCounter.increment(); // tracked in Prometheus
    }
}
```

---

## Q10. Design a load balancer strategy for a stateless REST API vs a stateful WebSocket service. What changes?

**Answer:**

| | Stateless REST | Stateful WebSocket |
|---|---|---|
| Strategy | **Round-robin** or **least connections** | **Sticky sessions** (IP hash or cookie) |
| Session state | None — any instance handles any request | Connection tied to one server |
| Scaling | Add instances freely | Must route same client to same instance |
| Failure | Any other instance takes over instantly | Session lost if instance dies |

```yaml
# NGINX — round-robin for REST (default)
upstream rest_api {
    server app1:8080;
    server app2:8080;
    server app3:8080;
}

# NGINX — sticky sessions for WebSocket
upstream ws_api {
    ip_hash;               # same IP always routes to same server
    server app1:8081;
    server app2:8081;
}
```

> For WebSocket at scale, externalize session state to Redis so any instance can resume a dropped connection.

---

## Q11. What is the Circuit Breaker pattern and when does it trip?

**Answer:**
Circuit Breaker prevents cascading failures by stopping calls to a failing service and allowing it time to recover. It has three states: **Closed** (normal), **Open** (blocking calls), **Half-Open** (testing recovery).

```
CLOSED → (failure threshold crossed) → OPEN → (timeout elapsed) → HALF-OPEN
HALF-OPEN → (success) → CLOSED
HALF-OPEN → (failure) → OPEN
```

```java
// Using Resilience4j
@Service
public class ProductService {

    @CircuitBreaker(name = "productService", fallbackMethod = "fallbackProduct")
    @Retry(name = "productService")
    public Product getProduct(Long id) {
        return externalProductApi.fetch(id); // might fail
    }

    public Product fallbackProduct(Long id, Exception ex) {
        // return cached/default product when circuit is open
        return new Product(id, "Default Product", BigDecimal.ZERO);
    }
}
```

```yaml
resilience4j:
  circuitbreaker:
    instances:
      productService:
        failure-rate-threshold: 50       # open after 50% failures
        wait-duration-in-open-state: 10s # wait before half-open
        sliding-window-size: 10          # evaluate last 10 calls
```

---

## Q12. Explain the Saga pattern for distributed transactions across microservices.

**Answer:**
In a microservices system, you can't use a single DB transaction across services. The **Saga pattern** breaks a distributed transaction into a sequence of local transactions, each publishing an event. On failure, compensating transactions undo the work.

```
Order Saga (Choreography style):

OrderService   → places order      → publishes OrderPlaced
PaymentService → charges payment   → publishes PaymentProcessed (or PaymentFailed)
InventoryService → reserves stock  → publishes StockReserved (or StockInsufficient)
ShippingService → schedules ship   → publishes ShipmentScheduled

On failure (PaymentFailed):
InventoryService ← compensate: release stock
OrderService     ← compensate: cancel order
```

```java
// Choreography via Spring Events / Kafka
@Service
public class PaymentService {

    @KafkaListener(topics = "order.placed")
    public void onOrderPlaced(OrderPlacedEvent event) {
        try {
            chargeCustomer(event.getCustomerId(), event.getAmount());
            kafkaTemplate.send("payment.processed", new PaymentProcessedEvent(event.getOrderId()));
        } catch (PaymentException e) {
            kafkaTemplate.send("payment.failed", new PaymentFailedEvent(event.getOrderId()));
        }
    }
}

// Compensating transaction listener
@Service
public class OrderService {

    @KafkaListener(topics = "payment.failed")
    public void onPaymentFailed(PaymentFailedEvent event) {
        orderRepo.updateStatus(event.getOrderId(), OrderStatus.CANCELLED); // compensate
    }
}
```

---

## Q13. What is the difference between the Factory Method and Abstract Factory patterns? Give a real scenario.

**Answer:**

| | Factory Method | Abstract Factory |
|---|---|---|
| Creates | One product | Families of related products |
| How | Subclass decides what to create | Factory object creates the family |
| Use case | Single object type varies | Multiple related objects must match |

```java
// Factory Method — subclass decides which Notification to create
public abstract class NotificationFactory {
    public abstract Notification createNotification(); // factory method
}

public class EmailNotificationFactory extends NotificationFactory {
    public Notification createNotification() { return new EmailNotification(); }
}

public class SmsNotificationFactory extends NotificationFactory {
    public Notification createNotification() { return new SmsNotification(); }
}

// Abstract Factory — creates a family of related UI components
public interface UIFactory {
    Button createButton();
    Checkbox createCheckbox();
    Dialog createDialog();
}

public class DarkThemeFactory implements UIFactory {
    public Button createButton()   { return new DarkButton(); }
    public Checkbox createCheckbox(){ return new DarkCheckbox(); }
    public Dialog createDialog()   { return new DarkDialog(); }
}

public class LightThemeFactory implements UIFactory {
    public Button createButton()   { return new LightButton(); }
    public Checkbox createCheckbox(){ return new LightCheckbox(); }
    public Dialog createDialog()   { return new LightDialog(); }
}
```

---

## Q14. When would you use a Builder pattern vs a Constructor? Demonstrate with a real example.

**Answer:**
Use **Builder** when an object has many optional fields, or when telescoping constructors become unreadable. Builder makes construction explicit and readable.

```java
// BAD — telescoping constructors, unreadable
public User(String name, String email, String phone, String address, boolean active) { ... }
new User("Alice", "alice@x.com", null, null, true); // what are those nulls?

// GOOD — Builder pattern
public class User {
    private final String name;
    private final String email;
    private final String phone;   // optional
    private final String address; // optional
    private final boolean active;

    private User(Builder b) {
        this.name = b.name;
        this.email = b.email;
        this.phone = b.phone;
        this.address = b.address;
        this.active = b.active;
    }

    public static class Builder {
        private final String name;   // required
        private final String email;  // required
        private String phone;
        private String address;
        private boolean active = true;

        public Builder(String name, String email) {
            this.name = name; this.email = email;
        }
        public Builder phone(String phone)     { this.phone = phone; return this; }
        public Builder address(String address) { this.address = address; return this; }
        public Builder active(boolean active)  { this.active = active; return this; }
        public User build() { return new User(this); }
    }
}

// Clear and readable
User user = new User.Builder("Alice", "alice@x.com")
    .phone("555-1234")
    .active(true)
    .build();

// Lombok shortcut
@Builder
public class User {
    private String name;
    private String email;
    private String phone;
}
```

---

## Q15. What is the Observer pattern and how does Spring use it internally?

**Answer:**
**Observer** (Publish-Subscribe) lets objects subscribe to events from a publisher. When the publisher fires an event, all subscribers are notified automatically.

```java
// Custom Spring Application Event
public class OrderPlacedEvent extends ApplicationEvent {
    private final Order order;
    public OrderPlacedEvent(Object source, Order order) {
        super(source);
        this.order = order;
    }
    public Order getOrder() { return order; }
}

// Publisher — fires the event
@Service
public class OrderService {

    @Autowired ApplicationEventPublisher publisher;

    @Transactional
    public void placeOrder(Order order) {
        orderRepo.save(order);
        publisher.publishEvent(new OrderPlacedEvent(this, order)); // notify all listeners
    }
}

// Subscribers — react independently, decoupled from OrderService
@Component
public class EmailListener {
    @EventListener
    public void onOrderPlaced(OrderPlacedEvent event) {
        emailService.sendConfirmation(event.getOrder());
    }
}

@Component
public class InventoryListener {
    @EventListener
    @Async // handle asynchronously
    public void onOrderPlaced(OrderPlacedEvent event) {
        inventoryService.deductStock(event.getOrder());
    }
}
```

---

## Q16. What is the Decorator pattern and when is it better than inheritance?

**Answer:**
**Decorator** wraps an object to add behavior at runtime without modifying the original class. It is more flexible than inheritance because you can combine decorators.

```java
// Component interface
public interface DataWriter {
    void write(String data);
}

// Concrete component
public class FileDataWriter implements DataWriter {
    public void write(String data) { Files.writeString(Path.of("file.txt"), data); }
}

// Decorator 1 — adds compression
public class CompressedDataWriter implements DataWriter {
    private final DataWriter writer;
    public CompressedDataWriter(DataWriter writer) { this.writer = writer; }

    public void write(String data) {
        String compressed = compress(data);
        writer.write(compressed); // delegate
    }
}

// Decorator 2 — adds encryption
public class EncryptedDataWriter implements DataWriter {
    private final DataWriter writer;
    public EncryptedDataWriter(DataWriter writer) { this.writer = writer; }

    public void write(String data) {
        String encrypted = encrypt(data);
        writer.write(encrypted);
    }
}

// Combine at runtime — compression + encryption + file write
DataWriter writer = new EncryptedDataWriter(
                        new CompressedDataWriter(
                            new FileDataWriter()));
writer.write("sensitive data");

// Java I/O uses this pattern extensively:
InputStream is = new BufferedInputStream(
                    new GZIPInputStream(
                        new FileInputStream("file.gz")));
```

---

## Q17. Explain the Strategy pattern with a sorting/discount scenario.

**Answer:**
**Strategy** defines a family of algorithms, encapsulates each one, and makes them interchangeable at runtime.

```java
// Strategy interface
public interface DiscountStrategy {
    BigDecimal apply(BigDecimal price);
}

// Concrete strategies
public class NoDiscount implements DiscountStrategy {
    public BigDecimal apply(BigDecimal price) { return price; }
}

public class PercentageDiscount implements DiscountStrategy {
    private final double percent;
    public PercentageDiscount(double percent) { this.percent = percent; }
    public BigDecimal apply(BigDecimal price) {
        return price.multiply(BigDecimal.valueOf(1 - percent / 100));
    }
}

public class FlatDiscount implements DiscountStrategy {
    private final BigDecimal flat;
    public FlatDiscount(BigDecimal flat) { this.flat = flat; }
    public BigDecimal apply(BigDecimal price) { return price.subtract(flat); }
}

// Context — uses whichever strategy is injected
public class PriceCalculator {
    private final DiscountStrategy strategy;

    public PriceCalculator(DiscountStrategy strategy) {
        this.strategy = strategy;
    }

    public BigDecimal calculate(BigDecimal basePrice) {
        return strategy.apply(basePrice);
    }
}

// At runtime — swap strategy based on customer type
DiscountStrategy strategy = switch (customer.getType()) {
    case VIP      -> new PercentageDiscount(20);
    case SEASONAL -> new FlatDiscount(new BigDecimal("10"));
    default       -> new NoDiscount();
};

BigDecimal finalPrice = new PriceCalculator(strategy).calculate(product.getPrice());
```

---

## Q18. What is the Proxy pattern and how does Spring AOP use it?

**Answer:**
**Proxy** wraps an object and intercepts calls to add cross-cutting behavior (logging, security, transactions) without modifying the original class. Spring AOP creates proxies automatically via `@Transactional`, `@Cacheable`, `@Async`.

```
Client → [Spring Proxy] → Real Service
               ↓
        begin transaction  (before)
        call real method
        commit/rollback    (after)
```

```java
// Manual proxy example
public interface OrderService {
    void placeOrder(Order order);
}

public class OrderServiceImpl implements OrderService {
    public void placeOrder(Order order) { orderRepo.save(order); }
}

public class LoggingOrderServiceProxy implements OrderService {
    private final OrderService delegate;

    public LoggingOrderServiceProxy(OrderService delegate) {
        this.delegate = delegate;
    }

    public void placeOrder(Order order) {
        log.info("Placing order: {}", order.getId());
        long start = System.currentTimeMillis();
        delegate.placeOrder(order); // delegate to real service
        log.info("Order placed in {}ms", System.currentTimeMillis() - start);
    }
}

// Spring AOP does this automatically with @Around
@Aspect @Component
public class LoggingAspect {

    @Around("execution(* com.example.service.*.*(..))")
    public Object log(ProceedingJoinPoint pjp) throws Throwable {
        log.info("Calling {}", pjp.getSignature());
        long start = System.currentTimeMillis();
        Object result = pjp.proceed(); // call real method
        log.info("Done in {}ms", System.currentTimeMillis() - start);
        return result;
    }
}
```

---

## Q19. Design a fault-tolerant order processing system. What patterns would you combine?

**Answer:**
A production-grade fault-tolerant system combines multiple patterns:

```
Client
  ↓
[API Gateway]  ← rate limiting, auth
  ↓
[Order Service]
  ├── Circuit Breaker   → stops calls to failing Payment/Inventory service
  ├── Retry + Backoff   → retries transient failures (network blip)
  ├── Bulkhead          → isolates thread pools so one failure doesn't block all
  ↓
[Message Queue — Kafka/SQS]   → decouples, buffers, enables async
  ↓
[Payment Service]  [Inventory Service]  [Notification Service]
  ↓
[Saga Coordinator]  → compensating transactions on failure
  ↓
[Dead Letter Queue] → captures failed messages for manual review
```

```java
@Service
public class OrderService {

    // Circuit Breaker — stops if payment service is down
    @CircuitBreaker(name = "payment", fallbackMethod = "queueForRetry")
    // Retry — retries 3x with exponential backoff before circuit opens
    @Retry(name = "payment")
    // Bulkhead — limits concurrent calls to 10 so other features aren't starved
    @Bulkhead(name = "payment")
    public void processPayment(Order order) {
        paymentServiceClient.charge(order);
    }

    public void queueForRetry(Order order, Exception ex) {
        // push to retry queue instead of failing immediately
        retryQueue.send(order);
        log.warn("Payment service unavailable, queued order {}", order.getId());
    }
}
```

---

## Q20. What are JEE/Jakarta EE Patterns and how do they map to modern Spring Boot?

**Answer:**
Java EE patterns solve common enterprise problems. Spring Boot implements most of them as annotations or built-in features.

| JEE Pattern | Problem Solved | Spring Boot Equivalent |
|---|---|---|
| **Service Locator** | Find services without hardcoding | `@Autowired` / `ApplicationContext` |
| **DAO (Data Access Object)** | Abstract DB access from business logic | `@Repository` + Spring Data JPA |
| **Transfer Object (DTO)** | Pass data between layers without exposing entities | Plain Java class / record |
| **Session Facade** | Single entry point for complex business operations | `@Service` |
| **Front Controller** | Central entry point for all HTTP requests | `DispatcherServlet` (built-in) |
| **Intercepting Filter** | Pre/post-process HTTP requests | `@Filter` / `HandlerInterceptor` |
| **Business Delegate** | Decouple presentation from business tier | `@Service` called from `@Controller` |
| **Composite Entity** | Manage coarse-grained persistent objects | JPA `@Entity` with `@OneToMany` |

```java
// DAO Pattern → Spring Data Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
}

// Transfer Object (DTO) — never expose entity directly
public record UserDTO(Long id, String name, String email) {
    public static UserDTO from(User user) {
        return new UserDTO(user.getId(), user.getName(), user.getEmail());
    }
}

// Session Facade — service orchestrates multiple DAOs
@Service
public class UserFacade {
    private final UserRepository userRepo;
    private final AddressRepository addressRepo;
    private final EmailService emailService;

    public UserDTO registerUser(RegisterRequest req) {
        User user = userRepo.save(new User(req.name(), req.email()));
        addressRepo.save(new Address(user.getId(), req.address()));
        emailService.sendWelcome(user);
        return UserDTO.from(user);
    }
}

// Intercepting Filter → Spring HandlerInterceptor
@Component
public class RequestLoggingInterceptor implements HandlerInterceptor {
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
        log.info("{} {}", req.getMethod(), req.getRequestURI());
        return true; // continue to controller
    }
}
```

---

## Quick Reference Cheat Sheet

| Pattern / Principle | One-line Rule |
|---|---|
| **SRP** | One class = one reason to change |
| **OCP** | Extend by adding new classes, not modifying existing |
| **LSP** | Subtypes must be substitutable for their base type |
| **ISP** | Don't force classes to implement methods they don't need |
| **DIP** | Depend on interfaces, not concrete classes |
| **DDD** | Model code around business domain (Entities, Aggregates, Events) |
| **Hexagonal** | Domain at center; ports = interfaces; adapters = implementations |
| **Factory Method** | Subclass decides which object to create |
| **Abstract Factory** | Creates families of related objects |
| **Builder** | Step-by-step construction of complex objects |
| **Singleton** | One instance globally (use Spring beans, not static) |
| **Observer** | Publisher fires events; subscribers react independently |
| **Strategy** | Swap algorithms at runtime via interface |
| **Decorator** | Wrap object to add behavior without subclassing |
| **Proxy** | Intercept calls to add cross-cutting concerns (AOP) |
| **Circuit Breaker** | Stop calling failing service; return fallback |
| **Saga** | Distributed transaction via chained local transactions + compensation |
| **Caching** | Cache-aside: read from cache, miss → DB → populate cache |
| **Sharding** | Split data across DBs by shard key (hash or range) |
| **Observability** | Metrics (Prometheus) + Logs (ELK) + Traces (Jaeger) |
