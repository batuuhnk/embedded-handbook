# 01 · Embedded C

> C features that behave differently, or matter more, when the code runs on a microcontroller. Each topic covers what the feature does, when I use it, and what can go wrong.

**Contents:** [volatile](#volatile) · [const](#const) · [static](#static) · [Struct padding & bit-fields](#struct-padding-and-bit-fields) · [Pointers & function pointers](#pointers-and-function-pointers) · [Memory layout](#memory-layout) · [Macro vs inline](#macro-vs-inline-function) · [Undefined behaviour](#undefined-behaviour) · [MISRA C](#misra-c) · [Endianness](#endianness) · [Fixed-width types](#fixed-width-types-stdinth)

---

## volatile

`volatile` tells the compiler that the value can change in ways it cannot see, so it must not optimise accesses away. Every read and write in the source becomes a real memory access.

**Required for:**
- **Memory-mapped hardware registers.** Without it, the compiler may read a status register once and reuse a stale value.
- **Variables shared with an ISR.** A flag set in an interrupt and polled in a loop can be "optimised" into an infinite loop, because the loop body never appears to change it.

**What it does *not* do:** `volatile` gives no atomicity. A 32-bit (or larger) shared value can still be torn by an interrupt on some architectures, and read-modify-write sequences (`x++`) are never atomic. Shared data larger than a single naturally-aligned word needs a critical section or a mutex.

> [!WARNING]
> **Classic symptom:** works in Debug (`-O0`), fails in Release. Missing `volatile` on an ISR-shared variable is the first thing I check.

---

## const

`const` makes accidental writes a compile-time error. On embedded targets it also decides **where data lives**: large `const` tables (lookup tables, calibration defaults) are normally placed in Flash instead of RAM by the linker.

`const` and `volatile` can be combined. A read-only hardware status register is both:

```c
const volatile uint32_t *const STATUS = (const volatile uint32_t *)0x40021000u;
```

| Declaration | What is constant |
|---|---|
| `const int *p` | the pointed-to value |
| `int *const p` | the pointer itself |
| `const int *const p` | both |

---

## static

`static` means different things in different places:

1. **Local variable inside a function:** keeps its value between calls. It lives in `.data`/`.bss`, not on the stack.
2. **Variable or function at file scope:** *internal linkage*, visible only inside that `.c` file.

I use file-scope `static` for module encapsulation: private state and helpers stay inside the module, names cannot collide, and only the module's own functions can touch its state.

**Risk:** a function-local `static` makes the function **non-reentrant**. If two tasks (or a task and an ISR) call it, they share hidden state. See [reentrancy](02-rtos-and-concurrency.md#reentrancy).

---

## Struct padding and bit-fields

### Padding

Compilers align members to their natural size and insert padding between them:

```c
struct example {      // naive expectation: 6 bytes
    uint8_t  a;       // 1 + 3 padding
    uint32_t b;       // 4
    uint8_t  c;       // 1 + 3 trailing padding
};                    // sizeof == 12 on a typical 32-bit target
```

This matters when a struct is cast straight onto a byte buffer for a protocol frame. Padding (and the other side's different compiler or architecture) silently corrupts the data.

**Options:**
- **Order members largest → smallest.** This removes most padding at no cost.
- **`__attribute__((packed))` / `#pragma pack`.** This removes all padding, but unaligned access is slower on some cores and faults on others. I use it only for wire-format structs, never as a default.
- **Best for portability:** explicit serialise/deserialise functions that write bytes one by one.

### Bit-fields

Bit-fields (`uint8_t flag : 1;`) are convenient for register maps and packed flags, but the **bit order and layout are implementation-defined**. When code must be portable across compilers or architectures, I use explicit masks and shifts instead:

```c
#define CTRL_EN_Pos   (0u)
#define CTRL_EN_Msk   (1u << CTRL_EN_Pos)

reg |=  CTRL_EN_Msk;          /* set   */
reg &= ~CTRL_EN_Msk;          /* clear */
```

---

## Pointers and function pointers

- **Dangling pointers.** Avoiding `malloc` removes most of these. Returning the address of a local variable is still a common source.
- **NULL dereference.** On Cortex-M this usually ends in a HardFault. I check pointer parameters at module boundaries, as MISRA also expects.

**Function pointers** are how C does polymorphism. I use them for:
- **Callbacks.** A driver notifies "transfer complete" without knowing who is listening.
- **Driver interfaces.** A `struct` of function pointers lets the same upper layer run on different hardware.
- **State machines and jump tables.** Adding a state becomes one table row instead of another `case` in a growing `switch`.

```c
typedef struct {
    int32_t (*init)(void);
    int32_t (*read)(uint8_t *buf, uint16_t len);
    int32_t (*write)(const uint8_t *buf, uint16_t len);
} bus_ops_t;
```

**Risk:** an uninitialised or out-of-range table entry jumps to garbage. Every call through a table gets a bounds and NULL check.

---

## Memory layout

| Section | Contents | Where |
|---|---|---|
| `.text` | code, `const` data | Flash |
| `.data` | initialised globals/statics | stored in Flash, copied to RAM at startup |
| `.bss` | zero-initialised globals/statics | RAM, zeroed at startup (no Flash cost) |
| heap | `malloc`/`free` | RAM, usually avoided |
| stack | locals, call frames (one per RTOS task) | RAM |

Knowing this turns into practical memory work. For example, I make a 4 KB lookup table `const` to move it to Flash, and keep large buffers out of task stacks.

---

## Macro vs inline function

| | Macro | `static inline` function |
|---|---|---|
| Type checking | none | full |
| Side effects (`SQ(i++)`) | argument evaluated multiple times | evaluated once |
| Debugging | invisible to the debugger | steppable |
| Compile-time constants, `#ifdef` | ✔ | ✘ |

Classic macro bug: `#define SQ(x) x*x` → `SQ(a+b)` expands to `a+b*a+b`. If a macro is unavoidable, I parenthesise everything: `((x)*(x))`.

**Rule I follow:** function-like logic goes in `static inline` functions. Macros are only for constants, conditional compilation and register bit definitions.

---

## Undefined behaviour

Undefined behaviour (UB) is code whose result the C standard does not define. It may "work" until the optimisation level, compiler version or timing changes, and on embedded targets that failure tends to appear in the field, rarely and without a way to reproduce it.

Common sources:
- **Signed integer overflow** (`INT32_MAX + 1`). Unsigned overflow is defined: it wraps.
- **Reading an uninitialised variable.**
- **Modifying a variable twice without a sequence point** (`i = i++ + ++i`).
- **Out-of-bounds array access.** This is the most common and most dangerous case, because nothing traps on a bare MCU.
- **Shifting by ≥ the type width**, or shifting a negative value left.

In safety-critical work, avoiding UB is a verification requirement, not just style: if behaviour is undefined, no test can define the expected result.

---

## MISRA C

MISRA C restricts the parts of C that are undefined, implementation-defined or easy to misuse, so that the code becomes predictable and analysable. The rule categories I find most valuable in practice (MISRA C:2012):

- **No dynamic memory** (Rule 21.3). It avoids fragmentation and non-deterministic timing.
- **Explicit, well-defined type conversions** (essential type model, Rules 10.x). Silent signed/unsigned mixing is a frequent source of field bugs.
- **Every `switch` has a `default`, no silent fall-through** (Rules 16.x).
- **Restricted `goto`** (Rule 15.1, advisory) and a **single point of exit** (Rule 15.5, advisory). Both keep control flow traceable.
- **Fixed-width types** instead of plain `int`/`long` (Directive 4.6).
- **Bounded complexity.** Simpler functions are cheaper to test and cover.

I treat MISRA as a team agreement backed by a static analyser, not a checklist applied at the end.

---

## Endianness

- **Little-endian:** least-significant byte at the lowest address. ARM Cortex-M is little-endian by default.
- **Big-endian:** most-significant byte first. This is "network byte order", and many avionics and industrial interfaces use it.

Casting a struct onto a byte buffer bakes in the host's endianness *and* its padding. Explicit pack/unpack helpers avoid both problems:

```c
static inline void put_be16(uint8_t *p, uint16_t v) { p[0] = (uint8_t)(v >> 8); p[1] = (uint8_t)v; }
static inline uint16_t get_be16(const uint8_t *p)  { return (uint16_t)((p[0] << 8) | p[1]); }
```

---

## Fixed-width types (`stdint.h`)

`int` can be 16 bits on one target and 32 bits on another. For anything where size matters (register maps, protocol fields, EEPROM records) I use `uint8_t`, `int16_t`, `uint32_t` and so on. Plain `int`/`long` appear only where the exact size truly does not matter.

---

### Quick reference

| Topic | One-line takeaway |
|---|---|
| `volatile` | Stops the compiler optimising accesses away; provides no atomicity |
| `const volatile` | Read-only to software, changeable by hardware |
| `static` | Persistence inside functions, encapsulation at file scope |
| Padding | Order members by size; use `packed` only for wire formats |
| Bit-fields | Layout is implementation-defined; use masks when portability matters |
| Macro vs inline | Use `static inline` for logic; keep macros for constants and `#ifdef` |
| UB | Most MISRA rules exist to remove it |
| `stdint.h` | Use exact-width types wherever size matters |

[← Back to index](../README.md) · [Next: RTOS & Concurrency →](02-rtos-and-concurrency.md)
