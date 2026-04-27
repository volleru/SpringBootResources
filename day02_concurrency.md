# Day 2 — Advanced Concurrency, Locks, JMM & Non-Blocking Programming

## Topics Covered
- ThreadPoolExecutor internals and tuning
- ForkJoinPool and work-stealing
- Lock hierarchy: synchronized → ReentrantLock → StampedLock
- Concurrent collections internals
- Non-blocking programming: CAS, ABA, AtomicReference
- Java Memory Model — reordering, barriers
- Design question: thread-safe rate limiter

---

## Q1: How does `ThreadPoolExecutor` work internally? How do you size a thread pool correctly?

### Answer

### ThreadPoolExecutor State Machine

```
submit(task)
     │
     ▼
Current threads < corePoolSize?
     │ Yes → create new thread (even if idle threads exist)
     │ No
     ▼
WorkQueue full?
     │ No → enqueue task (idle thread picks it up)
     │ Yes
     ▼
Current threads < maximumPoolSize?
     │ Yes → create new thread (temporary, above core)
     │ No
     ▼
RejectedExecutionHandler
  - AbortPolicy (default) → throws RejectedExecutionException
  - CallerRunsPolicy       → runs task on calling thread (back-pressure)
  - DiscardPolicy          → silently drops task
  - DiscardOldestPolicy    → drops oldest queued task, retries submit
```

### Constructor Parameters Explained

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    4,                              // corePoolSize: min threads kept alive
    16,                             // maximumPoolSize: max threads allowed
    60L, TimeUnit.SECONDS,          // keepAliveTime: idle non-core thread TTL
    new LinkedBlockingQueue<>(1000),// workQueue: bounded queue (CRITICAL)
    new ThreadFactory() { ... },    // threadFactory: name your threads!
    new ThreadPoolExecutor.CallerRunsPolicy()  // rejection: apply back-pressure
);

// Allow core threads to also time out (useful for bursty workloads)
executor.allowCoreThreadTimeOut(true);
```

### Why Bounded Queue is Critical

```java
// DANGEROUS — unbounded queue, OOM under load
Executors.newFixedThreadPool(10);
// Internally: new LinkedBlockingQueue<>()  ← no bound!

// Under sustained high load:
// - Queue grows without limit
// - Heap exhausted → OutOfMemoryError
// - Service dies silently

// CORRECT — bounded queue + CallerRunsPolicy
new ThreadPoolExecutor(10, 10, 0, SECONDS,
    new LinkedBlockingQueue<>(500),
    new CallerRunsPolicy());  // back-pressure: slows caller instead of crashing
```

### Thread Pool Sizing Formulas

```
CPU-bound tasks:
  threads = N_cpu + 1
  (one extra covers for occasional page faults / OS scheduling jitter)

I/O-bound tasks (blocking DB, HTTP calls):
  threads = N_cpu × (1 + wait_time / cpu_time)
  Example: 8 cores, 90% of time waiting on DB → 8 × (1 + 9) = 80 threads

Mixed workloads:
  Measure with load test → target 70-80% CPU utilization
  Start with: threads = N_cpu × 2, tune from metrics

Key insight: more threads ≠ faster for CPU-bound work
  Extra threads → context switching overhead → slower
```

### Monitoring a Thread Pool

```java
// Expose these as metrics (JMX or Micrometer)
executor.getPoolSize()          // current thread count
executor.getActiveCount()       // threads actively executing
executor.getQueue().size()      // backlog depth
executor.getCompletedTaskCount()
executor.getTaskCount()

// Alert when:
// queue.size() > 80% of capacity → scaling needed
// activeCount == maximumPoolSize → at thread limit, tasks queuing
```

---

## Q2: Explain ForkJoinPool and work-stealing. When should you use it over ThreadPoolExecutor?

### Answer

### Work-Stealing Algorithm

Standard thread pool: **single shared queue** — all threads compete for tasks → contention under load.

ForkJoinPool: **each thread has its own deque** (double-ended queue):
- Thread pushes/pops its own tasks from **its own deque head** (no contention)
- Idle threads **steal from the tail** of other threads' deques

```
Thread-0 deque:  [task4] [task3] [task2] [task1]  ← pushes/pops from left
                                              ↑
Thread-1 (idle) steals from right ───────────┘
```

### Fork/Join Pattern

```java
class SumTask extends RecursiveTask<Long> {

    private static final int THRESHOLD = 10_000;
    private final long[] array;
    private final int from;
    private final int to;

    SumTask(long[] array, int from, int to) {
        this.array = array;
        this.from = from;
        this.to = to;
    }

    @Override
    protected Long compute() {
        int size = to - from;

        if (size <= THRESHOLD) {
            // Base case: compute directly
            long sum = 0;
            for (int i = from; i < to; i++) {
                sum += array[i];
            }
            return sum;
        }

        // Divide
        int mid = from + size / 2;
        SumTask left = new SumTask(array, from, mid);
        SumTask right = new SumTask(array, mid, to);

        left.fork();              // submit left subtask asynchronously
        long rightResult = right.compute();   // compute right in current thread
        long leftResult = left.join();        // wait for left result

        return leftResult + rightResult;
    }
}

// Usage
ForkJoinPool pool = new ForkJoinPool(Runtime.getRuntime().availableProcessors());
long total = pool.invoke(new SumTask(array, 0, array.length));
```

### RecursiveTask vs RecursiveAction

| Class | Returns | Use for |
|-------|---------|---------|
| `RecursiveTask<V>` | A value | Aggregation (sum, merge sort, max) |
| `RecursiveAction` | void | Side-effects (parallel sort in-place, bulk writes) |

### ForkJoinPool vs ThreadPoolExecutor

| Aspect | ThreadPoolExecutor | ForkJoinPool |
|--------|-------------------|-------------|
| Task type | Independent, uniform | Divide-and-conquer, recursive |
| Queue structure | Shared queue | Per-thread deques |
| Blocking tasks | Works fine | Avoid — blocks worker threads, starves pool |
| Common pool | No | `ForkJoinPool.commonPool()` (shared JVM-wide) |
| Best for | I/O, service calls, independent tasks | CPU-bound divide-and-conquer, parallel streams |

### Parallel Streams Use ForkJoinPool

```java
// Uses ForkJoinPool.commonPool() — size = N_cpu - 1
list.parallelStream().map(this::compute).collect(toList());

// Override pool size for a specific operation
ForkJoinPool custom = new ForkJoinPool(8);
custom.submit(() ->
    list.parallelStream().map(this::compute).collect(toList())
).get();
```

---

## Q3: `synchronized` vs `ReentrantLock` vs `StampedLock` — when do you use each?

### Answer

### synchronized — The Baseline

```java
class Counter {
    private int count = 0;

    // Implicit monitor lock on 'this'
    synchronized void increment() {
        count++;
    }

    // Explicit block — prefer this (locks on minimum scope)
    void increment2() {
        synchronized (this) {
            count++;
        }
    }
}
```

**Characteristics:**
- JVM-managed: lock acquisition/release handled by `monitorenter`/`monitorexit` bytecodes
- Reentrant (same thread can acquire multiple times)
- No try-lock, no timeout, no interruptibility
- Bias locking (pre-Java 15), adaptive spinning, inflated to OS mutex under contention

### ReentrantLock — Explicit Lock With More Control

```java
class BoundedBuffer<T> {
    private final ReentrantLock lock = new ReentrantLock(true); // fair=true → FIFO ordering
    private final Condition notFull  = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();
    private final Queue<T> queue = new ArrayDeque<>();
    private final int capacity;

    BoundedBuffer(int capacity) { this.capacity = capacity; }

    void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity) {
                notFull.await();          // releases lock + waits
            }
            queue.add(item);
            notEmpty.signal();            // wake one waiter
        } finally {
            lock.unlock();               // ALWAYS in finally
        }
    }

    T take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await();
            }
            T item = queue.poll();
            notFull.signal();
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

**ReentrantLock extras over synchronized:**

```java
// Try-lock without blocking
if (lock.tryLock()) {
    try { /* critical section */ }
    finally { lock.unlock(); }
} else {
    // do something else — no blocking
}

// Try-lock with timeout
if (lock.tryLock(500, TimeUnit.MILLISECONDS)) { ... }

// Interruptible lock acquisition
lock.lockInterruptibly();  // throws InterruptedException if thread interrupted while waiting

// Inspect lock state
lock.isHeldByCurrentThread()
lock.getQueueLength()       // threads waiting to acquire
lock.isLocked()
```

### ReadWriteLock — Concurrent Reads, Exclusive Writes

```java
class Cache {
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock  = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();
    private final Map<String, Object> map = new HashMap<>();

    Object get(String key) {
        readLock.lock();        // multiple threads can hold simultaneously
        try {
            return map.get(key);
        } finally {
            readLock.unlock();
        }
    }

    void put(String key, Object value) {
        writeLock.lock();       // exclusive — blocks all readers + writers
        try {
            map.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }
}
```

### StampedLock — Optimistic Reads (Java 8+)

StampedLock adds an **optimistic read mode**: read without acquiring a lock, then validate. If validation fails (write happened concurrently), upgrade to a real read lock.

```java
class Point {
    private final StampedLock lock = new StampedLock();
    private double x, y;

    void move(double deltaX, double deltaY) {
        long stamp = lock.writeLock();
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            lock.unlockWrite(stamp);
        }
    }

    double distanceFromOrigin() {
        // Optimistic read — no lock acquired
        long stamp = lock.tryOptimisticRead();
        double currentX = x, currentY = y;

        if (!lock.validate(stamp)) {
            // Write happened during our read — fall back to real read lock
            stamp = lock.readLock();
            try {
                currentX = x;
                currentY = y;
            } finally {
                lock.unlockRead(stamp);
            }
        }
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }
}
```

### Decision Matrix

```
Need mutual exclusion?
├── Simple critical section, no special needs → synchronized
├── Need tryLock / timeout / interruptible?   → ReentrantLock
├── Read-heavy, few writes?
│   ├── Reads short, rarely conflict          → StampedLock (optimistic)
│   └── Reads long, writes frequent           → ReentrantReadWriteLock
└── Lock-free operations?                     → Atomic classes (see Q4)
```

---

## Q4: Explain CAS (Compare-And-Swap). What is the ABA problem and how is it fixed?

### Answer

### CAS Operation

CAS is a **single atomic CPU instruction** (`CMPXCHG` on x86):
```
CAS(memory_address, expected_value, new_value):
  if *memory_address == expected_value:
    *memory_address = new_value
    return true   // success
  else:
    return false  // someone else modified it — retry
```

No locks — threads contend at the hardware level, not the OS scheduler level. Much faster than a mutex under low-moderate contention.

### Java AtomicInteger CAS Loop

```java
AtomicInteger counter = new AtomicInteger(0);

// Under the hood (simplified):
int increment() {
    while (true) {
        int current = counter.get();          // read
        int next = current + 1;
        if (counter.compareAndSet(current, next)) {  // CAS
            return next;                      // success
        }
        // CAS failed → another thread changed the value → retry
    }
}

// Modern Java — use lambdas with updateAndGet (same CAS loop internally)
int result = counter.updateAndGet(v -> v + 1);
```

### AtomicReference — Lock-Free Data Structures

```java
// Lock-free Treiber stack
class LockFreeStack<T> {

    private final AtomicReference<Node<T>> top = new AtomicReference<>(null);

    void push(T value) {
        Node<T> newHead = new Node<>(value);
        while (true) {
            Node<T> current = top.get();
            newHead.next = current;
            if (top.compareAndSet(current, newHead)) {
                return;  // success
            }
            // retry if CAS failed
        }
    }

    T pop() {
        while (true) {
            Node<T> current = top.get();
            if (current == null) return null;
            if (top.compareAndSet(current, current.next)) {
                return current.value;
            }
        }
    }

    private static class Node<T> {
        final T value;
        Node<T> next;
        Node(T value) { this.value = value; }
    }
}
```

### The ABA Problem

```
Thread 1: reads top = Node(A) → Node(B) → Node(C)
Thread 2: pops A, pops B, pushes A back
          stack is now: Node(A) → Node(C)
Thread 1: CAS succeeds (top is still A!) → sets top = Node(B)
          BUT Node(B) was already freed/reused!
          Node(B).next still points to old Node(C) — CORRUPTION
```

**Fix: AtomicStampedReference** — includes a version counter with the reference

```java
AtomicStampedReference<Node<T>> top =
    new AtomicStampedReference<>(null, 0);

void push(T value) {
    Node<T> newHead = new Node<>(value);
    while (true) {
        int[] stampHolder = new int[1];
        Node<T> current = top.get(stampHolder);    // get ref + stamp atomically
        int currentStamp = stampHolder[0];
        newHead.next = current;
        if (top.compareAndSet(current, newHead,
                              currentStamp, currentStamp + 1)) {  // CAS with stamp
            return;
        }
    }
}
```

### CAS Limitations

| Limitation | Explanation |
|-----------|-------------|
| **ABA problem** | Value same but identity changed — use `AtomicStampedReference` |
| **Spin loop burns CPU** | Under high contention, CAS keeps failing → wasted cycles → consider `LongAdder` |
| **Only one variable** | CAS is atomic on one location — multi-variable atomicity still needs locks |

### LongAdder vs AtomicLong

```java
// AtomicLong — contention under high throughput (all threads CAS same cell)
AtomicLong counter = new AtomicLong();
counter.incrementAndGet();  // contended: multiple threads spinning on one cell

// LongAdder — stripes across cells, reduces contention
LongAdder adder = new LongAdder();
adder.increment();          // each thread updates its own cell
adder.sum();                // sums all cells at read time (slightly stale OK)

// Rule of thumb:
// High-write, low-read counters → LongAdder (metrics, request counters)
// Low-write, high-read, need exact value → AtomicLong
```

---

## Q5: Compare `ConcurrentHashMap`, `CopyOnWriteArrayList`, and `BlockingQueue`. Internal mechanics?

### Answer

### ConcurrentHashMap Internals

**Java 7**: Segment-based locking — 16 segments, each a small HashMap with its own lock.

**Java 8+**: CAS + `synchronized` on individual bucket heads only:
- **Read operations**: lock-free (volatile reads of `Node`)
- **Write on empty bucket**: CAS to insert first node
- **Write on non-empty bucket**: `synchronized` on head node only
- **Resize**: done concurrently — multiple threads cooperate, each migrating a range of buckets

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// Atomic conditional operations — use these, not get/put separately
map.putIfAbsent("key", 1);
map.computeIfAbsent("key", k -> expensiveCompute(k));  // compute only if absent
map.compute("key", (k, v) -> v == null ? 1 : v + 1);   // atomic read-modify-write
map.merge("key", 1, Integer::sum);                      // merge with existing

// WARNING: size() is approximate under concurrent modification
// For exact counting: use computeIfAbsent + AtomicLong per key
```

### CopyOnWriteArrayList Internals

Every write creates a **completely new copy** of the underlying array:
```
Write (add/set/remove):
  1. Acquire ReentrantLock
  2. Copy existing array to new array with modification
  3. Replace volatile reference to array
  4. Release lock

Read (get/iterator):
  1. Read volatile reference to array (snapshot at that moment)
  2. No lock needed — reads always see a consistent snapshot
```

```java
CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();

// Iterator is a snapshot — never throws ConcurrentModificationException
// BUT modifications during iteration are not visible in that iterator
for (String s : list) {
    // safe to iterate even while another thread modifies the list
    // modifications don't affect this iteration snapshot
}

// Use when:
//   - Reads >> Writes
//   - Iteration correctness matters more than write speed
//   - Small list (copy cost is O(n))
// Avoid for high write rate or large lists
```

### BlockingQueue Implementations

```
Producer ──► [BlockingQueue] ──► Consumer
              put() blocks if full
              take() blocks if empty
```

| Implementation | Capacity | Ordering | Use Case |
|---------------|---------|---------|---------|
| `ArrayBlockingQueue` | Bounded (fixed) | FIFO | Producer-consumer, back-pressure |
| `LinkedBlockingQueue` | Bounded or unbounded | FIFO | ThreadPoolExecutor workqueue |
| `PriorityBlockingQueue` | Unbounded | Priority | Task scheduling by priority |
| `SynchronousQueue` | 0 (no buffer) | N/A | Direct handoff — put blocks until take |
| `DelayQueue` | Unbounded | Delay expiry | Scheduled task execution |
| `LinkedTransferQueue` | Unbounded | FIFO | Producer waits until consumer takes |

```java
BlockingQueue<Task> queue = new ArrayBlockingQueue<>(100);

// Producer
queue.put(task);                              // blocks if full
boolean added = queue.offer(task, 1, SECONDS); // timeout-based

// Consumer
Task task = queue.take();                     // blocks if empty
Task task2 = queue.poll(1, SECONDS);          // timeout-based

// Drain in batch (efficient consumer)
List<Task> batch = new ArrayList<>(20);
queue.drainTo(batch, 20);                     // takes up to 20 tasks atomically
```

---

## Q6 (Design Question): Design a thread-safe rate limiter that allows N requests per second. Discuss trade-offs.

### Answer

> Verbalize trade-offs BEFORE writing code. This is what the interviewer is evaluating.

### Design Discussion (Say This First)

> "I see a few approaches — token bucket, fixed window counter, and sliding window. Let me walk through the trade-offs.
>
> **Fixed window counter** is simplest — reset a counter every second. But it has a boundary problem: a client can make 2N requests in a 1-second window spanning two periods (N at the end of period 1, N at the start of period 2).
>
> **Token bucket** is the industry standard — tokens replenish at a constant rate, burst is allowed up to bucket size. This is what most API gateways use. It's fair, handles bursts gracefully, and is O(1).
>
> **Sliding window log** is most accurate but stores per-request timestamps — O(N) memory per client.
>
> I'll implement token bucket with CAS-based lock-free updates since this is a hot path."

### Implementation — Token Bucket (Lock-Free)

```java
import java.util.concurrent.atomic.AtomicLong;

/**
 * Token bucket rate limiter.
 * Allows up to `capacity` tokens; replenishes `refillRatePerSecond` tokens/sec.
 * Lock-free: uses a single AtomicLong encoding (tokens, lastRefillNanos) via bit-packing.
 * Simpler alternative below uses two separate volatiles (slight TOCTOU but acceptable for rate limiting).
 */
public class TokenBucketRateLimiter {

    private final long capacity;
    private final double refillRatePerNano;  // tokens per nanosecond

    // Encode state as single long to enable atomic CAS:
    // Upper 32 bits = tokens (scaled), lower 32 bits = timestamp epoch seconds
    // Simpler approach: use AtomicLong for tokens + volatile for timestamp
    private volatile long availableTokens;
    private volatile long lastRefillNanos;
    private final Object lock = new Object();

    public TokenBucketRateLimiter(long capacity, long refillRatePerSecond) {
        this.capacity = capacity;
        this.refillRatePerNano = refillRatePerSecond / 1_000_000_000.0;
        this.availableTokens = capacity;
        this.lastRefillNanos = System.nanoTime();
    }

    public boolean tryAcquire() {
        return tryAcquire(1);
    }

    public boolean tryAcquire(int tokens) {
        synchronized (lock) {  // See discussion below for lock-free variant
            refill();
            if (availableTokens >= tokens) {
                availableTokens -= tokens;
                return true;
            }
            return false;
        }
    }

    private void refill() {
        long now = System.nanoTime();
        long elapsed = now - lastRefillNanos;
        long newTokens = (long) (elapsed * refillRatePerNano);
        if (newTokens > 0) {
            availableTokens = Math.min(capacity, availableTokens + newTokens);
            lastRefillNanos = now;
        }
    }
}
```

### Lock-Free Variant (CAS-Based)

```java
public class LockFreeTokenBucket {

    private final long capacity;
    private final long refillRatePerSecond;
    // Pack (tokens << 32 | lastSecond) into one long for atomic CAS
    private final AtomicLong state;

    public LockFreeTokenBucket(long capacity, long refillRatePerSecond) {
        this.capacity = capacity;
        this.refillRatePerSecond = refillRatePerSecond;
        long nowSec = System.currentTimeMillis() / 1000;
        this.state = new AtomicLong(pack(capacity, nowSec));
    }

    public boolean tryAcquire() {
        while (true) {
            long current = state.get();
            long tokens = unpackTokens(current);
            long lastSec = unpackSeconds(current);
            long nowSec = System.currentTimeMillis() / 1000;

            // Refill
            long elapsed = nowSec - lastSec;
            tokens = Math.min(capacity, tokens + elapsed * refillRatePerSecond);
            lastSec = nowSec;

            if (tokens < 1) {
                return false;  // rate limited
            }

            long newState = pack(tokens - 1, lastSec);
            if (state.compareAndSet(current, newState)) {
                return true;  // CAS success
            }
            // CAS failed → retry
        }
    }

    private long pack(long tokens, long seconds) {
        return (tokens << 32) | (seconds & 0xFFFFFFFFL);
    }

    private long unpackTokens(long state) {
        return state >>> 32;
    }

    private long unpackSeconds(long state) {
        return state & 0xFFFFFFFFL;
    }
}
```

### Distributed Rate Limiter (Bonus — Shows Senior Thinking)

> "For a single JVM, the above works. But in a microservices cluster with 10 pods, each pod has its own counter — a client hitting different pods can exceed the global limit.
>
> Solutions:
> 1. **Sticky routing** — route client to same pod (violates stateless design, complicates load balancing)
> 2. **Redis + Lua script** — atomic token bucket in Redis, all pods share one counter. The Lua script is executed atomically server-side. Adds network RTT (~1ms) per request — acceptable for API gateway, not for ultra-low-latency.
> 3. **Sliding window in Redis** — `ZADD` with score=timestamp, `ZCOUNT` for window, `ZREMRANGEBYSCORE` to expire. More memory but no boundary problem.
> 4. **Token bucket with local leeway** — each pod maintains a local bucket for a fraction (1/N) of the global limit. Approximate but low-latency."

```lua
-- Redis Lua script for atomic token bucket
-- KEYS[1] = rate limiter key
-- ARGV[1] = capacity, ARGV[2] = refill_rate, ARGV[3] = requested_tokens, ARGV[4] = now_ms
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])   -- tokens per millisecond
local requested = tonumber(ARGV[3])
local now = tonumber(ARGV[4])

local data = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(data[1]) or capacity
local last_refill = tonumber(data[2]) or now

local elapsed = now - last_refill
tokens = math.min(capacity, tokens + elapsed * refill_rate)

if tokens >= requested then
    tokens = tokens - requested
    redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
    redis.call('PEXPIRE', key, 60000)
    return 1  -- allowed
else
    redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
    return 0  -- rate limited
end
```

### Trade-Off Summary (Memorize This Table)

| Approach | Accuracy | Memory | Latency | Best For |
|----------|---------|--------|---------|---------|
| Fixed window | Low (boundary burst) | O(1) | O(1) | Simple, non-critical |
| Sliding window log | Exact | O(requests) | O(log N) | Low traffic, exact enforcement |
| Token bucket (local) | Good | O(1) | O(1) | Single instance, per-user |
| Token bucket (Redis) | Good | O(clients) | O(1)+RTT | Distributed, moderate latency ok |
| Sliding window (Redis) | Exact | O(clients×req) | O(log N)+RTT | Distributed, exact enforcement |

---

## Key Concurrency Pitfalls (Rapid-Fire)

### Deadlock

```java
// Classic deadlock pattern
synchronized (lockA) {
    synchronized (lockB) { /* ... */ }  // Thread 1: A then B
}
synchronized (lockB) {
    synchronized (lockA) { /* ... */ }  // Thread 2: B then A → DEADLOCK
}

// Fix: always acquire locks in the same global order
// Or: use tryLock with timeout and back off
```

### Livelock

Two threads keep responding to each other's actions but neither makes progress. Like two people in a hallway both stepping aside to let the other pass.

```java
// Fix: introduce randomized backoff
Thread.sleep(ThreadLocalRandom.current().nextLong(0, 100));
```

### Starvation

A thread never gets CPU time because higher-priority threads always preempt it.
Fix: use fair locks (`new ReentrantLock(true)`) or a priority-aware work queue.

### False Sharing

Two variables on the same cache line (64 bytes) cause unnecessary cache invalidation:

```java
// BAD: counter0 and counter1 likely on same cache line
long counter0 = 0;
long counter1 = 0;

// GOOD: pad to separate cache lines
@Contended  // Java 8+ — JVM adds padding automatically
long counter0 = 0;
@Contended
long counter1 = 0;
// Or use: -XX:-RestrictContended JVM flag to enable @Contended
```

---

## Key Numbers to Memorize

| Fact | Value |
|------|-------|
| `synchronized` reentrance | Yes — same thread can acquire multiple times |
| Default `ForkJoinPool.commonPool()` parallelism | `N_cpu - 1` |
| ConcurrentHashMap default concurrency level (Java 8+) | Lock per bucket head (effectively N buckets) |
| `volatile` guarantees | Visibility + ordering, NOT atomicity |
| CAS instruction on x86 | `CMPXCHG` |
| `LongAdder` vs `AtomicLong` | LongAdder wins under high write contention |
| `@Contended` padding size | 64 bytes (one cache line) |

---

## Pre-Day Checklist

- [ ] Can explain ThreadPoolExecutor state machine (queue-full → max threads → rejection)
- [ ] Can explain why unbounded queue is dangerous in `newFixedThreadPool`
- [ ] Can describe ForkJoinPool work-stealing with a diagram
- [ ] Know when to use `synchronized` vs `ReentrantLock` vs `StampedLock`
- [ ] Can explain CAS and ABA problem with fix
- [ ] Can implement a token bucket rate limiter and discuss trade-offs
- [ ] Know `LongAdder` vs `AtomicLong` trade-off

---

*Previous: [Day 1 — JVM & GC](day01_jvm_gc.md) | Next: [Day 3 — Spring Core & Boot](day03_spring_core.md)*
