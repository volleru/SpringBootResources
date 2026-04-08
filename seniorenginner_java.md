# Senior Java Engineer Interview Questions (10+ Years Experience)

---

## 1. Java Core & Advanced

**Q1. Explain the Java Memory Model (JMM) and how it affects concurrent programming.**
- Discuss happens-before relationships, visibility guarantees, volatile semantics
- How JMM prevents instruction reordering

**Q2. What is the difference between `volatile`, `synchronized`, and `AtomicInteger`? When would you use each?**
- volatile: visibility but not atomicity
- synchronized: mutual exclusion + visibility
- Atomic classes: lock-free CAS operations

**Q3. Explain how `HashMap` works internally in Java 8+. What changed from earlier versions?**
- Array of buckets, LinkedList → TreeNode (red-black tree) when bucket size > 8
- Hash collision handling, load factor, rehashing

**Q4. What are the differences between `String`, `StringBuilder`, and `StringBuffer`? How does String interning work?**

**Q5. Explain `equals()` and `hashCode()` contract. What happens if you override only one?**

**Q6. What is the difference between checked and unchecked exceptions? When should you use each?**

**Q7. Explain Java Generics — type erasure, wildcards (`? extends T` vs `? super T`), and bounded type parameters.**

**Q8. What are functional interfaces? How do lambdas and method references work under the hood?**

**Q9. Explain `CompletableFuture` and how it differs from `Future`. How do you handle exceptions in async pipelines?**

**Q10. What is the difference between `parallelStream()` and `stream()`? When can parallelStream cause issues?**

---

## 2. JVM Internals & Performance

**Q11. Explain JVM architecture — ClassLoader, Runtime Data Areas, Execution Engine.**

**Q12. Describe the different garbage collectors in Java (G1, ZGC, Shenandoah, CMS). How do you choose one?**
- G1: balanced throughput/latency
- ZGC/Shenandoah: ultra-low pause times
- When to use each based on heap size and latency SLAs

**Q13. How do you diagnose and fix a memory leak in a Java application?**
- Tools: heap dumps, VisualVM, MAT, jmap, jstack
- Common causes: static collections, listeners not deregistered, ThreadLocal misuse

**Q14. What is JIT compilation? Explain tiered compilation and how HotSpot optimizes code.**
- C1 (client) vs C2 (server) compilers
- Inlining, escape analysis, dead code elimination

**Q15. How do you tune JVM heap settings for a high-throughput microservice?**
- `-Xms`, `-Xmx`, `-XX:NewRatio`, GC tuning flags

**Q16. What is a thread dump and how do you analyze one? How do you detect deadlocks?**

**Q17. Explain class loading delegation model (parent-first). When would you break it?**

**Q18. What is metaspace and how does it differ from PermGen?**

---

## 3. Concurrency & Multithreading

**Q19. Explain the `java.util.concurrent` package — `ExecutorService`, `ThreadPoolExecutor`, `ForkJoinPool`.**
- Core/max pool size, queue types, rejection policies
- When to use ForkJoinPool vs ThreadPoolExecutor

**Q20. What is the difference between `ReentrantLock` and `synchronized`? What extra features does `ReentrantLock` provide?**
- Fairness, tryLock, lockInterruptibly, Condition variables

**Q21. Explain `CountDownLatch`, `CyclicBarrier`, `Semaphore`, and `Phaser`. Give real-world use cases.**

**Q22. What is a `ConcurrentHashMap` and how does it achieve thread safety without locking the whole map?**
- Segment-level locking (Java 7) vs CAS + synchronized on bucket head (Java 8+)

**Q23. Explain the producer-consumer pattern. How would you implement it using `BlockingQueue`?**

**Q24. What is a race condition? Give an example and explain how to fix it.**

**Q25. What is thread starvation and how do you prevent it?**

**Q26. Explain `ThreadLocal`. What are its use cases and potential pitfalls (memory leaks in thread pools)?**

**Q27. How does `StampedLock` differ from `ReadWriteLock`? When is it preferable?**

---

## 4. System Design

**Q28. Design a URL shortener (like bit.ly). Walk through the full system.**
- Hashing strategy, collision handling, redirection, analytics, expiry

**Q29. Design a distributed rate limiter. How do you handle it across multiple nodes?**
- Token bucket vs leaky bucket, Redis-based sliding window

**Q30. Design a notification service that handles millions of push/email/SMS notifications.**
- Message queues, fan-out patterns, retry/dead letter queues, idempotency

**Q31. Design a real-time leaderboard for a gaming platform.**
- Redis sorted sets, eventual consistency, sharding

**Q32. How would you design a distributed cache? What are the trade-offs between write-through, write-behind, and cache-aside patterns?**

**Q33. Design an API gateway. What cross-cutting concerns does it handle?**
- Auth, rate limiting, circuit breaking, request routing, observability

**Q34. How would you design a system to handle 1 million concurrent WebSocket connections?**

**Q35. Design a search autocomplete system.**
- Trie vs inverted index, prefix matching, ranking, caching

**Q36. How would you design a distributed job scheduler?**
- Leader election, at-least-once vs exactly-once semantics, Quartz vs custom

**Q37. Explain the CAP theorem. Give examples of systems that prioritize CP vs AP.**

---

## 5. Microservices & Distributed Systems

**Q38. What are the main challenges of microservices compared to monoliths? When would you NOT use microservices?**

**Q39. Explain the Saga pattern for distributed transactions. Compare choreography vs orchestration.**

**Q40. How do you handle partial failures in a microservices architecture? Explain circuit breakers, bulkheads, and timeouts.**

**Q41. What is service discovery? Compare client-side (Ribbon) vs server-side (AWS ALB) discovery.**

**Q42. How do you implement idempotency in APIs? Why is it important in distributed systems?**

**Q43. Explain the outbox pattern. How does it solve the dual-write problem?**

**Q44. What is eventual consistency? How do you design UX around it?**

**Q45. How do you handle schema evolution in event-driven systems? (Avro, Protobuf, backward/forward compatibility)**

**Q46. What is CQRS and Event Sourcing? When would you use them together?**

**Q47. How do you implement distributed tracing? What tools have you used? (Jaeger, Zipkin, OpenTelemetry)**

---

## 6. Spring Boot & Frameworks

**Q48. Explain Spring's IoC container and dependency injection. What is the difference between `@Component`, `@Service`, `@Repository`, and `@Controller`?**

**Q49. How does Spring Boot auto-configuration work? How would you write a custom auto-configuration?**

**Q50. Explain Spring transaction management. What is transaction propagation and isolation? What is the difference between `REQUIRED` and `REQUIRES_NEW`?**

**Q51. How does Spring Security handle authentication and authorization? Explain the filter chain.**

**Q52. What is the difference between `@Transactional` on a class vs a method? What are common pitfalls (self-invocation)?**

**Q53. Explain Spring AOP — proxies, pointcuts, advice types. What are the limitations of Spring AOP vs AspectJ?**

**Q54. How do you handle distributed configuration in Spring Boot? (Spring Cloud Config, Kubernetes ConfigMaps)**

**Q55. Explain reactive programming with Spring WebFlux. When would you choose it over Spring MVC?**

---

## 7. Databases & Data Access

**Q56. Explain ACID properties. How does each one protect data integrity?**

**Q57. What is the N+1 query problem in ORM? How do you detect and fix it in Hibernate/JPA?**
- `FetchType.LAZY` vs `EAGER`, `@EntityGraph`, JOIN FETCH

**Q58. Explain database indexing strategies — B-tree, composite indexes, covering indexes, partial indexes.**

**Q59. When would you use a NoSQL database over a relational one? Compare document, key-value, column-family, and graph databases.**

**Q60. How do you implement optimistic vs pessimistic locking in JPA?**

**Q61. Explain database connection pooling. What are the key parameters to tune in HikariCP?**

**Q62. How do you handle database migrations in a CI/CD pipeline? (Flyway vs Liquibase)**

**Q63. What is database sharding? What are the challenges (cross-shard queries, rebalancing)?**

---

## 8. Design Patterns & Architecture

**Q64. Explain SOLID principles with Java examples. Which one is most commonly violated in your experience?**

**Q65. What is the difference between Factory, Abstract Factory, and Builder patterns? When do you use each?**

**Q66. Explain the Strategy pattern vs the Template Method pattern.**

**Q67. What is the Decorator pattern? How does Java I/O use it?**

**Q68. Explain the Observer pattern. How does it relate to event-driven architecture?**

**Q69. What is hexagonal architecture (ports and adapters)? What problem does it solve?**

**Q70. How do you approach refactoring a legacy monolith toward microservices? (Strangler Fig pattern)**

---

## 9. DevOps, CI/CD & Cloud

**Q71. How do you containerize a Java application? What are best practices for Java Docker images?**
- Multi-stage builds, JVM memory settings in containers, distroless images

**Q72. Explain Kubernetes concepts relevant to Java services — pods, services, deployments, ConfigMaps, liveness/readiness probes.**

**Q73. What is a blue-green deployment vs a canary release? When would you use each?**

**Q74. How do you implement health checks in a Spring Boot application for Kubernetes?**

**Q75. Explain the 12-factor app methodology. How does it guide microservice design?**

---

## 10. Leadership & Architecture Decisions

**Q76. Tell me about a time you made a key architectural decision. What trade-offs did you consider?**

**Q77. How do you approach technical debt? How do you prioritize paying it down?**

**Q78. How do you conduct a code review? What do you look for beyond functionality?**

**Q79. How do you mentor junior engineers? What techniques have worked well for you?**

**Q80. How do you handle disagreements about technical approach within a team?**

**Q81. Describe a production incident you led. How did you manage the response and post-mortem?**

**Q82. How do you evaluate whether to build vs buy a component?**

**Q83. How do you ensure non-functional requirements (performance, security, reliability) are met in a project?**

---

## Quick-Fire Concepts (Expect Brief Precise Answers)

- Difference between `Comparable` and `Comparator`
- What is `Optional` and when should you NOT use it?
- Difference between `List.of()` and `Arrays.asList()`
- What is `var` in Java 10+ and what are its limitations?
- Explain sealed classes and records (Java 16+)
- What is `try-with-resources` and how does it work?
- Difference between `Error` and `Exception`
- What is the difference between heap and stack memory?
- Explain `finalize()` — why is it deprecated?
- What is a phantom reference and when would you use it?

---

## Coding Challenges (Senior Level)

1. Implement a thread-safe LRU cache without using `LinkedHashMap`
2. Design and implement a rate limiter using token bucket algorithm
3. Implement a non-blocking producer-consumer queue
4. Write a custom `CompletableFuture`-based retry mechanism with exponential backoff
5. Implement a simple dependency injection container using reflection
6. Write a deep clone utility using serialization and without it
7. Implement `flatMap` for a custom `Optional`-like monad
8. Design a mini event bus with subscribe/publish and wildcard topic support

---

*Prepared for: Senior Java Engineer (10+ years experience)*
*Focus areas: Java internals, concurrency, system design, distributed systems, architecture*
