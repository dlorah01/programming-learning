---
title: "Lesson 24 — Interior mutability: Cell, RefCell, Mutex, RwLock"
---

# Lesson 24 — Interior mutability: `Cell`, `RefCell`, `Mutex`, `RwLock`

**Module 8 · Smart pointers** · Status: generated · Edition used in examples: **2024** · External dependencies: none

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux. As before, ✓ marks outputs, diagnostics and measurements I actually ran; unmarked statements are documented behavior I did not run. Everything is **stable Rust**.

Builds on [Lesson 10](10_references_and_borrowing.md) (shared XOR mutable), [Lesson 22](22_gats_send_sync_phantom.md) (`Send`/`Sync`) and [Lesson 23](23_box_rc_arc.md) (`Rc`/`Arc`, which are read-only until this lesson gives them a way to change).

---

## 1. The problem: mutation through a shared reference

The borrow rules from Lesson 10 say: at any moment you have either many `&T` or one `&mut T`. That rule is checked at **compile time** and it is deliberately conservative: some correct programs cannot be expressed with it.

- A counter or cache that is updated by a method taking `&self`.
- Several `Rc` owners (Lesson 23) that all need to change the same value.
- A value shared across threads that must be updated safely.

**Interior mutability** is the escape hatch: a type that lets you mutate its contents through `&self`, enforcing the aliasing rule by some other means. The standard library offers four main tools, and they differ in *how* they enforce it:

| Type | Enforcement | Thread-safe? | Cost of a violation |
|---|---|---|---|
| `Cell<T>` | never hands out a reference to the inside; you copy/move values in and out | no (`Send`, not `Sync`) | impossible by construction |
| `RefCell<T>` | counts borrows at **run time** | no (`Send`, not `Sync`) | **panic** (or `Err` with `try_borrow*`) |
| `Mutex<T>` | one holder at a time, others **wait** | yes | blocking, possible deadlock, poisoning |
| `RwLock<T>` | many readers or one writer, others wait | yes | blocking, possible deadlock, poisoning |

All of them are built on one compiler primitive, `UnsafeCell<T>`, which is the only legal way to get a `*mut T` out of a `&T`. Using any other route is undefined behavior (documented; not run). You will almost never use `UnsafeCell` directly (Lesson 36), but one consequence is visible even in safe code ✓:

```rust
use std::cell::UnsafeCell;
use std::mem::size_of;
fn main() {
    println!("{} {}", size_of::<UnsafeCell<u64>>(), size_of::<Option<std::cell::Cell<&u8>>>());
    println!("{}", size_of::<std::cell::RefCell<u64>>());
    println!("{}", size_of::<std::sync::Mutex<u64>>());
    println!("{}", size_of::<std::sync::RwLock<u64>>());
}
```

✓ Output `8 16`, `16`, `16`, `24`. `UnsafeCell<u64>` is the same size as a `u64`, but `Option<Cell<&u8>>` is **16** bytes, not the 8 you get from `Option<&u8>` (Lesson 09): `UnsafeCell` opts out of the "null means `None`" niche, because the compiler can no longer assume the value inside is unchanged. The `RefCell`, `Mutex` and `RwLock` sizes include a borrow flag or lock word; they are measurements of this toolchain, not language guarantees.

---

## 2. `Cell<T>`: copy values in and out

`Cell` never lets you borrow its contents. You `get` a copy (if `T: Copy`), `set` a new value, or `replace`, `take`, `swap` whole values ✓:

```rust
use std::cell::Cell;
fn main() {
    let c = Cell::new(5);
    let r1 = &c; let r2 = &c;
    r1.set(r1.get() + 1);
    r2.set(r2.get() * 2);
    println!("{} {}", c.get(), c.replace(100));
    println!("{}", c.take());
    let s = Cell::new(String::from("hi"));
    let old = s.take();
    s.set(String::from("new"));
    println!("{old} {}", s.into_inner());
    println!("{} {}", std::mem::size_of::<Cell<u32>>(), std::mem::size_of::<std::cell::RefCell<u32>>());
}
```

✓ Output `12 12`, `100`, `hi new`, `4 16`. Two shared references (`r1`, `r2`) both mutate `c`, which the borrow checker would never allow with `i32` directly. `Cell<u32>` is 4 bytes (no overhead); `RefCell<u32>` is 16 here, because of its borrow counter. `take` leaves `Default::default()` behind (`0` for the integer, `""` for the string).

`get` needs `T: Copy`, because it duplicates the value; for other types you must use `take`/`replace`/`swap` (you can't copy out of a cell, and you can't look inside without taking) ✓:

```rust
use std::cell::Cell;
fn main() {
    let c = Cell::new(String::from("x"));
    let s: String = c.get();
    println!("{s}");
}
```

✓ `error[E0599]: the method `get` exists for struct `Cell<String>`, but its trait bounds were not satisfied`, note ``String: Copy``.

A typical use is a counter inside a struct that is otherwise used through `&self` ✓:

```rust
use std::cell::Cell;
struct Counter { hits: Cell<u32> }
impl Counter {
    fn lookup(&self, k: u32) -> u32 { self.hits.set(self.hits.get() + 1); k * 2 }
}
fn main() {
    let c = Counter { hits: Cell::new(0) };
    let a = &c; let b = &c;
    a.lookup(1); b.lookup(2); a.lookup(3);
    println!("{}", c.hits.get());
}
```

✓ Output `3`. `Cell` has no runtime cost and no panic, which is why it is the first choice for small `Copy` state (flags, counters, ids). Its limit is that you cannot hold a reference into it.

**Not shareable across threads.** `Cell` and `RefCell` are `Send` (you may move them to another thread) but **not `Sync`** (two threads may not hold `&` to the same one) ✓:

```rust
use std::cell::{Cell, RefCell};
fn needs_sync<T: Sync>() {}
fn needs_send<T: Send>() {}
fn main() {
    needs_send::<Cell<u8>>();
    needs_send::<RefCell<u8>>();
    needs_sync::<Cell<u8>>();
    needs_sync::<RefCell<u8>>();
}
```

✓ Two errors, both `error[E0277]`, on the `Sync` lines: "`Cell<u8>` cannot be shared between threads safely" and the same for `RefCell<u8>`, with a note suggesting `RwLock` or an atomic (`AtomicU8`, Lesson 28). The `Send` lines compile. Without `Sync` the compiler rejects any attempt to share one across threads, which is exactly what makes the non-atomic mutation safe (Lesson 22).

---

## 3. `RefCell<T>`: borrow checking at run time

`RefCell` gives you real references, `Ref<T>` (shared) and `RefMut<T>` (exclusive), and counts them while they exist. The rules are the ones from Lesson 10, enforced when the program runs. Violating them **panics** ✓:

```rust
use std::cell::RefCell;
fn main() {
    let c = RefCell::new(vec![1, 2]);
    {
        let a = c.borrow();
        let b = c.borrow();
        println!("{} {} {:?}", a.len(), b.len(), c.try_borrow_mut().is_err());
    }
    c.borrow_mut().push(3);
    println!("{:?}", c.borrow());
    let r = c.borrow();
    let w = c.try_borrow_mut();
    println!("{}", w.is_err());
    drop(r);
    println!("{:?}", c.try_borrow_mut().is_ok());
}
```

✓ Output `2 2 true`, `[1, 2, 3]`, `true`, `true`. Many `borrow()`s coexist; `try_borrow_mut()` fails (returns `Err`) while any exist and succeeds once they are dropped. The non-`try_` versions panic instead ✓:

```rust
use std::cell::RefCell;
fn main() {
    let c = RefCell::new(1);
    let _a = c.borrow();
    let _b = c.borrow_mut();
}
```

✓ The program compiles and aborts with `thread 'main' panicked at ...: RefCell already borrowed` (exit code 101; borrowing mutably twice gives the same message). Borrowing shared while a mutable borrow is alive says `RefCell already mutably borrowed` ✓.

This is the key trade: the check moved from compile time to run time. The compiler no longer tells you about the bug; a test, or a user, finds it.

### 3.1 Where borrows hide

The easy mistakes are *implicit* borrows that live longer than you expect ✓:

```rust
use std::cell::RefCell;
fn main() {
    let c = RefCell::new(vec![1, 2, 3]);
    let total: i32 = c.borrow().iter().sum();
    c.borrow_mut().push(total);
    println!("{:?}", c.borrow());
    c.borrow_mut().push(c.borrow().len() as i32);
}
```

✓ Output `[1, 2, 3, 6]`, then a panic `RefCell already mutably borrowed` from the last line. The first two lines are fine: the temporary `Ref` from `c.borrow()` dies at the end of its statement. In the last line, `c.borrow_mut()` is evaluated first and its `RefMut` is alive while the argument `c.borrow().len()` runs. Fix: `let n = c.borrow().len() as i32; c.borrow_mut().push(n);`.

Iterating while mutating is the classic case ✓:

```rust
use std::cell::RefCell;
fn main() {
    let v = RefCell::new(vec![1, 2, 3]);
    for x in v.borrow().iter() {
        if *x == 2 { v.borrow_mut().push(4); }
    }
}
```

✓ Compiles, panics with `RefCell already borrowed`: the `Ref` produced by `v.borrow()` lives for the whole loop.

### 3.2 `Rc<RefCell<T>>`: shared and mutable, single thread

Combining Lesson 23 with this one gives shared mutable state ✓:

```rust
use std::cell::RefCell;
use std::rc::Rc;
fn main() {
    let shared = Rc::new(RefCell::new(Vec::<i32>::new()));
    let a = Rc::clone(&shared);
    let b = Rc::clone(&shared);
    a.borrow_mut().push(1);
    b.borrow_mut().push(2);
    println!("{:?} {}", shared.borrow(), Rc::strong_count(&shared));
    let g = shared.borrow();
    let first = g[0];
    drop(g);
    shared.borrow_mut().push(first);
    println!("{:?}", shared);
    let g = shared.borrow();
    shared.borrow_mut().push(9);
    println!("{}", g.len());
}
```

✓ Output `[1, 2] 3`, `RefCell { value: [1, 2, 1] }`, then a panic at `borrow_mut` while `g` is alive. Copy what you need out of the guard, drop it, *then* mutate. This pattern is common in single-threaded UI and tree code, but it makes the borrow structure invisible: a function deep in the call stack may `borrow_mut()` the same cell as its caller. When you can, prefer passing `&mut T` down the call stack.

### 3.3 `Ref::map`, `replace`, `take`, `get_mut`

`Ref::map` narrows a guard to a part of the data without ending the borrow; you can return it from a method ✓:

```rust
use std::cell::{Ref, RefCell};
struct Db { rows: RefCell<Vec<String>> }
impl Db {
    fn first(&self) -> Ref<'_, String> { Ref::map(self.rows.borrow(), |v| &v[0]) }
}
fn main() {
    let db = Db { rows: RefCell::new(vec!["a".into(), "b".into()]) };
    {
        let f = db.first();
        println!("{} {}", *f, db.rows.try_borrow_mut().is_err());
    }
    println!("{}", db.rows.try_borrow_mut().is_ok());
    let _g = db.rows.borrow();
    match db.rows.try_borrow_mut() { Err(e) => println!("{e}"), Ok(_) => {} }
}
```

✓ Output `a true`, `true`, `RefCell already borrowed` (the `Display` of the `BorrowMutError`). While the returned `Ref` exists the cell stays borrowed; callers of `first()` hold the borrow whether they know it or not. Other operations that take the whole value (✓ run):

```rust
use std::cell::RefCell;
fn main() {
    let c = RefCell::new(5);
    let old = c.replace(6);
    let swapped = c.replace_with(|&mut o| o * 10);
    println!("{old} {swapped} {}", c.borrow());
    let mut c2 = RefCell::new(String::from("a"));
    c2.get_mut().push('b');
    println!("{}", c2.into_inner());
    let a = RefCell::new(1); let b = RefCell::new(2);
    a.swap(&b);
    println!("{} {}", a.borrow(), b.borrow());
    let x = RefCell::new(vec![1]);
    let y = x.take();
    println!("{:?} {:?}", y, x.borrow());
}
```

✓ Output `5 6 60`, `ab`, `2 1`, `[1] []`. Note `replace_with` returns the *previous* value (6) while storing 60. `get_mut` takes `&mut self`, so the borrow checker already proves exclusivity and there is no runtime check (and no panic). Whenever you own the `RefCell` mutably, use it.

### 3.4 Callbacks and re-entrancy

Observer-style code is where `RefCell` panics reach production: a callback runs while the container is still borrowed, and it touches the container ✓:

```rust
use std::cell::RefCell;
// observer: callbacks stored in RefCell<Vec<Box<dyn Fn>>>
struct Bus { subs: RefCell<Vec<Box<dyn Fn(i32)>>> }
impl Bus {
    fn emit(&self, v: i32) { for f in self.subs.borrow().iter() { f(v); } }
    fn subscribe(&self, f: Box<dyn Fn(i32)>) { self.subs.borrow_mut().push(f); }
}
fn main() {
    let bus = std::rc::Rc::new(Bus { subs: RefCell::new(vec![]) });
    let log = std::rc::Rc::new(RefCell::new(Vec::new()));
    let l = std::rc::Rc::clone(&log);
    bus.subscribe(Box::new(move |v| l.borrow_mut().push(v)));
    bus.emit(1); bus.emit(2);
    println!("{:?}", log.borrow());
    let b2 = std::rc::Rc::clone(&bus);
    bus.subscribe(Box::new(move |_| b2.subscribe(Box::new(|_| {}))));
    bus.emit(3);
}
```

✓ Prints `[1, 2]`, then panics `RefCell already borrowed` on the last `emit`: the loop holds a `Ref` of `subs` while a callback calls `subscribe`, which needs `borrow_mut`. The debugging challenge below fixes this, and the practical challenge's `EventBus` is the robust version.

---

## 4. One-time initialization: `OnceCell`, `OnceLock`, `LazyLock`

Some values are set once and then only read. `OnceCell<T>` (single thread) and `OnceLock<T>` (thread-safe, usable in a `static`) are interior mutability with a "write at most once" rule; `LazyLock<T>` runs an initializer on first access ✓:

```rust
use std::cell::{OnceCell, RefCell};
use std::sync::{OnceLock, LazyLock};
use std::collections::HashMap;
static CFG: OnceLock<String> = OnceLock::new();
static TABLE: LazyLock<HashMap<u8, &'static str>> = LazyLock::new(|| { println!("init table"); HashMap::from([(1, "one"), (2, "two")]) });
fn main() {
    let c: OnceCell<u32> = OnceCell::new();
    println!("{:?}", c.get());
    println!("{}", c.get_or_init(|| { println!("computing"); 42 }));
    println!("{}", c.get_or_init(|| 7));
    println!("{:?}", c.set(9));
    println!("{:?}", CFG.set("a".into()));
    println!("{:?}", CFG.set("b".into()));
    println!("{:?}", CFG.get());
    println!("before table");
    println!("{:?} {:?}", TABLE.get(&1), TABLE.get(&3));
    let memo: RefCell<HashMap<u64, u64>> = RefCell::new(HashMap::new());
    fn fib(n: u64, memo: &RefCell<HashMap<u64, u64>>) -> u64 {
        if n < 2 { return n; }
        if let Some(&v) = memo.borrow().get(&n) { return v; }
        let v = fib(n - 1, memo) + fib(n - 2, memo);
        memo.borrow_mut().insert(n, v);
        v
    }
    println!("{}", fib(50, &memo));
}
```

✓ Output:

```text
None
computing
42
42
Err(9)
Ok(())
Err("b")
Some("a")
before table
init table
Some("one") None
12586269025
```

`get_or_init` runs its closure once (`computing` printed once, the second closure never ran); `set` after initialization returns `Err(value)` handing your value back; the `LazyLock` initializer ran at the first access, after `before table`. The `fib` memo works because each borrow is a short temporary: `memo.borrow().get(&n)` ends at the end of the `if let` body's scrutinee before the recursive call, and `borrow_mut` happens after both recursive calls returned. If you wrote `let guard = memo.borrow_mut();` at the top of `fib`, the recursion would panic. These are `static`-friendly replacements for the old `static mut` patterns (Lesson 36 explains why `static mut` is discouraged).

---

## 5. `Mutex<T>`: one at a time, across threads

`Mutex` makes the aliasing rule hold across threads by **blocking**: `lock()` returns a `MutexGuard` that gives `&mut T` access and unlocks when dropped (RAII, Lesson 08). The data lives *inside* the mutex, so you cannot touch it without holding the lock ✓:

```rust
use std::sync::{Arc, Mutex, RwLock};
use std::thread;
fn main() {
    let m = Arc::new(Mutex::new(Vec::new()));
    let hs: Vec<_> = (0..4).map(|i| { let m = Arc::clone(&m); thread::spawn(move || { m.lock().unwrap().push(i); }) }).collect();
    for h in hs { h.join().unwrap(); }
    let mut v = m.lock().unwrap().clone(); v.sort();
    println!("{v:?}");
    let rw = RwLock::new(5);
    { let r1 = rw.read().unwrap(); let r2 = rw.read().unwrap(); println!("{} {} {}", *r1, *r2, rw.try_write().is_err()); }
    *rw.write().unwrap() += 1;
    println!("{:?}", rw.read().unwrap());
    let m2 = Mutex::new(0);
    let g = m2.lock().unwrap();
    println!("{}", m2.try_lock().is_err());
    drop(g);
    println!("{}", m2.try_lock().is_ok());
    let mut m3 = Mutex::new(7);
    *m3.get_mut().unwrap() += 1;
    println!("{}", m3.into_inner().unwrap());
}
```

✓ Output `[0, 1, 2, 3]`, `5 5 true`, `6`, `true`, `true`, `8`. (The vector order was sorted for determinism; the order in which threads push is not.) `lock()` returns a `Result` (explained below). `try_lock()` returns immediately with an error if the lock is held. `get_mut()` and `into_inner()` need no locking because `&mut`/ownership already prove exclusivity.

### 5.1 Scoped threads and why `Arc` is sometimes unnecessary

With `thread::scope` (Lesson 22) the threads borrow the mutex directly, no `Arc` needed ✓:

```rust
use std::sync::Mutex;
use std::thread;
fn main() {
    let m = Mutex::new(0);
    thread::scope(|s| {
        for _ in 0..4 { s.spawn(|| { for _ in 0..1000 { *m.lock().unwrap() += 1; } }); }
    });
    println!("{}", m.lock().unwrap());
}
```

✓ Output `4000`. Without `scope`, a plain `thread::spawn` that borrows `m` is rejected ✓: `error[E0373]: closure may outlive the current function, but it borrows `m``, with the help to use `move` (and `m` then needs to be an `Arc`). And an `Rc<Mutex<_>>` can't be moved into a thread at all ✓: `error[E0277]: `Rc<std::sync::Mutex<i32>>` cannot be sent between threads safely`.

### 5.2 Poisoning

If a thread panics **while holding** the lock, the mutex is marked *poisoned*. Later `lock()` calls return `Err(PoisonError)`, because the protected data may be half-updated. The `Err` still contains the guard, so you decide ✓:

```rust
use std::sync::{Arc, Mutex};
use std::thread;
fn main() {
    let m = Arc::new(Mutex::new(vec![1, 2, 3]));
    let m2 = Arc::clone(&m);
    let r = thread::spawn(move || {
        let mut g = m2.lock().unwrap();
        g.push(4);
        panic!("boom");
    }).join();
    println!("{}", r.is_err());
    println!("{}", m.is_poisoned());
    match m.lock() {
        Ok(_) => println!("ok"),
        Err(e) => { let g = e.into_inner(); println!("recovered {:?}", *g); }
    }
    m.clear_poison();
    println!("{} {:?}", m.is_poisoned(), m.lock().unwrap());
}
```

✓ Output (after the panic message of the child thread) `true`, `true`, `recovered [1, 2, 3, 4]`, `false [1, 2, 3, 4]`. The push happened before the panic, so the data is intact and we can choose to recover with `PoisonError::into_inner` (`m.lock().unwrap_or_else(PoisonError::into_inner)`) ✓, or `clear_poison` to reset the flag. The `Display` text of a poison error is `poisoned lock: another task failed inside` ✓. Poisoning is a **hint** that an invariant may be broken, not a proof either way: whether it is safe to continue depends on what the panicking code was doing. `.lock().unwrap()` is the common idiom because it turns a poisoned lock into a second panic, which is the conservative choice; recovering is right only when updates cannot leave the data inconsistent (the practical challenge's `SharedLog::push` is an example).

### 5.3 Deadlock and guard lifetimes

Locking a `Mutex` twice on one thread is a bug: the documentation says the call may deadlock or panic, and on this toolchain the program simply **hung** ✓ (a program with `let a = m.lock().unwrap(); let b = m.lock().unwrap();` did not finish; I killed it with `timeout` after 5 seconds, exit code 124). `try_lock` shows the same state without hanging ✓:

```rust
use std::sync::Mutex;
fn main() {
    let m = Mutex::new(1);
    let _g1 = m.lock().unwrap();
    println!("trying second lock on same thread");
    let _g2 = m.try_lock();
    println!("{}", _g2.is_err());
}
```

✓ Output `trying second lock on same thread`, `true`. Lock ordering and deadlock avoidance across threads are Lesson 29.

When is a temporary guard released? Rust drops temporaries at the end of the enclosing statement, and a `match` scrutinee's temporaries live to the end of the whole `match` ✓:

```rust
use std::sync::Mutex;
fn main() {
    let m = Mutex::new(vec![1, 2, 3]);
    // temporary guard lives to end of the whole statement in a match scrutinee
    match m.lock().unwrap().first() {
        Some(x) => println!("first {x} locked={}", m.try_lock().is_err()),
        None => {}
    }
    // guard dropped at end of let statement
    let n = m.lock().unwrap().len();
    println!("{n} locked={}", m.try_lock().is_err());
    // if-let: the guard is also alive inside the body
    if let Some(x) = m.lock().unwrap().first().copied() {
        println!("in if-let {x} locked={}", m.try_lock().is_err());
    } else {
        println!("else locked={}", m.try_lock().is_err());
    }
}
```

✓ Output `first 1 locked=true`, `3 locked=false`, `in if-let 1 locked=true`. Inside the `match` arm and the `if let` body the lock is **still held**; after a plain `let n = ...len();` it is released. Edition 2024 changed when `if let` temporaries are dropped relative to the `else` branch (documented; this program does not test the `else` path), so don't rely on edge-case timing: bind what you need to a local, end the statement, and call other code afterward. Holding a guard across a slow call or a callback is the most common cause of lock contention and deadlock.

**`MutexGuard` is `!Send`** on this platform ✓: moving a guard to another thread (`s.spawn(move || { let _g = g; })`) gives `error[E0277]: `MutexGuard<'_, i32>` cannot be sent between threads safely`. The reason is that some operating systems require the thread that locked a mutex to be the one that unlocks it (documented).

---

## 6. `RwLock<T>`: many readers or one writer

`RwLock` allows any number of `read()` guards or exactly one `write()` guard (✓ `5 5 true` above: two readers coexist, `try_write` fails). It helps when reads are frequent and long and writes are rare. Be careful:

- Its fairness policy (does a waiting writer block new readers?) depends on the OS and is not specified; don't write code that relies on it.
- A thread that holds a read guard and then asks for a write lock on the same lock can deadlock (documented).
- It needs more bookkeeping than `Mutex` (✓ 24 bytes vs 16 for a `u64` here), and is not always faster; measure (Lesson 41) before choosing it.

### 6.1 `Send` and `Sync` of the locks

A lock makes a non-`Sync` type shareable *only if exclusive access suffices*. `Mutex<T>` is `Sync` when `T: Send`; `RwLock<T>` is `Sync` only when `T: Send + Sync`, because many readers see `&T` at once ✓:

```rust
use std::sync::{Mutex, RwLock};
use std::cell::Cell;
fn s<T: Sync>() {}
fn p<T: Send>() {}
fn main() {
    s::<Mutex<Cell<u8>>>();
    p::<Mutex<Cell<u8>>>();
    s::<RwLock<Cell<u8>>>();
}
```

✓ One error, on the last line only: "`Cell<u8>` cannot be shared between threads safely ... required for `RwLock<Cell<u8>>` to implement `Sync`". And a `Mutex` cannot launder a non-`Send` value: `p::<Mutex<Rc<u8>>>()` gives `error[E0277]: `Rc<u8>` cannot be sent between threads safely`, "required for `Mutex<Rc<u8>>` to implement `Send`" ✓.

---

## 7. Choosing

| Need | Use |
|---|---|
| Small `Copy` value changed via `&self`, one thread | `Cell<T>` |
| Non-`Copy` value changed via `&self`, one thread; you need real references | `RefCell<T>` (and keep borrows short) |
| Shared and mutable, one thread | `Rc<RefCell<T>>` (Lesson 23) |
| Shared and mutable, several threads | `Arc<Mutex<T>>` or `Arc<RwLock<T>>` |
| Single integer/bool flag shared between threads | atomics (Lesson 28) |
| Set once, read many | `OnceCell` / `OnceLock` / `LazyLock` |
| Hand work between threads | channels (Lesson 27) rather than shared state |
| You hold `&mut` anyway | `get_mut` / plain `&mut`: no interior mutability needed |

A design rule: prefer **ownership and `&mut`** first. Interior mutability is for the cases where the structure of the program really does have shared access. Each use should have a one-sentence justification.

---

## 8. Common mistakes and edge cases

- **Holding a `Ref`/`RefMut` across a call that may borrow the same cell** (✓ panics): copy the data out, drop the guard, then call.
- **Hidden temporaries** (`c.borrow_mut().push(c.borrow().len())` ✓ panics) and `for x in c.borrow().iter()` while mutating inside (✓ panics).
- **Callbacks under a borrow or lock** (§3.4): snapshot the list first (practical challenge).
- **Locking twice on one thread** (✓ hangs).
- **Holding a lock across slow work** (I/O, sleeping, another lock): contention and deadlock. Compute first, lock briefly.
- **`.lock().unwrap()` everywhere without thinking about poisoning**: a panic in one worker then panics every other thread that touches the lock. Decide per lock whether to propagate or recover.
- **Using `Mutex` to guard something that should be an atomic** (counters, flags): more code, more risk, no benefit.
- **`Rc<RefCell<_>>` cycles** leak (Lesson 23) and `Weak` is the cure (Lesson 25).
- **`RefCell` in a type that must be `Sync`**: not possible; use `Mutex`/`RwLock`.
- **Mutating through `&self` when `&mut self` would do**: if you can change the signature, do.
- **Thinking `Cell`/`RefCell` make code "safe to share"**: they don't; they make it single-threaded by `!Sync`.

---

## 9. Performance implications

- `Cell` has no overhead (✓ `Cell<u32>` is 4 bytes). `RefCell` adds a borrow counter and a check on each `borrow`/`borrow_mut` (✓ 16 bytes for a `u32`); the check is cheap compared with most work, but it is a branch and a panic path in hot loops.
- `Mutex`/`RwLock` cost a word or two (✓ 16 and 24 bytes for a `u64` here), an atomic operation on lock/unlock, and, under contention, a system call and a context switch. I did not benchmark any of them: contention behavior depends on the OS, the core count, and the hold time (Lesson 41).
- Shorter critical sections matter more than the choice of lock. Shard (`Vec<Mutex<_>>`) or use message passing before reaching for something exotic.
- A `static` `OnceLock`/`LazyLock` costs an atomic read on each access after initialization (documented); cache the reference if it's in a tight loop.

---

## 10. Security and reliability

- **Runtime panics are reachable.** A `RefCell` double-borrow depends on call order, which may depend on input (event sequences, message order). In a long-running service a panic is a denial of service. Library code that cannot prove the borrow structure should use `try_borrow*` and return an error.
- **Poisoned locks can cascade.** With `.lock().unwrap()`, one panic poisons the lock and every later user panics. For caches and logs, recover; for invariant-heavy state, fail loudly or rebuild the state.
- **Poison recovery can expose half-updated state.** The data you get from `into_inner()` may violate your invariants (e.g. a balance was debited but not yet credited). Never recover blindly in code that moves money or handles authorization.
- **Deadlocks are availability bugs.** Holding a lock while calling user code, a callback, or another lock invites them (Lesson 29). A hung thread holding a lock can freeze a whole service. Use `try_lock` or timeouts where the design allows.
- **Race windows in "check then act".** `if cache.read()... .is_none() { cache.write()... insert }` leaves a gap; use `entry`/`get_or_insert`-style APIs under one write guard, or accept duplicated work, as `RwCache` does deliberately (it never holds a lock while computing).
- **Don't hold locks across `.await`** (Lesson 30–31): a blocking guard in an async task can stall the executor thread; async code has its own mutex types.
- **Interior mutability is not a safety boundary.** `Mutex<Vec<Secret>>` still leaves secrets in memory after drop; zeroization is separate (Lesson 48).

---

## 11. Exercises

Try each before opening the answer. They are not marked complete unless you tell me you did them.

1. **(Easy)** What does this print? `let c = Cell::new(3); let v = c.replace(4); let w = c.get(); c.swap(&Cell::new(10)); println!("{v} {w} {}", c.get());`
2. **(Easy)** Why does `Cell<String>` have no `get`, and how do you read the string?
3. **(Medium)** `c.borrow_mut().push(c.borrow().len() as i32)` panics. Why, and how do you fix it?
4. **(Medium)** Which of `Cell<u8>`, `RefCell<u8>`, `Mutex<Cell<u8>>`, `RwLock<Cell<u8>>` are `Send`? Which are `Sync`?
5. **(Medium)** A worker panics while holding a `Mutex<Vec<i32>>`. What happens to the next `lock()`, and how can you continue?
6. **(Hard)** Write `fn first_word(db: &RefCell<Vec<String>>) -> Ref<'_, str>` returning the first word of the first row without cloning.
7. **(Hard)** Why does the `fib` memo with `RefCell<HashMap>` work, and what single-line change breaks it?
8. **(Hard)** Describe two ways to make an observer list safe against callbacks that subscribe during `emit`.

<details markdown="1">
<summary>Answers</summary>

1. ✓ `3 4 10`. `replace(4)` returns the old value 3; `get()` is 4; the swap stores 10.
2. `get` duplicates the value, which needs `T: Copy`; `String` isn't (✓ `E0599 ... trait bounds were not satisfied`, `String: Copy`). To read it you must take it (`let s = c.take(); ...; c.set(s)`), `replace`, or use `RefCell` instead if you need to look at it without moving it.
3. ✓ `borrow_mut()` is evaluated first and its `RefMut` stays alive while `borrow()` runs, so the shared borrow panics (`RefCell already mutably borrowed`). Compute the argument first: `let n = c.borrow().len() as i32; c.borrow_mut().push(n);`.
4. `Cell<u8>`: Send, not Sync. `RefCell<u8>`: Send, not Sync (✓ §2). `Mutex<Cell<u8>>`: Send and Sync (✓ the `Mutex` lines compile). `RwLock<Cell<u8>>`: Send, **not** Sync (✓ the error is on the `RwLock` line; `RwLock<T>: Sync` needs `T: Send + Sync`). I ran the `Sync` check for `Mutex<Cell<u8>>` and `RwLock<Cell<u8>>` and the `Send` check for `Mutex<Cell<u8>>`; `RwLock<Cell<u8>>: Send` follows from the documented `T: Send` rule and was not separately run.
5. ✓ The mutex is poisoned: `lock()` returns `Err(PoisonError)`; `.unwrap()` would panic the new thread. To continue, take the guard from the error (`e.into_inner()` or `.unwrap_or_else(PoisonError::into_inner)`) after deciding the data is acceptable, and optionally `clear_poison()`.
6. One possibility: `fn first_word(db: &RefCell<Vec<String>>) -> Ref<'_, str> { Ref::map(db.borrow(), |v| v[0].split_whitespace().next().unwrap_or("")) }`. `Ref::map` keeps the borrow alive and narrows the reference (✓ compiled and run: with a row `"hello world"` it returned `hello`, and `try_borrow_mut` failed while the `Ref` was alive; that `Ref::map` accepts unsized targets is why `Ref<'_, str>` works). The caller holds the cell borrowed for as long as it holds the `Ref`. It panics if the vector is empty (`v[0]`); use `get(0)` if that's possible.
7. Each borrow is a short temporary: `memo.borrow().get(&n)` ends before the recursive calls, and `borrow_mut()` happens after they return. Adding `let _g = memo.borrow_mut();` at the top of `fib` makes every recursive call try to borrow a cell that is already mutably borrowed, so it panics (the practical challenge's `Memo` mutation test does the equivalent ✓).
8. (a) Snapshot: clone the list of `Rc<dyn Fn>` under a short borrow, release the borrow, then call (what `EventBus` does ✓). (b) Queue: callbacks that want to subscribe push into a separate "pending" `RefCell<Vec<_>>` that `emit` merges after the loop. A third option is to document and forbid re-entrancy and use `try_borrow_mut` to return an error rather than panic.

</details>

---

## 12. Debugging challenge

This event bus works until a subscriber adds another subscriber:

```rust
use std::cell::RefCell;
use std::rc::Rc;
struct Bus { subs: RefCell<Vec<Box<dyn Fn(i32)>>> }
impl Bus {
    fn new() -> Self { Bus { subs: RefCell::new(Vec::new()) } }
    fn subscribe(&self, f: Box<dyn Fn(i32)>) { self.subs.borrow_mut().push(f); }
    fn emit(&self, v: i32) { for f in self.subs.borrow().iter() { f(v); } }
}
fn main() {
    let bus = Rc::new(Bus::new());
    let b = Rc::clone(&bus);
    bus.subscribe(Box::new(move |v| {
        if v == 1 { b.subscribe(Box::new(|v| println!("late subscriber got {v}"))); }
    }));
    bus.emit(1);
    bus.emit(2);
}
```

It compiles but panics. Explain why, fix it so that the output is `late subscriber got 2`, and say what the fix changes about *when* the new subscriber is first called.

<details markdown="1">
<summary>Solution</summary>

✓ The program compiles and panics at the `borrow_mut()` inside `subscribe` with `RefCell already borrowed` (exit 101). `emit` iterates `self.subs.borrow()`; the first callback runs *while that `Ref` is alive* and calls `bus.subscribe`, which needs `self.subs.borrow_mut()`. The borrow checker can't see this: both calls go through `&Bus`.

**Fix.** Make callbacks cheaply cloneable (`Rc<dyn Fn>`), clone the list under a short borrow, release the borrow, then call:

```rust
use std::cell::RefCell;
use std::rc::Rc;
struct Bus { subs: RefCell<Vec<Rc<dyn Fn(i32)>>> }
impl Bus {
    fn new() -> Self { Bus { subs: RefCell::new(Vec::new()) } }
    fn subscribe(&self, f: Rc<dyn Fn(i32)>) { self.subs.borrow_mut().push(f); }
    fn emit(&self, v: i32) {
        let snapshot: Vec<Rc<dyn Fn(i32)>> = self.subs.borrow().clone();
        for f in snapshot { f(v); }
    }
}
fn main() {
    let bus = Rc::new(Bus::new());
    let b = Rc::clone(&bus);
    bus.subscribe(Rc::new(move |v| {
        if v == 1 { b.subscribe(Rc::new(|v| println!("late subscriber got {v}"))); }
    }));
    bus.emit(1);
    bus.emit(2);
}
```

✓ Output `late subscriber got 2`. The snapshot is a `Vec` of `Rc` clones (counter bumps, no callback copies), and the `Ref` temporary ends at the semicolon, so callbacks run with the cell unborrowed. Semantics change: a subscriber added during an `emit` is **not** called for that event (it was not in the snapshot) and is first called by the next one; that is why the output has no line for `emit(1)`. That's usually the behavior you want, but decide it on purpose and document it. A subscriber that was removed during the emit still runs once for that event, for the same reason.

Note the cycle risk: the first callback captures an `Rc<Bus>` (`b`) and the bus owns the callback, so the bus is never freed (Lesson 23). Capture a `Weak<Bus>` (Lesson 25) in real code.

</details>

---

## 13. Practical challenge: `cellab`

Create a library crate `cellab` (edition 2024) with four components, each demonstrating one rule from this lesson:

1. `Memo<K, V>`: a `RefCell<HashMap>` memoizer. `get(&self, key, f: impl FnOnce(&Self) -> V) -> V` must let `f` call `get` again (recursion), so it must not hold a borrow while `f` runs. Track how many values were actually computed with a `Cell<usize>`.
2. `EventBus<T>`: single-threaded observers: `subscribe -> SubId`, `unsubscribe(SubId) -> bool`, `emit(&T)`. Callbacks may subscribe, unsubscribe themselves, and emit recursively.
3. `SharedLog<T>`: a cloneable `Arc<Mutex<Vec<T>>>` handle that keeps working after a panic poisoned the lock (`push`, `len`, `snapshot`, `with`, `is_poisoned`).
4. `RwCache<K, V>`: a thread-safe get-or-compute cache that reads under a shared lock, writes under an exclusive lock, and **never holds a lock while the compute function runs**. Racing threads must agree on one stored value.
5. Tests for every behavior above (including re-entrancy, poison recovery, 8 threads pushing, 50 rounds of racing threads); a `compile_fail` doctest for `Memo: !Sync` and one for `EventBus: !Send`; a usage doctest.
6. It must pass `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, and `cargo test --locked` (run `cargo generate-lockfile` once first).

<details markdown="1">
<summary>Solution</summary>

**`src/lib.rs`**

```rust
//! `cellab`: interior mutability done carefully.
//!
//! * [`Memo`]: single-threaded memoization with `RefCell`; never holds a
//!   borrow while the user's function runs, so recursive functions work.
//! * [`EventBus`]: single-threaded observers; `emit` takes a snapshot of the
//!   subscribers first, so a callback may subscribe, unsubscribe or emit.
//! * [`SharedLog`]: `Arc<Mutex<Vec<T>>>` that recovers from poisoning.
//! * [`RwCache`]: multi-threaded get-or-compute cache; never holds a lock
//!   while the user's function runs.

use std::cell::{Cell, RefCell};
use std::collections::HashMap;
use std::hash::Hash;
use std::rc::Rc;
use std::sync::{Arc, Mutex, MutexGuard, PoisonError, RwLock};

/// Memoizes `f(k)`. `get` takes `&self`, so a closure can call back into it.
///
/// `Memo` is `Send` (a `RefCell` can move between threads) but not `Sync`
/// (it cannot be shared between threads):
///
/// ```compile_fail
/// fn needs_sync<T: Sync>(_: &T) {}
/// needs_sync(&cellab::Memo::<u8, u8>::new());
/// ```
///
/// Usage:
///
/// ```
/// let m = cellab::Memo::new();
/// assert_eq!(m.get(2u8, |_| 4u8), 4);
/// assert_eq!(m.computed(), 1);
/// ```
pub struct Memo<K, V> {
    map: RefCell<HashMap<K, V>>,
    computed: Cell<usize>,
}

impl<K: Hash + Eq + Clone, V: Clone> Memo<K, V> {
    pub fn new() -> Self {
        Self {
            map: RefCell::new(HashMap::new()),
            computed: Cell::new(0),
        }
    }

    /// Returns the cached value, or computes it with `f`.
    ///
    /// The `RefCell` borrow is released before `f` runs and re-taken to
    /// insert, so `f` may call `get` again (recursion) without panicking.
    pub fn get(&self, key: K, f: impl FnOnce(&Self) -> V) -> V {
        if let Some(v) = self.map.borrow().get(&key) {
            return v.clone();
        }
        let v = f(self);
        self.computed.set(self.computed.get() + 1);
        self.map.borrow_mut().insert(key, v.clone());
        v
    }

    /// How many times a value was actually computed.
    pub fn computed(&self) -> usize {
        self.computed.get()
    }

    pub fn len(&self) -> usize {
        self.map.borrow().len()
    }

    pub fn is_empty(&self) -> bool {
        self.len() == 0
    }
}

impl<K: Hash + Eq + Clone, V: Clone> Default for Memo<K, V> {
    fn default() -> Self {
        Self::new()
    }
}

/// Identifies a subscription so it can be removed.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct SubId(u64);

type Callback<T> = Rc<dyn Fn(&T)>;

/// A single-threaded event bus whose callbacks may re-enter the bus.
///
/// It stores `Rc` callbacks, so it is neither `Send` nor `Sync`:
///
/// ```compile_fail
/// fn needs_send<T: Send>(_: T) {}
/// needs_send(cellab::EventBus::<u8>::new());
/// ```
pub struct EventBus<T> {
    subs: RefCell<Vec<(SubId, Callback<T>)>>,
    next_id: Cell<u64>,
}

impl<T> EventBus<T> {
    pub fn new() -> Self {
        Self {
            subs: RefCell::new(Vec::new()),
            next_id: Cell::new(0),
        }
    }

    pub fn subscribe(&self, f: impl Fn(&T) + 'static) -> SubId {
        let id = SubId(self.next_id.get());
        self.next_id.set(id.0 + 1);
        self.subs.borrow_mut().push((id, Rc::new(f)));
        id
    }

    /// Returns `true` if the subscription existed.
    pub fn unsubscribe(&self, id: SubId) -> bool {
        let mut subs = self.subs.borrow_mut();
        let before = subs.len();
        subs.retain(|(i, _)| *i != id);
        subs.len() != before
    }

    pub fn len(&self) -> usize {
        self.subs.borrow().len()
    }

    pub fn is_empty(&self) -> bool {
        self.len() == 0
    }

    /// Calls every subscriber that exists *when emit starts*.
    ///
    /// The list is cloned (cheap `Rc` clones) and the borrow released before
    /// any callback runs, so callbacks can subscribe, unsubscribe or emit.
    pub fn emit(&self, event: &T) {
        let snapshot: Vec<Callback<T>> = self
            .subs
            .borrow()
            .iter()
            .map(|(_, f)| Rc::clone(f))
            .collect();
        for f in snapshot {
            f(event);
        }
    }
}

impl<T> Default for EventBus<T> {
    fn default() -> Self {
        Self::new()
    }
}

/// A cloneable, thread-safe log. A panic while holding the lock poisons a
/// `Mutex`; this log keeps working because appending cannot leave the `Vec`
/// in a broken state.
pub struct SharedLog<T> {
    inner: Arc<Mutex<Vec<T>>>,
}

impl<T> Clone for SharedLog<T> {
    fn clone(&self) -> Self {
        Self {
            inner: Arc::clone(&self.inner),
        }
    }
}

impl<T> SharedLog<T> {
    pub fn new() -> Self {
        Self {
            inner: Arc::new(Mutex::new(Vec::new())),
        }
    }

    fn lock(&self) -> MutexGuard<'_, Vec<T>> {
        self.inner.lock().unwrap_or_else(PoisonError::into_inner)
    }

    pub fn push(&self, v: T) {
        self.lock().push(v);
    }

    pub fn len(&self) -> usize {
        self.lock().len()
    }

    pub fn is_empty(&self) -> bool {
        self.len() == 0
    }

    pub fn is_poisoned(&self) -> bool {
        self.inner.is_poisoned()
    }

    /// Runs `f` with the lock held. Keep `f` short and never call back into
    /// this log from inside it (that would deadlock or panic).
    pub fn with<R>(&self, f: impl FnOnce(&mut Vec<T>) -> R) -> R {
        f(&mut self.lock())
    }
}

impl<T: Clone> SharedLog<T> {
    pub fn snapshot(&self) -> Vec<T> {
        self.lock().clone()
    }
}

impl<T> Default for SharedLog<T> {
    fn default() -> Self {
        Self::new()
    }
}

/// A thread-safe get-or-compute cache. Readers share the lock; the compute
/// function runs with **no lock held**, so it may be slow or re-enter the
/// cache. Two threads may race and both compute; the first insert wins and
/// both get the same stored value.
pub struct RwCache<K, V> {
    map: RwLock<HashMap<K, V>>,
}

impl<K: Hash + Eq, V: Clone> RwCache<K, V> {
    pub fn new() -> Self {
        Self {
            map: RwLock::new(HashMap::new()),
        }
    }

    pub fn get_or_compute(&self, key: K, f: impl FnOnce() -> V) -> V {
        if let Some(v) = self
            .map
            .read()
            .unwrap_or_else(PoisonError::into_inner)
            .get(&key)
        {
            return v.clone();
        }
        let fresh = f();
        let mut w = self.map.write().unwrap_or_else(PoisonError::into_inner);
        w.entry(key).or_insert(fresh).clone()
    }

    pub fn len(&self) -> usize {
        self.map
            .read()
            .unwrap_or_else(PoisonError::into_inner)
            .len()
    }

    pub fn is_empty(&self) -> bool {
        self.len() == 0
    }
}

impl<K: Hash + Eq, V: Clone> Default for RwCache<K, V> {
    fn default() -> Self {
        Self::new()
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::sync::atomic::{AtomicUsize, Ordering};
    use std::thread;

    fn fib(n: u64, m: &Memo<u64, u64>) -> u64 {
        if n < 2 {
            return n;
        }
        m.get(n, |m| fib(n - 1, m) + fib(n - 2, m))
    }

    #[test]
    fn memo_supports_recursion_and_computes_once() {
        let m = Memo::new();
        assert_eq!(fib(50, &m), 12_586_269_025);
        assert_eq!(m.computed(), 49); // n = 2..=50
        assert_eq!(m.len(), 49);
        assert_eq!(fib(50, &m), 12_586_269_025);
        assert_eq!(m.computed(), 49);
    }

    #[test]
    fn event_bus_delivers_in_order_and_unsubscribes() {
        let bus = EventBus::new();
        let log = Rc::new(RefCell::new(Vec::new()));
        let (l1, l2) = (Rc::clone(&log), Rc::clone(&log));
        let a = bus.subscribe(move |v: &i32| l1.borrow_mut().push(("a", *v)));
        bus.subscribe(move |v: &i32| l2.borrow_mut().push(("b", *v)));
        bus.emit(&1);
        assert!(bus.unsubscribe(a));
        assert!(!bus.unsubscribe(a));
        bus.emit(&2);
        assert_eq!(*log.borrow(), vec![("a", 1), ("b", 1), ("b", 2)]);
        assert_eq!(bus.len(), 1);
    }

    #[test]
    fn callbacks_may_reenter_the_bus() {
        let bus = Rc::new(EventBus::new());
        let hits = Rc::new(Cell::new(0));
        let (b, h) = (Rc::clone(&bus), Rc::clone(&hits));
        // Subscribes another listener while an emit is in progress.
        bus.subscribe(move |_: &u8| {
            let h = Rc::clone(&h);
            b.subscribe(move |_| h.set(h.get() + 1));
        });
        bus.emit(&0); // new subscriber is NOT called during this emit
        assert_eq!(hits.get(), 0);
        assert_eq!(bus.len(), 2);
        bus.emit(&0); // now it is (and one more is added by the first)
        assert_eq!(hits.get(), 1);
        assert_eq!(bus.len(), 3);
    }

    #[test]
    fn callback_may_unsubscribe_itself_and_emit_recursively() {
        let bus = Rc::new(EventBus::new());
        let count = Rc::new(Cell::new(0));
        let id = Rc::new(Cell::new(None::<SubId>));
        let (b, c, i) = (Rc::clone(&bus), Rc::clone(&count), Rc::clone(&id));
        let sid = bus.subscribe(move |depth: &u8| {
            c.set(c.get() + 1);
            if let Some(me) = i.get() {
                b.unsubscribe(me);
            }
            if *depth < 3 {
                b.emit(&(depth + 1));
            }
        });
        id.set(Some(sid));
        bus.emit(&0);
        // The snapshot kept the first call alive; once it unsubscribed itself
        // the nested emit saw no subscribers.
        assert_eq!(count.get(), 1);
        assert!(bus.is_empty());
    }

    #[test]
    fn shared_log_survives_poisoning() {
        let log = SharedLog::new();
        log.push(1);
        let l2 = log.clone();
        let r = thread::spawn(move || {
            l2.with(|v| {
                v.push(2);
                panic!("boom while locked");
            })
        })
        .join();
        assert!(r.is_err());
        assert!(log.is_poisoned());
        log.push(3); // still usable
        assert_eq!(log.snapshot(), vec![1, 2, 3]);
    }

    #[test]
    fn shared_log_concurrent_pushes() {
        let log = SharedLog::new();
        thread::scope(|s| {
            for t in 0..8u32 {
                let log = log.clone();
                s.spawn(move || {
                    for i in 0..500 {
                        log.push(t * 1000 + i);
                    }
                });
            }
        });
        assert_eq!(log.len(), 4000);
        let mut v = log.snapshot();
        v.sort_unstable();
        v.dedup();
        assert_eq!(v.len(), 4000);
    }

    #[test]
    fn rw_cache_stores_one_value_and_does_not_hold_lock_while_computing() {
        let cache = RwCache::new();
        let calls = AtomicUsize::new(0);
        // Re-entering the cache from inside the compute function would
        // deadlock if the write lock were held while computing.
        let v = cache.get_or_compute(1u32, || {
            calls.fetch_add(1, Ordering::SeqCst);
            cache.get_or_compute(2u32, || 20u32) + 1
        });
        assert_eq!(v, 21);
        assert_eq!(cache.len(), 2);
        assert_eq!(cache.get_or_compute(1, || unreachable!()), 21);
        assert_eq!(calls.load(Ordering::SeqCst), 1);
    }

    #[test]
    fn rw_cache_racing_threads_agree_on_the_stored_value() {
        for _ in 0..50 {
            let cache = RwCache::new();
            let seen: Vec<usize> = thread::scope(|s| {
                let hs: Vec<_> = (0..8usize)
                    .map(|t| {
                        let cache = &cache;
                        s.spawn(move || cache.get_or_compute("k", || t))
                    })
                    .collect();
                hs.into_iter().map(|h| h.join().unwrap()).collect()
            });
            assert!(seen.iter().all(|&v| v == seen[0]), "{seen:?}");
            assert_eq!(cache.len(), 1);
        }
    }

    #[test]
    fn auto_traits() {
        fn send_sync<T: Send + Sync>() {}
        send_sync::<SharedLog<String>>();
        send_sync::<RwCache<String, String>>();
    }
}
```

✓ With this exact file: `cargo fmt --check` clean, `cargo clippy --all-targets --locked -- -D warnings` clean, and `cargo test --locked` → 9 unit tests, 1 passing doctest and 2 `compile_fail` doctests passed. In a separate crate that depends on `cellab` I confirmed the doctest errors are the intended ones ✓: `E0277` "`RefCell<HashMap<u8, u8>>` cannot be shared between threads safely" (and the same for `Cell<usize>`) for `Memo`, and `E0277` "`Rc<(dyn Fn(&u8) + 'static)>` cannot be sent between threads safely" for `EventBus`. Note I first wrote the `Memo` doctest as "not `Send`" and it failed: `RefCell` and `Cell` **are** `Send`; the property `Memo` lacks is `Sync`.

Mutation checks (each applied to a copy, then `cargo test --lib`):

| Change | Result ✓ |
|---|---|
| `Memo::get` keeps its `borrow()` guard alive while `f` runs | `memo_supports_recursion_and_computes_once` FAILED: `RefCell already borrowed` |
| `EventBus::emit` iterates `self.subs.borrow()` directly instead of a snapshot | 2 tests FAILED (`callbacks_may_reenter_the_bus`, `callback_may_unsubscribe_itself_and_emit_recursively`): `RefCell already borrowed` |
| `RwCache` runs `f()` after taking the write lock | the re-entrancy test **hung** (killed by `timeout` after 20 s, exit 124); a hang is a failure mode of a lock test, so CI needs a timeout |
| `SharedLog::lock` uses `.lock().unwrap()` instead of recovering | `shared_log_survives_poisoning` FAILED |

**Design notes.**

- **Short borrows are the whole game.** `Memo::get` writes `if let Some(v) = self.map.borrow().get(&key) { return v.clone(); }`, which ends the borrow before `f` runs; `borrow_mut()` appears only for the insert, after `f` returned.
- **Snapshot-then-call** in `EventBus::emit` costs one `Vec` allocation per emit and `Rc` counter bumps; it buys re-entrancy. Semantics are documented in the tests: a subscriber added during an emit is not called for that event.
- **Poison recovery is a per-type decision.** `SharedLog` can recover because pushing to a `Vec` can't leave it half-updated in a way that matters to readers. `with` hands the caller a `&mut Vec`, so a caller's panic *inside* `with` could leave the data in an odd state; the type documents this.
- **`RwCache` may compute twice.** Two threads that miss at the same time both run `f`; `entry().or_insert` makes the first insert win, and both return that stored value (✓ test: 50 rounds of 8 racing threads agree). That trades duplicate work for never holding a lock during user code. If `f` is expensive and must run once, use `OnceLock` per key or a per-key lock, at the cost of more machinery.
- **Auto traits** (`SharedLog`/`RwCache` are `Send + Sync` ✓ by `auto_traits` test; `Memo` is `Send` but not `Sync`; `EventBus` is neither) fall out of the field types (Lesson 22).
- **Not production code:** no capacity limits, no eviction, no weak references from callbacks to the bus, and no timeouts on locks.

</details>

---

## 14. Interview questions

<details markdown="1">
<summary>Answers</summary>

1. **What is interior mutability?** Mutating data through a shared reference, with the aliasing rule enforced by the type (copy-only `Cell`, runtime-counted `RefCell`, blocking `Mutex`/`RwLock`) instead of the compiler.
2. **What is `UnsafeCell`?** The language primitive that makes shared-reference mutation legal; every interior-mutability type wraps it. It also disables niche optimizations (✓ `Option<Cell<&u8>>` is 16 bytes).
3. **`Cell` vs `RefCell`?** `Cell` never lends references (copy/replace/take/swap; no overhead, no panics); `RefCell` lends `Ref`/`RefMut` and checks at run time (a counter, panics on violation).
4. **What happens on a `RefCell` violation?** `borrow`/`borrow_mut` panic (✓ `already borrowed` / `already mutably borrowed`); `try_borrow*` return `Err`.
5. **Why are `Cell` and `RefCell` not `Sync`?** Their checks and updates are not atomic; sharing one across threads would be a data race. They are `Send` (✓).
6. **When is `Mutex<T>: Sync`? `RwLock<T>: Sync`?** `Mutex<T>: Sync` if `T: Send`; `RwLock<T>: Sync` if `T: Send + Sync` (✓ `Mutex<Cell<u8>>` is `Sync`, `RwLock<Cell<u8>>` is not).
7. **What is mutex poisoning?** If a thread panics while holding the guard, later `lock()` calls return `Err(PoisonError)` as a warning that the data may be inconsistent; the error carries the guard so you can recover.
8. **Should you `.lock().unwrap()`?** It is the conservative default (propagates the panic). Recover with `into_inner` when updates are atomic/harmless; never recover blindly for invariant-heavy state.
9. **What happens if one thread locks the same `Mutex` twice?** Documented: may deadlock or panic; ✓ here it hung.
10. **Why is `MutexGuard` not `Send`?** Some platforms require unlocking on the thread that locked (✓ E0277 when moved to another thread).
11. **Why can a `match` on `m.lock().unwrap().first()` keep the lock?** The temporary guard lives until the end of the whole `match` (✓ `try_lock` fails inside the arm).
12. **How do you share mutable state between threads?** `Arc<Mutex<T>>`/`Arc<RwLock<T>>`, atomics for single words, or channels; with scoped threads you don't need `Arc` (✓).
13. **How do you avoid `RefCell` panics in callbacks?** Don't call user code under a borrow: snapshot, release, then call (✓).
14. **What are `OnceCell`, `OnceLock`, `LazyLock`?** Write-once cells (single-thread / thread-safe / with an initializer) for lazy statics and config; `set` after init returns the value back in `Err` (✓).
15. **`RwLock` vs `Mutex`?** `RwLock` allows concurrent readers but costs more and has OS-dependent fairness; not automatically faster; measure.
16. **Is `Rc<RefCell<T>>` a good default?** It is common in single-threaded graph/UI code, but it hides the borrow structure and can leak via cycles; try ownership, indices, or `&mut` first.
17. **How can interior mutability be a security problem?** Reachable panics (DoS), blocking/deadlock, recovered poisoned state with broken invariants, and check-then-act races.

</details>

---

## 15. Key takeaways

- Interior mutability moves the "one writer or many readers" rule from the compiler to the type: `Cell` (copy in/out), `RefCell` (run-time counted), `Mutex`/`RwLock` (blocking, thread-safe), `OnceCell`/`OnceLock`/`LazyLock` (write once). All rest on `UnsafeCell`.
- `Cell`/`RefCell` are `Send` but **not `Sync`** (✓); `Mutex<T>` is `Sync` when `T: Send`, `RwLock<T>` needs `T: Send + Sync` (✓). A guard is not `Send` (✓).
- A `RefCell` violation is a **panic**, not a compile error (✓ three variants); keep borrows short and never call user code, callbacks or recursion while holding one. Prefer `try_borrow*` in libraries.
- A panic while holding a `Mutex` **poisons** it (✓); recovering is a decision about your invariants, not a formality. Locking twice on one thread hangs (✓); hold locks briefly, never across slow or user code.
- Temporary guards live to the end of the statement, a `match`/`if let` body included (✓).
- Use `Rc<RefCell<T>>` for single-threaded sharing and `Arc<Mutex<T>>`/`Arc<RwLock<T>>` across threads, but reach for ownership, `&mut`, channels or atomics first.

**Connections.** Lesson 10's aliasing rule is now enforced dynamically; Lesson 22's `Send`/`Sync` explain every `E0277` here; Lesson 23's `Rc`/`Arc` gain mutation. Next: Lesson 25 (`Weak`, reference cycles, `Pin`) closes the cycle problem from Lesson 23 (and the bus-captures-itself issue above), Lesson 27 covers threads and channels, Lesson 28 covers atomics, Lesson 29 locks and deadlocks, and Lesson 36 covers `UnsafeCell` directly. See the [roadmap](00_roadmap.md) and the [course index](index.md).

---

## 16. References

- *The Rust Programming Language*, ch. 15.5 "`RefCell<T>` and the Interior Mutability Pattern" and ch. 16.3 "Shared-State Concurrency": <https://doc.rust-lang.org/book/ch15-05-interior-mutability.html>
- `std::cell` module (`Cell`, `RefCell`, `OnceCell`, `UnsafeCell`, `Ref::map`): <https://doc.rust-lang.org/std/cell/index.html>
- `std::sync::Mutex` (poisoning, deadlock behavior), `RwLock`, `OnceLock`, `LazyLock`: <https://doc.rust-lang.org/std/sync/struct.Mutex.html>, <https://doc.rust-lang.org/std/sync/struct.RwLock.html>, <https://doc.rust-lang.org/std/sync/struct.OnceLock.html>
- *The Rustonomicon*: "Concurrency and Parallelism", "Send and Sync": <https://doc.rust-lang.org/nomicon/send-and-sync.html>
- Rust error code index, E0277, E0373, E0599: <https://doc.rust-lang.org/error_codes/E0277.html>

The URLs above are the standard documentation locations; I did not open them during this session (the sandbox cannot reach them), so if a link has moved, search the docs site for the page title. Unchecked from memory: the Book chapter numbers and anchors; the documentation statements that a double lock "may deadlock or panic", that `RwLock` fairness is unspecified and a read-then-write on one thread can deadlock, why `MutexGuard` is `!Send`, the `UnsafeCell` aliasing rule and its effect on niches (the 16-byte size is measured, the reason is documented), the edition-2024 change to `if let` temporaries (not tested here), the cost of atomics and kernel-level lock behavior (not measured), and the Lesson 10, 22 and 23 pointers (re-read only by title, not reopened).
