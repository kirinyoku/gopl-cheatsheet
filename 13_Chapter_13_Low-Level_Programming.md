# Chapter 13. Low-Level Programming

Low-level systems programming in Go: memory layout and type sizes (`unsafe.Sizeof`), memory alignment and struct field padding optimization, universal `unsafe.Pointer` casting, address arithmetic with `uintptr` and the GC pointer movement trap, deep equality for cyclic graphs, and C interoperability via `cgo`.

---

### Table of Contents
* [13.1. Type Memory Layout: `unsafe.Sizeof`, `Alignof`, and `Offsetof`](#131-type-memory-layout-unsafesizeof-alignof-and-offsetof)
* [13.2. Pointers, `unsafe.Pointer`, and Address Arithmetic](#132-pointers-unsafepointer-and-address-arithmetic)
* [13.3. Deep Equality for Cyclic Data Structures](#133-deep-equality-for-cyclic-data-structures)
* [13.4. Interoperating with C via `cgo`](#134-interoperating-with-c-via-cgo)
* [13.5. The Dangers of `unsafe`](#135-the-dangers-of-unsafe)

---

## 13.1. Type Memory Layout: `unsafe.Sizeof`, `Alignof`, and `Offsetof`

The `unsafe` package is implemented directly by the compiler and provides mechanisms to inspect the physical memory representation of Go data types.

### Standard Type Sizes (`unsafe.Sizeof`)
Calls to `unsafe.Sizeof(x)` are compile-time constants (type `uintptr`) representing the fixed byte size of a type (1 machine word = 8 bytes on 64-bit, 4 bytes on 32-bit):

| Go Data Type | Size in Bytes (64-bit) | Internal Representation |
| :--- | :--- | :--- |
| `bool`, `int8`, `uint8`, `byte` | 1 byte | 1 byte |
| `int16`, `uint16` | 2 bytes | 2 bytes |
| `int32`, `uint32`, `rune` | 4 bytes | 4 bytes |
| `int64`, `uint64`, `float64` | 8 bytes (1 word) | 8 bytes |
| `int`, `uint`, `uintptr`, `*T` | 1 word (8 bytes) | Pointer / machine word |
| `map`, `func`, `chan` | 1 word (8 bytes) | Pointer to heap runtime descriptor |
| `string` | 2 words (16 bytes) | `ptr` (byte array pointer) + `len` |
| `[]T` (slice) | 3 words (24 bytes) | `ptr` + `len` + `cap` |
| `interface{}` / `any` | 2 words (16 bytes) | `type` (type descriptor) + `value` (data pointer) |

---

### Memory Alignment and Padding
Hardware reads memory in multi-byte chunks, requiring data addresses to be multiples of their size (2 for `int16`, 4 for `int32`, 8 for `int64` and pointers). To enforce alignment boundaries, the compiler inserts unused **padding bytes**.

> [!TIP]
> **Optimizing Struct Memory**: Arrange struct fields in descending order of size (largest to smallest) to minimize padding waste:
>
> ```go
> // Suboptimal: 24 bytes (due to alignment padding)
> type Inefficient struct {
>     a bool    // 1 byte + 7 bytes padding
>     b float64 // 8 bytes
>     c int16   // 2 bytes + 6 bytes padding
> }
>
> // Optimal: 16 bytes (tightly packed)
> type Efficient struct {
>     b float64 // 8 bytes
>     c int16   // 2 bytes
>     a bool    // 1 byte (+ 5 bytes trailing padding)
> }
> ```

* `unsafe.Alignof(x)` — Required alignment boundary in bytes for type `x`.
* `unsafe.Offsetof(x.f)` — Byte offset of field `f` from the beginning of struct `x`.

---

## 13.2. Pointers, `unsafe.Pointer`, and Address Arithmetic

`unsafe.Pointer` is an untyped pointer (analogous to `void*` in C) capable of holding the address of any Go variable.

### Valid Conversion Diagram:
$$\text{Typed Pointer } *T \longleftrightarrow \text{unsafe.Pointer} \longleftrightarrow \text{uintptr}$$

* `unsafe.Pointer` values are tracked and updated by the **Garbage Collector (GC)**.
* `uintptr` is an unsigned integer type used to perform numerical address arithmetic (offset addition).

### Idiomatic Pointer Arithmetic:
```go
// Accessing a struct field directly via byte offset:
pb := (*int16)(unsafe.Pointer(uintptr(unsafe.Pointer(&x)) + unsafe.Offsetof(x.b)))
*pb = 42
```

> [!CAUTION]
> **The Deadly Temporary `uintptr` Trap**:  
> ```go
> // BUGGY:
> tmp := uintptr(unsafe.Pointer(&x)) + unsafe.Offsetof(x.b)
> pb := (*int16)(unsafe.Pointer(tmp)) // CORRUPT: x may have moved in memory!
> ```
> During goroutine stack resizing or garbage collection sweeps, valid `unsafe.Pointer` references are updated automatically to reflect moved heap memory. A raw `uintptr` number is **not updated** $\to$ converting it back later points to invalid or reassigned memory!  
> *Rule*: Pointer arithmetic conversions must occur **within a single expression**.

> [!NOTE]
> **Modern note:** Standard Go added type-safe helpers that eliminate manual `uintptr` arithmetic for slice and string conversions: `unsafe.Slice(ptr, len)` and `unsafe.SliceData(s)` (Go 1.17), `unsafe.String(b, len)` and `unsafe.StringData(s)` (Go 1.20).

---

## 13.3. Deep Equality for Cyclic Data Structures

To compare object graphs containing cyclic references (such as `a.next = a`) without entering infinite recursion, visited pointer addresses are recorded in a lookup map:

```go
type comparison struct {
    x, y unsafe.Pointer
    t    reflect.Type
}

seen := make(map[comparison]bool)
```

---

## 13.4. Interoperating with C via `cgo`

```go
package bzip

/*
#cgo CFLAGS: -I/usr/include
#cgo LDFLAGS: -lbz2
#include <stdlib.h>
#include <bzlib.h>
*/
import "C"
import "unsafe"

func CompressInit(stream *C.bz_stream) {
    C.BZ2_bzCompressInit(stream, 9, 0, 30)
}
```

### `cgo` Memory and Type Rules:
1. **String Conversions**:
   * Go to C: `cs := C.CString(goStr)` — **Allocates memory in the C heap**.
   * **Mandatory cleanup**: `defer C.free(unsafe.Pointer(cs))` (requires `#include <stdlib.h>`).
   * C to Go: `goStr := C.GoString(cs)` — Copies bytes into Go-managed memory.
2. **C Types**: Accessible via the `C.` package prefix (`C.int`, `C.uint`, `C.char`).

> [!NOTE]
> **Modern note:** When cross-compiling (`GOOS` different from the host), cgo is **disabled by default**, as it requires a cross-platform C toolchain. Pure Go libraries are preferred; libraries such as `github.com/ebitengine/purego` allow invoking C dynamic libraries without cgo via runtime dynamic symbol loading.

---

## 13.5. The Dangers of `unsafe`

> *«Use unsafe only when strictly necessary, and encapsulate it behind a completely safe public API.»*

### 3 Primary Risks:
1. **Loss of Portability**: Code becomes tied to architecture word sizes (32-bit vs 64-bit) and byte endianness.
2. **Compatibility Breakage**: Internal runtime layouts and compiler conventions are not guaranteed by the Go 1 compatibility promise and may break in future releases.
3. **Memory Corruption**: Erroneous pointer arithmetic causes silent heap corruption, memory leaks, and severe runtime crashes.
