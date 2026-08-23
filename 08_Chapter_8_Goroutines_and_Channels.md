# Chapter 8. Goroutines and Channels

Concurrent programming using the CSP (Communicating Sequential Processes) paradigm: lightweight goroutines, synchronous and buffered channels, pipelines, synchronization with `sync.WaitGroup`, counting semaphores for concurrency limiting, multiplexing with `select`, and broadcast cancellation patterns.

---

### Table of Contents
* [8.1. Goroutines](#81-goroutines)
* [8.2. Channels](#82-channels)
* [8.3. Pipelines and Channel Closing Rules](#83-pipelines-and-channel-closing-rules)
* [8.4. Unidirectional Channel Types](#84-unidirectional-channel-types)
* [8.5. Coordinating Concurrent Tasks with `sync.WaitGroup`](#85-coordinating-concurrent-tasks-with-syncwaitgroup)
* [8.6. Concurrency Limiting with Counting Semaphores](#86-concurrency-limiting-with-counting-semaphores)
* [8.7. Multiplexing with `select`](#87-multiplexing-with-select)
* [8.8. Broadcast Cancellation](#88-broadcast-cancellation)

---

## 8.1. Goroutines

Go embraces the **CSP model**: independent concurrent activities (goroutines) communicate by passing messages through channels rather than sharing mutable memory directly.

> *«Do not communicate by sharing memory; instead, share memory by communicating.»*

### Core Rules
* **Launching**: Use the `go` keyword before a function call:
  ```go
  go doWork() // Executes asynchronously in a new goroutine
  ```
* **Main Goroutine**: The `main()` function executes in the primary goroutine. When `main()` returns, **all background goroutines are immediately terminated**.
* **No Preemptive Kill**: Go provides no external mechanism to forcibly kill a running goroutine. A goroutine must terminate cooperatively by responding to a cancellation signal.

---

## 8.2. Channels

A channel is a thread-safe, typed conduit for transferring data and coordinating synchronization:

```go
ch := make(chan int) // Unbuffered channel
ch <- 42             // Send value to channel
v := <-ch            // Receive value from channel
close(ch)            // Close channel
```

### 1. Unbuffered Channels (Synchronous)
* `make(chan int)` — Buffer capacity is 0.
* **Rendezvous Principle**:
  * A send operation blocks until another goroutine receives from the channel.
  * A receive operation blocks until another goroutine sends to the channel.
  * Guarantees a strict **happens-before** synchronization boundary between sender and receiver.

### 2. Buffered Channels (Asynchronous)
* `make(chan string, 3)` — Buffer capacity is 3.
* Sends block **only when the buffer is full**.
* Receives block **only when the buffer is empty**.

> [!CAUTION]
> **Goroutine Leaks**:  
> If a goroutine blocks attempting to send to an unbuffered channel that no goroutine is reading from, it **remains blocked in memory permanently**. The Go garbage collector **does not** collect blocked goroutines!  
> *Solution*: Ensure proper buffer sizing for known response counts (e.g., `make(chan string, 3)`).

---

## 8.3. Pipelines and Channel Closing Rules

Goroutines are often chained into pipelines where the output channel of one stage serves as the input to the next:

```go
// Read all values until the channel is closed:
for x := range ch {
    fmt.Println(x)
}
```

### Closed Channel Behavior:
1. Writing to a closed channel $\to$ **RUNTIME PANIC**.
2. Closing an already closed channel $\to$ **RUNTIME PANIC**.
3. Reading from a closed channel: Drains remaining buffered values, then yields the type's zero value immediately without blocking.
4. Testing closure status: `val, ok := <-ch` (`ok == false` when the channel is drained and closed).

> [!IMPORTANT]
> A channel should be closed **only by the sender**, and only when the receiver must be explicitly notified that no further data will arrive.

---

## 8.4. Unidirectional Channel Types

Function signatures can restrict channel operations to enforce directionality:
* `chan<- int` — **Send-only** channel. Reading is a compile error.
* `<-chan int` — **Receive-only** channel. Writing or closing is a compile error.

```go
func generate(out chan<- int) {
    for i := 0; i < 10; i++ { out <- i }
    close(out)
}

func printValues(in <-chan int) {
    for val := range in { fmt.Println(val) }
}
```

---

## 8.5. Coordinating Concurrent Tasks with `sync.WaitGroup`

When the number of concurrent tasks is dynamic and not known up front:

```go
func processFiles(filenames []string) {
    var wg sync.WaitGroup

    for _, f := range filenames {
        wg.Add(1) // Increment counter BEFORE launching goroutine!
        go func(name string) {
            defer wg.Done() // Decrement counter on completion
            process(name)
        }(f) // Pass loop variable by value!
    }

    wg.Wait() // Block until counter returns to 0
}
```

> [!NOTE]
> **Modern note (Go 1.22+):** Passing `f` as a function parameter is no longer strictly necessary in modules declaring `go 1.22`+, because loop variables are scoped per iteration. However, for compatibility with older codebases, the pattern above remains good practice.

---

## 8.6. Concurrency Limiting with Counting Semaphores

To prevent exhausting operating system limits (such as open file descriptors or network sockets), limit concurrency with a buffered token channel:

```go
// Semaphore allowing at most 20 concurrent goroutines:
var sema = make(chan struct{}, 20)

func crawl(url string) []string {
    sema <- struct{}{}        // Acquire token (blocks if 20 are active)
    defer func() { <-sema }() // Guaranteed token release

    return fetch(url)
}
```

> [!NOTE]
> **Modern note:** In production systems, concurrency throttling is commonly structured using `golang.org/x/sync/errgroup`: `group.SetLimit(20)` + `group.Go(...)` with unified error propagation and `group.Wait()`. For acquiring $N$ units of a shared resource, `golang.org/x/sync/semaphore` provides weighted semaphores.

---

## 8.7. Multiplexing with `select`

The `select` statement enables waiting on multiple channel operations simultaneously:

```go
select {
case msg := <-ch1:
    fmt.Println("Received from ch1:", msg)
case ch2 <- outMsg:
    fmt.Println("Sent to ch2")
case <-time.After(5 * time.Second):
    fmt.Println("Timed out after 5 seconds")
default:
    fmt.Println("Non-blocking fallback: no channel ready")
}
```

### `select` Properties:
* If multiple channels are ready simultaneously, Go picks one **pseudo-randomly** to ensure fair scheduling.
* `select {}` blocks the goroutine indefinitely.
* A `nil` channel case in a `select` is never chosen (useful for dynamically disabling branches).

---

## 8.8. Broadcast Cancellation

The cleanest pattern for broadcasting a cancellation signal across any number of worker goroutines is **closing a shared `done` channel**:

```go
var done = make(chan struct{})

func cancelled() bool {
    select {
    case <-done:
        return true // Channel closed -> cancellation signal
    default:
        return false
    }
}

// Trigger cancellation across all listeners:
close(done) // All receiving goroutines unblock immediately!
```

> [!NOTE]
> **Modern note:** In modern Go, cancellation is managed using the `context` package (introduced in Go 1.7): `ctx, cancel := context.WithCancel(context.Background())`, with workers listening on `<-ctx.Done()`. This utilizes the exact same closed-channel broadcast mechanism while adding support for deadlines, timeouts (`WithTimeout`), and cross-cutting request values.
