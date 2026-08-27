# Chapter 10. Packages and the Go Tool Chain

Modular code architecture in Go: why compilation is exceptionally fast (DAG dependency graphs), `package` and `import` mechanics, blank imports (`import _`) and the self-registration pattern, avoiding package name stuttering, CLI toolchain (`build`, `install`, `get`), cross-compilation (`GOOS`/`GOARCH`), build tags, private `internal/` packages, and documentation via `go doc`.

---

### Table of Contents
* [10.1. Why the Go Compiler Is Fast](#101-why-the-go-compiler-is-fast)
* [10.2. Import Paths and Package Names](#102-import-paths-and-package-names)
* [10.3. Blank Imports (`import _`)](#103-blank-imports-import-_)
* [10.4. Package Naming Conventions](#104-package-naming-conventions)
* [10.5. CLI Toolchain (`go tool`)](#105-cli-toolchain-go-tool)
* [10.6. Internal Packages (`internal/`)](#106-internal-packages-internal)
* [10.7. Documentation (`godoc` and `go doc`)](#107-documentation-godoc-and-go-doc)

---

## 10.1. Why the Go Compiler Is Fast

1. **Explicit Header Imports**: All imports appear strictly at the top of each file, allowing the compiler to construct the complete dependency tree without parsing function bodies.
2. **Strict Ban on Cyclic Imports**: The package dependency graph must form a strict Directed Acyclic Graph (**DAG**), enabling parallel compilation of independent packages across all CPU cores.
3. **Self-Contained Object Files**: Compiled package object files embed complete metadata about all their exported transitive dependencies. To compile an importer, the compiler reads exactly one object file per direct import.

---

## 10.2. Import Paths and Package Names

Every `.go` source file starts with a `package <name>` statement.

* **Standard Library**: Short import paths (`"fmt"`, `"net/http"`, `"bytes"`).
* **Third-Party Packages**: URL-prefixed module paths (`"github.com/lib/pq"`).
* **Name Matching Rule**: By convention, the package name matches the final directory segment of the import path (e.g., `"encoding/json"` $\to$ `package json`).
* **Exceptions**:
  * `package main` — Entry point for executable binaries.
  * `package foo_test` — External test packages (breaks cyclic test dependencies).

```go
import (
    "crypto/rand"
    mrand "math/rand" // Alias to resolve naming collision
)
```

---

## 10.3. Blank Imports (`import _`)

> Unused imports produce compile-time errors in Go.

A **blank import** `import _ "path"` imports a package **solely for its initialization side effects** — executing `init()` functions and registering drivers.

### The Self-Registration Pattern

```go
// 1. Automatic image decoder registration in image.RegisterFormat:
import (
    "image"
    _ "image/png"
    _ "image/jpeg"
)
img, format, err := image.Decode(r) // Automatically identifies PNG or JPEG

// 2. Database driver registration in database/sql:
import (
    "database/sql"
    _ "github.com/lib/pq" // Registers the "postgres" SQL driver
)
db, err := sql.Open("postgres", dataSourceName)
```

---

## 10.4. Package Naming Conventions

> [!TIP]
> **Package names should be**:
> * Short, lowercase, singular words: `bytes`, `json`, `sync`, `time`, `http`.
> * Descriptive: `imageutil` or `ioutil` is preferred over ambiguous `util` or `helper`.

### Avoid "Stuttering"
Because exported identifiers are always qualified with their package name, identifier names must not repeat the package name:

| Anti-Pattern | Idiomatic Go | Usage in Code |
| :--- | :--- | :--- |
| `http.HttpClient` | `http.Client` | `client := http.Client{}` |
| `bytes.ByteBuffer` | `bytes.Buffer` | `buf := bytes.Buffer{}` |
| `json.JSONMarshal` | `json.Marshal` | `json.Marshal(v)` |
| `rand.RandInt` | `rand.Int` | `rand.Int()` |

---

## 10.5. CLI Toolchain (`go tool`)

* `go build` — Compiles packages (for `package main`, creates an executable in the current directory).
* `go run main.go` — Compiles into a temporary build cache and executes immediately.
* `go install` — Compiles and moves the binary to `$GOBIN` (default `$HOME/go/bin`) for terminal access.
* `go get <path>` — **Adds or updates dependencies in `go.mod`** (since Go 1.18, it no longer installs binaries).

> [!NOTE]
> **Modern note (Go Modules):** In legacy `GOPATH` workflows, `go get` fetched and built binaries. Today, project dependencies are managed via `go.mod`: `go get`, `go mod tidy`, and `go mod download`. CLI development tools are installed using explicit versions: `go install golang.org/x/tools/cmd/goimports@latest` (the `@version` suffix is required outside a module).

### Native Cross-Compilation
Build for any target operating system and CPU architecture without foreign toolchains:
```bash
GOOS=linux GOARCH=arm64 go build -o myapp-arm64
GOOS=windows GOARCH=amd64 go build -o myapp.exe
```

### Conditional Compilation (Build Tags)
1. **File Suffixes**: `file_linux.go`, `file_windows.go`, `asm_amd64.s`.
2. **Build Directives in Header**:
   ```go
   //go:build linux || darwin
   ```
   The book references legacy `// +build` comments, which were superseded by `//go:build` in Go 1.17.

---

## 10.6. Internal Packages (`internal/`)

Go provides a built-in access control mechanism for restricting private code within a repository:
* Any package containing an `internal` path segment (e.g., `net/http/internal/chunked`) can be imported **only by packages located inside its parent directory tree** (`net/http/...`).
* Any external package attempting to import an `internal` package fails with a compile error.

---

## 10.7. Documentation (`godoc` and `go doc`)

* Comments immediately preceding an exported declaration are extracted automatically as documentation.
* The first sentence must begin with the name of the declared identifier:
  ```go
  // Fprintf formats according to a format specifier and writes to w.
  func Fprintf(w io.Writer, format string, a ...any) (int, error)
  ```
* Command-line inspection: `go doc http.Client.Do`.
* Modern web documentation: `pkgsite`.

> [!NOTE]
> **Modern note:** The standalone `godoc` web server command was removed from the standard Go distribution in Go 1.12. Today, browse documentation online via `pkg.go.dev` or run a local instance with `go run golang.org/x/pkgsite/cmd/pkgsite@latest`. The terminal `go doc` tool remains fully supported.
