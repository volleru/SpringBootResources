It leaks by:

    Holding references too long
    Caching without limits
    Growing objects silently

Garbage Collector can’t free what you’re still referencing.
🚨 Leak #1: Static Collections (The Classic Killer)

public class Cache {
    public static Map<String, Object> DATA = new HashMap<>();
}

Every request adds data.
Nothing removes it.
✔ GC can’t clean static references
✔ Memory grows forever
Fix

    Avoid static state
    Use bounded caches
    Prefer managed beans

🚨 Leak #2: Unbounded Caches (The Silent One)

@Cacheable("users")
public User getUser(Long id) { }

Works great.
Until:

    Millions of keys
    No eviction
    Memory spikes

Fix

    Use eviction policies
    Monitor cache size

spring.cache.caffeine.spec: maximumSize=10000

Caches must forget.
🚨 Leak #3: Hibernate Persistence Context
Hibernate remembers everything it touches in a transaction.

@Transactional
public void importData(List<Item> items) {
    items.forEach(entityManager::persist);
}

Large list → OOM 💥
Fix

entityManager.flush();
entityManager.clear();

Or use batching.
🚨 Leak #4: Listeners & Callbacks

applicationEventPublisher.publishEvent(event);

Custom listeners that:

    Hold references
    Never deregister
    Accumulate state

Memory never frees.
Fix

    Keep listeners stateless
    Avoid storing large objects
    Clean up explicitly

🚨 Leak #5: ThreadLocals (Extremely Dangerous)

ThreadLocal<UserContext> context = new ThreadLocal<>();

In thread pools:

    Threads live forever
    ThreadLocal values stay forever

Fix

try {
   context.set(value);
} finally {
   context.remove();
}

ThreadLocals must be cleaned always.
🚨 Leak #6: Logging & MDC Abuse

MDC.put("payload", bigObject.toString());

MDC uses ThreadLocal under the hood.
Large objects = long-lived memory.
Fix
✔ Keep MDC small
✔ Always clear MDC

MDC.clear();

🚨 Leak #7: Controllers Holding State

@RestController
public class UserController {
    private List<User> users = new ArrayList<>();
}

Controllers are singletons.
That list grows forever.
Fix
Controllers should be stateless.
🚨 Leak #8: Large JSON Payloads

    Huge request bodies
    Deep object graphs
    Stored in memory during processing

Under load → memory pressure.
Fix
✔ Validate payload size
✔ Stream when possible
✔ Avoid nested structures
🔍 How to Confirm a Memory Leak
Don’t guess.
✔ Watch heap usage over time
✔ Check GC frequency
✔ Look for steady growth
Tools:

    VisualVM
    JProfiler
    Heap dumps
    GC logs

🧠 Patterns That Prevent Leaks
✔ Stateless beans
✔ Bounded caches
✔ Short transactions
✔ Clean ThreadLocals
✔ Avoid static state
🎯 Final Thought
Spring Boot memory leaks are rarely bugs.
They’re design decisions that outlived their purpose.
If memory keeps growing,
something is holding on —
and now you know where to look.
