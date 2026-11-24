---
layout: default
title: "Lesson 12 — Variance, higher-ranked bounds, reading borrow-checker diagnostics, hard borrow problems"
---

# Lesson 12 — Variance, higher-ranked bounds, reading borrow-checker diagnostics, hard borrow problems

**Module 4 · Borrowing and lifetimes** · Status: generated · Edition used in examples: **2024** · External dependencies: none

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux. As before, ✓ marks outputs and diagnostics I actually ran; unmarked statements are documented behavior I did not run. Everything is **stable Rust**. This is the last lesson of Module 4: after it, you should be able to read almost any borrow-checker error, explain *why* it is there, and know which restructuring to reach for.

Builds on [Lesson 10](10_references_and_borrowing.md) (aliasing XOR mutability, reborrows), [Lesson 11](11_lifetimes.md) (annotations, elision, `'static`) and [Lesson 08](08_stack_heap_drop.md) (drop order, stack limits).

---

## 1. Subtyping in Rust is about lifetimes

Most languages have subtyping between *types* (a `Cat` is an `Animal`). Rust has almost none; its subtyping exists for **lifetimes**: if `'long` outlives `'short` (written `'long: 'short`), then `&'long T` can be used where `&'short T` is expected. "Longer-lived is a subtype of shorter-lived."

**Variance** answers: when `'long` is a subtype of `'short`, how are `F<'long>` and `F<'short>` related for a type constructor `F`?

| Variance | Meaning | Example |
|---|---|---|
| **Covariant** | `F<'long>` can be used as `F<'short>` | `&'a T`, `Box<T>`, `Vec<T>`, `Option<T>`, `*const T`, function **return** types |
| **Contravariant** | `F<'short>` can be used as `F<'long>` (it flips) | function **argument** types: `fn(&'a T)` is contravariant in `'a` |
| **Invariant** | no substitution either way, the lifetime must match exactly | `&mut T` (in `T`), `Cell<T>`, `*mut T`, `UnsafeCell<T>` |

I verified ✓ the entries for `&T`, `Box`, `Vec`, `&mut T`, `Cell`, `*const T`, `*mut T`, and function arguments below; the others are from the Reference and the Rustonomicon and I did not run them.

**Why you care.** Variance is why `fn show(s: &str)` accepts a `&'static str` literal *and* a borrow of a local, and why some programs are rejected with confusing "lifetime may not live long enough" errors: an `&mut` or a `Cell` is quietly forbidding the flexibility you expected. It is also the thing you must get right in `unsafe` code that defines its own pointer types (Lesson 36).

---

## 2. Covariance: the common, harmless case

```rust
fn print_all(v: Vec<&str>) { for s in v { print!("{s} "); } println!(); }
fn shorten<'long: 'short, 'short>(x: &'long str) -> &'short str { x }
fn main() {
    let v: Vec<&'static str> = vec!["a", "b"];
    let local = String::from("zz");
    let mut w: Vec<&str> = v;      // Vec<&'static str> -> Vec<&'short str>: covariance
    w.push(&local);
    print_all(w);
    println!("{}", shorten("lit"));
    let b: Box<&'static str> = Box::new("boxed");
    let b2: Box<&str> = b;
    println!("{b2}");
}
```

✓ Output `a b zz `, `lit`, `boxed`. A `Vec<&'static str>` was treated as a `Vec<&'short str>`, and then a shorter-lived borrow (`&local`) was pushed into it. This is sound **because `w` is a new owner** of the moved `v`: nobody else still sees the "all `'static`" view of that vector. Read-only (shared) access is always safe to shorten: whatever you read from a `&'long T`, a `&'short T` promise is weaker and still true.

---

## 3. Invariance: why `&mut T` cannot be covariant

Suppose `&mut &'static str` were covariant, so it could be treated as `&mut &'short str`. Then you could write a short-lived reference *through* it into a place that everyone else still believes holds a `'static` one:

```rust
fn main() {
    let mut forever: &'static str = "static";
    {
        let local = String::from("short");
        let r = &mut forever;
        *r = &local;       // store a short-lived ref in a 'static slot?
    }
    println!("{forever}"); // would read freed memory
}
```

✓ `error[E0597]: `local` does not live long enough`. The compiler refuses: the **`T` inside `&mut T` is invariant**. The same through a function boundary:

```rust
fn overwrite<'a>(slot: &mut &'a str, val: &'a str) { *slot = val; }
fn main() {
    let mut s: &'static str = "static";
    let local = String::from("short");
    overwrite(&mut s, &local);
    println!("{s}");
}
```

✓ `E0597`. `overwrite` forces the slot's `'a` and `val`'s `'a` to be *equal*, and since `&mut s` cannot be shortened, `'a` is `'static`, and `&local` isn't. (The fix in real code is usually to declare `let mut s: &str = "static";`, letting `'a` be inferred as the shorter lifetime. ✓ the practical challenge's test does exactly that.)

Contrast with a **shared** reference in the same position, which is covariant:

```rust
fn show<'a>(slot: &&'a str, _v: &'a str) { println!("{slot}"); }
fn main() {
    let s: &'static str = "static";
    let local = String::from("short");
    show(&s, &local);       // & is covariant in T: fine
}
```

✓ Output `static`. Read-only access can't smuggle anything in.

**Interior mutability is the same story**, because `Cell<T>` lets you write through `&`:

```rust
use std::cell::Cell;
fn set_it<'a>(c: &Cell<&'a str>, v: &'a str) { c.set(v); }
fn main() {
    let c: Cell<&'static str> = Cell::new("static");
    let local = String::from("short");
    set_it(&c, &local);
    println!("{}", c.get());
}
```

✓ `E0597`. `Cell<T>` (and `RefCell`, `Mutex`, `UnsafeCell`) are invariant in `T` for exactly the reason `&mut T` is: they allow writes. (Lesson 24.)

**Rule of thumb:** *if you can write a `T` through it, it's invariant in `T`; if you can only read, it's covariant; if you only consume (take as an argument), it's contravariant.*

---

## 4. Contravariance, `PhantomData`, and raw pointers

Function arguments flip the direction. A function that accepts *any* lifetime can be used where one accepting only `'static` is expected, but not the reverse:

```rust
fn takes_any(s: &str) -> usize { s.len() }
fn takes_static(s: &'static str) -> usize { s.len() }
fn main() {
    let f: fn(&'static str) -> usize = takes_any;      // OK
    println!("{}", f("abc"));
    let local = String::from("x");
    let g: for<'a> fn(&'a str) -> usize = takes_static;  // E0308
    println!("{}", g(&local));
}
```

✓ The first line compiles; the second fails with `error[E0308]: mismatched types`. A `fn` that insists on `'static` can't be called with arbitrary borrows. (The `for<'a>` here is the higher-ranked syntax of Section 5.) Contravariance is rare in everyday code; it matters most for callbacks.

### A struct's variance is inferred from its fields

You never declare variance. The compiler computes it from the field types, and where a parameter occurs in several positions the **most restrictive** result wins (co + inv → inv, as documented). `PhantomData<X>` (Lesson 22) lets you *choose* a variance for a parameter that has no real field, which is how you control what unsafe pointer types promise:

```rust
use std::marker::PhantomData;
use std::cell::Cell;
struct Co<'a>(PhantomData<&'a ()>);
struct Contra<'a>(PhantomData<fn(&'a ())>);
fn co<'long: 'short, 'short>(x: Co<'long>) -> Co<'short> { x }
fn contra<'long: 'short, 'short>(x: Contra<'short>) -> Contra<'long> { x }
fn main() { let _ = co(Co(PhantomData)); let _ = contra(Contra(PhantomData)); }
```

✓ Both compile (the compiler only warns about the unused `Cell` import). And the invariant one:

```rust
use std::marker::PhantomData;
use std::cell::Cell;
struct Inv<'a>(PhantomData<Cell<&'a ()>>);
fn inv<'long: 'short, 'short>(x: Inv<'long>) -> Inv<'short> { x }
fn main() { let _ = inv(Inv(PhantomData)); }
```

✓ `error: lifetime may not live long enough`, with these helpful notes: "requirement occurs because of the type `Inv<'_>`, which makes the generic argument `'_` invariant" and "the struct `Inv<'a>` is invariant over the parameter `'a`". When you see a "may not live long enough" error that makes no sense, look for an `&mut`, `Cell`, `*mut` or `dyn Trait` hiding an invariant position.

### Raw pointers

`*const T` is covariant and `*mut T` invariant; ✓ `struct Mine2<T> { p: *const T }` converts from `Mine2<&'a str>` to `Mine2<&'b str>`, while the same struct with `*mut T` fails with the "lifetime may not live long enough" error. A hand-written `Vec`-like type that stores `*mut T` is therefore **invariant** unless you use something like `NonNull<T>` (covariant) and say so deliberately (Lesson 36).

---

## 5. Higher-ranked trait bounds: `for<'a>`

Sometimes a bound must hold for **every** lifetime, not one particular lifetime chosen by the caller. That's a higher-ranked trait bound (HRTB):

```rust
fn call_with_local<F>(f: F) -> usize where F: for<'a> Fn(&'a str) -> usize {
    let s = String::from("local text");
    f(&s)
}
fn call_sugar<F: Fn(&str) -> usize>(f: F) -> usize { let s = String::from("hello"); f(&s) }
fn main() { println!("{} {}", call_with_local(|s| s.len()), call_sugar(|s| s.len() * 2)); }
```

✓ Output `10 10`. Read `for<'a> Fn(&'a str) -> usize` as: "for **all** lifetimes `'a`, `F` can be called with a `&'a str`." Inside `call_with_local`, the string `s` has a lifetime that no caller could name, so `F` *must* accept every lifetime. The second version shows that you already write HRTBs all the time: `Fn(&str) -> usize` is **sugar** for the higher-ranked form (Lesson 11's elision rules, applied inside `Fn(...)` sugar).

What goes wrong without it:

```rust
fn call_with_local<'x, F>(f: F) -> usize where F: Fn(&'x str) -> usize {
    let s = String::from("local text");
    f(&s)               // F only promised to work for ONE lifetime 'x, chosen by the caller
}
fn main() { println!("{}", call_with_local(|s| s.len())); }
```

✓ `error[E0597]: `s` does not live long enough`. A fixed `'x` outlives the function body; `&s` doesn't.

### HRTB where the output borrows from the input

```rust
fn ident(s: &str) -> &str { s }
fn apply<F>(f: F) -> usize where F: for<'a> Fn(&'a str) -> &'a str { f("abc").len() }
fn main() { println!("{}", apply(ident)); println!("{}", apply(|s| &s[1..])); }
```

✓ Output `3` and `2`. And storing such a callback in a struct:

```rust
struct Handler { f: Box<dyn for<'a> Fn(&'a str) -> &'a str> }
struct Handler2 { f: Box<dyn Fn(&str) -> &str> }   // same, via elision
fn main() {
    let h = Handler { f: Box::new(|s| s.trim()) };
    let h2 = Handler2 { f: Box::new(|s| &s[..1]) };
    println!("[{}] [{}]", (h.f)("  x  "), (h2.f)("yz"));
}
```

✓ Output `[x] [y]`. (`dyn` and trait objects are Lesson 21.)

### The closure-inference trap

A closure that returns its borrowed argument, bound to a variable with no expected type, can fail:

```rust
fn main() {
    let f = |s: &str| s;       // closure signature inferred in a way that fails
    let a = String::from("x");
    println!("{}", f(&a));
}
```

✓ `error: lifetime may not live long enough`. When a closure is passed **directly** to a function with an `Fn(&'a str) -> &'a str` bound, the compiler knows the expected shape and infers the higher-ranked signature; when it's first bound to a `let`, there is no expected type to guide it. Fixes: pass the closure straight into the function that bounds it (as above), or use a named `fn` item (✓ `apply(ident)` works), or give an explicit signature through a helper function whose parameter has the bound. This is a known sharp edge, not something you did wrong.

---

## 6. Reading borrow-checker diagnostics

You have now seen the whole family. The practical skill is reading the output, which has a regular structure. Here is a real one ✓:

```text
error[E0502]: cannot borrow `v` as mutable because it is also borrowed as immutable
 --> q03_msg_read.rs:4:5
  |
3 |     let first = &v[0];
  |                  - immutable borrow occurs here
4 |     v.clear();
  |     ^^^^^^^^^ mutable borrow occurs here
5 |     println!("{first}");
  |                ----- immutable borrow later used here
```

**Method:**

1. **Read the title.** It names the *conflict* (shared vs mutable, moved vs used, outlives).
2. **Find the three labels.** (a) where the first borrow *starts* ("occurs here"), (b) where the conflicting action happens (`^^^` carets), (c) where the first borrow is *used again* ("later used here"). The borrow is live from (a) to (c).
3. **Fix by breaking the overlap.** End the first borrow earlier (use the value sooner, copy out what you need), or move the conflicting action after (c), or change what is borrowed (clone, split into disjoint pieces, take ownership).
4. **Read the `help:` and `note:` lines**, but verify them. Suggestions like "consider cloning" fix the error mechanically and may be the wrong design; "consider adding `'static`" is usually wrong (Lesson 11).
5. **`rustc --explain E0502`** (or `cargo`'s link) prints a longer explanation with examples.

Other messages from this module, as the compiler actually prints them ✓:

- **Closure borrows:** `error[E0502]: cannot borrow `count` as immutable because it is also borrowed as mutable` with the label "first borrow occurs due to use of `count` in closure". A closure that mutates a captured variable holds a `&mut` to it for as long as the closure is alive: `let mut inc = || count += 1; println!("{count}"); inc();` fails because `inc` is used again at the end.
- **Move in a loop:** `error[E0382]: use of moved value: `s`` with "value moved here, in previous iteration of loop" and a `help: consider cloning`. The real fix is usually to pass a reference or restructure so the value is consumed once.
- **Thread closure:** `error[E0373]: closure may outlive the current function, but it borrows `v`` with `help: ... use the `move` keyword`. Here the suggestion is right, if you really want the thread to own `v`.
- **`error: lifetime may not live long enough`** has no code. It says "this function promised A but is returning B"; look at the signature, the labels name which lifetime must outlive which, and check for invariance (Sections 3–4).

Errors in this family often come **in pairs** (the naive `IterMut` below prints a "lifetime may not live long enough" error *and* an E0506 for the same line). Fix the first, recompile, and the second often disappears.

---

## 7. The hard borrow problems and the patterns that solve them

### 7.1 `take`, `replace`, `swap`: move out of a `&mut`

You cannot move out of a reference (E0507, Lesson 07), but you can swap something in. These functions are the standard escape hatch, and the reason `Option<Box<Node>>` is the standard shape for linked structures:

```rust
fn main() {
    let mut a = vec![1, 2];
    let mut b = vec![3];
    let tmp = std::mem::replace(&mut a, vec![]);
    a.extend(&b);
    b = tmp;
    println!("{a:?} {b:?}");
    let mut o = Some(5);
    let taken = o.take();
    println!("{o:?} {taken:?}");
}
```

✓ Output `[3] [1, 2]`, `None Some(5)`.

### 7.2 Method calls borrow the whole struct; field accesses borrow one field

```rust
struct App { items: Vec<i32>, total: i32 }
impl App {
    fn add_to_total(&mut self, x: i32) { self.total += x; }
    fn sum_bad(&mut self) { for x in &self.items { self.add_to_total(*x); } }
}
fn main() { let mut a = App { items: vec![1, 2], total: 0 }; a.sum_bad(); println!("{}", a.total); }
```

✓ `E0502`: the loop borrows `self.items`, and `self.add_to_total` needs all of `*self` mutably. (Lesson 10 Section 6.) Three fixes ✓ (all compile and produced the same totals):

```rust
struct App { items: Vec<i32>, total: i32 }
fn add_to(total: &mut i32, x: i32) { *total += x; }
impl App {
    fn sum_fields(&mut self) { for x in &self.items { add_to(&mut self.total, *x); } }   // disjoint fields
    fn sum_copy(&mut self) { let items = self.items.clone(); for x in items { self.total += x; } }
    fn sum_take(&mut self) { let items = std::mem::take(&mut self.items); for x in &items { self.total += *x; } self.items = items; }
}
fn main() {
    let mut a = App { items: vec![1, 2], total: 0 };
    a.sum_fields(); a.sum_copy(); a.sum_take();
    println!("{} {:?}", a.total, a.items);
}
```

✓ Output `9 [1, 2]`. Prefer the first (pass only the fields a helper needs, which also documents what it touches), or restructure into two structs. The `clone` costs an allocation; the `take` dance is correct but easy to break if the loop body can `return` or panic (the vector would be left empty).

### 7.3 Indices instead of references (arenas)

When data forms a graph, store nodes in a `Vec` and refer to them by **index**. Indices are plain `Copy` numbers, so they carry no borrow:

```rust
#[derive(Debug)]
struct Node { name: &'static str, children: Vec<usize> }
fn main() {
    let mut arena = vec![Node { name: "root", children: vec![] }, Node { name: "kid", children: vec![] }];
    arena[0].children.push(1);
    let first_child = arena[0].children[0];
    arena[first_child].name = "kid2";
    println!("{arena:?}");
}
```

✓ Output `[Node { name: "root", children: [1] }, Node { name: "kid2", children: [] }]`. The trade-off: an index can be stale or out of range (a logic error that panics or reads the wrong node, never memory unsafety), and nothing stops you from keeping an index after removing the node. Typed IDs (Lesson 15) and generational indices reduce that risk.

### 7.4 Linked-list cursors

Walking a `&mut Option<Box<Node>>` to the end of a list used to be a famous borrow-checker stumbling block. On `rustc 1.97.0` ✓ all of these compiled and ran: the plain `while let Some(node) = cur { cur = &mut node.next; }` loop (`push_back`), the older `while cur.is_some() { cur = &mut cur.as_mut().unwrap().next; }` form, and a loop that *returns* a `&mut` node from inside the walk when it finds a match (`find_or_push_back`). I am only reporting what 1.97 accepts; I don't know in which release each started to compile, and code written against older compilers may carry workarounds it no longer needs.

What is **still** rejected is the conditional-return shape from Lesson 11 (`if let Some(v) = map.get(&k) { return v; } map.insert(..)`, ✓ E0502 on this toolchain), so keep `entry` handy.

Reversing in place (✓ `[3, 2, 1]` becomes `[1, 2, 3]`) is a pure `take()` exercise:

```rust
fn reverse(&mut self) {
    let mut prev = None;
    let mut cur = self.head.take();
    while let Some(mut node) = cur {
        cur = node.next.take();     // detach the rest
        node.next = prev;            // point this node backwards
        prev = Some(node);           // move node into prev
    }
    self.head = prev;
}
```

### 7.5 A mutable iterator: the `take` trick

Handing out `&mut` items one by one needs the iterator to *give up* its own `&mut` to the current node so the item can outlive the call to `next`. The tempting version fails:

```rust
struct Node { v: i32, next: Option<Box<Node>> }
struct IterMut<'a> { next: Option<&'a mut Node> }
impl<'a> Iterator for IterMut<'a> {
    type Item = &'a mut i32;
    fn next(&mut self) -> Option<Self::Item> {
        match &mut self.next {
            Some(n) => { self.next = n.next.as_deref_mut(); Some(&mut n.v) }
            None => None,
        }
    }
}
fn main() {}
```

✓ Two errors: `error: lifetime may not live long enough` ("assignment requires that `'1` must outlive `'a`") and `error[E0506]: cannot assign to `self.next` because it is borrowed`. This is the **debugging challenge** (Section 14). The working version ✓ is in the practical challenge below.

### 7.6 Deep structures and recursive `Drop`

A `Box<Node>` chain drops recursively. ✓ Building a 200,000-node chain and dropping it on the main thread in a **debug** build gave `thread 'main' has overflowed its stack`, `fatal runtime error: stack overflow, aborting`, exit status 134 (Lesson 08's abort), while the same program compiled with `-O` ran to completion (the optimizer happened to remove the recursion; never rely on it). A hostile input that builds a deep structure can therefore crash a service. The cure is an iterative `Drop` (as in the practical challenge); ✓ removing it from my list makes the 200,000-element test abort with a stack overflow, so the test does protect against the regression. The same issue applies to any recursive algorithm over untrusted depth (parsing nested JSON, pretty-printing, comparing trees).

### 7.7 Last resorts

In order: (1) restructure (smaller functions that take fields, split borrows, indices, `take`); (2) clone small data; (3) move the check to run time with `Rc<RefCell<T>>`/`Arc<Mutex<T>>` (Lessons 23–24, 29), accepting panics or lock contention; (4) `unsafe` with a safe API (Lesson 36). Reaching for (3) because you don't yet see (1) is how Rust code ends up with graphs of `Rc<RefCell<..>>` that panic with `already borrowed` in production.

---

## 8. Common mistakes and edge cases

- **Assuming the compiler rejects only wrong code.** It also rejects some correct code (conditional returns of borrows); know the alternatives.
- **Writing `T: 'static` or `&'static` to make an error go away** (Lesson 11).
- **Letting a `&mut` or `Cell` sneak into a type and expecting covariance.** If you can write through it, it's invariant.
- **Using `*mut T` in a hand-rolled container** and getting invariance by accident; use `NonNull<T>` plus `PhantomData<T>` (Lesson 36).
- **Treating `Fn(&str) -> &str` closures bound by `let`** as always working; pass them straight to the bounded function or use an `fn` item.
- **Recursive `Drop` or recursive traversal on untrusted depth.**
- **Building a long-lived `Rc<RefCell<..>>` graph to avoid learning the borrow rules**; it moves failures from compile time to production.
- **Believing a clever `unsafe` workaround "just" removes the error.** If the compiler is right, you have created undefined behavior.

---

## 9. Performance implications

- Variance, subtyping and HRTBs have **no run-time cost**; they are erased like all lifetimes.
- A singly linked list is almost never faster than a `Vec` or `VecDeque` for general use: nodes are separate allocations, so traversal chases pointers (typically poor cache behavior) and every push allocates. I did not benchmark it; Lesson 41 shows how to measure rather than guess. Use lists to teach ownership, or when you need O(1) splicing and have measured.
- Iterative `Drop` costs a loop but avoids the stack overflow; it does not change the asymptotic cost.
- `clone`-to-appease-the-checker is an allocation per call; prefer restructuring in hot paths.

---

## 10. Security and reliability

- **Stack exhaustion from deep structures** (Section 7.6) is a denial-of-service vector: treat nesting depth as untrusted input, add depth limits, and use iterative algorithms or an explicit stack where depth is attacker-controlled.
- The borrow checker eliminates use-after-free and aliasing bugs in safe code; it does not prevent logic errors with indices (stale or out-of-range), panics from `RefCell`, or deadlocks from `Mutex`.
- Wrongly declared variance in `unsafe` containers is a soundness bug: if your type is covariant but allows writes, safe code can build a dangling reference. When writing such types, add a `compile_fail` test for the invariance you rely on (as the practical challenge does).
- Prefer returning `Result` over `unwrap` in code that walks structures built from input. `linkedlab` itself has no `unwrap`/`expect` in library code: absent elements are `Option`/`None`.

---

## 11. Exercises

Try each before opening the answer. They are not marked complete unless you tell me you did them.

1. **(Easy)** For each type say covariant, contravariant or invariant in `T`: `Vec<T>`, `&mut T`, `Cell<T>`, `fn(T)`, `fn() -> T`, `*const T`, `*mut T`.
2. **(Easy)** Why does `let mut s: &'static str = "x"; let r = &mut s; *r = &String::from("y");` not compile?
3. **(Medium)** Write `fn longest_by<F>(a: &str, b: &str, key: F) -> usize where F: ...` so that it calls `key` on both strings and on a local `String` you create inside. What must the bound on `F` look like, and why?
4. **(Medium)** The closure `let f = |s: &str| s;` fails to compile. Show two ways to get a working "return the argument" function value.
5. **(Medium)** Fix `sum_bad` from Section 7.2 in a way that does not clone and does not use `mem::take`.
6. **(Hard)** Explain why `IterMut::next` needs `self.next.take()`, and why `Iter::next` (the shared version) does not.
7. **(Hard)** Make `List<T>` safe against deep recursion in `Drop`, and write a test that proves it. What would you assert, and how did I confirm the test is meaningful?
8. **(Hard)** A struct has fields `a: &'a str` and `b: Cell<&'b str>`. What variance does it have over `'a` and over `'b`, and what does that imply for converting `S<'long, 'long>` to `S<'short, 'short>`?

<details markdown="1">
<summary>Answers</summary>

1. ✓ for `Vec<T>` (covariant), `&mut T` (invariant), `Cell<T>` (invariant), `fn(T)` (contravariant in the argument), `*const T` (covariant), `*mut T` (invariant). `fn() -> T` is covariant in `T` (documented; I did not run a test).
2. ✓ It fails with `E0716: temporary value dropped while borrowed`. The slot is typed `&'static str`, and because `&mut T` is invariant in `T` that type cannot be shortened to accept a shorter-lived reference. With a named local instead of a temporary (`let local = String::from("y"); *r = &local;`) the same rule gives E0597 ✓ (Section 3).
3. `F: for<'a> Fn(&'a str) -> usize`. The function passes a reference to a local `String`, whose lifetime no caller could have named, so `F` must accept every lifetime (✓ `call_with_local` in the practical challenge; the fixed-lifetime version fails with E0597).
4. ✓ Pass it directly where the bound is known (`apply(|s| s)` printed `3` with `F: for<'a> Fn(&'a str) -> &'a str`), or use a function item: `fn ident(s: &str) -> &str { s }` and then `apply(ident)`.
5. ✓ Pass only the field a helper needs (`fn add_to(total: &mut i32, x: i32)`, called as `add_to(&mut self.total, *x)`) or inline the body: `for x in &self.items { self.total += *x; }`. Both borrow disjoint fields.
6. `IterMut` stores an `Option<&'a mut Node>`, and a `&mut` is not `Copy`. To return `&'a mut node.v` the iterator must hand the node's `&mut` to the caller and move on; `self.next.take()` moves the `&mut` out of `self` (leaving `None`), so there is exactly one live `&mut` per node. ✓ The version that borrowed `&mut self.next` instead failed with the E0506 / "lifetime may not live long enough" pair. `Iter` holds `Option<&'a Node>`, which is `Copy`, so `self.next.map(...)` copies it out without any trick.
7. Implement `Drop` for `List<T>` with a loop that `take`s each node's `next` (as in `linkedlab`). Test: push 200,000 elements then drop. ✓ Without the `Drop` impl this test aborts with `has overflowed its stack` on this toolchain (debug build, test threads have a small stack), and with it the test passes. Note the number comes from my run: other stack sizes and optimization levels will need different counts.
8. Covariant over `'a` (a shared reference field) and **invariant** over `'b` (`Cell` allows writes). `S<'long, 'long>` cannot become `S<'short, 'short>` because of `'b`; only `'a` can shorten. (From the rules in Sections 3–4; I did not compile this exact struct.)

</details>

---

## 12. Debugging challenge

A teammate wrote a mutable iterator over a linked list:

```rust
struct Node { v: i32, next: Option<Box<Node>> }
struct IterMut<'a> { next: Option<&'a mut Node> }
impl<'a> Iterator for IterMut<'a> {
    type Item = &'a mut i32;
    fn next(&mut self) -> Option<Self::Item> {
        match &mut self.next {
            Some(n) => { self.next = n.next.as_deref_mut(); Some(&mut n.v) }
            None => None,
        }
    }
}
```

✓ It produces two errors (`lifetime may not live long enough`: "assignment requires that `'1` must outlive `'a`", and `E0506: cannot assign to `self.next` because it is borrowed`). The function "looks" right: it walks forward and hands out each value. What is the compiler protecting you from, and what is the fix?

<details markdown="1">
<summary>Solution</summary>

`match &mut self.next` borrows the field `self.next` for the lifetime of the **method call** (`'1`, the `&mut self` borrow), then the code tries to *assign to that same field* while `n` (derived from the borrow) is still live and also hand out `&mut n.v` with the longer lifetime `'a`. Allowing it would let the iterator overwrite the very slot that `n` and the returned `&mut i32` borrow from, and the returned reference would be tied to the iterator's `&mut self` instead of to the list.

Fix: **move the `&mut Node` out of the iterator** so nothing in `self` is borrowed, then store the next link back:

```rust
fn next(&mut self) -> Option<Self::Item> {
    self.next.take().map(|node| {
        self.next = node.next.as_deref_mut();
        &mut node.v
    })
}
```

✓ With this version a `for x in list.iter_mut() { *x *= 10; }` loop turned `1, 2` into `10 20`. `take()` leaves `None` in `self.next` and gives you the original `&'a mut Node`, whose lifetime is `'a`, not the lifetime of the `&mut self` call; each node's `&mut` exists in exactly one place at a time, which is the aliasing rule (Lesson 10) satisfied by construction.

</details>

---

## 13. Practical challenge: `linkedlab`

Create a library crate `linkedlab` (edition 2024) with:

1. A generic `List<T>` (private `Node<T>`, `head: Option<Box<Node<T>>>`, a `len` counter) with `new`, `len`, `is_empty`, `push_front`, `pop_front`, `push_back` (cursor walk), `peek`, `peek_mut`, `reverse`, `iter`, `iter_mut`, `find_or_push_back` (returns `&mut T`, appends if absent), and `FromIterator`.
2. `Iter<'a, T>` with `Item = &'a T` and `IterMut<'a, T>` with `Item = &'a mut T`, written with the `take` pattern so that two items can be held at once.
3. An **iterative `Drop`** with a test that builds and drops a 200,000-node list.
4. `overwrite<'a>(slot: &mut &'a str, val: &'a str)` with a `compile_fail` doctest demonstrating invariance, and `call_with_local<F>` with an HRTB bound.
5. Tests for: empty/one/many cases, `len` bookkeeping, `reverse`, holding two `iter_mut` items at once, each element dropped exactly once (use `Rc::strong_count` as a probe), the HRTB function with both a closure and an `fn` item.
6. It must pass `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, and `cargo test --locked` (run `cargo generate-lockfile` once first).

<details markdown="1">
<summary>Solution</summary>

**`src/lib.rs`**

```rust
//! linkedlab: a singly linked list that exercises the hard corners of the borrow checker
//! (cursors, `take`, mutable iterators), plus small demonstrations of invariance and
//! higher-ranked bounds.
//!
//! A linked list is a teaching device here; `Vec` or `VecDeque` is almost always faster.

struct Node<T> {
    value: T,
    next: Option<Box<Node<T>>>,
}

/// A singly linked list with O(1) `push_front`/`pop_front` and O(n) `push_back`.
pub struct List<T> {
    head: Option<Box<Node<T>>>,
    len: usize,
}

impl<T> Default for List<T> {
    fn default() -> Self {
        Self::new()
    }
}

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, len: 0 }
    }

    pub fn len(&self) -> usize {
        self.len
    }

    pub fn is_empty(&self) -> bool {
        self.len == 0
    }

    pub fn push_front(&mut self, value: T) {
        // `take` leaves `None` behind, so the old head is *moved* without a borrow.
        let old = self.head.take();
        self.head = Some(Box::new(Node { value, next: old }));
        self.len += 1;
    }

    pub fn pop_front(&mut self) -> Option<T> {
        self.head.take().map(|node| {
            let node = *node;
            self.head = node.next;
            self.len -= 1;
            node.value
        })
    }

    /// Walks a `&mut Option<Box<Node>>` cursor to the empty slot at the end.
    pub fn push_back(&mut self, value: T) {
        let mut cur = &mut self.head;
        while let Some(node) = cur {
            cur = &mut node.next;
        }
        *cur = Some(Box::new(Node { value, next: None }));
        self.len += 1;
    }

    pub fn peek(&self) -> Option<&T> {
        self.head.as_ref().map(|n| &n.value)
    }

    pub fn peek_mut(&mut self) -> Option<&mut T> {
        self.head.as_mut().map(|n| &mut n.value)
    }

    /// Reverses in place by re-linking nodes: no allocation, no clones.
    pub fn reverse(&mut self) {
        let mut prev = None;
        let mut cur = self.head.take();
        while let Some(mut node) = cur {
            cur = node.next.take();
            node.next = prev;
            prev = Some(node);
        }
        self.head = prev;
    }

    pub fn iter(&self) -> Iter<'_, T> {
        Iter {
            next: self.head.as_deref(),
        }
    }

    pub fn iter_mut(&mut self) -> IterMut<'_, T> {
        IterMut {
            next: self.head.as_deref_mut(),
        }
    }

    /// Returns a mutable reference to the first element equal to `value`, appending it if absent.
    pub fn find_or_push_back(&mut self, value: T) -> &mut T
    where
        T: PartialEq,
    {
        let mut cur = &mut self.head;
        while let Some(node) = cur {
            if node.value == value {
                return &mut node.value;
            }
            cur = &mut node.next;
        }
        self.len += 1;
        &mut cur.insert(Box::new(Node { value, next: None })).value
    }
}

/// Dropping a long list recursively would overflow the stack (each `Box<Node>` drop calls the
/// next), so unlink iteratively. See `long_list_drops_without_overflow`.
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut cur = self.head.take();
        while let Some(mut node) = cur {
            cur = node.next.take();
        }
    }
}

pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
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

pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<&'a mut T> {
        // `take` moves the `&mut` out of `self`, so each node is handed out exactly once.
        self.next.take().map(|node| {
            self.next = node.next.as_deref_mut();
            &mut node.value
        })
    }
}

impl<T> FromIterator<T> for List<T> {
    fn from_iter<I: IntoIterator<Item = T>>(iter: I) -> Self {
        let mut list = List::new();
        let mut tail = &mut list.head;
        for value in iter {
            let node = tail.insert(Box::new(Node { value, next: None }));
            tail = &mut node.next;
            list.len += 1;
        }
        list
    }
}

/// Writes `val` into `slot`. Because `&mut T` is **invariant** in `T`, `slot`'s `&'a str`
/// cannot be silently treated as a longer-lived `&'static str` by the caller:
///
/// ```compile_fail
/// let mut s: &'static str = "static";
/// let local = String::from("short");
/// linkedlab::overwrite(&mut s, &local);
/// println!("{s}");
/// ```
pub fn overwrite<'a>(slot: &mut &'a str, val: &'a str) {
    *slot = val;
}

/// Calls `f` with a reference to a string that exists only inside this function.
/// `F: for<'a> Fn(&'a str) -> usize` (a higher-ranked bound) means `f` must work for **every**
/// lifetime, which is the only way it can accept our local.
pub fn call_with_local<F>(f: F) -> usize
where
    F: for<'a> Fn(&'a str) -> usize,
{
    let s = String::from("local text");
    f(&s)
}

#[cfg(test)]
mod tests {
    use super::*;

    fn to_vec<T: Clone>(l: &List<T>) -> Vec<T> {
        l.iter().cloned().collect()
    }

    #[test]
    fn push_pop_front() {
        let mut l = List::new();
        assert!(l.is_empty() && l.pop_front().is_none());
        l.push_front(1);
        l.push_front(2);
        assert_eq!((l.len(), l.peek()), (2, Some(&2)));
        assert_eq!(l.pop_front(), Some(2));
        assert_eq!(l.pop_front(), Some(1));
        assert_eq!(l.pop_front(), None);
        assert_eq!(l.len(), 0);
    }

    #[test]
    fn push_back_keeps_order() {
        let mut l = List::new();
        for i in 1..=4 {
            l.push_back(i);
        }
        assert_eq!(to_vec(&l), [1, 2, 3, 4]);
        assert_eq!(l.len(), 4);
    }

    #[test]
    fn peek_mut_and_iter_mut() {
        let mut l: List<i32> = (1..=3).collect();
        *l.peek_mut().unwrap() += 100;
        for x in l.iter_mut() {
            *x *= 2;
        }
        assert_eq!(to_vec(&l), [202, 4, 6]);
        assert!(List::<i32>::new().peek_mut().is_none());
    }

    #[test]
    fn iter_mut_items_can_be_held_together() {
        // Possible only because `Item = &'a mut T` borrows the list, not the iterator.
        let mut l: List<i32> = (1..=3).collect();
        let mut it = l.iter_mut();
        let a = it.next().unwrap();
        let b = it.next().unwrap();
        std::mem::swap(a, b);
        assert_eq!(to_vec(&l), [2, 1, 3]);
    }

    #[test]
    fn reverse_cases() {
        let mut empty: List<i32> = List::new();
        empty.reverse();
        assert!(empty.is_empty());
        let mut one: List<i32> = [7].into_iter().collect();
        one.reverse();
        assert_eq!(to_vec(&one), [7]);
        let mut l: List<i32> = (1..=4).collect();
        l.reverse();
        assert_eq!(to_vec(&l), [4, 3, 2, 1]);
        assert_eq!(l.len(), 4);
    }

    #[test]
    fn find_or_push_back_finds_and_appends() {
        let mut l: List<i32> = List::new();
        l.find_or_push_back(1);
        l.find_or_push_back(2);
        assert_eq!(l.len(), 2);
        // Existing key: found, not appended. Mutating it changes the stored value.
        *l.find_or_push_back(1) *= 10;
        assert_eq!((to_vec(&l), l.len()), (vec![10, 2], 2));
        // New key: appended at the end.
        *l.find_or_push_back(7) += 1;
        assert_eq!((to_vec(&l), l.len()), (vec![10, 2, 8], 3));
    }

    #[test]
    fn long_list_drops_without_overflow() {
        let mut l = List::new();
        for i in 0..200_000u32 {
            l.push_front(i);
        }
        assert_eq!(l.len(), 200_000);
        drop(l);
    }

    #[test]
    fn drops_every_element_exactly_once() {
        use std::rc::Rc;
        let probe = Rc::new(());
        {
            let mut l = List::new();
            for _ in 0..5 {
                l.push_back(Rc::clone(&probe));
            }
            assert_eq!(Rc::strong_count(&probe), 6);
            let _ = l.pop_front();
            assert_eq!(Rc::strong_count(&probe), 5);
        }
        assert_eq!(Rc::strong_count(&probe), 1);
    }

    #[test]
    fn overwrite_with_same_lifetime() {
        let a = String::from("a");
        let b = String::from("b");
        let mut s: &str = &a;
        overwrite(&mut s, &b);
        assert_eq!(s, "b");
    }

    #[test]
    fn higher_ranked_bound_accepts_closures_and_fns() {
        assert_eq!(call_with_local(|s| s.len()), 10);
        fn count_spaces(s: &str) -> usize {
            s.matches(' ').count()
        }
        assert_eq!(call_with_local(count_spaces), 1);
    }
}
```

✓ With this exact file: `cargo fmt --check` clean, `cargo clippy --all-targets --locked -- -D warnings` clean, `cargo test --locked` → 10 unit tests and 1 doctest passed.

**Design notes.**

- `push_front`/`pop_front` use `Option::take` to move the old head instead of borrowing it. `pop_front` unboxes with `let node = *node;` so it can move `node.next` and `node.value` out separately (you can move out of a `Box` by dereferencing it, unlike out of a `&mut`).
- `push_back` and `find_or_push_back` use the cursor form `let mut cur = &mut self.head; while let Some(node) = cur { cur = &mut node.next; }`. On this toolchain both compile (Section 7.4). `find_or_push_back` ends with `cur.insert(...)`, which stores the new node and returns `&mut Box<Node<T>>` in one step.
- `FromIterator` builds the list in order in O(n) by keeping a `tail: &mut Option<Box<Node<T>>>` cursor. The `list.len += 1` and `tail` borrows touch different fields of `list`, so they coexist.
- `Drop` for `List<T>` unlinks nodes in a loop: each `node` is dropped at the end of its iteration with `next` already detached, so drop depth stays constant. Dropping `List` is the only place that needs this since `Node` has no `Drop` impl of its own.
- `drops_every_element_exactly_once` counts strong references to an `Rc<()>` probe: 6 after 5 pushes, 5 after one pop, 1 after the list is dropped. It would catch a leak or a double drop in the list code.
- One test of mine was wrong on the first run (`find_or_push_back` mutated the very key it later searched for, so the next search correctly missed): the code was fine, the test was not, and I rewrote the test to avoid mutating the key.
- The `compile_fail` doctest passes if the snippet fails to compile *for any reason*. I confirmed the identical shape fails with E0597 in Section 3 but did not check the error code inside the doctest.

</details>

---

## 14. Interview questions

<details markdown="1">
<summary>Answers</summary>

1. **What is variance?** The rule for how a generic type's subtyping relates to its parameter's: covariant (same direction), contravariant (reversed), invariant (none). In Rust it applies to lifetimes (and, through them, types containing lifetimes).
2. **Is `&'a T` covariant?** Yes, in both `'a` and `T`: a longer-lived reference can stand in for a shorter-lived one.
3. **Why is `&mut T` invariant in `T`?** Because you can write a `T` through it. If it were covariant you could store a shorter-lived value into a slot others believe is longer-lived, creating a dangling reference.
4. **Which standard types are invariant in `T`, and why?** `Cell<T>`, `RefCell<T>`, `Mutex<T>`, `UnsafeCell<T>`, `*mut T`: all permit writing a `T` through a shared or raw handle.
5. **Where does contravariance show up?** Function arguments: `fn(&'a T)` is contravariant in `'a`; a function accepting any lifetime is usable where one accepting `'static` is expected.
6. **How is a struct's variance determined?** From its fields; the most restrictive position wins. `PhantomData<X>` lets you declare it for parameters that have no field.
7. **What is a higher-ranked trait bound?** `for<'a> F: Fn(&'a T)`: the bound holds for every lifetime `'a`, not one fixed lifetime picked by the caller. `Fn(&str)` is sugar for it.
8. **When do you need `for<'a>` written out?** When you must name the lifetime in the output or in multiple places, e.g. `for<'a> Fn(&'a str) -> &'a str`, or in a `dyn` type in a struct.
9. **Why did `let f = |s: &str| s;` fail?** With no expected signature, closure inference doesn't produce the higher-ranked type; passing it directly to a function with an `Fn(&str) -> &str` bound, or using a function item, works.
10. **How do you read a borrow error?** Find where the first borrow starts, the conflicting action, and where the first borrow is used again; fix by breaking that overlap. Then consider the suggestions.
11. **Why does `self.next.take()` appear in a mutable linked-list iterator?** To move the `&mut Node` out of the iterator so it can be handed out for lifetime `'a`, instead of borrowing it through `&mut self`.
12. **Why implement `Drop` iteratively for a linked list?** A `Box` chain drops recursively; a long list overflows the stack (observed: 200,000 nodes in a debug build, exit 134).
13. **How do you avoid borrowing all of `self` in a method?** Pass only the needed fields to helper functions, split the struct, or use disjoint field accesses directly.
14. **When is `Rc<RefCell<T>>` the right answer to a borrow error?** When the data really is shared with dynamic lifetimes (graphs, observers); not as a way to avoid learning the borrow rules. It trades compile errors for runtime panics.

</details>

---

## 15. Key takeaways

- Rust's subtyping is about lifetimes. **Covariant**: read-only holders (`&T`, `Box<T>`, `Vec<T>`, `*const T`, return types). **Invariant**: anything you can write through (`&mut T`, `Cell<T>`, `*mut T`). **Contravariant**: function arguments.
- If you can write a `T` through it, it's invariant in `T`; "lifetime may not live long enough" errors that surprise you often hide an invariant position (the compiler's note even says so).
- `for<'a>` means "for every lifetime"; `Fn(&str) -> usize` is sugar for it. You need it whenever the callee creates the reference itself.
- To read a borrow error: title → the three labels (start, conflict, later use) → break the overlap → check the `help`.
- Patterns for hard borrows: `take`/`replace`/`swap`, helpers that take fields, indices into an arena, `entry`, the `take` trick for `&mut` iterators, and (last) run-time checks or `unsafe`.
- Recursive `Drop` on deep structures overflows the stack; make it iterative and treat depth as untrusted input.
- Several classic cursor problems compile on this toolchain (1.97); the conditional-return-then-mutate shape still does not.

**Connections.** This closes Module 4. Lesson 10's aliasing rule and Lesson 11's lifetimes now have their type-theory footing (variance, HRTB). Lesson 08's drop order and stack limits explain the deep-list overflow. Next: Module 5 starts with Lesson 13 (structs, enums, methods), where lifetimes in `impl` blocks and `Self` appear constantly. Lesson 18 (closures) returns to `Fn`/`FnMut`/`FnOnce` and the capture rules behind the closure errors above; Lesson 19–21 apply HRTBs to generics and trait objects; Lesson 22 covers `PhantomData`, `Send`/`Sync` and GATs; Lesson 24 moves aliasing-xor-mutability to run time (`Cell`, `RefCell`); Lesson 36 shows how `unsafe` lets you declare variance for hand-written pointer types. See the [roadmap](00_roadmap.md) and the [course index](index.md).

---

## 16. References

- *The Rustonomicon*, "Subtyping and Variance": <https://doc.rust-lang.org/nomicon/subtyping.html>
- *The Rust Reference*, "Subtyping and variance": <https://doc.rust-lang.org/reference/subtyping.html>
- *The Rustonomicon*, "Higher-Rank Trait Bounds": <https://doc.rust-lang.org/nomicon/hrtb.html>
- *The Rust Reference*, "Trait and lifetime bounds" (including `for<'a>`): <https://doc.rust-lang.org/reference/trait-bounds.html>
- *Learning Rust With Entirely Too Many Linked Lists*: <https://rust-unofficial.github.io/too-many-lists/>
- *The Rust Programming Language*, ch. 4 (ownership), ch. 10.3 (lifetimes), ch. 15 (smart pointers): <https://doc.rust-lang.org/book/>
- Rust standard library documentation: `std::mem::take`, `replace`, `swap`; `Option::take`, `as_deref_mut`, `Option::insert`: <https://doc.rust-lang.org/std/mem/fn.take.html>, <https://doc.rust-lang.org/std/option/enum.Option.html#method.take>, <https://doc.rust-lang.org/std/option/enum.Option.html#method.insert>
- Rust standard library documentation: `std::marker::PhantomData`, `std::ptr::NonNull`, `std::cell::Cell`: <https://doc.rust-lang.org/std/marker/struct.PhantomData.html>, <https://doc.rust-lang.org/std/ptr/struct.NonNull.html>, <https://doc.rust-lang.org/std/cell/struct.Cell.html>
- Rust error code index: E0373, E0382, E0499, E0502, E0506, E0507, E0597: <https://doc.rust-lang.org/error_codes/E0373.html>, <https://doc.rust-lang.org/error_codes/E0382.html>, <https://doc.rust-lang.org/error_codes/E0502.html>, <https://doc.rust-lang.org/error_codes/E0506.html>, <https://doc.rust-lang.org/error_codes/E0597.html>
- The Polonius project (a more precise borrow-checking analysis): <https://rust-lang.github.io/polonius/>

The URLs above are the standard documentation locations; I did not open every one during this session, so if a link has moved, search the docs site for the page title. Unchecked from memory: the full variance table for standard types beyond the entries marked ✓, and the status of Polonius.
