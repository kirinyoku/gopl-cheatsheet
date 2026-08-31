# Chapter 12. Reflection

Metaprogramming in Go: the necessity of reflection, `reflect.Type` and `reflect.Value`, distinguishing exact `Type` from structural `Kind`, recursive data structure traversal, addressability (`CanAddr`) and settability (`CanSet`), mutating variables through pointers with `Elem()`, unexported field mutation restrictions, parsing struct field tags, dynamic method inspection, and the 3 operational costs of reflection.

---

### Table of Contents

* [12.1. Why Reflection Is Needed](#121-why-reflection-is-needed)
* [12.2. Fundamental Types: `reflect.Type` and `reflect.Value`](#122-fundamental-types-reflecttype-and-reflectvalue)
* [12.3. Navigating Composite Data Structures](#123-navigating-composite-data-structures)
* [12.4. Setting and Mutating Variables via `reflect.Value`](#124-setting-and-mutating-variables-via-reflectvalue)
* [12.5. Accessing Struct Field Tags](#125-accessing-struct-field-tags)
* [12.6. Inspecting Type Methods](#126-inspecting-type-methods)
* [12.7. The Costs and Caveats of Reflection](#127-the-costs-and-caveats-of-reflection)

---

## 12.1. Why Reflection Is Needed

Reflection enables a program to **inspect and manipulate types, values, and memory representations dynamically at runtime** without knowing concrete types at compile time.

### Primary Use Cases:

* Generic serialization and deserialization (`encoding/json`, `encoding/xml`, S-expressions).
* Universal formatting of arbitrary values (`fmt.Printf("%v", x)`).
* Template engines (`text/template`, `html/template`).
* Automatic HTTP request binding into struct fields.

> [!NOTE]
> A `switch x.(type)` construct cannot replace reflection, because it is impossible to statically enumerate the infinite universe of composite slices, maps, and struct types.

> [!NOTE]
> **Modern note (Go 1.18+):** Generics now handle many use cases that previously demanded reflection — offering complete compile-time type safety with zero runtime overhead (e.g., `slices`, `maps`, generic containers). Reflection remains essential when types are known only at runtime (tag-based serializers, ORMs, plugin architectures).

---

## 12.2. Fundamental Types: `reflect.Type` and `reflect.Value`

```go
import "reflect"

t := reflect.TypeOf(3)    // reflect.Type (dynamic type descriptor, "int")
v := reflect.ValueOf(3)   // reflect.Value (dynamic value descriptor, 3)

t2 := v.Type()            // Obtains reflect.Type from reflect.Value
x := v.Interface().(int)  // Unpacking: reflect.Value -> any -> int
```

### Distinguishing `Type` from `Kind`:

* **`Type`** describes the exact concrete or named type (`time.Duration`, `*os.File`, `main.Movie`).
* **`Kind`** describes the underlying structural category (`reflect.Int64`, `reflect.Pointer`, `reflect.Struct`, `reflect.Slice`, `reflect.Map`, `reflect.Interface`, `reflect.Invalid`).

```go
var d time.Duration = 1 * time.Second
v := reflect.ValueOf(d)

fmt.Println(v.Type()) // "time.Duration"
fmt.Println(v.Kind()) // reflect.Int64
```

---

## 12.3. Navigating Composite Data Structures

Core `reflect.Value` methods for recursive inspection:

| Structural Category (`Kind`) | Methods on `reflect.Value` |
| :--- | :--- |
| **`Slice`, `Array`** | `v.Len()`, `v.Index(i)` (fetches the $i$-th element) |
| **`Struct`** | `v.NumField()`, `v.Field(i)`, `v.Type().Field(i).Name` |
| **`Map`** | `v.MapKeys()`, `v.MapIndex(key)`, `v.SetMapIndex(key, val)` |
| **`Pointer`, `Interface`** | `v.IsNil()`, `v.Elem()` (dereferences pointer or unpacks interface payload) |

> [!WARNING]
> When traversing arbitrary object graphs recursively (such as cyclic linked structures), failing to track visited pointer addresses will trigger **infinite recursion and a stack overflow**.

---

## 12.4. Setting and Mutating Variables via `reflect.Value`

To modify a variable through reflection, its `reflect.Value` must be **Addressable** (`CanAddr`) and **Settable** (`CanSet`):

```go
x := 2

// WRONG: Value passed by copy
v := reflect.ValueOf(x)
// v.CanAddr() == false -> v.SetInt(3) triggers a RUNTIME PANIC!

// CORRECT: Pass a pointer and dereference with Elem()
v = reflect.ValueOf(&x).Elem()
fmt.Println(v.CanAddr()) // true
fmt.Println(v.CanSet())  // true

v.SetInt(3) // Updates x to 3
fmt.Println(x) // 3
```

> [!CAUTION]
> **Unexported Field Mutation Restriction**:  
> Reflection can **read** unexported struct fields, but attempting to **write** to them (`v.Field(i).Set(...)`) triggers a **runtime panic**. For unexported private fields, `v.CanSet()` always evaluates to `false`.

---

## 12.5. Accessing Struct Field Tags

```go
type SearchParams struct {
    Query      string `http:"q"`
    MaxResults int    `http:"max"`
    Exact      bool   `http:"x"`
}

func Unpack(req *http.Request, ptr any) error {
    v := reflect.ValueOf(ptr).Elem() // Struct behind pointer
    for i := 0; i < v.NumField(); i++ {
        fieldInfo := v.Type().Field(i)   // reflect.StructField
        tag := fieldInfo.Tag.Get("http") // Parses value for key "http"

        if val := req.FormValue(tag); val != "" {
            // Parse string val and call v.Field(i).SetString/SetInt/...
        }
    }
    return nil
}
```

* In addition to `Tag.Get(name)`, `Tag.Lookup(name) (value string, ok bool)` (introduced in Go 1.7) distinguishes between an empty tag and an absent tag.

---

## 12.6. Inspecting Type Methods

Reflection allows discovering all exported methods declared on a type and invoking them dynamically:

```go
t := reflect.TypeOf(time.Hour)
for i := 0; i < t.NumMethod(); i++ {
    m := t.Method(i)
    fmt.Printf("Method: %s, Signature: %s\n", m.Name, m.Type)
}
```

---

## 12.7. The Costs and Caveats of Reflection

> *«Reflection is a powerful tool, but it should be used with extreme care and moderation.»*

### 3 Major Drawbacks:

1. **Fragility**: Type mismatches are invisible to the compiler and result in fatal runtime `panic` failures.
2. **Reduced Readability and Maintainability**: Reflected code obscures data flow; IDE refactoring and static analysis tools cannot reliably track reflective field and method lookups.
3. **Performance Overhead**: Reflection operations are typically **1 to 2 orders of magnitude slower** than direct statically compiled Go code. Avoid reflection in hot, performance-critical code paths.
