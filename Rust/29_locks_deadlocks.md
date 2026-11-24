---
layout: default
title: "Lesson 29 — Locks, races, deadlocks"
---

# Lesson 29 — Locks, races, deadlocks

**Module 10 · Concurrency and async** · Status: generated · Edition used in examples: **2024** · External dependencies: none (standard library only)

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux with 2 cores. As before, ✓ marks outputs, diagnostics and measurements I actually ran; unmarked statements are documented behavior I did not run. Everything is **stable Rust**. Deadlock demonstrations are run with a **watchdog** (a channel with `recv_timeout`, then `process::exit`) so the demo can report "no progress" instead of hanging the session. Timing-dependent behavior is stated as such.

Builds on [Lesson 24](24_interior_mutability.md) (`Mutex`, `RwLock`, poisoning, guard temporaries), [Lesson 27](27_threads_channels.md) (threads, channels, the pool shutdown protocol) and [Lesson 28](28_atomics_memory_ordering.md) (what a lock is made of: Acquire on lock, Release on unlock). `async` locks are Lessons 30–31.

---

## 1. Three different bugs people call "concurrency bugs"

| Bug | What happens | Does Rust's type system stop it? |
|---|---|---|
| **Data race** | Unsynchronized access, one write: undefined behavior | Yes, in safe code (Lessons 22, 24) |
| **Race condition** (logic race) | Result depends on timing: check-then-act, lost update, wrong order of events | **No** |
| **Deadlock** | Threads wait for each other forever | **No** |

A `Mutex` removes data races by construction and gives you a place to *write* the correct logic; it does not make the logic correct. Safe Rust can deadlock, and a deadlock is not "unsafe": the language does not consider it a memory-safety problem, but it is a complete outage of whatever those threads were doing. This lesson is about the second and third rows.

---

## 2. Lock discipline: the basics

```rust
use std::sync::Mutex;
fn main() {
    let m = Mutex::new(5);
    let g = m.lock().unwrap();
    println!("{:?}", m.try_lock().map(|_| ()).map_err(|e| matches!(e, std::sync::TryLockError::WouldBlock)));
    drop(g);
    let n = *m.lock().unwrap() + 1; // guard is a temporary: released at the end of the statement
    println!("{n}");
    if let Some(_) = m.try_lock().ok() {
        println!("inside: try_lock again -> {}", m.try_lock().is_ok());
    }
    match *m.lock().unwrap() {
        5 => println!("match arm: try_lock -> {}", m.try_lock().is_ok()),
        _ => {}
    }
}
```

✓ Output:

```text
Err(true)
6
inside: try_lock again -> false
match arm: try_lock -> false
```

What it shows:

- **`try_lock` never blocks.** It returns `Err(TryLockError::WouldBlock)` when someone holds the lock (✓ `Err(true)` means "it was `WouldBlock`"), or `Err(Poisoned(..))` after a panic (Lesson 24).
- **The guard is released when it is dropped.** A guard in a temporary lives until the end of the *enclosing statement*. In `let n = *m.lock().unwrap() + 1;` that is the `;`.
- **A temporary in a `match` scrutinee or an `if let` scrutinee lives for the whole body** (✓ the last two lines: the lock was still held inside the arm and inside the block, so `try_lock` failed). This is the classic way to hold a lock far longer than intended, and the classic way to deadlock yourself by calling a method that locks the same mutex from inside the arm. (Edition 2024 changed when `if let` temporaries are dropped *relative to the `else` branch*; I did not test that difference here, and it does not change what happens inside the body.) Fix: copy what you need out in a separate `let` statement and let the guard drop first.
- **`MutexGuard` is `!Send`** ✓:

```rust,ignore
let g = m.lock().unwrap();
thread::spawn(move || println!("{}", *g));   // ✓ error[E0277]: `MutexGuard<'_, i32>` cannot be sent between threads safely
```

The reason: some platforms require the thread that locked a mutex to be the one that unlocks it (from memory), so the type forbids moving the guard away.

### 2.1 The rules of thumb

1. **Hold locks for as short a time as possible**, and never across blocking calls (I/O, `join`, `recv`, `sleep`, a call into code you do not control, such as a callback). Everything that waits while holding a lock extends the wait of everyone behind it, and may complete a deadlock cycle.
2. **Never call unknown code while holding a lock.** A callback can lock another mutex, or the same one.
3. **Do the work outside, publish inside.** Compute the result without the lock; take the lock only to swap it in.
4. **One lock per invariant.** Fields that must stay consistent with each other belong behind the *same* lock (Lesson 24: do not split an invariant over two mutexes).

---

## 3. Deadlock

A deadlock needs four conditions at once (the Coffman conditions; from memory): *mutual exclusion* (the resource is exclusive), *hold and wait* (a thread holds one resource while waiting for another), *no preemption* (resources are not taken away), and *circular wait* (A waits for B, B waits for A). Break any one and deadlock is impossible. In lock-based programs, breaking **circular wait** with a global lock order is the standard technique.

### 3.1 Self-deadlock

```rust
use std::sync::Mutex;
fn main() {
    let m = Mutex::new(1);
    let _g1 = m.lock().unwrap();
    let _g2 = m.lock().unwrap(); // same thread, same mutex
}
```

The documentation for `Mutex::lock` says this call "might panic" or deadlock if the current thread already holds the lock; the behavior is unspecified (from memory of the docs). ✓ On this machine (Linux, `rustc 1.97.0`) it **hangs**: I ran it inside a scoped thread with a one-second watchdog and got `Err(Timeout)`. Do not rely on either behavior. The `std` `Mutex` is **not reentrant**.

The same applies to `RwLock`. ✓ A thread that holds a *read* lock and then asks for the *write* lock on the same `RwLock` waits for itself forever (watchdog: `Err(Timeout)`). There is no lock "upgrade" in `std`: drop the read guard first, then take the write lock, and **re-check** what you read, because the world may have changed in between.

### 3.2 The classic: two locks, opposite order

```rust
use std::sync::{mpsc, Arc, Mutex};
use std::thread;
use std::time::Duration;
fn main() {
    let a = Arc::new(Mutex::new(0));
    let b = Arc::new(Mutex::new(0));
    let (tx, rx) = mpsc::channel();
    for (name, first, second) in [("T1", a.clone(), b.clone()), ("T2", b.clone(), a.clone())] {
        let tx = tx.clone();
        thread::spawn(move || {
            let _g1 = first.lock().unwrap();
            thread::sleep(Duration::from_millis(100)); // makes the interleaving certain
            let _g2 = second.lock().unwrap();
            tx.send(name).unwrap();
        });
    }
    match rx.recv_timeout(Duration::from_secs(2)) {
        Ok(n) => println!("{n} finished"),
        Err(e) => println!("no progress: {e:?}"),
    }
    println!("a try_lock ok? {}", a.try_lock().is_ok());
    println!("b try_lock ok? {}", b.try_lock().is_ok());
    std::process::exit(0);
}
```

✓ Output:

```text
no progress: Timeout
a try_lock ok? false
b try_lock ok? false
```

T1 holds `a` and waits for `b`; T2 holds `b` and waits for `a`. Both locks are held and neither thread will ever finish. The `sleep` is there only to make the bad interleaving certain in a demo. **Real deadlocks have no sleep**: they show up once in a million requests, under load, in production, which is why you design them out instead of testing for them. In a realistic variant (✓ run once, two threads each doing a million `lock a; lock b` iterations in opposite order, no `sleep`) the program also made no progress within 5 seconds for either thread (`Err(Timeout)` twice). That run proves the bug is real, not that it is frequent; deadlocks depend on timing.

### 3.3 Fix 1: a global lock order

If every thread acquires locks in the same global order, a cycle cannot form. For accounts, order by id or index:

```rust
use std::sync::{Arc, Mutex};
use std::thread;
struct Account { id: u32, bal: Mutex<i64> }
fn transfer(a: &Account, b: &Account, amt: i64) {
    let (first, second) = if a.id < b.id { (a, b) } else { (b, a) };
    let mut g1 = first.bal.lock().unwrap();
    let mut g2 = second.bal.lock().unwrap();
    let (from, to) = if a.id < b.id { (&mut *g1, &mut *g2) } else { (&mut *g2, &mut *g1) };
    *from -= amt;
    *to += amt;
}
fn main() {
    let x = Arc::new(Account { id: 1, bal: Mutex::new(1000) });
    let y = Arc::new(Account { id: 2, bal: Mutex::new(1000) });
    let hs: Vec<_> = (0..4).map(|i| {
        let (x, y) = (x.clone(), y.clone());
        thread::spawn(move || for _ in 0..20_000 {
            if i % 2 == 0 { transfer(&x, &y, 1) } else { transfer(&y, &x, 1) }
        })
    }).collect();
    for h in hs { h.join().unwrap(); }
    println!("{} {}", x.bal.lock().unwrap(), y.bal.lock().unwrap());
}
```

✓ Prints `1000 1000`: 80 000 transfers in both directions complete, money is conserved. Two details that bite: (1) **`from == to`** would lock the same mutex twice, which is a self-deadlock (§3.1), so handle it first (the `Bank` in the practical crate does); (2) the order must be a *total* order that every code path agrees on. Ordering by memory address works for locks of the same type (from memory; compare with `std::ptr::eq` plus `as *const _ as usize`) but an explicit `id` or `level` is clearer and survives moves.

### 3.4 Fix 2: enforce the order in the type (`LeveledMutex`)

Documentation of "always lock A before B" rots. The practical crate encodes the rule at run time: each `LeveledMutex` has a numeric level, a thread-local list remembers the levels the current thread holds, and `lock()` returns `Err(LevelViolation)` if you ask for a level that is not strictly higher. A violation is reported **at the first wrong acquisition on one thread**, even when no other thread is around, which is far better than a one-in-a-million hang. (This is the idea behind lock-hierarchy checkers in several systems; from memory, the `parking_lot` crate also offers an optional deadlock detector, which I did not test.)

### 3.5 Other ways out

- **Hold only one lock at a time.** If you can copy the data out, release, then lock the next, there is no "hold and wait".
- **`try_lock` and back off.** Take the first lock, `try_lock` the second, and if it is busy release everything and retry (with a random delay to avoid livelock, where threads keep retrying in lockstep). It avoids deadlock but adds retry logic.
- **Message passing** (Lesson 27): one thread owns the state, others send it requests. No shared lock, no cycle. Cost: latency and the owner thread's backlog.
- **Timeouts**: `Condvar::wait_timeout`, channel `recv_timeout`, and `try_lock` loops turn a hang into an error you can log. `Mutex::lock` itself has **no timeout** in `std`.
- **Don't lock at all**: immutable shared data (`Arc<T>`), atomics for single values (Lesson 28), or copy-on-write snapshots (`Arc` swapped under a short lock).

---

## 4. `Condvar`: waiting for a condition

A `Mutex` answers "who may touch this data now?". A `Condvar` (condition variable) answers "how do I sleep until the data is in the state I need?", without spinning (Lesson 28 §6).

```rust
use std::sync::{Arc, Condvar, Mutex};
use std::thread;
use std::time::Duration;
fn main() {
    let pair = Arc::new((Mutex::new(false), Condvar::new()));
    let p2 = Arc::clone(&pair);
    let t = thread::spawn(move || {
        thread::sleep(Duration::from_millis(50));
        let (m, cv) = &*p2;
        *m.lock().unwrap() = true;
        cv.notify_one();
    });
    let (m, cv) = &*pair;
    let mut ready = m.lock().unwrap();
    while !*ready {
        ready = cv.wait(ready).unwrap();
    }
    println!("ready = {}", *ready);
    drop(ready);
    t.join().unwrap();
}
```

✓ Prints `ready = true`. The protocol, which is always the same:

1. The *state* (`bool` here) lives **inside** the mutex.
2. A waiter locks, checks the condition **in a loop**, and calls `cv.wait(guard)`. `wait` atomically releases the mutex and puts the thread to sleep; when woken it **re-acquires** the mutex before returning.
3. A notifier changes the state under the lock, then calls `notify_one` or `notify_all`.

### 4.1 Why the loop is mandatory

- **Spurious wake-ups**: the documentation permits `wait` to return without any notification.
- **Stolen wake-ups**: between the notify and the waiter re-acquiring the lock, another thread may consume the state (two consumers, one item).
- **Lost notifications**: a notification sent when nobody is waiting is **not remembered**. ✓ A waiter that calls `wait` without checking state, after `notify_one` already happened, sleeps forever (watchdog: `Err(Timeout)`). The state in the mutex is what makes the protocol robust: the waiter checks the state first, and only sleeps if the condition is false.

`wait_while(guard, |state| condition_to_keep_waiting)` packages the loop (✓ `count = 3` after three increments). `wait_timeout` and `wait_timeout_while` return the guard plus a `WaitTimeoutResult` (✓ `timed_out() == true` when nothing notified within 30 ms). With timeouts, compute one **deadline** and wait for the remainder, otherwise every spurious wake-up restarts the full timeout (the crate's `push_timeout` does this).

### 4.2 Using it correctly

- Use **one** mutex with a given `Condvar` (using different mutexes with the same condvar is documented to panic; from memory). Keep them together in one struct.
- `notify_one` wakes one waiter, `notify_all` wakes all. If different waiters wait for *different* conditions on the same condvar, `notify_one` may wake the wrong one and the right one sleeps on: use `notify_all`, or (better) one condvar per condition, as `BoundedQueue` does with `not_empty` and `not_full`. `close()` must use `notify_all` (✓ mutation M3 below hangs when it uses `notify_one`).
- Release the lock before notifying (`drop(guard)`) when it is convenient: the woken thread can then lock immediately. Notifying while still holding the lock is correct but may cause an extra wake-and-block.
- `wait` returns `Err(PoisonError)` if the mutex was poisoned (Lesson 24), like `lock`. Decide once, in a helper, whether to recover (`into_inner`) or propagate, as the crate's `lock`/`wait` helpers do.

### 4.3 `Barrier`

`std::sync::Barrier::new(n)`: the first `n - 1` callers of `wait()` block until the `n`-th arrives; then all continue, and exactly one of them gets `is_leader() == true` (✓ `leaders = 1` with three threads). It is a `Mutex` plus `Condvar` underneath and is reusable.

---

## 5. `RwLock` in practice

Many readers or one writer (✓ two simultaneous read guards; `try_write` fails while readers exist; `try_read` fails while a writer exists; both succeed again after release).

- It only pays off when reads dominate **and** the critical section is long enough to benefit from running in parallel. For short sections a plain `Mutex` is often as fast or faster, because an `RwLock` has more bookkeeping (Lesson 24 measured sizes: `RwLock<u64>` 24 bytes vs `Mutex<u64>` 16). I give no timing numbers; measure (Lesson 41).
- **Writer starvation** and fairness are *implementation-defined*: the standard library does not promise whether new readers are admitted while a writer waits (from memory of the docs). Do not build correctness on a particular policy.
- **Recursive read locking** may deadlock if a writer is queued between your two `read()` calls on some implementations (from memory). Do not take the read lock twice on one thread.
- Everything in §3 applies: the read→write "upgrade" self-deadlock (✓ §3.1), lock ordering across several `RwLock`s, guards held in `match` scrutinees.

---

## 6. Granularity: coarse versus fine locks

| Strategy | Pros | Cons |
|---|---|---|
| **One big lock** | Simple, no ordering problem, easy to reason about | Serializes everything |
| **Lock per object** (`Vec<Mutex<_>>`, per-account) | Parallelism | Needs a lock order (§3.3), multi-object operations are harder, snapshots need all locks |
| **Sharding** (hash key → one of N locks) | Bounded number of locks, scales with cores | Cross-shard operations again need ordering |
| **Lock-free / atomics** | No blocking | Hard to get right (Lesson 28 ABA, memory ordering) |

Start with one lock. Split only when a profile shows contention (Lesson 41), and when you split, write down the lock order **first**. A consistent snapshot across many fine locks needs all of them, taken in order (the crate's `Bank::total`).

---

## 7. Common mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Locking the same `Mutex` twice on one thread | Hang (✓ on this platform) or panic | Restructure; never call back into code that locks again |
| Two locks in opposite orders | Deadlock (✓ watchdog) | Global order (§3.3) or `LeveledMutex` (§3.4) |
| `wait` without a condition loop | Lost/spurious wake-up bugs (✓ lost notification hangs) | `while !cond { guard = cv.wait(guard)? }` or `wait_while` |
| `if` instead of `while` around `wait` | Proceeds when another thread took the item: queue grows past capacity (✓ crate mutation M2) | `while` |
| `notify_one` on shutdown | Some waiters never wake (✓ M3 hangs) | `notify_all` |
| Forgetting to notify after changing state (✓ M5, M6) | Waiters sleep forever | Notify after every change that can satisfy a waiter |
| Holding a guard across I/O, `join`, `recv`, `sleep` or a callback | Long stalls; deadlock cycles through the other party | Copy data out, drop the guard |
| Guard in a `match`/`if let` scrutinee, then re-locking in the arm | Self-deadlock (✓ §2) | `let value = *m.lock().unwrap();` first |
| Read lock then write lock on the same `RwLock` | Self-deadlock (✓) | Drop, re-lock, re-check |
| Lock order depends on data (`transfer(a, b)` vs `transfer(b, a)`) | Deadlock under opposite requests | Sort by id first; handle `a == b` |
| Treating poisoning as "can't happen" | `unwrap()` cascades panics through every thread | Decide: propagate, or `into_inner()` if invariants hold |
| Testing deadlock-freedom by "it ran fine" | Misses rare interleavings | Design the order; add a watchdog to tests (§9) |

---

## 8. Performance and reliability notes

- **Uncontended** locking is cheap; **contended** locking is where time goes: threads sleep, wake, bounce the cache line. I give no numbers (a 2-core sandbox would not transfer); Lesson 41 shows how to measure.
- Holding a lock during a slow operation turns every other thread's request into that latency. The fix is almost always to shrink the critical section, not to change the lock type.
- **Convoys and priority inversion**: when a lock holder is descheduled, all waiters stall. Real-time systems use priority inheritance (from memory); `std` does not offer it.
- **Fairness**: `std`'s `Mutex` makes no fairness promise (from memory); a thread can in principle be starved under heavy contention.
- **Observability**: a stuck process shows its blocked threads in a debugger or `gdb`/`rust-gdb` backtrace of all threads (Lesson 43 covers production diagnosis). Log lock wait times at the boundaries you control.
- **Blockchain-adjacent note**: nodes combine mempools, state caches, peer tables and RPC handlers on shared structures. The deadlocks that bite are *cross-component*: a handler holds the state lock and calls into the mempool, while the mempool holds its lock and calls into state. A written lock hierarchy (state before mempool before peers, say) plus bounded queues at the boundaries (the crate's `BoundedQueue`) is the discipline.
- **Security**: lock-induced stalls are a denial-of-service lever. An attacker who can make a handler hold a global lock for a long time (a slow request, an expensive deserialization performed *inside* the lock) can freeze a node. Parse and validate **before** taking the lock; bound everything; use timeouts.

---

## 9. Testing concurrent code

- Wrap potentially-hanging tests in a **watchdog** like the crate's `within(secs, f)`: run `f` on a thread, `recv_timeout` its result, fail with "probable deadlock" instead of hanging CI. (The stuck thread is leaked; fine in a test process.)
- **Assert invariants, not interleavings**: total money is conserved, queue length never exceeds capacity (sample it from another thread), concurrency never exceeds the permit count.
- **Stress**, but do not mistake it for proof: ✓ the capacity test below failed in its mutant on the first run, yet my *first* version of the test suite let the same mutant pass, because the invariant was never observed. A test only catches what it measures.
- Model checkers (loom, Lesson 28) work for lock-based code too: loom has its own `Mutex` and `Condvar`. I did not use them in this lesson's crate.

---

## 10. Exercises

Answers are collapsed. Try each before opening it.

1. **(Warm-up)** What is the difference between `lock()` and `try_lock()`?
2. **(Warm-up)** In `let n = *m.lock().unwrap() + 1;`, when is the lock released?
3. **(Easy)** Why must a `Condvar` wait be inside a `while` loop?
4. **(Medium)** Two threads run `transfer(a, b)` and `transfer(b, a)` concurrently, each locking `from` then `to`. Explain how this deadlocks, and give two different fixes.
5. **(Medium)** Write a `Latch` (a one-shot gate): `Latch::new()`, `wait(&self)` blocks until `open(&self)` is called, after which `wait` returns immediately and forever. Use `Mutex<bool>` and `Condvar`.
6. **(Medium-hard)** What is wrong with the following, and what is the minimal fix?
   ```rust,ignore
   match *state.lock().unwrap() {
       State::Idle => start(&state),   // start() locks `state` too
       State::Busy => {}
   }
   ```
7. **(Hard)** A `RwLock<HashMap<K, V>>` cache is read on the fast path. On a miss you compute the value and insert it. Write the `get_or_insert_with` logic so it is correct and does not hold the write lock while computing. What race remains, and how do you handle it?
8. **(Hard)** Why does `close()` in `BoundedQueue` call `notify_all` on both condvars, and what exactly happens to a producer blocked in `push` when the queue is closed?

<details markdown="1">
<summary>Answers</summary>

1. `lock()` blocks until the lock is free (and returns `Err` if poisoned). `try_lock()` returns immediately with `Err(WouldBlock)` if it is held (✓ §2), or `Err(Poisoned)`.
2. At the end of the statement, the `;`. The guard is a temporary (✓ §2). (Contrast with `let g = m.lock().unwrap();`, where it lives until `g` is dropped.)
3. Because wake-ups can be spurious, and because another thread may have consumed the state between the notification and your re-acquiring the lock. The condition must be re-checked every time you wake (§4.1). `wait_while` does it for you.
4. T1 locks `a` then wants `b`; T2 locks `b` then wants `a`: circular wait (✓ §3.2). Fixes: (a) lock in a global order, e.g. by account id, handling `a == b` (✓ §3.3); (b) hold one lock at a time, e.g. withdraw from `a` (lock, subtract, unlock), then deposit into `b` — but then money is "in flight" and the total is briefly inconsistent, so you need a protocol for failure (refund) and for snapshots; or (c) `try_lock` the second and back off. (Message passing to a single owner thread is a fourth option.)
5. ```rust
   use std::sync::{Condvar, Mutex};
   pub struct Latch { open: Mutex<bool>, cv: Condvar }
   impl Latch {
       pub fn new() -> Self { Self { open: Mutex::new(false), cv: Condvar::new() } }
       pub fn open(&self) {
           *self.open.lock().unwrap_or_else(|e| e.into_inner()) = true;
           self.cv.notify_all();               // all waiters, not one
       }
       pub fn wait(&self) {
           let g = self.open.lock().unwrap_or_else(|e| e.into_inner());
           let _g = self.cv.wait_while(g, |open| !*open).unwrap_or_else(|e| e.into_inner());
       }
   }
   fn main() {
       let l = Latch::new();
       l.open();
       l.wait(); // returns immediately: the state is "open", not a lost notification
       println!("done");
   }
   ```
   The state (`bool`) is what makes late waiters work; a bare `Condvar` would lose the notification (✓ §4.1).
6. The temporary guard from `state.lock()` lives for the whole `match`, so `start(&state)` tries to lock a mutex the same thread already holds: self-deadlock (✓ §2 and §3.1). Minimal fix: `let current = *state.lock().unwrap();` (for a `Copy` state) and `match current { … }`, so the guard is dropped at the `;`. If `State` is not `Copy`, clone or extract the discriminant.
7. Fast path: `if let Some(v) = cache.read().unwrap().get(&k) { return v.clone(); }` — but see the scrutinee rule: the read guard lives for the `if let` body, which is fine here because we return. Slow path: compute **without any lock**, then `let mut w = cache.write().unwrap();` and `w.entry(k).or_insert(computed).clone()`. The race that remains: two threads miss on the same key and both compute (duplicate work). `entry().or_insert` keeps the first inserted value, so the result stays consistent. If computing is expensive or has side effects, use a per-key `OnceLock`/`OnceCell` in the map or a "pending" marker so one thread computes and others wait (Lesson 24's `OnceLock`).
8. Producers waiting for space sleep on `not_full`; consumers waiting for items sleep on `not_empty`. When the queue closes, **every** sleeper must wake and re-check the `closed` flag, so `notify_all` on both (✓ the mutation using `notify_one` hung: only one blocked thread woke). A producer blocked in `push` wakes, sees `closed`, and returns `Err(PushError::Closed(value))` — handing its value back (✓ test `close_wakes_blocked_consumers_and_producers`). Consumers drain remaining items first and get `None` only when closed *and* empty.

</details>

---

## 11. Debugging challenge

A worker reports it "processes jobs", but after the first `shutdown()` call some workers never exit and `join` hangs.

```rust
use std::collections::VecDeque;
use std::sync::{Arc, Condvar, Mutex};
use std::thread;
use std::time::Duration;

struct Queue {
    jobs: Mutex<VecDeque<u32>>,
    cv: Condvar,
    done: Mutex<bool>,
}

fn worker(q: Arc<Queue>) -> usize {
    let mut handled = 0;
    loop {
        let mut jobs = q.jobs.lock().unwrap();
        if jobs.is_empty() {
            if *q.done.lock().unwrap() {
                return handled;
            }
            jobs = q.cv.wait(jobs).unwrap();
        }
        if let Some(_job) = jobs.pop_front() {
            handled += 1;
        }
    }
}

fn main() {
    let q = Arc::new(Queue {
        jobs: Mutex::new(VecDeque::new()),
        cv: Condvar::new(),
        done: Mutex::new(false),
    });
    let hs: Vec<_> = (0..3).map(|_| { let q = q.clone(); thread::spawn(move || worker(q)) }).collect();
    for j in 0..10 {
        q.jobs.lock().unwrap().push_back(j);
        q.cv.notify_one();
    }
    thread::sleep(Duration::from_millis(100));
    *q.done.lock().unwrap() = true;
    q.cv.notify_one();
    let total: usize = hs.into_iter().map(|h| h.join().unwrap()).sum();
    println!("handled {total}");
}
```

✓ I ran this exact program with a 15-second limit: it **hung** (killed by the limit). I then changed only `notify_one` to `notify_all` at shutdown and ran it six times: all six printed `handled 10`. So the missing broadcast is what hangs it *here*, but the program has two further defects that the six lucky runs do not exonerate.

Find the problems and fix them.

<details markdown="1">
<summary>Solution</summary>

1. **Shutdown uses `notify_one`** while three workers sleep: one wakes and exits, the other two sleep forever and `join` hangs (✓ measured above; the same defect in the crate, mutation M3, hangs too). Shutdown must `notify_all`.
2. **The condition spans two mutexes** (`jobs` and `done`), and the notifier does not hold the one the waiter sleeps on. A worker can check `done` (false) and then, before it reaches `wait`, the main thread sets `done` and notifies: **that notification is lost** and the worker sleeps forever. I did not manage to trigger this window in six runs (it is narrow), so this one is reasoning, not a measurement. The state the condition depends on must be protected by the *same* mutex the `Condvar` is used with.
3. **`if` instead of a `while` loop around `wait`**: after waking, the worker pops without re-checking the condition. It only avoids misbehaving because the code happens to tolerate an empty queue; it also never re-checks `done` after waking, so a worker woken by shutdown goes round the loop on luck of the structure. Loop on one predicate.

Fix: put both fields under one mutex and loop on one predicate:

```rust
use std::collections::VecDeque;
use std::sync::{Arc, Condvar, Mutex};
use std::thread;

struct State { jobs: VecDeque<u32>, done: bool }
struct Queue { state: Mutex<State>, cv: Condvar }

fn worker(q: Arc<Queue>) -> usize {
    let mut handled = 0;
    loop {
        let mut st = q.cv.wait_while(q.state.lock().unwrap(), |s| s.jobs.is_empty() && !s.done).unwrap();
        match st.jobs.pop_front() {
            Some(_job) => { drop(st); handled += 1; }  // do the work with the lock released
            None => return handled,                    // empty and done
        }
    }
}

fn main() {
    let q = Arc::new(Queue { state: Mutex::new(State { jobs: VecDeque::new(), done: false }), cv: Condvar::new() });
    let hs: Vec<_> = (0..3).map(|_| { let q = q.clone(); thread::spawn(move || worker(q)) }).collect();
    for j in 0..10 {
        q.state.lock().unwrap().jobs.push_back(j);
        q.cv.notify_one();
    }
    q.state.lock().unwrap().done = true;
    q.cv.notify_all();
    let total: usize = hs.into_iter().map(|h| h.join().unwrap()).sum();
    println!("handled {total}");
}
```

✓ The fixed version prints `handled 10` (all jobs are drained before the workers see "empty and done"). Point 1 was measured; points 2 and 3 are reasoning from the §4 rules (the crate's mutations M2 and M5 show the same classes of bug failing).

</details>

---

## 12. Practical challenge: `locklab`

Build a crate with:

1. `BoundedQueue<T>`: blocking `push`/`pop`, non-blocking `try_push`/`try_pop`, `push_timeout`, `close()` (pops drain, then `None`; pushes fail with the value handed back), `len`, `is_empty`. Capacity ≥ 1.
2. `Semaphore`: `acquire`, `try_acquire`, RAII `SemPermit`, `available`.
3. `Bank`: accounts behind individual locks; `transfer(from, to, amount)` that cannot deadlock (including `from == to`), returns typed errors, never creates or destroys money; `total()` as a consistent snapshot.
4. `LeveledMutex<T>`: a mutex that rejects out-of-order (and same-level) acquisition per thread, with guards droppable in any order and `!Send`.

Tests must use a **watchdog** so a deadlock fails instead of hanging. Poisoning must be handled by a deliberate policy.

<details markdown="1">
<summary>Solution (spoilers: full source)</summary>

`Cargo.toml`: a standard `cargo new --lib locklab --edition 2024`, no dependencies.

`src/lib.rs`:

```rust
//! `locklab`: blocking primitives built from `Mutex` and `Condvar`, and a
//! lock-ordering discipline that makes deadlocks impossible by construction.
//!
//! * [`BoundedQueue`]: blocking MPMC queue with close and timeouts.
//! * [`Semaphore`]: counting semaphore with RAII permits.
//! * [`Bank`]: transfers between accounts, locking in index order.
//! * [`LeveledMutex`]: a mutex that refuses out-of-order acquisition.

use std::cell::RefCell;
use std::collections::VecDeque;
use std::fmt;
use std::ops::{Deref, DerefMut};
use std::sync::{Condvar, Mutex, MutexGuard};
use std::time::{Duration, Instant};

/// Locks `m`, ignoring poisoning. Every critical section in this crate leaves
/// its data consistent at each point where it could panic, so the data behind
/// a poisoned lock is still valid.
fn lock<T>(m: &Mutex<T>) -> MutexGuard<'_, T> {
    m.lock().unwrap_or_else(|e| e.into_inner())
}

/// Waits on `cv`, ignoring poisoning (same reasoning as [`lock`]).
fn wait<'a, T>(cv: &Condvar, g: MutexGuard<'a, T>) -> MutexGuard<'a, T> {
    cv.wait(g).unwrap_or_else(|e| e.into_inner())
}

// ------------------------------------------------------------ BoundedQueue

struct QueueState<T> {
    items: VecDeque<T>,
    closed: bool,
}

/// Why a push failed. The value is handed back.
#[derive(Debug, PartialEq, Eq)]
pub enum PushError<T> {
    /// The queue is closed.
    Closed(T),
    /// The queue is full (`try_push`) or stayed full (`push_timeout`).
    Full(T),
}

/// A blocking multi-producer multi-consumer queue with a fixed capacity.
///
/// ```
/// use locklab::BoundedQueue;
/// let q = BoundedQueue::new(2);
/// q.push(1).unwrap();
/// q.push(2).unwrap();
/// assert!(q.try_push(3).is_err());
/// assert_eq!(q.pop(), Some(1));
/// q.close();
/// assert_eq!(q.pop(), Some(2));
/// assert_eq!(q.pop(), None);
/// ```
pub struct BoundedQueue<T> {
    state: Mutex<QueueState<T>>,
    not_empty: Condvar,
    not_full: Condvar,
    capacity: usize,
}

impl<T> BoundedQueue<T> {
    /// # Panics
    /// If `capacity` is zero (nothing could ever be pushed).
    pub fn new(capacity: usize) -> Self {
        assert!(capacity > 0, "capacity must be at least 1");
        Self {
            state: Mutex::new(QueueState {
                items: VecDeque::with_capacity(capacity),
                closed: false,
            }),
            not_empty: Condvar::new(),
            not_full: Condvar::new(),
            capacity,
        }
    }

    /// Blocks while the queue is full. Fails only if the queue is closed.
    pub fn push(&self, value: T) -> Result<(), PushError<T>> {
        let mut st = lock(&self.state);
        // Always re-check the condition in a loop: wake-ups can be spurious
        // and another thread may have taken the slot first.
        while st.items.len() == self.capacity && !st.closed {
            st = wait(&self.not_full, st);
        }
        if st.closed {
            return Err(PushError::Closed(value));
        }
        st.items.push_back(value);
        drop(st); // release before notifying so the woken thread can lock at once
        self.not_empty.notify_one();
        Ok(())
    }

    /// Never blocks.
    pub fn try_push(&self, value: T) -> Result<(), PushError<T>> {
        let mut st = lock(&self.state);
        if st.closed {
            return Err(PushError::Closed(value));
        }
        if st.items.len() == self.capacity {
            return Err(PushError::Full(value));
        }
        st.items.push_back(value);
        drop(st);
        self.not_empty.notify_one();
        Ok(())
    }

    /// Waits at most `timeout` in total for space.
    pub fn push_timeout(&self, value: T, timeout: Duration) -> Result<(), PushError<T>> {
        let deadline = Instant::now() + timeout;
        let mut st = lock(&self.state);
        while st.items.len() == self.capacity && !st.closed {
            let left = deadline.saturating_duration_since(Instant::now());
            if left.is_zero() {
                return Err(PushError::Full(value));
            }
            st = self
                .not_full
                .wait_timeout(st, left)
                .unwrap_or_else(|e| e.into_inner())
                .0;
        }
        if st.closed {
            return Err(PushError::Closed(value));
        }
        st.items.push_back(value);
        drop(st);
        self.not_empty.notify_one();
        Ok(())
    }

    /// Blocks until an item arrives. Returns `None` only when the queue is
    /// closed **and** drained.
    pub fn pop(&self) -> Option<T> {
        let mut st = lock(&self.state);
        loop {
            if let Some(v) = st.items.pop_front() {
                drop(st);
                self.not_full.notify_one();
                return Some(v);
            }
            if st.closed {
                return None;
            }
            st = wait(&self.not_empty, st);
        }
    }

    /// Never blocks.
    pub fn try_pop(&self) -> Option<T> {
        let v = lock(&self.state).items.pop_front();
        if v.is_some() {
            self.not_full.notify_one();
        }
        v
    }

    /// Refuses new items and wakes every waiter. Items already queued can
    /// still be popped.
    pub fn close(&self) {
        lock(&self.state).closed = true;
        // `notify_all`, not `notify_one`: every blocked thread must re-check.
        self.not_empty.notify_all();
        self.not_full.notify_all();
    }

    pub fn len(&self) -> usize {
        lock(&self.state).items.len()
    }

    pub fn is_empty(&self) -> bool {
        self.len() == 0
    }
}

// --------------------------------------------------------------- Semaphore

/// Counting semaphore. Permits are returned when the [`SemPermit`] drops.
pub struct Semaphore {
    free: Mutex<usize>,
    cv: Condvar,
}

#[must_use = "dropping the permit immediately releases it"]
pub struct SemPermit<'a> {
    sem: &'a Semaphore,
}

impl Semaphore {
    pub fn new(permits: usize) -> Self {
        Self {
            free: Mutex::new(permits),
            cv: Condvar::new(),
        }
    }

    /// Blocks until a permit is free.
    pub fn acquire(&self) -> SemPermit<'_> {
        let g = lock(&self.free);
        // `wait_while` packages the "loop until the predicate is false" idiom.
        let mut g = self
            .cv
            .wait_while(g, |n| *n == 0)
            .unwrap_or_else(|e| e.into_inner());
        *g -= 1;
        SemPermit { sem: self }
    }

    pub fn try_acquire(&self) -> Option<SemPermit<'_>> {
        let mut g = lock(&self.free);
        if *g == 0 {
            return None;
        }
        *g -= 1;
        Some(SemPermit { sem: self })
    }

    pub fn available(&self) -> usize {
        *lock(&self.free)
    }
}

impl Drop for SemPermit<'_> {
    fn drop(&mut self) {
        *lock(&self.sem.free) += 1;
        self.sem.cv.notify_one();
    }
}

// -------------------------------------------------------------------- Bank

/// Errors from [`Bank::transfer`].
#[derive(Debug, PartialEq, Eq)]
pub enum TransferError {
    NoSuchAccount(usize),
    Insufficient { needed: u64, available: u64 },
}

impl fmt::Display for TransferError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            TransferError::NoSuchAccount(i) => write!(f, "no such account {i}"),
            TransferError::Insufficient { needed, available } => {
                write!(f, "needed {needed}, available {available}")
            }
        }
    }
}

impl std::error::Error for TransferError {}

/// A set of accounts, each behind its own lock (fine-grained locking).
///
/// Two transfers `a -> b` and `b -> a` running together would deadlock if each
/// locked "its own" account first. `transfer` therefore always locks the
/// **lower index first**, whatever the direction: a global lock order.
pub struct Bank {
    accounts: Vec<Mutex<u64>>,
}

impl Bank {
    pub fn new(balances: &[u64]) -> Self {
        Self {
            accounts: balances.iter().map(|&b| Mutex::new(b)).collect(),
        }
    }

    pub fn balance(&self, i: usize) -> Option<u64> {
        self.accounts.get(i).map(|m| *lock(m))
    }

    pub fn transfer(&self, from: usize, to: usize, amount: u64) -> Result<(), TransferError> {
        for i in [from, to] {
            if i >= self.accounts.len() {
                return Err(TransferError::NoSuchAccount(i));
            }
        }
        if from == to {
            // Locking the same Mutex twice would deadlock; handle it up front.
            let g = lock(&self.accounts[from]);
            return if *g >= amount {
                Ok(())
            } else {
                Err(TransferError::Insufficient {
                    needed: amount,
                    available: *g,
                })
            };
        }
        let (lo, hi) = if from < to { (from, to) } else { (to, from) };
        let mut glo = lock(&self.accounts[lo]);
        let mut ghi = lock(&self.accounts[hi]);
        let (src, dst) = if from < to {
            (&mut *glo, &mut *ghi)
        } else {
            (&mut *ghi, &mut *glo)
        };
        if *src < amount {
            return Err(TransferError::Insufficient {
                needed: amount,
                available: *src,
            });
        }
        // Checked on both sides so a panic can never leave money half-moved.
        let new_dst = dst.checked_add(amount).expect("balance overflow");
        *src -= amount;
        *dst = new_dst;
        Ok(())
    }

    /// Sum of all balances. Locks every account in index order, so the total
    /// is a consistent snapshot.
    pub fn total(&self) -> u64 {
        let guards: Vec<_> = self.accounts.iter().map(lock).collect();
        guards.iter().map(|g| **g).sum()
    }
}

// ------------------------------------------------------------ LeveledMutex

thread_local! {
    /// Levels of the `LeveledMutex`es this thread currently holds.
    static HELD: RefCell<Vec<u32>> = const { RefCell::new(Vec::new()) };
}

/// Returned when a thread tries to take a lock whose level is not strictly
/// greater than every level it already holds.
#[derive(Debug, PartialEq, Eq)]
pub struct LevelViolation {
    pub wanted: u32,
    pub held_max: u32,
}

impl fmt::Display for LevelViolation {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(
            f,
            "lock order violation: wanted level {} while holding level {}",
            self.wanted, self.held_max
        )
    }
}

impl std::error::Error for LevelViolation {}

/// A mutex with a *level*. A thread may only acquire locks in strictly
/// increasing level order; breaking the rule is reported instead of risking a
/// deadlock. The check is per thread and costs a thread-local access.
///
/// ```
/// use locklab::LeveledMutex;
/// let low = LeveledMutex::new(1, "config");
/// let high = LeveledMutex::new(2, "cache");
/// let _a = low.lock().unwrap();
/// let _b = high.lock().unwrap();      // 1 then 2: allowed
/// ```
pub struct LeveledMutex<T> {
    level: u32,
    inner: Mutex<T>,
}

/// Proof of holding a [`LeveledMutex`]. It is `!Send` (it wraps a
/// `MutexGuard`), which this design needs: the held-levels list is
/// per thread, so the guard must be dropped on the thread that created it.
///
/// ```compile_fail
/// let m = locklab::LeveledMutex::new(1, 0);
/// let g = m.lock().unwrap();
/// std::thread::spawn(move || { let _ = &g; });
/// ```
pub struct LeveledGuard<'a, T> {
    guard: MutexGuard<'a, T>,
    level: u32,
}

impl<T> LeveledMutex<T> {
    pub fn new(level: u32, value: T) -> Self {
        Self {
            level,
            inner: Mutex::new(value),
        }
    }

    pub fn lock(&self) -> Result<LeveledGuard<'_, T>, LevelViolation> {
        HELD.with(|h| {
            let mut h = h.borrow_mut();
            if let Some(&held_max) = h.iter().max()
                && self.level <= held_max
            {
                return Err(LevelViolation {
                    wanted: self.level,
                    held_max,
                });
            }
            // The check passed, so blocking here cannot close a cycle with
            // other threads that follow the same rule.
            let guard = lock(&self.inner);
            h.push(self.level);
            Ok(LeveledGuard {
                guard,
                level: self.level,
            })
        })
    }
}

impl<T> Deref for LeveledGuard<'_, T> {
    type Target = T;
    fn deref(&self) -> &T {
        &self.guard
    }
}

impl<T> DerefMut for LeveledGuard<'_, T> {
    fn deref_mut(&mut self) -> &mut T {
        &mut self.guard
    }
}

impl<T> Drop for LeveledGuard<'_, T> {
    fn drop(&mut self) {
        // Guards may be dropped in any order: remove *this* level.
        HELD.with(|h| {
            let mut h = h.borrow_mut();
            if let Some(pos) = h.iter().rposition(|&l| l == self.level) {
                h.remove(pos);
            }
        });
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::collections::HashSet;
    use std::sync::atomic::{AtomicUsize, Ordering};
    use std::sync::{Arc, mpsc};
    use std::thread;

    /// Runs `f` on a thread and fails (instead of hanging the test run) if it
    /// does not finish in time. The stuck thread is leaked.
    fn within<R: Send + 'static>(secs: u64, f: impl FnOnce() -> R + Send + 'static) -> R {
        let (tx, rx) = mpsc::channel();
        thread::spawn(move || {
            let _ = tx.send(f());
        });
        rx.recv_timeout(Duration::from_secs(secs))
            .expect("timed out: probable deadlock")
    }

    #[test]
    fn queue_fifo_capacity_and_close() {
        let q = BoundedQueue::new(2);
        q.push(1).unwrap();
        q.try_push(2).unwrap();
        assert_eq!(q.try_push(3), Err(PushError::Full(3)));
        assert_eq!(q.len(), 2);
        assert_eq!(q.try_pop(), Some(1));
        q.close();
        assert_eq!(q.push(9), Err(PushError::Closed(9)));
        assert_eq!(q.pop(), Some(2));
        assert_eq!(q.pop(), None);
        assert_eq!(q.try_pop(), None);
        assert!(q.is_empty());
    }

    #[test]
    fn queue_never_exceeds_capacity_under_contention() {
        within(20, || {
            let q = Arc::new(BoundedQueue::new(2));
            let stop = Arc::new(std::sync::atomic::AtomicBool::new(false));
            let sampler = {
                let (q, stop) = (Arc::clone(&q), Arc::clone(&stop));
                thread::spawn(move || {
                    let mut peak = 0;
                    while !stop.load(Ordering::Relaxed) {
                        peak = peak.max(q.len());
                    }
                    peak
                })
            };
            let producers: Vec<_> = (0..6)
                .map(|_| {
                    let q = Arc::clone(&q);
                    thread::spawn(move || {
                        for i in 0..2000 {
                            q.push(i).unwrap();
                        }
                    })
                })
                .collect();
            let consumer = {
                let q = Arc::clone(&q);
                thread::spawn(move || {
                    let mut n = 0;
                    while n < 12_000 {
                        if q.pop().is_some() {
                            n += 1;
                        }
                    }
                })
            };
            for p in producers {
                p.join().unwrap();
            }
            consumer.join().unwrap();
            stop.store(true, Ordering::Relaxed);
            assert!(sampler.join().unwrap() <= 2, "queue exceeded its capacity");
        });
    }

    #[test]
    fn push_timeout_expires_on_full_queue() {
        let q = BoundedQueue::new(1);
        q.push(1).unwrap();
        let start = Instant::now();
        let r = q.push_timeout(2, Duration::from_millis(60));
        assert_eq!(r, Err(PushError::Full(2)));
        assert!(start.elapsed() >= Duration::from_millis(50));
    }

    #[test]
    fn many_producers_and_consumers_lose_nothing() {
        within(20, || {
            let q = Arc::new(BoundedQueue::new(4));
            let producers: Vec<_> = (0..4u64)
                .map(|p| {
                    let q = Arc::clone(&q);
                    thread::spawn(move || {
                        for i in 0..500u64 {
                            q.push(p * 1000 + i).unwrap();
                        }
                    })
                })
                .collect();
            let consumers: Vec<_> = (0..3)
                .map(|_| {
                    let q = Arc::clone(&q);
                    thread::spawn(move || {
                        let mut got = Vec::new();
                        while let Some(v) = q.pop() {
                            got.push(v);
                        }
                        got
                    })
                })
                .collect();
            for p in producers {
                p.join().unwrap();
            }
            q.close();
            let mut all = HashSet::new();
            let mut n = 0;
            for c in consumers {
                for v in c.join().unwrap() {
                    n += 1;
                    all.insert(v);
                }
            }
            assert_eq!((n, all.len()), (2000, 2000));
        });
    }

    #[test]
    fn close_wakes_blocked_consumers_and_producers() {
        within(10, || {
            let q = Arc::new(BoundedQueue::<u8>::new(1));
            let c = {
                let q = Arc::clone(&q);
                thread::spawn(move || q.pop())
            };
            thread::sleep(Duration::from_millis(50));
            q.close();
            assert_eq!(c.join().unwrap(), None);

            let q = Arc::new(BoundedQueue::new(1));
            q.push(1).unwrap();
            let p = {
                let q = Arc::clone(&q);
                thread::spawn(move || q.push(2))
            };
            thread::sleep(Duration::from_millis(50));
            q.close();
            assert_eq!(p.join().unwrap(), Err(PushError::Closed(2)));
        });
    }

    #[test]
    #[should_panic(expected = "capacity")]
    fn zero_capacity_panics() {
        let _ = BoundedQueue::<u8>::new(0);
    }

    #[test]
    fn semaphore_limits_concurrency() {
        within(20, || {
            let sem = Arc::new(Semaphore::new(3));
            let now = Arc::new(AtomicUsize::new(0));
            let peak = Arc::new(AtomicUsize::new(0));
            let hs: Vec<_> = (0..8)
                .map(|_| {
                    let (sem, now, peak) = (Arc::clone(&sem), Arc::clone(&now), Arc::clone(&peak));
                    thread::spawn(move || {
                        for _ in 0..50 {
                            let _p = sem.acquire();
                            let n = now.fetch_add(1, Ordering::SeqCst) + 1;
                            peak.fetch_max(n, Ordering::SeqCst);
                            thread::yield_now();
                            now.fetch_sub(1, Ordering::SeqCst);
                        }
                    })
                })
                .collect();
            for h in hs {
                h.join().unwrap();
            }
            assert!(peak.load(Ordering::SeqCst) <= 3);
            assert_eq!(sem.available(), 3);
        });
    }

    #[test]
    fn semaphore_try_acquire() {
        let s = Semaphore::new(1);
        let p = s.try_acquire().unwrap();
        assert!(s.try_acquire().is_none());
        drop(p);
        assert_eq!(s.available(), 1);
    }

    #[test]
    fn bank_opposite_transfers_do_not_deadlock_and_conserve_money() {
        within(20, || {
            let bank = Arc::new(Bank::new(&[1000, 1000, 1000]));
            let hs: Vec<_> = (0..4usize)
                .map(|i| {
                    let bank = Arc::clone(&bank);
                    thread::spawn(move || {
                        for k in 0..20_000usize {
                            let (a, b) = if (i + k) % 2 == 0 { (0, 1) } else { (1, 0) };
                            let _ = bank.transfer(a, b, 1);
                            let _ = bank.transfer(2, i % 2, 1);
                        }
                    })
                })
                .collect();
            for h in hs {
                h.join().unwrap();
            }
            assert_eq!(bank.total(), 3000);
        });
    }

    #[test]
    fn bank_errors_and_self_transfer() {
        let b = Bank::new(&[10, 5]);
        assert_eq!(
            b.transfer(0, 1, 11),
            Err(TransferError::Insufficient {
                needed: 11,
                available: 10
            })
        );
        assert_eq!(b.transfer(0, 7, 1), Err(TransferError::NoSuchAccount(7)));
        assert_eq!(b.transfer(0, 0, 3), Ok(()));
        assert!(b.transfer(0, 0, 11).is_err());
        assert_eq!(b.transfer(1, 0, 5), Ok(()));
        assert_eq!((b.balance(0), b.balance(1)), (Some(15), Some(0)));
        assert_eq!(b.balance(9), None);
        assert_eq!(
            b.transfer(0, 9, 1).unwrap_err().to_string(),
            "no such account 9"
        );
    }

    #[test]
    fn leveled_mutex_enforces_increasing_order() {
        let a = LeveledMutex::new(1, 0);
        let b = LeveledMutex::new(2, 0);
        {
            let _ga = a.lock().unwrap();
            let _gb = b.lock().unwrap();
        }
        let gb = b.lock().unwrap();
        let err = a.lock().err().unwrap();
        assert_eq!(
            err,
            LevelViolation {
                wanted: 1,
                held_max: 2
            }
        );
        assert!(err.to_string().contains("wanted level 1"));
        // same level twice is also rejected: it would be a self-deadlock
        assert!(b.lock().is_err());
        drop(gb);
        // after release the order is free again
        let _ga = a.lock().unwrap();
    }

    #[test]
    fn leveled_guards_may_drop_in_any_order() {
        let a = LeveledMutex::new(1, 1);
        let b = LeveledMutex::new(2, 2);
        let ga = a.lock().unwrap();
        let gb = b.lock().unwrap();
        drop(ga); // not LIFO
        assert!(a.lock().is_err(), "still holding level 2");
        drop(gb);
        assert!(a.lock().is_ok());
    }

    #[test]
    fn leveled_state_is_per_thread() {
        let b = Arc::new(LeveledMutex::new(2, 0));
        let a = Arc::new(LeveledMutex::new(1, 0));
        let _gb = b.lock().unwrap();
        let a2 = Arc::clone(&a);
        // another thread holds nothing, so it may take level 1
        assert!(thread::spawn(move || a2.lock().is_ok()).join().unwrap());
    }

    #[test]
    fn poisoned_state_is_still_usable() {
        let q = Arc::new(BoundedQueue::new(2));
        let q2 = Arc::clone(&q);
        let _ = thread::spawn(move || {
            let _g = q2.state.lock().unwrap();
            panic!("poison the queue lock");
        })
        .join();
        assert!(q.state.is_poisoned());
        q.push(1).unwrap();
        assert_eq!(q.pop(), Some(1));
    }
}
```

**Verification ✓.** `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings` and `cargo test --locked` pass: 14 unit tests, 3 doctests (one is `compile_fail`: `LeveledGuard` is `!Send`, error E0277 ✓ confirmed from an external crate).

**Mutation checks ✓** (each mutation applied to a copy; unit tests run with a 90-second limit):

| # | Mutation | Result |
|---|---|---|
| M1 | `Bank::transfer`: lock `from` then `to` (no global order) | **caught**: the opposite-transfers test timed out "probable deadlock" |
| M2 | `push`: `while` full → `if` full | **caught** by `queue_never_exceeds_capacity_under_contention` (3 of 3 runs; one run also showed a downstream "Disconnected"). **My first 13-test suite did not catch it** (13 passed); the capacity-sampling test was added because of this |
| M3 | `close`: `notify_all` → `notify_one` | **caught**: consumer test timed out |
| M4 | `LeveledMutex`: `<=` → `<` (same level allowed) | **caught by hanging**: the test that relocks the same level self-deadlocked and the run was killed by the time limit (exit 124). A hang is a failure, but not a nice one: that test should use the watchdog too |
| M5 | `pop`: no `not_full.notify_one()` | **caught**: producers stayed blocked, timeout |
| M6 | `SemPermit::drop`: no `notify_one()` | **caught**: semaphore test timed out |
| M7 | `Bank::transfer`: remove the `from == to` special case | **caught by hanging** (self-deadlock; exit 124) |

All seven mutants were detected. The honest caveats: M4 and M7 are detected only by a run being killed, which in a CI job means a slow failure; and the stress tests are probabilistic, so "3 of 3 runs" is evidence, not proof.

**Design notes.**

- Every critical section leaves the data valid at each point that could panic (`Bank::transfer` computes `checked_add` *before* changing either balance), so the crate's `lock` helper **recovers from poisoning** with `into_inner()`. That is a *policy* choice and is documented on the helper. A crate whose invariants can be half-updated should propagate the poison instead.
- `push` releases the mutex (`drop(st)`) before `notify_one`, so the woken consumer can take the lock immediately.
- `push_timeout` computes a deadline once and waits for the remainder (`saturating_duration_since`), so spurious wake-ups do not extend the total wait.
- Two condvars (`not_empty`, `not_full`) on one mutex let each notification target the right kind of waiter.
- `Bank::total` takes every lock in index order, the same global order as `transfer`, so it can never deadlock with transfers and returns a consistent snapshot.
- `LeveledMutex` holds a thread-local `Vec<u32>`; `Drop` removes *that guard's* level, so out-of-order drops work. The guard is `!Send` because the bookkeeping is per thread.
- Not handled: fairness (any waiter may win), priority inheritance, and `LeveledMutex` has no `try_lock`.

</details>

---

## 13. Interview questions

1. What are the necessary conditions for a deadlock, and which one do lock hierarchies break?
2. Why is `Mutex::lock` called twice on one thread dangerous, and what does `std` promise?
3. Why does a `Condvar` need a `Mutex` and a loop?
4. `notify_one` vs `notify_all`: when is each correct?
5. Can safe Rust deadlock? Can it have a data race?
6. How would you add a timeout to acquiring a `std::sync::Mutex`?
7. When is `RwLock` better than `Mutex`, and what are its pitfalls?
8. You have a service where a handler holds `state` and calls the mempool, and the mempool holds its lock and calls `state`. Diagnose it and fix it.

<details markdown="1">
<summary>Answers</summary>

1. Mutual exclusion, hold-and-wait, no preemption, circular wait. A global lock order (hierarchy) breaks **circular wait**.
2. `std` does not define it: the docs say the call might panic or deadlock (from memory). ✓ On this Linux toolchain it hung. `std::sync::Mutex` is not reentrant.
3. The mutex protects the state the condition depends on, and `wait` atomically releases it while sleeping, so the notifier can change the state without a lost-wakeup window. The loop handles spurious and stolen wake-ups (§4.1).
4. `notify_one` when exactly one waiter can make progress and all waiters are interchangeable (one item added). `notify_all` when the state change may let several proceed, when waiters wait for different conditions on one condvar, or for shutdown/broadcast.
5. Yes to deadlock (✓ §3.2); no to data race (in safe code, Lessons 22 and 24).
6. `std` has no `lock_timeout`. Options: loop on `try_lock` with a deadline and a short sleep/`yield_now`; use a `Mutex<T>` + `Condvar` of your own; or use a crate whose mutex offers timeouts (`parking_lot` has `try_lock_for`; from memory, not tested).
7. Reads dominate and the read section is long enough to benefit from parallelism. Pitfalls: read→write upgrade self-deadlock (✓), implementation-defined starvation policy, recursive read locking hazards, and more overhead than `Mutex` for short sections.
8. Circular wait via opposite lock orders. Fix: define a hierarchy (state, then mempool) and never call "up"; or release the first lock before calling into the other component (copy out what you need); or make the interaction asynchronous through a bounded queue.

</details>

---

## 14. Key takeaways

- A lock prevents data races; it does not prevent logic races or deadlock. Safe Rust can deadlock.
- Four conditions make a deadlock; breaking **circular wait** with a global order is the standard fix (✓ opposite-order deadlock, ✓ ordered version completes). Enforce the order in code, not in comments (`LeveledMutex`).
- Self-deadlock: `std` locks are not reentrant (✓ hangs); a guard in a `match`/`if let` scrutinee stays alive for the whole body (✓); `RwLock` has no upgrade (✓).
- `Condvar`: state inside the mutex, wait in a loop (or `wait_while`), notify after changing state, `notify_all` for shutdown. Notifications are not remembered (✓).
- Never hold a lock across blocking or unknown code; do work outside, publish inside.
- Test with a watchdog and invariant assertions; a mutant can pass a weak suite (✓ M2 did until I added the capacity-sampling test).
- Prefer message passing or a single lock until a profile says otherwise.

**Connections.** Lesson 24 introduced `Mutex`, `RwLock` and poisoning; Lesson 27's pool shutdown is the `Condvar`/`close` protocol seen again; Lesson 28's Acquire/Release is what a lock *is*. Next: Lesson 30 starts `async`: futures, `await`, and executors, which replace blocking waits with tasks (and bring a new rule: never hold a `std` `MutexGuard` across an `.await`, Lesson 31).

---

## 15. References

- `std::sync::Mutex`, `Condvar`, `RwLock`, `Barrier`, `TryLockError`: <https://doc.rust-lang.org/std/sync/index.html>
- `Mutex::lock` (behavior when the current thread already holds the lock) and `Condvar::wait` (spurious wake-ups): <https://doc.rust-lang.org/std/sync/struct.Mutex.html> and <https://doc.rust-lang.org/std/sync/struct.Condvar.html>
- *The Rust Programming Language*, ch. 16.3 "Shared-State Concurrency": <https://doc.rust-lang.org/book/ch16-03-shared-state.html>
- *The Rustonomicon*, "Concurrency": <https://doc.rust-lang.org/nomicon/concurrency.html>
- Mara Bos, *Rust Atomics and Locks* (ch. 9 "Building Our Own Locks", which builds `Mutex`, `Condvar` and `RwLock`): <https://marabos.nl/atomics/>
- Coffman, Elphick, Shoshani, "System Deadlocks" (1971), the source of the four conditions (from memory).

The URLs above are the standard documentation locations; I did not open them during this session (the sandbox cannot reach them), so if a link has moved, search the docs site for the page title. Unchecked or from memory: the exact wording of the `Mutex::lock` documentation about panicking or deadlocking; the reason `MutexGuard` is `!Send`; that using different mutexes with one `Condvar` panics; `RwLock` fairness, starvation and recursive-read behavior; the Coffman conditions; the edition-2024 change to `if let` temporaries (not tested); that `parking_lot` has a deadlock detector and `try_lock_for`; priority inheritance; and the address-ordering technique. The debugging challenge's original program was run once and hung; the lost-notification window in it was not reproduced. The fixed program, the watchdog demos, the ordered-transfer program and all crate results were run on this machine's `rustc 1.97.0`.
