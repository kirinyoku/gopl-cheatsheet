# Chapter 11. Testing

The Go testing ecosystem: the `go test` toolchain, `*_test.go` file conventions, table-driven tests, `t.Errorf` vs `t.Fatalf`, randomized testing and native fuzzing, mocking via `defer`, breaking import cycles with external `package foo_test` and `export_test.go`, code coverage analysis, performance benchmarks (`-benchmem`), profiling with `pprof`, and self-verifying executable `ExampleXxx` documentation.

---

### Table of Contents

* [11.1. The `go test` Tool](#111-the-go-test-tool)
* [11.2. Test Functions (`TestXxx`)](#112-test-functions-testxxx)
* [11.3. Table-Driven Tests](#113-table-driven-tests)
* [11.4. Randomized and White-Box Testing](#114-randomized-and-white-box-testing)
* [11.5. External Tests and Breaking Import Cycles](#115-external-tests-and-breaking-import-cycles)
* [11.6. Code Coverage Analysis](#116-code-coverage-analysis)
* [11.7. Performance Benchmarks (`BenchmarkXxx`)](#117-performance-benchmarks-benchmarkxxx)
* [11.8. Profiling with `pprof`](#118-profiling-with-pprof)
* [11.9. Example Functions (`ExampleXxx`)](#119-example-functions-examplexxx)

---

## 11.1. The `go test` Tool

Go takes a minimalist approach to testing: no heavy third-party testing frameworks or complex assertion DSLs are required. Tests are written in standard Go inside source files named with a `*_test.go` suffix.

* Files matching `*_test.go` are **ignored** during standard `go build` runs and compiled **only** when invoking `go test`.
* Go test files support 4 distinct function categories:
  1. `TestXxx(t *testing.T)` — Functional, unit, and integration tests.
  2. `BenchmarkXxx(b *testing.B)` — Performance and allocation benchmarks.
  3. `ExampleXxx()` — Executable documentation examples.
  4. `FuzzXxx(f *testing.F)` — Native coverage-guided fuzz testing (added in Go 1.18).

---

## 11.2. Test Functions (`TestXxx`)

```go
func TestIsPalindrome(t *testing.T) {
    if !IsPalindrome("kayak") {
        t.Errorf(`IsPalindrome("kayak") = false, want true`)
    }
}
```

### Core `*testing.T` Methods:

* `t.Error(...)` / `t.Errorf(...)` — Records a test failure and marks the test failed, but **continues executing subsequent checks** (providing a complete picture of all failures in one run).
* `t.Fatal(...)` / `t.Fatalf(...)` — Records a failure and **aborts the current test function immediately** (used when critical setup or preconditions fail).
* `t.Logf(...)` — Formats and logs informational debug messages (displayed only on test failure or with the `-v` verbose flag).

---

## 11.3. Table-Driven Tests

The idiomatic industry standard for organizing test suites in Go:

```go
func TestIsPalindrome(t *testing.T) {
    tests := []struct {
        input string
        want  bool
    }{
        {"", true},
        {"a", true},
        {"kayak", true},
        {"A man, a plan, a canal: Panama", true},
        {"palindrome", false},
    }

    for _, tc := range tests {
        if got := IsPalindrome(tc.input); got != tc.want {
            t.Errorf("IsPalindrome(%q) = %v, want %v", tc.input, got, tc.want)
        }
    }
}
```

### Useful `go test` Flags:

* `go test -v` — Verbose mode, displaying individual test names and execution duration.
* `go test -run="French|Canal"` — Filters tests to run using a regex pattern matching test names.

> [!NOTE]
> **Modern note (Go 1.7+):** Modern table tests include a descriptive `name` field and dispatch test cases as isolated **subtests** via `t.Run(tc.name, func(t *testing.T) { ... })`. This enables clear failure attribution, targeted test execution (`-run TestX/case_name`), and concurrent execution with `t.Parallel()`. In test helpers, call `t.Helper()` (Go 1.9) to preserve accurate line numbers in failure logs.

---

## 11.4. Randomized and White-Box Testing

* **Randomized Testing & Seed Logging**:
  Generates pseudo-random input payloads to discover unexpected edge-case failures.
  > [!IMPORTANT]
  > Always log the random generator seed (`t.Logf("Seed: %d", seed)`) so any discovered test failure can be deterministically reproduced.
* **Testing Command-Line Programs (`package main`)**:
  Decouple output generation logic `echo(out io.Writer, ...)` from `main()`. In tests, pass `out = new(bytes.Buffer)` to inspect stdout cleanly.
* **White-Box Testing (Mocking with `defer`)**:
  ```go
  func TestQuota(t *testing.T) {
      saved := notifyUser
      defer func() { notifyUser = saved }() // Guaranteed restoration of real dependency

      notifyUser = func(user, msg string) { /* Test stub callback */ }
      CheckQuota("user@example.com")
  }
  ```

> [!NOTE]
> **Modern note (Go 1.18+):** Native fuzz testing is built directly into the standard toolchain:
> ```go
> func FuzzIsPalindrome(f *testing.F) {
>     f.Add("kayak") // Seed inputs
>     f.Fuzz(func(t *testing.T, s string) { IsPalindrome(s) })
> }
> ```
> Run fuzzing using `go test -fuzz=FuzzIsPalindrome`. Failing inputs are automatically persisted in `testdata/fuzz/` and re-tested automatically during standard `go test` runs.

---

## 11.5. External Tests and Breaking Import Cycles

If an internal test for a low-level package like `net/url` needs to import a higher-level package like `net/http` to perform integration tests, a cyclic dependency error occurs.

**Solution**:
* Place the test into an external test package: `package url_test` (inside `url_test.go`).
* `go test` compiles it as an independent external consumer package, cleanly breaking the cycle.

### The `export_test.go` Pattern (Accessing Unexported Identifiers):

To expose an unexported symbol `internalHelper` exclusively to external tests in `package url_test` without polluting the public API:

```go
// File export_test.go (compiled ONLY during go test):
package url

var InternalHelper = internalHelper
```

---

## 11.6. Code Coverage Analysis

```bash
# Generate a coverage profile:
go test -coverprofile=c.out

# Open an interactive HTML coverage visualization in the browser:
go tool cover -html=c.out

# Count execution frequencies per code block:
go test -covermode=count -coverprofile=c.out
```

> [!NOTE]
> **Modern note (Go 1.20+):** Coverage analysis is available for production binaries: `go build -cover` produces a binary that writes coverage data to the directory specified by `GOCOVERDIR`, capturing test coverage from end-to-end integration and staging runs.

---

## 11.7. Performance Benchmarks (`BenchmarkXxx`)

```go
func BenchmarkIsPalindrome(b *testing.B) {
    for i := 0; i < b.N; i++ {
        IsPalindrome("A man, a plan, a canal: Panama")
    }
}
```

* `b.N` — The iteration count, dynamically adjusted by the testing harness until the benchmark runs long enough for statistically meaningful results (minimum 1 second).
* `go test -bench=.` — Executes all benchmarks in the package.
* `go test -bench=. -benchmem` — Includes heap allocation metrics (`B/op` for bytes per operation, `allocs/op` for distinct allocation count).

> [!NOTE]
> **Modern note (Go 1.24+):** The idiomatic benchmark loop is `for b.Loop() { IsPalindrome(...) }`, which manages iterations and timer resets automatically while preventing compiler dead-code elimination. The legacy `b.N` loop remains supported.

---

## 11.8. Profiling with `pprof`

```bash
# Collect CPU or heap memory profile:
go test -run=NONE -bench=MyBenchmark -cpuprofile=cpu.out
go test -memprofile=mem.out

# Open interactive browser interface with call graph:
go tool pprof -http=:8080 cpu.out
```

---

## 11.9. Example Functions (`ExampleXxx`)

```go
func ExampleIsPalindrome() {
    fmt.Println(IsPalindrome("kayak"))
    fmt.Println(IsPalindrome("hello"))
    // Output:
    // true
    // false
}
```

1. **Live Documentation**: Automatically rendered on `pkg.go.dev` documentation pages.
2. **Self-Verifying Tests**: The `// Output:` comment instructs `go test` to capture stdout and assert equality against the documented output.
3. **Guard Against Drift**: Examples must compile and pass tests, guaranteeing documentation never becomes obsolete.
