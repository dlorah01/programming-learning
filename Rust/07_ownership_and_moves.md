---
layout: default
title: "Lesson 07 — Ownership, moves, Copy, and Clone"
---

# Lesson 07 — Ownership, moves, `Copy`, and `Clone`

**Module 3 · Ownership and memory** · Status: generated · Edition used in examples: **2024** · External dependencies: none

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux. As before, ✓ marks outputs and diagnostics I actually ran; unmarked statements are documented behavior I did not run. Sizes are for a **64-bit** target.

Builds on [Lesson 01](01_foundations.md) (ownership at a glance), [Lesson 06](06_strings_conversions_patterns.md) (`String`, matching moves), and [Lesson 05](05_functions_and_control_flow.md) (slices). This lesson explains the `E0382` errors you have already met.

---

## 1. The problem ownership solves

Every program that allocates memory must decide **who frees it, and when**.

| Approach | Who frees | Cost | Failure mode |
|---|---|---|---|
| Manual (C, C++ `new`/`delete`) | The programmer | No runtime overhead | Leaks, double free, use-after-free |
| Garbage collection (Java, Go, Python) | A runtime collector | Memory overhead, pauses, a runtime | Mostly none for memory; resources (files, locks) still need care |
| Reference counting everywhere | Each owner, dynamically | Counter updates, cycles leak | Cycles |
| **Ownership (Rust)** | The compiler decides statically; the program frees deterministically | No runtime overhead for the tracking itself | Compile errors instead of runtime bugs |

**History.** Rust's ownership system descends from academic work on *affine* and *linear* types (a value may be used at most once) and *region-based memory management*, which research languages such as Cyclone explored; C++ had already popularized RAII (resources are released in destructors, Lesson 08). Rust's contribution was making this the default, checked, ergonomic discipline of an industrial language, with borrowing (Lesson 10) as the escape hatch that makes it practical. Early Rust (before 1.0) had garbage-collected pointers and several pointer kinds; they were removed in favor of the model below. (The exact timeline is described in the project's own history; I am summarizing it, not quoting it.)

**The three rules** (from *The Rust Programming Language*):

1. Each value has exactly one **owner**.
2. There can be only one owner at a time.
3. When the owner goes out of scope, the value is **dropped** (its destructor runs and its memory is freed).

Everything in this lesson follows from these rules plus one more: *assigning, passing, or returning a value moves ownership* unless the type is `Copy`.

---

## 2. Moves

```rust
fn main() {
    let a = String::from("hello");
    let heap_before = a.as_ptr();
    let stack_a = &a as *const String as usize;
    let b = a;
    let heap_after = b.as_ptr();
    let stack_b = &b as *const String as usize;
    println!("heap buffer unchanged: {}", heap_before == heap_after);
    println!("binding moved to a new stack slot: {}", stack_a != stack_b);
    println!("{b}");
}
```

✓ output:

```text
heap buffer unchanged: true
binding moved to a new stack slot: true
hello
```

**What a move is, mechanically.** A `String` is a small struct on the stack (pointer, capacity, length: 24 bytes ✓) pointing at a heap buffer. `let b = a;` copies those 24 bytes (a shallow, bitwise copy) into `b`'s slot; the **heap buffer is not touched** ✓. The compiler then treats `a` as *uninitialized*: it will not let you read it, and it will not run `a`'s destructor. Exactly one destructor runs, for `b`. A deep copy would cost an allocation and a memcpy of the text; a move costs a handful of register or stack stores (and often nothing at all after optimization).

### 2.1 Using a moved value (E0382)

```rust
fn main() {
    let a = String::from("hello");
    let b = a;
    println!("{a} {b}");
}
```

✓ `error[E0382]: borrow of moved value: `a``, with three labels: "move occurs because `a` has type `String`, which does not implement the `Copy` trait", "value moved here", and "value borrowed here after move". It also suggests `a.clone()` — read that as "if a copy is genuinely what you want", not as the default fix.

Why forbid it? If both `a` and `b` were usable, both would drop at scope end and free the same heap buffer twice (a double free), or one would free it while the other still used it (use-after-free). Rust prevents the *possibility* by making the old name unusable.

### 2.2 Moves into functions

```rust
fn length(s: String) -> usize { s.len() }
fn main() {
    let s = String::from("hello");
    let n = length(s);
    println!("{n} {s}");
}
```

✓ `error[E0382]: borrow of moved value: `s``, plus a note: "consider changing this parameter type in function `length` to borrow instead if owning the value isn't necessary" pointing at `s: String`. Passing a value to a function moves it into the parameter; the callee now owns it and drops it when it returns (unless it returns or stores it).

Three ways to fix, in order of preference:

```rust
fn length(s: &str) -> usize { s.len() }        // 1. borrow (Lesson 10)
fn shout(mut s: String) -> String { s.push('!'); s } // 2. take ownership and give it back

fn main() {
    let s = String::from("hello");
    let n = length(&s);
    let s = shout(s);          // moved in, returned, rebound (shadowing, Lesson 04)
    println!("{n} {s}");
}
```

✓ prints `5 hello!`. The third option is `.clone()`, covered in Section 4.

**API design rule of thumb.** Take `&T` / `&str` / `&[T]` when you only need to read; take `&mut T` when you modify in place; take `T` by value when you need to store it, return it, or transform it into something else (the caller then knows the value is consumed).

### 2.3 Moves in loops and conditionals

```rust
fn consume(s: String) { println!("{s}"); }
fn main() {
    let s = String::from("x");
    for _ in 0..2 { consume(s); }
}
```

✓ `error[E0382]: use of moved value: `s`` with the label "value moved here, in previous iteration of loop". The compiler reasons about *every control-flow path*, including loops.

A move inside a branch makes the value "maybe moved" afterwards. The compiler handles this with **drop flags**: a hidden runtime boolean per such variable, so the destructor runs only if the value was not moved. ✓ Demonstration:

```rust
struct Noisy(&'static str);
impl Drop for Noisy { fn drop(&mut self) { println!("drop {}", self.0); } }
fn eat(n: Noisy) { println!("eating {}", n.0); }

fn main() {
    let flag = std::env::args().count() > 1;
    let n = Noisy("n");
    if flag { eat(n); }
    println!("end of main");
}
```

✓ With no arguments: `end of main`, then `drop n` (dropped at the end of `main`). ✓ With one argument: `eating n`, `drop n` (dropped at the end of `eat`), then `end of main`. Either way, `drop n` prints exactly once. (After the `if`, `n` cannot be *used*; only its drop is conditional.)

A moved-from variable can be **re-assigned** and used again, if it is `mut`:

```rust
fn main() {
    let mut s = String::from("one");
    let t = s;
    s = String::from("two");   // s is initialized again
    println!("{s} {t}");
}
```

✓ prints `two one`.

### 2.4 Who drops what, and in what order

```rust
struct Noisy(&'static str);
impl Drop for Noisy { fn drop(&mut self) { println!("drop {}", self.0); } }
fn eat(n: Noisy) { println!("in eat {}", n.0); }
fn pass(n: Noisy) -> Noisy { println!("in pass {}", n.0); n }

fn main() {
    let a = Noisy("a"); let b = Noisy("b"); let c = Noisy("c");
    eat(a);
    let b = pass(b);
    println!("main body {} {}", b.0, c.0);
}
```

✓ output:

```text
in eat a
drop a
in pass b
main body b c
drop b
drop c
```

- `a` is dropped when `eat` returns (the parameter's scope ends), *before* `main` continues.
- `b` was moved into `pass` and moved back out, so it survives to the end of `main`.
- Locals are dropped in **reverse order of declaration** at the end of their scope: the new `b` (declared last), then `c`. (The first `b` was already moved.) Lesson 08 covers `Drop`, temporaries, and field drop order in depth.

### 2.5 Partial moves

You can move one field out of a struct (if the struct itself has no `Drop` implementation); the other fields stay usable, but the whole struct does not.

```rust
#[derive(Debug)]
struct User { name: String, email: String, age: u32 }
fn main() {
    let u = User { name: "ada".into(), email: "a@x".into(), age: 36 };
    let name = u.name;
    println!("{name} {} {}", u.email, u.age);
    println!("{u:?}");
}
```

✓ `error[E0382]: borrow of partially moved value: `u``, "value partially moved here", note: "partial move occurs because `u.name` has type `String`, which does not implement the `Copy` trait". Line 6, which only touches `u.email` and `u.age`, was accepted; line 7, which needs all of `u`, was not.

### 2.6 What you *cannot* move out of

Ownership means "I am responsible for dropping this". You can't take that responsibility from something you merely borrow, or from a slot that must stay initialized.

```rust
fn main() {
    let v = vec![String::from("a"), String::from("b")];
    let first = v[0];
    println!("{first}");
}
```

✓ `error[E0507]: cannot move out of index of `Vec<String>``, with two suggestions: borrow (`&v[0]`) or clone (`v[0].clone()`). Moving `v[0]` out would leave a hole in the vector, and the vector's destructor would later drop an element that was already moved.

```rust
fn take(r: &String) -> String { *r }
fn main() { println!("{}", take(&String::from("x"))); }
```

✓ `error[E0507]: cannot move out of `*r` which is behind a shared reference`. The reference's owner is elsewhere and will drop the `String` itself.

**How to move out when you really need to** — replace the value with something else so the slot stays valid:

```rust
use std::mem;

#[derive(Debug)]
struct Slot { name: String, tags: Vec<String> }

fn main() {
    let mut s = Slot { name: "a".into(), tags: vec!["t1".into(), "t2".into()] };
    let old_tags = mem::take(&mut s.tags);              // leaves Vec::default() behind
    let old_name = mem::replace(&mut s.name, "b".into()); // leaves "b" behind
    println!("{old_tags:?} {old_name} {s:?}");
    let mut x = String::from("x");
    let mut y = String::from("y");
    mem::swap(&mut x, &mut y);
    println!("{x} {y}");
    let mut o = Some(3);
    let t = o.take();                                    // Option::take: leaves None
    println!("{o:?} {t:?}");
}
```

✓ output:

```text
["t1", "t2"] a Slot { name: "b", tags: [] }
y x
None Some(3)
```

`mem::take`, `mem::replace`, `mem::swap`, and `Option::take` move a value out of a `&mut` place by *putting a valid value back*. They are safe because the slot is never left uninitialized. For `Vec<T>`, `remove(i)`, `swap_remove(i)`, `pop()`, and `drain(..)` move elements out. `into_iter()` consumes the vector and yields each element by value.

### 2.7 Iteration and ownership

```rust
fn main() {
    let names = vec![String::from("a"), String::from("b")];
    for n in names { println!("{n}"); }
    println!("{}", names.len());
}
```

✓ `error[E0382]: borrow of moved value: `names``, with the label "`names` moved due to this implicit call to `.into_iter()`" and the help "consider iterating over a slice of the `Vec<String>`'s content to avoid moving into the `for` loop" (`for n in &names`). `for x in v` consumes `v`; `for x in &v` borrows it; `for x in &mut v` borrows it mutably. ✓ A borrowing iteration followed by a consuming one works:

```rust
fn main() {
    let v = vec![String::from("a"), String::from("b")];
    for s in &v { print!("{s} "); }
    println!("{}", v.len());
    let lens: Vec<usize> = v.iter().map(|s| s.len()).collect();
    let owned: Vec<String> = v.into_iter().map(|s| s + "!").collect();
    println!("{lens:?} {owned:?}");
}
```

✓ prints `a b 2` then `[1, 1] ["a!", "b!"]`.

---

## 3. `Copy`: types that are duplicated instead of moved

Some values are so cheap and so self-contained that "moving" them and "copying" them are the same thing, and the old name can safely stay valid. Such types implement the marker trait `Copy`.

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
struct Point { x: i32, y: i32 }

fn shift(p: Point) -> Point { Point { x: p.x + 1, ..p } }

fn main() {
    let a = 5; let b = a; println!("{a} {b}");
    let p = Point { x: 1, y: 2 };
    let q = shift(p);
    println!("{p:?} {q:?}");                 // p still usable
    let t = (1, 'c', 2.5); let u = t; println!("{t:?} {u:?}");
    let arr = [1, 2, 3]; let arr2 = arr; println!("{arr:?} {arr2:?}");
    let r = &p; let r2 = r; println!("{r:?} {r2:?}");
}
```

✓ output:

```text
5 5
Point { x: 1, y: 2 } Point { x: 2, y: 2 }
(1, 'c', 2.5) (1, 'c', 2.5)
[1, 2, 3] [1, 2, 3]
Point { x: 1, y: 2 } Point { x: 1, y: 2 }
```

| `Copy` | Not `Copy` |
|---|---|
| All integers, floats, `bool`, `char` | `String`, `Vec<T>`, `Box<T>` (they own heap memory) |
| Shared references `&T` | Mutable references `&mut T` |
| Raw pointers, function pointers, `!`, `()` | Anything with a `Drop` implementation |
| Tuples, arrays, `Option<T>`, and your structs/enums **if all components are `Copy`** | Types containing any non-`Copy` field |

✓ I confirmed with a `fn is_copy<T: Copy>()` probe that `i32`, `&str`, `(i32, f64)`, `Option<u8>`, `fn(i32) -> i32`, `char`, and `&String` are `Copy`, and that `String`, `[String; 2]`, `&mut i32`, `Option<String>`, and `Vec<u8>` are not.

**Rules enforced by the compiler:**

```rust
#[derive(Clone, Copy)]
struct Named { id: u32, name: String }
```

✓ `error[E0204]: the trait `Copy` cannot be implemented for this type` — the field `name` is not `Copy`.

```rust
#[derive(Clone, Copy)]
struct Handle(u32);
impl Drop for Handle { fn drop(&mut self) {} }
```

✓ `error[E0184]: the trait `Copy` cannot be implemented for this type; the type has a destructor`. A type that releases a resource when dropped cannot be bitwise-duplicated, or the resource would be released twice.

```rust
#[derive(Copy)]
struct P(u32);
```

✓ `error[E0277]: the trait bound `P: Clone` is not satisfied`: `Clone` is a supertrait of `Copy`, so you derive both (`#[derive(Clone, Copy)]`).

**Design points.**

- `Copy` is **opt-in**. Even when every field is `Copy`, Rust does not make your struct `Copy` automatically: adding `Copy` is a promise in your public API, and you can't later remove it without breaking callers (they may rely on the old name staying valid). Derive it for small, plain-data types (points, ids, flags); don't for types that may grow non-`Copy` fields.
- Large `Copy` types are legal but copying them is a `memcpy` each time. ✓ A `struct Big { data: [u8; 4096] }` is 4096 bytes; passing it by value copies 4096 bytes (the optimizer may remove some copies), while `&Big` is 8 bytes. Pass big values by reference.
- `&T` is `Copy` (many readers are fine), `&mut T` is not (two writers are not). With `&mut`, passing it to a function **reborrows** implicitly rather than moving, which is why this works ✓ and prints `3`:

  ```rust
  fn bump(x: &mut i32) { *x += 1; }
  fn main() {
      let mut n = 0;
      let r = &mut n;
      bump(r);
      bump(r);
      let r2 = r;     // an explicit assignment moves r
      bump(r2);
      println!("{}", n);
  }
  ```

  while using `r` after `let r2 = r;` is ✓ `error[E0382]: use of moved value: `r``. Reborrowing is Lesson 10.
- `Copy` is a *language-level* trait: it changes how moves behave. You cannot implement it with custom logic; it always means "bitwise copy".

---

## 4. `Clone`: explicit, possibly expensive duplication

`Clone::clone(&self) -> Self` creates an independent duplicate and may run arbitrary code (allocate, copy, bump a counter). You write the call, so cost is visible in the source.

```rust
fn main() {
    let a = String::from("hello");
    let b = a.clone();
    println!("{} {}", a.as_ptr() == b.as_ptr(), a == b);
    let v = vec![String::from("x"), String::from("y")];
    let w = v.clone();
    println!("{}", v[0].as_ptr() == w[0].as_ptr());
    let mut c = String::from("tmp");
    c.clone_from(&a);
    println!("{c}");
}
```

✓ output:

```text
false true
false
hello
```

- A cloned `String` has its **own heap buffer** (pointers differ) but equal contents.
- Cloning a `Vec<String>` is **deep**: each element is cloned too (the first elements' pointers differ). Cost grows with the amount of data.
- `clone_from(&mut self, source: &Self)` can reuse the destination's existing allocation instead of dropping and reallocating; useful in loops.
- For `Copy` types, `clone()` is the same as a copy. Calling `.clone()` on a `Copy` value is redundant, and Clippy's `clone_on_copy` flags it (Lesson 02).

**`Rc::clone` / `Arc::clone` are different:** they bump a reference count instead of copying data (Lesson 23). `Clone` means "make an independent value", whose cost depends entirely on the type.

### 4.1 Cloning is not a fix for borrow-checker errors

The compiler's suggestions often say "consider cloning". It is sometimes right (small data, genuinely needing two copies, a one-off in non-hot code), but reflexively cloning to silence errors:

- hides design problems (who really owns this data?),
- costs allocations and time, especially inside loops,
- can introduce *semantic* bugs: after `let mut b = a.clone(); b.push('x');`, the change is not visible through `a`.

Ask first: can I borrow? can the function take a reference? can ownership be moved *into* the function and returned? is a different data structure (indices instead of references, `Rc`) the real answer?

For secrets (keys, tokens), every clone is another copy of the secret in memory that you must account for; avoid casual `Clone` on secret types (Lessons 48 and 52).

---

## 5. Closures and `move`

```rust
fn main() {
    let s = String::from("hi");
    let f = move || println!("{s}");
    f();
    println!("{s}");
}
```

✓ `error[E0382]: borrow of moved value: `s``. The `move` keyword makes the closure take ownership of captured variables; afterwards the original name is unusable. Without `move`, the closure would capture by reference where it can. This matters for threads (Lesson 27): a spawned thread may outlive the function, so it must own its captures (or have them proven to live long enough). Closures get a full treatment in Lesson 18.

---

## 6. Leaks are safe; moves are not optional

`std::mem::forget(value)` takes ownership and *never runs the destructor*:

```rust
struct Noisy(&'static str);
impl Drop for Noisy { fn drop(&mut self) { println!("drop {}", self.0); } }
fn main() {
    let a = Noisy("dropped");
    let b = Noisy("forgotten");
    std::mem::forget(b);
    let _ = &a;
    println!("end");
}
```

✓ output: `end`, `drop dropped` (no `drop forgotten`). Leaking memory is **memory-safe** (it is not UB), which is why `forget` is a safe function; Lesson 01 listed leaks among what safe Rust does not prevent. Consequence for API design: **never rely on `Drop` running for memory safety** of unsafe code (this is the lesson of the "leakpocalypse" that shaped scoped-thread APIs), and do not rely on `Drop` alone for security-critical cleanup either.

---

## 7. Size and performance

✓ On this machine:

```text
String 24   Vec<u8> 24   Box<[u8;4096]> 8   Big([u8;4096]) 4096   &Big 8
Option<String> 24   Option<Box<u8>> 8
```

- A move copies `size_of::<T>()` bytes in the worst case. For `String`/`Vec`/`Box` that is 24, 24, 8 bytes regardless of how much data they own: moving a million-element `Vec` is as cheap as moving an empty one.
- A move of an inline large value (a `[u8; 4096]` field, a big struct) really copies those bytes, unless the optimizer removes the copy. Heap-allocate large values you pass around (`Box`) or pass references.
- `Option<String>` is the same size as `String` ✓: `None` is encoded in a niche (a bit pattern that is never a valid `String`), another example of Lesson 01's niche optimization.
- Cloning allocates. In hot loops, hoist clones out of the loop, use `clone_from`, borrow, or restructure ownership.
- The optimizer routinely elides moves (constructing values directly in their final place). Don't contort code around imagined move costs; measure (Lesson 41).

---

## 8. Common mistakes and edge cases

1. Reaching for `.clone()` to satisfy the compiler without asking who should own the data.
2. Taking ownership (`fn f(s: String)`) in functions that only read: callers are forced to clone or give up their value. Prefer `&str` / `&T`.
3. `for x in v` when you still need `v` afterwards: use `for x in &v` or `v.iter()`.
4. Moving a field out of a struct and then using the whole struct (partial move).
5. Trying to move out of a vector by indexing (`v[0]`), out of a reference (`*r`), or out of a `&mut` without leaving something behind. Use `mem::take`/`replace`, `Option::take`, `swap_remove`, `remove`, or `drain`.
6. Deriving `Copy` on types that may later own heap data (an API-breaking removal).
7. Assuming moves are expensive (they're usually `memcpy` of a few words) or assuming moves never copy (large inline values do).
8. Expecting `Drop` to run (leaks, `mem::forget`, `Rc` cycles, process exit via `std::process::exit` or abort skip destructors).
9. Cloning a value and mutating the clone while expecting the original to change.
10. Forgetting that a `move` closure moves **all** captured variables, not just some.

## 9. Performance implications (summary)

Ownership checking is entirely compile-time; the runtime cost of the *discipline* is zero. Runtime costs come from the operations you choose: `clone` (allocation + copy), moving large inline values (memcpy), and drop flags for conditional moves (a hidden boolean and a branch). Moves of heap-owning handles are constant-size.

## 10. Security and reliability

- Single ownership gives **deterministic cleanup**: a file closes, a lock releases, a buffer frees at a known point. That determinism is what lets Rust avoid use-after-free and double-free in safe code (Lesson 01).
- "Move" is also a *capability* tool: a function taking `self` consumes the object, so the type system enforces "use at most once". Nonces, one-time tokens, and finalized builders become compile-time guarantees. The `Ticket` type in the practical challenge demonstrates it. Note that this protects against misuse **within your program**, not against the same token being replayed over a network.
- Don't treat ownership as a secrets-management system. Moves leave bytes behind in the old stack slot and freed heap memory is not zeroed automatically; clearing secrets needs explicit techniques (for example the `zeroize` crate, whose behavior I did not verify in this lesson) and care about copies the optimizer or allocator makes (Lesson 48).
- Leaks are safe but can still be denial-of-service: unbounded caches or detached tasks that hold values forever are reliability bugs (Lessons 25 and 32).

---

## 11. Exercises

Try each before opening the answer. They are not marked complete unless you tell me you did them.

1. **(Easy)** Without compiling, which line fails? Fix it minimally.
   ```rust
   let a = String::from("x");
   let b = a;
   let c = b.clone();
   println!("{a} {b} {c}");
   ```
2. **(Easy)** Which of these are `Copy`: `i32`, `&str`, `String`, `(i32, f64)`, `[String; 2]`, `&mut i32`, `Option<u8>`, `Option<String>`, `fn(i32) -> i32`, `char`? Write a tiny program that proves each answer to the compiler.
3. **(Medium)** Write `fn longest_owned(a: String, b: String) -> String` returning the longer one (`a` on ties) with no clones, and `fn append_all(dst: &mut String, parts: Vec<String>)` that consumes `parts`.
4. **(Medium)** Write `fn first_or_default(v: &mut Vec<String>) -> String`: remove and return the first element, or an empty `String` if there is none. No clones.
5. **(Medium)** Given `struct Pair { a: String, b: String }` and `let mut p = ...;`, swap the two fields in place without cloning and without `unsafe`.
6. **(Hard)** Explain why a method `fn peek(&self) -> Option<String>` on `struct Stack { items: Vec<String> }` cannot return `self.items.last()` directly without cloning, and give the better signature. Then write `fn into_sorted(self) -> Vec<String>`.
7. **(Hard)** For `let _a = N("a"); let _b = N("b"); let _c = N("c"); let _ = N("d");` (where `N` prints on drop), predict the drop order, including when `d` is dropped. Run it to check.

<details markdown="1">
<summary>Answers</summary>

1. ✓ The `println!` fails: `a` was moved into `b` (E0382, "borrow of moved value: `a`"). Minimal fix: drop `{a}` from the print, or clone before moving (`let b = a.clone();`) if you really need both.
2. ✓ Probe: `fn is_copy<T: Copy>() {}` called with each type. `Copy`: `i32`, `&str`, `(i32, f64)`, `Option<u8>`, `fn(i32) -> i32`, `char`. Not `Copy` (the call fails to compile): `String`, `[String; 2]`, `&mut i32`, `Option<String>`.
3. ✓ Tested.
   ```rust
   fn longest_owned(a: String, b: String) -> String {
       if b.len() > a.len() { b } else { a }
   }
   fn append_all(dst: &mut String, parts: Vec<String>) {
       for p in parts { dst.push_str(&p); }
   }
   ```
   `longest_owned("ab", "abc")` → `abc`; `append_all` on `">"` with `["x", "y"]` → `>xy`. The losing `String` in `longest_owned` is dropped when the function returns.
4. ✓ Tested.
   ```rust
   fn first_or_default(v: &mut Vec<String>) -> String {
       if v.is_empty() { String::new() } else { v.remove(0) }
   }
   ```
   (`remove(0)` is O(n) because it shifts the rest; for a queue use `VecDeque`, Lesson 16.)
5. ✓ Tested: `std::mem::swap(&mut p.a, &mut p.b);`. Borrowing two *different fields* mutably at once is accepted because the compiler tracks fields separately (disjoint paths). Two borrows of the same variable, like `&mut v[0]` and `&mut v[1]`, would not be (Lesson 10).
6. `self.items.last()` is `Option<&String>`, a borrow of data owned by the stack; returning an owned `String` would require moving it out of `&self` (E0507) or cloning. The better signature borrows: `fn peek(&self) -> Option<&String>` (or `Option<&str>`), tied to `&self`'s lifetime (Lesson 11). ✓ Tested together with:
   ```rust
   fn into_sorted(mut self) -> Vec<String> {
       self.items.sort();
       self.items
   }
   ```
   Taking `self` by value means the caller gives up the stack, and the vector is *moved* out, not copied.
7. ✓ Output: `drop d` first (a `let _ = expr;` drops the temporary immediately), then `end of main`, then at scope end in reverse declaration order: `drop c`, `drop b`, `drop a`.

</details>

---

## 12. Debugging challenge

This program is meant to run each job and report the total payload size. The compiler complains. Fix it **without** `clone()` and without changing `Job`.

```rust
#[derive(Debug)]
struct Job { id: u32, payload: String }

fn run(job: Job) -> usize { job.payload.len() }

fn main() {
    let jobs = vec![
        Job { id: 1, payload: "aa".into() },
        Job { id: 2, payload: "bbb".into() },
    ];
    let first = jobs[0];
    let n = run(first);
    let total: usize = jobs.iter().map(|j| run(j)).sum();
    println!("{n} {total}");
}
```

Explain (a) why the compiler reports only **one** error at first, and which error is hiding behind it, (b) what each error means in ownership terms, and (c) what `run`'s signature should be.

<details markdown="1">
<summary>Solution</summary>

(a) ✓ The first compile reports only `error[E0308]: mismatched types` at `run(j)` inside the closure (`j` is a `&Job`, `run` expects a `Job`). The ownership error is hidden because rustc checks types before it runs the **borrow checker**; if type checking fails, borrow checking does not run. ✓ When I removed the `total` line to get past type checking, the next error appeared: `error[E0507]: cannot move out of index of `Vec<Job>`` at `let first = jobs[0];`. This is the same phased reporting you saw with privacy errors in Lesson 03: fixing one error can reveal the next.

(b) E0308: `iter()` yields references; the function wants ownership. E0507: `jobs[0]` would move a `Job` out of the vector, leaving a hole that the vector's destructor would later drop again; indexing only gives you a place to *borrow*, not to take from.

(c) `run` only reads the payload, so it should borrow:

```rust
#[derive(Debug)]
struct Job { id: u32, payload: String }

fn run(job: &Job) -> usize { job.payload.len() }

fn main() {
    let jobs = vec![
        Job { id: 1, payload: "aa".into() },
        Job { id: 2, payload: "bbb".into() },
    ];
    let first = &jobs[0];
    let n = run(first);
    let total: usize = jobs.iter().map(|j| run(j)).sum();
    println!("{n} {total}");
}
```

✓ prints `2 5` (with a harmless warning that `id` is never read). If `run` genuinely needed to consume jobs, the call would be `jobs.into_iter().map(run).sum()` and `jobs` could not be used afterwards.

</details>

---

## 13. Practical challenge: `ownlab`

Create a library crate `ownlab` (edition 2024) with:

1. A `Buffer` type with a **consuming builder**: `Buffer::new().with(b"abc").with(b"de").finish()` returns `Vec<u8>`. No cloning anywhere.
2. `take_longest(v: &mut Vec<String>) -> Option<String>`: remove and return the longest string (first one on ties), `None` if empty.
3. `extract_long(v: &mut Vec<String>, min: usize) -> Vec<String>`: move every string with `len() >= min` out of `v`, keeping the relative order of both groups, without cloning (prove it in a test by comparing heap pointers).
4. A single-use `Ticket` whose `redeem(self)` consumes it. Document, with a `compile_fail` doc test, that redeeming twice does not compile.
5. A test that a moved value is dropped exactly once, and a test that `mem::forget` skips the drop (use a `Cell<u32>` counter; `Cell` is Lesson 24, treat it as "a counter you can bump through a shared reference" for now).
6. It must pass `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, and `cargo test --locked` (run `cargo generate-lockfile` once first).

<details markdown="1">
<summary>Solution</summary>

**`src/lib.rs`**

```rust
//! Ownership exercises: consuming builders, moving out of `&mut`,
//! single-use tokens, and drop accounting.

/// A byte buffer built by a consuming (by-value) builder chain.
#[derive(Debug, Default, PartialEq)]
pub struct Buffer {
    data: Vec<u8>,
}

impl Buffer {
    pub fn new() -> Self {
        Self::default()
    }

    /// Takes `self` by value and returns it, so calls chain without clones.
    #[must_use]
    pub fn with(mut self, bytes: &[u8]) -> Self {
        self.data.extend_from_slice(bytes);
        self
    }

    /// Consumes the buffer and hands back the bytes.
    pub fn finish(self) -> Vec<u8> {
        self.data
    }

    pub fn len(&self) -> usize {
        self.data.len()
    }

    pub fn is_empty(&self) -> bool {
        self.data.is_empty()
    }
}

/// Removes and returns the longest string (the first one if tied) without
/// cloning. Returns `None` when `v` is empty.
pub fn take_longest(v: &mut Vec<String>) -> Option<String> {
    let mut best: Option<usize> = None;
    for (i, s) in v.iter().enumerate() {
        match best {
            Some(b) if v[b].len() >= s.len() => {}
            _ => best = Some(i),
        }
    }
    best.map(|i| v.remove(i))
}

/// Moves every string with `len() >= min` out of `v` into the returned
/// vector, keeping the relative order of both groups. No string is cloned.
pub fn extract_long(v: &mut Vec<String>, min: usize) -> Vec<String> {
    let all = std::mem::take(v);
    let (long, short): (Vec<String>, Vec<String>) = all.into_iter().partition(|s| s.len() >= min);
    *v = short;
    long
}

/// A single-use ticket: redeeming consumes it, so a second redemption does
/// not compile.
///
/// ```
/// use ownlab::Ticket;
/// let t = Ticket::new(7);
/// assert_eq!(t.redeem(), 7);
/// ```
///
/// ```compile_fail
/// use ownlab::Ticket;
/// let t = Ticket::new(7);
/// let _first = t.redeem();
/// let _second = t.redeem(); // error[E0382]: use of moved value: `t`
/// ```
#[derive(Debug)]
pub struct Ticket(u32);

impl Ticket {
    pub fn new(id: u32) -> Self {
        Ticket(id)
    }

    pub fn redeem(self) -> u32 {
        self.0
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::cell::Cell;

    #[test]
    fn builder_chains_without_clones() {
        let bytes = Buffer::new().with(b"abc").with(b"de").finish();
        assert_eq!(bytes, b"abcde");
        let b = Buffer::new();
        assert!(b.is_empty());
        assert_eq!(b.with(b"x").len(), 1);
    }

    #[test]
    fn take_longest_cases() {
        let mut v: Vec<String> = vec![];
        assert_eq!(take_longest(&mut v), None);
        let mut v = vec!["aa".to_string(), "bbbb".to_string(), "cccc".to_string()];
        assert_eq!(take_longest(&mut v).as_deref(), Some("bbbb"));
        assert_eq!(v, ["aa", "cccc"]);
    }

    #[test]
    fn extract_long_moves_and_keeps_order() {
        let mut v: Vec<String> = ["a", "bbb", "cc", "dddd"]
            .iter()
            .map(|s| s.to_string())
            .collect();
        let ptr_before = v[1].as_ptr();
        let long = extract_long(&mut v, 3);
        assert_eq!(long, ["bbb", "dddd"]);
        assert_eq!(v, ["a", "cc"]);
        assert_eq!(long[0].as_ptr(), ptr_before, "moved, not cloned");
        let mut empty: Vec<String> = vec![];
        assert!(extract_long(&mut empty, 1).is_empty());
    }

    struct Probe<'a>(&'a Cell<u32>);
    impl Drop for Probe<'_> {
        fn drop(&mut self) {
            self.0.set(self.0.get() + 1);
        }
    }

    fn consume(_p: Probe<'_>) {}

    #[test]
    fn moved_value_is_dropped_exactly_once() {
        let drops = Cell::new(0);
        let p = Probe(&drops);
        consume(p);
        assert_eq!(drops.get(), 1);
    }

    #[test]
    fn forget_skips_the_drop() {
        let drops = Cell::new(0);
        std::mem::forget(Probe(&drops));
        assert_eq!(drops.get(), 0);
    }

    #[test]
    fn ticket_redeems_once() {
        assert_eq!(Ticket::new(3).redeem(), 3);
    }
}
```

✓ With this exact file: `cargo fmt --check` clean, `cargo clippy --all-targets --locked -- -D warnings` clean, `cargo test --locked` → 6 unit tests and 2 doc tests passed (one of them the `compile_fail` test).

**Design notes.**

- `Buffer::with(mut self, ...) -> Self` takes and returns ownership, so the builder chain moves one value along; the `#[must_use]` attribute makes it a warning to call `with` and discard the result (which would silently drop the buffer).
- `take_longest` first finds the *index* using shared borrows, then calls `remove(i)` (a move out of the vector that keeps it valid). Separating "find" from "take" avoids holding a borrow while mutating (Lesson 10).
- `extract_long` uses `mem::take(v)` to move the whole vector out and leave an empty one behind, partitions by value with `into_iter().partition`, then assigns the short group back with `*v = short`. Every `String` moves; none is cloned, which the test shows by checking that the first extracted string still has the same heap pointer.
- The `Ticket` doc tests use `compile_fail`: rustdoc *expects* that code to fail to compile, so the guarantee is checked by the test suite. (Be aware that the test passes if the code fails to compile for *any* reason, so a typo in the example would still "pass". rustdoc documents an optional error-code marker such as `compile_fail,E0382` for a more precise check; I did not verify how strictly stable rustdoc enforces it. Lesson 39 covers doc tests.)
- The `Probe` tests demonstrate drop accounting: after `consume(p)`, the counter is `1`; after `mem::forget`, it stays `0`.

</details>

---

## 14. Interview questions

<details markdown="1">
<summary>Answers</summary>

1. **State the ownership rules.** Each value has one owner; there is one owner at a time; the value is dropped when the owner goes out of scope. Assignment and function calls move ownership unless the type is `Copy`.
2. **What does a move actually do at runtime?** A shallow bitwise copy of the value's bytes to the new location (for `String`: pointer, capacity, length). The heap data is untouched; the compiler marks the source uninitialized and doesn't drop it.
3. **Why is `let b = a;` followed by using `a` an error for `String` but not for `i32`?** `String` is not `Copy`, so the assignment moves; `i32` is `Copy`, so it is duplicated and both remain valid.
4. **What is the difference between `Copy` and `Clone`?** `Copy` is a marker for types that can be duplicated by a bitwise copy; the compiler applies it implicitly and you can't customize it. `Clone` is an explicit method that can run arbitrary code (often allocating). `Copy` requires `Clone`.
5. **Can a type that implements `Drop` be `Copy`?** No (E0184): duplicating it bitwise would run the destructor twice for one resource.
6. **Why can't you move out of a vector by indexing?** It would leave an uninitialized hole that the vector's own drop code would later read (E0507). Use `remove`, `swap_remove`, `drain`, `into_iter`, `mem::take`, or borrow with `&v[i]`.
7. **How do you take a value out of a `&mut` reference?** Replace it: `mem::replace`, `mem::take`, `mem::swap`, or `Option::take`. The slot always stays valid.
8. **What are drop flags?** Hidden runtime booleans the compiler adds when a value is moved conditionally, so the destructor runs only if the value is still owned at scope end.
9. **Is it safe to leak memory in Rust? Why?** Yes: leaks cannot cause memory unsafety, so `mem::forget` is a safe function. Therefore unsafe code must not rely on destructors running.
10. **When does `for x in v` move `v`?** Always for an owned `Vec`: it calls `IntoIterator::into_iter(v)`. Use `&v` / `v.iter()` to borrow.
11. **Should a function take `String` or `&str` when it only reads the text?** `&str`: it accepts `String` (via `&`), literals, and slices, and doesn't force the caller to give up ownership or clone.
12. **Is moving a big `Vec` expensive?** No: 24 bytes are copied regardless of length. Moving a big *inline* array or struct copies all its bytes (unless optimized away).
13. **Why derive `Copy` cautiously on public types?** It is a promise: removing it later breaks code that relied on the original staying valid after use. It also makes accidental large copies easy if the type grows.
14. **What does the `move` keyword do on a closure?** The closure captures variables by value (moving or copying them in) instead of by reference.

</details>

---

## 15. Key takeaways

- Each value has one owner; when the owner leaves scope, the value is dropped. Assignment, argument passing, and returning **move** ownership unless the type is `Copy`.
- A move is a shallow bitwise copy; the old binding becomes unusable (E0382). Heap data is not copied. Moves in branches and loops are tracked on every path (with drop flags at runtime when needed).
- You can't move out of borrowed content, an index, or a `&mut` slot (E0507); use `mem::take`/`replace`/`swap`, `Option::take`, or the collection's removal methods.
- `Copy` (opt-in, bitwise, no destructor, all fields `Copy`) vs `Clone` (explicit, possibly expensive): prefer borrowing to cloning, and don't clone just to silence the compiler.
- Function signatures express ownership: `&T` read, `&mut T` modify, `T` consume.
- Leaks are safe; destructors may not run (forget, cycles, `exit`). Don't base memory safety or security-critical cleanup on `Drop` alone.
- The type checker runs before the borrow checker, so a second error may appear only after you fix the first.

**Connections.** Lesson 01 introduced ownership; Lesson 06's `a + &b` and matching a `String` out of an `Option` were moves. Next: Lesson 08 (stack and heap, `Drop`, RAII, allocation and deallocation) goes under the hood of what "dropped" means and the order of drops. Lesson 09 covers sizes, alignment and niches (`Option<String>` = 24 bytes). Lesson 10 introduces references and borrowing, the alternative to moving and cloning; Lesson 11 lifetimes; Lesson 18 closures and `move`; Lesson 23 `Box`/`Rc`/`Arc` (cheap clones and shared ownership); Lesson 27 threads (moving values into spawned threads). See the [roadmap](00_roadmap.md) and the [course index](index.md).

---

## 16. References

- *The Rust Programming Language*, ch. 4.1 "What Is Ownership?": <https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html>
- *The Rust Programming Language*, ch. 4.2 "References and Borrowing": <https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html>
- *The Rust Reference*, destructors (drop scopes and order): <https://doc.rust-lang.org/reference/destructors.html>
- *The Rust Reference*, the `Copy` trait and special types and traits: <https://doc.rust-lang.org/reference/special-types-and-traits.html>
- Rust standard library documentation, `Clone`: <https://doc.rust-lang.org/std/clone/trait.Clone.html>
- Rust standard library documentation, `Copy`: <https://doc.rust-lang.org/std/marker/trait.Copy.html>
- Rust standard library documentation, `std::mem` (`take`, `replace`, `swap`, `forget`): <https://doc.rust-lang.org/std/mem/index.html>
- Rust standard library documentation, `Option::take`: <https://doc.rust-lang.org/std/option/enum.Option.html#method.take>
- Rust error code index: E0382, E0507, E0204, E0184: <https://doc.rust-lang.org/error_codes/E0382.html>
- Rust Nomicon, "Leaking" (why destructors cannot be relied on for safety): <https://doc.rust-lang.org/nomicon/leaking.html>
- *Clippy lint list*, `clone_on_copy` and `redundant_clone`: <https://rust-lang.github.io/rust-clippy/master/index.html>

The URLs above are the standard documentation locations; I did not open every one during this session, so if a link has moved, search the docs site for the page title.
