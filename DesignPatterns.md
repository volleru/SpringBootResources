# Design Patterns Reference

---

## GOF Patterns (Gang of Four)

> 23 classic patterns from the book *"Design Patterns"* by Gamma, Helm, Johnson & Vlissides (1994).  
> Grouped into 3 categories: **Creational**, **Structural**, **Behavioral**.

---

### Creational Patterns — *How objects are created*

| Pattern | When to Use | Real-World Example |
|---|---|---|
| **Singleton** | Need exactly one instance shared app-wide | DB connection pool, Logger, Config manager |
| **Factory Method** | Subclass decides which object to create; caller doesn't know the type | `DocumentFactory` creates PDF or Word doc based on input |
| **Abstract Factory** | Create families of related objects without specifying concrete classes | UI toolkit: `WindowsFactory` vs `MacFactory` creates buttons/checkboxes |
| **Builder** | Object has many optional fields; want to build it step-by-step | Building an HTTP request, SQL query, or complex config object |
| **Prototype** | Creating a new object is expensive; clone an existing one instead | Cloning a pre-configured game character or report template |

---

### Structural Patterns — *How objects are composed*

| Pattern | When to Use | Real-World Example |
|---|---|---|
| **Adapter** | Make an incompatible interface work with your code | Wrapping a legacy payment API to match your new `PaymentGateway` interface |
| **Bridge** | Separate abstraction from implementation so both can vary independently | `RemoteControl` (abstraction) works with `TV` or `Radio` (implementation) |
| **Composite** | Treat individual objects and groups of objects uniformly | File system: `File` and `Folder` both implement `FileSystemItem` |
| **Decorator** | Add behavior to an object at runtime without changing its class | Adding logging, caching, or compression to an existing service |
| **Facade** | Simplify a complex subsystem behind a single clean interface | `OrderFacade` hides inventory, payment, and shipping service calls |
| **Flyweight** | Share common state among many small objects to save memory | Rendering thousands of tree objects in a game — share texture/color |
| **Proxy** | Control access to an object (lazy load, security, logging) | Virtual proxy for lazy DB loading; security proxy to check permissions |

---

### Behavioral Patterns — *How objects communicate*

| Pattern | When to Use | Real-World Example |
|---|---|---|
| **Chain of Responsibility** | Pass a request along a chain; each handler decides to process or forward | Middleware pipeline, support ticket escalation (L1 → L2 → L3) |
| **Command** | Encapsulate a request as an object; support undo/redo, queuing | Text editor undo/redo, job queue, transaction log |
| **Iterator** | Traverse a collection without exposing its internal structure | Java's `for-each`, cursor over DB result set |
| **Mediator** | Reduce direct dependencies between many objects; centralize communication | Chat room (mediator) where users don't talk to each other directly |
| **Memento** | Capture and restore an object's state without breaking encapsulation | Undo in a text editor, game save/load |
| **Observer** | Notify multiple objects when one object changes state | Event listeners, pub/sub, stock price alerts |
| **State** | Object changes behavior when its internal state changes | Order status: `Pending` → `Shipped` → `Delivered` each behaves differently |
| **Strategy** | Swap algorithms at runtime; choose behavior based on context | Sorting strategy, payment method (card / UPI / wallet) |
| **Template Method** | Define skeleton of an algorithm; subclasses fill in specific steps | Data parser: `readFile()` → `parse()` → `validate()` — only `parse()` changes |
| **Visitor** | Add new operations to objects without modifying their classes | Tax calculation visitor that works across `Book`, `Electronics`, `Food` |
| **Interpreter** | Define a grammar and interpret sentences in it | SQL parser, expression evaluator, regex engine |

---

## Non-GOF Patterns

> Architectural and enterprise patterns that emerged after GOF. These solve distributed systems, data access, and application structure problems.

---

### Data & Persistence Patterns

| Pattern | When to Use | Real-World Example |
|---|---|---|
| **Repository** | Abstract data access behind an interface; decouple business logic from DB | `BookRepository` hides whether you use SQL, MongoDB, or in-memory map |
| **DAO (Data Access Object)** | Provide CRUD operations for a specific entity; older sibling of Repository | `BookDao` with `save()`, `findById()`, `delete()` |
| **DTO (Data Transfer Object)** | Transfer data between layers without exposing domain model internals | API returns `BookDTO` with only `title` and `author`, not the full entity |
| **Active Record** | Object knows how to save/load itself; model and persistence combined | Rails models: `book.save()`, `Book.find(id)` |
| **Unit of Work** | Track all changes in a transaction; flush them all at once | JPA `EntityManager` — tracks dirty objects and flushes in one commit |
| **Identity Map** | Cache loaded objects to avoid duplicate DB reads in the same request | JPA first-level cache: loading same entity twice returns the same object |

---

### Application Architecture Patterns

| Pattern | When to Use | Real-World Example |
|---|---|---|
| **MVC (Model-View-Controller)** | Separate UI, business logic, and data; each can change independently | Spring MVC: Controller handles request, Service processes, View renders |
| **Service Layer** | Centralize business logic and transaction boundaries above DAOs | `OrderService.placeOrder()` calls inventory, payment, notification |
| **CQRS** | Read and write workloads are very different; scale or optimize them separately | Write side: `PlaceOrderCommand`; Read side: `OrderSummaryQuery` with a flat read model |
| **Event Sourcing** | Store every state change as an event; replay events to rebuild state | Bank account: store `MoneyDeposited`, `MoneyWithdrawn` events instead of current balance |
| **Outbox Pattern** | Guarantee that a DB write and a message publish happen together atomically | Save order to DB and publish `OrderPlaced` event in the same transaction via an outbox table |
| **Strangler Fig** | Gradually replace a legacy system by routing traffic to new code piece by piece | Migrate a monolith to microservices route by route without a big-bang rewrite |

---

### Distributed Systems Patterns

| Pattern | When to Use | Real-World Example |
|---|---|---|
| **Saga** | Long-running transaction across multiple services; no distributed ACID | Borrow book: validate → reserve → update member → audit → notify; rollback each step on failure |
| **Circuit Breaker** | Stop calling a failing service; fail fast and recover gracefully | Payment service is down — open the circuit, return a cached response, retry after timeout |
| **Bulkhead** | Isolate failures in one part of the system so they don't cascade | Separate thread pools for inventory calls and payment calls — one slow doesn't block the other |
| **Retry** | Transient failures are common; automatically retry with backoff | Retry DB connection or HTTP call up to 3 times with exponential backoff |
| **API Gateway** | Single entry point for all client requests; handles routing, auth, rate limiting | Kong / Spring Cloud Gateway in front of all microservices |
| **Sidecar** | Attach helper functionality to a service without changing the service code | Envoy proxy alongside each microservice handles mTLS, logging, tracing |
| **Service Mesh** | Manage all service-to-service communication with observability and security | Istio managing retries, circuit breaking, and tracing across 50 microservices |
| **Backends for Frontends (BFF)** | Different clients (mobile, web) need different API shapes | `MobileOrderAPI` returns minimal data; `WebOrderAPI` returns full details |

---

### Concurrency Patterns

| Pattern | When to Use | Real-World Example |
|---|---|---|
| **Producer-Consumer** | Decouple work generation from work processing; handle different speeds | Order service (producer) puts orders on a queue; fulfillment service (consumer) processes them |
| **Thread Pool** | Reuse a fixed set of threads instead of creating new ones per task | Java `ExecutorService` — avoid thread creation overhead for every HTTP request |
| **Read-Write Lock** | Many readers are fine concurrently; writers need exclusive access | Cache: many threads can read, but only one can update at a time |
| **Scheduler** | Run tasks at specific times or intervals | Cron job to send daily report emails, cleanup expired sessions |

---

## Quick Cheat Sheet

| You want to… | Use this pattern |
|---|---|
| Create one shared instance | Singleton |
| Build complex objects step by step | Builder |
| Switch algorithms at runtime | Strategy |
| React to state changes | Observer |
| Add features without changing existing code | Decorator |
| Simplify a messy subsystem | Facade |
| Rollback a multi-step transaction | Saga |
| Stop hammering a failing service | Circuit Breaker |
| Separate reads from writes | CQRS |
| Abstract database access | Repository / DAO |
| Hide domain internals from the API layer | DTO |
| Gradually migrate a legacy system | Strangler Fig |
| Guarantee DB write + event publish atomically | Outbox Pattern |
