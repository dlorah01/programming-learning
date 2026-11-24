---
layout: default
title: "Lesson 19 — Generics, trait bounds, and monomorphization"
---

# Lesson 19 — Generics, trait bounds, and monomorphization

**Module 7 · Generics and traits** · Status: generated · Edition used in examples: **2024** · External dependencies: none

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux. As before, ✓ marks outputs and diagnostics I actually ran; unmarked statements are documented behavior I did not run. Everything is **stable Rust**.

Builds on [Lesson 13](13_structs_enums_methods.md) (structs, enums, `impl` blocks, derives), [Lesson 15](15_newtype_typestate.md) (`PhantomData`, newtypes, the orphan rule), [Lesson 16](16_collections.md) (`HashMap<K, V>`, `K: Hash + Eq`), [Lesson 17](17_iterators.md) (`Iterator` and adapter signatures) and [Lesson 18](18_closures.md) (`F: Fn(A) -> B`).

---

## 1. The problem generics solve

Without generics you would write `largest_i32`, `largest_f64`, `largest_str`... or give up types and use something like `Any`. Rust's answer is **parametric polymorphism**: write the function once with a *type parameter* and let the compiler check it for all types that qualify.

```rust
fn largest<T: PartialOrd>(items: &[T]) -> Option<&T> {
    let mut it = items.iter();
    let mut best = it.next()?;
    for x in it { if x > best { best = x; } }
    Some(best)
}
#[derive(Debug)]
struct Pair<T> { a: T, b: T }
impl<T> Pair<T> { fn swap(self) -> Pair<T> { Pair { a: self.b, b: self.a } } }
impl<T: PartialOrd + Copy> Pair<T> { fn max(&self) -> T { if self.a > self.b { self.a } else { self.b } } }
#[derive(Debug)]
enum Either<L, R> { Left(L), Right(R) }
fn main() {
    println!("{:?} {:?} {:?}", largest(&[3, 9, 2]), largest(&["b", "a"]), largest::<f64>(&[]));
    let p = Pair { a: 1, b: 2 };
    println!("{} {:?}", p.max(), p.swap());
    let e: Either<i32, &str> = Either::Right("r");
    println!("{e:?} {:?}", Either::<i32, &str>::Left(1));
    let v: Vec<u8> = Vec::<u8>::with_capacity(3);
    println!("{}", v.capacity());
}
```

✓ Output `Some(9) Some("b") None`, `2 Pair { a: 2, b: 1 }`, `Right("r") Left(1)`, `3`. `Vec<T>`, `Option<T>`, `Result<T, E>`, `HashMap<K, V>` are all ordinary generic types; you have been using generics since Lesson 14.

Syntax: type parameters go in angle brackets after the name (`fn largest<T>`, `struct Pair<T>`, `enum Either<L, R>`, `impl<T> Pair<T>`). The `impl<T>` is *declaring* `T` for the block; `Pair<T>` is *using* it. The **turbofish** `::<>` supplies parameters explicitly when inference cannot (`largest::<f64>(&[])`, `Vec::<u8>::with_capacity`).

### 1.1 Rust generics vs C++ templates and Java generics

- **C++ templates** are checked *at instantiation*: a template body that uses `a + b` fails to compile only when you instantiate it with a type lacking `+`, with a long error. Rust checks the generic body **once, at its definition**, against the declared bounds. If the body compiles, every type satisfying the bounds works (no instantiation-time surprises).
- **Java generics** erase types at run time and box primitives. Rust **monomorphizes**: it generates specialized machine code for each concrete type (Section 6), so `Vec<u8>` really stores bytes and calls are direct.

The cost of Rust's choice is more generated code (Section 6) and the need to *state* what you can do with `T`.

---

## 2. Trait bounds: saying what `T` can do

Inside a generic function you know *nothing* about `T` except what the bounds promise. A bare `T` cannot be added, printed, compared or cloned ✓:

```rust
fn sum<T>(a: T, b: T) -> T { a + b }
fn main() { println!("{}", sum(1, 2)); }
```

✓ `error[E0369]: cannot add `T` to `T``, with the help to restrict `T` with `std::ops::Add<Output = T>`. And for methods ✓:

```rust
fn show<T>(x: T) -> String { x.to_string() }
fn main() { println!("{}", show(1)); }
```

✓ `error[E0599]: no method named `to_string` found for type parameter `T` in the current scope`. Add bounds ✓:

```rust
use std::fmt::Display;
use std::ops::Add;
fn sum<T: Add<Output = T>>(a: T, b: T) -> T { a + b }
fn show<T: Display>(x: T) -> String { x.to_string() }
fn main() { println!("{} {} {}", sum(1, 2), sum(1.5, 2.0), show("hi")); }
```

✓ Output `3 3.5 hi`. The compiler's error messages are the best guide to which bound is missing; read the `help:` lines.

The other direction, passing a type that does *not* meet the bound, is a different error at the call site ✓:

```rust
use std::fmt::Display;
fn show<T: Display>(x: T) -> String { format!("{x}") }
struct NoDisplay;
fn main() { println!("{}", show(NoDisplay)); }
```

✓ `error[E0277]: `NoDisplay` doesn't implement `std::fmt::Display``. So: **E0369/E0599 = the body asks for more than the bounds give; E0277 = the caller gives less than the bounds ask.**

### 2.1 Three spellings of the same bound

```rust
use std::collections::HashMap;
use std::fmt::{Debug, Display};
use std::hash::Hash;
fn count<K, I>(items: I) -> HashMap<K, usize>
where K: Hash + Eq, I: IntoIterator<Item = K> {
    let mut m = HashMap::new();
    for k in items { *m.entry(k).or_insert(0) += 1; }
    m
}
fn describe(x: &(impl Display + Debug)) -> String { format!("{x} / {x:?}") }
fn describe2<T>(x: &T) -> String where T: Display + Debug + ?Sized { format!("{x} / {x:?}") }
fn main() {
    let m = count("abca".chars());
    let mut v: Vec<_> = m.into_iter().collect(); v.sort();
    println!("{v:?}");
    println!("{} | {}", describe(&"hi"), describe2("yo"));
}
```

✓ Output `[('a', 2), ('b', 1), ('c', 1)]` and `hi / "hi" | yo / "yo"`.

- **Inline**: `fn f<T: A + B>(x: T)`: short bounds.
- **`where` clause**: `fn f<T>(x: T) where T: A + B`: use it when bounds are long, involve several parameters, or mention associated types (`I: IntoIterator<Item = K>`).
- **`impl Trait` in argument position**: `fn f(x: impl A + B)` is sugar for an anonymous generic parameter. It is shorter, but you **cannot name the type** and **cannot use the turbofish** on it ✓: `fn take(_: impl Copy) {}` then `take::<i32>(5)` gives `error[E0107]: function takes 0 generic arguments but 1 generic argument was supplied`. Use a named parameter when callers may need to specify it, or when two arguments must have the *same* type (`fn f<T: A>(x: T, y: T)` vs `fn f(x: impl A, y: impl A)`, which allows two different types).

### 2.2 `Sized` and `?Sized`

Every type parameter has an implicit bound `T: Sized` (its size is known at compile time), because the compiler must know how big a `T` is to pass it by value. So a function generic over `T` cannot be called with `str` or `[i32]` ✓:

```rust
fn show<T: std::fmt::Debug>(x: &T) { println!("{x:?}"); }
fn main() { show("a str"); }
```

✓ `error[E0277]: the size for values of type `str` cannot be known at compilation time`. If you only use `T` behind a reference, relax the bound with `?Sized` ("may or may not be sized") ✓:

```rust
fn show<T: std::fmt::Debug + ?Sized>(x: &T) { println!("{x:?}"); }
fn main() { show("a str"); show(&[1, 2][..]); show(&5); }
```

✓ Output `"a str"`, `[1, 2]`, `5`. This is why the standard library writes `impl<T: Display + ?Sized> ...` in many places. (Lesson 22 covers dynamically sized types and `dyn Trait` in detail.)

### 2.3 Conditional methods: bounds on `impl` blocks

Put bounds on the **`impl`**, not on the struct, so the type can exist for any `T` and methods appear only when they make sense ✓:

```rust
use std::fmt::Display;
struct W<T>(T);
impl<T: Display> W<T> { fn show(&self) -> String { format!("[{}]", self.0) } }
impl<T> W<T> { fn inner(&self) -> &T { &self.0 } }
struct NoDisp;
fn main() {
    println!("{}", W(5).show());
    let w = W(NoDisp);
    let _ = w.inner();
    println!("{}", w.show());
}
```

✓ `error[E0599]: the method `show` exists for struct `W<NoDisp>`, but its trait bounds were not satisfied`. Without the last line it prints `[5]`. `W<NoDisp>` is a perfectly good type; it just lacks `show`. This is how `Vec<T>::sort` exists only for `T: Ord`, and how `Option<T>::cloned` exists only for `Option<&T>` with `T: Clone`. The Rust API guidelines recommend *not* repeating bounds on struct definitions unless the struct itself needs them (for example `HashMap<K, V>` has no bound on the struct, and `Hash + Eq` on the methods ✓ with a `Counter<K>` of my own that printed `2 1 0`).

---

## 3. Traits: defining shared behavior

A **trait** is a set of method signatures (with optional default bodies) that types can implement. Generics + traits are how Rust does polymorphism without inheritance.

```rust
trait Shape {
    fn area(&self) -> f64;
    fn name(&self) -> String { String::from("shape") }     // default method
    fn describe(&self) -> String { format!("{} with area {:.2}", self.name(), self.area()) }
}
struct Circle(f64);
struct Rect { w: f64, h: f64 }
impl Shape for Circle { fn area(&self) -> f64 { std::f64::consts::PI * self.0 * self.0 } fn name(&self) -> String { "circle".into() } }
impl Shape for Rect { fn area(&self) -> f64 { self.w * self.h } }
fn total_area<S: Shape>(shapes: &[S]) -> f64 { shapes.iter().map(Shape::area).sum() }
fn print_it(s: &impl Shape) { println!("{}", s.describe()); }
fn main() {
    print_it(&Circle(1.0));
    print_it(&Rect { w: 2.0, h: 3.0 });
    println!("{}", total_area(&[Rect { w: 1.0, h: 1.0 }, Rect { w: 2.0, h: 2.0 }]));
}
```

✓ Output `circle with area 3.14`, `shape with area 6.00`, `5`. `describe` and (for `Rect`) `name` use the trait's default bodies; `Circle` overrides `name`. Note `shapes.iter().map(Shape::area)` passes a trait method path where a closure is expected (Lesson 18).

A generic function with `S: Shape` takes a slice of **one** concrete type. Mixing two types is a type error, because `&[A, B]` is not a valid array ✓:

```rust
trait Shape { fn area(&self) -> f64; }
struct A; struct B;
impl Shape for A { fn area(&self) -> f64 { 1.0 } }
impl Shape for B { fn area(&self) -> f64 { 2.0 } }
fn total<S: Shape>(shapes: &[S]) -> f64 { shapes.iter().map(|s| s.area()).sum() }
fn main() {
    println!("{}", total(&[A, B]));
}
```

✓ `error[E0308]: mismatched types`. To hold different shapes in one collection you need **trait objects** (`Vec<Box<dyn Shape>>`, Lesson 21). The choice between the two is Section 7.

### 3.1 Traits must be in scope

A trait's methods are only callable when the trait is imported (`use`) ✓:

```rust
mod shapes {
    pub trait Area { fn area(&self) -> f64; }
    pub struct Sq(pub f64);
    impl Area for Sq { fn area(&self) -> f64 { self.0 * self.0 } }
}
fn main() {
    let s = shapes::Sq(2.0);
    println!("{}", s.area());
}
```

✓ `error[E0599]: no method named `area` found for struct `Sq` in the current scope`, with the help "trait `Area` which provides `area` is implemented but not in scope". Add `use shapes::Area;`. (This is why preludes exist.)

### 3.2 Same method name from several traits: disambiguation

```rust
trait Pilot { fn fly(&self) -> &'static str; }
trait Wizard { fn fly(&self) -> &'static str; }
struct Human;
impl Pilot for Human { fn fly(&self) -> &'static str { "captain speaking" } }
impl Wizard for Human { fn fly(&self) -> &'static str { "up!" } }
impl Human { fn fly(&self) -> &'static str { "*waves arms*" } }
trait Animal { fn baby_name() -> String; }
struct Dog;
impl Dog { fn baby_name() -> String { "Spot".into() } }
impl Animal for Dog { fn baby_name() -> String { "puppy".into() } }
fn main() {
    let h = Human;
    println!("{} | {} | {}", h.fly(), Pilot::fly(&h), <Human as Wizard>::fly(&h));
    println!("{} {}", Dog::baby_name(), <Dog as Animal>::baby_name());
}
```

✓ Output `*waves arms* | captain speaking | up!` and `Spot puppy`. An inherent method wins over trait methods; the **fully qualified syntax** `<Type as Trait>::method(...)` picks a specific one (and is required for associated functions without `self`). If two traits provide the method and there is no inherent one, `Human.fly()` is ambiguous ✓ `error[E0034]: multiple applicable items in scope`.

### 3.3 Supertraits and associated consts

A trait can require another trait (`trait Named: Display`), so its default methods may use the supertrait's methods; and traits can carry constants ✓:

```rust
use std::fmt::{Debug, Display};
trait Named: Display { fn id(&self) -> u32; fn label(&self) -> String { format!("#{} {}", self.id(), self) } }
struct U(u32);
impl Display for U { fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result { write!(f, "user{}", self.0) } }
impl Named for U { fn id(&self) -> u32 { self.0 } }
fn show<T: Named + Debug>(t: &T) -> String { format!("{} {:?}", t.label(), t.id()) }
#[derive(Debug)] struct V(u32);
impl Display for V { fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result { write!(f, "v") } }
impl Named for V { fn id(&self) -> u32 { self.0 } }
fn main() { println!("{}", U(7).label()); println!("{}", show(&V(3))); }
```

✓ Output `#7 user7` and `#3 v 3`. And associated constants, resolved at compile time per type ✓:

```rust
trait Unit { const NAME: &'static str; const FACTOR: f64; fn to_base(v: f64) -> f64 { v * Self::FACTOR } }
struct Km; struct Mile;
impl Unit for Km { const NAME: &'static str = "km"; const FACTOR: f64 = 1000.0; }
impl Unit for Mile { const NAME: &'static str = "mi"; const FACTOR: f64 = 1609.344; }
fn show<U: Unit>(v: f64) -> String { format!("{v} {} = {} m", U::NAME, U::to_base(v)) }
fn main() { println!("{} | {}", show::<Km>(2.0), show::<Mile>(1.0)); }
```

✓ Output `2 km = 2000 m | 1 mi = 1609.344 m`. Here `U` appears only in the *return-type-less* position (no value of type `U` is passed), so the turbofish chooses it. This "type as a parameter, no value" idiom is common for units, codecs and chain parameters.

---

## 4. Blanket impls and conditional trait impls

A **blanket impl** implements a trait for *every* type meeting a bound:

```rust
use std::fmt::Display;
trait Describe { fn describe(&self) -> String; }
impl<T: Display + ?Sized> Describe for T { fn describe(&self) -> String { format!("<{}>", self) } }
fn main() { println!("{} {} {}", 5.describe(), "str".describe(), String::from("s").describe()); }
```

✓ Output `<5> <str> <s>`. The standard library does this a lot: `impl<T: Display> ToString for T`, `impl<I: Iterator> IntoIterator for I`, `impl<T, U: From<T>> Into<U> for T`. That is why implementing `Display` gives you `to_string()` and implementing `From<A> for B` gives you `a.into()` for free (✓ `len_of(impl Into<Meters>)` accepted both `2.5` and `3u32` with `From` impls for `f64` and `u32`).

Blanket impls are powerful and **restrictive**: once there is `impl<T: Display> Describe for T`, you cannot add `impl Describe for i32`, because `i32: Display`, so the two would overlap ✓:

```rust
use std::fmt::Display;
trait Describe { fn describe(&self) -> String; }
impl<T: Display> Describe for T { fn describe(&self) -> String { format!("{self}") } }
impl Describe for i32 { fn describe(&self) -> String { "int".into() } }
fn main() {}
```

✓ `error[E0119]: conflicting implementations of trait `Describe` for type `i32``. This is the **coherence** rule: for any type and trait there may be at most one applicable impl. (Specialization, which would allow the more specific one to win, is not stable; Lesson 20 covers coherence and the orphan rule in full.) Remedies: remove the blanket impl, give the special case its own trait, or wrap the type in a newtype (Lesson 15).

Two more generic-declaration errors worth recognizing ✓:

```rust
struct S;
impl<T: Default> S { fn make() -> i32 { 1 } }
fn main() {}
```

✓ `error[E0207]: the type parameter `T` is not constrained by the impl trait, self type, or predicates`: `T` appears nowhere the compiler could deduce it from, so the impl can never be selected. And an unused parameter ✓:

```rust
struct Wrapper<T> { n: u32 }
fn main() {}
```

✓ `error[E0392]: type parameter `T` is never used`. If you want a marker type, hold a `PhantomData<T>` (Lesson 15, and Lesson 22 for variance). Wrong argument counts give ✓ `E0107` (`struct takes 1 generic argument but 2 generic arguments were supplied`; `missing generics for enum `Option``).

---

## 5. Generic code that returns, converts and infers

The caller can decide the type through the **return type**, or the compiler infers it ✓:

```rust
fn make<T: Default>() -> T { T::default() }
fn pair<A: Clone, B>(a: &A, b: B) -> (A, B) { (a.clone(), b) }
fn main() {
    let a: i32 = make(); let s: String = make(); let v = make::<Vec<u8>>();
    println!("{a} {s:?} {v:?} {:?}", pair(&"x", 2));
}
```

✓ Output `0 "" [] ("x", 2)`. When nothing determines `T` the compiler asks you ✓:

```rust
fn main() {
    let x = Default::default();
    println!("{:?}", x);
}
```

✓ `error[E0790]: cannot call associated function on trait without specifying the corresponding `impl` type`. Annotate (`let x: i32 = Default::default();`). `parse` is the classic case: `"7".parse::<u8>()` or `let n: u8 = "7".parse()?;`, and a helper `fn parse_all<T: FromStr>(s: &str) -> Vec<T>` is called as `parse_all::<i32>(..)` or with an annotated binding ✓ (`[1, 2, 4]` and `[1.5, 2.0]`).

Accepting `impl Into<X>` makes an API flexible for callers: `fn describe<T: Into<String>>(s: T)` accepted `&str` and `String` ✓. Don't overdo it; each such parameter is another monomorphized copy (Section 6).

---

## 6. Monomorphization: what the compiler generates

For each distinct set of concrete type arguments a generic function is used with, the compiler generates a **separate specialized copy** of the code. That process is **monomorphization**. You can see it in the symbols of a program ✓. I compiled this (debug build, so nothing is inlined away):

```rust
#[inline(never)]
pub fn double_it<T: std::ops::Add<Output = T> + Copy>(x: T) -> T { x + x }
#[inline(never)]
pub fn double_dyn(x: &dyn Fn() -> i64) -> i64 { x() + x() }
fn main() {
    println!("{} {} {}", double_it(1i32), double_it(1.5f64), double_it(7u8));
    println!("{}", double_dyn(&|| 21));
    println!("{}", double_dyn(&|| 4));
}
```

✓ Output `2 3 14`, `42`, `8`. `nm -C` on the binary listed **three** generic copies, `m01::double_it::<f64>`, `m01::double_it::<u8>` and `m01::double_it::<i32>`, and **one** `m01::double_dyn` (the `dyn` version is compiled once, whatever closure you pass). A second `double_it(2i32)` call reuses the `i32` copy; an instance exists per distinct *type*, not per call. Two caveats from my own run: with `-O` and constant arguments the optimizer folded the calls away and no `double_it` symbols remained at all (a copy that is inlined or constant-folded need not exist); using `std::hint::black_box(arg)` to stop that, the optimized binary again had exactly the three instances.

What this means:

- **Speed**: each copy is compiled for its exact types, so `a + a` on `i32` is a single add instruction and can be inlined. No boxing, no virtual call. Generic code is typically as fast as hand-specialized code.
- **Code size and compile time**: more distinct instantiations means more code to generate and optimize (this is a known reason for large Rust binaries and slow builds). Generic functions that are big and instantiated many times are the usual culprits. Common mitigations: keep the generic part small and move the bulk into a non-generic inner function (`fn f<P: AsRef<Path>>(p: P) { inner(p.as_ref()) }`), or use `&dyn Trait` for cold paths. I did not measure compile times or binary sizes in this lesson.
- **Generic types too**: `Vec<u8>` and `Vec<String>` are different types with different layouts. `size_of` follows the parameter ✓: `Wrap<u8>` 1, `Wrap<u64>` 8, `Wrap<String>` 24; `Option<u8>` 2, `Option<u64>` 16, `Option<&u8>` 8 (the niche, Lesson 09); a `PhantomData<T>` field adds nothing ✓ (`Tagged<String>` and `Tagged<()>` both 4).
- **Errors are reported on the generic code**, not at each instantiation (with rare exceptions in const-generic evaluation). That is the payoff of bounds.

### 6.1 Static dispatch vs dynamic dispatch

```rust
trait Speak { fn speak(&self) -> String; }
struct Dog; struct Cat;
impl Speak for Dog { fn speak(&self) -> String { "woof".into() } }
impl Speak for Cat { fn speak(&self) -> String { "meow".into() } }
fn static_dispatch<S: Speak>(s: &S) -> String { s.speak() }
fn dynamic_dispatch(s: &dyn Speak) -> String { s.speak() }
fn main() {
    println!("{} {}", static_dispatch(&Dog), static_dispatch(&Cat));
    let zoo: Vec<Box<dyn Speak>> = vec![Box::new(Dog), Box::new(Cat)];
    println!("{:?}", zoo.iter().map(|a| dynamic_dispatch(a.as_ref())).collect::<Vec<_>>());
    println!("{} {}", std::mem::size_of::<&Dog>(), std::mem::size_of::<&dyn Speak>());
}
```

✓ Output `woof meow`, `["woof", "meow"]`, `8 16`. A `&Dog` is a single pointer; a `&dyn Speak` is a fat pointer (data pointer plus vtable pointer), 16 bytes.

| | Generics (`T: Trait`, `impl Trait`) | Trait objects (`dyn Trait`) |
|---|---|---|
| Dispatch | static, resolved at compile time | dynamic, via a vtable |
| Code | one copy per type (monomorphized) | one copy |
| Inlining | yes | no (indirect call) |
| Mixed types in one collection | no | yes (`Vec<Box<dyn Trait>>`) |
| Needs `Sized` / object safety | `Sized` by default | trait must be object safe (Lesson 21) |
| Typical use | hot paths, libraries, zero-cost abstractions | plugins, heterogeneous collections, reducing code size |

Default to generics; reach for `dyn` when you need the heterogeneity or want to cut instantiations.

---

## 7. Const generics

A generic parameter can also be a **constant value** (an integer, `bool` or `char`), most often an array length. `[T; N]` is itself generic over `N`:

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
struct Matrix<const R: usize, const C: usize>([[i32; C]; R]);
impl<const R: usize, const C: usize> Matrix<R, C> {
    fn transpose(&self) -> Matrix<C, R> {
        let mut out = [[0; R]; C];
        for r in 0..R { for c in 0..C { out[c][r] = self.0[r][c]; } }
        Matrix(out)
    }
}
fn first_n<const N: usize>(v: &[u8]) -> Option<[u8; N]> { v.get(..N)?.try_into().ok() }
fn sum_arr<T: Copy + std::iter::Sum<T>, const N: usize>(a: [T; N]) -> T { a.iter().copied().sum() }
fn main() {
    let m = Matrix([[1, 2, 3], [4, 5, 6]]);
    println!("{:?}", m.transpose());
    println!("{:?} {:?}", first_n::<2>(&[1, 2, 3]), first_n::<4>(&[1, 2, 3]));
    println!("{} {}", sum_arr([1, 2, 3]), sum_arr([0.5; 4]));
    println!("{} {}", std::mem::size_of::<[u8; 7]>(), std::mem::size_of::<Matrix<2, 3>>());
}
```

✓ Output:

```text
Matrix([[1, 4], [2, 5], [3, 6]])
Some([1, 2]) None
6 2
7 24
```

`first_n::<4>` returns `None` because the slice has only three bytes (`try_into` from slice to array checks the length). `Matrix<2, 3>` is 6 `i32`s, 24 bytes: the dimensions cost nothing at run time. The payoff is that **dimension mismatches become type errors** ✓:

```rust
struct M<const R: usize, const C: usize>([[i32; C]; R]);
fn mul<const R: usize, const K: usize, const C: usize>(a: &M<R, K>, b: &M<K, C>) -> M<R, C> {
    let mut out = [[0; C]; R];
    for r in 0..R { for c in 0..C { for k in 0..K { out[r][c] += a.0[r][k] * b.0[k][c]; } } }
    M(out)
}
fn main() {
    let a = M([[1, 2, 3], [4, 5, 6]]);   // 2x3
    let b = M([[1, 2], [3, 4]]);         // 2x2
    let _ = mul(&a, &b);
}
```

✓ `error[E0308]: mismatched types` ("expected `3`, found `2`"): the inner dimension `K` must be the same on both sides. The practical challenge builds a full matrix type this way. Limits (as of this toolchain): const parameters may only be primitive integers, `bool` or `char`, and expressions over them such as `[T; N + 1]` need the unstable `generic_const_exprs` feature, so `Matrix<T, R, K>` where `K = C + 1` can't be written on stable (from memory; I did not test it). Also remember that large arrays live on the stack: `[u8; 10_000_000]` as a local can overflow it (Lesson 08); use `Vec` or `Box<[u8]>`.

### 7.1 Operators are generic traits

Operators are traits in `std::ops` with default type parameters: `trait Add<Rhs = Self> { type Output; fn add(self, rhs: Rhs) -> Self::Output; }`. You can implement it several times for different `Rhs`, and use `Add<Output = T>` as a bound ✓:

```rust
use std::ops::Add;
#[derive(Debug, Clone, Copy, PartialEq)]
struct V2(f32, f32);
impl Add for V2 { type Output = V2; fn add(self, o: V2) -> V2 { V2(self.0 + o.0, self.1 + o.1) } }
impl Add<f32> for V2 { type Output = V2; fn add(self, o: f32) -> V2 { V2(self.0 + o, self.1 + o) } }
fn sum_all<T: Add<Output = T> + Default + Copy>(xs: &[T]) -> T { xs.iter().fold(T::default(), |a, &b| a + b) }
fn main() {
    println!("{:?} {:?}", V2(1.0, 2.0) + V2(1.0, 1.0), V2(1.0, 2.0) + 0.5);
    println!("{} {}", sum_all(&[1, 2, 3]), sum_all(&[0.5, 0.25]));
}
```

✓ Output `V2(2.0, 3.0) V2(1.5, 2.5)` and `6 0.75`. The `Output` is an **associated type** (Lesson 20 explains why it is an associated type rather than another parameter).

---

## 8. Common mistakes and edge cases

- **Forgetting a bound** (E0369/E0599 ✓) or **over-bounding** (requiring `Clone` when `&T` would do). State the weakest bounds the body needs.
- **Putting bounds on the struct** instead of the `impl`. It forces every user of the type to carry the bound.
- **`impl Trait` arguments when you need the turbofish or two equal types** (E0107 ✓). Use a named parameter.
- **Forgetting `?Sized`** when you only take `&T` and want `str`/slices (E0277 ✓).
- **Trait not imported** (E0599 ✓, "implemented but not in scope").
- **Ambiguous methods** from two traits (E0034 ✓): use `<T as Trait>::method`.
- **Overlapping impls** (E0119 ✓), usually from a blanket impl plus a specific one.
- **Unused or unconstrained parameters** (E0392, E0207 ✓). Use `PhantomData` or remove them.
- **`derive` on generic types adds bounds** (`#[derive(Clone)] struct S<T>` requires `T: Clone`), including for marker-only parameters (Lesson 15's `Id<T>` problem).
- **Cannot infer** (E0790/E0283 ✓): annotate or turbofish.
- **Mixing types in one generic slice/Vec** (E0308 ✓): use an enum or `dyn`.
- **Expecting generic functions to be dyn-callable**: a function with its own type parameters makes a trait not object safe (Lesson 21).
- **Monomorphization bloat**: many instantiations of a big generic function.
- **Numeric generics**: `T: Add<Output = T>` says nothing about overflow; for `i8`, `a + b` can panic (debug) or wrap (release). Use a checked trait when inputs are untrusted (the practical challenge's `CheckedNum`).
- **Const-generic arrays on the stack** can overflow it.

---

## 9. Performance implications

- Generics are zero-cost at run time in the usual sense: the call is direct and inlinable, and there is no boxing. But the cost moves to compile time and binary size (Section 6), which I did not measure here.
- `dyn Trait` adds an indirect call per method and prevents inlining across it; a hot loop calling a `dyn` method per element is usually slower than the generic version, and that difference is the thing to *measure* (Lesson 41), not assume.
- Prefer the non-generic inner function pattern for large bodies behind small generic wrappers (`AsRef<Path>`, `Into<String>`, `impl Fn`).
- Generic code over `T: Copy` small types passes by value (registers); over large types it moves memory (`memcpy`); take `&T` when `T` may be large.
- `impl Trait` in return position keeps static dispatch (Lesson 17's iterator chains); `Box<dyn Trait>` returns allocate.
- Const generics let the compiler unroll and vectorize fixed-size loops, and avoid heap allocation for fixed-size data. Whether it does is for the optimizer and the benchmark to say.

---

## 10. Security and reliability

- **Bounds are contracts, not proofs.** `T: Ord` promises a total order, but a hand-written `Ord` can be inconsistent; sorting and tree structures may then panic or behave unpredictably (Lesson 16) while remaining memory safe. Treat user-supplied trait impls as untrusted logic.
- **Generic numeric code and overflow.** A function `fn add<T: Add<Output = T>>` inherits each type's overflow behavior. For amounts, balances and sizes, require checked operations (the `CheckedNum` trait in the challenge returns `None`; ✓ a matrix product of `i8` values overflowed to `None` instead of wrapping) or use wider types.
- **Validated types stay validated.** Generic containers like `SortedVec<T>` keep their invariant only because the field is private and there is no `DerefMut` (Lesson 15). A generic wrapper that exposes `&mut T` or `&mut Vec<T>` can break its own invariant.
- **Blanket impls widen your API surface.** `impl<T: Display> Describe for T` applies to types you have never seen, including foreign ones. Check that this is intended, especially for traits used in security decisions (authorization, serialization of secrets): a blanket "serialize everything that is `Debug`" can leak fields.
- **Sealing** (Lesson 15) keeps downstream crates from adding impls to a trait whose soundness relies on a closed set of types.
- **Trait-bound confusion** (`dyn Trait` vs generic) can hide runtime failures: downcasting and `Any` move checks to run time; prefer generics and enums where the set of types is known.
- **Compile-time dimension checks** (const generics) turn a class of runtime index/shape bugs into compile errors, and arrays are bounds-checked at run time anyway (Lesson 05).
- **Unbounded monomorphization from untrusted input is not possible** (types are fixed at compile time), but a generic function that allocates based on a *value* parameter still needs size limits (Lesson 16).

---

## 11. Exercises

Try each before opening the answer. They are not marked complete unless you tell me you did them.

1. **(Easy)** Make `fn sum<T>(a: T, b: T) -> T { a + b }` compile for any type that supports `+`, and explain the difference between E0369 and E0277 in this context.
2. **(Easy)** Write `largest<T: PartialOrd>(&[T]) -> Option<&T>` that returns `None` for an empty slice.
3. **(Medium)** Given `struct Pair<T> { x: T, y: T }`, add `new` for every `T`, and a method `cmp_display` that exists only when `T: Display + PartialOrd` and says which of `x`/`y` is larger.
4. **(Medium)** `impl<T: Display> Describe for T {}` exists. Why does `impl Describe for i32 {}` conflict, and give two ways forward.
5. **(Medium)** Write `BoundedStack<T, const N: usize>` whose `push` returns `Err(item)` when full instead of growing. What happens with `N = 0`?
6. **(Medium)** Predict how many copies of `double_it` the compiler generates for the calls `double_it(1i32)`, `double_it(1.5f64)`, `double_it(7u8)`, `double_it(2i32)`. How could you check?
7. **(Hard)** Write `fn show<T: Debug>(x: &T)` that also accepts `"text"` (a `&str`) and `&[1, 2][..]` (a slice). What bound do you change, and why?
8. **(Hard)** Write a generic `average` returning `Option<f64>` for any slice of `Copy` values convertible to `f64`. What does the first naive attempt `fn average<T>(xs: &[T]) -> T` report?

<details markdown="1">
<summary>Answers</summary>

1. ✓ `fn sum<T: std::ops::Add<Output = T>>(a: T, b: T) -> T { a + b }`; `sum(1, 2)` is `3` and `sum(1.5, 2.0)` is `3.5`. **E0369** ("cannot add `T` to `T`") is reported on the *generic body*: it asks for `+` but `T` has no such bound. **E0277** is reported at a *call site* when the argument type does not satisfy the bounds (for example `show(NoDisplay)` against `T: Display`).
2. ✓ (run: `largest(&[3, 9, 2])` is `Some(9)`, `largest(&["b", "a"])` is `Some("b")`, `largest::<f64>(&[])` is `None`)
   ```rust
   fn largest<T: PartialOrd>(items: &[T]) -> Option<&T> {
       let mut it = items.iter();
       let mut best = it.next()?;
       for x in it { if x > best { best = x; } }
       Some(best)
   }
   ```
   `PartialOrd` (not `Ord`) allows `f64`. `NaN` comparisons are all false, so a `NaN` first element would never be replaced; if that matters, handle it explicitly. The `?` on `it.next()` returns `None` for an empty slice.
3. ✓ (run: `Pair::new(3, 8).cmp_display()` printed `largest is y = 8`; with `"b", "a"` it printed `largest is x = b`)
   ```rust
   use std::fmt::Display;
   struct Pair<T> { x: T, y: T }
   impl<T> Pair<T> { fn new(x: T, y: T) -> Self { Pair { x, y } } }
   impl<T: Display + PartialOrd> Pair<T> {
       fn cmp_display(&self) -> String {
           if self.x >= self.y { format!("largest is x = {}", self.x) } else { format!("largest is y = {}", self.y) }
       }
   }
   ```
4. ✓ The blanket impl already covers `i32` (because `i32: Display`), and coherence allows only one impl per type and trait: `E0119` ("conflicting implementations"). Ways forward: (a) drop the blanket impl and implement `Describe` for the specific types you want; (b) make the blanket impl cover less (a narrower bound or a separate helper trait) and give the special case a different trait; (c) wrap the type in a newtype (Lesson 15) and implement the specific behavior on the wrapper. Specialization would allow the specific impl to override the blanket one but is not stable. (Design answer; the error itself was run.)
5. ✓ (run)
   ```rust
   struct BoundedStack<T, const N: usize> { items: Vec<T> }
   impl<T, const N: usize> BoundedStack<T, N> {
       fn new() -> Self { BoundedStack { items: Vec::with_capacity(N) } }
       fn push(&mut self, x: T) -> Result<(), T> {
           if self.items.len() >= N { Err(x) } else { self.items.push(x); Ok(()) }
       }
       fn pop(&mut self) -> Option<T> { self.items.pop() }
   }
   ```
   With `N = 2`: two pushes are `Ok(())`, the third is `Err("c")` (the rejected item is handed back, as in Lesson 15's failed transitions), then after a `pop` a push succeeds. With `N = 0`, `with_capacity(0)` doesn't allocate and every `push` returns `Err` ✓ (`BoundedStack<u8, 0>` was created and printed empty). The capacity is part of the type, so `BoundedStack<_, 2>` and `BoundedStack<_, 3>` are different types.
6. ✓ **Three**: one each for `i32`, `f64` and `u8`. The second `i32` call reuses the `i32` copy. Check with `nm -C` on a debug build (I saw `double_it::<f64>`, `::<u8>` and `::<i32>`), or `cargo asm`/`--emit asm`. Beware that an optimized build may inline or constant-fold copies away (✓ it did for me until I used `black_box`).
7. ✓ Add `?Sized`: `fn show<T: Debug + ?Sized>(x: &T)`. Without it, `T` must be `Sized`, and `str`/`[i32]` are not (`E0277` "the size for values of type `str` cannot be known at compilation time"). With `?Sized`, `show("a str")`, `show(&[1, 2][..])` and `show(&5)` all work (✓ `"a str"`, `[1, 2]`, `5`), which is allowed because `T` is only used behind a reference.
8. ✓ (run: `average(&[1, 2, 4])` is `Some(2.3333333333333335)`, `average(&[1.5f32, 2.5])` is `Some(2.0)`, `average::<u8>(&[])` is `None`)
   ```rust
   fn average<T: Copy + Into<f64>>(xs: &[T]) -> Option<f64> {
       if xs.is_empty() { return None; }
       let total: f64 = xs.iter().map(|&x| x.into()).sum();
       Some(total / xs.len() as f64)
   }
   ```
   `Into<f64>` exists for `u8`, `u16`, `u32`, `i8`, `i16`, `i32`, `f32` (lossless conversions) but **not** for `i64`/`u64` (they cannot always be represented exactly), so those would not compile with this signature. The naive version is the debugging challenge below.

</details>

---

## 12. Debugging challenge

A teammate writes a generic `average`:

```rust
fn average<T>(xs: &[T]) -> T {
    let mut total = T::default();
    for x in xs { total = total + *x; }
    total / xs.len()
}
fn main() { println!("{}", average(&[1, 2, 3])); }
```

It does not compile. Fix it, step by step, and explain what each error is telling you. (The compiler reports the errors in stages.)

<details markdown="1">
<summary>Solution</summary>

**Stage 1.** ✓ `error[E0599]: no associated function or constant named `default` found for type parameter `T``: the body calls `T::default()`, but `T` has no `Default` bound. Add `T: Default`.

**Stage 2.** ✓ With only `T: Default`: `error[E0369]: cannot add `T` to `T`` and `error[E0369]: cannot divide `T` by `usize``. The body uses `+` and `/` and the type doesn't promise them. Add `T: Add<Output = T>`; for the division the real problem is deeper:

**Stage 3.** ✓ With `T: Default + Add<Output = T> + Copy + Div<Output = T>`: `error[E0308]: mismatched types`. `Div<Output = T>` means `T / T`, but the code divides by a `usize` (`xs.len()`). There is no generic way to turn a length into a `T`. (Also, `*x` moves out of a borrowed slice unless `T: Copy`.) Adding more and more operator bounds is the wrong direction: the function mixes two different kinds of numbers.

**The fix is to change the design, not just the bounds.** Compute in `f64`, which every common numeric type can convert to, and return `Option` because the average of nothing is undefined (a division by zero would otherwise yield `NaN`):

```rust
fn average<T: Copy + Into<f64>>(xs: &[T]) -> Option<f64> {
    if xs.is_empty() { return None; }
    let total: f64 = xs.iter().map(|&x| x.into()).sum();
    Some(total / xs.len() as f64)
}
```

✓ `average(&[1, 2, 4])` is `Some(2.3333333333333335)`, `average(&[1.5f32, 2.5])` is `Some(2.0)`, `average::<u8>(&[])` is `None`. Lesson: errors in generic bodies mean "the bounds don't say you can do this"; sometimes the answer is a bound, and sometimes it is a different signature. The `Into<f64>` bound documents exactly which conversion you rely on. (If you must support `i64`/`u64`, accept the precision loss explicitly with `as f64` on a concrete type, or use a decimal/big-number type for money.)

</details>

---

## 13. Practical challenge: `genlab`

Create a library crate `genlab` (edition 2024) with:

1. Traits `Zero` and `One` (associated consts) and `CheckedNum` (`checked_add`, `checked_mul`), implemented with a `macro_rules!` for the integer types (`Zero`, `One`, `CheckedNum`) and for `f32`/`f64` (`Zero`, `One` only).
2. `Matrix<T, const ROWS: usize, const COLS: usize>` with: `new`, `rows`, `cols`, `get(row, col) -> Option<&T>`, `transpose` (`R x C` to `C x R`), `zeros`, `identity` (only for square matrices), `Add` for equal dimensions, `Mul` for `(R x C) * (C x K) = (R x K)`, and `try_mul` that returns `None` on overflow for `CheckedNum` types. A dimension mismatch in `*` must be a compile error (a `compile_fail` doctest).
3. `SortedVec<T: Ord>` that is always sorted: `new`, `from_vec`, `insert` (after equal elements, returns the index), `contains` (binary search), `remove`, `into_vec`, `Default`, `FromIterator`, and read-only access to slice methods through `Deref` (no `DerefMut`).
4. A blanket-impl trait `Describe` for every `T: Display + ?Sized`.
5. `largest_by_key<T, K: Ord>(&[T], impl Fn(&T) -> K) -> Option<&T>` returning the **first** of equal maxima.
6. Tests: identity is neutral (ints and floats); a rectangular product with a known result; transpose shape and round trip; `Add` and `zeros`; `0`-row matrices; `try_mul` overflow on `i8` and `u64`; `SortedVec` insert positions, removal, `FromIterator`, and a **model test** against a sorted `Vec` over thousands of pseudo-random operations; `Describe` on `i32`, `str`, `String`, `f64` and `Box<dyn Display>`; `largest_by_key` ties and empty input. A usage doctest.
7. It must pass `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, and `cargo test --locked` (run `cargo generate-lockfile` once first).

<details markdown="1">
<summary>Solution</summary>

**`src/lib.rs`**

```rust
//! Generic building blocks: a const-generic `Matrix`, a `SortedVec`, a blanket-impl
//! trait, and a generic "largest by key" helper.
//!
//! ```
//! use genlab::{Matrix, SortedVec};
//!
//! let a = Matrix::new([[1, 2, 3], [4, 5, 6]]); // 2x3
//! let b = a.transpose();                       // 3x2
//! let c = a * b;                               // (2x3) * (3x2) = 2x2
//! assert_eq!(c, Matrix::new([[14, 32], [32, 77]]));
//!
//! let mut s = SortedVec::from_vec(vec![5, 1, 3]);
//! s.insert(4);
//! assert_eq!(&*s, &[1, 3, 4, 5]);
//! ```
//!
//! Multiplying matrices whose inner dimensions disagree does not compile:
//!
//! ```compile_fail
//! use genlab::Matrix;
//! let a = Matrix::new([[1, 2, 3], [4, 5, 6]]); // 2x3
//! let b = Matrix::new([[1, 2], [3, 4]]);       // 2x2
//! let _ = a * b;                               // needs a 3-row right-hand side
//! ```

use std::array;
use std::fmt::Display;
use std::ops::{Add, Deref, Mul};

/// Types with an additive identity.
pub trait Zero {
    /// The value `0`.
    const ZERO: Self;
}

/// Types with a multiplicative identity.
pub trait One {
    /// The value `1`.
    const ONE: Self;
}

/// Integer-like types whose arithmetic can be checked for overflow.
pub trait CheckedNum: Copy {
    /// `self + rhs`, or `None` on overflow.
    fn checked_add(self, rhs: Self) -> Option<Self>;
    /// `self * rhs`, or `None` on overflow.
    fn checked_mul(self, rhs: Self) -> Option<Self>;
}

macro_rules! impl_int {
    ($($t:ty),* $(,)?) => {$(
        impl Zero for $t { const ZERO: Self = 0; }
        impl One for $t { const ONE: Self = 1; }
        impl CheckedNum for $t {
            fn checked_add(self, rhs: Self) -> Option<Self> { <$t>::checked_add(self, rhs) }
            fn checked_mul(self, rhs: Self) -> Option<Self> { <$t>::checked_mul(self, rhs) }
        }
    )*};
}
impl_int!(i8, i16, i32, i64, u8, u16, u32, u64, usize, isize);

macro_rules! impl_float {
    ($($t:ty),* $(,)?) => {$(
        impl Zero for $t { const ZERO: Self = 0.0; }
        impl One for $t { const ONE: Self = 1.0; }
    )*};
}
impl_float!(f32, f64);

/// A `ROWS x COLS` matrix whose dimensions are part of its type.
#[derive(Debug, Clone, Copy, PartialEq)]
pub struct Matrix<T, const ROWS: usize, const COLS: usize> {
    data: [[T; COLS]; ROWS],
}

impl<T, const R: usize, const C: usize> Matrix<T, R, C> {
    /// Wraps an array of rows.
    pub fn new(data: [[T; C]; R]) -> Self {
        Matrix { data }
    }

    /// Number of rows (a compile-time constant).
    pub const fn rows(&self) -> usize {
        R
    }

    /// Number of columns (a compile-time constant).
    pub const fn cols(&self) -> usize {
        C
    }

    /// The element at `(row, col)`, or `None` when out of range.
    pub fn get(&self, row: usize, col: usize) -> Option<&T> {
        self.data.get(row)?.get(col)
    }
}

impl<T: Copy, const R: usize, const C: usize> Matrix<T, R, C> {
    /// Swaps rows and columns: an `R x C` matrix becomes `C x R`.
    pub fn transpose(&self) -> Matrix<T, C, R> {
        Matrix {
            data: array::from_fn(|c| array::from_fn(|r| self.data[r][c])),
        }
    }
}

impl<T: Zero + Copy, const R: usize, const C: usize> Matrix<T, R, C> {
    /// A matrix of zeros.
    pub fn zeros() -> Self {
        Matrix {
            data: [[T::ZERO; C]; R],
        }
    }
}

impl<T: Zero + One + Copy, const N: usize> Matrix<T, N, N> {
    /// The `N x N` identity matrix (only exists for square matrices).
    pub fn identity() -> Self {
        Matrix {
            data: array::from_fn(|r| array::from_fn(|c| if r == c { T::ONE } else { T::ZERO })),
        }
    }
}

impl<T: Zero + CheckedNum, const R: usize, const C: usize> Matrix<T, R, C> {
    /// Matrix product that returns `None` instead of overflowing.
    pub fn try_mul<const K: usize>(&self, rhs: &Matrix<T, C, K>) -> Option<Matrix<T, R, K>> {
        let mut out = [[T::ZERO; K]; R];
        for (r, out_row) in out.iter_mut().enumerate() {
            for (c, cell) in out_row.iter_mut().enumerate() {
                let mut acc = T::ZERO;
                for k in 0..C {
                    let term = CheckedNum::checked_mul(self.data[r][k], rhs.data[k][c])?;
                    acc = CheckedNum::checked_add(acc, term)?;
                }
                *cell = acc;
            }
        }
        Some(Matrix { data: out })
    }
}

impl<T: Copy + Add<Output = T>, const R: usize, const C: usize> Add for Matrix<T, R, C> {
    type Output = Matrix<T, R, C>;

    fn add(self, rhs: Self) -> Self::Output {
        Matrix {
            data: array::from_fn(|r| array::from_fn(|c| self.data[r][c] + rhs.data[r][c])),
        }
    }
}

impl<T, const R: usize, const C: usize, const K: usize> Mul<Matrix<T, C, K>> for Matrix<T, R, C>
where
    T: Zero + Copy + Add<Output = T> + Mul<Output = T>,
{
    type Output = Matrix<T, R, K>;

    fn mul(self, rhs: Matrix<T, C, K>) -> Self::Output {
        let mut out = [[T::ZERO; K]; R];
        for (r, out_row) in out.iter_mut().enumerate() {
            for (c, cell) in out_row.iter_mut().enumerate() {
                let mut acc = T::ZERO;
                for k in 0..C {
                    acc = acc + self.data[r][k] * rhs.data[k][c];
                }
                *cell = acc;
            }
        }
        Matrix { data: out }
    }
}

/// A `Vec` that is always sorted. The inner `Vec` is private and there is no
/// `DerefMut`, so callers cannot break the ordering.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct SortedVec<T>(Vec<T>);

impl<T: Ord> SortedVec<T> {
    /// An empty sorted vector.
    pub fn new() -> Self {
        SortedVec(Vec::new())
    }

    /// Sorts `v` and wraps it.
    pub fn from_vec(mut v: Vec<T>) -> Self {
        v.sort();
        SortedVec(v)
    }

    /// Inserts `value` after any equal elements and returns its index.
    pub fn insert(&mut self, value: T) -> usize {
        let index = self.0.partition_point(|x| *x <= value);
        self.0.insert(index, value);
        index
    }

    /// Binary-search membership test, O(log n).
    pub fn contains(&self, value: &T) -> bool {
        self.0.binary_search(value).is_ok()
    }

    /// Removes one element equal to `value`; returns whether one was found.
    pub fn remove(&mut self, value: &T) -> bool {
        match self.0.binary_search(value) {
            Ok(index) => {
                self.0.remove(index);
                true
            }
            Err(_) => false,
        }
    }

    /// Gives back the sorted `Vec`.
    pub fn into_vec(self) -> Vec<T> {
        self.0
    }
}

impl<T: Ord> Default for SortedVec<T> {
    fn default() -> Self {
        Self::new()
    }
}

impl<T> Deref for SortedVec<T> {
    type Target = [T];

    fn deref(&self) -> &[T] {
        &self.0
    }
}

impl<T: Ord> FromIterator<T> for SortedVec<T> {
    fn from_iter<I: IntoIterator<Item = T>>(iter: I) -> Self {
        SortedVec::from_vec(iter.into_iter().collect())
    }
}

/// A one-line description, available for every `Display` type (including `str`).
pub trait Describe {
    /// Returns `<text>` for the value's `Display` form.
    fn describe(&self) -> String;
}

impl<T: Display + ?Sized> Describe for T {
    fn describe(&self) -> String {
        format!("<{self}>")
    }
}

/// The first element whose key is largest, or `None` for an empty slice.
pub fn largest_by_key<T, K: Ord>(items: &[T], key: impl Fn(&T) -> K) -> Option<&T> {
    let mut iter = items.iter();
    let mut best = iter.next()?;
    let mut best_key = key(best);
    for item in iter {
        let k = key(item);
        if k > best_key {
            best = item;
            best_key = k;
        }
    }
    Some(best)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn identity_is_neutral_for_multiplication() {
        let m = Matrix::new([[1, 2], [3, 4]]);
        assert_eq!(m * Matrix::identity(), m);
        assert_eq!(Matrix::identity() * m, m);
        let f = Matrix::new([[1.5, 0.0], [2.0, -1.0]]);
        assert_eq!(f * Matrix::identity(), f);
    }

    #[test]
    fn product_of_rectangular_matrices() {
        let a = Matrix::new([[1, 2, 3], [4, 5, 6]]);
        let b = Matrix::new([[7, 8], [9, 10], [11, 12]]);
        let c: Matrix<i32, 2, 2> = a * b;
        assert_eq!(c, Matrix::new([[58, 64], [139, 154]]));
        assert_eq!((c.rows(), c.cols()), (2, 2));
        let a_t_a: Matrix<i32, 3, 3> = a.transpose() * a;
        assert_eq!(a_t_a.get(0, 0), Some(&17));
        assert_eq!(a_t_a.get(3, 0), None);
    }

    #[test]
    fn transpose_swaps_dimensions_and_roundtrips() {
        let a = Matrix::new([[1, 2, 3], [4, 5, 6]]);
        let t = a.transpose();
        assert_eq!((t.rows(), t.cols()), (3, 2));
        assert_eq!(t, Matrix::new([[1, 4], [2, 5], [3, 6]]));
        assert_eq!(t.transpose(), a);
    }

    #[test]
    fn add_and_zeros() {
        let a = Matrix::new([[1, 2], [3, 4]]);
        assert_eq!(a + Matrix::zeros(), a);
        assert_eq!(a + a, Matrix::new([[2, 4], [6, 8]]));
        let empty: Matrix<u8, 0, 3> = Matrix::zeros();
        assert_eq!(empty.rows(), 0);
    }

    #[test]
    fn try_mul_reports_overflow_instead_of_wrapping() {
        let a = Matrix::new([[100i8, 100]]);
        let b = Matrix::new([[1i8], [1]]);
        assert_eq!(a.try_mul(&b), None, "100 + 100 does not fit in i8");
        let small = Matrix::new([[10i8, 10]]);
        assert_eq!(small.try_mul(&b), Some(Matrix::new([[20i8]])));
        let wide = Matrix::new([[100i32, 100]]);
        let one = Matrix::new([[1i32], [1]]);
        assert_eq!(wide.try_mul(&one), Some(Matrix::new([[200]])));
        let big = Matrix::new([[u64::MAX]]);
        assert_eq!(big.try_mul(&Matrix::new([[2u64]])), None);
    }

    #[test]
    fn sorted_vec_keeps_order_on_every_operation() {
        let mut s = SortedVec::from_vec(vec![5, 1, 3]);
        assert_eq!(&*s, &[1, 3, 5]);
        assert_eq!(s.insert(4), 2);
        assert_eq!(s.insert(3), 2, "inserted after the existing 3");
        assert_eq!(&*s, &[1, 3, 3, 4, 5]);
        assert!(s.contains(&4));
        assert!(!s.contains(&2));
        assert!(s.remove(&3));
        assert!(!s.remove(&9));
        assert_eq!(s.len(), 4);
        assert_eq!(s.first(), Some(&1));
        let collected: SortedVec<_> = "hello".chars().collect();
        assert_eq!(collected.into_vec(), vec!['e', 'h', 'l', 'l', 'o']);
        assert!(SortedVec::<i32>::default().is_empty());
    }

    #[test]
    fn sorted_vec_matches_a_sorted_vec_model() {
        let mut state = 7u64;
        let mut next = move || {
            state = state
                .wrapping_mul(6364136223846793005)
                .wrapping_add(1442695040888963407);
            (state >> 33) as u32 % 20
        };
        let mut s = SortedVec::new();
        let mut model: Vec<u32> = Vec::new();
        for _ in 0..2000 {
            let x = next();
            if next() % 3 == 0 {
                let removed = s.remove(&x);
                let expected = model
                    .iter()
                    .position(|y| *y == x)
                    .map(|i| model.remove(i))
                    .is_some();
                assert_eq!(removed, expected);
            } else {
                s.insert(x);
                model.push(x);
                model.sort();
            }
            assert_eq!(&*s, model.as_slice());
        }
    }

    #[test]
    fn blanket_describe_covers_sized_and_unsized_display_types() {
        assert_eq!(5.describe(), "<5>");
        assert_eq!("str".describe(), "<str>");
        assert_eq!(String::from("s").describe(), "<s>");
        assert_eq!(2.5f64.describe(), "<2.5>");
        let boxed: Box<dyn Display> = Box::new('c');
        assert_eq!(boxed.describe(), "<c>");
    }

    #[test]
    fn largest_by_key_returns_the_first_of_equals() {
        let words = ["bb", "a", "cc", "d"];
        assert_eq!(largest_by_key(&words, |w| w.len()), Some(&"bb"));
        assert_eq!(largest_by_key(&[3, 9, 9, 1], |x| *x), Some(&9));
        assert_eq!(largest_by_key::<i32, i32>(&[], |x| *x), None);
        let pairs = [(1, 'a'), (3, 'b'), (3, 'c')];
        assert_eq!(largest_by_key(&pairs, |p| p.0), Some(&(3, 'b')));
    }
}
```

✓ With this exact file: `cargo fmt --check` clean, `cargo clippy --all-targets --locked -- -D warnings` clean, `cargo test --locked` → 9 unit tests, 1 passing doctest and 1 `compile_fail` doctest passed. In a separate binary crate I confirmed ✓ that multiplying a `2x3` by a `2x2` matrix gives `error[E0308]: mismatched types` ("expected `3`, found `2`"): the `compile_fail` doctest only proves that *some* error occurs, the external crate shows which.

**Design notes.**

- **Dimensions are types.** `Matrix<T, R, C>` stores `[[T; C]; R]`; the `Mul` impl is generic over `R`, `C` and `K` and is only defined for `Matrix<T, R, C> * Matrix<T, C, K>`, so the inner dimensions must match at compile time. `identity` lives in an `impl<T, const N: usize> Matrix<T, N, N>` block, so it does not even exist for non-square matrices.
- **Bounds on the `impl` blocks, not the struct** (Section 2.3): `Matrix` itself has no bounds; `transpose` needs `T: Copy`, `zeros` needs `Zero + Copy`, `Mul` needs `Zero + Copy + Add + Mul`, `try_mul` needs `Zero + CheckedNum`. Each method asks for the weakest bounds it uses.
- **`array::from_fn`** builds arrays without an initial value, so `transpose`, `identity` and `Add` need no `Default`.
- **Overflow policy is explicit.** `Mul`/`Add` use the type's operators (they panic in debug builds and wrap in release on overflowing integers); `try_mul` uses `CheckedNum` and returns `None`. Floats don't implement `CheckedNum`, so `try_mul` simply does not exist for them.
- **The macro** keeps ten integer impls to a few lines (macros are Lesson 34). Inside the macro, `<$t>::checked_add(self, rhs)` calls the inherent method on the primitive type, not the trait method (an inherent method wins), which avoids infinite recursion.
- **`SortedVec`** holds its `Vec` privately and offers only `Deref` to `[T]` (read-only), so the ordering invariant cannot be broken from outside (the lesson of Lesson 15's `Leaky` example). `insert` uses `partition_point` (`x <= value`) so equal elements keep insertion order.
- **Blanket `Describe`** uses `?Sized` so it applies to `str`, which is why `"str".describe()` works. It is only a demonstration; a real library would think twice before implementing a trait for every `Display` type (Section 10).
- **Not a numerics library.** There is no `Sub`, no scalar multiplication, no inversion, no generic floating-point error handling, and large matrices live on the stack.

</details>

---

## 14. Interview questions

<details markdown="1">
<summary>Answers</summary>

1. **What are generics and why does Rust have them?** Type (and const) parameters that let one definition work for many types while being fully type-checked, with no run-time cost.
2. **What is monomorphization?** The compiler generates a specialized copy of a generic function or type for each concrete set of type arguments it is used with, so calls are direct and inlinable. Trade-off: more compile time and code size.
3. **How do Rust generics differ from C++ templates?** Rust checks the generic body once against its declared bounds (errors at the definition); C++ templates are checked at instantiation. Rust requires explicit bounds.
4. **What is a trait bound?** A requirement on a type parameter (`T: Display + Clone`) that both limits callers and grants the body access to those methods.
5. **Inline bounds vs `where` clause vs `impl Trait`?** Equivalent for simple cases; `where` is clearer for long or multi-parameter bounds and associated-type constraints; `impl Trait` in argument position is sugar for an anonymous parameter (no turbofish, two arguments can have different types).
6. **What do E0369, E0599 and E0277 mean in generic code?** E0369/E0599: the body uses an operator or method the bounds do not provide. E0277: an argument does not satisfy the bounds.
7. **What is `Sized` and when do you write `?Sized`?** All type parameters are `Sized` by default; `?Sized` relaxes that so you can accept `str`, `[T]` or `dyn Trait` behind a pointer.
8. **Why put bounds on `impl` blocks rather than structs?** The type can exist for any `T`, methods appear conditionally, and users are not forced to repeat bounds.
9. **What is a blanket impl, and what is the catch?** An impl for all types matching a bound (`impl<T: Display> ToString for T`). It restricts other impls: overlapping ones are rejected (E0119).
10. **Generics vs trait objects?** Generics: static dispatch, one copy per type, inlining, homogeneous. `dyn Trait`: dynamic dispatch through a vtable, one copy, heterogeneous collections, requires object safety.
11. **What are const generics?** Generic parameters that are values (integers, `bool`, `char`), mainly array lengths. They let the type system check sizes (`Matrix<2, 3> * Matrix<3, 4>`).
12. **What is the turbofish and when is it needed?** `::<T>` supplies generic arguments explicitly (`"7".parse::<u8>()`, `Vec::<i32>::new()`), needed when inference has nothing to go on (E0790/E0283).
13. **How do you call a specific trait's method when names collide?** Fully qualified syntax: `<Type as Trait>::method(&x)` or `Trait::method(&x)`.
14. **What is a supertrait?** A trait that requires another (`trait Named: Display`), letting default methods use the supertrait's methods.
15. **Why is `T: Add<Output = T>` not enough for safe arithmetic?** It says nothing about overflow; integer `+` can panic (debug) or wrap (release). Use checked traits or wider types for untrusted values.
16. **How can generics bloat a binary and how do you mitigate it?** Each instantiation is a separate copy. Keep generic wrappers thin and delegate to a non-generic inner function, or use `dyn Trait` for cold code.
17. **What are E0207 and E0392?** An impl type parameter that is not constrained by the impl header (E0207), and a struct type parameter that is never used (E0392; use `PhantomData`).
18. **Can `derive` on a generic struct surprise you?** Yes, it adds `T: Trait` bounds on the generated impl, even if `T` is only a marker (`Id<T>` not `Copy` when `T` isn't).

</details>

---

## 15. Key takeaways

- Generics let you write code once for many types; Rust checks it once against the bounds you declare, and monomorphizes it into specialized copies, which is why it is fast and why binaries can grow.
- Bounds are the contract. Errors in the body (E0369/E0599) mean "your bounds don't promise that"; errors at the call (E0277) mean "your argument doesn't meet the bounds". Read the `help:` lines.
- Use inline bounds for short ones, `where` for long ones, and `impl Trait` arguments for brevity (no turbofish). Add `?Sized` when you only use `T` behind a reference.
- Put bounds on `impl` blocks so types exist broadly and methods appear conditionally. Traits provide defaults, supertraits and associated constants; bring them into scope with `use`, and disambiguate with `<T as Trait>::`.
- Blanket impls are powerful but conflict with every overlapping impl (E0119). Unused or unconstrained parameters are errors (E0392, E0207); use `PhantomData` for markers.
- Static dispatch (generics) is inlinable and homogeneous; dynamic dispatch (`dyn Trait`) is flexible and indirect. Default to generics, use `dyn` for heterogeneity or to reduce instantiations.
- Const generics put sizes in the type, turning shape mismatches into compile errors at no run-time cost; operator traits are generic too, and generic arithmetic needs an overflow policy.

**Connections.** Lesson 13's `impl` blocks and derives, Lesson 15's newtypes, `PhantomData` and sealed traits, Lesson 16's `K: Hash + Eq`, Lesson 17's `Iterator` signatures and Lesson 18's `F: Fn(A) -> B` are all instances of this lesson. Next: Lesson 20 (associated types, coherence and the orphan rule, `Add`'s `Output`, `Iterator::Item`, why not a type parameter), Lesson 21 (trait objects, object safety, vtables), and Lesson 22 (GATs, `Send`/`Sync`, `PhantomData` and variance, dynamically sized types). Lesson 33 covers API design (bounds placement, sealed traits, `impl Into`), Lesson 34 covers `macro_rules!` (used for the integer impls here), Lesson 41 covers measuring the cost of generics, and Lesson 42 covers compile time and code size. See the [roadmap](00_roadmap.md) and the [course index](index.md).

---

## 16. References

- *The Rust Programming Language*, ch. 10 "Generic Types, Traits, and Lifetimes" (10.1 generic data types, 10.2 traits: defining shared behavior, including `impl Trait`, trait bounds and blanket implementations): <https://doc.rust-lang.org/book/ch10-00-generics.html>
- *The Rust Reference*, "Generic parameters", "Trait and lifetime bounds", "Implementations" (coherence), and "Const generics": <https://doc.rust-lang.org/reference/items/generics.html>, <https://doc.rust-lang.org/reference/trait-bounds.html>, <https://doc.rust-lang.org/reference/items/implementations.html>
- *The Rust Programming Language*, ch. 20.2 "Advanced Traits" (fully qualified syntax, supertraits, default generic type parameters, operator overloading): <https://doc.rust-lang.org/book/ch20-02-advanced-traits.html>
- `std::marker::Sized`: <https://doc.rust-lang.org/std/marker/trait.Sized.html>; `std::ops::Add`: <https://doc.rust-lang.org/std/ops/trait.Add.html>; `std::array::from_fn`: <https://doc.rust-lang.org/std/array/fn.from_fn.html>
- *Rust API Guidelines*: "Data structures do not duplicate derived trait bounds" (C-STRUCT-BOUNDS) and "Generics minimize assumptions about their parameters": <https://rust-lang.github.io/api-guidelines/>
- RFC 2000 "Const generics": <https://rust-lang.github.io/rfcs/2000-const-generics.html>
- Rust error code index: E0369, E0599, E0277, E0119, E0207, E0392, E0107, E0034, E0790: <https://doc.rust-lang.org/error_codes/E0369.html>, <https://doc.rust-lang.org/error_codes/E0119.html>, <https://doc.rust-lang.org/error_codes/E0207.html>
- *The Rustonomicon*, "Exotically Sized Types": <https://doc.rust-lang.org/nomicon/exotic-sizes.html>

The URLs above are the standard documentation locations; I did not open every one during this session (the sandbox cannot reach them), so if a link has moved, search the docs site for the page title. Unchecked from memory: the Rust Book chapter numbers and anchors, the API-guidelines item names, the RFC 2000 number, the claim that const generics were stabilized around Rust 1.51 and that `generic_const_exprs` is still unstable, that specialization is not stable, the list of numeric types with `Into<f64>`, and the statement that large instantiation counts are a common cause of big binaries and slow builds (I did not measure compile times or sizes).
