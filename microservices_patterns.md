# Microservices Patterns

---

## 1. Decomposition Patterns

### 1.1 Decompose by Business Capability
Split services based on what the business does — each service owns one business function.

**Example:**
```
E-commerce app:
├── OrderService       → handles order lifecycle
├── PaymentService     → handles payments
├── InventoryService   → manages stock
└── NotificationService→ sends emails/SMS
```
**Use when:** Aligning teams to business domains (Conway's Law).

---

### 1.2 Decompose by Subdomain (DDD)
Use Domain-Driven Design bounded contexts to define service boundaries.

**Example:**
```
Banking:
├── AccountService     → core banking (core subdomain)
├── LoanService        → loan processing (core subdomain)
├── ReportingService   → statements (supporting subdomain)
└── AuthService        → login/auth (generic subdomain)
```
**Use when:** Complex domains where business logic drives the design.

---

### 1.3 Strangler Fig Pattern
Gradually migrate a monolith to microservices by routing traffic feature by feature.

**Flow:**
```
Client → Facade/Proxy
              ├── /orders   → New OrderService (microservice)
              ├── /payments → New PaymentService (microservice)
              └── /legacy   → Old Monolith (shrinking over time)
```
**Use when:** Migrating legacy monoliths without a big-bang rewrite.

---

## 2. Communication Patterns

### 2.1 API Gateway
Single entry point for all clients. Handles routing, auth, rate limiting, SSL termination.

```
Mobile App  ──┐
Web App     ──┤──→ API Gateway ──→ UserService
3rd Party   ──┘                ──→ OrderService
                               ──→ ProductService
```
**Use when:** Multiple clients need different data shapes; cross-cutting concerns (auth, logging).

---

### 2.2 Backend for Frontend (BFF)
Separate API Gateway per client type — each BFF is optimized for its client's needs.

```
Mobile App  → Mobile BFF  → (lightweight responses, push notifications)
Web App     → Web BFF     → (full data, server-side rendering)
3rd Party   → Partner BFF → (rate-limited, versioned API)
```
**Use when:** Mobile and web need very different data; avoid over-fetching/under-fetching.

---

### 2.3 Synchronous Communication (REST / gRPC)
Direct request-response between services.

```
OrderService ──REST──→ InventoryService
             ←──────── { "available": true }
```
**Use when:** Immediate response needed; simple query/command flow.

---

### 2.4 Asynchronous Messaging (Event-Driven)
Services communicate via message broker; sender doesn't wait for response.

```
OrderService ──→ [Kafka/RabbitMQ] ──→ PaymentService
                                   ──→ NotificationService
                                   ──→ InventoryService
```
**Use when:** Loose coupling needed; high throughput; services should not be blocked.

---

### 2.5 Service Mesh (Sidecar Pattern)
Infrastructure layer handles service-to-service communication (retries, mTLS, circuit breaking) via sidecar proxy (Istio/Envoy) — no code changes needed.

```
[OrderService] ←→ [Envoy Sidecar] ←→ [Envoy Sidecar] ←→ [PaymentService]
```
**Use when:** Large number of services; need observability and security without polluting business code.

---

## 3. Data Management Patterns

### 3.1 Database per Service
Each service owns its own database — no shared database between services.

```
OrderService    → orders_db (PostgreSQL)
ProductService  → products_db (MongoDB)
CartService     → cart_db (Redis)
```
**Use when:** Services must be independently deployable and scalable; different data models needed.

---

### 3.2 Saga Pattern
Manage distributed transactions across services using a sequence of local transactions.

**Choreography Saga (event-driven):**
```
OrderService → [order.created] → PaymentService → [payment.done] → InventoryService
                                                 → [payment.failed] → OrderService (compensate)
```

**Orchestration Saga (central coordinator):**
```
SagaOrchestrator → PaymentService  → success
                 → InventoryService → success
                 → ShippingService  → fail → compensate PaymentService, InventoryService
```
**Use when:** Multi-step business transactions spanning multiple services (e.g. order placement).

---

### 3.3 CQRS (Command Query Responsibility Segregation)
Separate read model and write model into different services/stores.

```
Write Side:  Client → CommandAPI → Write DB (normalized, PostgreSQL)
                                 → publishes events

Read Side:   Client → QueryAPI  → Read DB (denormalized, Elasticsearch/Redis)
                                 ← synced from events
```
**Use when:** Read and write workloads have very different scaling/performance requirements.

---

### 3.4 Event Sourcing
Store state as a sequence of events rather than current state snapshot.

```
Account events:
  [AccountCreated, MoneyDeposited(100), MoneyWithdrawn(30), MoneyDeposited(50)]
  → Current balance = 120
```
**Use when:** Full audit trail needed; need to rebuild state at any point in time; works naturally with CQRS.

---

### 3.5 API Composition
Aggregate data from multiple services in one query layer (no distributed join in DB).

```
Client → ProductCompositeService
              ├── calls ProductService  → product details
              ├── calls ReviewService   → reviews
              └── calls PriceService    → pricing
              → combines and returns single response
```
**Use when:** Client needs data from multiple services in one call; avoids N+1 client round trips.

---

## 4. Reliability Patterns

### 4.1 Circuit Breaker
Stop calling a failing service; return fallback response. Resets after a timeout.

```
States: CLOSED → OPEN → HALF-OPEN → CLOSED

CLOSED:     requests pass through normally
OPEN:       requests fail fast (don't call downstream), return fallback
HALF-OPEN:  let one request through to test if service recovered
```
**Tools:** Resilience4j, Hystrix
**Use when:** Prevent cascading failures; protect upstream services from slow/failed downstream.

---

### 4.2 Retry Pattern
Automatically retry failed requests with backoff.

```java
// Exponential backoff:
Retry 1: wait 1s
Retry 2: wait 2s
Retry 3: wait 4s
→ fail with error
```
**Use when:** Transient network failures; idempotent operations only (GET, PUT, not POST).

---

### 4.3 Bulkhead Pattern
Isolate failures by partitioning resources — one service failure doesn't starve others.

```
Thread pool per downstream service:
OrderService thread pool   → 20 threads (for calling PaymentService)
ProductService thread pool → 10 threads (for calling InventoryService)
→ PaymentService slow? Only PaymentService threads blocked, rest unaffected
```
**Use when:** Prevent one slow dependency from consuming all threads.

---

### 4.4 Timeout Pattern
Set maximum wait time on any remote call.

```
OrderService → PaymentService (timeout: 3s)
             → if no response in 3s → fail fast, return error
```
**Use when:** Always — every remote call should have a timeout.

---

### 4.5 Health Check / Readiness & Liveness Probes
Services expose health endpoints; orchestrator (K8s) monitors and restarts unhealthy pods.

```
GET /health/live  → 200 OK  (is the process alive?)
GET /health/ready → 200 OK  (is it ready to serve traffic?)
```
**Use when:** Always in containerized/K8s environments.

---

## 5. Observability Patterns

### 5.1 Distributed Tracing
Track a request as it flows through multiple services using a correlation/trace ID.

```
Request → TraceId: abc123
  OrderService    (spanId: 1)
  PaymentService  (spanId: 2, parentSpanId: 1)
  InventoryService(spanId: 3, parentSpanId: 1)
```
**Tools:** Jaeger, Zipkin, OpenTelemetry
**Use when:** Debugging latency issues; understanding service dependency chains.

---

### 5.2 Log Aggregation
Centralize logs from all services into a single searchable store.

```
Service A logs ──┐
Service B logs ──┤──→ Fluentd/Logstash → Elasticsearch → Kibana dashboard
Service C logs ──┘
```
**Tools:** ELK Stack, Splunk, GCP Cloud Logging
**Use when:** Always — you cannot SSH into 100 pods to read logs.

---

### 5.3 Metrics Aggregation
Collect and visualize service metrics (latency, error rate, throughput).

```
Services → Prometheus (scrape) → Grafana (visualize)
         → Alerts on SLO breach
```
**Use when:** SLA monitoring; capacity planning; alerting.

---

## 6. Security Patterns

### 6.1 Access Token (JWT / OAuth2)
Services validate identity via tokens — no shared session state.

```
Client → AuthService → JWT token
Client → OrderService (Authorization: Bearer <token>)
       → OrderService validates token → processes request
```
**Use when:** Stateless authentication across services.

---

### 6.2 Mutual TLS (mTLS)
Both client and server authenticate each other with certificates — prevents impersonation.

```
ServiceA cert ←→ ServiceB cert (both verified)
→ encrypted + authenticated channel
```
**Use when:** Zero-trust security model; service-to-service auth inside cluster.

---

## 7. Deployment Patterns

### 7.1 Sidecar
Deploy helper container alongside main container in same pod — handles cross-cutting concerns.

```
Pod:
├── Main container    (app code)
└── Sidecar container (logging / service mesh proxy / config watcher)
```
**Use when:** Adding capabilities (logging, metrics, auth) without modifying app code.

---

### 7.2 Blue-Green Deployment
Run two identical environments (blue=current, green=new). Switch traffic after validation.

```
Load Balancer → Blue (v1) ← currently live
             → Green (v2) ← new version, tested
→ switch: Load Balancer → Green (v2)
→ Blue kept as rollback
```
**Use when:** Zero-downtime deployments; instant rollback needed.

---

### 7.3 Canary Deployment
Gradually roll out new version to a small % of users, then increase.

```
Load Balancer → v1 (95% traffic) ← stable
             → v2 (5% traffic)  ← canary (new)
→ monitor errors/latency
→ if healthy: increase to 50% → 100%
→ if issues: rollback canary to 0%
```
**Use when:** Risk mitigation; real-world testing before full rollout.

---

### 7.4 Service Registry & Discovery
Services register themselves; others discover them dynamically — no hardcoded IPs.

```
Service starts → registers in Eureka/Consul
Other service  → queries registry → gets IP:port → calls service
```
**Tools:** Eureka, Consul, K8s DNS
**Use when:** Dynamic scaling where IPs change frequently.

---

## 8. Configuration Patterns

### 8.1 Externalized Configuration
All config (DB URLs, feature flags, timeouts) stored outside the service binary.

```
Service → reads config from:
          ├── ConfigMap (K8s)
          ├── Vault (secrets)
          └── Config Server (Spring Cloud Config)
```
**Use when:** Same image deployed to dev/staging/prod with different config.

---

### 8.2 Feature Toggles
Enable/disable features at runtime without redeployment.

```java
if (featureFlags.isEnabled("new-payment-flow")) {
    return newPaymentService.process(order);
} else {
    return legacyPaymentService.process(order);
}
```
**Use when:** Dark launches; A/B testing; gradual rollouts; kill switches.

---

## Pattern Summary Table

| Category | Pattern | Key Benefit |
|---|---|---|
| Decomposition | Business Capability | Team alignment |
| Decomposition | Strangler Fig | Safe monolith migration |
| Communication | API Gateway | Single entry point |
| Communication | BFF | Client-specific APIs |
| Communication | Saga | Distributed transactions |
| Data | CQRS | Read/write scaling |
| Data | Event Sourcing | Full audit trail |
| Reliability | Circuit Breaker | Prevent cascading failures |
| Reliability | Bulkhead | Resource isolation |
| Observability | Distributed Tracing | End-to-end visibility |
| Security | mTLS | Zero-trust auth |
| Deployment | Canary | Low-risk rollouts |
| Deployment | Blue-Green | Zero-downtime + instant rollback |
