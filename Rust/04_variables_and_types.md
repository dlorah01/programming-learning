---
title: "Lesson 04 — Variables, mutability, shadowing, primitives, and numeric behavior"
---

# Lesson 04 — Variables, mutability, shadowing, primitives, and numeric behavior

**Module 2 · Syntax and fundamentals** · Status: generated · Edition used in examples: **2024** · External dependencies: none

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux. As before, ✓ marks outputs and diagnostics I actually ran; unmarked statements are documented behavior I did not run. Sizes and `usize` limits below are for a **64-bit** target.

Builds on [Lesson 01](01_foundations.md) (overflow semantics, UB) and [Lesson 03](03_cargo_projects.md) (profiles: debug vs release).

---

## 1. `let`, immutability, and definite initialization

```rust
fn main() {
    let x = 5;          // immutable binding
    let mut y = 5;      // mutable binding
    y += 1;
    let z;              // declared, not yet initialized
    if y > 3 { z = "big"; } else { z = "small"; }
    println!("{x} {y} {z}");
}
```

✓ prints `5 6 big`.

**Design rationale.** Bindings are immutable by default because most values are written once. When a reader sees `let x`, they know `x` never changes, which makes code, and the borrow checker's reasoning in later lessons, much simpler. Mutation is still available, but you must say so (`mut`), which makes it visible in review.

Note what immutability means: the *binding* cannot be reassigned or mutably borrowed. It says nothing about whether a type has interior mutability (Lesson 24).

### 1.1 Assigning to an immutable binding (E0384)

```rust
fn main() {
    let x = 5;
    x = 6;
    println!("{x}");
}
```

✓ rustc 1.97.0 reports `error[E0384]: cannot assign twice to immutable variable `x``. (It also warns that the first value is never read.) Fix: `let mut x = 5;`, or better, ask whether you need to mutate at all.

### 1.2 Definite initialization (E0381)

A binding may be declared first and initialized later, but the compiler performs **definite-assignment analysis**: every path from the declaration to a use must have assigned it.

```rust
fn main() {
    let n: i32;
    let flag = std::env::args().count() > 5;
    if flag { n = 1; }
    println!("{n}");
}
```

✓ `error[E0381]: used binding `n` is possibly-uninitialized`. The diagnostic points at the `if` and notes "an `else` arm might be missing here". Fix: initialize on both branches, or write `let n = if flag { 1 } else { 0 };`.

Why this matters: in C, reading an uninitialized local is undefined behavior or an indeterminate value. In safe Rust it is a compile error, so no safe program can observe uninitialized stack memory. (Deliberately uninitialized memory exists, but only through `unsafe` and `MaybeUninit`, Lesson 37.)

A binding assigned exactly once after declaration can stay immutable, as `z` above shows. That is the idiomatic way to initialize conditionally.

---

## 2. Shadowing versus `mut`

```rust
fn main() {
    let input = "  42  ";
    let input = input.trim();          // &str -> &str
    let input: u32 = input.parse().unwrap(); // &str -> u32 (new type!)
    println!("{input}");
    let x = 1;
    {
        let x = x + 10;
        println!("inner {x}");
    }
    println!("outer {x}");
}
```

✓ prints `42`, `inner 11`, `outer 1`.

`let` can **shadow** a name: it creates a *new* variable that happens to reuse the name. The old one still exists until its scope ends (it is just unreachable by that name).

| | `mut` | shadowing |
|---|---|---|
| Same variable? | yes, same memory location | no, a new variable |
| Can change type? | no | yes |
| Result immutable afterwards? | no | yes (unless you write `let mut`) |
| Typical use | accumulators, loop state | step-by-step transformation (`trim`, then `parse`) |

`mut` does **not** let you change a type:

```rust
fn main() {
    let mut s = "hello";
    s = s.len();
    println!("{s}");
}
```

✓ `error[E0308]: mismatched types ... expected `&str`, found `usize``.

**Idiom.** Shadow when you refine a value through stages and the old version is no longer needed. Use `mut` when the same logical value evolves. Avoid shadowing in long functions where a reader could confuse two different variables: choose new names.

**Compiler view.** Shadowing is purely a name-resolution matter; after resolution each `let` is a separate local with its own stack slot (or register). There is no runtime cost.

---

## 3. Constants and statics

```rust
use std::sync::atomic::{AtomicU32, Ordering};

const MAX_USERS: u32 = 1_000;
const SECS_PER_DAY: u64 = 24 * 60 * 60;
const fn square(n: u32) -> u32 { n * n }
const SQ: u32 = square(12);
static GREETING: &str = "hi";
static COUNTER: AtomicU32 = AtomicU32::new(0);

fn main() {
    COUNTER.fetch_add(1, Ordering::Relaxed);
    println!("{MAX_USERS} {SECS_PER_DAY} {SQ} {GREETING} {}", COUNTER.load(Ordering::Relaxed));
}
```

✓ prints `1000 86400 144 hi 1`.

| | `const` | `static` |
|---|---|---|
| Type annotation | required | required |
| Identity | no address; inlined at each use | one fixed memory location for the whole program (`'static`) |
| Mutable? | never | only via interior mutability (`AtomicU32`, `Mutex` ...) or `static mut` (avoid) |
| Initializer | constant expression, evaluated at compile time | constant expression, evaluated at compile time |

Conventions: `SCREAMING_SNAKE_CASE`. Use `const` by default; use `static` when you need a single address (for example a lookup table you want only once in the binary, or a global atomic).

`const fn` marks a function callable in constant evaluation. The set of operations allowed in `const fn` has grown over releases; check the Reference for what your minimum supported Rust version permits.

### 3.1 Constant evaluation catches overflow at compile time

```rust
const X: u8 = 255 + 1;
fn main() { println!("{X}"); }
```

✓ `error[E0080]: attempt to compute `u8::MAX + 1_u8`, which would overflow`. Unlike runtime arithmetic, constant evaluation is the same in debug and release: overflow is always an error.

Overflow that the compiler can see in ordinary code is also rejected, by a lint that is deny-by-default:

```rust
fn main() { let x: u8 = 255 + 1; println!("{x}"); }
```

✓ `error: this arithmetic operation will overflow` (lint `arithmetic_overflow`). Lints are heuristics, though: they catch obvious cases, not data-dependent overflow (Section 6).

### 3.2 `static mut` and edition 2024

```rust
static mut N: u32 = 0;
fn main() {
    unsafe { N += 1; }
    let r = unsafe { &N };
    println!("{r}");
}
```

✓ In edition 2024 this is a **hard error**: `creating a shared reference to mutable static`, from `#[deny(static_mut_refs)]`, with the note that it is undefined behavior if the static is mutated or a mutable reference is created while the shared reference lives. The compiler suggests `&raw const N`.

Why: any access to a `static mut` is `unsafe` because nothing prevents two threads (or two overlapping references) from aliasing it. References to it make aliasing mistakes easy to commit and hard to see. With a raw pointer, no reference is created:

```rust
#![allow(static_mut_refs)]
static mut N: u32 = 0;
fn main() { unsafe { N += 1; } println!("{}", unsafe { *&raw const N }); }
```

✓ prints `1`. (The `allow` is only needed on older editions where it is a warning; shown here for completeness.) The preferred fix is not clever pointers but a safe type: an `AtomicU32`, a `Mutex`, or `OnceLock` (Lessons 24 and 28). Global mutable state is also a design smell: it hides dependencies and makes testing hard.

---

## 4. Primitive types

### 4.1 Integers

| Signed | Unsigned | Bits |
|---|---|---|
| `i8` | `u8` | 8 |
| `i16` | `u16` | 16 |
| `i32` | `u32` | 32 |
| `i64` | `u64` | 64 |
| `i128` | `u128` | 128 |
| `isize` | `usize` | pointer width (64 on x86-64) |

Signed integers use **two's complement**, so the range is asymmetric: one more negative value than positive. ✓ Ranges on this machine:

```text
i8 -128..127                       u8 0..255
i16 -32768..32767                  u16 0..65535
i32 -2147483648..2147483647        u32 0..4294967295
i64 -9223372036854775808..9223372036854775807
u64 0..18446744073709551615
i128 -170141183460469231731687303715884105728..170141183460469231731687303715884105727
u128 0..340282366920938463463374607431768211455
usize 0..18446744073709551615 (64-bit)
```

If a literal's type is not constrained by context, an integer literal defaults to **`i32`** and a float literal to **`f64`**. `usize`/`isize` are used for sizes and indexing, because they are the width of an address.

### 4.2 Literals

```rust
fn main() {
    let a = 1_000_000;    // underscores are ignored
    let b = 0xff;         // hex
    let c = 0o77;         // octal
    let d = 0b1010_1010u8;// binary with a type suffix
    let e = b'A';         // byte literal: u8
    let f = 2.5e3;        // f64
    let g = 7u64;
    let h = 1.5f32;
    println!("{a} {b} {c} {d} {e} {f} {g} {h}");
}
```

✓ prints `1000000 255 63 170 65 2500 7 1.5`.

An out-of-range literal is rejected (deny-by-default lint `overflowing_literals`):

```rust
fn main() { let x: u8 = 256; println!("{x}"); }
```

✓ `error: literal out of range for `u8``, "the literal `256` does not fit into the type `u8` whose range is `0..=255`".

### 4.3 `bool`, `char`, unit, never

- `bool`: `true` or `false`, 1 byte. No implicit conversion from integers: `if 1 {}` is an error.
- `char`: a **Unicode scalar value** (any code point except surrogates `U+D800..=U+DFFF`), always 4 bytes. It is *not* a byte and not a user-perceived character: `é` can be one `char` or two (`e` plus a combining accent).
- `()` (unit): the type with exactly one value, zero bytes. It is what a function without a return type returns.
- `!` (never): the type of expressions that never produce a value (`panic!`, `return`, `loop {}`). It coerces to any type, which is why `let n: u32 = match s { ... => panic!() }` type-checks.

✓ sizes on x86-64 (from `size_of`):

```text
bool 1  char 4  () 0  u8 1  u16 2  u32 4  u64 8  u128 16
usize 8  f32 4  f64 8  &u8 8  &str 16  Option<u8> 2  Option<&u8> 8
```

Note `Option<&u8>` is 8 bytes, the same as `&u8`: references are never null, so `None` uses the null pattern (a *niche*, Lesson 01; layouts in Lesson 09).

Bytes versus chars:

```rust
fn main() {
    let s = "héllo €";
    println!("{} bytes, {} chars", s.len(), s.chars().count());
    for c in ['a', 'é', '€', '😀'] {
        println!("{c} {} {:#x}", c.len_utf8(), c as u32);
    }
    println!("{:?}", char::from_u32(0x1F600));
}
```

✓ output:

```text
10 bytes, 7 chars
a 1 0x61
é 2 0xe9
€ 3 0x20ac
😀 4 0x1f600
Some('😀')
```

`str::len` is the UTF-8 byte length. Full string handling is Lesson 06.

### 4.4 Floating point

`f32` and `f64` follow IEEE 754 binary floating point. This is where intuition from integers fails.

```rust
fn main() {
    println!("{}", 0.1 + 0.2);
    println!("{}", 0.1 + 0.2 == 0.3);
    println!("{}", ((0.1f64 + 0.2) - 0.3).abs() < f64::EPSILON);
    let nan = f64::NAN;
    println!("{} {} {}", nan == nan, nan < 1.0, nan.is_nan());
    println!("{} {} {}", 1.0 / 0.0, -1.0 / 0.0, 0.0 / 0.0);
    println!("{} {}", 0.0 == -0.0, (-0.0f64).is_sign_negative());
    let mut v = vec![3.5, -1.0, f64::NAN, 2.0];
    v.sort_by(|a, b| a.total_cmp(b));
    println!("{v:?}");
    println!("{}", 16_777_217i32 as f32);
    println!("{:?}", 1.0f64.partial_cmp(&f64::NAN));
}
```

✓ output:

```text
0.30000000000000004
false
true
false false true
inf -inf NaN
true true
[-1.0, 2.0, 3.5, NaN]
16777216
None
```

Lessons:

- `0.1 + 0.2 != 0.3` because most decimal fractions are not exactly representable in binary. Never use `==` on computed floats; compare with a tolerance suited to the magnitudes involved (the `f64::EPSILON` check above is only appropriate near 1.0; scale it for other magnitudes).
- Float division by zero does **not** panic: it gives `inf`, `-inf`, or `NaN`. Integer division by zero panics (Section 5).
- `NaN` is unequal to everything, itself included. So floats implement `PartialEq`/`PartialOrd` but **not** `Eq`/`Ord`. Consequences: `f64` cannot be a `HashMap` key and `Vec<f64>::sort()` does not compile:

```rust
fn main() { let mut v = vec![2.0, 1.0]; v.sort(); println!("{v:?}"); }
```

✓ `error[E0277]: the trait bound `{float}: Ord` is not satisfied`. Use `sort_by(|a, b| a.total_cmp(b))` (a total order that places NaN last, as shown above) or `partial_cmp` with explicit NaN handling.
- `16_777_217` (2²⁴+1) cannot be represented in `f32`, so the cast rounds to `16777216`: `f32` has a 24-bit significand.

**Money, identifiers and amounts.** Never use floats for currency or token amounts. Use integers in the smallest unit (cents, lamports, wei) with checked arithmetic, which is the pattern blockchain code requires (Lessons 48–55).

---

## 5. Integer arithmetic edge cases

```rust
use std::hint::black_box;
fn main() {
    println!("{} {} {} {}", 7 / 2, -7 / 2, 7 % 3, -7 % 3);
    println!("{} {}", (-7i32).rem_euclid(3), (-7i32).div_euclid(3));
    println!("{:?} {:?}", 5i32.checked_div(0), 5i32.checked_div(2));
    println!("{:?}", i32::MIN.checked_div(-1));
    println!("{:?} {:?}", i32::MIN.checked_abs(), (-5i32).checked_abs());
    println!("{:?} {:?}", 1u32.checked_shl(31), 1u32.checked_shl(32));
    println!("{}", i32::MIN.wrapping_abs());
    println!("{}", 200u8.saturating_add(100));
    println!("{:?}", 200u8.overflowing_add(100));
    let z = black_box(0);
    println!("{}", 10 / z);
}
```

✓ output (the last line panics):

```text
3 -3 1 -1
2 -3
None Some(2)
None
None Some(5)
Some(2147483648) None
-2147483648
255
(44, true)
thread 'main' panicked at e06_arith.rs:13:20:
attempt to divide by zero
```

Points to remember:

- **Division truncates toward zero**, and `%` takes the sign of the dividend: `-7 / 2 == -3`, `-7 % 3 == -1`. For a non-negative modulus use `rem_euclid` (`2`) and the matching `div_euclid` (`-3`). This matters for hashing, ring buffers, and wrap-around indexing.
- **Integer division or remainder by zero panics** in every build mode. `i32::MIN / -1` also overflows (the true result 2³¹ does not fit) and panics in every build mode. `checked_div` returns `None` for both. (`std::hint::black_box` is used above only so the compiler cannot see the zero and reject the program at compile time.)
- `abs` of `MIN` overflows for the same reason; `checked_abs` returns `None`.
- Shifting by the bit width or more is an overflow: debug panics, release masks the shift amount; `checked_shl` returns `None`. Prefer the explicit method.
- The four families (Lesson 01): `checked_*` → `Option`; `wrapping_*` → modulo 2ⁿ; `saturating_*` → clamp; `overflowing_*` → value plus flag (`200u8 + 100` → `(44, true)`).

### 5.1 `usize` and the `len() - 1` trap

```rust
fn last_index(v: &[i32]) -> usize { v.len() - 1 }
fn main() {
    let v: Vec<i32> = Vec::new();
    println!("{:?}", v.len().checked_sub(1));
    println!("{}", last_index(&v));
}
```

✓ In a **debug** build: `None`, then `panicked ... attempt to subtract with overflow`. In a **release** build (`-O`, overflow checks off by default): `None`, then `18446744073709551615`, a wrapped value that will later be used as an index or loop bound. Same source, two behaviors. Unsigned subtraction near zero is one of the most common sources of off-by-one and out-of-bounds bugs. Prefer `checked_sub`, `saturating_sub`, `split_last()`, `last()`, or restructure with iterators.

If you want overflow checks in release builds, set `overflow-checks = true` in the profile (Lesson 03). Do not rely on the default for correctness: write arithmetic whose meaning is the same in both.

---

## 6. Conversions: `as`, `From`, `TryFrom`

```rust
fn main() {
    println!("{}", 300i32 as u8);
    println!("{}", -1i32 as u32);
    println!("{}", 3.99f64 as i32);
    println!("{}", -1.5f64 as u8);
    println!("{}", f64::NAN as i32);
    println!("{}", 1e20f64 as i32);
    println!("{}", 200u8 as i8);
    println!("{}", 'A' as u32);
    println!("{}", 97u8 as char);
    println!("{}", true as i32);
    println!("{}", u64::MAX as f32);
    println!("{:?}", u8::try_from(300i32));
    println!("{:?}", u8::try_from(200i32));
    println!("{}", i64::from(7i32));
    println!("{}", f64::from(7i32));
    println!("{:?}", char::try_from(65u32));
    println!("{:?}", char::from_u32(0xD800));
}
```

✓ output, one line per statement:

```text
44                      300i32 as u8     (300 mod 256)
4294967295              -1i32 as u32     (same bits)
3                       3.99 as i32      (truncates toward zero)
0                       -1.5 as u8       (saturates to 0)
0                       NAN as i32       (NaN becomes 0)
2147483647              1e20 as i32      (saturates to i32::MAX)
-56                     200u8 as i8      (reinterprets the bits)
65                      'A' as u32
a                       97u8 as char
1                       true as i32
18446744000000000000    u64::MAX as f32  (rounded to the nearest f32)
Err(TryFromIntError(PosOverflow))
Ok(200)
7
7
Ok('A')
None                    0xD800 is a surrogate, not a valid char
```

(The annotations on the right are my explanation, not program output.)

| Mechanism | Fails how | Use for |
|---|---|---|
| `From`/`Into` | cannot fail; only implemented when lossless (`i64::from(i32)`, `u32::from(u8)`) | widening |
| `TryFrom`/`TryInto` | returns `Err` | narrowing, signed ↔ unsigned, untrusted input |
| `as` | never fails and **never tells you** | bit reinterpretation, deliberate truncation, float-to-int with saturation |

`as` between integers silently truncates or reinterprets; since Rust 1.45, float-to-integer `as` saturates and maps NaN to 0 (before that it was undefined behavior in some cases). Those rules are well-defined, but "well-defined" is not the same as "what you meant".

**Rule of thumb.** In production code, reach for `From` when the compiler allows it (it documents "this is lossless"), `TryFrom` when it does not, and reserve `as` for places where you can state in a comment why truncation or reinterpretation is intended. Clippy's `cast_possible_truncation`, `cast_sign_loss`, and `cast_possible_wrap` (pedantic group) flag the risky ones; enabling them in security-sensitive crates is reasonable.

---

## 7. Type aliases do not create new types

```rust
type UserId = u64;
type OrderId = u64;
fn ship(user: UserId, order: OrderId) { println!("user {user} order {order}"); }
fn main() {
    let u: UserId = 1;
    let o: OrderId = 99;
    ship(o, u);   // arguments swapped
}
```

✓ This **compiles and runs** (`user 99 order 1`). An alias is only another name for the same type. To make the compiler reject mixing them you need distinct types: *newtypes* (`struct UserId(u64);`, Lesson 15). Aliases are good for shortening long types (`type Result<T> = std::result::Result<T, MyError>;`), not for safety.

---

## 8. `let _` versus `let _name`

```rust
struct Guard(&'static str);
impl Drop for Guard {
    fn drop(&mut self) { println!("drop {}", self.0); }
}
fn main() {
    let _ = Guard("ignored");
    println!("after let _");
    let _kept = Guard("kept");
    println!("after let _kept");
}
```

✓ output:

```text
drop ignored
after let _
after let _kept
drop kept
```

`let _ = expr;` does not bind anything: the temporary is dropped at the end of that statement. `let _kept = expr;` binds a variable that lives to the end of the scope, and the leading underscore only silences the unused-variable warning. This difference is a classic bug with lock guards: `let _ = mutex.lock();` releases the lock immediately, so the "protected" code runs unprotected (Lessons 24 and 29). Drop order and RAII are covered in Lesson 08.

---

## 9. Common mistakes and edge cases

1. Assuming release and debug arithmetic agree. They do not by default (Section 5.1).
2. Casting with `as` to "make the types match" without thinking about the value range.
3. Comparing floats with `==`, or sorting them with a comparator that panics on NaN (`partial_cmp(..).unwrap()`).
4. Using `f32` for large counters: above 2²⁴ it cannot represent every integer.
5. Taking `s.len()` as a character count.
6. Using `usize` for values that are not sizes or indexes (IDs, timestamps, amounts). It changes width between platforms: on a 32-bit target it is 4 bytes. Persisted or wire formats should use fixed-width types (`u32`, `u64`).
7. `let _ = guard;` instead of `let _guard = guard;`.
8. Declaring a variable without an initial value and assigning in only some branches (E0381).
9. Believing shadowing mutates. The earlier binding is unchanged.
10. Reaching for `static mut`; use atomics, `Mutex`, or `OnceLock`.

## 10. Performance implications

- Integer arithmetic is as fast as C's for the same operation. Overflow checks in debug builds cost time but are removed by default in release; `overflow-checks = true` in release adds a branch on each operation, which is usually cheap but not free; measure before deciding (Lesson 41).
- `checked_*` compiles to the hardware add plus an overflow-flag test, then a branch. For hot loops where overflow is impossible by construction, arithmetic on a widened type (`u64::from(a) * u64::from(b)`) avoids the branch.
- Prefer the smallest type that is *correct*, not merely sufficient for today: narrower types shrink arrays and structs (cache effects, Lessons 09 and 42), but conversions cost readability.
- `const` values are inlined; `const fn` calls in constant context cost nothing at runtime.
- Float vs integer: no general rule; benchmark. Do not change a type for speed without data.

## 11. Security and reliability

- Memory safety does not prevent **logic overflow**. A wrapped balance, a truncated length, or a negative-turned-huge index are correctness and security bugs even though no memory is corrupted (Lesson 01).
- Untrusted numeric input (network lengths, file sizes, amounts): parse into wide types, validate ranges, convert with `TryFrom`, do arithmetic with `checked_*`. Never allocate based on an unchecked length.
- Blockchain code (Lessons 48–55) is unusually sensitive: token amounts are integers in base units, and unchecked or wrapping arithmetic on balances has caused real losses on chains. Do not infer the behavior of any specific program from this lesson; consult the framework's documented arithmetic and build settings.
- Floats are non-deterministic across some platforms and optimizations in subtle ways (for example fused multiply-add availability). Consensus-critical code should avoid floating point entirely (Lesson 50).
- Do not use `unwrap()` on conversions from external data; return an error.

---

## 12. Exercises

Try each before opening the answer. They are not marked complete unless you tell me you did them.

1. **(Easy)** Without running it, what does this print, and does it compile?
   ```rust
   fn main() {
       let x = 5;
       let x = x * 2;
       { let x = "ten"; println!("{x}"); }
       println!("{x}");
   }
   ```
2. **(Easy)** Define `SECS_PER_DAY: u32` and then `SECS_PER_WEEK: u32` as constants computed from it. Why is `static mut` the wrong tool for this?
3. **(Medium)** What are `(-9) / 4`, `(-9) % 4`, `(-9i32).rem_euclid(4)`, and `(-9i32).div_euclid(4)`? Verify by running.
4. **(Medium)** Rewrite `let n = value as u8;` (where `value: i32`) so that out-of-range input is an error rather than a silent truncation.
5. **(Medium)** Sort a `Vec<f32>` that may contain NaN so that NaN values go last, without `unwrap`.
6. **(Hard)** Implement `fn average(a: u32, b: u32) -> u32` that is correct for all inputs (including `u32::MAX`) with no overflow and no widening to `u64`.
7. **(Hard)** Write `fn parse_len(bytes: &[u8]) -> Option<usize>` that interprets the first 4 bytes as a big-endian `u32` length, and returns `None` if there are fewer than 4 bytes or the length exceeds a `MAX: usize = 1 << 20`. Use `TryFrom`, no `as`.

<details markdown="1">
<summary>Answers</summary>

1. ✓ It compiles. Output: `ten` then `10`. The inner `x` shadows only within its block.
2. `const SECS_PER_DAY: u32 = 24 * 60 * 60; const SECS_PER_WEEK: u32 = SECS_PER_DAY * 7;` ✓ gives `604800`. Constants are computed at compile time, are never mutable, and need no `unsafe`; `static mut` is `unsafe` to touch, has an address, and would suggest a value that can change.
3. ✓ `-2`, `-1`, `3`, `-3`. (Truncating division gives −2 remainder −1; Euclidean gives remainder 3 and quotient −3, since −3×4 + 3 = −9.)
4. `let n = u8::try_from(value)?;` (in a function returning `Result` with a compatible error), or `u8::try_from(value).map_err(...)`.
5. ✓ `v.sort_by(|a, b| a.total_cmp(b));` (`total_cmp` is a total order; positive NaN sorts after all numbers. Negative-signed NaNs sort first, so normalize or filter if the sign of NaN can vary.)
6. ✓ `(a & b) + ((a ^ b) >> 1)` (tested: `avg(u32::MAX, u32::MAX) == u32::MAX`, `avg(3, 4) == 3`) computes the floor average without overflow. Alternative: `a / 2 + b / 2 + (a % 2 + b % 2) / 2`. Note: recent Rust versions also provide `midpoint` methods on integers; check the standard-library documentation for your minimum supported version before relying on it.
7. ```rust
   const MAX: usize = 1 << 20;
   fn parse_len(bytes: &[u8]) -> Option<usize> {
       let head: [u8; 4] = bytes.get(..4)?.try_into().ok()?;
       let n = usize::try_from(u32::from_be_bytes(head)).ok()?;
       (n <= MAX).then_some(n)
   }
   ```
   ✓ Tested: `[0,0,1,0]` → `Some(256)`, two bytes → `None`, length `0x00200000` → `None`, trailing bytes ignored. `usize::try_from(u32)` is needed because `usize` may be 16 bits on some targets, so no infallible `From<u32>` exists.

</details>

---

## 13. Debugging challenge

This function is meant to return a percentage from 0 to 100:

```rust
fn percent(part: u32, total: u32) -> u8 {
    (part * 100 / total) as u8
}
```

Find **three** distinct bugs, show an input that triggers each, and fix it so that nothing can panic or silently produce a wrong answer.

<details markdown="1">
<summary>Solution</summary>

1. **Overflow in `part * 100`.** `percent(50_000_000, 100_000_000)` is a legitimate 50%, but `50_000_000 * 100 = 5×10⁹` exceeds `u32::MAX`. ✓ Debug build: panics with `attempt to multiply with overflow`. ✓ Release build: wraps and silently returns `7` instead of `50`.
2. **Division by zero.** `percent(3, 0)` ✓ panics with `attempt to divide by zero` in every build mode.
3. **Silent truncation by `as u8`.** `percent(3, 1)` computes `300`; `300 as u8` ✓ returns `44` with no warning. Any `part > total` yields a nonsense answer or, past 255%, a wrapped one.

Fix:

```rust
fn percent(part: u32, total: u32) -> Option<u8> {
    if total == 0 || part > total {
        return None;
    }
    let p = u64::from(part) * 100 / u64::from(total);
    u8::try_from(p).ok()
}
```

✓ `percent(1, 2)`, `percent(50_000_000, 100_000_000)` → `Some(50)`; `percent(5, 2)` and `percent(3, 0)` → `None`; `percent(u32::MAX, u32::MAX)` → `Some(100)`.

Design notes: widening to `u64` is lossless (`From`), so the multiplication cannot overflow (`u32::MAX * 100 < u64::MAX`). The `part > total` check makes the result ≤ 100, so the final `try_from` cannot fail in practice, but keeping it avoids an `as` and a comment explaining why truncation is safe. Whether `part > total` should be `None`, a clamp to 100, or an error type is a *contract decision*; encode it in the signature and document it (Lesson 14 turns `Option` into a proper `Result`).

</details>

---

## 14. Practical challenge: a small checked-numbers library

Create a library crate `numlab` (edition 2024) with:

1. `fib_checked(n: u32) -> Option<u64>`: the `n`th Fibonacci number (F(0)=0, F(1)=1), `None` if it does not fit in `u64`.
2. `percent` from the debugging challenge.
3. `to_index(i: i64) -> Option<usize>`, rejecting negatives, with no `as`.
4. Unit tests that include the boundary cases.
5. Pass `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, and `cargo test --locked`. (Remember `cargo generate-lockfile` once, so `--locked` has a lockfile to check, as in Lesson 03.)

Bonus: find the largest `n` for which `fib_checked` works with `u128` instead of `u64`, by experiment rather than by memory.

<details markdown="1">
<summary>Solution</summary>

**`src/lib.rs`**

```rust
//! Checked integer helpers.

/// Returns the `n`th Fibonacci number (F(0) = 0, F(1) = 1), or `None` if it
/// does not fit in a `u64`.
pub fn fib_checked(n: u32) -> Option<u64> {
    if n == 0 {
        return Some(0);
    }
    let (mut prev, mut cur) = (0u64, 1u64);
    for _ in 1..n {
        let next = prev.checked_add(cur)?;
        prev = cur;
        cur = next;
    }
    Some(cur)
}

/// Whole-number percentage of `part` in `total`, rounded down.
/// `None` when `total` is zero or `part` exceeds `total`.
pub fn percent(part: u32, total: u32) -> Option<u8> {
    if total == 0 || part > total {
        return None;
    }
    let p = u64::from(part) * 100 / u64::from(total);
    u8::try_from(p).ok()
}

/// Converts a signed value to an index, rejecting negatives.
pub fn to_index(i: i64) -> Option<usize> {
    usize::try_from(i).ok()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn fib_small_values() {
        let got: Vec<_> = (0..10).map(|n| fib_checked(n).unwrap()).collect();
        assert_eq!(got, [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]);
    }

    #[test]
    fn fib_boundary() {
        assert_eq!(fib_checked(93), Some(12_200_160_415_121_876_738));
        assert_eq!(fib_checked(94), None);
    }

    #[test]
    fn percent_edges() {
        assert_eq!(percent(0, 5), Some(0));
        assert_eq!(percent(1, 3), Some(33));
        assert_eq!(percent(5, 5), Some(100));
        assert_eq!(percent(u32::MAX, u32::MAX), Some(100));
        assert_eq!(percent(1, 0), None);
        assert_eq!(percent(6, 5), None);
    }

    #[test]
    fn index_rejects_negative() {
        assert_eq!(to_index(-1), None);
        assert_eq!(to_index(7), Some(7));
    }
}
```

✓ With this exact file: `cargo fmt --check` clean, `cargo clippy --all-targets --locked -- -D warnings` clean, `cargo test --locked` → 4 passed.

**Why the loop is shaped this way.** The pair `(prev, cur)` holds `(F(k-1), F(k))`. Starting at `k = 1` and running `n - 1` iterations ends at `k = n`. The `?` on `checked_add` returns `None` as soon as the *next* value would overflow. A common off-by-one is to run the loop `n` times, which computes F(n+1) and reports overflow one index too early; the boundary test (`fib_checked(93)` is `Some`, 94 is `None`) catches it. F(93) = 12200160415121876738 is the largest Fibonacci number that fits in `u64`.

**Bonus.** ✓ I ran the same recurrence with `u128` and `checked_add`: the last index that fits is **186** (F(186) = 332825110087067562321196029789634457848).

Reflection: tests at the exact boundary (93/94) are worth more than many mid-range tests. Property-based tests that compare against a wider type (`u128`) are the next step (Lesson 40).

</details>

---

## 15. Interview questions

<details markdown="1">
<summary>Answers</summary>

1. **Why are Rust variables immutable by default?** Most values are written once; default immutability makes code easier to read and reason about (including for the compiler) and makes mutation an explicit, reviewable choice.
2. **Shadowing vs `mut`?** Shadowing creates a new variable (can change type, can leave the result immutable); `mut` allows reassignment of the same variable of the same type.
3. **`const` vs `static`?** `const` is inlined, has no address, and is evaluated at compile time; `static` is one memory location with `'static` lifetime. Mutable statics require `unsafe` (or interior mutability).
4. **What happens on integer overflow?** Debug builds panic by default; release builds wrap by default (two's complement). Both are defined, not UB. Overflow behavior is controlled by `overflow-checks` and can be made explicit with `checked_`, `wrapping_`, `saturating_`, `overflowing_` methods.
5. **Why is `f64` not `Ord`?** NaN is not comparable (not equal even to itself), so ordering is partial; use `partial_cmp` or `total_cmp`.
6. **What does `as` do for `-1i32 as u32`, `300i32 as u8`, `3.9f64 as i32`?** Reinterprets bits (`4294967295`), truncates (`44`), truncates toward zero (`3`). Float-to-int `as` saturates and maps NaN to 0.
7. **When should you use `TryFrom` instead of `as`?** Whenever a conversion might lose information and the input is not fully under your control.
8. **Is a type alias a new type?** No; it is another name for the same type, so it offers no protection against mixing values. Use a newtype.
9. **What is the difference between `let _ = f();` and `let _x = f();`?** The first drops the value immediately; the second keeps it until end of scope. It matters for `Drop` types such as lock guards.
10. **Can Rust read an uninitialized local variable?** Not in safe code: definite-assignment analysis rejects it (E0381). Only `unsafe` code with `MaybeUninit` can handle uninitialized memory, with obligations on the programmer.
11. **Why does `v.len() - 1` worry you?** `len()` is `usize`; on an empty collection the subtraction overflows: panic in debug, wraps to a huge number in release.
12. **Is memory-safe overflow therefore harmless?** No. No memory is corrupted, but the logical result is wrong; for lengths, balances, or indexes that can be a vulnerability.

</details>

---

## 16. Key takeaways

- `let` is immutable by default; `mut` opts in. Definite-initialization analysis means safe code cannot read uninitialized locals.
- Shadowing makes a new variable (type may change); `mut` mutates one variable.
- `const` is a compile-time value (overflow is a hard error); `static` is one address; avoid `static mut`, which edition 2024 makes hard to misuse.
- Integers are fixed width; the default literal type is `i32`. Overflow is defined but differs between debug and release, so write code whose meaning does not depend on the profile.
- Floats are IEEE 754: inexact, NaN, no `Ord`, no panic on division by zero. Do not use them for money.
- `as` is silent and lossy; prefer `From` (lossless) and `TryFrom` (checked). A type alias is not a new type. `let _` drops now; `let _x` drops at scope end.

**Connections.** Lesson 01's overflow semantics and `checked_*` family are now applied to real code; Lesson 03's profiles explain why debug and release differ. Next: Lesson 05 (functions, expressions versus statements, control flow, tuples, arrays, slices). Conversions return in Lesson 06 (`From`/`TryFrom` traits in depth), moves and `Drop` in Lessons 07–08, memory layout and niches in Lesson 09, `Option`/`Result` in Lesson 14, newtypes in Lesson 15, atomics in Lesson 28, and numeric safety in blockchain code in Lessons 48–55. See the [roadmap](00_roadmap.md) and the [course index](index.md).

---

## 17. References

- *The Rust Programming Language*, ch. 3.1 "Variables and Mutability": <https://doc.rust-lang.org/book/ch03-01-variables-and-mutability.html>
- *The Rust Programming Language*, ch. 3.2 "Data Types": <https://doc.rust-lang.org/book/ch03-02-data-types.html>
- *The Rust Reference*, numeric types: <https://doc.rust-lang.org/reference/types/numeric.html>
- *The Rust Reference*, operator expressions (arithmetic overflow, type cast expressions): <https://doc.rust-lang.org/reference/expressions/operator-expr.html>
- *The Rust Reference*, constant evaluation: <https://doc.rust-lang.org/reference/const_eval.html>
- *The Rust Reference*, items: constants and statics: <https://doc.rust-lang.org/reference/items/static-items.html>
- *Rust Edition Guide*, Rust 2024: disallow references to `static mut`: <https://doc.rust-lang.org/edition-guide/rust-2024/static-mut-references.html>
- Rust standard library documentation, primitive `f64` (`total_cmp`, NaN semantics) and the integer types (`checked_*`, `rem_euclid`): <https://doc.rust-lang.org/std/primitive.f64.html>, <https://doc.rust-lang.org/std/primitive.i32.html>
- Rust standard library documentation, `TryFrom` and `From`: <https://doc.rust-lang.org/std/convert/trait.TryFrom.html>
- IEEE Std 754-2019, *Standard for Floating-Point Arithmetic* (purchase required; the std docs above summarize the behavior used here).

The URLs above are the standard documentation locations; I did not open every one during this session, so if a link has moved, search the docs site for the page title.
