---
title: "Lesson 21 — Trait objects, static vs dynamic dispatch, and dyn compatibility"
---

# Lesson 21 — Trait objects, static vs dynamic dispatch, and dyn compatibility

**Module 7 · Generics and traits** · Status: generated · Edition used in examples: **2024** · External dependencies: none

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux. As before, ✓ marks outputs, diagnostics and assembly I actually ran; unmarked statements are documented behavior I did not run. Everything is **stable Rust**.

Builds on [Lesson 18](18_closures.md) (`Box<dyn Fn>`, fn pointers), [Lesson 19](19_generics_and_bounds.md) (monomorphization, static vs dynamic dispatch, `?Sized`) and [Lesson 20](20_associated_types_coherence.md) (associated types, blanket impls, the orphan rule, `Box<dyn Trait>` and `&T` forwarding impls). Terminology note: the Rust reference and the compiler now say **"dyn compatible"**; older books and answers say **"object safe"**. They mean the same thing.

---

## 1. The problem: one collection, many types

Generics (Lesson 19) give you one type per instantiation: `Vec<Circle>` or `Vec<Rect>`, never both. Sometimes the set of types is open (plugins, handlers, rules loaded at start-up) or you want one compiled copy of a function for all of them. Rust's second tool for polymorphism is the **trait object**: a value of some unknown concrete type, accessed only through a trait, with the method looked up at run time.

```rust
use std::{mem::{size_of, size_of_val}, rc::Rc, sync::Arc};
trait Shape { fn area(&self) -> f64; fn name(&self) -> String { "shape".into() } }
struct Circle(f64);
struct Rect { w: f64, h: f64 }
struct Big([u64; 8]);
impl Shape for Circle { fn area(&self) -> f64 { 3.0 * self.0 * self.0 } fn name(&self) -> String { "circle".into() } }
impl Shape for Rect { fn area(&self) -> f64 { self.w * self.h } }
impl Shape for Big { fn area(&self) -> f64 { self.0.iter().sum::<u64>() as f64 } }
fn total(shapes: &[Box<dyn Shape>]) -> f64 { shapes.iter().map(|s| s.area()).sum() }
fn main() {
    let shapes: Vec<Box<dyn Shape>> =
        vec![Box::new(Circle(1.0)), Box::new(Rect { w: 2.0, h: 3.0 }), Box::new(Big([1; 8]))];
    for s in &shapes { println!("{} {} size_of_val={}", s.name(), s.area(), size_of_val(&**s)); }
    println!("{}", total(&shapes));
    println!("{} {} {} {} {}",
        size_of::<&Circle>(), size_of::<&dyn Shape>(), size_of::<Box<dyn Shape>>(),
        size_of::<Rc<dyn Shape>>(), size_of::<Arc<dyn Shape>>());
    println!("{} {}", size_of::<Option<Box<dyn Shape>>>(), size_of::<*const dyn Shape>());
    let r: Rc<dyn Shape> = Rc::new(Circle(2.0));
    println!("{}", r.area());
}
```

✓ Output:

```text
circle 3 size_of_val=8
shape 6 size_of_val=16
shape 8 size_of_val=64
17
8 16 16 16 16
16 16
12
```

Three unrelated types, one `Vec`. Everything that follows explains these lines.

---

## 2. What `dyn Trait` is

`dyn Shape` is a **dynamically sized type** (DST): its size depends on which concrete type is behind it, so the compiler cannot know it. That has two consequences, both visible as errors ✓:

```rust
trait Speak { fn speak(&self) -> String; }
struct Dog;
impl Speak for Dog { fn speak(&self) -> String { "woof".into() } }
fn main() { let d: dyn Speak = Dog; }
```

✓ `error[E0308]: mismatched types` and `error[E0277]: the size for values of type `dyn Speak` cannot be known at compilation time`. And passing one by value ✓:

```rust
trait Speak { fn speak(&self) -> String; }
fn by_value(s: dyn Speak) {}
fn main() {}
```

✓ `error[E0277]: the size for values of type `(dyn Speak + 'static)` cannot be known at compilation time`. A `dyn Trait` therefore always lives **behind a pointer**: `&dyn Trait`, `&mut dyn Trait`, `Box<dyn Trait>`, `Rc<dyn Trait>`, `Arc<dyn Trait>`, `*const dyn Trait`. This is also why `T: ?Sized` (Lesson 19 §2.2) is how you accept them in generic code.

Turning a `Box<Circle>` into a `Box<dyn Shape>` is an **unsizing coercion**: the compiler knows the concrete type at that spot and attaches the extra information a trait object needs. It happens implicitly at `let` bindings with a type annotation, function arguments, and `vec![...]` elements with an annotated element type (as above). The same works for `Rc`, `Arc` and references ✓ (`Rc<dyn Shape>` above).

### 2.1 Fat pointers

A pointer to a sized type is one machine word. A pointer to a `dyn` is **two words**: the data pointer and a pointer to a **vtable** ✓: `&Circle` is 8 bytes, while `&dyn Shape`, `Box<dyn Shape>`, `Rc<dyn Shape>`, `Arc<dyn Shape>` and `*const dyn Shape` are each 16 (on this 64-bit target). `Option<Box<dyn Shape>>` is also 16: the data pointer is never null, so `None` uses the null niche (Lesson 09). `size_of_val(&**s)` on the trait object reports the size of the **concrete** value behind it (8, 16 and 64 bytes above), which is how `Box<dyn Shape>` knows how much memory to free.

### 2.2 The vtable and what a call does

For each (concrete type, trait) pair that is coerced to a trait object, the compiler emits a **vtable**: a read-only table holding, roughly, a function to drop the value, its size and alignment, and one function pointer per trait method. The exact layout is not specified by the language and can change between compiler versions. A method call on a `dyn` loads the function pointer from the vtable and calls it indirectly. I looked at what this compiles to ✓:

```rust
trait Speak { fn speak(&self) -> u32; }
struct Dog; struct Cat;
impl Speak for Dog { fn speak(&self) -> u32 { 7 } }
impl Speak for Cat { fn speak(&self) -> u32 { 9 } }
#[inline(never)] fn via_dyn(s: &dyn Speak) -> u32 { s.speak() + 1 }
#[inline(never)] fn via_generic<S: Speak>(s: &S) -> u32 { s.speak() + 1 }
fn main() {
    let a = std::hint::black_box(&Dog as &dyn Speak);
    println!("{} {} {}", via_dyn(a), via_generic(&Dog), via_generic(&Cat));
}
```

✓ Output `8 8 10`. In the unoptimized assembly (`--emit asm`), `via_dyn` contains `callq *24(%rsi)`, an **indirect call** through a slot of the vtable that arrives in the second argument register (the slot at byte offset 24 is consistent with three header words before the first method, which matches the description above, but I did not inspect the vtable data itself). `via_generic` appears as **two separate functions**, one for `Cat` and one for `Dog` (Lesson 19 §6). In the `-O` build the generic copies were folded away entirely (no `via_generic` symbol), while `via_dyn` remained a function containing an indirect call.

What the call costs and what it prevents:

- an indirect call instead of a direct one (cheap on modern CPUs when predictable, but not free);
- **no inlining across the call**, so the optimizer cannot specialize the body for the concrete type or remove it;
- usually a heap allocation if you use `Box`, `Rc` or `Arc`;
- one compiled copy of the *caller* regardless of how many types go through it (smaller code than a generic function instantiated for each type).

I did not benchmark any of this; Lesson 41 shows how to measure it properly instead of assuming.

---

## 3. Dyn compatibility: which traits can be `dyn`

Not every trait can build a vtable. A trait is **dyn compatible** when every method can be called through a vtable without knowing `Self`. The rules, with the error the compiler gave me for each ✓ (all are `error[E0038]: the trait `X` is not dyn compatible`, followed by a note explaining which feature is responsible):

| Feature in the trait | Why it breaks | Compiler's reason ✓ |
|---|---|---|
| Method with its own type parameter: `fn g<T: Debug>(&self, t: T)` | one vtable slot cannot hold unboundedly many monomorphized copies | "method `g` has generic type parameters" |
| `impl Trait` in argument position: `fn log(&self, x: impl Display)` | it is a hidden type parameter | "method `log` has generic type parameters" |
| Method returning `Self`: `fn dup(&self) -> Self` | the caller of a `dyn` doesn't know `Self`'s size | "method `dup` references the `Self` type in its return type" |
| Associated function without a receiver: `fn make() -> u8` | nothing to look the vtable up from | "associated function `make` has no `self` parameter" |
| Associated constant: `const N: u8` | not part of a per-value vtable | "it contains associated const `N`" |
| `Sized` supertrait (such as `Clone`) | `dyn Trait` is not `Sized` | "the trait is not dyn compatible because it requires `Self: Sized`" |
| `async fn` in the trait | its return type is a hidden `impl Future` | "method `go` is `async`" |
| `-> impl Trait` return | the same: a hidden concrete type | "method `items` references an `impl Trait` type in its return type" |

Example ✓:

```rust
trait Gen { fn g<T: std::fmt::Debug>(&self, t: T); }
struct S;
impl Gen for S { fn g<T: std::fmt::Debug>(&self, t: T) { println!("{t:?}"); } }
fn main() { let _x: Box<dyn Gen> = Box::new(S); }
```

✓ `error[E0038]: the trait `Gen` is not dyn compatible`. The error appears where you *use* `dyn Gen`, not where you define the trait; defining a non-dyn-compatible trait is perfectly fine and common (`Clone`, `Default`, `Iterator`'s `Self: Sized` methods).

A few more examples of the table:

```rust
trait Dup { fn dup(&self) -> Self; }
struct S;
impl Dup for S { fn dup(&self) -> S { S } }
fn main() { let _x: &dyn Dup = &S; }
```

✓ E0038 (returns `Self`). `fn main() { let _v: Vec<Box<dyn Clone>> = Vec::new(); }` is ✓ E0038 too ("requires `Self: Sized`"). An associated function (`fn make() -> u8`), an associated const (`const N: u8`) and an `impl Display` argument each gave E0038 as listed in the table (the three programs are the same shape as the ones above; I ran each).

### 3.1 `where Self: Sized`: opting individual methods out

You can keep a trait dyn compatible and still have generic or by-value methods by marking those methods `where Self: Sized`. They are then simply **not available on `dyn Trait`** ✓:

```rust
use std::fmt::Debug;
trait Tr {
    fn name(&self) -> String;
    fn generic<T: Debug>(&self, t: T) -> String where Self: Sized { format!("{} {:?}", self.name(), t) }
    fn consume(self) -> String where Self: Sized { self.name() }
    fn make() -> Self where Self: Sized;
}
struct S;
impl Tr for S { fn name(&self) -> String { "S".into() } fn make() -> S { S } }
fn use_dyn(t: &dyn Tr) -> String { t.name() }
fn main() {
    let s = S::make();
    println!("{} {} {}", use_dyn(&s), s.generic(5), s.consume());
}
```

✓ Output `S S 5 S`. This is exactly how `Iterator` works: `next` is callable on `dyn Iterator`, while adapters like `map` and `filter` (which take `self` and are generic) carry `where Self: Sized` ✓:

```rust
fn main() {
    let it: Box<dyn Iterator<Item = u32>> = Box::new(1..4);
    let v: Vec<u32> = it.map(|x| x * 2).filter(|x| *x > 2).collect();
    println!("{v:?}");
}
```

✓ Output `[4, 6]`. (`Box<dyn Iterator>` can use the adapters because `Box<I>` itself implements `Iterator` for `I: Iterator + ?Sized`: the forwarding-impl pattern from Lesson 20 §6. Note the iterator must pin its associated type: `dyn Iterator<Item = u32>`, Lesson 20 §2.3.) Trying to call a `Self: Sized` method on a trait object is an error ✓:

```rust
trait Tr { fn name(&self) -> String; fn consume(self) -> String where Self: Sized { self.name() } }
struct S;
impl Tr for S { fn name(&self) -> String { "S".into() } }
fn main() { let b: Box<dyn Tr> = Box::new(S); println!("{}", b.consume()); }
```

✓ `error: the `consume` method cannot be invoked on a trait object`, pointing at "this has a `Sized` requirement".

Other receivers are allowed: a method may take `self: Box<Self>` (also `Rc<Self>`, `Arc<Self>`, `Pin<..>`), which is how you write a consuming method on a boxed trait object, such as state machines ✓:

```rust
trait State {
    fn next(self: Box<Self>) -> Box<dyn State>;
    fn name(&self) -> &'static str;
}
struct A; struct B;
impl State for A { fn next(self: Box<Self>) -> Box<dyn State> { Box::new(B) } fn name(&self) -> &'static str { "A" } }
impl State for B { fn next(self: Box<Self>) -> Box<dyn State> { Box::new(A) } fn name(&self) -> &'static str { "B" } }
fn main() {
    let mut s: Box<dyn State> = Box::new(A);
    for _ in 0..3 { print!("{} ", s.name()); s = s.next(); }
    println!("{}", s.name());
}
```

✓ Output `A B A B`. (Compare with the typestate approach of Lesson 15, which checks transitions at compile time; this one trades that for run-time flexibility.)

### 3.2 Making `Clone`-like behavior work: `clone_box`

`Clone` requires `Sized`, so `Box<dyn Shape>` can't be cloned by derive. The standard workaround adds a dyn-compatible method that returns a boxed clone, via a helper supertrait with a blanket impl, then implements `Clone` for the box ✓:

```rust
trait Shape: ShapeClone { fn area(&self) -> f64; }
trait ShapeClone { fn clone_box(&self) -> Box<dyn Shape>; }
impl<T: Shape + Clone + 'static> ShapeClone for T {
    fn clone_box(&self) -> Box<dyn Shape> { Box::new(self.clone()) }
}
impl Clone for Box<dyn Shape> {
    fn clone(&self) -> Self { self.clone_box() }
}
#[derive(Clone)]
struct Sq(f64);
impl Shape for Sq { fn area(&self) -> f64 { self.0 * self.0 } }
fn main() {
    let a: Vec<Box<dyn Shape>> = vec![Box::new(Sq(2.0))];
    let b = a.clone();
    println!("{} {}", a[0].area(), b[0].area());
}
```

✓ Output `4 4`. Look at the pieces with Lesson 20 in mind: the blanket impl serves every future `Shape + Clone` type; `impl Clone for Box<dyn Shape>` is allowed by the orphan rule because `Box` is fundamental and `dyn Shape` is a local trait object, so `Box<dyn Shape>` counts as local. (The `dyn-clone` crate packages this pattern; I'm naming it from memory.)

---

## 4. Supertraits, upcasting and `Any`

A trait object of a sub-trait can call the supertrait's methods, and since a recent release it can be **upcast** to a supertrait object ✓:

```rust
trait Animal { fn name(&self) -> String; }
trait Dog: Animal { fn bark(&self) -> String; }
struct D;
impl Animal for D { fn name(&self) -> String { "d".into() } }
impl Dog for D { fn bark(&self) -> String { "woof".into() } }
fn describe(a: &dyn Animal) -> String { a.name() }
fn main() {
    let d: &dyn Dog = &D;
    println!("{} {}", d.bark(), d.name());
    println!("{}", describe(d));
    let b: Box<dyn Dog> = Box::new(D);
    let a: Box<dyn Animal> = b;
    println!("{}", a.name());
}
```

✓ Output `woof d`, `d`, `d`. The upcasts `&dyn Dog` → `&dyn Animal` and `Box<dyn Dog>` → `Box<dyn Animal>` compile on 1.97. (I believe trait upcasting was stabilized around Rust 1.86; before that you needed an `as_animal` helper method. That version number is from memory.)

### 4.1 `dyn Any` and downcasting

`std::any::Any` is implemented for every `'static` type and lets you ask a trait object "what are you really?" ✓:

```rust
use std::any::Any;
struct A(u8);
struct B;
fn main() {
    let items: Vec<Box<dyn Any>> = vec![Box::new(A(7)), Box::new(B), Box::new(5i32)];
    for it in &items {
        if let Some(a) = it.downcast_ref::<A>() { println!("A({})", a.0); }
        else if it.is::<B>() { println!("B"); }
        else if let Some(n) = it.downcast_ref::<i32>() { println!("i32 {n}"); }
    }
    let b: Box<dyn Any> = Box::new(String::from("s"));
    match b.downcast::<i32>() {
        Ok(n) => println!("{n}"),
        Err(back) => println!("not i32, still {:?}", back.downcast_ref::<String>()),
    }
}
```

✓ Output `A(7)`, `B`, `i32 5`, `not i32, still Some("s")`. `Box<dyn Any>::downcast` returns the box back on failure, so nothing is lost. Use `Any` sparingly: it moves a type check from compile time to run time, which is the opposite of what the rest of this course tries to do. Reasonable uses are plugin registries and error types that want to expose a concrete cause (Lesson 26).

Two traps ✓:

```rust
use std::any::Any;
fn find<R: Any>(items: &[Box<dyn Any>]) -> Option<&R> {
    items.iter().find_map(|i| i.downcast_ref::<R>())
}
fn find2<R>(items: &[Box<dyn Any>]) -> Option<&R> {
    items.iter().find_map(|i| i.downcast_ref::<R>())
}
fn main() {
    let v: Vec<Box<dyn Any>> = vec![Box::new(5u8)];
    println!("{:?}", find::<u8>(&v));
    println!("{:?}", find2::<u8>(&v));
}
```

✓ `error[E0310]: the parameter type `R` may not live long enough` for `find2` only: downcasting needs `R: 'static` (which `R: Any` implies). Second trap, a box inside a box ✓:

```rust
use std::any::Any;
fn main() {
    let b: Box<dyn Any> = Box::new(5u8);
    let wrapped: Box<dyn Any> = Box::new(b);
    println!("{} {}", wrapped.is::<u8>(), wrapped.is::<Box<dyn Any>>());
}
```

✓ Output `false true`: the thing inside `wrapped` *is* a `Box<dyn Any>`, so the `u8` check fails. The same shape bites when you add an `as_any(&self) -> &dyn Any` method through a blanket impl, as the practical challenge does: calling it on a `&Box<dyn Rule>` describes the **box**, not the rule inside (✓ test `as_any_on_the_box_describes_the_box_not_the_rule`). Dereference first: `(**boxed).as_any()`.

---

## 5. Lifetimes and thread-safety of trait objects

**Default object lifetime.** `Box<dyn Speak>` in a type or signature means `Box<dyn Speak + 'static>`: the concrete type may not borrow anything. A borrowed implementor is rejected ✓:

```rust
trait Speak { fn speak(&self) -> String; }
struct Dog;
impl Speak for Dog { fn speak(&self) -> String { "woof".into() } }
struct Holder { s: Box<dyn Speak> }
fn make<'a>(d: &'a Dog) -> Holder { Holder { s: Box::new(Wrap(d)) } }
struct Wrap<'a>(&'a Dog);
impl Speak for Wrap<'_> { fn speak(&self) -> String { self.0.speak() } }
fn main() { let d = Dog; println!("{}", make(&d).s.speak()); }
```

✓ `error: lifetime may not live long enough` ("coercion requires that `'a` must outlive `'static`"). The fix is to give the trait object (and the holder) a lifetime ✓:

```rust
trait Speak { fn speak(&self) -> String; }
struct Dog;
impl Speak for Dog { fn speak(&self) -> String { "woof".into() } }
struct Wrap<'a>(&'a Dog);
impl Speak for Wrap<'_> { fn speak(&self) -> String { self.0.speak() } }
struct Holder<'a> { s: Box<dyn Speak + 'a> }
fn make<'a>(d: &'a Dog) -> Holder<'a> { Holder { s: Box::new(Wrap(d)) } }
fn main() { let d = Dog; println!("{}", make(&d).s.speak()); }
```

✓ Output `woof`. (For plain `&'a dyn Trait` the default is `+ 'a`, so references rarely need this spelled out. Lesson 12 covers lifetimes in general.)

**Auto traits are not implied.** A `dyn Speak` is not `Send` or `Sync` unless you say so ✓:

```rust
trait Speak { fn speak(&self) -> String; }
struct Dog;
impl Speak for Dog { fn speak(&self) -> String { "woof".into() } }
fn main() {
    let s: Box<dyn Speak> = Box::new(Dog);
    let h = std::thread::spawn(move || s.speak());
    println!("{}", h.join().unwrap());
}
```

✓ `error[E0277]: `dyn Speak` cannot be sent between threads safely`. Write `Box<dyn Speak + Send + Sync>` where you need it ✓ (`Rc<dyn Speak>` with `Rc::clone` printed `woof 2`, and a `Box<dyn Speak + Send + Sync>` moved into `thread::spawn` printed `woof`), or make `Send + Sync` supertraits of your trait so every implementor must be thread-safe (the practical challenge does that). Lesson 22 covers `Send` and `Sync` in depth and Lesson 27 uses them with threads.

---

## 6. Choosing: generics, `dyn`, or an enum

```rust
// closed set: enum; open set: dyn
#[derive(Debug)] enum Shape { Circle(f64), Rect(f64, f64) }
impl Shape { fn area(&self) -> f64 { match self { Shape::Circle(r) => 3.0 * r * r, Shape::Rect(w, h) => w * h } } }
trait Area { fn area(&self) -> f64; }
struct Circle(f64); struct Rect(f64, f64);
impl Area for Circle { fn area(&self) -> f64 { 3.0 * self.0 * self.0 } }
impl Area for Rect { fn area(&self) -> f64 { self.0 * self.1 } }
fn main() {
    let e = [Shape::Circle(1.0), Shape::Rect(2.0, 3.0)];
    let d: [Box<dyn Area>; 2] = [Box::new(Circle(1.0)), Box::new(Rect(2.0, 3.0))];
    println!("{} {}", e.iter().map(Shape::area).sum::<f64>(), d.iter().map(|s| s.area()).sum::<f64>());
    println!("{} {}", std::mem::size_of::<Shape>(), std::mem::size_of::<Box<dyn Area>>());
}
```

✓ Output `9 9` and `24 16`. Same answer; the enum stores values inline (24 bytes: the largest variant plus a tag, Lesson 09) with no allocation and no indirect call, but you must edit the enum to add a type. The trait object is 16 bytes of pointer plus a heap allocation per value, and anyone can add a type.

| | Generic `T: Trait` | `dyn Trait` | `enum` |
|---|---|---|---|
| Set of types | open, known at compile time | open, decided at run time | closed, you edit the enum |
| Mixed in one collection | no | yes | yes |
| Dispatch | static, inlinable | indirect call | `match` (a branch) |
| Code size | grows per type | one copy | one copy |
| Memory | by value | behind a pointer (usually heap) | inline, size of the largest variant |
| Adding a type | in any crate | in any crate | edit the enum (breaking for exhaustive matches) |
| Needs dyn compatibility | no | yes | no |

Returning one of several types also forces the choice. `impl Trait` in return position names **one** hidden type, so two branches with different types fail ✓:

```rust
trait Speak { fn speak(&self) -> String; }
struct Dog; struct Cat;
impl Speak for Dog { fn speak(&self) -> String { "woof".into() } }
impl Speak for Cat { fn speak(&self) -> String { "meow".into() } }
fn pick(dog: bool) -> impl Speak { if dog { Dog } else { Cat } }
fn main() { println!("{}", pick(true).speak()); }
```

✓ `error[E0308]: `if` and `else` have incompatible types`. With a trait object ✓:

```rust
trait Speak { fn speak(&self) -> String; }
struct Dog; struct Cat;
impl Speak for Dog { fn speak(&self) -> String { "woof".into() } }
impl Speak for Cat { fn speak(&self) -> String { "meow".into() } }
fn pick(dog: bool) -> Box<dyn Speak> { if dog { Box::new(Dog) } else { Box::new(Cat) } }
fn main() { println!("{} {}", pick(true).speak(), pick(false).speak()); }
```

✓ Output `woof meow`. (A small enum implementing `Speak` is the third option and avoids the allocation.) Practical guidance: start generic; reach for `dyn` when you need heterogeneous storage, plugin-style extension, to keep a hot generic function from exploding into many copies, or to hide a type in an API; prefer an enum when you control the full set.

---

## 7. Common mistakes and edge cases

- **Defining a trait with a generic method or `-> Self`, then using `dyn`** (E0038 ✓). Mark the offenders `where Self: Sized`, or split them into a second trait.
- **Putting `Clone` (or `Sized`) as a supertrait** and then boxing it (E0038 ✓). Use the `clone_box` pattern.
- **Forgetting `?Sized`** on a generic helper that should accept `&dyn Trait` (E0277 "size cannot be known" ✓). Write `fn describe<P: Plugin + ?Sized>(p: &P)`.
- **Assuming `Send`/`Sync`** (E0277 ✓) or `'static` (E0597/"may not live long enough" ✓) for a trait object.
- **Calling a `Self: Sized` method on a `dyn`** ("cannot be invoked on a trait object" ✓).
- **`as_any` through a blanket impl on a `Box`** (describes the box ✓), and **`Box<dyn Any>` in a `Box<dyn Any>`** (✓).
- **Downcasting generics without `'static`** (E0310 ✓).
- **Comparing or hashing `dyn Trait` values**: `PartialEq`, `Hash` and `Ord` are generic over `Self` or `Rhs`, so they are not dyn compatible in a useful way; add a method like `fn eq_dyn(&self, other: &dyn Any) -> bool` or a stable `fn id(&self) -> u64` instead.
- **Associated types** on a dyn trait must be pinned (`dyn Iterator<Item = u32>`, E0191, Lesson 20).
- **Overusing `Box<dyn Trait>` in hot loops** when a generic or an enum would do (measure first).

---

## 8. Performance implications

- A trait object call is an indirect call and blocks inlining (✓ the `callq *24(%rsi)` above). Whether it matters depends on how hot the call is and how predictable the target is; I have not measured it here (Lesson 41).
- `Box<dyn Trait>` allocates once per value and frees on drop; a `Vec<Box<dyn Trait>>` is a vector of pointers to separate heap blocks, with the cache behavior that implies (Lesson 16). An enum, or a `Vec` per concrete type, keeps data contiguous.
- Generic code is monomorphized once per type (more code); `dyn` code is compiled once. A big generic function used with many types can be turned into a `&dyn Trait` function on a cold path to shrink the binary (Lesson 19 §6).
- Passing `&dyn Trait` costs nothing to create (a pointer plus a static vtable pointer); passing `Box<dyn Trait>` moves a 16-byte value.
- `Option<Box<dyn Trait>>` is the same size as `Box<dyn Trait>` ✓ (16).
- Fat pointers can't be stored in a single atomic word; a lock-free structure that holds a `dyn` needs another approach (Lesson 28).

---

## 9. Security and reliability

- **A trait object is not a sandbox.** Plugin code behind `dyn Rule` runs with all your process's privileges. It may panic, loop forever, allocate without limit or read your files. Rust's guarantees stop memory-unsafe behavior in safe code; they do not stop misbehaving logic. Isolate untrusted plugins in a separate process or a Wasm runtime (Lesson 47), not behind a trait.
- **Panics cross `dyn` calls like any call.** If plugin code panics, the caller unwinds (or aborts, depending on the panic strategy). Decide up front how a failing rule affects the system; do not rely on `catch_unwind` as a general error mechanism (Lesson 26 discusses error handling around panics).
- **Bound what plugins can make you do.** In the practical challenge `check_all` caps the number of reported violations (✓ test) so a hostile or buggy rule set cannot make an error report grow without limit; the cap is part of the type's configuration.
- **Make thread-safety part of the contract.** `trait Rule: Send + Sync` means every implementor must be safe to share, and the compiler checks it (✓ `assert_send_sync::<RuleSet>()` and a 4-thread test). Leaving it off makes `RuleSet` unusable across threads, which is a safe failure.
- **Downcasting is a type-confusion check, not a security check.** `downcast_ref::<T>()` is safe and correct, but code that branches on concrete types can silently skip a new implementor. Prefer adding a method to the trait.
- **Never choose a `dyn` implementation from untrusted input by type name** (reflection-style registries). Map a validated tag to a constructor from a fixed table.
- **Dyn compatibility is part of your public API.** Adding a generic method (without `where Self: Sized`), a `Self`-returning method or an associated const to a published trait breaks every `dyn` user (E0038), a semver hazard (Lesson 33).
- **Equality and ordering over trait objects** must be designed explicitly (see mistakes); an inconsistent `Ord` breaks sorted collections without memory unsafety (Lesson 16).

---

## 10. Exercises

Try each before opening the answer. They are not marked complete unless you tell me you did them.

1. **(Easy)** Make a `Vec<Box<dyn Shape>>` holding a circle, a rectangle and a 64-byte struct, and sum their areas.
2. **(Easy)** Predict `size_of` for `&Circle`, `&dyn Shape`, `Box<dyn Shape>` and `Option<Box<dyn Shape>>` on a 64-bit target, and say why `size_of_val` of the trait object behind a `Box` varies.
3. **(Medium)** For each of the following traits, say whether `&dyn Trait` compiles and why: `fn g<T: Debug>(&self, t: T)`; `fn dup(&self) -> Self`; `fn make() -> u8`; `const N: u8`. How can the trait keep the offending item and stay usable as `dyn`?
4. **(Medium)** Make `Vec<Box<dyn Shape>>` cloneable.
5. **(Medium)** `thread::spawn(move || s.speak())` with `s: Box<dyn Speak>` fails. Why, and what are two ways to fix it?
6. **(Medium)** `fn pick(dog: bool) -> impl Speak { if dog { Dog } else { Cat } }` fails. Why, and what is the smallest fix that keeps the call syntax?
7. **(Hard)** A struct must hold a trait object that borrows from a local value. Give the signature that compiles, and the error you get without the lifetime.
8. **(Hard)** Given `Vec<Box<dyn Any>>`, write a function that finds the first element of type `R` and explain the error for a version without `R: Any`.

<details markdown="1">
<summary>Answers</summary>

1. ✓ (run: output `circle 3 size_of_val=8`, `shape 6 size_of_val=16`, `shape 8 size_of_val=64`, and the total `17`): see the program in Section 1. The key lines are `let shapes: Vec<Box<dyn Shape>> = vec![Box::new(Circle(1.0)), Box::new(Rect { w: 2.0, h: 3.0 }), Box::new(Big([1; 8]))];` and `fn total(shapes: &[Box<dyn Shape>]) -> f64 { shapes.iter().map(|s| s.area()).sum() }`. The annotation on the `let` is what triggers the unsizing coercion for each element.
2. ✓ `8 16 16 16`. A reference to a sized type is one pointer; `&dyn` and `Box<dyn>` are fat pointers (data pointer plus vtable pointer). `Option<Box<dyn Shape>>` stays 16 because the null niche encodes `None`. `size_of_val(&**s)` reports the size of the concrete type behind the pointer (8, 16 and 64 bytes in the example), which the vtable records.
3. ✓ None compile as `dyn`: all give E0038. (a) generic method: "has generic type parameters"; (b) returns `Self`: "references the `Self` type in its return type"; (c) no receiver: "has no `self` parameter"; (d) "contains associated const `N`". For (a), (b) and (c) add `where Self: Sized` to the method (✓ the trait then compiled as `&dyn Tr`, with those methods unavailable on the trait object, and `S::make()`/`s.generic(5)`/`s.consume()` still worked on the concrete type). An associated const cannot be opted out of this way; move it to a separate trait or expose it through a method.
4. ✓ (run: `4 4`)
   ```rust
   trait Shape: ShapeClone { fn area(&self) -> f64; }
   trait ShapeClone { fn clone_box(&self) -> Box<dyn Shape>; }
   impl<T: Shape + Clone + 'static> ShapeClone for T {
       fn clone_box(&self) -> Box<dyn Shape> { Box::new(self.clone()) }
   }
   impl Clone for Box<dyn Shape> {
       fn clone(&self) -> Self { self.clone_box() }
   }
   #[derive(Clone)]
   struct Sq(f64);
   impl Shape for Sq { fn area(&self) -> f64 { self.0 * self.0 } }
   fn main() {
       let a: Vec<Box<dyn Shape>> = vec![Box::new(Sq(2.0))];
       let b = a.clone();
       println!("{} {}", a[0].area(), b[0].area());
   }
   ```
   `Vec<T>: Clone` needs `T: Clone`, and `Box<dyn Shape>: Clone` now holds.
5. ✓ `error[E0277]: `dyn Speak` cannot be sent between threads safely`: a bare `dyn Speak` makes no promise about `Send`. Fix 1: annotate the box, `Box<dyn Speak + Send + Sync>` (✓ it compiled and the thread printed `woof`; `Send` alone suffices to move it). Fix 2: make it a supertrait, `trait Speak: Send + Sync`, so every implementor must be thread-safe (✓ the challenge's `Plugin: ... + Send + Sync` worked this way in the debugging challenge).
6. ✓ `error[E0308]: `if` and `else` have incompatible types`: `impl Speak` stands for **one** hidden concrete type, but the branches return `Dog` and `Cat`. Smallest fix: return `Box<dyn Speak>` and box each branch (✓ printed `woof meow`; call syntax `pick(true).speak()` is unchanged). Alternatives: an enum `Pet { Dog, Cat }` implementing `Speak` (no allocation, closed set), or two functions.
7. ✓ `struct Holder<'a> { s: Box<dyn Speak + 'a> }` with `fn make<'a>(d: &'a Dog) -> Holder<'a>`. Without it, `Holder { s: Box<dyn Speak> }` means `+ 'static`, and the compiler says `error: lifetime may not live long enough` ("coercion requires that `'a` must outlive `'static`"). Both versions were run.
8. ✓ `fn find<R: Any>(items: &[Box<dyn Any>]) -> Option<&R> { items.iter().find_map(|i| i.downcast_ref::<R>()) }` returned `Some(5)` for `find::<u8>` on a vector holding `5u8`. Without the bound: `error[E0310]: the parameter type `R` may not live long enough`, because `Any` is only implemented for `'static` types and `downcast_ref` needs that; `R: Any` implies `R: 'static`.

</details>

---

## 11. Debugging challenge

A teammate starts a plugin system:

```rust
trait Plugin: Clone { fn name(&self) -> String; }
#[derive(Clone)]
struct Echo;
impl Plugin for Echo { fn name(&self) -> String { "echo".into() } }
fn main() {
    let v: Vec<Box<dyn Plugin>> = vec![Box::new(Echo)];
    println!("{}", v.len());
}
```

Fix it so plugins can be stored as trait objects **and** cloned. Then they add `fn describe<P: Plugin>(p: &P) -> String { p.name() }` and call it with `&*boxed`, which fails. Fix that. Finally they move the boxed plugin into `std::thread::spawn` and that fails too. Explain each error and fix all three.

<details markdown="1">
<summary>Solution</summary>

**Stage 1.** ✓ `error[E0038]: the trait `Plugin` is not dyn compatible`. `Clone` has `Sized` as a supertrait (and `clone` returns `Self`), so any trait that extends it requires `Self: Sized`, which a `dyn Plugin` can't satisfy. Remove `Clone` from the supertraits and use the `clone_box` pattern from §3.2:

```rust
trait Plugin: PluginClone { fn name(&self) -> String; }
trait PluginClone { fn clone_box(&self) -> Box<dyn Plugin>; }
impl<T: Plugin + Clone + 'static> PluginClone for T {
    fn clone_box(&self) -> Box<dyn Plugin> { Box::new(self.clone()) }
}
impl Clone for Box<dyn Plugin> {
    fn clone(&self) -> Self { self.clone_box() }
}
```

**Stage 2.** ✓ With `fn describe<P: Plugin>(p: &P) -> String` and the call `describe(&*b)` (where `b: Box<dyn Plugin>`): `error[E0277]: the size for values of type `dyn Plugin` cannot be known at compilation time`. `&*b` is a `&dyn Plugin`, so `P` would be `dyn Plugin`, but every type parameter is implicitly `Sized`. Relax it (Lesson 19 §2.2): `fn describe<P: Plugin + ?Sized>(p: &P) -> String { p.name() }`.

**Stage 3.** ✓ `std::thread::spawn(move || b.name())` fails with `error[E0277]: `dyn Plugin` cannot be sent between threads safely`: a `dyn Plugin` promises nothing about `Send`. Make the guarantee part of the trait, so no implementor can forget it:

```rust
trait Plugin: PluginClone + Send + Sync { fn name(&self) -> String; }
```

(or write `Box<dyn Plugin + Send>` at the use site if only some callers need it). The final version compiled and printed `echo echo` (from `describe(&*b)` and `b.clone().name()`) and `echo` from the spawned thread ✓.

Lesson: dyn compatibility, `?Sized`, and auto traits (`Send`/`Sync`) are the three things to check whenever a generic design meets `dyn`.

</details>

---

## 12. Practical challenge: `dynlab`

Create a library crate `dynlab` (edition 2024) for pluggable transaction rules:

1. `Tx { from, to, amount, fee }` and `Violation { rule, reason }` (with `Display`).
2. A dyn-compatible trait `Rule: Send + Sync` with `name(&self) -> &str` and `check(&self, &Tx) -> Result<(), String>`, plus helper supertraits so that `Box<dyn Rule>` can be **cloned** (`clone_box`) and a rule can be **downcast** (`as_any`).
3. Rules: `MaxAmount(u64)`, `MinFee(u64)`, `NoSelfTransfer`, `Blocklist(Vec<u32>)`, and a closure-based `FnRule<F>`.
4. `RuleSet` holding `Vec<Box<dyn Rule>>` with a builder (`with`), `Clone`, `check_fast` (first failure), `check_all` (all failures, capped by a configurable maximum, default 16) and `find::<R>()` returning the first rule of concrete type `R`.
5. Two free functions `count_passing_generic<R: Rule>` and `count_passing_dyn(&dyn Rule)` that must agree.
6. Tests: dyn compatibility and `Send + Sync` assertions; evaluation order and error text; the violation cap (including a cap of zero); the empty set; blocklist and closure rules; clone independence; `find` downcasting; the `Box<dyn Rule>` `as_any` trap; static and dynamic dispatch agree; sharing a `RuleSet` across four threads. A usage doctest and a `compile_fail` doctest for a non-dyn-compatible trait.
7. It must pass `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, and `cargo test --locked` (run `cargo generate-lockfile` once first).

<details markdown="1">
<summary>Solution</summary>

**`src/lib.rs`**

```rust
//! `dynlab`: pluggable transaction rules as trait objects.
//!
//! A [`Rule`] is dyn compatible, so a [`RuleSet`] can hold rules of different
//! types in one `Vec<Box<dyn Rule>>`, clone them, and find a concrete rule
//! again with a downcast.
//!
//! ```
//! use dynlab::{MaxAmount, NoSelfTransfer, RuleSet, Tx};
//!
//! let rules = RuleSet::new().with(MaxAmount(100)).with(NoSelfTransfer);
//! assert!(rules.check_fast(&Tx { from: 1, to: 2, amount: 50, fee: 1 }).is_ok());
//! let err = rules.check_all(&Tx { from: 1, to: 1, amount: 500, fee: 1 }).unwrap_err();
//! assert_eq!(err.len(), 2);
//! ```
//!
//! A trait with a generic method is not dyn compatible, so it cannot be used
//! as `dyn`:
//!
//! ```compile_fail
//! trait Visit { fn visit<T: std::fmt::Debug>(&self, t: T); }
//! fn takes(_: &dyn Visit) {}
//! ```

use std::any::Any;
use std::fmt;

/// A minimal transfer.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Tx {
    pub from: u32,
    pub to: u32,
    pub amount: u64,
    pub fee: u64,
}

/// Why a rule rejected a transaction.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Violation {
    pub rule: String,
    pub reason: String,
}

impl fmt::Display for Violation {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}: {}", self.rule, self.reason)
    }
}

/// Helper supertrait that lets `Box<dyn Rule>` be cloned.
pub trait RuleClone {
    fn clone_box(&self) -> Box<dyn Rule>;
}

impl<T: Rule + Clone + 'static> RuleClone for T {
    fn clone_box(&self) -> Box<dyn Rule> {
        Box::new(self.clone())
    }
}

/// Helper supertrait that exposes a rule as `&dyn Any` for downcasting.
pub trait AsAny {
    fn as_any(&self) -> &dyn Any;
}

impl<T: Any> AsAny for T {
    fn as_any(&self) -> &dyn Any {
        self
    }
}

/// A check on one transaction. `Send + Sync` so rule sets can be shared across threads.
pub trait Rule: RuleClone + AsAny + Send + Sync {
    /// A short stable name for reports.
    fn name(&self) -> &str;
    /// `Ok(())` if the transaction passes, otherwise the reason it fails.
    fn check(&self, tx: &Tx) -> Result<(), String>;
}

impl Clone for Box<dyn Rule> {
    fn clone(&self) -> Self {
        self.clone_box()
    }
}

/// Rejects transfers above a limit.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct MaxAmount(pub u64);

impl Rule for MaxAmount {
    fn name(&self) -> &str {
        "max-amount"
    }
    fn check(&self, tx: &Tx) -> Result<(), String> {
        if tx.amount > self.0 {
            Err(format!("amount {} exceeds {}", tx.amount, self.0))
        } else {
            Ok(())
        }
    }
}

/// Rejects fees below a floor.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct MinFee(pub u64);

impl Rule for MinFee {
    fn name(&self) -> &str {
        "min-fee"
    }
    fn check(&self, tx: &Tx) -> Result<(), String> {
        if tx.fee < self.0 {
            Err(format!("fee {} below {}", tx.fee, self.0))
        } else {
            Ok(())
        }
    }
}

/// Rejects transfers to oneself.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct NoSelfTransfer;

impl Rule for NoSelfTransfer {
    fn name(&self) -> &str {
        "no-self-transfer"
    }
    fn check(&self, tx: &Tx) -> Result<(), String> {
        if tx.from == tx.to {
            Err(format!("account {} sends to itself", tx.from))
        } else {
            Ok(())
        }
    }
}

/// Rejects transfers touching blocked accounts.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Blocklist(pub Vec<u32>);

impl Rule for Blocklist {
    fn name(&self) -> &str {
        "blocklist"
    }
    fn check(&self, tx: &Tx) -> Result<(), String> {
        match [tx.from, tx.to].into_iter().find(|a| self.0.contains(a)) {
            Some(a) => Err(format!("account {a} is blocked")),
            None => Ok(()),
        }
    }
}

/// A rule built from a closure (cloneable, shareable closures only).
#[derive(Clone)]
pub struct FnRule<F> {
    name: &'static str,
    check: F,
}

impl<F> FnRule<F> {
    pub fn new(name: &'static str, check: F) -> Self {
        FnRule { name, check }
    }
}

impl<F> Rule for FnRule<F>
where
    F: Fn(&Tx) -> Result<(), String> + Clone + Send + Sync + 'static,
{
    fn name(&self) -> &str {
        self.name
    }
    fn check(&self, tx: &Tx) -> Result<(), String> {
        (self.check)(tx)
    }
}

/// An ordered list of rules of different types.
#[derive(Clone)]
pub struct RuleSet {
    rules: Vec<Box<dyn Rule>>,
    max_violations: usize,
}

impl Default for RuleSet {
    fn default() -> Self {
        RuleSet::new()
    }
}

impl RuleSet {
    /// An empty set that reports at most 16 violations from `check_all`.
    pub fn new() -> Self {
        RuleSet {
            rules: Vec::new(),
            max_violations: 16,
        }
    }

    /// Changes the cap on violations reported by [`RuleSet::check_all`].
    pub fn with_max_violations(mut self, max: usize) -> Self {
        self.max_violations = max;
        self
    }

    /// Adds a rule (any type implementing [`Rule`]).
    pub fn with<R: Rule + 'static>(mut self, rule: R) -> Self {
        self.rules.push(Box::new(rule));
        self
    }

    pub fn len(&self) -> usize {
        self.rules.len()
    }

    pub fn is_empty(&self) -> bool {
        self.rules.is_empty()
    }

    /// Stops at the first failing rule.
    pub fn check_fast(&self, tx: &Tx) -> Result<(), Violation> {
        for rule in &self.rules {
            rule.check(tx).map_err(|reason| Violation {
                rule: rule.name().to_string(),
                reason,
            })?;
        }
        Ok(())
    }

    /// Runs every rule and reports up to the configured maximum number of violations.
    pub fn check_all(&self, tx: &Tx) -> Result<(), Vec<Violation>> {
        let mut found = Vec::new();
        for rule in &self.rules {
            if found.len() >= self.max_violations {
                break;
            }
            if let Err(reason) = rule.check(tx) {
                found.push(Violation {
                    rule: rule.name().to_string(),
                    reason,
                });
            }
        }
        if found.is_empty() { Ok(()) } else { Err(found) }
    }

    /// Finds the first rule of concrete type `R`.
    pub fn find<R: Rule + 'static>(&self) -> Option<&R> {
        // `(**rule)` is the `dyn Rule` itself; calling `as_any` on the `Box`
        // would describe the box, not the rule inside it.
        self.rules
            .iter()
            .find_map(|rule| (**rule).as_any().downcast_ref::<R>())
    }
}

/// Static dispatch: the compiler generates a copy for each rule type.
pub fn count_passing_generic<R: Rule>(rule: &R, txs: &[Tx]) -> usize {
    txs.iter().filter(|tx| rule.check(tx).is_ok()).count()
}

/// Dynamic dispatch: one copy, one indirect call per transaction.
pub fn count_passing_dyn(rule: &dyn Rule, txs: &[Tx]) -> usize {
    txs.iter().filter(|tx| rule.check(tx).is_ok()).count()
}

#[cfg(test)]
mod tests {
    use super::*;

    fn tx(from: u32, to: u32, amount: u64, fee: u64) -> Tx {
        Tx {
            from,
            to,
            amount,
            fee,
        }
    }

    fn assert_send_sync<T: Send + Sync>() {}

    #[test]
    fn rule_is_dyn_compatible_and_sets_are_shareable() {
        let _: &dyn Rule = &NoSelfTransfer;
        assert_send_sync::<RuleSet>();
        assert_send_sync::<Box<dyn Rule>>();
    }

    #[test]
    fn rules_run_in_order_and_fast_stops_at_first() {
        let rules = RuleSet::new()
            .with(MinFee(5))
            .with(MaxAmount(10))
            .with(NoSelfTransfer);
        let bad = tx(1, 1, 99, 1);
        let first = rules.check_fast(&bad).unwrap_err();
        assert_eq!(first.rule, "min-fee");
        assert_eq!(first.to_string(), "min-fee: fee 1 below 5");
        let all = rules.check_all(&bad).unwrap_err();
        let names: Vec<_> = all.iter().map(|v| v.rule.as_str()).collect();
        assert_eq!(names, ["min-fee", "max-amount", "no-self-transfer"]);
        assert!(rules.check_all(&tx(1, 2, 10, 5)).is_ok());
        assert!(rules.check_fast(&tx(1, 2, 10, 5)).is_ok());
    }

    #[test]
    fn violation_cap_is_enforced() {
        let rules = RuleSet::new()
            .with_max_violations(2)
            .with(MinFee(5))
            .with(MaxAmount(10))
            .with(NoSelfTransfer);
        assert_eq!(rules.check_all(&tx(1, 1, 99, 1)).unwrap_err().len(), 2);
        let none = RuleSet::new().with_max_violations(0).with(MinFee(5));
        assert!(
            none.check_all(&tx(1, 2, 1, 0)).is_ok(),
            "a zero cap reports nothing"
        );
        assert!(RuleSet::default().is_empty());
    }

    #[test]
    fn empty_set_accepts_everything() {
        let rules = RuleSet::new();
        assert_eq!(rules.len(), 0);
        assert!(rules.check_all(&tx(0, 0, u64::MAX, 0)).is_ok());
    }

    #[test]
    fn blocklist_and_closure_rules() {
        let even_amount = FnRule::new("even-amount", |t: &Tx| {
            if t.amount.is_multiple_of(2) {
                Ok(())
            } else {
                Err(format!("{} is odd", t.amount))
            }
        });
        let rules = RuleSet::new().with(Blocklist(vec![7, 9])).with(even_amount);
        assert!(rules.check_fast(&tx(1, 2, 4, 0)).is_ok());
        assert_eq!(
            rules.check_fast(&tx(1, 9, 4, 0)).unwrap_err().reason,
            "account 9 is blocked"
        );
        assert_eq!(
            rules.check_fast(&tx(1, 2, 3, 0)).unwrap_err().rule,
            "even-amount"
        );
    }

    #[test]
    fn cloned_sets_are_independent_values() {
        let a = RuleSet::new().with(MaxAmount(10));
        let b = a.clone().with(MinFee(1));
        assert_eq!((a.len(), b.len()), (1, 2));
        assert!(a.check_fast(&tx(1, 2, 5, 0)).is_ok());
        assert!(b.check_fast(&tx(1, 2, 5, 0)).is_err());
    }

    #[test]
    fn find_downcasts_to_the_concrete_rule() {
        let rules = RuleSet::new().with(MaxAmount(10)).with(Blocklist(vec![3]));
        assert_eq!(rules.find::<MaxAmount>(), Some(&MaxAmount(10)));
        assert_eq!(rules.find::<Blocklist>().map(|b| b.0.len()), Some(1));
        assert!(rules.find::<MinFee>().is_none());
    }

    #[test]
    fn as_any_on_the_box_describes_the_box_not_the_rule() {
        let boxed: Box<dyn Rule> = Box::new(MaxAmount(10));
        assert!(
            boxed.as_any().downcast_ref::<MaxAmount>().is_none(),
            "Box<dyn Rule> is itself Any"
        );
        assert!(boxed.as_any().is::<Box<dyn Rule>>());
        assert!((*boxed).as_any().downcast_ref::<MaxAmount>().is_some());
    }

    #[test]
    fn static_and_dynamic_dispatch_agree() {
        let txs: Vec<Tx> = (0..50)
            .map(|i| tx(i, i % 7, u64::from(i) * 3, u64::from(i % 4)))
            .collect();
        let rule = MaxAmount(60);
        let g = count_passing_generic(&rule, &txs);
        let d = count_passing_dyn(&rule, &txs);
        let boxed: Box<dyn Rule> = Box::new(rule);
        assert_eq!(g, d);
        assert_eq!(g, count_passing_dyn(&*boxed, &txs));
        assert_eq!(g, 21);
    }

    #[test]
    fn rules_can_be_shared_across_threads() {
        let rules = std::sync::Arc::new(RuleSet::new().with(MaxAmount(5)));
        let handles: Vec<_> = (0..4u64)
            .map(|i| {
                let rules = std::sync::Arc::clone(&rules);
                std::thread::spawn(move || rules.check_fast(&tx(1, 2, i * 3, 0)).is_ok())
            })
            .collect();
        let results: Vec<bool> = handles.into_iter().map(|h| h.join().unwrap()).collect();
        assert_eq!(results, [true, true, false, false]);
    }
}
```

✓ With this exact file: `cargo fmt --check` clean, `cargo clippy --all-targets --locked -- -D warnings` clean, `cargo test --locked` → 10 unit tests, 1 passing doctest and 1 `compile_fail` doctest passed. Two real compiler complaints shaped the final code: my first `find<R: Rule>` failed with `error[E0310]: the parameter type `R` may not live long enough` (fixed with `R: Rule + 'static`, §4.1), and clippy 0.1.97 rejected `t.amount % 2 == 0` in a test closure in favor of `is_multiple_of(2)`. I also mutated the code to check the tests have teeth: replacing `(**rule).as_any()` with `rule.as_any()` in `find` made `find_downcasts_to_the_concrete_rule` fail. In a separate crate I confirmed ✓ that the `compile_fail` doctest's trait gives `error[E0038]: the trait `Visit` is not dyn compatible`; the doctest alone proves only that *some* error occurs.

**Design notes.**

- **Helper supertraits.** `RuleClone` and `AsAny` each have a blanket impl (`impl<T: Rule + Clone + 'static> RuleClone for T`, `impl<T: Any> AsAny for T`), so a new rule only writes `name` and `check`. `trait Rule: RuleClone + AsAny + Send + Sync` makes both helpers, and thread-safety, part of every rule's contract.
- **The `Box` trap.** `Box<dyn Rule>` is itself `'static`, so it also gets `AsAny` from the blanket impl. Calling `as_any` on the box (or on a `&Box<dyn Rule>`, which autoderef reaches first) describes the box; `(**rule).as_any()` reaches the `dyn Rule` inside. The test pins this behavior down.
- **Orphan rule check.** `impl Clone for Box<dyn Rule>` is legal in this crate because `Box` is fundamental and `dyn Rule` is a local trait (Lesson 20 §5).
- **Static versus dynamic dispatch side by side.** The two counting functions have identical bodies; the test asserts equal results (21 of 50 transactions pass `MaxAmount(60)`).
- **Defaults chosen for safety.** `check_all` returns at most `max_violations` entries (default 16) so a rule set that fails everything cannot produce an unbounded report; a cap of `0` reports nothing (documented by a test, which may surprise callers; `check_fast` is the strict check).
- **Not a policy engine.** Rules are stateless, there is no ordering by cost, no panic isolation (§9), and `FnRule` requires `Clone + Send + Sync + 'static` closures so the whole set stays cloneable and shareable.

</details>

---

## 13. Interview questions

<details markdown="1">
<summary>Answers</summary>

1. **What is a trait object?** A value of an unknown concrete type accessed through a trait: `&dyn Trait`, `Box<dyn Trait>`, etc. Methods are dispatched at run time through a vtable.
2. **Static vs dynamic dispatch?** Static (generics): the compiler generates a copy per type and the call is direct and inlinable. Dynamic (`dyn`): one copy, an indirect call through a vtable, no inlining across it, and heterogeneous collections become possible.
3. **What is a fat pointer?** A pointer plus metadata: for `dyn Trait`, the data pointer and a vtable pointer (16 bytes on 64-bit); for slices, pointer plus length.
4. **What is in a vtable?** Per (type, trait) pair: a drop function, the type's size and alignment, and one function pointer per method (exact layout unspecified).
5. **What is dyn compatibility (object safety)?** The set of rules that let a trait build a vtable: no generic methods, no `Self` in return or argument positions, no receiver-less functions or associated consts, no `Sized` supertrait, no `async`/`-> impl Trait` methods, unless opted out with `where Self: Sized`.
6. **Why isn't `Clone` dyn compatible and how do you work around it?** `clone` returns `Self` and the trait requires `Sized`. Add a `clone_box(&self) -> Box<dyn Trait>` method via a helper supertrait and implement `Clone for Box<dyn Trait>`.
7. **What does `where Self: Sized` on a method do?** It removes the method from the vtable (unavailable on `dyn`), letting a trait stay dyn compatible while having generic or by-value methods (as `Iterator` does).
8. **Why can `Box<dyn Iterator>` call `.map()`?** `Box<I>` implements `Iterator` for `I: Iterator + ?Sized`, and the adapter methods are available on that sized `Box`.
9. **What does `Box<dyn Trait>` default to for lifetimes?** `+ 'static`. Use `Box<dyn Trait + 'a>` to hold borrowed implementors.
10. **Is `dyn Trait` `Send`?** Not automatically. Write `dyn Trait + Send` (and `+ Sync`) or make them supertraits.
11. **How do you get the concrete type back from a trait object?** `dyn Any` with `downcast_ref`/`downcast` (requires `'static`), or an `as_any` method on your trait. Prefer adding trait methods over downcasting.
12. **What is trait upcasting?** Converting `&dyn Sub` to `&dyn Super` (and `Box`), stable on recent compilers (I recall 1.86).
13. **When would you choose an enum over `dyn`?** When you own the complete set of variants: no allocation, no indirect call, exhaustive matching.
14. **Why does `fn f() -> impl Trait` fail for two different returns?** It denotes one hidden concrete type; return `Box<dyn Trait>` or an enum for several.
15. **Why do generic functions need `?Sized` to accept `&dyn Trait`?** Type parameters are `Sized` by default and `dyn Trait` is not.
16. **How does a trait object affect performance?** An indirect call per method, no inlining across it, and usually a heap allocation; benefits: less code. Measure before choosing.
17. **Does `dyn Trait` isolate untrusted code?** No. It is a dispatch mechanism; isolation needs a process or Wasm boundary.
18. **How do you share a `Vec<Box<dyn Trait>>` across threads?** Require `Send + Sync` on the trait (or `dyn Trait + Send + Sync`) and wrap in `Arc`.
19. **Where does the E0038 error appear?** At the use of `dyn Trait`, not at the trait definition.

</details>

---

## 14. Key takeaways

- A trait object is a pointer to a value of unknown type plus a vtable pointer (two words), created by an unsizing coercion; calls go through the vtable, are not inlined, and let one collection hold many types.
- Only dyn-compatible traits can be `dyn`: no generic methods, `Self` returns, receiver-less functions, associated consts, `Sized` supertraits, `async` or `-> impl Trait` methods, unless `where Self: Sized` hides them from the trait object. E0038 shows up where `dyn` is used.
- `Clone` and similar need a helper (`clone_box`); `Any` provides downcasting when you truly need the concrete type, with `'static` and the box-in-a-box traps.
- `Box<dyn Trait>` means `+ 'static` and no `Send`/`Sync`; spell out `+ 'a`, `+ Send + Sync`, or put them in supertraits.
- Choose generics for speed and open compile-time sets, `dyn` for open run-time sets and smaller code, and enums for closed sets. Measure before optimizing.
- A trait object is not a sandbox: bound what plugins can make you do, make thread-safety part of the trait, and treat the dyn-compatibility of a public trait as API.

**Connections.** Lesson 19's monomorphization and `?Sized`, Lesson 20's forwarding impls for `&T`/`Box<T>`, the orphan rule for `Box<dyn Local>` and associated types on `dyn Iterator<Item = T>`, Lesson 18's `Box<dyn Fn>` and Lesson 15's typestate (compile-time) versus `State` objects (run-time) all meet here. Next: Lesson 22 (generic associated types, `Send`/`Sync` in depth, `PhantomData` and variance, dynamically sized types), then Lesson 23 (`Box`, `Rc`, `Arc`), which uses `Rc<dyn Trait>` and `Arc<dyn Trait>` heavily. Lesson 26 covers error design (`Box<dyn Error>`, `source`, downcasting errors), Lesson 33 covers API design (semver of dyn compatibility, sealed traits), Lesson 41 covers measuring dispatch costs, and Lesson 47 covers Wasm as a real plugin boundary. See the [roadmap](00_roadmap.md) and the [course index](index.md).

---

## 15. References

- *The Rust Programming Language*, ch. 18.2 "Using Trait Objects That Allow for Values of Different Types" and the "Dyn compatibility / object safety" discussion in ch. 17 or 18 (chapter numbering changed between book editions): <https://doc.rust-lang.org/book/ch18-02-trait-objects.html>
- *The Rust Reference*, "Trait objects" and "Traits" (including "Dyn compatibility"): <https://doc.rust-lang.org/reference/types/trait-object.html>, <https://doc.rust-lang.org/reference/items/traits.html>
- *The Rust Reference*, "Lifetime elision" (default object lifetime bounds) and "Dynamically sized types": <https://doc.rust-lang.org/reference/lifetime-elision.html#default-trait-object-lifetimes>, <https://doc.rust-lang.org/reference/dynamically-sized-types.html>
- `std::any::Any`: <https://doc.rust-lang.org/std/any/trait.Any.html>; `std::boxed::Box`: <https://doc.rust-lang.org/std/boxed/struct.Box.html>; `std::marker::Send`, `Sync`, `Sized`
- RFC 255 "Object safety" (original rules), RFC 3324 "dyn upcasting coercion": <https://rust-lang.github.io/rfcs/0255-object-safety.html>, <https://rust-lang.github.io/rfcs/3324-dyn-upcasting.html>
- *The Rustonomicon*, "Exotically Sized Types" (DSTs): <https://doc.rust-lang.org/nomicon/exotic-sizes.html>
- *Rust API Guidelines*: "Traits are object-safe if they may be useful as a trait object" (C-OBJECT): <https://rust-lang.github.io/api-guidelines/flexibility.html>
- Rust error code index: E0038, E0277, E0308, E0310: <https://doc.rust-lang.org/error_codes/E0038.html>, <https://doc.rust-lang.org/error_codes/E0310.html>

The URLs above are the standard documentation locations; I did not open them during this session (the sandbox cannot reach them), so if a link has moved, search the docs site for the page title. Unchecked from memory: the Book chapter numbers and anchors, the RFC numbers (255, 3324), that trait upcasting stabilized in Rust 1.86 (I confirmed it compiles on 1.97, not the release), the layout of the vtable (I saw only the `callq *24(%rsi)` call, not the table), the claim that `self: Rc<Self>`/`Arc<Self>`/`Pin<..>` receivers are also allowed (I ran `self: Box<Self>` only), the `dyn-clone` crate's name and purpose, the API-guidelines item name, and the performance statements about indirect calls and cache behavior (I measured nothing).
