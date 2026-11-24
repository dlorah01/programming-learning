---
layout: default
title: "Lesson 28 — Atomics and memory ordering"
---

# Lesson 28 — Atomics and memory ordering

**Module 10 · Concurrency and async** · Status: generated · Edition used in examples: **2024** · External dependencies: none in the lesson text; the practical crate uses `loom` 0.7.2 (✓ downloaded and run) only under `--cfg loom`

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux with 2 cores. As before, ✓ marks outputs, diagnostics and measurements I actually ran; unmarked statements are documented behavior I did not run. Everything is **stable Rust**. Two honest limits shape this lesson:

1. **x86-64 is a strong memory model.** Many ordering bugs *cannot be observed* on this machine even though they are bugs. So I do not claim "I ran it and it worked" for weak orderings. Instead I use **loom**, a model checker that explores thread interleavings and flags orderings that are too weak, and I show the x86-64 assembly each ordering compiles to.
2. **Miri is not installed here** (`cargo miri` is unavailable on stable in this sandbox), so nothing below was checked with Miri.

Builds on [Lesson 22](22_gats_send_sync_phantom.md) (`Send`/`Sync`), [Lesson 24](24_interior_mutability.md) (`UnsafeCell`, `Mutex`) and [Lesson 27](27_threads_channels.md) (threads, `scope`). Locks, `Condvar` and deadlocks are Lesson 29; raw pointers and writing `unsafe` properly are Lesson 36.

---

## 1. Why atomics exist

A plain `usize` shared between threads cannot be written by two threads at once in safe Rust, because the borrow rules forbid it. Concurrent unsynchronized access with at least one write is a **data race**, which is undefined behavior. `Mutex` (Lesson 24) fixes this by making threads take turns. An **atomic** is the other tool: a type whose individual operations are indivisible and which the CPU and compiler promise not to tear, reorder arbitrarily, or optimize away.

Atomics live in `std::sync::atomic`: `AtomicBool`, `AtomicI8`…`AtomicI64`, `AtomicIsize`, `AtomicU8`…`AtomicU64`, `AtomicUsize`, and `AtomicPtr<T>`. (`AtomicI64`/`AtomicU64` exist on targets that have 64-bit atomics; that is most, but not all, targets. From memory.) They are `Sync`, so they can be shared by `&` reference, in an `Arc`, or in a `static` (✓ `AtomicUsize::new` is `const`). You mutate them through `&self`, which is *interior mutability* done in hardware.

Sizes ✓ (x86-64): `AtomicBool` 1 byte, `AtomicI32` 4, `AtomicU64` 8, `AtomicUsize` 8; `AtomicU64` has alignment 8.

### 1.1 Atomic read-modify-write versus load-then-store

The most important distinction in the lesson. Compare two ways to count:

```rust
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::thread;
static HITS: AtomicUsize = AtomicUsize::new(0);
static LOST: AtomicUsize = AtomicUsize::new(0);
fn main() {
    thread::scope(|s| {
        for _ in 0..8 {
            s.spawn(|| {
                for _ in 0..10_000 {
                    HITS.fetch_add(1, Relaxed);          // one indivisible step
                    let v = LOST.load(Relaxed);          // two steps: another thread can
                    LOST.store(v + 1, Relaxed);          // slip in between them
                }
            });
        }
    });
    println!("{}", HITS.load(Relaxed));
    println!("{}", LOST.load(Relaxed) <= 80_000);
}
```

✓ `HITS` is always `80000`. The load/store version is memory-safe (every individual access is atomic) but **logically wrong**: two threads read the same `v`, both write `v + 1`, one increment is lost. ✓ In four runs the loser counter ended below 80000 every time; the exact value varies from run to run, which is why I print only comparisons.

> Atomic does not mean "this *sequence* of operations is atomic". Each call is atomic; your *algorithm* is only correct if the whole decision is one atomic call (or a retry loop around one, §4).

---

## 2. The API tour

```rust
use std::sync::atomic::{AtomicBool, AtomicI32, AtomicU64, AtomicUsize, Ordering::*};
fn main() {
    let a = AtomicUsize::new(5);
    println!("{} {}", a.fetch_add(3, Relaxed), a.load(Relaxed));            // 5 8
    println!("{} {}", a.fetch_sub(1, Relaxed), a.swap(100, SeqCst));        // 8 7
    println!("{:?}", a.compare_exchange(100, 7, AcqRel, Acquire));          // Ok(100)
    println!("{:?}", a.compare_exchange(100, 8, AcqRel, Acquire));          // Err(7)
    println!("{} {}", a.fetch_max(50, Relaxed), a.fetch_min(10, Relaxed));  // 7 50
    println!("{:?}", a.fetch_update(SeqCst, SeqCst, |v| if v < 100 { Some(v * 2) } else { None })); // Ok(10)
    println!("{:?}", a.fetch_update(SeqCst, SeqCst, |v| if v < 5 { Some(v) } else { None }));       // Err(20)
    println!("{}", a.load(SeqCst));                                          // 20
    let w = AtomicU64::new(u64::MAX);
    println!("{} {}", w.fetch_add(2, Relaxed), w.load(Relaxed));            // 18446744073709551615 1
    let b = AtomicBool::new(false);
    println!("{} {} {}", b.fetch_or(true, Relaxed), b.fetch_and(false, Relaxed), b.fetch_xor(true, Relaxed)); // false true false
    println!("{:?}", b);                                                     // true
    let mut m = AtomicI32::new(1);
    *m.get_mut() += 1;                                                       // &mut self: no atomics needed
    println!("{}", m.into_inner());                                          // 2
}
```

✓ Every value in the comments is the real output. Points to take from it:

- **`fetch_*` and `swap` return the *previous* value**, not the new one (`fetch_add(3)` on 5 returned 5).
- **Arithmetic wraps** (✓ `u64::MAX + 2` gives a stored `1`, no panic, even in debug builds). Overflow checks do not apply to atomics. That is a real hazard for counters that must not wrap (§8) and for `fetch_sub` below zero.
- **`compare_exchange(current, new, success, failure)`** writes `new` only if the value equals `current`, returning `Ok(previous)` or `Err(actual)`. `compare_exchange_weak` may fail spuriously even when equal (it can be cheaper on some architectures), so it belongs inside a loop.
- **`fetch_update(set_order, fetch_order, f)`** is a ready-made compare-exchange loop: it calls `f` on the current value, and if `f` returns `Some(new)` tries to install it, retrying if another thread changed the value meanwhile. `f` may therefore run **several times**, so it must be free of side effects. `Err(previous)` means `f` returned `None` (✓ `Err(20)`).
- **`get_mut` and `into_inner`** need `&mut self`/`self`, which prove nobody else holds a reference, so they are plain non-atomic accesses.

---

## 3. Memory ordering

### 3.1 The problem orderings solve

Atomicity gives you *one variable*. Real protocols involve *two*: write some data, then set a flag; the reader checks the flag, then reads the data. Compilers reorder instructions and CPUs reorder memory operations (store buffers, caches) as long as a **single** thread cannot tell. Another thread can tell. An **ordering** is your instruction to the compiler and CPU about how much reordering around *this* atomic operation is forbidden.

The Rust model is the C++20 memory model (from memory; the Rust reference says Rust follows the C++20 atomics model, minus `consume`). The core idea is **happens-before**: if operation A happens-before B, then everything A wrote is visible to B. Without such an edge, a read may see an old value, and a non-atomic read of data written by another thread is a data race, hence UB.

### 3.2 The five orderings

| Ordering | Allowed on | Guarantee (informal) |
|---|---|---|
| `Relaxed` | load, store, RMW | Atomicity only. No ordering with respect to other memory. All threads agree on the order of operations on *this one variable*. |
| `Acquire` | load, RMW | Nothing after this load in program order can be moved before it. If it reads a value written by a `Release` store, everything before that store becomes visible. |
| `Release` | store, RMW | Nothing before this store can be moved after it. Pairs with an `Acquire` load of the same variable. |
| `AcqRel` | RMW | Both, for a read-modify-write that both reads and writes. |
| `SeqCst` | all | Acquire/Release **plus** a single total order of all `SeqCst` operations that every thread agrees on. |

Release **publishes**, Acquire **receives**. The pairing is the whole mechanism: a `Release` store only helps a reader that does an `Acquire` (or stronger) load *of that same atomic* and observes the stored value.

The standard use, message passing:

```rust
use std::sync::atomic::{AtomicBool, AtomicUsize, Ordering};
use std::thread;
static DATA: AtomicUsize = AtomicUsize::new(0);
static READY: AtomicBool = AtomicBool::new(false);
fn main() {
    let t = thread::spawn(|| {
        DATA.store(42, Ordering::Relaxed);     // 1. prepare
        READY.store(true, Ordering::Release);  // 2. publish
    });
    while !READY.load(Ordering::Acquire) {     // 3. wait for it
        std::hint::spin_loop();
    }
    assert_eq!(DATA.load(Ordering::Relaxed), 42); // 4. guaranteed to see 42
    t.join().unwrap();
}
```

Because step 2 is `Release` and step 3 is an `Acquire` that sees `true`, step 1 happens-before step 4. (Here `DATA` is itself atomic, so a stale read would not be UB, but it would be a *wrong answer*; with a non-atomic `DATA` it would be UB. The ordering guarantee is the same.) This program spins; spinning is discussed in §6.

### 3.3 Invalid combinations are rejected

Some orderings make no sense on some operations (a load writes nothing, so "release" is meaningless). Rust catches the obvious cases with a **deny-by-default lint**, `invalid_atomic_ordering`, not with an error code:

```rust,ignore
a.load(Ordering::Release);                                         // ✓ error
a.store(1, Ordering::Acquire);                                     // ✓ error
a.compare_exchange(0, 1, Ordering::Relaxed, Ordering::Release);    // ✓ error
```

✓ The messages: "atomic loads cannot have `Release` or `AcqRel` ordering"; "atomic stores cannot have `Acquire` or `AcqRel` ordering"; "`compare_exchange`'s failure ordering may not be `Release` or `AcqRel`, since a failed `compare_exchange` does not result in a write"; each with the note that `#[deny(invalid_atomic_ordering)]` is on by default. The failure ordering of a compare-exchange is the ordering of the *load* that happens when the exchange fails. (Older Rust versions required the failure ordering to be no stronger than the success ordering; I did not test whether that rule still applies on 1.97, so choose failure `Relaxed` or `Acquire` as the examples do and you avoid the question.)

### 3.4 Fences

`std::sync::atomic::fence(order)` is an ordering with no memory location attached. A `Release` fence before a `Relaxed` store, plus an `Acquire` fence after a `Relaxed` load that sees it, works like Release/Acquire on the variable. Fences are rarely needed in application code; prefer ordering the atomic operation itself.

---

## 4. What the orderings cost, and why you cannot test them here

### 4.1 x86-64 assembly ✓

I compiled one tiny function per operation with `rustc -O --emit asm` and read the output:

| Operation | x86-64 instruction ✓ |
|---|---|
| `load` with `Relaxed`, `Acquire` or `SeqCst` | `movq (%rdi), %rax`, the same plain load |
| `store` with `Relaxed` or `Release` | `movq %rsi, (%rdi)`, a plain store |
| `store` with `SeqCst` | `xchgq %rsi, (%rdi)`, an atomic exchange, which acts as a full barrier |
| `fetch_add` with `Relaxed` **and** `SeqCst` | `lock xaddq`, the same instruction |
| `compare_exchange(…, AcqRel, Acquire)` | `lock cmpxchgq` then `sete` |
| `swap` | `xchgq` |
| `fence(SeqCst)` | `lock orl $0, -64(%rsp)` |
| `fence(Acquire)` | nothing (just `retq`) |
| plain `*a += 1` on `&mut u64` | `incq (%rdi)`, not atomic |

Read this table carefully, because it teaches a trap. On x86-64 `Relaxed`, `Acquire` and `Release` loads and stores are all the **same plain instructions**, and read-modify-writes are already full barriers. That is x86's strong ("total store order") model. The *hardware* hides the difference, so a program with `Relaxed` where `Release` was needed will usually **appear to work on x86**. The ordering still matters in two ways: the **compiler** may reorder or optimize based on it, and on weaker CPUs (ARM, RISC-V, POWER; from memory) `Acquire`/`Release` do compile to different instructions and missing ones really do fail. Only the `SeqCst` store is visibly more expensive here (an `xchg` instead of a `mov`).

I am showing x86-64 only; I did not inspect any other architecture's output.

### 4.2 Checking orderings with loom ✓

A model checker can do what testing on x86 cannot. **loom** (a crate by the Tokio project) re-implements `std`'s atomics, `UnsafeCell`, `Arc`, `thread`, `Mutex` and friends; `loom::model(|| …)` runs your closure many times, exploring the allowed interleavings of threads and the allowed visibility of values according to the memory model, and fails the test if an assertion breaks or if it detects a data race on a `loom::cell::UnsafeCell`.

I wrote the message-passing protocol of §3.2 with a *non-atomic* payload and varied the orderings (a separate probe test file, not part of the `atomlab` crate; the shape is shown here):

```rust,ignore
struct Shared { data: UnsafeCell<usize>, ready: AtomicBool }   // loom's types
unsafe impl Sync for Shared {}

fn message_passing(store: Ordering, load: Ordering) {
    loom::model(move || {
        let s = Arc::new(Shared { data: UnsafeCell::new(0), ready: AtomicBool::new(false) });
        let s2 = s.clone();
        let t = thread::spawn(move || {
            s2.data.with_mut(|p| unsafe { *p = 42 });   // write the payload
            s2.ready.store(true, store);                // publish
        });
        if s.ready.load(load) {
            let v = s.data.with(|p| unsafe { *p });     // read the payload
            assert_eq!(v, 42);
        }
        t.join().unwrap();
    });
}
```

Results ✓ (`loom` 0.7.2, `RUSTFLAGS="--cfg loom" cargo test --release`):

| store / load | loom result ✓ |
|---|---|
| `Release` / `Acquire` | **passes** |
| `Relaxed` / `Relaxed` | **fails**: "Causality violation: Concurrent read and write accesses." |
| `Release` / `Relaxed` | **fails**, same message |
| `Relaxed` / `Acquire` | **fails**, same message |

Both halves are needed, which is the "pairing" rule in action. The same harness with two threads doing `load` then `store(v + 1)` ✓ **fails** the final `assert_eq!(count, 2)`, while `fetch_add` ✓ passes.

Limits of this evidence: loom checks *your model of the program* with its own implementation of the memory model; it is a bug-finder, not a proof, and, from memory, its documentation lists some behaviors it does not model (and it bounds the number of threads and the amount of exploration). A loom pass means "no violation found in the explored executions", not "correct on every CPU".

---

## 5. Choosing an ordering

Work from the *protocol*, not from the operation:

1. **Is the atomic the only shared data, and are you just counting or collecting statistics?** `Relaxed`. A hit counter, an id generator, a metrics gauge: each value is meaningful by itself and nothing else is published through it.
2. **Does the atomic announce that *other* memory is ready (a flag, a pointer, a state)?** The writer does `Release`, the reader does `Acquire`. Use `AcqRel` on read-modify-writes that do both.
3. **Do you need all threads to agree on one global order of several different atomics?** That is the only thing `SeqCst` adds. It is needed by algorithms like Dekker/Peterson mutual exclusion and in rare multi-variable protocols.
4. **Unsure?** Use `SeqCst`. It is never *incorrect* when a weaker ordering would be correct; it can only be slower, and on x86-64 the difference is mostly the `SeqCst` store (§4.1). Premature weakening is how subtle bugs ship; measure (Lesson 41) before you relax anything that works.

### 5.1 Patterns

**Counter / id generator** (`Relaxed`, §1.1). The returned values are unique because `fetch_add` is atomic; no other memory depends on them.

**Stop flag**: `AtomicBool` that workers poll. `Relaxed` is enough if the flag is only a request to stop; use `Release`/`Acquire` if workers must see state written before the flag was set.

**Compare-and-swap loop** for "check, then change" decisions: take a permit only if fewer than N exist (§7's `Limiter`). The decision and the update are one atomic step, and the retry happens when another thread got there first.

**Lock**: `compare_exchange(false, true, Acquire, Relaxed)` to lock, `store(false, Release)` to unlock. Everything the previous holder wrote under the lock happens-before what the next holder reads. This is exactly what `Mutex` does internally (Lesson 29, with the OS parking threads instead of spinning).

**Publication**: a slot written once then read by many (`OnceSlot` below; `std::sync::OnceLock` is the production version, Lesson 24).

---

## 6. Spinning, the ABA problem, and other edge cases

- **Spin loops** (`while !flag.load(Acquire) { … }`) burn a core. `std::hint::spin_loop()` tells the CPU it is in a spin-wait (it lowers power and helps hyper-threads; it does **not** yield to the OS scheduler). `thread::yield_now()` asks the scheduler to run something else. For anything that may wait more than a few hundred nanoseconds, use a `Mutex`/`Condvar`/channel (Lessons 27 and 29) which put the thread to sleep. A spin lock held by a thread the OS has descheduled makes every waiter spin for a full time slice.
- **Priority inversion and starvation**: a spin lock gives no fairness. A low-priority holder can be preempted while a high-priority thread spins on it. Do not use spin locks in code that may be preempted while holding the lock unless you have thought this through (it is common in kernels, which control preemption; in user space it is usually a mistake).
- **The ABA problem**: a compare-exchange checks "is the value still `A`?" but cannot tell if it changed `A → B → A` in between. For counters that is harmless; for lock-free linked structures (a stack where `A` is a node pointer that was popped, freed, and reallocated) it is a classic source of memory corruption. Real lock-free structures need tagged pointers, hazard pointers, or epoch-based reclamation (the `crossbeam-epoch` crate does the latter; from memory). **Do not write lock-free data structures with raw pointers until you have read Lessons 36–38.**
- **Atomic ≠ lock-free ≠ wait-free**: `AtomicU64` operations are lock-free on the targets that support them; on a target without a 64-bit atomic instruction the type may not exist, and `is_lock_free`-style guarantees are platform specific (from memory).
- **Alignment and `AtomicPtr`**: `AtomicPtr<T>` stores a raw pointer; dereferencing it afterwards is `unsafe` and needs everything from Lesson 36 (who frees it? is it still valid?).
- **False sharing**: two unrelated atomics on the same cache line make threads fight over it, even though they never touch the same variable. Padding to a cache line fixes it. I do not quote numbers; Lesson 41 teaches how to measure it.
- **Atomics do not give you a consistent view of several values.** Reading `a` then `b` can see an old `b` with a new `a`. If the invariant spans two values, put both under a `Mutex` or pack them into one atomic.

---

## 7. A tour of idiomatic uses, in the practical crate

The crate at the end of this lesson contains four small primitives. Their key lines, with the orderings spelled out:

| Primitive | Key operations | Why those orderings |
|---|---|---|
| `IdGen` | `fetch_add(1, Relaxed)` | Only uniqueness matters; no other memory is published. |
| `Limiter` | `fetch_update(Acquire, Relaxed, …)` to take; `fetch_sub(1, Release)` in `Drop` | One atomic "check then increment"; the release on drop pairs with the next taker's acquire. |
| `SpinLock<T>` | CAS `Acquire` to lock; `store(false, Release)` to unlock | The lock **is** the happens-before edge between critical sections. |
| `OnceSlot<T>` | CAS `EMPTY→WRITING` (`Acquire`); write; `store(READY, Release)`; readers `load(Acquire)` | Publish the value's bytes through the state variable. |

`SpinLock` and `OnceSlot` contain `unsafe` and `unsafe impl Sync`. That is the real shape of the obligation (Lesson 22 told you `Sync` is a promise): *the type's own protocol must prevent data races*. Every `unsafe` block carries a `// SAFETY:` comment saying why. The compiler cannot check that comment; **loom and review are the checks**.

---

## 8. Common mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| `load` then `store(v + 1)` | Lost updates (✓ §1.1, ✓ loom) | `fetch_add`, or a `compare_exchange` loop |
| "Check then act" on two calls: `if x.load() > 0 { x.fetch_sub(1) }` | Both threads pass the check; the counter underflows and wraps (✓ debugging challenge) | `fetch_update` with `checked_sub` |
| Publishing data with `Relaxed` | Reader may see the flag but not the data: works on x86, fails on ARM, and loom ✓ shows the data race | `Release` store + `Acquire` load |
| Using `Release` on the writer but `Relaxed` on the reader (or the reverse) | No happens-before edge (✓ loom fails both) | Pair them |
| Reaching for `SeqCst` everywhere "to be safe", then believing it fixes logic bugs | It orders memory, it does not make *sequences* atomic | Fix the algorithm first |
| Counters that must not wrap, using `fetch_add`/`fetch_sub` | Silent wrap (✓ `u64::MAX + 2` stored `1`) | `fetch_update` with `checked_add`/`checked_sub` |
| Building a spin lock for long critical sections | Wasted CPU, priority inversion | `Mutex` |
| `unsafe impl Sync` without a written argument | Data race through safe code | A `// SAFETY:` comment you can defend, plus loom |
| Trusting a test that passes on x86 | Weak-memory bugs hide | Reason from happens-before; use loom |

---

## 9. Performance implications

- A relaxed `fetch_add` is a `lock`-prefixed instruction (✓ `lock xaddq`): much cheaper than a mutex under light contention, but **not free**, because it takes exclusive ownership of the cache line. Under heavy contention many threads incrementing one atomic serialize on that line. Sharding (one counter per thread, summed on read) is the usual fix (from memory; measure first, Lesson 41).
- On x86-64 the only ordering cost visible in the assembly is `SeqCst` stores (✓ `xchg` vs `mov`). On weaker architectures `Acquire`/`Release`/`SeqCst` add barrier instructions (from memory; not inspected here).
- Atomic operations also constrain the *compiler*: a `Relaxed` operation is still not elided or merged arbitrarily.
- I give no timing numbers in this lesson; a number measured on this 2-core sandbox would not transfer to your machine. Lesson 41 shows how to benchmark correctly.

---

## 10. Security and reliability

- **Memory safety is the part Rust does check; protocol correctness is not.** A `Relaxed` publication bug in `unsafe` code is a data race, i.e. undefined behavior, and its symptoms (corrupted state, wrong balances) can be exploitable. Rust does not make such code secure.
- **Check-then-act on shared counters is a classic authorization/quota bug**: "limit 3 per user" implemented as `if count.load() < 3 { count.fetch_add(1) }` allows more than 3 under concurrency (✓ the same shape as the debugging challenge). Use one atomic decision.
- **Wrapping counters** used for ids, nonces or sequence numbers repeat after 2^64 calls (practically never) or, with `u8`/`u16`/`u32`, much sooner. Decide whether repetition is acceptable; do not use `Relaxed` counters as *security tokens* (they are guessable; use a CSPRNG, Lesson 48).
- **Blockchain-adjacent note**: validators, mempools and RPC servers share counters and caches across threads. Rate limiters, nonce allocators and "already seen" sets are exactly the places where a lost update or a double-spend-style race lives. Atomics are the right tool for a counter, wrong for anything with an invariant across more than one value.
- **Availability**: spin loops without a bound turn a stuck thread into a stuck core. Prefer timeouts and parked waiting.
- **Never claim "lock-free, therefore safe"**: lock-free structures trade deadlock for subtler bugs (ABA, memory reclamation). Use well-tested crates (`crossbeam`, `std`) unless you have a specific reason.

---

## 11. Exercises

Answers are collapsed. Try each before opening it.

1. **(Warm-up)** What does `a.fetch_add(3, Relaxed)` return when `a` holds 5? What does `a.load(Relaxed)` return afterwards?
2. **(Warm-up)** What is the difference between `a.swap(v, o)` and `a.store(v, o)`?
3. **(Easy)** Why is `static COUNT: AtomicUsize = AtomicUsize::new(0);` allowed, while `static mut COUNT: usize = 0;` needs `unsafe` to touch?
4. **(Medium)** A counter is incremented by 8 threads with `let v = c.load(SeqCst); c.store(v + 1, SeqCst);`. `SeqCst` is the strongest ordering. Is the result correct? Why or why not?
5. **(Medium)** Write `fn dec_if_positive(c: &AtomicUsize) -> bool` that decrements `c` only if it is above zero, correctly under concurrency, without a mutex.
6. **(Medium-hard)** Which ordering(s) must change if the reader in §3.2 used `READY.load(Relaxed)` and you want the program to stay correct? Which side would you change, and to what?
7. **(Hard)** Why is the unlock in a spin lock a `store(false, Release)` rather than `Relaxed`, and why is spinning on a `Relaxed` load *inside* the retry loop still fine?
8. **(Hard)** `fetch_update` may call its closure more than once. Give an example closure that would be a bug for that reason, and say how to fix it.

<details markdown="1">
<summary>Answers</summary>

1. It returns `5` (the previous value); the next `load` returns `8`. ✓ (§2)
2. Both write the new value. `swap` also **returns the previous value** and is a read-modify-write (so it can use `Acquire`/`AcqRel`); `store` returns nothing and is a pure write (so `Acquire`/`AcqRel` are invalid for it).
3. A `static` requires `Sync`, and `AtomicUsize` is `Sync` because every access is atomic and goes through `&self`. A `static mut` has no such guarantee, so the compiler cannot prove accesses are race-free and requires `unsafe` (and in edition 2024 even taking a reference to a `static mut` is denied by default, from memory).
4. **Not correct.** `SeqCst` orders the *accesses*, it does not make the pair of accesses one step. Two threads can both load the same `v`. ✓ loom fails this exactly (§4.2) and the §1.1 program loses updates. Use `fetch_add`.
5. ```rust
   use std::sync::atomic::{AtomicUsize, Ordering};
   fn dec_if_positive(c: &AtomicUsize) -> bool {
       c.fetch_update(Ordering::AcqRel, Ordering::Relaxed, |n| n.checked_sub(1)).is_ok()
   }
   ```
   `checked_sub` returns `None` at zero, so `fetch_update` returns `Err` without writing. ✓ the same pattern was run in the debugging challenge (`Ok(1000)`, and `Err(0)` leaving `0`).
6. The **reader**. A `Relaxed` load gives no happens-before edge with the writer's `Release` store, so change the reader to `Acquire` (✓ loom's `Release/Relaxed` row fails). Changing only the writer cannot help; changing the writer to `SeqCst` would also work if the reader were `SeqCst`, but `Release`/`Acquire` is the minimal pair.
7. Unlock must make the critical section's writes visible to the next holder: the next holder's `Acquire` CAS pairs with this `Release` store (✓ in the practical crate, weakening it to `Relaxed` fails loom with "Concurrent write accesses to `UnsafeCell`"). The inner spin can be `Relaxed` because it is only a *hint* to retry: nothing is read from protected memory until the successful `Acquire` CAS. Its purpose is to avoid hammering the cache line with failing read-modify-writes.
8. A closure with a side effect, e.g. `|n| { log.push(n); Some(n + 1) }`, or `|n| { id_counter.fetch_add(1, …); … }`: a retry runs it again, so the log gets duplicates and the other counter over-counts. Fix: make the closure a pure function of `n`; do side effects once, after `fetch_update` has returned `Ok(prev)`.

</details>

---

## 12. Debugging challenge

This ticket seller "never oversells", the author says. Tickets are limited to 200 000, 8 threads each try 200 000 times.

```rust
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::thread;

struct Tickets {
    left: AtomicUsize,
}

impl Tickets {
    fn try_take(&self) -> bool {
        if self.left.load(Relaxed) > 0 {
            self.left.fetch_sub(1, Relaxed);
            true
        } else {
            false
        }
    }
}

fn main() {
    let t = Tickets { left: AtomicUsize::new(200_000) };
    let sold = AtomicUsize::new(0);
    thread::scope(|s| {
        for _ in 0..8 {
            s.spawn(|| {
                for _ in 0..200_000 {
                    if t.try_take() {
                        sold.fetch_add(1, Relaxed);
                    }
                }
            });
        }
    });
    println!("sold {} (only 200000 existed), left {}", sold.load(Relaxed), t.left.load(Relaxed));
}
```

✓ I compiled this with `-O` and ran it six times on 2 cores. Four runs printed `sold 200000 … left 0`; two did not: one printed `sold 511939 … left 18446744073709239677` and another `sold 1600000 … left 18446744073708151616`. The bug is a race, so it is intermittent: **four clean runs prove nothing**.

Find the bug, explain the huge `left` value, and fix it.

<details markdown="1">
<summary>Solution</summary>

`try_take` is *check then act*: `load() > 0` and `fetch_sub(1)` are two separate atomic operations. Near the end two threads can both see `left == 1`, both pass the check, and both subtract. `left` goes from 1 to 0 to `usize::MAX` (✓ atomics wrap: `fetch_sub(1)` on 0 stored `usize::MAX`), which is greater than zero, so **every later call passes the check too** and "sells" a ticket. That explains both `sold` above 200 000 and the 1.8×10¹⁹ `left`.

Fix: make the decision and the update one atomic step.

```rust
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
struct Tickets { left: AtomicUsize }
impl Tickets {
    fn try_take(&self) -> bool {
        self.left
            .fetch_update(Relaxed, Relaxed, |n| n.checked_sub(1))
            .is_ok()
    }
}
fn main() {
    let t = Tickets { left: AtomicUsize::new(1) };
    println!("{} {}", t.try_take(), t.try_take());
}
```

✓ `fetch_update` with `checked_sub` returned `Ok(1000)` leaving `999`, and `Err(0)` leaving `0` (no wrap). `Relaxed` is fine here because nothing else is published through the counter; if taking a ticket must also *hand over* data, use `Acquire`/`Release`. The lesson: the problem was never the ordering, it was the two-step algorithm. This is the same shape as the crate's `Limiter`, and loom ✓ catches it when the CAS loop is replaced by load/store (practical challenge, mutation M5).

</details>

---

## 13. Practical challenge: `atomlab`

Build a small crate with four primitives, then **prove the orderings with loom**.

1. `IdGen`: `next(&self) -> u64`, unique across threads, wrapping at `u64::MAX`; usable in a `static`.
2. `Limiter`: `try_acquire(&self) -> Option<Permit<'_>>` with at most `max` permits at once, never overshooting; dropping a permit frees one slot.
3. `SpinLock<T>`: `lock`, `try_lock`, a guard with `Deref`/`DerefMut`, unlock on drop (including on panic); `Sync` only when `T: Send`.
4. `OnceSlot<T>`: `set(&self, T) -> Result<(), T>` (exactly one winner), `get(&self) -> Option<&T>`.

Requirements: the crate compiles for plain `std` and under `--cfg loom` with the same source; every `unsafe` has a `// SAFETY:` comment; plain tests with real threads; loom tests that would **fail** if an ordering were too weak.

Cargo notes ✓: loom must be a `[target.'cfg(loom)'.dependencies]` entry, **not** a dev-dependency. I first tried a dev-dependency and `RUSTFLAGS="--cfg loom" cargo test --test …` failed to compile the library (E0433, `loom` unresolved) because dev-dependencies are not available to the library target itself. Declare the `loom` cfg for the `unexpected_cfgs` lint with `[lints.rust] unexpected_cfgs = { level = "warn", check-cfg = ["cfg(loom)"] }`.

Run:

```text
cargo test                                                      # ordinary tests
RUSTFLAGS="--cfg loom" cargo test --release --lib               # loom model checking
```

<details markdown="1">
<summary>Solution (spoilers: full source)</summary>

`Cargo.toml`:

```toml
[package]
name = "atomlab"
version = "0.1.0"
edition = "2024"

[dependencies]

[target.'cfg(loom)'.dependencies]
loom = "0.7.2"

[lints.rust]
unexpected_cfgs = { level = "warn", check-cfg = ["cfg(loom)"] }
```

`src/lib.rs`:

```rust
//! `atomlab`: small lock-free building blocks with their memory orderings
//! spelled out and model-checked with `loom`.
//!
//! * [`IdGen`]: unique ids with `fetch_add(Relaxed)` (atomicity only).
//! * [`Limiter`]: at most N permits, via a compare-and-swap loop.
//! * [`SpinLock`]: mutual exclusion with Acquire/Release.
//! * [`OnceSlot`]: write-once publication of a value to other threads.
//!
//! Run the ordinary tests with `cargo test`, and the model checker with
//! `RUSTFLAGS="--cfg loom" cargo test --release --lib loom`.

use std::fmt;
use std::marker::PhantomData;
use std::ops::{Deref, DerefMut};

#[cfg(loom)]
use loom::{
    cell::UnsafeCell,
    sync::atomic::{AtomicBool, AtomicU8, AtomicU64, AtomicUsize, Ordering},
};
#[cfg(not(loom))]
use std::{
    cell::UnsafeCell,
    sync::atomic::{AtomicBool, AtomicU8, AtomicU64, AtomicUsize, Ordering},
};

/// Yields while spinning. Under `loom` the scheduler must be told to switch.
#[inline]
fn spin_hint() {
    #[cfg(loom)]
    loom::thread::yield_now();
    #[cfg(not(loom))]
    std::hint::spin_loop();
}

/// `UnsafeCell` wrapper with one API for both `std` and `loom`.
struct Slot<T>(UnsafeCell<T>);

impl<T> Slot<T> {
    fn new(value: T) -> Self {
        Slot(UnsafeCell::new(value))
    }

    fn with<R>(&self, f: impl FnOnce(*const T) -> R) -> R {
        #[cfg(loom)]
        {
            self.0.with(f)
        }
        #[cfg(not(loom))]
        {
            f(self.0.get())
        }
    }

    fn with_mut<R>(&self, f: impl FnOnce(*mut T) -> R) -> R {
        #[cfg(loom)]
        {
            self.0.with_mut(f)
        }
        #[cfg(not(loom))]
        {
            f(self.0.get())
        }
    }
}

// ------------------------------------------------------------------ IdGen

/// Hands out unique `u64` ids. The counter wraps after `u64::MAX`, so ids
/// repeat only after 2^64 calls.
pub struct IdGen {
    next: AtomicU64,
}

impl IdGen {
    /// Starts at 0.
    #[cfg(not(loom))]
    pub const fn new() -> Self {
        Self::starting_at(0)
    }

    #[cfg(loom)]
    pub fn new() -> Self {
        Self::starting_at(0)
    }

    #[cfg(not(loom))]
    pub const fn starting_at(first: u64) -> Self {
        Self {
            next: AtomicU64::new(first),
        }
    }

    #[cfg(loom)]
    pub fn starting_at(first: u64) -> Self {
        Self {
            next: AtomicU64::new(first),
        }
    }

    /// Returns a fresh id. `Relaxed` is enough: callers need each id to be
    /// distinct, and the counter itself orders nothing else.
    pub fn next(&self) -> u64 {
        self.next.fetch_add(1, Ordering::Relaxed)
    }
}

impl Default for IdGen {
    fn default() -> Self {
        Self::new()
    }
}

// ---------------------------------------------------------------- Limiter

/// Allows at most `max` permits to exist at once.
pub struct Limiter {
    in_use: AtomicUsize,
    max: usize,
}

/// A permit; dropping it frees one slot.
#[must_use = "dropping the permit immediately frees the slot"]
pub struct Permit<'a> {
    limiter: &'a Limiter,
}

impl Limiter {
    pub fn new(max: usize) -> Self {
        Self {
            in_use: AtomicUsize::new(0),
            max,
        }
    }

    /// Takes a permit if fewer than `max` exist. A compare-and-swap loop
    /// (`fetch_update`) makes "check then increment" one atomic step, so the
    /// count can never overshoot `max`, however many threads race.
    pub fn try_acquire(&self) -> Option<Permit<'_>> {
        self.in_use
            .fetch_update(Ordering::Acquire, Ordering::Relaxed, |n| {
                (n < self.max).then_some(n + 1)
            })
            .ok()
            .map(|_| Permit { limiter: self })
    }

    /// Permits currently held (a snapshot: it may change immediately).
    pub fn in_use(&self) -> usize {
        self.in_use.load(Ordering::Relaxed)
    }
}

impl Drop for Permit<'_> {
    fn drop(&mut self) {
        self.limiter.in_use.fetch_sub(1, Ordering::Release);
    }
}

// --------------------------------------------------------------- SpinLock

/// A minimal spin lock. It never sleeps: use it only for very short critical
/// sections, and never hold it across blocking calls. A panic while the guard
/// is alive releases the lock (there is no poisoning).
pub struct SpinLock<T> {
    locked: AtomicBool,
    value: Slot<T>,
}

// SAFETY: access to `value` is serialized by `locked`; handing out `&mut T`
// to whichever thread holds the lock moves `T` between threads, hence `T: Send`.
unsafe impl<T: Send> Sync for SpinLock<T> {}
// SAFETY: moving the lock moves the `T` inside it.
unsafe impl<T: Send> Send for SpinLock<T> {}

/// Proof of holding a [`SpinLock`]; unlocks on drop.
pub struct SpinGuard<'a, T> {
    lock: &'a SpinLock<T>,
    // The guard must not be sent to another thread in this simple design.
    _not_send: PhantomData<*mut ()>,
}

impl<T> SpinLock<T> {
    pub fn new(value: T) -> Self {
        Self {
            locked: AtomicBool::new(false),
            value: Slot::new(value),
        }
    }

    /// Spins until the lock is free. **Acquire** on success pairs with the
    /// **Release** in `unlock`, so everything the previous holder wrote
    /// is visible to the new holder.
    pub fn lock(&self) -> SpinGuard<'_, T> {
        while self
            .locked
            .compare_exchange_weak(false, true, Ordering::Acquire, Ordering::Relaxed)
            .is_err()
        {
            // Spin on a plain load (Relaxed) so waiting threads do not keep
            // fighting over the cache line with failed read-modify-writes.
            while self.locked.load(Ordering::Relaxed) {
                spin_hint();
            }
        }
        SpinGuard {
            lock: self,
            _not_send: PhantomData,
        }
    }

    /// Takes the lock if it is free right now.
    pub fn try_lock(&self) -> Option<SpinGuard<'_, T>> {
        self.locked
            .compare_exchange(false, true, Ordering::Acquire, Ordering::Relaxed)
            .ok()
            .map(|_| SpinGuard {
                lock: self,
                _not_send: PhantomData,
            })
    }
}

impl<T> Deref for SpinGuard<'_, T> {
    type Target = T;
    fn deref(&self) -> &T {
        // SAFETY: holding the guard means we hold the lock.
        self.lock.value.with(|p| unsafe { &*p })
    }
}

impl<T> DerefMut for SpinGuard<'_, T> {
    fn deref_mut(&mut self) -> &mut T {
        // SAFETY: holding the guard means we hold the lock exclusively.
        self.lock.value.with_mut(|p| unsafe { &mut *p })
    }
}

impl<T> Drop for SpinGuard<'_, T> {
    fn drop(&mut self) {
        self.lock.locked.store(false, Ordering::Release);
    }
}

impl<T> fmt::Debug for SpinLock<T> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_struct("SpinLock")
            .field("locked", &self.locked.load(Ordering::Relaxed))
            .finish_non_exhaustive()
    }
}

// --------------------------------------------------------------- OnceSlot

const EMPTY: u8 = 0;
const WRITING: u8 = 1;
const READY: u8 = 2;

/// A cell that can be set once, from any thread, and then read by everyone.
///
/// `set` publishes the value with a **Release** store; `get` observes it with
/// an **Acquire** load, which is what makes the value's bytes visible.
///
/// A `SpinLock<Rc<u8>>` is not `Sync`, because `Rc` is not `Send`:
///
/// ```compile_fail
/// fn needs_sync<T: Sync>() {}
/// needs_sync::<atomlab::SpinLock<std::rc::Rc<u8>>>();
/// ```
///
/// Usage:
///
/// ```
/// let slot = atomlab::OnceSlot::new();
/// assert!(slot.get().is_none());
/// assert_eq!(slot.set(5), Ok(()));
/// assert_eq!(slot.set(6), Err(6));
/// assert_eq!(slot.get(), Some(&5));
/// ```
pub struct OnceSlot<T> {
    state: AtomicU8,
    value: Slot<Option<T>>,
}

// SAFETY: `value` is written once, before the Release store of READY, and
// only read after an Acquire load observed READY; `&T` is shared across
// threads (`T: Sync`) and `T` moves in from the setting thread (`T: Send`).
unsafe impl<T: Send + Sync> Sync for OnceSlot<T> {}

impl<T> OnceSlot<T> {
    pub fn new() -> Self {
        Self {
            state: AtomicU8::new(EMPTY),
            value: Slot::new(None),
        }
    }

    /// Stores `value` if the slot is empty; otherwise gives it back.
    pub fn set(&self, value: T) -> Result<(), T> {
        if self
            .state
            .compare_exchange(EMPTY, WRITING, Ordering::Acquire, Ordering::Relaxed)
            .is_err()
        {
            return Err(value);
        }
        // SAFETY: we won the EMPTY -> WRITING transition, so no other thread
        // writes, and readers do not read until they see READY.
        self.value.with_mut(|p| unsafe { *p = Some(value) });
        self.state.store(READY, Ordering::Release);
        Ok(())
    }

    /// The value, if it has been published.
    pub fn get(&self) -> Option<&T> {
        if self.state.load(Ordering::Acquire) == READY {
            // SAFETY: READY was published with Release after the write, and
            // the value is never modified again.
            self.value.with(|p| unsafe { (*p).as_ref() })
        } else {
            None
        }
    }
}

impl<T> Default for OnceSlot<T> {
    fn default() -> Self {
        Self::new()
    }
}

#[cfg(all(test, not(loom)))]
mod tests {
    use super::*;
    use std::collections::HashSet;
    use std::sync::Arc;
    use std::sync::atomic::AtomicUsize as StdUsize;
    use std::thread;

    #[test]
    fn ids_are_unique_across_threads_and_wrap() {
        let ids = IdGen::new();
        let all: Vec<Vec<u64>> = thread::scope(|s| {
            let hs: Vec<_> = (0..8)
                .map(|_| s.spawn(|| (0..2000).map(|_| ids.next()).collect::<Vec<_>>()))
                .collect();
            hs.into_iter().map(|h| h.join().unwrap()).collect()
        });
        let set: HashSet<u64> = all.iter().flatten().copied().collect();
        assert_eq!(set.len(), 16_000);
        assert_eq!(ids.next(), 16_000);

        let w = IdGen::starting_at(u64::MAX);
        assert_eq!((w.next(), w.next(), w.next()), (u64::MAX, 0, 1));
        static GLOBAL: IdGen = IdGen::new();
        assert_eq!(GLOBAL.next(), 0);
    }

    #[test]
    fn limiter_never_exceeds_its_maximum() {
        let lim = Limiter::new(3);
        let over = StdUsize::new(0);
        let got = StdUsize::new(0);
        thread::scope(|s| {
            for _ in 0..8 {
                s.spawn(|| {
                    for _ in 0..5000 {
                        if let Some(_p) = lim.try_acquire() {
                            got.fetch_add(1, Ordering::Relaxed);
                            if lim.in_use() > 3 {
                                over.fetch_add(1, Ordering::Relaxed);
                            }
                        }
                    }
                });
            }
        });
        assert_eq!(over.load(Ordering::Relaxed), 0);
        assert!(got.load(Ordering::Relaxed) > 0);
        assert_eq!(lim.in_use(), 0, "all permits returned");
    }

    #[test]
    fn limiter_single_thread_semantics() {
        let lim = Limiter::new(2);
        let a = lim.try_acquire().unwrap();
        let b = lim.try_acquire().unwrap();
        assert!(lim.try_acquire().is_none());
        drop(a);
        let c = lim.try_acquire().unwrap();
        assert_eq!(lim.in_use(), 2);
        drop((b, c));
        assert_eq!(lim.in_use(), 0);
        assert!(Limiter::new(0).try_acquire().is_none());
    }

    #[test]
    fn spinlock_provides_mutual_exclusion() {
        let lock = SpinLock::new(0u64);
        thread::scope(|s| {
            for _ in 0..4 {
                s.spawn(|| {
                    for _ in 0..5000 {
                        *lock.lock() += 1; // plain, non-atomic increment
                    }
                });
            }
        });
        assert_eq!(*lock.lock(), 20_000);
    }

    #[test]
    fn spinlock_try_lock_and_panic_release() {
        let lock = Arc::new(SpinLock::new(vec![1]));
        let g = lock.lock();
        assert!(lock.try_lock().is_none());
        drop(g);
        assert!(lock.try_lock().is_some());

        let l2 = Arc::clone(&lock);
        let r = thread::spawn(move || {
            let mut g = l2.lock();
            g.push(2);
            panic!("while holding the lock");
        })
        .join();
        assert!(r.is_err());
        assert_eq!(*lock.lock(), vec![1, 2], "lock was released by unwinding");
    }

    struct Counted(Arc<StdUsize>);
    impl Drop for Counted {
        fn drop(&mut self) {
            self.0.fetch_add(1, Ordering::SeqCst);
        }
    }

    #[test]
    fn once_slot_sets_once_and_drops_once() {
        let drops = Arc::new(StdUsize::new(0));
        {
            let slot = OnceSlot::new();
            assert!(slot.get().is_none());
            assert!(slot.set(Counted(Arc::clone(&drops))).is_ok());
            let rejected = slot.set(Counted(Arc::clone(&drops))).unwrap_err();
            drop(rejected);
            assert_eq!(drops.load(Ordering::SeqCst), 1);
            assert!(slot.get().is_some());
        }
        assert_eq!(drops.load(Ordering::SeqCst), 2);
    }

    #[test]
    fn once_slot_has_exactly_one_winner_among_racing_setters() {
        for _ in 0..200 {
            let slot = OnceSlot::new();
            let wins: usize = thread::scope(|s| {
                let hs: Vec<_> = (0..8usize)
                    .map(|i| {
                        let slot = &slot;
                        s.spawn(move || usize::from(slot.set(i).is_ok()))
                    })
                    .collect();
                hs.into_iter().map(|h| h.join().unwrap()).sum()
            });
            assert_eq!(wins, 1);
            assert!(slot.get().is_some());
        }
    }

    #[test]
    fn auto_traits() {
        fn send_sync<T: Send + Sync>() {}
        send_sync::<IdGen>();
        send_sync::<Limiter>();
        send_sync::<SpinLock<Vec<u8>>>();
        send_sync::<OnceSlot<String>>();
    }
}

#[cfg(all(test, loom))]
mod loom_tests {
    use super::*;
    use loom::sync::Arc;
    use loom::thread;

    #[test]
    fn loom_spinlock_serializes_two_writers() {
        loom::model(|| {
            let lock = Arc::new(SpinLock::new(0usize));
            let hs: Vec<_> = (0..2)
                .map(|_| {
                    let lock = Arc::clone(&lock);
                    thread::spawn(move || {
                        *lock.lock() += 1;
                    })
                })
                .collect();
            for h in hs {
                h.join().unwrap();
            }
            assert_eq!(*lock.lock(), 2);
        });
    }

    #[test]
    fn loom_once_slot_publishes_the_value_to_readers() {
        loom::model(|| {
            let slot = Arc::new(OnceSlot::new());
            let s2 = Arc::clone(&slot);
            let writer = thread::spawn(move || {
                s2.set(42usize).unwrap();
            });
            if let Some(v) = slot.get() {
                assert_eq!(*v, 42);
            }
            writer.join().unwrap();
            assert_eq!(slot.get(), Some(&42));
        });
    }

    #[test]
    fn loom_limiter_never_exceeds_one() {
        loom::model(|| {
            let lim = Arc::new(Limiter::new(1));
            let inside = Arc::new(AtomicUsize::new(0));
            let hs: Vec<_> = (0..2)
                .map(|_| {
                    let (lim, inside) = (Arc::clone(&lim), Arc::clone(&inside));
                    thread::spawn(move || {
                        if let Some(_p) = lim.try_acquire() {
                            let n = inside.fetch_add(1, Ordering::SeqCst) + 1;
                            assert!(n <= 1, "two permits at once");
                            inside.fetch_sub(1, Ordering::SeqCst);
                        }
                    })
                })
                .collect();
            for h in hs {
                h.join().unwrap();
            }
            assert_eq!(lim.in_use(), 0);
        });
    }

    #[test]
    fn loom_ids_are_distinct() {
        loom::model(|| {
            let ids = Arc::new(IdGen::new());
            let a = {
                let ids = Arc::clone(&ids);
                thread::spawn(move || ids.next())
            };
            let b = ids.next();
            let a = a.join().unwrap();
            assert_ne!(a, b);
        });
    }
}
```

**Verification ✓.** `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings` and `cargo test --locked` all pass: 8 unit tests, 2 doctests (one is `compile_fail`: `SpinLock<Rc<u8>>` is not `Sync`). Under loom, 4 tests pass.

**Mutation checks ✓** (each applied to a copy, loom tests run in release mode):

| # | Mutation | Result |
|---|---|---|
| M1 | `SpinLock` unlock `Release` → `Relaxed` | **loom fails**: "Concurrent write accesses to `UnsafeCell`" |
| M2 | `SpinLock` lock CAS `Acquire` → `Relaxed` | **loom fails**, same message |
| M3 | `OnceSlot::set` publish `Release` → `Relaxed` | **loom fails**: "Concurrent read and write accesses" |
| M4 | `OnceSlot::get` load `Acquire` → `Relaxed` | **loom fails**, same message |
| M5 | `Limiter::try_acquire` CAS loop → `load` then `store` | **loom fails** (an assertion in the test; the lost update corrupts the permit count, which first shows up as an `assert_eq!` on the final `in_use`). The ordinary threaded test also failed on this mutant in my single run |
| M6 | `IdGen::next` `fetch_add` → `load` then `store` | **loom fails**: two threads got the same id |

Every mutant was caught by loom. I note two honest caveats. First, M1–M4 are caught **only** by loom: the ordinary threaded tests pass on x86-64 with the weakened orderings (I did not run the ordinary tests on those four mutants; I am relying on the x86 assembly in §4.1, which shows `Relaxed`/`Release`/`Acquire` are identical instructions there, so this is expected rather than measured). Second, a first draft of the `Limiter` had no loom test; M5 only became convincing once `loom_limiter_never_exceeds_one` existed.

**Design notes.**

- `Slot<T>` hides the difference between `std::cell::UnsafeCell` (`get()`) and `loom::cell::UnsafeCell` (`with`/`with_mut`), so the same source runs in both modes. loom's cell tracks accesses and reports races; `std`'s cannot.
- `IdGen::new` is `const` only when not under loom (loom's atomics are not `const`), so there are two `cfg`-gated definitions.
- `SpinLock::lock` spins on a `Relaxed` load between CAS attempts ("test and test-and-set"); under loom it calls `loom::thread::yield_now()` because loom needs to be told that a thread is waiting.
- `OnceSlot` uses three states: `EMPTY → WRITING → READY`. The `WRITING` state is what makes `set` safe under races: only the CAS winner writes. A loser gets its value back as `Err(value)`.
- The `PhantomData<*mut ()>` in `SpinGuard` makes the guard `!Send`, matching `std::sync::MutexGuard`.
- Not handled (deliberately): `OnceSlot` has no `Drop`-specific code (the contained `Option<T>` drops normally ✓ the drop-count test), and `SpinLock` has no poisoning and no fairness.

</details>

---

## 14. Interview questions

1. What is a data race, and how does Rust prevent it in safe code?
2. What does `Ordering::Relaxed` guarantee, and what does it not?
3. Explain Release/Acquire with an example.
4. When do you actually need `SeqCst`?
5. Why is `compare_exchange_weak` allowed to fail spuriously, and when do you use it?
6. What is the ABA problem?
7. Why might a bug in your atomics never show up on your x86 laptop?
8. When would you pick an atomic over a `Mutex`, and when not?

<details markdown="1">
<summary>Answers</summary>

1. Two threads access the same memory without ordering, at least one writing: undefined behavior. Safe Rust prevents it with the borrow rules (no `&mut` aliasing) and `Send`/`Sync`: shared mutation needs a type that is `Sync` and synchronizes internally (atomics, `Mutex`, …).
2. Atomicity of that single operation, plus all threads agree on the modification order of *that variable*. It gives no ordering relationship to other memory, so it cannot be used to publish other data (✓ loom fails `Relaxed`/`Relaxed` message passing).
3. Writer: write data, then `store(flag, Release)`. Reader: `load(flag, Acquire)`; if it sees the new value, it also sees the data written before the store. The two operations on the same atomic form the happens-before edge (§3.2).
4. When correctness depends on all threads agreeing on a single total order of operations across *several* atomics (e.g. Dekker-style algorithms where each thread writes its flag then reads the other's). Otherwise Acquire/Release suffices. When unsure, `SeqCst` is the safe default.
5. On architectures that implement compare-and-swap with load-linked/store-conditional (ARM, from memory), the store-conditional can fail for reasons other than a changed value; the weak form avoids an internal retry loop. Use it when you are already in a retry loop. Use the strong form for one-shot attempts such as `try_lock` (as the crate does).
6. A compare-exchange sees the same value `A` and assumes nothing changed, though it went `A → B → A`. Harmless for counters; dangerous for pointer-based lock-free structures where `A` was freed and reused. Mitigations: tagged pointers/counters, hazard pointers, epoch reclamation.
7. x86-64's memory model is strong: `Relaxed`, `Acquire` and `Release` loads/stores compile to the same plain `mov` instructions (✓ §4.1), so hardware reordering that the weaker orderings permit never happens there. The bug appears on ARM or after a compiler optimization. Use loom/reasoning, not "it passed".
8. Atomic: a single independent value (counter, flag, id, state), tiny critical operations, no invariant across variables. `Mutex`: anything with an invariant across several fields, anything longer than a few instructions, anything where waiting threads should sleep. Default to `Mutex`; reach for an atomic when it is clearly a single value.

</details>

---

## 15. Key takeaways

- Each atomic operation is indivisible; a *sequence* of them is not (✓ load/store loses updates, ✓ loom). Make decisions single-step with `fetch_*`, `compare_exchange`, or `fetch_update`.
- `fetch_*` return the previous value; arithmetic wraps silently (✓); `checked_*` inside `fetch_update` prevents underflow.
- Orderings describe visibility of *other* memory: `Relaxed` = atomicity only; `Release` publishes, `Acquire` receives, they must be paired (✓ loom: all three mismatched pairs fail); `SeqCst` adds a global order. Invalid combinations are a deny-by-default lint (✓).
- x86-64 hides ordering bugs (✓ identical instructions for `Relaxed`/`Acquire`/`Release` loads and stores). Use loom to check protocols; a passing test on x86 proves little.
- Spin locks and lock-free structures are specialist tools: prefer `Mutex`, channels, and `std`'s `OnceLock` unless you have measured a need.
- Any `unsafe impl Sync` is a written argument about your protocol. Write the `// SAFETY:` comment, then test it with loom.

**Connections.** Lesson 22's `Send`/`Sync` rules govern what `SpinLock` and `OnceSlot` may promise; Lesson 24's `UnsafeCell` is the primitive underneath every interior-mutability type here; Lesson 27's threads are how we exercised them. Next: Lesson 29 builds the sleeping counterpart (`Mutex` internals, `Condvar`, `RwLock`, deadlocks and lock ordering). Lesson 36 revisits `unsafe` rigorously, and Lesson 41 shows how to benchmark contention instead of guessing.

---

## 16. References

- `std::sync::atomic` (types, `Ordering`, `fence`, `compare_exchange`, `fetch_update`): <https://doc.rust-lang.org/std/sync/atomic/index.html>
- `std::sync::atomic::Ordering` (the orderings, with the C++ memory model note): <https://doc.rust-lang.org/std/sync/atomic/enum.Ordering.html>
- *The Rustonomicon*, "Atomics": <https://doc.rust-lang.org/nomicon/atomics.html>
- *The Rust Reference*, "Behavior considered undefined" (data races): <https://doc.rust-lang.org/reference/behavior-considered-undefined.html>
- Mara Bos, *Rust Atomics and Locks* (O'Reilly, 2023; free online at <https://marabos.nl/atomics/>): the best book-length treatment.
- `loom` crate documentation: <https://docs.rs/loom>
- Lint `invalid_atomic_ordering`: <https://doc.rust-lang.org/rustc/lints/listing/deny-by-default.html>

The URLs above are the standard documentation locations; I did not open them during this session (the sandbox cannot reach them, though it can download crates from crates.io, which is how `loom` 0.7.2 was fetched), so if a link has moved, search the docs site for the page title. Unchecked or from memory: the statement that Rust follows the C++20 memory model without `consume`; the list of atomic widths and target availability; that ARM/RISC-V/POWER emit distinct instructions for acquire/release and weak CAS uses LL/SC; that `crossbeam-epoch` provides epoch-based reclamation; the details of what loom does and does not model, and its exploration bounds; that edition 2024 denies references to `static mut` by default; the old "failure ordering no stronger than success" rule and its current status; and the advice on sharding counters and false sharing (no measurements were taken). Everything marked ✓ was run on this machine's `rustc 1.97.0` and, for loom, `loom 0.7.2`.
