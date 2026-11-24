---
layout: default
title: "Lesson 15 — Newtypes, typestate, and state machines"
---

# Lesson 15 — Newtypes, typestate, and state machines

**Module 5 · Data modeling** · Status: generated · Edition used in examples: **2024** · External dependencies: none

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux. As before, ✓ marks outputs and diagnostics I actually ran; unmarked statements are documented behavior I did not run. Everything is **stable Rust**.

Builds on [Lesson 13](13_structs_enums_methods.md) (structs, enums, private fields, runtime state machines), [Lesson 14](14_option_result_errors.md) (`Result`, `FromStr`, error types), [Lesson 09](09_memory_layout.md) (`repr(transparent)`, zero-sized types, niches) and [Lesson 07](07_ownership_and_moves.md) (a move ends the old value's life).

---

## 1. The idea: let the compiler check your domain rules

Lesson 13 said "make illegal states unrepresentable". This lesson gives you the two main tools for doing it with *types*:

1. **Newtypes**: wrap a primitive in a one-field struct so that a `UserId` is not an `OrderId`, meters are not feet, and a validated email is not just any `String`.
2. **Typestate**: encode the *state* of a value in its *type*, so calling an operation in the wrong state is a compile error instead of a runtime check.

Both cost nothing at run time (✓ shown below). Their price is more types and more boilerplate, so the skill is knowing when they pay off.

---

## 2. Newtypes

### 2.1 Distinct types for distinct meanings

```rust
#[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]
#[repr(transparent)]
struct Meters(f64);
#[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]
struct Feet(f64);
fn climb(height: Meters) -> Meters { Meters(height.0 + 1.0) }
fn main() {
    let h = Meters(10.0);
    println!("{:?}", climb(h));
    println!("{} {}", std::mem::size_of::<Meters>(), std::mem::size_of::<f64>());
    let f = Feet(30.0);
    println!("{:?}", climb(f));
}
```

✓ `error[E0308]: mismatched types` on `climb(f)` ("expected `Meters`, found `Feet`"). Removing that line, ✓ the program prints `Meters(11.0)` and `8 8`: the wrapper has the same size as the `f64` (and `#[repr(transparent)]`, Lesson 09, additionally guarantees the same layout and ABI, which matters for FFI). This is the pattern behind famous unit-confusion failures: both values are just numbers, and only a type system can tell them apart.

Use newtypes for:

- **Units and quantities**: `Meters`, `Seconds`, `Cents`, `Lamports`, `Wei`.
- **Identifiers**: `UserId(u64)` vs `OrderId(u64)`, so you cannot pass one where the other is expected.
- **Validated data**: `Email`, `Port`, `NonEmptyString`, `Percent`: a value of the type *is proof* the check passed.
- **Untrusted vs trusted**: `Unvalidated<T>` and `Validated<T>`, `Sanitized` strings.
- **Implementing traits for foreign types** (Section 2.3).

### 2.2 Validated newtypes: parse, don't validate

The pattern from Lesson 13, in full: a private field, a fallible constructor (`FromStr`/`TryFrom`/`new -> Result`), and read-only accessors. After construction nobody re-checks, because the type *cannot* hold an invalid value.

```rust
use std::{fmt, str::FromStr};
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct Email(String);
#[derive(Debug, PartialEq, Eq)]
pub enum EmailError { Empty, MissingAt, EmptyPart }
impl fmt::Display for EmailError { fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result { write!(f, "{self:?}") } }
impl std::error::Error for EmailError {}
impl FromStr for Email {
    type Err = EmailError;
    fn from_str(s: &str) -> Result<Self, EmailError> {
        let s = s.trim();
        if s.is_empty() { return Err(EmailError::Empty); }
        let (local, domain) = s.split_once('@').ok_or(EmailError::MissingAt)?;
        if local.is_empty() || domain.is_empty() { return Err(EmailError::EmptyPart); }
        Ok(Email(s.to_ascii_lowercase()))
    }
}
impl Email { pub fn as_str(&self) -> &str { &self.0 } pub fn into_inner(self) -> String { self.0 } }
impl AsRef<str> for Email { fn as_ref(&self) -> &str { &self.0 } }
fn send(to: &Email) -> String { format!("sent to {}", to.as_str()) }
fn main() {
    println!("{:?}", "Ana@Example.com".parse::<Email>());
    println!("{:?} {:?} {:?}", "".parse::<Email>(), "nope".parse::<Email>(), "@x".parse::<Email>());
    let e: Email = "a@b".parse().unwrap();
    println!("{}", send(&e));
}
```

✓ Output: `Ok(Email("ana@example.com"))`, `Err(Empty) Err(MissingAt) Err(EmptyPart)`, `sent to a@b`. (This `Email` check is deliberately minimal; real email validation is far more involved and mostly belongs to sending a confirmation message. The point is the *shape*: `send` takes `&Email` and never needs to validate.)

The privacy is what makes it work. If the inner field is public, anyone can build an invalid value:

```rust
mod m { pub struct Percent(u8); impl Percent { pub fn new(v: u8) -> Option<Percent> { (v <= 100).then_some(Percent(v)) } pub fn get(&self) -> u8 { self.0 } } }
fn main() { let p = m::Percent(250); let q = m::Percent::new(50).unwrap(); println!("{}", q.0); }
```

✓ Two errors: `error[E0603]: tuple struct constructor `Percent` is private` (a tuple struct with a private field has a private constructor outside its module) and `error[E0616]: field `0` of struct `Percent` is private`. Only `Percent::new` can make one.

Provide the usual conversions deliberately: `TryFrom<u8>` to construct, `From<Percent> for u8` to get out, `Display`, and `AsRef` for cheap borrowing. ✓ A `Percent` with `TryFrom<u8>` printed `Ok(Percent(100)) Err(OutOfRange(101))` and `42% 42` (exercise 2).

The standard library already has validated newtypes: ✓ `NonZeroU32::new(0)` is `None`, and `Option<NonZeroU32>` is 4 bytes (the zero bit pattern is the niche, Lesson 09):

```rust
use std::num::NonZeroU32;
fn main() {
    let n = NonZeroU32::new(5).unwrap();
    println!("{} {:?} {}", n, NonZeroU32::new(0), std::mem::size_of::<Option<NonZeroU32>>());
    println!("{}", n.get() * 2);
}
```

✓ Output `5 None 4` and `10`.

### 2.3 The orphan rule, and the newtype escape hatch

You may implement a trait for a type only if **the trait or the type is local to your crate** (this prevents two crates from giving conflicting implementations of the same trait for the same type). So you cannot make `Vec<i32>` implement `Display`:

```rust
use std::fmt;
impl fmt::Display for Vec<i32> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result { write!(f, "{} items", self.len()) }
}
fn main() {}
```

✓ `error[E0117]: only traits defined in the current crate can be implemented for types defined outside of the crate`. Wrap the foreign type in a local newtype, and the impl is allowed:

```rust
use std::fmt;
struct Scores(Vec<i32>);
impl fmt::Display for Scores {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result { write!(f, "{} scores, max {:?}", self.0.len(), self.0.iter().max()) }
}
fn main() { println!("{}", Scores(vec![3, 9, 4])); }
```

✓ Output `3 scores, max Some(9)`. (Lesson 20 covers coherence and the orphan rule in full.)

### 2.4 The `Deref` trap

It is tempting to implement `Deref` on a newtype so all the inner type's methods "just work". That also *removes* the protection the newtype was supposed to give. A read-only `Deref` to a slice is fine for a sorted wrapper. `DerefMut` to the `Vec` is not:

```rust
use std::ops::{Deref, DerefMut};
struct Sorted(Vec<i32>);
impl Sorted { fn new(mut v: Vec<i32>) -> Self { v.sort(); Sorted(v) } fn is_sorted_ok(&self) -> bool { self.0.windows(2).all(|w| w[0] <= w[1]) } }
impl Deref for Sorted { type Target = [i32]; fn deref(&self) -> &[i32] { &self.0 } }
struct Leaky(Vec<i32>);
impl Deref for Leaky { type Target = Vec<i32>; fn deref(&self) -> &Vec<i32> { &self.0 } }
impl DerefMut for Leaky { fn deref_mut(&mut self) -> &mut Vec<i32> { &mut self.0 } }
fn main() {
    let s = Sorted::new(vec![3, 1, 2]);
    println!("{:?} {} {}", &*s, s.len(), s.is_sorted_ok());      // read-only slice view keeps the invariant
    let mut l = Leaky(vec![1, 2, 3]);
    l.push(0);                                                   // invariant "ascending" silently broken
    println!("{:?}", *l);
}
```

✓ Output `[1, 2, 3] 3 true` and `[1, 2, 3, 0]`: the `Leaky` wrapper's "ascending" invariant was broken with no complaint. And ✓ with only the slice `Deref`, `s.push(0)` fails: `error[E0599]: no method named `push` found for struct `Sorted``. The Rust API guidelines say `Deref` is for smart pointers; for domain newtypes prefer explicit methods (`as_slice`, `iter`) over `Deref`, and never `DerefMut` when there is an invariant.

### 2.5 Typed IDs with `PhantomData`, and a derive trap

When you have many id kinds, a generic `Id<T>` avoids writing a newtype per kind. The `T` does not appear in any field, so use `PhantomData<T>` (Lesson 22):

```rust
use std::marker::PhantomData;
struct User; struct Order;
#[derive(Debug)]
struct Id<T> { raw: u64, _t: PhantomData<T> }
impl<T> Id<T> { fn new(raw: u64) -> Self { Id { raw, _t: PhantomData } } }
fn find_user(id: Id<User>) -> u64 { id.raw }
fn main() {
    let u: Id<User> = Id::new(1);
    let o: Id<Order> = Id::new(1);
    println!("{}", find_user(u));
    println!("{}", find_user(o));
}
```

✓ `E0308` on the second call (expected `Id<User>`, found `Id<Order>`). Now the trap: adding `#[derive(Clone, Copy)]` to `Id<T>`.

```rust
use std::marker::PhantomData;
struct User;                     // note: User is not Clone or Copy
#[derive(Debug, Clone, Copy)]
struct Id<T> { raw: u64, _t: PhantomData<T> }
fn take(_: Id<User>) {}
fn main() {
    let u: Id<User> = Id { raw: 1, _t: PhantomData };
    take(u);
    take(u);
}
```

✓ `error[E0382]: use of moved value: `u``, plus the compiler's hint: "consider manually implementing `Clone` to avoid undesired bounds". `#[derive(Clone, Copy)]` on a generic struct adds the bound `T: Clone` / `T: Copy`, even though `T` is only a marker. `User` is neither, so `Id<User>` is not `Copy` and the second `take(u)` is a use after move. Fix: implement the traits by hand without the bound (this is the **debugging challenge**):

```rust
use std::marker::PhantomData;
struct User;
#[derive(Debug)]
struct Id<T> { raw: u64, _t: PhantomData<fn() -> T> }
impl<T> Clone for Id<T> { fn clone(&self) -> Self { *self } }
impl<T> Copy for Id<T> {}
impl<T> PartialEq for Id<T> { fn eq(&self, o: &Self) -> bool { self.raw == o.raw } }
impl<T> Eq for Id<T> {}
fn take(_: Id<User>) {}
fn main() {
    let u: Id<User> = Id { raw: 1, _t: PhantomData };
    take(u); take(u);
    println!("{}", u == u.clone());
    println!("{} {}", std::mem::size_of::<Id<User>>(), std::mem::size_of::<u64>());
}
```

✓ Output `true` and `8 8`. (`PhantomData<fn() -> T>` also avoids tying `Send`/`Sync` and drop-check behavior to `T`, because a function pointer type is always `Send + Sync`; Lesson 22 explains why you would choose this form. The derive also adds `T: Debug` to the `Debug` derive.)

---

## 3. Typestate

### 3.1 Encode the state in the type

A *typestate* API gives each state its own type (usually a type parameter with zero-sized marker types), puts each operation only in the `impl` block of the states where it is legal, and makes transitions **consume** the old value and return the new one:

```rust
use std::marker::PhantomData;
struct Open; struct Closed;
struct Door<S> { name: String, _s: PhantomData<S> }
impl Door<Closed> {
    fn new(name: &str) -> Self { Door { name: name.into(), _s: PhantomData } }
    fn open(self) -> Door<Open> { println!("{} opens", self.name); Door { name: self.name, _s: PhantomData } }
}
impl Door<Open> {
    fn close(self) -> Door<Closed> { println!("{} closes", self.name); Door { name: self.name, _s: PhantomData } }
    fn walk_through(&self) { println!("walking through {}", self.name); }
}
fn main() {
    let d = Door::new("front");
    let d = d.open();
    d.walk_through();
    let d = d.close();
    println!("{} {}", std::mem::size_of::<Door<Open>>(), std::mem::size_of::<String>());
    let _ = d;
}
```

✓ Output `front opens`, `walking through front`, `front closes`, `24 24`: `Door<Open>` is exactly as big as its `String` (the marker and `PhantomData` are zero-sized, Lesson 09). The state costs no memory and no checks: the "if (!open) error" branch does not exist because the call cannot be written.

The invalid uses are compile errors ✓:

```rust
let d: Door<Closed> = Door { _s: PhantomData };
d.walk_through();            // error[E0599]: no method named `walk_through` found for struct `Door<Closed>`
```

```rust
let o = d.open();
o.walk_through();
let again = d.open();        // error[E0382]: use of moved value: `d`
```

(Each ran as a small program, with the `impl` blocks shown above.) The second is Lesson 07's move semantics doing the work: `open(self)` consumed `d`, so the "closed door" no longer exists once it has been opened. That is how typestate also prevents *reuse*: a transfer that has been submitted cannot be submitted again.

### 3.2 Typestate builders: required fields checked at compile time

```rust
use std::marker::PhantomData;
struct NoUrl; struct HasUrl;
struct ReqBuilder<U> { url: Option<String>, retries: u32, _u: PhantomData<U> }
#[derive(Debug)] struct Req { url: String, retries: u32 }
impl ReqBuilder<NoUrl> {
    fn new() -> Self { ReqBuilder { url: None, retries: 0, _u: PhantomData } }
    fn url(self, u: &str) -> ReqBuilder<HasUrl> { ReqBuilder { url: Some(u.into()), retries: self.retries, _u: PhantomData } }
}
impl<U> ReqBuilder<U> { fn retries(mut self, n: u32) -> Self { self.retries = n; self } }   // available in every state
impl ReqBuilder<HasUrl> {
    fn build(self) -> Req { Req { url: self.url.expect("HasUrl implies Some"), retries: self.retries } }
}
fn main() {
    let r = ReqBuilder::new().retries(2).url("http://x").build();
    println!("{r:?}");
}
```

✓ Output `Req { url: "http://x", retries: 2 }`. Forgetting the URL is ✓ `error[E0599]: no method named `build` found for struct `ReqBuilder<NoUrl>``. Note the `expect` after `HasUrl`: the state proves the `Option` is `Some`, but the compiler doesn't know that, so there is one `expect` whose message states the reason. A tighter design stores the URL in the state type itself (`HasUrl(String)`), removing the `Option` and the `expect`.

### 3.3 Sealing the set of states

If `State` is a public trait, downstream crates can implement it and invent states you did not plan for. The **sealed trait** pattern makes the trait unimplementable outside your crate by requiring a supertrait from a private module:

```rust
mod sealed { pub trait Sealed {} }
pub trait State: sealed::Sealed { const NAME: &'static str; }
pub struct A; pub struct B;
impl sealed::Sealed for A {} impl sealed::Sealed for B {}
impl State for A { const NAME: &'static str = "a"; }
impl State for B { const NAME: &'static str = "b"; }
fn name<S: State>() -> &'static str { S::NAME }
fn main() { println!("{} {}", name::<A>(), name::<B>()); }
```

✓ Output `a b`. From another crate, `impl statelab::State for Rogue` fails ✓ with `error[E0277]: the trait bound `Rogue: statelab::sealed::Sealed` is not satisfied` (the practical challenge's doctest covers it). Sealing also lets you add methods to the trait later without breaking anyone.

### 3.4 When typestate is the wrong tool

Typestate needs the state to be known **at compile time**. It does not fit when:

- **The state comes from data**: a database row, a network message, user input. You cannot choose `Door<Open>` vs `Door<Closed>` from a runtime value. Use the runtime enum of Lesson 13, or an enum that wraps the typed states.
- **You need a collection of mixed states.** `Vec<Door<Open>>` can't also hold `Door<Closed>`:

```rust
use std::marker::PhantomData;
struct Open; struct Closed;
struct Door<S> { id: u32, _s: PhantomData<S> }
enum AnyDoor { Open(Door<Open>), Closed(Door<Closed>) }
fn main() {
    let doors = vec![AnyDoor::Open(Door { id: 1, _s: PhantomData }), AnyDoor::Closed(Door { id: 2, _s: PhantomData })];
    for d in &doors { match d { AnyDoor::Open(d) => println!("{} open", d.id), AnyDoor::Closed(d) => println!("{} closed", d.id) } }
    let bad = vec![Door::<Open> { id: 1, _s: PhantomData }, Door::<Closed> { id: 2, _s: PhantomData }];
}
```

✓ The `AnyDoor` enum works (`1 open`, `2 closed` before the final error), and the mixed `vec!` fails with `error[E0308]: mismatched types`. The enum is the bridge between static and dynamic states, as `AnyTransfer` is in the practical challenge.

- **There are very many states or transitions**, or transitions depend on many events: the type-level encoding becomes verbose. A table-driven runtime machine is better.
- **The API is for beginners or has a stable surface**: typestate types appear in public signatures and error messages and are harder to read.

---

## 4. Pure state machines: `(State, Event) -> Result<State, Error>`

For protocols and anything driven by data, make the machine a **pure function** from the current state and an event to the next state. No hidden mutation, no I/O, easy to test exhaustively, deterministic (which is exactly what blockchain state transitions require):

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
enum State { Idle, Running { pid: u32 }, Stopped { code: i32 } }
#[derive(Debug, Clone, Copy)]
enum Event { Start(u32), Exit(i32), Reset }
#[derive(Debug, PartialEq, Eq)]
struct Invalid { from: State, event: &'static str }
fn next(s: State, e: Event) -> Result<State, Invalid> {
    match (s, e) {
        (State::Idle, Event::Start(pid)) => Ok(State::Running { pid }),
        (State::Running { .. }, Event::Exit(code)) => Ok(State::Stopped { code }),
        (State::Stopped { .. }, Event::Reset) => Ok(State::Idle),
        (from, Event::Start(_)) => Err(Invalid { from, event: "start" }),
        (from, Event::Exit(_)) => Err(Invalid { from, event: "exit" }),
        (from, Event::Reset) => Err(Invalid { from, event: "reset" }),
    }
}
fn main() {
    let mut s = State::Idle;
    for e in [Event::Start(7), Event::Start(8), Event::Exit(0), Event::Reset, Event::Reset] {
        match next(s, e) { Ok(n) => { println!("{s:?} --{e:?}--> {n:?}"); s = n; } Err(x) => println!("rejected: {x:?}") }
    }
}
```

✓ Output:

```text
Idle --Start(7)--> Running { pid: 7 }
rejected: Invalid { from: Running { pid: 7 }, event: "start" }
Running { pid: 7 } --Exit(0)--> Stopped { code: 0 }
Stopped { code: 0 } --Reset--> Idle
rejected: Invalid { from: Idle, event: "reset" }
```

The three trailing arms handle "event not valid in this state" per event; the `match` on the pair is exhaustive (the compiler would reject a missing combination). A rejected event leaves `s` unchanged in the caller because `next` takes the state by value and returns a new one. Because `next` is a function of its inputs only, you can test every `(state, event)` pair (a table test), replay a recorded event log to rebuild any state, and share the same function between a client and a validator.

---

## 5. Common mistakes and edge cases

- **Public inner field on a validating newtype.** Anyone can bypass the check (E0603 or silent misuse). Keep it private.
- **`Deref`/`DerefMut` on a newtype with an invariant** (Section 2.4). Prefer explicit methods.
- **`#[derive(Clone, Copy, PartialEq)]` on `Id<T>`/`Marker<T>`** adds `T: Clone` etc. Implement by hand for marker generics (Section 2.5).
- **Forgetting `#[repr(transparent)]`** on a newtype you pass across FFI: without it the layout of a one-field struct is not formally guaranteed to match the inner type (in practice it does; Lesson 09).
- **Newtypes without conversions.** If callers can't get the value out or in conveniently, they will work around the type. Provide `new`/`TryFrom`, `get`/`into_inner`/`as_*`, `Display`.
- **Arithmetic on newtypes**: `Meters + Meters` doesn't compile until you implement `std::ops::Add` (Lesson 19). Be deliberate about which operations make sense (adding two `UserId`s does not).
- **Typestate for runtime-driven state**: it can't work; use an enum.
- **Transitions that lose data**: when a transition fails, give the value back (`Failed { transfer, error }` in the practical challenge) instead of consuming it for nothing.
- **Too many states in the type parameter** (`Conn<A, B, C, D>`): combinatorial explosion. Group them into one state type per phase.
- **`expect` after a state proof.** Fine, but explain the invariant in the message, or store the data in the state type.

---

## 6. Performance implications

- Newtypes and zero-sized state markers compile away. ✓ `Meters` is 8 bytes like `f64`; `Door<Open>` is the size of its `String`; `Id<User>` is 8 bytes like `u64`; `Transfer<Draft>` and `Transfer<Validated>` have equal size (tested in the practical challenge). Generated code is the same as with raw primitives; I did not inspect the assembly in this lesson.
- Typestate is monomorphized: each state gets its own copy of the methods defined on it, so many states times many generic methods means more compiled code. For a few states this is negligible.
- A typestate transition moves the value (a `memcpy` of its size, Lesson 07), usually optimized away; large state-carrying structs can be passed by `Box` if needed.
- Validated newtypes shift checks to construction time: validate once at the boundary instead of on every use.
- `NonZero*` types enable the niche optimization (`Option<NonZeroU32>` is 4 bytes ✓).

---

## 7. Security and reliability

- **Validated types at trust boundaries.** Parse addresses, amounts and identifiers into types as they enter the program (the `Address` parser rejects bad length, missing prefix and non-hex digits, and returns errors instead of panicking, even for multi-byte characters: tested). Downstream code then cannot receive a malformed value.
- **Make dangerous operations require a proof type.** For example, a function that executes a transfer can take `Transfer<Validated>`, so it is impossible to call it on an unchecked one. (The type system proves the *order of operations*, not that your validation logic is correct.)
- **Don't confuse unit types**: mixing lamports and SOL, wei and gwei, or cents and dollars is a classic source of financial bugs. Newtype the unit; convert explicitly with checked arithmetic.
- **Zero is special.** `Amount(NonZeroU64)` makes a zero-value transfer unrepresentable; decide per domain which values are legal.
- **Sealed state traits** stop other crates from adding states that bypass your checks.
- **Typestate is not access control.** Anyone with the right type can call the next method; authorization still needs real checks. And `unsafe` or a public constructor can forge a state: keep constructors private.
- **Deterministic, pure transitions** (Section 4) are easier to audit, fuzz (Lesson 40) and replay, which matters for consensus-critical code (Lessons 48–57).
- A toy "state machine" in a tutorial is not a protocol design: real payment or chain logic needs idempotency, retries, reorg handling and persistence that this lesson does not cover.

---

## 8. Exercises

Try each before opening the answer. They are not marked complete unless you tell me you did them.

1. **(Easy)** Why does `climb(Feet(30.0))` fail to compile if `climb` takes `Meters`, and what does it cost at run time to have the distinct types?
2. **(Easy)** Write `Percent(u8)` that can only hold 0 to 100, with `TryFrom<u8>`, `From<Percent> for u8`, and `Display` as `42%`.
3. **(Medium)** Make `Display` work for `Vec<i32>` in your own crate. What error do you get, and what are two fixes?
4. **(Medium)** A `Sorted(Vec<i32>)` newtype implements `Deref<Target = [i32]>` but not `DerefMut`. What can callers do with it, and what can't they, and why does that matter?
5. **(Medium)** Extend the `Door` typestate with a `Locked` state: `Closed -> lock -> Locked -> unlock -> Closed`. What happens if you call `open` on a locked door?
6. **(Medium)** Why does `Id<User>` stop being `Copy` when `User` isn't, and what is the fix?
7. **(Hard)** Write the pure transition function for a three-state traffic light with events `Timer` and `Fault`, and say how you would test every `(state, event)` pair.
8. **(Hard)** You must store transfers loaded from a database in a `Vec` and process them by their stored status. How do you combine typestate with that?

<details markdown="1">
<summary>Answers</summary>

1. ✓ `Meters` and `Feet` are distinct types even though both wrap `f64`; there is no implicit conversion, so you get `E0308`. At run time it costs nothing: `size_of::<Meters>() == size_of::<f64>()` (8) ✓.
2. ✓ (run)
   ```rust
   use std::fmt;
   #[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
   pub struct Percent(u8);
   #[derive(Debug, PartialEq, Eq)]
   pub struct OutOfRange(pub u8);
   impl TryFrom<u8> for Percent {
       type Error = OutOfRange;
       fn try_from(v: u8) -> Result<Self, OutOfRange> { if v <= 100 { Ok(Percent(v)) } else { Err(OutOfRange(v)) } }
   }
   impl From<Percent> for u8 { fn from(p: Percent) -> u8 { p.0 } }
   impl fmt::Display for Percent { fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result { write!(f, "{}%", self.0) } }
   ```
   `try_from(100)` is `Ok(Percent(100))`, `try_from(101)` is `Err(OutOfRange(101))`, and `42` prints as `42%` and converts back to `42` ✓.
3. ✓ `error[E0117]: only traits defined in the current crate can be implemented for types defined outside of the crate`. Fix 1: a local newtype (`struct Scores(Vec<i32>)`) with the `Display` impl ✓. Fix 2: define your own trait (`trait Summary { fn summary(&self) -> String; }`) and implement it for `Vec<i32>` (allowed because the trait is local; not run in this lesson).
4. They can read: `len`, `iter`, indexing, `binary_search`, anything on `[i32]` that takes `&self` (✓ `s.len()` worked). They cannot mutate: ✓ `s.push(0)` is `E0599`. That preserves the "sorted" invariant. With `DerefMut` to `Vec` (the `Leaky` example) the invariant was broken ✓ (`[1, 2, 3, 0]`).
5. ✓ Add `struct Locked;`, `impl Door<Closed> { fn lock(self) -> Door<Locked> }` and `impl Door<Locked> { fn unlock(self) -> Door<Closed> }`. The chain `d.lock().unlock().open()` compiled; calling `open` on a `Door<Locked>` gave `error[E0599]: no method named `open` found for struct `Door<Locked>``.
6. `#[derive(Clone, Copy)]` on `struct Id<T>` generates `impl<T: Clone> Clone` and `impl<T: Copy> Copy`; `User` is neither, so `Id<User>` isn't `Copy` and reusing it is `E0382` ✓. Implement `Clone`/`Copy`/`PartialEq` by hand without bounds on `T` (✓ the `8 8` / `true` example).
7. For example `enum Light { Red, Green, Yellow, Flashing }`, `enum Ev { Timer, Fault }` and `fn next(l: Light, e: Ev) -> Light` (Fault from anywhere goes to `Flashing`; Timer cycles `Red -> Green -> Yellow -> Red`; Timer on `Flashing` stays). To test every pair, loop over arrays of all states and all events (`for s in ALL_STATES { for e in ALL_EVENTS { ... } }`) and assert the expected result from a small table; because the function is pure and `match` is exhaustive, nothing else is needed. (Design answer; not compiled.)
8. Keep typed states for the code that knows the state statically, and put them in an enum for storage: `enum AnyTransfer { Draft(Transfer<Draft>), ... }`. On load, build the matching variant from the stored status (a `TryFrom<Row>` that returns an error for unknown statuses); when you need to operate, `match` on the enum and call the typed method in the branch where the state is known. Provide a run-time `submit(self, id) -> Result<AnyTransfer, (AnyTransfer, WrongState)>` wrapper for callers who can't match (✓ `AnyTransfer::submit` in the practical challenge).

</details>

---

## 9. Debugging challenge

A teammate defines typed ids so that a `UserId` and an `OrderId` can't be mixed up, and wants them to be `Copy`:

```rust
use std::marker::PhantomData;
struct User;                     // a plain marker type
#[derive(Debug, Clone, Copy)]
struct Id<T> { raw: u64, _t: PhantomData<T> }
fn take(_: Id<User>) {}
fn main() {
    let u: Id<User> = Id { raw: 1, _t: PhantomData };
    take(u);
    take(u);
}
```

✓ It fails with `error[E0382]: use of moved value: `u``. "But I derived `Copy`!" Why does the compiler disagree, and what is the fix?

<details markdown="1">
<summary>Solution</summary>

`#[derive(Clone, Copy)]` on a generic struct produces `impl<T: Clone> Clone for Id<T>` and `impl<T: Copy> Copy for Id<T>`: the derive assumes the type parameter is stored and so must be `Copy` itself. `User` is a plain struct that is neither, so **`Id<User>` is not `Copy`** even though `Id` contains only a `u64` and a zero-sized marker. The first `take(u)` therefore moves `u`, and the second is a use after move. (The compiler hints: "consider manually implementing `Clone` to avoid undesired bounds".)

Fix: implement the traits by hand, without a bound on `T`:

```rust
impl<T> Clone for Id<T> { fn clone(&self) -> Self { *self } }
impl<T> Copy for Id<T> {}
impl<T> PartialEq for Id<T> { fn eq(&self, o: &Self) -> bool { self.raw == o.raw } }
impl<T> Eq for Id<T> {}
```

and use `PhantomData<fn() -> T>` so that the marker doesn't imply ownership of a `T` (affecting auto traits and drop-check, Lesson 22). ✓ With these, `take(u); take(u);` compiles, `u == u.clone()` is `true`, and `size_of::<Id<User>>()` is 8. The alternative is to derive `Clone, Copy` on the marker types too (`#[derive(Clone, Copy)] struct User;`), which satisfies the derived bound but forces every marker to carry those derives forever.

</details>

---

## 10. Practical challenge: `statelab`

Create a library crate `statelab` (edition 2024) with a transfer lifecycle:

1. Newtypes: `Address([u8; 20])` with `FromStr` for `0x` plus 40 hex digits (accept upper and lower case) and `Display` as lowercase hex; typed errors for missing prefix, wrong length and bad digit (with its position); no panics on any input, including multi-byte characters. `Amount(NonZeroU64)` with `new(u64) -> Result` rejecting zero; `Nonce(pub u64)`; `TxId(pub u64)`.
2. A sealed `State` trait (with `const NAME`) implemented for `Draft`, `Validated`, `Submitted { id }`, `Confirmed { id, block }`.
3. `Transfer<S: State>` with private fields and accessors. `Transfer<Draft>::new(from, to, amount, nonce)` rejects `from == to`. `validate(self, balance) -> Result<Transfer<Validated>, Failed<Draft>>` where `Failed` hands the draft back with the error; `submit` (Validated → Submitted), `confirm` (Submitted → Confirmed).
4. `AnyTransfer` enum over the four typed states, `From` impls for each, `state_name()`, and a run-time `submit` that returns the original value and a `WrongState` error on misuse.
5. Tests: address round trip and case, every address error, `Amount` zero and the `Option` size, same-address rejection, the whole happy path, failed validation returning the draft (and a successful retry), zero-cost state markers (`size_of`), and `AnyTransfer` run-time checks. `compile_fail` doctests for (a) `submit` on a draft, (b) reuse after a transition, (c) implementing `State` outside the crate, and a passing doctest of the happy path.
6. It must pass `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, and `cargo test --locked` (run `cargo generate-lockfile` once first).

<details markdown="1">
<summary>Solution</summary>

**`src/lib.rs`**

```rust
//! statelab: newtypes for validated values and a typestate transfer lifecycle.
//!
//! A transfer moves through `Draft -> Validated -> Submitted -> Confirmed`. Each state is a
//! type, so calling an operation in the wrong state does not compile:
//!
//! ```
//! use statelab::{Address, Amount, Nonce, Transfer, TxId};
//!
//! let from: Address = "0x1111111111111111111111111111111111111111".parse()?;
//! let to: Address = "0x2222222222222222222222222222222222222222".parse()?;
//! let draft = Transfer::new(from, to, Amount::new(50)?, Nonce(0))?;
//! let validated = draft.validate(100).map_err(|f| f.error)?;
//! let submitted = validated.submit(TxId(7));
//! let confirmed = submitted.confirm(1_234);
//! assert_eq!(confirmed.state().block, 1_234);
//! # Ok::<(), Box<dyn std::error::Error>>(())
//! ```
//!
//! Submitting a draft is a compile error:
//!
//! ```compile_fail
//! use statelab::{Address, Amount, Nonce, Transfer, TxId};
//! let a: Address = "0x1111111111111111111111111111111111111111".parse().unwrap();
//! let b: Address = "0x2222222222222222222222222222222222222222".parse().unwrap();
//! let draft = Transfer::new(a, b, Amount::new(1).unwrap(), Nonce(0)).unwrap();
//! let _ = draft.submit(TxId(1)); // no `submit` on Transfer<Draft>
//! ```
//!
//! So is using a transfer after it was consumed by a transition:
//!
//! ```compile_fail
//! use statelab::{Address, Amount, Nonce, Transfer};
//! let a: Address = "0x1111111111111111111111111111111111111111".parse().unwrap();
//! let b: Address = "0x2222222222222222222222222222222222222222".parse().unwrap();
//! let draft = Transfer::new(a, b, Amount::new(1).unwrap(), Nonce(0)).unwrap();
//! let _first = draft.validate(10);
//! let _second = draft.validate(10); // `draft` was moved
//! ```
//!
//! The set of states is closed (sealed), so downstream crates cannot invent new ones:
//!
//! ```compile_fail
//! struct Rogue;
//! impl statelab::State for Rogue {
//!     const NAME: &'static str = "rogue";
//! }
//! ```

use std::fmt;
use std::num::NonZeroU64;
use std::str::FromStr;

/// A 20-byte address, written as `0x` followed by 40 hex digits.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)]
pub struct Address([u8; 20]);

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum AddressError {
    MissingPrefix,
    BadLength { got: usize },
    BadDigit { index: usize },
}

impl fmt::Display for AddressError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AddressError::MissingPrefix => write!(f, "address must start with 0x"),
            AddressError::BadLength { got } => {
                write!(f, "address needs 40 hex digits after 0x, got {got}")
            }
            AddressError::BadDigit { index } => write!(f, "invalid hex digit at position {index}"),
        }
    }
}

impl std::error::Error for AddressError {}

fn hex_val(b: u8) -> Option<u8> {
    match b {
        b'0'..=b'9' => Some(b - b'0'),
        b'a'..=b'f' => Some(b - b'a' + 10),
        b'A'..=b'F' => Some(b - b'A' + 10),
        _ => None,
    }
}

impl FromStr for Address {
    type Err = AddressError;

    fn from_str(s: &str) -> Result<Self, AddressError> {
        let digits = s.strip_prefix("0x").ok_or(AddressError::MissingPrefix)?;
        let bytes = digits.as_bytes();
        if bytes.len() != 40 {
            return Err(AddressError::BadLength { got: bytes.len() });
        }
        let mut out = [0u8; 20];
        for (i, pair) in bytes.chunks_exact(2).enumerate() {
            let hi = hex_val(pair[0]).ok_or(AddressError::BadDigit { index: 2 * i })?;
            let lo = hex_val(pair[1]).ok_or(AddressError::BadDigit { index: 2 * i + 1 })?;
            out[i] = (hi << 4) | lo;
        }
        Ok(Address(out))
    }
}

impl fmt::Display for Address {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "0x")?;
        for b in self.0 {
            write!(f, "{b:02x}")?;
        }
        Ok(())
    }
}

/// A strictly positive amount (in the smallest unit). Zero is unrepresentable.
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct Amount(NonZeroU64);

impl Amount {
    pub fn new(units: u64) -> Result<Amount, TransferError> {
        NonZeroU64::new(units)
            .map(Amount)
            .ok_or(TransferError::ZeroAmount)
    }

    pub fn get(self) -> u64 {
        self.0.get()
    }
}

/// A per-sender sequence number. A plain newtype: any `u64` is valid.
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct Nonce(pub u64);

/// An identifier assigned when a transfer is submitted.
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct TxId(pub u64);

#[derive(Debug, Clone, PartialEq, Eq)]
#[non_exhaustive]
pub enum TransferError {
    ZeroAmount,
    SameAddress,
    InsufficientFunds { needed: u64, available: u64 },
}

impl fmt::Display for TransferError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            TransferError::ZeroAmount => write!(f, "amount must be greater than zero"),
            TransferError::SameAddress => write!(f, "sender and recipient must differ"),
            TransferError::InsufficientFunds { needed, available } => {
                write!(f, "insufficient funds: need {needed}, have {available}")
            }
        }
    }
}

impl std::error::Error for TransferError {}

mod sealed {
    pub trait Sealed {}
}

/// Marker trait for the states of a [`Transfer`]. Sealed: only this crate can add states.
pub trait State: sealed::Sealed {
    const NAME: &'static str;
}

/// Created but not yet checked against the sender's balance.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Draft;

/// Checked against the sender's balance.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Validated;

/// Sent to the network; carries the id it was given.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Submitted {
    pub id: TxId,
}

/// Included in a block.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Confirmed {
    pub id: TxId,
    pub block: u64,
}

macro_rules! state {
    ($ty:ty, $name:literal) => {
        impl sealed::Sealed for $ty {}
        impl State for $ty {
            const NAME: &'static str = $name;
        }
    };
}
state!(Draft, "draft");
state!(Validated, "validated");
state!(Submitted, "submitted");
state!(Confirmed, "confirmed");

/// A transfer in state `S`. The fields are private; the only way to obtain a value is through
/// [`Transfer::new`] and the transitions below.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Transfer<S: State> {
    from: Address,
    to: Address,
    amount: Amount,
    nonce: Nonce,
    state: S,
}

/// A rejected transition hands the transfer back so the caller can fix it and retry.
#[derive(Debug)]
pub struct Failed<S: State> {
    pub transfer: Transfer<S>,
    pub error: TransferError,
}

impl<S: State> Transfer<S> {
    pub fn from(&self) -> Address {
        self.from
    }

    pub fn to(&self) -> Address {
        self.to
    }

    pub fn amount(&self) -> Amount {
        self.amount
    }

    pub fn nonce(&self) -> Nonce {
        self.nonce
    }

    pub fn state(&self) -> &S {
        &self.state
    }

    pub fn state_name(&self) -> &'static str {
        S::NAME
    }
}

impl Transfer<Draft> {
    pub fn new(
        from: Address,
        to: Address,
        amount: Amount,
        nonce: Nonce,
    ) -> Result<Transfer<Draft>, TransferError> {
        if from == to {
            return Err(TransferError::SameAddress);
        }
        Ok(Transfer {
            from,
            to,
            amount,
            nonce,
            state: Draft,
        })
    }

    /// Checks the sender's balance. On failure the draft is returned inside [`Failed`].
    pub fn validate(self, balance: u64) -> Result<Transfer<Validated>, Failed<Draft>> {
        if balance < self.amount.get() {
            let error = TransferError::InsufficientFunds {
                needed: self.amount.get(),
                available: balance,
            };
            return Err(Failed {
                transfer: self,
                error,
            });
        }
        Ok(Transfer {
            from: self.from,
            to: self.to,
            amount: self.amount,
            nonce: self.nonce,
            state: Validated,
        })
    }
}

impl Transfer<Validated> {
    pub fn submit(self, id: TxId) -> Transfer<Submitted> {
        Transfer {
            from: self.from,
            to: self.to,
            amount: self.amount,
            nonce: self.nonce,
            state: Submitted { id },
        }
    }
}

impl Transfer<Submitted> {
    pub fn confirm(self, block: u64) -> Transfer<Confirmed> {
        let id = self.state.id;
        Transfer {
            from: self.from,
            to: self.to,
            amount: self.amount,
            nonce: self.nonce,
            state: Confirmed { id, block },
        }
    }
}

/// A transfer whose state is only known at run time (for example, loaded from storage).
/// Typestate cannot express "whatever state the database says", so an enum carries the choice.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum AnyTransfer {
    Draft(Transfer<Draft>),
    Validated(Transfer<Validated>),
    Submitted(Transfer<Submitted>),
    Confirmed(Transfer<Confirmed>),
}

macro_rules! any_from {
    ($state:ident) => {
        impl From<Transfer<$state>> for AnyTransfer {
            fn from(t: Transfer<$state>) -> Self {
                AnyTransfer::$state(t)
            }
        }
    };
}
any_from!(Draft);
any_from!(Validated);
any_from!(Submitted);
any_from!(Confirmed);

/// Returned when a run-time transition is asked for in the wrong state.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct WrongState {
    pub action: &'static str,
    pub actual: &'static str,
}

impl fmt::Display for WrongState {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(
            f,
            "cannot {} a transfer that is {}",
            self.action, self.actual
        )
    }
}

impl std::error::Error for WrongState {}

impl AnyTransfer {
    pub fn state_name(&self) -> &'static str {
        match self {
            AnyTransfer::Draft(t) => t.state_name(),
            AnyTransfer::Validated(t) => t.state_name(),
            AnyTransfer::Submitted(t) => t.state_name(),
            AnyTransfer::Confirmed(t) => t.state_name(),
        }
    }

    /// The run-time counterpart of `Transfer<Validated>::submit`: the same rule, checked
    /// dynamically. A rejected call returns the original value untouched.
    pub fn submit(self, id: TxId) -> Result<AnyTransfer, (AnyTransfer, WrongState)> {
        match self {
            AnyTransfer::Validated(t) => Ok(t.submit(id).into()),
            other => {
                let err = WrongState {
                    action: "submit",
                    actual: other.state_name(),
                };
                Err((other, err))
            }
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    const A: &str = "0x1111111111111111111111111111111111111111";
    const B: &str = "0x2222222222222222222222222222222222222222";

    fn addr(s: &str) -> Address {
        s.parse().unwrap()
    }

    fn draft(units: u64) -> Transfer<Draft> {
        Transfer::new(addr(A), addr(B), Amount::new(units).unwrap(), Nonce(3)).unwrap()
    }

    #[test]
    fn address_round_trip_and_case() {
        let a = addr("0xAbCdEf0123456789aBcDeF0123456789abcdef01");
        assert_eq!(a.to_string(), "0xabcdef0123456789abcdef0123456789abcdef01");
        assert_eq!(a.to_string().parse::<Address>(), Ok(a));
    }

    #[test]
    fn address_errors() {
        assert_eq!("".parse::<Address>(), Err(AddressError::MissingPrefix));
        assert_eq!("1111".parse::<Address>(), Err(AddressError::MissingPrefix));
        assert_eq!(
            "0x".parse::<Address>(),
            Err(AddressError::BadLength { got: 0 })
        );
        assert_eq!(
            format!("0x{}", "1".repeat(39)).parse::<Address>(),
            Err(AddressError::BadLength { got: 39 })
        );
        assert_eq!(
            format!("0x{}", "1".repeat(41)).parse::<Address>(),
            Err(AddressError::BadLength { got: 41 })
        );
        let bad = format!("0x{}g{}", "1".repeat(5), "1".repeat(34));
        assert_eq!(
            bad.parse::<Address>(),
            Err(AddressError::BadDigit { index: 5 })
        );
        // 40 *bytes* but not 40 ASCII digits: multi-byte characters must not panic.
        let wide = format!("0x{}", "é".repeat(20));
        assert!(wide.parse::<Address>().is_err());
    }

    #[test]
    fn amount_rejects_zero() {
        assert_eq!(Amount::new(0), Err(TransferError::ZeroAmount));
        assert_eq!(Amount::new(5).unwrap().get(), 5);
        assert_eq!(
            std::mem::size_of::<Option<Amount>>(),
            std::mem::size_of::<u64>()
        );
    }

    #[test]
    fn new_rejects_same_address() {
        let r = Transfer::new(addr(A), addr(A), Amount::new(1).unwrap(), Nonce(0));
        assert_eq!(r.unwrap_err(), TransferError::SameAddress);
    }

    #[test]
    fn happy_path_through_every_state() {
        let d = draft(50);
        assert_eq!(d.state_name(), "draft");
        let v = d.validate(50).unwrap();
        assert_eq!(v.state_name(), "validated");
        let s = v.submit(TxId(9));
        assert_eq!((s.state_name(), s.state().id), ("submitted", TxId(9)));
        let c = s.confirm(77);
        assert_eq!(
            *c.state(),
            Confirmed {
                id: TxId(9),
                block: 77
            }
        );
        assert_eq!(
            (c.from(), c.to(), c.amount().get(), c.nonce()),
            (addr(A), addr(B), 50, Nonce(3))
        );
    }

    #[test]
    fn failed_validation_returns_the_draft() {
        let d = draft(50);
        let before = d.clone();
        let failed = d.validate(49).unwrap_err();
        assert_eq!(
            failed.error,
            TransferError::InsufficientFunds {
                needed: 50,
                available: 49
            }
        );
        assert_eq!(failed.transfer, before);
        assert_eq!(
            failed.error.to_string(),
            "insufficient funds: need 50, have 49"
        );
        // The caller can retry with a bigger balance.
        assert!(failed.transfer.validate(1_000).is_ok());
    }

    #[test]
    fn states_cost_nothing_extra() {
        use std::mem::size_of;
        assert_eq!(size_of::<Draft>(), 0);
        assert_eq!(
            size_of::<Transfer<Draft>>(),
            size_of::<Transfer<Validated>>()
        );
    }

    #[test]
    fn any_transfer_checks_at_run_time() {
        let v: AnyTransfer = draft(5).validate(5).unwrap().into();
        let submitted = v.submit(TxId(1)).unwrap();
        assert_eq!(submitted.state_name(), "submitted");

        let d: AnyTransfer = draft(5).into();
        let (back, err) = d.clone().submit(TxId(1)).unwrap_err();
        assert_eq!(back, d);
        assert_eq!(err.to_string(), "cannot submit a transfer that is draft");
    }
}
```

✓ With this exact file: `cargo fmt --check` clean, `cargo clippy --all-targets --locked -- -D warnings` clean, `cargo test --locked` → 8 unit tests, 1 passing doctest and 3 `compile_fail` doctests passed. In a separate binary crate depending on `statelab` I confirmed ✓ the real errors: `E0599` (no `submit` on `Transfer<Draft>`), `E0382` (use of moved value `draft`), and `E0277` (`Rogue: statelab::sealed::Sealed` not satisfied).

**Design notes.**

- **Private fields, public accessors.** The only constructor is `Transfer::new`, and every transition rebuilds the struct with a new `state`. The state types carry the data that exists only in that state (`Submitted { id }`, `Confirmed { id, block }`), so you can't read an id before submission.
- **`Failed<S>` returns ownership on error.** A consuming transition would otherwise lose the transfer on failure. The test `failed_validation_returns_the_draft` shows the caller getting the identical draft back and retrying.
- **Sealed `State`.** The `sealed` module is private, so downstream crates cannot name `Sealed` and cannot implement `State`. A small `macro_rules!` implements both traits for each state (declarative macros are Lesson 34).
- **`Address::from_str`** works on bytes (`as_bytes`, `chunks_exact(2)`), never slices the `str` at arbitrary offsets, so non-ASCII input returns an error instead of panicking on a char boundary (Lesson 06). The length check counts bytes after the prefix, and the multi-byte test (`"é".repeat(20)` is 40 bytes) reaches the digit check and fails there.
- **`Amount(NonZeroU64)`** makes zero unrepresentable and gives `Option<Amount>` the size of a `u64` (tested).
- **`AnyTransfer`** bridges to run-time states: `From` for each typed transfer, and a dynamic `submit` that returns `(original, WrongState)` on a wrong-state call, the dynamic version of what the compiler enforces statically.
- **Not real security.** There is no signing, balance lookup or networking here; "validate" just compares a number. The lesson is about the type-level structure.
- The `compile_fail` doctests pass if the snippet fails to compile for *any* reason; I checked the intended error codes with the external crate above, not inside the doctests.

</details>

---

## 11. Interview questions

<details markdown="1">
<summary>Answers</summary>

1. **What is the newtype pattern and why use it?** A single-field tuple struct that wraps another type to give it a distinct identity: prevents mixing units/ids, carries validation, and lets you implement foreign traits. It's zero-cost.
2. **Is a newtype really zero-cost?** In size and generally in generated code, yes (`size_of::<Meters>() == size_of::<f64>()` ✓); `#[repr(transparent)]` additionally guarantees layout/ABI equality.
3. **What is the orphan rule and how do newtypes help?** You can implement a trait for a type only if the trait or type is local. Wrapping a foreign type in a local newtype makes the type local.
4. **Why is `Deref` on a newtype risky?** It exposes the inner type's whole API (especially with `DerefMut`), bypassing the invariants the newtype exists to protect.
5. **What is typestate?** Encoding the state of a value in its type so that invalid operations don't compile; transitions consume the old state and return the new one.
6. **How do you implement typestate in Rust?** A generic struct `T<S>` with zero-sized marker types and `PhantomData<S>`, `impl` blocks per state, and `self`-consuming transition methods.
7. **When shouldn't you use typestate?** When the state is only known at run time, when you need mixed-state collections, with many states/events, or when API simplicity matters more; use a runtime enum instead.
8. **How do you combine typestate with runtime data?** Wrap the typed states in an enum (`AnyX`), construct the right variant when loading, and match on it to call typed methods.
9. **What is a sealed trait?** A public trait with a private supertrait, so no other crate can implement it; used to close a set of states or to retain freedom to extend the trait.
10. **Why does `#[derive(Clone, Copy)]` on `Id<T>` surprise people?** It adds `T: Clone`/`T: Copy` bounds even if `T` is only a `PhantomData` marker; implement the traits manually.
11. **What does `PhantomData<T>` do?** It's a zero-sized marker that tells the compiler the struct logically owns/uses a `T` (affecting variance, drop check and auto traits) without storing one.
12. **Why return the value inside the error of a consuming transition?** So the caller can recover and retry instead of losing the value (`Failed { transfer, error }`).
13. **What does a pure transition function buy you?** Determinism, easy table-driven tests, replayability of an event log, and sharing the same logic between client and validator.
14. **What does typestate not prove?** That your validation logic is right, or that the caller is authorized; it only enforces the *order* and *kind* of operations the types allow.

</details>

---

## 12. Key takeaways

- A newtype gives a primitive a distinct, checked identity at zero cost: units, ids, validated data, foreign-trait impls. Keep the inner field private and provide fallible constructors plus conversions.
- `Deref`/`DerefMut` on invariant-carrying newtypes defeats the purpose; prefer explicit methods.
- Typestate puts the state in the type: operations live in per-state `impl` blocks, transitions consume `self` and return the next state, and misuse is `E0599`/`E0382` at compile time. States are zero-sized markers with `PhantomData`; seal the state trait.
- Use typestate when the sequence is known at compile time; use a runtime enum (or an enum wrapping typed states) when it comes from data, and a pure `(State, Event) -> Result<State, Error>` function for protocol-style machines.
- Beware `derive` on generic marker structs (it adds `T: Trait` bounds); implement `Clone`/`Copy`/`PartialEq` by hand and use `PhantomData<fn() -> T>`.
- Give failed consuming transitions a way to return the value; keep rejected transitions side-effect free.
- Types prove structure and ordering, not business correctness or authorization.

**Connections.** Lesson 13's private fields, constructors and runtime enums are the foundation; Lesson 14's `Result`/`FromStr`/error design is how newtype constructors fail; Lesson 09's layout facts explain why the markers cost nothing; Lesson 07's moves make typestate transitions un-reusable. Next: Lesson 16 (collections: `Vec`, `HashMap`, `BTreeMap`, `VecDeque`, ...) and Lesson 17 (iterators). Lesson 19–20 (traits, coherence) revisit the orphan rule and `std::ops` for newtype arithmetic; Lesson 22 covers `PhantomData`, `Send`/`Sync` and variance for marker generics; Lesson 34 introduces the declarative macros used in `statelab`; Lessons 48–57 use validated types and deterministic state machines for addresses, amounts, transactions and on-chain accounts. See the [roadmap](00_roadmap.md) and the [course index](index.md).

---

## 13. References

- *Rust API Guidelines*: "Newtypes provide static distinctions" (C-NEWTYPE), "Sealed traits protect against downstream implementations" (C-SEALED), "Deref should only be implemented for smart pointers" (C-DEREF): <https://rust-lang.github.io/api-guidelines/>
- *The Rust Programming Language*, ch. 20.2 "Advanced Traits" (newtype pattern, orphan rule workaround): <https://doc.rust-lang.org/book/ch20-02-advanced-traits.html>
- *The Rust Reference*, "Type layout: `repr(transparent)`", and "Trait implementation coherence": <https://doc.rust-lang.org/reference/type-layout.html#the-transparent-representation>, <https://doc.rust-lang.org/reference/items/implementations.html#trait-implementation-coherence>
- *The Embedded Rust Book*, "Static Guarantees" (typestate programming): <https://docs.rust-embedded.org/book/static-guarantees/typestate-programming.html>
- Rust standard library documentation: `std::marker::PhantomData`, `std::num::NonZeroU64`, `std::ops::Deref`: <https://doc.rust-lang.org/std/marker/struct.PhantomData.html>, <https://doc.rust-lang.org/std/num/struct.NonZero.html>, <https://doc.rust-lang.org/std/ops/trait.Deref.html>
- *The Rustonomicon*, "PhantomData": <https://doc.rust-lang.org/nomicon/phantom-data.html>
- Rust error code index: E0117 (orphan rule), E0599, E0382, E0603, E0616: <https://doc.rust-lang.org/error_codes/E0117.html>, <https://doc.rust-lang.org/error_codes/E0599.html>, <https://doc.rust-lang.org/error_codes/E0603.html>
- Alexis King, "Parse, don't validate": <https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/>
- Rust By Example, "Traits: Disambiguating overlapping traits" and "PhantomData: unit clarification": <https://doc.rust-lang.org/rust-by-example/generics/phantom.html>

The URLs above are the standard documentation locations; I did not open every one during this session, so if a link has moved, search the docs site for the page title. Unchecked from memory: the exact numbering of the API-guidelines checklist item names, the Rust Book chapter number for advanced traits, and the Rust By Example page slugs.
