# The Go Programming Language — Notes & Reference

<p align="center">
  Quick reference for the book <b>«The Go Programming Language»</b> (Alan A. A. Donovan, Brian W. Kernighan)
</p>

<p align="center">
  <a href="ru/README.md">На русском</a>
</p>

---

> [!NOTE]
> ### About
> Concise notes and cheat sheet based on **«The Go Programming Language»** by Alan A. A. Donovan and Brian W. Kernighan.
>
> Designed for quick lookup of syntax, idioms, standard library packages, runtime details, and common pitfalls. For complete explanations and in-depth discussions, refer to the original book.

---

## 📚 Chapters

* [**Chapter 1. Tutorial (Introduction to Go)**](01_Chapter_1_Tutorial.md)  
  *Program structure, `for` loops, command-line arguments (`os.Args`), `map` data structures, streaming I/O with `io.Writer`, HTTP client, first goroutines, channels, and a minimalist web server with mutexes.*

* [**Chapter 2. Program Structure**](02_Chapter_2_Program_Structure.md)  
  *Naming conventions and export rules (Public/Private), Zero Values, pointers, compiler escape analysis (stack vs heap), named types, package `init()` ordering, scope, and variable shadowing pitfalls (`:=`).*

* [**Chapter 3. Basic Data Types**](03_Chapter_3_Basic_Data_Types.md)  
  *Integers and bitwise arithmetic, floating-point numbers and `NaN` comparison mechanics, boolean operators, string immutability and UTF-8 encoding, runes (`rune`), efficient concatenation with `bytes.Buffer`, constants, `iota` generator, and untyped literals.*

* [**Chapter 4. Composite Types**](04_Chapter_4_Composite_Types.md)  
  *Fixed-size arrays, dynamic slices (internals of `ptr/len/cap`, growth strategy of `append`, in-place algorithms), hash tables (`map`) and `nil map` write hazards, structs, composition via anonymous field embedding, JSON serialization and struct tags, `text/template` and `html/template`.*

* [**Chapter 5. Functions**](05_Chapter_5_Functions.md)  
  *Signatures and pass-by-value semantics, growable dynamic stack and recursion, multiple and named return values, 5 error-handling strategies (`error` and `io.EOF`), first-class functions, closures and loop variable capture mechanics, variadic functions, `defer`, `panic`, and `recover`.*

* [**Chapter 6. Methods**](06_Chapter_6_Methods.md)  
  *Method declarations and receivers, choosing between value and pointer receivers (`T` vs `*T`), safe invocations on `nil` receivers, composition instead of inheritance (method promotion), method values and method expressions, package-level encapsulation.*

* [**Chapter 7. Interfaces**](07_Chapter_7_Interfaces.md)  
  *Interfaces as behavioral contracts, implicit satisfaction (without `implements`), empty interface `any`, `(type, value)` internal pair, typed `nil`-pointer interface semantics, `sort.Interface`, `http.Handler`, type assertions (`x.(T)`), `type switch`, and interface design principles.*

* [**Chapter 8. Goroutines and Channels**](08_Chapter_8_Goroutines_and_Channels.md)  
  *CSP concurrency model, lifecycle of goroutines and `main()`, synchronous unbuffered channels (`happens-before`), pipelines and channel closing rules, unidirectional channels, buffered channels and goroutine leaks, `sync.WaitGroup`, counting semaphores, multiplexing with `select`, and broadcast cancellation.*

* [**Chapter 9. Concurrency and Shared Variables**](09_Chapter_9_Concurrency_and_Shared_Variables.md)  
  *Data races, prevention strategies, `sync.Mutex` and non-reentrancy, `sync.RWMutex`, memory visibility and CPU barriers, lazy thread-safe initialization with `sync.Once`, race detector (`-race`), non-blocking Singleflight cache, and in-depth comparison of goroutines vs OS threads.*

* [**Chapter 10. Packages and the Go Tool Chain**](10_Chapter_10_Packages_and_the_Go_Tool_Chain.md)  
  *Package architecture and compiler speed (dependency DAG), blank imports (`_`) and self-registration pattern, naming conventions without stuttering, CLI tools (`build`, `install`, `get`), cross-compilation (`GOOS`/`GOARCH`), build tags, private `internal/` packages, and documentation with `godoc`.*

* [**Chapter 11. Testing**](11_Chapter_11_Testing.md)  
  *`go test` tool, `*_test.go` conventions, table-driven tests, `t.Errorf` vs `t.Fatalf`, fuzzing and randomized testing, mocking with `defer`, breaking import cycles with `package foo_test` and `export_test.go`, code coverage, benchmarks (`-benchmem`), profiling (`pprof`), and executable `ExampleXxx` tests.*

* [**Chapter 12. Reflection**](12_Chapter_12_Reflection.md)  
  *Purpose of `reflect`, `reflect.Type` and `reflect.Value` descriptors, difference between `Type` and `Kind`, recursive traversal of complex data, addressability (`CanAddr`) and settability (`CanSet`), updating via `Elem()`, protecting unexported fields, struct field tags (`field.Tag.Get`), and the costs of reflection.*

* [**Chapter 13. Low-Level Programming**](13_Chapter_13_Low-Level_Programming.md)  
  *Memory layout of data types (`unsafe.Sizeof`), alignment and padding in structs, field packing optimization, universal `unsafe.Pointer` pointers and address arithmetic with `uintptr`, GC memory movement hazards, deep equality for cyclic graphs, and C integration via `cgo` (`import "C"`).*

---

## 🗂 Alphabetical Glossary

An alphabetical reference of key terms, language mechanics, and standard library concepts with direct links to corresponding chapter sections:

### A

* **Address & Dereference (`&`, `*`)** — Taking variable addresses and accessing underlying values $\to$ [Chapter 2](02_Chapter_2_Program_Structure.md), [Chapter 13](13_Chapter_13_Low-Level_Programming.md)
* **Alignment & Padding** — Memory address boundaries and struct size optimization $\to$ [Chapter 13](13_Chapter_13_Low-Level_Programming.md)
* **Anonymous Fields (Embedding)** — Struct composition without explicit field names $\to$ [Chapter 4](04_Chapter_4_Composite_Types.md), [Chapter 6](06_Chapter_6_Methods.md)
* **Anonymous Functions (Function Literals)** — First-class closures declared inline $\to$ [Chapter 5](05_Chapter_5_Functions.md)
* **`append` Function** — Dynamic slice growth algorithm and underlying array reallocation $\to$ [Chapter 4](04_Chapter_4_Composite_Types.md)
* **Arrays (`[N]T`) vs Slices (`[]T`)** — Fixed-size value types vs lightweight dynamic headers $\to$ [Chapter 4](04_Chapter_4_Composite_Types.md)

### B

* **Basic Data Types** — Integers, floating-point numbers, complex numbers, and booleans $\to$ [Chapter 3](03_Chapter_3_Basic_Data_Types.md)
* **Benchmarks (`BenchmarkXxx`)** — Performance measurement, iteration scaling (`b.N`), and allocation profiling (`-benchmem`) $\to$ [Chapter 11](11_Chapter_11_Testing.md)
* **Bitwise Operations** — Bit manipulation operators (`&`, `|`, `^`, `&^`, `<<`, `>>`) $\to$ [Chapter 3](03_Chapter_3_Basic_Data_Types.md)
* **Blank Identifier (`_`)** — Discarding unused values and blank imports for side-effects $\to$ [Chapter 1](01_Chapter_1_Tutorial.md), [Chapter 10](10_Chapter_10_Packages_and_the_Go_Tool_Chain.md)
* **Buffered Channels** — Asynchronous queues with fixed capacity $\to$ [Chapter 8](08_Chapter_8_Goroutines_and_Channels.md)
* **Build Tags & Constraints** — Platform-specific compilation (`//go:build`) and file suffixes $\to$ [Chapter 10](10_Chapter_10_Packages_and_the_Go_Tool_Chain.md)
* **`bytes.Buffer` / `strings.Builder`** — Efficient zero-allocation string building $\to$ [Chapter 3](03_Chapter_3_Basic_Data_Types.md)

### C

* **Cancellation (Broadcast)** — Terminating groups of goroutines via channel closing `close(done)` $\to$ [Chapter 8](08_Chapter_8_Goroutines_and_Channels.md)
* **Cgo (`import "C"`)** — Interoperability with C libraries and memory management $\to$ [Chapter 13](13_Chapter_13_Low-Level_Programming.md)
* **Channels (`chan`)** — Typed synchronization and communication pipes $\to$ [Chapter 8](08_Chapter_8_Goroutines_and_Channels.md)
* **Closures** — Capturing surrounding lexical variables and loop-variable capture mechanics $\to$ [Chapter 5](05_Chapter_5_Functions.md)
* **Command-Line Arguments** — Working with `os.Args` and parsing flags with `flag` $\to$ [Chapter 1](01_Chapter_1_Tutorial.md), [Chapter 2](02_Chapter_2_Program_Structure.md)
* **Composition over Inheritance** — Embedding structs and promoting methods $\to$ [Chapter 4](04_Chapter_4_Composite_Types.md), [Chapter 6](06_Chapter_6_Methods.md)
* **Constants & `iota`** — Compile-time expressions, enum generator, and untyped literals $\to$ [Chapter 3](03_Chapter_3_Basic_Data_Types.md)
* **Coverage Analysis** — Code coverage profiling and visual HTML inspection $\to$ [Chapter 11](11_Chapter_11_Testing.md)
* **Cross-Compilation** — Native builds for any OS/CPU target using `GOOS` and `GOARCH` $\to$ [Chapter 10](10_Chapter_10_Packages_and_the_Go_Tool_Chain.md)

### D

* **Data Races** — Concurrent memory access with at least one write, and prevention rules $\to$ [Chapter 9](09_Chapter_9_Concurrency_and_Shared_Variables.md)
* **Deadlock** — Permanent program hang due to cyclic locking or blocked channels $\to$ [Chapter 8](08_Chapter_8_Goroutines_and_Channels.md), [Chapter 9](09_Chapter_9_Concurrency_and_Shared_Variables.md)
* **Deferred Calls (`defer`)** — LIFO cleanup invocation upon function exit $\to$ [Chapter 5](05_Chapter_5_Functions.md)
* **Documentation (`godoc`, `go doc`)** — Doc comments format, convention, and local doc server $\to$ [Chapter 10](10_Chapter_10_Packages_and_the_Go_Tool_Chain.md)
* **Dynamic Stack** — Auto-growing goroutine stacks starting at 2 KB $\to$ [Chapter 5](05_Chapter_5_Functions.md), [Chapter 9](09_Chapter_9_Concurrency_and_Shared_Variables.md)

### E

* **Empty Interface (`any` / `interface{}`)** — The universal type satisfied by all values $\to$ [Chapter 7](07_Chapter_7_Interfaces.md)
* **Encapsulation** — Package-level visibility via uppercase/lowercase identifier names $\to$ [Chapter 6](06_Chapter_6_Methods.md)
* **Errors (`error`)** — Explicit error values, 5 handling strategies, and sentinel `io.EOF` $\to$ [Chapter 5](05_Chapter_5_Functions.md)
* **Escape Analysis** — Compiler analysis determining stack vs heap allocation $\to$ [Chapter 2](02_Chapter_2_Program_Structure.md)
* **Example Functions (`ExampleXxx`)** — Executable, self-verifying documentation tests $\to$ [Chapter 11](11_Chapter_11_Testing.md)
* **Export Rules** — Public (capitalized) vs package-private (lowercase) declarations $\to$ [Chapter 2](02_Chapter_2_Program_Structure.md)

### F

* **Functions** — Signatures, multiple returns, named results, and variadic parameters $\to$ [Chapter 5](05_Chapter_5_Functions.md)
* **Fuzzing & Random Testing** — Deterministic pseudo-random test inputs with logged seeds $\to$ [Chapter 11](11_Chapter_11_Testing.md)

### G

* **`GOMAXPROCS`** — Controlling the number of OS threads executing user-level Go code $\to$ [Chapter 9](09_Chapter_9_Concurrency_and_Shared_Variables.md)
* **Goroutine Leak** — Blocked goroutines permanently retained in memory $\to$ [Chapter 8](08_Chapter_8_Goroutines_and_Channels.md)
* **Goroutines** — Lightweight concurrent threads managed by the Go runtime $\to$ [Chapter 1](01_Chapter_1_Tutorial.md), [Chapter 8](08_Chapter_8_Goroutines_and_Channels.md), [Chapter 9](09_Chapter_9_Concurrency_and_Shared_Variables.md)

### H

* **`http.Handler` & `HandlerFunc`** — Standard web handler interfaces and function adapters $\to$ [Chapter 7](07_Chapter_7_Interfaces.md)

### I

* **`init()` Functions** — Package initialization functions executed automatically at startup $\to$ [Chapter 2](02_Chapter_2_Program_Structure.md)
* **Interfaces** — Behavioral contracts, implicit implementation, and `(type, value)` internals $\to$ [Chapter 7](07_Chapter_7_Interfaces.md)
* **Internal Packages (`internal/`)** — Enforcing encapsulation across module boundaries $\to$ [Chapter 10](10_Chapter_10_Packages_and_the_Go_Tool_Chain.md)
* **Iteration (`for range`)** — Traversing slices, arrays, maps, strings (by rune), and channels $\to$ [Chapter 1](01_Chapter_1_Tutorial.md), [Chapter 4](04_Chapter_4_Composite_Types.md), [Chapter 8](08_Chapter_8_Goroutines_and_Channels.md)

### J

* **JSON Serialization** — Marshaling, unmarshaling, stream encoders, and struct field tags $\to$ [Chapter 4](04_Chapter_4_Composite_Types.md)

### L

* **Low-Level Programming (`unsafe`)** — Memory inspection, pointer conversions, and alignment $\to$ [Chapter 13](13_Chapter_13_Low-Level_Programming.md)

### M

* **Maps (`map[K]V`)** — Hash tables, comma-ok idiom, and the panic risk of writing to a `nil map` $\to$ [Chapter 4](04_Chapter_4_Composite_Types.md)
* **Memory Barriers & Visibility** — Enforcing CPU cache synchronization and compiler ordering $\to$ [Chapter 9](09_Chapter_9_Concurrency_and_Shared_Variables.md)
* **Method Expressions (`T.Method`)** — Functions with an explicit first receiver parameter $\to$ [Chapter 6](06_Chapter_6_Methods.md)
* **Method Values (`obj.Method`)** — Binding a method to a specific object instance $\to$ [Chapter 6](06_Chapter_6_Methods.md)
* **Methods** — Functions declared with a receiver parameter (`T` or `*T`) $\to$ [Chapter 6](06_Chapter_6_Methods.md)
* **Mocking & White-Box Testing** — Dependency stubbing with `defer` restoration $\to$ [Chapter 11](11_Chapter_11_Testing.md)
* **Mutexes (`sync.Mutex`, `sync.RWMutex`)** — Mutual exclusion locks for shared state $\to$ [Chapter 9](09_Chapter_9_Concurrency_and_Shared_Variables.md)

### N

* **Named Types (`type`)** — Defining distinct domain types and attaching methods $\to$ [Chapter 2](02_Chapter_2_Program_Structure.md)
* **`NaN` (Not a Number)** — Float edge-cases and equality comparison mechanics (`math.IsNaN`) $\to$ [Chapter 3](03_Chapter_3_Basic_Data_Types.md)
* **Nil Receiver Calls** — Allowing methods to be safely invoked on `nil` pointer values $\to$ [Chapter 6](06_Chapter_6_Methods.md)
* **Nil-Pointer in Interface Semantics** — Why an interface containing a typed `nil` pointer is not equal to `nil` $\to$ [Chapter 7](07_Chapter_7_Interfaces.md)

### P

* **Packages & Imports** — Code organization, DAG dependency rule, and avoiding stuttering $\to$ [Chapter 10](10_Chapter_10_Packages_and_the_Go_Tool_Chain.md)
* **Panic & Recover** — Halting execution on severe programmer bugs and graceful recovery $\to$ [Chapter 5](05_Chapter_5_Functions.md)
* **Pipelines** — Chaining stages of concurrent data processing through channels $\to$ [Chapter 8](08_Chapter_8_Goroutines_and_Channels.md)
* **Pointers & `new()`** — Memory addressing, pointer dereferencing, and zero-initialization $\to$ [Chapter 2](02_Chapter_2_Program_Structure.md)
* **Profiling (`pprof`)** — CPU, memory heap, and goroutine blocking analysis $\to$ [Chapter 11](11_Chapter_11_Testing.md)

### R

* **Race Detector (`-race`)** — Dynamic runtime data race analyzer based on ThreadSanitizer $\to$ [Chapter 9](09_Chapter_9_Concurrency_and_Shared_Variables.md)
* **Reflection (`reflect`)** — Inspecting and mutating types and values at runtime (`Type` vs `Kind`, `CanSet`) $\to$ [Chapter 12](12_Chapter_12_Reflection.md)
* **Runes (`rune`)** — Unicode code points represented as `int32` $\to$ [Chapter 3](03_Chapter_3_Basic_Data_Types.md)

### S

* **Scope vs Lifetime** — Lexical visibility at compile-time vs variable duration in memory $\to$ [Chapter 2](02_Chapter_2_Program_Structure.md)
* **`select` Multiplexing** — Multi-channel event waiting, non-blocking default, and pseudo-random choice $\to$ [Chapter 8](08_Chapter_8_Goroutines_and_Channels.md)
* **Semaphores (Counting)** — Concurrency rate-limiting via buffered token channels $\to$ [Chapter 8](08_Chapter_8_Goroutines_and_Channels.md)
* **Singleflight (Deduplication Cache)** — Concurrent memoization cache suppressing duplicate calls $\to$ [Chapter 9](09_Chapter_9_Concurrency_and_Shared_Variables.md)
* **Slices** — Dynamic views over underlying arrays (`ptr`, `len`, `cap`), in-place algorithms $\to$ [Chapter 4](04_Chapter_4_Composite_Types.md)
* **`sort.Interface`** — Abstract sorting contract (`Len`, `Less`, `Swap`) $\to$ [Chapter 7](07_Chapter_7_Interfaces.md)
* **Strings** — Immutable byte sequences in UTF-8 format $\to$ [Chapter 3](03_Chapter_3_Basic_Data_Types.md)
* **Struct Field Tags** — Metadata attributes for JSON/XML/HTTP mapping $\to$ [Chapter 4](04_Chapter_4_Composite_Types.md), [Chapter 12](12_Chapter_12_Reflection.md)
* **Structs** — Composite data layouts in contiguous memory $\to$ [Chapter 4](04_Chapter_4_Composite_Types.md)
* **`sync.Once`** — Lazy, thread-safe one-time initialization $\to$ [Chapter 9](09_Chapter_9_Concurrency_and_Shared_Variables.md)
* **`sync.WaitGroup`** — Coordinating completion across dynamic sets of workers $\to$ [Chapter 8](08_Chapter_8_Goroutines_and_Channels.md)

### T

* **Table-Driven Tests** — Standard Go idiom for parametrized testing $\to$ [Chapter 11](11_Chapter_11_Testing.md)
* **Templates (`text/template`, `html/template`)** — Data-driven text output and contextual XSS auto-escaping $\to$ [Chapter 4](04_Chapter_4_Composite_Types.md)
* **Type Assertions (`x.(T)`)** — Dynamic extraction of concrete or interface types from an interface $\to$ [Chapter 7](07_Chapter_7_Interfaces.md)
* **Type Switch (`switch x.(type)`)** — Multi-way type branching construct $\to$ [Chapter 7](07_Chapter_7_Interfaces.md)

### U

* **Unbuffered Channels** — Synchronous rendezvous communication guaranteeing `happens-before` $\to$ [Chapter 8](08_Chapter_8_Goroutines_and_Channels.md)
* **Unidirectional Channels (`chan<-`, `<-chan`)** — Type-safe restriction to send-only or receive-only operations $\to$ [Chapter 8](08_Chapter_8_Goroutines_and_Channels.md)
* **`unsafe.Pointer` & `uintptr`** — Arbitrary memory pointer conversion and address arithmetic caveats $\to$ [Chapter 13](13_Chapter_13_Low-Level_Programming.md)

### V

* **Variable Shadowing** — Unintentional masking of outer variables caused by `:=` inside nested blocks $\to$ [Chapter 2](02_Chapter_2_Program_Structure.md)
* **Variadic Functions (`...T`)** — Functions accepting variable numbers of trailing arguments $\to$ [Chapter 5](05_Chapter_5_Functions.md)

### Z

* **Zero Values** — Deterministic default initialization for uninitialized memory $\to$ [Chapter 2](02_Chapter_2_Program_Structure.md)
