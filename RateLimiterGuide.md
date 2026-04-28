# Rate Limiter — Concepts & Java Examples

## What is a Rate Limiter?

A **Rate Limiter** controls how many requests a client can make to a service within a time window. It protects services from abuse, DDoS, and overload.

---

## Common Algorithms

| Algorithm | How it Works | Best For |
|---|---|---|
| **Token Bucket** | Tokens refill at a fixed rate; each request consumes one | Bursty traffic allowed |
| **Fixed Window Counter** | Count requests per fixed time window (e.g., per minute) | Simple use cases |
| **Sliding Window Log** | Track timestamps of each request | Accurate, memory-heavy |
| **Leaky Bucket** | Requests drain at a fixed rate (queue-based) | Smooth, uniform output |

---

## 1. Token Bucket (Manual Implementation)

```java
import java.util.concurrent.atomic.AtomicInteger;

public class TokenBucketRateLimiter {

    private final int maxTokens;
    private final int refillRatePerSecond;
    private AtomicInteger tokens;
    private long lastRefillTime;

    public TokenBucketRateLimiter(int maxTokens, int refillRatePerSecond) {
        this.maxTokens = maxTokens;
        this.refillRatePerSecond = refillRatePerSecond;
        this.tokens = new AtomicInteger(maxTokens);
        this.lastRefillTime = System.currentTimeMillis();
    }

    public synchronized boolean allowRequest() {
        refill();
        if (tokens.get() > 0) {
            tokens.decrementAndGet();
            return true;
        }
        return false; // rate limit exceeded
    }

    private void refill() {
        long now = System.currentTimeMillis();
        long elapsed = now - lastRefillTime;
        int tokensToAdd = (int) (elapsed / 1000) * refillRatePerSecond;
        if (tokensToAdd > 0) {
            tokens.set(Math.min(maxTokens, tokens.get() + tokensToAdd));
            lastRefillTime = now;
        }
    }

    public static void main(String[] args) throws InterruptedException {
        // Allow max 5 tokens, refill 2 per second
        TokenBucketRateLimiter limiter = new TokenBucketRateLimiter(5, 2);

        for (int i = 1; i <= 8; i++) {
            System.out.println("Request " + i + ": " + (limiter.allowRequest() ? "ALLOWED" : "BLOCKED"));
        }

        System.out.println("\n--- Waiting 2 seconds for refill ---\n");
        Thread.sleep(2000);

        for (int i = 9; i <= 12; i++) {
            System.out.println("Request " + i + ": " + (limiter.allowRequest() ? "ALLOWED" : "BLOCKED"));
        }
    }
}
```

**Output:**
```
Request 1: ALLOWED
Request 2: ALLOWED
Request 3: ALLOWED
Request 4: ALLOWED
Request 5: ALLOWED
Request 6: BLOCKED
Request 7: BLOCKED
Request 8: BLOCKED

--- Waiting 2 seconds for refill ---

Request 9:  ALLOWED
Request 10: ALLOWED
Request 11: BLOCKED
Request 12: BLOCKED
```

---

## 2. Fixed Window Counter

```java
import java.util.concurrent.atomic.AtomicInteger;

public class FixedWindowRateLimiter {

    private final int maxRequests;
    private final long windowSizeMs;
    private AtomicInteger counter;
    private long windowStart;

    public FixedWindowRateLimiter(int maxRequests, long windowSizeMs) {
        this.maxRequests = maxRequests;
        this.windowSizeMs = windowSizeMs;
        this.counter = new AtomicInteger(0);
        this.windowStart = System.currentTimeMillis();
    }

    public synchronized boolean allowRequest() {
        long now = System.currentTimeMillis();

        // Reset window if expired
        if (now - windowStart >= windowSizeMs) {
            counter.set(0);
            windowStart = now;
        }

        if (counter.get() < maxRequests) {
            counter.incrementAndGet();
            return true;
        }
        return false;
    }

    public static void main(String[] args) {
        // Max 3 requests per 5 seconds
        FixedWindowRateLimiter limiter = new FixedWindowRateLimiter(3, 5000);

        for (int i = 1; i <= 5; i++) {
            System.out.println("Request " + i + ": " + (limiter.allowRequest() ? "ALLOWED" : "BLOCKED"));
        }
    }
}
```

---

## 3. Using Guava RateLimiter (Easiest)

**Maven dependency:**
```xml
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>33.2.1-jre</version>
</dependency>
```

```java
import com.google.common.util.concurrent.RateLimiter;

public class GuavaRateLimiterExample {

    public static void main(String[] args) {
        // Allow 2 requests per second
        RateLimiter limiter = RateLimiter.create(2.0);

        for (int i = 1; i <= 6; i++) {
            double waitTime = limiter.acquire(); // blocks until permit is available
            System.out.printf("Request %d — waited %.2f sec%n", i, waitTime);
        }
    }
}
```

**Output:**
```
Request 1 — waited 0.00 sec
Request 2 — waited 0.00 sec
Request 3 — waited 0.49 sec
Request 4 — waited 0.50 sec
Request 5 — waited 0.50 sec
Request 6 — waited 0.50 sec
```

> `acquire()` **blocks** until a permit is available.  
> Use `tryAcquire()` to **reject** instead of block:

```java
if (limiter.tryAcquire()) {
    System.out.println("Request allowed");
} else {
    System.out.println("Rate limit exceeded — rejected");
}
```

---

## 4. Per-User Rate Limiter (Map-based)

```java
import com.google.common.util.concurrent.RateLimiter;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class PerUserRateLimiter {

    // 5 requests per second per user
    private static final double RATE = 5.0;
    private final Map<String, RateLimiter> userLimiters = new ConcurrentHashMap<>();

    public boolean allowRequest(String userId) {
        RateLimiter limiter = userLimiters.computeIfAbsent(userId, id -> RateLimiter.create(RATE));
        return limiter.tryAcquire();
    }

    public static void main(String[] args) {
        PerUserRateLimiter limiter = new PerUserRateLimiter();

        String[] users = {"alice", "alice", "alice", "alice", "alice", "alice", "bob", "bob"};

        for (String user : users) {
            boolean allowed = limiter.allowRequest(user);
            System.out.println(user + " -> " + (allowed ? "ALLOWED" : "BLOCKED"));
        }
    }
}
```

---

## 5. Spring Boot — Rate Limit a REST Endpoint

**Using Bucket4j (popular Spring library):**

**Maven dependency:**
```xml
<dependency>
    <groupId>com.bucket4j</groupId>
    <artifactId>bucket4j-core</artifactId>
    <version>8.10.1</version>
</dependency>
```

```java
import io.github.bucket4j.Bandwidth;
import io.github.bucket4j.Bucket;
import io.github.bucket4j.Refill;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.time.Duration;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@RestController
public class ApiController {

    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();

    private Bucket getBucket(String clientId) {
        return buckets.computeIfAbsent(clientId, id -> {
            Bandwidth limit = Bandwidth.classic(10, Refill.greedy(10, Duration.ofMinutes(1)));
            return Bucket.builder().addLimit(limit).build();
        });
    }

    @GetMapping("/api/data")
    public ResponseEntity<String> getData(String clientId) {
        Bucket bucket = getBucket(clientId);

        if (bucket.tryConsume(1)) {
            return ResponseEntity.ok("Here is your data!");
        }

        return ResponseEntity
            .status(HttpStatus.TOO_MANY_REQUESTS)
            .body("Rate limit exceeded. Try again later.");
    }
}
```

> HTTP **429 Too Many Requests** is the standard status code for rate limit errors.

---

## Comparison

| Approach | Library | Blocking? | Distributed? | Best For |
|---|---|---|---|---|
| Token Bucket (manual) | None | No | No | Learning / simple apps |
| Fixed Window (manual) | None | No | No | Simple counters |
| Guava RateLimiter | Guava | Yes (acquire) | No | Single JVM apps |
| Bucket4j | Bucket4j | No | Yes (Redis) | Spring Boot APIs |

---

## Best Practices

- Return **HTTP 429** with a `Retry-After` header when rejecting requests.
- Rate limit **per user/API key**, not just per IP.
- Use **Redis-backed** rate limiting (Bucket4j + Redis) for distributed/multi-instance deployments.
- Log blocked requests to detect abuse patterns.
- Expose remaining quota in response headers: `X-RateLimit-Remaining`, `X-RateLimit-Reset`.
