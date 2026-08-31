# Chapter 4. Composite Types

An in-depth guide to composite data structures in Go: fixed-size arrays, dynamic slices and their internal memory layout (`ptr/len/cap`), hash maps (`map`), structs and composition via embedding, JSON serialization with struct tags, and templating with `text/template` and `html/template`.

---

### Table of Contents

* [4.1. Arrays](#41-arrays)
* [4.2. Slices](#42-slices)
* [4.3. Maps (Hash Tables)](#43-maps-hash-tables)
* [4.4. Structs and Composition](#44-structs-and-composition)
* [4.5. JSON Serialization](#45-json-serialization)
* [4.6. Text and HTML Templates](#46-text-and-html-templates)

---

## 4.1. Arrays

An array is a **fixed-length** sequence of elements of a single type. In Go, arrays serve primarily as the low-level backing storage for slices.

### Array Characteristics

* **Length is part of the type**: `[3]int` and `[4]int` are distinct, incompatible types. You cannot assign one to the other or pass mismatched lengths to functions.
* **Pass-by-value**: Passing an array to a function **copies the entire array by value**, not by reference. To avoid copy overhead for large arrays, pass a pointer like `*[32]byte`.
* **Equality**: Arrays can be compared using `==` as long as their element types are comparable.

```go
var a [3]int                  // [0, 0, 0] — zero-initialized
q := [3]int{1, 2, 3}          // Explicit size literal
r := [...]int{1, 2, 3, 4}     // Compiler infers length (4)

// Keyed initialization (unspecified indices default to zero value):
days := [...]string{1: "Mon", 2: "Tue", 7: "Sun"} // Length 8 (indices 0..7)
```

---

## 4.2. Slices

A slice is a lightweight 3-word descriptor referencing a contiguous segment of an **underlying array**.

### Internal Memory Layout

A slice consists of exactly 3 machine words (24 bytes on 64-bit platforms):
1. `ptr` — Pointer to the first element accessible by the slice.
2. `len` — Current number of slice elements (`len(s)`).
3. `cap` — Capacity: the total distance from the slice start to the end of the backing array (`cap(s)`).

```go
s := make([]int, len, cap) // Allocates array and returns slice descriptor
```

```text
Slice s: [ ptr | len | cap ]
           │
           ▼
Array:   [ 0 | 1 | 2 | 3 | 4 | 5 | ... ]
         ├───────────┤
             len (active elements)
         ├─────────────────────────────┤
                      cap (capacity)
```

### Slice Mechanics

* **Shared Backing Array**: Multiple slices can reference overlapping segments of the same underlying array. Modifying elements through one slice is visible in others.
* **Comparison**: Slices **cannot be compared with `==`** (except against `nil`). For byte slices, use `bytes.Equal(a, b)`.

> [!TIP]
> To check if a slice is empty, always evaluate `len(s) == 0` rather than `s == nil`. An empty slice literal `[]int{}` is not `nil`, but its length is `0`.

### Growing Slices with `append`

The built-in `append` function adds elements to the end of a slice:
* If `len < cap` $\to$ The element is written into an unused slot in the existing backing array, and `len` increments.
* If `len == cap` $\to$ A **new, larger backing array is allocated**, existing elements are copied over, and a new slice descriptor is returned.
* **Growth strategy**: Small slices grow by roughly $2\times$, while larger slices transition smoothly to a growth factor of ~$1.25\times$.

```go
s = append(s, 10)
s = append(s, 20, 30)
s = append(s, otherSlice...) // Append slice via unpack operator '...'
```

### Idiomatic In-Place Slice Operations

```go
// 1. Stack (LIFO)
stack = append(stack, v)                // Push
top := stack[len(stack)-1]              // Peek
stack = stack[:len(stack)-1]            // Pop

// 2. Order-preserving removal at index i
func remove(slice []int, i int) []int {
    copy(slice[i:], slice[i+1:])
    return slice[:len(slice)-1]
}

// 3. Fast removal at index i (order not preserved, O(1))
func removeQuick(slice []int, i int) []int {
    slice[i] = slice[len(slice)-1]
    return slice[:len(slice)-1]
}
```

> [!NOTE]
> **Modern note (Go 1.21+):** The standard library includes generic packages `slices` and `maps`:
> * `slices.Equal`, `slices.Clone`, `slices.Delete`, `slices.Insert`, `slices.Index`, `slices.Contains`.
> * `maps.Clone`, `maps.Equal`, `maps.DeleteFunc` (and iterators `maps.Keys` / `maps.Values` added in Go 1.23).

---

## 4.3. Maps (Hash Tables)

`map[KeyType]ValueType` is an unordered collection of key-value pairs with $O(1)$ average-time lookups.

### Key Rules

* **Key Requirements**: The key type must support the `==` comparison operator (integers, strings, booleans, pointers, structs composed of comparable fields). Slices, maps, and functions cannot serve as map keys.
* **No Address-Of Operator**: Taking the address of a map value (`&m["key"]`) is **prohibited** because hash table growth can relocate items in memory.
* **Unordered Iteration**: `for k, v := range m` yields keys in non-deterministic random order. To sort keys, copy them into a slice and sort via `sort.Strings()` (or `slices.Sort()`).

```go
// Creation:
ages := make(map[string]int)
ages["alice"] = 25

// Lookup and deletion:
val, ok := ages["bob"] // ok == false if key is absent
delete(ages, "alice")  // Safe to call even if key does not exist
```

> [!CAUTION]
> **The `nil map` Trap**:  
> Reading from an uninitialized map `var m map[string]int` (where `m == nil`) is safe and yields the value type's zero value. However, **writing to a `nil map` triggers a runtime panic**! Always initialize maps with `make` or `{}` before writing.

### Implementing Sets with `struct{}`

```go
seen := make(map[string]struct{})
seen["item"] = struct{}{} // struct{} takes zero bytes of memory

if _, ok := seen["item"]; ok {
    // Item is present in set
}
```

---

## 4.4. Structs and Composition

A struct groups heterogeneous fields together into a single contiguous block of memory:

```go
type Employee struct {
    ID        int
    Name      string
    Salary    int
    ManagerID int
}

p := &Employee{ID: 1, Name: "Ivan"}
fmt.Println(p.Name) // Automatic pointer dereferencing (instead of (*p).Name)
```

### Composition via Embedding (Anonymous Fields)

Go eschews inheritance in favor of **composition through anonymous struct embedding**:

```go
type Point struct {
    X, Y int
}

type Circle struct {
    Point  // Anonymous field (embedding)
    Radius int
}

type Wheel struct {
    Circle // Embedding Circle, which itself embeds Point
    Spokes int
}
```

* **Field Promotion**: Fields of embedded types are promoted and directly accessible on the outer struct:
  ```go
  var w Wheel
  w.X = 10     // Equivalent to w.Circle.Point.X = 10
  w.Radius = 5 // Equivalent to w.Circle.Radius = 5
  ```

---

## 4.5. JSON Serialization

The standard `encoding/json` package serializes Go structs to JSON (marshaling) and parses JSON into structs (unmarshaling).

```go
type Movie struct {
    Title  string   `json:"title"`           // Field name in JSON
    Year   int      `json:"released"`
    Color  bool     `json:"color,omitempty"` // Omits field if empty / zero value
    Actors []string `json:"actors"`
    Secret string   `json:"-"`               // Completely excludes field from JSON
}
```

### JSON Processing Guidelines

* **Exported Fields Only**: Unexported fields (starting with a lowercase letter) are silently ignored by the JSON encoder.
* **Marshaling (Go $\to$ JSON)**:
  `data, err := json.Marshal(movie)` or indented `json.MarshalIndent(movie, "", "  ")`.
* **Unmarshaling (JSON $\to$ Go)**:
  `err := json.Unmarshal(data, &targetStruct)` (always pass a pointer target).
* **Streaming I/O**:
  `json.NewDecoder(r).Decode(&v)` and `json.NewEncoder(w).Encode(v)` stream data directly to/from I/O readers and writers without buffering complete payloads in memory.

> [!NOTE]
> **Modern note:** In standard Go, `omitempty` cannot omit zero-value struct types like `time.Time{}`. Go 1.24 introduced `json:"...,omitzero"` to omit any field matching its type's zero value. Furthermore, Go 1.25 provides an experimental engine (`GOEXPERIMENT=jsonv2`, package `encoding/json/v2`) while preserving backwards compatibility.

---

## 4.6. Text and HTML Templates

The `text/template` and `html/template` packages inject structured data into templates:

* `{{.}}` — The current context cursor.
* `{{.Field}}` — Accesses a struct field or map key.
* `{{range .Items}} ... {{end}}` — Iterates through collections.
* `{{if .Condition}} ... {{else}} ... {{end}}` — Conditional branches.
* `{{.Title | printf "%.30s"}}` — Pipelines for chaining processing functions.

> [!NOTE]
> **Contextual Auto-Escaping in `html/template`**:  
> The `html/template` package contextually sanitizes special characters (`<`, `>`, `&`, `"`) across HTML, JavaScript, CSS, and URL contexts to guard against Cross-Site Scripting (XSS) vulnerabilities. To output pre-trusted raw HTML without sanitization, use the `template.HTML` type.
