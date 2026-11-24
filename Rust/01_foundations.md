---
layout: default
title: "Lesson 01 — Foundations: what Rust is and why it works this way"
---

# Lesson 01 — Foundations: what Rust is and why it works this way

**Module 1 · Foundations and toolchain** · Status: generated · Edition used in examples: **2024** · Dependencies: none (`std` only)

All code in this lesson was compiled and run with `rustc 1.97.0 (2d8144b78 2026-07-07)` on x86-64 Linux, using `rustc --edition 2024`. The *Rust edition 2024* requires Rust **1.85.0 or newer**.

---

## 0. Prerequisites and minimal setup

You need a terminal, a text editor, and a working `rustc`. The full toolchain (rustup, Cargo, Clippy, rustfmt) is Lesson 02; for now the minimum is:

```text
# install rustup from https://www.rust-lang.org/tools/install, then:
rustc --version
```

Vocabulary used throughout:

- **Compiler**: a program that translates source code into another form (here: machine code).
- **Undefined behavior (UB)**: an execution for which the language imposes *no* requirements. The compiler is allowed to assume UB never happens, so a program that triggers it can crash, silently corrupt data, or appear to work. Most memory-safety vulnerabilities in C and C++ are UB in disguise.
- **Stack / heap** (informal for now): the stack holds fixed-size, short-lived data tied to function calls; the heap holds dynamically sized data that can outlive a call. Lesson 08 makes this precise.

---

## 1. What Rust is

Rust is a **statically typed, ahead-of-time compiled systems programming language** with **no garbage collector and no mandatory runtime**, whose **safe subset rules out undefined behavior** (use-after-free, double free, buffer overruns, data races, null dereferences, reads of uninitialized memory).

### 1.1 Historical context

- **2006**: Graydon Hoare starts Rust as a personal project.
- **Later**: Mozilla sponsors it; a motivating use case is a new, safer, parallel browser engine (Servo).
- **15 May 2015**: Rust **1.0**, with a *stability promise*: code that compiles on stable Rust keeps compiling on later stable releases, with narrow documented exceptions (for example soundness fixes and some type-inference changes).
- **2021**: the Rust Foundation is established; development is run by open teams under the Rust project.

### 1.2 The problem Rust answers

| Approach | What you get | What you pay |
|---|---|---|
| C / C++ | Full control of memory layout and performance | Memory safety is the programmer's job; the compiler does not stop UB |
| Garbage-collected languages (Java, Go, C#, Python…) | Memory safety | A runtime, less control over layout and pause times, and (in most of them) no compile-time guarantee against data races |
| **Rust** | Memory safety **and** C-like control | A stricter compiler, and the effort of learning *why* it rejects your program |

Public reports from large C/C++ codebases (Microsoft's security response center and the Chromium project have each published figures of roughly 70% for their own serious vulnerabilities) attribute a majority of severe bugs to memory-safety errors. That is the engineering motivation: move whole classes of bugs from *runtime and review* to *compile time*. (See references for the primary sources; treat the figures as per-codebase statistics, not a universal constant.)

### 1.3 Design philosophy

1. **Safety by construction in the safe subset.** Checked at compile time wherever possible, at run time otherwise (e.g. array bounds).
2. **Control without hidden costs.** You decide where data lives and when it is freed. Abstractions should not add run-time cost beyond what you would write by hand (Section 6).
3. **Explicit over implicit.** No implicit numeric conversions, no implicit deep copies, no null, no exceptions. Failure is a *value* (`Option`, `Result`).
4. **Immutable by default.** Mutation is opt-in (`mut`) and tracked by the type system.
5. **An honest escape hatch.** `unsafe` exists for what the compiler cannot verify; it is a *contract you take on*, localized and greppable (Lesson 36+).

---

## 2. Compiled languages and what `rustc` does

An **ahead-of-time (AOT) compiled** language translates the whole program to machine code *before* it runs. Contrast with an **interpreter** (executes source or bytecode directly) and a **JIT** (compiles hot code while running). The output of `rustc` is a native executable that needs no Rust runtime; it depends only on the operating system and, by default on `*-linux-gnu` targets, the platform C library.

### 2.1 Hello, world

```rust
fn main() {
    println!("Hello, Rust!");
}
```

```text
rustc --edition 2024 hello.rs     # produces ./hello
./hello
```

- `fn main()` is the entry point.
- `println!` ends in `!`: it is a **macro**, not a function. Macros run at compile time; here the format string is parsed and **checked against its arguments during compilation**, and macros can take a variable number of arguments, which ordinary functions cannot.
- `{c}` inside a format string captures the variable `c` by name.
- Statements end with `;`. A final expression *without* a semicolon is the block's value (Lesson 05).

> `rustc` invoked directly defaults to **edition 2015** unless you pass `--edition`. Cargo-generated projects set the edition explicitly. Always pass it.

### 2.2 The compilation pipeline (mental model)

```text
source → tokens → AST → (macro expansion, name resolution) → HIR
       → type checking (inference + trait solving) → THIR → MIR
       → borrow checking, MIR optimizations → monomorphization
       → LLVM IR → LLVM optimization → machine code → linker → executable
```

- **HIR/THIR**: desugared, typed forms of your program.
- **MIR** (mid-level IR): a control-flow-graph form on which the **borrow checker** runs. Most diagnostics you will read in this course (`E0382`, `E0499`, `E0502`, `E0505`…) come from here.
- **Monomorphization**: each generic function is compiled once *per concrete type* it is used with (Lesson 19). This is a major source of both speed and compile time.
- **LLVM** is the default code-generation backend; others exist, but treat LLVM as the baseline.

You can inspect intermediate forms yourself (verified on 1.97.0):

```text
rustc --edition 2024 -O --crate-type=lib --emit=mir     lib.rs
rustc --edition 2024 -O --crate-type=lib --emit=llvm-ir lib.rs
rustc --edition 2024 -O --crate-type=lib --emit=asm     lib.rs
```

Rust compiles more slowly than, say, Go. The causes are real work, not accidents: type inference and trait solving, borrow checking, monomorphization, and heavy LLVM optimization. Incremental compilation and build profiles (Lesson 03) mitigate it.

---

## 3. Static, strong typing

**Static**: every expression has a type fixed at compile time; type errors are compile errors.
**Strong**: the language does not silently convert between unrelated types.
**Inferred**: you rarely annotate local variables; the compiler infers types *within a function body*. Function signatures must be explicit, which keeps errors local and APIs readable.

Correct:

```rust
fn main() {
    let a = 5; // inferred as i32 (the default integer type)
    let b: u64 = 5;
    let c = a as u64 + b; // explicit conversion: Rust never widens implicitly
    println!("{c}");
}
```

Incorrect:

```rust
fn main() {
    let a: i32 = 5;
    let b: u64 = 5;
    let c = a + b; // error: i32 + u64
    println!("{c}");
}
```

The compiler rejects this with `error[E0308]: mismatched types` and `error[E0277]: cannot add `u64` to `i32``.

**Why?** Implicit widening, narrowing, and signed/unsigned mixing are classic sources of silent bugs in C-family languages. Rust forces you to write the conversion, so the decision is visible in review.

> `as` casts are *explicit but lossy*: integer-to-integer `as` truncates or sign-reinterprets, and float-to-integer `as` saturates. Prefer `From`/`TryFrom` when you can (Lesson 06). Treat `as` as a code smell in security-sensitive arithmetic.

---

## 4. Memory safety: what it means and how Rust gets it

In safe Rust, the compiler plus a few cheap runtime checks guarantee the absence of: use-after-free, double free, dangling references, buffer overruns, reads of uninitialized variables, null-reference dereference, and **data races**. The mechanisms (each gets a full lesson later):

| Mechanism | Rule | Prevents | Lesson |
|---|---|---|---|
| **Ownership** | Each value has exactly one owner; when the owner goes out of scope the value is dropped | Leaks-by-forgetting, double free, use-after-free | 07–08 |
| **Borrowing** | At any time: many shared references (`&T`) **or** one exclusive reference (`&mut T`) — "aliasing XOR mutability" | Iterator invalidation, data races, unexpected aliasing mutation | 10 |
| **Lifetimes** | A reference cannot outlive the value it points to | Dangling references | 11 |
| **No null; `Option<T>`** | "Maybe absent" is a type you must handle | Null dereference | 14 |
| **Definite initialization** | A variable must be assigned on every path before use | Uninitialized reads | 04 |
| **Bounds checks** | Indexing is checked at run time | Buffer overruns | 16 |
| **`Send` / `Sync`** | Types declare whether they can cross / be shared between threads | Data races | 22, 27 |

### 4.1 Use after move (a first taste of ownership)

Incorrect:

```rust
fn main() {
    let s = String::from("hi");
    let t = s; // ownership of the heap buffer moves to `t`
    println!("{s} {t}"); // error: `s` was moved
}
```

`error[E0382]: borrow of moved value: `s``. Why: `String` owns a heap buffer. If both `s` and `t` were valid, both would free it at scope end: a **double free**. Rust prevents that by making assignment a *move*: after `let t = s;`, `s` is unusable.

Correct (explicit deep copy):

```rust
fn main() {
    let s = String::from("hi");
    let t = s.clone(); // two independent heap buffers
    println!("{s} {t}");
}
```

### 4.2 Data races are compile errors

Incorrect:

```rust
use std::thread;

fn main() {
    let mut count = 0;
    let h = thread::spawn(|| {
        count += 1; // error: the thread may outlive `count`
    });
    h.join().unwrap();
    println!("{count}");
}
```

The compiler reports `error[E0373]: closure may outlive the current function, but it borrows `count``, and also `error[E0502]` for the later read. A spawned thread may run longer than the function that created `count`, so borrowing it is unsound.

Correct, using a **scoped thread**, which is guaranteed to finish before the scope ends:

```rust
use std::thread;

fn main() {
    let mut count = 0;
    thread::scope(|s| {
        s.spawn(|| {
            count += 1;
        });
    });
    println!("{count}");
}
```

Output: `1`. Same idea, now provably safe. Lesson 27 covers threads; keep the *principle*: the compiler reasons about who may access data, when.

### 4.3 What remains a run-time check: indexing

```rust
use std::hint::black_box;

fn main() {
    let v = vec![1, 2, 3];
    let i = black_box(10usize);

    println!("{:?}", v.get(i)); // None: the caller decides what to do

    println!("{}", v[i]); // panics, never reads past the buffer
}
```

Actual output: `None`, then a panic: `index out of bounds: the len is 3 but the index is 10`, process exit code **101**. In C the equivalent read is UB. (`black_box` hides the value from the optimizer and from compile-time lints so the check happens at run time; you will rarely need it outside demos and benchmarks.)

### 4.4 What Rust does **not** guarantee

"Memory safe" is **not** "bug free" or "secure". Safe Rust does not prevent:

| Not prevented | Notes |
|---|---|
| Logic errors, wrong business rules | The type system checks shapes, not intent |
| Memory **leaks** | Leaking is considered safe (`mem::forget`, `Rc` cycles). Leaks are not UB |
| Deadlocks, livelocks, starvation | Data races are prevented; *race conditions* (e.g. check-then-act on external state) are not |
| Panics and thus denial of service | `unwrap`, indexing, overflow-in-debug, `RefCell` double borrow all panic |
| Resource exhaustion | Unbounded input still exhausts memory/CPU |
| Bugs in `unsafe` code, including inside dependencies | Safe code is only as sound as the `unsafe` beneath it |
| Compiler or LLVM bugs | Rare, but they exist and are tracked |
| Cryptographic misuse, side channels, supply-chain compromise | Entirely outside the borrow checker's scope |

---

## 5. Integer overflow: defined, but build-dependent

Overflow is **not UB** in Rust. What happens depends on the profile:

- **Debug** (default `dev` profile): `overflow-checks` on → the operation **panics**.
- **Release** (default `release` profile): checks off → the result **wraps** (two's complement). Configurable via the `overflow-checks` profile setting.

You choose behavior explicitly with method families: `checked_*` → `Option`, `wrapping_*`, `saturating_*`, `overflowing_*` → `(value, bool)`.

```rust
use std::hint::black_box;

fn main() {
    let x: u8 = 255;
    println!("checked    : {:?}", x.checked_add(1));
    println!("wrapping   : {}", x.wrapping_add(1));
    println!("overflowing: {:?}", x.overflowing_add(1));
    println!("saturating : {}", x.saturating_add(1));

    let y = black_box(255u8);
    let z = y + 1; // debug: panics. release (default profile): wraps to 0.
    println!("plain +    : {z}");
}
```

Observed on rustc 1.97.0:

```text
# rustc --edition 2024 ex.rs           (debug)
checked    : None
wrapping   : 0
overflowing: (0, true)
saturating : 255

thread 'main' panicked at ex.rs:13:13:
attempt to add with overflow          # exit code 101

# rustc --edition 2024 -O ex.rs        (optimized, overflow checks off)
checked    : None
wrapping   : 0
overflowing: (0, true)
saturating : 255
plain +    : 0                         # silently wrapped, exit code 0
```

**Security implication:** code whose correctness depends on a plain `+` (balances, lengths, offsets, token amounts) behaves *differently* in debug and release. For anything where wrap-around is a vulnerability, use `checked_*` and handle `None`, or widen the type. This matters enormously in blockchain code (Lessons 53–55).

---

## 6. Zero-cost abstractions

The term (from C++) has two halves: **you don't pay for what you don't use**, and **what you do use, you could not reasonably hand-write any faster**. In Rust it covers iterators, closures, `Option`, generics, and newtypes: they are designed to compile down to the code you would have written manually.

```rust
fn sum_squares_loop(v: &[u32]) -> u64 {
    let mut total = 0u64;
    for &x in v {
        total += (x as u64) * (x as u64);
    }
    total
}

fn sum_squares_iter(v: &[u32]) -> u64 {
    v.iter().map(|&x| (x as u64) * (x as u64)).sum()
}

fn main() {
    let data: Vec<u32> = (1..=1000).collect();
    assert_eq!(sum_squares_loop(&data), sum_squares_iter(&data));
    println!("{}", sum_squares_iter(&data));
}
```

Output: `333833500`. What I **did** verify (rustc 1.97.0, `-O`, x86-64, `--emit=asm`): both functions compile to **vectorized loops with a scalar tail, using SSE instructions, with no function calls**. The instruction sequences are structurally similar but **not byte-identical**, so do not read "zero-cost" as "identical assembly". I did not benchmark them and make no speed claim; Lesson 41 teaches how to measure honestly.

**Layout is another zero-cost example.** Measured on this machine (64-bit):

```text
&u32              = 8        Option<&u32>      = 8
Box<u32>          = 8        Option<Box<u32>>  = 8
u32               = 4        Option<u32>       = 8
String            = 24       Vec<u8>           = 24
&str              = 16       ()                = 0
```

`Option<&u32>` is the same size as `&u32`: references can never be null, so the compiler reuses the null bit-pattern to represent `None` (a *niche optimization*). You get a null-safe type for free. `Option<u32>` is larger because every `u32` bit pattern is valid, so a separate tag is needed.

**What "zero-cost" does *not* mean:**

- Not zero **compile-time** cost (monomorphization, inlining).
- Not zero **binary-size** cost (each instantiation is separate machine code).
- Not zero **cognitive** cost (abstraction layers can obscure what runs).
- Not guaranteed: it depends on the optimizer. **Debug builds do not optimize**, and iterator chains in debug mode can be dramatically slower than in release. Never judge Rust performance from a debug build.

---

## 7. Editions

An **edition** is an opt-in, per-crate bundle of changes that would otherwise be breaking. Existing editions: **2015**, **2018**, **2021**, **2024** (2024 stabilized in Rust **1.85.0**, 2025-02-20). New ones appear roughly every three years.

Key properties:

- Set per crate: `edition = "2024"` in `Cargo.toml` (or `--edition 2024` for `rustc`).
- **Crates of different editions interoperate freely** and link into one program; editions change how *source* is parsed and some surface semantics, not the compiled ABI-level model. A 2024 crate can depend on a 2015 crate and vice versa.
- Migration is assisted: `cargo fix --edition` (Lesson 02).
- Editions never remove the stability promise; they are how Rust evolves without a "Rust 2.0" break.

Illustrative changes (consult the Edition Guide for the authoritative per-edition list): 2018 made `async` a keyword and introduced non-lexical lifetimes (a more precise borrow checker); 2021 changed closure capture to capture disjoint fields; 2024 reserved the `gen` keyword, changed return-position `impl Trait` lifetime capture rules, requires `unsafe extern` blocks, and adjusted the scope of some temporaries (tail expressions, `if let`).

---

## 8. Stable, beta, nightly

Rust ships on a **six-week train**: `nightly` → `beta` → `stable`.

| Channel | Purpose | Experimental features |
|---|---|---|
| **stable** | Production; covered by the stability promise | No |
| **beta** | Stable-to-be; testing the next release | No |
| **nightly** | Daily builds; unstable features behind `#![feature(...)]` | **Yes**, may change or vanish |

Rules of thumb:

- Default to **stable**. Only reach for nightly for a specific, understood reason (some tooling such as certain Miri/sanitizer/`-Z` workflows, or a feature you accept may break).
- Libraries should declare an **MSRV** (`rust-version` in `Cargo.toml`) and test against it.
- Pin a toolchain per project with `rust-toolchain.toml` (Lesson 02).
- This course uses **stable only**, and flags anything experimental explicitly.

---

## 9. Common mistakes and edge cases

1. **Expecting a garbage-collected mental model.** Assignment of non-`Copy` values *moves*; copying is explicit (`.clone()`).
2. **Cloning to silence the borrow checker.** It compiles, but you have hidden a design question and added allocations. Understand the diagnostic first (Lessons 10–12).
3. **Trusting plain `+` in release builds** (Section 5).
4. **Benchmarking a debug build.**
5. **Compiling with bare `rustc` and forgetting `--edition`**, silently getting 2015 rules.
6. **Reading "safe" as "secure"** (Section 4.4).
7. **Using `as` for narrowing conversions** without thinking about truncation.
8. **`unwrap()` in production paths**: a panic is a crash and, in a service, a potential denial of service.

---

## 10. Security and reliability notes

- Memory safety removes a *large class* of vulnerabilities; it does not remove input-validation, authorization, protocol, cryptographic, or logic vulnerabilities.
- Panics are reliability events. Decide deliberately whether a failure is *recoverable* (`Result`) or a *bug* (panic), and keep untrusted input from reaching panicking paths.
- Arithmetic on externally controlled numbers must be explicit about overflow (`checked_*`).
- Every `unsafe` block and every dependency that uses `unsafe` is part of your trusted computing base.

---

## 11. Exercises

Use only `rustc --edition 2024` for now. Mark them done yourself; they are not pre-checked.

1. **Run it.** Compile and run Hello, Rust. Then change `println!` to `println` (no `!`) and read the error. What does it tell you about macros?
2. **Predict, then verify.** Without compiling, predict the error code for the use-after-move example, then compile and compare. Fix it in two different ways (hint: clone, or reorder so `s` is not used after the move).
3. **Types.** Make the failing `i32 + u64` example compile in two different ways: convert `a` with `a as u64`, and with `u64::try_from(a).unwrap()`. Which one is safer if `a` could be negative, and why? (Try `a = -1` and compare.)
4. **Observe overflow.** Compile the overflow program in debug and with `-O`. Record both outputs. Then add `-C overflow-checks=on` to the `-O` build and note the difference.
5. **Safe sum.** Write `fn checked_sum(values: &[u8]) -> Option<u8>` that returns `None` on overflow, using `checked_add` and `?`. Test it on `[]`, `[255]`, `[255, 1]`.
6. **Bounds.** Write a function `fn get_or_default(v: &[i32], i: usize) -> i32` that never panics. Then explain why `v[i]` panics instead of reading garbage.
7. **Inspect.** Compile the two `sum_squares` functions with `--emit=asm -O` and find the vectorized loop in each. What differs?
8. **Reasoning.** A teammate says: "Rust guarantees our service is secure." Write three concrete counterexamples from Section 4.4.

---

## 12. Debugging challenge

This program does not compile. **Predict the error before compiling**, then explain, in your own words, what bug the compiler is preventing.

```rust
fn longest_name() -> &String {
    let s = String::from("ferris");
    &s
}

fn main() {
    println!("{}", longest_name());
}
```

<details markdown="1">
<summary>Solution (open after you try)</summary>

The compiler reports `error[E0106]: missing lifetime specifier`. A reference return type needs to say *which input* it borrows from, but this function has **no inputs**, so there is nothing valid to borrow from.

Even if you could silence that, the real bug is a **dangling reference**: `s` is owned by the function and dropped when it returns, so `&s` would point to freed memory. In C this compiles and yields UB. Fixes: return the owned value (`-> String`, `s`), or return a `&'static str` for a literal. Lesson 11 explains lifetimes properly.

</details>

---

## 13. Interview questions

<details markdown="1">
<summary>1. What does "memory safe" mean in Rust, and what are its limits?</summary>

In safe Rust, the compiler (ownership, borrowing, lifetimes, `Send`/`Sync`, definite initialization) and a few run-time checks (bounds) guarantee no use-after-free, double free, dangling references, buffer overruns, uninitialized reads, null dereference, or data races. Limits: `unsafe` code and its dependencies, compiler bugs, and everything that is not memory safety (logic bugs, leaks, deadlocks, panics, overflow semantics, crypto/auth/supply-chain issues).

</details>

<details markdown="1">
<summary>2. Does Rust prevent memory leaks?</summary>

No. Leaks are memory-*safe*: `std::mem::forget` and `Rc`/`Arc` reference cycles leak without UB. RAII frees memory in the normal case, but it cannot prove every value is eventually dropped.

</details>

<details markdown="1">
<summary>3. What is a zero-cost abstraction? Give an example and a caveat.</summary>

An abstraction you don't pay for unless you use it and that compiles to roughly what you'd write by hand. Example: iterator chains, `Option<&T>` having the same size as `&T`. Caveats: it relies on the optimizer (debug builds are slow), and it can cost compile time and binary size (monomorphization).

</details>

<details markdown="1">
<summary>4. What happens on integer overflow?</summary>

It is defined behavior, not UB. With overflow checks on (default in debug) it panics; with them off (default in release) it wraps in two's complement. Use `checked_*`, `wrapping_*`, `saturating_*`, or `overflowing_*` to state the intended behavior explicitly.

</details>

<details markdown="1">
<summary>5. Can crates with different editions be used together?</summary>

Yes. The edition is per crate; all crates are compiled by the same compiler into a shared internal representation, so a 2024-edition crate can depend on a 2018-edition one and vice versa.

</details>

<details markdown="1">
<summary>6. Stable vs. nightly: when would you use nightly, and how do you stop it leaking into a project?</summary>

Use nightly only for a specific need (unstable features, certain tools). Contain it by pinning with `rust-toolchain.toml`, isolating nightly-only code, documenting why, and keeping CI on stable for everything that can be. Stable is covered by the stability promise; nightly features can change or be removed.

</details>

<details markdown="1">
<summary>7. Why is `println!` a macro?</summary>

Because it needs a variadic, compile-time-checked format string. The macro parses the format string at compile time and verifies argument count and trait requirements (`Display`/`Debug`), which an ordinary function cannot do.

</details>

---

## 14. Practical challenge

**Overflow-safe summation.** Create `sums.rs` with three functions over `&[u8]`:

1. `checked_sum` → `Option<u8>`: `None` if the true sum does not fit.
2. `saturating_sum` → `u8`: clamps at `u8::MAX`.
3. `exact_sum` → `u64`: always exact.

Add a `main` that prints all three for `[10, 20, 30]` and `[200, 100]`, and unit tests covering the empty slice, a single `255`, and `[255, 1]`. Run with:

```text
rustc --edition 2024 sums.rs && ./sums
rustc --edition 2024 --test sums.rs && ./sums
```

Do not look at the solution until you have tried it.

<details markdown="1">
<summary>Solution (verified: compiles, runs, 3 tests pass on rustc 1.97.0)</summary>

```rust
/// Sums `values` without ever wrapping or panicking.
/// Returns `None` if the true sum does not fit in a `u8`.
fn checked_sum(values: &[u8]) -> Option<u8> {
    let mut total: u8 = 0;
    for &v in values {
        total = total.checked_add(v)?; // `?` returns None early on overflow
    }
    Some(total)
}

/// Sums with saturation: clamps at `u8::MAX` instead of failing.
fn saturating_sum(values: &[u8]) -> u8 {
    values.iter().fold(0u8, |acc, &v| acc.saturating_add(v))
}

/// Widens first, so the sum is always exact.
fn exact_sum(values: &[u8]) -> u64 {
    values.iter().map(|&v| u64::from(v)).sum()
}

fn main() {
    let small = [10u8, 20, 30];
    let big = [200u8, 100];

    for data in [&small[..], &big[..]] {
        println!(
            "{:?} -> checked: {:?}, saturating: {}, exact: {}",
            data,
            checked_sum(data),
            saturating_sum(data),
            exact_sum(data)
        );
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn checked_detects_overflow() {
        assert_eq!(checked_sum(&[10, 20, 30]), Some(60));
        assert_eq!(checked_sum(&[200, 100]), None);
        assert_eq!(checked_sum(&[]), Some(0));
        assert_eq!(checked_sum(&[255]), Some(255));
        assert_eq!(checked_sum(&[255, 1]), None);
    }

    #[test]
    fn saturating_clamps() {
        assert_eq!(saturating_sum(&[200, 100]), 255);
        assert_eq!(saturating_sum(&[1, 2, 3]), 6);
    }

    #[test]
    fn exact_never_overflows() {
        assert_eq!(exact_sum(&[200, 100]), 300);
        assert_eq!(exact_sum(&[255; 1000]), 255_000);
    }
}
```

Observed output:

```text
[10, 20, 30] -> checked: Some(60), saturating: 60, exact: 60
[200, 100] -> checked: None, saturating: 255, exact: 300
```

Design notes: `?` on `Option` propagates `None` (Lesson 14 explains it fully). `u64::from(v)` is a lossless `From` conversion, preferable to `as`. The three functions encode three *different contracts*; choosing between them is an API design decision, not a style choice.

</details>

---

## 15. Key takeaways

- Rust = AOT-compiled, statically typed, no GC, memory-safe in its safe subset, with an explicit `unsafe` escape hatch.
- The central trade: **do the hard reasoning at compile time** (ownership, borrowing, lifetimes) so run time stays fast and safe.
- Memory safety ≠ security ≠ correctness. Know the table in Section 4.4.
- Overflow is defined but profile-dependent; state your intent with `checked_*`/`wrapping_*`/`saturating_*`.
- Zero-cost means *no unnecessary run-time overhead*, not free compile time, free binary size, or guaranteed identical machine code. Measure, never assume.
- Editions evolve syntax per crate and interoperate; use stable by default.

**Connections.** This is the first lesson, so it plants the ideas the next ones expand: toolchain and Cargo (Lesson 02), numeric types and mutability (Lesson 04), moves and `Drop` (Lessons 07–08), references and lifetimes (Lessons 10–11), threads and `Send`/`Sync` (Lessons 22, 27), unsafe (Lessons 36–38). See the [roadmap](00_roadmap.md) for the full path and the [course index](index.md) for navigation.

---

## 16. References

- *The Rust Programming Language* (the Book), ch. 1 and 3.2 (integer overflow): <https://doc.rust-lang.org/book/>
- *The Rust Reference*, "Behavior considered undefined": <https://doc.rust-lang.org/reference/behavior-considered-undefined.html>
- *The Rust Reference*, overflow in operator expressions: <https://doc.rust-lang.org/reference/expressions/operator-expr.html#overflow>
- *The Rust Edition Guide*: <https://doc.rust-lang.org/edition-guide/>
- Rust 2024 edition, stabilized in 1.85.0: <https://doc.rust-lang.org/edition-guide/rust-2024/index.html>
- *Announcing Rust 1.0* (2015-05-15): <https://blog.rust-lang.org/2015/05/15/Rust-1.0/>
- Rust Forge (current channel versions and the release process): <https://forge.rust-lang.org/>
- *The Rustonomicon* (what safe Rust guarantees and what `unsafe` promises): <https://doc.rust-lang.org/nomicon/>
- *The rustc Dev Guide* (compiler pipeline, HIR/MIR): <https://rustc-dev-guide.rust-lang.org/>
- Microsoft Security Response Center, "A proactive approach to more secure code" (2019): <https://www.microsoft.com/en-us/msrc/blog/2019/07/a-proactive-approach-to-more-secure-code>
- Chromium Project, "Memory safety": <https://www.chromium.org/Home/chromium-security/memory-safety/>
