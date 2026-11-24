---
layout: default
title: "Lesson 36 — Unsafe contracts, raw pointers, and undefined behavior"
---

# Lesson 36 — Unsafe contracts, raw pointers, and undefined behavior

**Module 12 · Unsafe Rust and low-level programming** · Status: generated · Edition used in examples: **2024** · External dependencies: none

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux. As before, ✓ marks outputs, diagnostics and measurements I actually ran; unmarked statements are documented behavior I did not run. Everything is **stable Rust**. **Miri is not installed in this environment and I could not install it** (`rustup component add miri` failed), so nothing in this lesson was checked with Miri; Lesson 38 covers it. What I *did* use to look for undefined behavior (UB): the debug-build checks inside the standard library, `valgrind`, and AddressSanitizer (the latter only through `RUSTC_BOOTSTRAP=1 -Zsanitizer=address`, an unsupported trick on the stable compiler that I used just to show you its output). I could not open the Rust documentation sites, so the language rules below are from memory plus what the compiler accepted; the closing paragraph lists them.

Builds on [Lesson 33](33_modules_api_design_rustdoc.md) (privacy is what keeps an unsafe abstraction sound; sealed traits), [Lesson 28](28_atomics_memory_ordering.md) (data races and atomics), [Lesson 22](22_gats_send_sync_phantom.md) (`Send`/`Sync`, `PhantomData`), and [Lesson 34](34_declarative_macros.md) (the `plain!` macro below). Lesson 37 (`MaybeUninit`, FFI, `repr(C)`) is next; it appears here only as plain text.

---

## 1. What `unsafe` means

Rust has two languages: the **safe** one, in which the compiler proves that no UB can happen, and the **unsafe** one, in which *you* promise it. The keyword `unsafe` does not turn the compiler's checks off (the borrow checker and the type checker still run on unsafe code); it unlocks a short list of operations that the compiler cannot verify:

1. dereference a **raw pointer** (`*const T`, `*mut T`);
2. call an **`unsafe fn`** (including most FFI functions and methods like `get_unchecked`);
3. access or modify a **`static mut`**;
4. implement an **`unsafe trait`** (such as `Send`, `Sync`, `GlobalAlloc`);
5. access a **field of a `union`**;
6. (edition 2024) declare `unsafe extern` blocks and use `unsafe` attributes such as `#[unsafe(no_mangle)]` (from memory; Lesson 37 covers FFI).

Creating, casting and comparing raw pointers is safe, but most pointer *methods* (`add`, `read`, `write`, `copy`) are `unsafe fn`s, so you meet rule 2 there (§3.2). Do not guess whether an operation is safe: read its signature.

The compiler reports each missing `unsafe` with `E0133` and a note saying why ✓:

```rust
// ERROR E0133
fn main() {
    let x = 5;
    let p = &x as *const i32;      // creating a raw pointer is safe
    println!("{}", *p);            // dereferencing is not
}
```

```text
error[E0133]: dereference of raw pointer is unsafe and requires unsafe block
  = note: raw pointers may be null, dangling or unaligned; they can violate aliasing rules and cause data races: all of these are undefined behavior
```

```rust
// ERROR E0133
unsafe fn danger() -> i32 { 1 }
fn main() { println!("{}", danger()); }
```

```text
error[E0133]: call to unsafe function `danger` is unsafe and requires unsafe block
  = note: consult the function's documentation for information on how to avoid undefined behavior
```

```rust
// ERROR E0133
union IntOrFloat { i: u32, f: f32 }
fn main() {
    let u = IntOrFloat { f: 1.0 };
    println!("{:#x}", u.i);
}
```

```text
error[E0133]: access to union field is unsafe and requires unsafe block
  = note: the field may not be properly initialized: using uninitialized data will cause undefined behavior
```

With the block the same program prints `0x3f800000` ✓, identical to the safe `1.0f32.to_bits()` ✓. **Prefer the safe API when there is one** (`to_bits`, `from_le_bytes`, `get`, `split_at_mut`); the union is only needed when no safe equivalent exists.

### 1.1 Soundness is the real goal

An API is **sound** if no sequence of *safe* calls can cause UB. An API is **unsound** if it can, even if "nobody would call it that way". Unsound safe APIs are bugs in the library, not in the caller. That is the whole contract of unsafe Rust:

- Inside an `unsafe` block you uphold the preconditions of each unsafe operation.
- Around it you build a safe API whose invariants guarantee those preconditions for *every* safe caller.
- If the argument depends on a caller's good behavior, the function must be marked `unsafe fn` (or `unsafe trait`), and its `# Safety` documentation states the obligation.

### 1.2 The edition-2024 details ✓

Two defaults tightened in edition 2024.

An `unsafe fn` body is no longer implicitly an unsafe block; operations in it still need `unsafe { }`. Without it ✓:

```rust
unsafe fn read(p: *const i32) -> i32 {
    *p
}
fn main() { let x = 3; println!("{}", unsafe { read(&x) }); }
```

```text
warning[E0133]: dereference of raw pointer is unsafe and requires unsafe block
  = note: `#[warn(unsafe_op_in_unsafe_fn)]` (part of `#[warn(rust_2024_compatibility)]`) on by default
```

(It is a warning in edition 2024; the program still ran and printed `3`. `rawlab` upgrades it with `#![deny(unsafe_op_in_unsafe_fn)]`.) The point: the obligations *of calling* the function (`# Safety`) and the obligations *inside* it are now visibly separate, and each unsafe operation inside gets its own small block with its own justification.

Taking a **reference** to a `static mut` is a hard error ✓:

```rust
// ERROR (no code): deny-by-default lint `static_mut_refs`
static mut COUNTER: u32 = 0;
fn main() {
    unsafe {
        COUNTER += 1;
        println!("{}", COUNTER);
    }
}
```

```text
error: creating a shared reference to mutable static
  = note: shared references to mutable statics are dangerous; it's undefined behavior if the static is mutated or if a mutable reference is created for it while the shared reference lives
  = note: `#[deny(static_mut_refs)]` (part of `#[deny(rust_2024_compatibility)]`) on by default
```

(`println!` takes a reference.) Raw-pointer syntax avoids the reference: `let p = &raw const COUNTER; p.read()` ✓ compiles and prints `1`. But the real fix is not to use `static mut` at all: `AtomicU32`, `Mutex`, `OnceLock` or `thread_local!` ([Lesson 28](28_atomics_memory_ordering.md), [Lesson 29](29_locks_deadlocks.md)). A `static mut` accessed from two threads is a data race, which is UB.

---

## 2. Undefined behavior

**UB** means the program has no defined meaning. It is not "implementation-defined" or "a crash": the compiler *assumes* UB never happens and optimizes accordingly, so a program that contains it can misbehave anywhere, including before the offending line, and differently in debug and release builds, or on the next compiler version. A test passing says nothing.

The list of UB in Rust (from the Reference, from memory; it is not guaranteed complete, and the aliasing model is still being finalized) includes:

- **Dereferencing** a pointer that is null, dangling (the allocation was freed, or the pointee's scope ended), out of bounds, or **misaligned** for its type.
- Producing a **reference** (or a value of any type) that is not valid: a `&T`/`&mut T` that is null, dangling or misaligned (even if never used); a `bool` that is not 0 or 1; a `char` outside the Unicode scalar range; an enum with an invalid discriminant; a `str` that is not UTF-8; reading **uninitialized** memory as an integer, float, `bool`, or pointer.
- Breaking the **aliasing rules**: a `&mut T` must be the only way to reach that memory while it is live; a `&T` means the memory is not modified (except through `UnsafeCell`) while it is live. The precise rules are being modeled as *Stacked Borrows* and *Tree Borrows* (from memory), which Miri implements.
- **Data races**: unsynchronized concurrent access where at least one is a write ([Lesson 28](28_atomics_memory_ordering.md)).
- Calling a function with the **wrong ABI**, unwinding through a frame that does not allow it, and violating a `# Safety` condition of a standard library function.
- Mutating immutable data (a `static` that is not `mut`, a `const`, memory behind a `&T` without `UnsafeCell`).
- Misusing `unsafe` intrinsics such as `unreachable_unchecked` and `unchecked_add`, and creating values from `transmute` that are invalid for the target type.

What is **not** UB (and so safe code may do it): integer overflow (it panics in debug or wraps in release, both defined), memory leaks (`mem::forget`, `Rc` cycles), deadlocks, panics, and logic errors. Rust does not promise a program is correct or secure, only that safe code cannot cause the UB above.

### 2.1 What the standard library checks for you in debug builds ✓

Since recent compiler versions, standard-library `unsafe` functions contain checks of their preconditions that are **enabled when the *user's* crate is compiled with debug assertions**. Violating one is a panic that aborts, with a message like the following (each program below is a one-liner I ran in a debug build; every one exited with status 134):

| Program | Message ✓ |
|---|---|
| `*v.get_unchecked(10)` on a 3-element `Vec` | `unsafe precondition(s) violated: slice::get_unchecked requires that the index is within the slice` |
| `slice::from_raw_parts(p.add(1) as *const u32, 2)` from a `[u8; 16]` | `slice::from_raw_parts requires the pointer to be aligned and non-null, and the total size of the slice not to exceed `isize::MAX`` |
| `NonNull::new_unchecked(null_mut())` | `NonNull::new_unchecked requires that the pointer is non-null` |
| `ptr::copy_nonoverlapping(p, p.add(1), 3)` | `ptr::copy_nonoverlapping requires that both pointer arguments are aligned and non-null and the specified memory ranges do not overlap` |
| `char::from_u32_unchecked(0xD800)` | `invalid value for `char`` |
| `u32::MAX.unchecked_add(1)` | `u32::unchecked_add cannot overflow` |
| `mem::zeroed::<&u8>()` | `attempted to zero-initialize type `&u8`, which is invalid` (also aborts) |

Every such message ends with: *"This Undefined Behavior check is optional, and cannot be relied on for safety."* That sentence is the point. Two things to take away:

- Some UB is also caught **at compile time**: `slice::from_raw_parts(ptr::null::<u32>(), 1)` fails to compile with `error: calling this function with a null pointer is undefined behavior, even if the result of the function is unused` ✓.
- Much UB is **not** detected. `MaybeUninit::<bool>::uninit().assume_init()` is UB, and in my run it printed `false` and exited with status 0 ✓: no check, no diagnostic, and a nondeterministic result is also an allowed outcome. The checks are off in release builds (from memory: they follow `debug_assertions`), and a lucky run proves nothing.

---

## 3. Raw pointers

`*const T` and `*mut T` are pointers without lifetimes, borrow checking, non-nullness or validity guarantees. They are `Copy`, can be null, can be compared and cast, and are neither `Send` nor `Sync` by default.

### 3.1 Creating and casting (safe)

```rust
use std::ptr::{self, NonNull};

fn main() {
    let mut v = [10, 20, 30, 40];
    let p = v.as_mut_ptr();            // *mut i32 from a slice
    let c = p as *const i32;           // cast *mut -> *const (safe)
    let b = p.cast::<u8>();            // change the pointee type (safe; usage is not)
    let null: *const i32 = ptr::null();
    println!("{} {}", null.is_null(), c == p as *const i32);
    let _ = b;
    // References coerce to raw pointers:
    let x = 5;
    let r: *const i32 = &x;
    println!("{}", r.is_null());
    let nn = NonNull::new(p).unwrap();
    println!("{}", unsafe { *nn.as_ptr() });
}
```

Use `&raw const place` / `&raw mut place` (stable) to take a raw pointer to a place **without creating an intermediate reference**. This matters for packed structs and for places that are not valid for a reference. The compiler enforces it for packed fields ✓:

```rust
// ERROR E0793
#[repr(C, packed)]
struct P { a: u8, b: u32 }
fn main() {
    let p = P { a: 1, b: 2 };
    let r = &p.b;
    println!("{}", r);
}
```

```text
error[E0793]: reference to field of packed struct is unaligned
  = note: creating a misaligned reference is undefined behavior (even if that reference is never dereferenced)
  = help: copy the field contents to a local variable, or replace the reference with a raw pointer and use `read_unaligned`/`write_unaligned` ...
```

The accepted way ✓ (prints `1 2 size=5 align=1`):

```rust
#[repr(C, packed)]
struct P { a: u8, b: u32 }
fn main() {
    let p = P { a: 1, b: 2 };
    let b = unsafe { (&raw const p.b).read_unaligned() };
    let a = p.a;
    println!("{a} {b} size={} align={}", size_of::<P>(), align_of::<P>());
}
```

`NonNull<T>` is a raw pointer that is never null (and covariant in `T`). It gives `Option<NonNull<T>>` the same size as a pointer thanks to the null niche ✓:

```rust
use std::ptr::NonNull;
fn main() {
    println!("{} {}", size_of::<Option<NonNull<u8>>>(), size_of::<Option<*const u8>>());
}
```

Prints `8 16` ✓: `Option<*const u8>` needs a separate tag, `Option<NonNull<u8>>` uses the null pattern as `None`.

### 3.2 Arithmetic, reads and writes

```rust
use std::ptr;

fn main() {
    let mut v = [10, 20, 30, 40];
    let p = v.as_mut_ptr();
    unsafe {
        let mut sum = 0;
        for i in 0..v.len() {
            sum += *p.add(i);
        }
        println!("sum = {sum}");
        p.add(1).write(99);
        ptr::swap(p, p.add(3));
        println!("{v:?}");
    }
}
```

✓ Prints `sum = 100` and `[40, 99, 30, 10]`.

Key rules for `ptr.add(n)`, `sub`, `offset` (from memory of the docs): the result **and the whole range between** must stay inside the same allocation (or one byte past its end); that is a requirement on the *arithmetic*, not only on the later dereference. `wrapping_add` has no such requirement but also gives you no right to dereference. `offset_from` needs both pointers in the same allocation. For zero-sized types `add` does not move the address.

Reads and writes (all `unsafe`):

| Function | Requirement / meaning |
|---|---|
| `*p`, `p.read()` / `ptr::read(p)` | `p` valid for reads, aligned, pointing to an initialized valid `T`; **copies the bits** (it does not move or invalidate the source, so for non-`Copy` types you now own a second "owner": avoid double drops) |
| `*p = v`, `p.write(v)` / `ptr::write` | valid for writes, aligned; `write` does **not** drop the old value (`*p = v` does) |
| `read_unaligned`, `write_unaligned` | no alignment requirement ✓ |
| `ptr::copy(src, dst, n)` | like `memmove`: ranges may overlap |
| `ptr::copy_nonoverlapping` | like `memcpy`: ranges must not overlap ✓ (debug check above) |
| `ptr::swap(a, b)` | `a` and `b` may be the same or overlap |
| `ptr::drop_in_place(p)` | runs the destructor in place; the memory is then uninitialized |
| `slice::from_raw_parts(_mut)(p, n)` | `p` aligned, non-null (even for `n == 0`), `n` elements initialized, total size ≤ `isize::MAX`, not mutated for the lifetime; for `_mut`, no other access either |

### 3.3 Alignment, provenance, and integer casts ✓

A pointer to a `u32` must be 4-byte aligned. A byte buffer's address is only guaranteed 1-aligned, so casting `*const u8` to `*const u32` and dereferencing is UB unless you know the address is aligned. Use `read_unaligned`, or `is_aligned()` to check:

```rust
fn main() {
    let x = 7u32;
    let p = &x as *const u32;
    let a = p.addr();
    let q = p.with_addr(a);              // keeps provenance
    println!("{}", unsafe { *q });
    let e = p.expose_provenance();
    let r = std::ptr::with_exposed_provenance::<u32>(e);
    println!("{}", unsafe { *r });
    let bytes = [1u8, 0, 0, 0, 2, 0, 0, 0, 3];
    let bp = bytes.as_ptr();
    println!("{} {}", bp.is_aligned(), unsafe { bp.add(1).cast::<u32>().is_aligned() });
    println!("{}", unsafe { bp.add(1).cast::<u32>().read_unaligned() });
}
```

✓ Prints `7`, `7`, `true false` (the second pointer is 1 byte past the first, so misaligned for `u32`), and `33554432` (the four bytes `[0, 0, 0, 2]` read little-endian).

**Provenance**: a pointer is not just an address. It also carries *which allocation it may access* (and, under Tree Borrows, with what permission; from memory). Two pointers with equal addresses are not interchangeable if their provenance differs. `ptr.addr()` gives the address without provenance, `ptr.with_addr(a)` rebuilds a pointer from a new address while keeping the original provenance, and `expose_provenance()` / `with_exposed_provenance()` are the explicit escape hatch for pointer↔integer round trips ✓ (these four APIs compiled and ran on `rustc 1.97.0`; the model behind them is "strict provenance", from memory). Rule of thumb: do arithmetic on pointers, not on integers cast from pointers.

---

## 4. Contracts: how to write and review unsafe code

### 4.1 Three places a contract appears

- `unsafe fn`: the **caller** must satisfy conditions the compiler can't check. Document them in a `# Safety` section; a reviewer needs to be able to check a call site against it.
- `unsafe trait`: the **implementor** promises something (like `Send`: "this type may be moved across threads"; `Plain` below: "every bit pattern is valid").
- `unsafe { ... }`: the author **discharges** an obligation: they claim to have satisfied the preconditions of each unsafe operation inside. A `// SAFETY:` comment states how.

```rust
use std::ptr;

/// Moves the value out of `slot`, leaving it logically uninitialized.
///
/// # Safety
///
/// * `slot` must be valid for reads, aligned, and point to an initialized `T`.
/// * The caller must not use or drop the value at `slot` again.
pub unsafe fn take_raw<T>(slot: *mut T) -> T {
    // SAFETY: the caller promised validity, alignment and initialization, and
    // that the value will not be read or dropped again, so duplicating the bits
    // into the return value does not create two owners.
    unsafe { ptr::read(slot) }
}

fn main() {
    let mut s = std::mem::ManuallyDrop::new(String::from("hi"));
    // SAFETY: `s` is a live, initialized String; ManuallyDrop ensures it is
    // never dropped, so the returned String is the only owner.
    let owned = unsafe { take_raw(&mut *s as *mut String) };
    println!("{owned}");
}
```

(✓ prints `hi`.) Clippy can enforce the habit ✓: `-W clippy::missing_safety_doc` reports `unsafe function's docs are missing a `# Safety` section`, and `-W clippy::undocumented_unsafe_blocks` reports `unsafe block missing a safety comment`. The `rawlab` crate turns the second one into an error with `#![warn(clippy::undocumented_unsafe_blocks)]` plus `-D warnings`.

### 4.2 Make the unsafe part small, and put a privacy wall around it

The recipe for a sound abstraction (this is exactly where [Lesson 33](33_modules_api_design_rustdoc.md) pays off):

1. **State the invariant** in one sentence ("`ptr` points to the next item and `remaining` items are valid, initialized and exclusively borrowed for `'a`").
2. **Keep the invariant in private fields.** If a safe caller could change `remaining`, they could break the safety argument of the unsafe block that trusts it. Privacy is the soundness boundary, so the module that contains the `unsafe` code and its invariants should be as small as you can make it.
3. **Use the smallest unsafe block** with one operation per block and one `SAFETY` comment per block.
4. **Re-check at the boundary**: validate lengths, ranges and offsets in safe code before the unsafe call (`copy_within_checked` checks every range with `checked_add` first).
5. **Tie lifetimes to real owners**. A function returning `&'a [T]` built from a raw pointer with an *unconstrained* `'a` compiles for any `'a` and is almost always unsound ✓:

   ```rust
   fn make_static<'a>(p: *const u8, n: usize) -> &'a [u8] {
       unsafe { std::slice::from_raw_parts(p, n) }
   }
   fn main() { let v = vec![1u8, 2]; println!("{}", make_static(v.as_ptr(), 2).len()); }
   ```

   ✓ This compiles and runs (`2`): the compiler is not wrong, you told it `'a` could be anything. Derive the lifetime from a real borrow (`fn f(s: &[u8]) -> &[u8]`) instead.
6. **Avoid `unsafe` entirely where you can**: `#![forbid(unsafe_code)]` makes the compiler refuse it ✓ (`error: usage of an `unsafe` block`) and is a useful marker for crates that don't need it.

### 4.3 `unsafe impl Send` is a promise

A struct containing a raw pointer is neither `Send` nor `Sync` ✓:

```rust
// ERROR E0277
struct Handle(*mut u8);
fn needs_send<T: Send>(_: T) {}
fn main() { needs_send(Handle(std::ptr::null_mut())); }
```

```text
error[E0277]: `*mut u8` cannot be sent between threads safely
  = help: within `Handle`, the trait `Send` is not implemented for `*mut u8`
```

Adding `unsafe impl Send for Handle {}` compiles ✓ and silences the compiler, but it does **not** make the type thread-safe; it asserts that you checked it. The comment must say why moving this value to another thread is fine (for example "the pointer is never dereferenced" or "it points to heap data owned exclusively by this value"). `IterMutLite` in the practical challenge has such impls, justified by "it behaves like `&mut [T]`".

---

## 5. Finding UB: tools and their limits

| Tool | What it caught here | Limits |
|---|---|---|
| Debug-build checks in `core`/`std` (§2.1) ✓ | `get_unchecked`, `from_raw_parts`, `copy_nonoverlapping` misuse, invalid `char`, `unchecked_add` overflow, `zeroed::<&T>` | optional, debug only, covers only the listed functions; `assume_init` on a `bool` was **not** caught |
| `valgrind` (stable, no recompilation) ✓ | heap out-of-bounds read: `Invalid read of size 1`; use-after-free: `Invalid read of size 4 ... 0 bytes inside a block of size 12 free'd` | sees memory-access errors on the heap, not aliasing violations, stack overflows or type-validity UB; slow |
| AddressSanitizer ✓ (via `RUSTC_BOOTSTRAP=1 rustc -Zsanitizer=address`, unsupported on stable) | `heap-use-after-free ... READ of size 4`, and a clean run over `rawlab`'s 14 integration tests | same family as valgrind (plus stack/global overflows); needs nightly flags officially |
| Miri | the best tool for aliasing, validity, uninitialized memory and provenance, but **not available here** (Lesson 38 introduces it) | interprets the program: slow; only the paths you execute |
| Tests, fuzzing, `loom` | logic and (for `loom`) interleavings | none of them prove absence of UB |

No tool proves the absence of UB. Their job is to find bugs in the paths you run. The proof is the written argument in the `SAFETY` comments.

A subtle point about valgrind ✓: for the debugging challenge below, a normal run printed **garbage** that changed from run to run (`3828801774`, `3393995593`, `2251911085`), while under valgrind the program printed the *correct* value `60` (valgrind keeps freed blocks intact) **and** reported the invalid read. Correct-looking output under a checker is not evidence of correctness.

---

## 6. Performance, reliability, security

- **Performance.** `unsafe` is not faster by itself. Most of what people try to speed up (`get_unchecked` to skip bounds checks) is already optimized by LLVM in loops that iterate with iterators or `for i in 0..len`, and the check is cheap relative to a cache miss. Measure first (Lesson 41); an `unsafe` micro-optimization that saves 1% and creates one UB path is a bad trade. Real wins: avoiding allocation/copies (`from_raw_parts`, `copy`), interfacing with C, and data structures the borrow checker can't express (`split_at_mut`, intrusive lists). I did not benchmark anything in this lesson.
- **Reliability.** UB is the worst kind of bug: it may pass every test on your machine, then break under another optimization level, CPU, or compiler release. Keep unsafe code in few, small, well-commented, heavily tested modules with a fallback safe reference implementation to compare against (as `rawlab`'s tests do against `slice::split_at_mut_checked`, `iter_mut` and `copy_within`).
- **Security.** Memory-safety bugs (out-of-bounds, use-after-free) are the classic source of exploitable vulnerabilities, and in a Rust codebase they can only come from `unsafe` code, from FFI, or from a bug in the compiler or standard library. That is the benefit of the boundary: you audit the `unsafe` blocks instead of the whole program. It is not a guarantee: a `Vec<u8>` handed to a C parser can still be exploited, an unsound `unsafe impl Send` is exploitable, and Rust does not protect against logic bugs, injection, or malicious dependencies (Lesson 35 showed build-time code execution). Review unsafe code like cryptography: with a checklist, a second pair of eyes and tooling. Tools that list the `unsafe` usage in your dependency tree exist (`cargo geiger`, from memory; I did not run it).

---

## 7. Common mistakes

1. **Using `unsafe` to silence the borrow checker** instead of fixing the design (`E0499` ✓ below).
2. **Unconstrained lifetimes** from `from_raw_parts` (§4.2).
3. **Dereferencing after the owner is gone**: returning a pointer to a local, a `String` that is dropped, or a `Vec` that reallocates (the debugging challenge).
4. **Assuming alignment**: casting `*const u8` to `*const T` and dereferencing; use `read_unaligned` ✓.
5. **Forgetting that `ptr::read` duplicates ownership**: two `String`s that own the same buffer are a double free.
6. **`mem::transmute` between types of different layout** (the compiler stops sizes that differ: `E0512` ✓ `u32` to `u64`; it can't stop everything else).
7. **`MaybeUninit`/`mem::zeroed` misuse** for types with invalid zero patterns (references, `NonNull`, `NonZero`) ✓ `zeroed::<&u8>` aborts. (Lesson 37 treats `MaybeUninit`.)
8. **`static mut`** (denied by default for references in edition 2024 ✓); use atomics or locks.
9. **An `unsafe impl Send/Sync` without an argument**.
10. **Large `unsafe` blocks** that mix several operations, so no comment can justify each.
11. **A safe function that is secretly unsafe**: a `pub fn` whose soundness depends on its caller's arguments (should be `unsafe fn`).
12. **Trusting tests**: "it works in debug and release" is not a proof (§2).

The first one deserves its demonstration ✓: a safe attempt at splitting a slice into two `&mut` halves fails

```rust
// ERROR E0499
fn split_at_mut(v: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
    (&mut v[..mid], &mut v[mid..])
}
fn main() { let mut a = [1, 2, 3, 4]; let (l, r) = split_at_mut(&mut a, 2); l[0] = r[0]; println!("{a:?}"); }
```

with `error[E0499]: cannot borrow `*v` as mutable more than once at a time`. The borrow checker can't see that the halves are disjoint; the *reasoning* (`mid <= len` makes the ranges disjoint) is exactly an unsafe contract. The standard library solves it once, with `unsafe`, inside `split_at_mut`; `rawlab::split_at_mut_checked` shows how, and §9 implements it.

---

## 8. Exercises

Answers are collapsed. Try each before opening it.

1. **(Warm-up)** List the operations that need `unsafe` and say which one `let p = &x as *const i32;` is.
2. **(Warm-up)** What is the difference between `unsafe fn` and an `unsafe { }` block, in terms of who promises what?
3. **(Easy)** Is each of these UB? (a) reading `*p` where `p = ptr::null::<i32>()`; (b) `i32::MAX + 1` in release mode; (c) a `Vec` leaked with `mem::forget`; (d) two threads incrementing a `static mut` counter.
4. **(Medium)** Rewrite `static mut COUNTER: u32` and `COUNTER += 1` safely.
5. **(Medium)** Write `my_swap<T>(a: &mut T, b: &mut T)` using `ptr::read` and `ptr::copy_nonoverlapping`/`write`, with a `SAFETY` comment. Why is it fine for `String`?
6. **(Medium-hard)** `copy` or `copy_nonoverlapping` for shifting `[1,2,3,4,5]` one slot to the right inside the same array? What happens if you pick the wrong one?
7. **(Hard)** `struct Raw(*mut u8);` is moved to another thread. State two different invariants that would justify `unsafe impl Send for Raw {}`, and one that would not.
8. **(Hard)** Why is `bool` not `Plain` in `rawlab`, while `[u16; 4]` is?

<details markdown="1">
<summary>Answers</summary>

1. Dereference a raw pointer; call an `unsafe fn`; access or modify a `static mut`; implement an `unsafe trait`; access a union field (plus `unsafe extern` and unsafe attributes in edition 2024). `&x as *const i32` creates a raw pointer: **safe** (only the dereference is unsafe; ✓ the E0133 appeared at `*p`).
2. `unsafe fn` says "the **caller** must uphold the conditions in `# Safety`"; the function body then does the unsafe operations. An `unsafe { }` block says "**I**, the author, have checked that the preconditions of the operations in here hold", and should say why in a `SAFETY` comment. Similarly an `unsafe trait` is a promise by the implementor.
3. (a) UB (null deref). (b) Not UB: it wraps in release (or panics in debug), both defined. (c) Not UB: a leak is safe. (d) UB: a data race on a non-atomic.
4. ```rust
   use std::sync::atomic::{AtomicU32, Ordering};
   static COUNTER: AtomicU32 = AtomicU32::new(0);
   fn main() {
       COUNTER.fetch_add(1, Ordering::Relaxed);
       println!("{}", COUNTER.load(Ordering::Relaxed));
   }
   ```
   (✓ compiles and prints `1`). Use `Mutex` if you need more than one field to change together.
5. ```rust
   fn my_swap<T>(a: &mut T, b: &mut T) {
       // SAFETY: `a` and `b` are exclusive references, so both are valid,
       // aligned, initialized, and distinct (non-overlapping). We read `a`
       // into `tmp`, copy `b` over `a`, then write `tmp` over `b`: each value
       // is moved exactly once, so none is duplicated or dropped here.
       unsafe {
           let tmp = std::ptr::read(a);
           std::ptr::copy_nonoverlapping(b, a, 1);
           std::ptr::write(b, tmp);
       }
   }
   fn main() {
       let mut x = String::from("a");
       let mut y = String::from("b");
       my_swap(&mut x, &mut y);
       println!("{x}{y}");
   }
   ```
   (✓ prints `ba`.) Bitwise moving a `String` is fine because ownership is transferred, not duplicated: the old `a` bits are overwritten, not dropped. `std::mem::swap` does exactly this and you should use it.
6. `ptr::copy` (memmove): the ranges overlap. `copy_nonoverlapping` with overlapping ranges is UB; ✓ in a debug build the standard library's check aborts with `ptr::copy_nonoverlapping requires that both pointer arguments are aligned and non-null and the specified memory ranges do not overlap`; in release you get no guarantee at all (and `rawlab`'s mutation M11 shows the randomized test catching it only because of that debug check).
7. Justifying: (i) the pointer targets heap memory exclusively owned by `Raw` and freed in its `Drop`, so moving the owner moves exclusive access; (ii) it is a handle to a thread-safe resource (for example an atomic or a mutex-protected C object) whose API may be called from any thread. Not justifying: "I never use it from two threads at once" without enforcing it, or any pointer into data owned by another thread's stack frame, which can become dangling.
8. Every bit pattern of a `[u16; 4]` is a valid value and it has no padding, so `from_bytes` can build one from arbitrary bytes. `bool` is valid only for 0 and 1; `from_bytes::<bool>(&[3])` would create an invalid `bool`, which is UB. That is why `Plain` is a sealed `unsafe trait`: adding a type is a promise about validity that only the crate can make.

</details>

---

## 9. Debugging challenge

A helper builds a small buffer and hands back a raw pointer to it; `main` sums the three numbers. The author tests it and sees "reasonable" results in some runs.

```rust
fn make_buffer() -> *const u32 {
    let v = vec![10, 20, 30];
    v.as_ptr()
}

fn main() {
    let p = make_buffer();
    let total: u32 = (0..3).map(|i| unsafe { *p.add(i) }).sum();
    println!("total = {total}");
}
```

✓ Three consecutive runs printed `total = 3828801774`, `total = 3393995593`, `total = 2251911085`, and an optimized build (`-O`) printed `21883` and `22056`. The expected total is 60. Under `valgrind` it printed `total = 60` **and** reported `Invalid read of size 4 ... Address ... is 0 bytes inside a block of size 12 free'd`. AddressSanitizer reported `heap-use-after-free ... READ of size 4`. Explain, and fix.

<details markdown="1">
<summary>Solution</summary>

`v` is the owner of the heap buffer. When `make_buffer` returns, `v` is dropped and its buffer is **freed**; the returned pointer *dangles*. `main` then dereferences freed memory: UB (use after free). Raw pointers carry no lifetime, so the compiler had nothing to complain about. What the program printed is whatever the allocator left in that memory (the allocator typically writes its bookkeeping into the first bytes of a freed block, hence the huge, changing numbers; valgrind's allocator leaves the freed block untouched, so it printed 60 while still flagging the read, which is why "the output is right" proved nothing).

Fix: keep the owner alive for as long as the pointer is used, and let one type hold both, so the pointer can't outlive the data (✓ prints `total = 60` and valgrind is silent):

```rust
struct Buffer {
    data: Vec<u32>,
}

impl Buffer {
    fn new() -> Self {
        Buffer { data: vec![10, 20, 30] }
    }
    fn as_ptr(&self) -> *const u32 {
        self.data.as_ptr()
    }
}

fn main() {
    let buf = Buffer::new();
    let p = buf.as_ptr();
    // SAFETY: `buf` is alive until the end of `main`, owns 3 initialized u32
    // values, and is not mutated while `p` is in use.
    let total: u32 = (0..3).map(|i| unsafe { *p.add(i) }).sum();
    println!("total = {total}");
}
```

The better fix is to not use a raw pointer at all: return the `Vec` (or a `&[u32]` borrowed from something that outlives it) and iterate it safely; the borrow checker then rejects any attempt to use it after the owner is gone. Notice that even in the fixed version the `Vec` must not be pushed to while `p` is in use (reallocation would dangle `p` again): the `SAFETY` comment names that condition.

</details>

---

## 10. Interview questions

1. What does the `unsafe` keyword do and not do?
2. Name the unsafe operations.
3. What is undefined behavior, and why is it worse than a crash?
4. What is a sound abstraction? What makes `Vec<T>` sound even though it uses `unsafe` internally?
5. What is the difference between `ptr::copy` and `ptr::copy_nonoverlapping`?
6. What is pointer provenance?
7. When is `unsafe impl Send for T {}` correct?
8. How would you find UB in a project? Which tools and what are their limits?
9. Why is `ptr::read` on a `String` dangerous?
10. What is wrong with `fn f<'a>(p: *const u8, n: usize) -> &'a [u8]`?

<details markdown="1">
<summary>Answers</summary>

1. It unlocks operations the compiler can't verify (§1) and marks code where the programmer upholds the invariants. It does not disable the borrow checker or type checking, and does not make the code faster.
2. Raw-pointer dereference, `unsafe fn` calls, `static mut` access, `unsafe trait` impls, union field access (and `unsafe extern` / unsafe attributes in 2024).
3. UB means the program has no meaning; the optimizer assumes it never happens, so results can be arbitrary, inconsistent, and appear to work (✓ the dangling pointer printed three different totals; `assume_init` on a `bool` printed `false` silently). A crash at least tells you.
4. An API where safe callers can't cause UB. `Vec<T>` keeps `ptr`, `cap`, `len` private and maintains the invariants (`len <= cap`, first `len` elements initialized) in every method, so its internal unsafe blocks are justified for all safe uses. (The privacy wall is Lesson 33's visibility rules.)
5. `copy` handles overlapping ranges (memmove); `copy_nonoverlapping` requires disjoint ranges (memcpy) and is UB otherwise (✓ debug check aborts).
6. The extra information a pointer carries about which allocation (and permission) it may access, beyond its address. Equal addresses with different provenance are not interchangeable; use `with_addr`, `expose_provenance` and friends rather than ad-hoc integer round trips.
7. When the type is really safe to move to another thread: for example it exclusively owns its pointee and nothing else aliases it, or it wraps a thread-safe resource. The comment must state the argument. It is wrong if the pointee is shared and non-thread-safe or tied to a thread-local resource.
8. Debug-build checks and clippy lints (`undocumented_unsafe_blocks`), sanitizers/valgrind for memory errors, Miri for aliasing/validity/uninit (Lesson 38), tests against safe reference implementations, fuzzing. Each only covers executed paths; none proves absence of UB.
9. `ptr::read` copies the bits without invalidating the source, so you get two owners of the same heap buffer; when both are dropped, it is freed twice. You must make the source unused (`mem::forget`, `ManuallyDrop`, overwrite with `ptr::write`).
10. The lifetime `'a` is unconstrained, so the caller can pick `'static`; the compiler can't tie it to the data's real owner, so a safe caller can obtain a dangling slice (✓ it compiles). Derive the output lifetime from a borrowed input or make the function `unsafe` with a documented requirement.

</details>

---

## 11. Practical challenge: `rawlab`

Build a library crate `rawlab` of small safe functions implemented with `unsafe`. Requirements:

1. `split_at_mut_checked(&mut [T], mid) -> Option<(&mut [T], &mut [T])>`: `None` iff `mid > len`; correct for empty slices and zero-sized elements.
2. `IterMutLite<'a, T>`: a forward and double-ended `&mut T` iterator (`ExactSizeIterator`) built on raw pointers, correct for zero-sized types and element types with destructors; `Send`/`Sync` exactly when `T` is.
3. A sealed `unsafe trait Plain`, `bytes_of(&T) -> &[u8]`, `from_bytes::<T>(&[u8]) -> Option<T>` (exact length, any alignment), and `read_u32_le(bytes, offset)` with overflow-safe bounds checks.
4. `copy_within_checked(&mut [T: Copy], Range<usize>, dest) -> Result<(), RangeError>` handling overlap in both directions, with a `#[non_exhaustive]` error enum.
5. Every `unsafe` block has a `// SAFETY:` comment; every `unsafe fn`/`unsafe trait` has `# Safety`; `#![deny(unsafe_op_in_unsafe_fn)]`, `#![deny(missing_docs)]`, `clippy::undocumented_unsafe_blocks` enforced with `-D warnings`.
6. Tests: randomized comparisons with the standard library's safe equivalents, edge cases (empty, ZST, `Rc` elements, misaligned input, overflowing offsets), and a compile-fail doctest that the iterator is not `Send` for `Rc`.
7. Run the tests under a memory checker (valgrind or a sanitizer).

<details markdown="1">
<summary>Solution (spoilers: full source)</summary>

`Cargo.toml`:

```toml
[package]
name = "rawlab"
version = "0.1.0"
edition = "2024"

[dependencies]
```

`src/lib.rs`:

```rust
//! `rawlab`: small, safe APIs implemented with `unsafe`, written to show how
//! an unsafe contract is stated, checked and tested.
//!
//! * [`split_at_mut_checked`]: two disjoint `&mut` halves of one slice.
//! * [`IterMutLite`]: a `slice::IterMut` look-alike built on raw pointers.
//! * [`Plain`], [`bytes_of`], [`from_bytes`]: reinterpret plain-old-data.
//! * [`copy_within_checked`]: an overlap-safe `memmove` with bounds checks.
//!
//! Every `unsafe` block has a `// SAFETY:` comment saying why each
//! precondition holds, and every `unsafe fn` / `unsafe trait` has a
//! `# Safety` section. The crate was **not** run under Miri (not installed
//! in the environment where it was written); see the lesson for how it was
//! checked instead.
//!
//! ```
//! use rawlab::{bytes_of, copy_within_checked, split_at_mut_checked};
//! let mut a = [1, 2, 3, 4, 5];
//! let (l, r) = split_at_mut_checked(&mut a, 2).unwrap();
//! l[0] = r[0];
//! copy_within_checked(&mut a, 1..3, 3).unwrap();
//! assert_eq!(a, [3, 2, 3, 2, 3]);
//! assert_eq!(bytes_of(&0x0102_0304u32.to_be()), [1, 2, 3, 4]);
//! ```
//!
//! The iterator is not `Send` for element types that are not `Send`:
//!
//! ```compile_fail,E0277
//! fn needs_send<T: Send>(_: T) {}
//! let mut v = vec![std::rc::Rc::new(1)];
//! needs_send(rawlab::IterMutLite::new(&mut v));
//! ```

#![deny(missing_docs)]
#![deny(unsafe_op_in_unsafe_fn)]
#![warn(clippy::undocumented_unsafe_blocks)]

use core::fmt;
use core::marker::PhantomData;
use core::mem::size_of;
use core::ops::Range;
use core::ptr;
use core::slice;

/// Splits a slice into `[0, mid)` and `[mid, len)`, both mutable, or returns
/// `None` if `mid > len`.
///
/// Safe code cannot express this: two `&mut` into one slice are rejected by
/// the borrow checker (`E0499`), even though the halves are disjoint.
pub fn split_at_mut_checked<T>(s: &mut [T], mid: usize) -> Option<(&mut [T], &mut [T])> {
    let len = s.len();
    if mid > len {
        return None;
    }
    let p = s.as_mut_ptr();
    // SAFETY: `mid <= len`, so `[p, p+mid)` and `[p+mid, p+len)` both lie
    // inside the allocation of `s` and do not overlap. `p` comes from a valid
    // `&mut [T]`, so it is non-null and aligned (also for empty halves). The
    // returned slices borrow from `s` (same lifetime by elision), so `s`
    // cannot be used while they are alive, and no other `&mut` to this memory
    // exists: the originating exclusive borrow is the one we split.
    unsafe {
        Some((
            slice::from_raw_parts_mut(p, mid),
            slice::from_raw_parts_mut(p.add(mid), len - mid),
        ))
    }
}

/// A mutable iterator over a slice, built on raw pointers (a simplified
/// `core::slice::IterMut`).
///
/// Invariant: `ptr` points to the next item and `remaining` items are valid,
/// initialized, exclusively borrowed for `'a` and not yet yielded. The
/// remaining count (not an end pointer) makes zero-sized types work.
pub struct IterMutLite<'a, T> {
    ptr: *mut T,
    remaining: usize,
    _marker: PhantomData<&'a mut [T]>,
}

impl<'a, T> IterMutLite<'a, T> {
    /// Creates an iterator over `s`.
    pub fn new(s: &'a mut [T]) -> Self {
        IterMutLite {
            ptr: s.as_mut_ptr(),
            remaining: s.len(),
            _marker: PhantomData,
        }
    }
}

impl<'a, T> Iterator for IterMutLite<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<&'a mut T> {
        if self.remaining == 0 {
            return None;
        }
        let item = self.ptr;
        self.remaining -= 1;
        // SAFETY: `remaining > 0` was true, so `item` points to a live element
        // of the original slice, and the one-past-the-end pointer is allowed
        // (for zero-sized `T`, `add` does not move the address).
        self.ptr = unsafe { self.ptr.add(1) };
        // SAFETY: each element is yielded at most once (the cursor moved past
        // it), the original slice was exclusively borrowed for `'a`, so
        // handing out one `&mut` per element cannot alias.
        Some(unsafe { &mut *item })
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.remaining, Some(self.remaining))
    }
}

impl<T> DoubleEndedIterator for IterMutLite<'_, T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        if self.remaining == 0 {
            return None;
        }
        self.remaining -= 1;
        // SAFETY: after the decrement, `ptr + remaining` is the last
        // not-yet-yielded element, which is inside the original slice.
        let item = unsafe { self.ptr.add(self.remaining) };
        // SAFETY: that element is yielded exactly once (it is excluded from
        // the remaining range now), and the slice is exclusively borrowed.
        Some(unsafe { &mut *item })
    }
}

impl<T> ExactSizeIterator for IterMutLite<'_, T> {}

// SAFETY: `IterMutLite<'a, T>` behaves like `&'a mut [T]`, which is `Send`
// exactly when `T: Send`.
unsafe impl<T: Send> Send for IterMutLite<'_, T> {}
// SAFETY: like `&'a mut [T]`, sharing `&IterMutLite` exposes no way to reach
// the elements (no `&self` method touches them), so `T: Sync` is enough.
unsafe impl<T: Sync> Sync for IterMutLite<'_, T> {}

mod sealed {
    pub trait Sealed {}
}

/// Marker for "plain old data": types for which **every** bit pattern of
/// `size_of::<Self>()` bytes is a valid value, that contain no padding bytes,
/// no pointers or references, and no interior mutability.
///
/// The trait is sealed: only the types in this crate implement it.
///
/// # Safety
///
/// Implementors must be `Copy`, have no padding, and accept every bit pattern.
/// (`bool`, `char`, references, enums and structs with padding do **not**
/// qualify.)
pub unsafe trait Plain: Copy + 'static + sealed::Sealed {}

macro_rules! plain {
    ($($t:ty),* $(,)?) => {$(
        impl sealed::Sealed for $t {}
        // SAFETY: primitive numbers are `Copy`, have no padding, and every
        // bit pattern is a valid value (floats include NaNs).
        unsafe impl Plain for $t {}
    )*};
}
plain!(
    u8, u16, u32, u64, u128, usize, i8, i16, i32, i64, i128, isize, f32, f64
);

impl<T: Plain, const N: usize> sealed::Sealed for [T; N] {}
// SAFETY: an array of padding-free, any-bit-pattern elements has the same
// properties (arrays have no padding between elements).
unsafe impl<T: Plain, const N: usize> Plain for [T; N] {}

/// The bytes of a plain value, in native byte order.
pub fn bytes_of<T: Plain>(v: &T) -> &[u8] {
    // SAFETY: `v` is valid for `size_of::<T>()` bytes; `u8` has alignment 1;
    // `T: Plain` means all those bytes are initialized (no padding); the
    // result borrows `v` for the same lifetime and is read-only.
    unsafe { slice::from_raw_parts((v as *const T).cast::<u8>(), size_of::<T>()) }
}

/// Reinterprets exactly `size_of::<T>()` bytes as a `T` (alignment of the
/// input does not matter). Returns `None` for any other length.
pub fn from_bytes<T: Plain>(bytes: &[u8]) -> Option<T> {
    if bytes.len() != size_of::<T>() {
        return None;
    }
    // SAFETY: the slice has exactly `size_of::<T>()` readable bytes;
    // `read_unaligned` has no alignment requirement; `T: Plain` makes every
    // bit pattern a valid `T`.
    Some(unsafe { ptr::read_unaligned(bytes.as_ptr().cast::<T>()) })
}

/// Reads a little-endian `u32` at `offset`, or `None` if it does not fit.
pub fn read_u32_le(bytes: &[u8], offset: usize) -> Option<u32> {
    let end = offset.checked_add(4)?;
    from_bytes::<u32>(bytes.get(offset..end)?).map(u32::from_le)
}

/// Why [`copy_within_checked`] refused.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[non_exhaustive]
pub enum RangeError {
    /// The source range is reversed (`start > end`).
    InvalidRange,
    /// The source or the destination does not fit in the slice.
    OutOfBounds,
}

impl fmt::Display for RangeError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            RangeError::InvalidRange => f.write_str("range start is greater than its end"),
            RangeError::OutOfBounds => f.write_str("range is out of bounds"),
        }
    }
}

impl std::error::Error for RangeError {}

/// Copies `src` to the position `dest` inside the same slice. The ranges may
/// overlap (like C's `memmove`).
pub fn copy_within_checked<T: Copy>(
    s: &mut [T],
    src: Range<usize>,
    dest: usize,
) -> Result<(), RangeError> {
    if src.start > src.end {
        return Err(RangeError::InvalidRange);
    }
    let count = src.end - src.start;
    let dest_end = dest.checked_add(count).ok_or(RangeError::OutOfBounds)?;
    if src.end > s.len() || dest_end > s.len() {
        return Err(RangeError::OutOfBounds);
    }
    let p = s.as_mut_ptr();
    // SAFETY: both `[src.start, src.end)` and `[dest, dest + count)` were
    // checked to lie inside `s`, so the reads and writes are in bounds and
    // aligned; `T: Copy` means duplicating the bits is a valid copy and no
    // destructor is skipped; `ptr::copy` explicitly allows overlapping ranges.
    unsafe { ptr::copy(p.add(src.start), p.add(dest), count) };
    Ok(())
}
```

`tests/api.rs`:

```rust
//! Tests through the public API, including randomized comparisons with the
//! standard library's safe implementations.

use rawlab::{
    IterMutLite, Plain, RangeError, bytes_of, copy_within_checked, from_bytes, read_u32_le,
    split_at_mut_checked,
};

/// A tiny deterministic generator, so the tests need no dependencies.
struct Lcg(u64);
impl Lcg {
    fn next(&mut self, bound: usize) -> usize {
        self.0 = self
            .0
            .wrapping_mul(6364136223846793005)
            .wrapping_add(1442695040888963407);
        ((self.0 >> 33) as usize) % bound
    }
}

#[test]
fn split_at_mut_matches_std_for_every_mid() {
    for len in 0..8usize {
        for mid in 0..=len + 2 {
            let mut a: Vec<usize> = (0..len).collect();
            let mut b = a.clone();
            let ours = split_at_mut_checked(&mut a, mid).map(|(l, r)| (l.to_vec(), r.to_vec()));
            let std = b
                .split_at_mut_checked(mid)
                .map(|(l, r)| (l.to_vec(), r.to_vec()));
            assert_eq!(ours, std, "len {len} mid {mid}");
        }
    }
}

#[test]
fn split_halves_are_independent_and_writable() {
    let mut a = [1, 2, 3, 4, 5];
    let (l, r) = split_at_mut_checked(&mut a, 2).unwrap();
    assert_eq!((l.len(), r.len()), (2, 3));
    l[1] = 20;
    r[0] = 30;
    std::mem::swap(&mut l[0], &mut r[2]);
    assert_eq!(a, [5, 20, 30, 4, 1]);
}

#[test]
fn split_works_on_empty_and_zero_sized_elements() {
    let mut e: [u8; 0] = [];
    let (l, r) = split_at_mut_checked(&mut e, 0).unwrap();
    assert!(l.is_empty() && r.is_empty());
    let mut z = [(); 5];
    let (l, r) = split_at_mut_checked(&mut z, 3).unwrap();
    assert_eq!((l.len(), r.len()), (3, 2));
    assert!(split_at_mut_checked(&mut z, 6).is_none());
}

#[test]
fn iter_mut_lite_matches_iter_mut() {
    let mut a = vec![1, 2, 3, 4, 5];
    for x in IterMutLite::new(&mut a) {
        *x *= 10;
    }
    assert_eq!(a, [10, 20, 30, 40, 50]);
    let got: Vec<_> = IterMutLite::new(&mut a).map(|x| *x).collect();
    assert_eq!(got, [10, 20, 30, 40, 50]);
    let rev: Vec<_> = IterMutLite::new(&mut a).rev().map(|x| *x).collect();
    assert_eq!(rev, [50, 40, 30, 20, 10]);
}

#[test]
fn iter_mut_lite_meets_in_the_middle_and_reports_len() {
    let mut a = [1, 2, 3, 4, 5];
    let mut it = IterMutLite::new(&mut a);
    assert_eq!(it.len(), 5);
    assert_eq!(it.next().map(|x| *x), Some(1));
    assert_eq!(it.next_back().map(|x| *x), Some(5));
    assert_eq!(it.size_hint(), (3, Some(3)));
    assert_eq!(it.next().map(|x| *x), Some(2));
    assert_eq!(it.next_back().map(|x| *x), Some(4));
    assert_eq!(it.next().map(|x| *x), Some(3));
    assert!(it.next().is_none() && it.next_back().is_none());
    assert_eq!(it.len(), 0);
}

#[test]
fn iter_mut_lite_handles_zero_sized_and_empty() {
    let mut z = [(); 7];
    assert_eq!(IterMutLite::new(&mut z).count(), 7);
    assert_eq!(IterMutLite::new(&mut z).rev().count(), 7);
    let mut e: [String; 0] = [];
    assert!(IterMutLite::new(&mut e).next().is_none());
}

#[test]
fn iter_mut_lite_works_with_drop_types_without_double_drop() {
    use std::rc::Rc;
    let a = Rc::new(1);
    let mut v = vec![a.clone(), a.clone(), a.clone()];
    for x in IterMutLite::new(&mut v) {
        *x = Rc::new(2);
    }
    assert_eq!(Rc::strong_count(&a), 1, "old values dropped exactly once");
    drop(v);
}

#[test]
fn iter_mut_lite_is_send_and_sync_for_send_sync_elements() {
    fn assert_send<T: Send>() {}
    fn assert_sync<T: Sync>() {}
    assert_send::<IterMutLite<'static, u8>>();
    assert_sync::<IterMutLite<'static, u8>>();
    let mut a = [1, 2, 3];
    let it = IterMutLite::new(&mut a);
    std::thread::scope(|s| {
        s.spawn(move || {
            for x in it {
                *x += 1;
            }
        });
    });
    assert_eq!(a, [2, 3, 4]);
}

#[test]
fn bytes_of_and_from_bytes_round_trip() {
    assert_eq!(bytes_of(&0x0102_0304u32.to_be()), [1, 2, 3, 4]);
    assert_eq!(bytes_of(&[1u16, 2]).len(), 4);
    assert_eq!(bytes_of(&1.0f32), 1.0f32.to_ne_bytes());
    assert_eq!(
        from_bytes::<u32>(&0xdead_beefu32.to_ne_bytes()),
        Some(0xdead_beef)
    );
    assert_eq!(from_bytes::<[u8; 3]>(&[7, 8, 9]), Some([7, 8, 9]));
    assert_eq!(from_bytes::<f64>(&2.5f64.to_ne_bytes()), Some(2.5));
    assert_eq!(from_bytes::<u16>(&[1, 2, 3]), None, "too long");
    assert_eq!(from_bytes::<u16>(&[1]), None, "too short");
    assert_eq!(from_bytes::<u8>(&[]), None);
}

#[test]
fn from_bytes_ignores_alignment() {
    let buf = [0u8, 1, 0, 0, 0, 2, 0, 0, 0, 9];
    for off in 0..4 {
        let v = from_bytes::<u32>(&buf[off..off + 4]).unwrap();
        assert_eq!(v, u32::from_ne_bytes(buf[off..off + 4].try_into().unwrap()));
    }
}

#[test]
fn read_u32_le_checks_bounds_and_overflow() {
    let b = [1, 0, 0, 0, 2, 0, 0, 0];
    assert_eq!(read_u32_le(&b, 0), Some(1));
    assert_eq!(read_u32_le(&b, 4), Some(2));
    assert_eq!(read_u32_le(&b, 1), Some(0x0200_0000));
    assert_eq!(read_u32_le(&b, 5), None);
    assert_eq!(read_u32_le(&b, 8), None);
    assert_eq!(read_u32_le(&b, usize::MAX), None, "offset + 4 overflows");
    assert_eq!(read_u32_le(&[], 0), None);
    let le = 0x0403_0201u32;
    assert_eq!(read_u32_le(&[1, 2, 3, 4], 0), Some(le));
}

#[test]
fn copy_within_matches_std_including_overlaps() {
    let mut rng = Lcg(7);
    for _ in 0..2000 {
        let len = rng.next(12);
        let mut a: Vec<u32> = (0..len as u32).collect();
        let mut b = a.clone();
        let start = rng.next(len + 1);
        let end = start + rng.next(len + 1 - start);
        let dest = rng.next(len + 2);
        let ours = copy_within_checked(&mut a, start..end, dest);
        if dest + (end - start) <= len {
            b.copy_within(start..end, dest);
            assert_eq!(ours, Ok(()));
            assert_eq!(a, b, "len {len} src {start}..{end} dest {dest}");
        } else {
            assert_eq!(ours, Err(RangeError::OutOfBounds));
            assert_eq!(a, (0..len as u32).collect::<Vec<_>>(), "unchanged on error");
        }
    }
}

#[test]
fn copy_within_overlap_directions_and_errors() {
    let mut a = [1, 2, 3, 4, 5];
    copy_within_checked(&mut a, 0..4, 1).unwrap();
    assert_eq!(a, [1, 1, 2, 3, 4], "overlapping, moving right");
    copy_within_checked(&mut a, 1..5, 0).unwrap();
    assert_eq!(a, [1, 2, 3, 4, 4], "overlapping, moving left");
    let (hi, lo) = (3, 2);
    assert_eq!(
        copy_within_checked(&mut a, hi..lo, 0),
        Err(RangeError::InvalidRange)
    );
    assert_eq!(
        copy_within_checked(&mut a, 0..6, 0),
        Err(RangeError::OutOfBounds)
    );
    assert_eq!(
        copy_within_checked(&mut a, 0..2, usize::MAX),
        Err(RangeError::OutOfBounds),
        "dest + count overflows"
    );
    assert_eq!(
        copy_within_checked(&mut a, 5..5, 5),
        Ok(()),
        "empty at the end"
    );
    assert_eq!(
        RangeError::OutOfBounds.to_string(),
        "range is out of bounds"
    );
}

#[test]
fn plain_is_implemented_for_numbers_and_arrays_only() {
    fn is_plain<T: Plain>() {}
    is_plain::<u8>();
    is_plain::<i128>();
    is_plain::<f64>();
    is_plain::<[u16; 4]>();
    is_plain::<[[u8; 2]; 2]>();
}
```

**Verification ✓.** `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, `cargo test --locked` and `RUSTDOCFLAGS="-D warnings" cargo doc --no-deps` pass: 14 integration tests and 2 doctests (one `compile_fail,E0277`). I also ran the integration-test binary under `valgrind -q` (no error reports) and the 14 tests under AddressSanitizer (all passed). **Not run:** Miri. Those tools do not detect aliasing-model violations or every validity issue, so the `SAFETY` comments are an argument that has been reviewed only by me.

**Mutation checks ✓** (each mutation applied to a copy):

| # | Mutation | Result |
|---|---|---|
| M1 | `split_at_mut_checked` accepts `mid == len + 1` | **caught** by 2 tests |
| M2 | second half is one element too long | **caught** by 2 tests (the mutant creates an out-of-bounds slice) |
| M3 | first half overlaps the second by one element | **caught** by 3 tests |
| M4 | `next` decrement rewritten as `if remaining > 1 { -= 1 } else { = 0 }` | **equivalent mutant** (same value for every `remaining >= 1`); not caught, as expected |
| M5 | `next_back` returns the wrong element (`remaining - 1` saturating) | **caught** by 2 tests |
| M6 | `size_hint` has no upper bound | **caught** by the `ExactSizeIterator` check in `iter_mut_lite_meets_in_the_middle_and_reports_len` |
| M7 | `from_bytes` accepts longer input (`<` instead of `!=`) | **caught** |
| M8 | `from_bytes` accepts shorter input (`>` instead of `!=`) | **caught** (and the mutant reads past the slice: undefined behavior; the test failed) |
| M9 | `read_u32_le` uses `wrapping_add` for the end offset | **equivalent mutant**: a wrapped `end` is always smaller than `offset`, and `slice::get(offset..end)` already returns `None` for a reversed range, so the explicit `checked_add` is redundant defense in depth |
| M10 | `read_u32_le` decodes big-endian | **caught** |
| M11 | `ptr::copy_nonoverlapping` instead of `ptr::copy` | **caught**, but only by the standard library's debug-build precondition check (`ptr::copy_nonoverlapping requires ... do not overlap`), not by a value assertion; in a release build the result would be unspecified |
| M12 | destination bound check removed | **caught** (the run failed; the mutant writes outside the slice, so it is UB and I did not examine how the failure presented) |
| M13 | `dest + count` computed with `wrapping_add` | **caught** by `copy_within_overlap_directions_and_errors` |
| M14 | `bytes_of` returns one byte too few | **caught** |

M1 to M14 total: 12 caught, 2 equivalent. Two lessons from this table: the debug-build UB checks are a genuine second line of defense (M11), but a mutant that is UB is also one where "the test failed" is weaker evidence than it looks (M8, M12), and the test suite can only exercise paths, not prove the unsafe argument.

**Design notes.**

- `split_at_mut_checked` and `copy_within_checked` validate everything in safe code first (`mid <= len`, `checked_add`) so the unsafe block's preconditions follow from the checks directly above it.
- `IterMutLite` stores `ptr` and `remaining`, not an end pointer, so zero-sized types need no special case (their `add` does not change the address); it uses `PhantomData<&'a mut [T]>` to be invariant and to tie the iterator to the exclusive borrow, and implements `Send`/`Sync` by hand with comments (raw pointers would otherwise make it neither).
- `Plain` is sealed (Lesson 33) and is an `unsafe trait`, so the validity promise can only be made inside this crate. Padding bytes are the reason structs are excluded: reading padding as `u8` would read uninitialized memory.
- `from_bytes` uses `read_unaligned`, so alignment of the input slice is irrelevant (tested at offsets 0 to 3).
- Not handled: `bytes_of_mut`/`from_bytes_mut`, endianness-generic helpers, `Plain` for user types, `NonNull`-based storage (the iterator uses `*mut T`), and I did not fuzz or run Miri. The tests do not cover concurrent use beyond one scoped-thread test.

</details>

---

## 12. Key takeaways

- `unsafe` unlocks five operations the compiler can't check; it does not switch the borrow checker off. Raw pointers are cheap to create and dangerous to use (✓ `E0133`).
- UB is not a crash. The compiler assumes it never happens. Passing tests and a plausible output prove nothing (✓ garbage totals; ✓ `false` for an uninitialized `bool`).
- Soundness: safe callers must never be able to trigger UB. Document obligations with `# Safety`, discharge them with a `// SAFETY:` argument per block, and put the invariants in **private** fields (Lesson 33).
- Keep unsafe blocks small; validate in safe code first; derive lifetimes from real borrows (✓ `from_raw_parts` accepts any `'a`).
- Know the rules of the primitives: alignment (✓ `read_unaligned`), `copy` vs `copy_nonoverlapping` (✓ debug abort), in-bounds pointer arithmetic, provenance (✓ `with_addr`, `expose_provenance`), `ptr::read` duplicating ownership.
- Use every tool, trust none of them: debug checks, valgrind (✓ invalid reads), sanitizers (✓), Miri (not run here), randomized tests against safe references (✓).
- `static mut` references are denied in edition 2024 (✓); use atomics or locks. `unsafe impl Send` is a promise, not a fix.

**Connections.** [Lesson 33](33_modules_api_design_rustdoc.md)'s privacy and sealed traits are the soundness boundary for `rawlab`; [Lesson 28](28_atomics_memory_ordering.md) defines data races and the atomics that replace `static mut`; [Lesson 22](22_gats_send_sync_phantom.md) explains `Send`/`Sync` and `PhantomData`, which `IterMutLite` relies on; [Lesson 34](34_declarative_macros.md)'s `macro_rules!` generates the `Plain` impls (and why each impl has its own `SAFETY` comment). Next: Lesson 37 covers `MaybeUninit`, FFI and `repr(C)`.

---

## 13. References

- The Rustonomicon (the guide to unsafe Rust: aliasing, lifetimes, `Send`/`Sync`, uninitialized memory, `Vec` implementation): <https://doc.rust-lang.org/nomicon/>
- The Rust Reference, "Behavior considered undefined", "Unsafety" and the pointer types: <https://doc.rust-lang.org/reference/behavior-considered-undefined.html>
- The Rust Book, chapter 20.1 "Unsafe Rust" (chapter numbering may differ between editions): <https://doc.rust-lang.org/book/>
- `std::ptr` module documentation (safety requirements of `read`, `write`, `copy`, `NonNull`, the strict-provenance APIs): <https://doc.rust-lang.org/std/ptr/index.html>
- The Edition Guide, "Rust 2024": `unsafe_op_in_unsafe_fn`, `static_mut_refs`, `unsafe extern`: <https://doc.rust-lang.org/edition-guide/rust-2024/>
- Rust API Guidelines, "Future proofing" and unsafe-related items; Clippy lint documentation for `missing_safety_doc` and `undocumented_unsafe_blocks`: <https://rust-lang.github.io/rust-clippy/master/>
- Miri (the UB detector, Lesson 38): <https://github.com/rust-lang/miri>
- Stacked Borrows and Tree Borrows papers/notes (the proposed aliasing models): see the Miri repository's documentation.

The URLs above are the standard documentation locations; I did not open them during this session (the sandbox cannot reach the documentation sites), so if a link has moved, search for the page title. Unchecked or from memory: the full list of UB in §2 and its wording; that the standard-library precondition checks follow `debug_assertions` of the user's crate and are absent in release builds; the exact rules of `add`/`offset` (same allocation, one past the end) and of `slice::from_raw_parts`; the provenance model and the claim that Tree Borrows tracks permissions; Stacked/Tree Borrows as the models Miri implements; that `unsafe extern` and `#[unsafe(no_mangle)]` are edition-2024 items; `NonNull` covariance; `cargo geiger`; the Rust Book chapter number; and the whole of Miri's behavior (not installed, not run). Exercise answers 3, 6 and 7 combine ✓-run facts with reasoning from memory. Everything marked ✓ was run on this machine's `rustc 1.97.0`, with `valgrind` as installed in the sandbox, and (for the sanitizer lines) `RUSTC_BOOTSTRAP=1 -Zsanitizer=address`.
