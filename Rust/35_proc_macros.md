---
layout: default
title: "Lesson 35 — Procedural macros: derives, attributes, and function-like macros"
---

# Lesson 35 — Procedural macros: derives, attributes, and function-like macros

**Module 11 · Modules, API design, macros** · Status: generated · Edition used in examples: **2024** · External dependencies: `syn` 2.0.119 (feature `full`), `quote` 1.0.47, `proc-macro2` 1.0.107, and `trybuild` 1.0.122 for compile-fail tests (✓ all downloaded from crates.io and run; these are the versions `Cargo.lock` resolved in this sandbox today)

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux with 2 cores. As before, ✓ marks outputs, diagnostics and measurements I actually ran; unmarked statements are documented behavior I did not run. Everything is **stable Rust**. `cargo expand` is not installed; for one section I ran the compiler's nightly-only `-Zunpretty=expanded` flag on the stable compiler through `RUSTC_BOOTSTRAP=1`, which is an unsupported trick that I used only to show you real expansions (don't use it in a real project). I could not open the `syn`, `quote` or Rust documentation sites, so API details come from memory plus what the compiler accepted; the closing paragraph lists them.

Builds on [Lesson 34](34_declarative_macros.md) (`macro_rules!`, hygiene, tokens), [Lesson 33](33_modules_api_design_rustdoc.md) (facade crates, re-exports, semver), [Lesson 26](26_error_design.md) (error design) and [Lesson 19](19_generics_and_bounds.md) (generics and `where` clauses). The next lesson, Lesson 36, starts the unsafe-Rust module; it is mentioned here only as plain text.

---

## 1. What a procedural macro is

A **procedural macro** is a Rust function that the compiler *runs while compiling your crate*. It receives a stream of tokens and returns a stream of tokens. Unlike `macro_rules!` ([Lesson 34](34_declarative_macros.md)), it is ordinary Rust code: it can parse real syntax, loop, branch, look at attributes, and produce precise error messages.

There are three kinds, defined by what they take and where they may be used:

| Kind | Declaration | Input | Output | Used as |
|---|---|---|---|---|
| Derive | `#[proc_macro_derive(Name, attributes(helper))]` | the struct/enum/union definition | **extra items** (the original item is kept unchanged) | `#[derive(Name)]` |
| Attribute | `#[proc_macro_attribute]` | the attribute's arguments **and** the annotated item | **replacement** for the item | `#[name(args)] fn ...` |
| Function-like | `#[proc_macro]` | whatever is inside the parentheses | an expression, statements or items | `name!(...)` |

The compiler loads the macro crate as a dynamic library and calls it in-process. Consequences:

1. A proc-macro crate must be its own crate with `proc-macro = true` in `[lib]`. It is compiled for the **host** machine (the one running the compiler), even when you cross-compile.
2. It may only export proc macros ✓. Adding a plain `pub fn helper() {}` fails:

   ```text
   error: `proc-macro` crate types currently cannot export any items other than functions tagged with `#[proc_macro]`, `#[proc_macro_derive]`, or `#[proc_macro_attribute]`
   ```

   So shared types (traits, error types, runtime helpers) live in a normal *runtime* crate, and the macro crate is a dependency of it. That is why `serde` + `serde_derive` are two crates, and why this lesson has `proclab` + `proclab-macros`.
3. The macro runs with the **compiler's** privileges at build time (§11).

### 1.1 The smallest possible macro ✓

```rust
use proc_macro::TokenStream;

/// Expands to the literal `42`.
#[proc_macro]
pub fn answer(_input: TokenStream) -> TokenStream {
    "42".parse().unwrap()
}
```

In a crate with `[lib] proc-macro = true`, and a second crate that depends on it by path, `println!("{}", answer::answer!());` ✓ prints `42`. A `TokenStream` parses from a string, which is fine for toys but never for real work: you lose spans, and string-gluing code is unreadable. A macro that **panics** is reported as a compile error with the panic message, not as a crash ✓:

```text
error: proc macro panicked
  = help: message: this macro is broken
```

Panicking is still the wrong way to report bad input: the message has no useful location. §7 shows the right way.

---

## 2. The toolbox: `proc_macro`, `proc-macro2`, `syn`, `quote`

- **`proc_macro`**: the compiler's own API (`TokenStream`, `TokenTree`, `Span`). It exists only while the compiler is running the macro, so code that uses it can't be unit-tested normally.
- **`proc-macro2`**: a drop-in wrapper with the same shape that also works in ordinary tests and ordinary programs. Convert at the boundary (`.into()` both ways) and write all logic against `proc_macro2::TokenStream`. This is the key structure trick of this lesson: every macro in `proclab-macros` is a thin wrapper around an `expand(...) -> syn::Result<proc_macro2::TokenStream>` function, and `expand` is unit-tested with `syn::parse_quote!` (§9).
- **`syn`**: a parser for Rust syntax that produces typed trees: `DeriveInput` (a struct/enum/union definition), `ItemFn`, `LitStr`, `Ident`, `Generics`, `Attribute`, and so on. The `full` feature adds function bodies and all item kinds (derive-only macros can use the lighter `derive` feature; from memory).
- **`quote`**: `quote! { ... }` builds a token stream from a template. `#name` interpolates a value that implements `ToTokens`; `#( #items ),*` repeats over an iterator, like `macro_rules!` repetitions.

Dependency line used in this lesson:

```toml
[dependencies]
syn = { version = "2", features = ["full"] }
quote = "1"
proc-macro2 = "1"
```

✓ Cargo resolved `syn 2.0.119`, `quote 1.0.47`, `proc-macro2 1.0.107`, and noted that a newer `syn v3.0.9` exists ("available: v3.0.9"). This lesson uses syn 2 and I did not try syn 3, whose API may differ; pin the major version you wrote against.

A compile-time cost to know: a clean build of `proclab-macros` (compiling `proc-macro2`, `quote`, `unicode-ident`, `syn` with `full`, and the macros) took about 4.0 s wall-clock in the dev profile on this 2-core machine ✓ (one run, not a benchmark). Every downstream user pays for that once per clean build, which is why macro-heavy dependency trees are slow to build from scratch.

---

## 3. Derive macros

A derive macro receives the item and returns **additional** items, typically one `impl`. Our `FieldNames` derive:

```rust
// usage
use proclab::FieldNames;

#[derive(FieldNames)]
struct Plain {
    id: u32,
    name: String,
}

fn main() {
    println!("{:?}", <Plain as FieldNames>::FIELD_NAMES);
}
```

(This block needs the `proclab` crate from §15, so it is not a standalone program; the same behavior is tested there: `Plain::FIELD_NAMES == ["id", "name"]` ✓.)

The skeleton of every derive:

```rust
// inside the macro crate (excerpt)
#[proc_macro_derive(FieldNames, attributes(names))]
pub fn derive_field_names(input: TokenStream) -> TokenStream {
    let input = syn::parse_macro_input!(input as syn::DeriveInput);
    field_names::expand(&input)
        .unwrap_or_else(syn::Error::into_compile_error)
        .into()
}
```

1. `parse_macro_input!` parses the tokens as a `DeriveInput`; if parsing fails it returns a compile error for you.
2. `expand` returns `syn::Result<TokenStream2>`; an `Err` becomes `compile_error!(...)` at the right span via `into_compile_error`.
3. `attributes(names)` declares the **helper attribute** `#[names(...)]`; without that line the compiler rejects `#[names(skip)]` on a field as an unknown attribute.

### 3.1 Generics

A derive must keep the user's generics, bounds and `where` clause, or it breaks on any generic type. `split_for_impl` does the three pieces:

```rust
let (impl_generics, ty_generics, where_clause) = input.generics.split_for_impl();
quote! {
    impl #impl_generics ::proclab::FieldNames for #name #ty_generics #where_clause {
        const FIELD_NAMES: &'static [&'static str] = &[ #(#names),* ];
    }
}
```

✓ The test type `Generic<'a, T: Clone, const N: usize> where T: Send` (lifetime, bounded type parameter, const generic and where clause) derives correctly. The mutation that drops the generics from the `impl` ✓ fails to compile with `E0107` ("missing generics for struct") and `E0726` ("implicit elided lifetime not allowed here").

Standard derives such as `Clone` add a bound `T: Clone` for each type parameter. If your trait should bound parameters differently, add predicates to `generics.make_where_clause()` yourself (from memory).

### 3.2 Helper attributes and `parse_nested_meta`

`#[names(skip)]` and `#[names(rename = "x")]` are parsed with `Attribute::parse_nested_meta`:

```rust
attr.parse_nested_meta(|meta| {
    if meta.path.is_ident("skip") {
        opts.skip = true;
        Ok(())
    } else if meta.path.is_ident("rename") {
        opts.rename = Some(meta.value()?.parse()?);
        Ok(())
    } else {
        Err(meta.error("unsupported `names` option, expected `skip` or `rename = \"..\"`"))
    }
})?;
```

`meta.value()?.parse()?` reads `= "x"`. `meta.error(...)` creates an error **spanned at the offending option**, so the user sees the squiggle under `oops` and not under the whole struct (✓ §7). Reject unknown options: silently ignoring a typo like `#[names(skp)]` is a bug magnet.

### 3.3 What the compiler actually generates ✓

With the unsupported `RUSTC_BOOTSTRAP=1 cargo rustc -- -Zunpretty=expanded` trick, the derive for `struct P { x: i32, #[names(skip)] y: i32 }` and `enum M { A, B }` expanded to:

```rust
impl ::proclab::FieldNames for P {
    const FIELD_NAMES: &'static [&'static str] = &["x"];
}
impl ::proclab::EnumStr for M {
    const ALL: &'static [Self] = &[Self::A, Self::B];
    fn as_str(&self) -> &'static str {
        match self { Self::A => "A", Self::B => "B", }
    }
}
impl ::core::fmt::Display for M { /* ... */ }
impl ::core::str::FromStr for M {
    type Err = ::proclab::UnknownVariant;
    fn from_str(s: &str) -> ::core::result::Result<Self, Self::Err> {
        match s {
            "A" => ::core::result::Result::Ok(Self::A),
            "B" => ::core::result::Result::Ok(Self::B),
            other => ::core::result::Result::Err(::proclab::UnknownVariant::new("M", other)),
        }
    }
}
```

Note the **absolute paths** (`::proclab::...`, `::core::...`). Generated code is pasted into the user's crate, where `Result`, `Some` or `Display` might be shadowed or not imported. Always generate fully qualified paths.

### 3.4 Referring to your own crate

The expansion says `::proclab::FieldNames`. That path is valid in downstream crates that depend on `proclab`, but not inside `proclab` itself (its own tests and doctests). The standard trick is one line in the runtime crate's `lib.rs`:

```rust
extern crate self as proclab;
```

which makes `::proclab` name the current crate. Also note the pattern used by the practical challenge: the runtime crate re-exports the macros (`pub use proclab_macros::{FieldNames, ...}`), so users depend on one crate and write `use proclab::FieldNames;`. A trait and a derive macro with the **same name** can coexist because traits live in the type namespace and macros in the macro namespace; one `use` imports both (this is how `serde::Serialize` works; from memory).

---

## 4. Attribute macros

An attribute macro receives **two** token streams, the attribute arguments and the item, and replaces the item with whatever it returns. `#[traced]` wraps a function body so that entry and exit are recorded:

```rust
// inside the macro crate (excerpt)
pub fn expand(args: TokenStream, item: TokenStream) -> syn::Result<TokenStream> {
    let mut func: ItemFn = syn::parse2(item)?;
    if let Some(asyncness) = &func.sig.asyncness {
        return Err(syn::Error::new_spanned(
            asyncness,
            "#[traced] does not support async functions",
        ));
    }
    let label = parse_label(args, &func.sig.ident)?;
    let guard = Ident::new("__proclab_guard", Span::mixed_site());
    let stmts = &func.block.stmts;
    *func.block = syn::parse_quote!({
        let #guard = ::proclab::trace::Guard::enter(#label);
        #(#stmts)*
    });
    Ok(quote! { #func })
}
```

The recorded events go to a thread-local log through a guard type whose `Drop` records `leave`. That choice matters: because `leave` is in `Drop`, it is recorded on early `return`, on `?` and **while unwinding from a panic** ✓ (the tests assert `["enter panics", "leave panics"]` after a caught panic). A macro that appended "log leave" code after the original body would miss all three.

Decisions worth noticing:

- The macro keeps the signature (`func.sig`, attributes, visibility) untouched and only edits the block. Methods work too ✓ (`#[traced(name = "Counter::bump")]` on an `impl` method).
- **Do not nest the old block** (`{ let g = ...; #block }`) ✓. It looks equivalent, but the original braces stay in the output, and for a one-line function the compiler then reports `warning: unnecessary braces around block return value` **in the user's code** ✓ (§13). Splicing the statements (`#(#stmts)*`) keeps the tail expression's value and avoids the warning.
- Attribute arguments are parsed with `syn::meta::parser(...)` (the same `name = "x"` style as `parse_nested_meta`).
- `async fn` is rejected with a spanned error; guarding an `async` body needs more thought (the guard would live across `.await` points, and the future may be dropped at any of them, [Lesson 32](32_cancellation_streams_backpressure.md)).

---

## 5. Function-like macros

`hex!("deadBEEF")` decodes a string literal at compile time into `[0xde, 0xad, 0xbe, 0xef]`:

```rust
// inside the macro crate (excerpt)
pub fn expand(lit: &LitStr) -> syn::Result<TokenStream> {
    let bytes = decode(&lit.value()).map_err(|msg| syn::Error::new(lit.span(), msg))?;
    Ok(quote! { [ #(#bytes),* ] })
}
```

`#(#bytes),*` repeats over a `Vec<u8>`; `quote` writes each as a suffixed literal such as `222u8` (from memory), so the array has type `[u8; N]` and is usable in `const` context ✓ (`const KEY: [u8; 4] = hex!("DEADbeef");`). Bad input is a compile error pointing at the literal ✓ (`odd number of hex digits (3)` for `"abc"`, `invalid hex digit in byte 0` for `"zz"`). `decode` is a plain function with its own unit tests, including a non-ASCII input that must not panic ✓ (the loop works on bytes, not `char` boundaries).

Compare with `macro_rules!`: you could write `hex!` as a declarative macro only by matching hex digits as tokens (awkward, and `1e5` is a float token!). With a proc macro, the string literal is parsed once and validated with real code. That is the usual dividing line: **`macro_rules!` for pattern-based rewriting; proc macros for anything that needs real parsing, inspection of attributes, or computation.**

---

## 6. Spans and hygiene

Every token has a **span**: where it came from, used for error locations and for name resolution ("hygiene").

- `Span::call_site()`: resolves names as if the code were written where the macro was called. `quote!` uses this by default.
- `Span::mixed_site()`: behaves like `macro_rules!` hygiene: local variables and labels are resolved at the macro definition site, everything else at the call site.
- `Span::def_site()`: fully hygienic; it is unstable (nightly) as far as I know (from memory).
- Tokens copied from the input **keep their input span**, so errors in generated code that came from the user's tokens point at the user's code.

Why it matters, demonstrated ✓. `#[traced]` introduces a local variable. With `call_site` spans, a user parameter that happens to have the same name is *shadowed by the macro's variable*:

```rust
#[traced]
fn shadow(__proclab_guard: u32) -> u32 {
    __proclab_guard + 1
}
```

With `Span::mixed_site()` (our code) this compiles and `shadow(41) == 42` ✓. With the mutation that switches to `call_site` the same test fails to compile with `error[E0369]: cannot add `{integer}` to `Guard`` ✓: the user's `__proclab_guard + 1` resolved to the macro's `Guard`. Real users rarely pick such a name, but the same mechanism hits generated names like `result`, `value` or `tmp`, which users do use. Either use `mixed_site` identifiers or generate very unlikely names.

Spans also decide where errors point. In `enum_str` the "duplicate string" error is created from the *literal's* span, and `LitStr::new(&name, variant.ident.span())` gives synthesized literals the span of the variant, so a diagnostic about a generated name points at something the user wrote.

---

## 7. Reporting errors

Never `panic!` or `unwrap()` on user input. Build a `syn::Error`:

- `syn::Error::new(span, message)`: error at a `Span`.
- `syn::Error::new_spanned(tokens, message)`: error covering the tokens (any `ToTokens`: an `Ident`, a field, a whole variant).
- `err.combine(other)`: report several errors at once (we use it to show both occurrences of a duplicate name).
- `err.into_compile_error()`: the token stream `compile_error!{...}` carrying the span.

The UI tests of the practical challenge pin the user-visible messages ✓. Two of them (the other seven are in the crate):

```text
error: FieldNames needs a struct with named fields
 --> tests/ui/field_names_on_tuple.rs:4:8
  |
4 | struct Pair(u8, u8);
  |        ^^^^

error: duplicate name `x`
 --> tests/ui/field_names_duplicate.rs:6:22
  |
6 |     #[names(rename = "x")]
  |                      ^^^

error: first used here
 --> tests/ui/field_names_duplicate.rs:5:5
  |
5 |     x: u8,
  |     ^
```

Notes:

- Report *one problem per place*: point at the exact token.
- Validate **everything you rely on**: a derive that assumes named fields and uses `.unwrap()` produces "proc macro panicked" for `struct S(u8);`. A rejected input should explain what is supported.
- Keep going when you can: collect errors with `combine` rather than stopping at the first, so the user fixes several problems per compile. (Our macros stop at the first error for simplicity, except duplicates.)

---

## 8. Design guidance

- **Prefer not to write one.** Proc macros are slow to compile, hard to debug and hard for tools to understand. Try generics, traits with default methods, `const fn`, or `macro_rules!` first ([Lesson 34](34_declarative_macros.md)).
- **Keep the macro thin.** Generate code that calls into ordinary generic functions in the runtime crate. Smaller expansions mean faster builds and readable errors.
- **Generate boring code.** Absolute paths, no clever tricks, no reliance on what the user imported.
- **Follow derive conventions**: derive only on the types that need it; support generics; don't require the user to import anything but the macro name.
- **Document the accepted syntax** like an API ([Lesson 33](33_modules_api_design_rustdoc.md)): the crate docs of `proclab` double as the specification and a doctest ✓.
- **Version the pair together.** The macro crate's expansion is coupled to the runtime crate's API (`::proclab::trace::Guard::enter`). Pin them: `proclab-macros = { path = "...", version = "0.1.0" }` (here) or `=0.1.0` in a published crate (from memory), and change them in lock-step.
- **Semver.** Changing *what a macro accepts or generates* is a public-API change. Adding generated items can clash with user items; removing support breaks users.

---

## 9. Testing proc macros

Three layers, all used in `proclab`:

1. **Unit tests of `expand`** (inside the macro crate): parse with `syn::parse_quote!` and assert on the generated tokens (as a string) or on the error message ✓ (9 unit tests). Cheap and fast, but string comparison is brittle: the token printer inserts spaces (`for P < T >`, `where T : Send`) and assertions should check fragments, not whole outputs.
2. **Behavior tests** (an integration test crate that *uses* the macros): derive on realistic types, including generics, lifetimes, const generics and an empty struct; then assert on behavior ✓ (12 tests).
3. **Compile-fail ("UI") tests** with `trybuild`: each file in `tests/ui/` must fail with the exact diagnostics stored in a `.stderr` file next to it. `TRYBUILD=overwrite cargo test` writes the snapshots; **read them** before committing, because accepting garbage is easy ✓ (9 cases, 4.6 s for the whole `ui` test on a cold build). The downside is brittleness: compiler wording and span rendering change between Rust versions, so these snapshots may need regenerating on a toolchain upgrade (from memory), which is why large projects pin the toolchain used for UI tests.

Pitfalls found while writing this lesson ✓:

- `cargo fmt` reformats a one-line function into several lines. My first regression test for the `unused_braces` bug used a one-line function, was reformatted by `rustfmt`, and then **no longer triggered the bug**: the mutant that reintroduced the bug survived. I fixed it with `#[rustfmt::skip]` on that function and the mutant is now caught. Tests of syntax-sensitive behavior must pin the formatting they depend on.
- `cargo expand` or the nightly flags show what is generated, but a test is better than a look.

---

## 10. Common mistakes

1. **Panicking on bad input** instead of returning a spanned `syn::Error`.
2. **Relative or unqualified paths in generated code** (`Result`, `Display`, `proclab::X` without the leading `::`).
3. **Dropping generics** or the `where` clause in derives (✓ `E0107`/`E0726`).
4. **Ignoring unknown helper-attribute options**, which hides typos.
5. **Nesting the original block** in an attribute macro (✓ `unused_braces`), or evaluating the body inside a closure that changes `return` semantics.
6. **Hygiene accidents**: generated locals with common names under `call_site`.
7. **Not handling every `Data` variant** (`Struct`, `Enum`, `Union`) and every `Fields` shape (`Named`, `Unnamed`, `Unit`).
8. **Doing I/O or nondeterministic work** in the macro (reading files, network, time, random numbers): results differ between builds, and incremental compilation and caches assume expansion is a pure function of its input.
9. **Giant expansions** instead of calls into a runtime function.
10. **Testing only valid input.** The UI tests and unit tests for errors are half the value.
11. **Forgetting `attributes(...)`** for helper attributes.
12. **Using a macro where a trait with a blanket impl works.**

---

## 11. Performance, reliability, security

- **Compile time.** Dependencies (`syn` with `full` is the heavy one) are built once per clean build (✓ about 4 s here for the macro crate, one run). Each expansion is re-run whenever the crate is recompiled that contains the use site. Generated code is ordinary code, so it is as fast at run time as hand-written code with the same shape; the guard in `#[traced]` costs a thread-local access and a `Vec` push at both ends, which is fine for tracing and wrong for a hot loop. I did not benchmark any of this.
- **Reliability.** Expansion must be deterministic and free of side effects. A macro that reads the filesystem can make builds irreproducible and cause stale incremental results. Errors must be actionable; unsupported input must be rejected, not half-handled.
- **Security.** A procedural macro is **arbitrary code that runs at build time with your user's privileges**. This is not hypothetical ✓: the demo macro `noisy!` printed `noisy: running at COMPILE time, cwd = "user"` to stderr during `cargo build`, and it could just as easily have read `~/.ssh` or the environment, or opened a socket. Every proc-macro dependency (and every `build.rs`) is in your trusted computing base. Mitigations: audit and pin dependencies, use `cargo vet`/`cargo deny` or equivalent, build in a sandbox/CI without secrets, and be wary of `rust-analyzer` expanding macros in untrusted projects (it runs them too; from memory). Experiments with sandboxing macros (for example by compiling them to WebAssembly) exist but are not part of stable Rust as far as I know. Rust's memory safety says nothing about this: the compiler loads and runs the macro.
- **Generated `unsafe`.** If a derive emits `unsafe`, it takes on the safety obligations on behalf of every user; make the generated code *sound for all valid inputs* and document the preconditions (Lesson 36 covers unsafe code; mentioned only as plain text here).

---

## 12. Exercises

Answers are collapsed. Try each before opening it.

1. **(Warm-up)** Why must a proc-macro crate be separate from the crate that defines the traits it generates impls for? What can it export?
2. **(Warm-up)** Give the input and output of a derive macro, an attribute macro and a function-like macro.
3. **(Easy)** Why does the generated code say `::proclab::FieldNames` and why does `proclab` contain `extern crate self as proclab;`?
4. **(Medium)** Write a derive `TypeName` that adds an associated constant `TYPE_NAME: &'static str` to the type, supporting generics.
5. **(Medium)** Write a function-like macro `make_const_fn!(name, 42)` that expands to `fn name() -> i32 { 42 }`, with a custom `Parse` implementation.
6. **(Medium-hard)** Why must `#[traced]` use a `Drop` guard rather than appending a "leave" statement after the body? Name three situations where it matters.
7. **(Hard)** A derive needs the bound `T: Display` on every type parameter. What do you change in the `impl` generation?
8. **(Hard)** Your derive should report an error on a *field's type* when it is not supported, and report all bad fields in one compile. How?

<details markdown="1">
<summary>Answers</summary>

1. A proc-macro crate (`proc-macro = true`) is a compiler plugin built for the host and can only export proc macros (✓ the error message for a plain `pub fn`). A trait or runtime helper would have to live in a normal crate, so the usual layout is a runtime crate that depends on the macro crate and re-exports it, as in `proclab` / `proclab-macros`.
2. Derive: input is the item definition (`DeriveInput`); output is additional items, and the original is kept. Attribute: inputs are the attribute arguments and the item; output replaces the item. Function-like: input is the tokens inside `name!(...)`; output is an expression, statements or items depending on the call site.
3. The expansion is pasted into the *user's* crate, which depends on `proclab`, so `::proclab::...` is an absolute path that cannot be shadowed. Inside `proclab` itself (its tests and doctests) there is no external crate called `proclab`; `extern crate self as proclab;` makes `::proclab` refer to the current crate.
4. ```rust
   use proc_macro::TokenStream;
   use quote::quote;
   use syn::{DeriveInput, parse_macro_input};

   #[proc_macro_derive(TypeName)]
   pub fn type_name(input: TokenStream) -> TokenStream {
       let input = parse_macro_input!(input as DeriveInput);
       let name = &input.ident;
       let text = name.to_string();
       let (impl_generics, ty_generics, where_clause) = input.generics.split_for_impl();
       quote! {
           impl #impl_generics #name #ty_generics #where_clause {
               pub const TYPE_NAME: &'static str = #text;
           }
       }
       .into()
   }
   ```
   (It is an inherent `impl`, so a user can write `Point::TYPE_NAME`. Generic types keep their parameters and `where` clause.)
5. ```rust
   use proc_macro::TokenStream;
   use quote::quote;
   use syn::parse::{Parse, ParseStream};
   use syn::{Ident, LitInt, Token, parse_macro_input};

   struct Args {
       name: Ident,
       value: LitInt,
   }

   impl Parse for Args {
       fn parse(input: ParseStream) -> syn::Result<Self> {
           let name = input.parse()?;
           input.parse::<Token![,]>()?;
           let value = input.parse()?;
           Ok(Args { name, value })
       }
   }

   #[proc_macro]
   pub fn make_const_fn(input: TokenStream) -> TokenStream {
       let Args { name, value } = parse_macro_input!(input as Args);
       quote! { fn #name() -> i32 { #value } }.into()
   }
   ```
   A malformed call such as `make_const_fn!(f)` gives a compile error from `input.parse::<Token![,]>()?` pointing at the end of the input; `parse_macro_input!` converts the `Err` for you.
6. A "leave" statement placed after the body never runs on an early `return`, on `?` returning an `Err`, or while unwinding from a panic. `Drop` runs in all three (✓ the tests record `leave` for `early(true)`, for `fails(-1)` and for `panics()` under `catch_unwind`). (It would also be wrong to wrap the body in a closure to catch `return`: that changes what `return` and `?` mean inside the function.)
7. Add a predicate for each type parameter to the `where` clause before splitting:
   ```rust
   let mut generics = input.generics.clone();
   for param in generics.type_params_mut() {
       param.bounds.push(syn::parse_quote!(::core::fmt::Display));
   }
   let (impl_generics, ty_generics, where_clause) = generics.split_for_impl();
   ```
   (This sketch needs `syn`; I did not compile it separately. `type_params_mut` and `bounds.push` are from memory of the syn API.)
8. Create one `syn::Error` per bad field with `Error::new_spanned(&field.ty, "unsupported type ...")` (the span covers the type) and merge them: keep an `Option<syn::Error>` accumulator and call `acc.combine(e)` (or create it with the first error). Return `Err(acc)` after the loop. All errors are emitted in one compile. The duplicate-name error in `field_names.rs` uses `combine` this way for two locations.

</details>

---

## 13. Debugging challenge

An attribute macro `#[log_calls]` prints `call <name>` when a function starts. It works, but every function it is applied to now produces a warning *in the user's code*, pointing at the function's own braces. The macro crate:

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{ItemFn, parse_macro_input};

#[proc_macro_attribute]
pub fn log_calls(_args: TokenStream, item: TokenStream) -> TokenStream {
    let mut func = parse_macro_input!(item as ItemFn);
    let name = func.sig.ident.to_string();
    let body = &func.block;
    *func.block = syn::parse_quote!({
        eprintln!("call {}", #name);
        #body
    });
    quote!(#func).into()
}
```

and the user's `main.rs`:

```rust
use log_calls::log_calls;

#[log_calls]
fn double(x: u32) -> u32 { x * 2 }

fn main() {
    println!("{}", double(21));
}
```

✓ `cargo build` prints a warning, and the program prints `call double` and `42`:

```text
warning: unnecessary braces around block return value
 --> src/main.rs:4:26
  |
4 | fn double(x: u32) -> u32 { x * 2 }
  |                          ^^     ^^
  = note: `#[warn(unused_braces)]` (part of `#[warn(unused)]`) on by default
```

Explain why the warning appears (and why only for some functions), and fix the macro.

<details markdown="1">
<summary>Solution</summary>

`#body` interpolates the *original block* `{ x * 2 }`, braces included, as the last expression of the new block. The result is `{ eprintln!(...); { x * 2 } }`. The inner braces carry the **user's source spans**, so the compiler's `unused_braces` lint sees "a block whose only content is a value" in the user's own file and reports it as if the user had written it. (✓ A function whose body spans several lines did not trigger it in my runs, which is why the bug can hide; the lint's exact conditions are from memory, so treat "only one-line bodies" as an observation, not a rule.)

Fix: splice the *statements* of the original block into the new block instead of nesting the whole block (✓ the warning disappears and the output is unchanged, `call double` and `42`):

```rust
let stmts = &func.block.stmts;
*func.block = syn::parse_quote!({
    eprintln!("call {}", #name);
    #(#stmts)*
});
```

The tail expression is the last element of `stmts`, so the function still returns its value. The `proclab` crate does the same and has a regression test (`single_line_bodies_do_not_warn_and_keep_their_value`) with `#![deny(unused_braces)]` and `#[rustfmt::skip]` on the one-line function. Mutation M14b in §15 puts the bug back and is caught. General lesson: tokens you copy from the input keep their spans and so their lint behavior; code your macro writes itself gets `call_site` or `mixed_site` spans that user-facing lints skip.

</details>

---

## 14. Interview questions

1. What are the three kinds of procedural macros, and what does each receive and return?
2. Why do proc macros need their own crate, and what does that imply for a library like `serde`?
3. What do `syn`, `quote` and `proc-macro2` each do, and why `proc-macro2`?
4. How should a proc macro report invalid input?
5. What is a helper attribute and how do you declare it?
6. What is a span, and what is the difference between `call_site` and `mixed_site`?
7. How would you test a derive macro?
8. What are the security implications of using a proc-macro dependency?

<details markdown="1">
<summary>Answers</summary>

1. Derive: item definition in, extra items out. Attribute: (arguments, item) in, replacement item(s) out. Function-like: the tokens inside the call in, an expression/statements/items out.
2. The macro crate is compiled for the host and loaded by the compiler, and can export only proc macros (✓ the error for a plain function). So `serde` is the runtime crate with traits and `serde_derive` holds the macros; `serde` re-exports them (from memory) so users depend on one crate.
3. `syn` parses token streams into typed Rust syntax trees; `quote!` builds token streams from templates with interpolation; `proc-macro2` is a wrapper of the compiler's `proc_macro` types that also works outside a macro invocation, which makes the logic unit-testable (✓ our 9 unit tests use `parse_quote!` and plain `#[test]`s).
4. Return a `syn::Error` with a precise span (`new_spanned`, `meta.error`), convert it with `into_compile_error()`, and combine multiple errors where possible. Don't panic (✓ "proc macro panicked" gives no location).
5. An extra attribute that the derive understands, such as `#[names(skip)]`; it is declared in the derive attribute: `#[proc_macro_derive(FieldNames, attributes(names))]`. Parse it with `parse_nested_meta`, and reject unknown options.
6. A span is a token's source location and also the context in which its names are resolved. `call_site` resolves names as if written at the call; `mixed_site` resolves local variables and labels at the macro's definition (like `macro_rules!`), so generated locals can't collide with the user's (✓ a user variable `__proclab_guard` collided under `call_site`: `E0369`).
7. Unit-test `expand` with `parse_quote!` (✓), write integration tests that use the macros on varied types including generics (✓), and use `trybuild` UI tests for the error messages (✓, with the caveat that snapshots depend on the compiler version).
8. A proc macro runs arbitrary code at compile time with the developer's privileges (✓ `noisy!` ran at build time). A malicious or compromised macro crate can read files, environment variables and the network. Treat these dependencies as trusted code: audit, pin, and build in a restricted environment.

</details>

---

## 15. Practical challenge: `proclab`

Build a two-crate workspace: the runtime crate `proclab` and the macro crate `proclab-macros`. Requirements:

1. `#[derive(FieldNames)]` on structs with named fields: associated const `FIELD_NAMES`, declaration order, helper attributes `#[names(skip)]` and `#[names(rename = "x")]`, correct for generics, lifetimes, const generics and where-clauses, with specific errors for tuple structs, enums, unknown options, `skip` + `rename`, and duplicate names (with both locations).
2. `#[derive(EnumStr)]` for fieldless enums: `ALL`, `as_str`, `Display`, `FromStr` with an `UnknownVariant` error type; `#[enum_str(rename_all = "lowercase" | "snake_case")]` and per-variant `rename` (which wins); errors for non-unit variants, non-enums, generics, empty enums, bad `rename_all`, and duplicate strings.
3. `#[traced]` / `#[traced(name = "label")]` on functions and methods: record `enter`/`leave` in a thread-local log through a `Drop` guard, preserving the signature and return value, no `unused_braces` warning on one-line bodies, hygienic guard variable, error on `async fn` and on non-functions.
4. `hex!("..")`: byte array decoded at compile time, usable in `const`; spanned errors for odd length and invalid digits; no panic on non-ASCII input.
5. All macros are thin wrappers over `expand(...) -> syn::Result<proc_macro2::TokenStream>`; generated code uses only absolute paths (`::proclab`, `::core`).
6. Tests: unit tests of each `expand`, behavior tests, and `trybuild` UI tests with snapshots for every error class; `#![deny(missing_docs)]` and a main doctest.

<details markdown="1">
<summary>Solution (spoilers: full source)</summary>

`Cargo.toml` (workspace root and runtime crate):

```toml
[workspace]
members = ["proclab-macros"]

[package]
name = "proclab"
version = "0.1.0"
edition = "2024"

[dependencies]
proclab-macros = { path = "proclab-macros", version = "0.1.0" }

[dev-dependencies]
trybuild = "1"
```

`proclab-macros/Cargo.toml`:

```toml
[package]
name = "proclab-macros"
version = "0.1.0"
edition = "2024"

[lib]
proc-macro = true

[dependencies]
syn = { version = "2", features = ["full"] }
quote = "1"
proc-macro2 = "1"
```

`proclab-macros/src/lib.rs`:

```rust
//! Procedural macros for `proclab`. Use them through the `proclab` crate.
//!
//! Each macro is a thin wrapper: it converts `proc_macro::TokenStream` to
//! `proc_macro2` and calls an `expand` function that returns
//! `syn::Result<TokenStream2>`. Errors become `compile_error!` invocations
//! with the right span, so the compiler points at the user's code.

use proc_macro::TokenStream;

mod enum_str;
mod field_names;
mod hex;
mod traced;

/// Derives `proclab::FieldNames` for a struct with named fields.
///
/// Helper attribute: `#[names(skip)]` and `#[names(rename = "x")]` on fields.
#[proc_macro_derive(FieldNames, attributes(names))]
pub fn derive_field_names(input: TokenStream) -> TokenStream {
    let input = syn::parse_macro_input!(input as syn::DeriveInput);
    field_names::expand(&input)
        .unwrap_or_else(syn::Error::into_compile_error)
        .into()
}

/// Derives `proclab::EnumStr`, `Display` and `FromStr` for a fieldless enum.
///
/// Helper attribute: `#[enum_str(rename_all = "lowercase" | "snake_case")]`
/// on the enum and `#[enum_str(rename = "x")]` on variants.
#[proc_macro_derive(EnumStr, attributes(enum_str))]
pub fn derive_enum_str(input: TokenStream) -> TokenStream {
    let input = syn::parse_macro_input!(input as syn::DeriveInput);
    enum_str::expand(&input)
        .unwrap_or_else(syn::Error::into_compile_error)
        .into()
}

/// Records entry and exit of a function in `proclab::trace`.
///
/// `#[traced]` uses the function name; `#[traced(name = "label")]` overrides it.
#[proc_macro_attribute]
pub fn traced(args: TokenStream, item: TokenStream) -> TokenStream {
    let args = proc_macro2::TokenStream::from(args);
    let item = proc_macro2::TokenStream::from(item);
    traced::expand(args, item)
        .unwrap_or_else(syn::Error::into_compile_error)
        .into()
}

/// `hex!("deadBEEF")` expands to the byte array `[0xde, 0xad, 0xbe, 0xef]`,
/// checked at compile time.
#[proc_macro]
pub fn hex(input: TokenStream) -> TokenStream {
    let lit = syn::parse_macro_input!(input as syn::LitStr);
    hex::expand(&lit)
        .unwrap_or_else(syn::Error::into_compile_error)
        .into()
}
```

`proclab-macros/src/field_names.rs`:

```rust
use proc_macro2::TokenStream;
use quote::quote;
use syn::{Data, DeriveInput, Fields, LitStr};

/// Parsed `#[names(...)]` options of one field.
#[derive(Default)]
struct FieldOpts {
    skip: bool,
    rename: Option<LitStr>,
}

fn parse_opts(field: &syn::Field) -> syn::Result<FieldOpts> {
    let mut opts = FieldOpts::default();
    for attr in &field.attrs {
        if !attr.path().is_ident("names") {
            continue;
        }
        attr.parse_nested_meta(|meta| {
            if meta.path.is_ident("skip") {
                opts.skip = true;
                Ok(())
            } else if meta.path.is_ident("rename") {
                opts.rename = Some(meta.value()?.parse()?);
                Ok(())
            } else {
                Err(meta.error("unsupported `names` option, expected `skip` or `rename = \"..\"`"))
            }
        })?;
    }
    if opts.skip && opts.rename.is_some() {
        return Err(syn::Error::new_spanned(
            field,
            "`skip` and `rename` cannot be combined",
        ));
    }
    Ok(opts)
}

pub fn expand(input: &DeriveInput) -> syn::Result<TokenStream> {
    let fields = match &input.data {
        Data::Struct(s) => match &s.fields {
            Fields::Named(named) => &named.named,
            _ => {
                return Err(syn::Error::new_spanned(
                    &input.ident,
                    "FieldNames needs a struct with named fields",
                ));
            }
        },
        _ => {
            return Err(syn::Error::new_spanned(
                &input.ident,
                "FieldNames can only be derived for structs",
            ));
        }
    };

    let mut names: Vec<LitStr> = Vec::new();
    for field in fields {
        let opts = parse_opts(field)?;
        if opts.skip {
            continue;
        }
        let ident = field.ident.as_ref().expect("named field");
        let lit = opts
            .rename
            .unwrap_or_else(|| LitStr::new(&ident.to_string(), ident.span()));
        if let Some(prev) = names.iter().find(|n| n.value() == lit.value()) {
            let mut err = syn::Error::new(lit.span(), format!("duplicate name `{}`", lit.value()));
            err.combine(syn::Error::new(prev.span(), "first used here"));
            return Err(err);
        }
        names.push(lit);
    }

    let name = &input.ident;
    let (impl_generics, ty_generics, where_clause) = input.generics.split_for_impl();
    Ok(quote! {
        impl #impl_generics ::proclab::FieldNames for #name #ty_generics #where_clause {
            const FIELD_NAMES: &'static [&'static str] = &[ #(#names),* ];
        }
    })
}

#[cfg(test)]
mod tests {
    use super::*;
    use syn::parse_quote;

    fn out(input: DeriveInput) -> String {
        expand(&input).unwrap().to_string()
    }

    fn err(input: DeriveInput) -> String {
        expand(&input).unwrap_err().to_string()
    }

    #[test]
    fn lists_fields_in_order_and_keeps_generics() {
        let s = out(parse_quote! {
            struct P<T: Clone> where T: Send { x: T, y: u8 }
        });
        assert!(s.contains("for P < T >"), "{s}");
        assert!(s.contains("where T : Send"), "{s}");
        assert!(s.contains("[\"x\" , \"y\"]"), "{s}");
    }

    #[test]
    fn skip_and_rename() {
        let s = out(parse_quote! {
            struct P {
                #[names(rename = "ex")] x: u8,
                #[names(skip)] secret: u8,
                y: u8,
            }
        });
        assert!(s.contains("[\"ex\" , \"y\"]"), "{s}");
        assert!(!s.contains("secret"), "{s}");
    }

    #[test]
    fn rejects_bad_input_with_messages() {
        assert!(err(parse_quote! { struct T(u8); }).contains("named fields"));
        assert!(err(parse_quote! { enum E { A } }).contains("only be derived for structs"));
        assert!(err(parse_quote! { struct P { #[names(oops)] x: u8 } }).contains("unsupported"));
        assert!(
            err(parse_quote! { struct P { #[names(skip, rename = "a")] x: u8 } })
                .contains("cannot be combined")
        );
        assert!(
            err(parse_quote! { struct P { x: u8, #[names(rename = "x")] y: u8 } })
                .contains("duplicate name `x`")
        );
    }
}
```

`proclab-macros/src/enum_str.rs`:

```rust
use proc_macro2::TokenStream;
use quote::quote;
use syn::{Data, DeriveInput, Fields, LitStr};

#[derive(Clone, Copy)]
enum RenameAll {
    None,
    Lowercase,
    SnakeCase,
}

fn parse_rename_all(input: &DeriveInput) -> syn::Result<RenameAll> {
    let mut rule = RenameAll::None;
    for attr in &input.attrs {
        if !attr.path().is_ident("enum_str") {
            continue;
        }
        attr.parse_nested_meta(|meta| {
            if meta.path.is_ident("rename_all") {
                let lit: LitStr = meta.value()?.parse()?;
                rule = match lit.value().as_str() {
                    "lowercase" => RenameAll::Lowercase,
                    "snake_case" => RenameAll::SnakeCase,
                    _ => {
                        return Err(syn::Error::new(
                            lit.span(),
                            "expected \"lowercase\" or \"snake_case\"",
                        ));
                    }
                };
                Ok(())
            } else {
                Err(meta.error("unsupported option on the enum, expected `rename_all = \"..\"`"))
            }
        })?;
    }
    Ok(rule)
}

fn variant_rename(variant: &syn::Variant) -> syn::Result<Option<LitStr>> {
    let mut rename = None;
    for attr in &variant.attrs {
        if !attr.path().is_ident("enum_str") {
            continue;
        }
        attr.parse_nested_meta(|meta| {
            if meta.path.is_ident("rename") {
                rename = Some(meta.value()?.parse()?);
                Ok(())
            } else {
                Err(meta.error("unsupported option on a variant, expected `rename = \"..\"`"))
            }
        })?;
    }
    Ok(rename)
}

fn snake_case(s: &str) -> String {
    let mut out = String::new();
    for (i, ch) in s.chars().enumerate() {
        if ch.is_uppercase() {
            if i != 0 {
                out.push('_');
            }
            out.extend(ch.to_lowercase());
        } else {
            out.push(ch);
        }
    }
    out
}

fn apply(rule: RenameAll, ident: &str) -> String {
    match rule {
        RenameAll::None => ident.to_owned(),
        RenameAll::Lowercase => ident.to_lowercase(),
        RenameAll::SnakeCase => snake_case(ident),
    }
}

pub fn expand(input: &DeriveInput) -> syn::Result<TokenStream> {
    let Data::Enum(data) = &input.data else {
        return Err(syn::Error::new_spanned(
            &input.ident,
            "EnumStr can only be derived for enums",
        ));
    };
    if !input.generics.params.is_empty() {
        return Err(syn::Error::new_spanned(
            &input.generics,
            "EnumStr does not support generic enums",
        ));
    }
    if data.variants.is_empty() {
        return Err(syn::Error::new_spanned(
            &input.ident,
            "EnumStr needs at least one variant",
        ));
    }
    let rule = parse_rename_all(input)?;

    let mut idents = Vec::new();
    let mut texts: Vec<LitStr> = Vec::new();
    for variant in &data.variants {
        if !matches!(variant.fields, Fields::Unit) {
            return Err(syn::Error::new_spanned(
                variant,
                "EnumStr only supports unit variants (no fields)",
            ));
        }
        let text = match variant_rename(variant)? {
            Some(lit) => lit,
            None => LitStr::new(
                &apply(rule, &variant.ident.to_string()),
                variant.ident.span(),
            ),
        };
        if texts.iter().any(|t| t.value() == text.value()) {
            return Err(syn::Error::new(
                text.span(),
                format!("duplicate string `{}`", text.value()),
            ));
        }
        idents.push(&variant.ident);
        texts.push(text);
    }

    let name = &input.ident;
    Ok(quote! {
        impl ::proclab::EnumStr for #name {
            const ALL: &'static [Self] = &[ #(Self::#idents),* ];

            fn as_str(&self) -> &'static str {
                match self { #(Self::#idents => #texts),* }
            }
        }

        impl ::core::fmt::Display for #name {
            fn fmt(&self, f: &mut ::core::fmt::Formatter<'_>) -> ::core::fmt::Result {
                f.write_str(::proclab::EnumStr::as_str(self))
            }
        }

        impl ::core::str::FromStr for #name {
            type Err = ::proclab::UnknownVariant;
            fn from_str(s: &str) -> ::core::result::Result<Self, Self::Err> {
                match s {
                    #(#texts => ::core::result::Result::Ok(Self::#idents),)*
                    other => ::core::result::Result::Err(::proclab::UnknownVariant::new(
                        ::core::stringify!(#name),
                        other,
                    )),
                }
            }
        }
    })
}

#[cfg(test)]
mod tests {
    use super::*;
    use syn::parse_quote;

    #[test]
    fn snake_case_examples() {
        assert_eq!(snake_case("HttpServer"), "http_server");
        assert_eq!(snake_case("A"), "a");
        assert_eq!(snake_case("already_snake"), "already_snake");
        assert_eq!(snake_case("ABc"), "a_bc");
    }

    #[test]
    fn rename_precedence_and_errors() {
        let ok: DeriveInput = parse_quote! {
            #[enum_str(rename_all = "snake_case")]
            enum E { FooBar, #[enum_str(rename = "x")] Baz }
        };
        let s = expand(&ok).unwrap().to_string();
        assert!(s.contains("\"foo_bar\""), "{s}");
        assert!(s.contains("\"x\""), "{s}");
        let e = |i: DeriveInput| expand(&i).unwrap_err().to_string();
        assert!(e(parse_quote! { enum E { A(u8) } }).contains("unit variants"));
        assert!(e(parse_quote! { struct S; }).contains("only be derived for enums"));
        assert!(e(parse_quote! { enum E {} }).contains("at least one variant"));
        assert!(e(parse_quote! { enum E<T> { A } }).contains("generic"));
        assert!(
            e(parse_quote! { #[enum_str(rename_all = "kebab")] enum E { A } })
                .contains("expected \"lowercase\"")
        );
        assert!(
            e(parse_quote! { enum E { A, #[enum_str(rename = "A")] B } }).contains("duplicate")
        );
    }
}
```

`proclab-macros/src/hex.rs`:

```rust
use proc_macro2::TokenStream;
use quote::quote;
use syn::LitStr;

fn nibble(b: u8) -> Option<u8> {
    match b {
        b'0'..=b'9' => Some(b - b'0'),
        b'a'..=b'f' => Some(b - b'a' + 10),
        b'A'..=b'F' => Some(b - b'A' + 10),
        _ => None,
    }
}

/// Decodes a hex string, or explains what is wrong with it.
pub fn decode(s: &str) -> Result<Vec<u8>, String> {
    let bytes = s.as_bytes();
    if !bytes.len().is_multiple_of(2) {
        return Err(format!("odd number of hex digits ({})", bytes.len()));
    }
    let mut out = Vec::with_capacity(bytes.len() / 2);
    for (i, pair) in bytes.chunks(2).enumerate() {
        match (nibble(pair[0]), nibble(pair[1])) {
            (Some(h), Some(l)) => out.push(h << 4 | l),
            _ => return Err(format!("invalid hex digit in byte {i}")),
        }
    }
    Ok(out)
}

pub fn expand(lit: &LitStr) -> syn::Result<TokenStream> {
    let bytes = decode(&lit.value()).map_err(|msg| syn::Error::new(lit.span(), msg))?;
    Ok(quote! { [ #(#bytes),* ] })
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn decodes_mixed_case_and_empty() {
        assert_eq!(decode("deadBEEF").unwrap(), [0xde, 0xad, 0xbe, 0xef]);
        assert_eq!(decode("").unwrap(), Vec::<u8>::new());
        assert_eq!(decode("00ff").unwrap(), [0, 255]);
    }

    #[test]
    fn rejects_odd_and_invalid() {
        assert!(decode("abc").unwrap_err().contains("odd"));
        assert!(decode("zz").unwrap_err().contains("byte 0"));
        assert!(decode("00g0").unwrap_err().contains("byte 1"));
        assert!(decode("é0").is_err(), "non-ASCII must not panic");
    }
}
```

`proclab-macros/src/traced.rs`:

```rust
use proc_macro2::{Span, TokenStream};
use quote::quote;
use syn::{Ident, ItemFn, LitStr};

fn parse_label(args: TokenStream, default: &Ident) -> syn::Result<LitStr> {
    if args.is_empty() {
        return Ok(LitStr::new(&default.to_string(), default.span()));
    }
    let mut label = None;
    let parser = syn::meta::parser(|meta| {
        if meta.path.is_ident("name") {
            label = Some(meta.value()?.parse::<LitStr>()?);
            Ok(())
        } else {
            Err(meta.error("unsupported option, expected `name = \"..\"`"))
        }
    });
    syn::parse::Parser::parse2(parser, args)?;
    label.ok_or_else(|| syn::Error::new(Span::call_site(), "expected `name = \"..\"`"))
}

pub fn expand(args: TokenStream, item: TokenStream) -> syn::Result<TokenStream> {
    let mut func: ItemFn = syn::parse2(item)?;
    if let Some(asyncness) = &func.sig.asyncness {
        return Err(syn::Error::new_spanned(
            asyncness,
            "#[traced] does not support async functions",
        ));
    }
    let label = parse_label(args, &func.sig.ident)?;
    // `mixed_site` hygiene: this local cannot collide with a user variable.
    let guard = Ident::new("__proclab_guard", Span::mixed_site());
    // Splice the original statements in (instead of nesting the old block):
    // the tail expression keeps its value, and `unused_braces` stays quiet.
    let stmts = &func.block.stmts;
    *func.block = syn::parse_quote!({
        let #guard = ::proclab::trace::Guard::enter(#label);
        #(#stmts)*
    });
    Ok(quote! { #func })
}

#[cfg(test)]
mod tests {
    use super::*;

    fn run(args: TokenStream, item: TokenStream) -> Result<String, String> {
        expand(args, item)
            .map(|t| t.to_string())
            .map_err(|e| e.to_string())
    }

    #[test]
    fn wraps_the_body_and_keeps_the_signature() {
        let s = run(
            quote!(),
            quote! { pub fn add(a: u32, b: u32) -> u32 { a + b } },
        )
        .unwrap();
        assert!(s.contains("pub fn add (a : u32 , b : u32) -> u32"), "{s}");
        assert!(s.contains("Guard :: enter (\"add\")"), "{s}");
        assert!(s.contains("a + b"), "{s}");
    }

    #[test]
    fn custom_name_and_errors() {
        let s = run(quote!(name = "plus"), quote! { fn f() {} }).unwrap();
        assert!(s.contains("\"plus\""), "{s}");
        assert!(
            run(quote!(nom = "x"), quote! { fn f() {} })
                .unwrap_err()
                .contains("unsupported")
        );
        assert!(
            run(quote!(), quote! { async fn f() {} })
                .unwrap_err()
                .contains("async")
        );
        assert!(run(quote!(), quote! { struct S; }).is_err());
    }
}
```

`src/lib.rs` (the runtime crate):

```rust
//! `proclab`: a small library that shows how procedural macros are built,
//! tested and documented.
//!
//! * `#[derive(FieldNames)]`: the field names of a struct as a constant.
//! * `#[derive(EnumStr)]`: `as_str`, `Display` and `FromStr` for fieldless enums.
//! * `#[traced]`: record entry and exit of a function in a thread-local log.
//! * `hex!("..")`: a byte array decoded at compile time.
//!
//! ```
//! use proclab::{hex, EnumStr, FieldNames};
//!
//! #[derive(FieldNames)]
//! struct Point { x: i32, y: i32, #[names(skip)] cache: u8 }
//! assert_eq!(Point::FIELD_NAMES, ["x", "y"]);
//!
//! #[derive(Debug, PartialEq, EnumStr)]
//! #[enum_str(rename_all = "snake_case")]
//! enum Mode { FastPath, SlowPath }
//! assert_eq!("fast_path".parse::<Mode>(), Ok(Mode::FastPath));
//! assert_eq!(Mode::SlowPath.to_string(), "slow_path");
//!
//! assert_eq!(hex!("cafe"), [0xca, 0xfe]);
//! ```

#![deny(missing_docs)]

// The derives refer to `::proclab::...`; this makes that path valid inside
// this crate's own tests and doctests as well.
extern crate self as proclab;

use std::fmt;

pub use proclab_macros::{EnumStr, FieldNames, hex, traced};

/// Implemented by `#[derive(FieldNames)]`.
pub trait FieldNames {
    /// The names of the (non-skipped) fields, in declaration order.
    const FIELD_NAMES: &'static [&'static str];
}

/// Implemented by `#[derive(EnumStr)]`, together with `Display` and `FromStr`.
pub trait EnumStr: Sized + 'static {
    /// Every variant, in declaration order.
    const ALL: &'static [Self];
    /// The text of this variant.
    fn as_str(&self) -> &'static str;
}

/// The error of the `FromStr` impl generated by `#[derive(EnumStr)]`.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct UnknownVariant {
    type_name: &'static str,
    input: String,
}

impl UnknownVariant {
    /// Used by generated code.
    #[doc(hidden)]
    pub fn new(type_name: &'static str, input: &str) -> Self {
        UnknownVariant {
            type_name,
            input: input.to_owned(),
        }
    }

    /// The name of the enum that was being parsed.
    pub fn type_name(&self) -> &'static str {
        self.type_name
    }

    /// The rejected input.
    pub fn input(&self) -> &str {
        &self.input
    }
}

impl fmt::Display for UnknownVariant {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "unknown {}: {:?}", self.type_name, self.input)
    }
}

impl std::error::Error for UnknownVariant {}

/// The thread-local event log written by `#[traced]` functions.
pub mod trace {
    use std::cell::RefCell;

    thread_local! {
        static LOG: RefCell<Vec<String>> = const { RefCell::new(Vec::new()) };
    }

    /// Records `enter <label>` now and `leave <label>` when dropped, so the
    /// exit is recorded on early `return`, `?` and unwinding too.
    #[must_use = "the guard records the exit when it is dropped"]
    pub struct Guard {
        label: &'static str,
    }

    impl Guard {
        /// Starts a traced scope.
        pub fn enter(label: &'static str) -> Guard {
            LOG.with_borrow_mut(|l| l.push(format!("enter {label}")));
            Guard { label }
        }
    }

    impl Drop for Guard {
        fn drop(&mut self) {
            let label = self.label;
            LOG.with_borrow_mut(|l| l.push(format!("leave {label}")));
        }
    }

    /// Removes and returns this thread's log.
    pub fn take_log() -> Vec<String> {
        LOG.with_borrow_mut(std::mem::take)
    }
}
```

`tests/macros.rs`:

```rust
//! Behavior tests for the four macros, written as an ordinary downstream user
//! would use them.

#![deny(unused_braces)]

use proclab::trace::take_log;
use proclab::{EnumStr, FieldNames, UnknownVariant, hex, traced};

#[derive(FieldNames)]
struct Plain {
    #[allow(dead_code)]
    id: u32,
    #[allow(dead_code)]
    name: String,
}

#[derive(FieldNames)]
struct Renamed {
    #[names(rename = "ID")]
    #[allow(dead_code)]
    id: u32,
    #[names(skip)]
    #[allow(dead_code)]
    cache: Vec<u8>,
    #[allow(dead_code)]
    tail: bool,
}

#[derive(FieldNames)]
struct Generic<'a, T: Clone, const N: usize>
where
    T: Send,
{
    #[allow(dead_code)]
    items: [T; N],
    #[allow(dead_code)]
    label: &'a str,
}

#[derive(FieldNames)]
struct Empty {}

#[test]
fn field_names_lists_fields_in_declaration_order() {
    assert_eq!(Plain::FIELD_NAMES, ["id", "name"]);
    assert_eq!(Renamed::FIELD_NAMES, ["ID", "tail"]);
    assert_eq!(Generic::<u8, 3>::FIELD_NAMES, ["items", "label"]);
    assert!(Empty::FIELD_NAMES.is_empty());
}

#[test]
fn field_names_is_a_const() {
    const N: usize = Plain::FIELD_NAMES.len();
    let arr = [0u8; N];
    assert_eq!(arr.len(), 2);
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, EnumStr)]
enum Plainest {
    Alpha,
    Beta,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, EnumStr)]
#[enum_str(rename_all = "snake_case")]
enum Snake {
    HttpServer,
    #[enum_str(rename = "dns")]
    DomainNameSystem,
    Tcp,
}

#[derive(Debug, PartialEq, EnumStr)]
#[enum_str(rename_all = "lowercase")]
pub enum Loud {
    ALPHA,
    Beta,
}

#[test]
fn enum_str_round_trips_every_variant() {
    for v in Plainest::ALL {
        assert_eq!(v.to_string().parse::<Plainest>(), Ok(*v));
    }
    for v in Snake::ALL {
        assert_eq!(v.as_str().parse::<Snake>(), Ok(*v));
    }
    assert_eq!(Plainest::ALL.len(), 2);
    assert_eq!(
        Snake::ALL,
        [Snake::HttpServer, Snake::DomainNameSystem, Snake::Tcp]
    );
}

#[test]
fn enum_str_names_follow_the_rename_rules() {
    assert_eq!(Plainest::Alpha.as_str(), "Alpha");
    assert_eq!(Snake::HttpServer.as_str(), "http_server");
    assert_eq!(
        Snake::DomainNameSystem.as_str(),
        "dns",
        "per-variant rename wins"
    );
    assert_eq!(Loud::ALPHA.to_string(), "alpha");
    assert_eq!("beta".parse::<Loud>(), Ok(Loud::Beta));
}

#[test]
fn enum_str_reports_unknown_input() {
    let err: UnknownVariant = "nope".parse::<Snake>().unwrap_err();
    assert_eq!(err.type_name(), "Snake");
    assert_eq!(err.input(), "nope");
    assert_eq!(err.to_string(), "unknown Snake: \"nope\"");
    assert!("".parse::<Snake>().is_err());
    assert!(
        "Tcp".parse::<Snake>().is_err(),
        "matching is case-sensitive"
    );
}

#[test]
fn hex_decodes_at_compile_time() {
    const KEY: [u8; 4] = hex!("DEADbeef");
    assert_eq!(KEY, [0xde, 0xad, 0xbe, 0xef]);
    let empty: [u8; 0] = hex!("");
    assert!(empty.is_empty());
    assert_eq!(hex!("00ff").len(), 2);
}

#[traced]
fn add(a: u32, b: u32) -> u32 {
    a + b
}

#[traced(name = "custom")]
fn early(flag: bool) -> &'static str {
    if flag {
        return "early";
    }
    "late"
}

#[traced]
fn fails(n: i32) -> Result<i32, String> {
    if n < 0 {
        Err("negative")?;
    }
    Ok(n)
}

#[traced]
fn nested() -> u32 {
    add(1, 2) + add(3, 4)
}

#[traced]
fn panics() {
    panic!("boom");
}

struct Counter(u32);
impl Counter {
    #[traced(name = "Counter::bump")]
    fn bump(&mut self) -> u32 {
        self.0 += 1;
        self.0
    }
}

#[test]
fn traced_keeps_behavior_and_logs_enter_and_leave() {
    take_log();
    assert_eq!(add(2, 3), 5);
    assert_eq!(take_log(), ["enter add", "leave add"]);
}

#[test]
fn traced_logs_leave_on_early_return_and_question_mark() {
    take_log();
    assert_eq!(early(true), "early");
    assert_eq!(early(false), "late");
    assert_eq!(fails(-1), Err("negative".to_string()));
    assert_eq!(fails(3), Ok(3));
    assert_eq!(
        take_log(),
        [
            "enter custom",
            "leave custom",
            "enter custom",
            "leave custom",
            "enter fails",
            "leave fails",
            "enter fails",
            "leave fails"
        ]
    );
}

#[test]
fn traced_nests_and_works_on_methods() {
    take_log();
    assert_eq!(nested(), 10);
    assert_eq!(
        take_log(),
        [
            "enter nested",
            "enter add",
            "leave add",
            "enter add",
            "leave add",
            "leave nested"
        ]
    );
    let mut c = Counter(0);
    assert_eq!(c.bump(), 1);
    assert_eq!(take_log(), ["enter Counter::bump", "leave Counter::bump"]);
}

#[test]
fn traced_logs_leave_while_unwinding() {
    take_log();
    let r = std::panic::catch_unwind(panics);
    assert!(r.is_err());
    assert_eq!(take_log(), ["enter panics", "leave panics"]);
}

#[traced]
fn shadow(__proclab_guard: u32) -> u32 {
    __proclab_guard + 1
}

#[test]
fn a_user_variable_named_like_the_guard_does_not_collide() {
    take_log();
    assert_eq!(shadow(41), 42);
    assert_eq!(take_log(), ["enter shadow", "leave shadow"]);
}

// A one-line body used to trigger `unused_braces` in the *user's* code.
#[rustfmt::skip]
#[traced]
fn one_liner(a: u32) -> u32 { a + 1 }

#[test]
fn single_line_bodies_do_not_warn_and_keep_their_value() {
    take_log();
    assert_eq!(one_liner(1), 2);
    assert_eq!(take_log(), ["enter one_liner", "leave one_liner"]);
}
```

`tests/ui.rs`:

```rust
//! Compile-fail tests: each file in `tests/ui` must fail to compile with the
//! message recorded in the matching `.stderr` file.

#[test]
fn ui() {
    let t = trybuild::TestCases::new();
    t.compile_fail("tests/ui/*.rs");
}
```

The nine UI cases live in `tests/ui/*.rs`, each a few lines, with the compiler output saved beside it as `*.stderr` by `TRYBUILD=overwrite cargo test --test ui` and then read by hand: `field_names_on_tuple`, `field_names_on_enum`, `field_names_bad_option`, `field_names_duplicate`, `enum_str_with_fields`, `hex_odd`, `hex_invalid`, `traced_async`, `traced_on_struct`. Example (`tests/ui/field_names_bad_option.rs` and its snapshot):

```rust
use proclab::FieldNames;

#[derive(FieldNames)]
struct S {
    #[names(oops)]
    x: u8,
}

fn main() {}
```

```text
error: unsupported `names` option, expected `skip` or `rename = ".."`
 --> tests/ui/field_names_bad_option.rs:5:13
  |
5 |     #[names(oops)]
  |             ^^^^
```

**Verification ✓.** `cargo fmt --all --check`, `cargo clippy --workspace --all-targets --locked -- -D warnings`, `cargo test --workspace --locked` and `RUSTDOCFLAGS="-D warnings" cargo doc --workspace --no-deps` pass: 9 unit tests in the macro crate, 12 behavior tests, the `trybuild` test (9 UI cases) and 1 doctest.

**Mutation checks ✓** (each mutation applied to a copy):

| # | Mutation | Result |
|---|---|---|
| M1 | ignore `#[names(skip)]` | **caught** by `field_names_lists_fields_in_declaration_order` |
| M2 | drop generics from the `FieldNames` impl | **caught**: the test crate fails to compile (`E0107`, `E0726`) |
| M3 | disable the duplicate-name check | **caught** by the `trybuild` test (the expected error disappears; the unit test would catch it as well, but the run stopped at the first failing test binary) |
| M4 | ignore `rename` | **caught** by `field_names_lists_fields_in_declaration_order` |
| M5 | `snake_case` without underscores | **caught** by `enum_str_names_follow_the_rename_rules` |
| M6 | per-variant `rename` ignored | **caught** by the same test |
| M7 | `ALL` lists only the first variant | **caught** by `enum_str_round_trips_every_variant` |
| M8 | `UnknownVariant` gets a constant type name | **caught** by `enum_str_reports_unknown_input` |
| M9 | hex nibbles swapped | **caught** by `hex_decodes_at_compile_time` |
| M10 | uppercase hex digits rejected | **caught**, but only because the test crate (which calls `hex!("DEADbeef")`) stops compiling |
| M11 | `traced` creates no guard (`let _ = ...` drops it immediately) | **caught** by `traced_nests_and_works_on_methods` |
| M12 | guard identifier uses `call_site` instead of `mixed_site` | **caught** by the `shadow` test, which fails to compile (`E0369`) |
| M13 | `async fn` accepted | **caught** by the `trybuild` test |
| M14 | wrap the spliced statements in a nested `{ ... }` | **not caught, and equivalent for the lint**: braces written by the macro itself carry macro spans, and `unused_braces` stays quiet |
| M14b | re-introduce the original bug (interpolate the user's block) | **caught** by `single_line_bodies_do_not_warn_and_keep_their_value` (compile error under `#![deny(unused_braces)]`) after I added `#[rustfmt::skip]`; **not caught** by my first version of that test, because `rustfmt` had reflowed the one-line function |

Two of my first mutants were invalid: my first M7 (`ALL` sliced with `[..0]` in a `const`) was rejected by the compiler (`Index is not yet stable as a const trait`), which proves nothing about the tests, so I replaced it with the first-variant-only mutant above. The surviving-mutant story for M14b is real and is why §9 warns about formatting. A tooling note: an early clippy run flagged my own macro (`len() % 2 != 0` should use `is_multiple_of`) and I changed the code.

**Design notes.**

- Each macro is `fn(TokenStream) -> TokenStream` wrapping `expand(...) -> syn::Result<TokenStream2>`; only the wrappers touch the compiler's `proc_macro` API, so everything else is testable.
- Absolute paths everywhere in generated code; `extern crate self as proclab;` in the runtime crate makes the same expansions work in its own tests and doctests.
- `UnknownVariant::new` is `pub` but `#[doc(hidden)]`: generated code in other crates must be able to call it, and as Lesson 33 stressed, hidden is not private. It is therefore part of the semver surface the macro and runtime crate must keep in sync.
- `hex!` and `FIELD_NAMES`/`ALL` are `const`-friendly, so they work in constants and array lengths (tested).
- Not handled: `async fn` support in `traced`, enums/unions in `FieldNames`, enum variants with fields in `EnumStr`, `serde`-style `rename_all` rules beyond two, `#[names(...)]` on tuple fields, `no_std` (the log uses `std`), proper bounds on generic `FieldNames` impls (it adds none), and macro-hygiene across the `proclab` / `proclab-macros` version-skew. I did not fuzz the macro inputs or benchmark compile times beyond the single clean build above, and I did not publish or version the crates.

</details>

---

## 16. Key takeaways

- A proc macro is Rust code the compiler runs at build time: tokens in, tokens out. Three kinds: derive (adds items), attribute (replaces an item), function-like (an expression/items). It must live in a `proc-macro = true` crate that exports only macros (✓), so libraries ship a runtime crate plus a macro crate.
- Use `syn` to parse, `quote` to generate, `proc-macro2` so the logic is testable. Keep each macro a thin wrapper around `expand(...) -> syn::Result<TokenStream2>`.
- Report errors with `syn::Error` at the right span, never `panic!` (✓ "proc macro panicked", ✓ spanned UI snapshots).
- Generate absolute paths, keep the user's generics (✓ `E0107`/`E0726` otherwise), and use `mixed_site` for generated locals (✓ `E0369` with `call_site`).
- Put cleanup in `Drop` guards so it runs on early return, `?` and panics (✓).
- Tokens copied from the user keep their spans and lints: don't nest the original block (✓ `unused_braces`).
- Test three ways: unit tests of `expand`, behavior tests, and `trybuild` UI tests; beware formatting-sensitive tests and surviving mutants (✓ M14b).
- A proc macro is arbitrary code at build time (✓ `noisy!`): audit these dependencies. Rust does not make that safe.

**Connections.** [Lesson 34](34_declarative_macros.md) is the declarative counterpart and explains hygiene and tokens; [Lesson 33](33_modules_api_design_rustdoc.md)'s facade crates, `#[doc(hidden)]` and semver rules apply to the runtime/macro pair; [Lesson 26](26_error_design.md)'s error design shows up in `UnknownVariant`; [Lesson 19](19_generics_and_bounds.md)'s generics are what `split_for_impl` has to preserve; [Lesson 32](32_cancellation_streams_backpressure.md)'s cancellation rules explain why `#[traced]` rejects `async fn`. Next: Lesson 36 begins the unsafe-Rust module with raw pointers and undefined behavior.

---

## 17. References

- The Rust Reference, "Procedural Macros" (proc-macro crates, `proc_macro`, derive helper attributes, hygiene and spans): <https://doc.rust-lang.org/reference/procedural-macros.html>
- The `syn` crate documentation (`DeriveInput`, `ItemFn`, `Attribute::parse_nested_meta`, `parse_macro_input!`, `Error`): <https://docs.rs/syn>
- The `quote` crate documentation (`quote!`, interpolation, repetition, `ToTokens`): <https://docs.rs/quote>
- The `proc-macro2` crate documentation: <https://docs.rs/proc-macro2>
- The `trybuild` crate documentation (UI tests, `TRYBUILD=overwrite`): <https://docs.rs/trybuild>
- David Tolnay's `proc-macro-workshop` (exercises: derive builder, debug, seq, sorted, bitfield): <https://github.com/dtolnay/proc-macro-workshop>
- `cargo expand` (external tool): <https://github.com/dtolnay/cargo-expand>
- The Rust Book, "Macros" chapter (procedural macros section): <https://doc.rust-lang.org/book/>

The URLs above are the standard documentation locations; I did not open them during this session (the sandbox cannot reach the documentation sites, though it can download crates from crates.io), so if a link has moved, search for the page title. Unchecked or from memory: the exact `syn`/`quote` API signatures beyond what compiled here (for example `type_params_mut`, `make_where_clause`, `Error::combine` semantics), the `derive` vs `full` feature split in `syn`, that `Span::def_site` is nightly-only, the precise conditions of the `unused_braces` lint (I only observed one-line vs multi-line bodies), that `trybuild` snapshots depend on the compiler version, `serde`/`serde_derive` layout and re-export details, that `rust-analyzer` runs proc macros, `cargo vet`/`cargo deny` usage, WebAssembly-sandboxed macros, and publishing practices for version-pinning a macro crate. Exercise 7's snippet was not compiled. Everything marked ✓ was run on this machine's `rustc 1.97.0` with `syn` 2.0.119, `quote` 1.0.47, `proc-macro2` 1.0.107 and `trybuild` 1.0.122.
