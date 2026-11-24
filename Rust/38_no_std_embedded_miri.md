---
title: "Lesson 38 — no_std, embedded basics, and Miri"
---

# Lesson 38 — `no_std`, embedded basics, and Miri

**Module 12 · Unsafe Rust and low-level programming** · Status: generated · Edition used in examples: **2024** · External dependencies: none

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable` and `valgrind`, on x86-64 Linux. As before, ✓ marks outputs, diagnostics and measurements I actually ran; unmarked statements are documented behavior I did not run. Everything is **stable Rust**. Two limits you should know about up front:

- **I could not install any embedded target** (`rustup target add thumbv7em-none-eabihf` failed in this sandbox) and there is no emulator or board here. Nothing in this lesson was cross-compiled or run on a microcontroller. The `no_std` *language* behavior is real and was run on x86-64 Linux, including genuine freestanding executables that do not link `std` or a C library; the *hardware* parts (§7) are from memory.
- **Miri is not installed and I could not install it** (`rustup component add miri` failed again). §8 teaches Miri from documentation-level memory and says so; no Miri output appears in this lesson. What I used instead: the standard library's debug checks and `valgrind`.

I could not open the Rust documentation sites, so rules stated without ✓ are from memory; the closing paragraph lists them.

Builds on [Lesson 37](37_maybeuninit_ffi.md) (`MaybeUninit`, `repr(C)`, the `RingBuffer` below uses the same techniques), [Lesson 36](36_unsafe_raw_pointers.md) (unsafe contracts, volatile and raw pointers), [Lesson 28](28_atomics_memory_ordering.md) (atomics, the only shared state in `no_std` without locks), [Lesson 19](19_generics_and_bounds.md) (const generics for fixed capacity), [Lesson 26](26_error_design.md) (the error enum), and [Lesson 33](33_modules_api_design_rustdoc.md) (features and documentation). Lesson 39 (unit and integration tests, doc tests) is next; it appears here only as plain text.

---

## 1. Why this lesson exists

Rust is used on machines with 16 KB of RAM and no operating system, in kernels, bootloaders, WebAssembly modules, and smart-contract runtimes (Solana programs and Substrate runtimes compile to restricted targets with no ordinary OS, and the Substrate runtime is built `no_std`; from memory). All of them share one fact: the **standard library assumes an operating system** (files, threads, a heap, a clock, a process entry point). Rust splits the library so you can keep what does not need one.

Two ideas drive the lesson:

1. **Layering.** `core` (needs nothing), `alloc` (needs a heap), `std` (needs an OS). Writing `#![no_std]` library code means you can be used anywhere.
2. **Nothing is free.** Without the OS you supply the entry point, the panic behavior, maybe the allocator, and you talk to hardware through memory addresses that the compiler must not "optimize". That is where `unsafe` comes back (Lesson 36), and where **Miri** and other tools matter.

---

## 2. The three layers: `core`, `alloc`, `std`

| Layer | Needs | Contains (examples) | Does not contain |
|---|---|---|---|
| `core` | nothing | `Option`, `Result`, slices, `str`, iterators, `fmt`, `cell`, `sync::atomic`, `mem`, `ptr`, `MaybeUninit`, `time::Duration`, `error::Error`, `ops`, `future`/`task`, `hint` | heap types, files, threads, `Instant`, `println!`, `HashMap` |
| `alloc` | a global allocator | `Box`, `Vec`, `String`, `Rc`, `Arc`, `format!`, `vec!`, `BTreeMap`, `BTreeSet`, `VecDeque`, `BinaryHeap`, `borrow::Cow` | `HashMap` (needs a random seed from the OS), threads, I/O |
| `std` | an operating system | everything above (re-exported) plus `fs`, `net`, `thread`, `sync::Mutex`, `time::Instant`, `HashMap`, `env`, `process`, `println!` | |

`std` re-exports `core` and `alloc`, which is why `std::fmt::Display` and `core::fmt::Display` are the *same* trait. This has a practical consequence: an item used from `core` works in every environment, so library code that only needs `core` is the most reusable code you can write. Clippy has opt-in lints to enforce it (`std_instead_of_core`, `std_instead_of_alloc`, `alloc_instead_of_core`), which the practical crate turns on ✓. Clippy even caught me writing `impl std::error::Error`: the trait lives in **`core::error::Error`** (stable since Rust 1.81, from memory), so a `no_std` error type can implement it ✓ with no feature flag.

---

## 3. A `#![no_std]` library

Add `#![no_std]` at the crate root. The compiler then links `core` only, and `std` is no longer in the prelude. What breaks first ✓:

```rust
// ERROR E0425
#![no_std]
pub fn f() {
    let v: Vec<u8> = Vec::new(); // `Vec` is in `alloc`, not `core`
    let _ = v;
}
```

```text
error[E0425]: cannot find type `Vec` in this scope
error[E0433]: cannot find type `Vec` in this scope
```

and `println!("hi")` gives `error: cannot find macro `println` in this scope` ✓. To use heap types, opt in explicitly. A library that does this still needs **no allocator of its own**: the final binary chooses one ✓ (this compiled as a library):

```rust
#![no_std]
extern crate alloc;

use alloc::{boxed::Box, format, string::String, vec::Vec};

pub fn f() -> usize {
    let mut v: Vec<u8> = Vec::new();
    v.push(1);
    let s: String = format!("{}", 5);
    let b = Box::new(3);
    v.len() + s.len() + *b
}
```

`HashMap` is not in `alloc` ✓ (`use alloc::collections::HashMap;` gives `E0432: unresolved import`): its default hasher needs OS randomness. Use `BTreeMap`, or a third-party map with an explicit hasher.

### 3.1 The feature pattern

A library that wants to work both ways declares **additive** features (Lesson 33) and guards the code that needs them:

```toml
[features]
default = []
alloc = []
```

```rust,ignore
#![no_std]

#[cfg(feature = "alloc")]
extern crate alloc;

impl<T, const N: usize> RingBuffer<T, N> {
    #[cfg(feature = "alloc")]
    pub fn drain_to_vec(&mut self) -> alloc::vec::Vec<T> { /* ... */ }
}
```

Rules that keep this sane: features must only **add** items (never change the meaning of existing ones), the default build must stay `no_std`, and CI must build and test with and without each feature. A common alternative is `#![cfg_attr(not(feature = "std"), no_std)]`, which makes the crate use `std` when the feature is on; that is convenient but means the "no features" build is the only one proving `no_std` correctness. `nostdlab` (§14) uses an unconditional `#![no_std]` and tests both configurations ✓.

A caution about dependencies: **one** `std`-using dependency in a `no_std` build makes the whole build fail (or silently pull `std`). Check each dependency's `default-features` (many need `default-features = false`).

---

## 4. A `no_std` *binary*: what you must supply

A library only needs `#![no_std]`. A program also needs the things `std` normally hides. Try the minimum ✓:

```rust,ignore
#![no_std]
#![no_main]
```

```text
error: `#[panic_handler]` function required, but not found
error: unwinding panics are not supported without std
```

So:

1. **A panic handler**: `#[panic_handler] fn panic(info: &PanicInfo) -> !`. On a microcontroller it usually logs and halts, or resets. There is no unwinding without `std`'s support; configure `panic = "abort"` in the profile.
2. **An entry point**: `#![no_main]` removes Rust's `main`-calling runtime; you export the symbol the platform's startup expects. On a Cortex-M the `cortex-m-rt` crate provides the reset vector and an `#[entry]` macro (from memory); here, on Linux, I use `_start`.
3. **Possibly an allocator**, if you use `alloc`: `#[global_allocator]`.
4. **Memory routines**: the compiler emits calls to `memcpy`, `memmove`, `memset`, `memcmp`. Normally `libc` or `compiler_builtins` provides them (the latter ships them for many bare-metal targets, from memory). In my Linux demo with no C library the link failed with `undefined symbol: memcpy` until I defined them ✓. Do not write `memcpy` in a naive loop: the compiler may turn the loop into a call to `memcpy` (infinite recursion).

Here is a complete freestanding program that I compiled and ran ✓. It links **no `std` and no libc**; it makes the Linux `write` and `exit` system calls itself, and it uses `Vec` and `format!` with a 4 KiB bump allocator:

```rust
// FREESTANDING: build with the flags shown below the listing
#![no_std]
#![no_main]
extern crate alloc;
use alloc::{string::String, vec::Vec, format};
use core::alloc::{GlobalAlloc, Layout};
use core::arch::asm;
use core::cell::UnsafeCell;
use core::panic::PanicInfo;
use core::sync::atomic::{AtomicUsize, Ordering};

const ARENA: usize = 4096;
#[repr(align(16))]
struct Arena(UnsafeCell<[u8; ARENA]>);
struct Bump { arena: Arena, next: AtomicUsize }
unsafe impl Sync for Bump {}

unsafe impl GlobalAlloc for Bump {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        let mut cur = self.next.load(Ordering::Relaxed);
        loop {
            let start = (cur + l.align() - 1) & !(l.align() - 1);
            let end = match start.checked_add(l.size()) { Some(e) => e, None => return core::ptr::null_mut() };
            if end > ARENA { return core::ptr::null_mut(); }
            match self.next.compare_exchange(cur, end, Ordering::Relaxed, Ordering::Relaxed) {
                Ok(_) => return unsafe { (self.arena.0.get() as *mut u8).add(start) },
                Err(c) => cur = c,
            }
        }
    }
    unsafe fn dealloc(&self, _: *mut u8, _: Layout) {}
}

#[global_allocator]
static A: Bump = Bump { arena: Arena(UnsafeCell::new([0; ARENA])), next: AtomicUsize::new(0) };

#[panic_handler]
fn panic(_: &PanicInfo) -> ! { exit(101) }

fn write(fd: usize, buf: &[u8]) {
    unsafe { asm!("syscall", inlateout("rax") 1usize => _, in("rdi") fd, in("rsi") buf.as_ptr(), in("rdx") buf.len(), lateout("rcx") _, lateout("r11") _, options(nostack)); }
}
fn exit(code: usize) -> ! {
    unsafe { asm!("syscall", in("rax") 60usize, in("rdi") code, options(noreturn, nostack)); }
}


// A freestanding program must supply the memory routines the compiler emits calls to.
#[unsafe(no_mangle)]
pub unsafe extern "C" fn memcpy(d: *mut u8, s: *const u8, n: usize) -> *mut u8 {
    unsafe { asm!("rep movsb", inout("rcx") n => _, inout("rdi") d => _, inout("rsi") s => _, options(nostack, preserves_flags)); }
    d
}
#[unsafe(no_mangle)]
pub unsafe extern "C" fn memmove(d: *mut u8, s: *const u8, n: usize) -> *mut u8 {
    if (d as usize) <= (s as usize) || (d as usize) >= (s as usize) + n {
        unsafe { memcpy(d, s, n) }
    } else {
        let mut i = n;
        while i > 0 { i -= 1; unsafe { d.add(i).write_volatile(s.add(i).read()) }; }
        d
    }
}
#[unsafe(no_mangle)]
pub unsafe extern "C" fn memset(d: *mut u8, c: i32, n: usize) -> *mut u8 {
    unsafe { asm!("rep stosb", inout("rcx") n => _, inout("rdi") d => _, in("al") c as u8, options(nostack, preserves_flags)); }
    d
}
#[unsafe(no_mangle)]
pub unsafe extern "C" fn memcmp(a: *const u8, b: *const u8, n: usize) -> i32 {
    let mut i = 0;
    while i < n { let (x, y) = unsafe { (a.add(i).read_volatile(), b.add(i).read_volatile()) }; if x != y { return x as i32 - y as i32; } i += 1; }
    0
}
#[unsafe(no_mangle)]
pub unsafe extern "C" fn bcmp(a: *const u8, b: *const u8, n: usize) -> i32 { unsafe { memcmp(a, b, n) } }
// alloc is built with unwinding; with panic=abort nothing ever unwinds, but the symbol must exist.
#[unsafe(no_mangle)]
pub extern "C" fn _Unwind_Resume() -> ! { exit(102) }

core::arch::global_asm!(".globl _start", "_start:", "xor ebp, ebp", "and rsp, -16", "call start_rust", "ud2");

#[unsafe(no_mangle)]
pub extern "C" fn start_rust() -> ! {
    let mut v: Vec<u32> = Vec::new();
    for i in 0..5 { v.push(i * i); }
    let s: String = format!("squares={:?} used={}\n", v, A.next.load(Ordering::Relaxed));
    write(1, s.as_bytes());
    let big: Vec<u8> = Vec::with_capacity(1_000_000); // exceeds arena -> alloc error
    let _ = big;
    exit(0)
}
#[unsafe(no_mangle)]
pub extern "C" fn rust_eh_personality() {}
```

```text
$ rustc --edition 2024 -O -C panic=abort -C link-arg=-nostartfiles \
      -C link-arg=-static -C relocation-model=static a07.rs -o a07
$ ./a07
squares=[0, 1, 4, 9, 16] used=48
$ echo $?
101
```

The exit status 101 is deliberate ✓: the program then asks for a 1,000,000-byte `Vec`, the allocator returns null, the allocation-error path calls the panic handler, and my handler exits with 101. A heap that cannot satisfy a request is a reachable condition on a device, so decide what happens on purpose.

Things this demo teaches:

- The kernel enters `_start` with a stack alignment that compiled Rust code does not expect. My first version crashed with a segmentation fault when optimized ✓ (the debug build had worked by luck); the `global_asm!` stub that realigns `rsp` fixed it. On real microcontrollers the startup crate handles this.
- `rust_eh_personality` and `_Unwind_Resume` stubs were needed ✓ because the precompiled `core` and `alloc` still mention unwinding support. With `panic = "abort"` they are never executed.
- The no-heap-at-all version in §14 builds a **9,864-byte** stripped executable ✓ (626,416 bytes unstripped, because of debug information in `core`). For comparison, a plain `println!("hi")` program was 4,358,728 bytes at `-O` and 341,048 bytes with `-O -C panic=abort -C strip=symbols` ✓. These are single measurements on this machine, not a benchmark; they show *why* firmware avoids `std` formatting machinery, not exact savings.
- A bump allocator never frees. It is deterministic and tiny, and wrong for long-running programs. Real embedded projects use a crate allocator (from memory: `embedded-alloc`) or, better, no heap at all.

---

## 5. Living without a heap

Embedded code often has no allocator, because allocation can fail, fragment, and take unbounded time. The replacement is **fixed capacity known at compile time**, with const generics (Lesson 19):

- `[T; N]` and `MaybeUninit<[T; N]>` for storage;
- queues, strings and maps with capacity `N` that return an error (or the rejected value) when full instead of growing;
- `static` data placed in a known memory region.

`nostdlab`'s `RingBuffer<T, N>` and `FixedString<N>` (§14) show the pattern. Their `push`/`push_str` are **fallible by design**: a full buffer is a normal event, not a panic, and the rejected value is handed back so the caller can decide ✓ (tests: `ring_rejects_when_full_and_returns_the_value`, `fixed_string_formats_and_is_all_or_nothing`).

`core::fmt::Write` is how you format text with no heap: implement `write_str` for your buffer, then `write!(buf, "id={}", 42)` works ✓. Return `Err` rather than truncating silently, so a half-written log line cannot be mistaken for a complete one; and note the all-or-nothing rule keeps multibyte characters whole (a 3-byte `日` does not fit in 2 remaining bytes and leaves the string unchanged ✓).

---

## 6. Shared state without an OS

There are no threads or `Mutex` in `core`, but there is still concurrency: **interrupts** run in the middle of your code. The same data-race rules apply (Lesson 28), and `static mut` is denied by default in edition 2024 for good reason (Lesson 36). The tools, in the order to prefer them:

1. **Atomics** (`core::sync::atomic`) for flags and counters. Available sizes depend on the target; some small cores lack atomic read-modify-write instructions (from memory: you may get only loads and stores, or need a polyfill crate).
2. **Critical sections**: briefly disable interrupts around a read-modify-write. A `cortex-m`-style `Mutex<RefCell<T>>` that proves the interrupts are off is the usual idiom (from memory); the `critical-section` crate abstracts it.
3. **Ownership transfer**: a single-producer single-consumer queue (like `RingBuffer`, but split into producer and consumer halves) lets an interrupt handler and the main loop share data without locks. My `RingBuffer` is **not** safe to share this way: it takes `&mut self`, so the borrow checker forbids sharing it. Turning it into a lock-free SPSC queue requires atomics and careful orderings, which is deliberately out of scope.

---

## 7. Embedded basics

Everything in this section is from documentation-level memory unless marked ✓.

### 7.1 Targets and the toolchain

A **target triple** names architecture, vendor, OS and ABI. Bare-metal targets have `none` as the OS: for example `thumbv7em-none-eabihf` is a Cortex-M4/M7 with hardware floating point, and `riscv32imac-unknown-none-elf` is a RISC-V core. You install the precompiled `core` for a target with `rustup target add <triple>` (it failed in this sandbox) and build with `cargo build --target <triple>`. The usual project files are:

- `.cargo/config.toml` with `[build] target = "..."` and a `runner` (for example a flashing tool such as `probe-rs`);
- `memory.x`, a linker-script fragment telling the linker the flash and RAM addresses and sizes;
- a startup crate (`cortex-m-rt` for Cortex-M) providing the vector table and `#[entry]`.

`cargo` cannot build `std` for these targets (there is none), so all dependencies must be `no_std`. Add `panic = "abort"` to `[profile.dev]` and `[profile.release]` (the `Cargo.toml` profile setting; I used the equivalent `-C panic=abort` flag ✓), and typically `opt-level = "s"` or `"z"`, `lto = true`, `codegen-units = 1` for size.

### 7.2 Memory-mapped I/O and `volatile`

Hardware registers appear as ordinary addresses. Reading `0x4000_0000` is not reading memory; it may return a *different value every time*, and writing it may start a motor. The compiler does not know that. If you write a normal read in a loop, it may legally read **once** and loop forever. I compiled both versions at `-O` and looked at the assembly ✓:

```rust
#![no_std]

#[unsafe(no_mangle)]
pub fn spin_plain(p: &u8) {
    while *p == 0 {}
}

#[unsafe(no_mangle)]
pub unsafe fn spin_volatile(p: *const u8) {
    while unsafe { p.read_volatile() } == 0 {}
}
```

```text
spin_plain:
	cmpb	$0, (%rdi)        ; read ONCE
	je	.LBB0_1
	retq
.LBB0_1:
	jmp	.LBB0_1           ; ...then spin forever without reading again

spin_volatile:
.LBB1_1:
	cmpb	$0, (%rdi)        ; read on every iteration
	je	.LBB1_1
	retq
```

`read_volatile`/`write_volatile` (Lesson 36) tell the compiler that each access is an observable side effect: it may not remove, merge, duplicate or reorder them relative to other volatile accesses. They do **not** make anything atomic, do not order surrounding non-volatile accesses, and require a valid, aligned pointer. `nostdlab::reg::Reg32` wraps this: `unsafe fn new(addr)` carries the contract, the methods are safe. Its tests run against a normal `u32` variable ✓, so the logic is testable on a PC with no hardware.

In real projects you rarely write `Reg32` by hand: a **PAC** (peripheral access crate), generated from the vendor's SVD file by a tool (`svd2rust`, from memory), exposes every register and field as typed methods; a **HAL** builds safe, ownership-based APIs on top, and a **BSP** targets one specific board. The `embedded-hal` traits (from memory) let a driver for a sensor work with any HAL. The typestate pattern from [Lesson 15](15_newtype_typestate.md) is used heavily here: a pin type that changes from `Pin<Input>` to `Pin<Output>` when you call `into_output()`, so using it in the wrong mode is a compile error.

### 7.3 Bit fields

Registers pack several fields into one word. `get_bits`/`set_bits` in the crate return `None` for a field that does not fit and for a value that has too many bits (so `set_bits(0, 4, 2, 4)` is an error, not a silent mask ✓). The `32`-bit-wide field is special-cased because `1u32 << 32` is an overflow (a panic in debug builds), which the tests cover ✓.

### 7.4 Panics, asserts and code size

`unwrap()`, indexing and integer overflow checks all pull in panic machinery and formatting code. On small devices that costs both flash and determinism. The idioms: return `Result`/`Option` and handle them; use `get()` instead of `[i]`; and write the library so that no input can panic (the frame decoder never indexes before checking the length ✓: a length byte pointing past the end gives `Truncated`, tested).

### 7.5 `async` on embedded

`core::future` is in `core`, so `async fn` works with no OS; executors such as Embassy (from memory) turn interrupts into wakeups. Everything in Lessons 30 to 32 about futures still holds, minus Tokio.

---

## 8. Miri

**Miri** is an interpreter for Rust's mid-level IR that executes your program step by step and reports undefined behavior it can *prove happened on that execution*. It is distributed as a `rustup` component for the **nightly** toolchain (from memory). Usage, as documented:

```text
rustup +nightly component add miri
cargo +nightly miri test
cargo +nightly miri run
MIRIFLAGS="-Zmiri-strict-provenance" cargo +nightly miri test
```

**I could not run any of this**: `rustup component add miri` failed in this sandbox, so everything here is from memory and should be checked against the Miri README. What it is documented to detect (from memory):

- out-of-bounds and use-after-free accesses, including through raw pointers;
- reads of uninitialized memory and invalid values (a `bool` that is not 0 or 1, a dangling reference, an invalid enum discriminant);
- misaligned pointer dereferences;
- violations of the **aliasing rules** (Stacked Borrows by default; Tree Borrows is an alternative model selected by a flag);
- data races between threads, memory leaks reported at exit, and some misuse of `unsafe` APIs such as `from_raw_parts`.

What it cannot do:

- **It only checks the paths your tests execute**, on the inputs they use. A clean Miri run is evidence, not proof.
- It is **much slower** than native execution, so tests should be small.
- It cannot execute foreign code: a call to C (Lesson 37) is unsupported unless Miri has a shim for that function, so FFI-heavy code needs the Rust side tested with fakes. Inline assembly is unsupported (my freestanding demos cannot run under it; the logic in `nostdlab` can, because it is pure Rust).
- Its aliasing model is still being refined, so some code that Miri rejects may be allowed later, and the reverse. Treat a Miri error as "this is at best a risk; read the report".
- Hardware-dependent behavior (real MMIO, interrupts) is outside its model.

**How I would use it on `nostdlab`** (not run): `RingBuffer`, `FixedString` and `reg` are pure Rust with `unsafe` blocks over `MaybeUninit`, `from_utf8_unchecked`, and volatile accesses to ordinary memory, which is exactly Miri's target. I would run `cargo +nightly miri test` on the whole suite, then add `-Zmiri-many-seeds` for randomized scheduling if threads were involved (from memory). The tools I *did* run are weaker: valgrind found no errors in the test binary ✓, but valgrind does not check aliasing, validity (an invalid `bool`) or alignment the way Miri is documented to.

Other tools in the same family (all from memory, not run): AddressSanitizer/ThreadSanitizer on nightly, `cargo-fuzz` for crashing inputs, and `loom` for exploring thread interleavings of your own synchronization.

---

## 9. Performance, reliability, security

- **Size and time are budgets.** Measure them: `cargo size`/`cargo bloat` (third-party, from memory) for flash, a cycle counter or logic analyzer for time. I have given you only the x86-64 file sizes above.
- **Determinism.** No heap means no fragmentation and no out-of-memory surprise; the cost is choosing capacities up front. Document each capacity and what happens when it is exceeded.
- **Watchdogs and resets.** A panic handler that loops forever will hang a device; many designs reset instead. Decide per product.
- **Security.** Firmware parses untrusted bytes (radio, USB, serial). `frame::decode` shows the minimum: validate length before indexing, check the CRC, reject trailing bytes, never allocate based on an attacker-controlled length ✓. A CRC detects accidental corruption, **not** tampering: it is not authentication. Use a MAC or signature (Lesson on cryptography, later) when an attacker is in the model.
- **`unsafe` budget.** Hardware access needs it; keep every `unsafe` block behind a safe type with a `SAFETY` comment (Lesson 36), as `Reg32` does.

---

## 10. Common mistakes

1. Using `std` types in a `no_std` crate by habit (`std::fmt`, `std::error::Error`); import from `core` (or `alloc`).
2. A `std`-using dependency in a `no_std` build; check `default-features = false`.
3. Normal reads and writes to a device register (loop optimized away, as in the debugging challenge).
4. Assuming `volatile` makes an access atomic or orders other memory.
5. `static mut` for interrupt-shared data.
6. Panicking on bad input, or on a full buffer, in code that must keep running.
7. Allocation in an interrupt handler, or an unbounded `Vec` growth on a device with a fixed heap.
8. Features that change behavior instead of adding items; not testing the no-feature build.
9. Formatting with `{:?}` in hot or tiny firmware paths without checking code size.
10. Trusting a CRC for security.
11. Believing that a clean Miri or valgrind run proves the absence of UB.
12. Forgetting that integer overflow checks exist in debug and not in release: `1u32 << 32`, `a + b` and `usize` arithmetic on lengths need explicit `checked_*` handling in parsers.

---

## 11. Exercises

Answers are collapsed. Try each before opening it.

1. **(Warm-up)** Which layer (`core`, `alloc`, `std`) provides `Option`, `Vec`, `HashMap`, `Box`, `AtomicU32`, `Duration` and `Instant`?
2. **(Warm-up)** What does `#![no_std]` do, and what does it not do (for example, does it remove `Vec` from your program)?
3. **(Easy)** Write a `no_std` function `fn hex_nibble(n: u8) -> Option<char>` that returns the lowercase hex digit for 0 to 15 and `None` otherwise, with no allocation.
4. **(Medium)** Implement `core::fmt::Write` for a fixed `[u8; 16]` log buffer so that a write that does not fit returns `Err` and leaves the buffer unchanged.
5. **(Medium)** A loop `while STATUS.is_busy() {}` hangs in release builds only. Name two possible causes and the fix for each.
6. **(Medium-hard)** Why can you not just use `HashMap` in a `no_std` + `alloc` crate, and what are two alternatives?
7. **(Hard)** `RingBuffer::push` computes `(head + len) % N`. Why is the modulo safe when `N` is a const generic that could be 0?
8. **(Hard)** Your colleague says "we ran Miri and it was clean, so the `unsafe` in the ring buffer is correct." Give three reasons that conclusion is too strong.

<details markdown="1">
<summary>Answers</summary>

1. `Option`, `AtomicU32`, `Duration` are in `core`; `Vec` and `Box` are in `alloc`; `HashMap` and `Instant` need `std`. (`std` re-exports the first two groups.)
2. It links `core` instead of `std` and drops `std` from the prelude, so `std`-only items (`println!`, files, threads, `HashMap`) fail to compile ✓. It does not forbid heap types: `extern crate alloc;` brings `Vec`, `Box` and `String` back ✓, and the final binary must then provide a `#[global_allocator]`.
3. ```rust
   #![no_std]

   pub fn hex_nibble(n: u8) -> Option<char> {
       match n {
           0..=9 => Some((b'0' + n) as char),
           10..=15 => Some((b'a' + n - 10) as char),
           _ => None,
       }
   }

   #[cfg(test)]
   mod tests {
       use super::*;
       #[test]
       fn table() {
           assert_eq!(hex_nibble(0), Some('0'));
           assert_eq!(hex_nibble(10), Some('a'));
           assert_eq!(hex_nibble(15), Some('f'));
           assert_eq!(hex_nibble(16), None);
       }
   }
   ```
   (Unit tests in a `no_std` crate work on a host build because the test harness links `std`; run it with `cargo test`.)
4. ```rust
   use core::fmt::{self, Write};

   pub struct Log {
       buf: [u8; 16],
       len: usize,
   }

   impl Write for Log {
       fn write_str(&mut self, s: &str) -> fmt::Result {
           let end = self.len.checked_add(s.len()).ok_or(fmt::Error)?;
           if end > self.buf.len() {
               return Err(fmt::Error);
           }
           self.buf[self.len..end].copy_from_slice(s.as_bytes());
           self.len = end;
           Ok(())
       }
   }

   fn main() {
       let mut l = Log { buf: [0; 16], len: 0 };
       assert!(write!(l, "temp={}", 21).is_ok());
       assert!(l.write_str("0123456789ABCDEF").is_err());
       assert_eq!(&l.buf[..l.len], b"temp=21");
   }
   ```
   Note that `write!` with several pieces can fail *midway* (an earlier piece was written, a later one did not fit); `write_str` is all-or-nothing per call, not per `write!`.
5. (a) The status register is read with a normal load, so the optimizer hoists the load out of the loop (✓ `spin_plain` above): use `read_volatile` (or a PAC/`Reg32` that does). (b) The flag is set by an interrupt handler or another thread and read through a plain variable: use an atomic with at least `Acquire`/`Release` ordering (Lesson 28). A third possibility: the interrupt that sets the flag is masked or never fires.
6. The default hasher (`RandomState`) needs random keys from the operating system, and the standard `HashMap` is defined in `std`. Alternatives: `alloc::collections::BTreeMap` (ordered, no hashing), or a third-party hash map that takes an explicit hasher/seed (and a fixed-capacity one, such as `heapless`'s, for no heap; from memory).
7. `push` returns early when `len == N`, so the modulo is only reached when `len < N`, which implies `N > 0`. For `N == 0` the early return always fires, so a modulo by zero is unreachable ✓ (test `ring_of_capacity_zero_never_accepts`). In `pop` the modulo is reached only after `len > 0`, so `N > 0` as well.
8. (1) Miri checks only executed paths: if no test fills the ring across the wrap with `Rc` elements, that path was not examined. (2) Miri's models (aliasing, validity) are not the final specification, and it does not cover every kind of UB or any logic bug (a wrong index that stays in bounds is invisible to it). (3) Tests run under Miri are small and deterministic, so concurrency and rare inputs are barely sampled; plus the ring buffer's `Send`/`Sync` story and the hardware side (volatile, interrupts) are outside what it can interpret. A clean run is evidence; a reviewed `SAFETY` argument plus tests plus tools is the standard.

</details>

---

## 12. Debugging challenge

A driver waits for a hardware "ready" bit. It passes every test on the developer's PC (the test sets the flag first), works in debug builds on the board, and **hangs forever in release builds**.

```rust
pub struct Uart {
    status: *const u8,
}

impl Uart {
    /// # Safety
    /// `status` must be valid for reads for the lifetime of the `Uart`.
    pub unsafe fn new(status: *const u8) -> Uart {
        Uart { status }
    }

    pub fn wait_ready(&self) {
        // SAFETY: `new` requires `status` to be valid for reads.
        let status: &u8 = unsafe { &*self.status };
        while *status & 1 == 0 {}
    }
}
```

✓ The same shape (`while *p == 0 {}` over a reference) compiled at `-O` to a single `cmpb $0, (%rdi)` followed by `jmp .LBB0_1` that jumps to itself: one read, then an infinite loop. Explain, and fix.

<details markdown="1">
<summary>Solution</summary>

`&u8` promises the compiler that, for as long as the reference lives, the pointee is **not modified by anyone else** (shared references are immutable for non-`UnsafeCell` data). So reading `*status` again in the loop cannot give a different value, and the optimizer is allowed to read it once. In debug builds nothing is hoisted, so it "works". Creating that reference to memory that hardware changes is itself the bug, not just the loop.

Fix: never form a reference to the register; read through the raw pointer with a volatile access on every iteration:

```rust
pub struct Uart {
    status: *const u8,
}

impl Uart {
    /// # Safety
    /// `status` must be valid and aligned for volatile reads for the lifetime of the `Uart`.
    pub unsafe fn new(status: *const u8) -> Uart {
        Uart { status }
    }

    pub fn wait_ready(&self) {
        // SAFETY: `new` requires `status` to be valid for reads.
        while unsafe { self.status.read_volatile() } & 1 == 0 {}
    }
}
```

(✓ the volatile version compiled to a loop that re-reads on every iteration.) Production code usually goes further: a bounded loop with a timeout and an error return, because a device that never becomes ready should not hang the system.

</details>

---

## 13. Interview questions

1. *What is the difference between `core`, `alloc` and `std`?*
2. *What does a `#![no_std]` binary need to provide that a normal one gets for free?*
3. *Why is `HashMap` not in `alloc`?*
4. *Why do you use `read_volatile` for device registers? Does it make them atomic?*
5. *How do you share data between an interrupt handler and the main loop?*
6. *What does Miri check, and what can it not check?*
7. *How do you design a library to work both with and without a heap?*
8. *You have 2 KB of RAM. How do you structure a message queue?*

<details markdown="1">
<summary>Answers</summary>

1. `core` is the portable part that needs nothing; `alloc` adds heap types and needs a global allocator; `std` adds OS-dependent features (files, threads, time, `HashMap`) and re-exports the others.
2. A panic handler (`#[panic_handler]`), an entry point (`#![no_main]` plus the platform's startup symbol), `panic = "abort"`, optionally a `#[global_allocator]`, and sometimes memory routines (`memcpy` and friends) ✓. A missing panic handler is a compile error ✓.
3. Its default hasher uses random keys from the OS to resist hash-flooding, and `HashMap` is defined in `std` ✓ (`alloc::collections::HashMap` is `E0432`). Use `BTreeMap` or a map with an explicit hasher.
4. Because the compiler may otherwise elide, merge or hoist plain reads and writes (✓ the single read in `spin_plain`). `volatile` forces every access to happen as written, in order relative to other volatile accesses. It is not atomic and gives no ordering for ordinary memory.
5. Atomics for flags and counters; a critical section (briefly masking interrupts) for compound updates; or a lock-free single-producer single-consumer queue split into two halves. Never `static mut`.
6. It interprets your program and reports undefined behavior on the executed path: out-of-bounds, use-after-free, uninitialized or invalid values, misalignment, aliasing violations, some data races, leaks. It cannot check unexecuted paths, foreign code or inline assembly, hardware, or logic errors, and it is slow (from memory; I could not run it here).
7. `#![no_std]` unconditionally, `core`-only types by default, an additive `alloc` feature guarded with `#[cfg(feature = "alloc")] extern crate alloc;`, and CI that builds and tests both configurations ✓ (`cargo test` and `cargo test --features alloc`).
8. Fixed capacity at compile time (`RingBuffer<Msg, N>` with const generics), fallible `push` that returns the rejected message, no heap, bounded message size (a maximum payload as in `frame::MAX_PAYLOAD`), and a documented policy for "full" (drop oldest, drop newest, or apply backpressure).

</details>

---

## 14. Practical challenge: `nostdlab`

Build a library crate `nostdlab` for firmware-style code. Requirements:

1. `#![no_std]` unconditionally; no dependencies; an additive `alloc` feature; default build uses `core` only. Clippy's `std_instead_of_core` family enabled.
2. `RingBuffer<T, const N: usize>` on `MaybeUninit` storage: `new` (const), `push` returning the rejected value when full, `pop`, `front`, `iter` (exact size), `clear`, `Drop` that drops each element exactly once, `drain_to_vec` behind the `alloc` feature. Correct for `N == 0` and zero-sized types.
3. `FixedString<N>` implementing `core::fmt::Write`, all-or-nothing writes, always valid UTF-8.
4. A `frame` module: `0x7E, len, payload, crc16` encode and decode (CRC-16/CCITT-FALSE, check value `0x29B1` for `"123456789"`), `decode_prefix` for streams, `#[non_exhaustive]` error enum implementing `core::error::Error`, and no panic for any input.
5. `reg::Reg32` (volatile) and `get_bits`/`set_bits` with edge cases (width 0, 32, overflow of the field).
6. Every `unsafe` block has a `// SAFETY:` comment; every `unsafe fn` has `# Safety`.
7. Tests with and without the `alloc` feature, plus valgrind; and one genuinely freestanding executable that links the crate.

<details markdown="1">
<summary>Solution (spoilers: full source)</summary>

`Cargo.toml`:

```toml
[package]
name = "nostdlab"
version = "0.1.0"
edition = "2024"

[features]
default = []
alloc = []

[dependencies]
```

`src/lib.rs`:

```rust
//! `nostdlab`: building blocks for firmware-style code that runs without the
//! standard library and, in its default configuration, without a heap.
//!
//! * [`RingBuffer`]: a fixed-capacity FIFO queue on `MaybeUninit` storage.
//! * [`FixedString`]: a stack-allocated string that implements `core::fmt::Write`.
//! * [`frame`]: encode and decode length-prefixed frames protected by a CRC-16.
//! * [`reg`]: a volatile 32-bit register wrapper and bit-field helpers.
//!
//! The crate is `#![no_std]` unconditionally and never links `std`. Its error
//! type implements `core::error::Error`, which lives in `core`. The optional
//! `alloc` feature adds helpers that need a heap (`extern crate alloc`).
//!
//! ```
//! use nostdlab::{FixedString, RingBuffer};
//! use core::fmt::Write;
//!
//! let mut q: RingBuffer<u8, 2> = RingBuffer::new();
//! assert_eq!(q.push(1), Ok(()));
//! assert_eq!(q.push(2), Ok(()));
//! assert_eq!(q.push(3), Err(3)); // full: the value is handed back
//! assert_eq!(q.pop(), Some(1));
//!
//! let mut s = FixedString::<16>::new();
//! write!(s, "id={}", 42).unwrap();
//! assert_eq!(s.as_str(), "id=42");
//! ```

#![no_std]
#![deny(missing_docs)]
#![deny(unsafe_op_in_unsafe_fn)]
#![warn(clippy::undocumented_unsafe_blocks)]
#![warn(
    clippy::std_instead_of_core,
    clippy::std_instead_of_alloc,
    clippy::alloc_instead_of_core
)]

#[cfg(feature = "alloc")]
extern crate alloc;

mod fixed;
pub mod frame;
pub mod reg;
mod ring;

pub use fixed::FixedString;
pub use ring::{Iter, RingBuffer};
```

`src/ring.rs`:

```rust
use core::mem::MaybeUninit;

/// A first-in first-out queue with room for exactly `N` elements, stored
/// inline (no heap, no allocation).
///
/// `push` on a full queue returns the rejected value instead of panicking or
/// overwriting, so the caller decides what to drop.
pub struct RingBuffer<T, const N: usize> {
    buf: [MaybeUninit<T>; N],
    /// Index of the oldest element. Always `< N` when `N > 0`.
    head: usize,
    /// Number of initialized elements, starting at `head`, wrapping.
    len: usize,
}

impl<T, const N: usize> RingBuffer<T, N> {
    /// Creates an empty queue.
    pub const fn new() -> Self {
        RingBuffer {
            buf: [const { MaybeUninit::uninit() }; N],
            head: 0,
            len: 0,
        }
    }

    /// The fixed capacity `N`.
    pub const fn capacity(&self) -> usize {
        N
    }

    /// Number of elements currently stored.
    pub const fn len(&self) -> usize {
        self.len
    }

    /// Whether the queue holds no elements.
    pub const fn is_empty(&self) -> bool {
        self.len == 0
    }

    /// Whether the queue is at capacity.
    pub const fn is_full(&self) -> bool {
        self.len == N
    }

    /// Appends `value`, or returns it if the queue is full.
    pub fn push(&mut self, value: T) -> Result<(), T> {
        if self.len == N {
            return Err(value);
        }
        // `len < N` implies `N > 0`, so the modulo is defined.
        let idx = (self.head + self.len) % N;
        self.buf[idx].write(value);
        self.len += 1;
        Ok(())
    }

    /// Removes and returns the oldest element.
    pub fn pop(&mut self) -> Option<T> {
        if self.len == 0 {
            return None;
        }
        // SAFETY: `len > 0`, so the slot at `head` is initialized; after the
        // read we advance `head` and shrink `len`, so the slot is treated as
        // uninitialized from now on and the value is not duplicated.
        let value = unsafe { self.buf[self.head].assume_init_read() };
        self.head = (self.head + 1) % N;
        self.len -= 1;
        Some(value)
    }

    /// The oldest element, without removing it.
    pub fn front(&self) -> Option<&T> {
        if self.len == 0 {
            return None;
        }
        // SAFETY: `len > 0`, so the slot at `head` is initialized.
        Some(unsafe { self.buf[self.head].assume_init_ref() })
    }

    /// Iterates from the oldest to the newest element.
    pub fn iter(&self) -> Iter<'_, T, N> {
        Iter {
            queue: self,
            pos: 0,
        }
    }

    /// Removes all elements, dropping them from oldest to newest.
    pub fn clear(&mut self) {
        while self.pop().is_some() {}
    }

    /// Moves all elements into a `Vec`, oldest first.
    #[cfg(feature = "alloc")]
    pub fn drain_to_vec(&mut self) -> alloc::vec::Vec<T> {
        let mut out = alloc::vec::Vec::with_capacity(self.len);
        while let Some(v) = self.pop() {
            out.push(v);
        }
        out
    }
}

impl<T, const N: usize> Default for RingBuffer<T, N> {
    fn default() -> Self {
        Self::new()
    }
}

impl<T, const N: usize> Drop for RingBuffer<T, N> {
    fn drop(&mut self) {
        // `MaybeUninit` never drops its contents, so do it explicitly.
        self.clear();
    }
}

/// Iterator over a [`RingBuffer`], oldest first.
pub struct Iter<'a, T, const N: usize> {
    queue: &'a RingBuffer<T, N>,
    pos: usize,
}

impl<'a, T, const N: usize> Iterator for Iter<'a, T, N> {
    type Item = &'a T;

    fn next(&mut self) -> Option<&'a T> {
        if self.pos >= self.queue.len {
            return None;
        }
        let idx = (self.queue.head + self.pos) % N;
        self.pos += 1;
        // SAFETY: `pos < len`, so slot `head + pos (mod N)` is initialized,
        // and the shared borrow of the queue keeps it so for `'a`.
        Some(unsafe { self.queue.buf[idx].assume_init_ref() })
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        let rest = self.queue.len - self.pos;
        (rest, Some(rest))
    }
}

impl<T, const N: usize> ExactSizeIterator for Iter<'_, T, N> {}
```

`src/fixed.rs`:

```rust
use core::fmt;

/// A UTF-8 string with inline storage for at most `N` bytes.
///
/// Writes are all-or-nothing: if a string does not fit, nothing is appended
/// and `fmt::Error` is returned, so the contents are always valid UTF-8.
pub struct FixedString<const N: usize> {
    buf: [u8; N],
    len: usize,
}

impl<const N: usize> FixedString<N> {
    /// Creates an empty string.
    pub const fn new() -> Self {
        FixedString {
            buf: [0; N],
            len: 0,
        }
    }

    /// The contents.
    pub fn as_str(&self) -> &str {
        // SAFETY: `buf[..len]` is only ever extended by whole `&str`s in
        // `push_str`, and a concatenation of valid UTF-8 strings is valid.
        unsafe { core::str::from_utf8_unchecked(&self.buf[..self.len]) }
    }

    /// Length in bytes.
    pub const fn len(&self) -> usize {
        self.len
    }

    /// Whether the string is empty.
    pub const fn is_empty(&self) -> bool {
        self.len == 0
    }

    /// Bytes still available.
    pub const fn remaining(&self) -> usize {
        N - self.len
    }

    /// Appends `s`, or returns an error and leaves the string unchanged.
    pub fn push_str(&mut self, s: &str) -> Result<(), fmt::Error> {
        if s.len() > self.remaining() {
            return Err(fmt::Error);
        }
        self.buf[self.len..self.len + s.len()].copy_from_slice(s.as_bytes());
        self.len += s.len();
        Ok(())
    }

    /// Empties the string.
    pub fn clear(&mut self) {
        self.len = 0;
    }
}

impl<const N: usize> Default for FixedString<N> {
    fn default() -> Self {
        Self::new()
    }
}

impl<const N: usize> fmt::Write for FixedString<N> {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        self.push_str(s)
    }
}

impl<const N: usize> fmt::Display for FixedString<N> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.write_str(self.as_str())
    }
}

impl<const N: usize> fmt::Debug for FixedString<N> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        fmt::Debug::fmt(self.as_str(), f)
    }
}
```

`src/frame.rs`:

```rust
//! Length-prefixed frames: `0x7E, len, payload[len], crc_hi, crc_lo`.
//!
//! The CRC is CRC-16/CCITT-FALSE (polynomial `0x1021`, initial value
//! `0xFFFF`, no reflection) computed over `len` and the payload.

use core::fmt;

/// The first byte of every frame.
pub const START: u8 = 0x7E;
/// The largest payload a frame can carry (the length is one byte).
pub const MAX_PAYLOAD: usize = 255;
/// Bytes added around the payload: start, length and two CRC bytes.
pub const OVERHEAD: usize = 4;

/// Why encoding or decoding failed.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[non_exhaustive]
pub enum FrameError {
    /// The payload is longer than [`MAX_PAYLOAD`].
    PayloadTooLong,
    /// The output buffer cannot hold the frame.
    OutputTooSmall,
    /// The input is shorter than any frame, or shorter than its length field says.
    Truncated,
    /// The first byte is not [`START`].
    BadStart,
    /// The input has bytes after the end of the frame.
    TrailingBytes,
    /// The CRC does not match.
    BadCrc {
        /// CRC stored in the frame.
        found: u16,
        /// CRC computed over the received bytes.
        expected: u16,
    },
}

impl fmt::Display for FrameError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            FrameError::PayloadTooLong => f.write_str("payload longer than 255 bytes"),
            FrameError::OutputTooSmall => f.write_str("output buffer too small"),
            FrameError::Truncated => f.write_str("frame is truncated"),
            FrameError::BadStart => f.write_str("frame does not start with 0x7E"),
            FrameError::TrailingBytes => f.write_str("bytes after the end of the frame"),
            FrameError::BadCrc { found, expected } => {
                write!(f, "bad CRC: found {found:#06x}, expected {expected:#06x}")
            }
        }
    }
}

impl core::error::Error for FrameError {}

/// CRC-16/CCITT-FALSE of `data`.
pub fn crc16(data: &[u8]) -> u16 {
    let mut crc: u16 = 0xFFFF;
    for &byte in data {
        crc ^= u16::from(byte) << 8;
        for _ in 0..8 {
            crc = if crc & 0x8000 != 0 {
                (crc << 1) ^ 0x1021
            } else {
                crc << 1
            };
        }
    }
    crc
}

/// Encodes `payload` into `out` and returns the number of bytes written.
pub fn encode(payload: &[u8], out: &mut [u8]) -> Result<usize, FrameError> {
    let len = u8::try_from(payload.len()).map_err(|_| FrameError::PayloadTooLong)?;
    let total = payload.len() + OVERHEAD;
    if out.len() < total {
        return Err(FrameError::OutputTooSmall);
    }
    out[0] = START;
    out[1] = len;
    out[2..2 + payload.len()].copy_from_slice(payload);
    let crc = crc16(&out[1..2 + payload.len()]);
    out[2 + payload.len()..total].copy_from_slice(&crc.to_be_bytes());
    Ok(total)
}

/// A decoded frame, borrowing its payload from the input.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Frame<'a> {
    /// The payload bytes.
    pub payload: &'a [u8],
}

/// Decodes exactly one frame from `input` (no extra bytes allowed).
pub fn decode(input: &[u8]) -> Result<Frame<'_>, FrameError> {
    let (consumed, frame) = decode_prefix(input)?;
    if consumed != input.len() {
        return Err(FrameError::TrailingBytes);
    }
    Ok(frame)
}

/// Decodes the first frame in `input` and returns how many bytes it used,
/// so a caller can loop over a stream of back-to-back frames.
pub fn decode_prefix(input: &[u8]) -> Result<(usize, Frame<'_>), FrameError> {
    let [first, len, ..] = *input else {
        return Err(FrameError::Truncated);
    };
    if first != START {
        return Err(FrameError::BadStart);
    }
    let payload_end = 2 + usize::from(len);
    let total = payload_end + 2;
    let Some(rest) = input.get(payload_end..total) else {
        return Err(FrameError::Truncated);
    };
    let found = u16::from_be_bytes([rest[0], rest[1]]);
    let expected = crc16(&input[1..payload_end]);
    if found != expected {
        return Err(FrameError::BadCrc { found, expected });
    }
    Ok((
        total,
        Frame {
            payload: &input[2..payload_end],
        },
    ))
}
```

`src/reg.rs`:

```rust
//! A volatile register wrapper and bit-field helpers.

use core::ptr;

/// A 32-bit memory-mapped register.
///
/// Every access is a volatile read or write, so the compiler can neither
/// merge, reorder away nor delete it. `modify` is a read-modify-write and is
/// **not** atomic: an interrupt between the read and the write can be lost.
pub struct Reg32 {
    addr: *mut u32,
}

impl Reg32 {
    /// Wraps the register at `addr`.
    ///
    /// # Safety
    ///
    /// `addr` must be non-null, 4-byte aligned, and valid for volatile reads
    /// and writes of a `u32` for as long as the `Reg32` is used (for real
    /// hardware: the documented address of the register; for tests: a live,
    /// properly aligned `u32`). No other code may treat the location as
    /// ordinary memory while the register is in use.
    pub const unsafe fn new(addr: *mut u32) -> Reg32 {
        Reg32 { addr }
    }

    /// Reads the register.
    pub fn read(&self) -> u32 {
        // SAFETY: `new` requires `addr` to be valid and aligned for reads.
        unsafe { ptr::read_volatile(self.addr) }
    }

    /// Writes the register.
    pub fn write(&mut self, value: u32) {
        // SAFETY: `new` requires `addr` to be valid and aligned for writes.
        unsafe { ptr::write_volatile(self.addr, value) }
    }

    /// Reads, applies `f`, writes back, and returns the new value.
    pub fn modify(&mut self, f: impl FnOnce(u32) -> u32) -> u32 {
        let new = f(self.read());
        self.write(new);
        new
    }
}

/// A mask of `width` ones, or `None` if `width` is 0 or more than 32.
const fn mask(width: u32) -> Option<u32> {
    match width {
        0 => None,
        32 => Some(u32::MAX),
        1..=31 => Some((1u32 << width) - 1),
        _ => None,
    }
}

/// Extracts `width` bits of `value` starting at bit `lo`.
///
/// Returns `None` if the field is empty or does not fit in 32 bits.
pub const fn get_bits(value: u32, lo: u32, width: u32) -> Option<u32> {
    let Some(m) = mask(width) else { return None };
    if lo > 32 - width {
        return None;
    }
    Some((value >> lo) & m)
}

/// Replaces `width` bits of `value` starting at bit `lo` with `field`.
///
/// Returns `None` if the field does not fit in 32 bits or `field` has bits
/// set outside `width` (so a too-large value is an error, not a silent mask).
pub const fn set_bits(value: u32, lo: u32, width: u32, field: u32) -> Option<u32> {
    let Some(m) = mask(width) else { return None };
    if lo > 32 - width || field > m {
        return None;
    }
    Some((value & !(m << lo)) | (field << lo))
}
```

`tests/api.rs`:

```rust
//! Tests through the public API. Run with and without features:
//! `cargo test` and `cargo test --features alloc`.

use core::cell::Cell;
use core::fmt::Write;
use nostdlab::frame::{self, FrameError};
use nostdlab::reg::{Reg32, get_bits, set_bits};
use nostdlab::{FixedString, RingBuffer};
use std::rc::Rc;

#[derive(Debug)]
struct Tracked(Rc<Cell<usize>>);
impl Drop for Tracked {
    fn drop(&mut self) {
        self.0.set(self.0.get() + 1);
    }
}

#[test]
fn ring_is_fifo_and_wraps_around() {
    let mut q: RingBuffer<u32, 3> = RingBuffer::new();
    assert!(q.is_empty() && !q.is_full());
    assert_eq!(q.capacity(), 3);
    for round in 0..10u32 {
        assert_eq!(q.push(round * 10), Ok(()));
        assert_eq!(q.push(round * 10 + 1), Ok(()));
        assert_eq!(q.pop(), Some(round * 10));
        assert_eq!(q.pop(), Some(round * 10 + 1));
    }
    assert_eq!(q.pop(), None);
}

#[test]
fn ring_rejects_when_full_and_returns_the_value() {
    let mut q: RingBuffer<String, 2> = RingBuffer::new();
    q.push("a".into()).unwrap();
    q.push("b".into()).unwrap();
    assert!(q.is_full());
    assert_eq!(q.push("c".into()), Err("c".to_string()));
    assert_eq!(q.len(), 2);
    assert_eq!(q.front().map(String::as_str), Some("a"));
}

#[test]
fn ring_iter_follows_logical_order_across_the_wrap() {
    let mut q: RingBuffer<u8, 4> = RingBuffer::new();
    for v in [1, 2, 3, 4] {
        q.push(v).unwrap();
    }
    q.pop();
    q.pop();
    q.push(5).unwrap();
    q.push(6).unwrap();
    let got: Vec<u8> = q.iter().copied().collect();
    assert_eq!(got, [3, 4, 5, 6]);
    assert_eq!(q.iter().len(), 4);
}

#[test]
fn ring_of_capacity_zero_never_accepts() {
    let mut q: RingBuffer<u8, 0> = RingBuffer::new();
    assert!(q.is_empty() && q.is_full());
    assert_eq!(q.push(1), Err(1));
    assert_eq!(q.pop(), None);
    assert_eq!(q.iter().count(), 0);
}

#[test]
fn ring_works_with_zero_sized_types() {
    let mut q: RingBuffer<(), 3> = RingBuffer::new();
    q.push(()).unwrap();
    q.push(()).unwrap();
    assert_eq!(q.len(), 2);
    assert_eq!(q.pop(), Some(()));
}

#[test]
fn ring_drops_each_element_exactly_once() {
    let drops = Rc::new(Cell::new(0));
    {
        let mut q: RingBuffer<Tracked, 4> = RingBuffer::new();
        for _ in 0..4 {
            q.push(Tracked(drops.clone())).unwrap();
        }
        drop(q.pop()); // 1 drop now
        assert_eq!(drops.get(), 1);
        let rejected = q.push(Tracked(drops.clone()));
        assert!(rejected.is_ok());
        let again = q.push(Tracked(drops.clone())); // full: returned, then dropped
        assert!(again.is_err());
        drop(again);
        assert_eq!(drops.get(), 2);
    } // q dropped here: 4 elements remain
    assert_eq!(drops.get(), 6);
    assert_eq!(Rc::strong_count(&drops), 1);
}

#[cfg(feature = "alloc")]
#[test]
fn ring_drain_to_vec_is_oldest_first() {
    let mut q: RingBuffer<u8, 3> = RingBuffer::new();
    for v in [1, 2, 3] {
        q.push(v).unwrap();
    }
    q.pop();
    q.push(4).unwrap();
    assert_eq!(q.drain_to_vec(), [2, 3, 4]);
    assert!(q.is_empty());
}

#[test]
fn fixed_string_formats_and_is_all_or_nothing() {
    let mut s = FixedString::<8>::new();
    write!(s, "ab{}", 12).unwrap();
    assert_eq!(s.as_str(), "ab12");
    assert_eq!(s.remaining(), 4);
    assert!(s.push_str("toolong!").is_err());
    assert_eq!(s.as_str(), "ab12", "unchanged after a failed write");
    s.push_str("cdef").unwrap();
    assert_eq!(s.remaining(), 0);
    assert!(s.push_str("x").is_err());
    assert!(s.push_str("").is_ok(), "empty string always fits");
    assert_eq!(format!("{s} {s:?}"), "ab12cdef \"ab12cdef\"");
    s.clear();
    assert!(s.is_empty());
}

#[test]
fn fixed_string_keeps_multibyte_characters_whole() {
    let mut s = FixedString::<4>::new();
    s.push_str("é").unwrap(); // 2 bytes
    assert!(
        s.push_str("日").is_err(),
        "3 bytes do not fit in the remaining 2"
    );
    assert_eq!(s.as_str(), "é");
    s.push_str("é").unwrap();
    assert_eq!(s.len(), 4);
}

#[test]
fn crc16_matches_the_standard_check_value() {
    assert_eq!(frame::crc16(b"123456789"), 0x29B1);
    assert_eq!(frame::crc16(b""), 0xFFFF);
}

#[test]
fn frames_round_trip() {
    let mut buf = [0u8; 300];
    for payload in [&b""[..], b"x", b"hello", &[0x7E; 255]] {
        let n = frame::encode(payload, &mut buf).unwrap();
        assert_eq!(n, payload.len() + frame::OVERHEAD);
        let f = frame::decode(&buf[..n]).unwrap();
        assert_eq!(f.payload, payload);
    }
}

#[test]
fn frame_encode_errors() {
    let mut small = [0u8; 8];
    assert_eq!(
        frame::encode(b"abcde", &mut small),
        Err(FrameError::OutputTooSmall)
    );
    assert_eq!(frame::encode(b"abcd", &mut small), Ok(8), "exact fit");
    let mut big = [0u8; 400];
    assert_eq!(
        frame::encode(&[0; 256], &mut big),
        Err(FrameError::PayloadTooLong)
    );
}

#[test]
fn frame_decode_errors() {
    let mut buf = [0u8; 16];
    let n = frame::encode(b"abc", &mut buf).unwrap();
    let good = buf[..n].to_vec();

    assert_eq!(frame::decode(&[]), Err(FrameError::Truncated));
    assert_eq!(frame::decode(&[0x7E]), Err(FrameError::Truncated));
    assert_eq!(frame::decode(&good[..n - 1]), Err(FrameError::Truncated));

    let mut bad_start = good.clone();
    bad_start[0] = 0x7F;
    assert_eq!(frame::decode(&bad_start), Err(FrameError::BadStart));

    let mut flipped = good.clone();
    flipped[3] ^= 1;
    assert!(matches!(
        frame::decode(&flipped),
        Err(FrameError::BadCrc { .. })
    ));

    let mut trailing = good.clone();
    trailing.push(0);
    assert_eq!(frame::decode(&trailing), Err(FrameError::TrailingBytes));

    // A length field that points past the end must not panic.
    assert_eq!(
        frame::decode(&[0x7E, 255, 1, 2]),
        Err(FrameError::Truncated)
    );
}

#[test]
fn frame_stream_can_be_split_with_decode_prefix() {
    let mut buf = [0u8; 64];
    let a = frame::encode(b"one", &mut buf).unwrap();
    let b = frame::encode(b"two!", &mut buf[a..]).unwrap();
    let mut rest = &buf[..a + b];
    let mut seen = Vec::new();
    while !rest.is_empty() {
        let (used, f) = frame::decode_prefix(rest).unwrap();
        seen.push(f.payload.to_vec());
        rest = &rest[used..];
    }
    assert_eq!(seen, [b"one".to_vec(), b"two!".to_vec()]);
}

#[test]
fn frame_error_messages() {
    let e = FrameError::BadCrc {
        found: 1,
        expected: 0xABCD,
    };
    assert_eq!(e.to_string(), "bad CRC: found 0x0001, expected 0xabcd");
    assert_eq!(
        FrameError::BadStart.to_string(),
        "frame does not start with 0x7E"
    );
}

#[test]
fn frame_error_implements_the_core_error_trait() {
    fn is_error<E: core::error::Error>(_: &E) {}
    is_error(&FrameError::Truncated);
    let boxed: Box<dyn std::error::Error> = Box::new(FrameError::BadStart);
    assert_eq!(boxed.to_string(), "frame does not start with 0x7E");
}

#[test]
fn register_reads_and_writes_the_backing_word() {
    let mut backing: u32 = 0;
    // SAFETY: `backing` is a live, aligned u32 that nothing else touches.
    let mut r = unsafe { Reg32::new(&raw mut backing) };
    r.write(0x1234_0000);
    assert_eq!(r.read(), 0x1234_0000);
    let new = r.modify(|v| v | 0xF);
    assert_eq!(new, 0x1234_000F);
    assert_eq!(r.read(), 0x1234_000F);
}

#[test]
fn bit_field_helpers_cover_the_edges() {
    assert_eq!(get_bits(0b1011_0000, 4, 4), Some(0b1011));
    assert_eq!(get_bits(u32::MAX, 0, 32), Some(u32::MAX));
    assert_eq!(get_bits(u32::MAX, 31, 1), Some(1));
    assert_eq!(get_bits(1, 0, 0), None);
    assert_eq!(get_bits(1, 0, 33), None);
    assert_eq!(get_bits(1, 29, 4), None, "field runs past bit 31");
    assert_eq!(get_bits(1, 33, 1), None);

    assert_eq!(set_bits(0xFFFF_FFFF, 8, 8, 0), Some(0xFFFF_00FF));
    assert_eq!(set_bits(0, 28, 4, 0xA), Some(0xA000_0000));
    assert_eq!(set_bits(0, 0, 32, 7), Some(7));
    assert_eq!(set_bits(0, 4, 2, 4), None, "4 needs 3 bits");
    assert_eq!(set_bits(0, 31, 2, 0), None);
    assert_eq!(set_bits(0, 0, 0, 0), None);
}
```

A freestanding executable that uses the crate (`main.rs`; built as shown in the box below):

```rust
// FREESTANDING
#![no_std]
#![no_main]

use core::arch::asm;
use core::fmt::Write;
use core::panic::PanicInfo;
use nostdlab::{FixedString, RingBuffer, frame};

#[panic_handler]
fn panic(_: &PanicInfo) -> ! {
    exit(101)
}

fn write(fd: usize, buf: &[u8]) {
    // SAFETY: Linux x86-64 `write` syscall; `buf` is valid for `buf.len()` reads.
    unsafe {
        asm!("syscall", inlateout("rax") 1usize => _, in("rdi") fd, in("rsi") buf.as_ptr(),
             in("rdx") buf.len(), lateout("rcx") _, lateout("r11") _, options(nostack));
    }
}

fn exit(code: usize) -> ! {
    // SAFETY: Linux x86-64 `exit` syscall; it never returns.
    unsafe { asm!("syscall", in("rax") 60usize, in("rdi") code, options(noreturn, nostack)) }
}

// The kernel enters `_start` with `rsp` 16-byte aligned, but compiled Rust
// code expects the alignment it would have after a `call`. Realign, then call.
core::arch::global_asm!(
    ".globl _start",
    "_start:",
    "xor ebp, ebp",
    "and rsp, -16",
    "call start_rust",
    "ud2",
);

#[unsafe(no_mangle)]
pub extern "C" fn start_rust() -> ! {
    let mut wire = [0u8; 32];
    let n = frame::encode(b"ping", &mut wire).unwrap_or(0);
    let mut q: RingBuffer<u8, 8> = RingBuffer::new();
    for &b in &wire[..n] {
        let _ = q.push(b);
    }
    let mut line = FixedString::<64>::new();
    let _ = write!(line, "encoded {} bytes, queued {}, crc={:#06x}\n", n, q.len(), frame::crc16(b"123456789"));
    write(1, line.as_str().as_bytes());
    let decoded_ok = frame::decode(&wire[..n]).map(|f| f.payload == b"ping");
    exit(if decoded_ok == Ok(true) { 0 } else { 1 })
}

// Compiler-emitted calls the freestanding link needs (see the lesson).
#[unsafe(no_mangle)]
pub unsafe extern "C" fn memcpy(d: *mut u8, s: *const u8, n: usize) -> *mut u8 {
    // SAFETY: caller guarantees the ranges are valid and non-overlapping.
    unsafe { asm!("rep movsb", inout("rcx") n => _, inout("rdi") d => _, inout("rsi") s => _, options(nostack, preserves_flags)) };
    d
}
#[unsafe(no_mangle)]
pub unsafe extern "C" fn memset(d: *mut u8, c: i32, n: usize) -> *mut u8 {
    // SAFETY: caller guarantees `d` is valid for `n` writes.
    unsafe { asm!("rep stosb", inout("rcx") n => _, inout("rdi") d => _, in("al") c as u8, options(nostack, preserves_flags)) };
    d
}
#[unsafe(no_mangle)]
pub unsafe extern "C" fn memcmp(a: *const u8, b: *const u8, n: usize) -> i32 {
    let mut i = 0;
    while i < n {
        // SAFETY: caller guarantees both ranges are valid for `n` reads.
        let (x, y) = unsafe { (a.add(i).read_volatile(), b.add(i).read_volatile()) };
        if x != y {
            return i32::from(x) - i32::from(y);
        }
        i += 1;
    }
    0
}
#[unsafe(no_mangle)]
pub unsafe extern "C" fn bcmp(a: *const u8, b: *const u8, n: usize) -> i32 {
    // SAFETY: same contract as `memcmp`.
    unsafe { memcmp(a, b, n) }
}

// The precompiled `core` was built with unwinding support and still mentions
// the personality routine; with `panic=abort` it is never called.
#[unsafe(no_mangle)]
pub extern "C" fn rust_eh_personality() {}
```

```text
$ rustc --edition 2024 --crate-type rlib --crate-name nostdlab -O -C panic=abort src/lib.rs -o libnostdlab.rlib
$ rustc --edition 2024 -O -C panic=abort -C link-arg=-nostartfiles -C link-arg=-static \
      -C relocation-model=static --extern nostdlab=libnostdlab.rlib main.rs -o free
$ ./free
encoded 8 bytes, queued 8, crc=0x29b1
```

(exit status 0, and `ldd free` says `not a dynamic executable` ✓).

**Verification ✓.** `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings` and `cargo test --locked` pass **both** without features (17 integration tests, 1 doctest) and with `--features alloc` (18 integration tests, 1 doctest); `RUSTDOCFLAGS="-D warnings" cargo doc --no-deps --all-features` passes. The integration test binary under `valgrind --leak-check=full --errors-for-leak-kinds=definite` reported no errors (exit status 0). The CRC check value `0x29B1` matched on the first run. **Not run:** Miri, any embedded target, big-endian, or 32-bit targets (so the `usize` arithmetic in the frame code was not tested at 32 bits).

**Mutation checks ✓** (each applied to a copy with its own target directory, tests run with `--features alloc`):

| # | Mutation | Result |
|---|---|---|
| M1 | `push` full check `len == N` becomes `len > N` | **caught** |
| M2 | `pop` does not advance `head` (wraps in place) | **caught** |
| M3 | `push` ignores `head` when computing the slot | **caught** |
| M4 | `Drop` for `RingBuffer` does nothing (leaks the elements) | **caught** by the drop-count test |
| M5 | iterator ignores `head` | **caught** |
| M6 | `push_str` accepts one byte too many | **caught** (it would write out of bounds; the test failed with a panic) |
| M7 | CRC tests the wrong bit (`0x4000`) | **caught** by the check value |
| M8 | `decode` accepts trailing bytes (`consumed > len`) | **caught** |
| M9 | `encode` accepts an output buffer one byte too small | **caught** (panic on the out-of-range slice) |
| M10 | CRC computed without the length byte | **caught** |
| M11 | start-byte check inverted | **caught** |
| M12 | `set_bits` no longer rejects a field with too many bits | **caught** |
| M13 | 32-bit mask is `u32::MAX >> 1` | **caught** |
| M14 | `get_bits` bound check off by one | **caught** |
| M15 | `Reg32::read` ANDs the value with `!0` | **survived**: `x & !0 == x`, an **equivalent mutant** (my own poor mutation, not a test gap) |
| M16 | `modify` forgets to write back | **caught** |

Totals: 16 mutants, 15 caught, 1 equivalent. M4 is only "caught" because the test counts drops with `Rc`; a plain leak is invisible to assertions, which is why the valgrind run is part of the pipeline (Lesson 37's `Counter` leak mutant was the same story). M6 and M9 are mutants that are themselves out-of-bounds writes; "the test failed" there came from Rust's own slice bounds checks panicking, which is a safe-code backstop that a `get_unchecked` version would not have.

**Design notes.**

- The crate is `#![no_std]` unconditionally, so `std` cannot be used even by accident; the tests link `std` only because the test harness does.
- `RingBuffer` stores `[MaybeUninit<T>; N]`, `head` and `len`. Invariant: the `len` slots starting at `head` (wrapping) are initialized, all others are not. `pop` uses `assume_init_read` (moves the value out, then updates the indices), `Drop` pops everything (since `MaybeUninit` never drops).
- `FixedString` keeps UTF-8 validity by appending only whole `&str`s, which justifies the one `from_utf8_unchecked`.
- Decoding checks `get(payload_end..total)` before any indexing, so a lying length byte yields `Truncated`, never a panic ✓ (test `frame_decode_errors`).
- `Reg32::new` is the single `unsafe` entry point; `read`/`write` are safe methods whose `SAFETY` comments point at it. The registers' semantics (read side effects, write-one-to-clear) are device-specific and not modeled.
- Not handled: a lock-free SPSC split of the ring buffer, interrupt safety of `Reg32::modify`, endianness-generic frames, escape/byte-stuffing for a byte stream (a real serial protocol needs it, since `0x7E` can appear in the payload), any real hardware, and CRC speed (the bitwise loop is simple, not fast; table-driven is the usual optimization and I did not measure either).

</details>

---

## 15. Key takeaways

- `core` needs nothing, `alloc` needs a heap, `std` needs an OS. Prefer `core`/`alloc` paths in library code (✓ clippy enforces it; `core::error::Error` exists).
- `#![no_std]` removes `std`, not the heap: `extern crate alloc;` brings `Vec`/`String` back, and `HashMap` stays out (✓ `E0432`).
- A `no_std` binary must supply a panic handler (✓ compile error otherwise), an entry point, `panic = "abort"`, and maybe an allocator and `mem*` routines (✓ demonstrated on x86-64 Linux with no libc: 9,864 bytes stripped).
- Without a heap, design for fixed capacity and fallible operations; `core::fmt::Write` formats into your own buffer (✓).
- Device registers need volatile access; a `&u8` or a plain load is allowed to be read once (✓ the assembly), and volatile is neither atomic nor ordering (Lessons 28 and 36).
- Features must be additive; build and test with and without them (✓).
- Miri is the tool for detecting UB in the `unsafe` parts that are pure Rust; it is partial evidence, slow, and cannot run hardware or foreign code. I could not run it here, so use the Miri README as the authority.
- Tests, valgrind and review catch different things (✓ M4: leak invisible to assertions).

**Connections.** [Lesson 37](37_maybeuninit_ffi.md) supplies `MaybeUninit` and the drop-guard thinking used by `RingBuffer`; [Lesson 36](36_unsafe_raw_pointers.md) supplies volatile, `SAFETY` comments and the UB catalogue Miri targets; [Lesson 28](28_atomics_memory_ordering.md) is how interrupts and main code share flags; [Lesson 19](19_generics_and_bounds.md) covers const generics behind fixed capacity; [Lesson 15](15_newtype_typestate.md) is the pattern behind HAL pins; [Lesson 33](33_modules_api_design_rustdoc.md) covers features and docs; [Lesson 26](26_error_design.md) shaped `FrameError`. Next: Lesson 39 covers unit and integration tests and doc tests.

---

## 16. References

- The Embedded Rust Book (`no_std`, targets, memory-mapped registers, interrupts, PACs and HALs): <https://docs.rust-embedded.org/book/>
- The Embedonomicon (writing `#![no_std]` programs from scratch: panic handler, entry point, linker scripts): <https://docs.rust-embedded.org/embedonomicon/>
- The Rust Reference, "The `no_std` attribute", "Panic handler" and "Global allocator" sections: <https://doc.rust-lang.org/reference/>
- `core`, `alloc` and `std` crate documentation, especially `core::fmt::Write`, `core::ptr::read_volatile` and `core::error::Error`: <https://doc.rust-lang.org/core/>, <https://doc.rust-lang.org/alloc/>
- Miri README (installation, `MIRIFLAGS`, supported and unsupported features, Stacked/Tree Borrows): <https://github.com/rust-lang/miri>
- The Rustonomicon, "Implementing Vec" and "Unwinding": <https://doc.rust-lang.org/nomicon/>
- `rustup` documentation for adding targets and components: <https://rust-lang.github.io/rustup/>
- Clippy lint list (`std_instead_of_core`, `alloc_instead_of_core`, `undocumented_unsafe_blocks`): <https://rust-lang.github.io/rust-clippy/master/>

The URLs above are the standard documentation locations; I did not open them during this session (the sandbox cannot reach the documentation sites), so if a link has moved, search for the page title. Unchecked or from memory: **all of the Miri material in §8** (installation, flags, the list of detected errors, its limitations and the aliasing models) because Miri is not installed here; all embedded-hardware details in §7 (target triples, `cortex-m-rt`, `memory.x`, `svd2rust`, PAC/HAL/BSP layering, `embedded-hal`, Embassy, `critical-section`, `heapless`, `embedded-alloc`, `probe-rs`, atomic availability on small cores) because I could not install a target or run an emulator; that `core::error::Error` stabilized in 1.81; the Substrate/Solana `no_std` statements in §1; the claim that `compiler_builtins` provides `mem*` on bare-metal targets; the list of what `core` and `alloc` contain (I verified `Vec`, `println!` and `HashMap` are missing and `BTreeMap`, `Box`, `format!`, `String` are present); `HashMap`'s hasher reason; third-party tools (`cargo bloat`, `cargo-fuzz`, `loom`, sanitizers); and any statement about `loom`. The freestanding Linux demos are an unsupported-looking setup chosen only because no embedded target was available; real firmware uses a startup crate and linker script. Exercise answers 5, 6 and 8 combine ✓-run facts with reasoning from memory. Everything marked ✓ was run on this machine's `rustc 1.97.0` (x86-64 Linux), with `valgrind` as installed in the sandbox.
