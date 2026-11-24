---
layout: default
title: "Lesson 23 — Box, Rc, and Arc: owning pointers"
---

# Lesson 23 — `Box`, `Rc`, and `Arc`: owning pointers

**Module 8 · Smart pointers** · Status: generated · Edition used in examples: **2024** · External dependencies: none

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux. As before, ✓ marks outputs, diagnostics and measurements I actually ran; unmarked statements are documented behavior I did not run. Everything is **stable Rust**.

The ownership and move rules come from Lessons 07 to 09. Direct prerequisites: [Lesson 18](18_closures.md) (`Box<dyn Fn>`), [Lesson 21](21_trait_objects_dispatch.md) (`Box<dyn Trait>`, fat pointers) and [Lesson 22](22_gats_send_sync_phantom.md) (`Send`/`Sync`, GATs, `Rc` versus `Arc` thread-safety).

---

## 1. Why smart pointers

Plain ownership gives every value exactly one owner and every owner a stack slot of known size. Three situations break that:

1. **A value must live on the heap** or has a size the compiler can't determine (a recursive type, a trait object, a large buffer you don't want to copy).
2. **A value needs several owners** and should be freed when the last one goes away (shared nodes in a graph, a configuration read by many components).
3. **Several owners on several threads.**

Rust has one standard answer for each: `Box<T>`, `Rc<T>`, `Arc<T>`. All three are **owning** smart pointers: they free the heap allocation when the last owner is dropped (RAII, Lesson 08). They implement `Deref`, so `*p` and method calls reach the inner `T`.

| | Owners | Allocation | Thread-safe | Mutate the inner value |
|---|---|---|---|---|
| `Box<T>` | exactly one | one heap block holding `T` | `Send`/`Sync` as `T` | yes, through `&mut Box<T>` |
| `Rc<T>` | many | one block: two counters plus `T` | no (`!Send`, `!Sync`) | not through `&Rc<T>`; `Rc::get_mut`/`make_mut`, or interior mutability (Lesson 24) |
| `Arc<T>` | many | the same, with atomic counters | yes, if `T: Send + Sync` | same as `Rc` |

---

## 2. `Box<T>`: one owner on the heap

```rust
use std::mem::size_of;
#[derive(Debug)]
enum List { Cons(i32, Box<List>), Nil }
use List::{Cons, Nil};
fn sum(l: &List) -> i32 { match l { Cons(v, rest) => v + sum(rest), Nil => 0 } }
fn main() {
    let l = Cons(1, Box::new(Cons(2, Box::new(Cons(3, Box::new(Nil))))));
    println!("{} {}", sum(&l), size_of::<List>());
    println!("{} {} {} {}",
        size_of::<Box<u64>>(), size_of::<Option<Box<u64>>>(), size_of::<Box<[u8]>>(), size_of::<Box<dyn Fn()>>());
}
```

✓ Output `6 16`, then `8 8 16 16`. A `Box<T>` is one pointer (8 bytes), two for an unsized pointee (`[u8]`, `dyn Fn()`: a fat pointer, Lessons 21 and 22). `Option<Box<u64>>` is also 8 bytes: the box is never null, so `None` is the null pattern (the niche from Lesson 09).

### 2.1 Recursive types

Without indirection the compiler cannot compute a size ✓:

```rust
enum List { Cons(i32, List), Nil }
fn main() {}
```

✓ `error[E0072]: recursive type `List` has infinite size`, with the help "insert some indirection (e.g., a `Box`, `Rc`, or `&`) to break the cycle". `Cons(i32, Box<List>)` has a known size: an `i32` plus a pointer, 16 bytes with padding ✓. Trees and linked lists use `Option<Box<Node>>` (`None` is the end).

### 2.2 Moving in and out, and deref coercion

`Box` is special: you can move the value out with `*b`. `Rc` does not allow it ✓:

```rust
use std::rc::Rc;
fn main() {
    let b = Box::new(String::from("boxed"));
    let s: String = *b;           // moving out of a Box is allowed
    let r = Rc::new(String::from("rc"));
    let t: String = *r;           // but not out of an Rc
    println!("{s} {t}");
}
```

✓ `error[E0507]: cannot move out of an `Rc``. (An `Rc` may have other owners, so it cannot give the value away; `Rc::try_unwrap` is the controlled way, §3.) Using a box after moving it is the ordinary E0382 ✓: `let c = b; println!("{b}");` gives `error[E0382]: borrow of moved value: `b``. `b.clone()` makes a deep copy (a new allocation) when `T: Clone`.

**Deref coercion** (Lesson 11) lets a `&Box<String>` or `&Rc<String>` be passed where `&str` is expected ✓:

```rust
use std::rc::Rc;
fn print_len(s: &str) -> usize { s.len() }
fn main() {
    let b = Box::new(String::from("boxed"));
    let r = Rc::new(String::from("rc!"));
    println!("{} {} {}", print_len(&b), print_len(&r), b.to_uppercase());
    let leaked: &'static mut Vec<i32> = Box::leak(Box::new(vec![1, 2]));
    leaked.push(3);
    println!("{leaked:?}");
}
```

✓ Output `5 3 BOXED` and `[1, 2, 3]`. `&Box<String>` becomes `&String` becomes `&str` by repeated `Deref`. `Box::leak` deliberately gives up ownership and returns a `&'static mut T`: the allocation is never freed (useful for process-lifetime configuration, a memory leak anywhere else).

### 2.3 What `Box` really costs

I measured allocations with a counting `#[global_allocator]` that adds up the bytes requested ✓:

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::{rc::Rc, sync::Arc};
struct Counting;
static BYTES: AtomicUsize = AtomicUsize::new(0);
static CALLS: AtomicUsize = AtomicUsize::new(0);
unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        BYTES.fetch_add(l.size(), Relaxed);
        CALLS.fetch_add(1, Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) { unsafe { System.dealloc(p, l) } }
}
#[global_allocator]
static A: Counting = Counting;
fn measure<R>(label: &str, f: impl FnOnce() -> R) -> R {
    let (b0, c0) = (BYTES.load(Relaxed), CALLS.load(Relaxed));
    let r = f();
    println!("{label:<28} {:>3} bytes in {} allocation(s)", BYTES.load(Relaxed) - b0, CALLS.load(Relaxed) - c0);
    r
}
fn main() {
    let _b = measure("Box::new(1u64)", || Box::new(1u64));
    let rc = measure("Rc::new(1u64)", || Rc::new(1u64));
    let arc = measure("Arc::new(1u64)", || Arc::new(1u64));
    let _ = measure("Rc::clone", || Rc::clone(&rc));
    let _ = measure("Arc::clone", || Arc::clone(&arc));
    let _ = measure("Box::new([0u64; 4])", || Box::new([0u64; 4]));
    let _ = measure("Rc::new(1u8)", || Rc::new(1u8));
    let _ = measure("Rc<str> from \"hello\"", || Rc::<str>::from("hello"));
    let _ = measure("Box::new(()) (zero-sized)", || Box::new(()));
}
```

✓ Output:

```text
Box::new(1u64)                 8 bytes in 1 allocation(s)
Rc::new(1u64)                 24 bytes in 1 allocation(s)
Arc::new(1u64)                24 bytes in 1 allocation(s)
Rc::clone                      0 bytes in 0 allocation(s)
Arc::clone                     0 bytes in 0 allocation(s)
Box::new([0u64; 4])           32 bytes in 1 allocation(s)
Rc::new(1u8)                  24 bytes in 1 allocation(s)
Rc<str> from "hello"          24 bytes in 1 allocation(s)
Box::new(()) (zero-sized)      0 bytes in 0 allocation(s)
```

What it shows: a `Box` allocates exactly the pointee (8 bytes for a `u64`); a **zero-sized** pointee allocates nothing; `Rc` and `Arc` allocate **24 bytes** even for a `u64` or a `u8`, consistent with two machine-word counters (strong and weak) in front of the value (16 bytes of counts plus the value, rounded to alignment; I inferred the layout from these totals, I did not read the source); and `clone()` of an `Rc`/`Arc` allocates **nothing**: it only bumps a counter. (`Rc<str>` from `"hello"` is 24 bytes: 16 for the counters plus 5 for the text, rounded up to 8.) The measurement is for this toolchain and target, not a language guarantee.

### 2.4 Drop order

```rust
struct Noisy(&'static str);
impl Drop for Noisy { fn drop(&mut self) { println!("drop {}", self.0); } }
struct Pair { _first: Noisy, boxed: Box<Noisy>, _last: Noisy }
fn main() {
    let _p = Pair { _first: Noisy("first"), boxed: Box::new(Noisy("boxed")), _last: Noisy("last") };
    let moved = Box::new(Noisy("moved"));
    let _taken = *moved;
    println!("end of main");
}
```

✓ Output:

```text
end of main
drop moved
drop first
drop boxed
drop last
```

Locals drop in **reverse** order of declaration (`_taken` first; `moved` is an emptied box, only its allocation is released), and the fields of a struct drop in **declaration** order, with a `Box` field dropping its contents before freeing its memory. (Lesson 08.)

### 2.5 A tree: `Option<Box<Node>>` and the cursor idiom

```rust
#[derive(Debug)]
struct Node { key: i32, left: Option<Box<Node>>, right: Option<Box<Node>> }
fn insert(root: &mut Option<Box<Node>>, key: i32) -> bool {
    let mut cur = root;
    while let Some(node) = cur {
        if key == node.key { return false; }
        cur = if key < node.key { &mut node.left } else { &mut node.right };
    }
    *cur = Some(Box::new(Node { key, left: None, right: None }));
    true
}
fn inorder(n: &Option<Box<Node>>, out: &mut Vec<i32>) {
    if let Some(b) = n { inorder(&b.left, out); out.push(b.key); inorder(&b.right, out); }
}
fn main() {
    let mut root = None;
    let ins: Vec<bool> = [5, 3, 8, 1, 4, 3].iter().map(|&k| insert(&mut root, k)).collect();
    let mut v = vec![]; inorder(&root, &mut v);
    println!("{ins:?} {v:?}");
}
```

✓ Output `[true, true, true, true, true, false] [1, 3, 4, 5, 8]`. The loop keeps a `&mut Option<Box<Node>>` "cursor" and walks it down to the empty slot, then fills it. No recursion, no `unsafe`, and the borrow checker accepts it because each step reborrows from the node it just matched. Note `inorder` recurses: a degenerate (sorted-input) tree of a million nodes would overflow the stack there too, and so does the *drop* of such a tree, which brings us to the next point.

### 2.6 The deep-drop trap

Dropping a `Box` list is recursive: dropping a node drops its `next`, which drops its `next`, and so on, one stack frame per node. I built a list of `n` nodes and dropped it (8 MiB stack, `ulimit -s` was 8192) ✓:

```rust
struct Node { _value: u32, next: Option<Box<Node>> }
fn main() {
    let n: usize = std::env::args().nth(1).unwrap().parse().unwrap();
    let mut head: Option<Box<Node>> = None;
    for i in 0..n { head = Some(Box::new(Node { _value: i as u32, next: head })); }
    println!("built {n}");
    drop(head);
    println!("dropped {n}");
}
```

✓ Results: `n = 10 000` and `n = 100 000` worked in both debug and `-O` builds; at `n = 1 000 000` **both builds aborted** (exit status 134) with "thread 'main' has overflowed its stack", after printing `built 1000000`, and the same at 5 000 000. The fix is a `Drop` impl that unlinks the chain in a loop ✓:

```rust
struct Node { _value: u32, next: Option<Box<Node>> }
impl Drop for Node {
    fn drop(&mut self) {
        let mut cur = self.next.take();
        while let Some(mut node) = cur { cur = node.next.take(); }
    }
}
fn main() {
    let n: usize = 1_000_000;
    let mut head: Option<Box<Node>> = None;
    for i in 0..n { head = Some(Box::new(Node { _value: i as u32, next: head })); }
    drop(head);
    println!("dropped {n}");
}
```

✓ Output `dropped 1000000` (and `5 000 000` also finished, exit 0). Each loop iteration detaches one node's `next` before that node is dropped, so no node ever has a non-empty `next` when its own `drop` runs. This is also how the standard library's `LinkedList` avoids the problem (from memory). The practical challenge's `Stack` does exactly this and tests a million nodes.

---

## 3. `Rc<T>`: many owners, one thread

`Rc` ("reference counted") keeps a **strong count** of owners. `Rc::clone(&a)` increments it (no data is copied, ✓ no allocation above); dropping an `Rc` decrements it; at zero the value is dropped and the memory freed.

```rust
use std::rc::Rc;
fn main() {
    let a = Rc::new(String::from("shared"));
    println!("{}", Rc::strong_count(&a));
    let b = Rc::clone(&a);
    {
        let c = a.clone();
        println!("{} {} {}", Rc::strong_count(&a), Rc::ptr_eq(&a, &b), Rc::ptr_eq(&a, &c));
    }
    println!("{}", Rc::strong_count(&a));
    let d = Rc::new(String::from("shared"));
    println!("{} {}", a == d, Rc::ptr_eq(&a, &d));
    println!("{:?}", Rc::try_unwrap(a).map_err(|rc| Rc::strong_count(&rc)));
    println!("{:?}", Rc::try_unwrap(b));
}
```

✓ Output:

```text
1
3 true true
2
true false
Err(2)
Ok("shared")
```

`Rc::clone(&a)` is the idiomatic spelling (not `a.clone()`) because it signals "cheap pointer copy" to the reader. `==` compares the *values*; `Rc::ptr_eq` compares the *allocations* (`a == d` but not `ptr_eq`). `Rc::try_unwrap` returns the value only if this is the last owner (`Err(rc)` otherwise: there were still 2 strong owners, one of which, `b`, is then the last).

### 3.1 Shared means read-only

An `Rc` hands out only `&T`. Mutating through it is an error ✓:

```rust
use std::rc::Rc;
fn main() {
    let r = Rc::new(vec![1, 2]);
    r.push(3);
}
```

✓ `error[E0596]: cannot borrow data in an `Rc` as mutable`. Two ways to get mutation, both ✓:

```rust
use std::rc::Rc;
fn main() {
    let mut a = Rc::new(vec![1, 2]);
    Rc::get_mut(&mut a).unwrap().push(3);
    let b = Rc::clone(&a);
    println!("{:?}", Rc::get_mut(&mut a).is_none());
    Rc::make_mut(&mut a).push(4);          // shared: clones the Vec first
    println!("{a:?} {b:?} {}", Rc::ptr_eq(&a, &b));
    Rc::make_mut(&mut a).push(5);          // now unique: no clone
    println!("{a:?}");
}
```

✓ Output `true`, then `[1, 2, 3, 4] [1, 2, 3] false`, then `[1, 2, 3, 4, 5]`. `Rc::get_mut` gives `&mut T` **only if unique** (returns `None` once `b` exists). `Rc::make_mut` is **clone-on-write**: if shared it first clones the value into a new allocation, so `a` and `b` diverge; if unique it mutates in place with no copy (an `Rc<u64>` run printed `8 7 false` for the value 7 incremented through `make_mut` while another handle existed, and the earlier allocation counter showed zero allocations in the unique case). It needs `T: Clone`. The other way, **interior mutability** (`Rc<RefCell<T>>`), moves the borrow check to run time and is Lesson 24.

### 3.2 Cycles leak

Counts only reach zero if ownership is a DAG. Two `Rc`s pointing at each other keep each other alive forever ✓:

```rust
use std::cell::RefCell;
use std::rc::Rc;
struct Noisy(&'static str);
impl Drop for Noisy { fn drop(&mut self) { println!("drop {}", self.0); } }
struct Node { _name: Noisy, next: RefCell<Option<Rc<Node>>> }
fn main() {
    {
        let a = Rc::new(Node { _name: Noisy("a"), next: RefCell::new(None) });
        let b = Rc::new(Node { _name: Noisy("b"), next: RefCell::new(Some(Rc::clone(&a))) });
        *a.next.borrow_mut() = Some(Rc::clone(&b));
        println!("{} {}", Rc::strong_count(&a), Rc::strong_count(&b));
    }
    println!("scope ended");
    {
        let c = Rc::new(Node { _name: Noisy("c"), next: RefCell::new(None) });
        let d = Rc::new(Node { _name: Noisy("d"), next: RefCell::new(Some(Rc::clone(&c))) });
        println!("{} {}", Rc::strong_count(&c), Rc::strong_count(&d));
    }
    println!("end");
}
```

✓ Output:

```text
2 2
scope ended
2 1
drop d
drop c
end
```

The cycle `a` ⇄ `b` printed **no** `drop` lines: both are leaked (each count stays at 1 after the locals are gone, so neither reaches zero). The acyclic pair `c`, `d` dropped normally. (`RefCell` is used only to build the cycle; Lesson 24.) Leaking is memory-safe, so the compiler does not stop it. `Weak<T>` (Lesson 25) is the non-owning pointer you use for back-edges.

---

## 4. `Arc<T>`: many owners, many threads

`Arc` ("atomically reference counted") has the same API with atomic counter updates, so clones may be sent to other threads (when `T: Send + Sync`, Lesson 22) ✓:

```rust
use std::sync::{Arc, Mutex};
fn main() {
    let names: Arc<str> = Arc::from("shared name");
    let total = Arc::new(Mutex::new(0u32));
    let hs: Vec<_> = (1..=4).map(|i| {
        let (names, total) = (Arc::clone(&names), Arc::clone(&total));
        std::thread::spawn(move || { *total.lock().unwrap() += i; names.len() })
    }).collect();
    let lens: Vec<usize> = hs.into_iter().map(|h| h.join().unwrap()).collect();
    println!("{lens:?} {} {} {}", total.lock().unwrap(), Arc::strong_count(&names), Arc::strong_count(&total));
    let mut data = Arc::new(vec![1, 2, 3]);
    let snapshot = Arc::clone(&data);
    Arc::make_mut(&mut data).push(4);
    println!("{data:?} {snapshot:?}");
}
```

✓ Output `[11, 11, 11, 11] 10 1 1` and `[1, 2, 3, 4] [1, 2, 3]`. Four threads each held a clone, so after joining them the strong counts are back to 1. `Arc<str>` shares an immutable string without a `String` per owner. `Arc` does **not** make `T` mutable or thread-safe: `Arc<Vec<i32>>` is read-only, `Arc<Mutex<T>>` is how you mutate (Lesson 29 covers locks), and `Arc::make_mut` gives copy-on-write snapshots, as above.

Notes on using `Arc` correctly:

- Use `Rc` when everything runs on one thread: atomic read-modify-write operations on every clone and drop are the price of `Arc`. How much that costs depends on the CPU and contention; I did not measure it (Lesson 41).
- `Arc::strong_count` is a snapshot that can be stale the moment you read it with other threads running. Use it for diagnostics and tests, not for decisions.
- To take the value out when you may be racing with other droppers, prefer `Arc::into_inner` (and `Rc::into_inner`) to `try_unwrap`: when several clones are dropped concurrently, `into_inner` guarantees that exactly one of them gets the value, while `try_unwrap` can fail for all of them (from the documentation, remembered; see the note in §7).
- A thread-safe pointer can still contain a non-thread-safe value; the compiler checks `T: Send + Sync` for you (Lesson 22).

---

## 5. Choosing a pointer

| Situation | Use |
|---|---|
| One owner, recursive type, large value, trait object, or an owned slice/str that never grows | `Box<T>`, `Box<dyn Trait>`, `Box<[T]>`, `Box<str>` |
| Just looking at data owned elsewhere | `&T` / `&mut T`, not a smart pointer |
| Shared read-only data, one thread (graphs, caches, immutable trees) | `Rc<T>`, `Rc<str>`, `Rc<[T]>` |
| The same, across threads | `Arc<T>` |
| Shared mutable data | `Rc<RefCell<T>>` (one thread, Lesson 24), `Arc<Mutex<T>>` / `Arc<RwLock<T>>` (threads, Lesson 29) |
| Parent pointers / back-edges / caches that must not keep things alive | `Weak<T>` (Lesson 25) |
| "I only need shared access while this function runs" | a reference or `thread::scope` (Lesson 22), no refcount at all |

A good habit: reach for `&T` first, `Box` second, and `Rc`/`Arc` only when ownership genuinely has several owners with unpredictable lifetimes.

---

## 6. Structural sharing: persistent lists

Immutable data plus `Rc`/`Arc` gives **persistent** data structures: "modifying" a list returns a new one that reuses almost all of the old one. The practical challenge's `PList` shows it: `base.prepend(1)` and `base.prepend(10)` are two lists sharing every node of `base`. That is safe precisely because the nodes are never mutated, so there is no aliasing problem, and cheap because prepending allocates one node. It also uses the generic-associated-type pointer family from Lesson 22, so one implementation serves `Rc` and `Arc`.

---

## 7. Common mistakes and edge cases

- **Recursive types without indirection** (E0072 ✓). Use `Box` (or `Vec`/`Option<Box<_>>`).
- **Trying to mutate through an `Rc`/`Arc`** (E0596 ✓/E0594 ✓): use `get_mut`/`make_mut`, or interior mutability.
- **Moving out of an `Rc`** (E0507 ✓): clone the inner value, or `try_unwrap`/`into_inner`.
- **Reference cycles** (leak ✓). Use `Weak` for back-references (Lesson 25).
- **`Rc` across threads** (E0277 "cannot be sent", Lesson 22): use `Arc`.
- **Recursive drop of deep structures** (✓ stack overflow at a million nodes): write an iterative `Drop` or use a `Vec`/arena.
- **Recursive traversal of deep structures** (`inorder`, `sum`): the same overflow risk; use an explicit stack.
- **`a.clone()` on an `Rc<Vec<T>>` expecting a deep copy**: it only bumps a count. A deep copy is `(*a).clone()` or `Rc::make_mut`.
- **Using `Arc::strong_count` for logic** (stale snapshot).
- **`Rc::try_unwrap(...).unwrap()` in code that can be shared**: it panics when another owner exists; use `into_inner` or handle the `Err`.
- **Over-sharing**: wrapping everything in `Rc<RefCell<_>>` to "make the borrow checker go away" usually signals a design that wants a different ownership structure (indices into a `Vec`, a slab like Lesson 22's, or message passing).
- **`Box<T>` where `T` is small** just to be "safe": you pay an allocation and an indirection for nothing.
- **`Box::leak` / `into_raw`** without a plan for freeing: a deliberate leak or an unsafe obligation (Lesson 36).

---

## 8. Performance implications

- `Box::new` is one allocation (✓ 8 bytes for a `u64`) and a free on drop; moving a `Box` moves 8 (or 16) bytes, not the pointee, which is why a `Box` is used to avoid copying a large value around.
- `Rc::new`/`Arc::new` are one allocation each (✓ 24 bytes for small values: counters plus value). `clone` is allocation-free (✓) and costs a counter increment (atomic for `Arc`).
- Pointer chasing: a linked structure of `Box`/`Rc` nodes scatters data across the heap; iterating a `Vec<T>` is usually much faster than walking a `Box` list because of cache locality (Lesson 16). Prefer contiguous storage unless you need cheap splicing or sharing.
- `Rc<[T]>`/`Rc<str>`/`Arc<str>` keep the data and the counters in a single allocation (✓ `Rc<str>` 24 bytes for 5 characters).
- Copy-on-write (`make_mut`) copies only when shared (✓ no allocation when unique).
- Dropping large structures costs time proportional to their size and, if recursive, stack depth (§2.6).

---

## 9. Security and reliability

- **Untrusted depth is a denial-of-service vector.** If input can build a deeply nested structure (nested JSON, a long chain of nodes), the recursive drop, `Debug`, `Clone` or traversal of it can overflow the stack and abort the process (✓ a million-node `Box` list did). Limit nesting at parse time, use iterative algorithms, and write iterative `Drop` for self-referential containers. A stack overflow is an abort, not a catchable panic.
- **Leaks are safe but exhaust memory.** An `Rc` cycle (✓) or a forgotten `Box::leak` loses memory forever; an attacker who can trigger one per request can exhaust memory. Prefer acyclic ownership or `Weak`.
- **Shared handles hide aliasing.** With `Rc<RefCell<_>>`/`Arc<Mutex<_>>`, a bug in one component affects every holder; keep shared mutable state small and well-encapsulated.
- **Don't branch on counts across threads** (stale snapshot, §4), and in code racing on the last owner use `into_inner` (the concurrent-drop test in the challenge passed 200 rounds with it; replacing it with `Arc::try_unwrap(..).ok()` did **not** make my test fail in three tries, so that test is a smoke test for the race, not a proof; the guarantee comes from the documentation).
- **Persistent structures and secrets.** Structural sharing means "old" versions stay alive while any handle exists; if a list held a secret, old handles keep it in memory. Zeroization is a separate tool (Lesson 48).
- **`Box::into_raw`/`from_raw`** (not used here) must be paired exactly once; a double `from_raw` is a double free (Lesson 36).
- **Bound sizes**: `Box::new([0u8; N])` with attacker-chosen `N` is a compile-time constant, but `vec![0; n]` / `Box<[T]>` built from runtime lengths needs a limit (Lesson 16).

---

## 10. Exercises

Try each before opening the answer. They are not marked complete unless you tell me you did them.

1. **(Easy)** Fix `enum List { Cons(i32, List), Nil }` and say how big the fixed enum is on a 64-bit target.
2. **(Easy)** Predict the bytes and allocations for `Box::new(1u64)`, `Rc::new(1u64)`, `Rc::clone(&rc)` and `Box::new(())`.
3. **(Medium)** Why does `let t: String = *r;` fail for `r: Rc<String>` but `let s: String = *b;` work for `b: Box<String>`? What are two correct ways to get the `String` out of an `Rc`?
4. **(Medium)** `let r = Rc::new(vec![1, 2]); r.push(3);` fails. Give two fixes and predict the output of `make_mut` on a shared `Rc<Vec<i32>>` followed by a second `make_mut`.
5. **(Medium)** Explain why the nodes `a` and `b` in the cycle example never print `drop`, and say what you would change.
6. **(Hard)** Write a binary-search-tree `insert` over `Option<Box<Node>>` without recursion.
7. **(Hard)** A `Box` linked list of 1,000,000 nodes aborts when dropped. Why, and what is the fix?
8. **(Hard)** You need to share a read-only configuration (a `String`) among four worker threads and let a fifth thread build an updated copy. Which pointer and which API?

<details markdown="1">
<summary>Answers</summary>

1. ✓ `enum List { Cons(i32, Box<List>), Nil }` (the original fails with E0072). `size_of::<List>()` is **16** (✓): an `i32` plus an 8-byte pointer plus padding; `Nil` uses the box's null niche, so the enum needs no extra tag.
2. ✓ `Box::new(1u64)`: 8 bytes, 1 allocation. `Rc::new(1u64)`: 24 bytes, 1 allocation (two counters plus the value). `Rc::clone`: 0 bytes, 0 allocations (counter bump). `Box::new(())`: 0 bytes, 0 allocations (zero-sized).
3. A `Box` has one owner, so moving the value out just leaves the box empty. An `Rc` may have other owners that still need the value, so the compiler rejects `*r` with `E0507: cannot move out of an `Rc`` ✓. Correct ways: clone the inner value (`(*r).clone()` or `String::clone(&r)`), or take it only if you're the last owner with `Rc::try_unwrap(r)` (✓ `Ok("shared")` when unique, `Err(rc)` otherwise) or `Rc::into_inner(r)` (`Option<String>`).
4. ✓ Fixes: `Rc::get_mut(&mut r).unwrap().push(3)` (works only while unique; ✓ returned `None` once another clone existed), or `Rc::make_mut(&mut r).push(3)` (clone-on-write; needs `Vec: Clone`). Output of the second program: `[1, 2, 3, 4] [1, 2, 3] false` (the first `make_mut` cloned because `b` shared the vector, so `a` and `b` now point to different allocations) and then `[1, 2, 3, 4, 5]` (the second `make_mut` found `a` unique and mutated in place).
5. ✓ `a` holds an `Rc` to `b` and `b` holds one to `a`, so each strong count is 2; when the locals go away each count falls to 1, never 0, so neither node is dropped and the memory is leaked (the compiler allows it because leaks are not memory-unsafe). Make one direction non-owning: store `Weak<Node>` for the back-edge (Lesson 25), or restructure so ownership is a tree.
6. ✓ (run: inserting `5, 3, 8, 1, 4, 3` gave `[true, true, true, true, true, false]` and in-order `[1, 3, 4, 5, 8]`): the `insert(root: &mut Option<Box<Node>>, key: i32) -> bool` of §2.5. The loop `let mut cur = root; while let Some(node) = cur { ... cur = if key < node.key { &mut node.left } else { &mut node.right }; }` walks a `&mut Option<Box<Node>>` to the empty slot and then does `*cur = Some(Box::new(...))`.
7. ✓ Dropping a node drops its `next` field, which drops the next node, one stack frame per node; at 1,000,000 nodes the 8 MiB stack overflowed in both debug and release (exit 134). Fix: `impl Drop for Node { fn drop(&mut self) { let mut cur = self.next.take(); while let Some(mut n) = cur { cur = n.next.take(); } } }` (✓ 5,000,000 nodes dropped fine). Wrapping the list in a `Stack` type with this `Drop` (as in the challenge) keeps `Node` itself drop-free.
8. `Arc<String>` (or `Arc<str>` for an immutable string): clone it into each worker thread (`Arc::clone`, ✓ pattern in §4). The updating thread builds a new value and publishes it by replacing the shared pointer under a lock or a channel message, or uses `Arc::make_mut` on its *own* handle to get a copy-on-write snapshot (✓ `data` and `snapshot` diverged in §4) without disturbing the workers' view. `Rc` would not compile across threads (Lesson 22).

</details>

---

## 11. Debugging challenge

A teammate writes a shared node type:

```rust
use std::rc::Rc;
struct Node { value: i32, next: Option<Node> }
fn main() {
    let shared = Rc::new(Node { value: 1, next: None });
    shared.value += 1;
    let owned: Node = *shared;
    println!("{}", owned.value);
}
```

It does not compile. Fix it so the program prints a bumped value while another handle still sees the old one, and then moves the node out when it is the last owner. Explain each error.

<details markdown="1">
<summary>Solution</summary>

**Stage 1.** ✓ `error[E0072]: recursive type `Node` has infinite size`. `Node` contains an `Option<Node>` and so contains itself. The type errors are reported first, so you see only this one. Add indirection: `next: Option<Box<Node>>`.

**Stage 2.** ✓ With `Box`, the type is fine and the remaining errors appear together: `error[E0594]: cannot assign to data in an `Rc`` (for `shared.value += 1`) and `error[E0507]: cannot move out of an `Rc`` (for `let owned: Node = *shared;`). `Rc` is shared ownership: it gives out only `&Node`, so it can neither be mutated through nor moved out of while other owners may exist.

**Fix.** To mutate, use clone-on-write (`Node` must be `Clone`); to move out, require uniqueness:

```rust
use std::rc::Rc;
#[derive(Clone)]
struct Node { value: i32, next: Option<Box<Node>> }
fn main() {
    let mut shared = Rc::new(Node { value: 1, next: None });
    let other = Rc::clone(&shared);
    Rc::make_mut(&mut shared).value += 1;
    println!("{} {}", shared.value, other.value);
    drop(other);
    let owned: Node = Rc::try_unwrap(shared).ok().expect("still shared");
    println!("{} {}", owned.value, owned.next.is_none());
}
```

✓ Output `2 1` (the bumped copy and the untouched original) and then `2 true`. `make_mut` cloned the node because `other` existed. After `drop(other)` the count is 1, so `try_unwrap` succeeds. `expect` panics if you got the ownership reasoning wrong; in code where other owners may be racing, handle the `Err`/`None` instead.

Lesson: `Box` is for "one owner, give me indirection"; `Rc` is for "many owners, read-only"; the escape hatches (`get_mut`, `make_mut`, `try_unwrap`/`into_inner`) all check uniqueness at run time.

</details>

---

## 12. Practical challenge: `ptrlab`

Create a library crate `ptrlab` (edition 2024) with:

1. `Stack<T>`: a `Box`-node stack with `new`, `push`, `pop`, `peek`, `peek_mut`, `len`, `is_empty`, `iter` (top first), owning `IntoIterator`, `Debug`, `Default`, and an **iterative `Drop`**.
2. A trait `PtrFamily` with a generic associated type `Ptr<T>: Deref<Target = T> + Clone`, `new`, `into_inner` (returns the value only for the last owner) and `ptr_eq`; implementations `RcFamily` and `ArcFamily`.
3. `PList<T, P: PtrFamily>`: an immutable singly linked list with structural sharing: `new`, `prepend(&self, T) -> Self` (O(1), shares the old list), `head`, `tail`, `len` (O(1)), `is_empty`, `shares_head_with`, `iter`, `Clone` and `Debug` without bounds on `T` beyond what each needs, and an **iterative `Drop`** that frees only nodes it is the last owner of. Aliases `RcList<T>` and `ArcList<T>`.
4. Tests: LIFO behavior and iteration; each element dropped exactly once (including after sharing); a million-node drop for the stack, an `RcList` and an `ArcList`; structural sharing and the untouched original; empty-list edge cases; `ArcList` is `Send + Sync`; and a concurrent test where eight threads each hold a list sharing a 100-node base while the main thread drops its handle, repeated 200 times, asserting every element is dropped exactly once. A usage doctest and a `compile_fail` doctest showing an `RcList` is not `Send`.
5. It must pass `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, and `cargo test --locked` (run `cargo generate-lockfile` once first).

<details markdown="1">
<summary>Solution</summary>

**`src/lib.rs`**

```rust
//! `ptrlab`: owning pointers in practice.
//!
//! * [`Stack`] is a mutable stack built from `Box` nodes (one owner per node).
//! * [`PList`] is an immutable, shareable list built from `Rc` or `Arc` nodes
//!   (many owners per node). The pointer type is a parameter, using a generic
//!   associated type from Lesson 22, so the same code serves single-threaded
//!   ([`RcList`]) and multi-threaded ([`ArcList`]) programs.
//! * Both lists drop their nodes in a loop. Dropping a long chain of owning
//!   pointers recursively would overflow the stack.
//!
//! ```
//! use ptrlab::{ArcList, Stack};
//!
//! let mut s = Stack::new();
//! s.push(1);
//! s.push(2);
//! assert_eq!(s.pop(), Some(2));
//!
//! let base = ArcList::new().prepend(3).prepend(2);
//! let a = base.prepend(1);
//! let b = base.prepend(10);
//! assert_eq!(a.iter().copied().collect::<Vec<_>>(), [1, 2, 3]);
//! assert_eq!(b.iter().copied().collect::<Vec<_>>(), [10, 2, 3]);
//! assert!(a.tail().unwrap().shares_head_with(&b.tail().unwrap()));
//! ```
//!
//! An `Rc`-based list cannot cross threads:
//!
//! ```compile_fail
//! use ptrlab::RcList;
//! fn needs_send<T: Send>(_: T) {}
//! needs_send(RcList::<u8>::new());
//! ```

use std::fmt;
use std::ops::Deref;
use std::rc::Rc;
use std::sync::Arc;

// ---------------------------------------------------------------------------
// Stack: Box nodes, one owner each.
// ---------------------------------------------------------------------------

type Link<T> = Option<Box<StackNode<T>>>;

struct StackNode<T> {
    value: T,
    next: Link<T>,
}

/// A LIFO stack of boxed nodes.
pub struct Stack<T> {
    head: Link<T>,
    len: usize,
}

impl<T> Default for Stack<T> {
    fn default() -> Self {
        Stack::new()
    }
}

impl<T> Stack<T> {
    pub fn new() -> Self {
        Stack { head: None, len: 0 }
    }

    pub fn len(&self) -> usize {
        self.len
    }

    pub fn is_empty(&self) -> bool {
        self.len == 0
    }

    pub fn push(&mut self, value: T) {
        let node = Box::new(StackNode {
            value,
            next: self.head.take(),
        });
        self.head = Some(node);
        self.len += 1;
    }

    pub fn pop(&mut self) -> Option<T> {
        self.head.take().map(|node| {
            let node = *node; // moving out of a Box is allowed
            self.head = node.next;
            self.len -= 1;
            node.value
        })
    }

    pub fn peek(&self) -> Option<&T> {
        self.head.as_ref().map(|node| &node.value)
    }

    pub fn peek_mut(&mut self) -> Option<&mut T> {
        self.head.as_mut().map(|node| &mut node.value)
    }

    /// Iterates from the top of the stack down.
    pub fn iter(&self) -> Iter<'_, T> {
        Iter {
            next: self.head.as_deref(),
        }
    }
}

impl<T> Drop for Stack<T> {
    fn drop(&mut self) {
        // The compiler-generated drop of `Option<Box<Node>>` recurses once per
        // node. A loop keeps the stack depth constant.
        let mut cur = self.head.take();
        while let Some(mut node) = cur {
            cur = node.next.take();
        }
    }
}

/// Borrowing iterator over a [`Stack`].
pub struct Iter<'a, T> {
    next: Option<&'a StackNode<T>>,
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;
    fn next(&mut self) -> Option<&'a T> {
        self.next.map(|node| {
            self.next = node.next.as_deref();
            &node.value
        })
    }
}

/// Owning iterator over a [`Stack`], top first.
pub struct IntoIter<T>(Stack<T>);

impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<T> {
        self.0.pop()
    }
}

impl<T> IntoIterator for Stack<T> {
    type Item = T;
    type IntoIter = IntoIter<T>;
    fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }
}

impl<T: fmt::Debug> fmt::Debug for Stack<T> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_list().entries(self.iter()).finish()
    }
}

// ---------------------------------------------------------------------------
// PtrFamily: abstract over Rc and Arc with a generic associated type.
// ---------------------------------------------------------------------------

/// A family of shared-ownership pointers (`Rc` or `Arc`).
pub trait PtrFamily {
    type Ptr<T>: Deref<Target = T> + Clone;

    fn new<T>(value: T) -> Self::Ptr<T>;

    /// Gives back the value if this was the last pointer. Of several pointers
    /// dropped at the same time, exactly one gets the value (`Arc::into_inner`
    /// has that guarantee; `Rc::into_inner` is its single-threaded twin).
    fn into_inner<T>(ptr: Self::Ptr<T>) -> Option<T>;

    fn ptr_eq<T>(a: &Self::Ptr<T>, b: &Self::Ptr<T>) -> bool;
}

/// Single-threaded sharing (`Rc`, not `Send`).
pub struct RcFamily;
/// Thread-safe sharing (`Arc`).
pub struct ArcFamily;

impl PtrFamily for RcFamily {
    type Ptr<T> = Rc<T>;
    fn new<T>(value: T) -> Rc<T> {
        Rc::new(value)
    }
    fn into_inner<T>(ptr: Rc<T>) -> Option<T> {
        Rc::into_inner(ptr)
    }
    fn ptr_eq<T>(a: &Rc<T>, b: &Rc<T>) -> bool {
        Rc::ptr_eq(a, b)
    }
}

impl PtrFamily for ArcFamily {
    type Ptr<T> = Arc<T>;
    fn new<T>(value: T) -> Arc<T> {
        Arc::new(value)
    }
    fn into_inner<T>(ptr: Arc<T>) -> Option<T> {
        Arc::into_inner(ptr)
    }
    fn ptr_eq<T>(a: &Arc<T>, b: &Arc<T>) -> bool {
        Arc::ptr_eq(a, b)
    }
}

// ---------------------------------------------------------------------------
// PList: persistent list with structural sharing.
// ---------------------------------------------------------------------------

type PLink<T, P> = Option<<P as PtrFamily>::Ptr<ListNode<T, P>>>;

struct ListNode<T, P: PtrFamily> {
    value: T,
    next: PLink<T, P>,
    len: usize,
}

/// An immutable singly linked list whose nodes are shared between lists.
///
/// `prepend` is O(1) and leaves the original list untouched; the new list
/// shares every node of the old one.
pub struct PList<T, P: PtrFamily> {
    head: PLink<T, P>,
}

/// A [`PList`] for one thread.
pub type RcList<T> = PList<T, RcFamily>;
/// A [`PList`] that can be shared between threads when `T` allows it.
pub type ArcList<T> = PList<T, ArcFamily>;

impl<T, P: PtrFamily> PList<T, P> {
    pub fn new() -> Self {
        PList { head: None }
    }

    /// A new list with `value` in front; `self` is unchanged and shared.
    pub fn prepend(&self, value: T) -> Self {
        let len = self.len() + 1;
        PList {
            head: Some(P::new(ListNode {
                value,
                next: self.head.clone(),
                len,
            })),
        }
    }

    pub fn head(&self) -> Option<&T> {
        self.head.as_deref().map(|node| &node.value)
    }

    /// The list without its first element, sharing all remaining nodes.
    pub fn tail(&self) -> Option<Self> {
        self.head.as_deref().map(|node| PList {
            head: node.next.clone(),
        })
    }

    /// Number of elements, in O(1).
    pub fn len(&self) -> usize {
        self.head.as_deref().map_or(0, |node| node.len)
    }

    pub fn is_empty(&self) -> bool {
        self.head.is_none()
    }

    /// True if both lists start at the very same node (or are both empty).
    pub fn shares_head_with(&self, other: &Self) -> bool {
        match (&self.head, &other.head) {
            (Some(a), Some(b)) => P::ptr_eq(a, b),
            (None, None) => true,
            _ => false,
        }
    }

    pub fn iter(&self) -> PIter<'_, T, P> {
        PIter {
            next: self.head.as_deref(),
        }
    }
}

impl<T, P: PtrFamily> Default for PList<T, P> {
    fn default() -> Self {
        PList::new()
    }
}

impl<T, P: PtrFamily> Clone for PList<T, P> {
    fn clone(&self) -> Self {
        PList {
            head: self.head.clone(),
        }
    }
}

impl<T: fmt::Debug, P: PtrFamily> fmt::Debug for PList<T, P> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_list().entries(self.iter()).finish()
    }
}

impl<T, P: PtrFamily> Drop for PList<T, P> {
    fn drop(&mut self) {
        // Free nodes we are the last owner of, iteratively. Stop at the first
        // node somebody else still owns: that node and everything after it
        // stay alive for them.
        let mut cur = self.head.take();
        while let Some(ptr) = cur {
            match P::into_inner(ptr) {
                Some(mut node) => cur = node.next.take(),
                None => break,
            }
        }
    }
}

/// Borrowing iterator over a [`PList`].
pub struct PIter<'a, T, P: PtrFamily> {
    next: Option<&'a ListNode<T, P>>,
}

impl<'a, T, P: PtrFamily> Iterator for PIter<'a, T, P> {
    type Item = &'a T;
    fn next(&mut self) -> Option<&'a T> {
        self.next.map(|node| {
            self.next = node.next.as_deref();
            &node.value
        })
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::sync::atomic::{AtomicUsize, Ordering};

    fn assert_send_sync<T: Send + Sync>() {}

    /// Counts how many times it is dropped.
    struct Counted(Arc<AtomicUsize>);
    impl Drop for Counted {
        fn drop(&mut self) {
            self.0.fetch_add(1, Ordering::SeqCst);
        }
    }

    #[test]
    fn stack_is_lifo() {
        let mut s = Stack::new();
        assert!(s.is_empty());
        assert_eq!(s.pop(), None);
        for i in 1..=3 {
            s.push(i);
        }
        assert_eq!((s.len(), s.peek()), (3, Some(&3)));
        *s.peek_mut().unwrap() = 30;
        assert_eq!(s.iter().copied().collect::<Vec<_>>(), [30, 2, 1]);
        assert_eq!(format!("{s:?}"), "[30, 2, 1]");
        assert_eq!(s.pop(), Some(30));
        assert_eq!(s.into_iter().collect::<Vec<_>>(), [2, 1]);
    }

    #[test]
    fn stack_drops_each_element_exactly_once() {
        let drops = Arc::new(AtomicUsize::new(0));
        let mut s = Stack::new();
        for _ in 0..10 {
            s.push(Counted(Arc::clone(&drops)));
        }
        drop(s.pop());
        assert_eq!(drops.load(Ordering::SeqCst), 1);
        drop(s);
        assert_eq!(drops.load(Ordering::SeqCst), 10);
    }

    #[test]
    fn dropping_a_million_nodes_does_not_overflow_the_stack() {
        let mut s = Stack::new();
        for i in 0..1_000_000u32 {
            s.push(i);
        }
        assert_eq!(s.len(), 1_000_000);
        drop(s);
        let mut l = RcList::new();
        for i in 0..1_000_000u32 {
            l = l.prepend(i);
        }
        assert_eq!(l.len(), 1_000_000);
        drop(l);
        let mut a = ArcList::new();
        for i in 0..1_000_000u32 {
            a = a.prepend(i);
        }
        drop(a);
    }

    #[test]
    fn prepend_shares_structure_and_leaves_the_original_alone() {
        let base = RcList::new().prepend(3).prepend(2);
        let a = base.prepend(1);
        let b = base.prepend(10);
        assert_eq!(a.iter().copied().collect::<Vec<_>>(), [1, 2, 3]);
        assert_eq!(b.iter().copied().collect::<Vec<_>>(), [10, 2, 3]);
        assert_eq!(base.iter().copied().collect::<Vec<_>>(), [2, 3]);
        assert!(a.tail().unwrap().shares_head_with(&b.tail().unwrap()));
        assert!(a.tail().unwrap().shares_head_with(&base));
        assert!(!a.shares_head_with(&b));
        assert_eq!((a.len(), b.len(), base.len()), (3, 3, 2));
        assert_eq!(format!("{a:?}"), "[1, 2, 3]");
    }

    #[test]
    fn empty_list_edges() {
        let e = ArcList::<u8>::new();
        assert!(e.is_empty() && e.head().is_none() && e.tail().is_none());
        assert_eq!(e.len(), 0);
        assert!(e.shares_head_with(&ArcList::default()));
        let one = e.prepend(1);
        assert!(!one.shares_head_with(&e));
        assert_eq!(one.tail().unwrap().len(), 0);
        assert!(one.tail().unwrap().is_empty());
    }

    #[test]
    fn shared_nodes_are_dropped_exactly_once() {
        let drops = Arc::new(AtomicUsize::new(0));
        let mut base = RcList::new();
        for _ in 0..5 {
            base = base.prepend(Counted(Arc::clone(&drops)));
        }
        let a = base.prepend(Counted(Arc::clone(&drops)));
        let b = base.prepend(Counted(Arc::clone(&drops)));
        drop(base);
        assert_eq!(
            drops.load(Ordering::SeqCst),
            0,
            "a and b still own the shared nodes"
        );
        drop(a);
        assert_eq!(
            drops.load(Ordering::SeqCst),
            1,
            "only a's private head is freed"
        );
        drop(b);
        assert_eq!(drops.load(Ordering::SeqCst), 7);
    }

    #[test]
    fn arc_lists_are_send_and_sync() {
        assert_send_sync::<ArcList<u64>>();
        assert_send_sync::<ArcList<String>>();
    }

    #[test]
    fn concurrent_drops_free_every_node_exactly_once() {
        for _ in 0..200 {
            let drops = Arc::new(AtomicUsize::new(0));
            let mut base = ArcList::new();
            for _ in 0..100 {
                base = base.prepend(Counted(Arc::clone(&drops)));
            }
            let handles: Vec<_> = (0..8)
                .map(|_| {
                    let mine = base.prepend(Counted(Arc::clone(&drops)));
                    std::thread::spawn(move || mine.len())
                })
                .collect();
            drop(base);
            for h in handles {
                assert_eq!(h.join().unwrap(), 101);
            }
            assert_eq!(drops.load(Ordering::SeqCst), 100 + 8);
        }
    }
}
```

✓ With this exact file: `cargo fmt --check` clean, `cargo clippy --all-targets --locked -- -D warnings` clean, `cargo test --locked` → 8 unit tests, 1 passing doctest and 1 `compile_fail` doctest passed. Mutation checks: deleting the `Drop` impl of `Stack`, and separately of `PList`, made the million-node test abort with "has overflowed its stack" (SIGABRT). Replacing `Arc::into_inner(ptr)` with `Arc::try_unwrap(ptr).ok()` did **not** make the concurrent test fail in three runs, so that test would not catch this particular regression reliably (§9). In a separate crate I confirmed ✓ that the `compile_fail` program gives `error[E0277]: `Rc<ptrlab::ListNode<u8, RcFamily>>` cannot be sent between threads safely`; the doctest alone proves only that *some* error occurs.

**Design notes.**

- **Iterative drop in both lists.** `Stack::drop` repeatedly `take`s each node's `next` so no node is dropped with a live successor. `PList::drop` does the same but with shared nodes: it unwraps a node only if this handle is the last owner (`P::into_inner`), and **stops at the first node somebody else still owns**, since that node and everything after it must stay alive for them. Dropping the `None` result just decrements the count.
- **`into_inner`, not `try_unwrap`.** If two threads drop the last two handles of a node at the same time, `Arc::into_inner` guarantees that one of them receives the value and continues the loop, while `try_unwrap` could fail for both and leave the final drop to run recursively. (From the documentation, remembered.)
- **The pointer family is a GAT** (`type Ptr<T>`): `Option<P::Ptr<ListNode<T, P>>>` is the node link, so the same code gives `RcList<T>` (not `Send`) and `ArcList<T>` (`Send + Sync` when `T` is, ✓ checked by `assert_send_sync`). Auto traits flow through the projection for the concrete family.
- **`len` is stored per node**, so `len()` is O(1) and `prepend` computes `self.len() + 1`.
- **No cycles by construction.** Nodes are immutable and only point to older nodes, so reference counting cannot leak here.
- **Not a full persistent-collections library:** no `append`, no random access, `len` could overflow `usize` only after exhausting memory, and `Drop` of a shared list stops early by design.

</details>

---

## 13. Interview questions

<details markdown="1">
<summary>Answers</summary>

1. **What is `Box<T>` and when do you use it?** An owning pointer to a heap value: for recursive types, trait objects and unsized slices/strs, large values you want to move cheaply, and `Box::leak`ed statics.
2. **Why do recursive types need `Box`?** Otherwise their size is infinite (E0072); a `Box` is a fixed-size pointer.
3. **What does `Option<Box<T>>` cost in size?** The same as `Box<T>` (the null niche represents `None`).
4. **Difference between `Rc` and `Arc`?** Both refcount shared ownership; `Arc` uses atomics and is `Send`/`Sync` when its content is, `Rc` is single-threaded and cheaper.
5. **Does `Rc::clone` copy the data?** No: it increments the count (✓ zero allocations).
6. **Why can't you mutate through an `Rc`?** Shared ownership gives only `&T`; use `get_mut` (unique), `make_mut` (clone-on-write) or interior mutability.
7. **What does `Rc::make_mut` do?** Returns `&mut T`, cloning the value first if the `Rc` is shared, so other holders don't see the change.
8. **Why does `*rc` fail for non-Copy `T` while `*box` works?** A `Box` has a single owner so moving out is fine; an `Rc` may have other owners.
9. **How do reference cycles happen and how do you avoid them?** Two `Rc`s owning each other never reach zero and leak. Use `Weak` for back-references or avoid cycles.
10. **What is `Rc::try_unwrap`/`into_inner`?** Take the value out if this is the last strong owner. `Arc::into_inner` is preferred when several threads may drop concurrently because it guarantees one winner.
11. **Why can dropping a long `Box` list crash?** Drop recursion uses one stack frame per node (✓ overflow at 10^6 nodes). Write an iterative `Drop`.
12. **What is the size of an `Rc<u64>` allocation?** 24 bytes on this toolchain: two counters plus the value (measured, not guaranteed).
13. **How does `Box::leak` work and when is it fine?** It returns `&'static mut T` without freeing; fine for process-lifetime data, a leak otherwise.
14. **What is deref coercion with smart pointers?** `&Box<String>`/`&Rc<String>` automatically becomes `&str` via repeated `Deref`.
15. **What is a persistent data structure and how do `Rc`/`Arc` help?** Updates return new versions that share unchanged parts; immutable nodes make sharing safe and cheap.
16. **Why prefer `Rc::clone(&a)` to `a.clone()`?** Readability: it signals a cheap pointer copy, not a deep clone.
17. **When would you not use `Arc`?** Single-threaded code (use `Rc`), or when a borrow or `thread::scope` suffices.
18. **Is it safe to branch on `Arc::strong_count`?** No: with other threads running it can change right after you read it; use it for diagnostics.
19. **What can an attacker do with unbounded nesting?** Cause stack overflow (abort) in recursive drop/clone/traverse; limit depth and use iterative algorithms.

</details>

---

## 14. Key takeaways

- `Box<T>` = one owner on the heap (8 bytes, 16 for unsized pointees; zero-sized pointees allocate nothing). It fixes recursive types and holds trait objects and slices; you can move out of it with `*b`.
- `Rc<T>` = shared read-only ownership on one thread; `Arc<T>` = the same across threads, with atomic counts. Both allocate counters plus value in one block (24 bytes for small values ✓), and `clone` is an allocation-free count bump.
- Shared means read-only: use `get_mut` (unique) or `make_mut` (clone-on-write), or interior mutability (Lesson 24). You cannot move out of an `Rc` (E0507); `try_unwrap`/`into_inner` succeed only for the last owner.
- Ownership cycles leak (✓ no `drop` printed); use `Weak` for back-edges (Lesson 25).
- Deep owning chains overflow the stack when dropped recursively (✓ at 10^6 nodes); write an iterative `Drop`. The same applies to traversal and to untrusted nesting depth.
- Choose `&T` first, `Box` second, `Rc`/`Arc` when many owners really exist; persistent structures built from immutable `Rc`/`Arc` nodes share structure safely.

**Connections.** Lesson 08's RAII and drop order, Lesson 09's niches, Lesson 11's deref coercion, Lesson 16's cache-locality discussion, Lesson 21's `Box<dyn Trait>`, Lesson 22's `Send`/`Sync` rules and GATs (the `PtrFamily` trait here) all meet in this lesson. Next: Lesson 24 (interior mutability: `Cell`, `RefCell`, `Mutex`, `RwLock`) which gives shared pointers a way to mutate, and Lesson 25 (`Weak`, `Pin`), which breaks cycles and covers self-referential data. Lesson 27 (threads), Lesson 29 (locks) and Lesson 36 (unsafe, `Box::into_raw`) build directly on this one. See the [roadmap](00_roadmap.md) and the [course index](index.md).

---

## 15. References

- *The Rust Programming Language*, ch. 15 "Smart Pointers" (15.1 `Box<T>`, 15.2 `Deref`, 15.3 `Drop`, 15.4 `Rc<T>`, 15.6 reference cycles) and ch. 16.3 "Shared-State Concurrency" (`Arc<T>`): <https://doc.rust-lang.org/book/ch15-00-smart-pointers.html>
- `std::boxed::Box`, `std::rc::Rc`, `std::sync::Arc` (including `make_mut`, `get_mut`, `try_unwrap`, `into_inner`, `ptr_eq`): <https://doc.rust-lang.org/std/boxed/struct.Box.html>, <https://doc.rust-lang.org/std/rc/struct.Rc.html>, <https://doc.rust-lang.org/std/sync/struct.Arc.html>
- `std::alloc::GlobalAlloc` (the counting allocator): <https://doc.rust-lang.org/std/alloc/trait.GlobalAlloc.html>
- *The Rustonomicon*: "Ownership Based Resource Management" and "Drop Check": <https://doc.rust-lang.org/nomicon/obrm.html>
- *Learn Rust With Entirely Too Many Linked Lists* (stacks, persistent lists, `Rc`/`Arc` sharing): <https://rust-unofficial.github.io/too-many-lists/>
- Rust error code index: E0072, E0507, E0594, E0596, E0382: <https://doc.rust-lang.org/error_codes/E0072.html>, <https://doc.rust-lang.org/error_codes/E0507.html>

The URLs above are the standard documentation locations; I did not open them during this session (the sandbox cannot reach them), so if a link has moved, search the docs site for the page title. Unchecked from memory: the Book chapter numbers and anchors, that `Rc::into_inner`/`Arc::into_inner` exist as documented with the "exactly one winner" guarantee (the `Arc` guarantee is from the documentation; my test did not exercise the failure), the `Rc` allocation layout (inferred from byte counts), that the standard `LinkedList` drops iteratively, the cost of atomic operations (not measured), the "Learn Rust With Entirely Too Many Linked Lists" title and URL, and the Lesson 07 to 09 and Lesson 11 pointers (ownership and deref coercion were taught there, but I did not reopen those files).
