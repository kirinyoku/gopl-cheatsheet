# Chapter 2. Program Structure

Foundational principles of Go program organization: naming conventions and export rules, memory management (pointers, stack vs heap escape analysis), type definitions, package initialization via `init()`, and avoiding subtle variable shadowing bugs.

---

### Table of Contents
* [2.1. Names: Export Rules and Conventions](#21-names-export-rules-and-conventions)
* [2.2. The Four Kinds of Declarations](#22-the-four-kinds-of-declarations)
* [2.3. Variables, Pointers, and Memory Management](#23-variables-pointers-and-memory-management)
* [2.4. Assignments and the Comma-Ok Idiom](#24-assignments-and-the-comma-ok-idiom)
* [2.5. Type Declarations and Named Types](#25-type-declarations-and-named-types)
* [2.6. Packages, Files, and Package Initialization (`init`)](#26-packages-files-and-package-initialization-init)
* [2.7. Scope vs Lifetime and the Variable Shadowing Trap](#27-scope-vs-lifetime-and-the-variable-shadowing-trap)

---

## 2.1. Names: Export Rules and Conventions

### First-Letter Rule (Export Visibility)
Go does not use access modifier keywords like `public`, `private`, or `protected`. Visibility is controlled entirely by the case of the identifier's first letter:
* **Capital letter** (`Print`, `User`, `ContentType`) $\to$ **Exported** (publicly accessible across package boundaries).
* **Lowercase letter** (`calculate`, `user`, `internalState`) $\to$ **Unexported** (private, visible only within its package).
* **Package names** are always written in lowercase as a single word (`bytes`, `json`, `tempconv`).

### CamelCase Conventions
* Go standard style favors `camelCase` and `PascalCase` over `snake_case`.
* **Acronyms maintain uniform casing**: write `serveHTTP`, `escapeHTML`, `userID`, `urlPath` (rather than `serveHttp`, `escapeHtml`, `userId`, `url_path`).
* **Scoping length rule**: *"The smaller the variable's scope, the shorter its name"* (a local loop counter is `i`, a reader is `r`, a buffer is `b`).

---

## 2.2. The Four Kinds of Declarations

At the package level, exactly four declaration keywords exist:
1. `var` — Variables
2. `const` — Constants
3. `type` — Named types
4. `func` — Functions and methods

> [!NOTE]
> At package scope, declaration order **does not matter**: types and functions can reference each other regardless of whether they are declared earlier or later in the file.

> [!NOTE]
> **Modern note (Go 1.18+):** While the four keywords remain unchanged, `func` and `type` declarations now support type parameters (Generics), which did not exist when the book was written:
> ```go
> func Map[T, R any](s []T, f func(T) R) []R // Generic function
> type Set[T comparable] map[T]struct{}      // Generic type
> ```

---

## 2.3. Variables, Pointers, and Memory Management

### Zero Values
In Go, memory is always guaranteed to be safely initialized. Uninitialized variables receive their type's default zero value:

| Data Type | Zero Value |
| :--- | :--- |
| Numbers (`int`, `float64`, `byte`) | `0` |
| Booleans (`bool`) | `false` |
| Strings (`string`) | `""` (empty string) |
| Pointers, slices, maps, channels, interfaces, functions | `nil` |
| Structs (`struct`) | All fields recursively zero-initialized |

### Short Variable Declarations (`:=`)
The `:=` syntax is allowed **only inside function bodies**:
```go
in, err := os.Open("in.txt")     // Declares both in and err
out, err := os.Create("out.txt") // Declares out, reassigns err
```

> [!IMPORTANT]
> To reuse an existing variable on the left side of `:=`, **at least one variable on the left must be newly declared**. If all variables are already declared in the current lexical block, a compile error occurs.
> Note that variables in an **outer** lexical block are never reused by `:=` — instead, `:=` will silently create a new shadowed local variable (see Section 2.7).

---

### Pointers (`&` and `*`)
* `&x` — Address-of operator (yields type `*int`).
* `*p` — Pointer dereference (accesses the value stored at the address).
* The zero value of any pointer is `nil`.

```go
func createPointer() *int {
    v := 42
    return &v // In C/C++, returning a local stack address is undefined behavior. In Go, it is completely safe!
}
```

### Escape Analysis
Where a variable lives — **on the stack or on the heap** — is determined automatically by the compiler:
* If a variable is local and never referenced after the function returns $\to$ allocated on the fast **stack**.
* If a variable's address is returned, passed across goroutines, or stored in a persistent structure $\to$ it **escapes to the heap** and is managed by the garbage collector.

The built-in function `new(T)` allocates zero-initialized storage for type `T` and returns a pointer `*T` (syntactic sugar for `var dummy T; return &dummy`).

---

## 2.4. Assignments and the Comma-Ok Idiom

### Tuple Assignment
All right-hand side expressions are fully evaluated **before** variables on the left are updated, enabling in-place value swaps without temporary variables:
```go
x, y = y, x // Safe value swap
```

### The Comma-Ok Idiom
Used across Go to safely retrieve values alongside a boolean status flag:
```go
val, ok := m[key] // ok == true if key exists in map
val, ok := x.(T)  // ok == true if dynamic interface type matches T
val, ok := <-ch   // ok == false if channel is closed and empty
```

---

## 2.5. Type Declarations and Named Types

Declaring a new named type creates an explicit type boundary between logically distinct concepts, even if they share the same underlying representation:

```go
type Celsius float64
type Fahrenheit float64

var c Celsius = 100
var f Fahrenheit = 212

// c = f          // COMPILE ERROR: distinct types
// if c == f {}   // COMPILE ERROR: cannot compare mismatched temperature scales
```

### Attaching Methods to Named Types
Methods can be defined on any named type **declared within your package** (you cannot attach methods to types from external packages). For instance, implementing `fmt.Stringer`:

```go
func (c Celsius) String() string {
    return fmt.Sprintf("%g°C", c)
}
```

---

## 2.6. Packages, Files, and Package Initialization (`init`)

* **Structure**: All Go source files in the same directory belong to the same package and share declarations without explicit imports.
* **Initialization Order**: Dependencies are initialized bottom-up; package `main` is always initialized last.
* **Package-Internal Order**: Package-level variables are initialized first (in topological dependency order), followed by `init()` execution; files are processed in alphabetical order by the compiler.

### Initialization Functions (`init`)
```go
func init() {
    // Precomputing lookup tables, checking environment variables
}
```
* Invoked automatically by the runtime before program execution starts.
* Accept no arguments, return no values, and cannot be invoked manually.
* A single file or package may define multiple `init()` functions.

---

## 2.7. Scope vs Lifetime and the Variable Shadowing Trap

### Core Distinction
* **Scope** — The syntactic block of source code where a name is visible to the compiler (compile-time property).
* **Lifetime** — The duration for which a variable exists in memory (runtime property).

> [!CAUTION]
> **The Classic Variable Shadowing Trap**:  
> Accidentally using `:=` in an inner block creates a **new local variable**, leaving the outer package-level variable uninitialized:
>
> ```go
> var cwd string // Package-level variable
>
> func init() {
>     // BUG: := creates a LOCAL cwd variable; the package-level cwd remains ""!
>     cwd, err := os.Getwd() 
>     if err != nil {
>         log.Fatal(err)
>     }
> }
> ```
>
> **Correct pattern**:
> ```go
> var cwd string
>
> func init() {
>     var err error
>     cwd, err = os.Getwd() // Standard assignment to the outer variable
>     if err != nil {
>         log.Fatal(err)
>     }
> }
> ```
