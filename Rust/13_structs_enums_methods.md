---
layout: default
title: "Lesson 13 — Structs, enums, methods, and associated functions"
---

# Lesson 13 — Structs, enums, methods, and associated functions

**Module 5 · Data modeling** · Status: generated · Edition used in examples: **2024** · External dependencies: none

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux. As before, ✓ marks outputs and diagnostics I actually ran; unmarked statements are documented behavior I did not run. Everything is **stable Rust**.

Builds on [Lesson 07](07_ownership_and_moves.md) (moves, `Copy`/`Clone`, partial moves), [Lesson 09](09_memory_layout.md) (struct and enum layout, niches), [Lesson 10](10_references_and_borrowing.md) (`&self` vs `&mut self`) and [Lesson 06](06_strings_conversions_patterns.md) (pattern matching).

---

## 1. Why this lesson matters

Modules 1–4 gave you values, ownership and borrowing. Now you build your own types. In Rust, **types are the main design tool**: a good set of structs and enums makes whole classes of bugs impossible to write, because the compiler refuses programs that put the data in an illegal shape. Everything later in the course (traits, error design, async state machines, Solana accounts, Substrate pallets, transaction types) is built from the pieces in this lesson.

Three ideas run through it:

1. A **struct** is "this AND that AND that" (a product type). An **enum** is "this OR that OR that" (a sum type, tagged union).
2. Methods are functions in an `impl` block whose first parameter (`self`) says **how** they use the value: borrow it, mutate it, or consume it.
3. **Make illegal states unrepresentable**: choose a shape in which the invalid combinations simply cannot be written.

---

## 2. Structs

### 2.1 Three kinds

```rust
#[derive(Debug, Clone, PartialEq)]
struct User { name: String, email: String, active: bool, logins: u32 }
struct Point(i32, i32);          // tuple struct
struct Marker;                   // unit struct
fn main() {
    let name = String::from("ana");
    let u = User { name, email: "ana@example.com".into(), active: true, logins: 0 };  // field init shorthand
    let u2 = User { email: "ana2@example.com".into(), ..u.clone() };                  // struct update
    println!("{u2:?}");
    println!("{:#?}", u);
    let p = Point(3, 4);
    let Point(x, y) = p;
    println!("{} {} {}", p.0, x, y);
    let _m = Marker;
    println!("{} {}", std::mem::size_of::<Marker>(), std::mem::size_of::<Point>());
}
```

✓ Output: `User { name: "ana", email: "ana2@example.com", active: true, logins: 0 }`, then the pretty-printed `{:#?}` form of `u` (one field per line, trailing commas), `3 3 4`, and `0 8` (the unit struct is a zero-sized type, Lesson 09; `Point` is two `i32`s).

- **Named-field structs** are the default choice: field names document themselves.
- **Tuple structs** are for tiny wrappers (`struct Meters(f64);`, the *newtype* pattern, Lesson 15) where a name would add nothing. Access by `.0`, destructure with `let Point(x, y) = p`.
- **Unit structs** carry no data; they are markers or the carrier of a trait impl (Lesson 15, 21).
- **Field init shorthand** (`User { name, .. }`) when a variable has the same name as the field.
- **`#[derive(Debug)]`** generates the `{:?}` and `{:#?}` output. Derive it on nearly every type you define; it costs nothing at run time unless you print.

### 2.2 Struct update syntax moves fields

`..old` fills the remaining fields from another value, **moving** (or copying) each one:

```rust
#[derive(Debug)]
struct User { name: String, email: String, logins: u32 }
fn main() {
    let a = User { name: "a".into(), email: "a@x".into(), logins: 1 };
    let b = User { email: "b@x".into(), ..a };
    println!("{:?}", b);
    println!("{}", a.logins);          // Copy field: still usable
    println!("{}", a.name);            // String field was moved into b
}
```

✓ `error[E0382]: borrow of moved value: `a.name``, labeled "value moved here" at the `..a` line. `a.logins` (a `u32`, `Copy`) is still readable ✓; `a.name` was a partial move (Lesson 07). Use `..a.clone()` or `..Default::default()` when you don't want to give up `a`.

### 2.3 Field errors

```rust
struct P { x: i32, y: i32 }
fn main() {
    let p = P { x: 1 };                      // E0063
    let q = P { x: 1, y: 2, z: 3 };          // E0560
    println!("{}", p.w);                     // E0609
}
```

✓ `error[E0063]: missing field `y` in initializer of `P``, `error[E0560]: struct `P` has no field named `z``, `error[E0609]: no field `w` on type `P``. Every field must be initialized; there is no default zero. That is a feature: no struct can ever be observed half-built.

### 2.4 Privacy: fields are private to the module by default

Visibility is the tool that protects your invariants (Lesson 03 introduced modules). A field without `pub` can be read and written only inside the module that defines the struct (and its descendants):

```rust
mod bank {
    pub struct Account { pub id: u32, balance: i64 }
    impl Account { pub fn new(id: u32) -> Self { Account { id, balance: 0 } } pub fn balance(&self) -> i64 { self.balance } }
}
fn main() {
    let mut a = bank::Account::new(1);
    println!("{}", a.balance());
    a.balance = 1_000_000;                           // E0616
}
```

✓ `error[E0616]: field `balance` of struct `Account` is private`. And building one by hand from outside also fails (✓ run separately): `bank::Account { id: 2, balance: 5 }` gives `error[E0451]: field `balance` of struct `Account` is private`. So the only way to get an `Account` is through `new`, which sets the invariants. **If a struct has an invariant ("balance never exceeds the limit", "email contains `@`"), make the fields private and expose constructors and methods that maintain it.** Public fields are right for plain data bundles with no invariants.

---

## 3. Methods and associated functions

An `impl` block attaches functions to a type. The first parameter decides what kind of function it is:

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
struct Rect { w: u32, h: u32 }
impl Rect {
    const UNIT: Rect = Rect { w: 1, h: 1 };
    fn new(w: u32, h: u32) -> Self { Self { w, h } }          // associated function (no self)
    fn area(&self) -> u32 { self.w * self.h }                  // shared borrow
    fn scale(&mut self, k: u32) { self.w *= k; self.h *= k; }  // exclusive borrow
    fn into_square(self) -> Rect { let s = self.w.max(self.h); Rect { w: s, h: s } }  // takes ownership
    fn can_hold(&self, other: &Rect) -> bool { self.w >= other.w && self.h >= other.h }
}
impl Rect { fn perimeter(&self) -> u32 { 2 * (self.w + self.h) } }   // multiple impl blocks are fine
fn main() {
    let mut r = Rect::new(2, 3);
    println!("{} {}", r.area(), Rect::area(&r));               // method call sugar vs explicit path
    r.scale(2);
    println!("{:?} {}", r, r.perimeter());
    println!("{}", r.can_hold(&Rect::UNIT));
    let s = r.into_square();
    println!("{s:?} {r:?}");                                    // r is Copy: still usable
}
```

✓ Output `6 6`, `Rect { w: 4, h: 6 } 20`, `true`, `Rect { w: 6, h: 6 } Rect { w: 4, h: 6 }`.

| Form | `self` means | Use when |
|---|---|---|
| `fn f(&self)` | `self: &Self`, a shared borrow | reading (getters, `area`, `len`) |
| `fn f(&mut self)` | `self: &mut Self`, exclusive borrow | changing the value in place |
| `fn f(self)` | `self: Self`, **takes ownership** | converting or consuming (`into_*`, builders, finishing an operation) |
| `fn f(args)` (no `self`) | an *associated function*, called as `Type::f(args)` | constructors (`new`), helpers that don't need an instance |

- **`Self`** is an alias for the type being implemented, so renaming the type doesn't touch the body. **Associated constants** (`const UNIT`) live in the impl too.
- **Method call syntax `r.area()`** is sugar: the compiler picks the receiver form and automatically takes `&`/`&mut`/derefs (Lesson 10, auto-ref). `Rect::area(&r)` is the same call written out ("UFCS"). You need the explicit form to disambiguate when two traits provide the same method name (Lesson 19).
- Naming convention: `new`, `with_*`, `from_*` construct; `as_*` is a cheap borrowed view; `to_*` is an expensive conversion; `into_*` consumes. `get*`/`is_*` read.
- Rust has **no constructors** in the C++ sense: `new` is a convention for an ordinary associated function. A constructor that can fail should return `Result` (or `Option`), as `Item::new` does in the practical challenge.

### Consuming methods end the value's life

```rust
#[derive(Debug)]
struct Token(String);
impl Token { fn into_inner(self) -> String { self.0 } fn len(&self) -> usize { self.0.len() } }
fn main() {
    let t = Token("abc".into());
    let s = t.into_inner();
    println!("{s} {}", t.len());
}
```

✓ `error[E0382]: borrow of moved value: `t``, with a note that `into_inner` "takes ownership of the receiver `self`". Used on purpose, this is how the type system encodes "this can happen once" (Lesson 07's `Ticket`, Lesson 15's typestates).

### Builders: consuming vs `&mut` chaining

```rust
#[derive(Debug, Default)]
struct Request { url: String, retries: u32, timeout_ms: u64, headers: Vec<(String, String)> }
struct RequestBuilder { req: Request }
impl RequestBuilder {
    fn new(url: &str) -> Self { RequestBuilder { req: Request { url: url.to_string(), timeout_ms: 30_000, ..Default::default() } } }
    fn retries(mut self, n: u32) -> Self { self.req.retries = n; self }                // consuming builder
    fn header(mut self, k: &str, v: &str) -> Self { self.req.headers.push((k.into(), v.into())); self }
    fn build(self) -> Request { self.req }
}
struct Counter { n: u32 }
impl Counter { fn inc(&mut self) -> &mut Self { self.n += 1; self } }                  // &mut builder
fn main() {
    let r = RequestBuilder::new("http://x").retries(3).header("a", "b").build();
    println!("{r:?}");
    let mut c = Counter { n: 0 };
    c.inc().inc().inc();
    println!("{}", c.n);
}
```

✓ Output `Request { url: "http://x", retries: 3, timeout_ms: 30000, headers: [("a", "b")] }` and `3`. The *consuming* builder (`mut self -> Self`) fits one-shot construction and cannot be used halfway by accident. The `&mut Self` style fits objects you keep configuring. `..Default::default()` uses the `Default` derive (Section 5) for the remaining fields.

### Other receivers

`self: Box<Self>` (✓ `fn run(self: Box<Self>) -> String` compiled and ran: calling `t.run()` on a `Box<Task>` consumes the box), `self: Rc<Self>`, `self: Pin<&mut Self>` exist for special cases (trait objects, shared ownership, async: Lessons 21, 23, 25). You won't need them yet.

---

## 4. Enums

An enum value is **exactly one** of its variants, and each variant may carry its own data:

```rust
#[derive(Debug)]
enum Shape { Circle { r: f64 }, Rect { w: f64, h: f64 }, Triangle(f64, f64, f64), Empty }
impl Shape {
    fn area(&self) -> f64 {
        match self {
            Shape::Circle { r } => std::f64::consts::PI * r * r,
            Shape::Rect { w, h } => w * h,
            Shape::Triangle(a, b, c) => { let s = (a + b + c) / 2.0; (s * (s - a) * (s - b) * (s - c)).sqrt() }
            Shape::Empty => 0.0,
        }
    }
}
fn main() {
    for s in [Shape::Circle { r: 1.0 }, Shape::Rect { w: 2.0, h: 3.0 }, Shape::Triangle(3.0, 4.0, 5.0), Shape::Empty] {
        println!("{:?} -> {:.3}", s, s.area());
    }
    println!("{}", std::mem::size_of::<Shape>());
}
```

✓ Output `Circle { r: 1.0 } -> 3.142`, `Rect { w: 2.0, h: 3.0 } -> 6.000`, `Triangle(3.0, 4.0, 5.0) -> 6.000`, `Empty -> 0.000`, and size `32` (the largest variant is three `f64`s, 24 bytes, plus the tag rounded to alignment 8; Lesson 09's "tag + largest payload"). That size is observed on this compiler, not guaranteed.

Variants can be unit-like (`Empty`), tuple-like (`Triangle(..)`) or struct-like (`Circle { r }`). `Option<T>` and `Result<T, E>` (Lesson 14) are ordinary enums from the standard library; there is no null in the language, only `Option`.

**Compare with other languages.** A C `union` plus a hand-managed tag, or a class hierarchy with a visitor, becomes one `enum` and one `match`. The compiler knows the full list of cases, which gives exhaustiveness checking:

```rust
enum Color { Red, Green, Blue }
fn name(c: Color) -> &'static str { match c { Color::Red => "r", Color::Green => "g" } }
fn main() { println!("{}", name(Color::Blue)); }
```

✓ `error[E0004]: non-exhaustive patterns: `Color::Blue` not covered`. When you add a variant later, every `match` that doesn't have a wildcard becomes a compile error pointing at what to update. That is the main reason to avoid `_ =>` arms on enums you own: the wildcard silences exactly the errors you want.

### Enums as C-like integers

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
#[repr(u8)]
enum Level { Low = 1, Mid = 5, High = 10 }
impl TryFrom<u8> for Level {
    type Error = u8;
    fn try_from(v: u8) -> Result<Self, u8> { match v { 1 => Ok(Level::Low), 5 => Ok(Level::Mid), 10 => Ok(Level::High), other => Err(other) } }
}
fn main() {
    println!("{} {}", Level::Mid as u8, Level::High as i32);
    println!("{:?} {:?}", Level::try_from(5), Level::try_from(6));
    println!("{}", Level::Low < Level::High);
    let mut v = vec![Level::High, Level::Low, Level::Mid]; v.sort(); println!("{v:?}");
}
```

✓ Output `5 10`, `Ok(Mid) Err(6)`, `true`, `[Low, Mid, High]`. Enum → integer is a safe `as` cast (Lesson 04). **Integer → enum is not**: write `TryFrom` and return an error for unknown values, because an enum variable holding a value that isn't a variant is undefined behavior if you force it with `unsafe` (`transmute`, Lesson 09's advice).

### Matching moves too

```rust
enum Msg { Text(String), Quit }
fn main() {
    let m = Msg::Text("hi".into());
    match m { Msg::Text(s) => println!("{s}"), Msg::Quit => {} }
    if let Msg::Text(s) = m { println!("{s}"); }
}
```

✓ `error[E0382]: use of moved value`. Binding `s` by value moves the `String` out of `m` (a partial move). Match on `&m` (or use `ref`-free default binding modes, Lesson 06) when you only want to look.

### Making illegal states unrepresentable

```rust
// Bad: two fields that can disagree
struct ConnBad { connected: bool, socket: Option<u32> }
// Good: the type only allows legal combinations
enum ConnGood { Disconnected, Connected { socket: u32 } }
fn main() {
    let bad = ConnBad { connected: true, socket: None };    // compiles, but is nonsense
    let good = ConnGood::Connected { socket: 7 };
    if let ConnGood::Connected { socket } = good { println!("{socket} {}", bad.connected && bad.socket.is_none()); }
    let _ = ConnGood::Disconnected;
}
```

✓ Output `7 true`: the `bad` value compiled and is internally contradictory ("connected" but no socket). In `ConnGood` that state cannot be written. This is the most valuable modeling habit in the course: when you have flags plus optional data, ask whether it's really an enum whose variants carry the data. The practical challenge's `State` enum does this: `Shipped { tracking }` has a tracking number, `Created` has none, and no code can read a tracking number that doesn't exist.

---

## 5. Derives, and what they mean

`#[derive(...)]` asks the compiler to generate a trait implementation (Lesson 19 explains traits). The ones you will use constantly:

| Derive | Gives you | Rule / catch |
|---|---|---|
| `Debug` | `{:?}`, `{:#?}` | every field must be `Debug`; the format is for developers, not a stable interface |
| `Clone` | `.clone()` | explicit, possibly expensive copy |
| `Copy` | implicit bitwise copy on assignment | only if all fields are `Copy` and the type has no `Drop` (E0204/E0184, Lesson 07); requires `Clone` |
| `PartialEq`, `Eq` | `==`, `!=` | `Eq` promises reflexivity (`x == x`); floats can't (NaN) |
| `PartialOrd`, `Ord` | `<`, `sort()`, `max()` | **lexicographic by field order**; enums compare by **variant order** first |
| `Hash` | usable as a `HashMap`/`HashSet` key | must agree with `Eq` (equal values hash equal) |
| `Default` | `T::default()`, `..Default::default()` | every field must be `Default` |

```rust
use std::collections::{HashMap, HashSet};
#[derive(Debug, PartialEq, Eq, PartialOrd, Ord, Hash, Clone, Default)]
struct Version { major: u32, minor: u32, tag: String }
#[derive(Debug, PartialEq, Eq, PartialOrd, Ord, Hash, Clone, Copy)]
enum Prio { Low, High(u8), Urgent { n: u8 } }
fn main() {
    let a = Version { major: 1, minor: 9, tag: "z".into() };
    let b = Version { major: 2, minor: 0, tag: "a".into() };
    println!("{} {:?}", a < b, a.cmp(&b));                         // field order = comparison order
    let mut ps = vec![Prio::Urgent { n: 1 }, Prio::High(9), Prio::Low, Prio::High(2)];
    ps.sort();
    println!("{ps:?}");                                             // variant order first, then fields
    let mut set = HashSet::new(); set.insert(a.clone()); set.insert(a.clone());
    println!("{}", set.len());
    let mut m: HashMap<Prio, u32> = HashMap::new(); *m.entry(Prio::Low).or_insert(0) += 1;
    println!("{:?} {:?}", m, Version::default());
    let c = Version { major: 3, ..Default::default() };
    println!("{c:?}");
}
```

✓ Output `true Less`, `[Low, High(2), High(9), Urgent { n: 1 }]`, `1`, `{Low: 1} Version { major: 0, minor: 0, tag: "" }`, `Version { major: 3, minor: 0, tag: "" }`.

Two warnings:

1. **Derived ordering is lexicographic in declaration order.** If you reorder the fields (or variants) of a type that derives `Ord`, you change how it sorts, silently. If sort order is part of your behavior (versions, priorities), write a test, or implement `Ord` by hand.
2. **Floats are not `Eq`/`Ord`.** ✓ `#[derive(PartialEq, PartialOrd, Eq, Ord)] struct Reading { celsius: f64 }` fails with `error[E0277]: the trait bound `f64: Eq` is not satisfied` (and the same for `Ord`). Without `Eq`/`Ord` it compiles, and ✓ `NAN == NAN` is `false` and `partial_cmp` returns `None`. Use `total_cmp` (Lesson 04) or a wrapper type that rejects NaN when you need a total order.

Derive `Copy` only for small, plain-data types where an implicit copy is what you want. A `Copy` type's by-value methods silently work on a copy (see the debugging challenge).

---

## 6. State machines with enums

Enum + `match` + methods is the standard way to model a protocol, connection, order, transaction or parser. When a method needs to *transform* the current state into the next one through `&mut self`, `std::mem::replace` (or `take`, Lesson 12) lets you move the old state out and put a new one in:

```rust
#[derive(Debug)]
enum Conn { Idle, Connecting { attempts: u32 }, Open { id: u64 }, Closed }
impl Conn {
    fn step(&mut self) {
        *self = match std::mem::replace(self, Conn::Closed) {   // take the old state out, put a placeholder in
            Conn::Idle => Conn::Connecting { attempts: 1 },
            Conn::Connecting { attempts } if attempts >= 3 => Conn::Closed,
            Conn::Connecting { attempts } => Conn::Connecting { attempts: attempts + 1 },
            Conn::Open { id } => Conn::Open { id },
            Conn::Closed => Conn::Closed,
        };
    }
}
fn main() {
    let mut c = Conn::Idle;
    for _ in 0..5 { c.step(); println!("{c:?} {}", matches!(c, Conn::Connecting { .. })); }
    let _ = Conn::Open { id: 1 };
}
```

✓ Output: `Connecting { attempts: 1 } true`, `Connecting { attempts: 2 } true`, `Connecting { attempts: 3 } true`, `Closed false`, `Closed false`. `matches!` is a macro for "does this match this pattern" (Lesson 34). The `mem::replace` placeholder (`Closed`) is only ever observed if the `match` panics partway, so choose a placeholder that is safe to leave behind.

**Two styles of transitions:**

- **Runtime-checked** (what `orderlab` below does): `pay`, `ship`, `deliver`, `cancel` check the current state and return `Err(InvalidTransition)`. The order always exists in some valid state, and a rejected action leaves it unchanged.
- **Compile-time-checked (typestate)**: each state is its own type and a transition consumes the old value (`fn ship(self) -> Shipped`), so an invalid transition does not compile. That is Lesson 15. Use runtime checks when states come from data (a database row, a network message); use typestate when the sequence is known statically.

---

## 7. Common mistakes and edge cases

- **Public fields on a type with invariants.** Anyone can break the invariant; make fields private and validate in constructors.
- **Flags + `Option`s instead of an enum.** See "illegal states".
- **Using `_ =>` on your own enums**, hiding the compiler's help when variants change. Use it for foreign or `#[non_exhaustive]` enums (Section 8) and for genuinely uniform "everything else" behavior.
- **`self` instead of `&self`/`&mut self`.** Consumes the value, or (for `Copy` types) works on a hidden copy (debugging challenge).
- **Deriving `Copy` on a growing type.** Adding a `String` field later breaks every place that relied on the copy.
- **Derived `Ord` after reordering fields or variants**; **derived `Hash` + hand-written `Eq`** that disagree (breaks `HashMap`).
- **Comparing floats with derived `PartialEq`** and expecting NaN to equal itself.
- **Large enum variants**: an enum is as big as its largest variant; one fat variant makes every value fat. Box the big payload (Lesson 23) if sizes matter; Clippy's `large_enum_variant` warns (Lesson 09).
- **Forgetting that `match` on a non-`Copy` payload moves it.** Match on `&value` to inspect.
- **Integer → enum via `as` or `transmute`.** Not allowed or unsound; use `TryFrom`.

---

## 8. API design: stability and `#[non_exhaustive]`

For a **library**, adding a field to a public struct or a variant to a public enum would normally break downstream code (struct literals, exhaustive matches). `#[non_exhaustive]` on a type tells *other crates* that more may come: they must write `..` in struct patterns, can't build the struct with a literal, and must include a wildcard arm when matching the enum. Within your own crate it has no effect, so test it from outside. ✓ I used a small external crate that matched on a `#[non_exhaustive]` enum without a wildcard and got:

```text
error[E0004]: non-exhaustive patterns: `&_` not covered
   = note: `OrderError` is marked as non-exhaustive, so a wildcard `_` is necessary to match exhaustively
```

and trying to assign a private field from outside gave `error[E0616]: field `state` of struct `Order` is private`. The practical challenge's `OrderError` is `#[non_exhaustive]` and its doctest checks that a wildcard-less match does not compile. Use it for error enums and for config/option structs you expect to grow; avoid it for closed sets (`Ordering`-like) where exhaustive matching is the point. (Lesson 33 returns to API evolution and semver.)

---

## 9. Performance implications

- Structs and enums have **no hidden cost**: no header, no vtable, no GC. A struct is its fields (plus padding), an enum is tag + largest payload (Lesson 09). Methods are plain functions; calls usually inline.
- By-value `self` on a large struct copies/moves `size_of::<Self>()` bytes (Lesson 07). Pass `&self` unless you intend to consume.
- `Copy` types are passed in registers or copied; don't derive `Copy` on big structs, as implicit copies are easy to cause by accident.
- `match` on an enum compiles to a tag test or jump table, comparable to a C `switch`.
- Derived `Clone` clones every field (deep for `String`/`Vec`): `.clone()` on a large struct is an allocation per heap field. Derived `PartialEq`/`Ord` compare field by field and stop at the first difference.
- A `bool` + `Option<T>` pair is typically *larger* than a single enum carrying the same information; I did not measure the sizes in this lesson, but Lesson 09's tools (`size_of`) would show it.

---

## 10. Security and reliability

- **Validate at construction.** If every `Item` is created through `Item::new`, you never need to re-check "quantity > 0" or "SKU is not empty" downstream. Input from the network, a file or a database goes through a fallible constructor (`TryFrom`, `FromStr`, `new -> Result`), never into a struct literal with public fields.
- **Overflow in totals.** `Order::new` uses checked arithmetic (`checked_mul`, `checked_add`) and returns `Overflow`; in release builds plain `+` would wrap silently (Lesson 04). For money, token amounts and balances, prefer checked or saturating operations and an integer smallest-unit representation (cents, lamports, wei) over floats (Lessons 48–57).
- **Illegal transitions are errors, not panics**, and a rejected transition must leave the state unchanged (tested). In a financial or on-chain context, "partially applied" is the dangerous state.
- **Don't expose representation you may need to change.** Private fields plus methods let you change the layout later without breaking users.
- Rust's types do not make the *business logic* correct: `pay` accepting only the exact total is a rule I chose, and a different rule (partial payments, refunds, rounding) is a design decision the compiler cannot make for you.

---

## 11. Exercises

Try each before opening the answer. They are not marked complete unless you tell me you did them.

1. **(Easy)** Which of these fails, and with which error? (a) `P { x: 1 }` for `struct P { x: i32, y: i32 }`; (b) `p.z` on that struct; (c) reading a private field from another module.
2. **(Easy)** Given `#[derive(Clone, Copy)] struct Counter { n: u32 }`, what does `fn inc(mut self) -> u32 { self.n += 1; self.n }` do when you call `c.inc()` twice on a `let c`?
3. **(Medium)** Write `enum Shape` with `Circle`, `Square` and a method `perimeter(&self) -> f64`. Then add a `Triangle` variant. What does the compiler tell you?
4. **(Medium)** Predict the order after sorting `[Prio::Urgent{n:1}, Prio::High(9), Prio::Low, Prio::High(2)]` where `Prio` is `enum { Low, High(u8), Urgent{n:u8} }` with `#[derive(PartialOrd, Ord, ...)]`. Then predict what changes if you move `Low` to the end of the declaration.
5. **(Medium)** Make `struct Reading { celsius: f64 }` sortable. Two ways.
6. **(Medium)** Rewrite `struct Job { running: bool, started_at: Option<u64>, finished_at: Option<u64> }` as an enum that cannot represent "not running but has a start time and no finish time".
7. **(Hard)** Write a consuming builder and a `&mut` builder for a `Server { port: u16, workers: u8 }`. When is each better? Why can't you call `build()` twice on the consuming one?
8. **(Hard)** Why does `#[non_exhaustive]` need a test from *outside* the crate? Show how you'd test that a wildcard-less match fails to compile.

<details markdown="1">
<summary>Answers</summary>

1. ✓ (a) `E0063: missing field`; (b) `E0609: no field`; (c) `E0616: field is private` (and `E0451` if you try to build the struct with a literal).
2. ✓ With `Copy`, `inc(mut self)` receives a **copy**: it prints `1 1 0` for `c.inc(), c.inc(), c.n`. The counter never advances (see the debugging challenge). Without `Copy` the second call is E0382.
3. Add `Triangle` and every `match` without a wildcard fails with ✓ `E0004: non-exhaustive patterns: ... not covered`, naming the missing variant. That is the point.
4. ✓ `[Low, High(2), High(9), Urgent { n: 1 }]`: variant declaration order first, then fields. If `Low` is moved to the end of the declaration, derived `Ord` makes `Low` the *largest* variant, so it would sort last (derived behavior as documented; I did not run this reordering).
5. (a) Implement `PartialOrd` with `partial_cmp` and sort with `sort_by(|a, b| a.celsius.partial_cmp(&b.celsius).unwrap())` (panics on NaN, so only if NaN is impossible); or (b) `sort_by(|a, b| a.celsius.total_cmp(&b.celsius))`, which gives a total order including NaN (Lesson 04). Deriving `Ord` is rejected ✓ (E0277).
6. `enum Job { Queued, Running { started_at: u64 }, Finished { started_at: u64, finished_at: u64 } }`. "Finished without a start" and "running with a finish time" cannot be written. (Design answer; not compiled.)
7. Consuming: `fn workers(mut self, n: u8) -> Self`, ending in `fn build(self) -> Server`; `build(self)` moves the builder, so a second call is E0382 (the same effect as the `Token::into_inner` example ✓). `&mut Self` chaining suits a long-lived, repeatedly adjusted object and lets you `build(&self)` many times if `Server: Clone`; the consuming style suits one-shot construction and moving owned values (like `String`s) in without cloning.
8. Attributes like `non_exhaustive` only affect *other* crates, so a test inside the same crate always passes the match. Use a doctest (a separate crate from the library's point of view) marked `compile_fail` ✓ (as `OrderError` does), or a `trybuild`-style test (an external tool I haven't run in this course). I confirmed the real error code ✓ (`E0004`) with a separate binary crate that depends on `orderlab`.

</details>

---

## 12. Debugging challenge

A teammate wrote a counter and says "it works, but every call returns 1":

```rust
#[derive(Debug, Clone, Copy)]
struct Counter { n: u32 }
impl Counter { fn inc(mut self) -> u32 { self.n += 1; self.n } }
fn main() { let c = Counter { n: 0 }; println!("{} {} {}", c.inc(), c.inc(), c.n); }
```

✓ It compiles and prints `1 1 0`. Their first attempt (before `mut self`) was `fn inc(self) -> u32 { self.n += 1; self.n }`, which gave ✓ `error[E0594]: cannot assign to `self.n`, as `self` is not declared as mutable`, and the compiler's `help: consider changing this to be mutable` led them to `mut self`. Why is this program wrong although the compiler accepts it, and what is the right fix? What happens if `Counter` is **not** `Copy`?

<details markdown="1">
<summary>Solution</summary>

`inc(mut self)` takes the counter **by value**. Because `Counter` is `Copy`, each call `c.inc()` silently copies `c`, increments the copy, and returns the copy's value; the original `c` is never modified, so `c.n` stays 0. The compiler's suggestion fixed the *error*, not the *design*: the method was meant to change `c`, which needs `&mut self`.

```rust
#[derive(Debug, Clone, Copy)]
struct Counter { n: u32 }
impl Counter { fn inc(&mut self) -> u32 { self.n += 1; self.n } }
fn main() { let mut c = Counter { n: 0 }; println!("{} {} {}", c.inc(), c.inc(), c.n); }
```

✓ Prints `1 2 2` (and `c` must be declared `let mut`; without it ✓ `error[E0596]: cannot borrow `c` as mutable, as it is not declared as mutable`). If `Counter` is **not** `Copy`, the buggy version fails loudly instead: ✓ the second `c.inc()` is `error[E0382]: use of moved value: `c``. So `Copy` turned a compile error into a silent logic bug. Lesson: take the receiver the operation *means* (`&mut self` to change in place), be wary of "just add `mut`" fixes, and derive `Copy` only when copies are what you want.

</details>

---

## 13. Practical challenge: `orderlab`

Create a library crate `orderlab` (edition 2024) modeling an order lifecycle:

1. `Money(u64)` (cents; `Copy`, `Default`, ordered, hashable) with `ZERO`, `from_cents`, `cents`, `checked_add`, `checked_mul(u32)` and `Display` as `12.34`.
2. `OrderId(pub u64)`; `Item` with **private** fields and `Item::new(sku, qty, unit_price) -> Result<Item, OrderError>` (trims the SKU, rejects empty SKU and zero quantity) and `line_total() -> Option<Money>`.
3. `State` enum: `Created`, `Paid { amount }`, `Shipped { tracking }`, `Delivered`, `Cancelled { reason }`, with `name()` and `is_terminal()`.
4. `Order` with private fields, `Order::new(id, items) -> Result<Order, OrderError>` (non-empty, checked total) and `pay` (exact total only), `ship`, `deliver`, `cancel` (allowed from `Created` or `Paid` only). Rejected transitions must leave the order unchanged.
5. `#[non_exhaustive] enum OrderError` with `Display` and `std::error::Error`.
6. Tests for validation, overflow, the happy path, every rejected transition (checking the order is unchanged), cancel rules, error messages; a doctest showing normal use; and `compile_fail` doctests showing that (a) `order.state = ...` does not compile from outside the crate and (b) matching `OrderError` without a wildcard does not compile outside the crate.
7. It must pass `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, and `cargo test --locked` (run `cargo generate-lockfile` once first).

<details markdown="1">
<summary>Solution</summary>

**`src/lib.rs`**

```rust
//! orderlab: structs, enums and methods used to make illegal states unrepresentable.
//!
//! ```
//! use orderlab::{Item, Money, Order, OrderId};
//!
//! let items = vec![Item::new("pen", 3, Money::from_cents(150))?];
//! let mut order = Order::new(OrderId(1), items)?;
//! assert_eq!(order.total().to_string(), "4.50");
//! order.pay(Money::from_cents(450))?;
//! order.ship("TRACK-1")?;
//! order.deliver()?;
//! assert_eq!(order.state().name(), "delivered");
//! # Ok::<(), orderlab::OrderError>(())
//! ```

use std::error::Error;
use std::fmt;

/// An amount of money in cents. A newtype so cents and quantities cannot be mixed up.
#[derive(Debug, Clone, Copy, Default, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct Money(u64);

impl Money {
    pub const ZERO: Money = Money(0);

    pub const fn from_cents(cents: u64) -> Self {
        Money(cents)
    }

    pub const fn cents(self) -> u64 {
        self.0
    }

    pub fn checked_add(self, other: Money) -> Option<Money> {
        self.0.checked_add(other.0).map(Money)
    }

    pub fn checked_mul(self, qty: u32) -> Option<Money> {
        self.0.checked_mul(u64::from(qty)).map(Money)
    }
}

impl fmt::Display for Money {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}.{:02}", self.0 / 100, self.0 % 100)
    }
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct OrderId(pub u64);

/// Everything that can go wrong. `#[non_exhaustive]` lets a library add variants later without
/// breaking downstream `match` expressions: outside this crate they must write a wildcard arm.
///
/// ```compile_fail
/// use orderlab::OrderError;
/// fn describe(e: &OrderError) -> &'static str {
///     match e {
///         OrderError::EmptyOrder => "empty",
///         OrderError::EmptySku => "sku",
///         OrderError::ZeroQuantity { .. } => "qty",
///         OrderError::Overflow => "overflow",
///         OrderError::WrongAmount { .. } => "amount",
///         OrderError::EmptyTracking => "tracking",
///         OrderError::EmptyReason => "reason",
///         OrderError::InvalidTransition { .. } => "transition",
///     }
/// }
/// ```
#[derive(Debug, Clone, PartialEq, Eq)]
#[non_exhaustive]
pub enum OrderError {
    EmptyOrder,
    EmptySku,
    ZeroQuantity {
        sku: String,
    },
    Overflow,
    WrongAmount {
        expected: Money,
        got: Money,
    },
    EmptyTracking,
    EmptyReason,
    InvalidTransition {
        from: &'static str,
        action: &'static str,
    },
}

impl fmt::Display for OrderError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            OrderError::EmptyOrder => write!(f, "an order needs at least one item"),
            OrderError::EmptySku => write!(f, "item SKU must not be empty"),
            OrderError::ZeroQuantity { sku } => write!(f, "item {sku:?} has quantity zero"),
            OrderError::Overflow => write!(f, "order total overflows"),
            OrderError::WrongAmount { expected, got } => {
                write!(f, "wrong payment: expected {expected}, got {got}")
            }
            OrderError::EmptyTracking => write!(f, "tracking number must not be empty"),
            OrderError::EmptyReason => write!(f, "cancellation reason must not be empty"),
            OrderError::InvalidTransition { from, action } => {
                write!(f, "cannot {action} an order that is {from}")
            }
        }
    }
}

impl Error for OrderError {}

/// A validated line item. Fields are private so the invariants checked in `new` always hold.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Item {
    sku: String,
    qty: u32,
    unit_price: Money,
}

impl Item {
    pub fn new(sku: &str, qty: u32, unit_price: Money) -> Result<Item, OrderError> {
        let sku = sku.trim();
        if sku.is_empty() {
            return Err(OrderError::EmptySku);
        }
        if qty == 0 {
            return Err(OrderError::ZeroQuantity {
                sku: sku.to_owned(),
            });
        }
        Ok(Item {
            sku: sku.to_owned(),
            qty,
            unit_price,
        })
    }

    pub fn sku(&self) -> &str {
        &self.sku
    }

    pub fn qty(&self) -> u32 {
        self.qty
    }

    /// `None` if `qty * unit_price` overflows.
    pub fn line_total(&self) -> Option<Money> {
        self.unit_price.checked_mul(self.qty)
    }
}

/// Where an order is in its life. Each variant carries only the data that exists in that state.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum State {
    Created,
    Paid { amount: Money },
    Shipped { tracking: String },
    Delivered,
    Cancelled { reason: String },
}

impl State {
    pub fn name(&self) -> &'static str {
        match self {
            State::Created => "created",
            State::Paid { .. } => "paid",
            State::Shipped { .. } => "shipped",
            State::Delivered => "delivered",
            State::Cancelled { .. } => "cancelled",
        }
    }

    pub fn is_terminal(&self) -> bool {
        matches!(self, State::Delivered | State::Cancelled { .. })
    }
}

/// An order. Fields are private: the only way to change `state` is through the transition
/// methods, which check the current state first. A rejected transition leaves the order unchanged.
///
/// ```compile_fail
/// use orderlab::{Item, Money, Order, OrderId, State};
/// let item = Item::new("pen", 1, Money::from_cents(100)).unwrap();
/// let mut order = Order::new(OrderId(1), vec![item]).unwrap();
/// order.state = State::Delivered; // private field
/// ```
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Order {
    id: OrderId,
    items: Vec<Item>,
    total: Money,
    state: State,
}

impl Order {
    pub fn new(id: OrderId, items: Vec<Item>) -> Result<Order, OrderError> {
        if items.is_empty() {
            return Err(OrderError::EmptyOrder);
        }
        let mut total = Money::ZERO;
        for item in &items {
            let line = item.line_total().ok_or(OrderError::Overflow)?;
            total = total.checked_add(line).ok_or(OrderError::Overflow)?;
        }
        Ok(Order {
            id,
            items,
            total,
            state: State::Created,
        })
    }

    pub fn id(&self) -> OrderId {
        self.id
    }

    pub fn items(&self) -> &[Item] {
        &self.items
    }

    pub fn total(&self) -> Money {
        self.total
    }

    pub fn state(&self) -> &State {
        &self.state
    }

    fn invalid(&self, action: &'static str) -> OrderError {
        OrderError::InvalidTransition {
            from: self.state.name(),
            action,
        }
    }

    /// `Created -> Paid`; the amount must equal the total exactly.
    pub fn pay(&mut self, amount: Money) -> Result<(), OrderError> {
        match self.state {
            State::Created if amount == self.total => {
                self.state = State::Paid { amount };
                Ok(())
            }
            State::Created => Err(OrderError::WrongAmount {
                expected: self.total,
                got: amount,
            }),
            _ => Err(self.invalid("pay")),
        }
    }

    /// `Paid -> Shipped`.
    pub fn ship(&mut self, tracking: &str) -> Result<(), OrderError> {
        let tracking = tracking.trim();
        match self.state {
            State::Paid { .. } if tracking.is_empty() => Err(OrderError::EmptyTracking),
            State::Paid { .. } => {
                self.state = State::Shipped {
                    tracking: tracking.to_owned(),
                };
                Ok(())
            }
            _ => Err(self.invalid("ship")),
        }
    }

    /// `Shipped -> Delivered`.
    pub fn deliver(&mut self) -> Result<(), OrderError> {
        match self.state {
            State::Shipped { .. } => {
                self.state = State::Delivered;
                Ok(())
            }
            _ => Err(self.invalid("deliver")),
        }
    }

    /// `Created | Paid -> Cancelled`. Once shipped, an order can no longer be cancelled.
    pub fn cancel(&mut self, reason: &str) -> Result<(), OrderError> {
        let reason = reason.trim();
        match self.state {
            State::Created | State::Paid { .. } if reason.is_empty() => {
                Err(OrderError::EmptyReason)
            }
            State::Created | State::Paid { .. } => {
                self.state = State::Cancelled {
                    reason: reason.to_owned(),
                };
                Ok(())
            }
            _ => Err(self.invalid("cancel")),
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    fn money(c: u64) -> Money {
        Money::from_cents(c)
    }

    fn order() -> Order {
        let items = vec![
            Item::new("pen", 3, money(150)).unwrap(),
            Item::new("ink", 1, money(1_000)).unwrap(),
        ];
        Order::new(OrderId(7), items).unwrap()
    }

    #[test]
    fn money_display_and_ordering() {
        assert_eq!(money(0).to_string(), "0.00");
        assert_eq!(money(5).to_string(), "0.05");
        assert_eq!(money(1234).to_string(), "12.34");
        assert_eq!(Money::from_cents(u64::MAX).to_string().len(), 21); // no panic
        assert!(money(1) < money(2));
        assert_eq!(Money::default(), Money::ZERO);
    }

    #[test]
    fn money_checked_ops() {
        assert_eq!(money(u64::MAX).checked_add(money(1)), None);
        assert_eq!(money(u64::MAX).checked_mul(2), None);
        assert_eq!(money(10).checked_mul(3), Some(money(30)));
    }

    #[test]
    fn item_validation() {
        assert_eq!(Item::new("  ", 1, money(1)), Err(OrderError::EmptySku));
        assert_eq!(
            Item::new(" pen ", 0, money(1)),
            Err(OrderError::ZeroQuantity {
                sku: "pen".to_string()
            })
        );
        let item = Item::new(" pen ", 2, money(5)).unwrap();
        assert_eq!(
            (item.sku(), item.qty(), item.line_total()),
            ("pen", 2, Some(money(10)))
        );
    }

    #[test]
    fn order_creation() {
        let o = order();
        assert_eq!(o.total(), money(1_450));
        assert_eq!(o.state(), &State::Created);
        assert_eq!((o.id(), o.items().len()), (OrderId(7), 2));
        assert_eq!(Order::new(OrderId(1), vec![]), Err(OrderError::EmptyOrder));
    }

    #[test]
    fn order_total_overflow_is_an_error() {
        let big = Item::new("x", u32::MAX, money(u64::MAX)).unwrap();
        assert_eq!(Order::new(OrderId(1), vec![big]), Err(OrderError::Overflow));
        let a = Item::new("a", 1, money(u64::MAX)).unwrap();
        let b = Item::new("b", 1, money(1)).unwrap();
        assert_eq!(
            Order::new(OrderId(1), vec![a, b]),
            Err(OrderError::Overflow)
        );
    }

    #[test]
    fn happy_path() {
        let mut o = order();
        o.pay(money(1_450)).unwrap();
        assert_eq!(
            o.state(),
            &State::Paid {
                amount: money(1_450)
            }
        );
        o.ship(" T-1 ").unwrap();
        assert_eq!(
            o.state(),
            &State::Shipped {
                tracking: "T-1".to_string()
            }
        );
        o.deliver().unwrap();
        assert!(o.state().is_terminal());
    }

    #[test]
    fn rejected_transitions_leave_the_order_unchanged() {
        let mut o = order();
        let before = o.clone();
        assert_eq!(
            o.pay(money(1)),
            Err(OrderError::WrongAmount {
                expected: money(1_450),
                got: money(1)
            })
        );
        assert_eq!(
            o.ship("T"),
            Err(OrderError::InvalidTransition {
                from: "created",
                action: "ship"
            })
        );
        assert!(o.deliver().is_err());
        assert_eq!(o.cancel("  "), Err(OrderError::EmptyReason));
        assert_eq!(o, before);

        o.pay(money(1_450)).unwrap();
        let paid = o.clone();
        assert_eq!(
            o.pay(money(1_450)),
            Err(OrderError::InvalidTransition {
                from: "paid",
                action: "pay"
            })
        );
        assert_eq!(o.ship(" "), Err(OrderError::EmptyTracking));
        assert_eq!(o, paid);
    }

    #[test]
    fn cancel_rules() {
        let mut a = order();
        a.cancel("changed mind").unwrap();
        assert_eq!(
            a.state(),
            &State::Cancelled {
                reason: "changed mind".to_string()
            }
        );
        assert!(a.cancel("again").is_err());
        assert!(a.pay(money(1_450)).is_err());

        let mut b = order();
        b.pay(money(1_450)).unwrap();
        b.cancel("refund").unwrap();
        assert_eq!(b.state().name(), "cancelled");

        let mut c = order();
        c.pay(money(1_450)).unwrap();
        c.ship("T").unwrap();
        assert_eq!(
            c.cancel("too late"),
            Err(OrderError::InvalidTransition {
                from: "shipped",
                action: "cancel"
            })
        );
    }

    #[test]
    fn error_messages() {
        let e = OrderError::InvalidTransition {
            from: "paid",
            action: "pay",
        };
        assert_eq!(e.to_string(), "cannot pay an order that is paid");
        let e = OrderError::WrongAmount {
            expected: money(100),
            got: money(5),
        };
        assert_eq!(e.to_string(), "wrong payment: expected 1.00, got 0.05");
        let boxed: Box<dyn Error> = Box::new(OrderError::EmptyOrder);
        assert_eq!(boxed.to_string(), "an order needs at least one item");
    }
}
```

✓ With this exact file: `cargo fmt --check` clean, `cargo clippy --all-targets --locked -- -D warnings` clean, `cargo test --locked` → 9 unit tests, 1 usage doctest and 2 `compile_fail` doctests passed. I also built a separate binary crate against it and confirmed the compiler errors ✓ `E0616` (private field) and `E0004` (wildcard required for a `#[non_exhaustive]` enum).

**Design notes.**

- **Parse, don't validate.** `Item::new` and `Order::new` are the only ways to get these types, so a value that exists is already valid. `Money` is a newtype over `u64` cents, so cents can't be mixed up with quantities, and all arithmetic is checked.
- **State carries data.** `Shipped { tracking }` has the tracking number; `Created` has none, so there's nothing to forget to clear. `is_terminal` uses `matches!`.
- **Transitions** `match self.state` on the *current* state with guards (`State::Created if amount == self.total`), assign the new state only on success, and return a typed `Err` otherwise. No branch mutates before it has finished checking, which is why rejected transitions leave the order unchanged (tested by comparing with a clone). `match self.state` (not `&self.state`) works here because the patterns bind nothing by value: `Paid { .. }` ignores the data.
- **Errors are values**, with `Display` for humans and a source-free `std::error::Error` impl so callers can use `?` and `Box<dyn Error>` (Lesson 26 designs this in depth). `#[non_exhaustive]` lets the crate add error kinds later without a breaking change.
- **Derives:** `Debug, Clone, PartialEq, Eq` on everything that needs comparison in tests; `Copy` only on the small `Money` and `OrderId`. `Order` is `Clone` (cheap enough here, and the tests rely on it) but not `Copy`.
- **No `unwrap` in library code**; the doctest uses `?`.
- The `compile_fail` doctests pass if the snippet fails to compile for *any* reason; I checked the intended error codes with the external crate above, not inside the doctests themselves.

</details>

---

## 14. Interview questions

<details markdown="1">
<summary>Answers</summary>

1. **Struct vs enum?** A struct holds all its fields at once (product type); an enum holds exactly one variant, each with its own data (sum type).
2. **What are the `self` forms?** `&self` (read), `&mut self` (modify in place), `self` (consume), or none (an associated function called as `Type::f`).
3. **What does `Self` mean?** An alias for the implementing type, so renames don't touch method bodies.
4. **Is there a constructor?** No: `new` is a convention for an ordinary associated function. Fallible construction returns `Result`/`Option`.
5. **Why make struct fields private?** To protect invariants: the only way to build or change the value is through functions that check them. (E0616/E0451 if bypassed.)
6. **What does `..old` do in a struct literal?** Fills the remaining fields from `old`, moving/copying each; non-`Copy` fields are moved out of `old` (partial move, E0382 if you use them after).
7. **What does exhaustiveness checking give you?** Every `match` on an enum must cover all variants; adding a variant makes incomplete matches fail to compile (E0004). Avoid wildcards on your own enums to keep that benefit.
8. **What does derived `Ord` compare?** Fields lexicographically in declaration order; enums by variant order first, then payload. Reordering changes the result.
9. **Why can't `f64` derive `Eq`/`Ord`?** `NaN != NaN`, so equality isn't reflexive and there's no total order; use `total_cmp` or a wrapper.
10. **When do you derive `Copy`?** Small plain-data types where implicit copies are wanted and all fields are `Copy`; a `Copy` type's by-value methods operate on copies, which hides bugs like forgetting `&mut self`.
11. **How do you model "illegal states unrepresentable"?** Use an enum whose variants carry exactly the data valid in that state instead of flags and `Option`s that can disagree.
12. **How big is an enum?** Tag plus the largest variant payload, rounded for alignment, minus niche savings (Lesson 09).
13. **What is `#[non_exhaustive]` for?** Lets a library add variants/fields without breaking downstream code; downstream crates must use a wildcard/`..`. It has no effect inside the defining crate.
14. **How do you change a state stored behind `&mut self` when the next state needs data from the old one?** Move the old state out with `mem::replace`/`take` (or restructure), compute the new one, assign it back; or use a consuming typestate API.
15. **Builder: `self` or `&mut self`?** Consuming (`self -> Self`) for one-shot construction; `&mut Self` chaining for reusable, repeatedly modified builders.

</details>

---

## 15. Key takeaways

- Structs are products, enums are sums. Pick the shape that makes illegal states unrepresentable.
- Private fields + validating constructors protect invariants; public fields are for plain data. Errors you will see: E0063, E0560, E0609, E0616, E0451, E0599, E0004.
- Method receivers state intent: `&self` read, `&mut self` change, `self` consume. A consuming method ends the value's life (E0382 afterwards). Beware by-value `self` on `Copy` types.
- Derive `Debug` everywhere, `Clone`/`PartialEq`/`Eq`/`Hash`/`Ord`/`Default` as needed; know that `Ord` follows declaration order and that floats can't be `Eq`/`Ord`.
- Don't wildcard-match your own enums; do use `#[non_exhaustive]` on public types that will grow.
- State machines as enums: runtime-checked transitions that leave the value unchanged on error (this lesson), compile-time typestates (Lesson 15).
- No hidden costs: layout is fields (structs) or tag + largest payload (enums); use checked arithmetic for money-like quantities.

**Connections.** Lesson 06's patterns now destructure your own types; Lesson 07's moves explain struct update, consuming methods and match-moves; Lesson 09's layout rules size your enums; Lesson 10's `&self`/`&mut self` become the receiver table; Lesson 12's `take`/`replace` drive state transitions. Next: Lesson 14 (`Option`, `Result`, and error-handling basics) is these enums in the standard library; Lesson 15 adds newtypes and typestates; Lesson 16 collections; Lesson 19 traits, which `derive` generated for you here; Lesson 26 error design; Lessons 49 and 53–57 model transactions and on-chain accounts with exactly these tools. See the [roadmap](00_roadmap.md) and the [course index](index.md).

---

## 16. References

- *The Rust Programming Language*, ch. 5 "Using Structs to Structure Related Data" and ch. 6 "Enums and Pattern Matching": <https://doc.rust-lang.org/book/ch05-00-structs.html>, <https://doc.rust-lang.org/book/ch06-00-enums.html>
- *The Rust Reference*, "Structs", "Enumerations", "Associated items", "Visibility and privacy": <https://doc.rust-lang.org/reference/items/structs.html>, <https://doc.rust-lang.org/reference/items/enumerations.html>, <https://doc.rust-lang.org/reference/items/associated-items.html>, <https://doc.rust-lang.org/reference/visibility-and-privacy.html>
- *The Rust Reference*, "Attributes: `non_exhaustive`" and "Derive": <https://doc.rust-lang.org/reference/attributes/type_system.html#the-non_exhaustive-attribute>, <https://doc.rust-lang.org/reference/attributes/derive.html>
- *Rust API Guidelines* (naming conventions `as_`/`to_`/`into_`, builders, future-proofing): <https://rust-lang.github.io/api-guidelines/>
- Rust standard library documentation: `std::mem::replace`, `std::cmp::Ord` (derive semantics), `std::hash::Hash`, `Default`: <https://doc.rust-lang.org/std/mem/fn.replace.html>, <https://doc.rust-lang.org/std/cmp/trait.Ord.html>, <https://doc.rust-lang.org/std/hash/trait.Hash.html>, <https://doc.rust-lang.org/std/default/trait.Default.html>
- Rust error code index: E0004, E0063, E0382, E0451, E0560, E0594, E0599, E0609, E0616: <https://doc.rust-lang.org/error_codes/E0004.html>, <https://doc.rust-lang.org/error_codes/E0063.html>, <https://doc.rust-lang.org/error_codes/E0451.html>, <https://doc.rust-lang.org/error_codes/E0616.html>
- Alexis King, "Parse, don't validate" (the design idea behind validating constructors): <https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/>
- Rust by Example, "Structures" and "Enums": <https://doc.rust-lang.org/rust-by-example/custom_types.html>

The URLs above are the standard documentation locations; I did not open every one during this session, so if a link has moved, search the docs site for the page title. Unchecked from memory: the exact blog URL for "Parse, don't validate".
