# Senior Java Engineer — Interview Prep Plan (10 Days)

## Job Requirements Coverage

| Area | Days |
|------|------|
| Java 8+, JVM Internals, GC, Memory, Tuning | Day 1 |
| Advanced Concurrency, JMM, Non-blocking | Day 2 |
| Spring Boot, MVC, Data JPA, Batch | Day 3 |
| Spring Security, OAuth2, OIDC, JWT, OWASP | Day 4 |
| Spring Cloud, API Design, OpenAPI, GraphQL | Day 5 |
| Microservices Patterns, DDD, Hexagonal | Day 6 |
| Distributed Systems, CAP, SAGA, Consistency | Day 7 |
| Kafka, RabbitMQ, Event-Driven Design | Day 8 |
| Architecture, SOLID, Design Patterns, Scalability | Day 9 |
| Coding Round + System Design Mock | Day 10 |

## Files

| File | Content |
|------|---------|
| [day01_jvm_gc.md](day01_jvm_gc.md) | JVM internals, GC algorithms, memory model, classloading |
| [day02_concurrency.md](day02_concurrency.md) | Executors, ForkJoin, locks, JMM, non-blocking patterns |
| [day03_spring_core.md](day03_spring_core.md) | Spring Boot, MVC, Data JPA, Batch, REST |
| [day04_security.md](day04_security.md) | OAuth2, OIDC, JWT, mTLS, OWASP, secure coding |
| [day05_spring_cloud_api.md](day05_spring_cloud_api.md) | Spring Cloud, Gateway, Circuit Breaker, OpenAPI, GraphQL |
| [day06_microservices.md](day06_microservices.md) | Microservices patterns, DDD, hexagonal, service mesh |
| [day07_distributed_systems.md](day07_distributed_systems.md) | CAP, ACID, eventual consistency, SAGA, reliability |
| [day08_messaging.md](day08_messaging.md) | Kafka, RabbitMQ, streaming, partitioning, backpressure |
| [day09_architecture_patterns.md](day09_architecture_patterns.md) | SOLID, GoF patterns, scalability, observability stack |
| [day10_coding_mock.md](day10_coding_mock.md) | System design walkthroughs, coding exercises, trade-offs |

## Interview Strategy

### C1 Coding Round — What They're Really Testing
- **Not DSA speed** — they want to see design thinking
- Before writing any code, verbalize:
  - What trade-offs does this design make?
  - How does it handle failure?
  - How does it scale?
  - How would you observe/monitor it in production?

### Depth Signal Words (Use These)
- "happens-before guarantee" (not just "thread safety")
- "promotion failure" (not just "GC issue")
- "idempotency key" (not just "retry")
- "at-least-once / exactly-once semantics"
- "back-pressure" (not just "slow consumer")
- "saga orchestration vs choreography trade-off"

---

*Generated for interview prep — 10-day structured plan*
