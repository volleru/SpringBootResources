# Day 1 — JVM Internals, Garbage Collection & Memory Model

## Topics Covered
- JVM runtime memory regions
- GC algorithms: G1GC, ZGC, Shenandoah, ParallelGC
- Java Memory Model (JMM) and happens-before
- Classloading delegation model
- Production GC diagnosis (design question)

---

## Q1: Explain the JVM memory model. What are the different memory regions?

### Answer

The JVM divides memory into several runtime data areas:

```
┌──────────────────────────────────────────────────────────────┐
│                         JVM Memory                           │
│                                                              │
│  ┌──────────────────────┐   ┌──────────────────────────────┐ │
│  │        Heap          │   │    Non-Heap (Metaspace)      │ │
│  │                      │   │  - Class metadata            │ │
│  │  ┌────────────────┐  │   │  - Method bytecode           │ │
│  │  │   Young Gen    │  │   │  - Interned Strings          │ │
│  │  │  ┌──────────┐  │  │   │  - Static fields (Java 8+)  │ │
│  │  │  │  Eden    │  │  │   └──────────────────────────────┘ │
│  │  │  │  S0, S1  │  │  │                                    │
│  │  │  └──────────┘  │  │   ┌──────────────────────────────┐ │
│  │  └────────────────┘  │   │     Per-Thread Areas         │ │
│  │                      │   │  - PC Register               │ │
│  │  ┌────────────────┐  │   │  - JVM Stack (frames)        │ │
│  │  │   Old Gen      │  │   │    - Local variables         │ │
│  │  │   (Tenured)    │  │   │    - Operand stack           │ │
│  │  └────────────────┘  │   │    - Frame data              │ │
│  │                      │   │  - Native Method Stack       │ │
│  └──────────────────────┘   └──────────────────────────────┘ │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Code Cache (JIT compiled native)         │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

### Heap Regions

| Region | Purpose | GC Type |
|--------|---------|---------|
| **Eden** | New object allocation (bump-pointer, very fast) | Minor GC (Young GC) |
| **Survivor S0/S1** | Objects that survive Eden collection; copied back and forth | Minor GC |
| **Old Gen (Tenured)** | Objects promoted after `MaxTenuringThreshold` (default 15) GC cycles | Major/Full GC |

### Non-Heap Regions

| Region | Purpose | Size Control |
|--------|---------|-------------|
| **Metaspace** | Class metadata, method area, interned strings (replaced PermGen in Java 8) | `-XX:MaxMetaspaceSize` |
| **Code Cache** | JIT-compiled native machine code | `-XX:ReservedCodeCacheSize` |

### Per-Thread Areas (not shared)

- **PC Register**: address of currently executing JVM instruction
- **JVM Stack**: each method call creates a new stack frame; `StackOverflowError` when exhausted
- **Native Method Stack**: used when calling native (JNI) methods

### Key Flags

```bash
-Xms2g                    # initial heap size
-Xmx8g                    # max heap size
-XX:NewRatio=3            # Old:Young ratio = 3:1
-XX:SurvivorRatio=8       # Eden:Survivor ratio = 8:1
-XX:MaxTenuringThreshold=15
-XX:MaxMetaspaceSize=512m
-XX:ReservedCodeCacheSize=256m
```

---

## Q2: Compare G1GC, ZGC, and Shenandoah. When do you use each?

### Answer

### GC Algorithm Summary

| Property | SerialGC | ParallelGC | G1GC | ZGC | Shenandoah |
|----------|----------|-----------|------|-----|------------|
| STW pauses | All phases | All phases | Minor + some | Only initial/final mark | Only initial/final mark |
| Max pause target | N/A | Throughput-first | ~200ms | <10ms | <10ms |
| Heap size | Small (<4GB) | Medium | 4GB–32GB | Any (TB scale) | Medium–Large |
| CPU overhead | Lowest | Low | Moderate | Higher | Higher |
| Java version | All | All | Default since Java 9 | Production-ready Java 15+ | Java 12+ (OpenJDK) |
| Best for | Dev/test | Batch/Spark | General-purpose services | Latency-critical services | Latency-critical services |

### G1GC — How It Works

G1GC divides the heap into equal-sized **regions** (1–32MB each). It collects the regions with the most garbage first ("Garbage First").

```
Heap (G1 regions — each can be Eden, Survivor, Old, or Humongous)
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ E │ E │ S │ O │ O │ H │ E │ O │   E=Eden, S=Survivor
├───┼───┼───┼───┼───┼───┼───┼───┤   O=Old, H=Humongous (>0.5 region)
│ O │ E │ O │ O │ E │ O │ S │ O │
└───┴───┴───┴───┴───┴───┴───┴───┘
```

G1 GC phases:
1. **Young GC** — STW, collects Eden + Survivor
2. **Concurrent Marking** — concurrent, marks live objects in Old regions
3. **Mixed GC** — STW, collects Young + some Old regions
4. **Full GC** — STW (last resort), single-threaded fallback

### ZGC — How It Works

ZGC does almost everything **concurrently** while the application runs:
- Concurrent marking, relocation, and reference processing
- Uses **load barriers** (code injected at pointer dereference) to handle concurrent compaction
- Uses **colored pointers** (metadata bits in 64-bit pointers) to track object state

Pause phases (all very short): Initial Mark → Concurrent Mark → Final Mark → Concurrent Relocate

### When to Choose

```
Latency SLA?
├── > 200ms ok?   → ParallelGC (if batch) or G1GC (if service)
├── < 200ms?      → G1GC with tuning
└── < 10ms?       → ZGC or Shenandoah
                      ├── Heap > 1TB?  → ZGC
                      └── Medium heap? → Shenandoah (lower memory overhead)
```

### Key Tuning Flags

```bash
# G1GC
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200        # pause target (not a hard guarantee)
-XX:G1HeapRegionSize=16m        # larger regions = fewer humongous objects
-XX:G1NewSizePercent=5          # min Young gen size %
-XX:G1MaxNewSizePercent=60      # max Young gen size %
-XX:G1MixedGCCountTarget=8      # number of mixed GC rounds to drain old gen
-XX:G1HeapWastePercent=5        # stop mixed GC when old gen waste < 5%

# ZGC
-XX:+UseZGC
-XX:SoftMaxHeapSize=28g         # ZGC respects this as soft target
-Xmx32g

# Always enable GC logging in production
-Xlog:gc*:file=/var/log/app/gc.log:time,uptime:filecount=5,filesize=20m
```

---

## Q3: What is the Java Memory Model (JMM)? Why does `volatile` not always solve visibility problems?

### Answer

The JMM defines rules for **memory visibility and ordering** across threads. Without these rules, the JVM and CPU are free to reorder instructions and cache values in registers/CPU caches — meaning one thread may never see updates written by another.

### Happens-Before Rules (Exhaustive List)

A **happens-before** relationship means: if A happens-before B, then B is guaranteed to see all effects of A.

| Rule | What it means |
|------|--------------|
| **Program order** | Each action in a thread happens-before the next action in that same thread |
| **Monitor lock** | `unlock(m)` happens-before any subsequent `lock(m)` on the same monitor |
| **Volatile write** | Write to volatile field happens-before every subsequent read of that same field |
| **Thread start** | `thread.start()` happens-before any action in the started thread |
| **Thread join** | All actions in thread T happen-before `thread.join()` returns in another thread |
| **Transitivity** | If A hb B and B hb C, then A hb C |

### Volatile — What It Guarantees

```java
class SharedState {
    volatile boolean ready = false;  // volatile
    int value = 0;                   // NOT volatile

    // Thread A
    void writer() {
        value = 42;           // (1) write non-volatile
        ready = true;         // (2) volatile write — flushes ALL pending writes to main memory
    }

    // Thread B
    void reader() {
        while (!ready) {}     // (3) volatile read — acquires all writes visible to the volatile write
        System.out.println(value);  // (4) guaranteed to see 42
    }
}
```

Volatile guarantees:
- **Visibility**: writes flushed to main memory; reads bypass CPU cache
- **Ordering**: all writes before a volatile write are visible to all reads after the volatile read (acquire/release semantics)
- **64-bit atomicity**: `long` and `double` reads/writes are atomic on volatile fields (otherwise not guaranteed on 32-bit JVMs)

### Why Volatile Is NOT Enough

```java
// BROKEN — check-then-act is not atomic
volatile boolean initialized = false;

void init() {
    if (!initialized) {           // Thread A reads false
        // Thread B also reads false here — race!
        doExpensiveInit();
        initialized = true;
    }
}

// BROKEN — read-modify-write is not atomic
volatile int counter = 0;

void increment() {
    counter++;  // READ (0) + ADD (1) + WRITE (1)
                // Two threads both read 0, both write 1 → count stuck at 1
}
```

### Correct Alternatives

```java
// Atomic operations — CAS-based, lock-free
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();  // atomic

// Double-checked locking — correct with volatile (Java 5+)
class Singleton {
    private static volatile Singleton instance;

    static Singleton getInstance() {
        if (instance == null) {             // first check (no lock)
            synchronized (Singleton.class) {
                if (instance == null) {     // second check (with lock)
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}

// Synchronized — full mutual exclusion + visibility
synchronized void increment() {
    counter++;  // safe: only one thread at a time
}
```

---

## Q4: Explain the classloading delegation model. How do you handle classloader isolation?

### Answer

### Delegation Chain

```
Bootstrap ClassLoader
  (built into JVM, C++, loads java.*, javax.*, sun.*)
        │
        ▼
Platform ClassLoader   (Java 9+, formerly Extension CL)
  (loads jdk.*, com.sun.*, javax.* extensions)
        │
        ▼
Application ClassLoader
  (loads classpath: your JARs, -cp, CLASSPATH env)
        │
        ▼
Custom ClassLoader(s)
  (plugin systems, hot-reload, multi-tenant isolation)
```

### Delegation Algorithm (for each class request)

```
1. Check if class already loaded (cache hit → return it)
2. Delegate to parent.loadClass()
3. If parent throws ClassNotFoundException, try findClass() locally
4. If still not found, throw ClassNotFoundException
```

**Why parent-first?** Prevents user code from replacing `java.lang.String`, `java.lang.Object` etc.

### Breaking Delegation for Isolation (Child-First)

Used in: Tomcat (webapp isolation), Spring Boot fat JAR, OSGi, plugin systems.

```java
public class PluginClassLoader extends URLClassLoader {

    private static final Set<String> JDK_PREFIXES = Set.of(
        "java.", "javax.", "sun.", "com.sun.", "jdk."
    );

    public PluginClassLoader(URL[] urls, ClassLoader parent) {
        super(urls, parent);
    }

    @Override
    protected Class<?> loadClass(String name, boolean resolve)
            throws ClassNotFoundException {

        synchronized (getClassLoadingLock(name)) {
            // Always check cache first
            Class<?> cached = findLoadedClass(name);
            if (cached != null) {
                return cached;
            }

            // JDK classes must always come from parent (security requirement)
            if (JDK_PREFIXES.stream().anyMatch(name::startsWith)) {
                return super.loadClass(name, resolve);
            }

            // Child-first: try local JAR before delegating
            try {
                Class<?> found = findClass(name);  // search this ClassLoader's URLs
                if (resolve) {
                    resolveClass(found);
                }
                return found;
            } catch (ClassNotFoundException ignored) {
                // fall through to parent
            }

            return super.loadClass(name, resolve);
        }
    }
}
```

### Real-World Applications

| Use Case | ClassLoader Strategy |
|----------|---------------------|
| **Spring Boot fat JAR** | `LaunchedURLClassLoader` loads from nested JARs (`BOOT-INF/lib/`) |
| **Tomcat** | Each webapp has its own `WebappClassLoader` — prevents `commons-logging` version conflicts between apps |
| **OSGi (Equinox, Felix)** | Each bundle has its own CL; explicit `Import-Package`/`Export-Package` headers control sharing |
| **JRebel / hot-reload** | Custom CL watches filesystem; reloads changed classes without JVM restart |
| **Multi-tenant SaaS** | Per-tenant CL loads tenant-specific implementations |

### Diagnosing ClassLoader Issues

```bash
# See which ClassLoader loaded a class
System.out.println(MyClass.class.getClassLoader());

# NoClassDefFoundError vs ClassNotFoundException
# ClassNotFoundException  → class not on classpath at all
# NoClassDefFoundError    → class was on classpath at compile time but missing at runtime
#                           (often a missing transitive dependency)

# ClassCastException across ClassLoaders
# "Cannot cast Foo to Foo" — same name but loaded by different CLs
# Fix: ensure both sides use the same ClassLoader
```

---

## Q5 (Design Thinking): Production service has 3-4 second GC pauses on a 16GB heap. Walk through your diagnosis and fix.

### Answer

> This is the kind of open-ended question where showing **structured thinking** matters more than knowing the exact answer upfront. Narrate your process.

### Step 1 — Collect Data Before Guessing

```bash
# 1. Check if GC logging is enabled (if not, request a restart with it on)
-Xlog:gc*:file=/var/log/app/gc.log:time,uptime,pid:filecount=10,filesize=50m

# 2. Live heap info (no pause)
jcmd <pid> GC.heap_info
jcmd <pid> VM.native_memory summary

# 3. Thread dump (to check for lock contention alongside GC)
jcmd <pid> Thread.print > /tmp/threads.txt

# 4. GC stats summary
jstat -gcutil <pid> 1000 20   # sample every 1s, 20 times
# Output: S0 S1 E  O   M   CCS YGC YGCT FGC FGCT GCT
#          0  75 88 99  95  90  142 4.2  3   12.4  16.6
#                   ↑ Old gen 99% full → Full GC imminent/happening
```

### Step 2 — Read the GC Log Pattern

| Pattern | Root Cause |
|---------|-----------|
| `[Full GC ... 15G->14G ... 3.4s]` every few minutes | Old gen cannot be collected fast enough |
| Minor GC taking >500ms | Survivor space too small → premature promotion → Old gen fills up |
| `(Evacuation Failure)` in G1 log | Old gen full at time of Young GC — worst case |
| `(concurrent mark abort)` | Allocation faster than concurrent marking |
| Metaspace in GC log growing | Class leak (dynamic proxy generation, CGLIB) |

### Step 3 — Root Cause Categories and Fixes

```
3-4s GC pause
├── Full GC?
│   ├── Old gen retention (memory leak / large cache)
│   │   → heap dump: jcmd <pid> GC.heap_dump /tmp/heap.hprof
│   │   → analyze with Eclipse MAT or VisualVM
│   │   → look for retained objects: HttpSession, static Maps, caches
│   │
│   ├── Promotion failure (survivor space too small)
│   │   → -XX:SurvivorRatio=6 (more survivor space)
│   │   → -XX:MaxTenuringThreshold=5 (promote sooner, relieve survivor pressure)
│   │
│   └── Metaspace OOM (class leak)
│       → -XX:MaxMetaspaceSize=512m (cap it to get explicit OOM)
│       → profile with: jcmd <pid> VM.class_hierarchy | wc -l
│       → look for CGLIB$$, Proxy$, Lambda$ counts growing
│
└── Mixed GC (G1) too long?
    → -XX:G1MixedGCCountTarget=16   (spread mixed GC over more rounds)
    → -XX:G1HeapWastePercent=10     (stop mixed GC earlier)
    → -XX:MaxGCPauseMillis=100      (more aggressive pause target)
```

### Step 4 — Allocation Rate Investigation

```bash
# async-profiler — find hot allocation paths
./asprof -d 30 -e alloc -f /tmp/alloc.html <pid>

# Common culprits:
# - String concatenation in loops (use StringBuilder)
# - new ArrayList<>() without initial capacity in high-frequency code
# - Creating new objects per request that could be pooled (e.g., DateFormat, Formatter)
# - Jackson ObjectMapper created per-request (it's thread-safe, use a singleton)
```

### Step 5 — If GC Tuning Is Not Enough, Switch Collector

```bash
# Current: G1GC with 3-4s pauses
# SLA requires < 200ms

# Option 1: Tune G1GC aggressively
-XX:+UseG1GC -XX:MaxGCPauseMillis=100 -XX:G1HeapRegionSize=32m

# Option 2: Switch to ZGC (Java 15+, nearly pause-free)
-XX:+UseZGC -XX:SoftMaxHeapSize=12g -Xmx16g
# ZGC trades CPU for latency — verify CPU headroom exists first
```

### Communication Template for Interview

> "Before changing anything, I'd enable GC logging and run `jstat` to understand the pattern. 3-4s pauses typically mean Full GC, which usually means either old gen is retaining too much (memory leak or oversized cache), or allocation rate exceeds what concurrent GC can keep up with. I'd take a heap dump and analyze with MAT to find the largest retained object graphs. If it's an allocation rate problem, I'd use async-profiler to find the hot allocation paths. If tuning G1GC isn't enough and the latency SLA is strict, I'd evaluate switching to ZGC, but only after confirming we have CPU headroom since ZGC is more CPU-intensive."

---

## Key Numbers to Memorize

| Fact | Value |
|------|-------|
| Default `MaxTenuringThreshold` | 15 |
| G1GC default pause target | 200ms |
| ZGC production-ready since | Java 15 |
| Default G1 region size formula | `heap / 2048` (1MB–32MB range) |
| Minor GC frequency (healthy) | Every few seconds to minutes |
| Full GC frequency (healthy) | Never (or very rarely) |
| `jstat` sampling command | `jstat -gcutil <pid> 1000 20` |

---

## Pre-Day Checklist

- [ ] Can explain heap regions (Eden → Survivor → Old) with sizes
- [ ] Can compare G1GC vs ZGC trade-offs from memory
- [ ] Can explain happens-before with 3 concrete rules
- [ ] Can explain why `counter++` on a volatile is broken
- [ ] Can sketch the classloader delegation chain
- [ ] Know the flags: `-Xms`, `-Xmx`, `-XX:MaxMetaspaceSize`, `-XX:+UseZGC`

---

*Next: [Day 2 — Advanced Concurrency](day02_concurrency.md)*
