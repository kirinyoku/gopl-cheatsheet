# Chapter 6. Methods

An exploration of object-oriented design in Go: method declarations, value vs pointer receivers (`T` vs `*T`), safe calls on `nil` receivers, composition and method promotion via struct embedding, method values and expressions, and package-level encapsulation.

---

### Table of Contents
* [6.1. Method Declarations and Receivers](#61-method-declarations-and-receivers)
* [6.2. Value vs Pointer Receivers (`*T`)](#62-value-vs-pointer-receivers-t)
* [6.3. Composition Over Inheritance (Struct Embedding)](#63-composition-over-inheritance-struct-embedding)
* [6.4. Method Values and Method Expressions](#64-method-values-and-method-expressions)
* [6.5. Encapsulation: Hiding Implementation Details](#65-encapsulation-hiding-implementation-details)

---

## 6.1. Method Declarations and Receivers

In Go, a method is a function declared with an extra parameter before its name — the **receiver**, which binds the function to a specific named type:

```go
type Point struct{ X, Y float64 }

// Standard package-level function:
func Distance(p, q Point) float64 { ... }

// Method attached to the Point type:
func (p Point) Distance(q Point) float64 {
    return math.Hypot(q.X-p.X, q.Y-p.Y)
}
```

### Key Rules
* **Receiver Naming**: Go **does not use keywords like `this` or `self`**. Idiomatic convention favors short, 1–2 letter names derived from the type's initial letter (`p` for `Point`, `c` for `Client`).
* **Methods on Any Named Type**: Methods can be declared on structs, numbers, booleans, strings, slices, maps, channels, and function types:
  ```go
  type Path []Point

  func (path Path) Distance() float64 { ... } // Method on a custom slice type
  ```
* *Constraint*: The type must be defined in the same package, and its underlying type must not be a pointer or an interface.

---

## 6.2. Value vs Pointer Receivers (`*T`)

```go
func (p *Point) ScaleBy(factor float64) {
    p.X *= factor
    p.Y *= factor
}
```

### When to Use a Pointer Receiver `*T`:
1. When the method needs to **mutate the caller's state**.
2. When the receiver is a large struct and copying it on each call would be **computationally expensive**.

> [!IMPORTANT]
> **Consistency Rule**: If any method on a named type requires a pointer receiver `*T`, **all methods on that type should be defined with pointer receivers** for API consistency.

### Automatic Address-Of and Dereferencing
Go compiler automatically adapts receiver types at call sites:
* On a value `p`, calling `p.ScaleBy(2)` is compiled automatically as `(&p).ScaleBy(2)`.  
  *Caveat*: This works only for *addressable* variables. Calling pointer methods on non-addressable values (e.g., map lookups `m["k"].ScaleBy(2)` or function return values) causes a compile error.
* On a pointer `pptr`, calling `pptr.Distance(q)` is compiled automatically as `(*pptr).Distance(q)`.

### `nil` as a Valid Receiver
Methods can safely be called on `nil` pointer receivers if the method body explicitly handles `nil`:

```go
type IntList struct {
    Value int
    Tail  *IntList
}

// nil represents an empty list
func (list *IntList) Sum() int {
    if list == nil {
        return 0 // Safe nil handling
    }
    return list.Value + list.Tail.Sum()
}
```
*Standard library example*: `url.Values(nil).Get("key")` safely returns `""` without panicking.

---

## 6.3. Composition Over Inheritance (Struct Embedding)

Go avoids classical class hierarchies. Code reuse and behavioral composition are achieved via **anonymous field embedding**:

```go
type Point struct{ X, Y float64 }

func (p *Point) ScaleBy(factor float64) {
    p.X *= factor
    p.Y *= factor
}

type ColoredPoint struct {
    Point // Anonymous embedded struct
    Color color.RGBA
}
```

* **Method Promotion**: Methods belonging to `Point` are automatically promoted to `ColoredPoint`:
  ```go
  var cp ColoredPoint
  cp.ScaleBy(2) // Dispatches to Point.ScaleBy(&cp.Point, 2)
  ```
* **Crucial Difference**: `ColoredPoint` **is not a subtype of** `Point` (there is no subclass polymorphism). You cannot pass `cp` directly to a function expecting `Point` without explicitly accessing `cp.Point`.

### Embedded Mutex Pattern:
```go
var cache = struct {
    sync.Mutex // Embedded mutex
    data       map[string]string
}{
    data: make(map[string]string),
}

func Lookup(key string) string {
    cache.Lock() // Lock() promoted from sync.Mutex
    defer cache.Unlock()
    return cache.data[key]
}
```

---

## 6.4. Method Values and Method Expressions

Methods can be stored in variables and passed as standard function objects:

### 1. Method Value (`obj.Method`)
Binds a method to a specific instance:
```go
p := Point{1, 2}
distanceFromP := p.Distance // Yields func(Point) float64

fmt.Println(distanceFromP(q)) // p is captured and supplied automatically
```
*Ideal for callbacks*: `time.AfterFunc(time.Second, rocket.Launch)`.

### 2. Method Expression (`T.Method`)
Yields a function where the receiver becomes the explicit first parameter:
```go
dist := Point.Distance    // func(Point, Point) float64
scale := (*Point).ScaleBy // func(*Point, float64)

dist(p, q)
scale(&p, 2)
```

---

## 6.5. Encapsulation: Hiding Implementation Details

* **Encapsulation Unit is the PACKAGE, not the Struct**:
  * Identifiers starting with a **Capital letter** $\to$ Exported (Public).
  * Identifiers starting with a **lowercase letter** $\to$ Unexported (Private to the package).
  * All functions and methods within the same package have unrestricted access to private fields.
* **Go Getter and Setter Conventions**:
  * Getters **omit the `Get` prefix**: use `client.Timeout()`, `logger.Prefix()`.
  * Setters use the standard `Set` prefix: `logger.SetPrefix(p)`.
