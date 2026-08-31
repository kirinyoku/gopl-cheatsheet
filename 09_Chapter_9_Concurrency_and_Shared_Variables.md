# Chapter 9. Concurrency and Shared Variables

Traditional shared-memory concurrency mechanisms: the nature of data races, mutex synchronization (`sync.Mutex`, `sync.RWMutex`), memory visibility and CPU/compiler barriers, lazy one-time initialization with `sync.Once`, the `-race` detector, concurrent non-blocking caching (Singleflight), and an architectural comparison of goroutines vs OS threads.

---

### Table of Contents

* [9.1. Data Races](#91-data-races)
* [9.2. Mutual Exclusion with `sync.Mutex`](#92-mutual-exclusion-with-syncmutex)
* [9.3. Read/Write Mutexes (`sync.RWMutex`)](#93-readwrite-mutexes-syncrwmutex)
* [9.4. Memory Visibility and CPU Barriers](#94-memory-visibility-and-cpu-barriers)
* [9.5. Lazy Initialization with `sync.Once`](#95-lazy-initialization-with-synconce)
* [9.6. The Race Detector (`-race`)](#96-the-race-detector--race)
* [9.7. Concurrent Non-Blocking Cache (Singleflight)](#97-concurrent-non-blocking-cache-singleflight)
* [9.8. Goroutines vs OS Threads](#98-goroutines-vs-os-threads)

---

## 9.1. Data Races

> [!CAUTION]
> A **Data Race** occurs whenever two or more goroutines concurrently access the same memory location, and **at least one of the accesses is a write**.

* **Concurrent Reads**: Safe.
* **Read + Write** or **Write + Write**: Cause memory corruption, corrupted invariants, or runtime panics.
* *For multi-word types* (`slice` of 3 words, `string` of 2 words, `interface` of 2 words), a data race can tear an assignment, leaving a corrupted descriptor pointing to arbitrary invalid memory.

> **Go Axiom**: *«There is no such thing as a benign data race.»* Every data race is a critical software defect.

### 3 Strategies to Prevent Data Races:

1. **Immutability**: Initialize data structures fully before launching goroutines, then treat them as read-only.
2. **Confinement**: Restrict variable access to a single monitor goroutine, interacting strictly through channel requests.
3. **Mutual Exclusion**: Allow multiple goroutines access, but strictly one at a time using mutex locks.

---

## 9.2. Mutual Exclusion with `sync.Mutex`

```go
var (
    mu      sync.Mutex
    balance int
)

func Deposit(amount int) {
    mu.Lock()
    defer mu.Unlock()
    balance += amount
}

func Balance() int {
    mu.Lock()
    defer mu.Unlock()
    return balance
}
```

### Mutex Rules:

* **Always use `defer mu.Unlock()`**: Ensures locks are released on all return paths and in the event of a panic.
* **Go Mutexes are NOT Reentrant (Non-Recursive)**: Attempting to lock a mutex that the current goroutine already holds immediately triggers a fatal **Deadlock**.
* **Function Splitting Pattern**: Separate public locked methods from private unexported helper functions (e.g., `deposit(amount)`) that assume the lock is already held.

---

## 9.3. Read/Write Mutexes (`sync.RWMutex`)

Implements the *Multiple Readers / Single Writer* concurrency pattern:

```go
var (
    mu      sync.RWMutex
    balance int
)

func Balance() int {
    mu.RLock() // Concurrent non-blocking read lock
    defer mu.RUnlock()
    return balance
}

func Deposit(amount int) {
    mu.Lock()  // Exclusive write lock
    defer mu.Unlock()
    balance += amount
}
```

> [!TIP]
> `sync.RWMutex` provides a performance advantage only when **read operations overwhelmingly dominate writes** under high concurrent contention. In simple or write-heavy scenarios, standard `sync.Mutex` is faster and less complex.

---

## 9.4. Memory Visibility and CPU Barriers

Synchronization primitives serve not just for mutual exclusion, but to enforce **memory visibility across processor cores**:
* Modern multi-core CPUs reorder memory operations and buffer writes in local CPU caches.
* Mutex operations and channel communications establish **Memory Barriers**, forcing local CPU caches to synchronize with main RAM and enforcing a strict *happens-before* ordering.

---

## 9.5. Lazy Initialization with `sync.Once`

Thread-safe one-time initialization of singletons and heavy resources without custom double-checked locking:

```go
var (
    loadIconsOnce sync.Once
    icons         map[string]image.Image
)

func Icon(name string) image.Image {
    loadIconsOnce.Do(loadIcons) // Guaranteed to execute exactly once
    return icons[name]
}
```

> [!NOTE]
> **Modern note (Go 1.21+):** Convenient standard helpers now eliminate manual `sync.Once` variable pairing: `sync.OnceFunc(f)`, `sync.OnceValue(f)` (caches return value), and `sync.OnceValues(f)` (caches value + error). Example: `var config = sync.OnceValue(loadConfig)`.

---

## 9.6. The Race Detector (`-race`)

A built-in dynamic analysis tool for finding data races at runtime based on ThreadSanitizer:

```bash
go test -race ./...
go run -race main.go
go build -race
```

* Instruments memory accesses, reporting full conflicting goroutine stack traces when an unsynchronized read/write collision is detected.

---

## 9.7. Concurrent Non-Blocking Cache (Singleflight)

An idiomatic memoization pattern: while the first goroutine computes a slow result for a key, subsequent concurrent requests for the same key block on a `ready` channel without duplicating work:

```go
// Memo fields: mu sync.Mutex; cache map[string]*entry; f func(key string) (any, error)
type entry struct {
    res   result
    ready chan struct{} // Closed when result calculation finishes
}

func (memo *Memo) Get(key string) (any, error) {
    memo.mu.Lock()
    e := memo.cache[key]
    if e == nil {
        e = &entry{ready: make(chan struct{})}
        memo.cache[key] = e
        memo.mu.Unlock()

        e.res.val, e.res.err = memo.f(key) // Compute slow result
        close(e.ready)                     // Broadcast result to all waiters
    } else {
        memo.mu.Unlock()
        <-e.ready // Wait for computation to finish
    }
    return e.res.val, e.res.err
}
```

> [!NOTE]
> **Modern note:** In real-world Go applications, use the official package `golang.org/x/sync/singleflight`: `g.Do(key, fn)` deduplicates concurrent duplicate requests and returns the shared result directly.

---

## 9.8. Goroutines vs OS Threads

| Characteristic | OS Thread | Goroutine |
| :--- | :--- | :--- |
| **Stack Size** | Fixed size (~1–2 MB) | Dynamic (starts at **2 KB**, grows up to **1 GB**) |
| **Scalability** | Thousands of threads | **Hundreds of thousands to millions** |
| **Scheduling** | OS Kernel (preemptive) | Go Runtime (**m:n scheduler**, multiplexing $M$ goroutines onto $N$ OS threads) |
| **Context Switch Overhead** | Heavy (system calls, kernel registers, CPU TLB flush) | Lightweight (~dozens of nanoseconds, user space) |
| **Identity (ID)** | Exposes `Thread ID` | **No Goroutine ID by design** |

### Why Go Has No Goroutine ID

Go intentionally omits a Goroutine ID to prevent anti-patterns like Thread-Local Storage (TLS), which allows functions to implicitly depend on hidden global state. In Go, context and dependencies are always passed **explicitly** (e.g., via `context.Context`).

> [!NOTE]
> **Modern note (Go 1.14+):** Goroutines are **asynchronously preemptible**: tight loops containing no function calls can no longer starve the scheduler or delay garbage collection cycles.
