---
layout: default
title: "Lesson 05 — Functions, expressions, control flow, tuples, arrays, and slices"
---

# Lesson 05 — Functions, expressions, control flow, tuples, arrays, and slices

**Module 2 · Syntax and fundamentals** · Status: generated · Edition used in examples: **2024** · External dependencies: none

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux. As before, ✓ marks outputs and diagnostics I actually ran; unmarked statements are documented behavior I did not run. Sizes are for a **64-bit** target.

Builds on [Lesson 04](04_variables_and_types.md) (integers, `usize`, overflow, `as`) and [Lesson 01](01_foundations.md) (bounds checks, UB).

---

## 1. Functions

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

- Parameter types and the return type are **always written**. Inside a body the compiler infers types; at function boundaries it does not. That makes every function signature a complete, local contract, which keeps inference errors from spreading across the crate and keeps public APIs stable and readable.
- No `-> T` means the function returns `()` (unit).
- The last expression of the body, **without a semicolon**, is the return value. `return` exists for early exit.
- Rust has no function overloading and no default or named arguments. Alternatives are different function names, builder patterns (Lesson 13), and traits (Lesson 19).
- Functions can be declared anywhere in a module; order does not matter.

### 1.1 Parameters are patterns, arguments are moved or copied

A call `f(x)` passes `x` by value: `Copy` types are copied, other types are *moved* (Lesson 07). To let a function look at a value without taking it, pass a reference (`&T`, `&mut T`, Lesson 10). For sequences, prefer the slice parameter `&[T]` over `&Vec<T>` (Section 7).

### 1.2 Wrong argument types (E0308)

```rust
fn greet(name: &str, times: u32) -> String { name.repeat(times as usize) }
fn main() { println!("{}", greet("a", "3")); }
```

✓ `error[E0308]: mismatched types`, pointing at the argument and at the parameter in the signature. There is no implicit conversion between `&str` and an integer, nor between integer widths (Lesson 04).

### 1.3 Function pointers and the never type

```rust
fn double(x: i32) -> i32 { x * 2 }
fn apply(f: fn(i32) -> i32, x: i32) -> i32 { f(x) }
fn diverge() -> ! { panic!("never returns") }

fn main() {
    println!("{}", apply(double, 4));
    let v: u32 = if std::env::args().count() > 5 { diverge() } else { 7 };
    println!("{v}");
    let mut buf = [0u8; 4];
    fill(&mut buf);
    println!("{buf:?}");
}
fn fill(out: &mut [u8]) {
    for (i, b) in out.iter_mut().enumerate() { *b = i as u8 * 2; }
}
```

✓ prints `8`, `7`, `[0, 2, 4, 6]`.

- `fn(i32) -> i32` is a **function pointer** type: a plain address, no captured state. Closures that capture variables are a different mechanism (Lesson 18).
- `-> !` means the function never returns normally. Since `!` coerces to any type, `diverge()` type-checks in a branch that must produce a `u32`.
- `fill` takes `&mut [u8]` and so works on arrays, vectors, or any sub-slice. (`i as u8 * 2` is an `as` cast, Lesson 04: fine here because `i < 4`.)

### 1.4 Recursion and the stack

Each call uses stack space (Lesson 08 covers frames). The Rust language does **not** guarantee tail-call optimization.

```rust
fn depth(n: u64) -> u64 { if n == 0 { 0 } else { 1 + depth(n - 1) } }
fn main() { println!("{}", depth(std::hint::black_box(10_000_000))); }
```

✓ In a **debug** build: `thread 'main' has overflowed its stack` / `fatal runtime error: stack overflow, aborting` (the process aborts; it is not a catchable panic). ✓ With `-O` the same program printed `10000000`, because the optimizer happened to turn the recursion into a loop. That is an optimization you may observe but must not rely on: for untrusted or unbounded depth (parsers, tree walks over attacker data), use an explicit loop or an explicit stack, or enforce a depth limit.

---

## 2. Expressions versus statements

Rust is **expression-oriented**. Almost everything (blocks, `if`, `match`, `loop`) has a value.

- A **statement** performs an action and yields `()`: a `let` binding, an item declaration, or an expression followed by `;`.
- An **expression** evaluates to a value. A **block** `{ ... }` is an expression whose value is its last expression, if it has no trailing semicolon.

```rust
fn main() {
    let y = {
        let x = 3;
        x + 1            // no semicolon: the block's value
    };
    let z = if y > 3 { "big" } else { "small" };
    println!("{} {y} {z}", add(2, 3));
    let u = { 5; };      // semicolon: the block's value is ()
    println!("{u:?}");
}
fn add(a: i32, b: i32) -> i32 { a + b }
```

✓ prints `5 4 big` then `()`.

### 2.1 The classic mistake: a stray semicolon (E0308)

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b;
}
```

✓ `error[E0308]: mismatched types ... expected `i32`, found `()``, with the helpful note "implicitly returns `()` as its body has no tail or `return` expression" and "help: remove this semicolon to return this value". The `;` turns the value into a discarded statement; the body then has type `()`.

### 2.2 `if` is an expression, so branches must agree

```rust
fn main() {
    let n = 3;
    let s = if n > 2 { "many" } else { 0 };
    println!("{s}");
}
```

✓ `error[E0308]: `if` and `else` have incompatible types` ("expected `&str`, found integer"). And the condition must be a `bool` — there is no truthiness:

```rust
fn main() { let n = 1; if n { println!("yes"); } }
```

✓ `error[E0308]: mismatched types ... expected `bool`, found integer`. If an `if` has no `else`, the `if` block must have type `()`.

**Design rationale.** Making blocks and conditionals expressions removes a class of temporaries (`let mut s; if c { s = ...; } else { s = ...; }`) and lets values be initialized once and left immutable (Lesson 04). Disallowing integer truthiness eliminates `if (x = 0)`-style accidents and C's implicit "non-zero means true" ambiguity.

---

## 3. Control flow

### 3.1 `if`, `loop`, `while`, `for`

```rust
fn main() {
    let mut n = 0;
    let found = loop {                 // loop is an expression: break carries a value
        n += 7;
        if n % 5 == 0 { break n; }
    };
    println!("{found}");

    let mut c = 3;
    while c > 0 { print!("{c} "); c -= 1; }
    println!();

    for i in 0..3 { print!("{i} "); }
    for i in (0..=3).rev() { print!("{i} "); }
    for i in (0..10).step_by(4) { print!("{i} "); }
    println!();

    'outer: for i in 0..5 {
        for j in 0..5 {
            if i * j == 6 { println!("found {i} {j}"); break 'outer; }
        }
    }

    let label_val = 'a: { if found > 10 { break 'a "gt10"; } "le10" };
    println!("{label_val}");
}
```

✓ output:

```text
35
3 2 1 
0 1 2 3 2 1 0 0 4 8 
found 2 3
gt10
```

- `loop` runs until `break`; only `loop` can `break` with a value, because the compiler knows it exits only via `break`. `while` and `for` may run zero times, so they always evaluate to `()`.
- `for` loops work on anything implementing `IntoIterator` (Lesson 17). Ranges: `a..b` excludes `b`, `a..=b` includes it. `rev()` and `step_by(k)` adapt a range (`step_by(0)` panics).
- **Labels** (`'outer:`) let `break` and `continue` target an outer loop. **Labeled blocks** (`'a: { ... break 'a value; ... }`) give early exit from a block, a tidy alternative to nested `if`s.
- `for` is almost always preferable to a manual index loop with `while`: it cannot go out of bounds, and the iterator form lets the optimizer remove bounds checks (Section 6, Lesson 17). Clippy's `needless_range_loop` (Lesson 02) points this out.

### 3.2 Ranges are values, and can be empty

```rust
fn main() {
    println!("{} {}", (5..3).count(), (3..=3).count());
    println!("{} {}", (0u8..=255).count(), (0..0).is_empty());
    let r = 2..5;
    println!("{:?} {} {}", r, r.start, r.contains(&4));
}
```

✓ prints `0 1`, `256 true`, `2..5 2 true`. `5..3` is simply empty (it does not count downward). Note `0u8..=255` works: the inclusive range exists precisely so the top value of a type can be included (`0u8..256` would not compile, since 256 is not a `u8`).

### 3.3 Early return and `?`

```rust
fn classify(n: i32) -> &'static str {
    if n < 0 { return "neg"; }
    if n == 0 { return "zero"; }
    "pos"
}
```

✓ `classify(-1)`, `classify(0)`, `classify(9)` → `neg zero pos`. Early returns ("guard clauses") keep the main path flat. The `?` operator is the same idea for errors (Lesson 14). `match` and `if let` are Lesson 06 (patterns) and Lesson 13 (enums).

---

## 4. Tuples

A tuple groups a fixed number of values of possibly different types. The type is written `(i32, &str, f64)`.

```rust
fn min_max(v: &[i32]) -> Option<(i32, i32)> {
    let first = *v.first()?;
    Some(v.iter().fold((first, first), |(lo, hi), &x| (lo.min(x), hi.max(x))))
}

fn main() {
    let t = (1, "two", 3.0);
    let (a, b, c) = t;             // destructuring
    println!("{a} {b} {c} {}", t.1);   // field access by position
    println!("{:?}", min_max(&[3, 9, -2]));
    println!("{:?}", min_max(&[]));
    let unit = ();
    println!("{unit:?} {}", std::mem::size_of::<(u8, u32)>());
    println!("{:?}", (1, 2) < (1, 3));
    let one = (5,);                // one-element tuple needs the comma
    println!("{one:?}");
}
```

✓ output:

```text
1 two 3 two
Some((-2, 9))
None
() 8
true
(5,)
```

- Tuples return several values from a function without a named type. Beyond two or three fields, define a `struct` with named fields (Lesson 13): `(f64, f64, f64)` says nothing about which is which.
- `()` is the zero-length tuple (unit). `(5,)` is a 1-tuple; `(5)` is just `5`.
- Tuples of comparable types are compared lexicographically (`(1, 2) < (1, 3)`), which is handy for sorting by multiple keys.
- Layout: a tuple's field order in memory is **unspecified** (the compiler may reorder to minimize padding; Lesson 09). Don't assume `(u8, u32)` is laid out in source order. ✓ Observed: `size_of::<(u8, u32)>() == 8`, `size_of::<(u8, u32, u8)>() == 8` and `size_of::<(u32, u8)>() == 8`: reordering the two `u8`s together fits three fields in 8 bytes. Treat this as an observation, not a guarantee.

---

## 5. Arrays

An array `[T; N]` holds exactly `N` values of type `T`, stored inline and contiguously, with `N` part of the type.

```rust
fn main() {
    let a = [1, 2, 3];
    let z = [0u8; 4];                 // repeat expression: 4 zeros
    let b: [i32; 3] = [10, 20, 30];
    println!("{a:?} {z:?} {} {}", b.len(), std::mem::size_of_val(&b));
    let i = std::env::args().count() + 4;
    println!("{:?} {:?}", b.get(i), b.get(1));
    let c = a;                        // arrays of Copy types are Copy
    println!("{a:?} {c:?}");
    for x in a { print!("{x} "); }    // by value (since edition 2021)
    println!();
    let grid = [[0; 3]; 2];           // 2 rows of 3
    println!("{grid:?}");
    println!("{}", b[i]);             // runtime index out of range
}
```

✓ output:

```text
[1, 2, 3] [0, 0, 0, 0] 3 12
None Some(20)
[1, 2, 3] [1, 2, 3]
1 2 3 
[[0, 0, 0], [0, 0, 0]]

thread 'main' panicked at a08_arrays.rs:14:20:
index out of bounds: the len is 3 but the index is 5
```

Key facts:

- **Length is part of the type.** `[i32; 3]` and `[i32; 4]` are different types. A wrong-length initializer is a compile-time error:

  ```rust
  fn main() { let a: [i32; 3] = [1, 2]; println!("{a:?}"); }
  ```

  ✓ `error[E0308]: mismatched types` (expected an array with a size of 3, found one with a size of 2).
- Arrays live on the stack by default; `[0u8; 1_000_000]` is a 1 MB stack value, which is a risk on small stacks. For large or runtime-sized data use `Vec<T>` (Lesson 16).
- **Indexing is bounds-checked.** `b[i]` with `i >= len` panics (✓ above): the program stops safely instead of reading adjacent memory, which is the Lesson 01 guarantee. `get(i)` returns `Option<&T>` instead (✓ `None`).
- When the index and length are both known at compile time, the compiler rejects the program with a deny-by-default lint:

  ```rust
  fn main() { let a = [1, 2, 3]; println!("{}", a[5]); }
  ```

  ✓ `error: this operation will panic at runtime ... index out of bounds: the length is 3 but the index is 5` (lint `unconditional_panic`).
- Iterating an array by value (`for x in a`) is available from edition 2021; in the 2015 and 2018 editions `array.into_iter()` had different method-resolution behavior kept for compatibility. This course uses 2024.

---

## 6. Slices

A **slice** `[T]` is a view of a contiguous run of `T` somewhere in memory. Because its length is not part of the type, `[T]` is *dynamically sized*, so you almost always meet it behind a reference: `&[T]` (shared) or `&mut [T]` (exclusive).

A `&[T]` is a **fat pointer**: an address plus a length.

```rust
fn sum(xs: &[i32]) -> i32 { xs.iter().sum() }
fn first_word(s: &str) -> &str { s.split(' ').next().unwrap_or("") }

fn main() {
    let a = [1, 2, 3, 4, 5];
    let v = vec![10, 20, 30];
    println!("{} {} {}", sum(&a), sum(&v), sum(&a[1..3]));   // array, Vec, sub-slice
    let s = &a[..];
    println!("{:?} {:?} {:?} {}", &s[..2], &s[3..], &s[1..=2], s.len());
    println!("{} {}", std::mem::size_of::<&[i32]>(), std::mem::size_of::<&[i32; 5]>());
    println!("{:?} {:?}", s.first(), s.last());
    println!("{:?} {:?}", s.split_first(), s.split_at(2));
    println!("{:?}", s.windows(2).map(|w| w[1] - w[0]).collect::<Vec<_>>());
    println!("{:?}", s.chunks(2).collect::<Vec<_>>());
    println!("{}", first_word("hello world"));
    let mut m = [5, 3, 1];
    m.sort();
    m[..2].reverse();
    println!("{m:?}");
    let (l, r) = m.split_at_mut(1);
    l[0] += r[0];
    println!("{m:?}");
    let n = std::env::args().count() + 9;
    println!("{:?}", s.get(1..n));
    println!("{:?}", &s[1..n]);
}
```

✓ output:

```text
15 60 5
[1, 2] [4, 5] [2, 3] 5
16 8
Some(1) Some(5)
Some((1, [2, 3, 4, 5])) ([1, 2], [3, 4, 5])
[1, 1, 1, 1]
[[1, 2], [3, 4], [5]]
hello
[3, 1, 5]
[4, 1, 5]
None

thread 'main' panicked at a11_slices.rs:26:24:
range end index 10 out of range for slice of length 5
```

What to notice:

- One function `sum(&[i32])` accepts an array, a `Vec` (via deref coercion, Lesson 23), and a sub-range. This is why **parameters should be `&[T]` rather than `&Vec<T>` or `&[T; N]`**: it is strictly more general and costs nothing. Clippy flags `&Vec<T>` parameters (`ptr_arg`).
- `&[i32]` is **16 bytes** (pointer + length) while `&[i32; 5]` is **8 bytes** (the length is in the type) ✓. `&str` is the same fat pointer over UTF-8 bytes (Lesson 06).
- Range indexing is the only place you see `..` in index position: `a[1..3]` is elements 1 and 2. A bad range **panics** with a message (✓ "range end index 10 out of range for slice of length 5"); `get(range)` returns `None` instead. Reversed ranges panic too: ✓ `&a[3..2]` gives "slice index starts at 3 but ends at 2".
- The slice API is rich: `first`, `last`, `split_first`, `split_at`, `windows(n)` (overlapping), `chunks(n)` (non-overlapping, last may be short), `sort`, `reverse`, `iter`, `binary_search`, `contains`, and more. Reach for these instead of index arithmetic.
- `split_at_mut` yields two non-overlapping `&mut` halves at once. The borrow checker cannot prove that `&mut v[0]` and `&mut v[1]` are disjoint, but the standard library can, with a small piece of `unsafe` inside a safe API (Lesson 36 shows the pattern).
- **Slices borrow.** While a slice exists, the owner cannot be changed:

  ```rust
  fn main() {
      let mut v = vec![1, 2, 3];
      let first = &v[0];
      v.push(4);
      println!("{first}");
  }
  ```

  ✓ `error[E0502]: cannot borrow `v` as mutable because it is also borrowed as immutable`. The reason is physical: `push` may reallocate the buffer, which would leave `first` pointing at freed memory. Rust forbids that at compile time (Lesson 10).

**Performance.** Bounds checks cost a compare and a branch per indexed access, and often are removed by the optimizer when it can prove the index is in range (for example in `for x in &slice` loops or after an up-front length assertion). Idiomatic fixes for hot loops are iterator adapters, `chunks`/`windows`, or one `assert!(i < s.len())` before several accesses; **measure** before restructuring code for this (Lesson 41). Never reach for `get_unchecked` to "remove the check" without profiling and a safety argument: an out-of-bounds unchecked access is undefined behavior (Lesson 36).

---

## 7. Common mistakes and edge cases

1. A stray semicolon on the last expression (E0308).
2. `if`/`else` branches with different types; non-`bool` conditions.
3. `data.len() - w + 1` and similar: `usize` subtraction that underflows (Lesson 04). Prefer `windows`, `checked_sub`, or restructure.
4. Assuming a range counts down (`5..3` is empty) or that `..=` and `..` differ only by style (they differ on the endpoint and in overflow-safe loops over the max value).
5. `windows(0)` and `chunks(0)` panic ✓ (`window size must be non-zero`); `step_by(0)` panics. Validate sizes that come from outside.
6. Passing `&Vec<T>` or `&[T; N]` where `&[T]` would do.
7. Holding a slice of a `Vec` across a mutation of that `Vec` (E0502).
8. Using unbounded recursion on attacker-controlled depth: the stack overflow aborts the process.
9. Large arrays on the stack; arrays by value moved around (a 1 MB array copied on every call).
10. Treating tuple layout as ordered.
11. Indexing with an untrusted index without `get`.

## 8. Performance implications

- Function calls are cheap and small functions are usually inlined by the optimizer; do not hand-inline for speed. Marking functions `#[inline]` is a hint for cross-crate use, not a command (Lesson 42).
- Passing `&[T]` costs two machine words; passing `[T; N]` by value copies `N` elements (it may be optimized, but design as if it copies).
- Contiguous slices are cache-friendly, which is why `Vec<T>`/`[T]` iteration is typically fast (Lesson 42); linked structures are not.
- Iterator chains and indexed loops often compile to equivalent code (Lesson 01 observed one such case); prefer the clearer one, and verify only where it matters.

## 9. Security and reliability

- Bounds checks turn what would be a memory-corruption bug in C into a **panic**. A panic is safe but still a denial-of-service if an attacker can trigger it in a server; return errors for inputs that come from outside (`get`, `checked_*`, `windows`) and reserve indexing for indices you derived from the slice's own length.
- Do not size allocations or loops from untrusted lengths without a cap (Lesson 04 showed `TryFrom` for lengths).
- Stack overflow from deep recursion **aborts** the process; it cannot be caught with `catch_unwind`. Cap recursion depth for untrusted input.
- Panics in `Drop` or across FFI boundaries have special rules (Lessons 08 and 37); don't depend on `unwind` for correctness.

---

## 10. Exercises

Try each before opening the answer. They are not marked complete unless you tell me you did them.

1. **(Easy)** What does this return, and why does it not compile? Fix it in two different ways.
   ```rust
   fn square(x: i32) -> i32 { x * x; }
   ```
2. **(Easy)** Write `fn sign(n: i32) -> i32` returning `-1`, `0`, or `1` as a single `if`/`else if`/`else` expression with no `return`.
3. **(Medium)** Use a `loop` with `break value` to find the first power of two greater than 1000.
4. **(Medium)** Write `fn sum_pairs(v: &[i32]) -> Vec<i32>` that sums adjacent non-overlapping pairs (`[1,2,3,4,5]` → `[3, 7, 5]`) without indexing.
5. **(Medium)** Write `fn middle(v: &[i32]) -> Option<&[i32]>` returning the slice without its first and last element, `None` if there are fewer than 2 elements.
6. **(Hard)** Write `fn find_pair(sorted: &[i32], target: i32) -> Option<(usize, usize)>` using two indices that move inward (two-pointer technique) and returns indices of two different elements summing to `target`. State why it is `O(n)`.
7. **(Hard)** Write `fn rotate_left_by(v: &mut [i32], k: usize)` using `reverse` on slices (three reversals), correct for `k` larger than `v.len()` and for empty slices.

<details markdown="1">
<summary>Answers</summary>

1. ✓ (E0308 reproduced.) The body is `x * x;`, a statement, so the block has type `()` but the signature promises `i32` (E0308). Fix 1: remove the semicolon: `x * x`. Fix 2: `return x * x;` (the function then returns early; the trailing `;` after `return` is fine).
2. ✓ Tested.
   ```rust
   fn sign(n: i32) -> i32 {
       if n < 0 { -1 } else if n == 0 { 0 } else { 1 }
   }
   ```
   (`i32::signum` already exists; the exercise is about expressions.)
3. ✓ Prints `1024`.
   ```rust
   let mut p: u32 = 1;
   let first = loop {
       if p > 1000 { break p; }
       p *= 2;
   };
   // first == 1024
   ```
   `p *= 2` cannot overflow here because the loop stops at 1024; for an unbounded search use `checked_mul`.
4. ✓ `v.chunks(2).map(|c| c.iter().sum()).collect()`. The last chunk may have one element, which is why `[1,2,3,4,5]` ends in `5`.
5. ✓ Tested (`[1,2,3,4,5]` → `Some([2, 3, 4])`, `[1]` → `None`, `[1,2]` → `Some([])`).
   ```rust
   fn middle(v: &[i32]) -> Option<&[i32]> {
       if v.len() < 2 { None } else { Some(&v[1..v.len() - 1]) }
   }
   ```
   The `len() < 2` check also protects `v.len() - 1` from the Lesson 04 underflow trap. (`split_first` + `split_last` is another route.)
6. ✓ Tested (`[1,3,4,8]`, target 11 → `Some((1, 3))`; no pair → `None`; two `i32::MAX` with target −2 → `None`, no overflow).
   ```rust
   fn find_pair(sorted: &[i32], target: i32) -> Option<(usize, usize)> {
       if sorted.len() < 2 { return None; }
       let (mut i, mut j) = (0usize, sorted.len() - 1);
       while i < j {
           // widen to i64 so the sum cannot overflow
           let s = i64::from(sorted[i]) + i64::from(sorted[j]);
           match s.cmp(&i64::from(target)) {
               std::cmp::Ordering::Equal => return Some((i, j)),
               std::cmp::Ordering::Less => i += 1,
               std::cmp::Ordering::Greater => j -= 1,
           }
       }
       None
   }
   ```
   Each iteration moves `i` up or `j` down, so there are at most `n - 1` iterations: `O(n)` time, `O(1)` space. Correctness depends on the input being sorted.
7. ✓ Tested (`[1,2,3,4,5]` with `k = 7` → `[3, 4, 5, 1, 2]`, same as `rotate_left(2)`; an empty slice is fine).
   ```rust
   fn rotate_left_by(v: &mut [i32], k: usize) {
       if v.is_empty() { return; }
       let k = k % v.len();
       v[..k].reverse();
       v[k..].reverse();
       v.reverse();
   }
   ```
   The empty check prevents `k % 0` (a division by zero panic). The standard library also provides `slice::rotate_left`, which panics if `k > len`; implementing it yourself clarifies the invariants.

</details>

---

## 11. Debugging challenge

This function is meant to return the sum of every window of `w` consecutive elements:

```rust
fn window_sums(data: &[i32], w: usize) -> Vec<i32> {
    let mut out = Vec::new();
    for i in 0..data.len() - w + 1 {
        out.push(data[i..i + w].iter().sum());
    }
    out
}
```

It looks right for `window_sums(&[1, 2, 3, 4], 2)` (✓ `[3, 5, 7]`). Find **three** inputs that expose different bugs, explain the behavior in debug and release, and fix the function so it cannot panic or silently overflow.

<details markdown="1">
<summary>Solution</summary>

1. **Window larger than the data (`usize` underflow).** `window_sums(&[1, 2], 4)`: `data.len() - w` is `2 - 4`. ✓ Debug: panics with `attempt to subtract with overflow`. ✓ Release: wraps, the range becomes enormous, and the program panics later with `range end index 4 out of range for slice of length 2`. (For `w = data.len() + 1`, the release build wraps to `0` and, by accident, returns `[]`, so the bug is easy to miss.)
2. **Zero-width window.** `window_sums(&[1, 2], 0)` returns `[0, 0, 0]` in both builds ✓: a meaningless answer (3 empty windows) with no error. Decide the contract: reject it.
3. **Sum overflow.** `window_sums(&[i32::MAX, i32::MAX], 2)`: ✓ debug panics with `attempt to add with overflow`; ✓ release silently returns `[-2]`.

Fix:

```rust
fn window_sums(data: &[i32], w: usize) -> Option<Vec<i64>> {
    if w == 0 {
        return None;
    }
    Some(
        data.windows(w)
            .map(|win| win.iter().map(|&x| i64::from(x)).sum())
            .collect(),
    )
}
```

`windows(w)` yields nothing when `w > data.len()` and removes the `len() - w + 1` arithmetic entirely; widening to `i64` makes overflow impossible for any realistic input (it would take over 2³² elements of maximum magnitude to overflow an `i64`). `windows(0)` would panic ✓ (`window size must be non-zero`), so the explicit `w == 0` check, with the contract documented, is still required. The same fix is in the practical challenge below, with tests.

</details>

---

## 12. Practical challenge: `seqlab`, slice utilities with explicit contracts

Create a library crate `seqlab` (edition 2024) with:

1. `window_sums(data: &[i32], w: usize) -> Option<Vec<i64>>` (from the debugging challenge).
2. `run_lengths(data: &[u8]) -> Vec<(u8, usize)>`: run-length encoding (`[1,1,2]` → `[(1,2),(2,1)]`).
3. `search(sorted: &[i32], target: i32) -> Result<usize, usize>`: your own binary search with the same meaning as `slice::binary_search` (`Ok(index)` of a matching element, `Err(insertion_point)`), written with explicit `lo`/`hi` indices and no overflow.
4. Tests: edge cases for each, plus an **exhaustive** comparison of `search` against `binary_search` for every sorted array of length 0 to 5 over the values 0 to 3 and every target from −1 to 4.
5. It must pass `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, and `cargo test --locked` (run `cargo generate-lockfile` once first).

<details markdown="1">
<summary>Solution</summary>

**`src/lib.rs`**

```rust
//! Small slice utilities with explicit edge-case contracts.

/// Sums of every window of `w` consecutive elements, widened to `i64`.
///
/// Returns `None` if `w == 0`. If `w > data.len()` the result is empty.
pub fn window_sums(data: &[i32], w: usize) -> Option<Vec<i64>> {
    if w == 0 {
        return None;
    }
    Some(
        data.windows(w)
            .map(|win| win.iter().map(|&x| i64::from(x)).sum())
            .collect(),
    )
}

/// Run-length encoding: `[1, 1, 2]` becomes `[(1, 2), (2, 1)]`.
pub fn run_lengths(data: &[u8]) -> Vec<(u8, usize)> {
    let mut out: Vec<(u8, usize)> = Vec::new();
    for &b in data {
        match out.last_mut() {
            Some((last, n)) if *last == b => *n += 1,
            _ => out.push((b, 1)),
        }
    }
    out
}

/// Binary search in a sorted slice. `Ok(i)` means `sorted[i] == target`;
/// `Err(i)` is the index where `target` could be inserted to keep order.
pub fn search(sorted: &[i32], target: i32) -> Result<usize, usize> {
    let (mut lo, mut hi) = (0usize, sorted.len());
    while lo < hi {
        let mid = lo + (hi - lo) / 2;
        match sorted[mid].cmp(&target) {
            std::cmp::Ordering::Equal => return Ok(mid),
            std::cmp::Ordering::Less => lo = mid + 1,
            std::cmp::Ordering::Greater => hi = mid,
        }
    }
    Err(lo)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn window_sums_edges() {
        assert_eq!(window_sums(&[1, 2, 3, 4], 2), Some(vec![3, 5, 7]));
        assert_eq!(window_sums(&[1, 2], 3), Some(vec![]));
        assert_eq!(window_sums(&[], 1), Some(vec![]));
        assert_eq!(window_sums(&[1], 0), None);
        assert_eq!(
            window_sums(&[i32::MAX, i32::MAX], 2),
            Some(vec![4_294_967_294])
        );
    }

    #[test]
    fn run_lengths_cases() {
        assert_eq!(run_lengths(&[]), vec![]);
        assert_eq!(run_lengths(&[7]), vec![(7, 1)]);
        assert_eq!(run_lengths(&[1, 1, 2, 1]), vec![(1, 2), (2, 1), (1, 1)]);
    }

    #[test]
    fn search_agrees_with_std_on_small_inputs() {
        // every sorted array of length 0..=5 over values 0..=3, every target -1..=4
        for len in 0..=5usize {
            let mut data = vec![0i32; len];
            let combos = 4usize.pow(len as u32);
            for code in 0..combos {
                let mut c = code;
                for slot in data.iter_mut() {
                    *slot = (c % 4) as i32;
                    c /= 4;
                }
                if !data.windows(2).all(|w| w[0] <= w[1]) {
                    continue;
                }
                for target in -1..=4 {
                    match (search(&data, target), data.binary_search(&target)) {
                        (Ok(i), Ok(_)) => assert_eq!(data[i], target),
                        (Err(a), Err(b)) => assert_eq!(a, b, "{data:?} {target}"),
                        (mine, std) => panic!("{data:?} {target}: {mine:?} vs {std:?}"),
                    }
                }
            }
        }
    }
}
```

✓ With this exact file: `cargo fmt --check` clean, `cargo clippy --all-targets --locked -- -D warnings` clean, `cargo test --locked` → 3 passed.

**Design notes.**

- In `search`, the half-open interval `[lo, hi)` with `while lo < hi` has a clear invariant: the answer lies in `lo..=hi`. `mid = lo + (hi - lo) / 2` cannot overflow `usize` (the naive `(lo + hi) / 2` can for huge slices, a famous bug in widely used implementations). `hi - lo` cannot underflow because the loop condition guarantees `lo < hi`. `hi = mid` and `lo = mid + 1` shrink the interval every iteration, so the loop terminates.
- When the target occurs more than once, `Ok(i)` may be *any* matching index; the std documentation says the same, so the test compares `Ok` cases by value, not by index, and `Err` cases by exact insertion point.
- `run_lengths` uses `last_mut()` on the output vector and a match guard, avoiding any index arithmetic.
- The exhaustive test is a small preview of property-based testing (Lesson 40): with a small domain, you can check *all* inputs instead of hand-picking a few.

</details>

---

## 13. Interview questions

<details markdown="1">
<summary>Answers</summary>

1. **What is the difference between an expression and a statement in Rust?** An expression evaluates to a value; a statement performs an action and yields `()`. A trailing semicolon turns an expression into a statement. Blocks, `if`, `match`, and `loop` are expressions.
2. **Why does `fn f() -> i32 { 5; }` fail?** The `;` makes the body's final item a statement, so the block has type `()` rather than `i32` (E0308).
3. **Can `if` branches have different types?** No: `if` is an expression, so both branches must have the same type (or one must diverge with type `!`).
4. **What kinds of loops exist and which can return a value?** `loop`, `while`, `for`; only `loop` can `break` with a value, since it can exit only through `break`.
5. **What is a labeled block?** `'a: { ... break 'a value; ... }`, an expression that allows early exit with a value from inside a block.
6. **Array vs slice vs `Vec`?** `[T; N]`: fixed length known at compile time, stored inline. `[T]`: a dynamically sized view, used behind a pointer (`&[T]`), a fat pointer of address + length. `Vec<T>`: growable, heap-allocated, owning buffer.
7. **Why accept `&[T]` instead of `&Vec<T>`?** It accepts arrays, vectors, and sub-slices, and avoids tying the API to one container; `&Vec<T>` adds nothing since `Vec` derefs to a slice.
8. **What happens if you index out of bounds?** In safe Rust, a panic (not UB). `get` returns `None` instead. When the compiler can see the violation at compile time, the `unconditional_panic` lint rejects the program.
9. **How big is `&[i32]` and why?** Two words (16 bytes on 64-bit): pointer and length. `&[i32; 5]` is one word because the length is in the type.
10. **Does Rust guarantee tail-call optimization?** No. Deep recursion can overflow the stack, which aborts the process. Optimizers may remove the recursion in some builds but this is not a guarantee.
11. **Why does `split_at_mut` exist?** The borrow checker cannot see that two index ranges of one slice are disjoint; `split_at_mut` provides a safe API that returns two non-overlapping mutable slices (implemented with a small amount of `unsafe`).
12. **Why can't you `push` to a `Vec` while holding a reference to one of its elements?** `push` may reallocate and move the buffer, so the reference could dangle. The borrow checker rejects it (E0502).

</details>

---

## 14. Key takeaways

- Function signatures are fully explicit; the last expression without a semicolon is the return value. A stray `;` is a type error, not a silent bug.
- Rust is expression-oriented: blocks, `if`, `loop` produce values; conditions must be `bool`; branches must agree in type.
- `loop` can return a value; `for` over ranges and iterators is the default; labels and labeled blocks give structured early exit.
- Tuples are for small, ad-hoc groups (use structs once fields need names); their layout is unspecified.
- Arrays have their length in the type, are inline, and are bounds-checked. Slices are fat pointers (address + length) and are the right parameter type for sequences.
- Out-of-range slicing and indexing panic; `get`, `windows`, `chunks`, and iterators let you avoid index arithmetic. `usize` subtraction near zero is a repeating bug source (Lesson 04).
- No tail-call guarantee: bound recursion depth for untrusted input.

**Connections.** Lesson 04's `usize` underflow reappears in the debugging challenge; Lesson 01's bounds-check guarantee is now concrete. Next: Lesson 06 (strings as UTF-8 slices, text versus bytes, `From`/`TryFrom` in depth, and pattern matching with `match`). References and borrowing arrive in Lesson 10, `Vec` in Lesson 16, iterators in Lesson 17, closures in Lesson 18, and `unsafe` slice internals in Lesson 36. See the [roadmap](00_roadmap.md) and the [course index](index.md).

---

## 15. References

- *The Rust Programming Language*, ch. 3.3 "Functions": <https://doc.rust-lang.org/book/ch03-03-how-functions-work.html>
- *The Rust Programming Language*, ch. 3.5 "Control Flow": <https://doc.rust-lang.org/book/ch03-05-control-flow.html>
- *The Rust Programming Language*, ch. 4.3 "The Slice Type": <https://doc.rust-lang.org/book/ch04-03-slices.html>
- *The Rust Reference*, statements and expressions: <https://doc.rust-lang.org/reference/statements-and-expressions.html>
- *The Rust Reference*, loop expressions (labels, `break` with value): <https://doc.rust-lang.org/reference/expressions/loop-expr.html>
- *The Rust Reference*, array and slice types: <https://doc.rust-lang.org/reference/types/array.html>, <https://doc.rust-lang.org/reference/types/slice.html>
- Rust standard library documentation, primitive `slice`: <https://doc.rust-lang.org/std/primitive.slice.html>
- Rust standard library documentation, primitive `array` and primitive `tuple`: <https://doc.rust-lang.org/std/primitive.array.html>, <https://doc.rust-lang.org/std/primitive.tuple.html>
- Rust standard library documentation, `slice::binary_search`: <https://doc.rust-lang.org/std/primitive.slice.html#method.binary_search>
- Joshua Bloch, "Extra, Extra - Read All About It: Nearly All Binary Searches and Mergesorts are Broken" (Google Research blog, 2006), on the `(lo + hi) / 2` overflow.

The URLs above are the standard documentation locations; I did not open every one during this session, so if a link has moved, search the docs site for the page title.
