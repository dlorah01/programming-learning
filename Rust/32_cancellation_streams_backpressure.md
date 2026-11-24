---
title: "Lesson 32 — Cancellation, streams, and backpressure"
---

# Lesson 32 — Cancellation, streams, and backpressure

**Module 10 · Concurrency and async** · Status: generated · Edition used in examples: **2024** · External dependencies: `tokio` 1.54.1, `tokio-stream` 0.1.19, `futures` 0.3.34 (✓ all downloaded from crates.io and run; versions are what `Cargo.lock` resolved in this sandbox today)

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux with 2 cores. As before, ✓ marks outputs, diagnostics and measurements I actually ran; unmarked statements are documented behavior I did not run. Everything is **stable Rust**. Most examples run under Tokio's **paused clock**, so "30 ms later" is exact and the output is deterministic. I could not open the Tokio or `futures` documentation from the sandbox, so claims about which library functions are cancel-safe come from memory and are listed in the closing paragraph; the *consequences* shown below were run.

Builds on [Lesson 31](31_tokio.md) (Tokio tasks, `select!`, channels, graceful shutdown), [Lesson 30](30_futures_async.md) (futures are state machines; dropping cancels), [Lesson 25](25_weak_pin.md) (`Pin`/`Unpin`) and [Lesson 29](29_locks_deadlocks.md) (bounded resources and circular waits). This is the last lesson of Module 10; Module 11 (modules, API design, macros) starts with Lesson 33.

---

## 1. Cancellation is dropping

In async Rust you cancel by **dropping the future**. There is no other mechanism: no exception is thrown, no `finally` block runs, no flag is set. When a future that is suspended at an `.await` is dropped, the compiler-generated state machine drops the locals that were alive at that suspension point, in the usual reverse order (`Drop` runs), and **nothing after that `.await` ever executes**.

You cause cancellation, often without noticing, with:

- `tokio::select!`: all branches that did not win are dropped (Lesson 31 §2).
- `tokio::time::timeout`: when the timer fires, the inner future is dropped.
- `JoinHandle::abort()` and dropping a `JoinSet` / `AbortHandle`: the task is dropped at its next suspension point.
- Dropping the future yourself, or dropping the future that contains it (a server framework dropping a request handler when the client disconnects behaves this way in some frameworks; from memory, check yours).
- Dropping a runtime: all its tasks are dropped.

The demonstration, all ✓ (paused clock, current-thread runtime):

```rust
use std::sync::atomic::{AtomicU32, Ordering};
use std::sync::Arc;
use std::time::Duration;
use tokio::time::sleep;

struct Guard { name: &'static str, log: Arc<AtomicU32>, bit: u32 }
impl Drop for Guard { fn drop(&mut self) { self.log.fetch_or(self.bit, Ordering::SeqCst); println!("  drop {}", self.name); } }

#[tokio::main(flavor = "current_thread", start_paused = true)]
async fn main() {
    let log = Arc::new(AtomicU32::new(0));
    let l = log.clone();
    let h = tokio::spawn(async move {
        let _g = Guard { name: "in-task guard", log: l.clone(), bit: 1 };
        sleep(Duration::from_secs(60)).await;
        l.fetch_or(2, Ordering::SeqCst); // code after the await
        println!("after await (should not print)");
    });
    sleep(Duration::from_millis(10)).await;
    h.abort();
    let _ = h.await;
    println!("after abort: guard dropped={} later code ran={}", log.load(Ordering::SeqCst) & 1 == 1, log.load(Ordering::SeqCst) & 2 == 2);

    // cancelled by timeout (drop of the future)
    let l = log.clone();
    let r = tokio::time::timeout(Duration::from_millis(10), async move {
        let _g = Guard { name: "timeout guard", log: l, bit: 4 };
        sleep(Duration::from_secs(60)).await;
    }).await;
    println!("timeout elapsed={} guard dropped={}", r.is_err(), log.load(Ordering::SeqCst) & 4 == 4);

    // Drop cannot .await; it can spawn cleanup if a runtime is present
    struct AsyncCleanup(Arc<AtomicU32>);
    impl Drop for AsyncCleanup {
        fn drop(&mut self) {
            let l = self.0.clone();
            tokio::spawn(async move { sleep(Duration::from_millis(5)).await; l.fetch_or(8, Ordering::SeqCst); });
        }
    }
    drop(AsyncCleanup(log.clone()));
    sleep(Duration::from_millis(20)).await;
    println!("spawned cleanup ran: {}", log.load(Ordering::SeqCst) & 8 == 8);
}
```

Output:

```text
  drop in-task guard
after abort: guard dropped=true later code ran=false
  drop timeout guard
timeout elapsed=true guard dropped=true
spawned cleanup ran: true
```

What this shows:

1. **`Drop` runs on cancellation.** Both the aborted task and the timed-out future had their guard dropped.
2. **Code after the `.await` does not run.** The `fetch_or(2)` after the 60-second sleep never executed, so any "finish up" step you placed there is skipped. If that step must happen, it cannot live only after an `.await`.
3. **`Drop` cannot `.await`.** Cleanup that needs async work must either be started from `Drop` (✓ `tokio::spawn` inside `drop`, which works as long as a runtime is still running; if the runtime itself is shutting down, the spawned cleanup may never run, from memory) or be done explicitly with a method such as `shutdown().await` on the success path. Async `Drop` is not part of stable Rust (from memory).

---

## 2. Cancel safety

An operation is **cancel-safe** if dropping its future at any `.await` loses no data and leaves no half-done state that breaks later use. Whether a function is cancel-safe is part of its documented contract, and library authors usually say so (Tokio does, in each method's docs; from memory).

### 2.1 A cancel-unsafe operation: `read_exact`

```rust
use std::time::Duration;
use tokio::io::{duplex, AsyncReadExt, AsyncWriteExt};
use tokio::sync::mpsc;
use tokio::time::sleep;

#[tokio::main(flavor = "current_thread", start_paused = true)]
async fn main() {
    // 1. read_exact is NOT cancel-safe: bytes already read are lost when it is dropped
    let (mut tx, mut rx) = duplex(64);
    tx.write_all(b"abc").await.unwrap(); // only 3 of the 5 bytes we want
    let mut buf = [0u8; 5];
    tokio::select! {
        r = rx.read_exact(&mut buf) => println!("completed {:?}", r.is_ok()),
        _ = sleep(Duration::from_millis(10)) => println!("timer won; read_exact dropped"),
    }
    tx.write_all(b"de").await.unwrap();
    let mut buf2 = [0u8; 2];
    rx.read_exact(&mut buf2).await.unwrap();
    println!("next 2 bytes: {:?}  (the 'a','b','c' consumed earlier are gone)", std::str::from_utf8(&buf2).unwrap());

    // 2. FIX: keep one future alive across loop iterations by pinning it
    let (mut tx, mut rx) = duplex(64);
    tx.write_all(b"abc").await.unwrap();
    let mut buf = [0u8; 5];
    {
        let fut = rx.read_exact(&mut buf);
        tokio::pin!(fut);
        let mut timeouts = 0;
        loop {
            tokio::select! {
                r = &mut fut => { r.unwrap(); break; }
                _ = sleep(Duration::from_millis(10)) => {
                    timeouts += 1;
                    if timeouts == 1 { tx.write_all(b"de").await.unwrap(); }
                }
            }
        }
        println!("pinned read_exact timeouts={timeouts}");
    }
    println!("pinned result: {:?}", std::str::from_utf8(&buf).unwrap());

    // 3. mpsc::Receiver::recv IS cancel-safe: no message lost when dropped by select!
    let (mtx, mut mrx) = mpsc::channel::<u32>(4);
    tokio::select! {
        v = mrx.recv() => println!("got {:?}", v),
        _ = sleep(Duration::from_millis(10)) => println!("recv dropped by timer"),
    }
    mtx.send(7).await.unwrap();
    println!("recv after cancel -> {:?}", mrx.recv().await);
}
```

✓ Output:

```text
timer won; read_exact dropped
next 2 bytes: "de"  (the 'a','b','c' consumed earlier are gone)
pinned read_exact timeouts=1
pinned result: "abcde"
recv dropped by timer
recv after cancel -> Some(7)
```

Part 1: three of the five wanted bytes (`abc`) were in the pipe. `read_exact` read them into its buffer, then the timer won and `select!` **dropped** the `read_exact` future. The three bytes were already consumed from the stream and are gone: the next `read_exact` saw `de`. This silently corrupts any protocol with framing. `read_exact` is documented as not cancel-safe (from memory), and the experiment shows why.

Part 2 (the fix): create the future **once**, pin it (`tokio::pin!`), and in the loop select on `&mut fut`. A `&mut` to a pinned future is itself a future; when the *timer* branch wins, only the timer branch is dropped, and the `read_exact` keeps its progress. It completed with the full `abcde`.

Part 3: `mpsc::Receiver::recv` **is** cancel-safe (from memory of the docs): after being dropped by the timer, the later message `7` was still delivered ✓.

### 2.2 Strategies, in order of preference

1. **Use cancel-safe operations inside `select!`** (`recv`, `accept`, `read` into a buffer you own, `Notify::notified`, `Semaphore::acquire`, stream `next()`; from memory, check each in its docs).
2. **Keep state in a struct, not in locals.** A struct survives the drop of a future that borrowed it. §6's `Batcher` and `Chunks` do this: `next_batch().await` can be cancelled at any point and the buffered items are still there. Exercise 5 builds a cancel-safe frame reader this way.
3. **Reuse one pinned future across loop iterations** (`&mut fut`, as in part 2), so it is never dropped just because another branch won.
4. **Move the multi-step operation into its own task** and await the `JoinHandle`: dropping the *handle* detaches the task (Lesson 31), so the operation runs to completion even if the caller is cancelled.
5. **Make the effect atomic or compensating**: if an operation needs several awaited steps (debit, then credit), design it so that stopping half-way is either impossible (one transaction) or repairable (idempotent steps, a journal, a drop guard that undoes the first step).
6. **Guards for cleanup**: a `Drop` type that undoes or reports the half-done state (§6.3's `CancelGuard`, ✓ tested under abort and timeout).

---

## 3. `select!` in practice

`tokio::select!` waits on several futures, runs the handler of the first to complete, and drops the others. Details that cause bugs:

- **Fairness.** By default branches are polled in a **random order** each time, so a branch that is always ready cannot starve the others. `biased;` makes the order top-to-bottom. ✓ With two always-ready branches, 1000 iterations hit both branches in the default mode, and `biased` chose the first branch 1000 times and the second 0 times:

```rust
use std::future::ready;
#[tokio::main]
async fn main() {
    let (mut a, mut b) = (0, 0);
    for _ in 0..1000 {
        tokio::select! { _ = ready(()) => a += 1, _ = ready(()) => b += 1 }
    }
    println!("fair select: both branches chosen: {}", a > 100 && b > 100);
    let (mut a, mut b) = (0, 0);
    for _ in 0..1000 {
        tokio::select! { biased; _ = ready(()) => a += 1, _ = ready(()) => b += 1 }
    }
    println!("biased select: a={a} b={b}");
}
```

  Output: `fair select: both branches chosen: true` and `biased select: a=1000 b=0`. Use `biased` deliberately (for example, check a shutdown signal first), knowing it can starve later branches.
- **Preconditions.** A branch can be disabled with `, if condition` (✓ used in §3.1 below), and patterns can reject a result (`Some(x) = rx.recv() => ...`); when all branches are disabled, the optional `else =>` branch runs (from memory; I did not run `else`).
- **Handlers run after the other branches are dropped**, and may `.await` freely.
- **Borrowing.** The futures in `select!` borrow what they use; two branches cannot both borrow the same value mutably (the usual borrow checker rules apply).

### 3.1 The timer-reset bug ✓

```rust
use std::time::Duration;
use tokio::sync::mpsc;
use tokio::time::{sleep, interval, Instant};

// BUG: a fresh sleep() each loop iteration restarts the timer whenever another branch wins
#[tokio::main(flavor = "current_thread", start_paused = true)]
async fn main() {
    let (tx, mut rx) = mpsc::channel::<u32>(16);
    tokio::spawn(async move {
        for i in 0..10 { tx.send(i).await.unwrap(); sleep(Duration::from_millis(30)).await; }
    });
    let mut heartbeats = 0;
    let mut got = 0;
    loop {
        tokio::select! {
            m = rx.recv() => match m { Some(_) => got += 1, None => break },
            _ = sleep(Duration::from_millis(50)) => heartbeats += 1,   // never wins: messages every 30 ms
        }
    }
    println!("recreated sleep: messages={got} heartbeats={heartbeats}");

    // FIX 1: interval ticks on a fixed schedule
    let (tx, mut rx) = mpsc::channel::<u32>(16);
    tokio::spawn(async move {
        for i in 0..10 { tx.send(i).await.unwrap(); sleep(Duration::from_millis(30)).await; }
    });
    let mut tick = interval(Duration::from_millis(50));
    tick.tick().await; // first tick is immediate
    let (mut heartbeats, mut got) = (0, 0);
    loop {
        tokio::select! {
            m = rx.recv() => match m { Some(_) => got += 1, None => break },
            _ = tick.tick() => heartbeats += 1,
        }
    }
    println!("interval: messages={got} heartbeats>=4: {}", heartbeats >= 4);

    // FIX 2: one pinned Sleep, reset explicitly when you WANT a fresh deadline
    let start = Instant::now();
    let (tx, mut rx) = mpsc::channel::<u32>(16);
    tokio::spawn(async move {
        for i in 0..3 { tx.send(i).await.unwrap(); sleep(Duration::from_millis(30)).await; }
    });
    let deadline = sleep(Duration::from_millis(50));
    tokio::pin!(deadline);
    let mut idle_fired_at = None;
    loop {
        tokio::select! {
            m = rx.recv() => match m { Some(_) => {}, None => break },
            _ = &mut deadline, if idle_fired_at.is_none() => { idle_fired_at = Some(start.elapsed().as_millis()); }
        }
    }
    println!("pinned deadline fired at {:?} ms (fixed 50 ms from start)", idle_fired_at);
}
```

✓ Output:

```text
recreated sleep: messages=10 heartbeats=0
interval: messages=10 heartbeats>=4: true
pinned deadline fired at Some(50) ms (fixed 50 ms from start)
```

In the first loop, `sleep(50 ms)` is created **fresh on every iteration**. Messages arrive every 30 ms, so every iteration the message branch wins, the new sleep is dropped, and the next iteration starts a new 50 ms sleep: the heartbeat **never fires** (0 heartbeats in 10 messages). The code looks right and is wrong in a way tests rarely catch.

Fixes: use an `interval` (a ticker on a fixed schedule, ✓ at least 4 heartbeats), or create the `Sleep` once, pin it, and select on `&mut deadline` (✓ fired at exactly 50 ms). Reset it explicitly with `deadline.as_mut().reset(Instant::now() + d)` when you *want* a fresh deadline (✓ run in the debugging challenge). The crate's `Batcher` uses an absolute `deadline` Instant stored in the struct and `sleep_until(deadline)` each time, which has the same effect: the timer is "recreated" but always for the same instant.

---

## 4. Streams

A **stream** is the async analogue of an iterator: zero or more values over time.

```rust,ignore
pub trait Stream {
    type Item;
    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>>;
}
```

(The shape of the `futures_core::Stream` trait, written from memory.) `Poll::Ready(Some(x))` yields a value, `Ready(None)` ends the stream, `Pending` means "wake me later" with the same waker contract as futures (Lesson 30). `tokio_stream::Stream` is a re-export of that trait (✓ my crate imports it from `tokio_stream`). The standard library has **no stable `Stream`/`AsyncIterator`** in 1.97 (from memory), which is why every runtime and library depends on the `futures-core` trait.

Streams are **lazy** and consumed with the `StreamExt` extension traits (`tokio_stream::StreamExt` or `futures::StreamExt`; import the one whose combinators you need; they overlap and importing both causes ambiguity errors, from memory):

```rust
use std::time::Duration;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;
use futures::stream::{self, StreamExt};
use tokio::time::sleep;

#[tokio::main(flavor = "current_thread", start_paused = true)]
async fn main() {
    // laziness
    let s = stream::iter(1..=5).map(|x| { println!("  map {x}"); x * 2 });
    println!("stream created (nothing ran)");
    let v: Vec<_> = s.take(2).collect().await;
    println!("{:?}", v);

    // buffer_unordered(n): at most n futures in flight; completion order
    let now = Arc::new(AtomicUsize::new(0));
    let peak = Arc::new(AtomicUsize::new(0));
    let out: Vec<u32> = stream::iter([(1u32, 90u64), (2, 30), (3, 60), (4, 10)])
        .map(|(id, ms)| { let (now, peak) = (now.clone(), peak.clone()); async move {
            let n = now.fetch_add(1, Ordering::SeqCst) + 1; peak.fetch_max(n, Ordering::SeqCst);
            sleep(Duration::from_millis(ms)).await;
            now.fetch_sub(1, Ordering::SeqCst);
            id
        } })
        .buffer_unordered(2)
        .collect().await;
    println!("buffer_unordered(2) order={:?} peak={}", out, peak.load(Ordering::SeqCst));
    // buffered(n) keeps input order
    let out: Vec<u32> = stream::iter([(1u32, 90u64), (2, 30), (3, 60), (4, 10)])
        .map(|(id, ms)| async move { sleep(Duration::from_millis(ms)).await; id })
        .buffered(2).collect().await;
    println!("buffered(2) order={:?}", out);

    // unfold
    let s = stream::unfold(1u32, |n| async move { if n > 4 { None } else { Some((n * n, n + 1)) } });
    println!("unfold {:?}", s.collect::<Vec<_>>().await);
}
```

✓ Output:

```text
stream created (nothing ran)
  map 1
  map 2
[2, 4]
buffer_unordered(2) order=[2, 1, 3, 4] peak=2
buffered(2) order=[1, 2, 3, 4]
unfold [1, 4, 9, 16]
```

- Laziness: building the stream ran nothing, and `take(2)` pulled only two items through `map` (3, 4, 5 were never touched).
- **`buffer_unordered(n)`** turns a stream of futures into a stream of their outputs, running **at most `n` at once** (✓ measured peak = 2) and yielding in **completion** order. **`buffered(n)`** also runs `n` at once but yields in **input** order, so one slow item delays the items behind it. (The unordered order shown depends on timer ties; I print what I saw.) This is the standard "bounded concurrency" tool: much simpler than spawning with a semaphore when you do not need parallelism across threads.
- `stream::unfold(state, |s| async { Some((item, next_state)) })` builds a stream from async state.

### 4.1 Channels as streams, and per-item timeouts

```rust
use std::time::Duration;
use tokio::sync::mpsc;
use tokio::time::{sleep, Instant};
use tokio_stream::wrappers::ReceiverStream;
use tokio_stream::StreamExt;

#[tokio::main(flavor = "current_thread", start_paused = true)]
async fn main() {
    // bounded: the producer is paced by the consumer
    let (tx, mut rx) = mpsc::channel::<u32>(2);
    let start = Instant::now();
    let prod = tokio::spawn(async move {
        for i in 0..6 { tx.send(i).await.unwrap(); }
        start.elapsed()
    });
    let cons = tokio::spawn(async move {
        let mut n = 0;
        while let Some(_) = rx.recv().await { sleep(Duration::from_millis(100)).await; n += 1; }
        n
    });
    let took = prod.await.unwrap();
    println!("bounded: producer finished after >= 300ms virtual: {}", took >= Duration::from_millis(300));
    println!("consumed {}", cons.await.unwrap());

    // unbounded: the producer never waits; the backlog is the difference
    let (utx, mut urx) = mpsc::unbounded_channel::<u32>();
    let t0 = Instant::now();
    for i in 0..100_000 { utx.send(i).unwrap(); }
    println!("unbounded: 100000 sends took 0 virtual ms: {}", t0.elapsed() == Duration::ZERO);
    let mut received = 0;
    for _ in 0..10 { urx.recv().await.unwrap(); received += 1; }
    println!("backlog after consuming 10: {}", 100_000 - received);

    // channel as a stream
    let (tx, rx) = mpsc::channel::<u32>(4);
    tokio::spawn(async move { for i in 0..5 { tx.send(i).await.unwrap(); } });
    let v: Vec<u32> = ReceiverStream::new(rx).map(|x| x * 3).filter(|x| x % 2 == 0).collect().await;
    println!("{:?}", v);
    // per-item timeout
    let (tx, rx) = mpsc::channel::<u32>(4);
    tokio::spawn(async move { tx.send(1).await.unwrap(); sleep(Duration::from_secs(5)).await; let _ = tx.send(2).await; });
    let mut s = Box::pin(ReceiverStream::new(rx).timeout(Duration::from_millis(100)));
    println!("{:?}", s.next().await.map(|r| r.is_ok()));
    println!("{:?}", s.next().await.map(|r| r.is_ok()));
}
```

✓ Output:

```text
bounded: producer finished after >= 300ms virtual: true
consumed 6
unbounded: 100000 sends took 0 virtual ms: true
backlog after consuming 10: 99990
[0, 6, 12]
Some(true)
Some(false)
```

(The first half is the backpressure demonstration of §5.) `ReceiverStream::new(rx)` adapts an `mpsc::Receiver` to a `Stream`, and `.timeout(d)` (from `tokio_stream::StreamExt`) wraps every item in `Result<Item, Elapsed>`: the second `next()` printed `Some(false)`, meaning an `Err(Elapsed)` was produced because no item arrived within 100 ms.

### 4.2 Writing an adaptor, and a `Pin` surprise ✓

The practical crate's `chunks(stream, n)` groups items into `Vec`s. Its `poll_next` loops: poll the inner stream; push `Ready(Some)` items into a buffer and return a `Vec` when it is full; on `Ready(None)` flush the remainder and remember the stream ended; on `Pending` return `Pending` **while keeping the partial buffer in the adaptor**. Because the state is in the struct, `next().await` on it is cancel-safe (✓ test: a half-full chunk survives a `timeout` that drops `next()`).

The first version did not compile ✓:

```text
error[E0277]: `<S as Stream>::Item` cannot be unpinned
  ... required because it appears within the type `std::marker::PhantomData<<S as tokio_stream::Stream>::Item>`
  ... required because it appears within the type `alloc::raw_vec::RawVec<..>`
  ... required because it appears within the type `std::vec::Vec<<S as tokio_stream::Stream>::Item>`
  ... required because it appears within the type `Chunks<S>`
  note: required by a bound in `Pin::<&mut T>::get_mut`
```

`Pin::get_mut` needs `Chunks<S>: Unpin`. I had assumed `Vec<T>` is always `Unpin`, but it is **only `Unpin` when `T` is** (the auto trait looks through the `PhantomData<T>` inside `RawVec`). The fix is an explicit, safe `impl<S: Stream + Unpin> Unpin for Chunks<S> {}`, justified because the items are stored by value and never pinned in place. This is the same reasoning as `Join2` in Lesson 30.

---

## 5. Backpressure

**Backpressure** is the mechanism by which a slow consumer slows down its producer instead of letting work pile up. Without it, a fast producer plus a slow consumer means an ever-growing queue: memory exhaustion, latency that grows without bound, and then a crash.

The experiment (§4.1 output) ✓:

| Setup | Observation ✓ |
|---|---|
| bounded `mpsc::channel(2)`, consumer takes 100 ms per item, producer sends 6 | the producer's `send().await` waited: it finished only after ≥ 300 ms of virtual time, i.e. paced by the consumer |
| `mpsc::unbounded_channel`, 100 000 sends | all 100 000 `send`s completed in 0 virtual ms; after the consumer took 10, **99 990 were still queued** |

Where backpressure comes from and where it is silently lost:

- **Pull-based streams** carry it naturally: nothing is produced until the consumer polls.
- **Bounded channels** convert "full" into "the sender waits" (`send().await`), into an error (`try_send` → `Full`), or into dropping (your choice of policy: drop newest, drop oldest, fail the request; this is *load shedding*).
- **Semaphores** bound the number of in-flight operations (Lesson 31's `TaskPool`).
- It is **lost** by: unbounded channels, spawning a task per input item with no limit, `buffer_unordered(huge)`, and adaptors that buffer internally without a bound. A pipeline is as bounded as its *least* bounded stage.
- Bounds compose by addition: the `Batcher` holds at most `size` items plus whatever the channel capacity allows, so the worst case is `capacity + size` items per stage (✓ test: with capacity 2 and a batcher not pulling, the third `send` waited).
- **Cycles can deadlock.** Two stages that send to each other through full bounded channels wait for each other forever (Lesson 29's circular wait with channel capacity as the resource). Keep pipelines acyclic or use `try_send` on the back edge.

A load-shedding service must also *tell the caller* (HTTP 429/503, a queue-full error) instead of silently accepting work it cannot finish. Overload handling is a design decision to make explicitly, not an accident of queue sizes.

---

## 6. Putting it together: `streamlab`

The practical crate has three pieces, each applying a rule from this lesson:

1. **`Batcher`**: groups channel items into batches by size or age. Its state (`buf`, `deadline`) lives in the struct, so `next_batch()` is cancel-safe (✓ tests), it uses one absolute deadline per batch (§3.1), and its worst-case memory is bounded (§5).
2. **`chunks`**: a `Stream` adaptor keeping its partial chunk in the struct (§4.2), never polling the inner stream after it ended (✓ test with a strict stream that panics if polled again).
3. **`CancelGuard`**: a drop guard with `defuse()` (✓ runs on abort and on timeout, not after defuse). Use it to undo or report half-done work.

---

## 7. Common mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| `select!` over a non-cancel-safe future in a loop (`read_exact`, a hand-written multi-step `async fn`) | Lost data / half-done work (✓ §2.1) | cancel-safe primitives; state in a struct; pin and reuse `&mut fut` |
| Creating `sleep(..)` inside the `select!` loop | Timer restarts on every other event; heartbeat or idle timeout never fires (✓ §3.1) | `interval`, or one pinned `Sleep` / absolute deadline |
| Putting required cleanup after an `.await` | Skipped on cancellation (✓ §1) | `Drop` guard; spawn the work; make it idempotent |
| Expecting async cleanup in `Drop` | `Drop` cannot await | spawn from `Drop` (needs a live runtime), or an explicit `shutdown().await` |
| `biased;` with an always-ready first branch | Later branches starve (✓ 1000 vs 0) | order branches deliberately; add fairness manually |
| Two-step update with an `.await` in between (debit … credit) | A cancel between the steps loses money | one atomic step; run it in a detached task; journal / compensate |
| Unbounded channel / unbounded spawn between fast producer and slow consumer | Memory growth (✓ 99 990 queued) | bounded channel, `Semaphore`, `buffer_unordered(n)` |
| Cyclic pipelines with bounded channels | Deadlock | acyclic design; `try_send` on the back edge |
| `buffered(n)` when order does not matter | One slow item stalls the rest | `buffer_unordered(n)` |
| Importing both `tokio_stream::StreamExt` and `futures::StreamExt` | Ambiguous method errors (from memory) | import one, or call the other by path |
| Treating `JoinHandle` drop as cancel | Task keeps running (Lesson 31) | `abort()` or a `JoinSet` |
| Assuming a timed-out operation did not happen | It may have had partial or complete effects | idempotent operations, request ids |

---

## 8. Performance, reliability, security

- **Cost model**: cancellation by drop is free at runtime (it is just `Drop`); the cost is correctness. Streams add one `poll_next` call per item per adaptor layer; `buffer_unordered` stores `n` futures. I give no benchmark numbers (Lesson 41).
- **Reliability**: write down, for every `select!` and `timeout` in your code, "what is lost if this branch is dropped right now?" If the answer is not "nothing", fix it. Every retry (Lesson 31's `retry`) and every timeout can re-execute or abandon an operation, so effects must be idempotent or identified by a request id.
- **Idempotency and exactly-once**: a cancelled request may have *already* taken effect at the other side (the response was lost). Never conclude "it did not happen" from a timeout; ask the other side, or retry safely.
- **Security / DoS**: unbounded queues and unbounded concurrency are denial-of-service vulnerabilities. Bound request size, queue depth, in-flight count and per-connection buffers; apply timeouts to every wait that a peer controls. Load shedding is a security control.
- **Blockchain-adjacent note**: transaction submission is the canonical cancel-safety trap. Dropping a future that was in the middle of "sign, submit, await confirmation" does not undo the broadcast transaction: the chain may still include it. Make submission idempotent (nonce / signature / transaction id), persist the intent before sending, and reconcile on restart (Lessons 51–52). Mempool and gossip ingress must be bounded and shed load rather than grow without limit.

---

## 9. Exercises

Answers are collapsed. Try each before opening it.

1. **(Warm-up)** What exactly happens to a future that is dropped while suspended at an `.await`?
2. **(Warm-up)** What does "cancel-safe" mean, and is `mpsc::Receiver::recv` cancel-safe?
3. **(Easy)** In §3.1's first loop, messages arrive every 30 ms and the heartbeat is a fresh `sleep(50 ms)` each iteration. How many heartbeats fire in 10 messages, and why?
4. **(Medium)** `buffer_unordered(2)` vs `buffered(2)` on tasks of 90, 30, 60, 10 ms: which order does each yield (roughly), and which one can be slowed by a single slow item?
5. **(Medium-hard)** Write a cancel-safe `FrameReader` over any `AsyncRead`: `next_frame().await` returns the next length-prefixed frame (`u32` big-endian length, then that many bytes), cancel-safe, with a maximum frame size to avoid memory attacks.
6. **(Hard)** A function does `db.debit(a, x).await; db.credit(b, x).await;`. Under `select!`/`timeout` it can be cancelled between the two awaits. Give three designs that make this safe.
7. **(Hard)** Why does a pipeline of three stages joined by `mpsc::channel(10)` stay bounded, but one with an `unbounded_channel` in the middle does not?
8. **(Hard)** Two tasks forward messages to each other through `mpsc::channel(1)`. Describe a deadlock and two ways to avoid it.

<details markdown="1">
<summary>Answers</summary>

1. The compiler-generated state machine is dropped: every local alive at that suspension point is dropped (so their `Drop` impls run), and no code after the `.await` ever runs (✓ §1: guard dropped, "later code" did not run).
2. An operation is cancel-safe if dropping its future at any await point loses no data and leaves no broken state. `recv` is documented as cancel-safe (from memory) and ✓ a message sent after the `recv` was dropped was still delivered.
3. **Zero.** Each loop iteration creates a fresh 50 ms sleep; the message branch wins every ~30 ms, dropping that sleep (✓ `heartbeats=0`).
4. `buffer_unordered(2)`: completion order, e.g. `[2, 1, 3, 4]` ✓. `buffered(2)`: input order `[1, 2, 3, 4]` ✓. `buffered` can be held up by a slow early item (item 1 at 90 ms delays the delivery of items 2 and 3 even though they finished earlier).
5. ```rust
   use std::io;
   use tokio::io::{AsyncRead, AsyncReadExt};

   const MAX_FRAME: usize = 1 << 20;

   pub struct FrameReader<R> { r: R, buf: Vec<u8> }

   impl<R: AsyncRead + Unpin> FrameReader<R> {
       pub fn new(r: R) -> Self { Self { r, buf: Vec::new() } }

       /// Cancel-safe: partial bytes stay in `self.buf`.
       pub async fn next_frame(&mut self) -> io::Result<Option<Vec<u8>>> {
           loop {
               if self.buf.len() >= 4 {
                   let len = u32::from_be_bytes([self.buf[0], self.buf[1], self.buf[2], self.buf[3]]) as usize;
                   if len > MAX_FRAME {
                       return Err(io::Error::new(io::ErrorKind::InvalidData, "frame too large"));
                   }
                   if self.buf.len() >= 4 + len {
                       let frame = self.buf[4..4 + len].to_vec();
                       self.buf.drain(..4 + len);
                       return Ok(Some(frame));
                   }
               }
               let mut chunk = [0u8; 1024];
               let n = self.r.read(&mut chunk).await?; // `read` is cancel-safe: no byte is lost if dropped here
               if n == 0 {
                   return if self.buf.is_empty() { Ok(None) } else { Err(io::ErrorKind::UnexpectedEof.into()) };
               }
               self.buf.extend_from_slice(&chunk[..n]);
           }
       }
   }
   ```
   ✓ I compiled and ran it against a `tokio::io::duplex` pipe: half a frame written, `next_frame` dropped by a timeout, the rest written, and the next call returned the full frame; an oversized length returned `InvalidData`. The key difference from `read_exact`: partial progress is saved in `self.buf`, which survives the drop.
6. (a) **One atomic operation**: a database transaction (`transfer` as a single call that commits both or neither). (b) **Run it to completion in a spawned task** and await the `JoinHandle`; if the caller is cancelled the handle is dropped but the task finishes (detach). (c) **Compensate**: a `CancelGuard` created after `debit` that, if dropped without being defused, spawns a `credit` back (or records an intent in a journal for a recovery process); defuse after `credit` succeeds. Also make every step idempotent (operation id) so retries are safe.
7. With bounded channels each `send().await` waits when the next stage is behind, so the slowest stage paces all the previous ones and total in-flight work is the sum of the capacities. An unbounded channel never makes the sender wait, so the stage before it runs at full speed and the queue grows without limit (✓ 99 990 queued).
8. Task A is blocked in `send` to B because B's inbox is full, while B is blocked in `send` to A because A's inbox is full; neither receives, so neither frees space: circular wait. Avoid by (i) breaking the cycle: one direction uses `try_send` and drops or errors on `Full`; (ii) giving one side an unbounded or larger queue *plus* another limit (for example a semaphore); or (iii) restructuring as an acyclic pipeline, or having each task `select!` between receiving and sending so it keeps draining its own inbox while it waits to send.

</details>

---

## 10. Debugging challenge

A connection handler must process incoming messages **and** send a keep-alive ping every 50 ms. In a test where messages arrive every 30 ms, no ping is ever sent, and the peer eventually drops the connection as dead. (The clock is paused, so the run is instant and exact.)

```rust
use std::time::Duration;
use tokio::sync::mpsc;
use tokio::time::sleep;

#[tokio::main(flavor = "current_thread", start_paused = true)]
async fn main() {
    let (tx, mut rx) = mpsc::channel::<u32>(16);
    tokio::spawn(async move {
        for i in 0..10 {
            tx.send(i).await.unwrap();
            sleep(Duration::from_millis(30)).await; // a message every 30 ms
        }
    });
    let (mut pings, mut handled) = (0, 0);
    loop {
        tokio::select! {
            m = rx.recv() => match m { Some(_) => handled += 1, None => break },
            _ = sleep(Duration::from_millis(50)) => pings += 1, // "send a keep-alive ping every 50 ms"
        }
    }
    println!("handled={handled} pings={pings}");
}
```

✓ It prints `handled=10 pings=0`. Explain why, and fix it in two different ways.

<details markdown="1">
<summary>Solution</summary>

Each loop iteration creates a **new** `sleep(50 ms)`. A message arrives every 30 ms, so the `recv` branch always wins first; `select!` then drops the fresh sleep, and the next iteration starts another 50 ms from zero. The ping timer is restarted by every message and, as long as traffic is faster than the ping period, can never expire. It is the same bug as §3.1, and it only shows up with the "wrong" traffic rate, which is why it survives code review and light testing (it would fire fine on an idle connection, which is exactly when you least need a keep-alive).

Two fixes, both ✓ run:

```rust
use std::time::Duration;
use tokio::sync::mpsc;
use tokio::time::{interval, sleep, Instant};

async fn producer(tx: mpsc::Sender<u32>) {
    for i in 0..10 {
        tx.send(i).await.unwrap();
        sleep(Duration::from_millis(30)).await;
    }
}

#[tokio::main(flavor = "current_thread", start_paused = true)]
async fn main() {
    // Fix 1: interval
    let (tx, mut rx) = mpsc::channel::<u32>(16);
    tokio::spawn(producer(tx));
    let mut tick = interval(Duration::from_millis(50));
    tick.tick().await; // the first tick completes immediately; skip it
    let (mut pings, mut handled) = (0, 0);
    loop {
        tokio::select! {
            m = rx.recv() => match m { Some(_) => handled += 1, None => break },
            _ = tick.tick() => pings += 1,
        }
    }
    println!("interval: handled={handled} pings={pings}");

    // Fix 2: one pinned Sleep, reset after each ping
    let (tx, mut rx) = mpsc::channel::<u32>(16);
    tokio::spawn(producer(tx));
    let next_ping = sleep(Duration::from_millis(50));
    tokio::pin!(next_ping);
    let (mut pings, mut handled) = (0, 0);
    loop {
        tokio::select! {
            m = rx.recv() => match m { Some(_) => handled += 1, None => break },
            _ = &mut next_ping => {
                pings += 1;
                next_ping.as_mut().reset(Instant::now() + Duration::from_millis(50));
            }
        }
    }
    println!("pinned+reset: handled={handled} pings={pings}");
}
```

✓ Output:

```text
interval: handled=10 pings=6
pinned+reset: handled=10 pings=6
```

Both send pings at 50, 100, 150, 200, 250 and 300 ms (the producer finishes at 300 ms, so the sixth ping races with the channel closing; I print what I saw). `interval` keeps a fixed schedule independent of other events; the pinned `Sleep` is created once, survives the other branch winning, and is `reset` explicitly after each ping. Choose `interval` when you want a fixed rate, and a pinned, resettable `Sleep` when the deadline depends on events (for example "ping 50 ms after the *last* activity", where you would also `reset` it in the message branch).

</details>

---

## 11. Practical challenge: `streamlab`

Build a library crate with:

1. `Batcher<T>`: `new(rx, size, max_wait)`, `next_batch()` returning a batch when it has `size` items, or `max_wait` after its **first** item, or when the channel closes (flush the partial batch, then `None`). **Cancel-safe**; the deadline is not reset by later items; `pending()` reports buffered items.
2. `chunks(stream, n)`: `Stream` adaptor grouping items into `Vec`s of `n` (last may be shorter), fused after the end, never polling the inner stream after its end, cancel-safe.
3. `CancelGuard`: closure runs on drop unless `defuse()` was called.

Tests: paused clock for exact timing, a cancel-safety test for each, a backpressure test, a "strict" inner stream that panics if polled after the end.

<details markdown="1">
<summary>Solution (spoilers: full source)</summary>

`Cargo.toml`:

```toml
[package]
name = "streamlab"
version = "0.1.0"
edition = "2024"

[dependencies]
tokio = { version = "1.54.1", features = ["sync", "time", "macros"] }
tokio-stream = { version = "0.1.19", default-features = false }

[dev-dependencies]
tokio = { version = "1.54.1", features = ["rt", "sync", "time", "macros", "test-util"] }
tokio-stream = { version = "0.1.19", default-features = false }
```

`src/lib.rs`:

```rust
//! `streamlab`: cancellation-safe building blocks for async pipelines.
//!
//! * [`Batcher`]: groups items from a channel into batches by size or age.
//! * [`chunks`]: a `Stream` adaptor that groups items into fixed-size `Vec`s.
//! * [`CancelGuard`]: runs a closure when a task or future is dropped, unless defused.
//!
//! The rule behind the design: **state that must survive cancellation lives in
//! the struct, not in the local variables of an `async fn`.** A future that
//! is dropped at an `.await` loses its locals; a struct does not.

use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::Duration;

use tokio::sync::mpsc;
use tokio::time::{Instant, sleep_until};
use tokio_stream::Stream;

// ----------------------------------------------------------------- Batcher

/// Collects items from an `mpsc` receiver into batches. A batch is returned
/// as soon as it holds `size` items, or `max_wait` after its **first** item
/// arrived, or when the channel closes (the last partial batch is flushed).
///
/// [`Batcher::next_batch`] is **cancel-safe**: if the future is dropped (for
/// example by `select!` or `timeout`), no item is lost and the deadline of the
/// batch in progress is kept, because both live in the `Batcher`.
///
/// Memory is bounded: at most `size` items are held here, plus whatever the
/// channel's capacity allows. A full channel makes producers' `send` wait.
///
/// ```
/// # #[tokio::main(flavor = "current_thread")]
/// # async fn main() {
/// let (tx, rx) = tokio::sync::mpsc::channel(8);
/// let mut b = streamlab::Batcher::new(rx, 2, std::time::Duration::from_secs(1));
/// for i in 1..=3 {
///     tx.send(i).await.unwrap();
/// }
/// drop(tx);
/// assert_eq!(b.next_batch().await, Some(vec![1, 2]));
/// assert_eq!(b.next_batch().await, Some(vec![3]));
/// assert_eq!(b.next_batch().await, None);
/// # }
/// ```
pub struct Batcher<T> {
    rx: mpsc::Receiver<T>,
    size: usize,
    max_wait: Duration,
    buf: Vec<T>,
    /// `Some` exactly when `buf` is non-empty.
    deadline: Option<Instant>,
}

impl<T> Batcher<T> {
    /// # Panics
    /// If `size` is zero.
    pub fn new(rx: mpsc::Receiver<T>, size: usize, max_wait: Duration) -> Self {
        assert!(size > 0, "size must be at least 1");
        Self {
            rx,
            size,
            max_wait,
            buf: Vec::with_capacity(size),
            deadline: None,
        }
    }

    /// Items received but not yet returned.
    pub fn pending(&self) -> usize {
        self.buf.len()
    }

    fn take(&mut self) -> Vec<T> {
        self.deadline = None;
        std::mem::take(&mut self.buf)
    }

    /// The next batch, or `None` once the channel is closed and drained.
    pub async fn next_batch(&mut self) -> Option<Vec<T>> {
        loop {
            if self.buf.len() >= self.size {
                return Some(self.take());
            }
            match self.deadline {
                // Nothing buffered: wait for the first item. `recv` is cancel-safe.
                None => match self.rx.recv().await {
                    Some(item) => {
                        self.buf.push(item);
                        self.deadline = Some(Instant::now() + self.max_wait);
                    }
                    None => return None,
                },
                Some(deadline) => {
                    tokio::select! {
                        item = self.rx.recv() => match item {
                            Some(item) => self.buf.push(item),
                            // Closed: flush what we have.
                            None => return Some(self.take()),
                        },
                        // One absolute deadline for the whole batch. Writing
                        // `sleep(self.max_wait)` here instead would restart the
                        // timer every time an item arrives.
                        _ = sleep_until(deadline) => return Some(self.take()),
                    }
                }
            }
        }
    }
}

// ------------------------------------------------------------------ Chunks

/// Stream adaptor returned by [`chunks`].
#[must_use = "streams do nothing unless polled"]
pub struct Chunks<S: Stream + Unpin> {
    inner: S,
    n: usize,
    buf: Vec<S::Item>,
    done: bool,
}

/// Groups the items of `stream` into `Vec`s of `n`; the last `Vec` may be
/// shorter. It never waits for a full chunk once the stream has ended, and it
/// does **not** flush on a timer: a partial chunk stays buffered while the
/// inner stream is `Pending` (use [`Batcher`] when you need a time limit).
///
/// Because the partial chunk lives in the adaptor, `next().await` on it is
/// cancel-safe.
///
/// ```
/// use tokio_stream::StreamExt;
/// # #[tokio::main(flavor = "current_thread")]
/// # async fn main() {
/// let mut s = streamlab::chunks(tokio_stream::iter(1..=5), 2);
/// assert_eq!(s.next().await, Some(vec![1, 2]));
/// assert_eq!(s.next().await, Some(vec![3, 4]));
/// assert_eq!(s.next().await, Some(vec![5]));
/// assert_eq!(s.next().await, None);
/// # }
/// ```
///
/// # Panics
/// If `n` is zero.
pub fn chunks<S: Stream + Unpin>(stream: S, n: usize) -> Chunks<S> {
    assert!(n > 0, "chunk size must be at least 1");
    Chunks {
        inner: stream,
        n,
        buf: Vec::with_capacity(n),
        done: false,
    }
}

// `Vec<T>` is `Unpin` only when `T` is (the auto trait looks through its
// `PhantomData<T>`), so the compiler cannot derive `Unpin` for `Chunks<S>`
// when `S::Item` is not `Unpin`. The items are stored, never pinned in place,
// and `inner` is `Unpin`, so asserting it is sound and needs no `unsafe`.
impl<S: Stream + Unpin> Unpin for Chunks<S> {}

impl<S: Stream + Unpin> Stream for Chunks<S> {
    type Item = Vec<S::Item>;

    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>> {
        // `Chunks<S>` is `Unpin` (impl above), so `get_mut` is allowed.
        let this = self.get_mut();
        if this.done {
            return Poll::Ready(None);
        }
        loop {
            match Pin::new(&mut this.inner).poll_next(cx) {
                Poll::Ready(Some(item)) => {
                    this.buf.push(item);
                    if this.buf.len() == this.n {
                        return Poll::Ready(Some(std::mem::take(&mut this.buf)));
                    }
                }
                Poll::Ready(None) => {
                    this.done = true;
                    return if this.buf.is_empty() {
                        Poll::Ready(None)
                    } else {
                        Poll::Ready(Some(std::mem::take(&mut this.buf)))
                    };
                }
                Poll::Pending => return Poll::Pending,
            }
        }
    }
}

// ------------------------------------------------------------- CancelGuard

/// Runs a closure when dropped, unless [`CancelGuard::defuse`] was called.
///
/// Dropping is how a cancelled future or aborted task cleans up: its locals
/// are dropped, and nothing after the last `.await` runs. `Drop` cannot
/// `.await`, so the closure must be synchronous (it may send on a channel or
/// spawn follow-up work).
///
/// ```
/// let guard = streamlab::CancelGuard::new(|| println!("cleanup"));
/// guard.defuse(); // success path: nothing printed
/// ```
pub struct CancelGuard<F: FnOnce()> {
    on_drop: Option<F>,
}

impl<F: FnOnce()> CancelGuard<F> {
    pub fn new(on_drop: F) -> Self {
        Self {
            on_drop: Some(on_drop),
        }
    }

    /// Disarms the guard: the closure will not run.
    pub fn defuse(mut self) {
        self.on_drop = None;
    }
}

impl<F: FnOnce()> Drop for CancelGuard<F> {
    fn drop(&mut self) {
        if let Some(f) = self.on_drop.take() {
            f();
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::sync::Arc;
    use std::sync::atomic::{AtomicUsize, Ordering};
    use tokio::time::{sleep, timeout};
    use tokio_stream::StreamExt;
    use tokio_stream::wrappers::ReceiverStream;

    const MS: fn(u64) -> Duration = Duration::from_millis;

    #[tokio::test(start_paused = true)]
    async fn batches_by_size_then_flushes_the_rest_on_close() {
        let (tx, rx) = mpsc::channel(16);
        let mut b = Batcher::new(rx, 3, MS(1000));
        for i in 1..=5 {
            tx.send(i).await.unwrap();
        }
        drop(tx);
        assert_eq!(b.next_batch().await, Some(vec![1, 2, 3]));
        assert_eq!(b.next_batch().await, Some(vec![4, 5]));
        assert_eq!(b.next_batch().await, None);
        assert_eq!(b.next_batch().await, None, "stays closed");
    }

    #[tokio::test(start_paused = true)]
    async fn a_partial_batch_is_flushed_after_max_wait() {
        let (tx, rx) = mpsc::channel(16);
        let mut b = Batcher::new(rx, 10, MS(50));
        let start = Instant::now();
        tx.send('a').await.unwrap();
        let t2 = tx.clone();
        tokio::spawn(async move {
            sleep(MS(20)).await;
            t2.send('b').await.unwrap();
        });
        assert_eq!(b.next_batch().await, Some(vec!['a', 'b']));
        assert_eq!(start.elapsed(), MS(50));
        drop(tx);
    }

    #[tokio::test(start_paused = true)]
    async fn the_deadline_counts_from_the_first_item_not_the_latest() {
        let (tx, rx) = mpsc::channel(16);
        let mut b = Batcher::new(rx, 10, MS(50));
        tokio::spawn(async move {
            for i in 0..3u32 {
                tx.send(i).await.unwrap();
                sleep(MS(30)).await; // items at t = 0, 30, 60 ms
            }
        });
        let start = Instant::now();
        assert_eq!(b.next_batch().await, Some(vec![0, 1]));
        assert_eq!(start.elapsed(), MS(50));
        assert_eq!(b.next_batch().await, Some(vec![2])); // closed at t=90 -> flush
    }

    #[tokio::test(start_paused = true)]
    async fn next_batch_is_cancel_safe() {
        let (tx, rx) = mpsc::channel(16);
        let mut b = Batcher::new(rx, 5, MS(100));
        tx.send(1).await.unwrap();
        tx.send(2).await.unwrap();
        // Drop the future mid-way, twice.
        assert!(timeout(MS(10), b.next_batch()).await.is_err());
        tx.send(3).await.unwrap();
        assert!(timeout(MS(10), b.next_batch()).await.is_err());
        assert_eq!(b.pending(), 3, "items survive cancellation");
        // The original deadline (100 ms after item 1) is still in force.
        let start = Instant::now();
        assert_eq!(b.next_batch().await, Some(vec![1, 2, 3]));
        assert!(start.elapsed() <= MS(80));
        drop(tx);
    }

    #[tokio::test(start_paused = true)]
    async fn a_full_channel_blocks_producers_until_the_batcher_pulls() {
        let (tx, rx) = mpsc::channel(2);
        let mut b = Batcher::new(rx, 2, MS(1000));
        tx.send(1).await.unwrap();
        tx.send(2).await.unwrap();
        let blocked = timeout(MS(50), tx.send(3)).await;
        assert!(
            blocked.is_err(),
            "the channel is full and nobody is pulling"
        );
        assert_eq!(b.next_batch().await, Some(vec![1, 2]));
        tx.send(3).await.unwrap(); // room again
        drop(tx);
        assert_eq!(b.next_batch().await, Some(vec![3]));
    }

    #[test]
    #[should_panic(expected = "size must be at least 1")]
    fn zero_batch_size_panics() {
        let (_tx, rx) = mpsc::channel::<u8>(1);
        let _ = Batcher::new(rx, 0, MS(1));
    }

    #[tokio::test]
    async fn chunks_group_and_flush_the_remainder() {
        let got: Vec<Vec<u32>> = chunks(tokio_stream::iter(1..=7), 3).collect().await;
        assert_eq!(got, vec![vec![1, 2, 3], vec![4, 5, 6], vec![7]]);
        let exact: Vec<Vec<u32>> = chunks(tokio_stream::iter(1..=4), 2).collect().await;
        assert_eq!(exact, vec![vec![1, 2], vec![3, 4]]);
        let none: Vec<Vec<u32>> = chunks(tokio_stream::iter(Vec::<u32>::new()), 2)
            .collect()
            .await;
        assert!(none.is_empty());
        let ones: Vec<Vec<u32>> = chunks(tokio_stream::iter(1..=2), 1).collect().await;
        assert_eq!(ones, vec![vec![1], vec![2]]);
    }

    #[tokio::test(start_paused = true)]
    async fn chunks_keep_a_partial_chunk_while_the_source_is_pending() {
        let (tx, rx) = mpsc::channel(8);
        let mut s = chunks(ReceiverStream::new(rx), 3);
        tx.send(1).await.unwrap();
        tx.send(2).await.unwrap();
        // Cancel `next()` while one chunk is half full.
        assert!(timeout(MS(10), s.next()).await.is_err());
        tx.send(3).await.unwrap();
        assert_eq!(s.next().await, Some(vec![1, 2, 3]), "nothing was lost");
        tx.send(4).await.unwrap();
        drop(tx);
        assert_eq!(s.next().await, Some(vec![4]));
        assert_eq!(s.next().await, None);
        assert_eq!(s.next().await, None, "fused after the end");
    }

    /// A stream that panics if polled again after it returned `None`.
    struct Strict(u32, bool);
    impl Stream for Strict {
        type Item = u32;
        fn poll_next(mut self: Pin<&mut Self>, _: &mut Context<'_>) -> Poll<Option<u32>> {
            assert!(!self.1, "inner stream polled after it ended");
            if self.0 == 0 {
                self.1 = true;
                Poll::Ready(None)
            } else {
                self.0 -= 1;
                Poll::Ready(Some(self.0))
            }
        }
    }

    #[tokio::test]
    async fn chunks_never_poll_the_inner_stream_after_its_end() {
        let mut s = chunks(Strict(3, false), 2);
        assert_eq!(s.next().await, Some(vec![2, 1]));
        assert_eq!(s.next().await, Some(vec![0]));
        assert_eq!(s.next().await, None);
        assert_eq!(s.next().await, None);
    }

    #[test]
    #[should_panic(expected = "chunk size must be at least 1")]
    fn zero_chunk_size_panics() {
        let _ = chunks(tokio_stream::iter(0..1), 0);
    }

    #[test]
    fn guard_runs_on_drop_and_not_after_defuse() {
        let n = Arc::new(AtomicUsize::new(0));
        let n2 = Arc::clone(&n);
        drop(CancelGuard::new(move || {
            n2.fetch_add(1, Ordering::SeqCst);
        }));
        assert_eq!(n.load(Ordering::SeqCst), 1);
        let n3 = Arc::clone(&n);
        CancelGuard::new(move || {
            n3.fetch_add(10, Ordering::SeqCst);
        })
        .defuse();
        assert_eq!(n.load(Ordering::SeqCst), 1, "defused guard must not run");
    }

    #[tokio::test(start_paused = true)]
    async fn guard_runs_when_a_task_is_aborted_or_a_future_times_out() {
        let n = Arc::new(AtomicUsize::new(0));
        let n2 = Arc::clone(&n);
        let h = tokio::spawn(async move {
            let _g = CancelGuard::new(move || {
                n2.fetch_add(1, Ordering::SeqCst);
            });
            sleep(Duration::from_secs(60)).await;
        });
        sleep(MS(10)).await;
        h.abort();
        let _ = h.await;
        assert_eq!(n.load(Ordering::SeqCst), 1);

        let n3 = Arc::clone(&n);
        let r = timeout(MS(10), async move {
            let _g = CancelGuard::new(move || {
                n3.fetch_add(10, Ordering::SeqCst);
            });
            sleep(Duration::from_secs(60)).await;
        })
        .await;
        assert!(r.is_err());
        assert_eq!(n.load(Ordering::SeqCst), 11);
    }

    #[tokio::test(start_paused = true)]
    async fn guard_is_defused_on_the_success_path() {
        let n = Arc::new(AtomicUsize::new(0));
        let n2 = Arc::clone(&n);
        let r = timeout(MS(100), async move {
            let g = CancelGuard::new(move || {
                n2.fetch_add(1, Ordering::SeqCst);
            });
            sleep(MS(10)).await;
            g.defuse();
            "finished"
        })
        .await;
        assert_eq!(r, Ok("finished"));
        assert_eq!(n.load(Ordering::SeqCst), 0);
    }
}
```

**Verification ✓.** `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings` and `cargo test --locked` pass: 13 unit tests and 3 doctests.

**Mutation checks ✓** (each mutation applied to a copy):

| # | Mutation | Result |
|---|---|---|
| M1 | `sleep_until(deadline)` → `sleep(self.max_wait)` (timer restarts) | **caught** by 3 tests (flush timing, cancel-safety, deadline-from-first-item) |
| M2 | closed channel returns `None` instead of flushing the partial batch | **caught** by 3 tests |
| M3 | `chunks` drops the final partial chunk | **caught** by 2 tests |
| M4 | `defuse()` does nothing | **caught** by 2 tests |
| M5 | `done` flag never set after the end | **not caught by my first 12 tests** (the inner `ReceiverStream` happily returns `None` again, so the bug was invisible); **caught** after I added `chunks_never_poll_the_inner_stream_after_its_end` with a strict inner stream |
| M6 | `take()` does not clear the deadline | **caught** by 2 tests |
| M7 | `>= size` → `> size` (batch one item too big) | **caught** by 1 test |
| M8 | `next_batch` clears its buffer on entry (loses items on cancel) | **caught** by `next_batch_is_cancel_safe` |

**Design notes.**

- `deadline` is `Some` exactly when `buf` is non-empty; it is set when the *first* item arrives and cleared by `take()`. Everything that must survive cancellation is a field.
- The only awaits inside `next_batch` are `recv` (cancel-safe) and `sleep_until` (holds no data), which is why the whole function is cancel-safe. Adding any other await between "take an item" and "store it" would break that.
- `Chunks` implements `Unpin` explicitly (§4.2) and uses `Pin::new(&mut this.inner)`, so there is no `unsafe`.
- The test `a_full_channel_blocks_producers_until_the_batcher_pulls` shows the channel bound; with the batcher's own buffer the in-flight worst case is `capacity + size`.
- Not handled: no time-based flush in `chunks` (use `Batcher`), `Batcher` is single-consumer, `CancelGuard` closures cannot `.await`, and no metrics on dropped/shed items.

</details>

---

## 12. Interview questions

1. How do you cancel a future in Rust, and what happens to its local variables?
2. What is cancel safety? Give a cancel-safe and a cancel-unsafe example.
3. Why does `loop { select! { x = rx.recv() => .., _ = sleep(d) => .. } }` often never time out?
4. What does `biased;` do in `select!` and what is the risk?
5. What is a `Stream` and how does it differ from an `Iterator` and from a `Future`?
6. `buffer_unordered(n)` vs `buffered(n)` vs spawning tasks?
7. What is backpressure, and how can an unbounded channel defeat it?
8. How would you design a "debit then credit" operation that must survive cancellation?

<details markdown="1">
<summary>Answers</summary>

1. By dropping it. The state machine is dropped, so the locals alive at the current `.await` are dropped (their `Drop` runs); code after the `.await` never runs (✓ §1).
2. Cancel-safe: dropping the future at any await loses nothing. `mpsc::Receiver::recv` is (✓ message still delivered after a dropped `recv`). Unsafe: `read_exact` (✓ the three bytes already read were lost) or any `async fn` that takes an item from a source and awaits before storing it.
3. A new `sleep` is created on every iteration; whenever the other branch wins, the sleep is dropped and the timer restarts, so if events arrive more often than `d` the timeout branch never fires (✓ 0 heartbeats). Use `interval`, or create the sleep once and `select!` on `&mut` pinned sleep, resetting it deliberately.
4. By default branches are polled in random order; `biased;` polls top to bottom. It lets you prioritize (check shutdown first) but a branch that is always ready starves the later ones (✓ 1000 vs 0).
5. An `Iterator` yields items synchronously (`next()` may block). A `Future` yields **one** value, later. A `Stream` yields **zero or more** values over time (`poll_next`, `Pending` while waiting). It is lazy and consumed with `StreamExt`.
6. `buffer_unordered(n)`: at most `n` futures concurrently on one task, yielding in completion order (✓ peak 2); `buffered(n)`: same but input order (head-of-line blocking). Spawning tasks gives parallelism across threads and independent lifetimes but needs `'static + Send` and your own concurrency limit.
7. Backpressure = a slow consumer slows the producer. Bounded channels make `send` wait when full. An unbounded channel never makes the sender wait, so the producer runs ahead and the queue grows without limit (✓ 99 990 queued).
8. Make it one atomic transaction; or run it in a task that is not cancelled with the caller (detach); or use idempotent steps plus a compensation/journal guard that completes or reverses the first step if the second never happens (Exercise 6).

</details>

---

## 13. Key takeaways

- Cancellation is **dropping a future**: locals are dropped, `Drop` runs, nothing after the `.await` runs (✓). `select!`, `timeout`, `abort` and dropping a `JoinSet` all cancel.
- Know what is lost if any await is cut: `read_exact` loses bytes (✓), `recv` does not (✓). Keep state in structs, or pin and reuse a future (`&mut fut`), or detach the multi-step work.
- Never create the timer inside a `select!` loop (✓ 0 heartbeats): use `interval` or an absolute deadline.
- `biased;` makes branch order deterministic and can starve later branches (✓ 1000 vs 0).
- Streams are lazy async sequences (`poll_next`); `buffer_unordered(n)` bounds concurrency (✓ peak 2); adaptors should keep state in the struct, which makes them cancel-safe. `Vec<T>` is `Unpin` only if `T` is (✓ E0277).
- Backpressure needs **bounded** everything: bounded channels, semaphores, bounded buffers (✓ paced producer vs 99 990 backlog). Decide the overload policy explicitly.
- A timeout means "unknown outcome", not "did not happen": make operations idempotent.

**Connections.** Lesson 30 explained why dropping a future stops it; Lesson 31's `TaskPool`, `retry` and graceful shutdown rely on the cancellation rules here; Lesson 29's circular-wait analysis applies to bounded pipelines; Lesson 25's `Pin` explains `&mut fut` and the `Unpin` impl. Next: Lesson 33 starts Module 11 with modules, visibility, API design and `rustdoc`.

---

## 14. References

- Tokio `select!` macro docs (fairness, `biased`, preconditions, **cancellation safety** section): <https://docs.rs/tokio/latest/tokio/macro.select.html>
- Tokio tutorial chapters "Select" and "Streams": <https://tokio.rs/tokio/tutorial/select> and <https://tokio.rs/tokio/tutorial/streams>
- `tokio_stream` (`StreamExt`, `wrappers::ReceiverStream`, `iter`): <https://docs.rs/tokio-stream>
- `futures::stream` (`StreamExt::buffer_unordered`, `buffered`, `unfold`): <https://docs.rs/futures/latest/futures/stream/index.html>
- `tokio::sync::mpsc` (bounded vs unbounded, `try_send`): <https://docs.rs/tokio/latest/tokio/sync/mpsc/index.html>
- *Asynchronous Programming in Rust* (the Async Book), chapters on `select!` and streams: <https://rust-lang.github.io/async-book/>

The URLs above are the standard documentation locations; I did not open them during this session (the sandbox cannot reach them, though it can download crates from crates.io), so if a link has moved, search the docs site for the page title. Unchecked or from memory: which specific Tokio functions are documented as cancel-safe or not (`recv`, `read`, `accept`, `read_exact`, `write_all`, stream `next()`, `Notify::notified`, `Semaphore::acquire`); that async `Drop` is not stable and that std has no stable `Stream`/`AsyncIterator` in 1.97; the exact `Stream` trait definition; that a runtime shutting down may skip a spawned cleanup; `select!`'s `else` branch and the ambiguity error when both `StreamExt` traits are imported; whether specific web frameworks drop handlers on client disconnect; and the chain-submission remarks. Everything marked ✓ was run on this machine's `rustc 1.97.0` with `tokio` 1.54.1, `tokio-stream` 0.1.19 and `futures` 0.3.34.
