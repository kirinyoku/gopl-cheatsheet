# Chapter 7. Interfaces

Interfaces in Go as abstract behavioral contracts: implicit implementation without keywords, internal `(type, value)` dual-word structure, the deadly nil-pointer-in-interface trap, sorting with `sort.Interface`, HTTP handlers with `http.Handler`, type assertions, type switches, and idiomatic interface design principles.

---

### Table of Contents
* [7.1. Interfaces as Behavioral Contracts](#71-interfaces-as-behavioral-contracts)
* [7.2. Implicit Implementation](#72-implicit-implementation)
* [7.3. Interface Values Under the Hood](#73-interface-values-under-the-hood)
* [7.4. Sorting with `sort.Interface` and Web Handlers with `http.Handler`](#74-sorting-with-sortinterface-and-web-handlers-with-httphandler)
* [7.5. Type Assertions](#75-type-assertions)
* [7.6. Type Switches](#76-type-switches)
* [7.7. Interface Design Advice](#77-interface-design-advice)

---

## 7.1. Interfaces as Behavioral Contracts

In Go, there is a fundamental distinction between concrete and abstract types:
* **Concrete Type** (`int`, `*os.File`, `*bytes.Buffer`) — Defines the exact physical memory layout and operations directly manipulating that representation.
* **Interface (Abstract Type)** — Encapsulates data representation completely, defining only a contract of **observable behavior** through a set of methods.

```go
type Stringer interface {
    String() string
}
```
*Any type defining a `String() string` method automatically satisfies `fmt.Stringer` and is formatted cleanly by `fmt.Printf("%v", x)`.*

### Naming Conventions
Single-method interfaces are conventionally named by appending `-er` to the method name (`Reader`, `Writer`, `Closer`, `Formatter`).

### Interface Embedding (Composition)
Interfaces can be composed into larger contracts by embedding:
```go
type Reader interface { Read(p []byte) (n int, err error) }
type Writer interface { Write(p []byte) (n int, err error) }

type ReadWriter interface {
    Reader
    Writer
}
```

---

## 7.2. Implicit Implementation

Go has **no `implements` keyword**. A concrete type satisfies an interface automatically simply by declaring all required methods.

```go
var w io.Writer
w = os.Stdout         // OK: *os.File implements Write
w = new(bytes.Buffer) // OK: *bytes.Buffer implements Write
```

### Receiver Semantics (`T` vs `*T`):
If a method is defined on a pointer receiver `func (s *IntSet) String() string`, then `fmt.Stringer` is satisfied **only by `*IntSet`**, not by the value `IntSet`.

### The Empty Interface `any` (`interface{}`)
The empty interface contains zero methods $\to$ **every type satisfies it** (`any` is the official standard alias for `interface{}` since Go 1.18):
```go
var x any = 42
x = "text"
x = []int{1, 2, 3}
```

> [!TIP]
> **Static Compile-Time Interface Check**:
> ```go
> var _ io.Writer = (*MyType)(nil) // Compile error if MyType fails to implement io.Writer
> ```

---

## 7.3. Interface Values Under the Hood

Conceptually, an interface value consists of two machine words:
1. **Dynamic Type (Type Descriptor)** — Points to the runtime type information (`*bytes.Buffer`, `int`).
2. **Dynamic Value (Value Pointer)** — Points to the underlying data payload in memory.

```text
Interface Value: [ Dynamic Type (*os.File) | Dynamic Value Pointer (&os.Stdout) ]
```

> [!CAUTION]
> **The Critical Trap: A nil pointer inside an interface is NOT equal to nil!**  
> An interface value equals `nil` **only when both its dynamic type and dynamic value are nil**. If an interface contains a concrete type descriptor but a `nil` pointer value, evaluating `if iface != nil` yields `true`:
>
> ```go
> func test(out io.Writer) {
>     // out is NOT nil! out holds (*bytes.Buffer, nil)
>     if out != nil {
>         out.Write([]byte("data")) // RUNTIME PANIC! Calling method on a nil pointer
>     }
> }
>
> var buf *bytes.Buffer // buf == nil
> test(buf)             // Passing typed nil: interface receives (*bytes.Buffer, nil)
> ```
>
> **Correct practice**: Declare the variable directly using the interface type: `var buf io.Writer` (so the interface holds `(nil, nil)` initially).

---

## 7.4. Sorting with `sort.Interface` and Web Handlers with `http.Handler`

### Custom Sorting via `sort.Interface`
The `sort` package is abstracted from data representations via a 3-method contract:
```go
type Interface interface {
    Len() int           // Number of elements
    Less(i, j int) bool // Is element i less than element j?
    Swap(i, j int)      // Swap elements at indices i and j
}

type ByAge []Person
func (a ByAge) Len() int           { return len(a) }
func (a ByAge) Less(i, j int) bool { return a[i].Age < a[j].Age }
func (a ByAge) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }

sort.Sort(ByAge(people))
```

> [!NOTE]
> **Modern note:** Go 1.8 introduced `sort.Slice(people, func(i, j int) bool { return people[i].Age < people[j].Age })`. Furthermore, Go 1.21 added generic functions `slices.Sort` and `slices.SortFunc(people, func(a, b Person) int { return cmp.Compare(a.Age, b.Age) })`. Defining a custom 3-method type via `sort.Interface` is now needed only for non-slice data structures.

### Web Handlers (`http.Handler`)
```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}

// The http.HandlerFunc adapter converts a standard func(w, r) into an http.Handler:
mux.Handle("/list", http.HandlerFunc(myHandler))
```

---

## 7.5. Type Assertions

The type assertion `x.(T)` inspects the dynamic type of an interface `x`:

```go
var w io.Writer = os.Stdout

// 1. Concrete Type Assertion:
f := w.(*os.File) // f has concrete type *os.File (panics if incorrect)

// 2. Safe Form with Comma-Ok:
if f, ok := w.(*os.File); ok {
    // Safely use f as *os.File
}

// 3. Querying for an Additional Interface Contract:
if rw, ok := w.(io.ReadWriter); ok {
    // Object supports both reading and writing
}
```

---

## 7.6. Type Switches

A specialized `switch` construct that branches on an interface's dynamic type:

```go
func sqlQuote(x any) string {
    switch x := x.(type) {
    case nil:
        return "NULL"
    case int, int64:
        return fmt.Sprintf("%d", x)
    case bool:
        if x {
            return "TRUE"
        }
        return "FALSE"
    case string:
        return fmt.Sprintf("'%s'", strings.ReplaceAll(x, "'", "''"))
    default:
        panic(fmt.Sprintf("unsupported type: %T", x))
    }
}
```

---

## 7.7. Interface Design Advice

> [!TIP]
> 1. **Do not create interfaces prematurely**: Declare an interface only when you have at least two distinct concrete types requiring unified polymorphic handling.
> 2. **Small interfaces are the most powerful**: Interfaces with 1–2 methods (`io.Reader`, `io.Writer`, `fmt.Stringer`) maximize reuse and composability.
> 3. **The Golden Rule of Go**: *"Accept interfaces, return structs"* (demand only the minimum necessary behavior at function boundaries).
