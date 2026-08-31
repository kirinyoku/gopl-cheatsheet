# Chapter 5. Functions

A comprehensive guide to functions in Go: signatures and call semantics, the dynamic growable call stack, multiple and named return values, 5 error-handling strategies, first-class functions, closures and loop variable capture traps, variadic arguments, deferred cleanup (`defer`), and runtime failure handling via `panic` and `recover`.

---

### Table of Contents

* [5.1. Function Declarations and Call Semantics](#51-function-declarations-and-call-semantics)
* [5.2. Recursion and the Dynamic Stack](#52-recursion-and-the-dynamic-stack)
* [5.3. Multiple Return Values and Bare Returns](#53-multiple-return-values-and-bare-returns)
* [5.4. Error-Handling Strategies](#54-error-handling-strategies)
* [5.5. Anonymous Functions and Closures](#55-anonymous-functions-and-closures)
* [5.6. Variadic Functions (`...`)](#56-variadic-functions)
* [5.7. Deferred Function Calls (`defer`)](#57-deferred-function-calls-defer)
* [5.8. Panic and Recover](#58-panic-and-recover)

---

## 5.1. Function Declarations and Call Semantics

```go
func name(parameterList) (resultList) {
    body
}
```

### Core Rules

* **Signature (Function Type)**: Defined solely by parameter types and return types (e.g., `func(int, int) bool`). Parameter names do not affect the signature type.
* **No Default Values**: Go has **no default parameter values** and **no named keyword arguments** (`f(x=1)`). All arguments are supplied explicitly and positionally.
* **Strict Pass-by-Value**: Functions always receive **copies** of arguments.
  * For reference-like descriptor types (pointers `*T`, slices `[]T`, maps `map`, channels `chan`, functions `func`), the small descriptor header is copied by value, allowing the function to modify the underlying data.

---

## 5.2. Recursion and the Dynamic Stack

* Go functions can call themselves recursively, making them natural for traversing hierarchical tree structures (HTML DOMs, ASTs, graph networks).
* **The Dynamic Call Stack**:
  * In conventional languages (C, C++, Java), thread call stacks have a fixed size (typically 1–8 MB), causing deep recursion to trigger fatal `Stack Overflow` errors.
  * In Go, each goroutine starts with a tiny stack (~2 KB) that **dynamically grows and shrinks** on demand up to a configurable ceiling (`debug.SetMaxStack`, defaulting to ~1 GB on 64-bit platforms). Deep recursion is inherently safe and practical in Go.

---

## 5.3. Multiple Return Values and Bare Returns

Functions can return multiple values (the standard idiom being a result alongside an error):

```go
func findLinks(url string) ([]string, error) {
    resp, err := http.Get(url)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    var links []string
    // ... parse resp.Body and populate links ...
    return links, nil
}
```

### Named Result Values and Bare Returns

Return values can be named directly in the signature, initializing them to their type's zero value:

```go
func Count(url string) (words, images int, err error) {
    // words = 0, images = 0, err = nil
    words = 10
    images = 2
    return // Bare return: equivalent to return words, images, err
}
```

> [!TIP]
> Naming return values is excellent for documenting public APIs. However, avoid "bare returns" in long functions, as omitting explicit return expressions obscures data flow and harms readability.

---

## 5.4. Error-Handling Strategies

Go rejects exception hierarchies for routine errors. Errors are standard values implementing the built-in `error` interface (`nil` indicates success; non-`nil` indicates failure).

### 5 Idiomatic Error-Handling Strategies:

1. **Propagation with Context**:
   ```go
   doc, err := html.Parse(resp.Body)
   if err != nil {
       return nil, fmt.Errorf("parsing %s as HTML: %v", url, err)
   }
   ```
   *Error messages start with lowercase letters and omit trailing punctuation*, allowing chained context messages to read as natural, contiguous sentences.
2. **Retry with Exponential Backoff**: For transient network or database failures.
3. **Fatal Process Exit**: `log.Fatalf(...)` or `os.Exit(1)`. Allowed **only in `main`** (entry point). Libraries must never terminate the host process directly.
4. **Log and Continue**: `log.Printf("non-fatal: %v", err)`.
5. **Intentional Silence**: Discarding errors only when guaranteed safe (e.g., `os.RemoveAll(tmpDir)`).

### The `io.EOF` Sentinel Marker

To distinguish clean stream termination from network/disk I/O failures, the `io.EOF` sentinel error is used:

```go
for {
    r, _, err := in.ReadRune()
    if err == io.EOF {
        break // End of stream reached cleanly
    }
    if err != nil {
        return err // Actual I/O failure
    }
}
```

> [!NOTE]
> **Modern note (Go 1.13+):** Use the `%w` format verb for wrapping errors: `fmt.Errorf("reading %s: %w", url, err)`. Wrapped errors preserve the causal chain and are inspected with `errors.Is(err, io.EOF)` or `errors.As(err, &target)` instead of direct `==` comparisons.

---

## 5.5. Anonymous Functions and Closures

Functions in Go are first-class values: they can be assigned to variables, passed as arguments, and returned from other functions.

```go
func squares() func() int {
    var x int
    return func() int {
        x++
        return x * x
    }
}

f := squares()
fmt.Println(f()) // 1
fmt.Println(f()) // 4
fmt.Println(f()) // 9
```

* An anonymous function forms a **closure**: it captures references to outer variables (`x`), which the compiler automatically promotes to heap storage to maintain state across calls.

> [!CAUTION]
> **Loop Variable Capture in Closures**:  
> All closures created inside a loop share the **exact same memory address** of the loop variable!
>
> ```go
> // BUG: When executed later, dir will hold the value of the final iteration!
> for _, dir := range dirs {
>     tasks = append(tasks, func() { os.RemoveAll(dir) })
> }
>
> // FIX: Create an explicit local copy per iteration
> for _, dir := range dirs {
>     dir := dir // Local copy
>     tasks = append(tasks, func() { os.RemoveAll(dir) })
> }
> ```

> [!NOTE]
> **Modern note (Go 1.22+):** Modules declaring `go 1.22` or higher scope loop variables per iteration automatically, making both code variants work correctly. The `dir := dir` pattern is needed only when targeting legacy Go versions.

---

## 5.6. Variadic Functions (`...`)

Variadic functions accept a variable number of trailing arguments, received inside the function as a slice:

```go
func sum(vals ...int) int {
    total := 0
    for _, val := range vals {
        total += val
    }
    return total
}

sum(1, 2, 3)     // 6
nums := []int{1, 2, 3, 4}
sum(nums...)     // Unpack slice via trailing '...'
```

---

## 5.7. Deferred Function Calls (`defer`)

The `defer` statement schedules a function call to execute **immediately prior to exiting the surrounding function** (whether via normal `return` or during unwinding from a `panic`):

```go
func ReadFile(filename string) ([]byte, error) {
    f, err := os.Open(filename)
    if err != nil {
        return nil, err
    }
    defer f.Close() // Guaranteed cleanup on any return path
    return io.ReadAll(f)
}
```

### `defer` Execution Rules:

1. **LIFO Stack Order**: Multiple deferred statements execute in reverse order (the last deferred call runs first).
2. **Argument Evaluation Timing**: Arguments to deferred functions are evaluated **at the moment the `defer` line executes**, not when the function body eventually runs.
3. **Mutating Named Return Values**: A deferred closure can inspect and modify named return values before they are returned to the caller.

> [!WARNING]
> **`defer` Inside Loops**:  
> In long-running or infinite loops, `defer f.Close()` **does not execute until the entire enclosing function returns**, risking file descriptor exhaustion.  
> *Solution*: Wrap the loop body inside a dedicated helper function.

---

## 5.8. Panic and Recover

### Runtime Panics (`panic`)

* A `panic` halts normal execution immediately, unwinds the stack executing all deferred functions in the current goroutine, and terminates the program with a stack trace.
* **Usage**: Reserved for unrecoverable programmer bugs (out-of-bounds index, nil dereference, broken internal invariants) or `Must...` initialization helpers (`regexp.MustCompile`). Never use `panic` for expected I/O errors!

### Graceful Recovery (`recover`)

`recover()` intercepts an in-flight panic, halts stack unwinding, and returns the value supplied to `panic`:

```go
func Parse(input string) (syntax *Syntax, err error) {
    defer func() {
        if p := recover(); p != nil {
            err = fmt.Errorf("internal parser panic: %v", p)
        }
    }()
    // Parsing code that might panic on malformed AST
    return parseInternal(input), nil
}
```

> [!IMPORTANT]
> 1. `recover()` works **strictly inside deferred functions**. Called elsewhere, it simply returns `nil`.
> 2. Libraries and public packages **must never let panics escape across public package boundaries**: internal panics should be recovered and converted into structured `error` returns.
