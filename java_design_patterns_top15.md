# Top 15 Core Java Design Patterns with Coding Examples

Practical, interview-ready Java implementations of the most commonly asked design patterns. Grouped by the classic GoF categories: **Creational, Structural, Behavioral.**

---

## 🏗️ Creational Patterns

### 1. Singleton

**Intent:** Ensure a class has only one instance and provide a global point of access.

**Use case:** Configuration registry, connection pool holder, logger, cache.

**Best practice:** Use `enum` — thread-safe, serialization-safe, reflection-proof, lazy-loaded by JVM.

```java
public enum ConfigRegistry {
    INSTANCE;

    private final Map<String, String> settings = new ConcurrentHashMap<>();

    public String get(String key)          { return settings.get(key); }
    public void put(String key, String v)  { settings.put(key, v); }
}

// Usage
ConfigRegistry.INSTANCE.put("env", "prod");
String env = ConfigRegistry.INSTANCE.get("env");
```

**Alternative (lazy holder idiom):** thread-safe without synchronization overhead.

```java
public class DbPool {
    private DbPool() {}
    private static class Holder { static final DbPool I = new DbPool(); }
    public static DbPool getInstance() { return Holder.I; }
}
```

**Interview gotcha:** Double-checked locking needs `volatile` to be correct on JVMs pre-Java 5 memory model — and even then enum is simpler.

---

### 2. Factory Method

**Intent:** Define an interface for creating an object, but let subclasses (or a static method) decide which class to instantiate.

**Use case:** Decoupling client code from concrete classes — `Logger.getLogger()`, `Calendar.getInstance()`.

```java
sealed interface Notification permits EmailNotification, SmsNotification, PushNotification {
    void send(String to, String msg);
}

record EmailNotification() implements Notification {
    public void send(String to, String msg) { /* SMTP send */ }
}
record SmsNotification() implements Notification {
    public void send(String to, String msg) { /* Twilio send */ }
}
record PushNotification() implements Notification {
    public void send(String to, String msg) { /* FCM send */ }
}

public class NotificationFactory {
    public static Notification of(String channel) {
        return switch (channel.toLowerCase()) {
            case "email" -> new EmailNotification();
            case "sms"   -> new SmsNotification();
            case "push"  -> new PushNotification();
            default -> throw new IllegalArgumentException("unknown: " + channel);
        };
    }
}

// Usage
NotificationFactory.of("email").send("a@b.com", "Welcome");
```

---

### 3. Builder

**Intent:** Construct complex objects step-by-step. Useful when a class has many optional parameters.

**Use case:** Immutable DTOs, HTTP request builders (`HttpRequest.newBuilder()`), `StringBuilder`.

```java
public final class HttpRequest {
    private final String url;
    private final String method;
    private final Map<String, String> headers;
    private final String body;
    private final Duration timeout;

    private HttpRequest(Builder b) {
        this.url = b.url;
        this.method = b.method;
        this.headers = Map.copyOf(b.headers);
        this.body = b.body;
        this.timeout = b.timeout;
    }

    public static Builder builder() { return new Builder(); }

    public static final class Builder {
        private String url;
        private String method = "GET";
        private Map<String, String> headers = new HashMap<>();
        private String body = "";
        private Duration timeout = Duration.ofSeconds(30);

        public Builder url(String u)              { this.url = u; return this; }
        public Builder method(String m)           { this.method = m; return this; }
        public Builder header(String k, String v) { this.headers.put(k, v); return this; }
        public Builder body(String b)             { this.body = b; return this; }
        public Builder timeout(Duration d)        { this.timeout = d; return this; }

        public HttpRequest build() {
            Objects.requireNonNull(url, "url required");
            return new HttpRequest(this);
        }
    }
}

// Usage
HttpRequest req = HttpRequest.builder()
    .url("https://api.example.com/users")
    .method("POST")
    .header("Authorization", "Bearer ...")
    .body("{\"name\":\"Charan\"}")
    .timeout(Duration.ofSeconds(5))
    .build();
```

**Modern alternative:** `record` + static factory methods for simpler cases.

---

### 4. Prototype

**Intent:** Create new objects by cloning an existing instance (the prototype), avoiding expensive construction.

**Use case:** Heavy objects with mostly-shared state — game entities, document templates, ML model configurations.

```java
public class ReportTemplate implements Cloneable {
    private String title;
    private List<String> sections;
    private Map<String, String> styles;

    @Override
    public ReportTemplate clone() {
        try {
            ReportTemplate copy = (ReportTemplate) super.clone();
            copy.sections = new ArrayList<>(this.sections); // deep-copy mutable fields
            copy.styles = new HashMap<>(this.styles);
            return copy;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError(e);
        }
    }
}

// Usage
ReportTemplate base = loadDefaultTemplate();
ReportTemplate quarterly = base.clone();
quarterly.setTitle("Q2 Report");
```

**Modern alternative:** copy constructors or `record` `with` methods (Java 21+).

---

## 🧱 Structural Patterns

### 5. Adapter

**Intent:** Convert the interface of a class into another interface clients expect.

**Use case:** Integrating a legacy or third-party API with new code.

```java
// Existing interface our app uses
interface PaymentGateway {
    boolean charge(String customerId, BigDecimal amount);
}

// Third-party SDK we can't modify
class StripeSdk {
    public StripeResponse processPayment(String token, long amountInCents) { /* ... */ }
}

// Adapter bridges the two
class StripeAdapter implements PaymentGateway {
    private final StripeSdk sdk = new StripeSdk();

    @Override
    public boolean charge(String customerId, BigDecimal amount) {
        long cents = amount.multiply(BigDecimal.valueOf(100)).longValueExact();
        StripeResponse r = sdk.processPayment(customerId, cents);
        return r.isSuccess();
    }
}
```

---

### 6. Decorator

**Intent:** Add new behavior to an object dynamically without altering its class.

**Use case:** Java I/O streams (`BufferedReader`, `GzipInputStream`), HTTP middleware, logging/caching wrappers.

```java
interface DataSource {
    String read();
    void write(String data);
}

class FileDataSource implements DataSource {
    public String read()           { /* read file */ return ""; }
    public void write(String data) { /* write file */ }
}

// Base decorator
abstract class DataSourceDecorator implements DataSource {
    protected final DataSource wrappee;
    DataSourceDecorator(DataSource s) { this.wrappee = s; }
    public String read()           { return wrappee.read(); }
    public void write(String data) { wrappee.write(data); }
}

class EncryptionDecorator extends DataSourceDecorator {
    EncryptionDecorator(DataSource s) { super(s); }
    @Override public String read()           { return decrypt(super.read()); }
    @Override public void write(String data) { super.write(encrypt(data)); }
    private String encrypt(String s) { /* ... */ return s; }
    private String decrypt(String s) { /* ... */ return s; }
}

class CompressionDecorator extends DataSourceDecorator {
    CompressionDecorator(DataSource s) { super(s); }
    @Override public String read()           { return decompress(super.read()); }
    @Override public void write(String data) { super.write(compress(data)); }
    private String compress(String s)   { /* ... */ return s; }
    private String decompress(String s) { /* ... */ return s; }
}

// Usage — wrap layers
DataSource s = new EncryptionDecorator(new CompressionDecorator(new FileDataSource()));
s.write("sensitive data");  // compress, then encrypt, then save
```

---

### 7. Proxy

**Intent:** Provide a surrogate or placeholder for another object to control access to it.

**Use case:** Lazy loading, access control, remote proxies, caching, AOP-style logging.

```java
interface ImageService {
    byte[] load(String id);
}

class RealImageService implements ImageService {
    public byte[] load(String id) {
        // expensive: hit S3 / DB
        return new byte[]{};
    }
}

// Caching proxy
class CachingImageProxy implements ImageService {
    private final ImageService real = new RealImageService();
    private final Map<String, byte[]> cache = new ConcurrentHashMap<>();

    @Override
    public byte[] load(String id) {
        return cache.computeIfAbsent(id, real::load);
    }
}
```

Spring `@Transactional`, `@Cacheable`, and `@Async` are implemented as **dynamic proxies**.

---

### 8. Facade

**Intent:** Provide a unified, simplified interface to a complex subsystem.

**Use case:** API client wrapping multiple internal services; orchestrating microservices.

```java
// Complex subsystem
class InventoryService  { boolean reserve(String sku, int qty) { /* ... */ return true; } }
class PaymentService    { boolean charge(String customerId, BigDecimal amt) { return true; } }
class ShippingService   { String schedule(String sku, String addr) { return "TRACK-123"; } }
class NotificationService { void send(String customerId, String msg) {} }

// Facade
public class OrderFacade {
    private final InventoryService inv = new InventoryService();
    private final PaymentService pay = new PaymentService();
    private final ShippingService ship = new ShippingService();
    private final NotificationService notify = new NotificationService();

    public String placeOrder(String customer, String sku, int qty, BigDecimal amt, String addr) {
        if (!inv.reserve(sku, qty))               throw new IllegalStateException("out of stock");
        if (!pay.charge(customer, amt))           throw new IllegalStateException("payment failed");
        String tracking = ship.schedule(sku, addr);
        notify.send(customer, "Order placed: " + tracking);
        return tracking;
    }
}

// Usage — caller doesn't deal with any of the underlying services
String tracking = new OrderFacade().placeOrder("cust-1", "SKU-42", 1, new BigDecimal("99.99"), "BLR");
```

---

## 🎯 Behavioral Patterns

### 9. Strategy

**Intent:** Define a family of algorithms, encapsulate each one, and make them interchangeable.

**Use case:** Sort comparators, payment processors, compression algorithms, discount policies.

```java
@FunctionalInterface
interface PricingStrategy {
    BigDecimal apply(BigDecimal price);
}

class Cart {
    private final BigDecimal total;
    Cart(BigDecimal total) { this.total = total; }
    public BigDecimal checkout(PricingStrategy strategy) {
        return strategy.apply(total);
    }
}

// Strategies as lambdas
PricingStrategy noDiscount = p -> p;
PricingStrategy blackFriday = p -> p.multiply(new BigDecimal("0.70"));
PricingStrategy loyalty = p -> p.subtract(new BigDecimal("10"));

// Usage
Cart cart = new Cart(new BigDecimal("100"));
BigDecimal finalPrice = cart.checkout(blackFriday); // 70.00
```

Lambdas + functional interfaces make Strategy trivial in modern Java.

---

### 10. Observer

**Intent:** Define a one-to-many dependency so when one object changes state, all dependents are notified.

**Use case:** Event listeners, pub/sub, UI bindings, reactive streams.

```java
interface OrderListener {
    void onOrderPlaced(Order order);
}

class OrderService {
    private final List<OrderListener> listeners = new CopyOnWriteArrayList<>();

    public void subscribe(OrderListener l)   { listeners.add(l); }
    public void unsubscribe(OrderListener l) { listeners.remove(l); }

    public void placeOrder(Order o) {
        // save order...
        listeners.forEach(l -> l.onOrderPlaced(o));
    }
}

// Usage
OrderService svc = new OrderService();
svc.subscribe(o -> System.out.println("Email sent for " + o.id()));
svc.subscribe(o -> System.out.println("Inventory updated for " + o.id()));
svc.placeOrder(new Order("ORD-1"));
```

`CopyOnWriteArrayList` prevents `ConcurrentModificationException` if a listener unsubscribes during notification.

---

### 11. Command

**Intent:** Encapsulate a request as an object — allowing parameterization, queuing, logging, and undo.

**Use case:** Undo/redo (text editors), job queues, transactional operations, GUI button actions.

```java
interface Command {
    void execute();
    void undo();
}

class TextEditor {
    StringBuilder text = new StringBuilder();
}

class AddTextCommand implements Command {
    private final TextEditor editor;
    private final String add;

    AddTextCommand(TextEditor e, String add) { this.editor = e; this.add = add; }
    public void execute() { editor.text.append(add); }
    public void undo()    { editor.text.delete(editor.text.length() - add.length(), editor.text.length()); }
}

class CommandHistory {
    private final Deque<Command> stack = new ArrayDeque<>();
    public void execute(Command c) { c.execute(); stack.push(c); }
    public void undo()             { if (!stack.isEmpty()) stack.pop().undo(); }
}

// Usage
var editor = new TextEditor();
var history = new CommandHistory();
history.execute(new AddTextCommand(editor, "Hello, "));
history.execute(new AddTextCommand(editor, "World!"));
history.undo();  // removes "World!"
```

---

### 12. Template Method

**Intent:** Define the skeleton of an algorithm in a base class, letting subclasses override specific steps.

**Use case:** Frameworks (Spring `JdbcTemplate`, Servlet `service()` → `doGet()`/`doPost()`).

```java
abstract class DataProcessor {
    // Template method — final so subclasses can't change the skeleton
    public final void process() {
        var data = read();
        var transformed = transform(data);
        write(transformed);
    }

    protected abstract List<String> read();
    protected abstract List<String> transform(List<String> data);
    protected abstract void write(List<String> data);
}

class CsvProcessor extends DataProcessor {
    protected List<String> read()                       { return List.of("a,1", "b,2"); }
    protected List<String> transform(List<String> data) { return data.stream().map(String::toUpperCase).toList(); }
    protected void write(List<String> data)             { data.forEach(System.out::println); }
}
```

---

### 13. Iterator

**Intent:** Provide a way to access elements of a collection sequentially without exposing its underlying representation.

**Use case:** Custom collections, paged result sets, streaming APIs.

Java has this built-in via `Iterable<T>` and `Iterator<T>`. Implementing for a custom type:

```java
class CircularBuffer<T> implements Iterable<T> {
    private final Object[] buf;
    private int head = 0, size = 0;

    CircularBuffer(int cap) { this.buf = new Object[cap]; }

    public void add(T v) {
        buf[(head + size) % buf.length] = v;
        if (size < buf.length) size++; else head = (head + 1) % buf.length;
    }

    @Override
    public Iterator<T> iterator() {
        return new Iterator<>() {
            int i = 0;
            @Override public boolean hasNext() { return i < size; }
            @Override @SuppressWarnings("unchecked")
            public T next() {
                if (!hasNext()) throw new NoSuchElementException();
                return (T) buf[(head + i++) % buf.length];
            }
        };
    }
}

// Usage
var cb = new CircularBuffer<Integer>(3);
cb.add(1); cb.add(2); cb.add(3); cb.add(4);
for (int x : cb) System.out.println(x);  // 2, 3, 4
```

---

### 14. State

**Intent:** Allow an object to alter its behavior when its internal state changes — appears as if the object changed its class.

**Use case:** Workflow engines, vending machines, document lifecycle, TCP connection states.

```java
sealed interface OrderState permits New, Paid, Shipped, Delivered, Cancelled {}
record New() implements OrderState {}
record Paid() implements OrderState {}
record Shipped(String trackingId) implements OrderState {}
record Delivered(Instant at) implements OrderState {}
record Cancelled(String reason) implements OrderState {}

class Order {
    private OrderState state = new New();

    public void pay() {
        state = switch (state) {
            case New n -> new Paid();
            default -> throw new IllegalStateException("Cannot pay from " + state);
        };
    }

    public void ship(String tracking) {
        state = switch (state) {
            case Paid p -> new Shipped(tracking);
            default -> throw new IllegalStateException("Cannot ship from " + state);
        };
    }

    public void deliver() {
        state = switch (state) {
            case Shipped s -> new Delivered(Instant.now());
            default -> throw new IllegalStateException("Cannot deliver from " + state);
        };
    }

    public OrderState getState() { return state; }
}
```

Sealed interfaces + pattern matching (Java 21) make state machines elegant and exhaustive.

---

### 15. Chain of Responsibility

**Intent:** Pass a request along a chain of handlers — each decides to handle or forward.

**Use case:** Servlet filters, Spring Security filter chain, middleware (logging → auth → rate-limit → handler), exception handling.

```java
abstract class RequestHandler {
    protected RequestHandler next;

    public RequestHandler linkWith(RequestHandler n) { this.next = n; return n; }

    public final void handle(Request req) {
        if (process(req) && next != null) next.handle(req);
    }

    protected abstract boolean process(Request req); // true → continue chain
}

class AuthHandler extends RequestHandler {
    protected boolean process(Request req) {
        if (req.token() == null) { req.reject("unauthorized"); return false; }
        return true;
    }
}

class RateLimitHandler extends RequestHandler {
    private final Map<String, Integer> counts = new ConcurrentHashMap<>();
    protected boolean process(Request req) {
        if (counts.merge(req.clientIp(), 1, Integer::sum) > 100) {
            req.reject("rate limited");
            return false;
        }
        return true;
    }
}

class LoggingHandler extends RequestHandler {
    protected boolean process(Request req) {
        System.out.println("REQ: " + req);
        return true;
    }
}

// Build the chain
RequestHandler chain = new LoggingHandler();
chain.linkWith(new AuthHandler()).linkWith(new RateLimitHandler());

chain.handle(new Request(/* ... */));
```

---

## 🎓 Quick-Reference Summary

| # | Pattern | Type | One-line purpose |
|---|---|---|---|
| 1 | Singleton | Creational | One instance, global access |
| 2 | Factory Method | Creational | Decouple creation from concrete class |
| 3 | Builder | Creational | Step-by-step construction of complex objects |
| 4 | Prototype | Creational | Clone an existing instance |
| 5 | Adapter | Structural | Bridge incompatible interfaces |
| 6 | Decorator | Structural | Add behavior dynamically by wrapping |
| 7 | Proxy | Structural | Surrogate that controls access (lazy, cache, security) |
| 8 | Facade | Structural | One simple interface over a complex subsystem |
| 9 | Strategy | Behavioral | Interchangeable algorithms behind one interface |
| 10 | Observer | Behavioral | One-to-many event notification |
| 11 | Command | Behavioral | Encapsulate request as object (undo, queue, log) |
| 12 | Template Method | Behavioral | Algorithm skeleton with overridable steps |
| 13 | Iterator | Behavioral | Sequential access without exposing internals |
| 14 | State | Behavioral | Behavior changes with internal state |
| 15 | Chain of Responsibility | Behavioral | Pass request along a handler chain |

---

## 💡 Interview Tips

1. **Don't over-engineer.** Patterns solve problems — applying them to trivial code is an anti-pattern itself.
2. **Know real-world examples** in the JDK / Spring:
   - Singleton → `Runtime.getRuntime()`
   - Builder → `StringBuilder`, `HttpRequest.newBuilder()`, Lombok `@Builder`
   - Factory Method → `Calendar.getInstance()`, `Logger.getLogger()`
   - Decorator → `BufferedReader(new FileReader(...))`
   - Proxy → Spring AOP, JDK dynamic proxies
   - Observer → `java.util.Observer` (deprecated), Spring `ApplicationEventPublisher`
   - Strategy → `Comparator`, `Runnable`
   - Template Method → `JdbcTemplate`, `HttpServlet.service()`
   - Chain of Responsibility → Servlet `Filter`, Spring Security
3. **Mention modern alternatives** — lambdas often replace Strategy/Command, `record` replaces some Builders, sealed types + pattern matching simplify State.
4. **Be ready to spot anti-patterns** — Singleton overuse (god object), excessive inheritance instead of composition, premature abstraction.
5. **SOLID first** — patterns are tools to achieve SOLID, not a goal in themselves.
