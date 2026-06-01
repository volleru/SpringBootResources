# Go (Golang) Interview Questions & Answers

Mix of **conceptual** and **coding** problems commonly asked in backend interviews. Designed for engineers with Java/backend experience picking up or refreshing Go.

---

## 🧭 Section 1 — Language Fundamentals (Conceptual)

### 1. What makes Go different from Java or Python?

| Feature | Go |
|---|---|
| Compilation | Compiled to a single static binary |
| Memory | Garbage-collected, but explicit pointers (no pointer arithmetic) |
| Concurrency | Built-in via goroutines + channels (CSP model) |
| OOP | No classes/inheritance — composition via structs + interfaces |
| Generics | Added in Go 1.18 (type parameters) |
| Exceptions | None — explicit `error` return values; `panic`/`recover` for unrecoverable failures |
| Build system | `go mod` — simple, no Maven/Gradle complexity |
| Standard library | Batteries included (HTTP, JSON, crypto, testing) |

Designed for: **simple syntax, fast compile, easy concurrency, single binary deploy.**

---

### 2. What is a goroutine? How is it different from a thread?

A goroutine is a lightweight function executed concurrently by Go's runtime.

| | OS Thread | Goroutine |
|---|---|---|
| Stack size | 1–2 MB fixed | Starts at ~2 KB, grows dynamically |
| Scheduling | OS kernel | Go runtime (user-space scheduler) |
| Creation cost | Heavy syscall | Very cheap (~µs) |
| Communication | Shared memory + locks | Channels (preferred) |

You can run **millions** of goroutines in a single process. Go's M:N scheduler maps many goroutines onto a few OS threads (`GOMAXPROCS` controls how many).

```go
go doWork()  // starts a goroutine; the main function does not wait
```

---

### 3. What is a channel?

A channel is a typed conduit for sending values between goroutines — Go's primary primitive for safe communication.

```go
ch := make(chan int)        // unbuffered
buf := make(chan int, 10)   // buffered, capacity 10

go func() { ch <- 42 }()    // send
v := <-ch                   // receive
```

- **Unbuffered**: send blocks until a receiver is ready (rendezvous).
- **Buffered**: send blocks only when buffer is full; receive blocks when empty.
- **Closing**: `close(ch)`; receivers can detect with `v, ok := <-ch`.

**Mantra:** *"Don't communicate by sharing memory; share memory by communicating."*

---

### 4. What's the difference between a slice and an array?

| | Array | Slice |
|---|---|---|
| Size | Fixed at compile time | Dynamic |
| Type | `[5]int` (size is part of type) | `[]int` |
| Passing | Pass by value (copies whole array) | Pass by reference (header: ptr, len, cap) |
| Use case | Rare | Default choice |

```go
arr := [3]int{1, 2, 3}       // array
s := []int{1, 2, 3}          // slice
s = append(s, 4)             // grow
```

A slice is a struct with `{pointer, length, capacity}` pointing to an underlying array. `append` may allocate a new backing array when capacity is exceeded.

---

### 5. How does error handling work in Go?

No exceptions. Functions return an `error` value, checked explicitly:

```go
data, err := os.ReadFile("config.yaml")
if err != nil {
    return fmt.Errorf("read config: %w", err)  // wrap with %w
}
```

- `errors.Is(err, target)` — check sentinel errors.
- `errors.As(err, &target)` — type-assert wrapped errors.
- `panic` / `recover` — only for truly unrecoverable failures (rare).

**Idiom:** check `err` immediately, return early. Don't nest happy-path code inside `if err == nil { ... }`.

---

### 6. What is the difference between `defer`, `panic`, and `recover`?

- **`defer`** — schedules a function to run when the surrounding function returns. Runs in LIFO order. Used for cleanup (closing files, unlocking).
- **`panic`** — stops normal flow, unwinds the stack, runs deferred functions.
- **`recover`** — only useful inside a deferred function; stops the panic and returns the panic value.

```go
func safe() {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("recovered: %v", r)
        }
    }()
    risky()
}
```

`defer` is also handy for tracing entry/exit, mutex unlock, etc.

---

### 7. What is the difference between `make` and `new`?

- **`new(T)`** — allocates zeroed storage and returns a `*T`.
- **`make(T, args)`** — initializes slices, maps, channels. Returns the value (not pointer).

```go
p := new(int)              // *int, points to 0
s := make([]int, 0, 10)    // slice, len=0, cap=10
m := make(map[string]int)  // map ready to use
ch := make(chan int, 5)    // buffered channel
```

For structs you usually use `&MyStruct{}` rather than `new`.

---

### 8. How do Go interfaces work?

Interfaces are **structural / implicit**. No `implements` keyword — if a type has the required methods, it satisfies the interface.

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type File struct { /* ... */ }
func (f *File) Read(p []byte) (int, error) { /* ... */ }
// *File automatically satisfies Reader — no declaration needed
```

**The empty interface** (`interface{}` or `any` since 1.18) holds any type.

Common interfaces in the stdlib: `io.Reader`, `io.Writer`, `error`, `fmt.Stringer`, `sort.Interface`.

---

### 9. What is the zero value of common types?

| Type | Zero value |
|---|---|
| `int`, `float64` | `0` |
| `string` | `""` |
| `bool` | `false` |
| `pointer`, `slice`, `map`, `chan`, `func`, `interface` | `nil` |
| `struct` | All fields set to their zero values |

This is why Go has no "uninitialized variable" bugs — every declared variable has a usable default.

---

### 10. How do you write idiomatic Go concurrency safely?

- **Prefer channels** for communication between goroutines.
- Use **`sync.Mutex`** for protecting shared state when channels don't fit (e.g., caches, counters).
- Use **`sync.WaitGroup`** to wait for multiple goroutines to finish.
- Use **`context.Context`** for cancellation and timeouts.
- Run with **`go test -race`** during dev to catch data races.
- Don't leak goroutines — every goroutine should have a clear exit path.

---

## 💻 Section 2 — Coding Problems

### 11. Worker Pool Pattern

**Problem:** Process N jobs concurrently with a fixed number of workers.

```go
func workerPool(jobs <-chan int, results chan<- int, workers int) {
    var wg sync.WaitGroup
    for w := 0; w < workers; w++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for j := range jobs {
                results <- j * 2  // example "work"
            }
        }(w)
    }
    go func() { wg.Wait(); close(results) }()
}

func main() {
    jobs := make(chan int, 100)
    results := make(chan int, 100)
    workerPool(jobs, results, 4)

    for i := 1; i <= 10; i++ { jobs <- i }
    close(jobs)

    for r := range results { fmt.Println(r) }
}
```

**Key idea:** `close(jobs)` signals workers to exit `range`; `WaitGroup` lets main know when all results have been sent before closing `results`.

---

### 12. Fan-out / Fan-in

**Problem:** Run many producers, merge their outputs into one channel.

```go
func merge(channels ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup
    wg.Add(len(channels))
    for _, c := range channels {
        go func(ch <-chan int) {
            defer wg.Done()
            for v := range ch { out <- v }
        }(c)
    }
    go func() { wg.Wait(); close(out) }()
    return out
}
```

---

### 13. Use `context.Context` for cancellation

**Problem:** Cancel an HTTP request if it takes more than 2 seconds.

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()

req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
resp, err := http.DefaultClient.Do(req)
if err != nil {
    return fmt.Errorf("request: %w", err)
}
defer resp.Body.Close()
```

`context` should be the **first parameter** of any function that does I/O or might block.

---

### 14. Reverse a String (UTF-8 safe)

```go
func reverse(s string) string {
    runes := []rune(s)
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        runes[i], runes[j] = runes[j], runes[i]
    }
    return string(runes)
}
```

**Gotcha:** iterating over `s` byte-by-byte mangles multi-byte UTF-8 characters. Always convert to `[]rune` for character-wise operations.

---

### 15. FizzBuzz in Go

```go
func fizzBuzz(n int) {
    for i := 1; i <= n; i++ {
        switch {
        case i%15 == 0: fmt.Println("FizzBuzz")
        case i%3 == 0:  fmt.Println("Fizz")
        case i%5 == 0:  fmt.Println("Buzz")
        default:        fmt.Println(i)
        }
    }
}
```

Note the **tagless `switch`** — idiomatic Go for if/else chains.

---

### 16. Two Sum

```go
func twoSum(nums []int, target int) []int {
    seen := make(map[int]int)
    for i, n := range nums {
        if j, ok := seen[target-n]; ok {
            return []int{j, i}
        }
        seen[n] = i
    }
    return nil
}
```

**Idiomatic:** comma-ok form on map lookup.

---

### 17. Implement an LRU Cache

```go
type Node struct {
    key, val   int
    prev, next *Node
}

type LRU struct {
    cap        int
    cache      map[int]*Node
    head, tail *Node  // sentinels
}

func NewLRU(cap int) *LRU {
    head, tail := &Node{}, &Node{}
    head.next, tail.prev = tail, head
    return &LRU{
        cap:   cap,
        cache: make(map[int]*Node),
        head:  head,
        tail:  tail,
    }
}

func (l *LRU) Get(key int) int {
    if n, ok := l.cache[key]; ok {
        l.moveToFront(n)
        return n.val
    }
    return -1
}

func (l *LRU) Put(key, val int) {
    if n, ok := l.cache[key]; ok {
        n.val = val
        l.moveToFront(n)
        return
    }
    if len(l.cache) == l.cap {
        lru := l.tail.prev
        l.remove(lru)
        delete(l.cache, lru.key)
    }
    n := &Node{key: key, val: val}
    l.addToFront(n)
    l.cache[key] = n
}

func (l *LRU) addToFront(n *Node) {
    n.next = l.head.next
    n.prev = l.head
    l.head.next.prev = n
    l.head.next = n
}
func (l *LRU) remove(n *Node)     { n.prev.next = n.next; n.next.prev = n.prev }
func (l *LRU) moveToFront(n *Node) { l.remove(n); l.addToFront(n) }
```

**O(1)** for both `Get` and `Put`.

---

### 18. Producer-Consumer with buffered channel

```go
func producer(ch chan<- int, n int) {
    for i := 0; i < n; i++ { ch <- i }
    close(ch)
}

func consumer(ch <-chan int, done chan<- bool) {
    for v := range ch {
        fmt.Println("consumed", v)
    }
    done <- true
}

func main() {
    ch := make(chan int, 5)
    done := make(chan bool)
    go producer(ch, 10)
    go consumer(ch, done)
    <-done
}
```

Note channel direction in function signatures (`chan<-`, `<-chan`) — Go enforces this at compile time.

---

### 19. Build a simple HTTP server

```go
func main() {
    http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        fmt.Fprintln(w, `{"status":"ok"}`)
    })

    http.HandleFunc("/users/", func(w http.ResponseWriter, r *http.Request) {
        id := strings.TrimPrefix(r.URL.Path, "/users/")
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(map[string]string{"id": id})
    })

    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

Stdlib `net/http` is production-grade — no framework needed for simple services.

---

### 20. JSON marshal/unmarshal

```go
type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email,omitempty"`
}

// Marshal
u := User{ID: 1, Name: "Charan"}
data, _ := json.Marshal(u)
// {"id":1,"name":"Charan"}

// Unmarshal
var u2 User
_ = json.Unmarshal(data, &u2)
```

Struct tags (`` `json:"..."` ``) control field names. `omitempty` skips zero-value fields.

---

## ⚙️ Section 3 — Advanced Concepts

### 21. What is `select`?

`select` waits on multiple channel operations — like a `switch` for channels.

```go
select {
case msg := <-ch1:
    fmt.Println("got", msg)
case ch2 <- 42:
    fmt.Println("sent")
case <-time.After(1 * time.Second):
    fmt.Println("timeout")
default:
    fmt.Println("nothing ready")  // non-blocking
}
```

Critical for timeouts, cancellation, multiplexing channels.

---

### 22. How does the Go scheduler work (G-P-M model)?

- **G** — goroutine
- **M** — OS thread (machine)
- **P** — logical processor (holds a runnable queue of Gs)

`GOMAXPROCS` = number of P's. The scheduler multiplexes goroutines onto OS threads. **Preemption** ensures no single goroutine hogs a thread (since Go 1.14, even tight loops can be preempted).

---

### 23. What is a data race? How do you detect it?

A data race = two goroutines accessing the same memory concurrently, at least one writing, with no synchronization. Result is undefined behavior.

Detect with: `go test -race` or `go run -race main.go`.

Fix with: `sync.Mutex`, channels, `sync/atomic`, or by not sharing the data.

---

### 24. Difference between `sync.Mutex` and `sync.RWMutex`?

- **`Mutex`** — exclusive lock (one writer or reader at a time).
- **`RWMutex`** — many readers OR one writer. Use when reads vastly outnumber writes.

```go
var mu sync.RWMutex
mu.RLock()
v := cache[key]
mu.RUnlock()
```

For high-concurrency reads with rare writes, `RWMutex` is significantly faster. For balanced workloads, plain `Mutex` is often simpler and just as fast.

---

### 25. What are generics in Go (1.18+)?

Type parameters let you write functions/types that work over multiple types without `interface{}` + type assertions.

```go
func Map[T, U any](s []T, f func(T) U) []U {
    r := make([]U, len(s))
    for i, v := range s { r[i] = f(v) }
    return r
}

doubled := Map([]int{1, 2, 3}, func(n int) int { return n * 2 })
// [2 4 6]
```

Constraints (`comparable`, `~int`, custom interfaces) restrict allowed types.

---

### 26. Pointers vs values — when to use which?

Use **pointer receivers** when:
- The method mutates the receiver.
- The struct is large (avoid copying).
- Consistency: if any method on the type is a pointer receiver, make them all pointer receivers.

Use **value receivers** when:
- The type is small and immutable-like (`time.Time`, custom IDs).
- You want copy-on-call semantics.

```go
func (u *User) SetName(n string) { u.Name = n }  // pointer — mutates
func (u User) FullName() string  { return u.Name } // value — read-only
```

---

### 27. What's the difference between `var`, `:=`, and `const`?

```go
var x int = 5         // explicit type
var y = 5             // type inferred
z := 5                // short declaration (function scope only)
const Pi = 3.14159    // compile-time constant
```

`:=` only works inside functions. At package scope, use `var`.

---

### 28. How does Go handle dependencies?

`go mod` — modules system since Go 1.11.

```bash
go mod init github.com/me/myapp     # initialize
go get github.com/gin-gonic/gin     # add dep
go mod tidy                         # clean unused
go mod vendor                       # vendor for offline builds
```

`go.mod` lists direct deps + versions; `go.sum` is the checksum lock file.

---

### 29. How do you write a unit test in Go?

Tests live alongside code, in `_test.go` files:

```go
// math.go
func Add(a, b int) int { return a + b }

// math_test.go
func TestAdd(t *testing.T) {
    got := Add(2, 3)
    if got != 5 {
        t.Errorf("Add(2,3) = %d; want 5", got)
    }
}
```

Run: `go test ./...`
- **Table-driven tests** are idiomatic.
- **`testing.T.Run`** for subtests.
- **Benchmarks** with `func BenchmarkX(b *testing.B)` and `go test -bench=.`.

---

### 30. Common Go pitfalls and idioms

| Pitfall | Fix |
|---|---|
| Range loop var captured by goroutine | Pass as parameter: `go func(v int){...}(v)` (fixed in Go 1.22 with per-iteration scoping) |
| Forgetting to close channels | Only the **sender** closes; receivers detect via `range` or `v, ok := <-ch` |
| Nil map write panic | Always `make(map[..]..)` before writing |
| Slice aliasing surprises | `append` may or may not allocate; use `copy` when you need isolation |
| Leaky goroutines | Always have an exit path — `ctx.Done()`, channel close, or bounded work |
| Ignoring errors | `_ = doSomething()` is sometimes OK, but explicit is better than silent |
| `defer` in a hot loop | Each defer has a small cost; for tight loops, unlock manually |

---

## 📝 Quick-Reference Cheat Sheet

```go
// Slice
s := []int{}
s = append(s, 1)
for i, v := range s { ... }

// Map
m := map[string]int{}
v, ok := m["key"]
delete(m, "key")

// Channel
ch := make(chan int, 10)
ch <- 1                 // send
v := <-ch               // receive
close(ch)
for v := range ch { ... }

// Goroutine + WaitGroup
var wg sync.WaitGroup
wg.Add(1)
go func() { defer wg.Done(); /* work */ }()
wg.Wait()

// Context with timeout
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()

// Error wrapping
return fmt.Errorf("load config: %w", err)
errors.Is(err, fs.ErrNotExist)
```

---

## 💡 Tips for Go Interviews (coming from Java)

1. **No exceptions** — embrace `if err != nil { return err }`. It's verbose but explicit.
2. **No inheritance** — favor composition. Embed structs, don't extend them.
3. **No constructors** — by convention, write `NewXxx(...) *Xxx` factory functions.
4. **Pointers are not Java references** — passing a struct by value copies it.
5. **Interfaces are tiny** — `io.Reader` has one method. Java-style "fat" interfaces are non-idiomatic.
6. **Concurrency is first-class** — interviewers love goroutine/channel design questions.
7. **Read `Effective Go`** — Google's official style guide. Short, dense, essential.
8. **`gofmt`** is non-negotiable — formatting is not a style debate in Go.
