# Chapter 3. Basic Data Types

A comprehensive overview of basic data types in Go: signed and unsigned integers, floating-point numbers and special values (`NaN`, `Inf`), boolean logic, strings and UTF-8 internals, runes (`rune`), efficient string building via `bytes.Buffer` and `strings.Builder`, constants, and the `iota` enumerator.

---

### Table of Contents
* [3.1. Integers](#31-integers)
* [3.2. Floating-Point Numbers](#32-floating-point-numbers)
* [3.3. Complex Numbers and Boolean Types](#33-complex-numbers-and-boolean-types)
* [3.4. Strings, UTF-8 Encoding, and Runes](#34-strings-utf-8-encoding-and-runes)
* [3.5. Standard Packages for String Manipulation](#35-standard-packages-for-string-manipulation)
* [3.6. Constants, the `iota` Generator, and Untyped Literals](#36-constants-the-iota-generator-and-untyped-literals)

---

## 3.1. Integers

### Integer Types Breakdown

| Type | Size | Range / Purpose |
| :--- | :--- | :--- |
| `int8` / `uint8` | 1 byte | -128 to 127 / 0 to 255 |
| `int16` / `uint16` | 2 bytes | -32,768 to 32,767 / 0 to 65,535 |
| `int32` / `uint32` | 4 bytes | ~2 billion |
| `int64` / `uint64` | 8 bytes | Large values (~$9 \times 10^{18}$) |
| **`int` / `uint`** | 1 machine word (4 or 8 bytes) | Architecture-dependent (32/64-bit). **Primary integer type**. |
| **`byte`** | 1 byte | Built-in alias for `uint8` representing raw binary data |
| **`rune`** | 4 bytes | Built-in alias for `int32` representing Unicode code points |
| **`uintptr`** | 1 machine word | Unsigned integer sized to hold raw pointer addresses for `unsafe` |

> [!NOTE]
> In Go, `int` and `int64` (or `int32`) are treated as **distinct types by the compiler**, even on 64-bit architectures where their memory representation is identical. Implicit type conversions are never performed.

### Bitwise Operations
* `&` (AND), `|` (OR), `^` (XOR, or unary `^x` for bitwise NOT).
* `&^` — **Bit clear (AND NOT)**: `z = x &^ y` (clears any bits in `x` that are set in `y`).
* `<<` and `>>` — Bitwise shifts (in Go 1.5, shift amounts required unsigned integers; since Go 1.13, any integer type is accepted, though negative shift values panic at runtime or cause compile errors if constant).

> [!TIP]
> **Why `len()` returns a signed `int` rather than an unsigned `uint`**:  
> Using unsigned integers for counting in reverse loops causes infinite loops upon underflow past 0 (`0 - 1 == 18446744073709551615`):
> ```go
> // Safe because i is a signed int; loop terminates cleanly when i becomes -1:
> for i := len(slice) - 1; i >= 0; i-- {
>     // ...
> }
> ```
> *Go convention*: Use unsigned `uint` types exclusively for bitmasks, hashing, and cryptography; use signed `int` for general arithmetic, indexing, and counters.

---

## 3.2. Floating-Point Numbers

Two standard IEEE 754 floating-point types:
* `float32` — ~6 decimal digits of precision.
* `float64` — ~15 decimal digits of precision (**standard default choice**).

### Special Values and `NaN` Comparison Mechanics
* `+Inf` and `-Inf` — Positive and negative infinity (e.g., non-zero division by zero).
* `NaN` (Not a Number) — Undefined mathematical operations (`0.0 / 0.0`, `math.Sqrt(-1)`).

> [!CAUTION]
> **Any comparison involving `NaN` evaluates to `false`** (with the single exception of `!=`, which always evaluates to `true`).
> ```go
> nan := math.NaN()
> fmt.Println(nan == nan) // false
> fmt.Println(nan != nan) // true
> ```
> Always use `math.IsNaN(x)` to test for `NaN`.

---

## 3.3. Complex Numbers and Boolean Types

* **Complex Numbers**: `complex64` (two `float32` values) and `complex128` (two `float64` values). Literals: `z := 1 + 2i`. Component extraction: `real(z)`, `imag(z)`.
* **Booleans (`bool`)**: Can hold only `true` or `false`.
* **Short-Circuit Evaluation**: In `A && B`, `B` is never evaluated if `A` is `false`. This guarantees safety in guards like `if s != "" && s[0] == 'x'`.
* In Go, **there is no implicit conversion from numbers or pointers to boolean** (writing `if 1 { ... }` or `if ptr { ... }` is a compile error).

---

## 3.4. Strings, UTF-8 Encoding, and Runes

### String Internals
* A string in Go is an **immutable sequence of bytes**.
* `len(s)` returns the **number of raw bytes**, not characters.
* Indexing `s[i]` accesses the $i$-th individual byte.
* Slicing substrings `s[a:b]` is an $O(1)$ operation that creates a new header pointing to the existing underlying byte array without memory copying.

```go
// Standard escaped string:
s := "Hello\n"

// Raw string literal in backticks (preserves newlines and avoids escape processing):
raw := `SELECT id, name
        FROM users
        WHERE email = "test@example.com"`
```

### Distinguishing `byte`, `rune`, and `string`
* **UTF-8** is a variable-length encoding (ASCII = 1 byte, Cyrillic/Greek = 2 bytes, CJK/Emoji = 3–4 bytes).
* **Rune (`rune`)** represents a single Unicode code point (an `int32` alias), written in single quotes: `'A'`, `'я'`, `'🚀'`.

```go
s := "Привет"
fmt.Println(len(s))                    // 12 (bytes)
fmt.Println(utf8.RuneCountInString(s)) // 6 (characters/runes)

// A for-range loop over a string automatically decodes UTF-8 rune by rune:
for byteOffset, r := range s {
    fmt.Printf("byte: %d, rune: %c\n", byteOffset, r)
}
```

> [!WARNING]
> Converting `string(65)` yields the Unicode character for code 65, which is `"A"`, **not** the textual string `"65"`.  
> To convert numbers into formatted text, use the `strconv` package.

---

## 3.5. Standard Packages for String Manipulation

* **`strings`**: `Contains`, `HasPrefix`, `Join`, `Split`, `Replace`, `ToLower`, `TrimSpace`.
* **`bytes`**: Mirror functions operating directly on mutable byte slices `[]byte` without allocation overhead.
* **`bytes.Buffer` / `strings.Builder`**: Efficiently accumulating strings without repeatedly creating intermediate heap allocations:
  ```go
  var buf bytes.Buffer
  buf.WriteString("Hello, ")
  buf.WriteString("Go!")
  result := buf.String()
  ```
* **`strconv`**: Converting between primitive types and string representations:
  * Number $\to$ String: `strconv.Itoa(123)` or `fmt.Sprintf("%d", 123)`
  * String $\to$ Number: `x, err := strconv.Atoi("123")` or `strconv.ParseFloat("3.14", 64)`

> [!NOTE]
> **Modern note:** `strings.Builder` was introduced in Go 1.10 (after book publication). For building text strings, it is preferred over `bytes.Buffer` because it avoids copying memory during the final `.String()` call.

---

## 3.6. Constants, the `iota` Generator, and Untyped Literals

Constants (`const`) are strictly evaluated at compile time.

### The `iota` Enumerator
Automatically increments starting from `0` within a `const` group:

```go
// 1. Simple Enum
type Weekday int
const (
    Sunday Weekday = iota // 0
    Monday                // 1
    Tuesday               // 2
    Wednesday             // 3
)

// 2. Bitmasks (powers of 2)
type Flags uint
const (
    FlagUp   Flags = 1 << iota // 1 (0001)
    FlagBroadcast              // 2 (0010)
    FlagLoopback               // 4 (0100)
)

// 3. Storage Unit Multipliers
const (
    _   = 1 << (10 * iota) // Ignore iota == 0
    KiB                    // 1024
    MiB                    // 1,048,576
    GiB                    // 1,073,741,824
)
```

### Untyped Constants
Literals such as `42`, `3.14`, or `math.Pi` possess no fixed type initially and are computed with arbitrary precision by the compiler (at least 256 bits), adopting a concrete type only when assigned:

```go
var a float32 = math.Pi // Inferred as float32
var b float64 = math.Pi // Inferred as float64
```

> [!NOTE]
> Pay attention to integer division in constant expressions:
> * `5 / 9 * (f - 32)` $\to$ `0` (integer division `5 / 9` truncates to `0`).
> * `5.0 / 9.0 * (f - 32)` $\to$ Evaluated correctly with floating-point arithmetic.

> [!NOTE]
> **Modern note (Go 1.13+):** Number literal formats were expanded:
> * Prefixes: Binary `0b1010`, Octal `0o755` (legacy leading zero `0755` also supported), Hexadecimal `0xFF`.
> * Digit separators: `1_000_000`, `0x4B_CC`.
> * Hexadecimal floating-point with binary exponents: `0x1p-2` (= `0.25`).
