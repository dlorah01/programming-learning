---
title: "Lesson 27 — Threads, scoped threads, and channels"
---

# Lesson 27 — Threads, scoped threads, and channels

**Module 10 · Concurrency and async** · Status: generated · Edition used in examples: **2024** · External dependencies: none (standard library only)

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux. As before, ✓ marks outputs, diagnostics and measurements I actually ran; unmarked statements are documented behavior I did not run. Everything is **stable Rust**. Thread scheduling is nondeterministic: where an example's output could vary, the lesson prints only values that do not (sorted results, counts, per-sender order).

Builds on [Lesson 22](22_gats_send_sync_phantom.md) (`Send`/`Sync`, scoped threads in passing), [Lesson 23](23_box_rc_arc.md) (`Arc`), [Lesson 24](24_interior_mutability.md) (`Mutex`, poisoning) and [Lesson 26](26_error_design.md) (panics, unwinding, `catch_unwind`). Atomics are Lesson 28; locks and deadlocks are Lesson 29; `async` is Lessons 30 and 31.

---

## 1. Why threads, and what Rust adds

An OS thread is an independent flow of execution scheduled by the operating system, with its own stack, sharing the process's memory. Threads let a program use several cores and keep working while one flow blocks on I/O. Their classic danger is the **data race**: two threads access the same memory at the same time, at least one writes, and nothing orders them. In C and C++ that is undefined behavior and the bug is usually found in production.

Rust's ownership rules plus `Send`/`Sync` (Lesson 22) make data races in **safe code** a compile error. That does not make concurrent programs correct: deadlocks, lost wake-ups, ordering bugs ("check, then act") and resource exhaustion are all still possible, and this lesson shows some of them. What the compiler gives you is that the remaining bugs are *logic* bugs, not memory corruption.

Rust's standard threads are **1:1 OS threads** (each `std::thread` is one kernel thread). There is no runtime scheduler and no green threads in `std`; `async` tasks (Lesson 30) are the lightweight alternative.

---

## 2. `thread::spawn` and `JoinHandle`

```rust
use std::thread;
fn main() {
    let h = thread::spawn(|| { 6 * 7 });
    println!("{}", h.join().unwrap());
    let v = vec![1, 2, 3];
    let h = thread::spawn(move || v.iter().sum::<i32>());
    println!("{}", h.join().unwrap());
    let b = thread::Builder::new().name("worker-1".into()).stack_size(256 * 1024);
    let h = b.spawn(|| thread::current().name().map(str::to_owned)).unwrap();
    println!("{:?}", h.join().unwrap());
    println!("{:?}", thread::current().name());
    println!("{}", thread::available_parallelism().map(|n| n.get() > 0).unwrap_or(false));
    let ids: Vec<_> = (0..3).map(|_| thread::spawn(|| thread::current().id())).collect();
    let ids: Vec<_> = ids.into_iter().map(|h| h.join().unwrap()).collect();
    println!("{}", ids[0] != ids[1] && ids[1] != ids[2] && ids[0] != thread::current().id());
}
```

✓ Output `42`, `6`, `Some("worker-1")`, `Some("main")`, `true`, `true`. What this shows:

- `spawn` takes a closure and returns a `JoinHandle<T>`; `join()` blocks until the thread ends and returns `Result<T, Box<dyn Any + Send>>`: `Ok(value)` normally, `Err(payload)` if the thread **panicked** (Lesson 26 ✓).
- `thread::Builder` lets you name a thread (names appear in panic messages and debuggers) and set its stack size, and its `spawn` returns an `io::Result`, because creating a thread can fail (out of memory, process limits). `thread::spawn` panics in that case; a server that must survive resource exhaustion uses `Builder`.
- `thread::available_parallelism()` is the standard hint for how many CPU-bound threads to run (✓ it returned a positive number here; the value depends on the machine and any container limits, which is why I print only that it is positive).
- Each thread has its own `ThreadId` (✓ three spawned threads and the main thread all differ).

### 2.1 What the signature demands, and what the compiler says ✓

The full signature is `spawn<F, T>(f: F) -> JoinHandle<T> where F: FnOnce() -> T + Send + 'static, T: Send + 'static` (documented). Each bound shows up as an error:

**`'static`: the thread may outlive the function that created it**, so it cannot borrow locals:

```rust
use std::thread;
fn main() {
    let v = vec![1, 2, 3];
    let h = thread::spawn(|| println!("{v:?}"));
    h.join().unwrap();
}
```

✓ `error[E0373]: closure may outlive the current function, but it borrows `v`, which is owned by the current function`, help: use the `move` keyword. (Even though `join()` follows immediately, the compiler doesn't reason about that; scoped threads in Section 3 do.)

**`Send`: everything the closure captures must be safe to move to another thread:**

```rust
use std::rc::Rc;
use std::thread;
fn main() {
    let r = Rc::new(5);
    thread::spawn(move || println!("{r}")).join().unwrap();
}
```

✓ `error[E0277]: `Rc<i32>` cannot be sent between threads safely` (Lesson 22).

**`move` transfers ownership, so the parent can no longer use the value:**

```rust
use std::thread;
fn main() {
    let v = vec![1, 2, 3];
    let h = thread::spawn(move || println!("{v:?}"));
    println!("{v:?}");
    h.join().unwrap();
}
```

✓ `error[E0382]: borrow of moved value: `v``. To share, clone an `Arc` per thread (Lesson 23) or use a scoped thread.

### 2.2 Other facts about threads ✓

```rust
use std::thread;
use std::time::Duration;
fn main() {
    let t = thread::spawn(|| { thread::park(); "unparked" });
    thread::sleep(Duration::from_millis(30));
    t.thread().unpark();
    println!("{}", t.join().unwrap());
    let done = thread::spawn(|| 1);
    thread::sleep(Duration::from_millis(30));
    println!("{}", done.is_finished());
    println!("{}", done.join().unwrap());
    // detached thread is killed when main returns
    thread::spawn(|| { thread::sleep(Duration::from_millis(500)); println!("never printed"); });
    println!("main exits");
}
```

✓ Output `unparked`, `true`, `1`, `main exits`: the last spawned thread never got to print.

- **Dropping a `JoinHandle` detaches the thread**; it keeps running, but when `main` returns the whole process exits and detached threads are killed without running destructors. Join the threads whose work you need.
- `park`/`unpark` is the low-level "sleep until someone wakes me" primitive; channels, `Mutex` and `Condvar` (Lesson 29) are built on this kind of mechanism. Prefer the higher-level tools.
- There is **no safe way to kill a thread** from the outside in `std`. Cancellation is cooperative: send a message or set a flag the thread checks (Lesson 32 covers cancellation for async code).
- `thread_local!` gives each thread its own copy of a value ✓:

```rust
use std::cell::RefCell;
use std::thread;
thread_local! { static COUNTER: RefCell<u32> = RefCell::new(0); }
fn main() {
    COUNTER.with(|c| *c.borrow_mut() += 1);
    let h = thread::spawn(|| { COUNTER.with(|c| { *c.borrow_mut() += 10; *c.borrow() }) });
    println!("{} {}", h.join().unwrap(), COUNTER.with(|c| *c.borrow()));
}
```

✓ Output `10 1`: the spawned thread started from its own `0`, and main's copy is unaffected. Thread-locals suit per-thread caches and scratch buffers; they are the wrong tool for results you need to collect (return them from the thread instead).

---

## 3. Scoped threads: borrowing across threads

`thread::scope` creates a scope in which spawned threads may **borrow** from the enclosing stack, because the scope guarantees every thread is joined before it returns ✓:

```rust
use std::thread;
fn main() {
    let mut data = vec![1, 2, 3];
    let total = 10;
    let sum: i32 = thread::scope(|s| {
        let h1 = s.spawn(|| data.iter().sum::<i32>() + total);
        let h2 = s.spawn(|| data.len() as i32);
        h1.join().unwrap() + h2.join().unwrap()
    });
    println!("{sum}");
    thread::scope(|s| { s.spawn(|| data.push(4)); });
    println!("{data:?}");
    // chunked parallel sum
    let nums: Vec<u64> = (1..=1000).collect();
    let parts: Vec<u64> = thread::scope(|s| {
        let hs: Vec<_> = nums.chunks(250).map(|c| s.spawn(move || c.iter().sum::<u64>())).collect();
        hs.into_iter().map(|h| h.join().unwrap()).collect()
    });
    println!("{parts:?} {}", parts.iter().sum::<u64>());
}
```

✓ Output `19` (6 + 10 for the sum thread, plus 3 for the length thread), `[1, 2, 3, 4]`, `[31375, 93875, 156375, 218875] 500500`. Two threads read `data` at the same time through shared references; then a third mutates it after the first scope ended. The borrow rules still apply *across* threads: two threads trying to mutate the same vector is rejected ✓:

```rust
use std::thread;
fn main() {
    let mut data = vec![1, 2, 3];
    thread::scope(|s| {
        s.spawn(|| data.push(4));
        s.spawn(|| data.push(5));
    });
}
```

✓ `error[E0499]: cannot borrow `data` as mutable more than once at a time`. This is the data-race prevention in action: the same rule from Lesson 10, now covering threads. (To mutate shared data from several threads you need a `Mutex`, Lesson 24, or to give each thread its own disjoint piece, such as `chunks_mut`.)

**A panic inside a scoped thread** ✓:

```rust
use std::thread;
fn main() {
    let r = std::panic::catch_unwind(|| {
        thread::scope(|s| {
            s.spawn(|| { panic!("worker failed"); });
            s.spawn(|| println!("other thread still runs"));
        });
    });
    match r {
        Ok(()) => println!("no panic"),
        Err(p) => println!("scope panicked: {:?}", p.downcast_ref::<&str>().copied().or_else(|| p.downcast_ref::<String>().map(|s| s.as_str()))),
    }
}
```

✓ Output (the thread's own panic message goes to standard error): `other thread still runs`, then `scope panicked: Some("a scoped thread panicked")`. Every thread in the scope ran to completion, and since nobody called `join()` on the panicking one, `scope` itself panicked after joining all of them. If you `join()` a scoped handle yourself you get the thread's `Err(payload)` instead and the scope does not panic. The practical challenge's `parallel_map` joins explicitly and re-raises the **original payload** with `resume_unwind`.

Use scoped threads whenever the work is "split this data, process in parallel, collect": no `Arc`, no `'static`, no leaks.

---

## 4. Message passing with channels

"Do not communicate by sharing memory; share memory by communicating." A channel moves values from senders to a receiver; ownership moves with the value, so there is no shared mutable state to protect. `std::sync::mpsc` is **m**ultiple-**p**roducer, **s**ingle-**c**onsumer.

### 4.1 `mpsc::channel`: send, receive, disconnect ✓

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;
fn main() {
    let (tx, rx) = mpsc::channel();
    let tx2 = tx.clone();
    let a = thread::spawn(move || { for i in 0..3 { tx.send(("a", i)).unwrap(); } });
    let b = thread::spawn(move || { for i in 0..3 { tx2.send(("b", i)).unwrap(); } });
    a.join().unwrap(); b.join().unwrap();
    let got: Vec<_> = rx.iter().collect();   // ends because both senders were dropped
    let a_order: Vec<i32> = got.iter().filter(|(n, _)| *n == "a").map(|(_, i)| *i).collect();
    let b_order: Vec<i32> = got.iter().filter(|(n, _)| *n == "b").map(|(_, i)| *i).collect();
    println!("{} {:?} {:?}", got.len(), a_order, b_order);
    let (tx, rx) = mpsc::channel::<i32>();
    println!("{:?}", rx.try_recv());
    println!("{:?}", rx.recv_timeout(Duration::from_millis(20)));
    drop(tx);
    println!("{:?}", rx.try_recv());
    println!("{:?}", rx.recv());
    let (tx, rx) = mpsc::channel::<String>();
    drop(rx);
    let e = tx.send("lost".to_string()).unwrap_err();
    println!("{e} / returned value: {}", e.0);
}
```

✓ Output:

```text
6 [0, 1, 2] [0, 1, 2]
Err(Empty)
Err(Timeout)
Err(Disconnected)
Err(RecvError)
sending on a closed channel / returned value: lost
```

The rules, all visible above:

- Messages from **one sender** arrive in the order sent (`[0, 1, 2]` for each); the interleaving **between** senders is not specified, so don't depend on it (I printed them separately for that reason).
- `Sender` is cloneable (that's the "multiple producers"). The channel is **disconnected** for the receiver only when *every* `Sender` is dropped.
- `rx.iter()` / `for x in rx` ends when the channel is empty **and** disconnected; until then it blocks.
- `try_recv` never blocks (`Empty` or `Disconnected`), `recv_timeout` blocks up to a deadline (`Timeout`), `recv` blocks forever and returns `Err(RecvError)` only on disconnect.
- `send` fails when the receiver is gone, and the `SendError` **hands your value back** (`e.0` is `"lost"`), so nothing is silently lost. A failed `send` usually means "the consumer has shut down", so a producer typically stops.
- Sending **moves** the value ✓: `tx.send(s)` then using `s` gives `error[E0382]: borrow of moved value: `s``. That is the point: no aliasing.

`try_iter()` drains what is already queued without blocking ✓: after sending `0..5`, `rx.try_iter().take(2)` gave `[0, 1]` and a second `try_iter().collect()` gave `[2, 3, 4]` (note `take(2)` on `try_iter()` consumes only two items; the rest stayed in the channel).

### 4.2 The classic hang: a sender you forgot to drop ✓

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;
fn main() {
    let (tx, rx) = mpsc::channel();
    let worker_tx = tx.clone();
    thread::spawn(move || { worker_tx.send(1).unwrap(); });
    // BUG: the original `tx` is still alive in main
    let mut seen = vec![];
    loop {
        match rx.recv_timeout(Duration::from_millis(200)) {
            Ok(v) => seen.push(v),
            Err(e) => { println!("{seen:?} then {e:?}"); break; }
        }
    }
    drop(tx);
    println!("{:?}", rx.recv());
}
```

✓ Output `[1] then Timeout`, then `Err(RecvError)`. The worker finished, but `main` still holds a `Sender`, so the channel is not disconnected and a plain `for v in rx` would wait forever (I used `recv_timeout` here only so the example terminates). After `drop(tx)` the receiver finally sees the disconnect. **Rule: the code that creates the channel must drop its own `Sender` after handing clones to the producers.** The debugging challenge is this bug in a realistic program.

### 4.3 Bounded channels and back-pressure: `sync_channel` ✓

`mpsc::channel()` is **unbounded**: a fast producer can queue messages without limit until memory runs out. `sync_channel(n)` has capacity `n`; a full channel makes `send` block (or `try_send` fail), which slows the producer down to the consumer's pace: *back-pressure*.

```rust
use std::sync::mpsc::{self, TrySendError};
use std::thread;
fn main() {
    let (tx, rx) = mpsc::sync_channel::<u32>(2);
    println!("{:?} {:?}", tx.try_send(1), tx.try_send(2));
    match tx.try_send(3) { Err(TrySendError::Full(v)) => println!("full, got back {v}"), other => println!("{other:?}") }
    println!("{:?}", rx.recv());
    println!("{:?}", tx.try_send(3));
    drop(rx);
    println!("{:?}", tx.try_send(4));
    // rendezvous channel: capacity 0
    let (tx, rx) = mpsc::sync_channel::<u8>(0);
    println!("{:?}", tx.try_send(1));
    let h = thread::spawn(move || rx.recv().unwrap());
    tx.send(9).unwrap();
    println!("{}", h.join().unwrap());
}
```

✓ Output:

```text
Ok(()) Ok(())
full, got back 3
Ok(1)
Ok(())
Err(TrySendError::Disconnected(..))
Err(TrySendError::Full(..))
9
```

Capacity 2 accepted two messages; the third came back inside `TrySendError::Full(3)`; after one `recv` there was room again; with the receiver dropped, `try_send` reports `Disconnected`. With capacity **0** (a *rendezvous* channel) `try_send` fails unless a receiver is already waiting, and `send` completes only when the receiver takes the value (✓ `9` arrived).

Choose the capacity deliberately. Unbounded queues turn overload into memory exhaustion; small bounded queues turn overload into slowness, which can be shed or reported (Section 6).

### 4.4 Who may share what ✓

```rust
fn assert_send<T: Send>() {}
fn assert_sync<T: Sync>() {}
fn main() {
    assert_send::<std::sync::mpsc::Sender<u8>>();
    assert_sync::<std::sync::mpsc::Sender<u8>>();
    assert_send::<std::sync::mpsc::Receiver<u8>>();
    assert_sync::<std::sync::mpsc::Receiver<u8>>();
    assert_send::<std::sync::mpsc::Sender<std::rc::Rc<u8>>>();
}
```

✓ Two errors, both `E0277`: ``Receiver<u8>` cannot be shared between threads safely`` (the `assert_sync` on the `Receiver`) and ``Rc<u8>` cannot be sent between threads safely`` (required for `Sender<Rc<u8>>` to be `Send`). The `Sender` lines for `u8` compile: a `Sender<T>` is `Send` and `Sync` when `T: Send`, and a `Receiver` is `Send` but **not `Sync`**: one consumer. Two threads cannot both call `rx.recv()` through a shared reference ✓ (`E0277` "`Receiver<i32>` cannot be shared between threads safely" when two scoped threads borrow `rx`). If you want several consumers you wrap the receiver in a `Mutex` (below) or use a multi-consumer channel from a crate such as `crossbeam-channel` (not used here).

---

## 5. A worker pool by hand

Several workers sharing one receiver is the classic pool; the receiver goes in `Arc<Mutex<_>>` so one worker at a time waits on it ✓:

```rust
use std::sync::{mpsc, Arc, Mutex};
use std::thread;
// worker pool: shared receiver
fn main() {
    let (tx, rx) = mpsc::channel::<u32>();
    let rx = Arc::new(Mutex::new(rx));
    let (rtx, rrx) = mpsc::channel::<(usize, u32)>();
    let workers: Vec<_> = (0..3).map(|id| {
        let rx = Arc::clone(&rx);
        let rtx = rtx.clone();
        thread::spawn(move || loop {
            let job = rx.lock().unwrap().recv();   // guard dropped at end of this statement
            match job { Ok(n) => rtx.send((id, n * n)).unwrap(), Err(_) => break }
        })
    }).collect();
    drop(rtx);
    for n in 1..=8 { tx.send(n).unwrap(); }
    drop(tx);
    for w in workers { w.join().unwrap(); }
    let mut results: Vec<u32> = rrx.iter().map(|(_, v)| v).collect();
    results.sort();
    println!("{results:?}");
}
```

✓ Output `[1, 4, 9, 16, 25, 36, 49, 64]`. The shutdown protocol is the important part: the producer drops `tx`, so each worker's `recv()` returns `Err` once the queue is empty, the worker breaks out of its loop, `join()` returns, and dropping `rtx` (the original, after cloning one per worker) lets the results iterator end. Results arrive in nondeterministic order, so the program sorts them.

### 5.1 The bug that makes a pool serial ✓

Locking the receiver and **running the job under the same guard** serializes everything. The tempting form is:

```rust
while let Ok(job) = rx.lock().unwrap().recv() { job(); }
```

The `MutexGuard` is a temporary in the `while let` scrutinee, and temporaries of the scrutinee live until the end of the loop body (Lesson 24 showed the same rule for `match`), so the lock is held while `job()` runs. I measured the effect: four workers, eight jobs that each sleep 50 ms, recording the highest number of jobs running at once:

```rust
use std::sync::atomic::{AtomicUsize, Ordering::SeqCst};
use std::sync::{mpsc, Arc, Mutex};
use std::thread;
use std::time::Duration;
fn run(hold_lock_during_job: bool) -> usize {
    let (tx, rx) = mpsc::channel::<u32>();
    let rx = Arc::new(Mutex::new(rx));
    let active = Arc::new(AtomicUsize::new(0));
    let max = Arc::new(AtomicUsize::new(0));
    let hs: Vec<_> = (0..4).map(|_| {
        let (rx, active, max) = (Arc::clone(&rx), Arc::clone(&active), Arc::clone(&max));
        thread::spawn(move || {
            let work = |_n: u32| {
                let now = active.fetch_add(1, SeqCst) + 1;
                max.fetch_max(now, SeqCst);
                thread::sleep(Duration::from_millis(50));
                active.fetch_sub(1, SeqCst);
            };
            if hold_lock_during_job {
                // BUG: the guard is a temporary of the `while let` scrutinee and lives for the whole body
                while let Ok(n) = rx.lock().unwrap().recv() { work(n); }
            } else {
                loop {
                    let msg = rx.lock().unwrap().recv();
                    match msg { Ok(n) => work(n), Err(_) => break }
                }
            }
        })
    }).collect();
    for n in 0..8 { tx.send(n).unwrap(); }
    drop(tx);
    for h in hs { h.join().unwrap(); }
    max.load(SeqCst)
}
fn main() {
    println!("lock held during job: max concurrent = {}", run(true));
    println!("lock released first : max concurrent >= 2 is {}", run(false) >= 2);
}
```

✓ Output (the same in three runs) `lock held during job: max concurrent = 1` and `lock released first : max concurrent >= 2 is true`. The program with the bug still produces correct results, only slowly: nothing crashes, so the only way to find it is to measure or to write a test that proves two jobs overlap (the practical challenge's `workers_really_run_in_parallel`). The fix is to end the statement that locks, so the guard is dropped *before* the job runs: `let msg = rx.lock().unwrap().recv(); match msg { ... }`. The atomics here (`fetch_add`, `fetch_max`) are just counters for the measurement; atomics are Lesson 28. This test relies on 50 ms sleeps being long enough for threads to overlap, which held in every run here but is a timing argument, not a proof.

---

## 6. Design guidance

- **Spawning.** A thread has a stack (its size is configurable with `Builder::stack_size`; the default is platform-dependent) and kernel resources. Thread creation is cheap compared with a process but not free, and a thread per request/connection does not scale to tens of thousands of connections: use a fixed pool, or async tasks (Lesson 30). I did not measure thread-spawn cost here.
- **How many threads?** For CPU-bound work, about `available_parallelism()` workers; more only adds context switches. For blocking I/O, more threads can help, bounded by a limit you choose.
- **Prefer scoped threads** for fork-join work on borrowed data, **pools** for a stream of independent jobs, **channels** to move data between pipeline stages, and `Arc<Mutex<_>>` only for state that really is shared.
- **Channels: bounded or unbounded?** Bounded (`sync_channel`) by default for anything that can be overloaded; unbounded only when the producer is naturally limited.
- **Shutdown is a protocol.** Decide who closes the channel, how workers learn to stop (disconnect, a shutdown message, or a flag), and who joins them. Use `Drop` to make the safe path the default (the practical challenge's pool joins in `Drop`).
- **Errors.** `join()` returns `Err` for a panicked thread (Lesson 26); `send` returns `Err` when the receiver is gone. Don't `unwrap` these in code that must keep running; decide whether the failure should stop the system, restart the worker, or be reported.
- **Panic isolation.** In a pool, catch a job's panic so one bad job doesn't kill a worker (✓ practical challenge), and count it so it is not invisible.
- **Pipelines.** Each stage is a thread reading from one channel and writing to the next; bounded channels between stages give back-pressure end to end.
- **Data parallelism libraries** (for example `rayon`) provide parallel iterators on top of a work-stealing pool; the standard library does not, and this course uses only `std` here.

---

## 7. Common mistakes and edge cases

- **Borrowing a local in `spawn`** (✓ E0373): `move` it, clone an `Arc`, or use `thread::scope`.
- **Using a value after moving it into a thread or channel** (✓ E0382).
- **Sending `Rc`/non-`Send` data** (✓ E0277).
- **Forgetting to drop the original `Sender`** (✓ `Timeout` instead of a clean end, or a permanent hang).
- **Holding a lock while doing the work** (✓ max concurrency 1).
- **Unbounded channels with a fast producer**: memory grows without limit.
- **Assuming message order across senders**, or the order in which threads finish.
- **`unwrap()` on `join()` or `send()` everywhere**: a panic in one thread then cascades into the others.
- **Relying on `main` to wait for detached threads** (✓ "never printed").
- **Calling `rx.recv()` on a thread that is also the only sender** (waits forever, since its own `Sender` keeps the channel open).
- **Sleeping to "wait for the other thread"**: use `join`, a channel or a `Condvar` (Lesson 29).
- **Spawning one thread per tiny task**: the thread costs more than the work.
- **Blocking inside an `async` task with `std` threads/`recv()`**: it blocks an executor thread (Lessons 30 and 31).

---

## 8. Performance implications

- Parallel speedup is limited by the serial part of the work, by memory bandwidth, by contention on shared state, and by the cost of splitting and joining. I did not benchmark `parallel_map`; whether it beats a sequential loop depends on how much work each item does. Parallelizing a loop whose body takes nanoseconds usually makes it slower.
- Chunking matters: `parallel_map` gives each thread a contiguous chunk, which is cache-friendly and avoids per-item synchronization, but a chunk with slow items can leave other threads idle (imbalance). Work-stealing pools address this.
- A channel send/receive involves synchronization on both sides (and a wake-up if the receiver is parked); sending one tiny message per item can cost more than the work. Batch the items.
- The shared-receiver pool serializes the *dequeue* step behind a mutex; with very short jobs that mutex becomes the bottleneck. A multi-consumer queue or per-worker queues scales further (not measured).
- Thread-locals avoid sharing entirely and are often the fastest per-thread cache.
- False sharing (two threads writing different values that sit on the same cache line) can silently slow parallel code; Lesson 42 covers it.

---

## 9. Security and reliability

- **Unbounded queues are a memory-exhaustion vector.** If untrusted input can enqueue work faster than you process it, an unbounded channel grows until the process is killed. Use bounded queues and decide what to do when full: block, shed, or reject with an error (the pool's `submit` returns `Full`).
- **Thread-per-request is a DoS amplifier.** Each connection that creates a thread consumes stack and kernel resources; cap the number of workers, and use `Builder::spawn` (which returns an error) rather than `thread::spawn` (which panics) where exhaustion is a possibility.
- **Panics must not take the service down silently or loudly in a cascade.** Catch job panics in a pool and count them (✓ `panicked` stat); don't `unwrap()` a `join()` of a worker that can legitimately have failed.
- **Safe Rust prevents data races, not deadlocks, livelocks or logic races.** Check-then-act across a lock release is still a race (Lesson 29 covers lock ordering).
- **Joining in `Drop` can hang.** If a worker is stuck, `drop` blocks forever; for shutdown paths that must finish, use timeouts on the jobs themselves or a design where stuck workers can be abandoned.
- **Secrets in thread-locals and per-thread buffers persist** after use; zeroize them where it matters (Lesson 48). A thread's memory is not cleared when it exits.
- **Don't claim that "Rust is thread-safe" means "the program is correct"**: only data races in safe code are ruled out. `unsafe` code can break `Send`/`Sync` promises (Lessons 36 and 38).

---

## 10. Exercises

Try each before opening the answer. They are not marked complete unless you tell me you did them.

1. **(Easy)** Why does `thread::spawn(|| println!("{v:?}"))` fail to compile even if you call `join()` right after?
2. **(Easy)** What does `h.join()` return if the thread panicked? What if the thread returned `5`?
3. **(Medium)** Three producers send to one receiver and `main` runs `for v in rx`. The program never ends. Why, and what is the fix?
4. **(Medium)** When does `rx.recv()` return `Err`? When does `tx.send(x)` return `Err`, and what do you get back?
5. **(Medium)** Using `thread::scope`, sum the four chunks of `nums: Vec<u64>` in parallel without `Arc` or `'static`.
6. **(Hard)** A pool of four workers shares `Arc<Mutex<Receiver<Job>>>` and uses `while let Ok(job) = rx.lock().unwrap().recv() { job() }`. Everything works but is slow. Why?
7. **(Hard)** A producer reads from a network socket and sends each message to a processing thread through `mpsc::channel()`. During a traffic spike the process's memory use climbs. Diagnose and fix.
8. **(Hard)** You need several consumers for one queue and want a graceful shutdown that finishes queued work. Describe the protocol with the pieces in this lesson.

<details markdown="1">
<summary>Answers</summary>

1. The closure borrows `v`, but `spawn` requires `F: 'static`: the compiler only knows the thread *may* outlive the function; it does not use the later `join()` (✓ `E0373`). Fix: `move` the data in (or clone an `Arc`), or use `thread::scope`, which guarantees the join.
2. ✓ `join()` returns `Result<T, Box<dyn Any + Send>>`: `Err(payload)` after a panic (the payload is the panic message as `&str`/`String`), `Ok(5)` if the thread returned `5`.
3. ✓ `for v in rx` ends only when all `Sender`s are dropped. The three clones moved into the producers are dropped when those threads end, but `main` still holds the original `tx`, so the channel never disconnects. Fix: `drop(tx)` after spawning (✓ the fixed program printed `[0, 1, 10, 11, 20, 21]`; the unfixed one was killed by `timeout` with status 124).
4. ✓ `recv()` returns `Err(RecvError)` when the channel is empty **and** every `Sender` is dropped. `send(x)` returns `Err(SendError(x))` when the `Receiver` is dropped, giving you the value back in `e.0`.
5. ✓ (the program in Section 3): `let parts: Vec<u64> = thread::scope(|s| { let hs: Vec<_> = nums.chunks(250).map(|c| s.spawn(move || c.iter().sum::<u64>())).collect(); hs.into_iter().map(|h| h.join().unwrap()).collect() }); let total: u64 = parts.iter().sum();` gave `[31375, 93875, 156375, 218875] 500500`.
6. ✓ The `MutexGuard` produced by `rx.lock().unwrap()` is a temporary of the `while let` scrutinee and lives until the end of the loop body, so the lock is held while `job()` runs. Only one worker can run a job at a time (measured: max concurrency 1). Fix: `let msg = rx.lock().unwrap().recv();` (guard dropped at the semicolon), then match on `msg`.
7. The unbounded channel lets the producer outrun the consumer; the queue grows without limit. Use `sync_channel(n)` so the producer blocks (back-pressure; the socket then fills and TCP slows the sender), or `try_send` and drop/reject messages when full (✓ `TrySendError::Full(msg)` hands the message back), and expose queue depth as a metric.
8. Put the receiver in `Arc<Mutex<Receiver<_>>>`, clone the `Arc` into each worker, and make workers lock only to `recv()` (not to process). Shutdown: the owner drops the `Sender` (so no new work), workers keep receiving until the queue is empty and then see `Err`, exit their loops, and the owner `join()`s them. That is exactly `WorkerPool::shutdown` in the practical challenge.

</details>

---

## 11. Debugging challenge

This program is supposed to print the sorted numbers produced by three threads, but it never finishes (run it under `timeout 5`):

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();
    for id in 0..3 {
        let tx = tx.clone();
        thread::spawn(move || {
            for i in 0..2 {
                tx.send(id * 10 + i).unwrap();
            }
        });
    }
    let mut all: Vec<i32> = Vec::new();
    for v in rx {
        all.push(v);
    }
    all.sort();
    println!("{all:?}");
}
```

1. Why does it hang, and how can you prove your explanation? 2. Fix it. 3. In your fix, what guarantees that all six numbers are present when the loop ends? 4. What changes if one of the threads panics after sending one message?

<details markdown="1">
<summary>Solution</summary>

**1. Diagnosis.** ✓ With `timeout 5` the program printed nothing and exited with status 124 (killed). The three producer threads each get a clone of `tx` and finish quickly, dropping their clones. But `main` still owns the *original* `tx`, so there is always at least one live `Sender`, and `for v in rx` keeps waiting for more messages. To prove it, replace the loop with `rx.recv_timeout(Duration::from_millis(200))`: you receive all six values and then `Err(Timeout)`, not `Err(Disconnected)` (this is the experiment from Section 4.2: `[1] then Timeout`).

**2. Fix.** Drop the original sender once all clones have been handed out:

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();
    for id in 0..3 {
        let tx = tx.clone();
        thread::spawn(move || {
            for i in 0..2 {
                tx.send(id * 10 + i).unwrap();
            }
        });
    }
    drop(tx);
    let mut all: Vec<i32> = Vec::new();
    for v in rx {
        all.push(v);
    }
    all.sort();
    println!("{all:?}");
}
```

✓ Output `[0, 1, 10, 11, 20, 21]`, exit status 0. (An equally good fix is to avoid keeping an original at all: create the channel, and `move` the original `tx` into the last producer instead of cloning it.)

**3. Why all six arrive.** A channel delivers every message that was sent before the last `Sender` was dropped; disconnection is only reported once the queue has been drained. Each producer's sends happen before its clone is dropped, so by the time the receiver sees "empty and disconnected" every message has already been queued and received. The threads were never joined, and that is fine here: the channel's disconnect is the synchronization.

**4. A panic after one send.** The unwinding thread drops its `Sender` clone (destructors run, Lesson 26), so the channel can still disconnect, and the receiver gets the message sent before the panic but not the ones after it. The panic message goes to standard error and nobody sees the failure in the *result* unless the producer's `JoinHandle` is kept and joined (`Err(payload)`). The loop would print five numbers and succeed: a silent partial result. If completeness matters, keep the handles, `join()` them and check, or send a "done" message per producer and count them.

</details>

---

## 12. Practical challenge: `threadlab`

Create a library crate `threadlab` (edition 2024, no dependencies) with:

1. `parallel_map(items: &[T], threads: usize, f: impl Fn(&T) -> R + Sync) -> Vec<R>` (`T: Sync`, `R: Send`) using **scoped threads**: results in input order; `threads == 0` treated as 1; never more threads than items; empty input returns an empty `Vec`; if `f` panics, join every thread and then re-raise the original panic payload with `resume_unwind`.
2. `WorkerPool` with `new(workers, queue) -> io::Result<Self>` (zero workers is an `InvalidInput` error; worker threads are named with `Builder`), a shared `Arc<Mutex<Receiver<Job>>>`, a **bounded** queue, and:
   - `submit` (never blocks; `Err(Full)` / `Err(Closed)`) and `submit_blocking`;
   - `close(&mut self)` (stop accepting, queued jobs still run);
   - `shutdown(self) -> PoolStats` (close, run everything queued, join the workers) and a `Drop` that does the same;
   - job panics caught with `catch_unwind` and counted, the worker surviving;
   - the lock on the receiver **released before running each job**.
3. `SubmitError` with `Display` and `Error`; `PoolStats { completed, panicked }`.
4. Tests: `parallel_map` equals a sequential map for lengths 0 to 20 and thread counts 0 to 9; it borrows local data and really uses several threads; it re-raises a panic with the original message. For the pool: every job runs and `shutdown` drains the queue (sum of 1..=100); back-pressure (`Full` while the single worker is blocked and the queue is full, then recovery); two workers **provably run in parallel** (both jobs arrive before either is released, with timeouts instead of hangs); a panicking job doesn't kill the worker; `close` rejects new jobs but runs queued ones; dropping the pool waits for the jobs; zero workers is an error; the pool is `Send + Sync`. Doctests: a usage example, and a `compile_fail` showing a job cannot borrow a local.
5. It must pass `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, and `cargo test --locked` (run `cargo generate-lockfile` once first).

<details markdown="1">
<summary>Solution</summary>

**`src/lib.rs`**

```rust
//! `threadlab`: threads, scoped threads and channels, used carefully.
//!
//! * [`parallel_map`]: order-preserving data parallelism with scoped threads.
//! * [`WorkerPool`]: a fixed set of worker threads fed through a *bounded*
//!   queue, with back-pressure, panic isolation and graceful shutdown.

use std::error::Error;
use std::fmt;
use std::io;
use std::panic::{self, AssertUnwindSafe};
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::mpsc::{self, Receiver, SyncSender, TrySendError};
use std::sync::{Arc, Mutex, PoisonError};
use std::thread::{self, JoinHandle};

/// Applies `f` to every item using up to `threads` scoped threads and returns
/// the results **in input order**.
///
/// `threads == 0` is treated as 1, and more threads than items are never
/// spawned. The closure may borrow from the caller's stack because the
/// threads are scoped. If `f` panics, the panic is re-raised on the calling
/// thread after all threads have finished.
pub fn parallel_map<T, R, F>(items: &[T], threads: usize, f: F) -> Vec<R>
where
    T: Sync,
    R: Send,
    F: Fn(&T) -> R + Sync,
{
    if items.is_empty() {
        return Vec::new();
    }
    let threads = threads.clamp(1, items.len());
    let chunk = items.len().div_ceil(threads);
    thread::scope(|s| {
        let f = &f;
        let handles: Vec<_> = items
            .chunks(chunk)
            .map(|part| s.spawn(move || part.iter().map(f).collect::<Vec<R>>()))
            .collect();
        let mut out = Vec::with_capacity(items.len());
        let mut first_panic = None;
        for h in handles {
            match h.join() {
                Ok(v) => out.extend(v),
                Err(payload) => {
                    first_panic.get_or_insert(payload);
                }
            }
        }
        if let Some(payload) = first_panic {
            panic::resume_unwind(payload);
        }
        out
    })
}

type Job = Box<dyn FnOnce() + Send + 'static>;

/// Why a job could not be queued.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum SubmitError {
    /// The bounded queue is full right now (back-pressure).
    Full,
    /// The pool has been closed.
    Closed,
}

impl fmt::Display for SubmitError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            SubmitError::Full => f.write_str("job queue is full"),
            SubmitError::Closed => f.write_str("worker pool is closed"),
        }
    }
}

impl Error for SubmitError {}

/// What the workers did.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct PoolStats {
    /// Jobs that returned normally.
    pub completed: usize,
    /// Jobs that panicked (the worker survived).
    pub panicked: usize,
}

#[derive(Default)]
struct Counters {
    completed: AtomicUsize,
    panicked: AtomicUsize,
}

/// A fixed-size thread pool with a bounded job queue.
///
/// Jobs must be `'static`: they cannot borrow from the caller's stack.
///
/// ```compile_fail
/// let local = String::from("borrowed");
/// let pool = threadlab::WorkerPool::new(1, 1).unwrap();
/// pool.submit(|| println!("{local}")).unwrap();
/// ```
///
/// Usage:
///
/// ```
/// let squares = threadlab::parallel_map(&[1, 2, 3, 4], 2, |x| x * x);
/// assert_eq!(squares, vec![1, 4, 9, 16]);
/// ```
pub struct WorkerPool {
    sender: Option<SyncSender<Job>>,
    workers: Vec<JoinHandle<()>>,
    counters: Arc<Counters>,
}

impl WorkerPool {
    /// Starts `workers` threads sharing a queue that holds `queue` waiting
    /// jobs (`0` means a hand-off: a job is accepted only if a worker is
    /// ready to take it).
    ///
    /// Returns `InvalidInput` for zero workers, or the OS error if a thread
    /// could not be created (threads already started are shut down).
    pub fn new(workers: usize, queue: usize) -> io::Result<Self> {
        if workers == 0 {
            return Err(io::Error::new(
                io::ErrorKind::InvalidInput,
                "a pool needs at least one worker",
            ));
        }
        let (tx, rx) = mpsc::sync_channel::<Job>(queue);
        let rx = Arc::new(Mutex::new(rx));
        let counters = Arc::new(Counters::default());
        let mut pool = WorkerPool {
            sender: Some(tx),
            workers: Vec::with_capacity(workers),
            counters: Arc::clone(&counters),
        };
        for i in 0..workers {
            let rx = Arc::clone(&rx);
            let counters = Arc::clone(&counters);
            let handle = thread::Builder::new()
                .name(format!("pool-worker-{i}"))
                .spawn(move || worker_loop(&rx, &counters))?;
            pool.workers.push(handle);
        }
        Ok(pool)
    }

    /// Queues a job without waiting. `Err(Full)` means "try later or shed load".
    pub fn submit(&self, job: impl FnOnce() + Send + 'static) -> Result<(), SubmitError> {
        let sender = self.sender.as_ref().ok_or(SubmitError::Closed)?;
        sender.try_send(Box::new(job)).map_err(|e| match e {
            TrySendError::Full(_) => SubmitError::Full,
            TrySendError::Disconnected(_) => SubmitError::Closed,
        })
    }

    /// Queues a job, blocking while the queue is full (back-pressure).
    pub fn submit_blocking(&self, job: impl FnOnce() + Send + 'static) -> Result<(), SubmitError> {
        let sender = self.sender.as_ref().ok_or(SubmitError::Closed)?;
        sender.send(Box::new(job)).map_err(|_| SubmitError::Closed)
    }

    /// Stops accepting jobs. Queued jobs still run. Idempotent.
    pub fn close(&mut self) {
        self.sender = None;
    }

    /// Counters so far (a snapshot: workers may still be running).
    pub fn stats(&self) -> PoolStats {
        PoolStats {
            completed: self.counters.completed.load(Ordering::SeqCst),
            panicked: self.counters.panicked.load(Ordering::SeqCst),
        }
    }

    /// Closes the queue, waits for every queued and running job to finish and
    /// for all workers to exit, then returns the final counters.
    pub fn shutdown(mut self) -> PoolStats {
        self.close();
        self.join_workers();
        self.stats()
    }

    fn join_workers(&mut self) {
        for w in self.workers.drain(..) {
            // A worker thread only panics if the pool's own code is broken;
            // job panics are caught inside the loop.
            let _ = w.join();
        }
    }
}

impl Drop for WorkerPool {
    fn drop(&mut self) {
        self.close();
        self.join_workers();
    }
}

fn worker_loop(rx: &Mutex<Receiver<Job>>, counters: &Counters) {
    loop {
        // The guard is a temporary of this `let` statement, so the lock is
        // released *before* the job runs. Writing `while let Ok(job) =
        // rx.lock().unwrap().recv() { job() }` would keep it for the whole
        // body and serialize every job.
        let next = rx.lock().unwrap_or_else(PoisonError::into_inner).recv();
        let Ok(job) = next else { return };
        match panic::catch_unwind(AssertUnwindSafe(job)) {
            Ok(()) => counters.completed.fetch_add(1, Ordering::SeqCst),
            Err(_) => counters.panicked.fetch_add(1, Ordering::SeqCst),
        };
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::sync::atomic::AtomicUsize;
    use std::time::Duration;

    const LONG: Duration = Duration::from_secs(5);

    #[test]
    fn parallel_map_matches_sequential_for_every_shape() {
        for len in 0..=20usize {
            let items: Vec<usize> = (0..len).collect();
            let expected: Vec<usize> = items.iter().map(|x| x * x + 1).collect();
            for threads in 0..=9 {
                let got = parallel_map(&items, threads, |x| x * x + 1);
                assert_eq!(got, expected, "len={len} threads={threads}");
            }
        }
    }

    #[test]
    fn parallel_map_borrows_and_uses_several_threads() {
        let base = 1000;
        let names: Vec<String> = (0..16).map(|i| format!("n{i}")).collect();
        let ids = parallel_map(&names, 4, |n| (n.len() + base, thread::current().id()));
        assert_eq!(ids.len(), 16);
        let distinct: std::collections::HashSet<_> = ids.iter().map(|(_, id)| *id).collect();
        assert!(distinct.len() > 1, "expected more than one worker thread");
        assert!(!distinct.contains(&thread::current().id()));
    }

    #[test]
    fn parallel_map_reraises_panics_after_joining() {
        let items: Vec<u32> = (0..8).collect();
        let r = panic::catch_unwind(|| {
            parallel_map(&items, 4, |&x| {
                if x == 5 {
                    panic!("item five failed");
                }
                x
            })
        });
        let payload = r.unwrap_err();
        assert_eq!(
            payload.downcast_ref::<&str>().copied(),
            Some("item five failed")
        );
    }

    #[test]
    fn pool_runs_every_job_and_shutdown_drains_the_queue() {
        let pool = WorkerPool::new(3, 4).unwrap();
        let sum = Arc::new(AtomicUsize::new(0));
        for i in 1..=100usize {
            let sum = Arc::clone(&sum);
            pool.submit_blocking(move || {
                sum.fetch_add(i, Ordering::SeqCst);
            })
            .unwrap();
        }
        let stats = pool.shutdown();
        assert_eq!(
            stats,
            PoolStats {
                completed: 100,
                panicked: 0
            }
        );
        assert_eq!(sum.load(Ordering::SeqCst), 5050);
    }

    #[test]
    fn full_queue_pushes_back_and_recovers() {
        let pool = WorkerPool::new(1, 1).unwrap();
        let (started_tx, started_rx) = mpsc::channel();
        let (gate_tx, gate_rx) = mpsc::channel::<()>();
        pool.submit(move || {
            started_tx.send(()).unwrap();
            let _ = gate_rx.recv_timeout(LONG);
        })
        .unwrap();
        started_rx
            .recv_timeout(LONG)
            .expect("worker took the first job");
        assert_eq!(pool.submit(|| {}), Ok(())); // fills the single queue slot
        assert_eq!(pool.submit(|| {}), Err(SubmitError::Full));
        gate_tx.send(()).unwrap();
        // Once the worker drains the queue, submitting works again.
        let mut ok = false;
        for _ in 0..500 {
            if pool.submit(|| {}).is_ok() {
                ok = true;
                break;
            }
            thread::sleep(Duration::from_millis(10));
        }
        assert!(ok, "queue never freed up");
        // first job + the queued one + the one accepted after recovery;
        // the rejected submission never ran.
        assert_eq!(pool.shutdown().completed, 3);
    }

    #[test]
    fn workers_really_run_in_parallel() {
        let pool = WorkerPool::new(2, 2).unwrap();
        let (arrived_tx, arrived_rx) = mpsc::channel();
        let (go_tx, go_rx) = mpsc::channel::<()>();
        let go_rx = Arc::new(Mutex::new(go_rx));
        for _ in 0..2 {
            let arrived_tx = arrived_tx.clone();
            let go_rx = Arc::clone(&go_rx);
            pool.submit(move || {
                arrived_tx.send(()).unwrap();
                let _ = go_rx.lock().unwrap().recv_timeout(LONG);
            })
            .unwrap();
        }
        // Both jobs must be running at the same time to both arrive quickly.
        for _ in 0..2 {
            arrived_rx
                .recv_timeout(Duration::from_secs(2))
                .expect("second job never started: jobs are serialized");
        }
        go_tx.send(()).unwrap();
        go_tx.send(()).unwrap();
        assert_eq!(pool.shutdown().completed, 2);
    }

    #[test]
    fn a_panicking_job_does_not_kill_its_worker() {
        let pool = WorkerPool::new(1, 4).unwrap();
        pool.submit_blocking(|| panic!("job blew up")).unwrap();
        let (tx, rx) = mpsc::channel();
        pool.submit_blocking(move || tx.send("still alive").unwrap())
            .unwrap();
        assert_eq!(rx.recv_timeout(LONG), Ok("still alive"));
        assert_eq!(
            pool.shutdown(),
            PoolStats {
                completed: 1,
                panicked: 1
            }
        );
    }

    #[test]
    fn close_rejects_new_jobs_but_runs_queued_ones() {
        let mut pool = WorkerPool::new(1, 8).unwrap();
        let n = Arc::new(AtomicUsize::new(0));
        for _ in 0..5 {
            let n = Arc::clone(&n);
            pool.submit(move || {
                n.fetch_add(1, Ordering::SeqCst);
            })
            .unwrap();
        }
        pool.close();
        assert_eq!(pool.submit(|| {}), Err(SubmitError::Closed));
        assert_eq!(pool.submit_blocking(|| {}), Err(SubmitError::Closed));
        assert_eq!(pool.shutdown().completed, 5);
        assert_eq!(n.load(Ordering::SeqCst), 5);
    }

    #[test]
    fn dropping_the_pool_waits_for_the_jobs() {
        let n = Arc::new(AtomicUsize::new(0));
        {
            let pool = WorkerPool::new(2, 16).unwrap();
            for _ in 0..10 {
                let n = Arc::clone(&n);
                pool.submit_blocking(move || {
                    thread::sleep(Duration::from_millis(5));
                    n.fetch_add(1, Ordering::SeqCst);
                })
                .unwrap();
            }
        } // drop joins
        assert_eq!(n.load(Ordering::SeqCst), 10);
    }

    #[test]
    fn zero_workers_is_an_error_and_the_pool_is_thread_safe() {
        let e = WorkerPool::new(0, 1).err().expect("zero workers must fail");
        assert_eq!(e.kind(), io::ErrorKind::InvalidInput);
        fn send_sync<T: Send + Sync>() {}
        send_sync::<WorkerPool>();
        send_sync::<SubmitError>();
        assert_eq!(SubmitError::Full.to_string(), "job queue is full");
    }
}
```

✓ With this exact file: `cargo fmt --check` clean, `cargo clippy --all-targets --locked -- -D warnings` clean, and `cargo test --locked` → 10 unit tests, 1 passing doctest and 1 `compile_fail` doctest passed, in three consecutive runs (thread tests deserve repeated runs). In a separate crate that depends on `threadlab` I confirmed ✓ that the `compile_fail` program gives `error[E0373]: closure may outlive the current function, but it borrows `local``. My first version of `full_queue_pushes_back_and_recovers` expected 4 completed jobs and failed; the right number is 3 (the rejected submission never ran): a reminder to count what the test actually submits.

Mutation checks (each applied to a copy, then `cargo test --lib`):

| Change | Result ✓ |
|---|---|
| the worker keeps the receiver guard while running the job | `workers_really_run_in_parallel` FAILED (after its 2 s timeout) |
| no `catch_unwind` around jobs | `a_panicking_job_does_not_kill_its_worker` FAILED |
| `join_workers` joins nothing | 3 tests FAILED (`close_rejects_new_jobs_but_runs_queued_ones`, `dropping_the_pool_waits_for_the_jobs`, `workers_really_run_in_parallel`) |
| `submit` uses blocking `send` instead of `try_send` | `full_queue_pushes_back_and_recovers` FAILED (after ~5 s: the blocked `send` waited for the 5 s gate timeout) |
| `parallel_map` joins its handles in reverse order | `parallel_map_matches_sequential_for_every_shape` FAILED |
| `parallel_map` swallows the panic | `parallel_map_reraises_panics_after_joining` FAILED |

**Design notes.**

- **Order and panics in `parallel_map`.** Joining the handles in spawn order and extending the output chunk by chunk preserves input order. Joining *all* handles before re-raising (rather than propagating on the first failure) means no thread is left running when the panic reaches the caller, and `resume_unwind` keeps the original payload (✓ `"item five failed"`), unlike letting the scope raise its generic "a scoped thread panicked".
- **Bounded queue and two submit modes.** `sync_channel(queue)` bounds memory. `submit` is for callers who want to shed load (`Full` is an ordinary outcome, not a failure); `submit_blocking` is for callers who should be slowed down.
- **Where the lock is released.** `let next = rx.lock()...recv();` ends the statement, and with it the guard's lifetime, before `catch_unwind(job)` runs. The test that catches the mistake proves *overlap*, not speed.
- **Shutdown.** `close()` drops the only `SyncSender`; workers keep taking queued jobs until `recv()` reports disconnect, then return; `shutdown` joins them and reads the counters, and `Drop` does the same so a forgotten `shutdown` can't leak threads (it can block, see Section 9).
- **Construction failure.** The pool value exists before the first thread is spawned, so if spawning fails midway, returning the error drops the half-built pool and its `Drop` shuts down the workers already started.
- **Not covered:** per-job results or futures (return values go through your own channels), priorities, work stealing, timeouts on `shutdown`, resizing, and pinning threads to cores. The doctest sample uses `parallel_map` only.

</details>

---

## 13. Interview questions

<details markdown="1">
<summary>Answers</summary>

1. **How does Rust prevent data races?** Ownership and borrowing (no aliased mutable access) plus `Send`/`Sync` bounds on `spawn` and shared types make data races in safe code a compile error (✓ E0373, E0382, E0499, E0277 above).
2. **What does `thread::spawn` require of its closure?** `FnOnce() -> T + Send + 'static` and `T: Send + 'static`: it may outlive the caller and runs on another thread.
3. **What does `JoinHandle::join` return?** `Result<T, Box<dyn Any + Send>>`: the thread's return value, or the panic payload (✓).
4. **What happens to a thread if its `JoinHandle` is dropped, or `main` returns?** The thread is detached and keeps running; when `main` returns the process exits and kills it (✓ "never printed").
5. **Why do scoped threads exist?** They let threads borrow from the caller's stack, because the scope joins all of them before returning (✓ no `Arc`, no `'static`).
6. **What happens if a scoped thread panics?** The other threads finish; if you didn't `join` the panicking one, the scope panics with "a scoped thread panicked" (✓).
7. **What is `mpsc` and what are its guarantees?** A multi-producer single-consumer FIFO channel. Per-sender order is preserved; cross-sender interleaving is unspecified (✓).
8. **When does `recv` fail and when does `send` fail?** `recv`: channel empty and all senders dropped. `send`: receiver dropped, returning the value (✓).
9. **Why does my `for x in rx` loop never end?** A `Sender` is still alive, often the original one in the creating thread (✓ debugging challenge).
10. **Bounded vs unbounded channels?** Unbounded can grow without limit (memory DoS); bounded gives back-pressure via blocking `send` or failing `try_send` (✓ `Full(v)`).
11. **What is a rendezvous channel?** `sync_channel(0)`: `send` completes only when a receiver takes the value (✓).
12. **Why is `Receiver` not `Sync`?** It is a single consumer; sharing `&Receiver` between threads would allow concurrent `recv` (✓ E0277). Wrap it in a `Mutex` for a pool.
13. **How do you build a thread pool with `std`?** Fixed worker threads sharing `Arc<Mutex<Receiver<Job>>>`, with the lock released before running the job; shutdown by dropping the sender and joining (✓).
14. **What goes wrong with `while let Ok(job) = rx.lock().unwrap().recv() { job() }`?** The guard lives for the loop body, so the lock is held during the job and jobs run one at a time (✓ max concurrency 1).
15. **How do you cancel a thread?** There is no safe forced kill in `std`; use a message, a flag, or closing a channel that the thread checks.
16. **How many threads should I create?** CPU-bound: about `available_parallelism()`; blocking I/O: more, within a cap; many connections: async tasks (Lesson 30).
17. **What safety issues remain with threads in safe Rust?** Deadlocks, ordering and check-then-act logic races, unbounded queues, thread exhaustion, panics cascading, stuck joins.

</details>

---

## 14. Key takeaways

- Rust threads are 1:1 OS threads. `spawn` needs `Send + 'static`: errors E0373 (borrowing locals), E0277 (`Rc`), E0382 (use after `move`) are all ✓ the compiler enforcing it. `join()` returns the value or the panic payload.
- `thread::scope` lets threads borrow local data and guarantees they are joined; the usual borrow rules still forbid two threads mutating the same value (✓ E0499).
- `mpsc` channels move ownership of values: FIFO per sender, interleaving across senders unspecified, disconnect when all `Sender`s are dropped (✓), `send` returns your value on failure (✓). **Always drop your own `Sender`** after cloning (✓ hang).
- Use `sync_channel` for back-pressure; unbounded queues are a memory-exhaustion risk. `Receiver` is `Send` but not `Sync` (✓).
- A shared-receiver pool must release the lock *before* running the job (✓ otherwise concurrency is 1). Test that jobs really overlap.
- Shutdown is a protocol: close the queue, let workers drain it, join them. Catch job panics and count them. Prefer pools and scoped threads to thread-per-task.
- Safe Rust removes data races, not deadlocks or logic races.

**Connections.** Lesson 22's `Send`/`Sync` produce every `E0277` here; Lessons 23 and 24 supplied `Arc` and `Mutex`; Lesson 26's panic isolation and `catch_unwind` are used inside the pool. Next: Lesson 28 (atomics and memory ordering, the counters used in our measurements), Lesson 29 (locks, `Condvar`, deadlocks), Lessons 30 to 32 (`async`, Tokio, cancellation, streams, back-pressure for async), and Lesson 52 (reliability patterns for networked services). See the [roadmap](00_roadmap.md) and the [course index](index.md).

---

## 15. References

- *The Rust Programming Language*, ch. 16 "Fearless Concurrency" (threads, message passing, shared state, `Send` and `Sync`): <https://doc.rust-lang.org/book/ch16-00-concurrency.html>
- `std::thread` (`spawn`, `Builder`, `scope`, `JoinHandle`, `park`, `available_parallelism`, `thread_local!`): <https://doc.rust-lang.org/std/thread/index.html>
- `std::sync::mpsc` (`channel`, `sync_channel`, `Sender`, `SyncSender`, `Receiver`, error types): <https://doc.rust-lang.org/std/sync/mpsc/index.html>
- *The Rustonomicon*, "Concurrency": <https://doc.rust-lang.org/nomicon/concurrency.html>
- Rust error code index: E0373, E0382, E0499, E0277: <https://doc.rust-lang.org/error_codes/E0373.html>

The URLs above are the standard documentation locations; I did not open them during this session (the sandbox cannot reach them), so if a link has moved, search the docs site for the page title. Unchecked from memory: the Book chapter number and anchor; the exact `thread::spawn` signature (consistent with the compiler errors but quoted from memory); that the default thread stack size is platform-dependent (not measured); that `Sender` became `Sync` in a recent release (✓ observed: `assert_sync::<Sender<u8>>()` compiled on this toolchain, but I did not look up which version changed it); that `crossbeam-channel` and `rayon` offer multi-consumer channels and work-stealing parallel iterators (recalled, not used or downloaded here); the cost of thread creation and of channel operations (not measured); the guard-lifetime rule for `while let` scrutinees (✓ observed through the concurrency measurement, with the language rule itself recalled from the reference); and the Lesson 22 to 26 pointers (referenced by title). The concurrency tests rely on timing for failure detection (sleeps and 2 to 5 second timeouts); they passed in every run here but are not formal proofs.
