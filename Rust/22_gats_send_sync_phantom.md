---
title: "Lesson 22 — GATs, Send and Sync, PhantomData, variance, and unsized types"
---

# Lesson 22 — GATs, `Send` and `Sync`, `PhantomData`, variance, and unsized types

**Module 7 · Generics and traits** · Status: generated · Edition used in examples: **2024** · External dependencies: none

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux. As before, ✓ marks outputs and diagnostics I actually ran; unmarked statements are documented behavior I did not run. Everything is **stable Rust**.

The lifetime vocabulary used here (`'a: 'b`, `'static`, `where T: 'a`) comes from Lessons 10 to 12. Direct prerequisites: [Lesson 15](15_newtype_typestate.md) (`PhantomData`, typestate), [Lesson 17](17_iterators.md) (the lending-iterator limit), [Lesson 19](19_generics_and_bounds.md) (`?Sized`, E0392), [Lesson 20](20_associated_types_coherence.md) (associated types, projections) and [Lesson 21](21_trait_objects_dispatch.md) (fat pointers, `Send`/`Sync` on trait objects).

---

## 1. Four loosely related topics, one theme

Each topic answers "what does the type system know about this type that its fields do not say directly?":

1. **Generic associated types (GATs)**: an associated type that itself takes a lifetime or type parameter, so a trait can say "the item borrows from this particular call".
2. **`Send` and `Sync`**: which values may cross or be shared between threads.
3. **`PhantomData` and variance**: how a type parameter that has no field behaves for ownership, auto traits and lifetime subtyping.
4. **Dynamically sized types (DSTs)**: `str`, `[T]`, `dyn Trait` and your own.

---

## 2. Generic associated types

### 2.1 The problem: iterators that lend

Lesson 17 showed that `Iterator` cannot hand out items borrowing from the iterator itself. Here is the failing attempt for overlapping mutable windows ✓:

```rust
struct WindowsMut<'s, T> { slice: &'s mut [T], size: usize, pos: usize }
impl<'s, T> Iterator for WindowsMut<'s, T> {
    type Item = &'s mut [T];
    fn next(&mut self) -> Option<&'s mut [T]> {
        let w = self.slice.get_mut(self.pos..self.pos + self.size)?;
        self.pos += 1;
        Some(w)
    }
}
fn main() {}
```

✓ `error: lifetime may not live long enough`. `Item` is a single type fixed by the `impl`, with no way to say "the lifetime of *this* `next` call". If it compiled, a caller could collect two overlapping `&mut` windows at once, which is exactly the aliasing Rust forbids. The signature we want is "each item borrows from `&mut self` for only as long as the caller holds it".

### 2.2 The solution: an associated type with a lifetime parameter

```rust
trait LendingIterator {
    type Item<'a> where Self: 'a;
    fn next(&mut self) -> Option<Self::Item<'_>>;
}
struct WindowsMut<'s, T> { slice: &'s mut [T], size: usize, pos: usize }
impl<'s, T> LendingIterator for WindowsMut<'s, T> {
    type Item<'a> = &'a mut [T] where Self: 'a;
    fn next(&mut self) -> Option<Self::Item<'_>> {
        let end = self.pos.checked_add(self.size)?;
        let w = self.slice.get_mut(self.pos..end)?;
        self.pos += 1;
        Some(w)
    }
}
fn main() {
    let mut v = [1, 2, 3, 4];
    let mut w = WindowsMut { slice: &mut v, size: 2, pos: 0 };
    while let Some(win) = w.next() { win[1] += win[0]; }
    println!("{v:?}");
}
```

✓ Output `[1, 3, 6, 10]` (a running sum: each window is `[previous total, next]`, and the next window overlaps the one just modified). `type Item<'a>` is a **generic associated type**. `Self::Item<'_>` in `next` ties the item's lifetime to the `&mut self` borrow, so the borrow checker forces each window to be dropped before `next` is called again; you cannot hold two at once. The `while let` loop is the idiom, because `for` desugars to `Iterator` (Lesson 17). The practical challenge uses this trait.

Two syntax rules:

- **`where Self: 'a` is required on the trait declaration** in this shape. Leave it out and the compiler stops you ✓: `error: missing required bound on `Item``, with a note that "this bound is currently required to ensure that impls have maximum flexibility".
- Write the same `where Self: 'a` in the impl; it simply restates that the borrowed data outlives the lent item.

### 2.3 GATs with type parameters, and iterators over containers

GATs can take **type** parameters as well. A classic use is abstracting over pointer types ✓:

```rust
use std::{ops::Deref, rc::Rc, sync::Arc};
trait PointerFamily {
    type Ptr<T>: Deref<Target = T>;
    fn new<T>(t: T) -> Self::Ptr<T>;
}
struct RcFam;
struct ArcFam;
impl PointerFamily for RcFam { type Ptr<T> = Rc<T>; fn new<T>(t: T) -> Rc<T> { Rc::new(t) } }
impl PointerFamily for ArcFam { type Ptr<T> = Arc<T>; fn new<T>(t: T) -> Arc<T> { Arc::new(t) } }
fn make_pair<P: PointerFamily>(a: i32, s: &str) -> (P::Ptr<i32>, P::Ptr<String>) {
    (P::new(a), P::new(s.to_string()))
}
fn main() {
    let (a, s) = make_pair::<RcFam>(1, "x");
    let (b, t) = make_pair::<ArcFam>(2, "y");
    println!("{} {} {} {}", *a, *s, *b, *t);
}
```

✓ Output `1 x 2 y`. `P::Ptr<i32>` and `P::Ptr<String>` are two instances of one family. This "type constructor" use is common in library internals that want to be generic over `Rc` versus `Arc`.

And a container trait whose iterator borrows from the container ✓:

```rust
trait Container {
    type Item;
    type Iter<'a>: Iterator<Item = &'a Self::Item> where Self: 'a;
    fn iter<'a>(&'a self) -> Self::Iter<'a>;
}
struct Bag(Vec<i32>);
impl Container for Bag {
    type Item = i32;
    type Iter<'a> = std::slice::Iter<'a, i32>;
    fn iter<'a>(&'a self) -> Self::Iter<'a> { self.0.iter() }
}
fn total<C: Container<Item = i32>>(c: &C) -> i32 { c.iter().sum() }
fn main() { println!("{}", total(&Bag(vec![1, 2, 3]))); }
```

✓ Output `6`. Before GATs this required workarounds such as implementing the trait for `&'a Bag` instead. Note that this iterator is an ordinary `Iterator` (its items borrow from the *container*, which outlives the iterator); the lending case is the one where items borrow from the *iterator*.

### 2.4 Limits you will run into

- **Not dyn compatible** (Lesson 21) ✓: `fn takes(_: &mut dyn LendingIterator) {}` gives `error[E0038]: the trait `LendingIterator` is not dyn compatible`, "because it contains generic associated type `Item`". A trait object would have to name the item type for every lifetime.
- **A closure bound over a lent item can force `'static`.** I wrote the obvious convenience method on the trait: `fn for_each_item<F>(mut self, mut f: F) where Self: Sized, F: for<'a> FnMut(Self::Item<'a>)`. It compiles, and even worked on a type that happened to be `'static`, but calling it on a `WindowsMut` borrowing a local array ✓ failed with `error[E0597]: `v` does not live long enough`, "argument requires that `v` is borrowed for `'static`", and the note **"due to a current limitation of the type system, this implies a `'static` lifetime"**. This is a known limitation of `for<'a>` bounds that mention a GAT with `where Self: 'a` (the discussion is in the tracking issue the compiler prints, #87479). The workaround I used in the practical challenge: give the concrete type an inherent `for_each_window(self, f: impl FnMut(&mut [T]))`, which has no GAT in its bound. If you design a lending trait, expect to write `next` and implement helpers per type.
- **No `for` loops.** `for` takes an `IntoIterator`; use `while let Some(x) = it.next()`.
- **`std` has no lending iterator trait yet.** At the time of writing the standard library's iterators remain non-lending (from memory; I did not search for proposals). Crates provide their own, like the one here.
- GATs were stabilized in Rust 1.65 (from memory); everything above ran on 1.97.

---

## 3. `Send` and `Sync`

Two **marker traits** describe thread-safety, and the compiler implements them automatically ("auto traits") from a type's fields:

- `T: Send` means **ownership of a `T` can be moved to another thread**.
- `T: Sync` means **`&T` can be shared between threads**. Equivalently, `T: Sync` exactly when `&T: Send`.

A struct is `Send`/`Sync` if all its fields are, so one `Rc` poisons everything that contains it. They are `unsafe` traits to implement by hand because the compiler cannot check the promise. Most types need nothing written.

### 3.1 What is and is not thread-safe

I generated this table by compiling a tiny `fn need<T: Send>() {}` / `need::<Sync>()` assertion for each type ✓ (a "NO" is a compile error):

| Type | `Send` | `Sync` | Why |
|---|---|---|---|
| `i32`, `String`, `AtomicUsize`, `OnceLock<i32>` | ✓ | ✓ | plain data / atomics |
| `Rc<i32>`, `Vec<Rc<i32>>` | NO | NO | non-atomic reference count |
| `Arc<i32>`, `Arc<Mutex<i32>>` | ✓ | ✓ | atomic count; `Mutex<T>` synchronizes |
| `Arc<Cell<i32>>` | NO | NO | `Arc<T>` needs `T: Send + Sync` and `Cell` is not `Sync` |
| `Cell<i32>`, `RefCell<i32>` | ✓ | NO | movable, but `&Cell` allows unsynchronized mutation |
| `Mutex<i32>`, `RwLock<i32>` | ✓ | ✓ | |
| `Mutex<Rc<i32>>` | NO | NO | `Mutex<T>` is `Send`/`Sync` only if `T: Send` |
| `MutexGuard<'_, i32>` | NO | ✓ | must be unlocked on the thread that locked it |
| `*const u8` | NO | NO | raw pointers opt out; the author must decide |
| `&i32` | ✓ | ✓ | `&T` is `Send` iff `T: Sync` |
| `&Cell<i32>` | NO | NO | `Cell` is not `Sync` |
| `&mut Cell<i32>` | ✓ | NO | `&mut T` is `Send` iff `T: Send`; `Sync` iff `T: Sync` |
| `&mut Rc<i32>` | NO | NO | |
| `mpsc::Sender<i32>` | ✓ | ✓ | |
| `mpsc::Receiver<i32>` | ✓ | NO | single consumer |
| `Box<dyn Fn()>` | NO | NO | a `dyn` promises nothing (Lesson 21) |
| `Box<dyn Fn() + Send>` | ✓ | NO | |
| `Box<dyn Fn() + Send + Sync>` | ✓ | ✓ | |
| `JoinHandle<()>`, `Pin<Box<i32>>` | ✓ | ✓ | |
| `PhantomData<*const ()>` | NO | NO | the standard opt-out marker |
| `PhantomData<fn() -> Rc<i32>>` | ✓ | ✓ | a function pointer type is always thread-safe |

(`Sender<i32>` being `Sync` is a recent change; I'm reporting what 1.97 does.) Rules of thumb you can read off the table: `Rc` and `Cell`/`RefCell` are for single threads; `Arc` plus `Mutex`/`RwLock`/atomics are for sharing; `Mutex<T>` makes a `Sync` wrapper out of any `T: Send`.

### 3.2 The errors, and the fixes

`thread::spawn` requires `F: Send + 'static`. Moving an `Rc` into the closure fails ✓:

```rust
use std::rc::Rc;
fn main() {
    let r = Rc::new(5);
    let h = std::thread::spawn(move || println!("{r}"));
    h.join().unwrap();
}
```

✓ `error[E0277]: `Rc<i32>` cannot be sent between threads safely`, with "within this `{closure...}`, the trait `Send` is not implemented for `Rc<i32>`". Sharing a `RefCell` through an `Arc` fails with the **other** wording ✓:

```rust
use std::{cell::RefCell, sync::Arc};
fn main() {
    let c = Arc::new(RefCell::new(0));
    let c2 = Arc::clone(&c);
    let h = std::thread::spawn(move || { *c2.borrow_mut() += 1; });
    h.join().unwrap();
}
```

✓ `error[E0277]: `RefCell<i32>` cannot be shared between threads safely`. Read the verbs: "**sent**" is `Send`; "**shared**" is `Sync`. The fix swaps in the synchronized counterparts ✓:

```rust
use std::sync::{Arc, Mutex};
fn main() {
    let c = Arc::new(Mutex::new(0));
    let hs: Vec<_> = (0..4).map(|_| {
        let c = Arc::clone(&c);
        std::thread::spawn(move || { *c.lock().unwrap() += 1; })
    }).collect();
    for h in hs { h.join().unwrap(); }
    println!("{}", *c.lock().unwrap());
}
```

✓ Output `4`. (Lesson 27 covers threads and channels properly, Lesson 29 covers locks and deadlocks.)

### 3.3 Raw pointers and `unsafe impl Send`

A raw pointer field makes the whole struct neither `Send` nor `Sync` ✓:

```rust
struct Raw { p: *mut u8 }
fn main() {
    let mut x = 5u8;
    let r = Raw { p: &mut x };
    let h = std::thread::spawn(move || { let p = r.p; p as usize });
    h.join().unwrap();
}
```

✓ `error[E0277]: `*mut u8` cannot be sent between threads safely`. (A trap I hit: my first version wrote `let _ = r.p;` inside the closure and it compiled without any error. A `_` pattern does not read the place, so the closure captured nothing and there was nothing non-`Send` to complain about. The error appears once the closure really uses the field.)

If **you** know the pointer is safe to move, say so with an `unsafe impl` ✓:

```rust
// A raw pointer to heap data that this struct uniquely owns: moving it between threads is fine,
// but only the author can know that. `unsafe impl` is a promise to the compiler.
struct Owned { p: *mut u8 }
unsafe impl Send for Owned {}
impl Owned {
    fn new(v: u8) -> Owned { Owned { p: Box::into_raw(Box::new(v)) } }
    fn get(&self) -> u8 { unsafe { *self.p } }
}
impl Drop for Owned { fn drop(&mut self) { unsafe { drop(Box::from_raw(self.p)); } } }
fn main() {
    let o = Owned::new(9);
    let h = std::thread::spawn(move || o.get());
    println!("{}", h.join().unwrap());
}
```

✓ Output `9`. That `unsafe impl Send` is correct *only because* `Owned` uniquely owns the allocation, never hands out the pointer, and frees it exactly once. Had it kept a second pointer to the same data, the impl would be a data race waiting to happen, and nothing would flag it. **Treat every `unsafe impl Send/Sync` as a security-sensitive line in code review** (Lesson 36).

### 3.4 Opting out, and scoped threads

You can make a type `!Send`/`!Sync` on purpose by holding a marker field, for example to forbid moving a thread-bound handle ✓:

```rust
use std::{marker::PhantomData, rc::Rc};
fn is_send<T: Send>() {}
struct Plain { n: u32 }
struct HasRc { r: Rc<u32> }
struct NotSend { n: u32, _m: PhantomData<*const ()> }
fn main() { is_send::<Plain>(); is_send::<HasRc>(); is_send::<NotSend>(); }
```

✓ Two errors: `Rc<u32>` cannot be sent (through `HasRc`), and `*const ()` cannot be sent (through `NotSend`), while `Plain` passes. Negative impls (`impl !Send for T`) are not stable; the `PhantomData<*const ()>` trick is the stable way.

`thread::spawn` also demands `'static`, so a closure borrowing a local fails with the lifetime error ✓: `std::thread::spawn(|| println!("{v:?}"))` with `v` a local `Vec` gives `error[E0373]: closure may outlive the current function, but it borrows `v``. **Scoped threads** lift that restriction because the scope joins before it returns ✓:

```rust
use std::sync::Mutex;
fn main() {
    let mut data = vec![1, 2, 3, 4];
    let total = Mutex::new(0);
    std::thread::scope(|s| {
        let (a, b) = data.split_at_mut(2);
        s.spawn(|| { for x in a.iter_mut() { *x *= 10; } *total.lock().unwrap() += a.iter().sum::<i32>(); });
        s.spawn(|| { for x in b.iter_mut() { *x *= 10; } *total.lock().unwrap() += b.iter().sum::<i32>(); });
    });
    println!("{data:?} {}", total.into_inner().unwrap());
}
```

✓ Output `[10, 20, 30, 40] 100`. Each thread gets a disjoint `&mut` half (`split_at_mut`) and a shared `&Mutex`; no `Arc` is needed because everything outlives the scope.

---

## 4. `PhantomData`

A struct may have a type or lifetime parameter that no field mentions. Rust rejects it (E0392, Lesson 19) because it must know how the parameter behaves for **ownership/drop checking**, **auto traits** and **variance**. `PhantomData<X>` is a zero-sized field that tells the compiler "act as if this struct contains an `X`".

The choice of `X` matters; the same marker with a different `X` gives a different type:

| Marker field | Acts like it holds | Covariance in `T` | `Send`/`Sync` | Use when |
|---|---|---|---|---|
| `PhantomData<T>` | an owned `T` | covariant | as `T` | you own `T` values behind a raw pointer (your own `Vec`) |
| `PhantomData<&'a T>` | a borrow of `T` | covariant | as `&'a T` | you hold a raw pointer into borrowed data |
| `PhantomData<fn() -> T>` | nothing; only produces `T` | covariant | always | a tag/handle: ids, typestate (Lesson 15) |
| `PhantomData<fn(T)>` | nothing; only consumes `T` | **contravariant** | always | rare |
| `PhantomData<Cell<T>>` or `*mut T` | mutable access to `T` | **invariant** | `Send` if `T: Send`; never `Sync` for `Cell` | the parameter must not be substituted |
| `PhantomData<*const ()>` | a raw pointer | n/a | **neither** | forbid `Send`/`Sync` |

The `Send`/`Sync` column is checked by the matrix above ✓ (`PhantomData<fn() -> Rc<i32>>` is `Send + Sync`; `PhantomData<*const ()>` is neither), and the variance column by the tests in §5 ✓.

This is why Lesson 15's `Id<T>` used `PhantomData<fn() -> T>`: a typed id must be `Copy`, `Send` and `Sync` even if `T` is an `Rc`-containing type. The practical challenge's `Handle<T>` does the same, and a mutation test showed what `PhantomData<T>` would have cost ✓: changing the field made `Handle<Rc<u8>>` fail the `Send + Sync` assertions (three E0277 errors).

Two further roles, stated without examples I did not run: `PhantomData<T>` tells the **drop checker** that a type may drop a `T` (needed for correct ownership in collections written with raw pointers), and `PhantomData<&'a T>` ties the struct to a lifetime so raw-pointer-based borrowing types stay sound. Writing such types is unsafe territory (Lessons 36 and 37).

---

## 5. Variance

Lifetimes have a subtyping relation: if `'long` outlives `'short`, a `&'long T` can be used where a `&'short T` is expected, because longer-lived references are strictly more capable. **Variance** says how that relation lifts through a type constructor.

### 5.1 Covariant: a longer lifetime can stand in for a shorter one

Most types are covariant in their parameters ✓:

```rust
fn shrink<'a>(x: &'static str) -> &'a str { x }
fn shrink_vec<'a>(v: Vec<&'static str>) -> Vec<&'a str> { v }
fn shrink_box<'a>(b: Box<&'static str>) -> Box<&'a str> { b }
fn main() {
    let owned = String::from("short");
    let v = shrink_vec(vec!["a", "b"]);
    println!("{} {:?} {}", shrink("lit"), v, shrink_box(Box::new("bx")));
    let _ = owned;
}
```

✓ Output `lit ["a", "b"] bx`. `&'a T` (in `'a` and in `T`), `Box<T>`, `Vec<T>`, `Option<T>`, `[T]`, `*const T` and `fn() -> T` are covariant (the last two and the general list are from the Reference, from memory; I ran `&`, `Vec`, `Box` and `fn() -> T` through `PhantomData`).

### 5.2 Invariant: no substitution at all

Behind `&mut T` the parameter is **invariant**. The reason is soundness: if `&mut &'static str` could be treated as `&mut &'a str`, safe code could store a short-lived reference into a slot that promises `'static`. The compiler stops that ✓:

```rust
fn assign<'a>(slot: &mut &'a str, v: &'a str) { *slot = v; }
fn main() {
    let mut s: &'static str = "static";
    let owned = String::from("short-lived");
    assign(&mut s, &owned);
    println!("{s}");
}
```

✓ `error[E0597]: `owned` does not live long enough`: since `s` is declared `&'static str` and `&mut` is invariant, `assign` would need `'a = 'static`, so `&owned` would have to be borrowed for `'static`. Drop the annotation (`let mut s: &str = "static";`) and it compiles and prints `short-lived` ✓, because `s` can then simply have the shorter type. `Cell<T>` (interior mutability) is invariant for the same reason ✓: returning `Cell<&'static str>` as `Cell<&'a str>` gives `error: lifetime may not live long enough`.

### 5.3 Contravariant: the direction flips

A function that accepts a shorter-lived argument can be used where one accepting a longer-lived argument is expected ✓:

```rust
fn use_it(f: fn(&'static str)) { f("lit") }
fn takes_any(s: &str) { println!("any {s}"); }
fn main() {
    // fn(&'a str) can stand in for fn(&'static str): contravariant in the argument
    let f: fn(&str) = takes_any;
    use_it(f);
}
```

✓ Output `any lit`. `fn(T)` is contravariant in `T`. It is rare in practice, but it explains why callbacks accepting broader inputs are acceptable where narrower ones are required.

### 5.4 Choosing variance with `PhantomData`

All three, side by side ✓:

```rust
use std::marker::PhantomData;
use std::cell::Cell;
struct Cov<T>(PhantomData<T>);
struct Inv<T>(PhantomData<Cell<T>>);
struct Contra<T>(PhantomData<fn(T)>);
struct FnRet<T>(PhantomData<fn() -> T>);
fn cov<'a>(x: Cov<&'static str>) -> Cov<&'a str> { x }
fn fnret<'a>(x: FnRet<&'static str>) -> FnRet<&'a str> { x }
fn contra<'a>(x: Contra<&'a str>) -> Contra<&'static str> { x }
fn main() { let _ = cov(Cov(PhantomData)); let _ = fnret(FnRet(PhantomData)); let _ = contra(Contra(PhantomData)); }
```

✓ Compiles. The invalid directions fail ✓: `fn inv<'a>(x: Inv<&'static str>) -> Inv<&'a str> { x }` and `fn contra<'a>(x: Contra<&'static str>) -> Contra<&'a str> { x }` each give `error: lifetime may not live long enough`. For library types, variance is a semver-relevant property: making a covariant type invariant can break callers.

### 5.5 A related rule: drop check

A type with a `Drop` impl may use its borrowed data inside `drop`, so the data must still be alive when it runs. The compiler enforces that with the same machinery ✓:

```rust
struct Guard<'a>(&'a String);
impl Drop for Guard<'_> { fn drop(&mut self) { println!("drop {}", self.0); } }
fn main() {
    let g;
    let s = String::from("x");
    g = Guard(&s);
    println!("made");
}
```

✓ `error[E0597]: `s` does not live long enough`, with "borrow might be used here, when `g` is dropped and runs the `Drop` code for type `Guard`". `g` is declared first, so it is dropped last, after `s`. The same code with `struct Plain<'a>(&'a String);` (no `Drop` impl) compiled and printed `made` ✓: without a destructor nothing touches the reference at drop time. Fix: declare `s` before `g`.

---

## 6. Dynamically sized types

A type whose size is not known at compile time is a **DST**: `str`, `[T]` and `dyn Trait` (Lesson 21). You can only hold them behind a pointer, and the pointer carries the missing information: a `&str` or `&[T]` holds a length, a `&dyn Trait` a vtable. Hence ✓ all of `&str`, `&[u8]`, `Box<str>` and `&Header<[u8]>` are 16 bytes below. Heap-owning DST pointers exist too: `Box<str>`, `Box<[T]>`, `Rc<str>`, `Rc<[T]>`, `Arc<[T]>`:

```rust
use std::mem::{size_of, size_of_val};
use std::rc::Rc;
struct Header<T: ?Sized> { n: u8, data: T }
fn main() {
    let a: &Header<[u8; 3]> = &Header { n: 1, data: [7, 8, 9] };
    let b: &Header<[u8]> = a;
    println!("{} {} {}", b.n, b.data.len(), size_of_val(b));
    println!("{} {} {} {}",
        size_of::<&str>(), size_of::<&[u8]>(), size_of::<Box<str>>(), size_of::<&Header<[u8]>>());
    let rs: Rc<str> = Rc::from("shared");
    let rv: Rc<[i32]> = Rc::from(vec![1, 2, 3]);
    let bs: Box<[u8]> = vec![1u8, 2].into_boxed_slice();
    println!("{} {:?} {} {}", rs, rv, bs.len(), size_of::<Rc<str>>());
    println!("{} {}", size_of_val("héllo"), size_of_val(&[1u16, 2, 3][..]));
}
```

✓ Output:

```text
1 3 4
16 16 16 16
shared [1, 2, 3] 2 16
6 6
```

`Header<T: ?Sized>` is a **custom DST**: a struct whose *last* field may be unsized (only the last, from the Reference; I did not test the error for other positions). The coercion `&Header<[u8; 3]>` to `&Header<[u8]>` is the same unsizing coercion as `Box<Circle>` to `Box<dyn Shape>`, and `size_of_val` sees through it (1 + 3 = 4 bytes). `size_of_val("héllo")` is 6 because `é` takes two bytes in UTF-8 (Lesson 05). Prefer `Box<str>`/`Box<[T]>`/`Rc<str>` over `String`/`Vec<T>` for values that never grow: they drop the capacity field (a `String` is 24 bytes versus 16 for `Box<str>`, from Lesson 08/09 measurements), and `Rc<str>` shares one allocation for count and text.

---

## 7. Common mistakes and edge cases

- **Reaching for `Rc`/`RefCell` and then spawning threads** (E0277 "cannot be sent"/"cannot be shared" ✓): use `Arc` plus `Mutex`/`RwLock`/atomics, or scoped threads.
- **`unsafe impl Send/Sync` to silence an error** instead of understanding it. The impl is a promise you must be able to defend.
- **Believing `Arc<T>` makes `T` thread-safe**: `Arc<Cell<i32>>` and `Arc<RefCell<i32>>` are not (✓ table).
- **`PhantomData<T>` when you meant a tag** (accidentally owns `T`, inherits `T`'s `Send`/`Sync`/drop behavior): use `PhantomData<fn() -> T>`.
- **Forgetting that `&mut T` is invariant** and fighting the borrow checker over `&mut &'a str` or `&mut Vec<&'a str>` (✓ E0597).
- **Variance by accident**: adding a `Cell<&'a T>` field makes your struct invariant in `'a`.
- **Assuming `Drop` is harmless to borrows** (✓ the `Guard` example).
- **Expecting a `for` loop or `dyn` over a lending iterator**; both are unavailable (§2.4).
- **A `for<'a>` closure bound over a GAT item** can demand `'static` (✓ §2.4).
- **Forgetting `where Self: 'a`** on a GAT (✓ "missing required bound").
- **`let _ = x.field` in a closure does not capture** (✓), which can hide a missing `Send` error until the code changes.
- **Using `std::thread::spawn` with borrows** (E0373 ✓): use `thread::scope`.

---

## 8. Performance implications

- GATs, `Send`/`Sync`, `PhantomData` and variance have **no run-time cost**: they are compile-time properties. `PhantomData<X>` is zero-sized ✓ (`Seq`-style handles in the challenge add no bytes of their own).
- Lending iterators avoid allocating or cloning an item per step (the point of lending a window of a buffer or a reused line buffer: ✓ the `Lines` example reused one `String`), at the cost of a less convenient API (`while let`).
- `Arc` uses atomic reference counts; `Rc` does not. Choosing `Arc` where a single thread suffices costs atomic operations, which I did not measure here (Lesson 41). `Mutex` contention is a design problem, not a type-system one (Lesson 29).
- `Box<str>`/`Box<[T]>`/`Rc<str>` save the capacity word compared with `String`/`Vec`/`Rc<String>` and one level of indirection for `Rc<str>`.
- A generational slab (the challenge) trades 8 bytes per handle (`u32` index plus `u32` generation, so `Handle<T>` is 8 bytes regardless of `T`) for safe reuse of slots without per-lookup hashing; I did not benchmark it against a `HashMap`.

---

## 9. Security and reliability

- **`unsafe impl Send/Sync` is a trust boundary.** A wrong impl lets safe code create data races, i.e. undefined behavior, without any `unsafe` at the use site. Keep them rare, documented with the invariant that justifies them, and tested with Miri and thread sanitizers where possible (Lesson 38).
- **Stale handles are a real bug class** (use-after-free by another name, "ABA" in lock-free code). The challenge's generation counters make a removed handle unusable even after its slot is reused (✓ mutation test: ignoring the generation made `stale_handles_never_see_reused_slots` and the model test fail). When a generation would overflow, the slot is **retired** instead of wrapped (✓ test), so an old handle can never match again after 2^32 reuses (retiring leaks that slot until the slab is dropped, which is bounded by the configured limit).
- **Bound the container.** `Slab::with_max_slots` returns `Err(Full)` instead of growing without limit (✓ test), so a flood of inserts cannot exhaust memory. Capacity limits belong in the type, not in the caller's good manners (Lesson 16).
- **Typed handles prevent mixing up ids** (✓ the `compile_fail` doctest: a `Handle<u32>` cannot index a `Slab<&str>`, E0308), a cheap defense against confused-deputy mistakes such as using an account id as a token id (Lesson 15).
- **Thread-safety as a property of the data model**: `Slab<T>` is `Send + Sync` exactly when `T` is, and the compiler enforces it (✓ `compile_fail` doctest: `Slab<Rc<u8>>` is not `Send`). A `Mutex<Slab<T>>` is then a sound way to share one.
- **Variance and soundness** are the reason `&mut T` is invariant; a hand-written unsafe type that declares the wrong variance (for example by using `PhantomData<T>` where `PhantomData<Cell<T>>` is needed) can allow lifetime confusion. That is another reason to avoid hand-written unsafe containers when `Vec` or a slab will do.
- **`Rc`/`RefCell` misuse across threads is caught at compile time, but `RefCell` still panics at run time** on a double mutable borrow within a thread (Lesson 24); design to avoid that.

---

## 10. Exercises

Try each before opening the answer. They are not marked complete unless you tell me you did them.

1. **(Easy)** Using the `LendingIterator` of §2.2, make `WindowsMut` turn `[1, 2, 3, 4]` into running sums. Why can't `Iterator` express it?
2. **(Easy)** Which of these are `Send`? `Rc<i32>`, `Arc<Mutex<i32>>`, `Cell<i32>`, `MutexGuard<'_, i32>`, `*const u8`. Which are `Sync`?
3. **(Medium)** Fix `thread::spawn(move || println!("{r}"))` with `r: Rc<i32>`, and explain why `Arc<RefCell<i32>>` still fails.
4. **(Medium)** Predict which compile: (a) `fn f<'a>(x: Vec<&'static str>) -> Vec<&'a str> { x }`; (b) `fn f<'a>(x: Cell<&'static str>) -> Cell<&'a str> { x }`; (c) `fn f<'a>(x: &mut &'static str, y: &'a str) { *x = y; }`; (d) passing a `fn(&str)` where a `fn(&'static str)` is expected.
5. **(Medium)** For a typed id `Id<T>` that must be `Copy`, `Send` and `Sync` for any `T`, which `PhantomData` do you use, and what breaks with the alternatives?
6. **(Medium)** Write a `Header<T: ?Sized>` with a `u8` tag and trailing data, coerce `&Header<[u8; 3]>` to `&Header<[u8]>`, and print `size_of_val`.
7. **(Hard)** Why is `unsafe impl Send for Owned {}` correct for the struct in §3.3, and how could a small change make it unsound?
8. **(Hard)** The `Guard` example fails with E0597. Give two fixes, and explain why the version without `impl Drop` compiled.

<details markdown="1">
<summary>Answers</summary>

1. ✓ (run: `[1, 3, 6, 10]`): the code is the `LendingIterator`/`WindowsMut` program of §2.2; the loop body `win[1] += win[0]`. `Iterator::Item` is one type per impl with no lifetime of the `next` call, so it cannot say "this item borrows from this particular `&mut self`" (✓ the failing version reports `lifetime may not live long enough`); overlapping `&mut` windows would alias if it were allowed.
2. ✓ From the matrix: `Rc<i32>`: neither. `Arc<Mutex<i32>>`: both. `Cell<i32>`: `Send` only. `MutexGuard<'_, i32>`: `Sync` only. `*const u8`: neither.
3. ✓ Use `Arc` (✓ `Arc<Mutex<i32>>` with four threads printed `4`). `Arc<RefCell<i32>>` fails with `RefCell<i32>` cannot be shared between threads safely because `Arc<T>` is `Send`/`Sync` only if `T: Send + Sync`, and `RefCell` is not `Sync`. Replace `RefCell` with `Mutex` (or `RwLock`, or an atomic).
4. ✓ (a) compiles (`Vec` is covariant). (b) fails: `Cell<T>` is invariant (`lifetime may not live long enough`). (c) fails: `&mut T` is invariant in `T`, so the `'static` slot cannot take a shorter borrow (the §5.2 program gives E0597 with an annotated `'static`). (d) compiles: `fn(&'a str)` is usable as `fn(&'static str)` (contravariance; ✓ prints `any lit`).
5. `PhantomData<fn() -> T>` (✓ `Send + Sync` for every `T` and covariant). `PhantomData<T>` makes `Id<Rc<u8>>` neither `Send` nor `Sync` (✓ the mutation test produced three E0277 errors) and claims ownership of a `T`. `PhantomData<*const T>`/`*mut T` makes it `!Send`/`!Sync`. `PhantomData<Cell<T>>` makes it invariant and `!Sync`. Also implement `Clone`, `Copy`, `PartialEq`, `Eq`, `Hash` and `Debug` by hand, because `#[derive]` would require `T: Clone` and so on (Lesson 15).
6. ✓ (run: output `1 3 4`):
   ```rust
   use std::mem::size_of_val;
   struct Header<T: ?Sized> { n: u8, data: T }
   fn main() {
       let a: &Header<[u8; 3]> = &Header { n: 1, data: [7, 8, 9] };
       let b: &Header<[u8]> = a;
       println!("{} {} {}", b.n, b.data.len(), size_of_val(b));
   }
   ```
   The unsized field must be last. `size_of_val` is 4: the tag plus three bytes, with no padding because both fields have alignment 1.
7. It is correct because `Owned` is the **unique owner** of the heap allocation: no other pointer or reference to it exists, nothing aliases it, and `Drop` frees it exactly once, so moving the sole owner to another thread cannot race. Unsound variants: a method returning a copy of `self.p`, a `Clone` that copies the pointer (two owners, double free), or a `Sync` impl (`&Owned` shared across threads) with a mutating method. Review `unsafe impl` against the struct's whole API, not just its fields.
8. ✓ Fix 1: declare `let s = String::from("x");` **before** `let g;` so `s` outlives `g`. Fix 2: make `Guard` own a `String` (clone it) instead of borrowing. The `Plain` version compiled (✓ `made`) because a type without a `Drop` impl does not run code that could read the reference when it goes out of scope, so the borrow only needs to last until its last use.

</details>

---

## 11. Debugging challenge

A teammate sums chunks of a vector on several threads:

```rust
use std::cell::RefCell;
use std::rc::Rc;
fn main() {
    let total = Rc::new(RefCell::new(0));
    let data = vec![1, 2, 3, 4];
    let mut handles = vec![];
    for chunk in data.chunks(2) {
        let total = Rc::clone(&total);
        handles.push(std::thread::spawn(move || {
            *total.borrow_mut() += chunk.iter().sum::<i32>();
        }));
    }
    for h in handles { h.join().unwrap(); }
    println!("{}", total.borrow());
}
```

It does not compile. The compiler reports the problems in stages as you fix them. Make it work, explaining each error. (The correct output is `10`.)

<details markdown="1">
<summary>Solution</summary>

**Stage 1.** ✓ `error[E0277]: `Rc<RefCell<i32>>` cannot be sent between threads safely`. `thread::spawn` requires the closure to be `Send`, and the closure owns an `Rc`, whose reference count is not atomic. Replace `Rc` with `Arc`.

**Stage 2.** ✓ `error[E0277]: `RefCell<i32>` cannot be shared between threads safely`. `Arc<T>` can be sent only if `T: Send + Sync`, and `RefCell` is not `Sync` (it allows unsynchronized mutation through `&`). Replace `RefCell` with `Mutex`.

**Stage 3.** ✓ With `Arc<Mutex<i32>>` the thread-safety errors are gone and a new one appears: `error[E0597]: `data` does not live long enough`. `chunk` is a `&[i32]` borrowed from the local `data`, and `thread::spawn` requires `'static` (the thread might outlive `main`). Cloning the chunk into a `Vec` per thread would work (`chunk.to_vec()`) but allocates; **scoped threads** are the right tool because the scope joins every thread before returning:

```rust
use std::sync::Mutex;
fn main() {
    let total = Mutex::new(0);
    let data = vec![1, 2, 3, 4];
    std::thread::scope(|s| {
        for chunk in data.chunks(2) {
            let total = &total;
            s.spawn(move || {
                *total.lock().unwrap() += chunk.iter().sum::<i32>();
            });
        }
    });
    println!("{}", total.lock().unwrap());
}
```

✓ Output `10`. No `Arc` is needed any more: the scope guarantees the borrows of `total` and `data` end before `main` continues. (For this particular job, summing per-thread partial results and adding them after the join avoids the lock entirely.)

Lesson: read "sent" as `Send` and "shared" as `Sync`; `'static` problems are about *borrows*, solved by ownership transfer or scopes, not by `unsafe`.

</details>

---

## 12. Practical challenge: `slotlab`

Create a library crate `slotlab` (edition 2024) with:

1. `Handle<T>`: an 8-byte key (`u32` index, `u32` generation) with `PhantomData<fn() -> T>`, **manual** `Clone`, `Copy`, `PartialEq`, `Eq`, `Hash` and `Debug` so they need no bounds on `T`.
2. `Slab<T>`: a bounded generational slab with `new`, `with_max_slots`, `insert -> Result<Handle<T>, Full>`, `get`, `get_mut`, `contains`, `remove`, `len`, `is_empty` and `iter`. Removing a value must invalidate every copy of its handle even after the slot is reused; when a slot's generation would overflow, the slot must be **retired**, not wrapped.
3. A `LendingIterator` trait using a generic associated type, and `WindowsMut<'s, T>` implementing it (rejecting a window size of 0, with no arithmetic overflow for huge sizes), plus an inherent `for_each_window`.
4. Tests: `Handle<Rc<u8>>` is `Copy + Send + Sync`; `Slab<String>` is `Send + Sync`; handles are covariant in `T`; basic operations; stale handles after reuse; capacity limit; generation exhaustion; `Hash`/`Debug` without bounds on `T`; `iter` order; a 20,000-step pseudo-random model test with stale-handle checks; windows (running sums, edge cases). A usage doctest and two `compile_fail` doctests (handle used with the wrong slab; `Slab<Rc<u8>>` is not `Send`).
5. It must pass `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, and `cargo test --locked` (run `cargo generate-lockfile` once first).

<details markdown="1">
<summary>Solution</summary>

**`src/lib.rs`**

```rust
//! `slotlab`: a generational slab with typed handles, plus a lending iterator.
//!
//! * [`Handle<T>`] is a small `Copy` key for a value in a [`Slab<T>`]. It remembers
//!   `T` only through `PhantomData<fn() -> T>`, so it is `Copy`, `Send`, `Sync`
//!   and covariant in `T` whatever `T` is.
//! * Every slot carries a generation counter, so a stale handle (one whose value
//!   was removed) can never reach a newer value that reuses the slot.
//! * [`LendingIterator`] uses a generic associated type so items can borrow from
//!   the iterator itself, which `std::iter::Iterator` cannot express.
//!
//! ```
//! use slotlab::{LendingIterator, Slab, WindowsMut};
//!
//! let mut slab = Slab::new();
//! let a = slab.insert("alice").unwrap();
//! slab.remove(a);
//! let b = slab.insert("bob").unwrap(); // reuses the slot, new generation
//! assert!(slab.get(a).is_none());
//! assert_eq!(slab.get(b), Some(&"bob"));
//!
//! let mut v = [1, 2, 3, 4];
//! let mut w = WindowsMut::new(&mut v, 2).unwrap();
//! while let Some(win) = w.next() {
//!     win[1] += win[0];
//! }
//! assert_eq!(v, [1, 3, 6, 10]);
//! ```
//!
//! A handle only works with slabs of the same element type:
//!
//! ```compile_fail
//! use slotlab::Slab;
//! let mut numbers: Slab<u32> = Slab::new();
//! let words: Slab<&str> = Slab::new();
//! let h = numbers.insert(1).unwrap();
//! let _ = words.get(h);
//! ```
//!
//! A slab of non-thread-safe values is not `Send`:
//!
//! ```compile_fail
//! use slotlab::Slab;
//! fn needs_send<T: Send>(_: T) {}
//! needs_send(Slab::<std::rc::Rc<u8>>::new());
//! ```

use std::fmt;
use std::hash::{Hash, Hasher};
use std::marker::PhantomData;

/// A key for a value of type `T` in a [`Slab<T>`].
///
/// `PhantomData<fn() -> T>` ties the handle to `T` without owning one, so the
/// handle is `Send + Sync + Copy` for every `T` and covariant in `T`.
pub struct Handle<T> {
    index: u32,
    generation: u32,
    _marker: PhantomData<fn() -> T>,
}

impl<T> Handle<T> {
    fn new(index: u32, generation: u32) -> Self {
        Handle {
            index,
            generation,
            _marker: PhantomData,
        }
    }

    /// The slot number (stable while the value is alive).
    pub fn index(&self) -> u32 {
        self.index
    }

    /// The slot's generation when this handle was issued.
    pub fn generation(&self) -> u32 {
        self.generation
    }
}

// Manual impls: `#[derive]` would add a `T: Clone` (etc.) bound that a
// marker-only parameter must not require (Lesson 15).
impl<T> Clone for Handle<T> {
    fn clone(&self) -> Self {
        *self
    }
}
impl<T> Copy for Handle<T> {}
impl<T> PartialEq for Handle<T> {
    fn eq(&self, other: &Self) -> bool {
        self.index == other.index && self.generation == other.generation
    }
}
impl<T> Eq for Handle<T> {}
impl<T> Hash for Handle<T> {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.index.hash(state);
        self.generation.hash(state);
    }
}
impl<T> fmt::Debug for Handle<T> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_struct("Handle")
            .field("index", &self.index)
            .field("generation", &self.generation)
            .finish()
    }
}

/// The slab has reached its configured capacity.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Full;

impl fmt::Display for Full {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.write_str("slab is full")
    }
}

impl std::error::Error for Full {}

enum Slot<T> {
    Occupied {
        generation: u32,
        value: T,
    },
    Free {
        generation: u32,
        next_free: Option<u32>,
    },
    /// A slot whose generation counter is exhausted; it is never reused, so a
    /// wrapped counter can never revive an old handle.
    Retired,
}

/// A bounded slab with generational handles.
pub struct Slab<T> {
    slots: Vec<Slot<T>>,
    free_head: Option<u32>,
    len: usize,
    max_slots: usize,
}

impl<T> Default for Slab<T> {
    fn default() -> Self {
        Slab::new()
    }
}

impl<T> Slab<T> {
    /// A slab limited to `u32::MAX` slots.
    pub fn new() -> Self {
        Slab::with_max_slots(u32::MAX as usize)
    }

    /// A slab that never holds more than `max_slots` slots (clamped to `u32::MAX`).
    pub fn with_max_slots(max_slots: usize) -> Self {
        Slab {
            slots: Vec::new(),
            free_head: None,
            len: 0,
            max_slots: max_slots.min(u32::MAX as usize),
        }
    }

    /// Number of live values.
    pub fn len(&self) -> usize {
        self.len
    }

    pub fn is_empty(&self) -> bool {
        self.len == 0
    }

    /// Stores `value`, reusing a freed slot if there is one.
    pub fn insert(&mut self, value: T) -> Result<Handle<T>, Full> {
        if let Some(index) = self.free_head {
            let slot = &mut self.slots[index as usize];
            let Slot::Free {
                generation,
                next_free,
            } = *slot
            else {
                unreachable!("the free list points at a slot that is not free");
            };
            *slot = Slot::Occupied { generation, value };
            self.free_head = next_free;
            self.len += 1;
            return Ok(Handle::new(index, generation));
        }
        if self.slots.len() >= self.max_slots {
            return Err(Full);
        }
        let index = self.slots.len() as u32; // max_slots <= u32::MAX, so this cannot truncate
        self.slots.push(Slot::Occupied {
            generation: 0,
            value,
        });
        self.len += 1;
        Ok(Handle::new(index, 0))
    }

    /// Looks a value up; `None` for a removed value or a handle from a recycled slot.
    pub fn get(&self, handle: Handle<T>) -> Option<&T> {
        match self.slots.get(handle.index as usize)? {
            Slot::Occupied { generation, value } if *generation == handle.generation => Some(value),
            _ => None,
        }
    }

    pub fn get_mut(&mut self, handle: Handle<T>) -> Option<&mut T> {
        match self.slots.get_mut(handle.index as usize)? {
            Slot::Occupied { generation, value } if *generation == handle.generation => Some(value),
            _ => None,
        }
    }

    pub fn contains(&self, handle: Handle<T>) -> bool {
        self.get(handle).is_some()
    }

    /// Removes and returns the value; the handle (and every copy of it) becomes stale.
    pub fn remove(&mut self, handle: Handle<T>) -> Option<T> {
        let slot = self.slots.get_mut(handle.index as usize)?;
        match slot {
            Slot::Occupied { generation, .. } if *generation == handle.generation => {}
            _ => return None,
        }
        let next_generation = handle.generation.checked_add(1);
        let replacement = match next_generation {
            Some(generation) => Slot::Free {
                generation,
                next_free: self.free_head,
            },
            None => Slot::Retired,
        };
        let Slot::Occupied { value, .. } = std::mem::replace(slot, replacement) else {
            unreachable!("the slot was just checked to be occupied");
        };
        if next_generation.is_some() {
            self.free_head = Some(handle.index);
        }
        self.len -= 1;
        Some(value)
    }

    /// Iterates over live values in slot order.
    pub fn iter(&self) -> impl Iterator<Item = (Handle<T>, &T)> {
        self.slots
            .iter()
            .enumerate()
            .filter_map(|(i, slot)| match slot {
                Slot::Occupied { generation, value } => {
                    Some((Handle::new(i as u32, *generation), value))
                }
                _ => None,
            })
    }
}

/// An iterator whose items may borrow from the iterator itself.
///
/// `type Item<'a>` is a generic associated type: each call to `next` picks a
/// fresh lifetime, tied to the `&mut self` borrow. `std::iter::Iterator`
/// cannot do this because its `Item` has no lifetime parameter.
pub trait LendingIterator {
    type Item<'a>
    where
        Self: 'a;

    fn next(&mut self) -> Option<Self::Item<'_>>;
}

/// Overlapping mutable windows over a slice, lent one at a time.
pub struct WindowsMut<'s, T> {
    slice: &'s mut [T],
    size: usize,
    pos: usize,
}

impl<'s, T> WindowsMut<'s, T> {
    /// `None` if `size` is zero.
    pub fn new(slice: &'s mut [T], size: usize) -> Option<Self> {
        if size == 0 {
            None
        } else {
            Some(WindowsMut {
                slice,
                size,
                pos: 0,
            })
        }
    }

    /// Runs `f` on every window.
    ///
    /// This is an inherent method on purpose: the generic form
    /// `fn for_each_item<F: for<'a> FnMut(Self::Item<'a>)>` on the trait makes the
    /// compiler demand that `Self` outlive `'static` (a current GAT limitation).
    pub fn for_each_window<F: FnMut(&mut [T])>(mut self, mut f: F) {
        while let Some(window) = self.next() {
            f(window);
        }
    }
}

impl<'s, T> LendingIterator for WindowsMut<'s, T> {
    type Item<'a>
        = &'a mut [T]
    where
        Self: 'a;

    fn next(&mut self) -> Option<Self::Item<'_>> {
        let end = self.pos.checked_add(self.size)?;
        let window = self.slice.get_mut(self.pos..end)?;
        self.pos += 1;
        Some(window)
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::collections::HashSet;
    use std::rc::Rc;

    fn assert_send_sync<T: Send + Sync>() {}
    fn assert_copy<T: Copy>() {}

    #[test]
    fn handles_are_copy_send_sync_for_every_t() {
        assert_copy::<Handle<Rc<u8>>>();
        assert_send_sync::<Handle<Rc<u8>>>();
        assert_send_sync::<Handle<std::cell::Cell<u8>>>();
        assert_send_sync::<Slab<String>>();
        assert_send_sync::<Slab<std::sync::Mutex<Vec<u8>>>>();
    }

    #[test]
    fn handles_are_covariant_in_t() {
        fn shrink<'a>(h: Handle<&'static str>) -> Handle<&'a str> {
            h
        }
        let mut slab: Slab<&'static str> = Slab::new();
        let h = slab.insert("x").unwrap();
        let local = String::from("local");
        let h2: Handle<&str> = shrink(h);
        assert_eq!(h2.index(), 0);
        drop(local);
    }

    #[test]
    fn insert_get_remove() {
        let mut slab = Slab::new();
        let a = slab.insert(10).unwrap();
        let b = slab.insert(20).unwrap();
        assert_eq!(
            (slab.len(), slab.get(a), slab.get(b)),
            (2, Some(&10), Some(&20))
        );
        *slab.get_mut(a).unwrap() += 1;
        assert_eq!(slab.remove(a), Some(11));
        assert_eq!(slab.remove(a), None, "second remove of the same handle");
        assert!(!slab.contains(a));
        assert_eq!(slab.len(), 1);
        assert!(slab.get_mut(a).is_none());
    }

    #[test]
    fn stale_handles_never_see_reused_slots() {
        let mut slab = Slab::new();
        let old = slab.insert("first").unwrap();
        slab.remove(old);
        let new = slab.insert("second").unwrap();
        assert_eq!(new.index(), old.index(), "the slot is reused");
        assert_ne!(new.generation(), old.generation());
        assert_ne!(new, old);
        assert_eq!(slab.get(old), None);
        assert_eq!(slab.remove(old), None);
        assert_eq!(slab.get(new), Some(&"second"));
    }

    #[test]
    fn capacity_is_bounded_and_freed_slots_are_reused() {
        let mut slab = Slab::with_max_slots(2);
        let a = slab.insert('a').unwrap();
        slab.insert('b').unwrap();
        assert_eq!(slab.insert('c'), Err(Full));
        slab.remove(a);
        let c = slab.insert('c').unwrap();
        assert_eq!(c.index(), a.index());
        assert_eq!(slab.insert('d'), Err(Full));
        assert_eq!(Full.to_string(), "slab is full");
        let none: Slab<u8> = Slab::with_max_slots(0);
        assert!(none.is_empty());
    }

    #[test]
    fn exhausted_generation_retires_the_slot() {
        let mut slab = Slab::with_max_slots(4);
        slab.slots.push(Slot::Occupied {
            generation: u32::MAX,
            value: 7,
        });
        slab.len = 1;
        let last = Handle::new(0, u32::MAX);
        assert_eq!(slab.remove(last), Some(7));
        assert!(matches!(slab.slots[0], Slot::Retired));
        let next = slab.insert(8).unwrap();
        assert_eq!(next.index(), 1, "a retired slot is never reused");
        assert_eq!(slab.get(last), None);
        assert_eq!(slab.len(), 1);
    }

    #[test]
    fn handle_traits_do_not_require_t_bounds() {
        struct NoTraits;
        let mut slab: Slab<NoTraits> = Slab::new();
        let h = slab.insert(NoTraits).unwrap();
        let copy = h;
        let set: HashSet<_> = [h, copy].into_iter().collect();
        assert_eq!(set.len(), 1);
        assert_eq!(format!("{h:?}"), "Handle { index: 0, generation: 0 }");
    }

    #[test]
    fn iter_lists_live_values_in_slot_order() {
        let mut slab = Slab::new();
        let a = slab.insert("a").unwrap();
        let b = slab.insert("b").unwrap();
        let c = slab.insert("c").unwrap();
        slab.remove(b);
        let items: Vec<_> = slab.iter().map(|(h, v)| (h.index(), *v)).collect();
        assert_eq!(items, [(0, "a"), (2, "c")]);
        assert!(slab.iter().all(|(h, _)| slab.contains(h)));
        let _ = (a, c);
    }

    struct XorShift(u64);
    impl XorShift {
        fn next(&mut self) -> u64 {
            self.0 ^= self.0 << 13;
            self.0 ^= self.0 >> 7;
            self.0 ^= self.0 << 17;
            self.0
        }
    }

    #[test]
    fn random_operations_match_a_model() {
        let mut rng = XorShift(0x2545_F491_4F6C_DD1D);
        let mut slab: Slab<u64> = Slab::with_max_slots(64);
        let mut live: Vec<(Handle<u64>, u64)> = Vec::new();
        let mut stale: Vec<Handle<u64>> = Vec::new();
        for step in 0..20_000u64 {
            match rng.next() % 3 {
                0 => match slab.insert(step) {
                    Ok(h) => live.push((h, step)),
                    Err(Full) => assert_eq!(live.len(), 64),
                },
                1 if !live.is_empty() => {
                    let (h, v) = live.swap_remove(rng.next() as usize % live.len());
                    assert_eq!(slab.remove(h), Some(v));
                    stale.push(h);
                }
                _ if !live.is_empty() => {
                    let (h, v) = live[rng.next() as usize % live.len()];
                    assert_eq!(slab.get(h), Some(&v));
                }
                _ => {}
            }
            assert_eq!(slab.len(), live.len());
            if let Some(&h) = stale.get(rng.next() as usize % stale.len().max(1)) {
                assert!(slab.get(h).is_none(), "stale handle {h:?} resolved");
            }
        }
        assert_eq!(slab.iter().count(), live.len());
    }

    #[test]
    fn windows_mut_lends_overlapping_windows() {
        let mut v = [1, 2, 3, 4];
        let mut w = WindowsMut::new(&mut v, 2).unwrap();
        while let Some(win) = w.next() {
            win[1] += win[0];
        }
        assert_eq!(v, [1, 3, 6, 10]);
    }

    #[test]
    fn windows_mut_edge_cases() {
        let mut v = [1, 2, 3];
        assert!(WindowsMut::new(&mut v, 0).is_none());
        assert!(WindowsMut::new(&mut v, 4).unwrap().next().is_none());
        let mut count = 0;
        WindowsMut::new(&mut v, 3).unwrap().for_each_window(|w| {
            assert_eq!(w.len(), 3);
            count += 1;
        });
        assert_eq!(count, 1);
        let mut empty: [u8; 0] = [];
        assert!(WindowsMut::new(&mut empty, 1).unwrap().next().is_none());
        let mut huge = [1u8, 2];
        assert!(
            WindowsMut::new(&mut huge, usize::MAX)
                .unwrap()
                .next()
                .is_none(),
            "no overflow in pos + size"
        );
        let mut ones = [5, 6, 7];
        WindowsMut::new(&mut ones, 1)
            .unwrap()
            .for_each_window(|w| w[0] *= 2);
        assert_eq!(ones, [10, 12, 14]);
    }
}
```

✓ With this exact file: `cargo fmt --check` clean, `cargo clippy --all-targets --locked -- -D warnings` clean, `cargo test --locked` → 11 unit tests, 1 passing doctest and 2 `compile_fail` doctests passed. I mutated the code to check the tests have teeth: ignoring the generation in `get` failed `stale_handles_never_see_reused_slots` and the model test; replacing the `PhantomData<fn() -> T>` field with `PhantomData<T>` produced three E0277 errors in the `Send + Sync` assertions; and replacing `checked_add` with `wrapping_add` failed `exhausted_generation_retires_the_slot`. In a separate binary crate I confirmed ✓ that the two `compile_fail` programs give `error[E0308]: mismatched types` (wrong slab) and `error[E0277]: `Rc<u8>` cannot be sent between threads safely`; the doctests alone prove only that *some* error occurs.

**Design notes.**

- **Why `fn() -> T`:** the handle must be `Copy + Send + Sync` for any `T` and never claim to own one (§4). The manual trait impls avoid the `T: Clone`-style bounds that `derive` adds (Lesson 15).
- **Slot states:** `Occupied` (generation plus value), `Free` (generation to use next, plus the next free index: an intrusive free list, so reuse is O(1) with no extra allocation) and `Retired`. `remove` bumps the generation with `checked_add`; on overflow the slot becomes `Retired` and is never linked into the free list.
- **Bounded by construction:** `max_slots` is clamped to `u32::MAX`, so the `as u32` index cast cannot truncate. `Full` is a real error type with `Display` and `Error`.
- **Internal invariants:** two `unreachable!` calls assert invariants the code itself maintains (the free list only points at `Free` slots; a slot just checked as occupied is occupied). The model test exercises thousands of insert/remove/get sequences against a simple vector model to catch violations. They are panics, not undefined behavior, and the crate contains no `unsafe`.
- **Lending iterator and its limit:** `LendingIterator::next` returns `Option<Self::Item<'_>>`. My first draft put `for_each_item<F: for<'a> FnMut(Self::Item<'a>)>` on the trait and the compiler demanded `'static` (§2.4), so `for_each_window` is an inherent method taking `FnMut(&mut [T])`. `WindowsMut::new` returns `None` for size 0, and `pos + size` uses `checked_add` so a size of `usize::MAX` ends the iteration instead of overflowing (✓ test).
- **Not a general arena:** no iteration by mutable reference, no shrinking, no serialization, and a retired slot is never reclaimed.

</details>

---

## 13. Interview questions

<details markdown="1">
<summary>Answers</summary>

1. **What is a GAT and why was it added?** An associated type with its own lifetime or type parameters (`type Item<'a> where Self: 'a`). It lets a trait express items that borrow from the call (lending iterators) and type-constructor abstractions (pointer families).
2. **Why can't `Iterator` be a lending iterator?** `Item` has no lifetime parameter, so it cannot depend on the `&mut self` borrow of `next`; overlapping mutable items would alias.
3. **Why is `where Self: 'a` required on a GAT?** The compiler currently requires it so impls have maximum flexibility ("missing required bound" otherwise).
4. **What are limitations of GATs?** Traits with GATs are not dyn compatible; `for<'a>` bounds mentioning a GAT can force `'static`; no `for` loops over lending iterators.
5. **What do `Send` and `Sync` mean?** `Send`: a value can be moved to another thread. `Sync`: a shared reference can be used from several threads (`T: Sync` iff `&T: Send`).
6. **Why is `Rc` not `Send`? Why is `Cell` `Send` but not `Sync`?** `Rc`'s count is non-atomic. `Cell` may be moved safely but `&Cell` allows unsynchronized mutation.
7. **How are `Send`/`Sync` implemented?** As auto traits derived from fields; `unsafe impl` overrides; `PhantomData<*const ()>` opts out.
8. **When is `Arc<T>` `Send`?** When `T: Send + Sync`. `Mutex<T>` is `Sync` when `T: Send`.
9. **What do E0277 "cannot be sent" and "cannot be shared" mean?** The closure or type fails `Send` and `Sync` respectively.
10. **When is `unsafe impl Send` justified?** When the type's invariants guarantee no aliasing or thread-affine state (for example, unique ownership of a raw allocation). It must be argued, documented and reviewed.
11. **What is `PhantomData` for?** To tell the compiler how an unused parameter should behave for drop checking, auto traits and variance (and to avoid E0392).
12. **Why `PhantomData<fn() -> T>` for a handle?** It is `Send + Sync + covariant` for every `T` and does not claim ownership.
13. **Explain covariance, contravariance and invariance.** Covariant: `Foo<&'long T>` usable as `Foo<&'short T>`. Contravariant: the reverse (`fn(&'short T)` usable as `fn(&'long T)`). Invariant: no substitution (`&mut T`, `Cell<T>`).
14. **Why is `&mut T` invariant in `T`?** Otherwise safe code could write a shorter-lived value into a slot promised to hold a longer-lived one.
15. **What is drop check?** The rule that data borrowed by a type with a `Drop` impl must outlive the value, because `drop` may use it.
16. **What is a DST? Name three.** A type whose size is unknown at compile time: `str`, `[T]`, `dyn Trait` (and structs ending in one). Used through fat pointers.
17. **`Box<str>` versus `String`?** `Box<str>` is a fixed-length boxed string (16 bytes handle on 64-bit), `String` has capacity and can grow (24 bytes).
18. **`thread::spawn` versus `thread::scope`?** `spawn` needs `'static` closures; `scope` joins all threads before returning, so they can borrow locals.
19. **How would you prevent use-after-free of ids in an entity table?** Generational handles: bump a generation on removal, check it on lookup, retire exhausted slots, bound the table size.

</details>

---

## 14. Key takeaways

- GATs let an associated type depend on a lifetime or type parameter. They make lending iterators and pointer-family abstractions expressible; they are not dyn compatible, need `where Self: 'a`, and `for<'a>` closure bounds over them can force `'static`.
- `Send` means "may be moved to another thread", `Sync` means "`&T` may be shared". Both are auto traits; "sent" and "shared" in E0277 tell you which one is missing. `Rc`, `Cell` and `RefCell` are single-threaded; `Arc` plus `Mutex`/`RwLock`/atomics share across threads; `thread::scope` lets threads borrow.
- Every `unsafe impl Send/Sync` is a promise the compiler cannot check; keep them rare and justified.
- `PhantomData<X>` makes an unused parameter behave like `X`: pick `fn() -> T` for tags, `T` for ownership, `Cell<T>` for invariance, `*const ()` to forbid threads.
- Variance: most types are covariant, `&mut T` and `Cell<T>` are invariant for soundness, `fn(T)` is contravariant. A `Drop` impl extends borrows through drop check.
- DSTs (`str`, `[T]`, `dyn Trait`, structs with an unsized last field) live behind fat pointers; `Box<str>`, `Box<[T]>`, `Rc<str>` are compact owned forms.
- Generational handles with bounded capacity and retired slots turn a dangling-id bug class into `None`.

**Connections.** Lesson 15's `Id<T>` and typestate, Lesson 16's collections, Lesson 17's lending-iterator limit, Lesson 19's `?Sized` and E0392, Lesson 20's projections and Lesson 21's `Send`/`Sync` on trait objects all land here. Next: Lesson 23 (`Box`, `Rc`, `Arc`), which builds on the sharing rules above; Lesson 24 (interior mutability: `Cell`, `RefCell`, `Mutex`) explains why those types are not `Sync`; Lesson 27 (threads and channels), Lesson 28 (atomics and memory ordering) and Lesson 29 (locks and deadlocks) use `Send` and `Sync` constantly; Lesson 36 (unsafe and raw pointers) is where `unsafe impl Send` and hand-written variance belong; Lesson 38 covers Miri. See the [roadmap](00_roadmap.md) and the [course index](index.md).

---

## 15. References

- *The Rust Reference*, "Generic associated types" under "Associated Items", and "Type parameters": <https://doc.rust-lang.org/reference/items/associated-items.html>
- Rust blog, "Generic associated types to be stable in Rust 1.65": <https://blog.rust-lang.org/2022/10/28/gats-stabilization.html>; tracking issue #87479 (cited by the compiler): <https://github.com/rust-lang/rust/issues/87479>
- *The Rust Programming Language*, ch. 16.4 "Extensible Concurrency with the `Sync` and `Send` Traits" and ch. 16.1 (`thread::spawn`): <https://doc.rust-lang.org/book/ch16-04-extensible-concurrency-sync-and-send.html>
- `std::marker::Send`, `Sync`, `PhantomData`; `std::thread::scope`: <https://doc.rust-lang.org/std/marker/trait.Send.html>, <https://doc.rust-lang.org/std/marker/struct.PhantomData.html>, <https://doc.rust-lang.org/std/thread/fn.scope.html>
- *The Rustonomicon*: "Send and Sync", "Subtyping and Variance", "PhantomData", "Drop Check", "Exotically Sized Types": <https://doc.rust-lang.org/nomicon/send-and-sync.html>, <https://doc.rust-lang.org/nomicon/subtyping.html>, <https://doc.rust-lang.org/nomicon/phantom-data.html>, <https://doc.rust-lang.org/nomicon/dropck.html>, <https://doc.rust-lang.org/nomicon/exotic-sizes.html>
- *The Rust Reference*, "Subtyping and Variance", "Dynamically Sized Types", "Special types and traits" (auto traits): <https://doc.rust-lang.org/reference/subtyping.html>, <https://doc.rust-lang.org/reference/dynamically-sized-types.html>
- Rust error code index: E0277, E0373, E0392, E0597, E0038: <https://doc.rust-lang.org/error_codes/E0277.html>, <https://doc.rust-lang.org/error_codes/E0373.html>, <https://doc.rust-lang.org/error_codes/E0597.html>

The URLs above are the standard documentation locations; I did not open them during this session (the sandbox cannot reach them), so if a link has moved, search the docs site for the page title. Unchecked from memory: the Book chapter numbers and anchors, the blog post title and date for GAT stabilization and the 1.65 version number, the list of covariant std types beyond the ones I compiled (`Option`, `[T]`, `*const T`), the invariance of `*mut T`, that only the last struct field may be unsized, that `std` has no lending iterator trait today, the drop-check role of `PhantomData<T>` (explained, not run), the exact stabilization of `Sender: Sync` (I report only the 1.97 result), and the 24-versus-16-byte sizes of `String` and `Box<str>` (taken from earlier lessons' measurements, not rerun here).
