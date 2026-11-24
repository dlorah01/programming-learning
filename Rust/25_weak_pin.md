---
title: "Lesson 25 — Weak, reference cycles, and Pin"
---

# Lesson 25 — `Weak`, reference cycles, and `Pin`

**Module 8 · Smart pointers** · Status: generated · Edition used in examples: **2024** · External dependencies: none

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux. As before, ✓ marks outputs, diagnostics and measurements I actually ran; unmarked statements are documented behavior I did not run. Everything is **stable Rust**. **Miri is not installed in this sandbox** (`cargo miri` is unavailable for the stable toolchain), so the small amount of `unsafe` in this lesson was tested but *not* checked by an undefined-behavior detector; the lesson says where that matters.

Builds on [Lesson 23](23_box_rc_arc.md) (`Rc`/`Arc`, and the cycle leak), [Lesson 24](24_interior_mutability.md) (`RefCell`, the event bus that captures itself) and [Lesson 22](22_gats_send_sync_phantom.md) (`Send`/`Sync`, `PhantomData`).

---

## 1. Part one: `Weak<T>`

### 1.1 The problem

An `Rc` keeps its value alive. If two values own each other through `Rc`, neither count ever reaches zero and both leak (Lesson 23). A parent that owns its children and a child that owns its parent is exactly that shape. The fix is to make one direction **non-owning**: `Weak<T>`.

- A **strong** pointer (`Rc<T>`, `Arc<T>`) keeps the value alive.
- A **weak** pointer (`Weak<T>`) does not. To use the value you call `upgrade()`, which returns `Some(Rc<T>)` if the value still exists, or `None` if it has been dropped.

Rule of thumb: **owners are strong, back-references are weak**. Parent → child is strong, child → parent is weak; a subject → observers link is weak if the observer must be able to go away on its own.

### 1.2 The API in action ✓

```rust
use std::rc::{Rc, Weak};
struct Noisy(&'static str);
impl Drop for Noisy { fn drop(&mut self) { println!("drop {}", self.0); } }
fn main() {
    let strong = Rc::new(Noisy("value"));
    let weak: Weak<Noisy> = Rc::downgrade(&strong);
    println!("{} {}", Rc::strong_count(&strong), Rc::weak_count(&strong));
    println!("{}", weak.upgrade().is_some());
    let w2 = weak.clone();
    println!("{} {} {}", Rc::strong_count(&strong), Rc::weak_count(&strong), weak.ptr_eq(&w2));
    if let Some(s) = weak.upgrade() { println!("upgraded, strong={}", Rc::strong_count(&s)); }
    println!("strong={}", Rc::strong_count(&strong));
    drop(strong);
    println!("after drop: {:?} {} {}", weak.upgrade().map(|r| r.0), weak.strong_count(), weak.weak_count());
    let empty: Weak<Noisy> = Weak::new();
    println!("{} {}", empty.upgrade().is_none(), empty.strong_count());
}
```

✓ Output:

```text
1 1
true
1 2 true
upgraded, strong=2
strong=1
drop value
after drop: None 0 0
true 0
```

What to notice: `Rc::downgrade` makes the weak count 1 without touching the strong count; cloning a `Weak` raises the weak count to 2; a successful `upgrade()` creates a *temporary strong owner* (strong = 2 while `s` lives, back to 1 after). When the last strong pointer is dropped, the value is dropped (`drop value`) **even though weak pointers remain**; after that `upgrade()` gives `None`, `strong_count()` is 0 and `weak_count()` reports 0 as well. `Weak::new()` makes a weak pointer that never had a value, so `upgrade()` is always `None`.

### 1.3 What a `Weak` costs: the allocation outlives the value ✓

Dropping the last `Rc` runs the destructor of `T`, but the *memory block* (counters + space for `T`) cannot be freed while any `Weak` still points to it, because the `Weak` must be able to read the counters to learn that the value is gone. I measured this with a counting allocator that tracks live bytes:

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::rc::{Rc, Weak};
struct Counting;
static LIVE: AtomicUsize = AtomicUsize::new(0);
static ALLOCS: AtomicUsize = AtomicUsize::new(0);
unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 { LIVE.fetch_add(l.size(), Relaxed); ALLOCS.fetch_add(1, Relaxed); unsafe { System.alloc(l) } }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) { LIVE.fetch_sub(l.size(), Relaxed); unsafe { System.dealloc(p, l) } }
}
#[global_allocator] static A: Counting = Counting;
fn live() -> usize { LIVE.load(Relaxed) }
fn main() {
    println!("warm up stdout buffer");
    let base = live();
    let a0 = ALLOCS.load(Relaxed);
    let w: Weak<u8> = Weak::new();
    println!("Weak::new: live +{} allocs +{}", live() - base, ALLOCS.load(Relaxed) - a0);
    let rc = Rc::new([0u8; 1000]);
    println!("Rc<[u8;1000]>: live +{}", live() - base);
    let wk = Rc::downgrade(&rc);
    println!("after downgrade: live +{}", live() - base);
    drop(rc);
    println!("after dropping the only Rc (a Weak remains): live +{}", live() - base);
    println!("{}", wk.upgrade().is_none());
    drop(wk);
    println!("after dropping the Weak: live +{}", live() - base);
    drop(w);
}
```

✓ Output:

```text
warm up stdout buffer
Weak::new: live +0 allocs +0
Rc<[u8;1000]>: live +1016
after downgrade: live +1016
after dropping the only Rc (a Weak remains): live +1016
true
after dropping the Weak: live +0
```

`Weak::new()` allocates nothing. An `Rc<[u8; 1000]>` is 1016 bytes (1000 plus two counters, as in Lesson 23). `downgrade` allocates nothing. And after the only `Rc` is gone the 1016 bytes are **still live** (the array's destructor has run, but the block is retained) until the last `Weak` is dropped. (The first `println!` is there because the first print allocates stdout's buffer, which would otherwise show up in the numbers.) Consequence: a long-lived `Weak` to a large object retains its memory. A cache of `Weak<BigThing>` can hold on to megabytes of "dead" allocations until the entries are removed (the practical challenge's `prune`).

### 1.4 `Arc` and `Weak` across threads

`std::sync::Weak` (the weak counterpart of `Arc`) is `Send`/`Sync` when `T` is. `std::rc::Weak` is not `Send`, like `Rc` ✓:

```rust
use std::sync::{Arc, Weak, Mutex};
use std::thread;
fn main() {
    let a = Arc::new(Mutex::new(5));
    let w: Weak<Mutex<i32>> = Arc::downgrade(&a);
    let h = thread::spawn(move || { if let Some(s) = w.upgrade() { *s.lock().unwrap() += 1; true } else { false } });
    println!("{}", h.join().unwrap());
    println!("{} {}", a.lock().unwrap(), Arc::weak_count(&a));
}
```

✓ Output `true`, then `6 0` (the weak pointer moved into the thread was dropped when the thread finished). And:

```rust
use std::rc::{Rc, Weak};
fn main() {
    let w: Weak<u8> = { let r = Rc::new(1u8); Rc::downgrade(&r) };
    thread_check(w);
}
fn thread_check(w: Weak<u8>) { std::thread::spawn(move || { let _ = w.upgrade(); }).join().unwrap(); }
```

✓ `error[E0277]: `std::rc::Weak<u8>` cannot be sent between threads safely` (Lesson 22).

With `Arc`, `upgrade()` is a *check-then-act in a concurrent world*: the answer can change right after you get it. Treat `None` as "gone", never as "will stay gone" or "was never there", and never write `w.strong_count() > 0` followed by `w.upgrade().unwrap()`; just call `upgrade()` once and match on it (the practical challenge's registry does).

### 1.5 Breaking a cycle ✓

Lesson 23's two-node cycle leaked. The same program with weak back-links:

```rust
use std::rc::Rc;
use std::cell::RefCell;
struct Noisy(&'static str);
impl Drop for Noisy { fn drop(&mut self) { println!("drop {}", self.0); } }
struct N { _n: Noisy, other: RefCell<Option<Rc<N>>> }
fn main() {
    {
        let a = Rc::new(N { _n: Noisy("a"), other: RefCell::new(None) });
        let b = Rc::new(N { _n: Noisy("b"), other: RefCell::new(Some(a.clone())) });
        *a.other.borrow_mut() = Some(b.clone());
        println!("{} {}", Rc::strong_count(&a), Rc::strong_count(&b));
    }
    println!("end of scope");
}
```

✓ Output `2 2`, `end of scope`: **no `drop` lines**, the leak. Now with `Weak`:

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;
struct N { _n: Noisy, other: RefCell<Weak<N>> }
struct Noisy(&'static str);
impl Drop for Noisy { fn drop(&mut self) { println!("drop {}", self.0); } }
fn main() {
    {
        let a = Rc::new(N { _n: Noisy("a"), other: RefCell::new(Weak::new()) });
        let b = Rc::new(N { _n: Noisy("b"), other: RefCell::new(Rc::downgrade(&a)) });
        *a.other.borrow_mut() = Rc::downgrade(&b);
        println!("{} {} {} {}", Rc::strong_count(&a), Rc::weak_count(&a), Rc::strong_count(&b), Rc::weak_count(&b));
    }
    println!("end of scope");
}
```

✓ Output `1 1 1 1`, then `drop b`, `drop a`, `end of scope`. Each node has one strong owner (its local) and one weak reference from the other node; at the end of the scope the locals drop in reverse order, both counts reach zero, both values are dropped. (The `RefCell` is the same device as in Lesson 24: the link must be set after both nodes exist.)

### 1.6 A parent/child tree ✓

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;
struct Noisy(&'static str);
impl Drop for Noisy { fn drop(&mut self) { println!("drop {}", self.0); } }
struct Node { _name: Noisy, parent: RefCell<Weak<Node>>, children: RefCell<Vec<Rc<Node>>> }
fn node(n: &'static str) -> Rc<Node> { Rc::new(Node { _name: Noisy(n), parent: RefCell::new(Weak::new()), children: RefCell::new(vec![]) }) }
fn main() {
    let root = node("root");
    {
        let kid = node("kid");
        *kid.parent.borrow_mut() = Rc::downgrade(&root);
        root.children.borrow_mut().push(Rc::clone(&kid));
        println!("kid: strong={} weak={}", Rc::strong_count(&kid), Rc::weak_count(&kid));
        println!("root: strong={} weak={}", Rc::strong_count(&root), Rc::weak_count(&root));
        println!("parent alive: {}", kid.parent.borrow().upgrade().is_some());
    }
    println!("scope ended");
    println!("kid via root: strong={}", Rc::strong_count(&root.children.borrow()[0]));
    drop(root);
    println!("end");
}
```

✓ Output:

```text
kid: strong=2 weak=0
root: strong=1 weak=1
parent alive: true
scope ended
kid via root: strong=1
drop root
drop kid
end
```

The kid has two strong owners (its local and the root's child list) and no weak ones; the root has one strong owner and one weak reference (the kid's parent link). The kid outlives its local variable because the parent owns it. Dropping the root drops its value first (`drop root`), then its fields, including the children, so `drop kid` follows. The practical challenge makes this a reusable `TreeNode` type.

---

## 2. Part two: `Pin<P>`

### 2.1 The problem: moves are `memcpy`

In Rust, *moving* a value copies its bytes to a new location and forgets the old one (Lessons 07 and 09). That is fine for almost everything, because nothing points *into* the value. A **self-referential** value is different: it contains a pointer to one of its own fields. After a move the pointer still holds the **old** address.

We can see this without ever dereferencing a stale pointer; we just compare addresses ✓:

```rust
use std::marker::PhantomPinned;
use std::pin::Pin;
// Self-referential struct: why moving is a problem. We only COMPARE addresses; we never dereference a stale pointer.
struct SelfRef { data: String, ptr: *const String, _p: PhantomPinned }
impl SelfRef {
    fn new(s: &str) -> Pin<Box<SelfRef>> {
        let mut b = Box::pin(SelfRef { data: s.to_string(), ptr: std::ptr::null(), _p: PhantomPinned });
        let p: *const String = &b.data;
        unsafe { b.as_mut().get_unchecked_mut().ptr = p; }
        b
    }
    fn data_via_ptr(self: Pin<&Self>) -> &str { unsafe { &*self.ptr } }
    fn ptr_is_self(self: Pin<&Self>) -> bool { std::ptr::eq(self.ptr, &self.data) }
}
struct Unpinned { data: String, ptr: *const String }
fn main() {
    let b = SelfRef::new("hello");
    println!("{} {}", b.as_ref().data_via_ptr(), b.as_ref().ptr_is_self());
    // moving the Pin<Box<_>> only moves the box pointer, not the SelfRef
    let b2 = b;
    println!("{}", b2.as_ref().ptr_is_self());

    // Without pinning, moving the struct breaks the invariant:
    let mut u = Unpinned { data: "hi".to_string(), ptr: std::ptr::null() };
    u.ptr = &u.data;
    println!("{}", std::ptr::eq(u.ptr, &u.data));
    let moved = u;                // plain move: new address
    println!("{}", std::ptr::eq(moved.ptr, &moved.data));
}
```

✓ Output `hello true`, `true`, `true`, `false`. The pinned version stays consistent after `let b2 = b;`, because only the 8-byte box pointer moved. The unpinned struct was consistent until the move and inconsistent afterward: `moved.ptr` still holds the old address. (This check relies on the compiler placing `moved` at a different address than `u`; in an optimized build it may reuse the slot, so treat the `false` as an illustration for this debug build, not a guarantee.) Dereferencing such a stale pointer would be undefined behavior; that is the reason the type system gets involved.

Where does this matter in practice? Mostly in `async`: the compiler turns an `async` block into a state machine that can hold references to its own locals across `.await` points. That state machine must not move once it has started. That is why `Future::poll` takes `Pin<&mut Self>` (Lesson 30).

### 2.2 `Pin<P>` and `Unpin`

`Pin<P>` wraps a pointer `P` (such as `&mut T`, `Box<T>`, `Rc<T>`) with this promise: **the value `*P` will not be moved or its memory reused until it is dropped**, unless `T: Unpin`.

- **`Unpin`** is an auto trait (like `Send`): "moving this type is always fine even when it's behind a `Pin`". Nearly everything is `Unpin`: integers, `String`, `Vec<i32>`, boxes, references. For `Unpin` types, `Pin` does nothing: you can get `&mut T` back freely.
- **`PhantomPinned`** is a zero-sized marker that makes a type `!Unpin`. It is how a self-referential type opts out.
- Types that *contain* a `!Unpin` field are `!Unpin`.

I checked a matrix by compiling `fn u<T: Unpin>() {}` calls ✓:

| Type | `Unpin`? |
|---|---|
| `u32`, `String`, `Box<Fixed>`, `&mut Fixed`, `Pin<Box<Fixed>>` | yes |
| `PhantomPinned`, `Fixed` (a struct with a `PhantomPinned` field) | **no** |
| `Option<Fixed>`, `[Fixed; 2]`, `Vec<Fixed>`, a struct that contains a `Fixed` | **no** |

(`Fixed` is `struct Fixed { _p: PhantomPinned }`.) Note `Box<Fixed>` is `Unpin` even though `Fixed` is not: moving the *box* (a pointer) never moves the pointee. And an `async` block is `!Unpin` ✓: `fn is_unpin<T: Unpin>(_: &T)` called with `async { 1 }` gives `error[E0277]: `{async block@...}` cannot be unpinned`.

Sizes: `Pin<Box<T>>` and `Pin<&mut T>` are the same size as the pointers they wrap ✓ (`8 8` for a sized `T`); `Pin` is a zero-cost wrapper with no runtime check, only type-level restrictions.

### 2.3 What `Pin` allows and forbids ✓

For a `!Unpin` type, safe code can create a pinned pointer, can read through it, but cannot get `&mut T` out:

```rust
use std::marker::PhantomPinned;
use std::pin::Pin;
struct Fixed { v: u32, _p: PhantomPinned }
fn main() {
    let mut x = Fixed { v: 1, _p: PhantomPinned };
    let p = Pin::new(&mut x);
    println!("{}", p.v);
}
```

✓ `error[E0277]: `PhantomPinned` cannot be unpinned`, with the note "consider using the `pin!` macro". `Pin::new` is only for `Unpin` targets. To pin a `!Unpin` value in safe code you either box it (`Box::pin`) or pin it on the stack with `std::pin::pin!`. Then mutation is blocked ✓:

```rust
use std::marker::PhantomPinned;
use std::pin::Pin;
struct Fixed { v: u32, _p: PhantomPinned }
fn main() {
    let mut b: Pin<Box<Fixed>> = Box::pin(Fixed { v: 1, _p: PhantomPinned });
    println!("{}", b.v);          // reading is fine through Deref
    b.v = 2;                      // writing through DerefMut needs Unpin
    let r: &mut Fixed = &mut *b;
    let other = Fixed { v: 9, _p: PhantomPinned };
    std::mem::swap(r, &mut Box::new(other));
}
```

✓ Two errors: `error[E0594]: cannot assign to data in dereference of `Pin<Box<Fixed>>`` and `error[E0596]: cannot borrow data in dereference of `Pin<Box<Fixed>>` as mutable`, both with the help "trait `DerefMut` is required to modify through a dereference, but it is not implemented for `Pin<Box<Fixed>>`". That is the whole mechanism: `Pin<P>` implements `DerefMut` only when `T: Unpin`, so safe code cannot get a `&mut T` to a pinned `!Unpin` value, and therefore cannot `mem::swap` or `mem::replace` it (which would move it).

The ways to create and use pins (all ✓ in one program):

```rust
use std::marker::PhantomPinned;
use std::pin::{Pin, pin};
struct Fixed { v: u32, _p: PhantomPinned }
fn main() {
    let a = pin!(Fixed { v: 1, _p: PhantomPinned });   // Pin<&mut Fixed> on the stack
    println!("{}", a.v);
    let b: Pin<Box<Fixed>> = Box::pin(Fixed { v: 2, _p: PhantomPinned });
    println!("{}", b.v);
    // unsafe escape hatch: you promise not to move out
    let mut c = Box::pin(Fixed { v: 3, _p: PhantomPinned });
    unsafe { c.as_mut().get_unchecked_mut().v = 4; }
    println!("{}", c.v);
    // as_mut / as_ref give re-borrowable Pins
    let mut d = Box::pin(5u32);   // u32: Unpin, so get_mut is safe
    *d.as_mut().get_mut() += 1;
    println!("{}", d);
    let e: Box<u32> = Pin::into_inner(d);
    println!("{}", e);
}
```

✓ Output `1`, `2`, `4`, `6`, `6`. `Box::pin` allocates on the heap; `pin!` pins a value in place on the stack for the current scope; `as_mut()` reborrows a `Pin<&mut T>` so you can use the pin more than once; `get_unchecked_mut` is `unsafe` because *you* promise not to move out of the `&mut` you get (writing a field in place is fine); and for `Unpin` types `get_mut`/`into_inner` are safe.

### 2.4 `Pin` in `Future::poll` ✓

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll, RawWaker, RawWakerVTable, Waker};
fn noop_waker() -> Waker {
    fn clone(_: *const ()) -> RawWaker { RawWaker::new(std::ptr::null(), &VT) }
    fn noop(_: *const ()) {}
    static VT: RawWakerVTable = RawWakerVTable::new(clone, noop, noop, noop);
    unsafe { Waker::from_raw(RawWaker::new(std::ptr::null(), &VT)) }
}
struct CountDown(u32);
impl Future for CountDown {
    type Output = &'static str;
    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        if self.0 == 0 { Poll::Ready("done") } else { self.0 -= 1; cx.waker().wake_by_ref(); Poll::Pending }
    }
}
fn main() {
    let w = noop_waker();
    let mut cx = Context::from_waker(&w);
    let mut f = Box::pin(CountDown(2));
    loop {
        match f.as_mut().poll(&mut cx) { Poll::Ready(v) => { println!("{v}"); break; } Poll::Pending => println!("pending") }
    }
    let mut fut = Box::pin(async { let x = 20; let y = [1u8, 2, 3]; let r = &y; x + r.len() as i32 });
    match fut.as_mut().poll(&mut cx) { Poll::Ready(v) => println!("async block: {v}"), Poll::Pending => println!("p") }
}
```

✓ Output `pending`, `pending`, `done`, `async block: 23`. `CountDown` is `Unpin` (it only holds a `u32`), so `self.0 -= 1` works through `Pin<&mut Self>` (that needs `DerefMut`). The async block holds a reference `r` to its own local `y` across what could be a suspension point; `Box::pin` guarantees it never moves. A hand-written no-op waker takes `unsafe` (it is the standard minimal executor trick, simplified); real executors come in Lessons 30 and 31. This lesson only needs the idea: **`Pin<&mut Self>` in `poll` is how the future tells you "you may rely on staying where you are"**.

### 2.5 The full contract (documented, not run)

Pinning is stronger than "don't move": for a pinned `!Unpin` value, the memory must remain valid and not be reused until its destructor has run. In other words, you may not free or overwrite the storage without dropping the value first. Implementing this correctly for self-referential types, or projecting a `Pin<&mut Outer>` to a `Pin<&mut Field>` ("structural pinning"), requires `unsafe` and has subtle rules (e.g. a field is either structurally pinned or not, never both; `Drop` must not move pinned fields). In practice people use the `pin-project` family of crates or `pin!`; this course will not hand-write projections.

### 2.6 When you actually need `Pin`

Almost never as an author: you consume it. You meet `Pin` when you (a) implement `Future` by hand (Lesson 30), (b) store futures in collections (`Pin<Box<dyn Future<Output = T>>>`), or (c) write low-level intrusive data structures. If you find yourself wanting a self-referential struct, first consider alternatives: store **indices or offsets** instead of pointers, split the data so that one part owns and the other borrows, use an arena, or use `Rc`/`Weak` as above.

---

## 3. Common mistakes and edge cases

- **Cycles of `Rc`** (✓ leak): back-pointers must be `Weak`.
- **Treating `upgrade()` as infallible** (`.upgrade().unwrap()`): it panics once the target is gone (✓ the orphan case in the debugging challenge). Match on the `Option`.
- **Assuming dropping the last `Rc` frees memory while `Weak`s exist** (✓ 1016 bytes retained).
- **Checking `strong_count()` then `upgrade()`** with `Arc`: racy; just `upgrade()`.
- **Holding the upgraded `Rc` longer than needed**: it is a strong owner and delays the drop; take what you need and let it go.
- **`Weak` in a `Vec` that grows forever**: dead entries stay until you prune them.
- **`Pin::new` on a `!Unpin` type** (✓ E0277): use `Box::pin` or `pin!`.
- **Expecting `Pin<Box<T>>` to be immovable**: the *box* can move; `T` cannot.
- **Using `get_unchecked_mut` and then moving out** (`mem::swap`, `mem::replace`, `*r = ...` assigning a whole new value): this breaks the pin contract and is UB if the type is self-referential. Only touch fields in place.
- **Making a struct self-referential with a plain `*const` and no `PhantomPinned`**: nothing stops safe code from moving it (✓ the `Unpinned` example). Add `PhantomPinned` and hand out `Pin<Box<_>>` only.
- **Forgetting `Vec<T>` of a `!Unpin` type is itself `!Unpin`** (✓ matrix).

---

## 4. Performance implications

- `Rc::downgrade` and `Weak::clone` bump a counter; no allocation (✓ zero). `upgrade` checks the strong count and increments it if non-zero. For `Arc` these are atomic operations (cost not measured here, Lesson 41).
- `Weak::new` allocates nothing (✓), so `Option<Weak<T>>`-style "empty parent" fields are cheap. `size_of::<Weak<T>>()` is a pointer for sized `T` (I did not print it here; from the documentation).
- A `Weak` retains the whole block (✓ 1016 bytes) until it is dropped. For many small objects behind long-lived weak caches, memory held by dead objects can add up; prune periodically.
- `Pin` costs nothing at run time (✓ same size as the wrapped pointer); `Box::pin` costs one allocation, `pin!` none.
- A tree with `Weak` parent links needs one extra pointer per node and a `RefCell` borrow flag for the link (Lesson 24); a slab/arena with integer parent ids is smaller and more cache-friendly, at the price of manual lifetime management.

---

## 5. Security and reliability

- **Cycle leaks are a memory-exhaustion vector.** If untrusted input can create cyclic strong references (graph nodes, sessions that reference each other), each request can leak. Use `Weak` for back-edges, or an arena that is freed as a whole.
- **Dead-but-retained memory.** `Weak` keeps the allocation (✓). A registry of `Weak<Big>` that is never pruned grows without bound.
- **`upgrade()` races.** With `Arc`, "is it still alive?" may change between two statements. Use a single `upgrade()` and act on its result.
- **Recursive drop depth** (Lesson 23) also applies to trees: a deep child chain can overflow the stack when dropped. The practical challenge's `TreeNode` drops iteratively and was tested with a million-node chain (✓).
- **`Pin` plus `unsafe` is where memory-safety bugs live.** Violating the pin contract (moving a pinned self-referential value) is undefined behavior: use-after-move can read freed or reused memory. The `unsafe` in this lesson's `PinnedText` is small, but **I could not run Miri here**, so it was tested but not UB-checked; run `cargo +nightly miri test` yourself before trusting any real `unsafe` code (Lesson 38).
- **Never claim `Pin` is a security feature.** It prevents a class of *accidental* moves at compile time; it does not sandbox anything.

---

## 6. Exercises

Try each before opening the answer. They are not marked complete unless you tell me you did them.

1. **(Easy)** After `let a = Rc::new(1); let w = Rc::downgrade(&a); let w2 = w.clone(); let b = a.clone();` what are `strong_count` and `weak_count` of `a`? What does `w.upgrade()` give after `drop(a); drop(b);`?
2. **(Easy)** Why does `Weak::new()` not allocate, and what does `upgrade()` return on it?
3. **(Medium)** A cache holds `Vec<Weak<[u8; 1000]>>`. All the `Rc`s are dropped. How much memory is still allocated per entry, and when is it released?
4. **(Medium)** Write `fn root_of(node: &Rc<Node>) -> Rc<Node>` for a `Node` with `parent: RefCell<Weak<Node>>`, without recursion.
5. **(Medium)** Why does `Pin::new(&mut x)` fail for `x: Fixed` (a struct with `PhantomPinned`) and what are two safe ways to pin it?
6. **(Hard)** Which of these are `Unpin`: `Box<Fixed>`, `Vec<Fixed>`, `Option<Fixed>`, `&mut Fixed`, `Pin<Box<Fixed>>`, an `async` block?
7. **(Hard)** `Pin<Box<Fixed>>` lets you read `b.v` but not assign it. Explain why, and give the minimal way (with `unsafe`) and a safe design to modify a field.
8. **(Hard)** You maintain an observer list `Vec<Rc<dyn Fn(i32)>>` and observers hold an `Rc` back to the subject. Redesign it so that dropping the subject frees everything, and say what `notify` must do when an observer has gone.

<details markdown="1">
<summary>Answers</summary>

1. `strong_count` is 2 (`a` and `b`), `weak_count` is 2 (`w` and `w2`), the same pattern as ✓ §1.2. After `drop(a); drop(b);` the strong count is 0, the value is dropped, and `w.upgrade()` is `None` (✓ `after drop: None 0 0`).
2. It has no value and nothing to count, so there is no block to allocate (✓ `live +0 allocs +0`). `upgrade()` always returns `None`.
3. ✓ 1016 bytes per entry (1000 plus the two counters, the measured size of an `Rc<[u8; 1000]>`). The array's destructor ran, but the block stays until the `Weak` itself is dropped (the vector is cleared or the entry removed). Prune dead entries (`retain(|w| w.strong_count() > 0)`) to release it.
4. `fn root_of(node: &Rc<Node>) -> Rc<Node> { let mut cur = Rc::clone(node); loop { let parent = cur.parent.borrow().upgrade(); match parent { Some(p) => cur = p, None => return cur } } }`. The `borrow()` temporary ends at the semicolon of the `let`, so the borrow is released before `cur` is reassigned (Lesson 24). (I did not compile this exact function; the `depth` and `path_to_root` methods of the practical challenge use the same loop and were tested ✓.)
5. `Pin::new` requires `Target: Unpin`; `PhantomPinned` makes `Fixed` `!Unpin`, so ✓ `E0277: `PhantomPinned` cannot be unpinned`. Safe ways: `Box::pin(x)` (heap) or `pin!(x)` (stack, for the current scope) ✓.
6. ✓ From the matrix: `Box<Fixed>` yes; `&mut Fixed` yes; `Pin<Box<Fixed>>` yes; `Vec<Fixed>` **no**; `Option<Fixed>` **no**; an `async` block **no** (✓ E0277 "cannot be unpinned").
7. `DerefMut for Pin<P>` exists only when `P::Target: Unpin` (✓ errors E0594/E0596 mention the missing `DerefMut`); handing out `&mut Fixed` would let you `mem::swap` it, which moves a value that may be self-referential. Minimal unsafe: `unsafe { b.as_mut().get_unchecked_mut().v = 4; }` ✓, with the promise to modify in place only. Safe design: expose a method taking `self: Pin<&mut Self>` that does the unsafe once (as `PinnedText::set_text` does ✓) and keep the rest of the API safe.
8. Make the observer → subject link `Weak<Subject>` and the subject → observer link strong, or the reverse if observers should be able to disappear on their own (store `Weak<dyn Fn(i32)>` and let the owner of each observer hold the `Rc`). In `notify`, snapshot the list (Lesson 24), then for each `Weak` call `upgrade()`; if it returns `None`, skip it and remove the entry afterwards. Dropping the subject then frees it as soon as the last strong handle goes.

</details>

---

## 7. Debugging challenge

This tree compiles and runs, but it never frees anything, and one of its operations will panic in a plausible scenario:

```rust
use std::cell::RefCell;
use std::rc::Rc;
struct Node {
    name: &'static str,
    parent: RefCell<Option<Rc<Node>>>,
    children: RefCell<Vec<Rc<Node>>>,
}
impl Drop for Node { fn drop(&mut self) { println!("drop {}", self.name); } }
fn new(name: &'static str) -> Rc<Node> {
    Rc::new(Node { name, parent: RefCell::new(None), children: RefCell::new(vec![]) })
}
fn add_child(parent: &Rc<Node>, child: Rc<Node>) {
    *child.parent.borrow_mut() = Some(Rc::clone(parent));
    parent.children.borrow_mut().push(child);
}
fn main() {
    let root = new("root");
    add_child(&root, new("a"));
    add_child(&root, new("b"));
    println!("root strong = {}", Rc::strong_count(&root));
    let orphan = Rc::clone(&root.children.borrow()[0]);
    drop(root);
    println!("orphan parent: {}", orphan.parent.borrow().as_ref().unwrap().name);
    println!("end of main");
}
```

1. Run it and describe what is wrong (look at the counts and at the `drop` lines). 2. Fix the ownership so everything is freed. 3. After your fix, what should the last `println!` of the orphan's parent do, and why does `unwrap()` become dangerous?

<details markdown="1">
<summary>Solution</summary>

**1. The bug.** ✓ Output: `root strong = 3`, `orphan parent: root`, `end of main`, and **no `drop` lines at all**. Each child holds a strong `Rc` to the root (`parent: Some(Rc::clone(parent))`) and the root holds strong `Rc`s to the children: two strong cycles. The root's count is 3 (the local plus the two children) and `drop(root)` only lowers it to 2, so the root is never dropped, and neither are the children. The program also *appears* correct, since the orphan can still reach its parent: the leak is what keeps it alive.

**2. The fix.** Parent → child stays strong; child → parent becomes `Weak`:

```rust
use std::cell::RefCell;
use std::rc::{Rc, Weak};
struct Node {
    name: &'static str,
    parent: RefCell<Weak<Node>>,
    children: RefCell<Vec<Rc<Node>>>,
}
impl Drop for Node { fn drop(&mut self) { println!("drop {}", self.name); } }
fn new(name: &'static str) -> Rc<Node> {
    Rc::new(Node { name, parent: RefCell::new(Weak::new()), children: RefCell::new(vec![]) })
}
fn add_child(parent: &Rc<Node>, child: Rc<Node>) {
    *child.parent.borrow_mut() = Rc::downgrade(parent);
    parent.children.borrow_mut().push(child);
}
fn main() {
    let root = new("root");
    add_child(&root, new("a"));
    add_child(&root, new("b"));
    println!("root strong = {} weak = {}", Rc::strong_count(&root), Rc::weak_count(&root));
    let orphan = Rc::clone(&root.children.borrow()[0]);
    drop(root);
    match orphan.parent.borrow().upgrade() {
        Some(p) => println!("orphan parent: {}", p.name),
        None => println!("orphan has no parent any more"),
    }
    println!("end of main");
}
```

✓ Output:

```text
root strong = 1 weak = 2
drop root
drop b
orphan has no parent any more
end of main
drop a
```

The root now has one strong owner and two weak references; `drop(root)` frees it, and with it the child `b` (only the root owned `b`). Child `a` survives until the end of `main` because `orphan` holds it, and is dropped last.

**3.** After the fix the root is gone, so `upgrade()` returns `None` (✓ `orphan has no parent any more`). With the original `.as_ref().unwrap()` pattern translated naively to `orphan.parent.borrow().upgrade().unwrap()`, the program would **panic** here. A weak parent link is allowed to be dead; code that walks upward must treat `None` as a normal outcome (as `TreeNode::parent` does in the practical challenge: it returns `Option`). The bug and the "fix" differ in what they *promise*: strong parents guaranteed the link was valid and caused the leak; weak parents free memory and make validity something you check.

</details>

---

## 8. Practical challenge: `weaklab`

Create a library crate `weaklab` (edition 2024) with:

1. `TreeNode<T>`: an `Rc`-based tree handle with strong child links and a `Weak` parent link. API: `new`, `value`, `add_child(value) -> TreeNode`, `attach(&TreeNode) -> Result<(), AttachError>` (errors `HasParent` and `WouldCycle`), `parent() -> Option<TreeNode>`, `children()`, `detach() -> bool`, `depth()`, `subtree_len()`, `path_to_root()` (for `T: Clone`), `counts() -> (strong, weak)`. All traversal must be iterative, and dropping a million-node-deep chain must not overflow the stack.
2. `WeakRegistry<K, V>`: a thread-safe interner holding `Weak<V>` entries: `get_or_insert_with(key, make) -> Arc<V>` (live value is reused, dead entry is replaced, `make` runs with no lock held, racing threads agree), `prune() -> usize`, `len`, `is_empty`; poison-tolerant.
3. `PinnedText`: a `!Unpin` struct holding a `String` and a raw pointer to it, created only as `Pin<Box<Self>>`, with `text(self: Pin<&Self>)`, `set_text(self: Pin<&mut Self>, &str)` and `is_consistent`. Every `unsafe` block needs a `// SAFETY:` comment.
4. Tests for each behavior (exact-once drops, orphans, detach/attach/cycle refusal, a million-deep chain, registry reuse and replacement, 50 rounds of racing threads, pinned text surviving moves of the box and buffer reallocation); a `compile_fail` doctest that `TreeNode` is not `Send`, and one that `PinnedText` cannot be `Pin::new`ed; a usage doctest.
5. It must pass `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, and `cargo test --locked` (run `cargo generate-lockfile` once first).

<details markdown="1">
<summary>Solution</summary>

**`src/lib.rs`**

```rust
//! `weaklab`: `Weak`, cycles, and `Pin`.
//!
//! * [`TreeNode`]: a tree with strong child links and weak parent links, so
//!   there are no reference cycles; deep trees drop iteratively.
//! * [`WeakRegistry`]: a thread-safe interner that does not keep values alive.
//! * [`PinnedText`]: a tiny self-referential type that is `!Unpin`.

use std::cell::RefCell;
use std::collections::HashMap;
use std::hash::Hash;
use std::marker::PhantomPinned;
use std::pin::Pin;
use std::rc::{Rc, Weak};
use std::sync::{Arc, Mutex, MutexGuard, PoisonError, Weak as ArcWeak};

// ---------------------------------------------------------------- TreeNode

struct Inner<T> {
    value: T,
    parent: RefCell<Weak<Inner<T>>>,
    children: RefCell<Vec<Rc<Inner<T>>>>,
}

impl<T> Drop for Inner<T> {
    /// Dropping a node would normally recurse through its children, one stack
    /// frame per level. Move the descendants into a work list instead.
    fn drop(&mut self) {
        let mut work: Vec<Rc<Inner<T>>> = std::mem::take(self.children.get_mut());
        while let Some(rc) = work.pop() {
            // Only take the children if we are the last owner; otherwise
            // someone else still holds this node and it keeps its subtree.
            if let Ok(mut inner) = Rc::try_unwrap(rc) {
                work.append(inner.children.get_mut());
            }
        }
    }
}

/// Why [`TreeNode::attach`] refused.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum AttachError {
    /// The node to attach already has a parent.
    HasParent,
    /// Attaching would make a node its own ancestor.
    WouldCycle,
}

/// A handle to a tree node. Cloning the handle shares the node.
///
/// `TreeNode` uses `Rc`, so it is neither `Send` nor `Sync`:
///
/// ```compile_fail
/// fn needs_send<T: Send>(_: T) {}
/// needs_send(weaklab::TreeNode::new(1u8));
/// ```
///
/// Usage:
///
/// ```
/// use weaklab::TreeNode;
/// let root = TreeNode::new("root");
/// let leaf = root.add_child("a").add_child("b");
/// assert_eq!(leaf.path_to_root(), vec!["b", "a", "root"]);
/// drop(root);
/// // Only the root owned "a" (the leaf holds a weak link upward), so dropping
/// // the root freed it:
/// assert!(leaf.parent().is_none());
/// ```
pub struct TreeNode<T>(Rc<Inner<T>>);

impl<T> Clone for TreeNode<T> {
    fn clone(&self) -> Self {
        Self(Rc::clone(&self.0))
    }
}

impl<T> TreeNode<T> {
    pub fn new(value: T) -> Self {
        Self(Rc::new(Inner {
            value,
            parent: RefCell::new(Weak::new()),
            children: RefCell::new(Vec::new()),
        }))
    }

    pub fn value(&self) -> &T {
        &self.0.value
    }

    /// Creates a child with `value` and returns a handle to it.
    pub fn add_child(&self, value: T) -> TreeNode<T> {
        let child = TreeNode::new(value);
        *child.0.parent.borrow_mut() = Rc::downgrade(&self.0);
        self.0.children.borrow_mut().push(Rc::clone(&child.0));
        child
    }

    /// Attaches an existing root as the last child of `self`.
    pub fn attach(&self, child: &TreeNode<T>) -> Result<(), AttachError> {
        if child.0.parent.borrow().upgrade().is_some() {
            return Err(AttachError::HasParent);
        }
        let mut cur = Some(self.clone());
        while let Some(n) = cur {
            if Rc::ptr_eq(&n.0, &child.0) {
                return Err(AttachError::WouldCycle);
            }
            cur = n.parent();
        }
        *child.0.parent.borrow_mut() = Rc::downgrade(&self.0);
        self.0.children.borrow_mut().push(Rc::clone(&child.0));
        Ok(())
    }

    /// `None` for a root, or when the parent has already been dropped.
    pub fn parent(&self) -> Option<TreeNode<T>> {
        self.0.parent.borrow().upgrade().map(TreeNode)
    }

    pub fn children(&self) -> Vec<TreeNode<T>> {
        self.0
            .children
            .borrow()
            .iter()
            .map(|c| TreeNode(Rc::clone(c)))
            .collect()
    }

    /// Removes this node (with its subtree) from its parent. Returns whether
    /// it was attached.
    pub fn detach(&self) -> bool {
        let Some(parent) = self.0.parent.borrow().upgrade() else {
            return false;
        };
        *self.0.parent.borrow_mut() = Weak::new();
        let mut siblings = parent.children.borrow_mut();
        let before = siblings.len();
        siblings.retain(|c| !Rc::ptr_eq(c, &self.0));
        siblings.len() != before
    }

    /// Number of ancestors (a root has depth 0). Iterative.
    pub fn depth(&self) -> usize {
        let mut d = 0;
        let mut cur = self.parent();
        while let Some(n) = cur {
            d += 1;
            cur = n.parent();
        }
        d
    }

    /// Number of nodes in this subtree, including `self`. Iterative.
    pub fn subtree_len(&self) -> usize {
        let mut count = 0;
        let mut stack = vec![self.clone()];
        while let Some(n) = stack.pop() {
            count += 1;
            stack.extend(n.children());
        }
        count
    }

    /// `(strong, weak)` counts of the underlying allocation, for diagnostics.
    pub fn counts(&self) -> (usize, usize) {
        (Rc::strong_count(&self.0), Rc::weak_count(&self.0))
    }
}

impl<T: Clone> TreeNode<T> {
    /// Values from this node up to the root.
    pub fn path_to_root(&self) -> Vec<T> {
        let mut out = vec![self.0.value.clone()];
        let mut cur = self.parent();
        while let Some(n) = cur {
            out.push(n.0.value.clone());
            cur = n.parent();
        }
        out
    }
}

// ------------------------------------------------------------ WeakRegistry

/// An interner: equal keys share one `Arc<V>` for as long as somebody holds
/// it. The registry itself holds only `Weak` pointers, so it never keeps a
/// value alive.
pub struct WeakRegistry<K, V> {
    map: Mutex<HashMap<K, ArcWeak<V>>>,
}

impl<K: Hash + Eq, V> WeakRegistry<K, V> {
    pub fn new() -> Self {
        Self {
            map: Mutex::new(HashMap::new()),
        }
    }

    fn lock(&self) -> MutexGuard<'_, HashMap<K, ArcWeak<V>>> {
        self.map.lock().unwrap_or_else(PoisonError::into_inner)
    }

    /// Returns the live value for `key`, or builds one with `make`.
    ///
    /// `make` runs with no lock held. If two threads race, both may build a
    /// value; the first to register wins and both return that one.
    pub fn get_or_insert_with(&self, key: K, make: impl FnOnce() -> V) -> Arc<V>
    where
        K: Clone,
    {
        if let Some(v) = self.lock().get(&key).and_then(ArcWeak::upgrade) {
            return v;
        }
        let fresh = Arc::new(make());
        let mut map = self.lock();
        if let Some(existing) = map.get(&key).and_then(ArcWeak::upgrade) {
            return existing;
        }
        map.insert(key, Arc::downgrade(&fresh));
        fresh
    }

    /// Removes entries whose value has been dropped; returns how many.
    pub fn prune(&self) -> usize {
        let mut map = self.lock();
        let before = map.len();
        map.retain(|_, w| w.strong_count() > 0);
        before - map.len()
    }

    /// Entries currently stored (live or dead, until pruned).
    pub fn len(&self) -> usize {
        self.lock().len()
    }

    pub fn is_empty(&self) -> bool {
        self.len() == 0
    }
}

impl<K: Hash + Eq, V> Default for WeakRegistry<K, V> {
    fn default() -> Self {
        Self::new()
    }
}

// -------------------------------------------------------------- PinnedText

/// A string plus a raw pointer to that same string inside the same struct.
///
/// The type is `!Unpin`, and the only constructor returns `Pin<Box<Self>>`,
/// so after construction the value never moves and the pointer stays valid.
///
/// ```compile_fail
/// let mut t = weaklab::PinnedText::new_unpinned_for_doctest();
/// let _ = std::pin::Pin::new(&mut t);
/// ```
pub struct PinnedText {
    data: String,
    this: *const String,
    _pin: PhantomPinned,
}

impl PinnedText {
    pub fn new(text: &str) -> Pin<Box<Self>> {
        let mut boxed = Box::pin(PinnedText {
            data: text.to_owned(),
            this: std::ptr::null(),
            _pin: PhantomPinned,
        });
        let ptr: *const String = &boxed.data;
        // SAFETY: we only write the `this` field and never move the value out.
        unsafe { boxed.as_mut().get_unchecked_mut().this = ptr };
        boxed
    }

    /// Reads the text through the self-pointer.
    pub fn text(self: Pin<&Self>) -> &str {
        // SAFETY: `this` was set to `&self.data` after pinning and the value
        // cannot have moved since; `data` is alive as long as `self` is.
        unsafe { (*self.this).as_str() }
    }

    /// Replaces the text. The pointer stays valid because `data` itself does
    /// not move; only its heap buffer may be reallocated.
    pub fn set_text(self: Pin<&mut Self>, text: &str) {
        // SAFETY: we do not move `self`, only modify a field in place.
        let this = unsafe { self.get_unchecked_mut() };
        this.data.clear();
        this.data.push_str(text);
    }

    /// True if the stored pointer still equals the address of `data`.
    pub fn is_consistent(self: Pin<&Self>) -> bool {
        std::ptr::eq(self.this, &self.data)
    }

    /// Only for the `compile_fail` doctest: returns an unpinned value.
    #[doc(hidden)]
    pub fn new_unpinned_for_doctest() -> PinnedText {
        PinnedText {
            data: String::new(),
            this: std::ptr::null(),
            _pin: PhantomPinned,
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::cell::Cell;
    use std::thread;

    struct DropCount(Rc<Cell<usize>>);
    impl Drop for DropCount {
        fn drop(&mut self) {
            self.0.set(self.0.get() + 1);
        }
    }

    #[test]
    fn parent_links_are_weak_and_children_strong() {
        let root = TreeNode::new("root");
        let a = root.add_child("a");
        let b = a.add_child("b");
        assert_eq!(b.path_to_root(), vec!["b", "a", "root"]);
        assert_eq!(b.depth(), 2);
        assert_eq!(root.counts(), (1, 1)); // one strong handle, `a` points up weakly
        assert_eq!(a.counts(), (2, 1)); // `a` handle + root's child list
        assert_eq!(root.subtree_len(), 3);
        assert_eq!(root.children().len(), 1);
    }

    #[test]
    fn tree_is_freed_exactly_once_and_has_no_cycles() {
        let drops = Rc::new(Cell::new(0));
        {
            let root = TreeNode::new(DropCount(Rc::clone(&drops)));
            let a = root.add_child(DropCount(Rc::clone(&drops)));
            a.add_child(DropCount(Rc::clone(&drops)));
            root.add_child(DropCount(Rc::clone(&drops)));
            assert_eq!(drops.get(), 0);
        }
        assert_eq!(drops.get(), 4);
    }

    #[test]
    fn orphans_see_no_parent() {
        let root = TreeNode::new(1);
        let kid = root.add_child(2);
        drop(root);
        assert!(kid.parent().is_none());
        assert_eq!(kid.depth(), 0);
        assert_eq!(kid.path_to_root(), vec![2]);
    }

    #[test]
    fn detach_and_attach() {
        let root = TreeNode::new(0);
        let a = root.add_child(1);
        let b = a.add_child(2);
        assert!(b.detach());
        assert!(!b.detach());
        assert_eq!(a.subtree_len(), 1);
        assert_eq!(b.parent().map(|p| *p.value()), None);
        assert_eq!(root.attach(&b), Ok(()));
        assert_eq!(b.depth(), 1);
        assert_eq!(root.attach(&b), Err(AttachError::HasParent));
        // a is an ancestor-free root again after detach; attaching root under
        // its own descendant would create a cycle.
        assert!(a.detach());
        let c = a.add_child(3);
        assert_eq!(c.attach(&a), Err(AttachError::WouldCycle));
        assert_eq!(a.attach(&a), Err(AttachError::WouldCycle));
    }

    #[test]
    fn detached_subtree_is_freed() {
        let drops = Rc::new(Cell::new(0));
        let root = TreeNode::new(DropCount(Rc::clone(&drops)));
        let kid = root.add_child(DropCount(Rc::clone(&drops)));
        kid.add_child(DropCount(Rc::clone(&drops)));
        assert!(kid.detach());
        assert_eq!(drops.get(), 0); // `kid` handle still alive
        drop(kid);
        assert_eq!(drops.get(), 2);
        drop(root);
        assert_eq!(drops.get(), 3);
    }

    #[test]
    fn a_million_deep_chain_drops_without_overflow() {
        let root = TreeNode::new(0u32);
        let mut cur = root.clone();
        for i in 1..=1_000_000u32 {
            cur = cur.add_child(i);
        }
        assert_eq!(cur.depth(), 1_000_000);
        drop(cur);
        drop(root); // would overflow the stack with a recursive drop
    }

    #[test]
    fn registry_shares_live_values_and_forgets_dead_ones() {
        let reg: WeakRegistry<&str, String> = WeakRegistry::new();
        let a = reg.get_or_insert_with("k", || "first".to_string());
        let b = reg.get_or_insert_with("k", || unreachable!("must reuse"));
        assert!(Arc::ptr_eq(&a, &b));
        assert_eq!(Arc::strong_count(&a), 2);
        drop((a, b));
        assert_eq!(reg.len(), 1); // dead entry still stored
        let c = reg.get_or_insert_with("k", || "second".to_string());
        assert_eq!(*c, "second"); // dead entry replaced
        drop(c);
        assert_eq!(reg.prune(), 1);
        assert!(reg.is_empty());
    }

    #[test]
    fn registry_racing_threads_get_the_same_value() {
        for _ in 0..50 {
            let reg: WeakRegistry<u8, usize> = WeakRegistry::new();
            let got: Vec<Arc<usize>> = thread::scope(|s| {
                let hs: Vec<_> = (0..8usize)
                    .map(|t| {
                        let reg = &reg;
                        s.spawn(move || reg.get_or_insert_with(1, || t))
                    })
                    .collect();
                hs.into_iter().map(|h| h.join().unwrap()).collect()
            });
            assert!(got.iter().all(|a| Arc::ptr_eq(a, &got[0])));
        }
    }

    #[test]
    fn pinned_text_stays_consistent_when_the_box_moves() {
        let mut t = PinnedText::new("hello");
        assert!(t.as_ref().is_consistent());
        assert_eq!(t.as_ref().text(), "hello");
        t.as_mut().set_text(&"x".repeat(10_000)); // forces reallocation of the buffer
        assert!(t.as_ref().is_consistent());
        assert_eq!(t.as_ref().text().len(), 10_000);
        // Moving the Pin<Box<_>> moves only the box pointer.
        let mut v = vec![t];
        for i in 0..100 {
            v.push(PinnedText::new(&i.to_string()));
        }
        v.reverse();
        for p in &v {
            assert!(p.as_ref().is_consistent());
        }
        assert_eq!(v[0].as_ref().text(), "99");
        assert_eq!(v[100].as_ref().text().len(), 10_000);
    }

    #[test]
    fn auto_traits() {
        fn send_sync<T: Send + Sync>() {}
        send_sync::<WeakRegistry<String, String>>();
    }
}
```

✓ With this exact file: `cargo fmt --check` clean, `cargo clippy --all-targets --locked -- -D warnings` clean, and `cargo test --locked` → 10 unit tests, 1 passing doctest and 2 `compile_fail` doctests passed. In a separate crate that depends on `weaklab` I confirmed ✓ that the doctest errors are the intended ones: `E0277` "`Rc<weaklab::Inner<u8>>` cannot be sent between threads safely" and `E0277` "`PhantomPinned` cannot be unpinned". **Miri was not available, so `PinnedText`'s `unsafe` code is tested but not UB-checked.**

Mutation checks (each applied to a copy, then `cargo test --lib`):

| Change | Result ✓ |
|---|---|
| `Inner::drop` does nothing (recursive drop) | the million-deep test **aborted** with "has overflowed its stack" (SIGABRT) |
| `detach` does not clear the node's `parent` | `detach_and_attach` FAILED |
| `attach` skips the ancestor check | `detach_and_attach` FAILED |
| `PinnedText::new` sets the self-pointer *before* `Box::pin` moves the value | `pinned_text_stays_consistent_when_the_box_moves` FAILED (`is_consistent` is checked before any dereference) |
| `WeakRegistry` skips its second check under the write lock | **not caught**: the racing test passed in 4 separate runs (the window is tiny), so that test is a smoke test, not proof; the second check is justified by reasoning |

**Design notes.**

- **Ownership direction.** Parents own children (`Rc`), children point to parents with `Weak`; `attach` walks the new parent's ancestors to refuse a cycle, because with `Weak` a cycle would no longer leak but would make `depth` and `path_to_root` loop forever.
- **Iterative `Drop`.** `Inner::drop` moves the children into a work list and unwraps them with `Rc::try_unwrap`; a node that is still shared (someone holds a handle) keeps its own subtree, and the loop just drops that `Rc`. This is the Lesson 23 trick applied to a tree.
- **`get_or_insert_with` avoids holding the lock during `make`** (Lesson 24), so two racing threads may both construct a value; whoever registers first wins, and the other `Arc` is dropped.
- **The registry holds only `Weak`**, so it never keeps a value alive; entries for dead values linger until `prune` (✓ demonstrated in the tests: `len` is still 1 after both `Arc`s are dropped).
- **`PinnedText`** shows the pattern: private constructor returns `Pin<Box<Self>>`, pointer set after pinning, `get_unchecked_mut` used only to write a field in place, accessors take `Pin<&Self>`/`Pin<&mut Self>`. The self-pointer is redundant (you could just read `data`), which is the point of the lesson: use real self-reference only where unavoidable.
- **Not covered:** structural pinning/projection, thread-safe trees (`Arc` plus locks), arena-based alternatives, and cycle detection beyond `attach`.

</details>

---

## 9. Interview questions

<details markdown="1">
<summary>Answers</summary>

1. **What is `Weak<T>` and why does it exist?** A non-owning pointer to an `Rc`/`Arc` allocation: it doesn't keep the value alive. It breaks reference cycles (parent/child, observers, caches).
2. **How do you use a `Weak`?** `upgrade()` returns `Option<Rc<T>>`: `Some` while the value lives, `None` after (✓).
3. **Strong vs weak count?** The strong count decides when the value is dropped; the weak count decides when the allocation is freed. `Rc::downgrade` raises the weak count only (✓ `1 1`).
4. **When is the memory freed if a `Weak` remains?** The value is dropped when the strong count hits zero, but the block stays until the last `Weak` is gone (✓ 1016 bytes retained).
5. **Does `Weak::new()` allocate?** No (✓ 0 bytes, 0 allocations); `upgrade()` always gives `None`.
6. **How do you model parent/child trees?** Strong parent → child (`Vec<Rc<Node>>`), weak child → parent (`RefCell<Weak<Node>>`) (✓ drop order `root`, then `kid`).
7. **Is `Weak` thread-safe?** `std::sync::Weak` (for `Arc`) yes, if `T: Send + Sync`; `std::rc::Weak` is `!Send` (✓ E0277).
8. **Why is `strong_count()` followed by `upgrade().unwrap()` wrong with `Arc`?** The value can disappear in between; just match on `upgrade()`.
9. **What problem does `Pin` solve?** Moves are memcpys, so a value with a pointer into itself breaks if moved. `Pin<P>` forbids moving the pointee of `!Unpin` types (✓ the stale-pointer example).
10. **What is `Unpin`?** An auto trait meaning moving the type is harmless even when pinned; most types are `Unpin`. `PhantomPinned` opts out (✓ matrix).
11. **Why does `Pin::new(&mut x)` fail for some types?** `Pin::new` requires `Unpin` (✓ E0277 "cannot be unpinned"); use `Box::pin` or `pin!`.
12. **Why can't you assign through `Pin<Box<T>>` when `T: !Unpin`?** `DerefMut` for `Pin` requires `T: Unpin` (✓ E0594/E0596); handing out `&mut T` would allow `mem::swap`.
13. **Does `Pin<Box<T>>` make the `Box` immovable?** No, the pointer can move; the pointee stays put.
14. **Why does `Future::poll` take `Pin<&mut Self>`?** Compiled async state machines may hold references into themselves across `.await`; they must not move after the first poll (✓ async blocks are `!Unpin`).
15. **What is `get_unchecked_mut`?** An `unsafe` method yielding `&mut T` from `Pin<&mut T>`; you promise not to move the value out (writing fields in place is fine).
16. **Alternatives to self-referential structs?** Indices/offsets instead of pointers, arenas, splitting owner and borrower, `Rc`/`Weak`.
17. **Is `Pin` a security feature?** No; it prevents accidental moves at compile time, and misuse of its `unsafe` parts is undefined behavior.

</details>

---

## 10. Key takeaways

- `Weak<T>` is the non-owning partner of `Rc`/`Arc`: `upgrade()` gives `Option<Rc<T>>`. Owners are strong, back-references are weak. It fixes the leak from Lesson 23 (✓ `drop b`, `drop a` appear) and models parent links and caches.
- The strong count controls when the **value** is dropped; the **allocation** lives until the last `Weak` is gone (✓ 1016 bytes retained). `Weak::new()` and `downgrade` allocate nothing (✓).
- `upgrade()` can fail at any time (with `Arc`, between any two statements): match on it once, never `unwrap` it blindly.
- Moves are `memcpy`; a value with a pointer to itself breaks when moved (✓ stale pointer). `Pin<P>` makes `!Unpin` values immovable: no `DerefMut` (✓ E0594/E0596), `Pin::new` only for `Unpin` (✓ E0277), create with `Box::pin` / `pin!`. `Unpin` is auto; `PhantomPinned` opts out; `Vec<Fixed>`, `Option<Fixed>` and `async` blocks are `!Unpin` (✓).
- You consume `Pin` mostly through `Future::poll` (Lessons 30 and 31); hand-written `unsafe` pin code needs Miri (not available here) and extreme care.

**Connections.** Lesson 23's cycle leak is now fixed, Lesson 24's self-capturing event bus gets its `Weak` cure, Lesson 22's `Send`/`Sync` explains `Weak`'s error, and Lesson 21's trait objects reappear as `Pin<Box<dyn Future>>`. Next: Lesson 26 (error design) starts Module 9 and returns to `Result` at library scale; Lesson 30 uses `Pin` in `Future`, Lesson 36 covers raw pointers and `unsafe` properly, Lesson 38 covers Miri. See the [roadmap](00_roadmap.md) and the [course index](index.md).

---

## 11. References

- *The Rust Programming Language*, ch. 15.6 "Reference Cycles Can Leak Memory" (`Weak<T>`, the tree example): <https://doc.rust-lang.org/book/ch15-06-reference-cycles.html>
- `std::rc::Weak`, `std::sync::Weak`, `Rc::downgrade`, `Rc::weak_count`: <https://doc.rust-lang.org/std/rc/struct.Weak.html>, <https://doc.rust-lang.org/std/sync/struct.Weak.html>
- `std::pin` module documentation (the pinning contract, `Unpin`, `pin!`, structural pinning): <https://doc.rust-lang.org/std/pin/index.html>
- `std::marker::PhantomPinned`, `std::marker::Unpin`: <https://doc.rust-lang.org/std/marker/struct.PhantomPinned.html>
- *Asynchronous Programming in Rust* (the async book), the chapter on pinning: <https://rust-lang.github.io/async-book/>
- Rust error code index: E0277, E0594, E0596: <https://doc.rust-lang.org/error_codes/E0277.html>

The URLs above are the standard documentation locations; I did not open them during this session (the sandbox cannot reach them), so if a link has moved, search the docs site for the page title. Unchecked from memory: the Book chapter number and anchor, the async book's URL and chapter names, the exact wording of the pinning contract and the structural-pinning rules (summarized, not quoted), that `Weak<T>` is pointer-sized for sized `T` (not printed), the claim that `pin-project`-style crates are the common way to project pins, the statement that `Vec<T>: Unpin` only when `T: Unpin` (✓ observed for `Vec<Fixed>`, but the general rule is from memory), that a plain move may reuse a stack slot in an optimized build, the cost of atomic counters (not measured), and the Lesson 22, 23 and 24 pointers (referenced by title). Miri was not run on any `unsafe` code.
