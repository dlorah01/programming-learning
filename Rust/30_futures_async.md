---
layout: default
title: "Lesson 30 — Futures, async/await, and executors"
---

# Lesson 30 — Futures, `async`/`await`, and executors

**Module 10 · Concurrency and async** · Status: generated · Edition used in examples: **2024** · External dependencies: none (standard library only; the practical crate is also `std`-only and contains no `unsafe`)

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux with 2 cores. As before, ✓ marks outputs, diagnostics and measurements I actually ran; unmarked statements are documented behavior I did not run. Everything is **stable Rust**. This lesson deliberately builds the machinery by hand with only `std`, so you see what Tokio (Lesson 31) automates. Where a demo could hang, it runs behind a watchdog (a channel with `recv_timeout`).

Builds on [Lesson 25](25_weak_pin.md) (`Pin`, `Unpin`, async blocks are `!Unpin`), [Lesson 27](27_threads_channels.md) (threads, channels), [Lesson 28](28_atomics_memory_ordering.md) (the `unpark` token is an atomic hand-off) and [Lesson 29](29_locks_deadlocks.md) (blocking waits, `Condvar`). Tokio, cancellation and streams are Lessons 31 and 32.

---

## 1. What problem `async` solves

A thread (Lesson 27) is an OS resource: it has its own stack and the kernel schedules it. That is fine for dozens or hundreds of threads. A server with a hundred thousand mostly-idle connections wants something lighter: **many tasks multiplexed on few threads**, where a task that must wait (for a socket, a timer, a message) *gives up its thread* instead of blocking it.

Rust's answer:

- `async fn` and `async { }` blocks compile to **state machines** that implement the `Future` trait. A future is a value describing a computation that may not be finished.
- `.await` means "poll this future; if it is not ready, suspend *me* and let the thread run something else."
- An **executor** (runtime) owns the threads, polls tasks, and sleeps when none can progress.
- A **waker** is the callback a future uses to tell the executor "poll me again, something changed."

Two design decisions to know. First, Rust futures are **lazy** and **poll-based**: nothing runs until something polls them, and the executor pulls results, rather than futures pushing callbacks. Second, **the language and `std` give you the `Future` trait and `async`/`await` syntax but no executor**; runtimes live in crates (Tokio, async-std, smol; from memory, with Tokio dominant). That keeps `std` small and lets embedded systems bring their own executor, at the price that you must choose one.

`async` is not "faster threads". It helps when you have many tasks that spend most of their time waiting. For CPU-bound work, threads (or a thread pool, Lesson 27) are the right tool.

---

## 2. The `Future` trait

```rust,ignore
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

pub enum Poll<T> { Ready(T), Pending }
```

(That is the shape of the real definition in `core::future`; I wrote it out from the documented API, not from the sandbox.)

The contract of `poll`:

1. Try to make progress. If the result is available, return `Poll::Ready(value)`. **Never poll a future again after it returned `Ready`**: the behavior is unspecified; async blocks panic (✓ `` `async fn` resumed after completion``).
2. If you cannot finish, return `Poll::Pending` **after arranging for `cx.waker()` to be called when progress becomes possible.** If you return `Pending` and nobody will ever call the waker, the task is never polled again: a **lost wake-up**, the async version of a deadlock (✓ §3.3).
3. `poll` must not block. A blocking `poll` freezes the executor thread and every task on it (✓ §8).

`Context` carries a `&Waker`. A `Waker` is a cloneable, `Send + Sync` handle with `wake()` (consume) and `wake_by_ref()`. The executor creates it so that `wake()` re-queues the right task.

### 2.1 Why `Pin<&mut Self>`

A compiled async block may hold a reference to one of its own locals across an `.await`: it is a **self-referential** struct. Moving such a value would leave the reference dangling. `Pin` (Lesson 25) is the type-level promise "this value will not be moved again until dropped". `poll` takes `Pin<&mut Self>` so that a future is pinned once it starts being polled.

You see this at once if you try to call `poll` on an unpinned value ✓:

```rust,ignore
let mut s = S;                       // S implements Future
s.poll(&mut cx);                     // ✓ error[E0599]: no method named `poll` found for struct `S`
                                     //   help: consider pinning the expression: `std::pin::pin!(s)`
```

and the compiler confirms that the state machine of an `async fn` which borrows its own local across an `.await` is not `Unpin` ✓ (`error[E0277]: `{async fn body of borrow_across()}` cannot be unpinned`, "consider using the `pin!` macro"). `Box::pin(fut)` pins on the heap; `std::pin::pin!(fut)` pins on the stack.

---

## 3. A future and an executor from scratch

### 3.1 The smallest executor: `block_on`

```rust
use std::future::Future;
use std::pin::{Pin, pin};
use std::sync::Arc;
use std::task::{Context, Poll, Wake, Waker};
use std::thread::{self, Thread};

struct ThreadWaker(Thread);
impl Wake for ThreadWaker {
    fn wake(self: Arc<Self>) { self.0.unpark(); }
}

fn block_on<F: Future>(fut: F) -> F::Output {
    let mut fut = pin!(fut);
    let waker = Waker::from(Arc::new(ThreadWaker(thread::current())));
    let mut cx = Context::from_waker(&waker);
    loop {
        match fut.as_mut().poll(&mut cx) {
            Poll::Ready(v) => return v,
            Poll::Pending => thread::park(),
        }
    }
}

struct Countdown { n: u32, polls: u32 }
impl Future for Countdown {
    type Output = u32;
    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<u32> {
        self.polls += 1;
        if self.n == 0 {
            Poll::Ready(self.polls)
        } else {
            self.n -= 1;
            cx.waker().wake_by_ref(); // "poll me again"
            Poll::Pending
        }
    }
}

async fn add(a: u32, b: u32) -> u32 { a + b }

async fn run() -> u32 {
    let x = add(1, 2).await;
    let polls = Countdown { n: 3, polls: 0 }.await;
    x * 100 + polls
}

fn main() {
    println!("{}", block_on(add(20, 22)));
    println!("{}", block_on(run()));
    let fut = async { println!("body runs"); 7 };
    println!("created, not run yet");
    println!("{}", block_on(fut));
    let f2 = async { println!("never printed"); };
    drop(f2);
    println!("dropped");
    let mut cx = Context::from_waker(Waker::noop());
    let mut p = pin!(std::future::ready(5));
    println!("{:?}", p.as_mut().poll(&mut cx));
    let mut c = 0;
    let mut pf = pin!(std::future::poll_fn(move |_cx| { c += 1; if c < 3 { Poll::Pending } else { Poll::Ready(c) } }));
    println!("{:?} {:?} {:?}", pf.as_mut().poll(&mut cx), pf.as_mut().poll(&mut cx), pf.as_mut().poll(&mut cx));
}
```

✓ Output:

```text
42
304
created, not run yet
body runs
7
dropped
Ready(5)
Pending Pending Ready(3)
```

Reading it:

- `Wake` is the safe way to make a `Waker`: implement `wake(self: Arc<Self>)` and convert with `Waker::from(Arc<W>)`. (The raw `RawWaker` vtable API is `unsafe`; you rarely need it.)
- `block_on` parks the thread while `Pending`; `wake` unparks it. `unpark` leaves a **token** if the thread is not yet parked, so a wake that arrives *between* `poll` returning and `park` is not lost. `park` can also return spuriously, which is harmless because the loop polls again.
- `run()` returned `304`: `add` gave 3, and `Countdown { n: 3 }` was polled four times (three `Pending`, then `Ready(4)`). **`.await` is a loop around `poll`** inside the state machine: when the inner future is `Pending`, the outer one returns `Pending` too, and when it is polled again it resumes at the same `.await`.
- ✓ Futures are **lazy**: `"created, not run yet"` printed before `"body runs"`, and a dropped, never-polled future (`f2`) ran nothing.
- `std::future::ready(v)` is an already-complete future; `std::future::poll_fn(|cx| …)` builds a future from a closure; `Waker::noop()` is a waker that does nothing (stable since 1.85; from memory, but ✓ it compiled and ran here), handy for tests.

### 3.2 What `await` desugars to

Conceptually (this is the idea, not the exact compiler output):

```rust,ignore
// let v = inner.await;   becomes roughly
let mut inner = pin!(inner.into_future());
let v = loop {
    match inner.as_mut().poll(cx) {
        Poll::Ready(v) => break v,
        Poll::Pending => yield_pending(),   // return Pending from *our* poll, remembering where we were
    }
};
```

The compiler turns each `async` body into an enum-like **state machine**: one variant per suspension point, holding exactly the locals that live across that `.await`. That explains the sizes ✓:

| Future | `size_of_val` ✓ |
|---|---|
| `async fn small() -> u8 { 1 }` | 1 byte |
| `async { 1 }` block | 1 byte |
| `async fn` that keeps `[0u8; 1000]` alive across an `.await` | **1002 bytes** |
| same, but copies out the byte and lets the array die before the `.await` | **3 bytes** |
| `std::future::ready(1u64)` | 16 bytes |

Practical consequences: (a) futures are plain values, usually allocated on the stack or in one `Box` per task, with no hidden heap use per `await`; (b) **what you keep alive across an `.await` decides the size of the task** (large arrays or buffers held across `.await` make big futures that get moved around; keep them boxed or scope them out); (c) a future must be pinned before its first poll, but until then it can be moved freely.

### 3.3 The classic bug: `Pending` without a waker

```rust,ignore
struct FlagFuture(Arc<AtomicBool>);
impl Future for FlagFuture {
    type Output = ();
    fn poll(self: Pin<&mut Self>, _cx: &mut Context<'_>) -> Poll<()> {
        if self.0.load(Ordering::Acquire) { Poll::Ready(()) } else { Poll::Pending }   // never stores the waker!
    }
}
```

✓ With another thread setting the flag after 50 ms and `block_on` as above, the task never completed (watchdog: `Err(Timeout)` after 2 seconds). `block_on` parked after the first `Pending`, and nothing ever unparked it. The fix is in §11.

---

## 4. `async fn`, `async` blocks, and `.await`

- `async fn f(args) -> T` is sugar for `fn f(args) -> impl Future<Output = T>`; **calling it runs none of the body** ✓ (`work();` alone gives `warning: unused implementer of `Future` that must be used`, note "futures do nothing unless you `.await` or poll them").
- `async { … }` creates an anonymous future; `async move { … }` moves captured variables in.
- `.await` is only allowed inside async context ✓: `error[E0728]: `await` is only allowed inside `async` functions and blocks`.
- A future is not its output ✓: `let n: i32 = f() + 1;` gives `error[E0369]: cannot add `{integer}` to `impl Future<Output = i32>``. Forgetting `.await` is the most common beginner error and, thanks to those diagnostics, a cheap one.
- Errors with `?` work inside async functions exactly as in sync ones (Lesson 26); the future's `Output` is the `Result`.
- **Dropping a future cancels it.** There is no `cancel()`: when a future is dropped, its state machine is dropped, destroying whatever locals were alive at the last suspension point, and nothing after that `.await` ever runs. This is powerful and dangerous; Lesson 32 is about doing it safely.
- **Polling a finished future panics** ✓ (`` `async fn` resumed after completion``) for async blocks; other futures may do anything.

### 4.1 Recursion needs boxing ✓

```rust,ignore
async fn fact(n: u64) -> u64 {
    if n == 0 { 1 } else { n * fact(n - 1).await }
}
// ✓ error[E0733]: recursion in an async fn requires boxing
//   note: a recursive `async fn` call must introduce indirection such as `Box::pin`
//         to avoid an infinitely sized future
```

The state machine of `fact` would contain itself. Introduce a pointer:

```rust
use std::future::Future;
use std::pin::Pin;
fn fact(n: u64) -> Pin<Box<dyn Future<Output = u64> + Send>> {
    Box::pin(async move { if n == 0 { 1 } else { n * fact(n - 1).await } })
}
```

✓ Computes `3628800` for `fact(10)` (run with a trivial executor using `Waker::noop`). Every recursion level now allocates once.

### 4.2 `async fn` in traits and async closures ✓

Both work on stable Rust 1.97:

```rust
trait Store { async fn get(&self, k: u32) -> Option<String>; }
struct Mem;
impl Store for Mem {
    async fn get(&self, k: u32) -> Option<String> { (k < 3).then(|| format!("v{k}")) }
}
async fn use_store<S: Store>(s: &S) -> Option<String> { s.get(1).await }
// ...
let add = async |a: u32, b: u32| a + b;   // async closure
```

✓ These printed `Some("v1")` and `5`. A known limitation (from memory; I did not reproduce it here): a generic function that needs `S::get(..)` to return a `Send` future so that it can be `spawn`ed has trouble naming that bound for an `async fn` in a trait; the documented workarounds are return-position `impl Future + Send` in the trait, or the `trait_variant` crate. Dyn-compatibility of traits with `async fn` is also restricted (from memory).

---

## 5. `Send`, `'static`, and why spawning is picky

A multithreaded executor moves tasks between threads, so spawned futures must be `Send + 'static`. A future is `Send` only if **everything held across an `.await`** is `Send`.

```rust,ignore
spawn_like(async move {
    let _r = &rc;            // Rc<i32>
    tick().await;
});
// ✓ error: future cannot be sent between threads safely
//   = help: within `{async block}`, the trait `Send` is not implemented for `Rc<i32>`
//   note: captured value is not `Send`
```

and the most common real-world version, a `std::sync::MutexGuard` held across `.await`:

```rust,ignore
spawn_like(async {
    let g = M.lock().unwrap();
    tick().await;                    // guard is still alive here
    let _ = *g;
});
// ✓ error: future cannot be sent between threads safely
//   note: future is not `Send` as this value is used across an await
//   `g` has type `std::sync::MutexGuard<'_, i32>` which is not `Send`  ... `await occurs here, with `g` maybe used later`
```

This reuses Lessons 22 and 29: `MutexGuard` is `!Send`. The compiler error is a *gift*: holding a `std` mutex across an `.await` can **deadlock the executor** (the task is suspended while holding the lock; another task on the same thread tries to take it and blocks the thread, so the first task can never be polled to release it). Fixes: drop the guard before the `.await` (scope it in a block), copy the data out, or use an async-aware mutex (Lesson 31).

---

## 6. Executors: tasks, queues, wakers

A task is a pinned, boxed future plus the means to re-queue itself. The minimal design (implemented in the practical crate, ~100 lines, no `unsafe`):

1. `spawn(fut)` boxes and pins the future, wraps it in `Arc<Task>`, and sends the `Arc<Task>` into a queue.
2. `run()` loops: receive a task, build a `Waker` whose `wake()` sends the same `Arc<Task>` back to the queue, `poll` the future with that waker; if `Pending`, put the future back and wait for the next wake; if `Ready`, the task is done.
3. A leaf future (timer, socket) stores the **latest** waker it was polled with and calls it when the event happens, possibly from another thread.

The consequences you should know:

- **Wake-ups are hints, not guarantees of progress**: a task can be polled when it still cannot progress (it will return `Pending` again). A task woken twice may be queued twice. Futures must tolerate extra polls.
- **Store the latest waker, not the first.** The waker may change between polls (e.g. the future was moved into a different task, as with `join` or `select`). Using a stale waker wakes the wrong task. The crate has a test that polls a `Sleep` with two different wakers and checks only the second is woken (✓; and a mutant that only stores the first waker fails it).
- **A `JoinHandle` is itself a future** that completes when the task does; the task, on completion, wakes whoever awaits the handle.
- **Dropping a `JoinHandle` detaches the task** in this design (as in Tokio, from memory); it does not cancel it. Contrast with dropping a *future*, which does cancel it.
- A task that panics unwinds through `poll`; what happens next is executor policy (Tokio catches it and reports it through the `JoinHandle`; from memory. The lesson's mini executor lets the panic propagate out of `run`).

### 6.1 Single-threaded vs multi-threaded, and work stealing

A single-threaded executor never needs `Send` futures (Tokio offers a current-thread runtime with local tasks; from memory), and is simpler and deterministic. A multi-threaded executor runs one polling loop per worker with per-worker queues and **work stealing** to balance load (from memory; Tokio's scheduler is of this kind). The `Send + 'static` bound in §5 comes from the multi-threaded design.

---

## 7. Combining futures: `join`

`join2(a, b)` polls both futures each time it is polled and completes when both are done. Both make progress **concurrently, on one thread**: concurrency, not parallelism. The practical crate's version ✓ interleaves two tasks that yield twice as `a bb a bb`. It must (1) remember each finished output, (2) never poll a finished future again (✓ a test panics if it does), and (3) pass the same `cx` down so wake-ups reach the joined task. Real libraries (`futures::join!`, `tokio::join!`; from memory) also provide `select` (first to finish; the loser is **dropped**, i.e. cancelled), `try_join`, and streams (Lesson 32).

Contrast with sequential `.await`:

```rust,ignore
let a = fetch_a().await;   // starts fetch_b only after fetch_a is done
let b = fetch_b().await;
let (a, b) = join(fetch_a(), fetch_b()).await;   // both in flight at once
```

To run futures in *parallel* on several threads, spawn them as separate tasks on a multi-threaded runtime.

---

## 8. Blocking inside async

```rust
async fn slow_then_fast() -> u32 {
    std::thread::sleep(std::time::Duration::from_millis(100)); // blocks the thread
    1
}
```

A `poll` that blocks stalls the **whole executor thread**: every task scheduled on it waits. ✓ A sequence of two such calls took at least 200 ms and nothing else could have run meanwhile (the demo is single-threaded, so I measured only the lower bound `elapsed >= 200 ms`, which printed `true`). Use the runtime's async sleep/IO, or move blocking or CPU-heavy work to a dedicated thread (Tokio's `spawn_blocking`; from memory, Lesson 31). Blocking in `async` is the most common performance bug in async Rust code, and it does not show up as an error.

---

## 9. Common mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Calling an `async fn` and ignoring the result | Nothing runs (✓ `unused_must_use` warning) | `.await` it or spawn it |
| `Poll::Pending` without registering the waker | Task never polled again (✓ §3.3 hang) | Store `cx.waker().clone()` before returning `Pending` |
| Storing only the first waker | Wrong task woken after the future moves (✓ crate mutant) | Update the stored waker on every poll |
| Polling after `Ready` | Panic or unspecified behavior (✓ panic for async blocks) | Track completion (`Option<Output>`) |
| Holding `std::sync::MutexGuard` across `.await` | Not `Send` (✓ error), or executor deadlock | Scope the guard; async mutex |
| Holding `Rc`/`RefCell` borrow across `.await` in `spawn`ed code | Not `Send` (✓ error) | `Arc`, or drop before awaiting |
| Blocking calls in async code (`thread::sleep`, blocking I/O, heavy CPU) | Stalls the executor (✓ §8) | Async APIs or `spawn_blocking` |
| Recursive `async fn` | `E0733` (✓) | `Box::pin` |
| Holding big buffers across `.await` | Huge futures (✓ 1002 vs 3 bytes) | Scope them out, or `Box` them |
| Assuming `select`/drop is harmless | A cancelled future stops mid-way; partial work, held resources released only by `Drop` | Design for cancellation (Lesson 32) |
| Believing `join!` gives parallelism | It is concurrent on one task/thread | `spawn` for parallelism |
| Using `async` for CPU-bound work | No speedup, worse latency | Threads / rayon-style pools |

---

## 10. Performance and reliability notes

- **Cost model**: creating a future is usually just constructing a struct (✓ sizes in §3.2); each `.await` is a branch in a state machine, not a system call. The cost is in what the executor does: queues, wake-ups, atomics in wakers and `Arc<Task>` reference counts. I give no timing numbers here (a 2-core sandbox would not transfer); Lesson 41 shows how to benchmark.
- **Allocation**: one `Box` per spawned task (as in the crate), plus one per `Box::pin` recursion level. `join2` as written allocates two boxes to avoid `unsafe` pin-projection; production crates avoid them with pin-projection (`pin-project`-style code; from memory).
- **Fairness**: a task that never returns `Pending` (a long loop with no `.await`) starves its thread. `yield_now().await` (✓ in the crate: `Pending` once, then `Ready`) is the cooperative escape hatch; runtimes also add budgets (from memory).
- **Thread-per-sleep is for teaching.** The crate's `sleep` spawns an OS thread per timer (✓ 50 tasks sleeping 30 ms all finished); a real runtime keeps one timer structure for all timers.
- **Reliability**: lost wake-ups (§3.3), tasks waiting on each other forever, unbounded task spawning (memory exhaustion), and cancellation in the middle of a multi-step operation are the failure modes; none is a memory-safety bug, all are outages. Put timeouts on every external wait.
- **Security**: untrusted peers can open many connections or send slowly (Slowloris-style): bound the number of concurrent tasks, bound buffers, set read/handshake timeouts. A request handler that holds a global lock across `.await` lets one slow client stall everyone.
- **Blockchain-adjacent note**: nodes, indexers and relayers are I/O-bound systems with many peers and RPC calls in flight, which is exactly the async use case. They also have CPU-heavy parts (signature verification, hashing, state-trie updates) that must **not** run on the async threads; hand them to a blocking pool or dedicated threads, and keep deterministic consensus logic free of timers and scheduling effects (Lessons 49–50).

---

## 11. Exercises

Answers are collapsed. Try each before opening it.

1. **(Warm-up)** What does calling `async fn f() -> i32` do, and what type does it return?
2. **(Warm-up)** What must a future do before returning `Poll::Pending`, and why?
3. **(Easy)** Why does `poll` take `Pin<&mut Self>` rather than `&mut Self`?
4. **(Medium)** Write a future `Ready1`: `Ready1::new(n)` is `Pending` for the first `n` polls (waking itself each time) and then `Ready(n)`. Test it with `block_on`.
5. **(Medium)** With the crate's `join2`, two tasks each do `{ log(name); yield_now().await; log(name); }`. What is the log for names `a` and `b`? What would it be if the two were awaited one after the other?
6. **(Medium-hard)** This does not compile when spawned. Why, and give two fixes:
   ```rust,ignore
   spawn(async move {
       let rc = std::rc::Rc::new(5);
       yield_now().await;
       println!("{}", rc);
   });
   ```
7. **(Hard)** Make the debugging challenge's `FlagFuture` correct, and say what must be true of the setter side.
8. **(Hard)** Implement `timeout(fut, dur)` using the crate's `sleep`: it returns `Ok(output)` if `fut` finishes first and `Err(Elapsed)` otherwise. What happens to `fut` when the timer wins?

<details markdown="1">
<summary>Answers</summary>

1. It runs **nothing** of the body; it returns an anonymous future (`impl Future<Output = i32>`) holding the arguments. The body runs only when the future is polled (✓ §3.1 "created, not run yet").
2. Arrange for `cx.waker()` (cloned and stored) to be called when progress becomes possible. Otherwise nobody tells the executor to poll again (✓ §3.3).
3. An async state machine may contain references to its own fields, so it must not be moved once polling starts. `Pin` makes "will not move" part of the type; `&mut Self` would allow `mem::swap`/`mem::replace` and break the self-references (Lesson 25).
4. ```rust
   use std::future::Future;
   use std::pin::Pin;
   use std::task::{Context, Poll};
   struct Ready1 { n: u32, left: u32 }
   impl Ready1 { fn new(n: u32) -> Self { Ready1 { n, left: n } } }
   impl Future for Ready1 {
       type Output = u32;
       fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<u32> {
           if self.left == 0 { return Poll::Ready(self.n); }
           self.left -= 1;
           cx.waker().wake_by_ref();
           Poll::Pending
       }
   }
   ```
   With the `block_on` from §3.1: `block_on(Ready1::new(3))` returns `3` after three `Pending` polls. (✓ compiled and run in my check of this lesson's code.)
5. `a bb a bb`: `join2` polls `a` (logs, yields), then `b` (logs, yields), then re-polls `a` (logs), then `b` (logs). ✓ This exact sequence is the crate test `join2_interleaves_and_returns_both` (with the names `a` and `bb`). Awaited sequentially, the log would be `a a bb bb`.
6. `Rc` is not `Send` and it lives across the `.await`, so the future is not `Send` (✓ the same error as §5). Fixes: (a) use `Arc`; (b) end the `Rc`'s life before the `.await` by putting its use in an inner block `{ let rc = Rc::new(5); println!("{}", rc); }` and awaiting afterwards (if the value is needed afterwards, copy out what you need first).
7. See the debugging challenge's solution; the setter must set the flag **and then call `wake()` on the stored waker**. (Store before checking the flag, or check again after storing, to avoid a race in which the flag is set between your check and your store.)
8. See the companion code below. When the timer wins, `timeout` returns `Err(Elapsed)` and the inner future is simply never polled again and is dropped when the `Timeout` value is dropped: that is cancellation. Whatever the inner future had already done is not undone.
   ```rust
   use asynclab::{Sleep, sleep};
   use std::future::Future;
   use std::pin::Pin;
   use std::task::{Context, Poll};
   use std::time::Duration;

   #[derive(Debug, PartialEq, Eq)]
   pub struct Elapsed;

   pub struct Timeout<F: Future> { fut: Pin<Box<F>>, timer: Sleep }
   pub fn timeout<F: Future>(fut: F, dur: Duration) -> Timeout<F> {
       Timeout { fut: Box::pin(fut), timer: sleep(dur) }
   }
   impl<F: Future> Future for Timeout<F> {
       type Output = Result<F::Output, Elapsed>;
       fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
           let this = &mut *self;
           if let Poll::Ready(v) = this.fut.as_mut().poll(cx) { return Poll::Ready(Ok(v)); }
           if Pin::new(&mut this.timer).poll(cx).is_ready() { return Poll::Ready(Err(Elapsed)); }
           Poll::Pending
       }
   }
   ```
   (`Timeout` is `Unpin` because both fields are, so `&mut *self` is allowed. The order of the two checks matters: checking the future first means a future that finishes at the same moment as the timer wins.)

</details>

---

## 12. Debugging challenge

This future is supposed to complete when another thread sets a flag. It never does. (✓ I ran it behind a 2-second watchdog: `Err(Timeout)`.)

```rust
use std::future::Future;
use std::pin::Pin;
use std::sync::atomic::{AtomicBool, Ordering};
use std::sync::{mpsc, Arc};
use std::task::{Context, Poll, Wake, Waker};
use std::thread;
use std::time::Duration;

struct FlagFuture(Arc<AtomicBool>);
impl Future for FlagFuture {
    type Output = ();
    fn poll(self: Pin<&mut Self>, _cx: &mut Context<'_>) -> Poll<()> {
        if self.0.load(Ordering::Acquire) { Poll::Ready(()) } else { Poll::Pending }
    }
}

struct T(thread::Thread);
impl Wake for T { fn wake(self: Arc<Self>) { self.0.unpark(); } }

fn block_on<F: Future>(f: F) -> F::Output {
    let mut f = std::pin::pin!(f);
    let w = Waker::from(Arc::new(T(thread::current())));
    let mut cx = Context::from_waker(&w);
    loop {
        match f.as_mut().poll(&mut cx) {
            Poll::Ready(v) => return v,
            Poll::Pending => thread::park(),
        }
    }
}

fn main() {
    let flag = Arc::new(AtomicBool::new(false));
    let setter = flag.clone();
    let (tx, rx) = mpsc::channel();
    thread::spawn(move || {
        thread::sleep(Duration::from_millis(50));
        setter.store(true, Ordering::Release);
    });
    thread::spawn(move || {
        block_on(FlagFuture(flag));
        tx.send(()).unwrap();
    });
    println!("{:?}", rx.recv_timeout(Duration::from_secs(2)));
}
```

Why does it hang, and how do you fix it **correctly**, including the race a naive fix leaves open?

<details markdown="1">
<summary>Solution</summary>

`poll` returns `Pending` without storing the waker, and the setter only flips a flag. `block_on` parks after the first `Pending`; nobody ever calls `unpark`. The flag becoming `true` is invisible to a parked thread.

Fix: share a slot for the waker, store the waker in `poll`, and have the setter wake it after setting the flag. The subtle race: if `poll` checks the flag (`false`), the setter then sets the flag and looks for a waker (none yet), and only then does `poll` store its waker, the wake-up is lost. Avoid it by storing the waker **first** and checking the flag **after** (or doing both under one mutex, as below):

```rust
use std::future::Future;
use std::pin::Pin;
use std::sync::{mpsc, Arc, Mutex};
use std::task::{Context, Poll, Wake, Waker};
use std::thread;
use std::time::Duration;

struct Shared { set: bool, waker: Option<Waker> }
struct FlagFuture(Arc<Mutex<Shared>>);

impl Future for FlagFuture {
    type Output = ();
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        let mut s = self.0.lock().unwrap();
        if s.set {
            Poll::Ready(())
        } else {
            s.waker = Some(cx.waker().clone()); // latest waker, stored under the lock
            Poll::Pending
        }
    }
}

fn set_flag(shared: &Mutex<Shared>) {
    let mut s = shared.lock().unwrap();
    s.set = true;
    if let Some(w) = s.waker.take() { drop(s); w.wake(); }
}

struct T(thread::Thread);
impl Wake for T { fn wake(self: Arc<Self>) { self.0.unpark(); } }

fn block_on<F: Future>(f: F) -> F::Output {
    let mut f = std::pin::pin!(f);
    let w = Waker::from(Arc::new(T(thread::current())));
    let mut cx = Context::from_waker(&w);
    loop {
        match f.as_mut().poll(&mut cx) {
            Poll::Ready(v) => return v,
            Poll::Pending => thread::park(),
        }
    }
}

fn main() {
    let shared = Arc::new(Mutex::new(Shared { set: false, waker: None }));
    let s2 = shared.clone();
    let (tx, rx) = mpsc::channel();
    thread::spawn(move || {
        thread::sleep(Duration::from_millis(50));
        set_flag(&s2);
    });
    thread::spawn(move || {
        block_on(FlagFuture(shared));
        tx.send(()).unwrap();
    });
    println!("{:?}", rx.recv_timeout(Duration::from_secs(2)));
}
```

✓ This version printed `Ok(())`. Because the flag and the waker live under the same `Mutex`, "check, then register" and "set, then wake" cannot interleave badly (Lesson 29's rule: the state the condition depends on and the notification share one lock). It is the same structure as the `Condvar` protocol, with a `Waker` in place of the sleeping thread. The crate's `Sleep` and `JoinHandle` use this pattern.

</details>

---

## 13. Practical challenge: `asynclab`

Build a mini async runtime with **no `unsafe`** and **no dependencies**:

1. `block_on(fut)`: park/unpark waker.
2. `yield_now()` and `sleep(duration)` leaf futures. `sleep` must wake the **latest** waker, and its deadline starts at creation.
3. `join2(a, b)`: concurrent, never re-polls a finished future, `Unpin`.
4. `Executor`, `Spawner`, `JoinHandle<T>`: spawn from anywhere (including inside tasks), `run()` until all tasks finish, spawned futures `Send + 'static`, dropping a handle detaches.

Tests must use a watchdog so a lost wake-up fails rather than hangs, and must check the waker rules above.

<details markdown="1">
<summary>Solution (spoilers: full source)</summary>

`Cargo.toml`: a standard `cargo new --lib asynclab --edition 2024`, no dependencies.

`src/lib.rs`:

```rust
//! `asynclab`: a tiny `async` runtime in 100% safe, `std`-only Rust.
//!
//! * [`block_on`]: drive one future to completion on the current thread.
//! * [`Executor`] / [`Spawner`] / [`JoinHandle`]: run many tasks.
//! * [`sleep`], [`yield_now`]: leaf futures that register a [`Waker`].
//! * [`join2`]: poll two futures concurrently.
//!
//! It exists to show what an executor *is*; use Tokio (Lesson 31) for real work.

#![forbid(unsafe_code)]

use std::future::Future;
use std::pin::{Pin, pin};
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::mpsc::{Receiver, Sender, channel};
use std::sync::{Arc, Mutex, MutexGuard};
use std::task::{Context, Poll, Wake, Waker};
use std::thread::{self, Thread};
use std::time::{Duration, Instant};

fn lock<T>(m: &Mutex<T>) -> MutexGuard<'_, T> {
    // Every critical section here leaves the data valid, so poisoning is ignored.
    m.lock().unwrap_or_else(|e| e.into_inner())
}

// ---------------------------------------------------------------- block_on

struct ThreadWaker(Thread);

impl Wake for ThreadWaker {
    fn wake(self: Arc<Self>) {
        // `unpark` leaves a token if the thread is not parked yet, so a wake
        // that arrives between `poll` returning Pending and `park` is not lost.
        self.0.unpark();
    }
}

/// Runs `fut` to completion on the calling thread, parking it while the
/// future is `Pending`.
///
/// ```
/// let v = asynclab::block_on(async { 20 + 22 });
/// assert_eq!(v, 42);
/// ```
pub fn block_on<F: Future>(fut: F) -> F::Output {
    let mut fut = pin!(fut);
    let waker = Waker::from(Arc::new(ThreadWaker(thread::current())));
    let mut cx = Context::from_waker(&waker);
    loop {
        match fut.as_mut().poll(&mut cx) {
            Poll::Ready(v) => return v,
            // `park` may also return spuriously; polling again is always allowed.
            Poll::Pending => thread::park(),
        }
    }
}

// ------------------------------------------------------------- leaf futures

/// Completes on its second poll, after asking to be polled again.
pub fn yield_now() -> YieldNow {
    YieldNow { yielded: false }
}

#[must_use = "futures do nothing unless polled"]
pub struct YieldNow {
    yielded: bool,
}

impl Future for YieldNow {
    type Output = ();
    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        if self.yielded {
            Poll::Ready(())
        } else {
            self.yielded = true;
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}

struct SleepState {
    done: bool,
    waker: Option<Waker>,
}

/// Completes once `duration` has passed **since this call** (not since the
/// first poll). It uses one helper thread per sleep: fine for a lesson, wrong
/// for production, where a runtime keeps one timer structure for all sleeps.
pub fn sleep(duration: Duration) -> Sleep {
    Sleep {
        deadline: Instant::now() + duration,
        shared: Arc::new(Mutex::new(SleepState {
            done: false,
            waker: None,
        })),
        started: false,
    }
}

#[must_use = "futures do nothing unless polled"]
pub struct Sleep {
    deadline: Instant,
    shared: Arc<Mutex<SleepState>>,
    started: bool,
}

impl Future for Sleep {
    type Output = ();
    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        if Instant::now() >= self.deadline || lock(&self.shared).done {
            return Poll::Ready(());
        }
        // Store the *latest* waker on every poll: the task may have been
        // moved to a different waker since the previous poll.
        lock(&self.shared).waker = Some(cx.waker().clone());
        if !self.started {
            self.started = true;
            let shared = Arc::clone(&self.shared);
            let deadline = self.deadline;
            thread::spawn(move || {
                thread::sleep(deadline.saturating_duration_since(Instant::now()));
                let mut s = lock(&shared);
                s.done = true;
                if let Some(w) = s.waker.take() {
                    drop(s);
                    w.wake();
                }
            });
        }
        Poll::Pending
    }
}

// -------------------------------------------------------------------- join2

/// Polls `a` and `b` in turn until both are done, returning both outputs.
/// Both futures make progress concurrently (on this one thread).
///
/// ```
/// use asynclab::{block_on, join2};
/// let (x, y) = block_on(join2(async { 1 }, async { "two" }));
/// assert_eq!((x, y), (1, "two"));
/// ```
pub fn join2<A: Future, B: Future>(a: A, b: B) -> Join2<A, B> {
    Join2 {
        a: Box::pin(a),
        b: Box::pin(b),
        ra: None,
        rb: None,
    }
}

#[must_use = "futures do nothing unless polled"]
pub struct Join2<A: Future, B: Future> {
    a: Pin<Box<A>>,
    b: Pin<Box<B>>,
    ra: Option<A::Output>,
    rb: Option<B::Output>,
}

// The outputs are plain stored values (never pinned in place), and the
// futures are boxed, so moving a `Join2` moves nothing that was pinned.
impl<A: Future, B: Future> Unpin for Join2<A, B> {}

impl<A: Future, B: Future> Future for Join2<A, B> {
    type Output = (A::Output, B::Output);

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let this = self.get_mut();
        // Never poll a future again after it returned `Ready`.
        if this.ra.is_none()
            && let Poll::Ready(v) = this.a.as_mut().poll(cx)
        {
            this.ra = Some(v);
        }
        if this.rb.is_none()
            && let Poll::Ready(v) = this.b.as_mut().poll(cx)
        {
            this.rb = Some(v);
        }
        if this.ra.is_some() && this.rb.is_some() {
            match (this.ra.take(), this.rb.take()) {
                (Some(a), Some(b)) => Poll::Ready((a, b)),
                _ => unreachable!("both were just checked to be Some"),
            }
        } else {
            Poll::Pending
        }
    }
}

// ----------------------------------------------------------------- executor

type BoxFuture = Pin<Box<dyn Future<Output = ()> + Send>>;

struct Task {
    future: Mutex<Option<BoxFuture>>,
    queue: Sender<Arc<Task>>,
}

impl Wake for Task {
    fn wake(self: Arc<Self>) {
        self.wake_by_ref();
    }
    fn wake_by_ref(self: &Arc<Self>) {
        // A task woken twice is queued twice; the extra poll is harmless.
        // The receiver is gone only after the executor was dropped.
        let _ = self.queue.send(Arc::clone(self));
    }
}

struct JoinState<T> {
    result: Option<T>,
    waker: Option<Waker>,
}

/// Awaitable result of a spawned task. Dropping it **detaches** the task
/// (the task keeps running); it does not cancel it.
pub struct JoinHandle<T> {
    state: Arc<Mutex<JoinState<T>>>,
}

impl<T> Future for JoinHandle<T> {
    type Output = T;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<T> {
        let mut s = lock(&self.state);
        match s.result.take() {
            Some(v) => Poll::Ready(v),
            None => {
                s.waker = Some(cx.waker().clone());
                Poll::Pending
            }
        }
    }
}

/// Cloneable handle for submitting tasks, also usable from inside tasks.
#[derive(Clone)]
pub struct Spawner {
    queue: Sender<Arc<Task>>,
    live: Arc<AtomicUsize>,
}

impl Spawner {
    /// Queues `fut` to run. It must be `Send + 'static` because any thread
    /// may wake it and it outlives this call.
    ///
    /// A future holding an `Rc` across an `.await` is rejected:
    ///
    /// ```compile_fail
    /// let ex = asynclab::Executor::new();
    /// let rc = std::rc::Rc::new(1);
    /// ex.spawner().spawn(async move {
    ///     let _r = &rc;
    ///     asynclab::yield_now().await;
    /// });
    /// ```
    pub fn spawn<T, F>(&self, fut: F) -> JoinHandle<T>
    where
        T: Send + 'static,
        F: Future<Output = T> + Send + 'static,
    {
        let state = Arc::new(Mutex::new(JoinState {
            result: None,
            waker: None,
        }));
        let state2 = Arc::clone(&state);
        let wrapped = async move {
            let value = fut.await;
            let mut s = lock(&state2);
            s.result = Some(value);
            if let Some(w) = s.waker.take() {
                drop(s);
                w.wake(); // whoever awaits the JoinHandle
            }
        };
        self.live.fetch_add(1, Ordering::AcqRel);
        let task = Arc::new(Task {
            future: Mutex::new(Some(Box::pin(wrapped))),
            queue: self.queue.clone(),
        });
        let _ = self.queue.send(task);
        JoinHandle { state }
    }
}

/// Single-threaded executor: polls queued tasks on the thread that calls
/// [`Executor::run`].
pub struct Executor {
    rx: Receiver<Arc<Task>>,
    spawner: Spawner,
}

impl Executor {
    pub fn new() -> Self {
        let (tx, rx) = channel();
        Executor {
            rx,
            spawner: Spawner {
                queue: tx,
                live: Arc::new(AtomicUsize::new(0)),
            },
        }
    }

    pub fn spawner(&self) -> Spawner {
        self.spawner.clone()
    }

    /// Number of tasks that have not finished.
    pub fn live_tasks(&self) -> usize {
        self.spawner.live.load(Ordering::Acquire)
    }

    /// Runs until every task (including ones spawned meanwhile) has finished.
    /// If a task waits for something that will never happen, this blocks
    /// forever, like a deadlock. A panic inside a task propagates out of `run`.
    pub fn run(&self) {
        while self.live_tasks() > 0 {
            let Ok(task) = self.rx.recv() else { break };
            let mut slot = lock(&task.future);
            if let Some(mut fut) = slot.take() {
                let waker = Waker::from(Arc::clone(&task));
                let mut cx = Context::from_waker(&waker);
                match fut.as_mut().poll(&mut cx) {
                    Poll::Pending => *slot = Some(fut),
                    Poll::Ready(()) => {
                        self.spawner.live.fetch_sub(1, Ordering::AcqRel);
                    }
                }
            }
        }
    }
}

impl Default for Executor {
    fn default() -> Self {
        Self::new()
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::sync::mpsc;

    /// Fails instead of hanging if `f` does not finish in time.
    fn within<R: Send + 'static>(secs: u64, f: impl FnOnce() -> R + Send + 'static) -> R {
        let (tx, rx) = mpsc::channel();
        thread::spawn(move || {
            let _ = tx.send(f());
        });
        rx.recv_timeout(Duration::from_secs(secs))
            .expect("timed out: the future never completed")
    }

    #[test]
    fn block_on_ready_and_nested_awaits() {
        async fn add(a: u32, b: u32) -> u32 {
            a + b
        }
        assert_eq!(block_on(async { add(add(1, 2).await, 4).await }), 7);
    }

    #[test]
    fn futures_are_lazy() {
        let log = Arc::new(Mutex::new(Vec::new()));
        let l2 = Arc::clone(&log);
        let fut = async move {
            l2.lock().unwrap().push("ran");
        };
        assert!(log.lock().unwrap().is_empty(), "creating must not run it");
        let dropped = {
            let l3 = Arc::clone(&log);
            async move {
                l3.lock().unwrap().push("never");
            }
        };
        drop(dropped);
        block_on(fut);
        assert_eq!(*log.lock().unwrap(), vec!["ran"]);
    }

    #[test]
    fn yield_now_returns_pending_exactly_once() {
        let mut y = pin!(yield_now());
        let mut cx = Context::from_waker(Waker::noop());
        assert_eq!(y.as_mut().poll(&mut cx), Poll::Pending);
        assert_eq!(y.as_mut().poll(&mut cx), Poll::Ready(()));
    }

    #[test]
    fn sleep_waits_at_least_the_duration() {
        within(10, || {
            let start = Instant::now();
            block_on(sleep(Duration::from_millis(80)));
            assert!(start.elapsed() >= Duration::from_millis(80));
        });
    }

    #[test]
    fn sleep_deadline_starts_at_creation_not_first_poll() {
        within(10, || {
            let s = sleep(Duration::from_millis(60));
            thread::sleep(Duration::from_millis(100));
            let start = Instant::now();
            block_on(s); // already expired: must not wait another 60 ms
            assert!(start.elapsed() < Duration::from_millis(40));
        });
    }

    #[test]
    fn sleep_wakes_the_most_recent_waker() {
        struct Counting(AtomicUsize);
        impl Wake for Counting {
            fn wake(self: Arc<Self>) {
                self.0.fetch_add(1, Ordering::SeqCst);
            }
        }
        let (a, b) = (
            Arc::new(Counting(AtomicUsize::new(0))),
            Arc::new(Counting(AtomicUsize::new(0))),
        );
        let (wa, wb) = (Waker::from(Arc::clone(&a)), Waker::from(Arc::clone(&b)));
        let mut s = pin!(sleep(Duration::from_millis(60)));
        assert_eq!(
            s.as_mut().poll(&mut Context::from_waker(&wa)),
            Poll::Pending
        );
        // The task moved: the second poll comes with a different waker.
        assert_eq!(
            s.as_mut().poll(&mut Context::from_waker(&wb)),
            Poll::Pending
        );
        thread::sleep(Duration::from_millis(200));
        assert_eq!(
            a.0.load(Ordering::SeqCst),
            0,
            "stale waker must not be used"
        );
        assert_eq!(b.0.load(Ordering::SeqCst), 1, "latest waker must be woken");
        assert_eq!(
            s.as_mut().poll(&mut Context::from_waker(&wb)),
            Poll::Ready(())
        );
    }

    #[test]
    fn join2_interleaves_and_returns_both() {
        let log = Arc::new(Mutex::new(Vec::new()));
        let task = |name: &'static str, log: Arc<Mutex<Vec<&'static str>>>| async move {
            for _ in 0..2 {
                log.lock().unwrap().push(name);
                yield_now().await;
            }
            name.len()
        };
        let (a, b) = block_on(join2(task("a", log.clone()), task("bb", log.clone())));
        assert_eq!((a, b), (1, 2));
        assert_eq!(*log.lock().unwrap(), vec!["a", "bb", "a", "bb"]);
    }

    #[test]
    fn join2_does_not_repoll_a_finished_future() {
        struct OnlyOnce(bool);
        impl Future for OnlyOnce {
            type Output = ();
            fn poll(mut self: Pin<&mut Self>, _: &mut Context<'_>) -> Poll<()> {
                assert!(!self.0, "polled after completion");
                self.0 = true;
                Poll::Ready(())
            }
        }
        block_on(join2(OnlyOnce(false), async {
            yield_now().await;
            yield_now().await;
        }));
    }

    #[test]
    fn executor_runs_all_tasks_and_join_handles_return_values() {
        within(10, || {
            let ex = Executor::new();
            let sp = ex.spawner();
            let h1 = sp.spawn(async { 1 + 1 });
            let h2 = sp.spawn(async {
                yield_now().await;
                "done"
            });
            let sp2 = sp.clone();
            let out = Arc::new(Mutex::new(None));
            let out2 = Arc::clone(&out);
            sp.spawn(async move {
                let a = h1.await;
                let b = h2.await;
                let c = sp2.spawn(async { 40 }).await; // spawning from inside a task
                *out2.lock().unwrap() = Some((a, b, c));
            });
            ex.run();
            assert_eq!(*out.lock().unwrap(), Some((2, "done", 40)));
            assert_eq!(ex.live_tasks(), 0);
        });
    }

    #[test]
    fn many_sleeping_tasks_finish_concurrently() {
        within(10, || {
            let ex = Executor::new();
            let sp = ex.spawner();
            let done = Arc::new(AtomicUsize::new(0));
            for _ in 0..50 {
                let done = Arc::clone(&done);
                sp.spawn(async move {
                    sleep(Duration::from_millis(30)).await;
                    done.fetch_add(1, Ordering::SeqCst);
                });
            }
            ex.run();
            assert_eq!(done.load(Ordering::SeqCst), 50);
        });
    }

    #[test]
    fn dropping_a_join_handle_detaches_the_task() {
        within(10, || {
            let ex = Executor::new();
            let sp = ex.spawner();
            let flag = Arc::new(AtomicUsize::new(0));
            let f2 = Arc::clone(&flag);
            drop(sp.spawn(async move {
                yield_now().await;
                f2.store(1, Ordering::SeqCst);
            }));
            ex.run();
            assert_eq!(flag.load(Ordering::SeqCst), 1);
        });
    }

    #[test]
    fn wake_from_another_thread() {
        within(10, || {
            struct Flag {
                set: bool,
                waker: Option<Waker>,
            }
            struct Wait(Arc<Mutex<Flag>>);
            impl Future for Wait {
                type Output = ();
                fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
                    let mut f = self.0.lock().unwrap();
                    if f.set {
                        Poll::Ready(())
                    } else {
                        f.waker = Some(cx.waker().clone());
                        Poll::Pending
                    }
                }
            }
            let shared = Arc::new(Mutex::new(Flag {
                set: false,
                waker: None,
            }));
            let s2 = Arc::clone(&shared);
            thread::spawn(move || {
                thread::sleep(Duration::from_millis(50));
                let mut f = s2.lock().unwrap();
                f.set = true;
                if let Some(w) = f.waker.take() {
                    w.wake();
                }
            });
            block_on(Wait(shared));
        });
    }

    #[test]
    fn spawn_requires_send() {
        fn needs_send<T: Send>(_: &T) {}
        let fut = async {
            sleep(Duration::from_millis(1)).await;
        };
        needs_send(&fut);
    }
}
```

**Verification ✓.** `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings` and `cargo test --locked` pass: 13 unit tests, 3 doctests (one is `compile_fail`: spawning a future that holds an `Rc` across an `.await` is rejected with the `Send` error from §5). The exercise 8 `timeout` combinator, the exercise 4 `Ready1` future and the exercise 6 fix were compiled and run against the crate in a separate scratch project (✓ `Ok(5) Err(Elapsed)`, and `Ready1::new(3)` returned `3`).

**Mutation checks ✓** (each mutation applied to a copy; unit tests run with a 120-second limit):

| # | Mutation | Result |
|---|---|---|
| M1 | `Task::wake_by_ref` does nothing | **caught**: executor tests time out (watchdog) |
| M2 | `Sleep` stores the waker only on the first poll | **not caught by my first 12 tests**; caught after I added `sleep_wakes_the_most_recent_waker` ("stale waker must not be used") |
| M3 | `join2` never polls `b` until `a` is done | **caught**: interleaving test fails |
| M4 | spawned task does not wake the `JoinHandle` on completion | **caught**: timeout |
| M5 | `live` counter not decremented | **caught**: timeouts (`run` never returns) |
| M6 | `yield_now` returns `Pending` without waking | **caught** (the executor tests fail on the watchdog and the run also hung until the 120 s limit killed it: exit 124) |
| M7 | `join2` re-polls a finished `a` | **caught**: "polled after completion" |

I also discarded one mutation I wrote for "deadline starts at first poll" because the text I typed was a no-op (`+ Duration * 0`); that behavior is covered by the test `sleep_deadline_starts_at_creation_not_first_poll`, but I did **not** run a real mutant against it.

**Design notes.**

- `Wake for ThreadWaker` / `Wake for Task` are the safe route to a `Waker`. `#![forbid(unsafe_code)]` is on.
- A `Task` owns `Mutex<Option<BoxFuture>>`: `take()` while polling, put back on `Pending`, leave `None` after `Ready`. A wake that arrives after completion finds `None` and does nothing.
- Wakes go through an `mpsc` channel (unbounded). `run()` is a single consumer; `Sender` is cloned into every task, which is why the channel never disconnects while the executor lives.
- `live` counts unfinished tasks so `run()` knows when to stop; a task waiting on something that never happens makes `run()` block forever (documented, like a deadlock).
- `JoinHandle` completion is the same register-waker/set-result protocol as §12, under one mutex.
- `Join2` boxes its two futures (`Pin<Box<_>>`) so it can be `Unpin` and use no pin-projection `unsafe`; the cost is two allocations.
- Not handled: panics in tasks (they propagate out of `run`), cancellation of a spawned task, multi-threaded polling, a shared timer thread, `select`, I/O readiness. Tokio (Lesson 31) provides all of these.

</details>

---

## 14. Interview questions

1. What is a `Future` in Rust? What does `poll` return and what must it do before returning `Pending`?
2. Why are Rust futures lazy, and what are the consequences?
3. What is a `Waker`, and what happens if a future returns `Pending` without registering one?
4. Why `Pin`? Why is an `async` block `!Unpin`?
5. Why must futures given to a multi-threaded `spawn` be `Send + 'static`? Why is holding a `std::sync::MutexGuard` across `.await` a problem?
6. What does dropping a future mean? How is that different from dropping a `JoinHandle`?
7. What is the difference between concurrency and parallelism in `join!` vs `spawn`?
8. Why is blocking inside `async` code harmful, and how do you handle unavoidable blocking work?

<details markdown="1">
<summary>Answers</summary>

1. A value representing an asynchronous computation, implementing `poll(Pin<&mut Self>, &mut Context) -> Poll<Output>`. `Ready(v)` when done, `Pending` otherwise; before `Pending` it must arrange for the waker in `cx` to be called when it can progress (§2, ✓ §3.3).
2. A future does nothing until polled (✓ §3.1). Consequences: creating futures is cheap and side-effect free; forgetting `.await` silently does nothing (hence the `must_use` warning ✓); dropping a never-polled future costs nothing; control of *when* work happens is with the executor.
3. A handle the executor creates per task; `wake()` re-queues the task. Without it the task is never polled again (✓ the hang), the async analogue of a lost notification.
4. A compiled async body can contain references into itself across suspension points; moving it would invalidate them. `Pin` forbids moving after the first poll. The compiler checks that such futures are `!Unpin` (✓ `E0277` ... "cannot be unpinned").
5. The runtime may poll the task on any worker thread, so the future moves between threads (`Send`), and it can outlive the spawning scope (`'static`). A `MutexGuard` is `!Send` (✓ error) and, even on a single thread, holding it across a suspension can deadlock another task that wants the lock.
6. Dropping a future cancels it: its locals are dropped and no code after the last `.await` runs. Dropping a `JoinHandle` (in Tokio and in the lesson's executor) detaches the task, which keeps running; it does not cancel it.
7. `join!` polls several futures **within one task**, so they run concurrently on one thread at a time (interleaved). `spawn` creates separate tasks, which a multi-threaded runtime can run **in parallel** on different threads.
8. A blocking call occupies the executor thread; every task scheduled there stalls (✓ §8). Use async versions of I/O and timers, and send CPU-heavy or inherently blocking work to a dedicated thread or blocking pool, receiving the result through a channel or `JoinHandle`.

</details>

---

## 15. Key takeaways

- `async fn`/`async {}` produce lazy state machines implementing `Future`; calling them runs nothing (✓), and `.await` is a poll loop inside a state machine whose size is set by what lives across `.await` (✓ 1002 vs 3 bytes).
- `poll` must return `Pending` only after registering the waker, and a stored waker must be the **latest** one (✓ hang, ✓ mutant). Wake-ups are hints; extra polls are normal.
- An executor is a queue of `Arc<Task>`s plus wakers that re-queue them; `block_on` is the smallest case (park/unpark). No `unsafe` is needed (`Wake` trait).
- Spawned futures must be `Send + 'static`; holding `Rc` or `std` `MutexGuard` across `.await` is rejected (✓). Recursion needs `Box::pin` (✓ `E0733`).
- Dropping a future cancels it; dropping a `JoinHandle` detaches. Never block inside `async`; move blocking work off the executor.
- `async fn` in traits and async closures are stable (✓), with `Send`-bound ergonomics still rough (from memory).

**Connections.** Lesson 25's `Pin`/`Unpin` explain `poll`'s receiver; Lesson 22's `Send`/`Sync` explain every `spawn` error here; Lessons 27–29 supplied the thread, atomic and lock/condvar machinery that `block_on`, `Sleep` and `JoinHandle` are built from. Next: Lesson 31 puts a production runtime (Tokio) to work: tasks, async I/O, timers, channels, async mutexes, `spawn_blocking`, then Lesson 32 covers cancellation, streams and backpressure.

---

## 16. References

- `std::future` (`Future`, `ready`, `poll_fn`, `IntoFuture`): <https://doc.rust-lang.org/std/future/index.html>
- `std::task` (`Poll`, `Context`, `Waker`, `Wake`, `RawWaker`): <https://doc.rust-lang.org/std/task/index.html>
- *Asynchronous Programming in Rust* (the Async Book): <https://rust-lang.github.io/async-book/>
- *The Rust Reference*, "Await expressions" and "Async functions": <https://doc.rust-lang.org/reference/expressions/await-expr.html>
- Rust error index: E0728, E0733, E0277, E0599, E0369: <https://doc.rust-lang.org/error_codes/E0733.html>
- Jon Gjengset, "Crust of Rust: async/await" (video, from memory), and the Tokio tutorial: <https://tokio.rs/tokio/tutorial>

The URLs above are the standard documentation locations; I did not open them during this session (the sandbox cannot reach them), so if a link has moved, search the docs site for the page title. Unchecked or from memory: the exact text of the `Future`/`Poll` definitions; that `Waker::noop` is stable since 1.85; that Tokio, async-std and smol are the main runtimes and Tokio catches task panics into the `JoinHandle`, has a current-thread runtime, a work-stealing multi-thread scheduler and `spawn_blocking`; that `RawWaker` is the `unsafe` alternative; the limits of `async fn` in traits (Send bounds, dyn compatibility) and the `trait_variant` workaround; `futures`/`tokio` `join!`/`select!`/`try_join!`; pin-projection crates; runtime fairness budgets; and the "Crust of Rust" recommendation. Everything marked ✓ was run on this machine's `rustc 1.97.0`.
