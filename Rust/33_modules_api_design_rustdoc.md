---
layout: default
title: "Lesson 33 — Modules, visibility, API design, and rustdoc"
---

# Lesson 33 — Modules, visibility, API design, and rustdoc

**Module 11 · Modules, API design, macros** · Status: generated · Edition used in examples: **2024** · External dependencies: none (everything runs with `rustc`, `cargo`, `rustdoc` from the standard toolchain)

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux. As before, ✓ marks outputs, diagnostics and measurements I actually ran; unmarked statements are documented behavior I did not run. Everything is **stable Rust**. The sandbox cannot open the Rust documentation sites, so rules about the module system, the semver guidance and the API Guidelines are from memory unless a ✓ shows them in compiler output; the closing paragraph lists what to double-check. I did **not** run `cargo-semver-checks` (not installed); the breaking changes below were demonstrated by compiling a consumer against two versions of a library.

Builds on [Lesson 3](03_cargo_projects.md) (the first look at modules and `pub`), [Lesson 13](13_structs_enums_methods.md) (structs, enums, methods), [Lesson 15](15_newtype_typestate.md) (newtypes and making invalid states unrepresentable), [Lesson 19](19_generics_and_bounds.md) (bounds) and [Lesson 26](26_error_design.md) (error types, `#[non_exhaustive]`). Lesson 34 (macros) is next; it is mentioned here only as plain text.

---

## 1. Why this lesson matters

Until now the code you wrote was mostly *used by itself*. A library is different: other people (including you in six months) will depend on every name you make `pub`, and every public name is a promise. Rust gives you three tools to manage that promise:

1. **Privacy**: the compiler enforces who may name an item, read a field, or call a method.
2. **The type system**: you can make misuse a compile error (sealed traits, `#[non_exhaustive]`, private constructors, typestate).
3. **Documentation that is tested**: `rustdoc` turns `///` comments into HTML, and turns the code in them into tests.

The goal is a **small, hard-to-misuse, evolvable** public surface. This lesson covers the mechanics (modules, visibility, re-exports), the design rules (and the semver consequences of breaking them, which we will *run*), and `rustdoc`. The practical challenge builds a multi-file crate, `semverlab`, whose doctests act as an API contract.

---

## 2. The module system

A crate is a tree of **modules**. The root is `src/lib.rs` (library) or `src/main.rs` (binary). `mod name;` *declares* a child module and tells the compiler to find its source; `use` only brings a name into scope. A file nobody declares with `mod` is not compiled at all.

### 2.1 Where the compiler looks for the file

For `mod a;` in the crate root, the compiler accepts **either** of these (the second is the older style):

```text
src/a.rs            (and its children in src/a/b.rs)
src/a/mod.rs        (and its children in src/a/b.rs)
```

✓ I built both layouts (`a.rs` + `a/b.rs`, and `c/mod.rs` + `c/d.rs`) in one library and both compiled. Mixing them is allowed across *different* modules; for one module, supplying both files is an error, and supplying neither is an error:

```text
error[E0583]: file not found for module `missing`      ✓ (help names src/missing.rs or src/missing/mod.rs)
error[E0761]: file for module `a` found at both "src/a.rs" and "src/a/mod.rs"   ✓
```

Pick one layout per project (the `a.rs` + `a/` style is the modern default; `mod.rs` is still common in older code) and let `rustfmt` and `clippy` keep you honest.

Inline modules need no file:

```rust
mod geometry {
    pub mod shapes {
        pub struct Square(pub f64);
        pub fn unit() -> Square { Square(1.0) }
    }
    pub fn area(s: &shapes::Square) -> f64 { s.0 * s.0 }
}

use geometry::shapes::{self, Square};

fn main() {
    let s: Square = shapes::unit();
    println!("{}", geometry::area(&s));
}
```

### 2.2 Paths and `use`

- `crate::a::b` is an absolute path from the crate root; `self::` is the current module; `super::` is the parent. A path with no prefix is resolved relative to the current module, or as an external crate / prelude name.
- `use` creates a **name binding** in the current module. It does not copy anything and has no runtime cost.
- `use a::b::{self, C, D as E};` groups imports; `use a::*;` is a glob (convenient for `use super::*;` in test modules, discouraged elsewhere because it hides where names come from and can break when the upstream module gains a name).
- A wrong path is a *name resolution* error:

```rust
// ERROR E0432
mod a { pub mod b { pub fn f() {} } }
use a::b::g;          // no such item
use crate::a::c;      // no such module
fn main() { a::b::f(); }
```

✓ Both lines give `error[E0432]: unresolved import`.

---

## 3. Visibility

By default everything is **private to the module where it is defined and to that module's descendants**. (Privacy is about *modules*, not types: code in a child module can see its parent's private items, but not the other way round.) You widen it with:

| Syntax | Visible in |
|---|---|
| *(nothing)* | the current module and its descendants |
| `pub(self)` | same as nothing |
| `pub(super)` | the parent module (and its descendants) |
| `pub(in crate::a::b)` | the named ancestor module |
| `pub(crate)` | the whole crate |
| `pub` | everywhere the containing module is reachable |

(The `pub(in path)` form requires `path` to be an ancestor of the current module.) A `pub` item is only *really* public if every module on the path to it is reachable too: a `pub fn` inside a private `mod` is invisible to other crates unless you re-export it (§4).

### 3.1 Fields and methods have their own privacy

A `pub struct` can have private fields. Then:

- outside the module you cannot read the field, write it, **or construct the struct with a literal**;
- you must offer a constructor, and you control which values can exist (an invariant: "`balance` is never negative", "the version string was validated").

```rust
// ERROR E0616, E0624, E0603 (and E0451 is hidden until these are fixed)
mod bank {
    pub struct Account { pub owner: String, balance: u64 }
    impl Account {
        pub fn new(owner: &str) -> Self { Account { owner: owner.into(), balance: 0 } }
        fn audit(&self) -> u64 { self.balance }
        pub(crate) fn crate_only(&self) -> u64 { self.audit() }
    }
    fn secret() {}
    pub mod inner {
        pub(super) fn parent_only() {}
        pub(in crate::bank) fn bank_only() {}
    }
    pub fn call_inner() { inner::parent_only(); inner::bank_only(); }
}
fn main() {
    let a = bank::Account::new("ann");
    println!("{}", a.owner);
    println!("{}", a.balance);          // E0616: private field
    println!("{}", a.audit());          // E0624: private method
    bank::secret();                      // E0603: private function
    let b = bank::Account { owner: "x".into(), balance: 5 }; // E0451
    bank::inner::parent_only();          // E0603: pub(super) is visible to `bank`, not to the crate root
    println!("{}", a.crate_only());      // fine: same crate
}
```

✓ The compiler reports exactly four errors: `E0603` (`secret`), `E0603` (`parent_only`), `E0616` (`balance`) and `E0624` (`audit`). The struct-literal error is reported when it is the only one:

```rust
// ERROR E0451
mod bank { pub struct Account { pub owner: String, balance: u64 } }
fn main() { let _b = bank::Account { owner: "x".into(), balance: 5 }; }
```

✓ `error[E0451]: field `balance` of struct `Account` is private`.

**Why struct literals are blocked by one private field.** If you could write `Account { owner, balance: 5 }` outside the module, you could bypass `new`, so no invariant maintained by `new` would hold. A single private field closes the door, and also means *adding another private field later is not a breaking change* (nobody could have used a literal).

### 3.2 Tuple structs, enums and traits

- Tuple-struct fields are private by default too: `pub struct Id(u32);` cannot be constructed or destructured outside. `pub struct Id(pub u32);` can.
- **Enum variants and their fields are always as public as the enum**. You cannot hide a variant: that is why adding a variant to a public enum is a breaking change (§5.3).
- **Trait items** (methods) are as public as the trait. Trait *impls* have no visibility of their own.
- A `pub` function whose signature mentions a type less visible than the function triggers the *private-in-public* lints (the exact lint names and levels changed across Rust versions; check the current reference). The practical rule: **every type in a public signature should be nameable by the caller**.

---

## 4. Re-exports and the facade pattern

`pub use` creates a public **alias for a name**, not a new item:

```rust
pub mod net {
    pub struct Conn(u8);
    impl Conn { pub fn new() -> Self { Conn(0) } }
}
pub use net::Conn as Connection;

fn main() {
    let _c: Connection = net::Conn::new();
    println!("{}", std::any::type_name::<Connection>());
}
```

✓ Prints `v03::net::Conn`: `Connection` and `net::Conn` are the **same type**, so renaming through a re-export never creates a second type.

The facade pattern: keep your modules private and `pub use` just the intended API from the crate root. Users see a flat, stable list; you can move code between files freely, because moving a *private* module is not a public change. `semverlab` (§12) does this:

```rust
// lib.rs
mod bump;     // private modules...
mod error;
mod req;
mod version;

pub use bump::{Major, Minor, Part, Patch};   // ...and a deliberate public surface
pub use error::ParseError;
pub use req::{Op, VersionReq};
pub use version::Version;
```

Consequences to remember:

- The *path* of a public item is part of your API only if it is publicly reachable. `semverlab::Version` is API; `semverlab::version::Version` does not exist.
- If you re-export a type from a *dependency* (`pub use other_crate::Thing`), that dependency's version becomes part of **your** API: upgrading it across a breaking release is a breaking release for you.
- `#[doc(inline)]` / `#[doc(no_inline)]` control whether rustdoc shows a re-exported item inline (from memory).
- The *prelude* pattern (`pub mod prelude { pub use ...; }`) is a convenience for crates with many commonly-used names; use sparingly, because glob-importing a prelude can collide with other names.

---

## 5. API design

### 5.1 Private fields, constructors, accessors

Make fields private unless the type is plain data with no invariant (a `Point { x, y }` is fine as public fields). If you do have an invariant, enforce it at construction and keep it through every `&mut self` method. In `semverlab`, a `Version` cannot be built from a malformed string, because there is no way to build one except `Version::new(maj, min, pat)` or `parse`:

```rust
use semverlab::Version;
let v: Version = "1.4.2-rc.1".parse().unwrap();   // validated
let w = Version::new(1, 4, 2);
assert!(v < w);                                    // pre-release sorts before the release
```

(The snippet uses the crate from §12, so it is a doctest there, not a standalone block.) Accessors (`v.major()`) cost nothing at run time after inlining and leave you free to change the representation later (✓ the crate stores pre-release identifiers as an enum, not strings, and nothing outside notices).

### 5.2 Make misuse a compile error

From earlier lessons: newtypes and typestate ([Lesson 15](15_newtype_typestate.md)). Three more devices for *library* authors:

**Sealed traits.** Sometimes a trait is meant to be *used* by callers but only *implemented* by you (so you can add methods without breaking anyone). Make the supertrait unnameable:

```rust
mod private { pub trait Sealed {} }

/// Only types in this crate can implement `Shape`.
pub trait Shape: private::Sealed { fn area(&self) -> f64; }

pub struct Square(pub f64);
impl private::Sealed for Square {}
impl Shape for Square { fn area(&self) -> f64 { self.0 * self.0 } }

fn main() { println!("{}", Square(3.0).area()); }
```

Downstream crates see `Shape` (a `pub` trait) and its methods, but cannot name `private::Sealed` (the module is private), so they cannot implement it. ✓ Using `Shape`'s methods from another crate works; trying to `impl Shape for Circle` from another crate fails with `error[E0277]: the trait bound `Circle: sealed::private::Sealed` is not satisfied`. Adding a method to a sealed trait is **not** a breaking change (no one else implements it); say so in the docs.

**`#[non_exhaustive]`.** On a `struct`, an `enum`, or an enum *variant*, it tells other crates "this may grow". Downstream code must write `..` in patterns, may not use struct literals, and must have a wildcard arm when matching an enum. Inside your own crate nothing changes. ✓ Against a library whose `Config` and `Mode` were marked `#[non_exhaustive]`, a consumer got:

```text
error[E0639]: cannot create non-exhaustive struct using struct expression
error[E0638]: `..` required with struct marked as non-exhaustive
error[E0004]: non-exhaustive patterns: `_` not covered       (separate run: the enum match without `_`)
```

Provide a constructor for such a struct (`Config::new(name)` plus builder-style setters), so callers can still create it.

**`#[must_use]`.** Put it on functions whose result is the whole point (`fn parse(...) -> Result`, which already is `#[must_use]` via `Result`) and on types where ignoring a value is almost certainly a bug (guards, builders). ✓ `semverlab` puts it on `Version::bump`: calling `v.bump::<Minor>();` and ignoring the result gives `warning: unused return value of ... that must be used`. Without the attribute the new version would be silently discarded.

### 5.3 What counts as a breaking change (semver)

Semver says: a change that can break a downstream crate's *compiling code* needs a new **major** version (before 1.0, the minor number plays that role in Cargo's convention). We can make this concrete by compiling one consumer against two versions of a library.

Library v1:

```rust
pub struct Config { pub name: String }
pub enum Mode { Fast, Slow }
pub trait Plugin { fn run(&self); }
pub fn open(path: &str) -> usize { path.len() }
```

The consumer (it compiles against v1 ✓):

```rust
// consumer.rs  (linked as the extern crate `lib`)
struct P;
impl lib::Plugin for P { fn run(&self) {} }
fn main() {
    let c = lib::Config { name: "x".into() };
    let m = lib::Mode::Fast;
    match m { lib::Mode::Fast => {}, lib::Mode::Slow => {} }
    let _ = lib::open("p");
    let _ = (c, P);
}
```

Library v2 makes four changes that all *look* harmless:

```rust
pub struct Config { pub name: String, pub retries: u32 }       // added a pub field
pub enum Mode { Fast, Slow, Auto }                              // added a variant
pub trait Plugin { fn run(&self); fn name(&self) -> String; }   // added a required method
pub fn open(path: &str, create: bool) -> usize { path.len() + create as usize } // changed signature
```

✓ The same consumer against v2 fails with:

```text
error[E0046]: not all trait items implemented, missing: `name`
error[E0063]: missing field `retries` in initializer of `Config`
error[E0061]: this function takes 2 arguments but 1 argument was supplied
```

and, once those are fixed, `error[E0004]: non-exhaustive patterns: `Mode::Auto` not covered` (✓ run separately, because it is reported in a later compiler phase). Each of the four is a **major** change.

A checklist (from the Rust API/semver guidance; the full lists are in the Cargo book and the RFC, see references):

| Change | Breaking? |
|---|---|
| Remove or rename a public item, field, variant, method | **major** |
| Add a variant to a public enum without `#[non_exhaustive]` | **major** ✓ |
| Add a public field to a struct that was constructible with a literal | **major** ✓ |
| Add a required method (no default) to a public, **unsealed** trait | **major** ✓ (fine for a sealed trait, since nobody else implements it) |
| Change a function signature (arity, argument or return type, bounds) | **major** ✓ |
| Add a defaulted method to a trait | usually minor, but can break callers through method-name ambiguity with other traits (from memory) |
| Add a new public item | minor, but glob-importing users can get a name clash (from memory) |
| Loosen a bound, widen an argument type | usually minor; inference can break in corner cases (from memory) |
| Make a type stop being `Send`/`Sync`/`Unpin`, or add a private field with a `!Send` type | **major** (auto traits leak!) |
| Raise the MSRV or change what an optional feature enables | policy decision; document it |

The auto-trait row is easy to miss: adding a private `Rc<_>` field to a public struct removes `Send` and breaks users who sent it to another thread. Guard with a test (`fn assert_send_sync<T: Send + Sync>() {}`; `semverlab`'s tests do this ✓).

### 5.4 More design rules (the Rust API Guidelines, abridged, from memory)

- **Be flexible in arguments, specific in returns.** Take `&str`, `impl AsRef<Path>`, `impl IntoIterator<Item = T>`; return concrete types or `impl Trait` when callers should not name them.
- **Implement the common traits eagerly**: `Debug` (almost always), `Clone`, `PartialEq`/`Eq`, `Hash`, `Ord`, `Default`, `Display`, `FromStr`, `Error`. Types should be `Send + Sync` unless there is a reason.
- **`Eq`, `Ord` and `Hash` must agree**: if `a == b` then `a.cmp(&b) == Equal` and `hash(a) == hash(b)`. `semverlab`'s `Version` ignores build metadata in *all three*, and a mutation test confirms the tests notice if one of them disagrees (§12). The debugging challenge (§10) shows what breaks otherwise.
- **Names**: `as_`/`to_`/`into_` conversion conventions (`as_` cheap borrow, `to_` expensive, `into_` consumes), `iter`/`iter_mut`/`into_iter`, getters without `get_` (`v.major()`), `new`/`with_x`/`try_new`.
- **Errors**: a real error type with `Display`, `Error`, a stable set of variants, `#[non_exhaustive]` ([Lesson 26](26_error_design.md)); never `Result<_, String>` in a library API.
- **Builders** for types with many optional parameters; they let you add options later without breaking callers.
- **Don't expose dependencies' types** unless you are willing to version them with your own crate.
- **Keep the surface small.** Every `pub` item costs you forever; `pub(crate)` costs nothing.

---

## 6. rustdoc

`///` documents the next item, `//!` documents the enclosing module or crate. The text is Markdown. Conventional sections: a one-line summary, then `# Examples`, `# Errors` (when it returns `Result`), `# Panics`, `# Safety` (for `unsafe fn`).

```rust
/// Adds one.
///
/// # Examples
///
/// ```
/// assert_eq!(rd::add_one(1), 2);
/// ```
pub fn add_one(x: i32) -> i32 { x + 1 }
```

Useful commands: `cargo doc --no-deps --open`, `cargo test --doc`, `RUSTDOCFLAGS="-D warnings" cargo doc --no-deps`.

### 6.1 Doctests are tests

Every fenced Rust block in a doc comment is compiled and run by `cargo test` as a separate small program that links your crate. Details (from memory unless ✓):

- Lines that start with `# ` are compiled but hidden from the rendered docs; use them for imports and `fn main` scaffolding. To use `?` in a doctest, end with `# Ok::<(), ErrorType>(())`, as `semverlab`'s crate docs do.
- ✓ A failing doctest fails `cargo test --doc` (I wrote one `assert_eq!(rd::add_one(1), 3)` on purpose: `left: 2 right: 3`), and two doctests ran.
- Attributes on the fence: `ignore` (skip), `no_run` (compile only), `should_panic`, `compile_fail`, `edition2024`.

**`compile_fail` as an API contract.** A plain `compile_fail` passes on *any* error, including a typo in your example, which makes it nearly worthless. Pin the error code:

```rust
/// ```compile_fail,E0616
/// let v = semverlab::Version::new(1, 2, 3);
/// let _ = v.major; // use `v.major()`
/// ```
```

Now the doctest passes only if compilation fails **with E0616**. If a refactor makes `major` public, the doctest fails and tells you that you just changed your API. ✓ (§12 mutation M1b.) These contract doctests are a cheap substitute for a semver checker: they pin the specific promises you care about (private field, sealed trait, non-exhaustive enum).

### 6.2 Linking, lints and hiding

- **Intra-doc links**: ``[`Version::parse`]`` and ``[`crate::Op`]`` resolve to the item and stay correct when it moves. ✓ A broken one warns: `unresolved link to `missing_fn``.
- `#![warn(missing_docs)]` (or `deny`) makes every public item need docs. ✓ `warning: missing documentation for a function`. `semverlab` uses `deny`.
- `#![warn(unreachable_pub)]` flags `pub` items that cannot actually be reached from outside (a `pub fn` in a private module that is not re-exported). It is a nice reminder to use `pub(crate)`; I ran it on `semverlab` and it printed no warning (from memory how it treats `pub` trait/enum members).
- `#[doc(hidden)]` removes an item from the generated docs but **does not make it private**: callers can still use it and you can still break them. ✓ The hidden function was absent from `index.html` and no page was generated for it. It is for items that must be `pub` for a macro expansion to work, not a way to hide implementation details.
- `RUSTDOCFLAGS="-D warnings" cargo doc --no-deps` in CI turns broken links, bad HTML and similar problems into failures (✓ clean on `semverlab`).
- `#[cfg_attr(docsrs, doc(cfg(...)))]` and `docs.rs` metadata live in `Cargo.toml`; the details and nightly-only parts are out of scope here (from memory).

---

## 7. Common mistakes

1. **Making everything `pub` "for tests".** Unit tests live in a child module (`#[cfg(test)] mod tests`) and see private items anyway. Use `pub(crate)` if two modules of the crate need to talk.
2. **Public fields on a type with an invariant** (the invariant is now a convention, not a guarantee).
3. **Exhaustive public enums for things that will grow** (error kinds, protocol messages, options).
4. **Forgetting the auto-trait/semver link**: adding a `Cell`/`Rc` field later.
5. **Implementing `Eq` by hand and `Hash` by derive** (or vice versa) with different fields (§10).
6. **Re-exporting a dependency's type** and then upgrading the dependency across a breaking release without bumping your own major.
7. **`#[doc(hidden)]` as privacy.** It is not.
8. **Doctests that only `compile_fail`** (without an error code): they pass for the wrong reason.
9. **A glob re-export** (`pub use inner::*;`) in the root: every new `pub` item you add inside becomes public API automatically.
10. **Untested docs**: `ignore` on every example. They rot silently.

---

## 8. Performance, reliability, security

- **Performance.** Visibility, `use` and `pub use` cost nothing at run time; privacy is erased after type checking. Accessors are tiny functions that the optimizer inlines within a crate; across crates a non-generic, non-`#[inline]` function is only inlined with link-time optimization (from memory), so a hot accessor in a library may deserve `#[inline]`. Do not guess: measure (Lesson 41).
- **Compile time.** Many tiny crates, heavy generics and big macro expansions cost more than module structure does. Putting a trait bound in a `pub` signature that forces a downstream monomorphization is a cost you impose on users.
- **Reliability.** Privacy is how a library keeps its invariants. Every `pub` field and every public `&mut` method is a place where the invariant can be violated; review each. Semver discipline is operational reliability: one accidental breaking change in a patch release breaks every build that resolves `^1.x`.
- **Security.** Rust's privacy rules are a *design tool*, not a security boundary: any code in your process can do anything `unsafe` allows, and a dependency can read your memory. But for `unsafe` code, privacy is what makes a safety argument possible: "the length field is private, so only this module can change it, so the invariants checked here hold everywhere" (Lesson 36 builds on this). Do not derive `Debug` on types holding secrets without thinking (it prints them); implement it by hand. Do not parse untrusted input with a function that panics: `semverlab`'s `Version::parse` returns `Err` for every malformed input in the table in its tests (it has not been fuzzed; Lesson 40 covers fuzzing). Treat the supply chain as part of your API: pin and audit what you re-export.

---

## 9. Exercises

Answers are collapsed. Try each before opening it.

1. **(Warm-up)** What is the difference between `mod utils;` and `use utils::helper;`?
2. **(Warm-up)** A struct has one private field. Which three things can code outside its module not do?
3. **(Easy)** Which of these is visible in `crate::a::b::c`'s sibling module `crate::a::d`: a `pub(super)` item in `a::b`, a `pub(in crate::a)` item in `a::b`, a `pub(self)` item in `a::b`?
4. **(Medium)** Your crate has `pub mod net { pub struct Conn; }` and you later want to move `Conn` to `net::tcp`. Give a way to do that without a breaking change, and say what type identity guarantees you.
5. **(Medium)** Make `pub trait Codec { fn encode(&self) -> Vec<u8>; }` implementable only inside your crate, and explain how you can then add a method.
6. **(Medium-hard)** For each change, say breaking or not, and why: (a) add a field to a struct with only private fields; (b) add a variant to a `#[non_exhaustive]` enum; (c) change `fn f(x: String)` to `fn f(x: impl Into<String>)`; (d) add a private `Rc<()>` field to a public struct.
7. **(Hard)** Write a doctest that proves a public type `Token` is **not** `Clone`, and explain why you should pin the error code.
8. **(Hard)** `#[derive(PartialEq, Eq, Hash)]` on a struct whose `PartialEq` you then replace by hand to ignore one field. What else must you change, and what breaks in `HashMap` if you do not?

<details markdown="1">
<summary>Answers</summary>

1. `mod utils;` *declares* a module and makes the compiler load `utils.rs` (or `utils/mod.rs`); without it the file is not compiled. `use utils::helper;` only creates a name `helper` in the current scope for an item that already exists. You need the `mod` once and a `use` wherever you want a short name.
2. Read it (`E0616`), write it (also E0616), and construct the struct with a literal (`E0451`, ✓). Pattern-matching it without `..` is also an error. Constructors and accessors inside the module are the way in.
3. `crate::a::d` is a sibling inside `crate::a`. `pub(super)` in `a::b` is visible in `a` and everything under it, **including `a::d`**; `pub(in crate::a)` is the same set; `pub(self)` (private) is visible only in `a::b` and its children, so **not** in `a::d`.
4. Move it and add a re-export at the old path: `pub mod net { pub mod tcp { pub struct Conn; } pub use self::tcp::Conn; }`. Old paths keep working, and since `pub use` is an alias (✓ the `type_name` output showed the original path), `net::Conn` and `net::tcp::Conn` are the **same type**: trait impls and `TypeId` do not change.
5. ```rust
   mod private { pub trait Sealed {} }
   pub trait Codec: private::Sealed { fn encode(&self) -> Vec<u8>; }
   pub struct Json;
   impl private::Sealed for Json {}
   impl Codec for Json { fn encode(&self) -> Vec<u8> { b"{}".to_vec() } }
   fn main() { println!("{:?}", Json.encode()); }
   ```
   Since only your crate can implement `Codec`, adding a method (even without a default) only requires updating your own impls. Document that the trait is sealed.
6. (a) Not breaking: nobody outside can construct it or name the field. (b) Not breaking: the enum is `#[non_exhaustive]` so downstream matches already have a wildcard (✓ E0004 only without it). (c) Usually **not** breaking for callers passing `String`, but it **is** breaking for callers that named the function as a value with an explicit type (`let f: fn(String) = f;`), turbofish users, or inference with `Into` ambiguity (from memory), so it is "minor with caveats". (d) **Breaking**: the struct stops being `Send`/`Sync` automatically.
7. In the crate's docs, a `compile_fail` block that requires `Clone` for `Token`:

   ```text
   /// ```compile_fail,E0277
   /// fn needs_clone<T: Clone>(_: &T) {}
   /// let t: mycrate::Token = mycrate::Token::new();
   /// needs_clone(&t);
   /// ```
   ```

   The unsatisfied bound is reported as `E0277` (a `t.clone()` call on a non-`Clone` type would instead be `E0599`, no method found; I did not run this example, `mycrate` is a placeholder). Pinning the code keeps the test from passing because of a typo, a missing import or an unrelated error; a bare `compile_fail` passes on any compilation failure.
8. You must change `Hash` too so that **equal values hash equally**; otherwise `a == b` but their hashes differ. Then `HashSet`/`HashMap` may store both (they probably land in different buckets), `contains` misses, and results become non-deterministic with the random hasher seed. The same applies to `Ord`/`PartialOrd`. ✓ §10 shows it running.

</details>

---

## 10. Debugging challenge

A tag type compares names case-insensitively and carries a display note that equality ignores. A `HashSet<Tag>` is supposed to deduplicate tags, but it keeps both `Rust` and `rust`.

```rust
use std::collections::HashSet;
use std::hash::{Hash, Hasher};

#[derive(Debug, Clone)]
struct Tag {
    name: String,
    #[allow(dead_code)]
    note: String, // display-only, ignored by equality
}

impl PartialEq for Tag {
    fn eq(&self, other: &Self) -> bool {
        self.name.eq_ignore_ascii_case(&other.name)
    }
}
impl Eq for Tag {}

impl Hash for Tag {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.name.hash(state);
    }
}

fn main() {
    let a = Tag { name: "Rust".into(), note: "x".into() };
    let b = Tag { name: "rust".into(), note: "y".into() };
    assert_eq!(a, b);
    let mut set = HashSet::new();
    set.insert(a);
    println!("insert equal tag -> newly inserted: {}", set.insert(b.clone()));
    println!("len = {}, contains(b) = {}", set.len(), set.contains(&b));
}
```

✓ It prints (three runs, same result) `insert equal tag -> newly inserted: true` and `len = 2, contains(b) = true`. `assert_eq!(a, b)` passes, so the two tags are equal, yet the set holds both. Explain, and fix it.

<details markdown="1">
<summary>Solution</summary>

`HashSet` first hashes the value to pick a bucket, and only then compares with `==` inside that bucket. The contract is `a == b ⇒ hash(a) == hash(b)`. Here `eq` ignores ASCII case but `hash` feeds the original string, so `"Rust"` and `"rust"` hash to different values (with overwhelming probability, because the default hasher is seeded randomly per map), land in different buckets, and are never compared. The compiler cannot check this: both impls are valid Rust. The bug is silent (no panic), deterministic enough to look like a logic error, and only appears with unequal-looking-but-equal values.

Fix: hash exactly what `eq` compares, here the lower-cased name (✓ `newly inserted: false`, `len = 1`):

```rust
impl Hash for Tag {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.name.to_ascii_lowercase().hash(state);
    }
}
```

(Allocation-free alternative: hash each byte through `to_ascii_lowercase()`.) A good habit is to define a private "key" (`fn key(&self) -> String` or a tuple of references) and implement `PartialEq`, `Ord` and `Hash` all in terms of it. `semverlab`'s `Version` does it with `cmp`: `eq` is `cmp(..) == Equal`, `hash` hashes the same fields `cmp` looks at, and its test `build_metadata_is_ignored_by_eq_ord_and_hash` fails if you break the agreement (mutations M4 and M5 in §12).

</details>

---

## 11. Interview questions

1. What does `pub(crate)` mean, and when do you use `pub(super)`?
2. A struct has all public fields except one. What can downstream code not do, and why is that useful?
3. What is a sealed trait, and what does it let you do later?
4. What does `#[non_exhaustive]` do on an enum, a struct and a variant?
5. Name four changes that are breaking in a Rust library even though they look harmless.
6. `pub use` vs `use`? Does a re-export create a new type?
7. Is `#[doc(hidden)]` the same as private?
8. How would you stop an API's invariants being broken by accident, using only the module system?
9. Why are `compile_fail` doctests pinned with error codes?
10. What is the contract between `Eq`, `Ord` and `Hash`?

<details markdown="1">
<summary>Answers</summary>

1. `pub(crate)`: visible anywhere in the current crate, invisible to dependents. `pub(super)` (or `pub(in path)`): visible only in the parent module (or the named ancestor); useful for helpers shared by a small group of sibling modules without widening the crate-wide surface.
2. They cannot construct it with a literal or read the private field (✓ `E0451`, `E0616`); they must use your constructors, so every value was validated, and you may add more private fields later without breaking anyone.
3. A public trait with a private supertrait that outsiders cannot name, so only your crate can implement it (✓ external impl gives `E0277`). You can add methods or change impls freely.
4. On an enum: other crates must have a wildcard arm (✓ `E0004` without), and adding variants is non-breaking. On a struct: other crates cannot use a struct literal (✓ `E0639`) and must use `..` in patterns (✓ `E0638`). On a variant: same restrictions for that variant. No effect inside the defining crate.
5. Add an enum variant (✓ E0004), add a pub field to a literal-constructible struct (✓ E0063), add a required trait method (✓ E0046), change a function signature (✓ E0061); also making a type `!Send`, removing an impl, tightening a bound, renaming (from memory).
6. `use` is a private import for the current module. `pub use` also exports the name, creating an alias at another path. There is only one type (✓ `type_name` printed the original path), so impls and `TypeId` stay the same.
7. No. It only removes the item from generated documentation. Anyone can still call it, and changing it can still break them.
8. Keep invariant-bearing fields private, expose constructors that validate, accessors that return copies or shared references, and `pub(crate)` for internal helpers; put the type and all code that touches its fields in one small module.
9. A bare `compile_fail` passes if compilation fails for *any* reason (typo, missing import). With `compile_fail,E0616` the test passes only for the specific error you mean, so it actually pins the API property.
10. `a == b` ⇒ `a.cmp(&b) == Equal` and `hash(a) == hash(b)`. Breaking it makes `HashMap`/`HashSet`/`BTreeMap` misbehave silently (✓ §10).

</details>

---

## 12. Practical challenge: `semverlab`

Build a library crate `semverlab` with these requirements. It should look like a small public crate: private modules, a facade, documented items, and tests.

1. `Version`: `MAJOR.MINOR.PATCH[-PRERELEASE][+BUILD]`. Private fields. `new`, `parse` (also `FromStr`), accessors, `is_prerelease`, `pre_release`, `build_metadata`, `with_pre_release`, `Display` (round-trips), `Clone`, `Debug`.
2. Semver ordering: numbers compare numerically; a release is **greater** than its pre-release; pre-release identifiers compare left to right, numeric identifiers numerically and *before* alphanumeric ones, and a shorter list that is a prefix sorts first. **Build metadata is ignored** by `Eq`, `Ord` and `Hash`.
3. `ParseError`: `#[non_exhaustive]` enum with specific variants (empty input, wrong number of parts, invalid number, leading zero, number too large, invalid identifier, invalid operator), `Display` and `Error`.
4. A sealed trait `Part` with three marker types `Major`, `Minor`, `Patch` and `Version::bump::<P>()` (lower parts reset, pre-release and build dropped).
5. `VersionReq`: comma-separated `[op]version` with `=`, `>`, `>=`, `<`, `<=` (bare version means `=`), `matches(&Version)`, `Display`. `Op` is `#[non_exhaustive]`.
6. `#![deny(missing_docs)]`, a clean `RUSTDOCFLAGS="-D warnings" cargo doc`, a main doctest, and at least four `compile_fail,E0xxx` contract doctests (private field, struct literal, sealed trait, non-exhaustive match).
7. Integration tests in `tests/api.rs` that use only the public API, including a table of parse errors and a `Send + Sync` assertion.

<details markdown="1">
<summary>Solution (spoilers: full source)</summary>

`Cargo.toml`:

```toml
[package]
name = "semverlab"
version = "0.1.0"
edition = "2024"

[dependencies]
```

`src/lib.rs`:

```rust
//! `semverlab`: parse, compare and bump [semantic versions](https://semver.org).
//!
//! The crate is a worked example of API design: private modules behind a
//! small public facade, private fields with accessors, an extensible error
//! enum, a sealed trait, and doctests that double as contract tests.
//!
//! ```
//! use semverlab::{Version, VersionReq, Minor};
//!
//! let v: Version = "1.4.2-rc.1+build.7".parse()?;
//! assert!(v.is_prerelease());
//! assert!(v < Version::new(1, 4, 2));
//! assert_eq!(v.bump::<Minor>().to_string(), "1.5.0");
//!
//! let req = VersionReq::parse(">=1.2.0, <2.0.0")?;
//! assert!(req.matches(&Version::new(1, 9, 0)));
//! assert!(!req.matches(&Version::new(2, 0, 0)));
//! # Ok::<(), semverlab::ParseError>(())
//! ```
//!
//! # Contract tests
//!
//! Fields are private, so a `Version` can only be built through functions that
//! enforce the invariants:
//!
//! ```compile_fail,E0616
//! let v = semverlab::Version::new(1, 2, 3);
//! let _ = v.major; // use `v.major()`
//! ```
//!
//! ```compile_fail,E0451
//! let _ = semverlab::Version { major: 1, minor: 2, patch: 3, pre: vec![], build: vec![] };
//! ```
//!
//! [`Part`] is sealed: downstream crates cannot add their own parts.
//!
//! ```compile_fail,E0277
//! struct Epoch;
//! impl semverlab::Part for Epoch {
//!     fn bump(v: &semverlab::Version) -> semverlab::Version { v.clone() }
//! }
//! ```
//!
//! [`ParseError`] is `#[non_exhaustive]`, so a `match` needs a wildcard arm and
//! new variants are not a breaking change:
//!
//! ```compile_fail,E0004
//! fn name(e: semverlab::ParseError) -> &'static str {
//!     match e {
//!         semverlab::ParseError::Empty => "empty",
//!         semverlab::ParseError::WrongNumberOfParts => "parts",
//!         semverlab::ParseError::InvalidNumber { .. } => "number",
//!         semverlab::ParseError::LeadingZero { .. } => "zero",
//!         semverlab::ParseError::NumberTooLarge { .. } => "large",
//!         semverlab::ParseError::InvalidIdentifier => "ident",
//!         semverlab::ParseError::InvalidOperator => "op",
//!     }
//! }
//! ```

#![deny(missing_docs)]
#![warn(unreachable_pub)]

mod bump;
mod error;
mod req;
mod version;

pub use bump::{Major, Minor, Part, Patch};
pub use error::ParseError;
pub use req::{Op, VersionReq};
pub use version::Version;
```

`src/error.rs`:

```rust
use std::fmt;

/// Why a version or requirement string could not be parsed.
///
/// This enum is `#[non_exhaustive]`: new variants may be added in a minor
/// release, so downstream `match`es need a wildcard arm.
#[derive(Debug, Clone, PartialEq, Eq)]
#[non_exhaustive]
pub enum ParseError {
    /// The input was empty or only whitespace.
    Empty,
    /// The numeric core was not exactly `MAJOR.MINOR.PATCH`.
    WrongNumberOfParts,
    /// A numeric part was empty or contained a non-digit.
    InvalidNumber {
        /// `"major"`, `"minor"` or `"patch"`.
        part: &'static str,
    },
    /// A numeric part had a leading zero (`01`).
    LeadingZero {
        /// Which part, or `"prerelease"`.
        part: &'static str,
    },
    /// A number does not fit in `u64`.
    NumberTooLarge {
        /// Which part, or `"prerelease"`.
        part: &'static str,
    },
    /// A pre-release or build identifier was empty or had an invalid character.
    InvalidIdentifier,
    /// A comparison operator was not recognised.
    InvalidOperator,
}

impl fmt::Display for ParseError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ParseError::Empty => f.write_str("empty version string"),
            ParseError::WrongNumberOfParts => f.write_str("expected MAJOR.MINOR.PATCH"),
            ParseError::InvalidNumber { part } => write!(f, "invalid {part} number"),
            ParseError::LeadingZero { part } => write!(f, "leading zero in {part}"),
            ParseError::NumberTooLarge { part } => write!(f, "{part} number is too large"),
            ParseError::InvalidIdentifier => f.write_str("invalid pre-release or build identifier"),
            ParseError::InvalidOperator => f.write_str("invalid comparison operator"),
        }
    }
}

impl std::error::Error for ParseError {}
```

`src/version.rs`:

```rust
use std::cmp::Ordering;
use std::fmt;
use std::hash::{Hash, Hasher};
use std::str::FromStr;

use crate::error::ParseError;

/// One dot-separated pre-release identifier. Declaration order matters:
/// the derived `Ord` puts every numeric identifier before every alphanumeric
/// one, as the semver rules require.
#[derive(Debug, Clone, PartialEq, Eq, PartialOrd, Ord, Hash)]
enum Ident {
    Numeric(u64),
    Alpha(String),
}

/// A semantic version: `MAJOR.MINOR.PATCH[-PRERELEASE][+BUILD]`.
///
/// Ordering follows the semver precedence rules and **ignores build
/// metadata**; equality and hashing ignore it too, so `Eq`, `Ord` and `Hash`
/// agree with each other.
#[derive(Debug, Clone)]
pub struct Version {
    major: u64,
    minor: u64,
    patch: u64,
    pre: Vec<Ident>,
    build: Vec<String>,
}

fn parse_core(s: &str, part: &'static str) -> Result<u64, ParseError> {
    if s.is_empty() || !s.bytes().all(|b| b.is_ascii_digit()) {
        return Err(ParseError::InvalidNumber { part });
    }
    if s.len() > 1 && s.starts_with('0') {
        return Err(ParseError::LeadingZero { part });
    }
    s.parse().map_err(|_| ParseError::NumberTooLarge { part })
}

fn valid_ident(s: &str) -> bool {
    !s.is_empty() && s.bytes().all(|b| b.is_ascii_alphanumeric() || b == b'-')
}

fn parse_pre_ident(s: &str) -> Result<Ident, ParseError> {
    if !valid_ident(s) {
        return Err(ParseError::InvalidIdentifier);
    }
    if s.bytes().all(|b| b.is_ascii_digit()) {
        if s.len() > 1 && s.starts_with('0') {
            return Err(ParseError::LeadingZero { part: "prerelease" });
        }
        return s
            .parse()
            .map(Ident::Numeric)
            .map_err(|_| ParseError::NumberTooLarge { part: "prerelease" });
    }
    Ok(Ident::Alpha(s.to_owned()))
}

impl Version {
    /// A release version with no pre-release or build metadata.
    pub const fn new(major: u64, minor: u64, patch: u64) -> Self {
        Version {
            major,
            minor,
            patch,
            pre: Vec::new(),
            build: Vec::new(),
        }
    }

    /// Parses `MAJOR.MINOR.PATCH[-PRERELEASE][+BUILD]` (surrounding
    /// whitespace is not allowed).
    pub fn parse(input: &str) -> Result<Self, ParseError> {
        if input.trim().is_empty() {
            return Err(ParseError::Empty);
        }
        let (rest, build) = match input.split_once('+') {
            Some((r, b)) => (r, Some(b)),
            None => (input, None),
        };
        let (core, pre) = match rest.split_once('-') {
            Some((c, p)) => (c, Some(p)),
            None => (rest, None),
        };
        let parts: Vec<&str> = core.split('.').collect();
        let [major, minor, patch] = parts.as_slice() else {
            return Err(ParseError::WrongNumberOfParts);
        };
        let mut v = Version::new(
            parse_core(major, "major")?,
            parse_core(minor, "minor")?,
            parse_core(patch, "patch")?,
        );
        if let Some(pre) = pre {
            v.pre = pre
                .split('.')
                .map(parse_pre_ident)
                .collect::<Result<_, _>>()?;
        }
        if let Some(build) = build {
            if !build.split('.').all(valid_ident) {
                return Err(ParseError::InvalidIdentifier);
            }
            v.build = build.split('.').map(str::to_owned).collect();
        }
        Ok(v)
    }

    /// The major number.
    pub fn major(&self) -> u64 {
        self.major
    }

    /// The minor number.
    pub fn minor(&self) -> u64 {
        self.minor
    }

    /// The patch number.
    pub fn patch(&self) -> u64 {
        self.patch
    }

    /// `true` if the version has a pre-release part (`1.0.0-rc.1`).
    pub fn is_prerelease(&self) -> bool {
        !self.pre.is_empty()
    }

    /// The pre-release part without the leading `-`, if any.
    pub fn pre_release(&self) -> Option<String> {
        self.is_prerelease().then(|| {
            self.pre
                .iter()
                .map(|i| match i {
                    Ident::Numeric(n) => n.to_string(),
                    Ident::Alpha(s) => s.clone(),
                })
                .collect::<Vec<_>>()
                .join(".")
        })
    }

    /// The build metadata without the leading `+`, if any.
    pub fn build_metadata(&self) -> Option<String> {
        (!self.build.is_empty()).then(|| self.build.join("."))
    }

    /// Same core numbers with a different pre-release, validated.
    pub fn with_pre_release(mut self, pre: &str) -> Result<Self, ParseError> {
        self.pre = pre
            .split('.')
            .map(parse_pre_ident)
            .collect::<Result<_, _>>()?;
        Ok(self)
    }

    /// Crate-internal: replace the numeric core and drop pre-release and
    /// build metadata (what a bump does).
    pub(crate) fn with_core(&self, major: u64, minor: u64, patch: u64) -> Version {
        Version::new(major, minor, patch)
    }
}

impl fmt::Display for Version {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}.{}.{}", self.major, self.minor, self.patch)?;
        if let Some(pre) = self.pre_release() {
            write!(f, "-{pre}")?;
        }
        if let Some(build) = self.build_metadata() {
            write!(f, "+{build}")?;
        }
        Ok(())
    }
}

impl FromStr for Version {
    type Err = ParseError;
    fn from_str(s: &str) -> Result<Self, Self::Err> {
        Version::parse(s)
    }
}

impl Ord for Version {
    fn cmp(&self, other: &Self) -> Ordering {
        (self.major, self.minor, self.patch)
            .cmp(&(other.major, other.minor, other.patch))
            .then_with(|| match (self.pre.is_empty(), other.pre.is_empty()) {
                // A release is greater than any of its pre-releases.
                (true, true) => Ordering::Equal,
                (true, false) => Ordering::Greater,
                (false, true) => Ordering::Less,
                // Vec ordering is lexicographic, so a shorter identifier list
                // that is a prefix of a longer one sorts first, as required.
                (false, false) => self.pre.cmp(&other.pre),
            })
    }
}

impl PartialOrd for Version {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

impl PartialEq for Version {
    fn eq(&self, other: &Self) -> bool {
        self.cmp(other) == Ordering::Equal
    }
}

impl Eq for Version {}

impl Hash for Version {
    fn hash<H: Hasher>(&self, state: &mut H) {
        // Must ignore `build`, exactly like `eq`.
        self.major.hash(state);
        self.minor.hash(state);
        self.patch.hash(state);
        self.pre.hash(state);
    }
}
```

`src/bump.rs`:

```rust
use crate::version::Version;

mod sealed {
    pub trait Sealed {}
}

/// Which component [`Version::bump`] increments. Sealed: only [`Major`],
/// [`Minor`] and [`Patch`] exist.
pub trait Part: sealed::Sealed {
    /// The next version after bumping this part (pre-release and build
    /// metadata are dropped, lower parts reset to zero).
    fn bump(v: &Version) -> Version;
}

/// Bump the major number: `1.4.2` becomes `2.0.0`.
#[derive(Debug, Clone, Copy)]
pub struct Major;
/// Bump the minor number: `1.4.2` becomes `1.5.0`.
#[derive(Debug, Clone, Copy)]
pub struct Minor;
/// Bump the patch number: `1.4.2` becomes `1.4.3`.
#[derive(Debug, Clone, Copy)]
pub struct Patch;

impl sealed::Sealed for Major {}
impl sealed::Sealed for Minor {}
impl sealed::Sealed for Patch {}

impl Part for Major {
    fn bump(v: &Version) -> Version {
        v.with_core(v.major() + 1, 0, 0)
    }
}
impl Part for Minor {
    fn bump(v: &Version) -> Version {
        v.with_core(v.major(), v.minor() + 1, 0)
    }
}
impl Part for Patch {
    fn bump(v: &Version) -> Version {
        v.with_core(v.major(), v.minor(), v.patch() + 1)
    }
}

impl Version {
    /// The next version after bumping `P` (`Major`, `Minor` or `Patch`).
    ///
    /// ```
    /// use semverlab::{Version, Patch};
    /// assert_eq!(Version::new(1, 4, 2).bump::<Patch>(), Version::new(1, 4, 3));
    /// ```
    ///
    /// # Panics
    /// In debug builds, if the number would overflow `u64`.
    #[must_use = "bump returns the new version; it does not modify `self`"]
    pub fn bump<P: Part>(&self) -> Version {
        P::bump(self)
    }
}
```

`src/req.rs`:

```rust
use std::fmt;

use crate::error::ParseError;
use crate::version::Version;

/// A comparison operator in a [`VersionReq`].
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[non_exhaustive]
pub enum Op {
    /// `=`: equal (also the default when no operator is written).
    Eq,
    /// `>`
    Gt,
    /// `>=`
    Ge,
    /// `<`
    Lt,
    /// `<=`
    Le,
}

impl Op {
    fn symbol(self) -> &'static str {
        match self {
            Op::Eq => "=",
            Op::Gt => ">",
            Op::Ge => ">=",
            Op::Lt => "<",
            Op::Le => "<=",
        }
    }
}

#[derive(Debug, Clone, PartialEq, Eq)]
struct Comparator {
    op: Op,
    version: Version,
}

/// A set of comparators that must **all** hold, written comma-separated:
/// `">=1.2.0, <2.0.0"`. Pre-release versions are compared by plain precedence
/// (this crate has no special pre-release opt-in rule).
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct VersionReq {
    comparators: Vec<Comparator>,
}

impl VersionReq {
    /// Parses a comma-separated list of `[op]version`.
    pub fn parse(input: &str) -> Result<Self, ParseError> {
        if input.trim().is_empty() {
            return Err(ParseError::Empty);
        }
        let comparators = input
            .split(',')
            .map(|c| {
                let c = c.trim();
                let (op, rest) = if let Some(r) = c.strip_prefix(">=") {
                    (Op::Ge, r)
                } else if let Some(r) = c.strip_prefix("<=") {
                    (Op::Le, r)
                } else if let Some(r) = c.strip_prefix('>') {
                    (Op::Gt, r)
                } else if let Some(r) = c.strip_prefix('<') {
                    (Op::Lt, r)
                } else if let Some(r) = c.strip_prefix('=') {
                    (Op::Eq, r)
                } else if c.starts_with(|ch: char| ch.is_ascii_digit()) {
                    (Op::Eq, c)
                } else {
                    return Err(ParseError::InvalidOperator);
                };
                Ok(Comparator {
                    op,
                    version: Version::parse(rest.trim())?,
                })
            })
            .collect::<Result<_, _>>()?;
        Ok(VersionReq { comparators })
    }

    /// `true` if `v` satisfies every comparator.
    pub fn matches(&self, v: &Version) -> bool {
        self.comparators.iter().all(|c| match c.op {
            Op::Eq => *v == c.version,
            Op::Gt => *v > c.version,
            Op::Ge => *v >= c.version,
            Op::Lt => *v < c.version,
            Op::Le => *v <= c.version,
        })
    }
}

impl fmt::Display for VersionReq {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        for (i, c) in self.comparators.iter().enumerate() {
            if i > 0 {
                f.write_str(", ")?;
            }
            write!(f, "{}{}", c.op.symbol(), c.version)?;
        }
        Ok(())
    }
}
```

`tests/api.rs`:

```rust
//! Integration tests use only the public API, exactly like a downstream crate.

use std::collections::HashSet;

use semverlab::{Major, Minor, ParseError, Patch, Version, VersionReq};

fn v(s: &str) -> Version {
    s.parse()
        .unwrap_or_else(|e| panic!("{s:?} should parse: {e}"))
}

#[test]
fn the_semver_precedence_chain() {
    let chain = [
        "1.0.0-alpha",
        "1.0.0-alpha.1",
        "1.0.0-alpha.beta",
        "1.0.0-beta",
        "1.0.0-beta.2",
        "1.0.0-beta.11",
        "1.0.0-rc.1",
        "1.0.0",
        "1.0.1",
        "1.1.0",
        "2.0.0",
    ];
    let parsed: Vec<Version> = chain.iter().map(|s| v(s)).collect();
    for pair in parsed.windows(2) {
        assert!(pair[0] < pair[1], "{} should be < {}", pair[0], pair[1]);
    }
    let mut shuffled = parsed.clone();
    shuffled.reverse();
    shuffled.sort();
    assert_eq!(shuffled, parsed);
}

#[test]
fn numeric_identifiers_compare_numerically_and_before_alphanumeric() {
    assert!(v("1.0.0-2") < v("1.0.0-10"));
    assert!(v("1.0.0-9") < v("1.0.0-a"));
    assert!(v("1.0.0-a") < v("1.0.0-b"));
    assert!(
        v("1.0.0-a") < v("1.0.0-a.0"),
        "a longer list wins on equal prefix"
    );
}

#[test]
fn display_and_parse_round_trip() {
    for s in [
        "0.0.0",
        "1.2.3",
        "1.2.3-rc.1",
        "1.2.3+b.5",
        "1.2.3-x.7.z-9+meta-1.00",
    ] {
        assert_eq!(v(s).to_string(), s);
    }
}

#[test]
fn build_metadata_is_ignored_by_eq_ord_and_hash() {
    let (a, b) = (v("1.0.0+aaa"), v("1.0.0+bbb"));
    assert_eq!(a, b);
    assert_eq!(a.cmp(&b), std::cmp::Ordering::Equal);
    let set: HashSet<Version> = [a.clone(), b].into_iter().collect();
    assert_eq!(set.len(), 1, "equal values must hash alike");
    assert_eq!(
        a.build_metadata().as_deref(),
        Some("aaa"),
        "but it is still stored"
    );
}

#[test]
fn accessors_and_prerelease_helpers() {
    let x = v("3.4.5-beta.2+sha");
    assert_eq!((x.major(), x.minor(), x.patch()), (3, 4, 5));
    assert!(x.is_prerelease());
    assert_eq!(x.pre_release().as_deref(), Some("beta.2"));
    assert_eq!(Version::new(1, 0, 0).pre_release(), None);
    let y = Version::new(1, 0, 0).with_pre_release("rc.1").unwrap();
    assert_eq!(y.to_string(), "1.0.0-rc.1");
    assert_eq!(
        Version::new(1, 0, 0).with_pre_release("01"),
        Err(ParseError::LeadingZero { part: "prerelease" })
    );
}

#[test]
fn parse_errors_are_specific() {
    let cases: [(&str, ParseError); 13] = [
        ("", ParseError::Empty),
        ("   ", ParseError::Empty),
        ("1.2", ParseError::WrongNumberOfParts),
        ("1.2.3.4", ParseError::WrongNumberOfParts),
        ("a.2.3", ParseError::InvalidNumber { part: "major" }),
        ("1..3", ParseError::InvalidNumber { part: "minor" }),
        ("1.2.-3", ParseError::InvalidNumber { part: "patch" }),
        ("01.2.3", ParseError::LeadingZero { part: "major" }),
        ("1.2.03", ParseError::LeadingZero { part: "patch" }),
        (
            "99999999999999999999.0.0",
            ParseError::NumberTooLarge { part: "major" },
        ),
        ("1.2.3-", ParseError::InvalidIdentifier),
        ("1.2.3-a..b", ParseError::InvalidIdentifier),
        ("1.2.3+bad!", ParseError::InvalidIdentifier),
    ];
    for (input, want) in cases {
        assert_eq!(Version::parse(input), Err(want), "input {input:?}");
    }
    assert_eq!(
        Version::parse("1.2.3-01"),
        Err(ParseError::LeadingZero { part: "prerelease" })
    );
}

#[test]
fn errors_display_and_implement_error() {
    fn takes_error(_: &dyn std::error::Error) {}
    let e = Version::parse("1.2").unwrap_err();
    takes_error(&e);
    assert_eq!(e.to_string(), "expected MAJOR.MINOR.PATCH");
    let boxed: Box<dyn std::error::Error + Send + Sync> = e.into();
    assert!(boxed.to_string().contains("MAJOR"));
}

#[test]
fn bumping_resets_lower_parts_and_drops_pre_and_build() {
    let x = v("1.4.2-rc.1+b");
    assert_eq!(x.bump::<Major>().to_string(), "2.0.0");
    assert_eq!(x.bump::<Minor>().to_string(), "1.5.0");
    assert_eq!(x.bump::<Patch>().to_string(), "1.4.3");
    assert_eq!(
        x.to_string(),
        "1.4.2-rc.1+b",
        "bump does not modify the original"
    );
}

#[test]
fn requirements_match_ranges() {
    let r = VersionReq::parse(">=1.2.0, <2.0.0").unwrap();
    assert!(r.matches(&v("1.2.0")));
    assert!(r.matches(&v("1.99.9")));
    assert!(!r.matches(&v("1.1.9")));
    assert!(!r.matches(&v("2.0.0")));
    assert!(
        r.matches(&v("2.0.0-rc.1")),
        "2.0.0-rc.1 < 2.0.0 by plain precedence, so it is inside the range"
    );
    assert!(VersionReq::parse("1.0.0").unwrap().matches(&v("1.0.0+x")));
    assert!(VersionReq::parse("=1.0.0").unwrap().matches(&v("1.0.0")));
    assert!(VersionReq::parse(">1.0.0").unwrap().matches(&v("1.0.1")));
    assert!(VersionReq::parse("<=1.0.0").unwrap().matches(&v("1.0.0")));
    assert_eq!(r.to_string(), ">=1.2.0, <2.0.0");
}

#[test]
fn requirement_errors() {
    assert_eq!(VersionReq::parse(""), Err(ParseError::Empty));
    assert_eq!(
        VersionReq::parse("~1.2.0"),
        Err(ParseError::InvalidOperator)
    );
    assert_eq!(
        VersionReq::parse(">=1.2"),
        Err(ParseError::WrongNumberOfParts)
    );
    assert_eq!(
        VersionReq::parse(">=1.2.0,"),
        Err(ParseError::InvalidOperator)
    );
}

#[test]
fn auto_traits_and_sizes_are_what_the_api_promises() {
    fn send_sync<T: Send + Sync + 'static>() {}
    send_sync::<Version>();
    send_sync::<VersionReq>();
    send_sync::<ParseError>();
}
```

**Verification ✓.** `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, `cargo test --locked` and `RUSTDOCFLAGS="-D warnings" cargo doc --no-deps` all pass: 11 integration tests, 2 doctests that run, and 4 `compile_fail` doctests.

**Mutation checks ✓** (each mutation applied to a copy, run in its own target directory):

| # | Mutation | Result |
|---|---|---|
| M1 | make the `major` field `pub` | **caught**, but only by `#![deny(missing_docs)]` (the undocumented `pub` field fails to compile), not by a contract test |
| M1b | same, with a doc comment added to the now-public field | **caught** by the `compile_fail,E0616` doctest (the doctest then compiles, which is a failure) |
| M2 | remove the `Sealed` supertrait from `Part` | **caught** by the `compile_fail,E0277` doctest |
| M3 | remove `#[non_exhaustive]` from `ParseError` | **caught** by the `compile_fail,E0004` doctest |
| M4 | `Hash` also hashes `build` (Eq/Ord ignore it) | **caught** by `build_metadata_is_ignored_by_eq_ord_and_hash` |
| M5 | `Eq` also compares `build` (Ord/Hash ignore it) | **caught** by 2 tests (the same one plus `requirements_match_ranges`) |
| M6 | a release sorts *below* its pre-release | **caught** by 2 tests (the precedence chain and `requirements_match_ranges`) |
| M7 | swap the `Numeric`/`Alpha` variant order in `Ident` | **caught** by 2 tests (numeric-before-alphanumeric and the precedence chain) |
| M8 | accept leading zeros (`01.2.3`) | **caught** by `parse_errors_are_specific` |
| M9 | `bump::<Minor>` keeps the patch number | **caught** by `bumping_resets_lower_parts_and_drops_pre_and_build` |

All ten mutants were caught. One caution about my own method: my first mutation run used one shared `CARGO_TARGET_DIR` for all copies and produced misleading identical results (I believe cargo reused build output for the same package name, though I did not investigate); I discarded it and re-ran each mutant with a separate target directory, which is what the table reports.

**Design notes.**

- Modules `version`, `error`, `req` and `bump` are private; the crate root re-exports a deliberate list (§4). Moving code between those files is not a public change.
- `Ident` is a private enum with `#[derive(Ord)]`; the *declaration order* of its variants is part of the semantics (numeric before alphanumeric), so mutation M7 matters and there is a comment saying so.
- `with_core` is `pub(crate)`: `bump.rs` needs it, outsiders do not.
- `Eq`, `Ord` and `Hash` all derive from the same fields; `eq` is defined as `cmp == Equal` so the two cannot drift.
- `Part` is sealed through the private `sealed` module; the contract is pinned by a doctest. `Version::bump` is `#[must_use]` (✓ the unused-result warning appears in a scratch example).
- Not handled: Cargo-style `^` and `~` ranges, wildcard `1.*`, the pre-release opt-in rule of Cargo's `VersionReq`, `serde` support, and `u64` overflow in `bump` (it panics in debug builds; documented under `# Panics`). I did not fuzz the parser, and I did not run `cargo-semver-checks` on it.

</details>

---

## 13. Key takeaways

- A crate is a tree of modules; `mod` declares and loads, `use` only names. Both file layouts work; one module cannot have both (✓ `E0583`, `E0761`).
- Privacy is per module: private items are visible to descendants, not to the outside. A single private field blocks struct literals and lets you add fields later (✓ `E0451`, `E0616`, `E0624`, `E0603`).
- `pub use` is an alias, not a new type (✓). Use private modules plus a re-exported facade, and think of every `pub` item as a promise.
- Make misuse a compile error: private constructors, sealed traits (✓ `E0277`), `#[non_exhaustive]` (✓ `E0638`, `E0639`, `E0004`), `#[must_use]`.
- Know the breaking changes: added enum variant, added pub field, added required trait method, changed signature (✓ all four compiled and failed), and auto traits.
- `Eq`, `Ord` and `Hash` must agree (✓ `len = 2` in the debugging challenge).
- rustdoc is part of the API: doctests run, `compile_fail,E0xxx` pins contracts, intra-doc links and `missing_docs` catch rot, and `#[doc(hidden)]` is not privacy.
- Privacy is a design and safety-argument tool, not a security boundary.

**Connections.** [Lesson 3](03_cargo_projects.md) introduced modules and `pub`; [Lesson 13](13_structs_enums_methods.md) and [Lesson 15](15_newtype_typestate.md) give the types whose invariants privacy protects; [Lesson 19](19_generics_and_bounds.md)'s bounds are part of a public signature; [Lesson 22](22_gats_send_sync_phantom.md)'s auto traits explain why a private field can be a breaking change; [Lesson 26](26_error_design.md)'s error design is realized in `ParseError`. Next: Lesson 34 covers declarative macros and hygiene.

---

## 14. References

- The Rust Reference, "Visibility and privacy", "Modules", "Use declarations", and the `non_exhaustive` attribute: <https://doc.rust-lang.org/reference/>
- The Rust Book, chapter 7 "Managing Growing Projects with Packages, Crates, and Modules": <https://doc.rust-lang.org/book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html>
- Rust API Guidelines (naming, interoperability, documentation, predictability, flexibility, future proofing, including the *sealed trait* section): <https://rust-lang.github.io/api-guidelines/>
- The Cargo Book, "SemVer Compatibility" (the list of breaking and non-breaking changes): <https://doc.rust-lang.org/cargo/reference/semver.html>
- The rustdoc book (doc comments, doctests, `compile_fail`, intra-doc links, lints such as `missing_docs`): <https://doc.rust-lang.org/rustdoc/>
- Semantic Versioning 2.0.0 specification: <https://semver.org/>
- `cargo-semver-checks` (an automated checker for many of the changes in §5.3): <https://github.com/obi1kenobi/cargo-semver-checks>

The URLs above are the standard documentation locations; I did not open them during this session (the sandbox cannot reach them), so if a link has moved, search the site for the page title. Unchecked or from memory: the exact visibility table wording (`pub(self)`, `pub(in path)` constraints); the private-in-public lint names and levels in Rust 1.97; `#[doc(inline)]`/`#[doc(no_inline)]` behavior; `unreachable_pub` on trait and enum members; the semver table rows not marked ✓ (defaulted trait methods, glob-import clashes, loosening bounds, MSRV policy) and the claim that `impl Into<String>` can break explicit function-pointer uses; cross-crate inlining and `#[inline]`; the doctest hidden-line (`# `) rules and the full list of fence attributes; the API Guidelines checklist items; the semver 2.0 precedence examples (the `semverlab` tests encode my reading of the rules, which I did not check against the specification text or against the `semver` crate); `cargo-semver-checks` (not installed here). Everything marked ✓ was run on this machine's `rustc 1.97.0`.
