---
layout: default
title: "Lesson 08 — Stack, heap, Drop, and RAII"
---

# Lesson 08 — Stack and heap, `Drop`, RAII, allocation and deallocation

**Module 3 · Ownership and memory** · Status: generated · Edition used in examples: **2024** · External dependencies: none

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux (main-thread stack limit `ulimit -s` = 8192 KiB on the machine used). As before, ✓ marks outputs and diagnostics I actually ran; unmarked statements are documented behavior I did not run. Sizes are for a **64-bit** target.

Builds on [Lesson 07](07_ownership_and_moves.md) (moves, drop order previewed) and [Lesson 04](04_variables_and_types.md) (`let _` versus `let _x`).

---

## 1. Two kinds of memory

| | Stack | Heap |
|---|---|---|
| Layout | One contiguous region per thread, used last-in first-out; one **frame** per active function call | A pool managed by an **allocator**; blocks of any size, allocated and freed in any order |
| Allocation | Moving a pointer (done at function entry for the whole frame) | A call into the allocator, which must find a free block |
| Size must be known | Yes, at compile time (per frame) | No: decided at run time |
| Lifetime | Until the function returns | Until freed |
| Typical Rust types | integers, `char`, `bool`, tuples, arrays, `struct`s, the *handle* part of `String`/`Vec`/`Box` | the data owned by `String`, `Vec`, `Box`, `Rc`, `HashMap` |

Rust does not hide this: **a value lives on the stack unless a type puts it on the heap**. `Box<T>`, `Vec<T>`, `String`, `Rc<T>` and friends each hold a small fixed-size handle (a pointer, maybe a length and capacity) on the stack that points to heap data.

```rust
fn inner() -> usize { let y = 2u64; &y as *const u64 as usize }
fn main() {
    let x = 1u64;
    let boxed = Box::new(3u64);
    let v = vec![4u64; 4];
    let xa = &x as *const u64 as usize;
    let ya = inner();
    let ba = &*boxed as *const u64 as usize;
    let va = v.as_ptr() as usize;
    println!("callee local below caller local: {}", ya < xa);
    println!("box is far from the stack local: {}", if ba > xa { ba - xa } else { xa - ba } > 1_000_000);
    println!("vec buffer is far from the stack local: {}", if va > xa { va - xa } else { xa - va } > 1_000_000);
    println!("sizes: Box {} Vec {} [u64;4] {}", std::mem::size_of::<Box<u64>>(), std::mem::size_of::<Vec<u64>>(), std::mem::size_of::<[u64; 4]>());
    println!("{}", *boxed + v[0]);
}
```

✓ output (the exact addresses vary per run because of address-space randomization, so the program prints comparisons, not addresses):

```text
callee local below caller local: true
box is far from the stack local: true
vec buffer is far from the stack local: true
sizes: Box 8 Vec 24 [u64;4] 32
7
```

On this platform the callee's frame sits at a lower address than the caller's (the stack grows downward on x86-64), and the heap blocks are over a megabyte away from the stack variables. Neither fact is guaranteed by the Rust language; they are observations about this machine and OS.

**Why the distinction matters in practice.** Stack allocation is essentially free, and stack data has the best locality; heap allocation costs a call into the allocator (and a later free), may fragment memory, and goes through a pointer. But the stack is small and fixed (Section 7), and its data can't outlive its frame or change size. Good Rust uses the stack by default and the heap when size is dynamic, lifetime must extend beyond a scope, or the value is too large for a frame.

**The allocator.** By default Rust programs use the system allocator through `std::alloc::System` (on Linux, typically the C library's `malloc`/`free`); a different one can be selected with `#[global_allocator]` (Section 6.1), for example to count allocations, as done below. Which allocator is best (jemalloc, mimalloc, glibc) depends on the workload; measure (Lessons 41–42).

---

## 2. `Box<T>`: the simplest heap pointer

`Box<T>` owns one heap-allocated `T`. It is a single pointer (8 bytes ✓), and dropping the box drops the `T` and frees the memory.

```rust
fn main() {
    let b = Box::new(41);
    let c = *b + 1;                 // * dereferences
    println!("{c}");
    let mut s = Box::new(String::from("hi"));
    s.push('!');                    // auto-deref: methods of String work directly
    println!("{s} {}", s.len());
    let unsized_: Box<[i32]> = vec![1, 2, 3].into_boxed_slice();
    println!("{:?} {}", unsized_, std::mem::size_of::<Box<[i32]>>());
}
```

✓ output: `42`, `hi! 3`, `[1, 2, 3] 16`. Note `Box<[i32]>` is **16** bytes: a pointer plus a length (a fat pointer), while `Box<i32>` is 8.

When do you need `Box`?

1. **Recursive types.** A type can't contain itself directly because its size would be infinite:

   ```rust
   enum List { Cons(i32, List), Nil }
   ```

   ✓ `error[E0072]: recursive type `List` has infinite size`. Break the recursion with a pointer (the size of a `Box` is fixed):

   ```rust
   #[derive(Debug)]
   enum List { Cons(i32, Box<List>), Nil }
   use List::{Cons, Nil};

   fn sum(l: &List) -> i32 { match l { Cons(v, rest) => v + sum(rest), Nil => 0 } }

   fn main() {
       let l = Cons(1, Box::new(Cons(2, Box::new(Cons(3, Box::new(Nil))))));
       println!("{} {}", sum(&l), std::mem::size_of::<List>());
   }
   ```

   ✓ prints `6 16`. (Lesson 13 covers enums, Lesson 23 `Box` in depth.)
2. **Trait objects** (`Box<dyn Trait>`), covered in Lesson 21.
3. **Moving large values cheaply**: a `Box<[u8; 4096]>` moves as 8 bytes.
4. **Exactly one owner, heap storage, no sharing.** For shared ownership use `Rc`/`Arc` (Lesson 23).

---

## 3. `Drop`: code that runs when a value dies

```rust
trait Drop {
    fn drop(&mut self);
}
```

The compiler calls `drop` automatically when an owner goes out of scope: at the *end of the scope*, not after the value's last use. After your `drop` body finishes, the compiler **drops each field** (and, for `Vec`/`Box`, the owned contents) and frees heap memory; you never free fields yourself.

### 3.1 The drop order rules (✓ verified)

```rust
struct Noisy(&'static str);
impl Drop for Noisy { fn drop(&mut self) { println!("drop {}", self.0); } }
struct Pair { first: Noisy, second: Noisy }

fn main() {
    let _a = Noisy("a");
    let _b = Noisy("b");
    let _p = Pair { first: Noisy("p.first"), second: Noisy("p.second") };
    let _t = (Noisy("t0"), Noisy("t1"));
    let _v = vec![Noisy("v0"), Noisy("v1")];
    let _ = Noisy("underscore");
    let x = Noisy("x1");
    let x = Noisy("x2");
    let _ = &x;
    drop(Noisy("explicit"));
    println!("end of main");
}
```

✓ output (rustc also warns that the first `x` is unused and that the fields of `Pair` are never read):

```text
drop underscore
drop explicit
end of main
drop x2
drop x1
drop v0
drop v1
drop t0
drop t1
drop p.first
drop p.second
drop b
drop a
```

| Situation | Rule |
|---|---|
| Local variables | Dropped in **reverse order of declaration** at the end of the scope (`x2`, `x1`, `_v`, `_t`, `_p`, `_b`, `_a`) |
| Struct fields | In **declaration order** (`p.first` then `p.second`) |
| Tuple elements, array/`Vec` elements | In **order** (`t0, t1`; `v0, v1`) |
| `let _ = expr;` | The value is dropped **immediately** (`underscore` came first) |
| `drop(x)` | `std::mem::drop` simply takes `x` by value and returns, so the parameter drops at once |
| Shadowing | Does **not** drop the old value; both `x1` and `x2` lived until the end of `main` |
| Moved-from variables | Not dropped at the original place; the new owner drops it |
| Temporaries | At the end of the enclosing statement (with exceptions, Section 3.3) |

The slightly counter-intuitive pair is **locals reversed, fields forward**. The reason locals are reversed is that later variables may borrow from earlier ones; destroying in reverse order keeps those borrows valid until the borrower is gone.

### 3.2 What you cannot do with `Drop`

```rust
struct N;
impl Drop for N { fn drop(&mut self) {} }
fn main() { let n = N; n.drop(); }
```

✓ `error[E0040]: explicit use of destructor method`. Calling `drop` yourself would let the compiler run it a second time (a double free). Use `std::mem::drop(n)` instead: it moves `n` into a function that does nothing, so the normal drop runs exactly once, now.

```rust
struct N(String);
impl Drop for N { fn drop(&mut self) {} }
fn main() { let n = N(String::new()); drop(n); println!("{}", n.0); }
```

✓ `error[E0382]: borrow of moved value: `n``. After an early `drop(n)` the value is gone like any other move.

```rust
struct N { s: String }
impl Drop for N { fn drop(&mut self) {} }
fn main() { let n = N { s: String::new() }; let s = n.s; println!("{s}"); }
```

✓ `error[E0509]: cannot move out of type `N`, which implements the `Drop` trait`. A type with a destructor can't be taken apart by moving a field out (the destructor would then see a half-moved value). Use `mem::take`/`replace` (Lesson 07) or restructure. And (Lesson 07) a `Drop` type can't be `Copy` (E0184).

### 3.3 Edition 2024 changed some temporary lifetimes

Edition 2024 changed two drop-scope rules. The same source (both examples compiled with `--edition 2021` and `--edition 2024`):

```rust
struct D(&'static str);
impl Drop for D { fn drop(&mut self) { println!("drop {}", self.0); } }
impl D { fn value(&self) -> i32 { 1 } fn opt(&self) -> Option<i32> { None } }

fn f() -> i32 {
    let _local = D("local");
    D("temp").value()           // a temporary in the tail expression
}

fn main() {
    f();
    println!("--");
    if let Some(_) = D("scrutinee").opt() { println!("then"); } else { println!("else"); }
}
```

| Edition | Output |
|---|---|
| 2021 ✓ | `drop local`, `drop temp`, `--`, `else`, `drop scrutinee` |
| 2024 ✓ | `drop temp`, `drop local`, `--`, `drop scrutinee`, `else` |

- **Tail-expression temporaries:** in 2024 they are dropped *before* the function's local variables (previously after).
- **`if let` scrutinee temporaries:** in 2024 they are dropped *before the `else` block* runs (previously they lived through it).

These matter for lock guards and `RefCell` borrows held in temporaries: code that relied on the old order can deadlock or panic differently, which is one reason editions are opt-in per crate (Lesson 02: `cargo fix --edition` automates part of a migration, but read the Edition Guide's description of these two changes and test code that holds guards or borrows in temporaries).

### 3.4 `Drop` and panics

When a panic unwinds the stack, the runtime runs destructors of the live locals in each frame:

```rust
use std::panic::{catch_unwind, AssertUnwindSafe};
struct Noisy(&'static str);
impl Drop for Noisy { fn drop(&mut self) { println!("drop {}", self.0); } }

fn main() {
    let r = catch_unwind(AssertUnwindSafe(|| {
        let _a = Noisy("a");
        let _b = Noisy("b");
        if std::env::args().count() < 5 { panic!("boom"); }
        println!("not reached");
    }));
    println!("caught: {}", r.is_err());
}
```

✓ output: the panic message on stderr, then `drop b`, `drop a`, `caught: true`. Cleanup ran in reverse order during unwinding. Two caveats:

- **A panic inside `drop` while already unwinding aborts the process.** ✓ Observed (exit status 134, SIGABRT): after the first panic, a `Drop` that panics produced `panic in a destructor during cleanup` / `thread caused non-unwinding panic. aborting.` So `Drop` implementations must not panic; if cleanup can fail, record or log the error (and offer an explicit `close()`/`commit()` method returning `Result` for callers who want to handle it).
- **`panic = "abort"` skips unwinding entirely** (Lesson 03 profile option): no destructors run on panic. Don't rely on `Drop` for correctness in programs built that way, and also not for `std::process::exit` or a crash.

---

## 4. RAII: tie a resource's lifetime to a value's

**RAII** ("Resource Acquisition Is Initialization", a name from C++) means: acquiring a resource creates an object; releasing it is that object's destructor. Rust's ownership makes the pattern enforceable: the compiler guarantees the owner is dropped exactly once on every exit path (normal return, early `return`, `?`, `break`, and panic unwinding).

Standard-library examples: `File` closes its descriptor; `MutexGuard` unlocks; `Box`/`Vec`/`String` free memory; `BufWriter` flushes (best effort); `JoinHandle` detaches; `Rc` decrements a count.

Locks make the RAII point concrete (Lesson 24 teaches `Mutex`; for now it is "a lock that gives you a guard object; the lock is held while the guard exists"):

```rust
#![allow(let_underscore_lock)]
use std::sync::Mutex;
fn main() {
    let m = Mutex::new(0);
    { let _ = m.lock().unwrap();
      println!("after `let _ =`, try_lock ok: {}", m.try_lock().is_ok()); }
    { let _guard = m.lock().unwrap();
      println!("after `let _guard`, try_lock ok: {}", m.try_lock().is_ok()); }
    println!("after scope, try_lock ok: {}", m.try_lock().is_ok());
}
```

✓ output:

```text
after `let _ =`, try_lock ok: true
after `let _guard`, try_lock ok: false
after scope, try_lock ok: true
```

`let _ =` dropped the guard at once, so the lock was **not** held (`try_lock` succeeds); `let _guard =` kept it until the scope ended. This is exactly the Lesson 04 warning, and it is now enforced by the compiler: ✓ without the `allow`, rustc 1.97.0 rejects `let _ = m.lock().unwrap();` with `error: non-binding let on a synchronization lock` (deny-by-default lint `let_underscore_lock`) and suggests `let _unused = ...`. (The `#![allow(...)]` above exists only to run the demonstration.)

You can release early with `drop(guard)` ✓ (`try_lock` fails while held and succeeds after `drop(guard)`), or by limiting the guard's scope with a block: shorter lock scopes mean less contention.

### 4.1 Writing your own guard

A **scope guard** runs a closure on exit unless cancelled. The version in the practical challenge (Section 12) is:

```rust
pub struct ScopeGuard<F: FnOnce()> { cleanup: Option<F> }

impl<F: FnOnce()> Drop for ScopeGuard<F> {
    fn drop(&mut self) {
        if let Some(f) = self.cleanup.take() { f(); }
    }
}
```

The `Option` plus `take()` is a standard idiom: `FnOnce` consumes the closure, but `drop` only has `&mut self`, so you move the closure out by replacing it with `None` (Lesson 07's `Option::take`). `dismiss(self)` sets it to `None` so nothing runs.

---

## 5. Leaking and not-dropping on purpose

```rust
use std::mem::ManuallyDrop;
struct Noisy(&'static str);
impl Drop for Noisy { fn drop(&mut self) { println!("drop {}", self.0); } }

fn main() {
    let leaked: &'static mut String = Box::leak(Box::new(String::from("lives forever")));
    leaked.push('!');
    println!("{leaked}");
    let m = ManuallyDrop::new(Noisy("manual"));
    println!("holding {}", m.0);
    let mut m2 = ManuallyDrop::new(Noisy("manual2"));
    unsafe { ManuallyDrop::drop(&mut m2); }
    println!("end");
}
```

✓ output: `lives forever!`, `holding manual`, `drop manual2`, `end` (never `drop manual`).

- `Box::leak` turns a box into a `&'static mut T`: the memory is never freed (useful for process-lifetime configuration; otherwise a leak).
- `ManuallyDrop<T>` suppresses the automatic drop; you call `ManuallyDrop::drop` yourself, which is `unsafe` because calling it twice, or using the value afterwards, is undefined behavior (Lesson 36).
- `mem::forget` (Lesson 07) is the safe cousin: leaking is safe.

---

## 6. Counting allocations: seeing the heap

You can observe the allocator by wrapping it. A global allocator is an `unsafe impl GlobalAlloc` (the `unsafe` is because the allocator's contract, such as returning properly aligned unique blocks, is something the compiler can't check; Lesson 36). This wrapper only counts and delegates to `System`:

### 6.1 The measurement program

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0);
static REALLOCS: AtomicUsize = AtomicUsize::new(0);
static FREES: AtomicUsize = AtomicUsize::new(0);

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        FREES.fetch_add(1, Relaxed);
        unsafe { System.dealloc(p, l) }
    }
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new_size: usize) -> *mut u8 {
        REALLOCS.fetch_add(1, Relaxed);
        unsafe { System.realloc(p, l, new_size) }
    }
}

#[global_allocator]
static A: Counting = Counting;

fn counts() -> (usize, usize, usize) {
    (ALLOCS.load(Relaxed), REALLOCS.load(Relaxed), FREES.load(Relaxed))
}

fn measure<R>(f: impl FnOnce() -> R) -> (R, (usize, usize, usize)) {
    let before = counts();
    let r = f();
    let after = counts();
    (r, (after.0 - before.0, after.1 - before.1, after.2 - before.2))
}

fn takes(s: String) -> usize { s.len() }

fn main() {
    println!("warm-up");   // first print may allocate stdout's buffer; do it before measuring
    let rows: Vec<(&str, (usize, usize, usize))> = vec![
        ("String::new()", measure(|| String::new()).1),
        ("String::from(\"hello\") (kept)", { let (s, c) = measure(|| String::from("hello")); std::mem::forget(s); c }),
        ("String::from(\"hello\") (dropped)", measure(|| { let _s = String::from("hello"); }).1),
        ("clone of a String (dropped)", { let a = String::from("hello"); measure(|| { let _b = a.clone(); }).1 }),
        ("move a String into a function", { let a = String::from("hello"); measure(|| takes(a)).1 }),
        ("Box::new(5u32) (dropped)", measure(|| { let _b = Box::new(5u32); }).1),
        ("Box::new(()) (dropped)", measure(|| { let _b = Box::new(()); }).1),
        ("Vec::new() + 100 pushes of u32 (dropped)", measure(|| { let mut v = Vec::new(); for i in 0..100u32 { v.push(i); } }).1),
        ("Vec::with_capacity(100) + 100 pushes (dropped)", measure(|| { let mut v = Vec::with_capacity(100); for i in 0..100u32 { v.push(i); } }).1),
        ("array [u32; 100] on the stack", measure(|| { let a = [7u32; 100]; std::hint::black_box(&a); }).1),
    ];
    for (label, (a, r, f)) in rows {
        println!("{label:<50} alloc {a} realloc {r} free {f}");
    }
}
```

✓ output:

```text
warm-up
String::new()                                      alloc 0 realloc 0 free 0
String::from("hello") (kept)                       alloc 1 realloc 0 free 0
String::from("hello") (dropped)                    alloc 1 realloc 0 free 1
clone of a String (dropped)                        alloc 1 realloc 0 free 1
move a String into a function                      alloc 0 realloc 0 free 1
Box::new(5u32) (dropped)                           alloc 1 realloc 0 free 1
Box::new(()) (dropped)                             alloc 0 realloc 0 free 0
Vec::new() + 100 pushes of u32 (dropped)           alloc 1 realloc 5 free 1
Vec::with_capacity(100) + 100 pushes (dropped)     alloc 1 realloc 0 free 1
array [u32; 100] on the stack                      alloc 0 realloc 0 free 0
```

These are **counts of allocator calls**, not timings or benchmarks; they depend on this standard-library version's growth policy. What they teach:

- `String::new()` and `Vec::new()` allocate nothing until you add data.
- Creating a `String` from a literal: one allocation. Dropping it: one free. `clone()` is one more allocation. Moving a `String` into a function: **zero** allocations (and the callee frees it, `free 1`).
- A zero-sized value in a `Box` (`Box::new(())`) needs no memory: zero allocations.
- Growing a `Vec` one push at a time: one allocation plus **5 reallocations** for 100 `u32` values (capacities 4, 8, 16, 32, 64, 128 in this version). `with_capacity(100)` does the same work with one allocation and no reallocations. This is why you pre-size when the size is known (Lesson 16).
- Stack arrays never touch the allocator.

(`Relaxed` ordering is enough for these independent counters; Lesson 28 explains atomics and ordering.)

---

## 7. The stack is small and fixed

Each thread's stack has a fixed maximum size. The main thread's is set by the OS (8 MiB here ✓), and threads created with `std::thread::spawn` get 2 MiB by default (the standard library documents this default; it can be changed with `thread::Builder::stack_size`). Exceeding it is a **stack overflow**, which aborts the process (not a catchable panic).

```rust
#[inline(never)]
fn on_stack() -> u8 { let a = [1u8; 16_000_000]; a[std::hint::black_box(5)] }
#[inline(never)]
fn on_heap() -> u8 { let v = vec![1u8; 16_000_000]; v[std::hint::black_box(5)] }

fn main() {
    let which = std::env::args().nth(1).unwrap_or_default();
    match which.as_str() {
        "stack" => println!("{}", on_stack()),
        "vec" => println!("{}", on_heap()),
        "thread" => {
            let h = std::thread::spawn(|| { let a = [1u8; 3_000_000]; a[std::hint::black_box(5)] });
            println!("thread ok: {:?}", h.join().is_ok());
        }
        _ => println!("usage"),
    }
}
```

✓ Results in a debug build: no argument → `usage`; `vec` → `1`; `stack` → `has overflowed its stack` / `fatal runtime error: stack overflow, aborting`; `thread` (a 3 MB array in a 2 MiB thread) → `has overflowed its stack`.

Three consequences worth knowing:

1. **Large values belong on the heap.** `vec![0u8; N]` allocates on the heap directly. `Box::new([1u8; 16_000_000])` can still overflow the stack because the array may be built on the stack first and then moved ✓ (debug build overflowed; the same program compiled with `-O` printed `1`, because the optimizer built it in place). Don't depend on that optimization; use `vec![..].into_boxed_slice()` for big boxes.
2. **A function's stack frame is reserved on entry for all its locals, even in branches not taken.** ✓ A `main` with a 16 MB array inside an `if` that never ran still overflowed immediately, in both debug and release builds. Put big locals in a separate function (or on the heap) so they only cost stack when used.
3. **Deep recursion** consumes one frame per call (Lesson 05).

---

## 8. Common mistakes and edge cases

1. Assuming a value is dropped after its **last use**. Rust drops at the **end of the scope** (a borrow's lifetime can end earlier; destructors don't). Use `drop(x)` or an inner block to release early.
2. `let _ = guard;` (drop now) versus `let _g = guard;` (drop at scope end), especially with locks (now a deny-by-default lint for locks).
3. Calling `x.drop()` (E0040) instead of `drop(x)`.
4. Panicking in `Drop`, especially when it can run during unwinding (abort).
5. Relying on destructors for correctness when the process may `exit`, abort, or be built with `panic = "abort"`; leaks are safe, so unsafe code can't depend on drops either.
6. Wrong field order when destruction order matters (Section 9's challenge).
7. Putting huge arrays in locals or building huge `Box::new([..])` values in debug builds.
8. Growing a `Vec`/`String` by repeated pushes when the final size is known (use `with_capacity` / `reserve`).
9. Holding a `MutexGuard` or `RefCell` borrow in a temporary whose lifetime changed with the edition (2024 tail-expression and `if let` rules).
10. Believing heap allocation is "slow" in absolute terms or that the stack is always faster: measure with real data (Lessons 41–42).

## 9. Performance implications

- Heap allocation costs a function call into the allocator plus bookkeeping, and a matching free. Avoiding unnecessary allocations (reusing buffers, `with_capacity`, borrowing instead of cloning, `SmallVec`-style crates for small sizes) is often the first optimization to try, but profile first (Lesson 41).
- Realloc may copy; `Vec` growth is amortized O(1) per push thanks to geometric growth (the factor is an implementation detail), but a pre-sized vector avoids every copy.
- Dropping a large collection runs a destructor for each element if the element type needs dropping (`Vec<String>`), but is free for `Vec<u32>` (no per-element work, just one free). `mem::needs_drop::<T>()` tells you which case applies.
- Dropping can be expensive and happen at surprising places (end of scope): a deeply nested `Box` list drops recursively and can overflow the stack on very long lists; very large structures are sometimes dropped on a background thread to avoid latency spikes.
- Stack frames are fixed-size; keep big locals out of hot or recursive functions.

## 10. Security and reliability

- RAII gives **deterministic cleanup** on every exit path, the main defense against leaked locks, file descriptors, connections, and temp files in long-running services. Make cleanup objects and use `?` freely.
- `Drop` is **not** a guarantee: leaks (`forget`, `Rc` cycles, `Box::leak`), `process::exit`, aborts, kills, and power loss skip it. Use explicit `commit()`/`close()` for operations that must succeed or be reported, and make state recoverable (atomic rename, journaling, idempotent retries) rather than relying on cleanup.
- Resource exhaustion is a security issue: unbounded allocation from untrusted lengths, leaked file descriptors, and unbounded caches are denial-of-service vectors. Cap sizes (Lesson 04), reuse buffers, and monitor memory.
- Secrets: freed memory is not cleared; moved-from stack slots still hold bytes. Securely wiping requires explicit measures (for example crates such as `zeroize`, which I did not verify in this lesson) and awareness that copies may exist elsewhere (Lesson 48).
- The allocator is a trust boundary: a custom `#[global_allocator]` is `unsafe` code that every allocation in the program depends on.

---

## 11. Exercises

Try each before opening the answer. They are not marked complete unless you tell me you did them.

1. **(Easy)** Predict the order in which this prints, then check it: `let _a = N("a"); let _b = N("b"); let _ = N("c"); let d = N("d"); drop(d); println!("end");` where `N` prints its name when dropped.
2. **(Easy)** Why is `size_of::<Box<[i32]>>()` 16 while `size_of::<Box<i32>>()` is 8?
3. **(Medium)** Write `struct Guard<'a>(&'a Cell<u32>)` whose `Drop` increments the cell, and show with a function `take(g: Guard)` that moving it into a function drops it exactly once.
4. **(Medium)** Hold a `MutexGuard`, show with `try_lock` that the mutex is locked, release it early with `drop`, and show it is free again.
5. **(Medium)** Predict `capacity()` for a `Vec<u64>` after 10 pushes from `Vec::new()` versus after 10 pushes from `Vec::with_capacity(10)`, then check. How many allocator calls does each make?
6. **(Hard)** The recursive `enum Tree { Leaf, Node(Tree, i32, Tree) }` does not compile. Fix it with `Box`, then write `fn depth(&self) -> usize` and `fn insert(self, v: i32) -> Tree` (an unbalanced binary search tree).
7. **(Hard)** What happens if a `Drop` impl panics while the thread is already unwinding from another panic? Write a program that shows it and explain what that means for how you write destructors.

<details markdown="1">
<summary>Answers</summary>

1. ✓ Output: `drop c` (immediately), `drop d` (explicit `drop`), `end`, `drop b`, `drop a`. (Analogous to the order table in Section 3.1.)
2. `Box<i32>` points at a sized value: one machine word. `[i32]` is a dynamically sized slice, so the box needs the length as well: a fat pointer of pointer + length. ✓ 8 vs 16 bytes.
3. ✓ Tested.
   ```rust
   use std::cell::Cell;
   struct Guard<'a>(&'a Cell<u32>);
   impl Drop for Guard<'_> { fn drop(&mut self) { self.0.set(self.0.get() + 1); } }
   fn take(_g: Guard<'_>) {}
   fn main() {
       let c = Cell::new(0);
       let g = Guard(&c);
       take(g);
       println!("drops after move: {}", c.get());
   }
   ```
   Prints `drops after move: 1`: the parameter `_g` is dropped when `take` returns; the original binding `g` was moved-from, so nothing drops there. (`Cell` is Lesson 24; here it is a counter that can be bumped through a shared reference.)
4. ✓ Tested: `let guard = m.lock().unwrap();` then `m.try_lock().is_ok()` prints `false`; after `drop(guard)` it prints `true`.
5. ✓ (capacities measured; call counts inferred) Both vectors have capacity ≥ 10: `Vec::new()` + 10 pushes of `u64` gave capacity **16** (it grew 4, 8, 16: one allocation and two reallocations, inferred from the growth pattern measured in Section 6.1, not separately counted), while `with_capacity(10)` gave exactly **10** with one allocation and no reallocations. The 4-8-16 growth is the observed behavior of this standard-library version for small element sizes, not a documented guarantee (the documented guarantee is only amortized O(1) pushes).
6. ✓ Tested (inserting 5, 3, 8, 1 gives depth 3).
   ```rust
   #[derive(Debug)]
   enum Tree { Leaf, Node(Box<Tree>, i32, Box<Tree>) }
   impl Tree {
       fn depth(&self) -> usize {
           match self { Tree::Leaf => 0, Tree::Node(l, _, r) => 1 + l.depth().max(r.depth()) }
       }
       fn insert(self, v: i32) -> Tree {
           match self {
               Tree::Leaf => Tree::Node(Box::new(Tree::Leaf), v, Box::new(Tree::Leaf)),
               Tree::Node(l, x, r) => {
                   if v < x { Tree::Node(Box::new(l.insert(v)), x, r) }
                   else { Tree::Node(l, x, Box::new(r.insert(v))) }
               }
           }
       }
   }
   ```
   `insert` takes `self` by value and moves the boxes into the new tree: no cloning. (Note `l.insert(v)` auto-dereferences the `Box<Tree>` by value; Lesson 23 explains why moving out of a `Box` is allowed, unlike moving out of other pointers.)
7. ✓ Observed: the first panic message prints, then the destructor's panic prints, then `panic in a destructor during cleanup` and `thread caused non-unwinding panic. aborting.`; the process exits with status 134 (SIGABRT). Destructors should therefore never panic: swallow or log errors, and provide an explicit fallible `close()` for callers who need to handle failure.

</details>

---

## 12. Debugging challenge

A service holds a cache that flushes into the database when it is destroyed. At shutdown the database is closed *first*, so the final flush fails. The two `Drop` implementations are fine and must not change; only the struct changes.

```rust
struct Noisy(&'static str);
impl Drop for Noisy { fn drop(&mut self) { println!("drop {}", self.0); } }

struct Server { database: Noisy, cache: Noisy }

fn main() {
    let _s = Server {
        cache: Noisy("cache (flushes into database)"),
        database: Noisy("database"),
    };
    println!("running");
}
```

✓ Current output: `running`, `drop database`, `drop cache (flushes into database)`. Explain why, fix it, and describe a more robust fix than "reorder the fields".

<details markdown="1">
<summary>Solution</summary>

**Why.** Struct fields are dropped in **declaration order**, regardless of the order used in the struct literal expression. `database` is declared first, so it is destroyed first (✓ the output). The literal's field order is irrelevant, which is a common source of confusion.

**Fix 1: reorder the declaration** so the dependent object (`cache`) comes before the thing it depends on (`database`):

```rust
struct Server { cache: Noisy, database: Noisy }
```

✓ Output now: `running`, `drop cache (flushes into database)`, `drop database`.

**A more robust fix.** Field order is an invisible invariant that a later refactoring (alphabetizing fields, an auto-formatter-like tool, a derive) can silently break. Make it explicit:

- Add a comment at the declaration stating the required order; and/or
- Implement `Drop for Server` and run the shutdown steps in order there, for example by holding the dependencies in `Option` fields and calling `.take()` in the required sequence (the same idiom as `ScopeGuard`); and/or
- Express the dependency with ownership: make `Cache` own (or borrow) the `Database`, so it cannot outlive it. Then the compiler enforces the order, and the field-order trick is unnecessary.

Also test the shutdown order: the `Tracer` pattern in the practical challenge records drop order into a log that a unit test can assert on.

</details>

---

## 13. Practical challenge: `droplab`

Create a library crate `droplab` (edition 2024) with:

1. `ScopeGuard<F: FnOnce()>`: runs the closure on drop unless `dismiss(self)` was called. It must run on normal exit, on early `return`, and during panic unwinding.
2. `Tracer<'a>`: records its name into a shared `&RefCell<Vec<&'static str>>` when dropped (`RefCell` is Lesson 24: "a `Vec` you can push to through a shared reference").
3. A `Pair<'a>` with two `Tracer` fields.
4. Tests that assert, using the log, each of: reverse declaration order for locals; field, tuple and `Vec` element order; `let _ =` drops now but `let _x =` drops at scope end; shadowing does not drop; a value moved into a function is dropped when the function returns; the guard runs once on normal exit and on early return, can be dismissed, and runs during a caught panic.
5. It must pass `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, and `cargo test --locked` (run `cargo generate-lockfile` once first).

<details markdown="1">
<summary>Solution</summary>

**`src/lib.rs`**

```rust
//! RAII helpers and drop-order experiments.

use std::cell::RefCell;

/// Runs a closure when dropped, unless dismissed. Useful for cleanup that
/// must happen on every exit path: early return, `?`, and panic unwinding.
pub struct ScopeGuard<F: FnOnce()> {
    cleanup: Option<F>,
}

impl<F: FnOnce()> ScopeGuard<F> {
    pub fn new(cleanup: F) -> Self {
        ScopeGuard {
            cleanup: Some(cleanup),
        }
    }

    /// Cancels the cleanup (for example after a successful commit).
    pub fn dismiss(mut self) {
        self.cleanup = None;
    }
}

impl<F: FnOnce()> Drop for ScopeGuard<F> {
    fn drop(&mut self) {
        if let Some(f) = self.cleanup.take() {
            f();
        }
    }
}

/// Records its name in a shared log when dropped.
pub struct Tracer<'a> {
    name: &'static str,
    log: &'a RefCell<Vec<&'static str>>,
}

impl<'a> Tracer<'a> {
    pub fn new(name: &'static str, log: &'a RefCell<Vec<&'static str>>) -> Self {
        Tracer { name, log }
    }
}

impl Drop for Tracer<'_> {
    fn drop(&mut self) {
        self.log.borrow_mut().push(self.name);
    }
}

/// A struct whose fields drop in declaration order.
pub struct Pair<'a> {
    pub first: Tracer<'a>,
    pub second: Tracer<'a>,
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::cell::Cell;
    use std::panic::{AssertUnwindSafe, catch_unwind};

    fn log() -> RefCell<Vec<&'static str>> {
        RefCell::new(Vec::new())
    }

    #[test]
    fn locals_drop_in_reverse_declaration_order() {
        let l = log();
        {
            let _a = Tracer::new("a", &l);
            let _b = Tracer::new("b", &l);
            let _c = Tracer::new("c", &l);
        }
        assert_eq!(*l.borrow(), ["c", "b", "a"]);
    }

    #[test]
    #[allow(clippy::useless_vec)] // the point is to test `Vec`'s drop order
    fn fields_tuples_and_vecs_drop_in_order() {
        let l = log();
        {
            let _p = Pair {
                first: Tracer::new("p.first", &l),
                second: Tracer::new("p.second", &l),
            };
        }
        assert_eq!(*l.borrow(), ["p.first", "p.second"]);

        l.borrow_mut().clear();
        {
            let _t = (Tracer::new("t0", &l), Tracer::new("t1", &l));
            let _v = vec![Tracer::new("v0", &l), Tracer::new("v1", &l)];
        }
        assert_eq!(*l.borrow(), ["v0", "v1", "t0", "t1"]);
    }

    #[test]
    fn underscore_drops_now_named_underscore_drops_later() {
        let l = log();
        {
            let _ = Tracer::new("now", &l);
            assert_eq!(*l.borrow(), ["now"]);
            let _later = Tracer::new("later", &l);
            assert_eq!(l.borrow().len(), 1);
        }
        assert_eq!(*l.borrow(), ["now", "later"]);
    }

    #[test]
    #[allow(unused_variables)] // the first `x` is shadowed on purpose
    fn shadowing_does_not_drop() {
        let l = log();
        {
            let x = Tracer::new("x1", &l);
            let x = Tracer::new("x2", &l);
            let _ = &x;
            assert!(l.borrow().is_empty());
        }
        assert_eq!(*l.borrow(), ["x2", "x1"]);
    }

    #[test]
    fn moved_into_function_drops_at_callee_end() {
        fn eat(_t: Tracer<'_>) {}
        let l = log();
        let t = Tracer::new("moved", &l);
        eat(t);
        assert_eq!(*l.borrow(), ["moved"]);
    }

    #[test]
    fn guard_runs_on_normal_exit_and_early_return() {
        let ran = Cell::new(0);
        {
            let _g = ScopeGuard::new(|| ran.set(ran.get() + 1));
        }
        assert_eq!(ran.get(), 1);

        fn early(ran: &Cell<u32>, bail: bool) -> u32 {
            let _g = ScopeGuard::new(|| ran.set(ran.get() + 1));
            if bail {
                return 0;
            }
            1
        }
        let ran2 = Cell::new(0);
        assert_eq!(early(&ran2, true), 0);
        assert_eq!(early(&ran2, false), 1);
        assert_eq!(ran2.get(), 2);
    }

    #[test]
    fn guard_can_be_dismissed() {
        let ran = Cell::new(0);
        let g = ScopeGuard::new(|| ran.set(1));
        g.dismiss();
        assert_eq!(ran.get(), 0);
    }

    #[test]
    fn guard_runs_during_panic_unwinding() {
        let ran = Cell::new(0);
        let result = catch_unwind(AssertUnwindSafe(|| {
            let _g = ScopeGuard::new(|| ran.set(1));
            panic!("boom");
        }));
        assert!(result.is_err());
        assert_eq!(ran.get(), 1);
    }
}
```

✓ With this exact file: `cargo fmt --check` clean, `cargo clippy --all-targets --locked -- -D warnings` clean, `cargo test --locked` → 8 passed.

**Design notes.**

- Clippy initially objected to two things in the *tests* (✓ I hit both): `useless_vec` for the `vec![...]` (the test exists to check `Vec`'s drop order, so I allowed it with a comment) and `unused_variables` for the shadowed first `x` (the point of the test; also allowed with a comment). When a lint is wrong *for a stated reason*, a narrowly scoped `#[allow]` with that reason beside it is the right tool.
- `ScopeGuard::dismiss(mut self)` sets the option to `None` and then drops `self` at the end of the method; the `Drop` impl sees `None` and does nothing. `Option<F>` is what lets `drop(&mut self)` move the closure out.
- `Tracer` borrows the log (`&'a RefCell<...>`), so it cannot outlive it; the compiler enforces that the log is created before, and dropped after, the tracers. Declaring `l` before the inner block in each test is why the assertions after the block can still read it.
- `catch_unwind(AssertUnwindSafe(...))` runs the panicking closure and lets the test continue; `AssertUnwindSafe` asserts that observing shared state after the panic is acceptable (it's a `Cell`, so it is). The default panic hook prints the panic message to stderr during the test; that is expected.

</details>

---

## 14. Interview questions

<details markdown="1">
<summary>Answers</summary>

1. **Where do values live in Rust?** On the stack by default (including the fixed-size handles of `Box`, `Vec`, `String`); heap memory is used only by types that allocate (`Box`, `Vec`, `String`, `Rc`, `HashMap`, ...).
2. **When do you need `Box`?** For recursive types, trait objects (`Box<dyn Trait>`), moving large values cheaply, and when you need an owned heap value with a single owner.
3. **What does `Drop` do and when is it called?** `Drop::drop(&mut self)` is run automatically when an owned value goes out of scope (or is dropped explicitly), before its fields are dropped and its memory freed.
4. **In what order are things dropped?** Locals: reverse declaration order. Struct fields, tuple elements, and collection elements: in order. `let _ = x` drops immediately. Temporaries: end of the enclosing statement (with edition-2024 changes for tail expressions and `if let`).
5. **Why can't you call `x.drop()`?** The compiler would still run the destructor at scope end: a double drop. Use `std::mem::drop(x)`.
6. **Is dropping at last use or at end of scope?** End of scope (for values with destructors). Borrows end at last use (non-lexical lifetimes), destructors do not.
7. **What is RAII?** Binding a resource's lifetime to an object's lifetime so acquisition happens at construction and release at destruction, guaranteed on every path including unwinding.
8. **Can `Drop` code panic?** It can, but should not: a panic while already unwinding aborts the process.
9. **Is leaking memory unsafe in Rust?** No (`mem::forget` and `Box::leak` are safe), so safe code can't rely on destructors running, and unsafe code must not either.
10. **Why is a stack overflow different from a panic?** The runtime can't unwind: it aborts the process. It comes from deep recursion or giant locals; heap-allocate large data and bound recursion depth.
11. **What changed in edition 2024 about drop order?** Temporaries in a block's tail expression drop before the locals, and temporaries in an `if let` scrutinee drop before the `else` branch.
12. **How can you observe how many allocations a code path makes?** Wrap `System` in a counting `#[global_allocator]` (an `unsafe impl GlobalAlloc`), or use a profiler; compare counts before/after.
13. **Why pre-size a `Vec`?** To avoid repeated reallocations and copies while it grows (✓ 1 allocation and 0 reallocations versus 1 and 5 for 100 `u32` pushes).
14. **What does `mem::needs_drop::<T>()` tell you?** Whether dropping a `T` involves any work (drop glue): useful for understanding the cost of dropping collections.

</details>

---

## 15. Key takeaways

- Stack: fixed-size frames, free to allocate, small, and per-thread. Heap: dynamic, managed by the allocator, accessed through pointers held by `Box`/`Vec`/`String`. Moving a handle never moves the heap data.
- `Box<T>` is a single owning pointer: use it for recursive types, trait objects, and big moves.
- `Drop` runs at **scope end**, exactly once per owned value, in a well-defined order: locals reversed, fields/elements forward, `let _ =` immediately, temporaries per statement (edition-2024 rules differ in two cases).
- RAII turns resource management into ownership: locks, files, and memory are released on every exit path, including unwinding. Destructors must not panic and cannot be relied on for correctness (leaks, `exit`, aborts, `panic = "abort"`).
- Count and reduce allocations when it matters: `with_capacity`, borrow instead of clone; zero-sized and empty containers don't allocate.
- The stack is limited (8 MiB main thread here, 2 MiB spawned threads by default): keep big locals out of frames and bound recursion.

**Connections.** Lesson 07's moves and drop-order previews are formalized here; the `let _`/`let _guard` pitfall from Lesson 04 now has a compiler lint. Next: Lesson 09 (memory layout: size, alignment, padding, niches, `repr`) explains why `Option<String>` is 24 bytes and why tuple field order is unspecified. Lesson 16 covers `Vec`/`String` growth and collection performance; Lesson 21 trait objects; Lesson 23 `Box`/`Rc`/`Arc`; Lesson 24 `Mutex` and `RefCell`; Lesson 28 atomics and orderings; Lesson 36 `unsafe` (including `GlobalAlloc` and `ManuallyDrop`); Lessons 41–42 measuring allocation and performance. See the [roadmap](00_roadmap.md) and the [course index](index.md).

---

## 16. References

- *The Rust Programming Language*, ch. 15.1 "Using `Box<T>` to Point to Data on the Heap": <https://doc.rust-lang.org/book/ch15-01-box.html>
- *The Rust Programming Language*, ch. 15.3 "Running Code on Cleanup with the `Drop` Trait": <https://doc.rust-lang.org/book/ch15-03-drop.html>
- *The Rust Programming Language*, ch. 4.1 "What Is Ownership?" (stack and heap): <https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html>
- *The Rust Reference*, destructors (drop scopes, order, temporaries): <https://doc.rust-lang.org/reference/destructors.html>
- *Rust Edition Guide*, Rust 2024: temporary scope in tail expressions and `if let`: <https://doc.rust-lang.org/edition-guide/rust-2024/index.html>
- Rust standard library documentation, `Drop`: <https://doc.rust-lang.org/std/ops/trait.Drop.html>
- Rust standard library documentation, `std::alloc` (`GlobalAlloc`, `System`, `Layout`): <https://doc.rust-lang.org/std/alloc/index.html>
- Rust standard library documentation, `ManuallyDrop` and `std::mem` (`forget`, `needs_drop`): <https://doc.rust-lang.org/std/mem/struct.ManuallyDrop.html>, <https://doc.rust-lang.org/std/mem/index.html>
- Rust standard library documentation, `Vec` capacity and growth: <https://doc.rust-lang.org/std/vec/struct.Vec.html#capacity-and-reallocation>
- Rust standard library documentation, `std::thread` (default stack size) and `thread::Builder::stack_size`: <https://doc.rust-lang.org/std/thread/index.html#stack-size>
- *The Rustonomicon*, "Destructors" and "Leaking": <https://doc.rust-lang.org/nomicon/destructors.html>, <https://doc.rust-lang.org/nomicon/leaking.html>

The URLs above are the standard documentation locations; I did not open every one during this session, so if a link has moved, search the docs site for the page title.
