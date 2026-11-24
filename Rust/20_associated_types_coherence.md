---
layout: default
title: "Lesson 20 — Associated types, blanket impls, coherence, and the orphan rule"
---

# Lesson 20 — Associated types, blanket impls, coherence, and the orphan rule

**Module 7 · Generics and traits** · Status: generated · Edition used in examples: **2024** · External dependencies: none

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux. As before, ✓ marks outputs and diagnostics I actually ran; unmarked statements are documented behavior I did not run. Everything is **stable Rust** except where a section says otherwise.

Builds on [Lesson 13](13_structs_enums_methods.md) (impl blocks), [Lesson 14](14_option_result_errors.md) (`Result`, `FromStr`, `TryFrom`), [Lesson 15](15_newtype_typestate.md) (newtypes, the orphan rule E0117, sealed traits), [Lesson 17](17_iterators.md) (`Iterator::Item`, `IntoIterator`) and [Lesson 19](19_generics_and_bounds.md) (bounds, blanket impls, E0119, `Add<Output = T>`).

---

## 1. The question this lesson answers

Lesson 19 left three loose ends: why is `Add`'s result written `Output = T` instead of being another type parameter; why did a blanket impl make `impl Describe for i32` fail with E0119; and why can't you write `impl Display for Vec<i32>`? All three come from one design: **a trait can have associated types, and the compiler requires that for any (type, trait) pair there is at most one answer**. This lesson explains the rule and how to work with it.

---

## 2. Associated types

An **associated type** is a type placeholder declared inside a trait. Each implementation fills it in once.

```rust
trait Container {
    type Item;
    fn first(&self) -> Option<&Self::Item>;
    fn len(&self) -> usize;
}
struct Stack<T>(Vec<T>);
impl<T> Container for Stack<T> {
    type Item = T;
    fn first(&self) -> Option<&T> { self.0.first() }
    fn len(&self) -> usize { self.0.len() }
}
fn show_first<C: Container>(c: &C) -> String where C::Item: std::fmt::Debug {
    format!("{:?} of {}", c.first(), c.len())
}
fn main() {
    println!("{}", show_first(&Stack(vec![3, 4])));
    println!("{}", show_first(&Stack(Vec::<String>::new())));
}
```

✓ Output `Some(3) of 2` and `None of 0`. Inside the trait you name the placeholder `Self::Item`. In a generic function you name it through the type parameter: `C::Item`, which is shorthand for the fully qualified **projection** `<C as Container>::Item`. Note `show_first` has one type parameter, `C`, even though the trait involves two types (the container and its item): the item type is *determined by* `C`, so it does not need its own parameter.

### 2.1 Associated type or generic parameter?

You could have written `trait Container<T>` instead. The difference is how many implementations a type may have.

```rust
trait Conv<T> { fn conv(&self) -> T; }
struct S(u8);
impl Conv<u32> for S { fn conv(&self) -> u32 { self.0 as u32 } }
impl Conv<String> for S { fn conv(&self) -> String { format!("s{}", self.0) } }
fn main() {
    let a: u32 = S(1).conv();
    let b: String = S(2).conv();
    println!("{a} {b}");
}
```

✓ Output `1 s2`. A generic trait can be implemented **many times for the same type**, once per parameter value (this is how `From<A> for X` and `From<B> for X` coexist). The cost is that the caller must say which one, and when nothing says so ✓:

```rust
trait Conv<T> { fn conv(&self) -> T; }
struct S(u8);
impl Conv<u32> for S { fn conv(&self) -> u32 { self.0 as u32 } }
impl Conv<String> for S { fn conv(&self) -> String { format!("s{}", self.0) } }
fn main() {
    let c = S(3).conv();
}
```

✓ `error[E0283]: type annotations needed`. With an associated type there is **exactly one** implementation per type, so the item type is never ambiguous ✓:

```rust
struct S;
impl Iterator for S { type Item = u8; fn next(&mut self) -> Option<u8> { None } }
impl Iterator for S { type Item = u16; fn next(&mut self) -> Option<u16> { None } }
fn main() {}
```

✓ `error[E0119]: conflicting implementations of trait `Iterator` for type `S``. The associated type does not give you a second slot; it is *part of the answer* to "what is this type's `Iterator` impl".

| | Associated type (`trait T { type X; }`) | Generic parameter (`trait T<X>`) |
|---|---|---|
| Impls per `Self` | one | any number (one per `X`) |
| Caller must name `X` | no (it follows from `Self`) | often yes (E0283 when unclear) |
| Bound syntax | `T: Trait<X = u8>` or `T::X: Bound` | `T: Trait<u8>` |
| Typical meaning | "an output / component of this type" | "interoperates with this other type" |
| Std examples | `Iterator::Item`, `Deref::Target`, `FromStr::Err`, `TryFrom<T>::Error`, `Index<I>::Output`, `Add<Rhs>::Output`, `Future::Output` | `From<T>`, `PartialEq<Rhs>`, `Add<Rhs>`, `AsRef<T>`, `Index<Idx>` |

Notice that std mixes both in one trait: `Add<Rhs = Self>` has a **parameter** `Rhs` (a type can be added to several right-hand types) and an **associated type** `Output` (given `Self` and `Rhs`, the result is fixed). `Index<Idx>` and `TryFrom<T>` follow the same pattern. Rule of thumb: *if, given the implementing type (and the other parameters), there is only one sensible answer, make it an associated type.*

Reading the rule from the standard library's own traits ✓:

```rust
use std::{ops::Index, str::FromStr};
#[derive(Debug, PartialEq)]
struct Even(u32);
impl TryFrom<u32> for Even {
    type Error = String;
    fn try_from(v: u32) -> Result<Self, String> {
        if v % 2 == 0 { Ok(Even(v)) } else { Err(format!("{v} is odd")) }
    }
}
impl FromStr for Even {
    type Err = String;
    fn from_str(s: &str) -> Result<Self, String> {
        let n: u32 = s.parse().map_err(|e| format!("{e}"))?;
        Even::try_from(n)
    }
}
struct Grid { w: usize, d: Vec<u8> }
impl Index<(usize, usize)> for Grid {
    type Output = u8;
    fn index(&self, (r, c): (usize, usize)) -> &u8 { &self.d[r * self.w + c] }
}
fn main() {
    println!("{:?} {:?}", Even::try_from(4), Even::try_from(5));
    println!("{:?} {:?}", "8".parse::<Even>(), "x".parse::<Even>());
    let g = Grid { w: 2, d: vec![1, 2, 3, 4] };
    println!("{}", g[(1, 0)]);
}
```

✓ Output `Ok(Even(4)) Err("5 is odd")`, `Ok(Even(8)) Err("invalid digit found in string")`, `3`. (`s.parse()` is generic over `FromStr`, and `FromStr::Err` is what shows up in its `Result`; this is the connection to Lesson 14.) The `Grid` shows why `Index` needs the *parameter* (you could add `impl Index<usize> for Grid` too) while `Output` is associated. The `index` body uses `self.d[...]`, which panics when out of range (Lesson 16); a real grid would offer a `get` that returns `Option`.

A custom iterator is the same idea ✓:

```rust
struct Countdown(u32);
impl Iterator for Countdown {
    type Item = u32;
    fn next(&mut self) -> Option<u32> {
        if self.0 == 0 { None } else { self.0 -= 1; Some(self.0 + 1) }
    }
}
fn main() {
    for n in Countdown(3) { print!("{n} "); }
    println!("| {:?} {}", Countdown(4).collect::<Vec<_>>(), Countdown(5).map(u64::from).sum::<u64>());
}
```

✓ Output `3 2 1 | [4, 3, 2, 1] 15`. Implementing `Iterator` with one associated type and one method gives you every adapter from Lesson 17.

### 2.2 Declaring and implementing: the errors you will meet

**Forgetting the type** ✓:

```rust
trait T { type X; fn f(&self) -> Self::X; }
struct S;
impl T for S { fn f(&self) -> u8 { 1 } }
fn main() {}
```

✓ `error[E0046]: not all trait items implemented, missing: `X``. Add `type X = u8;`. The fixed version printed `1` when run ✓.

**Bounds on the associated type** are part of the trait's contract. Implementors must satisfy them, and generic code may rely on them ✓:

```rust
use std::fmt::Debug;
trait Graph {
    type Node: Debug + Clone + PartialEq;
    fn neighbors(&self, n: &Self::Node) -> Vec<Self::Node>;
}
struct Ring(u8);
impl Graph for Ring {
    type Node = u8;
    fn neighbors(&self, n: &u8) -> Vec<u8> { vec![(n + self.0 - 1) % self.0, (n + 1) % self.0] }
}
fn path_ok<G: Graph>(g: &G, a: G::Node, b: G::Node) -> bool { g.neighbors(&a).contains(&b) }
fn dbg_all<G: Graph>(g: &G, n: G::Node) -> String { format!("{:?}", g.neighbors(&n)) }
fn main() {
    let r = Ring(5);
    println!("{} {} {}", path_ok(&r, 0, 1), path_ok(&r, 0, 2), dbg_all(&r, 0));
}
```

✓ Output `true false [4, 1]`. `path_ok` can call `contains` (needs `PartialEq`) and `dbg_all` can format with `{:?}` only because `Node`'s declared bounds promise it. An implementor that violates the bound is rejected at the `impl` ✓:

```rust
use std::fmt::Debug;
trait Graph { type Node: Debug; }
struct NoDbg;
struct G;
impl Graph for G { type Node = NoDbg; }
fn main() {}
```

✓ `error[E0277]: `NoDbg` doesn't implement `Debug``. (The `Ring` arithmetic above uses `u8` and would overflow for huge rings; it is a demonstration, not a graph library.)

**Defaults** like `type X = u8;` inside a trait are *not* stable ✓: `error[E0658]: associated type defaults are unstable` (it is behind the `associated_type_defaults` feature, so you cannot use it on stable). Offer a default through a generic parameter (`Add<Rhs = Self>`) or a helper trait instead.

### 2.3 Using associated types in bounds

You can **pin** an associated type with `=` and you can **bound** it:

```rust
use std::fmt::Debug;
fn sum_u32<I: Iterator<Item = u32>>(it: I) -> u32 { it.sum() }
fn dump<I: Iterator<Item: Debug>>(it: I) -> String {
    it.map(|x| format!("{x:?}")).collect::<Vec<_>>().join(",")
}
fn dump2<I>(it: I) -> String where I: Iterator, I::Item: Debug {
    it.map(|x| format!("{x:?}")).collect::<Vec<_>>().join(",")
}
fn flatten_sum<I>(it: I) -> i64
where I: IntoIterator, I::Item: IntoIterator<Item = i64> {
    it.into_iter().flat_map(|inner| inner.into_iter()).sum()
}
fn main() {
    println!("{}", sum_u32([1u32, 2, 3].into_iter()));
    println!("{} {}", dump(["a", "b"].into_iter()), dump2(1..4));
    println!("{} {}", flatten_sum(vec![vec![1, 2], vec![3]]), flatten_sum([[10i64; 2], [5; 2]]));
}
```

✓ Output `6`, `"a","b" 1,2,3`, `6 30`. Three spellings are used: an equality constraint `Iterator<Item = u32>`, a bound on the projection in a `where` clause (`I::Item: Debug`), and the associated type bound shorthand `Iterator<Item: Debug>` (stable; it means the same as the `where` form). Passing an iterator of the wrong item type is an **E0271** ✓:

```rust
fn sum_u32<I: Iterator<Item = u32>>(it: I) -> u32 { it.sum() }
fn main() {
    println!("{}", sum_u32(vec![1u8].into_iter()));
}
```

✓ `error[E0271]: expected `IntoIter<u8>` to be an iterator that yields `u32`, but it yields `u8``. Compare with Lesson 19: **E0277** was "the type does not implement the trait"; **E0271** is "it does, but its associated type isn't the one you required".

**Naming mistakes** ✓. A misspelled associated type is `error[E0220]: associated type `Itm` not found for `I`` (`fn f<I: Iterator>(x: I::Itm) {}`). When two bounds both provide an `X`, `T::X` is ambiguous ✓:

```rust
trait A { type X; }
trait B { type X; }
fn f<T: A + B>(x: T::X) {}
fn main() {}
```

✓ `error[E0221]: ambiguous associated type `X` in bounds of `T``. Write the fully qualified form `<T as A>::X` (✓ a version doing that compiled and printed `7`).

**Trait objects must pin every associated type.** `dyn Iterator` alone does not name a complete type ✓:

```rust
fn f(it: Box<dyn Iterator>) {}
fn main() {}
```

✓ `error[E0191]: the value of the associated type `Item` in `Iterator` must be specified`. Write `Box<dyn Iterator<Item = u32>>` ✓:

```rust
fn evens(limit: u32) -> Box<dyn Iterator<Item = u32>> {
    Box::new((0..limit).filter(|x| x % 2 == 0))
}
fn main() { println!("{:?}", evens(7).collect::<Vec<_>>()); }
```

✓ Output `[0, 2, 4, 6]`. (Lesson 21 covers trait objects and what else limits them; Lesson 22 covers generic associated types, which let an associated type take its own parameters, including lifetimes, and is how "lending" iterators from Lesson 17 are eventually expressed.)

---

## 3. Associated types compose: a tiny parser library

Because each type has one `Output`, combinators can compute their own `Output` from their parts' with projections. This is the central skill of the lesson ✓:

```rust
trait Parser {
    type Output;
    fn parse<'a>(&self, input: &'a str) -> Option<(Self::Output, &'a str)>;
}
struct Digit;
impl Parser for Digit {
    type Output = u32;
    fn parse<'a>(&self, input: &'a str) -> Option<(u32, &'a str)> {
        let c = input.chars().next()?;
        Some((c.to_digit(10)?, &input[c.len_utf8()..]))
    }
}
struct Map<P, F>(P, F);
impl<P: Parser, B, F: Fn(P::Output) -> B> Parser for Map<P, F> {
    type Output = B;
    fn parse<'a>(&self, input: &'a str) -> Option<(B, &'a str)> {
        self.0.parse(input).map(|(o, rest)| ((self.1)(o), rest))
    }
}
struct Pair<A, B>(A, B);
impl<A: Parser, B: Parser> Parser for Pair<A, B> {
    type Output = (A::Output, B::Output);
    fn parse<'a>(&self, input: &'a str) -> Option<(Self::Output, &'a str)> {
        let (a, rest) = self.0.parse(input)?;
        let (b, rest) = self.1.parse(rest)?;
        Some(((a, b), rest))
    }
}
fn main() {
    let two = Map(Pair(Digit, Digit), |(a, b)| a * 10 + b);
    println!("{:?} {:?} {:?}", two.parse("42x"), two.parse("4x"), Digit.parse(""));
}
```

✓ Output `Some((42, "x")) None None`. Read the `Map` impl: `P::Output` is whatever the inner parser yields, `F` converts it to some `B`, and `Map<P, F>`'s own `Output` *is* `B`. `Pair` builds the tuple type `(A::Output, B::Output)` and the compiler checks every use. Iterator adapters (`Map`, `Zip`, `Chain` in `std::iter`) are built the same way, which is why `iter.zip(other).map(f)` has a precise type. The practical challenge uses this technique for binary codecs with a second associated type, `Error`.

(`&input[c.len_utf8()..]` slices at a character boundary on purpose; slicing a `&str` at a non-boundary panics, Lesson 05.)

---

## 4. Coherence: one answer per type and trait

**Coherence** is the rule that, across the whole program including all dependencies, any given trait has **at most one applicable `impl` for any given type**. That is what lets `x.to_string()` or `a + b` mean exactly one thing, with no "which implementation wins" configuration. The compiler enforces it in two parts: the **overlap check** (no two impls may apply to the same type) and the **orphan rule** (Section 5, who is allowed to write an impl at all).

### 4.1 Overlap

You already met overlap (Lesson 19: blanket impl plus a specific type). The compiler rejects two impls if *any* type could satisfy both, and "could" is decided conservatively:

```rust
use std::fmt::Display;
trait Tr {}
impl<T: Iterator> Tr for T {}
impl<T: Display> Tr for T {}
fn main() {}
```

✓ `error[E0119]: conflicting implementations of trait `Tr``. No standard type is both an `Iterator` and `Display`, but some type *could* implement both (yours, or one in another crate), so the pair is refused. The same happens when the bounds differ only in an associated type ✓:

```rust
trait Tr {}
impl<T: Iterator<Item = u8>> Tr for T {}
impl<T: Iterator<Item = u16>> Tr for T {}
fn main() {}
```

✓ `error[E0119]: conflicting implementations of trait `Tr``. A type can implement `Iterator` only once, so a human sees these are disjoint, but coherence does not use associated types to prove disjointness (this is a long-standing limitation; I confirmed the error, not the history). The fix in all these cases is to give the cases different traits or a wrapper, not to fight the checker.

### 4.2 "Upstream may add an impl"

Overlap checking also assumes other crates may *grow*. A foreign type you don't control may gain an impl in a later version ✓:

```rust
trait Tr {}
impl<T: Copy> Tr for T {}
impl Tr for String {}
fn main() {}
```

✓ `error[E0119]: conflicting implementations of trait `Tr` for type `String``, with the note **"upstream crates may add a new impl of trait `std::marker::Copy` for type `std::string::String` in future versions"**. Today `String` is not `Copy`, but the standard library is allowed to change that without breaking you, so the compiler refuses to let your code depend on it. Contrast a type **you** own ✓:

```rust
trait Tr { fn n(&self) -> &'static str; }
impl<T: Copy> Tr for T { fn n(&self) -> &'static str { "copy" } }
struct Local;
impl Tr for Local { fn n(&self) -> &'static str { "local" } }
fn main() { println!("{} {}", 1u8.n(), Local.n()); }
```

✓ Output `copy local`. `Local` is yours and not `Copy`, and nobody else can add `Copy for Local`, so the compiler accepts both impls. The practical consequence is a **semver rule** for library authors: adding a blanket impl, or adding an impl of a trait to a type that wasn't covered before, can break downstream code that has an overlapping impl (the API guidelines and RFC 1105 discuss which changes are breaking; I'm citing them from memory, see the references).

### 4.3 Specialization is not stable

The feature that would let a specific impl override a blanket one is called specialization. On stable it is unavailable ✓: `#![feature(specialization)]` gives `error[E0554]: `#![feature]` may not be used on the stable release channel`. Design as if it doesn't exist: separate traits, wrapper types, or enums.

---

## 5. The orphan rule: who may write an impl

The rule (the Rust Reference's "Orphan rules", from RFC 2451; wording from memory): for `impl<P1, .., Pn> Trait<T1, .., Tn> for T0`, the impl is allowed if **the trait is local**, or **at least one of the types `T0 .. Tn` is local** and **no uncovered type parameter appears in any type before the first local one**. "Local" means defined in your crate. A type parameter is **uncovered** if it stands alone (`T`) and **covered** if it is wrapped inside some type (`Vec<T>`, `W<T>`).

**Foreign trait on a foreign type: rejected** ✓:

```rust
use std::fmt;
impl fmt::Display for Vec<i32> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result { write!(f, "v") }
}
fn main() {}
```

✓ `error[E0117]: only traits defined in the current crate can be implemented for types defined outside of the crate`. Why: if you could, and another crate did the same, then a program using both would have two `Display for Vec<i32>`: coherence would be broken. The orphan rule makes that impossible, because only the owner of the trait or of the type can write the impl.

**What is allowed** ✓:

```rust
use std::fmt;
struct Local(u8);
impl fmt::Display for Local {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result { write!(f, "L{}", self.0) }
}
impl From<Local> for Vec<u8> { fn from(l: Local) -> Vec<u8> { vec![l.0] } }
impl From<Local> for String { fn from(l: Local) -> String { format!("{l}") } }
fn main() {
    let v: Vec<u8> = Local(7).into();
    let s: String = Local(8).into();
    println!("{v:?} {s} {}", Local(9));
}
```

✓ Output `[7] L8 L9`. Foreign trait, local type (`Display for Local`): fine. Foreign trait, foreign `Self`, but the trait's *parameter* is local (`From<Local> for Vec<u8>`): fine, because `T1 = Local` is local and nothing uncovered precedes it.

**Uncovered type parameters: rejected** ✓:

```rust
use std::fmt;
impl<T> fmt::Display for T {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result { write!(f, "x") }
}
fn main() {}
```

✓ `error[E0210]: type parameter `T` must be used as the type parameter for some local type (e.g., `MyStruct<T>`)`. And a variant where the parameter comes *before* the local type ✓: `struct Local; impl<T> From<Local> for T { ... }` gives `error[E0210]: type parameter `T` must be covered by another type when it appears before the first local type (`Local`)`. The reverse order, `impl<T> From<T> for Local`, passes the orphan rule but runs into overlap with the standard library's own `impl<T> From<T> for T` ✓:

```rust
struct Local;
impl<T> From<T> for Local { fn from(_: T) -> Local { Local } }
fn main() {}
```

✓ `error[E0119]: conflicting implementations of trait `From<Local>` for type `Local``, "conflicting implementation in crate `core`". (Orphan rule and overlap are separate checks; you can fail either.)

**Wrapping a local type** ✓. Plain `&`, `&mut`, `Box` (and `Pin`) are `#[fundamental]`: the compiler treats `&Local` or `Box<Local>` as local. Other containers are not:

```rust
use std::fmt;
struct Local;
impl fmt::Display for &Local {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result { write!(f, "&L") }
}
impl fmt::Display for Box<Local> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result { write!(f, "BoxL") }
}
fn main() { println!("{} {}", &Local, Box::new(Local)); }
```

✓ Output `&L BoxL`. But `impl Display for Vec<Local>` is ✓ `error[E0117]` (a `Vec<Local>` is not a local type), and so is `impl Display for (Local, i32)` ✓ (`error[E0117]: only traits defined in the current crate can be implemented for arbitrary types`). A **generic local wrapper** is fine because the parameter is covered ✓:

```rust
use std::fmt;
struct W<T>(T);
impl<T: fmt::Display> fmt::Display for W<T> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result { write!(f, "[{}]", self.0) }
}
fn main() { println!("{} {}", W(1), W("x")); }
```

✓ Output `[1] [x]`.

**Workarounds** (all ✓ run):

1. **Newtype** (Lesson 15): `struct Wrapper(Vec<String>); impl Display for Wrapper` printed `[a, b]`. Add `Deref` or forwarding methods if you want it to feel like the inner type; remember the invariant caveats from Lesson 15.
2. **Local trait on a foreign type** is always allowed: `trait Describe {..} impl Describe for Vec<i32> {..} impl Describe for i32 {..}` printed `2 ints / an int`. You own the trait, so the rule is satisfied. This is the *extension trait* pattern (Section 6).
3. **Put the impl where the owner is**: upstream it, or use a feature of the crate that provides the integration.

The orphan rule is sometimes annoying, but it is what makes it safe to add a dependency: no crate can change how another crate's types print, compare or serialize just by being linked in.

---

## 6. Blanket impls and extension traits

A blanket impl applies to a family of types. Two patterns are worth knowing.

**Forwarding through references and boxes.** If your trait takes `&self`, callers will hold `&T` and `Box<T>` and expect them to work ✓:

```rust
trait Name { fn name(&self) -> String; }
struct A;
impl Name for A { fn name(&self) -> String { "A".into() } }
impl<T: Name + ?Sized> Name for &T {
    fn name(&self) -> String { format!("&{}", (**self).name()) }
}
impl<T: Name + ?Sized> Name for Box<T> {
    fn name(&self) -> String { format!("box {}", (**self).name()) }
}
fn show<N: Name>(n: N) -> String { n.name() }
fn main() {
    let a = A;
    println!("{} | {} | {}", show(A), show(&a), show(Box::new(A)));
    let d: Box<dyn Name> = Box::new(A);
    println!("{}", show(d));
}
```

✓ Output `A | &A | box A` and `box A`. The `?Sized` lets the `Box` impl cover `Box<dyn Name>` (Lesson 19 §2.2). Real implementations forward faithfully; the labels here exist only to show *which* impl ran. This is how `Iterator for &mut I`, `Read for &mut R` and `Write for Box<W>` exist in std, and it is why you can pass `&mut reader` to a function that takes `R: Read`.

**Extension traits.** To add methods to every type that satisfies a bound, declare a trait with default methods and blanket-implement it ✓:

```rust
trait IterExt: Iterator {
    fn checked_sum_sq(self) -> Option<u64>
    where Self: Sized, Self::Item: Into<u64> {
        let mut total: u64 = 0;
        for x in self {
            let v: u64 = x.into();
            total = total.checked_add(v.checked_mul(v)?)?;
        }
        Some(total)
    }
}
impl<I: Iterator> IterExt for I {}
fn main() {
    println!("{:?} {:?} {:?}",
        [1u8, 2, 3].into_iter().checked_sum_sq(),
        [u64::MAX].into_iter().checked_sum_sq(),
        [1u32 << 31, 1 << 31].into_iter().checked_sum_sq());
}
```

✓ Output `Some(14) None Some(9223372036854775808)`. Every iterator in the program, including ones written later, now has `.checked_sum_sq()`, provided the trait is in scope (Lesson 19 §3.1). The `checked_*` calls follow the overflow advice from Lesson 19 §10 (an unchecked `v * v` on `u64` panics in debug builds and wraps in release). The popular `itertools` crate is built this way. The cost of the pattern is the coherence limit from Section 4: once `impl<I: Iterator> IterExt for I {}` exists, nobody can implement `IterExt` for a particular iterator type.

---

## 7. Common mistakes and edge cases

- **Using a type parameter where an associated type fits** (E0283 at every call site ✓), or the reverse: an associated type where you wanted several impls per type (E0119 ✓).
- **Forgetting to pin an associated type**: `dyn Iterator` (E0191 ✓), `Box<dyn Container>`.
- **Missing equality constraint**: a function generic over `S: Source` cannot assume `S::Item` is `u32` (the debugging challenge shows the exact error).
- **Adding `Debug` or `Clone` bounds on an associated type after release**: a breaking change for implementors.
- **Overlapping blanket impls** (E0119 ✓), including ones that are disjoint "in practice" and ones that differ only in an associated type ✓.
- **Implementing a foreign trait for a foreign type** (E0117 ✓) and **uncovered type parameters** (E0210 ✓); the cure is a newtype or a local trait.
- **Assuming a missing impl is permanent**: for foreign types the compiler assumes upstream may add one ("upstream crates may add a new impl" ✓).
- **Ambiguous associated type names** with two bounds (E0221 ✓); use `<T as Trait>::X`.
- **Expecting associated type defaults** (E0658 ✓); they are unstable.
- **Blanket impls hide your own types**: after `impl<T: Display> Tr for T`, a later type that implements `Display` silently gets `Tr`.

---

## 8. Performance implications

- Associated types exist only at compile time. A call through `C::Item` or `<S as Source>::Item` is resolved when the generic code is monomorphized (Lesson 19), so it costs the same as writing the concrete type. I did not benchmark this; it follows from how monomorphization works.
- Zero-sized combinators stay zero-sized ✓: `size_of::<Pair<U8, VarInt>>()` is `0` in the practical challenge, `Seq<VarInt>` is `8` (just the `usize` limit), and `Pair<U8, Seq<VarInt>>` is `8`. A `&dyn Codec<Value = u64, Error = VarIntError>` is `16` bytes (data pointer plus vtable pointer, Lesson 19 §6.1).
- Static dispatch through blanket impls (`&C`, `Box<C>`) adds a forwarding call the optimizer normally inlines. Dynamic dispatch (`Box<dyn Codec<..>>`) adds an indirect call per method; use it where flexibility pays for it, and measure (Lesson 41).
- Deeply nested combinator types (`Map<Pair<Map<...>>>`) produce long type names and slower compiles, a known trade-off of this style (Lesson 42).

---

## 9. Security and reliability

- **Coherence is a safety property for dependencies.** Because only the trait's or the type's owner may write an impl, linking another crate cannot silently replace how your types print, compare or serialize. It does **not** protect you from a malicious dependency (that is a supply-chain problem, Lesson 43), and it does not stop a crate's own impl from being wrong.
- **Associated-type contracts are documentation the compiler can't check.** `type Error: Error + 'static` says what an error can do, not that it is informative or that it avoids leaking secrets in `Display`.
- **Trait contracts that generic code trusts.** In the practical challenge, `Pair::decode` slices `&input[n1..]` and trusts `Codec::decode` to return `n1 <= input.len()`. A buggy third-party codec that returned a larger number would make `Pair` panic. Safe Rust prevents memory unsafety, not panics or logic errors, so document such contracts, and consider returning an error instead of slicing when a trait is open for outside implementors. Sealing the trait (Lesson 15) closes it.
- **Decoding untrusted bytes** (the challenge's whole theme): cap lengths *in the type* (`Seq::new(item, max_len)`), never size an allocation from an attacker-supplied number (✓ with `Vec::with_capacity(len)` instead of `len.min(input.len())`, a claimed length of 2^40 aborted the process with "memory allocation of 1099511627776 bytes failed"), reject trailing bytes when the format has a single length (`decode_exact`), and accept only **canonical** encodings where two byte strings must not mean the same value (✓ removing the canonical check made the property test fail on `[132, 0, 131]`). Non-canonical encodings matter in signatures, hashes and consensus code, where "the same message" must have one byte representation (Lessons 48 and 49).
- **Overflow in decoding**: the varint decoder rejects a tenth byte greater than 1 (`Overflow`) and more than ten bytes (`TooLong`), because shifting past 63 bits would otherwise lose information (✓ tests for each).
- **Error chains**: implement `Error::source` so logs show root causes (✓ the test walks `Pair`'s `source()`), and keep `Display` free of secrets. Lesson 26 covers error design more broadly.
- Rust does not make a decoder secure by itself: the bounds, canonical rules and tests above are what do.

---

## 10. Exercises

Try each before opening the answer. They are not marked complete unless you tell me you did them.

1. **(Easy)** Implement `Iterator` for `Countdown(u32)` yielding `n, n-1, ..., 1`. Which part is the associated type?
2. **(Easy)** `impl T for S { fn f(&self) -> u8 { 1 } }` against `trait T { type X; fn f(&self) -> Self::X; }` fails. What is the error code and the fix?
3. **(Medium)** Write `flatten_sum` that sums an iterable of iterables of `i64`, using only `where` clauses over `IntoIterator`.
4. **(Medium)** Which of these compile, and with what error otherwise? (a) `impl Display for Vec<i32>`; (b) `impl From<Local> for Vec<u8>`; (c) `impl<T> Display for T`; (d) `impl Display for Box<Local>`; (e) `impl Display for Vec<Local>`; (f) `impl Display for (Local, i32)`; (g) `impl<T> From<T> for Local`. (`Local` is a struct in your crate.)
5. **(Medium)** You are designing `Storage` with a key and a value type. When is `trait Storage { type Key; type Value; }` right, and when `trait Storage<K, V>`? Give one test for each.
6. **(Medium)** Why does `impl<T: Copy> Tr for T` plus `impl Tr for String` fail, but plus `impl Tr for Local` succeed?
7. **(Hard)** Implement `Map<P, F>` for the `Parser` trait of Section 3 so that `Map(Digit, |d| d * 2)` has `Output = u32` and `Map(Digit, |d| d.to_string())` has `Output = String`.
8. **(Hard)** Write an extension trait `IterExt` with `checked_sum_sq(self) -> Option<u64>` for any iterator whose items convert into `u64`, returning `None` on overflow. Why is the blanket impl `impl<I: Iterator> IterExt for I {}` allowed even though `Iterator` is foreign?

<details markdown="1">
<summary>Answers</summary>

1. ✓ (run: `Countdown(4).collect::<Vec<_>>()` is `[4, 3, 2, 1]`, and `for n in Countdown(3)` prints `3 2 1`)
   ```rust
   struct Countdown(u32);
   impl Iterator for Countdown {
       type Item = u32;
       fn next(&mut self) -> Option<u32> {
           if self.0 == 0 { None } else { self.0 -= 1; Some(self.0 + 1) }
       }
   }
   fn main() { println!("{:?}", Countdown(3).collect::<Vec<_>>()); }
   ```
   `type Item = u32;` is the associated type. Everything else (`map`, `sum`, `collect`) comes free from `Iterator`'s default methods.
2. ✓ `error[E0046]: not all trait items implemented, missing: `X``. Add `type X = u8;` in the `impl`. With it the program compiled and printed `1`.
   ```rust
   trait X { type X; fn f(&self) -> Self::X; }
   struct S;
   impl X for S { type X = u8; fn f(&self) -> u8 { 1 } }
   fn main() { println!("{}", S.f()); }
   ```
3. ✓ (run: `flatten_sum(vec![vec![1, 2], vec![3]])` is `6`; `flatten_sum([[10i64; 2], [5; 2]])` is `30`)
   ```rust
   fn flatten_sum<I>(it: I) -> i64
   where I: IntoIterator, I::Item: IntoIterator<Item = i64> {
       it.into_iter().flat_map(|inner| inner.into_iter()).sum()
   }
   fn main() { println!("{}", flatten_sum(vec![vec![1, 2], vec![3]])); }
   ```
   `I::Item` is itself an iterable, so it gets its own bound with an equality constraint on *its* `Item`. (A sum of untrusted `i64`s can overflow; use `checked_add` if that matters.)
4. ✓ (all run) (a) **E0117**. (b) compiles (local type as the trait parameter, nothing uncovered before it). (c) **E0210** (uncovered `T`). (d) compiles (`Box` is fundamental, so `Box<Local>` counts as local). (e) **E0117** (`Vec<Local>` is not local). (f) **E0117** ("arbitrary types"). (g) passes the orphan rule but fails overlap with core's `impl<T> From<T> for T`: **E0119**.
5. Use **associated types** when, given the storage type, the key and value are fixed (a `HashStore` stores one `Key` and one `Value`), when callers should not have to name them (`fn get(&self, k: &S::Key)`), and when you want exactly one impl per type. Use **parameters** when one type should implement the trait for several combinations (`impl Storage<u64, Blob> for Db` and `impl Storage<String, Row> for Db`), at the cost that callers will need annotations. Tests: "can the same type reasonably implement it twice?" (parameter) and "does inference ever need to pick between impls?" (E0283 ✓ means you chose a parameter where an associated type would have been simpler).
6. ✓ `String` is a foreign type, so the compiler must assume upstream could add `impl Copy for String`; it then reports E0119 with the note "upstream crates may add a new impl of trait `std::marker::Copy` for type `std::string::String` in future versions". `Local` is yours: nobody else can implement `Copy` for it, so the compiler can see that `Local: !Copy` today and forever, and accepts the pair (✓ `1u8.n()` printed `copy`, `Local.n()` printed `local`).
7. ✓ (run: `Map(Pair(Digit, Digit), |(a, b)| a * 10 + b).parse("42x")` is `Some((42, "x"))`)
   ```rust
   // `Parser` is the trait from Section 3.
   struct Map<P, F>(P, F);
   impl<P: Parser, B, F: Fn(P::Output) -> B> Parser for Map<P, F> {
       type Output = B;
       fn parse<'a>(&self, input: &'a str) -> Option<(B, &'a str)> {
           self.0.parse(input).map(|(o, rest)| ((self.1)(o), rest))
       }
   }
   ```
   The key is the **extra type parameter `B`**: `F: Fn(P::Output) -> B` ties the closure's result to `B`, and `type Output = B` publishes it. Without `B` the impl would have an unconstrained parameter (the E0207 of Lesson 19).
8. ✓ (run: `Some(14) None Some(9223372036854775808)` for `[1u8, 2, 3]`, `[u64::MAX]` and `[1u32 << 31, 1 << 31]`): see the code in Section 6. The blanket impl is allowed because **`IterExt` is a local trait**: the orphan rule only forbids foreign trait on foreign type. Local trait on any type is fine, including `impl<I: Iterator> IterExt for I`.

</details>

---

## 11. Debugging challenge

A teammate writes a small "source" abstraction and a summing function:

```rust
trait Source { type Item; fn next_item(&mut self) -> Option<Self::Item>; }
struct Nums(u32);
impl Source for Nums {
    type Item = u32;
    fn next_item(&mut self) -> Option<u32> {
        if self.0 == 0 { None } else { self.0 -= 1; Some(self.0) }
    }
}
fn total<S: Source>(mut s: S) -> u32 {
    let mut t = 0;
    while let Some(x) = s.next_item() { t += x; }
    t
}
fn main() { println!("{}", total(Nums(4))); }
```

It does not compile. Fix it. Then the teammate wants every `Iterator` and every `Vec<u32>` to be a `Source` too, and adds:

```rust
impl<I: Iterator> Source for I {
    type Item = I::Item;
    fn next_item(&mut self) -> Option<I::Item> { self.next() }
}
impl Source for Vec<u32> {
    type Item = u32;
    fn next_item(&mut self) -> Option<u32> { self.pop() }
}
```

and that does not compile either. Explain both errors and fix them.

<details markdown="1">
<summary>Solution</summary>

**Stage 1.** ✓ `error[E0277]: cannot add-assign `<S as Source>::Item` to `{integer}``. Inside `total`, `S` is any `Source`, so `S::Item` is *some* type the compiler knows nothing about; it cannot be added to a `u32` accumulator. The function must say what it needs: pin the associated type with an equality constraint.

```rust
fn total<S: Source<Item = u32>>(mut s: S) -> u32 {
    let mut t = 0;
    while let Some(x) = s.next_item() { t += x; }
    t
}
```

With this, `total(Nums(4))` compiles and prints `6` (it yields 3, 2, 1, 0). Alternatives: make `total` generic over the item (`S::Item: Into<u64>` and sum as `u64`), or add a bound like `S::Item: Add<Output = S::Item> + Default`; the equality constraint is simplest when the function really means `u32`. Note the error is **E0277**, not E0308: the failure is a missing operator impl on the projected type, not a plain type mismatch.

**Stage 2.** ✓ `error[E0119]: conflicting implementations of trait `Source` for type `Vec<u32>``, with the note **"upstream crates may add a new impl of trait `std::iter::Iterator` for type `std::vec::Vec<u32>` in future versions"**. The blanket `impl<I: Iterator> Source for I` already covers *every* type that is an `Iterator`. `Vec<u32>` is not one today, but it is a foreign type and the standard library may add that impl later, so the compiler refuses to let the two impls coexist (Section 4.2). Note that `impl Source for Nums` was **not** reported: `Nums` is a local type that does not implement `Iterator`, and no other crate can change that.

Two fixes (both ✓ compiled and printed `6 3`):

1. Keep the blanket impl, drop the `Vec<u32>` impl, and convert at the call site: `total(vec![1u32, 2].into_iter())`.
2. Drop the blanket impl and use a **newtype** (Lesson 15) for the cases you want: `struct Items(Vec<u32>); impl Source for Items { type Item = u32; fn next_item(&mut self) -> Option<u32> { self.0.pop() } }`, called as `total(Items(vec![1, 2]))`.

Lesson: a blanket impl over a foreign trait uses up the whole family. If you need special cases, don't blanket-implement; if you need the blanket, don't special-case foreign types.

</details>

---

## 12. Practical challenge: `assoclab`

Create a library crate `assoclab` (edition 2024) of composable binary codecs:

1. A trait `Codec` with associated types `Value` and `Error` (`Error: std::error::Error + 'static`) and methods `encode(&self, &Value, &mut Vec<u8>) -> Result<(), Error>` and `decode(&self, &[u8]) -> Result<(Value, usize), Error>` (value plus bytes consumed).
2. Codecs: `U8`, `U32Be`, `VarInt` (unsigned LEB128 `u64`, canonical form only, at most ten bytes, rejecting overflow), `Pair<A, B>` (value `(A::Value, B::Value)`, error `PairError<A::Error, B::Error>`) and `Seq<C>` (varint length then items, with a maximum length held in the codec, error `SeqError<C::Error>`).
3. Blanket impls `Codec for &C` and `Codec for Box<C>` (both `C: ?Sized`) so references and `Box<dyn Codec<..>>` work as codecs.
4. An extension trait `CodecExt` with a blanket impl for every codec: `encode_to_vec` and `decode_exact` (error if bytes are left over).
5. Proper `Display` and `Error::source` implementations for the combinator errors.
6. Tests: varint boundaries and exact byte strings (`300` encodes to `[0xAC, 0x02]`); every rejection path (eof, non-canonical, overflow, too long); pair error position and source chain; sequence round trip; length limit on both encode and decode; a claimed length of 2^40 must fail without a huge allocation; trailing bytes; `&C`, `Box<C>` and `Box<dyn Codec>` as codecs; and a pseudo-random property test that decoding arbitrary bytes never panics and, whenever it succeeds, re-encoding gives back exactly the consumed bytes. A usage doctest and a `compile_fail` doctest showing that the value type is checked.
7. It must pass `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, and `cargo test --locked` (run `cargo generate-lockfile` once first).

<details markdown="1">
<summary>Solution</summary>

**`src/lib.rs`**

```rust
//! `assoclab`: small composable binary codecs built on associated types.
//!
//! A [`Codec`] has two associated types: the `Value` it reads and writes and
//! the `Error` it can fail with. Combinators such as [`Pair`] and [`Seq`] build
//! their own `Value` and `Error` from the parts' associated types.
//!
//! ```
//! use assoclab::{Codec, CodecExt, Pair, Seq, U8, VarInt};
//!
//! let codec = Pair(U8, Seq::new(VarInt, 16));
//! let bytes = codec.encode_to_vec(&(7, vec![1, 300])).unwrap();
//! assert_eq!(bytes, [7, 2, 1, 0xAC, 0x02]);
//! assert_eq!(codec.decode_exact(&bytes).unwrap(), (7, vec![1, 300]));
//! ```
//!
//! The value type is checked by the compiler: `Pair<U8, VarInt>` has
//! `Value = (u8, u64)`, so a `u32` in the second slot does not compile.
//!
//! ```compile_fail
//! use assoclab::{CodecExt, Pair, U8, VarInt};
//! let _ = Pair(U8, VarInt).encode_to_vec(&(1u8, 2u32));
//! ```

use std::error::Error;
use std::fmt;

/// A reversible byte format for one kind of value.
///
/// Contract for implementors: `decode` returns `(value, consumed)` with
/// `consumed <= input.len()`. Combinators slice the input with `consumed` and
/// trust this. Decoding must never panic on any input.
pub trait Codec {
    /// The value this codec reads and writes.
    type Value;
    /// The error this codec can fail with.
    type Error: Error + 'static;

    /// Appends the encoding of `value` to `out`.
    fn encode(&self, value: &Self::Value, out: &mut Vec<u8>) -> Result<(), Self::Error>;

    /// Decodes one value from the front of `input`, returning it and the
    /// number of bytes consumed.
    fn decode(&self, input: &[u8]) -> Result<(Self::Value, usize), Self::Error>;
}

impl<C: Codec + ?Sized> Codec for &C {
    type Value = C::Value;
    type Error = C::Error;
    fn encode(&self, value: &Self::Value, out: &mut Vec<u8>) -> Result<(), Self::Error> {
        (**self).encode(value, out)
    }
    fn decode(&self, input: &[u8]) -> Result<(Self::Value, usize), Self::Error> {
        (**self).decode(input)
    }
}

impl<C: Codec + ?Sized> Codec for Box<C> {
    type Value = C::Value;
    type Error = C::Error;
    fn encode(&self, value: &Self::Value, out: &mut Vec<u8>) -> Result<(), Self::Error> {
        (**self).encode(value, out)
    }
    fn decode(&self, input: &[u8]) -> Result<(Self::Value, usize), Self::Error> {
        (**self).decode(input)
    }
}

/// Failure of a whole-buffer decode.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum ExactError<E> {
    /// The codec itself failed.
    Codec(E),
    /// The codec succeeded but did not use the whole input.
    Trailing { consumed: usize, total: usize },
}

impl<E: fmt::Display> fmt::Display for ExactError<E> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ExactError::Codec(e) => write!(f, "{e}"),
            ExactError::Trailing { consumed, total } => {
                write!(
                    f,
                    "{} trailing byte(s) after a {consumed}-byte value",
                    total - consumed
                )
            }
        }
    }
}

impl<E: Error + 'static> Error for ExactError<E> {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            ExactError::Codec(e) => Some(e),
            ExactError::Trailing { .. } => None,
        }
    }
}

/// Convenience methods available on every [`Codec`] (an extension trait with a
/// blanket impl).
pub trait CodecExt: Codec {
    /// Encodes into a fresh `Vec`.
    fn encode_to_vec(&self, value: &Self::Value) -> Result<Vec<u8>, Self::Error> {
        let mut out = Vec::new();
        self.encode(value, &mut out)?;
        Ok(out)
    }

    /// Decodes a value that must use the *entire* input.
    fn decode_exact(&self, input: &[u8]) -> Result<Self::Value, ExactError<Self::Error>> {
        let (value, consumed) = self.decode(input).map_err(ExactError::Codec)?;
        if consumed != input.len() {
            return Err(ExactError::Trailing {
                consumed,
                total: input.len(),
            });
        }
        Ok(value)
    }
}

impl<C: Codec + ?Sized> CodecExt for C {}

/// Input ended before the value was complete.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Eof;

impl fmt::Display for Eof {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.write_str("unexpected end of input")
    }
}

impl Error for Eof {}

/// One byte.
#[derive(Debug, Clone, Copy)]
pub struct U8;

impl Codec for U8 {
    type Value = u8;
    type Error = Eof;
    fn encode(&self, value: &u8, out: &mut Vec<u8>) -> Result<(), Eof> {
        out.push(*value);
        Ok(())
    }
    fn decode(&self, input: &[u8]) -> Result<(u8, usize), Eof> {
        input.first().map(|&b| (b, 1)).ok_or(Eof)
    }
}

/// A big-endian `u32`.
#[derive(Debug, Clone, Copy)]
pub struct U32Be;

impl Codec for U32Be {
    type Value = u32;
    type Error = Eof;
    fn encode(&self, value: &u32, out: &mut Vec<u8>) -> Result<(), Eof> {
        out.extend_from_slice(&value.to_be_bytes());
        Ok(())
    }
    fn decode(&self, input: &[u8]) -> Result<(u32, usize), Eof> {
        let (head, _) = input.split_first_chunk::<4>().ok_or(Eof)?;
        Ok((u32::from_be_bytes(*head), 4))
    }
}

/// Why a [`VarInt`] failed to decode.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum VarIntError {
    /// Input ended inside the number.
    Eof,
    /// The value does not fit in a `u64`.
    Overflow,
    /// More than ten bytes, all with the continuation bit set.
    TooLong,
    /// A valid number written with redundant zero bytes (only the shortest
    /// encoding is accepted, so each value has exactly one byte string).
    NonCanonical,
}

impl fmt::Display for VarIntError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.write_str(match self {
            VarIntError::Eof => "varint: unexpected end of input",
            VarIntError::Overflow => "varint: value does not fit in 64 bits",
            VarIntError::TooLong => "varint: more than 10 bytes",
            VarIntError::NonCanonical => "varint: not the shortest encoding",
        })
    }
}

impl Error for VarIntError {}

/// Unsigned LEB128 `u64`: seven bits per byte, high bit means "more".
/// Decoding accepts only the canonical (shortest) form.
#[derive(Debug, Clone, Copy)]
pub struct VarInt;

impl Codec for VarInt {
    type Value = u64;
    type Error = VarIntError;

    fn encode(&self, value: &u64, out: &mut Vec<u8>) -> Result<(), VarIntError> {
        let mut v = *value;
        loop {
            let byte = (v & 0x7f) as u8;
            v >>= 7;
            if v == 0 {
                out.push(byte);
                return Ok(());
            }
            out.push(byte | 0x80);
        }
    }

    fn decode(&self, input: &[u8]) -> Result<(u64, usize), VarIntError> {
        let mut value: u64 = 0;
        for (i, &byte) in input.iter().enumerate().take(10) {
            let payload = u64::from(byte & 0x7f);
            if i == 9 && payload > 1 {
                return Err(VarIntError::Overflow);
            }
            value |= payload << (7 * i);
            if byte & 0x80 == 0 {
                if i > 0 && payload == 0 {
                    return Err(VarIntError::NonCanonical);
                }
                return Ok((value, i + 1));
            }
        }
        if input.len() >= 10 {
            Err(VarIntError::TooLong)
        } else {
            Err(VarIntError::Eof)
        }
    }
}

/// Which half of a [`Pair`] failed.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum PairError<EA, EB> {
    First(EA),
    Second(EB),
}

impl<EA: fmt::Display, EB: fmt::Display> fmt::Display for PairError<EA, EB> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            PairError::First(e) => write!(f, "first field: {e}"),
            PairError::Second(e) => write!(f, "second field: {e}"),
        }
    }
}

impl<EA: Error + 'static, EB: Error + 'static> Error for PairError<EA, EB> {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            PairError::First(e) => Some(e),
            PairError::Second(e) => Some(e),
        }
    }
}

/// Two codecs back to back.
#[derive(Debug, Clone, Copy)]
pub struct Pair<A, B>(pub A, pub B);

impl<A: Codec, B: Codec> Codec for Pair<A, B> {
    type Value = (A::Value, B::Value);
    type Error = PairError<A::Error, B::Error>;

    fn encode(&self, value: &Self::Value, out: &mut Vec<u8>) -> Result<(), Self::Error> {
        self.0.encode(&value.0, out).map_err(PairError::First)?;
        self.1.encode(&value.1, out).map_err(PairError::Second)
    }

    fn decode(&self, input: &[u8]) -> Result<(Self::Value, usize), Self::Error> {
        let (a, n1) = self.0.decode(input).map_err(PairError::First)?;
        let (b, n2) = self.1.decode(&input[n1..]).map_err(PairError::Second)?;
        Ok(((a, b), n1 + n2))
    }
}

/// Why a [`Seq`] failed.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum SeqError<E> {
    /// The length prefix was invalid.
    Length(VarIntError),
    /// The sequence is longer than the configured maximum.
    TooLong { len: u64, max: usize },
    /// Element `index` failed.
    Item { index: usize, source: E },
}

impl<E: fmt::Display> fmt::Display for SeqError<E> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            SeqError::Length(e) => write!(f, "length prefix: {e}"),
            SeqError::TooLong { len, max } => {
                write!(f, "length {len} exceeds the maximum of {max}")
            }
            SeqError::Item { index, source } => write!(f, "element {index}: {source}"),
        }
    }
}

impl<E: Error + 'static> Error for SeqError<E> {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            SeqError::Length(e) => Some(e),
            SeqError::TooLong { .. } => None,
            SeqError::Item { source, .. } => Some(source),
        }
    }
}

/// A varint length followed by that many items. The maximum length is part
/// of the codec, so a hostile length prefix is rejected before any work.
#[derive(Debug, Clone, Copy)]
pub struct Seq<C> {
    item: C,
    max_len: usize,
}

impl<C> Seq<C> {
    /// `max_len` is the largest sequence this codec will encode or decode.
    pub fn new(item: C, max_len: usize) -> Self {
        Seq { item, max_len }
    }
}

impl<C: Codec> Codec for Seq<C> {
    type Value = Vec<C::Value>;
    type Error = SeqError<C::Error>;

    fn encode(&self, value: &Self::Value, out: &mut Vec<u8>) -> Result<(), Self::Error> {
        if value.len() > self.max_len {
            return Err(SeqError::TooLong {
                len: value.len() as u64,
                max: self.max_len,
            });
        }
        VarInt
            .encode(&(value.len() as u64), out)
            .map_err(SeqError::Length)?;
        for (index, item) in value.iter().enumerate() {
            self.item
                .encode(item, out)
                .map_err(|source| SeqError::Item { index, source })?;
        }
        Ok(())
    }

    fn decode(&self, input: &[u8]) -> Result<(Self::Value, usize), Self::Error> {
        let (len, mut offset) = VarInt.decode(input).map_err(SeqError::Length)?;
        if len > self.max_len as u64 {
            return Err(SeqError::TooLong {
                len,
                max: self.max_len,
            });
        }
        let len = usize::try_from(len).map_err(|_| SeqError::TooLong {
            len,
            max: self.max_len,
        })?;
        // Never trust the prefix for allocation: reserve at most one slot per input byte.
        let mut items = Vec::with_capacity(len.min(input.len()));
        for index in 0..len {
            let (item, used) = self
                .item
                .decode(&input[offset..])
                .map_err(|source| SeqError::Item { index, source })?;
            items.push(item);
            offset += used;
        }
        Ok((items, offset))
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    fn varint_bytes(v: u64) -> Vec<u8> {
        VarInt.encode_to_vec(&v).unwrap()
    }

    #[test]
    fn varint_roundtrip_and_lengths() {
        let cases: [(u64, usize); 8] = [
            (0, 1),
            (127, 1),
            (128, 2),
            (16_383, 2),
            (16_384, 3),
            (u64::from(u32::MAX), 5),
            (1 << 63, 10),
            (u64::MAX, 10),
        ];
        for (v, len) in cases {
            let bytes = varint_bytes(v);
            assert_eq!(bytes.len(), len, "length of {v}");
            assert_eq!(VarInt.decode(&bytes), Ok((v, len)));
        }
        assert_eq!(varint_bytes(300), [0xAC, 0x02]);
    }

    #[test]
    fn varint_rejects_bad_input() {
        assert_eq!(VarInt.decode(&[]), Err(VarIntError::Eof));
        assert_eq!(VarInt.decode(&[0x80]), Err(VarIntError::Eof));
        assert_eq!(VarInt.decode(&[0x80, 0x00]), Err(VarIntError::NonCanonical));
        assert_eq!(VarInt.decode(&[0xFF; 10]), Err(VarIntError::Overflow));
        let mut ten_and_more = vec![0xFF; 9];
        ten_and_more.push(0x02);
        assert_eq!(VarInt.decode(&ten_and_more), Err(VarIntError::Overflow));
        assert_eq!(VarInt.decode(&[0x80; 10]), Err(VarIntError::TooLong));
        assert_eq!(VarInt.decode(&[0x80; 11]), Err(VarIntError::TooLong));
        let mut max = vec![0xFF; 9];
        max.push(0x01);
        assert_eq!(VarInt.decode(&max), Ok((u64::MAX, 10)));
    }

    #[test]
    fn u32be_roundtrip_and_eof() {
        let bytes = U32Be.encode_to_vec(&0x0102_0304).unwrap();
        assert_eq!(bytes, [1, 2, 3, 4]);
        assert_eq!(U32Be.decode(&[1, 2, 3, 4, 9]), Ok((0x0102_0304, 4)));
        assert_eq!(U32Be.decode(&[1, 2, 3]), Err(Eof));
    }

    #[test]
    fn pair_roundtrip_and_error_position() {
        let codec = Pair(U8, VarInt);
        let bytes = codec.encode_to_vec(&(7, 300)).unwrap();
        assert_eq!(bytes, [7, 0xAC, 0x02]);
        assert_eq!(codec.decode(&bytes), Ok(((7, 300), 3)));
        assert_eq!(codec.decode(&[]), Err(PairError::First(Eof)));
        assert_eq!(
            codec.decode(&[7, 0x80]),
            Err(PairError::Second(VarIntError::Eof))
        );
        let err = codec.decode(&[7, 0x80]).unwrap_err();
        assert_eq!(
            err.to_string(),
            "second field: varint: unexpected end of input"
        );
        assert_eq!(
            err.source().unwrap().to_string(),
            "varint: unexpected end of input"
        );
    }

    #[test]
    fn seq_roundtrip() {
        let codec = Seq::new(VarInt, 10);
        for v in [vec![], vec![0], vec![1, 300, 0, u64::MAX]] {
            let bytes = codec.encode_to_vec(&v).unwrap();
            assert_eq!(codec.decode_exact(&bytes), Ok(v));
        }
    }

    #[test]
    fn seq_limits_and_hostile_prefix() {
        let codec = Seq::new(VarInt, 2);
        assert_eq!(
            codec.encode_to_vec(&vec![1, 2, 3]),
            Err(SeqError::TooLong { len: 3, max: 2 })
        );
        let huge = varint_bytes(1 << 40);
        assert_eq!(
            codec.decode(&huge),
            Err(SeqError::TooLong {
                len: 1 << 40,
                max: 2
            })
        );
        // Even with no effective limit, a huge claimed length and no data fails
        // at the first element instead of reserving gigabytes.
        let unlimited = Seq::new(U8, usize::MAX);
        assert_eq!(
            unlimited.decode(&huge),
            Err(SeqError::Item {
                index: 0,
                source: Eof
            })
        );
        assert_eq!(
            codec.decode(&[2, 1]),
            Err(SeqError::Item {
                index: 1,
                source: VarIntError::Eof
            })
        );
    }

    #[test]
    fn decode_exact_rejects_trailing_bytes() {
        assert_eq!(VarInt.decode_exact(&[1]), Ok(1));
        let err = VarInt.decode_exact(&[1, 2]).unwrap_err();
        assert_eq!(
            err,
            ExactError::Trailing {
                consumed: 1,
                total: 2
            }
        );
        assert_eq!(err.to_string(), "1 trailing byte(s) after a 1-byte value");
        assert_eq!(
            VarInt.decode_exact(&[]),
            Err(ExactError::Codec(VarIntError::Eof))
        );
    }

    #[test]
    fn blanket_impls_references_boxes_and_dyn() {
        let by_ref = Seq::new(&VarInt, 4);
        assert_eq!(by_ref.encode_to_vec(&vec![5]).unwrap(), [1, 5]);
        let boxed: Box<dyn Codec<Value = u64, Error = VarIntError>> = Box::new(VarInt);
        let codec = Pair(boxed, U8);
        let bytes = codec.encode_to_vec(&(300, 9)).unwrap();
        assert_eq!(bytes, [0xAC, 0x02, 9]);
        assert_eq!(codec.decode_exact(&bytes), Ok((300, 9)));
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

    fn check_canonical<C: Codec>(codec: &C, rng: &mut XorShift)
    where
        C::Value: PartialEq + fmt::Debug,
    {
        for _ in 0..20_000 {
            let len = (rng.next() % 24) as usize;
            // Bias bytes toward continuation bits and small numbers so decodes succeed often.
            let input: Vec<u8> = (0..len)
                .map(|_| match rng.next() % 4 {
                    0 => 0x80 | (rng.next() as u8 & 0x7f),
                    1 => (rng.next() % 4) as u8,
                    _ => rng.next() as u8,
                })
                .collect();
            if let Ok((value, used)) = codec.decode(&input) {
                assert!(used <= input.len());
                let again = codec.encode_to_vec(&value).unwrap();
                assert_eq!(again, input[..used], "non-canonical accept of {input:?}");
            }
        }
    }

    #[test]
    fn decode_never_panics_and_is_canonical() {
        let mut rng = XorShift(0x9E37_79B9_7F4A_7C15);
        check_canonical(&VarInt, &mut rng);
        check_canonical(&U32Be, &mut rng);
        check_canonical(&Pair(U8, VarInt), &mut rng);
        check_canonical(&Seq::new(VarInt, 8), &mut rng);
        check_canonical(&Seq::new(Pair(U8, VarInt), 3), &mut rng);
    }
}
```

✓ With this exact file: `cargo fmt --check` clean, `cargo clippy --all-targets --locked -- -D warnings` clean, `cargo test --locked` → 9 unit tests, 1 passing doctest and 1 `compile_fail` doctest passed. I also mutated the code to check the tests have teeth: replacing `Vec::with_capacity(len.min(input.len()))` with `Vec::with_capacity(len)` made `seq_limits_and_hostile_prefix` abort with "memory allocation of 1099511627776 bytes failed", and disabling the canonical check made both `varint_rejects_bad_input` and the property test fail (`non-canonical accept of [132, 0, 131]`). In a separate binary crate I confirmed ✓ that `Pair(U8, VarInt).encode_to_vec(&(1u8, 2u32))` is `error[E0308]: mismatched types` ("expected `u64`, found `u32`"); the `compile_fail` doctest alone proves only that *some* error occurs.

**Design notes.**

- **Two associated types, no type parameters on the trait.** A codec fixes both what it handles and how it fails; combinators compute `Value` and `Error` from their parts (`(A::Value, B::Value)`, `PairError<A::Error, B::Error>`), the same technique as the parser in Section 3.
- **Why `Error` is associated, not a parameter:** `Seq<C>`'s error type is `SeqError<C::Error>`. With a parameter you would have to thread `E` through every combinator signature and callers would need annotations (E0283, Section 2.1).
- **`Error: Error + 'static`** lets any codec error be boxed as `Box<dyn Error>` and walked with `source()` while still keeping precise types inside the library.
- **Combinator error types need their own `Error` impls with bounds** (`impl<EA: Error + 'static, EB: Error + 'static> Error for PairError<EA, EB>`), otherwise `Pair<A, B>` could not satisfy `Codec::Error: Error`.
- **Blanket impls for `&C` and `Box<C>`** with `?Sized` let `Box<dyn Codec<Value = u64, Error = VarIntError>>` slot into `Pair` (the dyn type pins both associated types, Section 2.3).
- **`CodecExt` is a blanket extension trait**: its methods are default bodies, and `impl<C: Codec + ?Sized> CodecExt for C {}` gives them to every codec, including future ones. Because the trait is local, the orphan rule is satisfied.
- **Security choices** (Section 9): the maximum length lives in `Seq`, the allocation is bounded by the input size, the varint decoder is canonical and overflow-checked, and `decode_exact` rejects trailing bytes. `Pair` and `Seq` slice the input with the consumed counts they are given; that is the documented trait contract, and a hostile *implementation* of `Codec` could still make them panic (Section 9).
- **Not a serialization framework.** No signed varints, no versioning, no endianness options beyond big-endian `u32`, no streaming, and `Seq` of zero-sized-encoding items is bounded only by `max_len`.

</details>

---

## 13. Interview questions

<details markdown="1">
<summary>Answers</summary>

1. **What is an associated type?** A type placeholder inside a trait that each impl fills in once (`Iterator::Item`). Generic code refers to it as `T::Item` or `<T as Trait>::Item`.
2. **Associated type or generic parameter?** Associated when the implementing type determines the answer (one impl per type); generic parameter when one type may implement the trait many times for different types (`From<A>` and `From<B>`). Std traits like `Add<Rhs>` use both: a parameter for the input, an associated type for the output.
3. **What happens if two impls of the same trait exist for one type?** E0119 (conflicting implementations). Coherence allows at most one applicable impl per type and trait in the whole program.
4. **State the orphan rule.** You may implement a trait for a type only if the trait or the type (or a trait parameter, with no uncovered type parameter before it) is local to your crate. It prevents two crates from defining conflicting impls of a foreign trait for a foreign type.
5. **What are E0117 and E0210?** E0117: foreign trait on a foreign type. E0210: an uncovered type parameter such as `impl<T> Display for T` (or `impl<T> From<Local> for T`).
6. **What does "covered" mean?** A type parameter is covered when it appears inside another type, like `Vec<T>` or `Wrapper<T>`; bare `T` is uncovered.
7. **What are fundamental types?** `&`, `&mut`, `Box` and `Pin` are treated as transparent by the orphan rule, so `&Local` and `Box<Local>` count as local.
8. **How do you work around the orphan rule?** A newtype wrapper, a local extension trait, or implementing in the crate that owns the trait or type.
9. **Why does `impl<T: Copy> Tr for T` conflict with `impl Tr for String`?** `String` is foreign, so upstream may add `Copy for String` later; coherence assumes that could happen. For a local type the compiler knows no one else can add the impl.
10. **Why do blanket impls make later impls impossible?** They apply to every type matching the bound, so any specific impl overlaps unless the compiler can prove the sets disjoint (it does not reason about associated types).
11. **What is E0271 and how does it differ from E0277?** E0277: a required trait is not implemented. E0271: the trait is implemented but an associated type differs from the required equality (`Iterator<Item = u32>` given `Item = u8`).
12. **Why must a trait object name its associated types?** `dyn Iterator` is not a complete type without `Item`; write `dyn Iterator<Item = T>` (E0191 otherwise).
13. **What is an extension trait?** A local trait with default methods blanket-implemented for a family of types (`impl<I: Iterator> Ext for I {}`), adding methods to foreign types legally.
14. **Is adding a blanket impl a breaking change for a library?** It can be, because downstream code with an overlapping impl will stop compiling (E0119). The same goes for adding an impl on a type not previously covered. Treat new impls as part of the API.
15. **Are associated type defaults stable?** No (E0658, `associated_type_defaults`).
16. **How would you design a binary decoder for untrusted input?** Cap lengths in the type, never allocate from an attacker-controlled number, accept only canonical encodings, reject trailing bytes, return typed errors with `source`, and property-test that no input panics and that decode-then-encode reproduces the consumed bytes.
17. **How are associated types represented at run time?** They aren't: they are resolved at compile time, so they add no run-time cost beyond the monomorphized code.
18. **Why can `impl<I: Iterator> MyTrait for I {}` be written by a downstream crate when `Iterator` is foreign?** Because `MyTrait` is local; the orphan rule only restricts foreign traits on foreign types.

</details>

---

## 14. Key takeaways

- An associated type is a trait-level placeholder filled in once per implementation. Use it when the implementing type determines the answer; use a generic parameter when a type can implement the trait several ways. Std mixes them (`Add<Rhs>::Output`).
- Name them with `T::Item` or `<T as Trait>::Item`; pin them with `Trait<Item = X>`; bound them with `where T::Item: Bound` or `Trait<Item: Bound>`. Generic code knows nothing about a projection beyond its declared bounds (the debugging challenge's E0277).
- Errors to know: E0046 (missing type), E0220/E0221 (unknown/ambiguous name), E0271 (equality mismatch), E0191 (unpinned `dyn`), E0658 (defaults unstable).
- Coherence gives one impl per (type, trait). Overlap (E0119) is decided conservatively: blanket impls, differing associated types and foreign types that "might" gain an impl all count as overlap.
- The orphan rule (E0117, E0210) says who may write an impl: the owner of the trait or the type. `&`, `&mut`, `Box` and `Pin` are transparent; other containers are not. Newtypes and local extension traits are the standard workarounds.
- Blanket impls and extension traits are powerful and use up the whole family; adding one is an API commitment. Forward `&T` and `Box<T>` for traits users will pass by reference.
- For parsing or decoding untrusted data, cap lengths, bound allocations, require canonical forms, reject trailing bytes, and test with random input; Rust prevents memory errors, not logic errors.

**Connections.** Lesson 13's impl blocks, Lesson 14's `FromStr`/`TryFrom` (their `Err`/`Error` are associated types), Lesson 15's newtypes and E0117, Lesson 17's `Iterator::Item`, `IntoIterator::IntoIter` and `FromIterator`, and Lesson 19's bounds, blanket impls and `Add<Output = T>` are all instances of this lesson. Next: Lesson 21 (trait objects, object safety, vtables, `dyn Any`) and Lesson 22 (generic associated types, `Send`/`Sync`, variance, dynamically sized types). Lesson 26 covers error design (including `source` chains), Lesson 33 covers API design (semver of trait impls, sealing), and Lessons 48 and 49 use canonical encodings for hashing and signatures. See the [roadmap](00_roadmap.md) and the [course index](index.md).

---

## 15. References

- *The Rust Programming Language*, ch. 20.2 "Advanced Traits" (associated types versus generics, default type parameters, fully qualified syntax, the newtype pattern for the orphan rule): <https://doc.rust-lang.org/book/ch20-02-advanced-traits.html>
- *The Rust Reference*, "Associated Items", "Implementations" (including "Trait implementation coherence" and the orphan rules): <https://doc.rust-lang.org/reference/items/associated-items.html>, <https://doc.rust-lang.org/reference/items/implementations.html>
- RFC 2451 "Re-rebalancing coherence" (the current orphan rule): <https://rust-lang.github.io/rfcs/2451-re-rebalancing-coherence.html>; RFC 1023 "Rebalancing coherence": <https://rust-lang.github.io/rfcs/1023-rebalancing-coherence.html>
- RFC 1105 "API evolution" (which changes, including adding impls, are breaking): <https://rust-lang.github.io/rfcs/1105-api-evolution.html>
- `std::iter::Iterator`, `std::ops::Add`, `std::ops::Index`, `std::convert::TryFrom`, `std::str::FromStr`, `std::error::Error`: <https://doc.rust-lang.org/std/iter/trait.Iterator.html>, <https://doc.rust-lang.org/std/error/trait.Error.html>
- *Rust API Guidelines*, "Sealed traits" (C-SEALED) and "Future proofing": <https://rust-lang.github.io/api-guidelines/future-proofing.html>
- Rust error code index: E0046, E0117, E0119, E0191, E0210, E0220, E0221, E0271, E0554, E0658: <https://doc.rust-lang.org/error_codes/E0117.html>, <https://doc.rust-lang.org/error_codes/E0210.html>, <https://doc.rust-lang.org/error_codes/E0271.html>
- LEB128 (the varint format used in the challenge): <https://en.wikipedia.org/wiki/LEB128>

The URLs above are the standard documentation locations; I did not open them during this session (the sandbox cannot reach them), so if a link has moved, search the docs site for the page title. Unchecked from memory: the exact wording of the orphan rule and its RFC numbers (2451, 1023), the RFC 1105 reference for impls as breaking changes, that `&`, `&mut`, `Box` and `Pin` are the `#[fundamental]` types, that the associated-type-bound syntax `Iterator<Item: Debug>` stabilized in Rust 1.79, that coherence does not use associated-type disjointness (I confirmed the error, not the reason), the std trait lists in the Section 2.1 table (`Future::Output`, `AsRef<T>`, `PartialEq<Rhs>`), and the statement that `Iterator for &mut I`, `Read for &mut R` and `Write for Box<W>` exist in std.
