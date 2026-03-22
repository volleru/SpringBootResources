# Scalable Cloud-Native Mail System Design
### Like Gmail / Yahoo Mail — 10M Users on Google Cloud

---

## System Requirements

### Functional Requirements
- Send and receive emails
- Inbox, Sent, Drafts, Spam, Trash folders
- Search emails (full-text)
- Attachments (up to 25MB per email)
- Read/unread status, labels, starring
- Real-time notifications when new email arrives
- Email threading (conversation view)
- Spam detection

### Non-Functional Requirements
- **10M total users**, **1M daily active users (DAU)**
- Each active user sends/receives avg **10KB of mail data per day**
- **Daily data ingested:** 1M users × 10KB = **10GB/day**
- **Annual storage growth:** ~3.65TB/year
- Email delivery latency: < 2 seconds for internal mail
- 99.99% uptime (< 52 mins downtime/year)
- Emails must never be lost
- Read your own writes consistency (you see your sent email immediately)

### Capacity Estimation

```
Daily Active Users        : 1,000,000
Avg emails sent/user/day  : 5
Avg email size            : 10KB (metadata + body)
Daily email volume        : 5M emails/day
Emails per second (peak)  : ~120 emails/sec (2x avg = 240 peak)
Daily storage             : 5M × 10KB = 50GB/day
With attachments (avg)    : ~200GB/day
3-year storage            : ~220TB (with replication)
Read:Write ratio          : 10:1 (people read more than write)
```

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         CLIENTS                                          │
│   Web Browser      Mobile App (iOS/Android)      Email Clients (SMTP)   │
└────────────┬──────────────────┬────────────────────────┬────────────────┘
             │                  │                        │
             ▼                  ▼                        ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    Google Cloud Load Balancer (HTTPS/WSS)                │
│                    Cloud Armor (DDoS, WAF, Rate Limiting)                │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
┌─────────────────┐  ┌─────────────────┐  ┌──────────────────┐
│   API Gateway   │  │   SMTP Gateway  │  │  WebSocket Server │
│  (Spring Boot)  │  │  (Inbound Mail) │  │  (Notifications)  │
│  GKE — 10 pods  │  │  GKE — 5 pods  │  │  GKE — 5 pods    │
└────────┬────────┘  └────────┬────────┘  └──────────────────┘
         │                    │
         ▼                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         Apache Kafka (Confluent Cloud)                   │
│  Topics: mail.inbound  mail.outbound  mail.notification  mail.search    │
└──────────┬──────────────────┬────────────────────┬──────────────────────┘
           │                  │                    │
           ▼                  ▼                    ▼
┌─────────────────┐  ┌──────────────────┐  ┌─────────────────────┐
│  Mail Processor │  │  Search Indexer  │  │  Notification Svc   │
│  Service        │  │  Service         │  │  (Push/WebSocket)   │
│  (Spring Boot)  │  │  (Spring Boot)   │  │  (Spring Boot)      │
└────────┬────────┘  └────────┬─────────┘  └─────────────────────┘
         │                    │
         ▼                    ▼
┌──────────────┐     ┌─────────────────┐     ┌──────────────────┐
│  Cloud Spanner│     │  Elasticsearch  │     │  Cloud Bigtable  │
│  (Metadata)  │     │  (Full-text     │     │  (Email Bodies & │
│              │     │   search)       │     │   Attachments)   │
└──────────────┘     └─────────────────┘     └──────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    Google Cloud Storage (GCS)                         │
│              Attachments, Large Bodies > 100KB                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Core Microservices

### 1. API Gateway Service
**Role:** Single entry point for all client requests. Auth validation, rate limiting, routing.

```java
// Spring Boot + Spring Cloud Gateway
@SpringBootApplication
public class ApiGatewayApplication {
    // Routes:
    // /api/v1/mail/**     → Mail Service
    // /api/v1/auth/**     → Auth Service
    // /api/v1/search/**   → Search Service
    // /ws/**              → WebSocket Service
}
```

**Tech:** Spring Cloud Gateway, Spring Security (JWT validation), Redis (rate limiting)
**GKE:** 10 pods, autoscale to 30 on traffic spike
**Rate Limits:**
- 100 requests/min per unauthenticated IP
- 1000 requests/min per authenticated user

---

### 2. Auth Service
**Role:** User registration, login, JWT issuance, session management.

```java
@RestController
@RequestMapping("/api/v1/auth")
public class AuthController {

    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(@RequestBody LoginRequest request) {
        // 1. Validate credentials against Cloud Spanner (users table)
        // 2. Issue JWT (access token 15min + refresh token 7days)
        // 3. Store refresh token in Redis with TTL
        // 4. Return tokens
    }
}
```

**JWT Payload:**
```json
{
  "sub": "user_id_123",
  "email": "user@example.com",
  "iat": 1711123456,
  "exp": 1711124356,
  "roles": ["USER"]
}
```

**Tech:** Spring Security, JWT (JJWT), Redis (refresh token store), Cloud Spanner (user data)
**GKE:** 3 pods

---

### 3. Mail Inbound Service (SMTP Gateway)
**Role:** Receives emails from external mail servers via SMTP protocol.

```
External Server → SMTP (port 25/587) → Mail Inbound Service → Kafka (mail.inbound)
```

```java
@Service
public class InboundMailService {

    @Autowired
    private KafkaTemplate<String, MailEvent> kafkaTemplate;

    public void receiveMail(MimeMessage message) {
        // 1. Validate SPF, DKIM, DMARC records
        // 2. Spam score check (Apache SpamAssassin or ML model)
        // 3. Virus scan (ClamAV)
        // 4. Parse headers, body, attachments
        // 5. Publish to Kafka

        MailEvent event = MailEvent.builder()
            .messageId(UUID.randomUUID().toString())
            .from(message.getFrom()[0].toString())
            .to(extractRecipients(message))
            .subject(message.getSubject())
            .bodyRef(storeBody(message))  // store large bodies in GCS
            .receivedAt(Instant.now())
            .spamScore(spamChecker.score(message))
            .build();

        kafkaTemplate.send("mail.inbound", event.getTo(), event);
    }
}
```

**Tech:** Spring Integration (SMTP), Apache James (SMTP server), Kafka producer
**GKE:** 5 pods

---

### 4. Mail Processor Service
**Role:** Core service — consumes from Kafka, stores emails, routes to recipients.

```java
@Service
public class MailProcessorService {

    @KafkaListener(
        topics = "mail.inbound",
        groupId = "mail-processor",
        concurrency = "6"
    )
    public void processInboundMail(ConsumerRecord<String, MailEvent> record,
                                   Acknowledgment ack) {
        MailEvent event = record.value();
        try {
            // 1. Determine target folder (Inbox or Spam based on spam score)
            String folder = event.getSpamScore() > 0.8 ? "SPAM" : "INBOX";

            // 2. Store email metadata in Cloud Spanner
            EmailMetadata metadata = emailMetadataRepo.save(EmailMetadata.builder()
                .messageId(event.getMessageId())
                .userId(resolveUserId(event.getTo()))
                .subject(event.getSubject())
                .from(event.getFrom())
                .folder(folder)
                .isRead(false)
                .bodyRef(event.getBodyRef())
                .receivedAt(event.getReceivedAt())
                .threadId(resolveThread(event))
                .build());

            // 3. Store body in Cloud Bigtable (fast row key lookup)
            bigtableWriter.write(event.getMessageId(), event.getBody());

            // 4. Publish notification event
            kafkaTemplate.send("mail.notification",
                event.getTo(), new NotificationEvent(metadata));

            // 5. Publish search index event
            kafkaTemplate.send("mail.search",
                event.getMessageId(), new SearchIndexEvent(metadata));

            ack.acknowledge();
        } catch (Exception e) {
            // Goes to DLT after 3 retries
            throw e;
        }
    }
}
```

**Tech:** Spring Kafka, Cloud Spanner, Cloud Bigtable, GCS
**GKE:** 8 pods, autoscale to 20

---

### 5. Mail API Service
**Role:** Serves client requests — list inbox, read email, send email, search.

```java
@RestController
@RequestMapping("/api/v1/mail")
public class MailController {

    // List inbox with pagination
    @GetMapping("/inbox")
    public Page<EmailSummaryDto> getInbox(
            @AuthenticationPrincipal UserPrincipal user,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "50") int size) {
        return mailService.getFolder(user.getId(), "INBOX",
            PageRequest.of(page, size, Sort.by("receivedAt").descending()));
    }

    // Get full email
    @GetMapping("/{messageId}")
    public EmailDetailDto getEmail(
            @AuthenticationPrincipal UserPrincipal user,
            @PathVariable String messageId) {
        // 1. Fetch metadata from Cloud Spanner (fast)
        // 2. Fetch body from Bigtable by messageId
        // 3. Mark as read (async update)
        // 4. Return combined response
        return mailService.getEmailDetail(user.getId(), messageId);
    }

    // Send email
    @PostMapping("/send")
    public ResponseEntity<SendResponse> sendEmail(
            @AuthenticationPrincipal UserPrincipal user,
            @RequestBody @Valid SendEmailRequest request) {
        // 1. Validate recipients
        // 2. Store in Sent folder
        // 3. Publish to mail.outbound Kafka topic
        // 4. Return immediately (async delivery)
        return ResponseEntity.accepted().body(mailService.send(user.getId(), request));
    }
}
```

**Caching strategy (Redis):**
```java
@Cacheable(value = "inbox", key = "#userId + ':' + #page")
public Page<EmailSummaryDto> getFolder(String userId, String folder, Pageable pageable) {
    // Cache inbox page 0 and 1 — most users only read first 2 pages
}

@CacheEvict(value = "inbox", key = "#userId + ':*'")
public void onNewEmailReceived(String userId) {
    // Invalidate user's inbox cache when new mail arrives
}
```

**GKE:** 10 pods, autoscale to 40

---

### 6. Search Service
**Role:** Index emails for full-text search using Elasticsearch.

```java
@Service
public class SearchIndexerService {

    @KafkaListener(topics = "mail.search", groupId = "search-indexer")
    public void indexEmail(SearchIndexEvent event) {
        EmailDocument doc = EmailDocument.builder()
            .messageId(event.getMessageId())
            .userId(event.getUserId())
            .subject(event.getSubject())
            .from(event.getFrom())
            .bodySnippet(event.getBodySnippet()) // first 500 chars only
            .receivedAt(event.getReceivedAt())
            .labels(event.getLabels())
            .build();

        elasticsearchClient.index(i -> i
            .index("emails-" + event.getUserId()) // per-user index
            .id(event.getMessageId())
            .document(doc));
    }
}

@RestController
@RequestMapping("/api/v1/search")
public class SearchController {

    @GetMapping
    public SearchResponse search(
            @AuthenticationPrincipal UserPrincipal user,
            @RequestParam String q,
            @RequestParam(defaultValue = "0") int from,
            @RequestParam(defaultValue = "20") int size) {

        SearchRequest request = SearchRequest.of(s -> s
            .index("emails-" + user.getId())
            .query(query -> query
                .multiMatch(m -> m
                    .fields("subject^3", "from^2", "bodySnippet")
                    .query(q)
                    .type(TextQueryType.BestFields)
                    .fuzziness("AUTO")
                ))
            .from(from).size(size)
            .highlight(h -> h.fields("subject", Map.of())
                              .fields("bodySnippet", Map.of())));

        return elasticsearchClient.search(request, EmailDocument.class);
    }
}
```

**GKE:** 4 pods

---

### 7. Notification Service
**Role:** Real-time push notifications when new email arrives.

```java
@Service
public class NotificationService {

    @KafkaListener(topics = "mail.notification", groupId = "notification-service")
    public void sendNotification(NotificationEvent event) {
        // 1. Check if user has active WebSocket session
        if (sessionRegistry.isOnline(event.getUserId())) {
            webSocketTemplate.convertAndSendToUser(
                event.getUserId(), "/queue/notifications",
                new NewMailNotification(event.getSubject(), event.getFrom())
            );
        }
        // 2. Send mobile push notification (FCM)
        fcmService.sendPush(event.getUserId(), event.getSubject());

        // 3. Update unread count in Redis
        redisTemplate.opsForValue().increment("unread:" + event.getUserId());
    }
}

// WebSocket config
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/queue", "/topic");
        config.setApplicationDestinationPrefixes("/app");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws").setAllowedOrigins("*").withSockJS();
    }
}
```

**GKE:** 5 pods with sticky sessions (session affinity on load balancer)

---

## Database Design

### Cloud Spanner — Email Metadata (SQL, Globally Distributed)

```sql
-- Users table
CREATE TABLE users (
    user_id       STRING(36) NOT NULL,
    email         STRING(255) NOT NULL,
    display_name  STRING(255),
    created_at    TIMESTAMP NOT NULL,
    storage_used  INT64 DEFAULT 0,  -- bytes used
    storage_quota INT64 DEFAULT 15737418240,  -- 15GB default
) PRIMARY KEY (user_id);

CREATE UNIQUE INDEX idx_users_email ON users(email);

-- Email metadata table
CREATE TABLE email_metadata (
    user_id      STRING(36) NOT NULL,
    message_id   STRING(36) NOT NULL,
    thread_id    STRING(36),
    subject      STRING(998),
    from_address STRING(255) NOT NULL,
    folder       STRING(50) NOT NULL,  -- INBOX, SENT, DRAFTS, SPAM, TRASH
    is_read      BOOL DEFAULT FALSE,
    is_starred   BOOL DEFAULT FALSE,
    received_at  TIMESTAMP NOT NULL,
    size_bytes   INT64,
    has_attachment BOOL DEFAULT FALSE,
    body_ref     STRING(500),  -- GCS path or Bigtable row key
    labels       ARRAY<STRING(50)>,
) PRIMARY KEY (user_id, received_at DESC, message_id),
  INTERLEAVE IN PARENT users ON DELETE CASCADE;

-- Threads table
CREATE TABLE threads (
    user_id     STRING(36) NOT NULL,
    thread_id   STRING(36) NOT NULL,
    subject     STRING(998),
    updated_at  TIMESTAMP NOT NULL,
    message_count INT64 DEFAULT 1,
    unread_count  INT64 DEFAULT 0,
) PRIMARY KEY (user_id, thread_id);

-- Labels table
CREATE TABLE labels (
    user_id    STRING(36) NOT NULL,
    label_id   STRING(36) NOT NULL,
    name       STRING(100) NOT NULL,
    color      STRING(7),
) PRIMARY KEY (user_id, label_id);
```

**Why Cloud Spanner:**
- Horizontally scalable SQL
- ACID transactions across regions
- 99.999% SLA
- No manual sharding

---

### Cloud Bigtable — Email Bodies (NoSQL, High Throughput)

```
Row key format: {userId}#{messageId}

Column families:
  body:
    content      → full email body (HTML/text)
    content_type → text/html or text/plain
  attachments:
    att_{n}_name → filename
    att_{n}_ref  → GCS path
    att_{n}_size → bytes
  headers:
    raw          → raw email headers blob
```

**Why Bigtable:**
- Single-digit millisecond reads at petabyte scale
- Perfect for key-based lookups (messageId)
- Handles variable-size values efficiently

---

### Redis (Memorystore) — Caching & Sessions

```
Keys:
  session:{token}              → userId (TTL: 24h)
  inbox:{userId}:{page}        → cached inbox JSON (TTL: 30s)
  unread:{userId}              → unread count integer
  rate_limit:{ip}:{minute}     → request count (TTL: 60s)
  ws_session:{userId}          → WebSocket server instance ID
  search_recent:{userId}       → recent search queries (list, max 10)
```

---

### Google Cloud Storage — Attachments & Large Bodies

```
Bucket structure:
  gs://mail-attachments-prod/
    {userId}/
      {year}/{month}/
        {messageId}/
          {filename}

  gs://mail-bodies-prod/
    {userId}/
      {messageId}/body.html    (for bodies > 100KB)
```

**Lifecycle policy:**
- Spam folder: delete after 30 days
- Trash: delete after 30 days
- Sent/Inbox: move to Nearline after 1 year, Coldline after 3 years

---

## Kafka Topics Design

```
Topic: mail.inbound
  Partitions: 24 (keyed by recipient email → same user, same partition)
  Replication: 3
  Retention: 24 hours
  Consumer groups: mail-processor, audit-logger

Topic: mail.outbound
  Partitions: 24 (keyed by sender userId)
  Replication: 3
  Retention: 24 hours
  Consumer groups: smtp-delivery, sent-folder-writer

Topic: mail.notification
  Partitions: 12
  Retention: 1 hour
  Consumer groups: notification-service

Topic: mail.search
  Partitions: 12
  Retention: 6 hours
  Consumer groups: search-indexer

Topic: mail.inbound.DLT
  Partitions: 6
  Retention: 7 days
  Consumer groups: dlt-monitor, manual-reprocessor
```

---

## Google Cloud Infrastructure

### GKE Cluster Design

```
Cluster: mail-system-prod (us-central1, multi-zone: a,b,c)

Node Pools:
  api-pool:
    Machine type: n2-standard-8 (8 vCPU, 32GB RAM)
    Min: 5, Max: 30 nodes
    Autoscaling: CPU > 60% → scale up

  processor-pool:
    Machine type: n2-standard-16 (16 vCPU, 64GB RAM)
    Min: 3, Max: 15 nodes
    High memory for Kafka consumer processing

  search-pool:
    Machine type: n2-highmem-8 (8 vCPU, 64GB RAM)
    Min: 3, Max: 10 nodes
    Elasticsearch needs high memory

Regional cluster spans 3 zones for high availability.
PodDisruptionBudget: minimum 2 pods always running for each service.
```

### Service Deployment (Kubernetes)

```yaml
# Example: Mail API Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mail-api-service
spec:
  replicas: 10
  selector:
    matchLabels:
      app: mail-api-service
  template:
    spec:
      containers:
      - name: mail-api
        image: gcr.io/mail-prod/mail-api:v1.2.3
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "2000m"
            memory: "2Gi"
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "prod"
        - name: DB_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 30
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: mail-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mail-api-service
  minReplicas: 10
  maxReplicas: 40
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
```

---

## Observability & Monitoring

### Metrics (Micrometer + Cloud Monitoring)

```java
// Spring Boot Actuator + Micrometer
@Configuration
public class MetricsConfig {

    @Bean
    MeterRegistryCustomizer<MeterRegistry> metricsCommonTags() {
        return registry -> registry.config()
            .commonTags("service", "mail-api", "env", "prod", "region", "us-central1");
    }
}

// Custom business metrics
@Service
public class MailMetrics {

    private final Counter emailsSentCounter;
    private final Counter emailsReceivedCounter;
    private final Timer emailDeliveryTimer;
    private final Gauge activeWebSocketSessions;

    public MailMetrics(MeterRegistry registry) {
        this.emailsSentCounter = Counter.builder("mail.sent.total")
            .description("Total emails sent")
            .register(registry);

        this.emailsReceivedCounter = Counter.builder("mail.received.total")
            .description("Total emails received")
            .register(registry);

        this.emailDeliveryTimer = Timer.builder("mail.delivery.latency")
            .description("End-to-end email delivery latency")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
    }
}
```

### Key Metrics to Monitor

| Metric | Alert Threshold | Severity |
|---|---|---|
| `mail.delivery.latency.p99` | > 5 seconds | Critical |
| `mail.delivery.latency.p50` | > 1 second | Warning |
| `kafka.consumer.lag` (mail.inbound) | > 10,000 | Critical |
| `kafka.consumer.lag` (mail.inbound) | > 1,000 | Warning |
| `mail.sent.total` rate drop | < 50% of baseline | Critical |
| `jvm.memory.used` | > 85% heap | Warning |
| `hikaricp.connections.pending` | > 5 | Warning |
| `http.server.requests.p99` | > 2 seconds | Warning |
| Elasticsearch indexing lag | > 30 seconds | Warning |
| DLT message count | > 0 | Critical |
| GCS storage bucket size | > 80% of quota | Warning |

---

### Distributed Tracing (OpenTelemetry + Cloud Trace)

```yaml
# application.yml
management:
  tracing:
    sampling:
      probability: 0.1   # 10% sampling in prod (100% in dev)
  otlp:
    tracing:
      endpoint: https://telemetry.googleapis.com/v1/traces

logging:
  pattern:
    console: "%d{HH:mm:ss} [%X{traceId}/%X{spanId}] %-5level %logger - %msg%n"
```

**Trace spans created automatically:**
- HTTP request → response
- Kafka produce / consume
- Database queries (Cloud Spanner)
- Redis calls
- GCS reads/writes

**Sample trace for email send:**
```
[TraceId: abc123]
  ├── POST /api/v1/mail/send (12ms total)
  │     ├── JWT validation (1ms)
  │     ├── Request validation (0.5ms)
  │     ├── Spanner write - email_metadata (3ms)
  │     ├── GCS write - body (4ms)
  │     └── Kafka produce - mail.outbound (1ms)
  │
  └── [Async] Kafka consume - mail processor (8ms)
        ├── Bigtable write (2ms)
        ├── Kafka produce - mail.notification (0.5ms)
        └── Kafka produce - mail.search (0.5ms)
```

---

### Logging (Structured JSON → Cloud Logging)

```java
// logback-spring.xml — JSON structured logs
// All logs ingested by Cloud Logging automatically on GKE
```

```json
{
  "timestamp": "2026-03-22T10:15:30.123Z",
  "severity": "INFO",
  "service": "mail-api-service",
  "traceId": "abc123def456",
  "spanId": "789xyz",
  "userId": "user_abc",
  "messageId": "msg_xyz",
  "action": "EMAIL_SENT",
  "latencyMs": 12,
  "recipientCount": 2,
  "sizeBytes": 8192
}
```

**Log-based alerts in Cloud Monitoring:**
- ERROR rate > 1% of requests → PagerDuty alert
- Any `DLT_MESSAGE_RECEIVED` log → Slack alert
- Authentication failures > 100/min from single IP → Security alert

---

### Dashboards (Grafana + Cloud Monitoring)

**Dashboard 1 — System Health:**
- Request rate (RPS) per service
- P50/P95/P99 latency per endpoint
- Error rate per service
- Pod count and CPU/memory utilization

**Dashboard 2 — Email Pipeline:**
- Emails ingested per second
- Kafka consumer lag per topic
- End-to-end delivery latency histogram
- Spam detection rate
- DLT queue size

**Dashboard 3 — Infrastructure:**
- GKE node utilization
- Cloud Spanner CPU and read/write latency
- Bigtable reads/writes per second
- Redis hit rate and eviction rate
- GCS storage growth over time

**Dashboard 4 — Business Metrics:**
- Daily active users
- Emails sent/received per hour
- New user registrations
- Storage used per user (p50, p95, p99)

---

## Security Design

### Authentication & Authorization
- **JWT** with 15-minute expiry + 7-day refresh token
- Refresh tokens stored in Redis, invalidated on logout
- **Google Identity Platform** for social login (OAuth2)

### Transport Security
- TLS 1.3 enforced everywhere (client → LB → service → service)
- **Cloud Armor** — WAF, DDoS protection, geo-blocking
- mTLS between internal GKE services (Istio service mesh)

### Data Security
- Email bodies encrypted at rest (Cloud KMS managed keys)
- Attachments encrypted in GCS (CSEK or CMEK)
- Spanner data encrypted at rest by default

### Email Security
- **SPF** — validate sender IP is authorized for domain
- **DKIM** — validate email signature
- **DMARC** — policy for failing SPF/DKIM
- **Spam detection** — ML model scoring (0.0 to 1.0)
- **Virus scanning** — ClamAV on all attachments

### Secrets Management
- All credentials in **Google Secret Manager**
- Zero hardcoded secrets in code or config files
- Service accounts with minimal IAM permissions (principle of least privilege)

---

## Fault Tolerance & Resilience

### Circuit Breakers (Resilience4j)

```java
@CircuitBreaker(name = "bigtable", fallbackMethod = "fallbackGetBody")
public String getEmailBody(String messageId) {
    return bigtableClient.readRow(messageId);
}

public String fallbackGetBody(String messageId, Exception e) {
    // Return cached version or "body temporarily unavailable"
    return redisTemplate.opsForValue().get("body_cache:" + messageId);
}
```

```yaml
resilience4j:
  circuitbreaker:
    instances:
      bigtable:
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
        sliding-window-size: 20
      spanner:
        failure-rate-threshold: 30
        wait-duration-in-open-state: 30s
```

### Kafka Dead Letter Topic Handling

```java
@KafkaListener(topics = "mail.inbound.DLT")
public void handleDLT(ConsumerRecord<String, MailEvent> record) {
    String failureReason = new String(
        record.headers().lastHeader("kafka_dlt-exception-message").value()
    );
    // 1. Alert on-call engineer via PagerDuty
    alerting.page("DLT message received: " + failureReason);
    // 2. Store in dead_letter_emails table for manual review
    deadLetterRepo.save(new DeadLetterEmail(record, failureReason));
}
```

### Graceful Shutdown

```yaml
# application.yml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 60s
```

On SIGTERM:
1. Stop accepting new HTTP requests
2. Finish in-flight requests (up to 60s)
3. Kafka consumers commit current offsets and stop polling
4. Close DB connections cleanly

---

## Scalability Bottlenecks & Solutions

| Bottleneck | At Scale | Solution |
|---|---|---|
| Inbox query (Cloud Spanner) | 10M users × frequent reads | Redis cache inbox page 0 (30s TTL), read replicas |
| Full-text search (Elasticsearch) | 10GB/day new data | Per-user sharding, async indexing via Kafka |
| Email body storage | 200GB/day | Bigtable for hot (< 30 days), GCS for cold |
| WebSocket connections | 1M concurrent connections | Horizontal scale + sticky sessions, XMPP/gRPC streaming as alternative |
| Attachment uploads | Large files, slow uploads | Resumable uploads directly to GCS via signed URLs (bypass backend) |
| Spam detection | 240 emails/sec | Async scoring via Kafka, ML model on Vertex AI |

### Signed URL for Direct Attachment Upload (bypasses backend):
```java
@GetMapping("/upload-url")
public String getUploadUrl(@RequestParam String filename,
                           @RequestParam long contentLength) {
    // Client uploads directly to GCS — never hits our backend
    BlobInfo blobInfo = BlobInfo.newBuilder(
        BucketInfo.of("mail-attachments-prod"),
        userId + "/" + messageId + "/" + filename
    ).build();

    URL signedUrl = storage.signUrl(blobInfo, 15, TimeUnit.MINUTES,
        Storage.SignUrlOption.httpMethod(HttpMethod.PUT),
        Storage.SignUrlOption.withV4Signature());

    return signedUrl.toString();
}
```

---

## Data Flow Summary

### Sending an Email:
```
User → POST /api/v1/mail/send
     → API Gateway (JWT validate, rate limit)
     → Mail API Service
     → Store metadata in Spanner (SENT folder)
     → Store body in Bigtable (if ≤ 100KB) or GCS (if > 100KB)
     → Publish to Kafka: mail.outbound
     → Return 202 Accepted immediately

Kafka Consumer (Outbound Processor):
     → If recipient is internal → publish to mail.inbound
     → If recipient is external → SMTP delivery to external server
```

### Receiving an Email:
```
External SMTP Server → SMTP Gateway (port 25)
     → SPF/DKIM/DMARC validation
     → Spam scoring
     → Virus scan
     → Publish to Kafka: mail.inbound

Kafka Consumer (Mail Processor):
     → Store metadata in Spanner (INBOX or SPAM)
     → Store body in Bigtable
     → Publish to mail.notification
     → Publish to mail.search

Notification Consumer:
     → If user online: WebSocket push
     → Mobile: FCM push notification
     → Increment unread count in Redis

Search Indexer Consumer:
     → Index in Elasticsearch (async, eventually consistent)
```

---

## Cost Estimation (Monthly, 10M Users)

| Service | Usage | Estimated Cost |
|---|---|---|
| GKE (n2-standard-8, ~20 nodes avg) | 24/7 | ~$8,000 |
| Cloud Spanner (10 nodes) | 10M users metadata | ~$6,000 |
| Cloud Bigtable (10 nodes) | Email bodies | ~$5,000 |
| Google Cloud Storage (50TB) | Attachments, cold storage | ~$1,000 |
| Elasticsearch (via GKE) | 6 × n2-highmem-8 | ~$3,000 |
| Confluent Kafka (managed) | 200GB/day throughput | ~$2,500 |
| Redis Memorystore (50GB) | Caching, sessions | ~$800 |
| Cloud CDN + Load Balancer | Global traffic | ~$500 |
| Cloud Armor (WAF) | DDoS protection | ~$500 |
| Cloud Monitoring + Logging | Metrics, traces, logs | ~$1,000 |
| **Total** | | **~$28,300/month** |

> **Per user cost:** ~$0.003/user/month (scales down as users grow due to fixed infra costs)

---

## Tech Stack Summary

| Layer | Technology | Why |
|---|---|---|
| **Language** | Java 21 (Virtual Threads) | Performance, ecosystem, your expertise |
| **Framework** | Spring Boot 3.x | Production-ready, auto-config, actuator |
| **API** | Spring MVC + Spring WebSocket | REST + real-time |
| **Messaging** | Apache Kafka (Confluent Cloud) | Durable, scalable event bus |
| **Cache** | Redis (Cloud Memorystore) | Sub-ms reads, session store |
| **SQL DB** | Cloud Spanner | Global SQL, no sharding |
| **NoSQL DB** | Cloud Bigtable | High-throughput key-value |
| **Search** | Elasticsearch | Full-text search |
| **Object Store** | Google Cloud Storage | Attachments, large bodies |
| **Container** | Docker + GKE | Cloud-native deployment |
| **Service Mesh** | Istio | mTLS, observability |
| **CI/CD** | Cloud Build + ArgoCD | GitOps deployment |
| **Secrets** | Google Secret Manager | Centralized secrets |
| **Tracing** | OpenTelemetry + Cloud Trace | Distributed tracing |
| **Metrics** | Micrometer + Cloud Monitoring | Metrics + alerting |
| **Logging** | Logback JSON + Cloud Logging | Structured logs |
| **Dashboards** | Grafana | Visualization |
| **Alerting** | PagerDuty + Slack | Incident management |
| **Resilience** | Resilience4j | Circuit breaker, retry |
| **Auth** | JWT + Google Identity | Secure auth |
| **Security** | Cloud Armor + Istio mTLS | Network security |

---

*Designed for: 10M users | 1M DAU | 10GB/day data | 99.99% availability*
*Stack: Java 21 + Spring Boot 3 + Kafka + Cloud Spanner + Bigtable + GKE*
