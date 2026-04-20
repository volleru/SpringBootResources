# Java Microservices Interview Questions & Answers

---

## 1. CORE MICROSERVICES CONCEPTS

### Q1. What are Microservices?
Microservices is an architectural style where an application is built as a collection of small, independently deployable services, each running in its own process and communicating via APIs (REST/gRPC/messaging).

---

### Q2. Monolithic vs Microservices?

| | Monolithic | Microservices |
|---|---|---|
| Deployment | Single unit | Independent per service |
| Scaling | Scale entire app | Scale individual service |
| Tech stack | One stack | Polyglot |
| Failure | One failure = full down | Isolated failures |
| Team | One big team | Small independent teams |

---

### Q3. What are the key principles of Microservices?
- Single Responsibility
- Loose Coupling
- High Cohesion
- Independent Deployability
- Decentralized Data Management
- Failure Isolation

---

## 2. SPRING BOOT & SPRING CLOUD

### Q4. What is Spring Boot and why is it used in Microservices?
Spring Boot provides auto-configuration, embedded servers (Tomcat/Jetty), and production-ready features (actuator, metrics) — making it ideal to quickly build standalone microservices without XML config.

---

### Q5. What is Spring Cloud?
Spring Cloud provides tools for building distributed systems:
- **Eureka** — Service Discovery
- **API Gateway** — Routing
- **Config Server** — Centralized config
- **Ribbon/LoadBalancer** — Client-side LB
- **Hystrix/Resilience4j** — Circuit Breaker
- **Sleuth/Zipkin** — Distributed Tracing

---

### Q6. How does Service Discovery work?
```
Microservice A starts
    ↓
Registers itself with Eureka Server (host, port, service name)
    ↓
Microservice B wants to call A
    ↓
Asks Eureka: "Where is Service A?"
    ↓
Eureka returns IP/port → B calls A directly
```

---

### Q7. What is API Gateway? Why use it?
Single entry point for all clients. Handles:
- Routing requests to correct service
- Authentication & Authorization
- Rate limiting
- Load balancing
- SSL termination
- Request/Response transformation

```java
// Spring Cloud Gateway route example
@Bean
public RouteLocator routes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("order-service", r -> r.path("/orders/**")
            .uri("lb://ORDER-SERVICE"))
        .route("user-service", r -> r.path("/users/**")
            .uri("lb://USER-SERVICE"))
        .build();
}
```

---

## 3. COMMUNICATION

### Q8. Synchronous vs Asynchronous communication?

| | Synchronous | Asynchronous |
|---|---|---|
| Protocol | REST, gRPC | Kafka, RabbitMQ |
| Coupling | Tight | Loose |
| Availability | Both must be up | Producer/consumer independent |
| Use case | Real-time queries | Events, notifications |

---

### Q9. What is Feign Client?
Declarative REST client in Spring Cloud — no need to write RestTemplate boilerplate.

```java
@FeignClient(name = "order-service")
public interface OrderClient {
    @GetMapping("/orders/{id}")
    Order getOrder(@PathVariable Long id);
}
```

---

### Q10. REST vs gRPC?

| | REST | gRPC |
|---|---|---|
| Protocol | HTTP/1.1 | HTTP/2 |
| Format | JSON | Protocol Buffers (binary) |
| Speed | Slower | 5-10x faster |
| Contract | Optional (OpenAPI) | Strict (.proto file) |
| Browser support | Yes | Limited |

---

## 4. RESILIENCE PATTERNS

### Q11. What is Circuit Breaker? How does it work?
Prevents cascading failures when a service is down.

```
CLOSED state → requests flow normally
    ↓ (failures exceed threshold)
OPEN state → requests fail fast (no actual call made)
    ↓ (after timeout)
HALF-OPEN state → allow few test requests
    ↓ (if success)
CLOSED again
```

```java
@CircuitBreaker(name = "orderService", fallbackMethod = "fallback")
public Order getOrder(Long id) {
    return orderClient.getOrder(id);
}

public Order fallback(Long id, Exception e) {
    return new Order("default"); // fallback response
}
```

---

### Q12. What is Bulkhead Pattern?
Isolates failures by limiting concurrent calls to a service — like compartments in a ship. If one service is slow, it doesn't exhaust all threads.

```java
@Bulkhead(name = "orderService", type = Bulkhead.Type.THREADPOOL)
public CompletableFuture<Order> getOrder(Long id) {
    return CompletableFuture.supplyAsync(() -> orderClient.getOrder(id));
}
```

---

### Q13. What is Retry Pattern?
```java
@Retry(name = "orderService", fallbackMethod = "fallback")
public Order getOrder(Long id) {
    return orderClient.getOrder(id);
}
// Retries 3 times before calling fallback
```

---

## 5. DATA MANAGEMENT

### Q14. How do Microservices handle databases?
Each microservice owns its own database — **Database per Service** pattern.
- User Service → User DB
- Order Service → Order DB
- Payment Service → Payment DB

> Never share a database between services.

---

### Q15. What is SAGA Pattern?
Manages distributed transactions across multiple services without 2-phase commit.

- **Choreography** — each service publishes events, others react
- **Orchestration** — central orchestrator tells each service what to do

```
Order Service → publishes OrderCreated event
    ↓
Payment Service → listens, processes payment → publishes PaymentDone
    ↓
Inventory Service → listens, reserves stock → publishes StockReserved
    ↓
Order confirmed
```

---

### Q16. What is CQRS?
Command Query Responsibility Segregation — separate models for read and write operations.
- **Command** → writes/updates (goes to write DB)
- **Query** → reads (goes to optimized read DB/cache)

---

## 6. SECURITY

### Q17. How do you secure Microservices?
- **JWT tokens** — stateless authentication
- **OAuth2 + OpenID Connect** — authorization
- **API Gateway** — centralize auth validation
- **mTLS** — service-to-service encryption
- **Secret Manager** — store credentials (not in code)

---

### Q18. How does JWT work in Microservices?
```
User logs in → Auth Service issues JWT token
    ↓
User sends JWT in every request header
    ↓
API Gateway validates JWT signature
    ↓
Request forwarded to microservice (no DB call needed)
```

```java
// Validate JWT in filter
@Override
protected void doFilterInternal(HttpServletRequest request, ...) {
    String token = request.getHeader("Authorization").substring(7);
    if (jwtUtil.validateToken(token)) {
        // set authentication in SecurityContext
    }
}
```

---

## 7. OBSERVABILITY

### Q19. What is Distributed Tracing?
Tracking a request across multiple microservices using a unique **Trace ID**.

Tools: **Zipkin, Jaeger, Google Cloud Trace**

```
Request → Service A (traceId: abc123, spanId: 1)
              ↓
          Service B (traceId: abc123, spanId: 2)
              ↓
          Service C (traceId: abc123, spanId: 3)
```

---

### Q20. What are the 3 pillars of Observability?

| Pillar | Tool | Purpose |
|---|---|---|
| **Logs** | ELK, Cloud Logging | What happened |
| **Metrics** | Prometheus, Grafana | How system performs |
| **Traces** | Zipkin, Jaeger | Where time was spent |

---

## 8. DEPLOYMENT

### Q21. What is a Docker container and why use it in Microservices?
Packages service + dependencies into an isolated unit. Ensures "works on my machine" = "works everywhere".

```dockerfile
FROM openjdk:17-slim
COPY target/order-service.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

### Q22. What is Kubernetes and how does it help?
Orchestrates containers — handles:
- Auto-scaling (HPA)
- Self-healing (restarts crashed pods)
- Rolling deployments (zero downtime)
- Service discovery & load balancing
- Config/Secret management

---

### Q23. What is a Sidecar pattern?
Deploy a helper container alongside your main service container in the same pod.
- Main container → your service
- Sidecar → logging agent, proxy (Envoy), config watcher

---

## 9. ADVANCED / SCENARIO BASED

### Q24. How do you handle versioning in Microservices APIs?
```
/api/v1/orders  → old clients
/api/v2/orders  → new clients (with breaking changes)
```

---

### Q25. What happens when one microservice is down?
Should use:
1. **Circuit Breaker** — stop calling failing service
2. **Fallback** — return cached/default response
3. **Retry with backoff** — retry after delay
4. **Timeout** — don't wait forever
5. **Bulkhead** — isolate thread pool

---

### Q26. How do you do zero-downtime deployment?
- **Rolling update** in Kubernetes
- **Blue-Green deployment** — run old & new side by side, switch traffic
- **Canary deployment** — route 5% traffic to new version, then gradually increase

---

### Q27. Difference between Orchestration vs Choreography in SAGA?

| | Orchestration | Choreography |
|---|---|---|
| Control | Central orchestrator | Decentralized events |
| Coupling | Medium | Loose |
| Debugging | Easier | Harder |
| Example | Camunda, AWS Step Functions | Kafka events |

---

### Q28. What is Event Sourcing?
Instead of storing current state, store a sequence of events that led to the state.

```
OrderCreated → PaymentProcessed → OrderShipped → OrderDelivered
```
- Full audit trail
- Can replay events to rebuild state
- Works well with CQRS

---

### Q29. How do you handle inter-service failures gracefully?
- **Timeouts** — set max wait time per call
- **Fallback responses** — serve cached/default data
- **Dead Letter Queue (DLQ)** — failed Kafka messages go to DLQ for retry
- **Idempotency keys** — ensure duplicate messages don't cause duplicate actions

---

### Q30. What is Service Mesh?
Infrastructure layer (Istio / Linkerd) that handles service-to-service communication:
- mTLS between services
- Traffic management & retries
- Observability (metrics, traces)
- No code changes needed — sidecar proxy handles it

---

## 10. QUICK FIRE ROUND

| Question | Answer |
|---|---|
| Default port Spring Boot? | 8080 |
| Eureka default port? | 8761 |
| What is Actuator? | Monitoring endpoints (/health, /metrics) |
| What is Config Server? | Centralized external config for all services |
| What is Service Mesh? | Infrastructure layer (Istio/Linkerd) for service-to-service comms |
| What is Idempotency? | Same request called multiple times = same result |
| CAP Theorem? | Can only guarantee 2 of: Consistency, Availability, Partition tolerance |
| What is Rate Limiting? | Restrict number of API calls per user/time window |
| What is Saga? | Distributed transaction pattern across microservices |
| What is Strangler Fig? | Gradually replace monolith with microservices |
| What is Back Pressure? | Consumer signals producer to slow down (Kafka, reactive streams) |
| What is Health Check? | /actuator/health endpoint to verify service is up |
| What is Load Balancing? | Distribute requests across multiple instances |
| What is mTLS? | Mutual TLS — both client and server authenticate each other |
| What is DLQ? | Dead Letter Queue — holds failed messages for retry |

---

## 11. KAFKA IN MICROSERVICES

### Q31. Why Kafka over REST for async communication?
- **Decoupling** — producer and consumer don't need to be online simultaneously
- **Durability** — messages persisted on disk
- **Replay** — can reprocess old events
- **High throughput** — millions of messages/sec
- **Ordering** — guaranteed within a partition

---

### Q32. Producer and Consumer in Spring Boot

```java
// Producer
@Service
public class OrderProducer {
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    public void sendOrder(String order) {
        kafkaTemplate.send("order-topic", order);
    }
}

// Consumer
@Service
public class PaymentConsumer {
    @KafkaListener(topics = "order-topic", groupId = "payment-group")
    public void processOrder(String order) {
        // process payment
    }
}
```

---

## 12. JAVA SPECIFIC

### Q33. What Java features are commonly used in Microservices?
- **CompletableFuture** — async non-blocking calls
- **Optional** — null safety
- **Stream API** — data processing
- **Records** (Java 16+) — immutable DTOs
- **Functional interfaces** — clean code with lambdas

---

### Q34. What is Reactive Programming in Microservices?
Non-blocking, async programming model using Project Reactor (WebFlux).

```java
// Blocking (traditional)
Order order = orderService.getOrder(id); // blocks thread

// Non-blocking (reactive)
Mono<Order> order = orderService.getOrder(id); // frees thread immediately
order.subscribe(o -> process(o));
```

Use WebFlux when: high concurrency, I/O-heavy services, streaming data.

---

### Q35. How do you write unit tests for Microservices?

```java
@SpringBootTest
@AutoConfigureMockMvc
class OrderControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private OrderService orderService;

    @Test
    void getOrder_returnsOrder() throws Exception {
        when(orderService.getOrder(1L)).thenReturn(new Order(1L, "PLACED"));

        mockMvc.perform(get("/orders/1"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.status").value("PLACED"));
    }
}
```

---

## 13. GCP + MICROSERVICES

### Q36. What GCP services are commonly used in Microservices architecture?

| GCP Service | Purpose |
|---|---|
| **GKE** | Run containerized microservices |
| **Cloud Run** | Serverless container execution |
| **Pub/Sub** | Async messaging between services |
| **Cloud SQL** | Managed relational DB per service |
| **Firestore** | NoSQL DB for microservices |
| **Cloud Spanner** | Globally distributed SQL DB |
| **API Gateway** | Manage and secure APIs |
| **Cloud Endpoints** | API management & monitoring |
| **Secret Manager** | Store credentials securely |
| **Cloud Build** | CI/CD pipelines |
| **Artifact Registry** | Store Docker images |
| **Cloud Trace** | Distributed tracing |
| **Cloud Monitoring** | Metrics & alerting |
| **Cloud Logging** | Centralized log management |

---

### Q37. What is GKE and how does it help Microservices?
Google Kubernetes Engine (GKE) is a managed Kubernetes service on GCP.

Benefits:
- Auto-scaling (HPA & VPA)
- Auto-repair and auto-upgrade of nodes
- Integrated with Cloud Logging, Cloud Monitoring, Cloud Trace
- Workload Identity for secure GCP service access
- Multi-zone and regional clusters for high availability

```yaml
# GKE Deployment example
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    spec:
      containers:
      - name: order-service
        image: gcr.io/my-project/order-service:v1
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

---

### Q38. What is Cloud Run and when to use it over GKE?

| | Cloud Run | GKE |
|---|---|---|
| Management | Fully serverless | You manage cluster |
| Scaling | Scale to zero | Min 1 pod always |
| Cost | Pay per request | Pay for cluster nodes |
| Use case | Stateless, event-driven | Long-running, stateful |
| Cold start | Yes | No |

Use **Cloud Run** for lightweight, event-driven microservices.
Use **GKE** for complex, stateful, high-traffic microservices.

---

### Q39. How does GCP Pub/Sub work in Microservices?
Similar to Kafka — decouples services via async messaging.

```
Order Service → publishes to Pub/Sub topic "order-created"
    ↓
Payment Service (subscriber) → receives message, processes payment
Notification Service (subscriber) → sends email to customer
```

```java
// Publish message
@Autowired
private PubSubTemplate pubSubTemplate;

public void publishOrder(String orderJson) {
    pubSubTemplate.publish("order-created", orderJson);
}

// Subscribe
@PubSubSubscriber(subscription = "payment-subscription")
public void handleOrder(BasicAcknowledgeablePubsubMessage message) {
    String payload = message.getPubsubMessage().getData().toStringUtf8();
    // process payment
    message.ack();
}
```

---

### Q40. What is Cloud Endpoints / API Gateway in GCP?
Manages, secures, and monitors your APIs deployed on GCP.

Features:
- Authentication (API key, JWT, OAuth2)
- Rate limiting & quota management
- Request/response logging
- OpenAPI spec support
- Works with GKE, Cloud Run, App Engine

---

### Q41. How do you manage secrets in GCP Microservices?
Use **Secret Manager** — never hardcode secrets in code or config files.

```java
// Access secret in Spring Boot
@Value("${spring.cloud.gcp.secretmanager.secret.db-password}")
private String dbPassword;
```

```yaml
# application.yaml
spring:
  cloud:
    gcp:
      secretmanager:
        secret:
          db-password: sm://my-project/db-password
```

---

### Q42. What is Workload Identity in GKE?
Allows GKE pods to access GCP services (Pub/Sub, Cloud SQL, Secret Manager) securely without service account key files.

```
Pod → Workload Identity → GCP Service Account → Access GCP APIs
```
No JSON key files needed — more secure than mounting key files.

---

### Q43. How do you implement CI/CD for Microservices on GCP?

```
Developer pushes code to GitHub/Cloud Source Repositories
    ↓
Cloud Build triggers automatically
    ↓
Build Docker image → Push to Artifact Registry
    ↓
Deploy to GKE (kubectl apply or Helm)
    ↓
Cloud Monitoring alerts on deployment health
```

```yaml
# cloudbuild.yaml
steps:
- name: 'gcr.io/cloud-builders/mvn'
  args: ['package', '-DskipTests']

- name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', 'gcr.io/$PROJECT_ID/order-service:$SHORT_SHA', '.']

- name: 'gcr.io/cloud-builders/docker'
  args: ['push', 'gcr.io/$PROJECT_ID/order-service:$SHORT_SHA']

- name: 'gcr.io/cloud-builders/kubectl'
  args: ['set', 'image', 'deployment/order-service',
         'order-service=gcr.io/$PROJECT_ID/order-service:$SHORT_SHA']
```

---

### Q44. How do you do distributed tracing on GCP?
Use **Cloud Trace** integrated with Spring Cloud Sleuth.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-gcp-starter-trace</artifactId>
</dependency>
```

```yaml
# application.yaml
spring:
  cloud:
    gcp:
      trace:
        enabled: true
        sampling-rate: 1.0  # 100% sampling
```

Every request gets a **Trace ID** — visible in GCP Console → Cloud Trace.

---

## 14. SPRING SECURITY IN MICROSERVICES

### Q45. How do you implement JWT authentication in Spring Boot?

**Step 1 — Add dependency:**
```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.11.5</version>
</dependency>
```

**Step 2 — JWT Utility:**
```java
@Component
public class JwtUtil {

    private final String SECRET = "mySecretKey";

    public String generateToken(String username) {
        return Jwts.builder()
            .setSubject(username)
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + 86400000)) // 1 day
            .signWith(SignatureAlgorithm.HS256, SECRET)
            .compact();
    }

    public String extractUsername(String token) {
        return Jwts.parser().setSigningKey(SECRET)
            .parseClaimsJws(token).getBody().getSubject();
    }

    public boolean validateToken(String token) {
        try {
            Jwts.parser().setSigningKey(SECRET).parseClaimsJws(token);
            return true;
        } catch (JwtException e) {
            return false;
        }
    }
}
```

**Step 3 — JWT Filter:**
```java
@Component
public class JwtFilter extends OncePerRequestFilter {

    @Autowired
    private JwtUtil jwtUtil;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String authHeader = request.getHeader("Authorization");

        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            String token = authHeader.substring(7);
            if (jwtUtil.validateToken(token)) {
                String username = jwtUtil.extractUsername(token);
                UsernamePasswordAuthenticationToken auth =
                    new UsernamePasswordAuthenticationToken(username, null, List.of());
                SecurityContextHolder.getContext().setAuthentication(auth);
            }
        }
        chain.doFilter(request, response);
    }
}
```

---

### Q46. How do you configure Spring Security for Microservices?

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Autowired
    private JwtFilter jwtFilter;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf().disable()
            .sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            .and()
            .authorizeHttpRequests()
                .requestMatchers("/auth/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            .and()
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

---

### Q47. What is OAuth2 and how does it work in Microservices?

```
User → Client App → Authorization Server (Google/Keycloak)
                          ↓
                    Issues Access Token (JWT)
                          ↓
Client App → Microservice (sends token in header)
                          ↓
              Microservice validates token
                          ↓
                    Returns response
```

```yaml
# Resource Server config
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://accounts.google.com
```

---

### Q48. What is the difference between Authentication and Authorization?

| | Authentication | Authorization |
|---|---|---|
| Question | Who are you? | What can you do? |
| Example | Login with username/password | Can user delete records? |
| Spring | `AuthenticationManager` | `@PreAuthorize`, roles |
| Token | Issues JWT | JWT contains roles/scopes |

---

### Q49. How do you implement Role-Based Access Control (RBAC)?

```java
// Method level security
@PreAuthorize("hasRole('ADMIN')")
@DeleteMapping("/users/{id}")
public void deleteUser(@PathVariable Long id) {
    userService.delete(id);
}

@PreAuthorize("hasAnyRole('ADMIN', 'MANAGER')")
@GetMapping("/reports")
public List<Report> getReports() {
    return reportService.getAll();
}
```

```yaml
# Enable method security
@EnableMethodSecurity
```

---

### Q50. How do you handle CORS in Spring Boot Microservices?

```java
@Configuration
public class CorsConfig {

    @Bean
    public CorsFilter corsFilter() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("https://yourdomain.com"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
        config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
        config.setAllowCredentials(true);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return new CorsFilter(source);
    }
}
```

---

### Q51. How do you secure service-to-service communication?
Use **mTLS (Mutual TLS)** — both services authenticate each other.

- In Kubernetes: use **Istio** service mesh (handles mTLS automatically)
- Without Istio: configure client certificates manually

```yaml
# Istio enables mTLS automatically via PeerAuthentication
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
spec:
  mtls:
    mode: STRICT
```

---

## 15. SYSTEM DESIGN

### Q52. Design a URL Shortener (like bit.ly) using Microservices

```
Client → API Gateway
              ↓
    ┌─────────────────────┐
    │   URL Service       │  → Generates short code
    │   (Spring Boot)     │  → Stores in Redis (cache)
    └─────────────────────┘  → Stores in MySQL (persistent)
              ↓
    ┌─────────────────────┐
    │   Analytics Service │  → Tracks clicks via Kafka events
    └─────────────────────┘
              ↓
    ┌─────────────────────┐
    │   Notification Svc  │  → Sends email reports
    └─────────────────────┘
```

Key design decisions:
- Short code: Base62 encoding of auto-increment ID
- Redis for fast redirects (sub-millisecond)
- Kafka for async click tracking
- CDN for global fast redirects

---

### Q53. Design an E-Commerce Order System using Microservices

```
Client → API Gateway (Auth + Rate Limiting)
              ↓
    ┌──────────────────────────────────────────┐
    │  User Service  │  Product Service         │
    │  (MySQL)       │  (MongoDB)               │
    └──────────────────────────────────────────┘
              ↓
    ┌─────────────────────┐
    │   Order Service     │ → Publishes "OrderCreated" to Kafka
    │   (PostgreSQL)      │
    └─────────────────────┘
              ↓
    ┌──────────────────────────────────────────┐
    │  Payment Service   │  Inventory Service   │
    │  (Stripe/Razorpay) │  (MySQL)             │
    └──────────────────────────────────────────┘
              ↓
    ┌─────────────────────┐
    │  Notification Svc   │ → Email/SMS via SendGrid/Twilio
    └─────────────────────┘
```

SAGA pattern for distributed transaction:
```
OrderCreated → PaymentProcessed → InventoryReserved → OrderConfirmed
     ↓ (failure)
PaymentFailed → OrderCancelled → InventoryReleased (compensating transactions)
```

---

### Q54. Design a Real-Time Notification System

```
Event Sources (Order, Payment, Shipping services)
    ↓ publish events
Kafka Topics
    ↓ consume
Notification Service
    ↓
    ├── Email (SendGrid)
    ├── SMS (Twilio)
    ├── Push (Firebase FCM)
    └── WebSocket (real-time in browser)
```

Key considerations:
- **Idempotency** — don't send duplicate notifications
- **Dead Letter Queue** — retry failed notifications
- **User preferences** — store in Redis (fast lookup)
- **Rate limiting** — don't spam users

---

### Q55. How do you scale a Microservice for high traffic?

**Horizontal Scaling:**
```yaml
# HPA in Kubernetes
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  scaleTargetRef:
    name: order-service
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

**Caching:**
```java
@Cacheable(value = "orders", key = "#id")
public Order getOrder(Long id) {
    return orderRepository.findById(id).orElseThrow();
}
```

**Database:**
- Read replicas for read-heavy workloads
- Connection pooling (HikariCP)
- Sharding for massive scale

---

### Q56. How do you design for High Availability in Microservices?

| Strategy | Implementation |
|---|---|
| Multiple replicas | min 3 pods per service in GKE |
| Multi-zone deployment | GKE regional cluster |
| Health checks | Liveness & Readiness probes |
| Circuit breaker | Resilience4j |
| DB redundancy | Cloud SQL with failover replica |
| CDN | Cloud CDN for static content |
| Load balancer | GCP Load Balancer / Ingress |

```yaml
# Liveness and Readiness probes
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 5
```

---

### Q57. What is the Strangler Fig Pattern?
Gradually migrate a monolith to microservices without big-bang rewrite.

```
Step 1: Monolith handles all traffic
Step 2: Extract "Payment" as microservice → route /payments to new service
Step 3: Extract "Orders" as microservice → route /orders to new service
Step 4: Keep extracting until monolith is retired
```

- Low risk — monolith still runs during migration
- API Gateway handles routing between old and new
- Teams can work independently on each service

---

### Q58. How do you handle distributed transactions without SAGA?

**Two-Phase Commit (2PC)** — not recommended for microservices:
- Phase 1: All services vote YES/NO
- Phase 2: Coordinator commits or rolls back all

Problems with 2PC in Microservices:
- Blocking — holds locks across services
- Single point of failure (coordinator)
- Does not work well with Kafka/async

**Better alternatives:**
- SAGA pattern (choreography or orchestration)
- Eventual consistency with compensating transactions
- Outbox pattern

---

### Q59. What is the Outbox Pattern?
Ensures messages are published to Kafka reliably without losing data.

```
Order Service writes to DB:
  ┌──────────────┐    ┌──────────────────┐
  │ orders table │    │ outbox table     │
  │ (new order)  │ +  │ (event to send)  │  ← same DB transaction
  └──────────────┘    └──────────────────┘
                              ↓
                    Outbox Poller (Debezium/CDC)
                              ↓
                         Kafka Topic
                              ↓
                    Other Microservices
```

Guarantees: **at-least-once delivery** — no lost messages even if app crashes.

---

### Q60. Design a Caching Strategy for Microservices

```
Request → Check Redis Cache
              ↓ (cache hit)
         Return cached data  ← fast response

              ↓ (cache miss)
         Query Database
              ↓
         Store in Redis (TTL: 5 min)
              ↓
         Return data
```

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(5))
            .disableCachingNullValues();

        return RedisCacheManager.builder(factory)
            .cacheDefaults(config)
            .build();
    }
}
```

Cache strategies:
- **Cache-Aside** — app checks cache, loads from DB on miss
- **Write-Through** — write to cache and DB simultaneously
- **Write-Behind** — write to cache first, async write to DB
- **TTL-based eviction** — expire stale data automatically

---

## QUICK REFERENCE — ALL PATTERNS

| Pattern | Problem Solved |
|---|---|
| API Gateway | Single entry point, auth, routing |
| Circuit Breaker | Prevent cascading failures |
| Bulkhead | Isolate thread pools per service |
| Retry | Handle transient failures |
| SAGA | Distributed transactions |
| CQRS | Separate read/write models |
| Event Sourcing | Full audit trail of state changes |
| Outbox | Reliable event publishing |
| Sidecar | Attach helper to main container |
| Strangler Fig | Gradual monolith migration |
| Service Mesh | Secure service-to-service comms |
| Database per Service | Data isolation per microservice |
| Cache-Aside | Reduce DB load with Redis |
| Idempotency | Safe to retry requests |

---

*Good luck with your interview!*
