# Chapter 1. Tutorial

An overview of basic Go syntax and conventions: running programs, control flow, stream I/O, strings and collections, concurrency fundamentals (goroutines and channels), and building a concurrent web server.

---

### Table of Contents
* [1.1. Hello, World: Program Structure and Execution](#11-hello-world-program-structure-and-execution)
* [1.2. Command-Line Arguments and Loops](#12-command-line-arguments-and-loops)
* [1.3. Finding Duplicate Lines: Files, Streams, and Maps](#13-finding-duplicate-lines-files-streams-and-maps)
* [1.4. Animated GIFs and the `io.Writer` Interface](#14-animated-gifs-and-the-iowriter-interface)
* [1.5. Fetching a URL: Basic HTTP Client](#15-fetching-a-url-basic-http-client)
* [1.6. Fetching URLs Concurrently: Goroutines and Channels](#16-fetching-urls-concurrently-goroutines-and-channels)
* [1.7. A Minimal Web Server and Data Race Protection](#17-a-minimal-web-server-and-data-race-protection)
* [1.8. Loose Ends: Language Constructs and Syntax Highlights](#18-loose-ends-language-constructs-and-syntax-highlights)

---

## 1.1. Hello, World: Program Structure and Execution

A minimal runnable Go program:

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, world!")
}
```

### Key Rules
* **Entry point**: An executable program always belongs to `package main` and defines an entry function `func main()`. Any other package name is compiled as a reusable library.
* **Compiler strictness**: Unused imports and unused local variables produce compile-time errors.
* **Semicolons `;`**: Automatically inserted by the lexer at line endings according to syntactic rules. As a result, the opening curly brace `{` **must always be on the same line**:
  ```go
  func main() { // Correct
  }

  func main()   // Compile error: lexer inserts ';' after main()
  {
  }
  ```
* **Code formatting**: All Go code is formatted with the standard `gofmt` (or `goimports`) tool. In the Go ecosystem, stylistic disputes over indentation and spacing are eliminated by design.

### CLI Commands
* `go run main.go` — Compiles source into a temporary directory (with build caching) and immediately executes the binary.
* `go build main.go` — Compiles a standalone, statically linked executable with no external runtime dependencies.

> [!NOTE]
> **Modern note:** The book was published in 2015 (Go 1.5), prior to Go Modules. Today, simply run `go mod init <module-name>` once in the project root, after which `go run` and `go build` work out of the box. The legacy `GOPATH` workspace mechanism is no longer used.

---

## 1.2. Command-Line Arguments and Loops

### Reading Arguments (`os.Args`)
`os.Args` is a slice of strings (`[]string`) containing the command-line arguments passed to the program:
* `os.Args[0]` — Name/path of the executing binary.
* `os.Args[1:]` — Slice of user-supplied arguments.

### Go's Only Loop Keyword: `for`
Go has no `while` or `do-while` loops. Every form of iteration is expressed using `for`:

```go
// 1. Traditional three-component loop
for i := 1; i < len(os.Args); i++ {
    fmt.Println(os.Args[i])
}

// 2. While-style loop (condition only)
for condition {
    // loop body
}

// 3. Infinite loop
for {
    // exit via break or return
}

// 4. Iterating over collections (range)
for idx, val := range os.Args[1:] {
    fmt.Printf("%d: %s\n", idx, val)
}
```

### Blank Identifier `_`
When the index or value from `range` is not needed, discard it using `_` to satisfy the compiler's unused-variable check:
```go
for _, arg := range os.Args[1:] {
    fmt.Println(arg)
}
```

> [!TIP]
> **String Concatenation Performance**: Repeatedly appending strings via `s += " "` in a loop is inefficient ($O(n^2)$), because Go strings are immutable and each concatenation allocates a new memory block.  
> For joins, use `strings.Join(os.Args[1:], " ")` or build strings efficiently using `bytes.Buffer` / `strings.Builder`.

> [!NOTE]
> **Modern note (Go 1.22+):**
> * Range over integers: `for i := range 10` is supported directly, replacing `for i := 0; i < 10; i++`.
> * Loop variables are now scoped per-iteration by default (when using `go 1.22+` module directive). In earlier versions, loop variables were shared across iterations, causing subtle closure capture bugs (see Chapter 5).

---

## 1.3. Finding Duplicate Lines: Files, Streams, and Maps

### Hash Tables (`map`)
* Initialization: `counts := make(map[string]int)`
* **Default zero values**: Accessing a non-existent key returns the zero value for the value type (`0` for numbers, `""` for strings, `false` for booleans). Direct in-place incrementing is therefore safe:
  ```go
  counts[line]++ // Safe even if line is not yet in the map
  ```
* Iteration order with `for key, val := range counts` is **intentionally randomized** to prevent code from relying on stable map ordering.

### Streaming Input with `bufio.Scanner`
Reads arbitrarily large input line-by-line without loading entire files into memory:

```go
scanner := bufio.NewScanner(os.Stdin) // or a file opened with os.Open()
for scanner.Scan() {
    line := scanner.Text()
    counts[line]++
}
if err := scanner.Err(); err != nil {
    fmt.Fprintf(os.Stderr, "Read error: %v\n", err)
}
```

### Reading Entire Small Files
* `os.ReadFile("file.txt")` returns `([]byte, error)` — reads the whole file into a byte slice, which can be split using `strings.Split(string(data), "\n")`.

> [!NOTE]
> Go avoids exception hierarchies (`try / catch / throw`). Functions report errors as explicit return values `(Result, error)`, checked immediately: `if err != nil { ... }`.

> [!NOTE]
> **Modern note (Go 1.16+):** The `io/ioutil` package is deprecated. Functions like `ioutil.ReadFile`, `ioutil.ReadAll`, and `ioutil.Discard` have been replaced by `os.ReadFile`, `io.ReadAll`, and `io.Discard`.

---

## 1.4. Animated GIFs and the `io.Writer` Interface

Generating Lissajous figures as GIF streams demonstrates the universality of Go's I/O interfaces:

```go
func lissajous(out io.Writer) {
    // Generate frames and encode stream
    gif.EncodeAll(out, &anim)
}
```

* The `io.Writer` interface abstracts the destination: `lissajous` works identically whether bytes are written to a local file (`*os.File`), standard output (`os.Stdout`), or an HTTP response body (`http.ResponseWriter`).

---

## 1.5. Fetching a URL: Basic HTTP Client

A minimal HTTP GET request using `net/http`:

```go
resp, err := http.Get("https://example.com")
if err != nil {
    log.Fatalf("HTTP request failed: %v", err)
}
defer resp.Body.Close() // CRITICAL: ensures connection socket is released

// Efficient streaming to stdout without buffer allocations:
_, err = io.Copy(os.Stdout, resp.Body)
```

> [!IMPORTANT]
> Always call `resp.Body.Close()`. Failing to close the response body causes socket leaks and prevents HTTP connection reuse (keep-alive pooling).

---

## 1.6. Fetching URLs Concurrently: Goroutines and Channels

Go was designed from the ground up for high-concurrency network servers.

```go
func fetch(url string, ch chan<- string) {
    start := time.Now()
    resp, err := http.Get(url)
    if err != nil {
        ch <- fmt.Sprint(err) // Send error to channel
        return
    }
    nbytes, _ := io.Copy(io.Discard, resp.Body)
    resp.Body.Close()
    secs := time.Since(start).Seconds()
    ch <- fmt.Sprintf("%.2fs  %7d  %s", secs, nbytes, url)
}

func main() {
    ch := make(chan string)
    for _, url := range os.Args[1:] {
        go fetch(url, ch) // Launch each fetch concurrently
    }
    for range os.Args[1:] {
        fmt.Println(<-ch) // Read results sequentially as they complete
    }
}
```

### Core Principles
1. **Goroutine (`go f()`)**: A lightweight user-space thread managed by the Go runtime. Spawning one takes fractions of a microsecond and requires only ~2 KB of initial memory.
2. **Channel (`chan`)**: A typed conduit for synchronization and safe communication between goroutines.
   * `ch <- val` — Send (blocks until received on an unbuffered channel).
   * `val := <-ch` — Receive (blocks until a value is sent).
3. **Synchronization**: Instead of concurrent goroutines writing directly to `os.Stdout`, results are communicated back through the channel and printed sequentially in `main`.

> [!NOTE]
> **Modern note:** While ~2 KB remains the baseline minimum stack size, since Go 1.19 initial stack sizes can be dynamically tuned based on historical allocation patterns.

---

## 1.7. A Minimal Web Server and Data Race Protection

In Go's standard `http.Server`, each incoming HTTP request is dispatched in **its own goroutine**.

```go
var (
    mu    sync.Mutex
    count int
)

func main() {
    http.HandleFunc("/", handler)
    http.HandleFunc("/count", counter)
    log.Fatal(http.ListenAndServe("localhost:8000", nil))
}

func handler(w http.ResponseWriter, r *http.Request) {
    mu.Lock()
    count++
    mu.Unlock()
    fmt.Fprintf(w, "URL.Path = %q\n", r.URL.Path)
}

func counter(w http.ResponseWriter, r *http.Request) {
    mu.Lock()
    fmt.Fprintf(w, "Count: %d\n", count)
    mu.Unlock()
}
```

> [!WARNING]
> Because handlers execute concurrently, unprotected mutations like `count++` cause **Data Races**. Critical sections must be guarded using synchronization primitives such as `sync.Mutex`.

---

## 1.8. Loose Ends: Language Constructs and Syntax Highlights

* **`switch` Without Fallthrough**:
  Go's `switch` does not fall through automatically; explicit `break` statements are unnecessary. To explicitly fall through to the next case, use `fallthrough`.
  ```go
  switch {
  case x > 0:
      return +1
  case x < 0:
      return -1
  default:
      return 0
  }
  ```
* **Initialization Statements in `if`**:
  Short-lived scope variables can be scoped directly inside an `if` header:
  ```go
  if err := r.ParseForm(); err != nil {
      log.Println(err)
  }
  ```
* **Pointers (`&` and `*`)**:
  `&x` takes the memory address of a variable; `*p` dereferences it. Go has **no pointer arithmetic** (`p++` is prohibited), guaranteeing memory safety. Low-level conversions require explicit `unsafe.Pointer` usage (see Chapter 13).
