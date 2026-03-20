# Java Concurrency & Multithreading — Senior Developer Interview Questions

---

## 1. Thread Pools & Executor Framework

**Q1. You have a web service with two thread pools — SYNC and ASYNC. How do you decide which worker goes into which pool? What happens if the SYNC pool gets saturated?**

> Real context: Worker routing between sync/async pools based on request type, blocking vs non-blocking operations.

**Expected answer:**
- SYNC pool: short-lived, CPU-bound or fast I/O workers. ASYNC pool: long-running, blocking I/O workers.
- Pool saturation → `RejectedExecutionException`. Mitigate with: bounded queues + rejection handler, circuit breaker, backpressure.
- Metrics: monitor active threads, queue depth, rejection rate.

---

**Q2. What is the difference between `setPersistBlockOneTime(true)` and always-blocking behavior in a worker? Why would you set this only for async pool workers?**

> Real context: `setPersistBlockOneTime` in web service workers.

**Expected answer:**
- `setPersistBlockOneTime` blocks the thread only once, then re-submits — prevents monopolizing threads in shared pools.
- In SYNC pool, you want predictable fast completion. In ASYNC pool, you can afford to block once per task without starving other requests.
- Without this, a single slow request could hold a SYNC thread for the entire duration.

---

**Q3. Explain `ThreadLocal` — when is it useful and what are the common pitfalls in a thread-pool based server?**

**Expected answer:**
- `ThreadLocal` gives each thread its own isolated variable copy. Useful for per-request context (user session, trace ID).
- **Pitfalls with thread pools**: threads are reused — `ThreadLocal` values from previous requests bleed into new ones unless explicitly `remove()`d.
- Memory leak: long-lived threads holding large objects via `ThreadLocal` in `WeakHashMap`-backed `ThreadLocalMap`.
- Always call `threadLocal.remove()` in a `finally` block or servlet filter.

---

**Q4. You see high CPU usage in production. Thread dump shows most threads in `WAITING` state on a `ReentrantLock`. What are the possible causes and how do you investigate?**

**Expected answer:**
- Causes: lock contention, long critical section, thread starvation if non-fair lock, deadlock.
- Investigation: `jstack`, look for `locked` and `waiting to lock` — identifies lock holder and waiters.
- Fix: reduce lock scope, use `ReadWriteLock` if reads dominate, switch to lock-free (`ConcurrentHashMap`, `AtomicReference`), or use `StampedLock` for optimistic reads.

---

## 2. Java Memory Model & Visibility

**Q5. What does `volatile` guarantee and what does it NOT guarantee?**

**Expected answer:**
- Guarantees: visibility (write is immediately visible to all threads), happens-before ordering.
- Does NOT guarantee: atomicity for compound operations (e.g., `volatile int i; i++` is not atomic).
- Use `AtomicInteger` for atomic increment, `volatile` for simple flags like `boolean running = true`.

---

**Q6. Explain the happens-before relationship. Give 3 concrete examples from Java.**

**Expected answer:**
1. Thread `start()` happens-before any action in the started thread.
2. `synchronized` block exit happens-before subsequent entry to the same monitor.
3. `volatile` write happens-before any subsequent volatile read of the same variable.
4. `Thread.join()` return happens-before all actions in the joined thread.
- Relevance: without happens-before, the JIT/CPU can reorder instructions causing subtle bugs.

---

## 3. CompletableFuture & Async Programming

**Q7. You have 3 downstream service calls (UDB, JCACHE, CAAS) that can run in parallel. How do you compose them with `CompletableFuture` and handle partial failures?**

**Expected answer:**
```java
CompletableFuture<UserData> udbFuture = CompletableFuture.supplyAsync(() -> callUdb(), executor);
CompletableFuture<CacheData> cacheFuture = CompletableFuture.supplyAsync(() -> callJcache(), executor);
CompletableFuture<CaasData> caasFuture = CompletableFuture.supplyAsync(() -> callCaas(), executor);

CompletableFuture.allOf(udbFuture, cacheFuture, caasFuture)
    .thenApply(v -> buildResponse(udbFuture.join(), cacheFuture.join(), caasFuture.join()))
    .exceptionally(ex -> fallbackResponse(ex));
```
- For partial failure tolerance: use `exceptionally()` on each individual future to return a default value.
- `allOf` waits for all; `anyOf` returns first completed.

---

**Q8. What is the difference between `thenApply`, `thenCompose`, and `thenCombine`?**

**Expected answer:**
- `thenApply(fn)`: maps result synchronously (like `Stream.map`). `fn` returns T.
- `thenCompose(fn)`: flatMap — `fn` returns `CompletableFuture<T>`, avoids nested futures.
- `thenCombine(other, fn)`: waits for two independent futures, combines their results.
- **Common mistake**: using `thenApply` when the function returns a future → `CompletableFuture<CompletableFuture<T>>`.

---

## 4. Concurrency Data Structures

**Q9. `ConcurrentHashMap` vs `Collections.synchronizedMap` — when would you choose one over the other?**

**Expected answer:**
- `synchronizedMap`: full mutex on every operation. Only one thread at a time. Simple but low throughput.
- `ConcurrentHashMap`: segment/bucket-level locking (Java 8+: CAS + synchronized on bin). Allows concurrent reads and multiple write threads.
- `ConcurrentHashMap` does NOT allow `null` keys/values.
- For atomic compound operations use `computeIfAbsent`, `merge`, `putIfAbsent` — these are atomic.
- `synchronizedMap` is needed when you need to iterate while holding the lock externally.

---

**Q10. You need a rate limiter shared across threads. Walk through implementing one using `AtomicInteger` and `ScheduledExecutorService`.**

**Expected answer:**
```java
AtomicInteger counter = new AtomicInteger(0);
int limit = 100; // per second

ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor();
scheduler.scheduleAtFixedRate(() -> counter.set(0), 1, 1, TimeUnit.SECONDS);

boolean tryAcquire() {
    return counter.incrementAndGet() <= limit;
}
```
- More robust: token bucket with `Semaphore` or use `java.util.concurrent.Semaphore.tryAcquire()`.
- Production: use Guava `RateLimiter` or a distributed rate limiter (Redis).

---

## 5. Deadlock & Race Conditions

**Q11. You have two services A and B. A acquires lock1 then lock2. B acquires lock2 then lock1. How do you detect and fix the deadlock?**

**Expected answer:**
- Detection: `jstack` shows threads in `BLOCKED` state forming a cycle.
- Fix options:
  1. **Lock ordering**: always acquire locks in a consistent global order.
  2. **Try-lock with timeout**: `lock.tryLock(timeout, unit)` — back off and retry.
  3. **Lock-free design**: use `ConcurrentHashMap`, `AtomicReference`.
  4. **Single lock**: merge into one coarser lock if performance allows.

---

**Q12. Explain a scenario where `synchronized` on `this` is dangerous in a web service handler.**

**Expected answer:**
- If the handler is a shared singleton (common in Jersey/Spring), synchronizing on `this` means all requests serialize — massive throughput drop.
- Fix: use fine-grained locking (lock on specific resource), `ConcurrentHashMap`, or avoid shared mutable state entirely.
- Also: `synchronized` on `this` is exploitable if external code also synchronizes on the same object.

---

## 6. JDK 21 — Virtual Threads

**Q13. What are virtual threads (JDK 21) and how do they change the traditional thread-per-request model?**

**Expected answer:**
- Virtual threads are lightweight threads managed by the JVM, not the OS. Millions can exist simultaneously.
- Traditional model: 1 OS thread per request → limited by OS thread count (~thousands).
- Virtual threads: blocking I/O (HTTP call, DB query) parks the virtual thread without blocking the OS thread → carrier thread picks up other work.
- **When NOT to use**: CPU-bound work (no benefit), `synchronized` blocks that pin to carrier thread (use `ReentrantLock` instead).
- Impact on thread pools: `newVirtualThreadPerTaskExecutor()` — no pooling needed.

---

## 7. Scenario-Based Questions

**Q14. In a high-traffic email service, you notice intermittent `OutOfMemoryError: unable to create new native thread`. What do you investigate?**

**Expected answer:**
- OS limit on threads per process (`ulimit -u`).
- Thread leak: threads created but never terminated — missing shutdown hooks, uncaught exceptions in thread factories.
- Thread pool misconfiguration — unbounded pool with high request rate.
- Steps: thread dump (`jstack`), count live threads, check for `TIMED_WAITING` threads stuck in sleep/wait.
- Fix: bound all thread pools, implement health checks on thread count.

---

**Q15. You're doing a canary deployment. The canary pod processes requests 3x slower than stable. Thread dump shows threads `WAITING` in `LockSupport.park`. What is happening?**

**Expected answer:**
- `LockSupport.park` is the underlying block used by `CompletableFuture`, `Semaphore`, `BlockingQueue.take()`, `ReentrantLock`.
- Likely causes: downstream service timeout too high, connection pool exhausted, queue backup.
- Compare canary vs stable: GC pauses, connection pool stats, queue depth.
- Prometheus metrics to check: active thread count, queue wait time, connection pool usage.
