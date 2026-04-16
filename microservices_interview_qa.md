# Microservices Interview Questions & Answers

---

## Section 1: Core Concepts

---

**Q1. What is a microservice? How is it different from a monolith?**

**A:**
A microservice is a small, independently deployable service that owns a single business capability and its own data store.

| Aspect | Monolith | Microservices |
|---|---|---|
| Deployment | Deploy entire app | Deploy individual service |
| Scaling | Scale entire app | Scale only bottleneck service |
| Technology | Single tech stack | Each service chooses its own |
| Failure | One bug can crash all | Failure isolated to one service |
| Team | One large team | Small independent teams |
| DB | Single shared DB | DB per service |

**Real example:** In a monolith, a memory leak in the report module crashes the entire e-commerce app. In microservices, the ReportService crashes but OrderService keeps running.

---

**Q2. What are the key principles of microservices?**

**A:**
1. **Single Responsibility** — each service does one thing
2. **Loose Coupling** — services are independent; change in one doesn't force change in another
3. **High Cohesion** — related logic is grouped inside one service
4. **Own Data Store** — no shared database
5. **Design for Failure** — expect downstream services to fail; build resilience
6. **Decentralized Governance** — teams choose their own tech stack
7. **Automation** — CI/CD, automated testing, automated deployment

---

**Q3. What are the advantages and disadvantages of microservices?**

**A:**

**Advantages:**
- Independent deployment and scaling
- Technology flexibility per service
- Fault isolation
- Faster development (small, focused teams)
- Easier to understand small codebases

**Disadvantages:**
- Distributed system complexity (network latency, partial failures)
- Data consistency challenges (no ACID across services)
- Operational overhead (many services to monitor/deploy)
- Testing complexity (integration tests harder)
- Requires mature DevOps/CI-CD culture

---

**Q4. How do microservices communicate with each other?**

**A:** Two styles:

**Synchronous (request-response):**
- **REST** — HTTP/JSON, widely used, simple
- **gRPC** — HTTP/2, Protocol Buffers, faster, strongly typed; good for internal service communication

**Asynchronous (event-driven):**
- **Message brokers** — Kafka, RabbitMQ, SQS
- Service publishes event, others subscribe — no direct coupling

**When to use which:**
- Use sync when you need an immediate response (e.g. payment authorization)
- Use async when services can process independently (e.g. send email after order)

---

**Q5. What is an API Gateway and why is it needed?**

**A:**
An API Gateway is a single entry point for all clients. It sits in front of all microservices.

**Responsibilities:**
- **Routing** — route `/orders` to OrderService, `/payments` to PaymentService
- **Authentication** — validate JWT token once, not in every service
- **Rate limiting** — throttle abusive clients
- **SSL termination** — handle HTTPS at gateway level
- **Load balancing** — distribute traffic
- **Request aggregation** — combine multiple service calls into one response
- **Caching** — cache common responses

**Tools:** Kong, AWS API Gateway, Nginx, Spring Cloud Gateway

**Without API Gateway:** Every client must know every service's IP/port. Adding auth means changing every service.

---

## Section 2: Data Management

---

**Q6. What is the Database per Service pattern? Why is it important?**

**A:**
Each microservice has its own private database — no other service can directly query it.

**Why important:**
- Services are truly independent; changing one DB schema doesn't affect others
- Each service can choose the right DB type (SQL, NoSQL, Redis, etc.)
- Prevents tight coupling through shared DB

**The problem it creates:** How do you join data across services?
**Solutions:** API Composition, CQRS, Event Sourcing

---

**Q7. How do you handle distributed transactions in microservices?**

**A:**
You cannot use traditional ACID transactions (single DB commit) across services. Instead use the **Saga pattern**.

**Two types:**

**Choreography Saga:** Services react to each other's events.
```
OrderService publishes "order.created"
  → PaymentService handles it, publishes "payment.completed"
  → InventoryService handles it, publishes "inventory.reserved"
  → if any step fails → publish compensating event → previous steps roll back
```
Pro: No central coordinator. Con: Hard to track overall flow.

**Orchestration Saga:** Central coordinator tells each service what to do.
```
SagaOrchestrator → tells PaymentService to charge
                 → tells InventoryService to reserve
                 → if failure → tells each to compensate
```
Pro: Clear flow. Con: Orchestrator becomes a bottleneck.

**Compensating transactions** are the rollback mechanism — you cannot "undo" a DB write, you write a new record that reverses the effect.

---

**Q8. What is CQRS? When would you use it?**

**A:**
CQRS = Command Query Responsibility Segregation.

Split the model into two:
- **Command side** — handles writes (create, update, delete) → normalized DB
- **Query side** — handles reads → denormalized read-optimized store (Elasticsearch, Redis)

```
Write: POST /orders → OrderCommandService → PostgreSQL
                    → publishes OrderCreated event
                    → Read DB updated asynchronously

Read:  GET /orders/search?user=123 → OrderQueryService → Elasticsearch
```

**Use when:**
- Read workload vastly exceeds write workload
- Queries need complex aggregations that are slow on normalized DB
- Read and write need to scale independently

**Trade-off:** Eventual consistency between write DB and read DB.

---

**Q9. What is Event Sourcing?**

**A:**
Instead of storing current state, store the **sequence of events** that led to that state.

```
Traditional:  accounts table → { id: 1, balance: 120 }

Event Sourcing:
  events table:
  1. AccountCreated(id=1)
  2. MoneyDeposited(amount=100)
  3. MoneyWithdrawn(amount=30)
  4. MoneyDeposited(amount=50)

  Current balance = replay events → 100 - 30 + 50 = 120
```

**Benefits:**
- Complete audit trail (who did what, when)
- Rebuild state at any point in time
- Events are the source of truth for other services
- Works naturally with CQRS

**Challenges:**
- Querying current state requires replaying events (mitigated by snapshots)
- Event schema changes require versioning

---

## Section 3: Reliability & Resilience

---

**Q10. What is the Circuit Breaker pattern? Explain the states.**

**A:**
Prevents cascading failures by stopping calls to a failing service and returning a fallback.

**Three states:**

```
CLOSED → normal operation, all requests go through
  → if failure rate > threshold → trip to OPEN

OPEN → requests fail immediately (don't call downstream)
     → return fallback response
  → after timeout period → move to HALF-OPEN

HALF-OPEN → allow one request through
  → if success → back to CLOSED
  → if failure → back to OPEN
```

**Example with Resilience4j:**
```java
@CircuitBreaker(name = "paymentService", fallbackMethod = "fallbackPayment")
public PaymentResponse charge(Order order) {
    return paymentClient.charge(order);
}

public PaymentResponse fallbackPayment(Order order, Exception e) {
    return PaymentResponse.pending(); // queue for retry later
}
```

**Use when:** Calling external services that can be slow or unreliable.

---

**Q11. What is the difference between Circuit Breaker, Retry, and Timeout? When do you use each?**

**A:**

| Pattern | What it does | Use when |
|---|---|---|
| **Timeout** | Fail fast after N seconds | Always — every remote call |
| **Retry** | Try again N times with backoff | Transient failures (network blip) |
| **Circuit Breaker** | Stop calling after repeated failures | Service is consistently failing |

**Combined usage:**
```
Request → Timeout(3s) → Retry(3 times, exponential backoff) → Circuit Breaker
```

**Important:** Retry only works for idempotent operations. Retrying a payment that already succeeded can charge twice — use idempotency keys.

---

**Q12. What is the Bulkhead pattern?**

**A:**
Isolate resources (thread pools, connection pools) per downstream dependency so one slow service doesn't starve all others.

```
Without Bulkhead:
  OrderService has 100 threads total
  PaymentService gets slow → all 100 threads stuck waiting
  → ProductService calls also fail (no threads available)

With Bulkhead:
  PaymentService thread pool: 30 threads
  ProductService thread pool: 20 threads
  InventoryService thread pool: 20 threads
  PaymentService slow → only 30 threads affected
  ProductService still has its 20 threads → still working
```

Named after ship bulkheads — compartments that prevent one hole from sinking the whole ship.

---

## Section 4: Service Discovery & Load Balancing

---

**Q13. What is Service Discovery? Client-side vs Server-side?**

**A:**
Service Discovery is the mechanism for services to find each other's network location dynamically (IP/port changes as services scale up/down).

**Client-side discovery:**
```
ServiceA → queries ServiceRegistry (Eureka) → gets list of ServiceB instances
         → client-side load balance → picks one → calls it directly
```
Client knows about all instances. Used in: Spring Cloud with Eureka + Ribbon.

**Server-side discovery:**
```
ServiceA → calls Load Balancer (AWS ALB / Nginx)
         → Load Balancer queries registry → picks instance → forwards request
```
Client doesn't know about discovery mechanism. Used in: K8s Services, AWS ELB.

---

## Section 5: Observability

---

**Q14. What is Distributed Tracing? How does it work?**

**A:**
Track a single request as it flows through multiple services.

Every request gets a **Trace ID** (unique ID for the whole request).
Each service creates a **Span** (unit of work within a trace) with start/end time.

```
TraceId: abc-123
  OrderService     span-1 (0ms–50ms)
  ├── PaymentService   span-2 (10ms–40ms)
  └── InventoryService span-3 (15ms–45ms)

→ Visualization shows which service is slow
```

**Tools:** Jaeger, Zipkin, AWS X-Ray, OpenTelemetry

**Without distributed tracing:** A 500ms latency issue could be in any of 10 services — you'd have to check logs in all of them manually.

---

**Q15. What are the three pillars of observability?**

**A:**

| Pillar | What | Tools |
|---|---|---|
| **Logs** | Timestamped events (what happened) | ELK, Splunk, GCP Logging |
| **Metrics** | Numeric measurements over time (error rate, latency, CPU) | Prometheus, Grafana |
| **Traces** | End-to-end request journey across services | Jaeger, Zipkin, OpenTelemetry |

**Logs** tell you what happened.
**Metrics** tell you something is wrong.
**Traces** tell you where it's wrong.

---

## Section 6: Security

---

**Q16. How do you secure communication between microservices?**

**A:**

**Authentication (who are you?):**
- Use **JWT tokens** — client authenticates once with AuthService, gets JWT, passes it in headers to all services
- Services validate JWT (signature, expiry, claims) without calling AuthService each time

**Authorization (what can you do?):**
- JWT contains roles/permissions — each service enforces its own rules
- Or use a centralized policy engine (OPA — Open Policy Agent)

**Service-to-service security:**
- **mTLS** — both services present certificates; encrypted + mutual authentication
- **Athenz/SPIFFE** — identity framework for service certificates

**In K8s:** Service Mesh (Istio) handles mTLS automatically — no code changes needed.

---

## Section 7: Deployment

---

**Q17. What is Blue-Green deployment vs Canary deployment?**

**A:**

**Blue-Green:**
```
Blue (v1) ← currently live (100% traffic)
Green (v2) ← new version deployed, tested

→ Switch: route 100% traffic to Green
→ Keep Blue for instant rollback
```
Pro: Instant cutover, instant rollback.
Con: Requires double infrastructure. No gradual validation.

**Canary:**
```
v1 ← 95% traffic
v2 ← 5% traffic (canary)
→ monitor errors/latency on v2
→ gradually increase: 5% → 25% → 50% → 100%
→ or rollback to 0% if issues
```
Pro: Gradual risk — real traffic validates new version.
Con: Slower rollout; both versions run in parallel for longer.

**When to use:**
- Blue-Green: Want instant rollback; batch/scheduled jobs
- Canary: Want gradual validation; high-traffic production systems

---

**Q18. What is the Sidecar pattern?**

**A:**
Deploy a helper container alongside the main application container in the same pod (K8s). The sidecar handles cross-cutting concerns so the app doesn't have to.

```
Pod:
├── Main container      → business logic (OrderService)
└── Sidecar container   → logging agent / Envoy proxy / config watcher
```

**Examples:**
- **Envoy proxy** (Istio) — handles mTLS, retries, circuit breaking
- **Log shipper** (Fluentd) — collects and forwards logs
- **Config watcher** — reloads config when ConfigMap changes

**Benefit:** App code stays clean — no logging/metrics/auth SDK needed in business code.

---

## Section 8: Advanced / Senior Level

---

**Q19. How do you handle eventual consistency in microservices? How do you explain it to stakeholders?**

**A:**
In a distributed system, you cannot have strong consistency without sacrificing availability (CAP theorem). Eventual consistency means data will be consistent — just not immediately.

**Technical handling:**
- Use async events — OrderService publishes `order.created`, InventoryService updates asynchronously
- Use idempotent consumers — if event delivered twice, result is same
- Use saga pattern for multi-step workflows
- Implement compensating transactions for rollbacks

**To stakeholders:**
"When a customer places an order, the order confirmation is instant. The inventory update and email notification happen within seconds. There's a brief window where the system is catching up — but the final state will always be correct. This is the trade-off that allows our system to handle millions of orders without slowing down."

---

**Q20. How do you prevent data inconsistency in the Saga pattern if one step fails midway?**

**A:**
Each step in a Saga must have a corresponding **compensating transaction** — a reverse action that undoes the effect.

```
Forward steps:         Compensating steps:
1. Reserve Inventory   → Cancel Reservation
2. Charge Payment      → Refund Payment
3. Create Shipment     → Cancel Shipment
```

**If step 3 (Create Shipment) fails:**
- Execute: Cancel Shipment (no-op, already failed)
- Execute: Refund Payment
- Execute: Cancel Reservation
→ system is back to consistent state

**Challenges:**
- Compensating transactions can also fail — need retry with idempotency
- Time between forward and compensating steps = window of inconsistency
- Solution: Design compensating transactions to be idempotent and retriable

---

**Q21. What is the difference between Orchestration and Choreography in Sagas?**

**A:**

| | Choreography | Orchestration |
|---|---|---|
| **How** | Services react to events | Central coordinator commands services |
| **Coupling** | Loose — services don't know each other | Tighter — orchestrator knows all |
| **Visibility** | Hard to see overall flow | Easy — orchestrator has full picture |
| **Failure handling** | Each service handles its own | Orchestrator handles all |
| **Complexity** | Simple per-service logic | Orchestrator can become complex |
| **Use when** | Simple flows, many services | Complex workflows, business process |

**Real world:** Payment processing is better as orchestration (complex, many steps, needs visibility). Order notification (email, SMS, push) is better as choreography (fire and forget, loose coupling).

---

**Q22. How do you test microservices?**

**A:**

**Testing pyramid for microservices:**

```
            /\
           /  \  E2E Tests (few)
          /----\
         /      \  Integration Tests
        /--------\
       /          \  Contract Tests (Pact)
      /------------\
     /              \  Unit Tests (many)
    /----------------\
```

1. **Unit tests** — test business logic in isolation, mock dependencies
2. **Contract tests (CDC)** — verify API contracts between services (Pact framework)
   - Consumer defines expected API behavior
   - Provider verifies it meets those expectations
   - Catches breaking API changes before deployment
3. **Integration tests** — test service with real DB, real message broker
4. **End-to-end tests** — test full flow through multiple services (slow, brittle, use sparingly)

**Key insight:** Contract tests replace most E2E tests — they verify the contract without spinning up all services.

---

**Q23. How would you migrate a monolith to microservices?**

**A:**
Use the **Strangler Fig pattern** — gradually strangle the monolith.

**Steps:**
1. **Identify boundaries** — find business capabilities with clear separation (DDD bounded contexts)
2. **Set up a facade** — put an API Gateway/proxy in front of the monolith
3. **Extract one service** — pick the simplest, most independent capability
4. **Route traffic** — proxy routes that feature to new service, rest goes to monolith
5. **Repeat** — extract next capability, monolith shrinks
6. **Data migration** — gradually migrate data ownership to new service DBs
7. **Decommission** — when monolith is empty, shut it down

**What NOT to do:**
- Don't do a big-bang rewrite — too risky, takes too long
- Don't split too early — understand the domain first (start with 3-5 services, not 50)
- Don't share DBs between new services and monolith

---

**Q24. What are the most common mistakes in microservices architecture?**

**A:**

1. **Too fine-grained too soon** — splitting a 1000-line class into 10 services before understanding the domain. Start coarse, split when you feel the pain.

2. **Shared database** — two services writing to same DB. Looks convenient, becomes a nightmare. Change DB schema → break both services.

3. **Synchronous chains** — ServiceA → ServiceB → ServiceC → ServiceD. One slow service makes everything slow. Use async where possible.

4. **No distributed tracing** — debugging a bug across 10 services without trace IDs is near impossible.

5. **Ignoring operational complexity** — microservices require CI/CD, container orchestration, centralized logging, distributed tracing, service discovery. Without DevOps maturity, microservices make things worse.

6. **Distributed monolith** — services that are physically separate but logically coupled (must deploy together, share code, chat over sync calls). You get the worst of both worlds.

7. **No idempotency** — retrying a non-idempotent operation (like charging a card) twice = double charge.

---

**Q25. CAP Theorem — how does it apply to microservices?**

**A:**
CAP Theorem states: a distributed system can guarantee at most **2 of 3**:
- **C**onsistency — every read gets the latest write
- **A**vailability — every request gets a response (not necessarily latest data)
- **P**artition Tolerance — system works even if network splits

**In microservices, P (Partition Tolerance) is non-negotiable** — network failures WILL happen.

So the real choice is **CP vs AP:**

| | CP (Consistent + Partition tolerant) | AP (Available + Partition tolerant) |
|---|---|---|
| **Behavior on failure** | Return error rather than stale data | Return stale data rather than error |
| **Use for** | Financial transactions, inventory counts | User profiles, product catalog, recommendations |
| **Example** | Bank account balance | Shopping cart, social media feed |

**In practice:** Most microservices choose AP with eventual consistency — users get fast responses with slightly stale data, which is acceptable for most use cases.
