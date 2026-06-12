# Java LLD & Concurrency Interview Problems

> The "build a working component" round — what you get asked once you clear the
> syntax screen. **RateLimiter, LRUCache, and Producer-Consumer** are the three
> canonical openers; this file takes them as the base and fans out to the full
> family of problems interviewers pull from the same well (caching, rate
> limiting, concurrency primitives, blocking queues, classic thread puzzles,
> and object-design LLD).
>
> For each problem: **what they're really testing**, the **approach**, a
> **complete, compilable Java solution**, **complexity**, and the **follow-ups**
> they escalate to. Read the "what they're testing" line first — it tells you
> which trade-off to talk out loud.

---

## How to attack any LLD/concurrency question

1. **Clarify the contract** — capacity? eviction policy? blocking vs non-blocking? single or multi-threaded? per-user or global?
2. **State the data structures** before coding. "HashMap for O(1) lookup + doubly linked list for O(1) eviction."
3. **Name the concurrency strategy** — `synchronized`, `ReentrantLock`, `ConcurrentHashMap`, lock-free CAS, or `BlockingQueue`. Say *why*.
4. **Identify the critical section** — the smallest region that must be atomic. Holding a lock too long kills throughput.
5. **Discuss correctness hazards** — race conditions, deadlock, livelock, starvation, lost wakeups, stale reads.
6. **Mention production concerns** — metrics, eviction logging, memory bounds, what happens under overload.

---

## Table of Contents

**Caching**
1. [LRU Cache](#1-lru-cache)
2. [LFU Cache](#2-lfu-cache)
3. [Cache with TTL / Expiry](#3-cache-with-ttl--expiry)

**Rate Limiting**
4. [Token Bucket Rate Limiter](#4-token-bucket-rate-limiter)
5. [Sliding Window Log Rate Limiter](#5-sliding-window-log-rate-limiter)
6. [Fixed Window Counter](#6-fixed-window-counter)
7. [Sliding Window Counter](#7-sliding-window-counter)
8. [Leaky Bucket Rate Limiter](#8-leaky-bucket-rate-limiter)

**Producer–Consumer & Queues**
9. [Producer–Consumer with BlockingQueue](#9-producerconsumer-with-blockingqueue)
10. [Producer–Consumer with wait/notify](#10-producerconsumer-with-waitnotify)
11. [Bounded Blocking Queue (from scratch)](#11-bounded-blocking-queue-from-scratch)

**Concurrency primitives (build them yourself)**
12. [Custom Semaphore](#12-custom-semaphore)
13. [Custom CountDownLatch](#13-custom-countdownlatch)
14. [Custom Thread Pool](#14-custom-thread-pool)
15. [Read-Write Lock](#15-read-write-lock)

**Classic thread puzzles (LeetCode concurrency)**
16. [Print in Order](#16-print-in-order)
17. [Print FooBar Alternately](#17-print-foobar-alternately)
18. [Print Zero-Even-Odd](#18-print-zero-even-odd)
19. [Dining Philosophers](#19-dining-philosophers)

**Object-design LLD**
20. [Thread-safe Singleton](#20-thread-safe-singleton)
21. [Object / Connection Pool](#21-object--connection-pool)
22. [Unique ID Generator (Snowflake)](#22-unique-id-generator-snowflake)
23. [Min Stack](#23-min-stack)
24. [Stack using Queues / Queue using Stacks](#24-stack-using-queues--queue-using-stacks)
25. [Logger Rate Limiter / Hit Counter](#25-logger-rate-limiter--hit-counter)
26. [Circuit Breaker](#26-circuit-breaker)

---

# Caching

## 1. LRU Cache

**What they're testing:** can you get **both** `get` and `put` to O(1)? Naive answers use a list scan (O(n)). The trick is HashMap + doubly linked list.

**Approach:** `HashMap<K, Node>` for O(1) lookup. A doubly linked list orders nodes by recency — head = most recently used, tail = least. On access/insert, move the node to the head; on overflow, evict the tail.

```java
import java.util.*;

public class LRUCache<K, V> {

    private static final class Node<K, V> {
        K key;
        V value;
        Node<K, V> prev, next;
        Node(K key, V value) { this.key = key; this.value = value; }
    }

    private final int capacity;
    private final Map<K, Node<K, V>> map = new HashMap<>();
    private final Node<K, V> head = new Node<>(null, null); // dummy
    private final Node<K, V> tail = new Node<>(null, null); // dummy

    public LRUCache(int capacity) {
        this.capacity = capacity;
        head.next = tail;
        tail.prev = head;
    }

    public V get(K key) {
        Node<K, V> node = map.get(key);
        if (node == null) {
            return null;
        }
        moveToFront(node);
        return node.value;
    }

    public void put(K key, V value) {
        Node<K, V> node = map.get(key);
        if (node != null) {
            node.value = value;
            moveToFront(node);
            return;
        }
        if (map.size() == capacity) {
            Node<K, V> lru = tail.prev;
            remove(lru);
            map.remove(lru.key);
        }
        Node<K, V> fresh = new Node<>(key, value);
        map.put(key, fresh);
        addToFront(fresh);
    }

    private void moveToFront(Node<K, V> node) {
        remove(node);
        addToFront(node);
    }

    private void addToFront(Node<K, V> node) {
        node.next = head.next;
        node.prev = head;
        head.next.prev = node;
        head.next = node;
    }

    private void remove(Node<K, V> node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }
}
```
- **Time:** O(1) for both `get` and `put`. **Space:** O(capacity).
- **Dummy head/tail** nodes remove all null-checks at the boundaries — say this out loud, interviewers love it.

**The "I know the shortcut" answer** — `LinkedHashMap` with access order does LRU in 6 lines:
```java
import java.util.*;

public class LRUCacheSimple<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;
    public LRUCacheSimple(int capacity) {
        super(capacity, 0.75f, true); // accessOrder = true
        this.capacity = capacity;
    }
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }
}
```
> Show the manual version first (proves you understand it), then mention this. **Follow-up:** "make it thread-safe" → wrap with `Collections.synchronizedMap`, or use a `ReentrantLock` around the manual version, or partition by key.

---

## 2. LFU Cache

**What they're testing:** eviction by **frequency**, not recency. Harder than LRU because you must track counts *and* break ties by recency among equal-frequency items, all in O(1).

**Approach:** three maps —
- `values`: key → value
- `counts`: key → access frequency
- `freqLists`: frequency → `LinkedHashSet` of keys at that frequency (insertion order = recency tiebreak).
- track `minFreq` to find the eviction candidate in O(1).

```java
import java.util.*;

public class LFUCache {
    private final int capacity;
    private int minFreq = 0;
    private final Map<Integer, Integer> values = new HashMap<>();
    private final Map<Integer, Integer> counts = new HashMap<>();
    private final Map<Integer, LinkedHashSet<Integer>> freqLists = new HashMap<>();

    public LFUCache(int capacity) {
        this.capacity = capacity;
    }

    public int get(int key) {
        if (!values.containsKey(key)) {
            return -1;
        }
        touch(key);
        return values.get(key);
    }

    public void put(int key, int value) {
        if (capacity == 0) {
            return;
        }
        if (values.containsKey(key)) {
            values.put(key, value);
            touch(key);
            return;
        }
        if (values.size() == capacity) {
            LinkedHashSet<Integer> minSet = freqLists.get(minFreq);
            int evict = minSet.iterator().next(); // oldest at min freq
            minSet.remove(evict);
            values.remove(evict);
            counts.remove(evict);
        }
        values.put(key, value);
        counts.put(key, 1);
        freqLists.computeIfAbsent(1, f -> new LinkedHashSet<>()).add(key);
        minFreq = 1;
    }

    private void touch(int key) {
        int freq = counts.get(key);
        counts.put(key, freq + 1);
        freqLists.get(freq).remove(key);
        if (freq == minFreq && freqLists.get(freq).isEmpty()) {
            minFreq++;
        }
        freqLists.computeIfAbsent(freq + 1, f -> new LinkedHashSet<>()).add(key);
    }
}
```
- **Time:** O(1) for `get`/`put`. **Space:** O(capacity).
- **Key insight:** `LinkedHashSet` gives both O(1) membership *and* insertion-order iteration — that's what makes the LRU tiebreak within a frequency bucket O(1).

---

## 3. Cache with TTL / Expiry

**What they're testing:** lazy vs active expiration. Do you delete on read, or run a sweeper thread?

**Approach (lazy + thread-safe):** store value with an expiry timestamp; evict on access if expired. Optionally add a background sweeper.

```java
import java.util.concurrent.*;

public class TTLCache<K, V> {

    private static final class Entry<V> {
        final V value;
        final long expiryAt; // epoch millis
        Entry(V value, long expiryAt) { this.value = value; this.expiryAt = expiryAt; }
    }

    private final ConcurrentHashMap<K, Entry<V>> map = new ConcurrentHashMap<>();

    public void put(K key, V value, long ttlMillis) {
        map.put(key, new Entry<>(value, System.currentTimeMillis() + ttlMillis));
    }

    public V get(K key) {
        Entry<V> e = map.get(key);
        if (e == null) {
            return null;
        }
        if (System.currentTimeMillis() >= e.expiryAt) {
            map.remove(key, e);   // remove only if unchanged (avoid racing a fresh put)
            return null;
        }
        return e.value;
    }

    /** Optional active sweeper to reclaim memory from never-read expired keys. */
    public void startSweeper(ScheduledExecutorService scheduler, long periodMillis) {
        scheduler.scheduleAtFixedRate(() -> {
            long now = System.currentTimeMillis();
            for (K key : map.keySet()) {
                Entry<V> e = map.get(key);
                if (e != null && now >= e.expiryAt) {
                    map.remove(key, e);
                }
            }
        }, periodMillis, periodMillis, TimeUnit.MILLISECONDS);
    }
}
```
- **`map.remove(key, e)`** (the 2-arg form) is a CAS — it only removes if the mapping still points to the *same* expired entry, so a concurrent fresh `put` isn't clobbered. This is the subtle correctness point.
- **Follow-up:** "evict the soonest-to-expire first under memory pressure" → add a `PriorityQueue`/`DelayQueue` keyed by expiry.

---

# Rate Limiting

> Know the four canonical algorithms and their trade-offs cold — interviewers
> almost always ask "which would you pick and why."

| Algorithm | Memory | Burst handling | Boundary spike issue |
|---|---|---|---|
| Token Bucket | O(1) per key | Allows bursts up to bucket size | No |
| Leaky Bucket | O(1) per key | Smooths to constant rate | No |
| Fixed Window | O(1) per key | Cheap | **Yes** — 2× burst at window edge |
| Sliding Window Log | O(n) per key | Exact | No |
| Sliding Window Counter | O(1) per key | Approximate, good | Mostly mitigated |

## 4. Token Bucket Rate Limiter

**What they're testing:** lazy refill (don't spawn a thread per bucket) and integer/time arithmetic under concurrency.

**Approach:** bucket holds tokens; refill *lazily* on each request based on elapsed time. A request consumes one token if available.

```java
public class TokenBucketRateLimiter {
    private final long capacity;
    private final double refillPerMilli; // tokens added per millisecond
    private double tokens;
    private long lastRefillTs;

    public TokenBucketRateLimiter(long capacity, double refillTokensPerSecond) {
        this.capacity = capacity;
        this.refillPerMilli = refillTokensPerSecond / 1000.0;
        this.tokens = capacity;
        this.lastRefillTs = System.currentTimeMillis();
    }

    public synchronized boolean allowRequest() {
        refill();
        if (tokens >= 1) {
            tokens -= 1;
            return true;
        }
        return false;
    }

    private void refill() {
        long now = System.currentTimeMillis();
        double added = (now - lastRefillTs) * refillPerMilli;
        if (added > 0) {
            tokens = Math.min(capacity, tokens + added);
            lastRefillTs = now;
        }
    }
}
```
- **Time:** O(1) per request. **Lazy refill** is the whole trick — no background thread needed.
- **Per-user:** keep a `ConcurrentHashMap<UserId, TokenBucketRateLimiter>`.
- **Follow-up:** "make it lock-free" → use `AtomicLong` packing tokens+timestamp and a CAS loop.

---

## 5. Sliding Window Log Rate Limiter

**What they're testing:** exactness vs memory. This is the precise-but-expensive option.

**Approach:** store the timestamp of every request in a deque; evict timestamps older than the window; allow if the remaining count < limit.

```java
import java.util.*;

public class SlidingWindowLogRateLimiter {
    private final int maxRequests;
    private final long windowMillis;
    private final Deque<Long> timestamps = new ArrayDeque<>();

    public SlidingWindowLogRateLimiter(int maxRequests, long windowMillis) {
        this.maxRequests = maxRequests;
        this.windowMillis = windowMillis;
    }

    public synchronized boolean allowRequest() {
        long now = System.currentTimeMillis();
        long boundary = now - windowMillis;
        while (!timestamps.isEmpty() && timestamps.peekFirst() <= boundary) {
            timestamps.pollFirst();
        }
        if (timestamps.size() < maxRequests) {
            timestamps.addLast(now);
            return true;
        }
        return false;
    }
}
```
- **Time:** amortized O(1). **Space:** O(maxRequests) per key — the downside; at high QPS this is a lot of timestamps.

---

## 6. Fixed Window Counter

**What they're testing:** awareness of the **boundary burst** flaw.

**Approach:** count requests per fixed time window; reset the counter when the window rolls over.

```java
public class FixedWindowRateLimiter {
    private final int maxRequests;
    private final long windowMillis;
    private long windowStart;
    private int count;

    public FixedWindowRateLimiter(int maxRequests, long windowMillis) {
        this.maxRequests = maxRequests;
        this.windowMillis = windowMillis;
        this.windowStart = System.currentTimeMillis();
    }

    public synchronized boolean allowRequest() {
        long now = System.currentTimeMillis();
        if (now - windowStart >= windowMillis) {
            windowStart = now;
            count = 0;
        }
        if (count < maxRequests) {
            count++;
            return true;
        }
        return false;
    }
}
```
- **The flaw to call out:** with limit=5/min, 5 requests at 0:59 and 5 at 1:01 = **10 requests in 2 seconds**. Fixed by the sliding window counter below.

---

## 7. Sliding Window Counter

**What they're testing:** the O(1) approximation that fixes the boundary spike — the answer most production systems actually use (e.g., Cloudflare).

**Approach:** keep counts for the current and previous window; weight the previous window by how much of it still overlaps the sliding window.

```java
public class SlidingWindowCounterRateLimiter {
    private final int maxRequests;
    private final long windowMillis;
    private long currentWindowStart;
    private int currentCount;
    private int previousCount;

    public SlidingWindowCounterRateLimiter(int maxRequests, long windowMillis) {
        this.maxRequests = maxRequests;
        this.windowMillis = windowMillis;
        this.currentWindowStart = System.currentTimeMillis();
    }

    public synchronized boolean allowRequest() {
        long now = System.currentTimeMillis();
        long elapsed = now - currentWindowStart;
        if (elapsed >= windowMillis) {
            if (elapsed >= 2 * windowMillis) {
                previousCount = 0;
            } else {
                previousCount = currentCount;
            }
            currentCount = 0;
            currentWindowStart = now;
            elapsed = 0;
        }
        double overlap = (double) (windowMillis - elapsed) / windowMillis;
        double estimated = previousCount * overlap + currentCount;
        if (estimated < maxRequests) {
            currentCount++;
            return true;
        }
        return false;
    }
}
```
- **Time/Space:** O(1) both — best of both worlds, with a small approximation error.

---

## 8. Leaky Bucket Rate Limiter

**What they're testing:** the difference between *smoothing* (leaky) and *bursting* (token). Leaky bucket enforces a constant outflow rate.

**Approach:** requests enter a fixed-size queue; a steady drain processes them at a constant rate. If the queue is full, reject.

```java
public class LeakyBucketRateLimiter {
    private final long capacity;
    private final double leakPerMilli;
    private double water;        // current queue level
    private long lastLeakTs;

    public LeakyBucketRateLimiter(long capacity, double leakPerSecond) {
        this.capacity = capacity;
        this.leakPerMilli = leakPerSecond / 1000.0;
        this.lastLeakTs = System.currentTimeMillis();
    }

    public synchronized boolean allowRequest() {
        leak();
        if (water + 1 <= capacity) {
            water += 1;
            return true;
        }
        return false;
    }

    private void leak() {
        long now = System.currentTimeMillis();
        double leaked = (now - lastLeakTs) * leakPerMilli;
        if (leaked > 0) {
            water = Math.max(0, water - leaked);
            lastLeakTs = now;
        }
    }
}
```
- **Token vs Leaky:** token bucket lets traffic burst then refill; leaky bucket forces a steady drip regardless of arrival pattern.

---

# Producer–Consumer & Queues

## 9. Producer–Consumer with BlockingQueue

**What they're testing:** do you reach for the right concurrent collection instead of hand-rolling locks? `BlockingQueue.put`/`take` block automatically when full/empty.

```java
import java.util.concurrent.*;

public class ProducerConsumerBlockingQueue {

    public static void main(String[] args) {
        BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(10);

        Runnable producer = () -> {
            try {
                for (int i = 0; i < 50; i++) {
                    queue.put(i);                  // blocks if full
                    System.out.println("Produced " + i);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        };

        Runnable consumer = () -> {
            try {
                while (true) {
                    Integer item = queue.take();   // blocks if empty
                    System.out.println("Consumed " + item);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        };

        new Thread(producer).start();
        new Thread(consumer).start();
    }
}
```
- **Always restore the interrupt flag** (`Thread.currentThread().interrupt()`) when catching `InterruptedException` — a classic interviewer gotcha.
- **Follow-up:** "graceful shutdown" → use a poison-pill sentinel object, or `poll(timeout)` + a `volatile boolean running` flag.

---

## 10. Producer–Consumer with wait/notify

**What they're testing:** the low-level mechanics `BlockingQueue` hides. Expect this when they say "without using `java.util.concurrent`."

**Three non-negotiables:** (1) wait inside a `while` loop, not `if` (guards against spurious wakeups + lost-wakeup races); (2) `notifyAll`, not `notify`; (3) all access inside `synchronized`.

```java
import java.util.*;

public class ProducerConsumerWaitNotify {
    private final Queue<Integer> buffer = new LinkedList<>();
    private final int capacity;

    public ProducerConsumerWaitNotify(int capacity) {
        this.capacity = capacity;
    }

    public void produce(int item) throws InterruptedException {
        synchronized (this) {
            while (buffer.size() == capacity) {
                wait();                 // release lock, wait for space
            }
            buffer.add(item);
            notifyAll();                // wake any waiting consumers
        }
    }

    public int consume() throws InterruptedException {
        synchronized (this) {
            while (buffer.isEmpty()) {
                wait();                 // release lock, wait for an item
            }
            int item = buffer.poll();
            notifyAll();                // wake any waiting producers
            return item;
        }
    }
}
```
- **Why `while` not `if`:** after waking, the condition may no longer hold (another thread grabbed the slot). Re-check.
- **Why `notifyAll` not `notify`:** with mixed waiters (producers + consumers on the same monitor), `notify` might wake the wrong kind and stall forever.

---

## 11. Bounded Blocking Queue (from scratch)

**What they're testing:** building `ArrayBlockingQueue` yourself with `ReentrantLock` + two `Condition`s — separate conditions avoid waking threads that can't proceed.

```java
import java.util.*;
import java.util.concurrent.locks.*;

public class BoundedBlockingQueue<T> {
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();

    public BoundedBlockingQueue(int capacity) {
        this.capacity = capacity;
    }

    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity) {
                notFull.await();
            }
            queue.add(item);
            notEmpty.signal();          // wake exactly one consumer
        } finally {
            lock.unlock();
        }
    }

    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await();
            }
            T item = queue.poll();
            notFull.signal();           // wake exactly one producer
        return item;
        } finally {
            lock.unlock();
        }
    }
}
```
- **Why two Conditions beat one monitor:** `notFull.signal()` wakes *only* a producer, so you can use `signal()` (one thread) instead of `signalAll()` — far less thundering-herd contention.
- **`unlock()` in `finally`** — always, or a thrown exception leaks the lock.

---

# Concurrency Primitives (build them yourself)

## 12. Custom Semaphore

**What they're testing:** the counting-permit pattern under the hood.

```java
public class CustomSemaphore {
    private int permits;

    public CustomSemaphore(int permits) {
        this.permits = permits;
    }

    public synchronized void acquire() throws InterruptedException {
        while (permits == 0) {
            wait();
        }
        permits--;
    }

    public synchronized void release() {
        permits++;
        notify();   // one waiter can now proceed
    }
}
```
- A mutex is just a semaphore with 1 permit. Mention that.

---

## 13. Custom CountDownLatch

**What they're testing:** one-shot "wait for N events" — a latch never resets (that's `CyclicBarrier`).

```java
public class CustomCountDownLatch {
    private int count;

    public CustomCountDownLatch(int count) {
        this.count = count;
    }

    public synchronized void await() throws InterruptedException {
        while (count > 0) {
            wait();
        }
    }

    public synchronized void countDown() {
        if (count > 0) {
            count--;
            if (count == 0) {
                notifyAll();   // release everyone waiting
            }
        }
    }
}
```
- **Latch vs Barrier:** latch = wait for N *events*, one-shot. Barrier = N *threads* wait for each other, reusable.

---

## 14. Custom Thread Pool

**What they're testing:** the core of `ThreadPoolExecutor` — N worker threads draining a shared blocking task queue.

```java
import java.util.concurrent.*;

public class CustomThreadPool {
    private final BlockingQueue<Runnable> taskQueue;
    private final Thread[] workers;
    private volatile boolean running = true;

    public CustomThreadPool(int poolSize, int queueCapacity) {
        this.taskQueue = new LinkedBlockingQueue<>(queueCapacity);
        this.workers = new Thread[poolSize];
        for (int i = 0; i < poolSize; i++) {
            workers[i] = new Worker();
            workers[i].start();
        }
    }

    public void submit(Runnable task) throws InterruptedException {
        if (!running) {
            throw new IllegalStateException("Pool is shut down");
        }
        taskQueue.put(task);   // blocks if the queue is full (backpressure)
    }

    public void shutdown() {
        running = false;
        for (Thread w : workers) {
            w.interrupt();
        }
    }

    private final class Worker extends Thread {
        @Override
        public void run() {
            while (running || !taskQueue.isEmpty()) {
                try {
                    Runnable task = taskQueue.poll(1, TimeUnit.SECONDS);
                    if (task != null) {
                        task.run();
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                } catch (RuntimeException e) {
                    // never let a bad task kill the worker thread
                    System.err.println("Task failed: " + e.getMessage());
                }
            }
        }
    }
}
```
- **The catch-RuntimeException point is gold:** an uncaught exception in `task.run()` would kill the worker thread permanently, silently shrinking your pool. Always isolate.
- **`poll(timeout)` not `take()`** so workers wake periodically to observe `running == false` during shutdown.

---

## 15. Read-Write Lock

**What they're testing:** allow concurrent readers but exclusive writers — and avoid writer starvation.

```java
public class SimpleReadWriteLock {
    private int readers = 0;
    private boolean writer = false;

    public synchronized void lockRead() throws InterruptedException {
        while (writer) {            // wait while a writer holds/wants the lock
            wait();
        }
        readers++;
    }

    public synchronized void unlockRead() {
        readers--;
        if (readers == 0) {
            notifyAll();
        }
    }

    public synchronized void lockWrite() throws InterruptedException {
        while (writer || readers > 0) {
            wait();
        }
        writer = true;
    }

    public synchronized void unlockWrite() {
        writer = false;
        notifyAll();
    }
}
```
- **Use case:** read-heavy caches/config. In production use `ReentrantReadWriteLock` or `StampedLock` (optimistic reads). **Follow-up:** "writers are starving" → add a `waitingWriters` counter and make readers yield when a writer is queued.

---

# Classic Thread Puzzles (LeetCode-style)

> These are pure synchronization exercises — small code, big on correctness.

## 16. Print in Order

> Three threads call `first()`, `second()`, `third()`; force the output order regardless of scheduling.

```java
import java.util.concurrent.*;

public class PrintInOrder {
    private final Semaphore s2 = new Semaphore(0);
    private final Semaphore s3 = new Semaphore(0);

    public void first(Runnable printFirst) {
        printFirst.run();
        s2.release();
    }

    public void second(Runnable printSecond) throws InterruptedException {
        s2.acquire();
        printSecond.run();
        s3.release();
    }

    public void third(Runnable printThird) throws InterruptedException {
        s3.acquire();
        printThird.run();
    }
}
```
- **Pattern:** semaphores as a relay baton. Each stage releases the next.

---

## 17. Print FooBar Alternately

> Two threads must print `foobarfoobar...` alternately, n times each.

```java
import java.util.concurrent.*;

public class FooBar {
    private final int n;
    private final Semaphore fooLock = new Semaphore(1); // foo goes first
    private final Semaphore barLock = new Semaphore(0);

    public FooBar(int n) { this.n = n; }

    public void foo(Runnable printFoo) throws InterruptedException {
        for (int i = 0; i < n; i++) {
            fooLock.acquire();
            printFoo.run();
            barLock.release();
        }
    }

    public void bar(Runnable printBar) throws InterruptedException {
        for (int i = 0; i < n; i++) {
            barLock.acquire();
            printBar.run();
            fooLock.release();
        }
    }
}
```
- **Two semaphores ping-ponging** — `foo` starts with 1 permit, `bar` with 0; each hands off to the other.

---

## 18. Print Zero-Even-Odd

> Thread A prints 0, thread B prints evens, thread C prints odds → `0102030405...`

```java
import java.util.concurrent.*;
import java.util.function.IntConsumer;

public class ZeroEvenOdd {
    private final int n;
    private final Semaphore zero = new Semaphore(1);
    private final Semaphore even = new Semaphore(0);
    private final Semaphore odd = new Semaphore(0);

    public ZeroEvenOdd(int n) { this.n = n; }

    public void zero(IntConsumer printNumber) throws InterruptedException {
        for (int i = 1; i <= n; i++) {
            zero.acquire();
            printNumber.accept(0);
            if (i % 2 == 1) {
                odd.release();
            } else {
                even.release();
            }
        }
    }

    public void even(IntConsumer printNumber) throws InterruptedException {
        for (int i = 2; i <= n; i += 2) {
            even.acquire();
            printNumber.accept(i);
            zero.release();
        }
    }

    public void odd(IntConsumer printNumber) throws InterruptedException {
        for (int i = 1; i <= n; i += 2) {
            odd.acquire();
            printNumber.accept(i);
            zero.release();
        }
    }
}
```
- **The `zero` thread is the conductor** — it prints 0 then routes the baton to even or odd based on parity.

---

## 19. Dining Philosophers

**What they're testing:** **deadlock avoidance**. Five philosophers, five forks; each needs both neighbors' forks. Naive "grab left then right" deadlocks.

**Fix:** impose a global lock-ordering — everyone picks up the lower-numbered fork first. (Alternatives: limit to 4 seated at once via a semaphore; have one philosopher pick up right-first.)

```java
import java.util.concurrent.locks.*;

public class DiningPhilosophers {
    private final ReentrantLock[] forks = new ReentrantLock[5];

    public DiningPhilosophers() {
        for (int i = 0; i < 5; i++) {
            forks[i] = new ReentrantLock();
        }
    }

    public void wantsToEat(int philosopher, Runnable eat) throws InterruptedException {
        int left = philosopher;
        int right = (philosopher + 1) % 5;
        int first = Math.min(left, right);   // always lock lower index first
        int second = Math.max(left, right);

        forks[first].lock();
        try {
            forks[second].lock();
            try {
                eat.run();
            } finally {
                forks[second].unlock();
            }
        } finally {
            forks[first].unlock();
        }
    }
}
```
- **Deadlock's 4 conditions:** mutual exclusion, hold-and-wait, no preemption, circular wait. **Lock ordering breaks circular wait** — the cleanest fix. Name the four conditions to the interviewer.

---

# Object-Design LLD

## 20. Thread-safe Singleton

**What they're testing:** do you know **why** double-checked locking needs `volatile`, and the cleaner alternatives.

```java
// (a) Initialization-on-demand holder — lazy, thread-safe, no synchronization cost. BEST.
public class SingletonHolder {
    private SingletonHolder() {}
    private static final class Holder {
        static final SingletonHolder INSTANCE = new SingletonHolder();
    }
    public static SingletonHolder getInstance() {
        return Holder.INSTANCE;   // JVM guarantees class-init is thread-safe & lazy
    }
}

// (b) Double-checked locking — the classic "do you know volatile?" answer.
public class SingletonDCL {
    private static volatile SingletonDCL instance;  // volatile is MANDATORY
    private SingletonDCL() {}
    public static SingletonDCL getInstance() {
        if (instance == null) {                     // 1st check (no lock)
            synchronized (SingletonDCL.class) {
                if (instance == null) {             // 2nd check (locked)
                    instance = new SingletonDCL();
                }
            }
        }
        return instance;
    }
}

// (c) Enum — simplest, serialization- and reflection-safe (Effective Java item 3).
public enum SingletonEnum {
    INSTANCE;
    public void doWork() { /* ... */ }
}
```
- **Why `volatile` in DCL:** `new SingletonDCL()` is *not* atomic — allocate, construct, assign reference. Without `volatile`, another thread can see a non-null but **half-constructed** object due to instruction reordering.

---

## 21. Object / Connection Pool

**What they're testing:** bounded resource reuse with blocking acquire — a `Semaphore` + a thread-safe pool.

```java
import java.util.concurrent.*;

public class ConnectionPool<T> {
    private final BlockingQueue<T> pool;

    public ConnectionPool(int size, ConnectionFactory<T> factory) {
        this.pool = new ArrayBlockingQueue<>(size);
        for (int i = 0; i < size; i++) {
            pool.offer(factory.create());
        }
    }

    public T acquire(long timeout, TimeUnit unit) throws InterruptedException {
        T conn = pool.poll(timeout, unit);
        if (conn == null) {
            throw new RuntimeException("Timed out waiting for a connection");
        }
        return conn;
    }

    public void release(T conn) {
        if (conn != null) {
            pool.offer(conn);   // return to pool for reuse
        }
    }

    public interface ConnectionFactory<T> {
        T create();
    }
}
```
- **`BlockingQueue` is the pool** — `poll(timeout)` blocks callers when exhausted instead of creating unbounded connections. **Follow-up:** validate connections on release; evict stale ones.

---

## 22. Unique ID Generator (Snowflake)

**What they're testing:** generating sortable, unique 64-bit IDs across distributed nodes without a central coordinator.

**Layout:** `[1 unused][41 bits timestamp][10 bits machine id][12 bits sequence]` — ~4096 IDs/ms/node.

```java
public class SnowflakeIdGenerator {
    private static final long EPOCH = 1_700_000_000_000L; // custom epoch
    private static final long MACHINE_BITS = 10L;
    private static final long SEQUENCE_BITS = 12L;
    private static final long MAX_MACHINE = (1L << MACHINE_BITS) - 1;
    private static final long MAX_SEQUENCE = (1L << SEQUENCE_BITS) - 1;

    private final long machineId;
    private long lastTimestamp = -1L;
    private long sequence = 0L;

    public SnowflakeIdGenerator(long machineId) {
        if (machineId < 0 || machineId > MAX_MACHINE) {
            throw new IllegalArgumentException("machineId out of range");
        }
        this.machineId = machineId;
    }

    public synchronized long nextId() {
        long now = System.currentTimeMillis();
        if (now < lastTimestamp) {
            throw new IllegalStateException("Clock moved backwards");
        }
        if (now == lastTimestamp) {
            sequence = (sequence + 1) & MAX_SEQUENCE;
            if (sequence == 0) {            // sequence exhausted this ms
                now = waitNextMillis(now);
            }
        } else {
            sequence = 0L;
        }
        lastTimestamp = now;
        return ((now - EPOCH) << (MACHINE_BITS + SEQUENCE_BITS))
                | (machineId << SEQUENCE_BITS)
                | sequence;
    }

    private long waitNextMillis(long now) {
        while (now <= lastTimestamp) {
            now = System.currentTimeMillis();
        }
        return now;
    }
}
```
- **Why not UUID:** UUIDs are random → terrible as DB primary keys (no locality, index fragmentation). Snowflake IDs are time-sortable.

---

## 23. Min Stack

**What they're testing:** O(1) `min()` alongside push/pop. The trick is a second stack tracking minimums.

```java
import java.util.*;

public class MinStack {
    private final Deque<Integer> stack = new ArrayDeque<>();
    private final Deque<Integer> mins = new ArrayDeque<>();

    public void push(int x) {
        stack.push(x);
        if (mins.isEmpty() || x <= mins.peek()) {
            mins.push(x);
        }
    }

    public void pop() {
        int popped = stack.pop();
        if (popped == mins.peek()) {
            mins.pop();
        }
    }

    public int top() {
        return stack.peek();
    }

    public int getMin() {
        return mins.peek();
    }
}
```
- **`x <= mins.peek()`** (not `<`) handles duplicate minimums correctly — push to `mins` on ties so pop stays balanced.
- **Time:** O(1) for all operations.

---

## 24. Stack using Queues / Queue using Stacks

**What they're testing:** understanding the LIFO↔FIFO conversion cost.

```java
import java.util.*;

// Queue using two stacks — amortized O(1) dequeue.
public class QueueUsingStacks<T> {
    private final Deque<T> in = new ArrayDeque<>();
    private final Deque<T> out = new ArrayDeque<>();

    public void enqueue(T x) {
        in.push(x);
    }

    public T dequeue() {
        if (out.isEmpty()) {
            while (!in.isEmpty()) {
                out.push(in.pop());   // reverse once; amortized O(1)
            }
        }
        return out.pop();
    }
}
```
- **Amortized O(1):** each element moves from `in` to `out` exactly once. **Follow-up:** Stack-using-queues makes `push` O(n) (rotate the queue) but keeps `pop` O(1) — the cost just moves.

---

## 25. Logger Rate Limiter / Hit Counter

**What they're testing:** "print a message only if not seen in the last 10 seconds" (LeetCode 359) and "count hits in the last 5 minutes."

```java
import java.util.*;

// Logger: allow a message at most once per 10 seconds.
public class Logger {
    private final Map<String, Integer> lastPrinted = new HashMap<>();

    public boolean shouldPrint(int timestamp, String message) {
        Integer last = lastPrinted.get(message);
        if (last == null || timestamp - last >= 10) {
            lastPrinted.put(message, timestamp);
            return true;
        }
        return false;
    }
}

// Hit counter: number of hits in the trailing 5-minute (300s) window.
class HitCounter {
    private final Deque<Integer> hits = new ArrayDeque<>();

    public void hit(int timestamp) {
        hits.addLast(timestamp);
    }

    public int getHits(int timestamp) {
        while (!hits.isEmpty() && timestamp - hits.peekFirst() >= 300) {
            hits.pollFirst();
        }
        return hits.size();
    }
}
```

---

## 26. Circuit Breaker

**What they're testing:** the resilience pattern — stop hammering a failing downstream, then probe for recovery. Three states: **CLOSED** (normal), **OPEN** (failing, reject fast), **HALF_OPEN** (probing).

```java
public class CircuitBreaker {
    private enum State { CLOSED, OPEN, HALF_OPEN }

    private final int failureThreshold;
    private final long openMillis;       // how long to stay OPEN before probing
    private State state = State.CLOSED;
    private int failureCount = 0;
    private long openedAt = 0;

    public CircuitBreaker(int failureThreshold, long openMillis) {
        this.failureThreshold = failureThreshold;
        this.openMillis = openMillis;
    }

    public synchronized boolean allowRequest() {
        if (state == State.OPEN) {
            if (System.currentTimeMillis() - openedAt >= openMillis) {
                state = State.HALF_OPEN;   // time to test the waters
                return true;
            }
            return false;                  // fail fast
        }
        return true;                       // CLOSED or HALF_OPEN
    }

    public synchronized void recordSuccess() {
        failureCount = 0;
        state = State.CLOSED;
    }

    public synchronized void recordFailure() {
        failureCount++;
        if (state == State.HALF_OPEN || failureCount >= failureThreshold) {
            state = State.OPEN;
            openedAt = System.currentTimeMillis();
        }
    }
}
```
- **The state machine is the answer.** A single failure in HALF_OPEN trips straight back to OPEN; a success resets to CLOSED. Mention Resilience4j / Hystrix as the real-world libraries.

---

# Cheat-sheet: which tool for which job

| Need | Reach for |
|---|---|
| O(1) cache with eviction | HashMap + doubly linked list (or `LinkedHashMap`) |
| Bounded handoff between threads | `ArrayBlockingQueue` / `LinkedBlockingQueue` |
| Wait for N tasks to finish | `CountDownLatch` |
| N threads rendezvous repeatedly | `CyclicBarrier` |
| Limit concurrent access to a resource | `Semaphore` |
| Read-heavy shared state | `ReentrantReadWriteLock` / `StampedLock` |
| Lock-free counter | `AtomicLong` / `LongAdder` |
| Thread-safe map | `ConcurrentHashMap` |
| Schedule delayed/periodic work | `ScheduledExecutorService` / `DelayQueue` |
| Producer rate control | Token bucket (bursty) / Leaky bucket (smooth) |

---

## Universal follow-ups they will ask

1. **"Make it thread-safe."** — Identify the critical section; pick the cheapest correct lock. Don't synchronize the whole method if a small block suffices.
2. **"Make it distributed."** — Move state to Redis (e.g., rate-limit counters with `INCR`+`EXPIRE`), or use consistent hashing to shard.
3. **"What about memory?"** — Bound every collection. Unbounded queues/caches are how you OOM in prod.
4. **"How do you test concurrency?"** — Many threads hammering shared state, assert invariants; tools like `jcstress`. Acknowledge that races are non-deterministic and timing-based tests are flaky.
5. **"What metrics would you emit?"** — hit/miss ratio (cache), allow/reject counts (rate limiter), queue depth, latency percentiles, breaker state transitions.

---

*These problems reward calm narration over speed. Talk through the data structures
and the concurrency hazard before you type — that's what separates a hire from a
pass.*
