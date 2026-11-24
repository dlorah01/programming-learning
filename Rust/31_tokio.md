---
title: "Lesson 31 — Tokio in practice"
---

# Lesson 31 — Tokio in practice

**Module 10 · Concurrency and async** · Status: generated · Edition used in examples: **2024** · External dependencies: `tokio` 1.54.1 and `tokio-util` 0.7.20 (✓ both downloaded from crates.io and run; versions are what `Cargo.lock` resolved in this sandbox today)

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux with 2 cores. As before, ✓ marks outputs, diagnostics and measurements I actually ran; unmarked statements are documented behavior I did not run. Everything is **stable Rust**. Timing in async programs is nondeterministic: the examples print comparisons ("took at least", "finished in this order") instead of raw milliseconds, and the tests use Tokio's **paused clock** where exactness matters. I could not open the Tokio documentation from the sandbox, so API statements not marked ✓ come from memory and are listed in the closing paragraph.

Builds on [Lesson 30](30_futures_async.md) (futures, wakers, executors, `Send + 'static`), [Lesson 29](29_locks_deadlocks.md) (locks, deadlocks, `Semaphore`-like gating) and [Lesson 27](27_threads_channels.md) (channels, graceful pool shutdown). Cancellation details, streams and backpressure in depth are Lesson 32.

---

## 1. What Tokio is

Lesson 30 built a toy executor. Tokio is the production version of the same idea, plus everything a toy lacks:

| Part | What it does |
|---|---|
| **Scheduler** | Runs tasks. `current_thread` (one thread) or `multi_thread` (a pool of workers; from memory, with work stealing). |
| **I/O driver ("reactor")** | Watches sockets via the OS readiness API (epoll/kqueue/IOCP, through the `mio` crate ✓ in the lockfile) and wakes the right task. |
| **Timer** | One shared timer structure for all `sleep`/`timeout`/`interval` calls (no thread per sleep, unlike Lesson 30's teaching version). |
| **Blocking pool** | Extra OS threads for `spawn_blocking`, so blocking work does not stall the async workers. |
| **Sync primitives** | Async-aware channels, mutex, semaphore, notify, barrier in `tokio::sync`. |
| **I/O types** | `TcpListener`, `TcpStream`, `UdpSocket`, files, process, signals, and the `AsyncRead`/`AsyncWrite` traits with extension methods. |

It is split into feature flags. `features = ["full"]` turns on everything (convenient for learning; a library should enable only what it uses, as the practical crate does: `rt`, `sync`, `time`, `net`, `io-util`, `macros`). `test-util` adds the paused clock (§9).

Facts worth knowing:

- Tokio is **a crate, not part of the language** (Lesson 30). Other runtimes exist (async-std, smol; from memory), and libraries that are written against Tokio's I/O types need Tokio. A library that wants to be runtime-agnostic should avoid depending on a specific runtime; this is a real ecosystem split.
- **Runtime flavors.** ✓ `Runtime::new()` (multi-thread) had **2** workers here (this machine has 2 cores); `Builder::new_current_thread()` had **1**; `Builder::new_multi_thread().worker_threads(3)` had **3** (read with `rt.metrics().num_workers()`). The default worker count is the number of CPU cores (from memory of the docs, consistent with what I saw).
- **`#[tokio::main]`** is a macro that builds a runtime and calls `block_on` on your `async fn main` body (from memory; it is the same as writing `Runtime::new().unwrap().block_on(async { ... })` by hand). `#[tokio::main(flavor = "current_thread")]` picks the single-thread flavor.
- **There must be a runtime.** ✓ Calling `tokio::spawn` with no runtime running panics with "there is no reactor running, must be called from the context of a Tokio 1.x runtime".
- **No runtime inside a runtime.** ✓ Creating a second runtime and calling `block_on` from inside an async task panics: "Cannot start a runtime from within a runtime. This happens because a function (like `block_on`) attempted to block the current thread while the thread is being used to drive asynchronous tasks. If you are in an async function, use `.await` on the future instead of blocking on it."

---

## 2. Tasks

```rust
use std::time::Duration;
use tokio::task::JoinSet;
use tokio::time::{sleep, timeout, Instant};

#[tokio::main]
async fn main() {
    // spawn + JoinHandle
    let h = tokio::spawn(async { 6 * 7 });
    println!("spawn -> {}", h.await.unwrap());

    // a panic is caught and reported through the JoinHandle
    let h = tokio::spawn(async { if true { panic!("task boom"); } 1 });
    let e = h.await.unwrap_err();
    println!("is_panic={} is_cancelled={}", e.is_panic(), e.is_cancelled());

    // abort
    let h = tokio::spawn(async { sleep(Duration::from_secs(60)).await; 1 });
    h.abort();
    let e = h.await.unwrap_err();
    println!("after abort: is_cancelled={}", e.is_cancelled());

    // join!
    let t = Instant::now();
    let (a, b) = tokio::join!(
        async { sleep(Duration::from_millis(100)).await; 1 },
        async { sleep(Duration::from_millis(100)).await; 2 }
    );
    println!("join! {} {} concurrent={}", a, b, t.elapsed() < Duration::from_millis(190));

    // select! drops the loser
    let r = tokio::select! {
        _ = sleep(Duration::from_millis(30)) => "timer",
        _ = sleep(Duration::from_secs(10)) => "slow",
    };
    println!("select -> {r}");

    // timeout
    let r = timeout(Duration::from_millis(30), sleep(Duration::from_secs(5))).await;
    println!("timeout -> {:?}", r.is_err());
    let r = timeout(Duration::from_millis(300), async { 9 }).await;
    println!("timeout ok -> {:?}", r);

    // JoinSet: results arrive in completion order
    let mut set = JoinSet::new();
    for (i, ms) in [(1, 90u64), (2, 30), (3, 60)] {
        set.spawn(async move { sleep(Duration::from_millis(ms)).await; i });
    }
    let mut order = vec![];
    while let Some(r) = set.join_next().await { order.push(r.unwrap()); }
    println!("joinset completion order {:?}", order);
}
```

✓ Output (the panic message goes to stderr from the worker thread; the rest to stdout):

```text
spawn -> 42
thread 'tokio-rt-worker' panicked at src/bin/t01.rs:12:44:
task boom
is_panic=true is_cancelled=false
after abort: is_cancelled=true
join! 1 2 concurrent=true
select -> timer
timeout -> true
timeout ok -> Ok(9)
joinset completion order [2, 3, 1]
```

What each part teaches:

- **`tokio::spawn(fut)`** starts a task and returns a `JoinHandle<T>`. The future must be `Send + 'static` on the multi-thread runtime (Lesson 30 §5: same reasons; ✓ the error for a `std::sync::MutexGuard` held across `.await` is the one from Lesson 30). The task starts running **immediately** in the background, even if you never `.await` the handle.
- **A panic inside a task does not take down the runtime or the process.** The worker thread printed the panic message, and `await`ing the handle returned `Err(JoinError)` with `is_panic() == true`. If you never await the handle, the panic is **silently lost** (only the stderr message remains). Always `.await` or otherwise observe the handles of tasks that matter.
- **`abort()`** requests cancellation; the task is dropped at its next suspension point and the handle resolves to a `JoinError` with `is_cancelled() == true`.
- **Dropping a `JoinHandle` detaches the task**; it keeps running (✓ §7: a flag set 50 ms later was still set). Dropping a **`JoinSet`** aborts all its tasks (✓ §7). Choose accordingly.
- **`join!`** runs several futures concurrently *inside the current task* (✓ two 100 ms sleeps finished in under 190 ms). It does not use extra threads. For parallelism, spawn tasks.
- **`select!`** waits for the first of several futures and **drops the rest**. Dropped means cancelled: whatever they were doing stops at the last `.await`. Some operations are *cancel-safe* (nothing is lost if dropped mid-way) and some are not; this is the main hazard of `select!` and Lesson 32's topic.
- **`timeout(d, fut)`** is `select!` between `fut` and a timer: on expiry it returns `Err(Elapsed)` and drops `fut`.
- **`JoinSet`** manages a dynamic group of tasks: `spawn`, `join_next` (completion order ✓ `[2, 3, 1]` for 90/30/60 ms), `abort_all`, `len`.

### 2.1 `!Send` work: `LocalSet`

Tasks that must stay on one thread (they hold an `Rc`, for instance) go on a `LocalSet` with `spawn_local`. ✓ On a current-thread runtime, `local.block_on(&rt, async { ... })` accepted a task holding an `Rc` and returned `6`.

---

## 3. Channels and shared state

```rust
use std::sync::Arc;
use tokio::sync::{broadcast, mpsc, oneshot, watch, Mutex, Semaphore};

#[tokio::main]
async fn main() {
    // bounded mpsc: send awaits when full (backpressure)
    let (tx, mut rx) = mpsc::channel::<u32>(2);
    tx.send(1).await.unwrap();
    tx.send(2).await.unwrap();
    println!("try_send when full: {:?}", tx.try_send(3).map_err(|e| matches!(e, mpsc::error::TrySendError::Full(_))));
    println!("recv {:?}", rx.recv().await);
    tx.try_send(3).unwrap();
    drop(tx);
    println!("drain {:?} {:?} {:?}", rx.recv().await, rx.recv().await, rx.recv().await);

    // oneshot
    let (otx, orx) = oneshot::channel();
    tokio::spawn(async move { otx.send("reply").unwrap(); });
    println!("oneshot {:?}", orx.await);
    let (otx, orx) = oneshot::channel::<u8>();
    drop(otx);
    println!("oneshot dropped sender {:?}", orx.await.is_err());

    // watch: only the latest value, many readers
    let (wtx, mut wrx) = watch::channel(0u32);
    wtx.send(1).unwrap(); wtx.send(2).unwrap();
    wrx.changed().await.unwrap();
    println!("watch latest {}", *wrx.borrow_and_update());

    // broadcast: every receiver gets every message; a slow receiver LOSES old ones
    let (btx, mut b1) = broadcast::channel::<u32>(2);
    let mut b2 = btx.subscribe();
    for i in 0..5 { btx.send(i).unwrap(); }
    println!("b1 first: {:?}", b1.recv().await);
    println!("b1 next: {:?}", b1.recv().await);
    drop(btx);
    println!("b2: {:?}", b2.recv().await);

    // tokio Mutex may be held across an await
    let m = Arc::new(Mutex::new(0));
    let hs: Vec<_> = (0..4).map(|_| { let m = m.clone(); tokio::spawn(async move {
        let mut g = m.lock().await;
        tokio::task::yield_now().await; // await while holding the guard
        *g += 1;
    }) }).collect();
    for h in hs { h.await.unwrap(); }
    println!("mutex {}", *m.lock().await);

    // semaphore
    let sem = Arc::new(Semaphore::new(2));
    let p1 = sem.clone().acquire_owned().await.unwrap();
    let _p2 = sem.clone().acquire_owned().await.unwrap();
    println!("try_acquire: {}", sem.try_acquire().is_ok());
    drop(p1);
    println!("after drop: {}", sem.try_acquire().is_ok());
}
```

✓ Output:

```text
try_send when full: Err(true)
recv Some(1)
drain Some(2) Some(3) None
oneshot Ok("reply")
oneshot dropped sender true
watch latest 2
b1 first: Err(Lagged(3))
b1 next: Ok(3)
b2: Err(Lagged(3))
mutex 4
try_acquire: false
after drop: true
```

| Channel | Shape | Use it for | Behavior you saw ✓ |
|---|---|---|---|
| `mpsc::channel(n)` | many senders → one receiver, **bounded** | work queues, commands to an actor | `send().await` waits when full, `try_send` fails with `Full`; after all senders drop, remaining items drain and then `recv` returns `None` |
| `mpsc::unbounded_channel()` | same, unbounded | rarely; only when the producer is bounded by other means | no backpressure: memory grows without limit if the consumer is slower (from memory of the docs; the same reasoning as Lesson 27's `mpsc`) |
| `oneshot` | one value, one sender, one receiver | request/response replies | the receiver resolves to `Err` if the sender is dropped without sending |
| `watch` | **latest value** with change notification, many readers | configuration, shutdown flags, "current state" | two sends before `changed()` collapse: the reader saw only `2` |
| `broadcast(n)` | every receiver sees every message | events/fan-out | capacity 2, 5 messages sent: the receiver was told it **lagged by 3**, then continued from the oldest retained message (`3`) |

Design guidance from the table: choose **bounded** channels by default, and decide on purpose what happens when they fill (the sender waits, drops the message, or fails). `broadcast` and `watch` are lossy by design: a consumer that must see every message needs `mpsc`.

### 3.1 Which mutex?

Lesson 30 showed that a `std::sync::MutexGuard` held across an `.await` is `!Send` (✓ compile error again here: "future cannot be sent between threads safely"). Tokio's own advice (from memory of the docs) is:

- Use **`std::sync::Mutex`** (or `parking_lot`) when the critical section is short and contains **no `.await`**. This is the common case and it is faster. ✓ A pattern that worked: lock in an inner block, push, drop the guard, *then* `.await`.
- Use **`tokio::sync::Mutex`** when you must hold the lock **across an `.await`** (✓ four tasks each held the guard across `yield_now().await` and the count was `4`; `try_lock` on a held lock failed, so it excludes properly).
- If you find yourself holding a lock across awaits often, consider an **actor**: one task owns the state and other tasks send it messages through an `mpsc` plus `oneshot` reply channels. No lock at all.

`Semaphore` limits concurrency (`acquire_owned` gives a permit that can move into a task; dropping the permit releases the slot ✓).

---

## 4. Blocking work

```rust
use std::time::{Duration, Instant};
// blocking inside async on a current_thread runtime: nothing else runs
#[tokio::main(flavor = "current_thread")]
async fn main() {
    let ticker = tokio::spawn(async move {
        let mut gaps = vec![];
        let mut last = Instant::now();
        for _ in 0..5 {
            tokio::time::sleep(Duration::from_millis(20)).await;
            gaps.push(last.elapsed().as_millis());
            last = Instant::now();
        }
        gaps
    });
    tokio::time::sleep(Duration::from_millis(30)).await;
    std::thread::sleep(Duration::from_millis(300)); // BLOCKS the only worker thread
    let gaps = ticker.await.unwrap();
    println!("max tick gap >= 250ms: {}", gaps.iter().max().unwrap() >= &250);

    // the same ticker, with the slow work moved to the blocking pool
    let ticker = tokio::spawn(async move {
        let mut gaps = vec![];
        let mut last = Instant::now();
        for _ in 0..5 {
            tokio::time::sleep(Duration::from_millis(20)).await;
            gaps.push(last.elapsed().as_millis());
            last = Instant::now();
        }
        gaps
    });
    let r = tokio::task::spawn_blocking(|| { std::thread::sleep(Duration::from_millis(300)); 42 }).await.unwrap();
    let gaps = ticker.await.unwrap();
    println!("blocking result {r}, max tick gap < 150ms: {}", gaps.iter().max().unwrap() < &150);
}
```

✓ Output:

```text
max tick gap >= 250ms: true
blocking result 42, max tick gap < 150ms: true
```

A ticker task that should wake every 20 ms was starved for at least 250 ms when the only worker thread was blocked by `std::thread::sleep` (the same mechanism as Lesson 30 §8, now with a real scheduler). With the slow work on `spawn_blocking`, the ticker kept ticking while the blocking pool thread slept. (The two numbers are thresholds I chose generously, not benchmarks.)

Rules:

- Anything that can block for more than "a few microseconds" (synchronous file or network libraries, `thread::sleep`, heavy hashing/compression/serialization, a CPU loop) belongs on **`spawn_blocking`** (a separate thread pool) or a dedicated thread, with the result brought back via the `JoinHandle` or a channel.
- `spawn_blocking` closures **cannot be cancelled**: `abort()` does not stop a closure already running (from memory of the docs); it is also not meant for infinite loops.
- There is also `block_in_place` for multi-thread runtimes (from memory; I did not run it): it tells the runtime "this worker is about to block", moving other tasks away. Prefer `spawn_blocking` until you know you need it.
- Long CPU loops without any `.await` also starve other tasks. Break them up with `yield_now().await`, or move them off the async workers.

---

## 5. I/O: an echo server in a few lines

```rust
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::{TcpListener, TcpStream};
#[tokio::main]
async fn main() {
    let listener = TcpListener::bind("127.0.0.1:0").await.unwrap();
    let addr = listener.local_addr().unwrap();
    let server = tokio::spawn(async move {
        loop {
            let (sock, _) = listener.accept().await.unwrap();
            tokio::spawn(async move {
                let (r, mut w) = sock.into_split();
                let mut lines = BufReader::new(r).lines();
                while let Some(line) = lines.next_line().await.unwrap() {
                    w.write_all(format!("echo:{line}\n").as_bytes()).await.unwrap();
                }
            });
        }
    });
    let mut c = TcpStream::connect(addr).await.unwrap();
    c.write_all(b"hello\nworld\n").await.unwrap();
    let (r, mut w) = c.split();
    let mut lines = BufReader::new(r).lines();
    println!("{:?}", lines.next_line().await.unwrap());
    println!("{:?}", lines.next_line().await.unwrap());
    w.shutdown().await.unwrap();
    server.abort();
}
```

✓ Output: `Some("echo:hello")` then `Some("echo:world")`. Notes:

- Port `0` asks the OS for a free port; `local_addr()` tells you which. (This binds to loopback only, which worked inside the sandbox.)
- Each connection becomes its own task. That is the standard model: **one task per connection**, thousands are cheap.
- `into_split()` gives owned halves that can move into different tasks; `split()` borrows halves from the stream.
- Methods like `read`, `write_all`, `read_exact`, `lines().next_line()` come from the extension traits `AsyncReadExt`, `AsyncWriteExt`, `AsyncBufReadExt` which you must import (a missing import is a "method not found" error, as for any trait).
- **The `unwrap()`s are for brevity.** A real server must not panic on a failed `accept` (descriptor exhaustion is a normal overload condition), must bound the number of connections, and must set read timeouts. The practical crate does the first two.

---

## 6. Time

- `sleep(d)`, `timeout(d, fut)`, `interval(d)` (a repeating ticker; what happens when a tick is missed is configurable via a "missed tick behavior" setting; from memory, not run) and `Instant` come from `tokio::time`. They are *the* way to wait in async code. Never `std::thread::sleep` (§4).
- **Time can be paused in tests** (feature `test-util`): ✓ with `#[tokio::main(flavor = "current_thread", start_paused = true)]` (the same attribute works on `#[tokio::test]`), a `sleep(3600s)` returned immediately in real time, and even a virtual *day* took under one second of wall-clock time:

```rust
use std::time::Duration;
use tokio::time::{sleep, Instant};
#[tokio::main(flavor = "current_thread", start_paused = true)]
async fn main() {
    let t = Instant::now();
    sleep(Duration::from_secs(3600)).await;
    println!("virtual elapsed >= 3600s: {}", t.elapsed() >= Duration::from_secs(3600));
    let real = std::time::Instant::now();
    sleep(Duration::from_secs(86_400)).await;
    println!("real time for a virtual day < 1s: {}", real.elapsed() < Duration::from_secs(1));
}
```

✓ Both lines print `true`. With the clock paused, when every task is waiting on a timer the runtime jumps the clock to the next deadline ("auto-advance"; from memory of the docs). The practical crate's retry tests use this to assert **exact** virtual elapsed times (e.g. 460 ms for four timed-out attempts plus backoffs) with no flakiness. It only controls Tokio's clock: real sockets and `std::time::Instant` are unaffected, so tests that mix real I/O with `start_paused` need care.

---

## 7. Graceful shutdown and cancellation

A server that is stopped with a kill signal loses in-flight work. The standard pattern:

1. One **shutdown signal** that every task can wait on: a `tokio_util::sync::CancellationToken` (cheap to clone; `child_token()` for hierarchies), or a `watch<bool>`/`broadcast`.
2. Each long-running task has a loop `select! { _ = token.cancelled() => break, work = next_work() => ... }`.
3. The owner cancels the token, then **awaits** the tasks (`JoinSet::join_next` until empty), optionally with an overall `timeout`, so everything finishes or is forcibly dropped.

```rust,ignore
let token = CancellationToken::new();
let mut set = JoinSet::new();
for i in 0..3 {
    let t = token.clone();
    set.spawn(async move {
        let mut ticks = 0u32;
        loop {
            tokio::select! {
                _ = t.cancelled() => return (i, ticks, "stopped"),
                _ = tokio::time::sleep(Duration::from_millis(10)) => ticks += 1,
            }
        }
    });
}
tokio::time::sleep(Duration::from_millis(55)).await;
token.cancel();
while let Some(r) = set.join_next().await { /* collect */ }
```

✓ Run as part of a larger program (with the `JoinSet` results collected and sorted): `[(0, true, "stopped"), (1, true, "stopped"), (2, true, "stopped")]`, where `true` means at least 3 ticks happened before the cancel.

Behaviors that decide whether shutdown is clean ✓:

| Action | Result ✓ |
|---|---|
| Drop a `JoinSet` with a task that would set a flag 100 ms later | flag **not** set: tasks aborted |
| Drop a `JoinHandle` of a task that sets a flag 50 ms later | flag **set**: task detached and kept running |
| `handle.abort()` | handle resolves to `JoinError` with `is_cancelled()` |

**Cancellation is the central hazard of async Rust.** A task can be dropped at any `.await`. Code that updates two things with an `.await` between them can be dropped half-way; code that holds a lock or a reservation relies on `Drop` to release it. Operations differ in *cancel safety*: Tokio documents which of its methods are cancel-safe (`TcpListener::accept` and `AsyncReadExt::read` are, `write_all` and `read_exact` are not, from memory of the docs, not re-verified). The practical crate races only cancel-safe calls against the shutdown token and awaits `write_all` to completion. Lesson 32 treats this in depth.

---

## 8. Putting it together: `tokiolab`

The practical crate combines the lesson: a **TaskPool** (semaphore-limited spawning with backpressure), **retry** with per-attempt timeout and capped exponential backoff, and a **TCP echo server** with a connection limit and graceful shutdown. Each is a pattern you will write again in real services (worker pools, resilient RPC clients, accept loops).

---

## 9. Testing async code

- `#[tokio::test]` runs the test body on a current-thread runtime (from memory; ✓ the crate's tests use it). `#[tokio::test(start_paused = true)]` adds the paused clock.
- **Assert on virtual time, not on sleeps.** ✓ `assert_eq!(start.elapsed(), Duration::from_millis(30))` after two backoffs is deterministic under the paused clock, and a real-time test of the same behavior would be slow and flaky.
- **Watchdog for hangs.** Wrap risky waits in `timeout`; for the paused clock this also means a deadlocked test fails after virtual time passes instead of hanging CI (but a task stuck on *real* I/O will not advance virtual time).
- Test shutdown explicitly: cancel, then assert the server task finishes within a timeout and that clients see EOF (✓ crate test).
- Test backpressure: the producer's `spawn`/`send` must **not** complete while the consumer is stalled (✓ crate test `spawn_waits_when_the_pool_is_full` uses `timeout(50 ms, pool.spawn(..))` and expects `Err`).

---

## 10. Common mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Blocking call in async code (`thread::sleep`, sync I/O, heavy CPU) | Starves other tasks (✓ §4) | async APIs, `spawn_blocking` |
| `std::sync::MutexGuard` held across `.await` | Not `Send` (✓ error) and possible executor deadlock (✓ debugging challenge) | scope the guard; `tokio::sync::Mutex`; actor |
| Not awaiting `JoinHandle`s | Panics and errors vanish silently (✓ §2) | await them or use `JoinSet` |
| Unbounded `spawn` per request/message | Memory exhaustion under load | `Semaphore`/`TaskPool`, bounded channels |
| Unbounded channel between fast producer and slow consumer | Memory grows without limit | bounded `mpsc` with backpressure |
| `select!` over non-cancel-safe futures | Lost data / half-done work | use cancel-safe futures; keep state outside the branch; Lesson 32 |
| `.await` between two related updates | Dropped in the middle | make the update atomic, or use a guard with `Drop` |
| No timeouts on network calls | A stuck peer holds resources forever | `timeout` everywhere |
| Nested `block_on` / second runtime inside async | Panic (✓) | `.await` instead |
| `tokio::spawn` outside a runtime | Panic (✓ "no reactor running") | create a runtime or use `#[tokio::main]` |
| Using `broadcast`/`watch` where every message matters | Silent loss (✓ `Lagged`) | `mpsc` |
| `unwrap()` on `accept`/`read` in a server | One bad connection kills the accept loop | handle errors per connection |
| Forgetting that dropping the handle detaches (but dropping a `JoinSet` aborts) | Orphan tasks or surprise cancellation | choose deliberately (✓ §7 table) |
| Enabling `features = ["full"]` in a library | Needless compile time and dependencies | enable only needed features |

---

## 11. Performance, reliability, security notes

- **Cost model**: spawning a task is an allocation plus a queue operation (the Lesson 30 design, optimized); I give no timing numbers here (a 2-core sandbox would not transfer; Lesson 41 teaches measuring). The performance mistakes that matter are structural: blocking the workers, unbounded queues, huge futures held across `.await`, and lock contention.
- **Multi-thread vs current-thread**: current-thread is simpler and allows `!Send` tasks; multi-thread uses all cores but demands `Send + 'static`. Pick multi-thread for servers, current-thread for tests, small tools and `!Send` code (from memory, consistent with ✓ behavior above).
- **Observability**: a task that is stuck is invisible to a thread dump (the threads are idle in the scheduler). Use `tracing` spans (Lesson 43) and, from memory, the `tokio-console` tool for task-level diagnostics (needs Tokio's unstable tracing support; not verified here).
- **Reliability**: put timeouts on every external wait; bound concurrency and queues; define shutdown order (stop accepting, drain, close dependencies); treat cancellation as normal control flow; make retries idempotent-safe (retrying a non-idempotent request can double-apply it) and add jitter in real clients.
- **Security**: connection limits and read timeouts blunt slow-client and flood attacks (the crate's `max_conns` stops *accepting* beyond the limit; a real server would also time out idle connections, which the crate does not). Bound message sizes and buffers before allocating. Never trust a peer to close a connection.
- **Blockchain-adjacent note**: RPC servers, indexers, relayers and P2P layers (Lessons 50–52, 58) are Tokio services. Keep **consensus-critical or deterministic computation off timers and off scheduling effects**: results must not depend on task interleaving or on which of two `select!` branches won. Use bounded queues between the network layer and the state machine, and do CPU-heavy verification (signatures, proofs) on `spawn_blocking` or dedicated threads.

---

## 12. Exercises

Answers are collapsed. Try each before opening it.

1. **(Warm-up)** What does `#[tokio::main]` do, and what does `flavor = "current_thread"` change?
2. **(Warm-up)** Why does `tokio::spawn` require the future to be `'static`?
3. **(Easy)** Three tasks sleep 90, 30 and 60 ms and are collected with a `JoinSet`. In what order do the results arrive, and why is that different from `join!`?
4. **(Medium)** A `broadcast::channel(2)` receives 5 messages (`0..5`) before a receiver reads. What does the receiver see, in order?
5. **(Medium)** Write `par_map(items, f)` that runs `f(item)` (an async function) for all items concurrently with a `JoinSet` and returns results **in input order**.
6. **(Medium-hard)** This handler runs on the multi-thread runtime and freezes the server under load. Why? Fix it:
   ```rust,ignore
   async fn hash_password(pw: String) -> String {
       slow_hash(&pw)            // 200 ms of pure CPU
   }
   ```
7. **(Hard)** Why can a bounded `mpsc` channel deadlock two tasks that send to each other, and how do you avoid it?
8. **(Hard)** Design the shutdown of a service with a TCP accept loop, a pool of worker tasks fed by a channel, and a metrics task. In what order do you stop things and why?

<details markdown="1">
<summary>Answers</summary>

1. It builds a Tokio runtime and runs the body of `async fn main` to completion on it with `block_on` (from memory). `flavor = "current_thread"` makes it a single-threaded scheduler (✓ one worker; the default multi-thread had 2 workers on this machine).
2. A spawned task runs on a runtime that outlives the function that spawned it and may run on another thread, so it cannot borrow from the caller's stack (the same reason as `thread::spawn`, Lesson 27). Move owned data in (`async move`, `Arc`, clones). For scoped parallelism you need other tools.
3. `[2, 3, 1]` (30, 60, 90 ms): `JoinSet::join_next` yields in **completion** order (✓). `tokio::join!` would return its results in the **argument order** once all are done.
4. `Err(Lagged(3))` first (it missed messages 0, 1, 2 because capacity 2 only retains 3 and 4), then `Ok(3)`, then `Ok(4)` (✓ the first two lines were run: `Err(Lagged(3))`, `Ok(3)`). The `4` is by the documented behavior, not printed in my run.
5. ```rust
   use std::future::Future;
   use tokio::task::JoinSet;
   async fn par_map<T, R, F, Fut>(items: Vec<T>, f: F) -> Vec<R>
   where
       T: Send + 'static,
       R: Send + 'static,
       F: Fn(T) -> Fut,
       Fut: Future<Output = R> + Send + 'static,
   {
       let mut set = JoinSet::new();
       for (i, item) in items.into_iter().enumerate() {
           let fut = f(item);
           set.spawn(async move { (i, fut.await) });
       }
       let mut out: Vec<(usize, R)> = Vec::new();
       while let Some(r) = set.join_next().await {
           out.push(r.expect("task panicked"));
       }
       out.sort_by_key(|(i, _)| *i);
       out.into_iter().map(|(_, r)| r).collect()
   }
   #[tokio::main]
   async fn main() {
       let r = par_map(vec![3u64, 1, 2], |n| async move {
           tokio::time::sleep(std::time::Duration::from_millis(n * 20)).await;
           n * 10
       }).await;
       println!("{r:?}");
   }
   ```
   ✓ Prints `[30, 10, 20]` (input order, although completion order was 1, 2, 3). Unbounded concurrency is a hazard for large inputs: add a `Semaphore` (as `TaskPool` does).
6. A CPU-bound call with no `.await` occupies a worker thread for 200 ms; with as many concurrent requests as workers, every worker is busy and the server cannot even accept connections (the Lesson 30 §8 / ✓ §4 effect). Fix: `tokio::task::spawn_blocking(move || slow_hash(&pw)).await.expect("hash task")`. For sustained CPU load, use a dedicated thread pool and bound it.
7. Two tasks each `send().await` into the other's full bounded channel and wait for the other to `recv`: a circular wait (Lesson 29 conditions again, now with channel capacity as the resource). Avoid: never block a task on sending into a channel that the receiver of *your* inbox feeds; use `try_send` with a drop/overflow policy, make one direction unbounded-but-otherwise-limited, or restructure as a pipeline (acyclic).
8. Stop **inputs first, outputs last**: (1) cancel the accept loop (no new connections); (2) let connection tasks finish or cancel them via child tokens; (3) close the work channel's senders so workers drain remaining jobs and exit when `recv` returns `None`; (4) `await` the workers; (5) stop the metrics task last so it records the shutdown; (6) apply an overall `timeout` after which remaining tasks are aborted. The crate's `serve_echo` implements steps 1–2.

</details>

---

## 13. Debugging challenge

This program is meant to print `done`. It **hangs forever** (✓ I ran it with a watchdog: no output after 3 seconds). There is only one thread; nothing is spawned.

```rust
use std::time::Duration;

#[tokio::main(flavor = "current_thread")]
async fn main() {
    let m = std::sync::Mutex::new(0);
    tokio::join!(
        async {
            let g = m.lock().unwrap();
            tokio::time::sleep(Duration::from_millis(10)).await;
            drop(g);
        },
        async {
            let _g = m.lock().unwrap();
        }
    );
    println!("done");
}
```

Explain the hang, and give two different fixes.

<details markdown="1">
<summary>Solution</summary>

`join!` polls the first future: it takes the `std` mutex, then awaits `sleep`, which returns `Pending`, so the first future is suspended **while still holding the lock**. `join!` now polls the second future, which calls `m.lock()`: a *blocking* call. The only thread is now blocked waiting for a lock whose owner is a suspended future that can only be resumed by that same thread. This is a deadlock with a single thread (Lesson 29's self-deadlock, but across two futures). Note that nothing was spawned, so the `Send` check that rejects `MutexGuard` across `.await` in `tokio::spawn` never ran: `join!` does not require `Send`, which is why the compiler accepted it.

Fix 1: do not hold the `std` guard across the `.await`:

```rust
use std::time::Duration;
#[tokio::main(flavor = "current_thread")]
async fn main() {
    let m = std::sync::Mutex::new(0);
    tokio::join!(
        async {
            { let mut g = m.lock().unwrap(); *g += 1; } // guard dropped here
            tokio::time::sleep(Duration::from_millis(10)).await;
        },
        async {
            let _g = m.lock().unwrap();
        }
    );
    println!("done");
}
```

Fix 2: use `tokio::sync::Mutex` and `.lock().await`, whose waiting is asynchronous, so the thread is free to run the first future to completion:

```rust
use std::time::Duration;
#[tokio::main(flavor = "current_thread")]
async fn main() {
    let m = tokio::sync::Mutex::new(0);
    tokio::join!(
        async {
            let g = m.lock().await;
            tokio::time::sleep(Duration::from_millis(10)).await;
            drop(g);
        },
        async {
            let _g = m.lock().await;
        }
    );
    println!("done");
}
```

✓ Both versions printed `done`. Prefer fix 1 when the critical section does not need to span an `.await`; use fix 2 (or an actor) when it does.

</details>

---

## 14. Practical challenge: `tokiolab`

Build a library crate with:

1. `TaskPool<T>`: `new(max)`, `async spawn(fut)` that **waits** for a free slot, `available()`, `capacity()`, `join_all()` returning each task's `Result<T, JoinError>`. The slot must be released however the task ends (including panic). Dropping the pool aborts running tasks.
2. `retry(policy, op)`: at most `attempts` tries, each limited by `per_attempt` (a timed-out attempt is dropped), exponential backoff `base * 2^(n-1)` capped at `max_backoff` with **no overflow** for any attempt number. The error says how many attempts were made and whether the last failure was an error or a timeout.
3. `serve_echo(listener, max_conns, token)`: echo server; at most `max_conns` connections at once (stop calling `accept` beyond that); on `token.cancel()` stop accepting, close the connections, wait for them, and return the number of accepted connections. Only race cancel-safe calls against the token.

Tests: use the paused clock for time-dependent logic and watchdog timeouts for anything that could hang.

<details markdown="1">
<summary>Solution (spoilers: full source)</summary>

`Cargo.toml`:

```toml
[package]
name = "tokiolab"
version = "0.1.0"
edition = "2024"

[dependencies]
tokio = { version = "1.54.1", features = ["rt", "sync", "time", "net", "io-util", "macros"] }
tokio-util = { version = "0.7.20", default-features = false }

[dev-dependencies]
tokio = { version = "1.54.1", features = ["rt", "rt-multi-thread", "sync", "time", "net", "io-util", "macros", "test-util"] }
```

`src/lib.rs`:

```rust
//! `tokiolab`: three small, production-shaped async building blocks.
//!
//! * [`TaskPool`]: spawn tasks with a hard concurrency limit and backpressure.
//! * [`retry`]: retries with a per-attempt timeout and capped exponential backoff.
//! * [`serve_echo`]: a TCP echo server with a connection limit and graceful shutdown.

use std::future::Future;
use std::io;
use std::sync::Arc;
use std::time::Duration;

use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::{TcpListener, TcpStream};
use tokio::sync::Semaphore;
use tokio::task::{JoinError, JoinSet};
use tokio::time::{sleep, timeout};
use tokio_util::sync::CancellationToken;

// ---------------------------------------------------------------- TaskPool

/// Runs tasks with at most `max` of them alive at once.
///
/// [`TaskPool::spawn`] **waits** for a free slot, so a producer that submits
/// faster than the tasks finish is slowed down instead of piling up an
/// unbounded number of tasks (backpressure). Dropping the pool **aborts** every
/// task that is still running; use [`TaskPool::join_all`] to wait for them.
///
/// ```
/// # #[tokio::main(flavor = "current_thread")]
/// # async fn main() {
/// let mut pool = tokiolab::TaskPool::new(2);
/// for i in 0..5u32 {
///     pool.spawn(async move { i * 10 }).await;
/// }
/// let mut out: Vec<u32> = pool.join_all().await.into_iter().map(Result::unwrap).collect();
/// out.sort();
/// assert_eq!(out, vec![0, 10, 20, 30, 40]);
/// # }
/// ```
pub struct TaskPool<T> {
    slots: Arc<Semaphore>,
    tasks: JoinSet<T>,
    max: usize,
}

impl<T: Send + 'static> TaskPool<T> {
    /// # Panics
    /// If `max` is zero: no task could ever start.
    pub fn new(max: usize) -> Self {
        assert!(max > 0, "max must be at least 1");
        Self {
            slots: Arc::new(Semaphore::new(max)),
            tasks: JoinSet::new(),
            max,
        }
    }

    /// Waits for a free slot, then spawns `fut` onto the current runtime.
    /// The slot is released when the task ends, however it ends (finished,
    /// panicked or aborted), because the permit is dropped with the task.
    pub async fn spawn<F>(&mut self, fut: F)
    where
        F: Future<Output = T> + Send + 'static,
    {
        let permit = Arc::clone(&self.slots)
            .acquire_owned()
            .await
            .expect("the pool never closes its semaphore");
        self.tasks.spawn(async move {
            let _permit = permit;
            fut.await
        });
    }

    /// Slots currently free.
    pub fn available(&self) -> usize {
        self.slots.available_permits()
    }

    /// The configured limit.
    pub fn capacity(&self) -> usize {
        self.max
    }

    /// Waits for every task and returns their results in completion order.
    /// A task that panicked or was cancelled is an `Err(JoinError)`.
    pub async fn join_all(mut self) -> Vec<Result<T, JoinError>> {
        let mut out = Vec::new();
        while let Some(r) = self.tasks.join_next().await {
            out.push(r);
        }
        out
    }
}

// ------------------------------------------------------------------- retry

/// How [`retry`] behaves.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct RetryPolicy {
    /// Total tries, at least 1.
    pub attempts: u32,
    /// Each try is cancelled (dropped) after this long.
    pub per_attempt: Duration,
    /// Delay before the second try; doubles each time.
    pub base_backoff: Duration,
    /// The delay never exceeds this.
    pub max_backoff: Duration,
}

/// Why the last attempt failed.
#[derive(Debug, PartialEq, Eq)]
pub enum LastFailure<E> {
    Error(E),
    TimedOut,
}

/// All attempts failed.
#[derive(Debug, PartialEq, Eq)]
pub struct RetryError<E> {
    pub attempts: u32,
    pub last: LastFailure<E>,
}

/// Delay before the attempt that follows failed attempt number `failed`
/// (1-based): `base * 2^(failed-1)`, capped at `max_backoff`, without overflow.
pub fn backoff_for(policy: &RetryPolicy, failed: u32) -> Duration {
    let shift = failed.saturating_sub(1).min(30);
    policy
        .base_backoff
        .checked_mul(1u32 << shift)
        .unwrap_or(policy.max_backoff)
        .min(policy.max_backoff)
}

/// Calls `op(attempt_number)` until it succeeds or the policy is exhausted.
/// An attempt that exceeds `per_attempt` is **dropped**, which cancels it.
///
/// There is no jitter here; real clients should add some so that many
/// clients do not retry in lockstep (that needs a random-number source).
///
/// # Panics
/// If `policy.attempts` is zero.
pub async fn retry<T, E, F, Fut>(policy: RetryPolicy, mut op: F) -> Result<T, RetryError<E>>
where
    F: FnMut(u32) -> Fut,
    Fut: Future<Output = Result<T, E>>,
{
    assert!(policy.attempts >= 1, "attempts must be at least 1");
    let mut n = 0;
    loop {
        n += 1;
        let failure = match timeout(policy.per_attempt, op(n)).await {
            Ok(Ok(v)) => return Ok(v),
            Ok(Err(e)) => LastFailure::Error(e),
            Err(_elapsed) => LastFailure::TimedOut,
        };
        if n >= policy.attempts {
            return Err(RetryError {
                attempts: n,
                last: failure,
            });
        }
        sleep(backoff_for(&policy, n)).await;
    }
}

// ---------------------------------------------------------------- echo server

/// Serves an echo protocol until `shutdown` is cancelled, then waits for all
/// connections to finish. At most `max_conns` connections are served at once:
/// beyond that the server stops calling `accept` (new clients wait in the
/// kernel's backlog) instead of spawning without limit. Returns how many
/// connections were accepted.
///
/// Every `await` that races against `shutdown` is cancel-safe (`accept` and
/// `read`; from the Tokio docs, not re-verified here). The write is awaited to
/// completion, so a connection never sends half an echo on shutdown.
pub async fn serve_echo(
    listener: TcpListener,
    max_conns: usize,
    shutdown: CancellationToken,
) -> io::Result<usize> {
    assert!(max_conns > 0, "max_conns must be at least 1");
    let slots = Arc::new(Semaphore::new(max_conns));
    let mut conns = JoinSet::new();
    let mut accepted = 0;
    loop {
        // Reap finished connections so the set does not grow forever.
        while conns.try_join_next().is_some() {}
        let permit = tokio::select! {
            _ = shutdown.cancelled() => break,
            p = Arc::clone(&slots).acquire_owned() => p.expect("never closed"),
        };
        let (sock, _peer) = tokio::select! {
            _ = shutdown.cancelled() => break,
            r = listener.accept() => r?,
        };
        accepted += 1;
        let token = shutdown.child_token();
        conns.spawn(async move {
            let _permit = permit;
            // An I/O error on one connection ends that connection only.
            let _ = echo(sock, token).await;
        });
    }
    while conns.join_next().await.is_some() {}
    Ok(accepted)
}

async fn echo(mut sock: TcpStream, stop: CancellationToken) -> io::Result<()> {
    let mut buf = [0u8; 1024];
    loop {
        let n = tokio::select! {
            _ = stop.cancelled() => break,
            r = sock.read(&mut buf) => r?,
        };
        if n == 0 {
            break; // peer closed
        }
        sock.write_all(&buf[..n]).await?;
    }
    sock.shutdown().await
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::sync::atomic::{AtomicBool, AtomicUsize, Ordering};
    use tokio::sync::oneshot;
    use tokio::time::Instant;

    #[tokio::test(start_paused = true)]
    async fn pool_never_exceeds_its_limit_and_finishes_everything() {
        let now = Arc::new(AtomicUsize::new(0));
        let peak = Arc::new(AtomicUsize::new(0));
        let mut pool = TaskPool::new(3);
        for i in 0..10u32 {
            let (now, peak) = (Arc::clone(&now), Arc::clone(&peak));
            pool.spawn(async move {
                let n = now.fetch_add(1, Ordering::SeqCst) + 1;
                peak.fetch_max(n, Ordering::SeqCst);
                sleep(Duration::from_millis(10)).await;
                now.fetch_sub(1, Ordering::SeqCst);
                i
            })
            .await;
        }
        let results = pool.join_all().await;
        assert_eq!(results.len(), 10);
        assert!(peak.load(Ordering::SeqCst) <= 3);
        assert_eq!(
            peak.load(Ordering::SeqCst),
            3,
            "the limit should be reached"
        );
    }

    #[tokio::test(start_paused = true)]
    async fn spawn_waits_when_the_pool_is_full() {
        let mut pool = TaskPool::new(1);
        let (release, wait) = oneshot::channel::<()>();
        pool.spawn(async move {
            let _ = wait.await;
        })
        .await;
        assert_eq!(pool.available(), 0);
        // The second spawn cannot get a slot, so it is still pending after 50 ms.
        let blocked = timeout(Duration::from_millis(50), pool.spawn(async {})).await;
        assert!(blocked.is_err(), "spawn should wait for a slot");
        release.send(()).unwrap();
        pool.spawn(async {}).await; // now it proceeds
        assert_eq!(pool.join_all().await.len(), 2);
    }

    #[tokio::test(start_paused = true)]
    async fn a_panicking_task_frees_its_slot_and_reports_the_panic() {
        let mut pool: TaskPool<u8> = TaskPool::new(1);
        pool.spawn(async { panic!("task failure") }).await;
        pool.spawn(async { 7 }).await; // would hang forever if the slot leaked
        let results = pool.join_all().await;
        assert_eq!(results.iter().filter(|r| matches!(r, Ok(7))).count(), 1);
        assert_eq!(
            results
                .iter()
                .filter(|r| r.as_ref().is_err_and(JoinError::is_panic))
                .count(),
            1
        );
    }

    #[tokio::test(start_paused = true)]
    async fn dropping_the_pool_aborts_running_tasks() {
        let ran = Arc::new(AtomicBool::new(false));
        let r2 = Arc::clone(&ran);
        {
            let mut pool = TaskPool::new(1);
            pool.spawn(async move {
                sleep(Duration::from_millis(100)).await;
                r2.store(true, Ordering::SeqCst);
            })
            .await;
        }
        sleep(Duration::from_millis(500)).await;
        assert!(!ran.load(Ordering::SeqCst));
    }

    #[test]
    #[should_panic(expected = "max must be at least 1")]
    fn zero_capacity_panics() {
        let _ = TaskPool::<()>::new(0);
    }

    fn policy() -> RetryPolicy {
        RetryPolicy {
            attempts: 4,
            per_attempt: Duration::from_millis(100),
            base_backoff: Duration::from_millis(10),
            max_backoff: Duration::from_millis(30),
        }
    }

    #[test]
    fn backoff_doubles_and_is_capped_without_overflow() {
        let p = policy();
        let v: Vec<u128> = (1..=5).map(|n| backoff_for(&p, n).as_millis()).collect();
        assert_eq!(v, vec![10, 20, 30, 30, 30]);
        assert_eq!(backoff_for(&p, u32::MAX), p.max_backoff);
        let big = RetryPolicy {
            base_backoff: Duration::MAX,
            max_backoff: Duration::from_secs(5),
            ..p
        };
        assert_eq!(backoff_for(&big, 3), Duration::from_secs(5));
    }

    #[tokio::test(start_paused = true)]
    async fn retry_succeeds_on_a_later_attempt_after_the_backoffs() {
        let start = Instant::now();
        let r: Result<&str, RetryError<&str>> = retry(policy(), |n| async move {
            if n < 3 { Err("not yet") } else { Ok("done") }
        })
        .await;
        assert_eq!(r, Ok("done"));
        // two failures -> sleeps of 10 ms and 20 ms of (virtual) time
        assert_eq!(start.elapsed(), Duration::from_millis(30));
    }

    #[tokio::test(start_paused = true)]
    async fn retry_reports_the_last_error_when_exhausted() {
        let calls = Arc::new(AtomicUsize::new(0));
        let c2 = Arc::clone(&calls);
        let r: Result<(), _> = retry(policy(), move |n| {
            c2.fetch_add(1, Ordering::SeqCst);
            async move { Err(n) }
        })
        .await;
        assert_eq!(
            r,
            Err(RetryError {
                attempts: 4,
                last: LastFailure::Error(4)
            })
        );
        assert_eq!(calls.load(Ordering::SeqCst), 4);
    }

    #[tokio::test(start_paused = true)]
    async fn retry_cancels_attempts_that_time_out() {
        let finished = Arc::new(AtomicUsize::new(0));
        let f2 = Arc::clone(&finished);
        let start = Instant::now();
        let r: Result<(), RetryError<()>> = retry(policy(), move |_| {
            let f2 = Arc::clone(&f2);
            async move {
                sleep(Duration::from_secs(10)).await; // far longer than per_attempt
                f2.fetch_add(1, Ordering::SeqCst);
                Ok(())
            }
        })
        .await;
        assert_eq!(
            r,
            Err(RetryError {
                attempts: 4,
                last: LastFailure::TimedOut
            })
        );
        assert_eq!(
            finished.load(Ordering::SeqCst),
            0,
            "dropped attempts never finish"
        );
        // 4 x 100 ms attempts + backoffs 10 + 20 + 30 ms
        assert_eq!(start.elapsed(), Duration::from_millis(460));
    }

    #[tokio::test]
    #[should_panic(expected = "attempts must be at least 1")]
    async fn zero_attempts_panics() {
        let p = RetryPolicy {
            attempts: 0,
            ..policy()
        };
        let _ = retry::<(), (), _, _>(p, |_| async { Ok(()) }).await;
    }

    async fn start_server(
        max: usize,
    ) -> (
        std::net::SocketAddr,
        CancellationToken,
        tokio::task::JoinHandle<io::Result<usize>>,
    ) {
        let listener = TcpListener::bind("127.0.0.1:0").await.unwrap();
        let addr = listener.local_addr().unwrap();
        let token = CancellationToken::new();
        let h = tokio::spawn(serve_echo(listener, max, token.clone()));
        (addr, token, h)
    }

    async fn roundtrip(c: &mut TcpStream, msg: &[u8]) -> Vec<u8> {
        c.write_all(msg).await.unwrap();
        let mut got = vec![0u8; msg.len()];
        c.read_exact(&mut got).await.unwrap();
        got
    }

    #[tokio::test]
    async fn echo_serves_clients_and_shuts_down_gracefully() {
        let (addr, token, server) = start_server(8).await;
        let mut a = TcpStream::connect(addr).await.unwrap();
        let mut b = TcpStream::connect(addr).await.unwrap();
        assert_eq!(roundtrip(&mut a, b"hello").await, b"hello");
        assert_eq!(roundtrip(&mut b, b"world!").await, b"world!");
        token.cancel();
        let accepted = timeout(Duration::from_secs(5), server)
            .await
            .expect("server must stop")
            .unwrap()
            .unwrap();
        assert_eq!(accepted, 2);
        // the server shut the connections down: clients now read EOF
        let mut rest = Vec::new();
        timeout(Duration::from_secs(5), a.read_to_end(&mut rest))
            .await
            .unwrap()
            .unwrap();
        assert!(rest.is_empty());
    }

    #[tokio::test]
    async fn echo_limits_concurrent_connections() {
        let (addr, token, server) = start_server(1).await;
        let mut first = TcpStream::connect(addr).await.unwrap();
        assert_eq!(roundtrip(&mut first, b"one").await, b"one");
        // The second client connects (kernel backlog) but is not served yet.
        let mut second = TcpStream::connect(addr).await.unwrap();
        second.write_all(b"two").await.unwrap();
        let mut buf = [0u8; 3];
        let early = timeout(Duration::from_millis(200), second.read_exact(&mut buf)).await;
        assert!(early.is_err(), "second client must wait for a slot");
        drop(first); // frees the slot
        timeout(Duration::from_secs(5), second.read_exact(&mut buf))
            .await
            .expect("served after the first left")
            .unwrap();
        assert_eq!(&buf, b"two");
        token.cancel();
        let _ = timeout(Duration::from_secs(5), server).await.unwrap();
    }
}
```

**Verification ✓.** `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings` and `cargo test --locked` pass: 12 unit tests and 1 doctest. (The crate depends on `tokio` 1.54.1 and `tokio-util` 0.7.20 as resolved in this sandbox's lockfile; your lockfile may resolve newer versions.)

**Mutation checks ✓** (each mutation applied to a copy; unit tests run with a 200-second limit):

| # | Mutation | Result |
|---|---|---|
| M1 | `TaskPool::spawn` drops the permit instead of moving it into the task | **caught**: peak concurrency exceeded 3 and the "pool is full" test failed |
| M2 | `backoff_for` without the `max_backoff` cap | **caught**: backoff test and the exact-elapsed test fail |
| M3 | `retry` without the per-attempt `timeout` | **caught**: the timed-out-attempt test fails (the attempt now runs its full 10 s of virtual time and returns `Ok`) |
| M4 | `serve_echo` effectively no connection limit (`max_conns * 1000` slots) | **caught**: `echo_limits_concurrent_connections` |
| M5 | connection loop ignores the shutdown token | **caught**: server does not stop; "server must stop" after the 5 s timeout (these tests take about 5 s to fail) |
| M7 | `retry` computes but never sleeps the backoff | **caught**: both exact-elapsed tests fail |
| M8 | permit is `mem::forget`-ed after a task finishes (slot leaks) | **caught**, but only by hanging: the run was killed by the 200 s limit (exit 124). A hang is a failure, but a slow one; the pool tests should wrap their waits in `timeout` (I did not change them) |

(There is no M6: I numbered the planned mutations before dropping one.)

**Design notes.**

- Backpressure comes from `Semaphore::acquire_owned().await` *before* spawning, and the owned permit travels **into** the task, so it is released by `Drop` on every path: success, panic (unwinding drops it) or abort. M1 and M8 are exactly the bugs this prevents.
- `JoinSet` ties the tasks' lifetime to the pool: dropping the pool aborts them (✓ test), `join_all` waits for them.
- `retry` is generic over the error and the future. `timeout(per_attempt, op(n))` drops a slow attempt, which cancels it (✓ the counter of finished attempts stays 0).
- `backoff_for` uses `checked_mul` and a shift capped at 30, so absurd attempt numbers and huge `base_backoff` cannot overflow or panic (✓ test with `Duration::MAX`).
- `serve_echo` acquires a connection permit **before** calling `accept`, so at the limit the kernel's backlog absorbs new clients and the process stops accepting; each connection task gets a `child_token()` and keeps its permit until it ends.
- Not handled: transient `accept` errors end the server with `Err` (a production loop would log and back off), no idle/read timeouts on connections, no jitter in `retry`, `TaskPool::spawn` takes `&mut self` (single owner; a shared pool would use a channel), results of finished tasks stay buffered until `join_all`.

</details>

---

## 15. Interview questions

1. What does a Tokio runtime consist of? What is the difference between `current_thread` and `multi_thread`?
2. Why must futures passed to `tokio::spawn` be `Send + 'static`?
3. What happens to a panic inside a spawned task? What if nobody awaits its `JoinHandle`?
4. `tokio::join!` vs `tokio::spawn`: concurrency or parallelism?
5. What does dropping a `JoinHandle` do? A `JoinSet`? A future inside `select!`?
6. When do you use `std::sync::Mutex` and when `tokio::sync::Mutex`?
7. Why is blocking inside an async task a problem, and what are the options?
8. How do you implement graceful shutdown of a Tokio service?

<details markdown="1">
<summary>Answers</summary>

1. A scheduler (executes tasks), an I/O driver (OS readiness events), a timer, and a blocking thread pool. `current_thread` runs all tasks on one thread (allows `!Send` tasks via `LocalSet`, simplest); `multi_thread` runs a pool of workers (✓ 2 workers on this 2-core machine) and needs `Send` futures.
2. The task can run on any worker thread (`Send`) and outlives the spawning function (`'static`, so no borrows of the caller's stack). Same reasoning as `thread::spawn` (Lesson 27) and Lesson 30 §5.
3. It is caught; the worker thread prints the panic message, and `.await` on the handle yields `Err(JoinError)` with `is_panic()` (✓). If nobody awaits the handle, the panic is effectively lost (only the stderr line) and your program continues, so supervise important tasks.
4. `join!` is concurrency inside one task (interleaving on one thread at a time, ✓ two 100 ms sleeps overlapped); `spawn` creates separate tasks that a multi-thread runtime can run in parallel.
5. Dropping a `JoinHandle` **detaches** the task (✓ it kept running). Dropping a `JoinSet` **aborts** its tasks (✓ the flag was never set). A future dropped by `select!` is **cancelled**: it stops at its last `.await`.
6. `std` for short critical sections with no `.await` inside (faster, fine in async code; ✓ pattern shown). `tokio::sync::Mutex` when the lock must be held across an `.await`. Often better: an actor owning the state.
7. It occupies a worker thread, so other tasks on that thread starve (✓ 250 ms tick gap). Options: async versions of the API, `spawn_blocking`, a dedicated thread, or `block_in_place` on multi-thread runtimes.
8. A shared cancellation signal (`CancellationToken`, `watch`), loops that `select!` on it, the owner cancels and then **awaits** all tasks (`JoinSet`), closing input first and draining queues, with an overall timeout and abort as a last resort (§7, ✓ the example and the crate's echo server).

</details>

---

## 16. Key takeaways

- Tokio = scheduler + I/O driver + timer + blocking pool + async sync/IO types; you pick the flavor and the features (✓ 1/2/3 workers by configuration).
- `spawn` runs a task immediately and returns a `JoinHandle`; panics surface only if you await it (✓); dropping a handle detaches, dropping a `JoinSet` aborts (✓); `abort` cancels at the next suspension point.
- `join!` = concurrent in one task, `select!` = race and **drop the losers**, `timeout` = race against a timer; cancellation is normal control flow, so know which operations are cancel-safe (Lesson 32).
- Use **bounded** channels and a `Semaphore` to apply backpressure (✓ `try_send` `Full`, `spawn` waits); `broadcast` and `watch` are lossy by design (✓ `Lagged(3)`, latest-only).
- Never block the workers: `spawn_blocking` for blocking or CPU-heavy work (✓ 250 ms stall vs ticks continuing); don't hold a `std` mutex across `.await` (✓ single-thread hang).
- Test time with the paused clock (✓ exact 30 ms/460 ms assertions), and test shutdown, backpressure and cancellation explicitly.

**Connections.** Lesson 30's `Waker`, `JoinHandle` and `block_on` are what Tokio's tasks, `JoinHandle` and `Runtime::block_on` implement at scale; Lesson 29's deadlock rules reappear for locks and bounded channels; Lesson 27's pool-shutdown protocol becomes `CancellationToken` + `JoinSet`. Next: Lesson 32 goes deep on cancellation safety, streams and backpressure; then Module 11 starts (modules, API design, macros).

---

## 17. References

- Tokio tutorial and documentation: <https://tokio.rs/tokio/tutorial> and <https://docs.rs/tokio>
- Tokio topics: graceful shutdown <https://tokio.rs/tokio/topics/shutdown>, bridging with sync code <https://tokio.rs/tokio/topics/bridging>, testing <https://tokio.rs/tokio/topics/testing>
- `tokio::task` (`spawn`, `JoinSet`, `spawn_blocking`, `LocalSet`), `tokio::sync` (`mpsc`, `oneshot`, `watch`, `broadcast`, `Mutex`, `Semaphore`), `tokio::time`: <https://docs.rs/tokio/latest/tokio/>
- `tokio_util::sync::CancellationToken`: <https://docs.rs/tokio-util/latest/tokio_util/sync/struct.CancellationToken.html>
- *Asynchronous Programming in Rust* (the Async Book): <https://rust-lang.github.io/async-book/>

The URLs above are the standard documentation locations; I did not open them during this session (the sandbox cannot reach them, though it can download crates from crates.io, which is how `tokio` 1.54.1 and `tokio-util` 0.7.20 were fetched), so if a link has moved, search the docs site for the page title. Unchecked or from memory: what `#[tokio::main]` expands to; the default worker count rule; work-stealing in the multi-thread scheduler; that `mio` uses epoll/kqueue/IOCP; the cancel-safety classification of `accept`, `read`, `write_all`, `read_exact`; unbounded-channel and `spawn_blocking` abort behavior; `block_in_place`; `interval` missed-tick behavior; the paused clock's auto-advance rule and `#[tokio::test]` using a current-thread runtime; Tokio's advice on `std` vs `tokio` mutex; `tokio-console`; the existence of other runtimes (async-std, smol); and the runtime-agnostic-library discussion. Everything marked ✓ was run on this machine's `rustc 1.97.0` with `tokio` 1.54.1.
