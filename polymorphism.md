# Interview Preparation — Senior Software Engineer
**Candidate:** Prarabdh Singh | **Req:** JR0027050  
**Hiring Manager:** Bhavin Shah (Core Mail Data)  
**Interviewer Coordinator:** Nirmal Thangaraj  
**Focus Areas:** Polymorphism · Inheritance · Composition · OOP Design

---

## Section 1 — Polymorphism (10 Questions)

---

### Q1. What is polymorphism? Explain compile-time vs runtime polymorphism with examples.

**Answer:**  
Polymorphism = "many forms" — the same interface behaves differently depending on the actual object.

**Compile-time (Static) — Method Overloading:**
```java
class Calculator {
    int add(int a, int b)          { return a + b; }
    double add(double a, double b) { return a + b; }
    int add(int a, int b, int c)   { return a + b + c; }
}
```
Resolved at **compile time** by the compiler based on method signature.

**Runtime (Dynamic) — Method Overriding:**
```java
class Animal {
    void sound() { System.out.println("Generic sound"); }
}
class Dog extends Animal {
    @Override
    void sound() { System.out.println("Woof"); }
}
class Cat extends Animal {
    @Override
    void sound() { System.out.println("Meow"); }
}

Animal a = new Dog();
a.sound(); // prints "Woof" — resolved at runtime via vtable
```
Resolved at **runtime** via dynamic dispatch (JVM method table).

---

### Q2. How does the JVM achieve runtime polymorphism internally?

**Answer:**  
The JVM uses a **vtable (virtual method table)**:
- Each class has a vtable — a list of method pointers.
- When a method is called on a reference, JVM looks up the actual object's class vtable at runtime.
- `invokevirtual` bytecode instruction triggers dynamic dispatch.
- `invokespecial` is used for `private`, `static`, constructors — no dynamic dispatch.

```
Animal ref → Dog object → Dog's vtable → Dog.sound()
```

`final` methods skip vtable lookup (devirtualization optimization by JIT).

---

### Q3. What is covariant return type? How does it relate to polymorphism?

**Answer:**  
Covariant return type allows an overriding method to return a **subtype** of the parent method's return type.

```java
class Animal {
    Animal create() { return new Animal(); }
}
class Dog extends Animal {
    @Override
    Dog create() { return new Dog(); } // Dog is subtype of Animal — valid
}
```
- Introduced in Java 5.
- Enables fluent APIs and builder patterns without casting.
- The override is still polymorphic — resolved at runtime.

---

### Q4. Can we achieve polymorphism with interfaces? How is it different from class-based polymorphism?

**Answer:**  
Yes — interfaces are the **preferred mechanism** for polymorphism in Java.

```java
interface Shape {
    double area();
}
class Circle implements Shape {
    double radius;
    public double area() { return Math.PI * radius * radius; }
}
class Rectangle implements Shape {
    double w, h;
    public double area() { return w * h; }
}

List<Shape> shapes = List.of(new Circle(), new Rectangle());
shapes.forEach(s -> System.out.println(s.area())); // runtime dispatch
```

**Difference from class-based:**
| | Interface | Abstract Class |
|---|---|---|
| Multiple inheritance | Yes (multiple interfaces) | No |
| State | No (default: stateless) | Yes |
| Constructor | No | Yes |
| Use case | Capability contract | Shared base behaviour |

---

### Q5. What is parametric polymorphism? How do Java Generics implement it?

**Answer:**  
Parametric polymorphism = same code works for any type via **type parameters**.

```java
class Box<T> {
    private T value;
    public void set(T value) { this.value = value; }
    public T get()           { return value; }
}

Box<String>  strBox = new Box<>();
Box<Integer> intBox = new Box<>();
```

Java uses **type erasure** — generic type info is removed at compile time:
- `Box<String>` becomes `Box` at runtime.
- Type checks inserted by compiler as casts.
- Implication: `instanceof Box<String>` is illegal; reflection loses generic info.

---

### Q6. What is the difference between method overloading and method hiding (static methods)?

**Answer:**  
```java
class Parent {
    static void display() { System.out.println("Parent static"); }
    void show()           { System.out.println("Parent instance"); }
}
class Child extends Parent {
    static void display() { System.out.println("Child static"); }  // hiding
    @Override void show() { System.out.println("Child instance"); } // overriding
}

Parent p = new Child();
p.display(); // "Parent static"  — static: resolved at compile time by ref type
p.show();    // "Child instance" — instance: resolved at runtime by object type
```

**Key rule:** Static methods are **hidden**, not overridden — no runtime polymorphism.

---

### Q7. Explain ad-hoc polymorphism vs subtype polymorphism.

**Answer:**

| Type | Java Mechanism | Resolution |
|---|---|---|
| **Ad-hoc** | Method overloading, operator overloading (not in Java) | Compile time |
| **Subtype** | Method overriding via inheritance/interface | Runtime |
| **Parametric** | Generics | Compile time (type erasure) |

Subtype polymorphism follows the **Liskov Substitution Principle** — any subtype must be usable wherever the supertype is expected without breaking behaviour.

---

### Q8. What is duck typing and does Java support it?

**Answer:**  
Duck typing = "If it walks like a duck and quacks like a duck, it's a duck" — type determined by **capability**, not inheritance.

**Java (statically typed) does NOT natively support duck typing.** You must explicitly declare `implements Interface`.

However, Java approximates it via:
1. **Interfaces** — structural contract without implementation coupling.
2. **Reflection** — check methods at runtime (not type-safe).
3. **Dynamic proxies** — `java.lang.reflect.Proxy`.

Java 8+ `var` and lambda expressions bring some flavour, but still fully type-checked at compile time.

---

### Q9. How would you prevent a method from being overridden while still allowing inheritance?

**Answer:**  
Use the `final` modifier on the method:

```java
class Base {
    final void criticalMethod() {
        // cannot be overridden
    }
    void normalMethod() {
        // can be overridden
    }
}
class Derived extends Base {
    // void criticalMethod() { } // compile error
    @Override void normalMethod() { } // fine
}
```

Use cases for `final` methods:
- Security-sensitive logic (authentication, crypto).
- Template method pattern skeleton steps.
- Performance — JIT can inline final methods (devirtualisation).

---

### Q10. Real-world scenario: You have a payment processing system with Card, UPI, and NetBanking. How would you design it using polymorphism?

**Answer:**

```java
interface PaymentProcessor {
    PaymentResult process(PaymentRequest request);
    boolean supports(PaymentMethod method);
}

class CardProcessor implements PaymentProcessor {
    public PaymentResult process(PaymentRequest req) { /* card logic */ }
    public boolean supports(PaymentMethod m) { return m == PaymentMethod.CARD; }
}

class UpiProcessor implements PaymentProcessor {
    public PaymentResult process(PaymentRequest req) { /* UPI logic */ }
    public boolean supports(PaymentMethod m) { return m == PaymentMethod.UPI; }
}

class PaymentService {
    private List<PaymentProcessor> processors;

    public PaymentResult pay(PaymentRequest req) {
        return processors.stream()
            .filter(p -> p.supports(req.getMethod()))
            .findFirst()
            .orElseThrow(() -> new UnsupportedPaymentException())
            .process(req);
    }
}
```

This is the **Strategy + Polymorphism** pattern. Adding a new payment method requires only a new class — no modification of existing code (**Open/Closed Principle**).

---

## Section 2 — Inheritance (10 Questions)

---

### Q11. What is the difference between `extends` and `implements`? When would you choose one over the other?

**Answer:**

| | `extends` | `implements` |
|---|---|---|
| Purpose | Inherit state + behaviour | Fulfil a contract |
| Multiple | No (single class) | Yes (multiple interfaces) |
| Constructor | Inherited (super()) | N/A |
| Use when | IS-A with shared implementation | IS-A with capability contract |

**Rule of thumb:** Prefer `implements` (interfaces) for polymorphism. Use `extends` only when:
- Significant shared implementation exists.
- The relationship is truly IS-A (not just behavioural).

**Bad use of extends:**
```java
class Stack extends Vector { } // Java's mistake — Stack IS-NOT-A Vector
```

**Good:**
```java
class EmailNotifier implements Notifier { }
class SmsNotifier  implements Notifier { }
```

---

### Q12. Explain constructor chaining in inheritance. What happens if you don't call `super()`?

**Answer:**

```java
class Animal {
    Animal(String name) {
        System.out.println("Animal: " + name);
    }
}
class Dog extends Animal {
    Dog(String name, String breed) {
        super(name);                          // must be first statement
        System.out.println("Dog: " + breed);
    }
}
```

- If you don't explicitly call `super()`, Java inserts `super()` (no-arg) automatically.
- If parent has no no-arg constructor, **compile error**.
- `this()` and `super()` cannot both be the first statement.

**Initialisation order:**
1. Static blocks (parent → child)
2. Instance blocks + fields (parent → child)
3. Constructors (parent → child)

---

### Q13. What is the diamond problem? How does Java handle it with interfaces?

**Answer:**

The diamond problem occurs when a class inherits the same method from two paths:

```
     A.greet()
    /         \
   B            C
    \         /
         D       ← which greet() does D inherit?
```

**Java's resolution for interfaces (Java 8+ default methods):**

```java
interface A { default void greet() { System.out.println("A"); } }
interface B extends A { default void greet() { System.out.println("B"); } }
interface C extends A { default void greet() { System.out.println("C"); } }

class D implements B, C {
    @Override
    public void greet() {
        B.super.greet(); // explicitly choose
    }
}
```

**Rules:**
1. Class method wins over interface default.
2. More specific interface wins over less specific.
3. Ambiguity → compiler error → must override explicitly.

---

### Q14. What is method hiding vs method overriding? What is the role of `@Override`?

**Answer:**

| | Overriding | Hiding |
|---|---|---|
| Applies to | Instance methods | Static methods |
| Resolution | Runtime (JVM vtable) | Compile time (reference type) |
| Annotation | `@Override` valid | `@Override` NOT valid for static |

```java
class Parent {
    void instanceMethod() { System.out.println("Parent instance"); }
    static void staticMethod() { System.out.println("Parent static"); }
}
class Child extends Parent {
    @Override
    void instanceMethod() { System.out.println("Child instance"); } // override

    static void staticMethod() { System.out.println("Child static"); } // hide
}

Parent ref = new Child();
ref.instanceMethod(); // Child instance — runtime
ref.staticMethod();   // Parent static — compile time
```

`@Override` — not mandatory but **strongly recommended**:
- Compiler error if method doesn't actually override anything.
- Documents intent clearly.

---

### Q15. What is fragile base class problem? How do you mitigate it?

**Answer:**  
When a change to a base class breaks subclasses that were working fine — even without changing the subclass.

```java
class Base {
    void methodA() { methodB(); } // calls methodB internally
    void methodB() { }
}
class Child extends Base {
    @Override void methodB() { /* custom logic */ }
}
// If Base.methodA() stops calling methodB(), Child's override is silently bypassed
```

**Mitigations:**
1. **Favour composition over inheritance** (see Section 3).
2. Use `final` on methods not designed for override.
3. Document clearly which methods are designed for override (Javadoc `@implSpec`).
4. Follow **Open/Closed Principle** — extend, don't modify.
5. Use abstract methods to explicitly require overrides.

---

### Q16. Explain `super` keyword — all use cases.

**Answer:**

```java
class Animal {
    String name = "Animal";
    Animal(String n) { this.name = n; }
    void sound() { System.out.println("Generic"); }
}

class Dog extends Animal {
    String name = "Dog"; // shadows parent field

    Dog(String n) {
        super(n);               // 1. Call parent constructor (must be first)
    }

    void info() {
        System.out.println(super.name);  // 2. Access parent field
        super.sound();                   // 3. Call parent method
    }
}
```

**Use cases:**
1. `super(args)` — invoke parent constructor.
2. `super.field` — access parent field hidden by child.
3. `super.method()` — invoke parent's overridden method.

**Cannot use `super` in static context** — no instance association.

---

### Q17. What is abstract class? When would you use abstract class over interface?

**Answer:**

```java
abstract class Template {
    // Template Method Pattern
    final void execute() {
        step1();         // common
        step2();         // custom
        step3();         // common
    }
    void step1() { /* default impl */ }
    abstract void step2(); // must override
    void step3() { /* default impl */ }
}
```

**Use abstract class when:**
- Sharing **state** (fields) across subclasses.
- Providing partial implementation (template method pattern).
- Strong IS-A relationship with common behaviour.
- Lifecycle management (constructors, protected hooks).

**Use interface when:**
- Defining a **capability contract** (Serializable, Comparable).
- Multiple inheritance of type is needed.
- No shared state required.

---

### Q18. What are the rules for method overriding? What is the impact of access modifiers?

**Answer:**

```java
class Parent {
    protected Number getValue() { return 42; }
}
class Child extends Parent {
    @Override
    public Integer getValue() { // valid: public > protected, Integer < Number (covariant)
        return 100;
    }
}
```

**Rules:**
1. Same method name and parameters.
2. Return type must be same or **covariant** (subtype).
3. Access modifier can be **same or wider** (never narrower).
   - `protected` → `public` ✓ | `public` → `protected` ✗
4. Cannot throw **new or broader checked exceptions** than parent.
5. Cannot override `final`, `static`, or `private` methods.
6. `@Override` recommended to catch mistakes.

---

### Q19. How does inheritance work with serialization in Java?

**Answer:**

```java
class Parent {          // NOT Serializable
    int parentField;
}
class Child extends Parent implements Serializable {
    int childField;
    private static final long serialVersionUID = 1L;
}
```

**Key rules:**
- Only `Child`'s fields are serialized.
- `Parent`'s fields (`parentField`) are NOT serialized since `Parent` is not `Serializable`.
- On deserialization, `Parent`'s **no-arg constructor** is called to initialise its fields.
- If `Parent` has no no-arg constructor → `InvalidClassException`.

**Solution:** Either make `Parent` implement `Serializable`, or implement `readObject`/`writeObject` to handle parent fields manually.

---

### Q20. Real-world: Design a notification system (Email, SMS, Push) using inheritance. What problems might arise and how would you fix them?

**Answer:**

**Naive (problematic) design:**
```java
abstract class Notifier {
    abstract void send(String msg);
    void log(String msg) { /* log */ }
    void retry(String msg) { /* retry logic */ }
}
class EmailNotifier extends Notifier {
    void send(String msg) { /* email */ }
}
```

**Problems:**
- Can't send via Email + SMS together (single inheritance).
- Retry/log logic locked in base class — not reusable elsewhere.
- Adding `SlackNotifier` breaks if Slack needs different retry logic.

**Better design (composition + interface):**
```java
interface Notifier {
    void send(String msg);
}
class EmailNotifier  implements Notifier { public void send(String m) { /* */ } }
class SmsNotifier    implements Notifier { public void send(String m) { /* */ } }

class CompositeNotifier implements Notifier {
    private List<Notifier> notifiers;
    public void send(String m) {
        notifiers.forEach(n -> n.send(m)); // fan-out
    }
}

class RetryingNotifier implements Notifier { // decorator
    private Notifier delegate;
    public void send(String m) {
        // retry logic wrapping delegate.send(m)
    }
}
```

Uses **Composite + Decorator patterns** — far more flexible.

---

## Section 3 — Composition (10 Questions)

---

### Q21. What is "Favour Composition over Inheritance"? When should you break this rule?

**Answer:**

**Composition** = a class contains references to other objects and delegates work to them.

```java
// Inheritance (tight coupling)
class LoggingList extends ArrayList<String> {
    @Override public boolean add(String s) {
        log(s);
        return super.add(s);
    }
}

// Composition (loose coupling — preferred)
class LoggingList {
    private List<String> delegate = new ArrayList<>();
    public boolean add(String s) {
        log(s);
        return delegate.add(s);
    }
}
```

**Why composition is preferred:**
- Avoids fragile base class problem.
- Allows changing implementation at runtime.
- Supports multiple behaviours without multiple inheritance.

**When inheritance IS appropriate:**
- True IS-A relationship (not just behavioural).
- Framework classes designed for extension (`HttpServlet`).
- When shared state across hierarchy is unavoidable.

---

### Q22. What is the Decorator pattern? How is it an example of composition?

**Answer:**

Adds behaviour by wrapping objects — without modifying the original class.

```java
interface Coffee {
    double cost();
    String description();
}
class SimpleCoffee implements Coffee {
    public double cost()        { return 1.0; }
    public String description() { return "Coffee"; }
}

// Decorator base
abstract class CoffeeDecorator implements Coffee {
    protected Coffee coffee;
    CoffeeDecorator(Coffee c) { this.coffee = c; }
}

class MilkDecorator extends CoffeeDecorator {
    MilkDecorator(Coffee c) { super(c); }
    public double cost()        { return coffee.cost() + 0.5; }
    public String description() { return coffee.description() + ", Milk"; }
}
class SugarDecorator extends CoffeeDecorator {
    SugarDecorator(Coffee c) { super(c); }
    public double cost()        { return coffee.cost() + 0.25; }
    public String description() { return coffee.description() + ", Sugar"; }
}

Coffee c = new SugarDecorator(new MilkDecorator(new SimpleCoffee()));
// cost: 1.75, description: "Coffee, Milk, Sugar"
```

Java uses this pattern: `BufferedReader` wraps `FileReader`, `Collections.unmodifiableList()` wraps `List`.

---

### Q23. What is dependency injection? How does it enable composition?

**Answer:**

DI = inject dependencies from outside rather than creating them internally.

```java
// Tight coupling (bad)
class OrderService {
    private PaymentProcessor processor = new StripeProcessor(); // hardcoded
}

// Composition via DI (good)
class OrderService {
    private final PaymentProcessor processor; // injected

    OrderService(PaymentProcessor processor) { // constructor injection
        this.processor = processor;
    }
}

// Wiring (Spring does this automatically)
OrderService svc = new OrderService(new StripeProcessor());
// or for testing:
OrderService svc = new OrderService(new MockPaymentProcessor());
```

**DI types in Spring:**
1. **Constructor injection** — preferred (immutable, testable).
2. **Setter injection** — optional dependencies.
3. **Field injection** (`@Autowired` on field) — avoid (hard to test, hides dependencies).

---

### Q24. What is the difference between HAS-A and IS-A relationships?

**Answer:**

| | IS-A | HAS-A |
|---|---|---|
| Mechanism | Inheritance (`extends`/`implements`) | Composition (field reference) |
| Coupling | Tight | Loose |
| Flexibility | Low | High |
| Example | `Dog IS-A Animal` | `Car HAS-A Engine` |

```java
// IS-A
class Dog extends Animal { }

// HAS-A (composition)
class Car {
    private Engine engine;       // HAS-A Engine
    private List<Wheel> wheels;  // HAS-A Wheels

    Car(Engine e, List<Wheel> w) {
        this.engine = e;
        this.wheels = w;
    }
}
```

**Test:** "Can I say X IS-A Y and have it make sense in ALL contexts?" If not, use HAS-A.

`Stack IS-A Vector`? — NO. Stack should not expose Vector methods like `insertElementAt()`. Use composition.

---

### Q25. What is the Strategy pattern? How does it use composition to replace inheritance?

**Answer:**

Replaces conditional logic / subclassing by injecting an algorithm at runtime.

```java
// Without Strategy: subclass explosion
class SortedList extends ArrayList { void sort() { /* bubble sort */ } }
class FastSortedList extends ArrayList { void sort() { /* quicksort */ } }

// With Strategy: composition
interface SortStrategy {
    void sort(List<?> list);
}
class BubbleSort implements SortStrategy { public void sort(List<?> l) { /* */ } }
class QuickSort  implements SortStrategy { public void sort(List<?> l) { /* */ } }
class MergeSort  implements SortStrategy { public void sort(List<?> l) { /* */ } }

class SortedList {
    private SortStrategy strategy; // composed

    SortedList(SortStrategy s) { this.strategy = s; }

    void setStrategy(SortStrategy s) { this.strategy = s; } // runtime switch!

    void sort(List<?> l) { strategy.sort(l); }
}
```

Switching sort algorithm = swap strategy object. No new subclass needed.

---

### Q26. What is the difference between aggregation and composition in OOP?

**Answer:**

Both are HAS-A. They differ in **lifecycle dependency**.

| | Composition | Aggregation |
|---|---|---|
| Lifecycle | Child dies with parent | Child lives independently |
| Ownership | Parent owns child | Parent borrows child |
| Example | House–Room | University–Student |

```java
// Composition: Room cannot exist without House
class House {
    private List<Room> rooms = new ArrayList<>(); // House creates Room
    House() { rooms.add(new Room()); }
    // when House is GC'd, Rooms are too
}

// Aggregation: Student exists independently of University
class University {
    private List<Student> students; // University receives Student
    University(List<Student> s) { this.students = s; }
    // Student objects live on after University is GC'd
}
```

---

### Q27. How would you use composition to make a class thread-safe without subclassing?

**Answer:**

```java
// Original non-thread-safe class (third-party, can't modify)
class Counter {
    private int count = 0;
    public int increment() { return ++count; }
    public int get()       { return count; }
}

// Thread-safe wrapper via composition
class ThreadSafeCounter {
    private final Counter counter = new Counter();
    private final Object lock = new Object();

    public int increment() {
        synchronized (lock) { return counter.increment(); }
    }
    public int get() {
        synchronized (lock) { return counter.get(); }
    }
}
```

- No subclassing needed — not fragile.
- Lock is encapsulated — clients can't bypass it.
- Can also use `AtomicInteger` as the composed type for lock-free version.

---

### Q28. Explain mixin-like behaviour in Java using interfaces with default methods.

**Answer:**

Mixins add reusable behaviour to a class without using the inheritance hierarchy.

```java
interface Auditable {
    default void logCreated()  { System.out.println("Created: "  + this); }
    default void logModified() { System.out.println("Modified: " + this); }
}

interface Validatable {
    default boolean validate() {
        // common validation logic
        return true;
    }
}

class Order implements Auditable, Validatable {
    // gets both logCreated(), logModified(), validate() for free
    // no inheritance chain needed
}
```

This is as close as Java gets to mixins. Limitation: default methods can't access state (no fields in interfaces) — use abstract class if state needed.

---

### Q29. How does the Proxy pattern use composition? Give a real-world example.

**Answer:**

Proxy wraps an object to add cross-cutting concerns (lazy loading, caching, security, logging) **without modifying the original**.

```java
interface UserService {
    User findById(long id);
}

class UserServiceImpl implements UserService {
    public User findById(long id) { /* DB call */ return new User(id); }
}

class CachingUserServiceProxy implements UserService {
    private final UserService delegate;
    private final Map<Long, User> cache = new ConcurrentHashMap<>();

    CachingUserServiceProxy(UserService s) { this.delegate = s; }

    public User findById(long id) {
        return cache.computeIfAbsent(id, delegate::findById); // composed
    }
}
```

**Spring AOP** uses dynamic proxies (`java.lang.reflect.Proxy` / CGLIB) to implement `@Transactional`, `@Cacheable`, `@Secured` — all composition-based at runtime.

---

### Q30. Senior design question: Design a logging framework for a microservice using OOP principles (polymorphism, inheritance, composition). What design decisions would you make?

**Answer:**

```java
// Abstraction (polymorphism)
interface Logger {
    void log(LogEvent event);
}

// Log event (immutable value object)
class LogEvent {
    final Level level;
    final String message;
    final Instant timestamp;
    final Map<String, String> context; // MDC/trace info
    // constructor + getters
}

enum Level { DEBUG, INFO, WARN, ERROR }

// Concrete implementations (polymorphism)
class ConsoleLogger implements Logger {
    public void log(LogEvent e) { System.out.printf("[%s] %s%n", e.level, e.message); }
}
class FileLogger implements Logger {
    private final Path path;
    public void log(LogEvent e) { /* write to file */ }
}
class JsonLogger implements Logger {
    public void log(LogEvent e) { /* write JSON to output stream */ }
}

// Composition: Filtering decorator
class FilteringLogger implements Logger {
    private final Logger delegate;
    private final Level minLevel;
    FilteringLogger(Logger l, Level min) { delegate = l; minLevel = min; }
    public void log(LogEvent e) {
        if (e.level.ordinal() >= minLevel.ordinal()) delegate.log(e);
    }
}

// Composition: Fan-out composite
class CompositeLogger implements Logger {
    private final List<Logger> loggers;
    CompositeLogger(Logger... l) { this.loggers = Arrays.asList(l); }
    public void log(LogEvent e) { loggers.forEach(l -> l.log(e)); }
}

// Composition: Async decorator
class AsyncLogger implements Logger {
    private final Logger delegate;
    private final ExecutorService executor = Executors.newSingleThreadExecutor();
    AsyncLogger(Logger l) { delegate = l; }
    public void log(LogEvent e) { executor.submit(() -> delegate.log(e)); }
}

// Usage
Logger logger = new AsyncLogger(
    new FilteringLogger(
        new CompositeLogger(new ConsoleLogger(), new FileLogger(path)),
        Level.INFO
    )
);
logger.log(new LogEvent(Level.ERROR, "Payment failed", Instant.now(), ctx));
```

**Design decisions:**
- **Interface-first** — swap implementations without client code changes.
- **Decorator for cross-cutting** — filtering, async, retry without modifying core classes.
- **Composite for fan-out** — log to multiple destinations transparently.
- **Immutable LogEvent** — safe to share across async threads.
- **No inheritance hierarchy** — avoids fragile base class problem.
- **Open/Closed** — new log targets = new class, no modifications.

---

## Quick Reference — Senior-Level Talking Points

| Concept | Key Insight to Mention |
|---|---|
| Runtime polymorphism | JVM vtable, `invokevirtual`, dynamic dispatch |
| Compile-time polymorphism | Overloading resolved by compiler by signature |
| Diamond problem | Java resolves with explicit `Interface.super.method()` |
| Fragile base class | Reason to prefer composition |
| Composition over inheritance | Loose coupling, runtime flexibility, testability |
| Decorator | Wraps objects to add behaviour — used in Java I/O, Spring AOP |
| Strategy | Replace subclass explosion with injectable algorithm |
| DI | Constructor injection preferred — immutable, testable |
| LSP | Subtype must be substitutable without breaking behaviour |
| SOLID | Tie all OOP concepts back to SOLID principles |
