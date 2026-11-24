---
title: "Lesson 14 — Option, Result, the ? operator, and error propagation"
---

# Lesson 14 — `Option`, `Result`, the `?` operator, and error propagation

**Module 5 · Data modeling** · Status: generated · Edition used in examples: **2024** · External dependencies: none

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux. As before, ✓ marks outputs and diagnostics I actually ran; unmarked statements are documented behavior I did not run. Everything is **stable Rust**.

Builds on [Lesson 13](13_structs_enums_methods.md) (enums, `Display`, validating constructors), [Lesson 06](06_strings_conversions_patterns.md) (`parse`, `FromStr`, `let else`), [Lesson 08](08_stack_heap_drop.md) (unwinding, panics), and [Lesson 09](09_memory_layout.md) (niches, `Option` sizes).

---

## 1. Errors are values

Rust has no exceptions. A function that can fail says so **in its return type**, and the caller must deal with it: the compiler will not let you use a `Result` as if it were the success value. Two standard enums carry almost all of this:

```rust
enum Option<T> { None, Some(T) }          // "maybe a value"
enum Result<T, E> { Ok(T), Err(E) }       // "a value, or why not"
```

**Why this design.** Languages with null references and unchecked exceptions let a failure travel invisibly through code until something far away crashes. Rust makes each possible absence or failure a *value* with a type: no null (the only way to have "no value" is `Option::None`, and you must handle it before touching the inner value), and no hidden control flow (every place that can fail is marked in the types and, with `?`, at the call). The cost is some syntax at each call site; the `?` operator (Section 4) removes most of it.

**The rule of thumb for this lesson:**

- *Expected* absence or failure (user input, files, network, parsing, "not found") → `Option` / `Result`.
- *Bugs* (violated invariants, impossible states) → panic (Section 8).

---

## 2. `Option<T>`

### 2.1 Combinators

You can always `match` on an `Option`, but the standard library has methods for the common shapes. A quick tour ✓:

```rust
fn main() {
    let words = vec!["apple", "", "kiwi"];
    let first_len = words.first().map(|w| w.len());
    let non_empty = words.iter().find(|w| !w.is_empty()).copied();
    println!("{first_len:?} {non_empty:?}");
    let x: Option<i32> = Some(4);
    println!("{:?} {:?} {:?}", x.map(|v| v * 2), x.and_then(|v| if v > 5 { Some(v) } else { None }), x.filter(|v| v % 2 == 0));
    println!("{} {} {}", x.unwrap_or(0), None::<i32>.unwrap_or_default(), None::<i32>.unwrap_or_else(|| 99));
    println!("{:?} {:?}", x.zip(Some('a')), x.xor(None));
    println!("{:?} {:?}", x.ok_or("missing"), None::<i32>.ok_or("missing"));
    let mut slot = Some(3);
    let old = slot.take();
    println!("{slot:?} {old:?}");
    let mut s2 = Some(1);
    let prev = s2.replace(2);
    println!("{s2:?} {prev:?}");
    let mut cache: Option<String> = None;
    cache.get_or_insert_with(|| "computed".to_string()).push('!');
    println!("{cache:?}");
    let name: Option<String> = Some("ana".into());
    let r: Option<&str> = name.as_deref();
    println!("{r:?} {:?}", name.as_ref().map(|s| s.len()));
    let nested: Option<Option<i32>> = Some(Some(1));
    println!("{:?}", nested.flatten());
}
```

✓ Output:

```text
Some(5) Some("apple")
Some(8) None Some(4)
4 0 99
Some((4, 'a')) Some(4)
Ok(4) Err("missing")
None Some(3)
Some(2) Some(1)
Some("computed!")
Some("ana") Some(3)
Some(1)
```

| Method | What it does |
|---|---|
| `map(f)` | transform the inner value if `Some`; `None` stays `None` |
| `and_then(f)` | like `map` but `f` itself returns an `Option` (flat-map, avoids `Option<Option<_>>`) |
| `filter(p)` | keep the value only if the predicate holds |
| `unwrap_or(v)`, `unwrap_or_default()`, `unwrap_or_else(f)` | supply a fallback |
| `ok_or(e)`, `ok_or_else(f)` | turn `None` into an `Err(e)`: `Option` → `Result` |
| `take()`, `replace(v)` | move the value out, leaving `None` / a new value (Lesson 12's tool for linked structures) |
| `as_ref()`, `as_mut()`, `as_deref()` | look *into* the option without consuming it (`&Option<String>` → `Option<&str>`) |
| `copied()`, `cloned()` | `Option<&T>` → `Option<T>` |
| `zip`, `xor`, `flatten` | combine / flatten |

Use `match`, `if let`, or `let else` when the logic is more than a one-liner; use combinators when they read as a sentence. Clippy suggests `map_or`/`is_some_and` rewrites; follow it when the result is clearer, not always. ✓ For example, a `match o { Some(s) => s.len(), None => 0 }` and `o.as_ref().map_or(0, |s| s.len())` both gave `3` and `0`.

### 2.2 `?` on `Option`

In a function that returns `Option`, `expr?` yields the inner value or **returns `None` immediately**:

```rust
fn first_char_upper(s: &str) -> Option<char> {
    let c = s.chars().next()?;          // returns None early
    c.to_uppercase().next()
}
fn middle(v: &[i32]) -> Option<i32> { let n = v.len(); let m = v.get(n / 2)?; Some(*m) }
fn main() { println!("{:?} {:?} {:?} {:?}", first_char_upper("hi"), first_char_upper(""), middle(&[1, 2, 3]), middle(&[])); }
```

✓ Output `Some('H') None Some(2) None`. The same pattern with `let else` (Lesson 06) is useful when you want a custom failure value:

```rust
fn port(s: Option<&str>) -> Result<u16, String> {
    let Some(raw) = s else { return Err("no port given".into()) };
    let Ok(p) = raw.parse::<u16>() else { return Err(format!("bad port {raw:?}")) };
    Ok(p)
}
fn main() { println!("{:?} {:?} {:?}", port(Some("80")), port(None), port(Some("99999"))); }
```

✓ Output `Ok(80) Err("no port given") Err("bad port \"99999\"")`.

### 2.3 Eager vs lazy arguments

Methods whose argument is a plain value (`ok_or(x)`, `unwrap_or(x)`, `map_or(x, f)`) evaluate `x` **before** the call, even when it isn't used. The `_else` forms take a closure and run it only if needed:

```rust
fn expensive(tag: &str) -> String { println!("computing {tag}"); tag.to_string() }
fn main() {
    let some: Option<i32> = Some(1);
    let _ = some.ok_or(expensive("eager"));
    let _ = some.ok_or_else(|| expensive("lazy"));
    let _ = some.map_or(expensive("eager2").len() as i32, |v| v);
    let _ = some.unwrap_or(expensive("eager3").len() as i32);
    let _ = some.unwrap_or_else(|| expensive("lazy3").len() as i32);
}
```

✓ Output: `computing eager`, `computing eager2`, `computing eager3`. The lazy forms printed nothing, because the value was `Some`. For cheap constants (`unwrap_or(0)`) eagerness is fine; for allocation, I/O or anything with side effects use the `_else` form. (Clippy's `or_fun_call` lint, from the `nursery`/`pedantic` groups as far as I remember, can flag this; I did not run it.)

---

## 3. `Result<T, E>`

```rust
use std::num::ParseIntError;
fn main() {
    for s in ["42", "-3", "x", "", "99999999999999999999"] {
        let r: Result<i64, ParseIntError> = s.parse();
        match r { Ok(n) => println!("ok {n}"), Err(e) => println!("err {e}") }
    }
}
```

(Each of these inputs is part of the ✓ output of the longer example in Section 4.) Result mirrors Option's combinators: `map`, `and_then`, `unwrap_or*`, plus `map_err` (change the error), `ok()` (drop the error, → `Option`), `err()`, `is_ok_and`, `unwrap_or_default`. The two enums convert into each other: `opt.ok_or(e)` → `Result`, `res.ok()` → `Option` (which **discards** the error; Section 6).

### `#[must_use]`

Ignoring a `Result` is a warning, because it would silently drop a failure:

```rust
fn risky() -> Result<u32, String> { Err("boom".into()) }
fn main() {
    risky();
    let _ = risky();
    println!("done");
}
```

✓ `warning: unused `Result` that must be used`, with `help: use `let _ = ...` to ignore the resulting value`. `let _ = risky();` silences it deliberately (and prints `done`). Treat the warning as an error in CI (`-D warnings`, Lesson 43); ignoring with `let _` should be rare and commented.

---

## 4. The `?` operator

`expr?` on a `Result` means: *if `Ok(v)`, evaluate to `v`; if `Err(e)`, return `Err(From::from(e))` from the current function.* ✓ These two functions behave the same:

```rust
use std::num::ParseIntError;
fn with_q(s: &str) -> Result<i32, ParseIntError> { let n = s.parse::<i32>()?; Ok(n + 1) }
fn manual(s: &str) -> Result<i32, ParseIntError> {
    let n = match s.parse::<i32>() { Ok(v) => v, Err(e) => return Err(From::from(e)) };
    Ok(n + 1)
}
fn main() { println!("{:?} {:?} {:?} {:?}", with_q("1"), manual("1"), with_q("z"), manual("z")); }
```

✓ Output `Ok(2) Ok(2) Err(ParseIntError { kind: InvalidDigit }) Err(ParseIntError { kind: InvalidDigit })`. (The real desugaring goes through the `Try` / `FromResidual` traits, which is why `?` also works on `Option`; the description above is the behavior you need to know.)

### The `From` conversion is the key feature

The error type of the function and the error type of the expression do **not** have to match. They only need `From<InnerError> for OuterError`, and `?` calls it for you:

```rust
use std::fmt;
use std::num::ParseIntError;
#[derive(Debug)]
enum AppError { Parse(ParseIntError), Negative(i64) }
impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self { AppError::Parse(e) => write!(f, "not a number: {e}"), AppError::Negative(n) => write!(f, "negative: {n}") }
    }
}
impl std::error::Error for AppError {}
impl From<ParseIntError> for AppError { fn from(e: ParseIntError) -> Self { AppError::Parse(e) } }
fn parse_positive(s: &str) -> Result<i64, AppError> {
    let n: i64 = s.trim().parse()?;       // ParseIntError -> AppError via From
    if n < 0 { return Err(AppError::Negative(n)); }
    Ok(n)
}
fn main() {
    for s in ["42", " 7 ", "-3", "x", "", "99999999999999999999"] {
        match parse_positive(s) { Ok(n) => println!("ok {n}"), Err(e) => println!("err {e} / {e:?}") }
    }
}
```

✓ Output:

```text
ok 42
ok 7
err negative: -3 / Negative(-3)
err not a number: invalid digit found in string / Parse(ParseIntError { kind: InvalidDigit })
err not a number: cannot parse integer from empty string / Parse(ParseIntError { kind: Empty })
err not a number: number too large to fit in target type / Parse(ParseIntError { kind: PosOverflow })
```

### The three `?` errors you will meet ✓

**1. Using `?` where the function returns `()`:**

```rust
fn main() {
    let n: i32 = "5".parse()?;
    println!("{n}");
}
```

✓ `error[E0277]: the `?` operator can only be used in a function that returns `Result` or `Option` (or another type that implements `FromResidual`)`, with `help: consider adding return type`. Fix: `fn main() -> Result<(), Box<dyn std::error::Error>>` (or handle the error).

**2. Using `?` on an `Option` in a `Result` function (or the reverse):**

```rust
fn f(v: &[i32]) -> Result<i32, String> {
    let x = v.first()?;          // Option in a Result fn
    Ok(*x)
}
fn main() { println!("{:?}", f(&[1])); }
```

✓ `error[E0277]: the `?` operator can only be used on `Result`s, not `Option`s, in a function that returns `Result``. Convert explicitly: `v.first().ok_or("empty")?` (the other direction: `.ok()?`, ✓ `tok.parse::<i32>().ok()?` inside an `Option` function gave `Some(12) None None` for `"12 x"`, `"a"`, `""`). The explicit conversion is a feature: you decide what the missing value means as an error.

**3. No `From` impl for your error type:**

```rust
#[derive(Debug)]
struct MyErr;
fn f(s: &str) -> Result<i32, MyErr> { let n = s.parse::<i32>()?; Ok(n) }
fn main() { println!("{:?}", f("1")); }
```

✓ `error[E0277]: `?` couldn't convert the error to `MyErr``: "the trait `From<ParseIntError>` is not implemented for `MyErr`" and a note "`MyErr` needs to implement `From<ParseIntError>`". (If you write `let n: i32 = s.parse()?` with the type on the binding instead, ✓ the compiler reports a different code: `error[E0271]: type mismatch resolving `<i32 as FromStr>::Err == MyErr``, because inference is steered by the annotation. Annotating the turbofish on `parse` gives the clearer message.)

### `main` can return a `Result`

```rust
use std::num::ParseIntError;
fn main() -> Result<(), ParseIntError> {
    let n: i32 = "12".parse()?;
    println!("{n}");
    let m: i32 = "abc".parse()?;
    println!("{m}");
    Ok(())
}
```

✓ Prints `12`, then `Error: ParseIntError { kind: InvalidDigit }` to stderr, and the **process exit status is 1** ✓. The `Err` is printed with its `Debug` format (so derive `Debug` on error types, and don't put secrets in them). For finer control return `std::process::ExitCode`: ✓ `ExitCode::from(2)` after an `eprintln!` exited with status 2, and `ExitCode::SUCCESS` is 0.

---

## 4b. Designing the error type

Four common choices, from simplest to richest:

1. **`Result<T, String>` / `&'static str`**: quick scripts; callers can't match on the kind.
2. **A custom `enum`** (like `AppError` and `ConfigError`): callers can `match` on kinds; implement `Display` (human message) and `std::error::Error` (so it works with `?` into boxed errors). *Best for libraries.*
3. **`Box<dyn Error>`**: any error converts into it with `?`, and plain strings convert via `.into()`. Good for applications and `main`:

```rust
use std::error::Error;
use std::fmt;
#[derive(Debug)] struct Custom(u8);
impl fmt::Display for Custom { fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result { write!(f, "custom {}", self.0) } }
impl Error for Custom {}
fn run(n: u8) -> Result<i32, Box<dyn Error>> {
    match n {
        0 => Err("plain str".into()),
        1 => Err(format!("formatted {n}").into()),
        2 => Err(Box::new(Custom(2))),
        3 => Ok("x".parse::<i32>()?),
        _ => Ok(n as i32),
    }
}
fn main() {
    for n in 0..5 {
        match run(n) {
            Ok(v) => println!("ok {v}"),
            Err(e) => println!("err {e} | is Custom: {}", e.downcast_ref::<Custom>().is_some()),
        }
    }
}
```

✓ Output: `err plain str | is Custom: false`, `err formatted 1 | is Custom: false`, `err custom 2 | is Custom: true`, `err invalid digit found in string | is Custom: false`, `ok 4`. You lose the static type of the error, but can recover it with `downcast_ref` (use sparingly: if callers need to branch on kinds, use an enum). If the error must cross threads, use `Box<dyn Error + Send + Sync>` (Lesson 27).

4. **Helper crates** (`thiserror` for deriving the boilerplate of enums, `anyhow` for application errors with context). Lesson 26 compares them; today you write the boilerplate by hand so you know what they generate.

### Chains of causes: `source()`

An error can carry the lower-level error that caused it. The `Error` trait has `source()` for that; a reporter walks the chain:

```rust
use std::error::Error;
use std::fmt;
#[derive(Debug)] struct Low;
impl fmt::Display for Low { fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result { write!(f, "disk full") } }
impl Error for Low {}
#[derive(Debug)] struct High { source: Low }
impl fmt::Display for High { fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result { write!(f, "cannot save file") } }
impl Error for High { fn source(&self) -> Option<&(dyn Error + 'static)> { Some(&self.source) } }
fn report(e: &dyn Error) -> String {
    let mut s = e.to_string();
    let mut cur = e.source();
    while let Some(inner) = cur { s.push_str(&format!(": {inner}")); cur = inner.source(); }
    s
}
fn main() { println!("{}", report(&High { source: Low })); }
```

✓ Output `cannot save file: disk full`. A convention worth following: **an error's `Display` describes only that level** ("cannot save file"), and the cause is available through `source()`; otherwise a reporter that prints both shows the cause twice.

### Adding context: `map_err` vs a blanket `From`

`From<ParseIntError> for AppError` is convenient, but a bare `ParseIntError` says nothing about **which** input was bad. When several fields parse as integers, attach the context at the call site with `map_err`:

```rust
use std::num::ParseIntError;
#[derive(Debug)] enum E { BadField { field: &'static str, source: ParseIntError } }
fn age(s: &str) -> Result<u8, E> { s.parse().map_err(|source| E::BadField { field: "age", source }) }
fn main() { println!("{:?}", age("300")); println!("{:?}", age("30")); }
```

✓ Output `Err(BadField { field: "age", source: ParseIntError { kind: PosOverflow } })` and `Ok(30)`. (The compiler also warns the fields are never read, since only `Debug` touches them; real code reads them in `Display`.) `cfglab` in the practical challenge deliberately has no `From<ParseIntError>` for this reason. Use `From` when the conversion is lossless and obvious (e.g. `io::Error` into your `Io(io::Error)` variant); use `map_err` when context is needed.

---

## 5. Collecting and combining results

Iterators of `Result` can be collected into a `Result` of a collection: you get all the values, or the **first** error ✓:

```rust
fn main() {
    let good = ["1", "2", "3"];
    let bad = ["1", "x", "3", "y"];
    let a: Result<Vec<i32>, _> = good.iter().map(|s| s.parse::<i32>()).collect();
    let b: Result<Vec<i32>, _> = bad.iter().map(|s| s.parse::<i32>()).collect();
    println!("{a:?}\n{b:?}");
    let c: Option<Vec<u32>> = ["1", "2"].iter().map(|s| s.parse().ok()).collect();
    println!("{c:?}");
    let total: Result<i32, _> = good.iter().map(|s| s.parse::<i32>()).sum();
    println!("{total:?}");
    let (oks, errs): (Vec<_>, Vec<_>) = bad.iter().map(|s| s.parse::<i32>()).partition(|r| r.is_ok());
    println!("{} ok, {} err", oks.len(), errs.len());
    let swallowed: Vec<i32> = bad.iter().filter_map(|s| s.parse().ok()).collect();   // silently drops errors
    println!("{swallowed:?}");
    let t: Option<Result<i32, String>> = Some(Ok(1));
    println!("{:?}", t.transpose());
}
```

✓ Output:

```text
Ok([1, 2, 3])
Err(ParseIntError { kind: InvalidDigit })
Some([1, 2])
Ok(6)
2 ok, 2 err
[1, 3]
Ok(Some(1))
```

- `collect::<Result<Vec<_>, _>>()` and `sum::<Result<_, _>>()` **short-circuit**: iteration stops at the first `Err` (so the later items are not even parsed).
- `partition` gives you both sides when you want to report *all* the errors.
- `filter_map(|x| x.parse().ok())` is shorter, but it **throws the errors away**. Sometimes right ("skip lines that aren't numbers"); when it isn't, it produces silently wrong output (the debugging challenge).
- `transpose` swaps `Option<Result<T,E>>` and `Result<Option<T>,E>`; the practical challenge uses it for optional keys that must be valid when present.

---

## 6. Panics: bugs, not control flow

A **panic** is for conditions the program cannot meaningfully continue from: a violated invariant, an impossible state, a bug. The common ways to get one ✓ (each run in the same program, with exit status 101):

| Code | Message (✓) |
|---|---|
| `result.unwrap()` | `called `Result::unwrap()` on an `Err` value: "bad"` |
| `result.expect("config must load")` | `config must load: "bad"` |
| `none.unwrap()` | `called `Option::unwrap()` on a `None` value` |
| `v[3]` on an empty vec | `index out of bounds: the len is 0 but the index is 3` |

Each prints `thread 'main' panicked at <file>:<line>:<col>:` and the message, and the process exits with status **101** ✓ (my sandbox environment had backtraces enabled, so extra backtrace lines appeared; that depends on `RUST_BACKTRACE`). By default a panic **unwinds** the stack, running `Drop` for each frame (Lesson 08); compiling with `panic = "abort"` skips unwinding and aborts instead (a Cargo profile setting; I did not exercise it here).

**When is `unwrap`/`expect` acceptable?**

- In tests and examples.
- When a previous check makes failure impossible *and you can say why*: prefer `expect("reason")` over `unwrap()`, and consider restructuring so the compiler proves it (a `let else`, a slice pattern).
- At the top of a short-lived CLI where crashing with a message is the desired behavior.

**When not?** On any input you don't control (files, network, user input, other services): a panic there is a bug that an attacker or a bad day can trigger. In a long-running server a panic on a worker thread may kill the thread, poison locks (Lesson 29) or, with `panic = "abort"`, kill the process. Never use panics (or `catch_unwind`) as ordinary error handling.

Related hidden panics: indexing `v[i]` (use `get`), slicing out of range or on a non-char boundary (Lesson 06), integer division by zero, `RefCell` double borrows (Lesson 24), `Option::expect`, arithmetic overflow in debug builds (Lesson 04).

---

## 7. How big are they? ✓

```rust
use std::mem::size_of;
fn main() {
    println!("Option<u32>={} Option<&u8>={} Option<Box<u8>>={} Result<u32,()>={}", size_of::<Option<u32>>(), size_of::<Option<&u8>>(), size_of::<Option<Box<u8>>>(), size_of::<Result<u32, ()>>());
    println!("Result<(),String>={} Result<(),Box<dyn std::error::Error>>={} Result<u8,std::num::ParseIntError>={}", size_of::<Result<(), String>>(), size_of::<Result<(), Box<dyn std::error::Error>>>(), size_of::<Result<u8, std::num::ParseIntError>>());
}
```

✓ Output: `Option<u32>=8 Option<&u8>=8 Option<Box<u8>>=8 Result<u32,()>=8` and `Result<(),String>=24 Result<(),Box<dyn std::error::Error>>=16 Result<u8,std::num::ParseIntError>=2`. (Lesson 09: `Option<&u8>` and `Option<Box<u8>>` are pointer-sized thanks to the null niche; `Option<u32>` needs a tag; `Result<(), String>` reuses `String`'s capacity niche, so it is as big as `String`; a boxed `dyn Error` is a two-word fat pointer. All numbers observed on this compiler, not guaranteed except the documented `Option` pointer niches.)

Consequences: `Result<T, E>` is as large as the larger of the two payloads plus possibly a tag. A big error type makes **every** `Ok` return big too, and Clippy has a lint (`result_large_err`, I believe with a 128-byte default threshold, from memory) that warns about it. Keep error types small, or box the rare large ones. The `?` operator itself costs a branch on the discriminant, and the `From::from` call is usually inlined.

---

## 8. Common mistakes and edge cases

- **`unwrap()` on external data.** The most common cause of avoidable production panics.
- **`.ok()` / `filter_map(..ok())` that swallows errors** where you meant to report them.
- **`?` everywhere without context**: an error that says only "invalid digit found in string" tells the user nothing. Add the field, line or file with `map_err` or a context type.
- **Returning `String` errors from libraries**: callers can't match on them. Use an enum.
- **Printing the cause in both `Display` and `source()`**: the report shows it twice.
- **Eager `unwrap_or(expensive())`** (Section 2.3).
- **Using `Result<T, ()>` or `Option<T>` when the caller needs to know why.**
- **Giant error enums** that every function returns (one variant per possible failure in the whole program). Prefer per-module error types that convert upward with `From`.
- **Deriving `Debug` on error types that hold secrets** (tokens, passwords, keys): `main` prints `Error: {:?}`. Redact in a manual `Debug` impl.
- **Unused `Result` from `write!`/`writeln!`/`flush`**: handle it or document the `let _`.
- **Matching on `err.to_string()`**: fragile; match on a variant or `kind()`.
- **Catching panics to continue** (`catch_unwind`) as a substitute for validation.

---

## 9. Performance implications

- `Option`/`Result` are plain enums: no allocation, no exceptions machinery. The success path pays a discriminant check; the optimizer often removes it.
- Large `E` types inflate every `Result` (Section 7). `Box<dyn Error>` costs one allocation **only when an error occurs**, which is usually fine since errors are the cold path.
- `Result<Vec<_>, _>` collection allocates once like a normal `collect`; on error the partial vector is dropped.
- Eager `unwrap_or(f())` can add real cost in loops (Section 2.3).
- Panicking with unwinding makes the *panic* path slow and the binary bigger than `panic = "abort"`; that is irrelevant if panics are bugs.
- I did not benchmark any of this; Lesson 41 shows how.

---

## 10. Security and reliability

- **Never trust input; never `unwrap` it.** Parsing a number, decoding a message or opening a file must return an error you handle. A reachable panic is a denial-of-service vector.
- **Validate ranges after parsing**: `"70000".parse::<u32>()` succeeds; whether 70000 is a legal port is your rule (`cfglab` checks `1..=65535` and reports `OutOfRange`).
- **Don't leak internals in user-facing errors**: paths, SQL, tokens, stack details. Log the detailed chain internally (`report`), show the user a short message.
- **Fail closed.** On an authorization or validation error, return an error; don't fall back to a permissive default with `unwrap_or(true)`.
- **Errors must leave state consistent.** If step 2 of 3 fails, did step 1 get rolled back? Lesson 13's order transitions check first and mutate last; do the same, or use transactions (Lesson 46).
- **Distinguish retryable from fatal errors** in networked and on-chain code (timeouts vs invalid input); the error type should let callers decide (Lessons 50–52).
- **Rust does not make error handling correct for you**: it makes ignoring a failure visible. `let _ = send()` is still a decision to ignore it.

---

## 11. Exercises

Try each before opening the answer. They are not marked complete unless you tell me you did them.

1. **(Easy)** Rewrite `match o { Some(s) => s.len(), None => 0 }` for `o: &Option<String>` with a combinator.
2. **(Easy)** What does `first_number("12 x")` do when it is written as `let n = tok.parse::<i32>()?;` inside a function returning `Option<i32>`? Fix it two ways.
3. **(Medium)** Write `parse_pair(s: &str) -> Result<(i32, i32), PairError>` for `"3, 4"`, with an error that says *which side* was bad and keeps the parse error as its cause.
4. **(Medium)** Predict the output (and which lines print) for `some.ok_or(expensive("a"))` vs `some.ok_or_else(|| expensive("b"))` where `some = Some(1)` and `expensive` prints.
5. **(Medium)** Collecting `["1", "x", "3", "y"]` into `Result<Vec<i32>, _>`: which error do you get, and which items were parsed?
6. **(Hard)** When should an error type implement `From<ParseIntError>`, and when should the caller use `map_err` instead?
7. **(Hard)** `fn main() -> Result<(), Box<dyn Error>>` returns `Err(...)`. What is printed, to which stream, with which exit status, and what does that imply for error types that contain secrets?
8. **(Hard)** Give two situations where a panic is the right choice and two where it is the wrong one.

<details markdown="1">
<summary>Answers</summary>

1. ✓ `o.as_ref().map_or(0, |s| s.len())`; both versions returned `3` for `Some("abc")` and `0` for `None`. (`as_ref` turns `&Option<String>` into `Option<&String>` so nothing is moved.)
2. ✓ It does not compile: `error[E0277]: the `?` operator can only be used on `Option`s, not `Result`s, in a function that returns `Option``. Fix A: `tok.parse::<i32>().ok()?` (✓ gives `Some(12)` for `"12 x"`, `None` for `"a"` and `""`). Fix B: change the function to return `Result<i32, E>` and convert the `Option` with `ok_or(...)`. A decides that "not a number" and "nothing there" are the same `None`; B keeps them distinguishable.
3. ✓ (all four calls below were run)
   ```rust
   use std::num::ParseIntError;
   #[derive(Debug, PartialEq)]
   enum PairError { NoComma, BadNumber { which: &'static str, source: ParseIntError } }
   fn parse_pair(s: &str) -> Result<(i32, i32), PairError> {
       let (a, b) = s.split_once(',').ok_or(PairError::NoComma)?;
       let a = a.trim().parse().map_err(|source| PairError::BadNumber { which: "left", source })?;
       let b = b.trim().parse().map_err(|source| PairError::BadNumber { which: "right", source })?;
       Ok((a, b))
   }
   ```
   Output: `Ok((3, 4))`, `Err(NoComma)`, `Err(BadNumber { which: "right", source: ParseIntError { kind: InvalidDigit } })` for `"3,x"`, and `",1"` is an error ✓. To make `source` visible to reporters, implement `Error::source` for `PairError` (as `ConfigError` does).
4. ✓ `ok_or(expensive("a"))` prints `computing a` even though the value is `Some`; `ok_or_else(|| expensive("b"))` prints nothing.
5. ✓ The first error: `Err(ParseIntError { kind: InvalidDigit })` from `"x"`. `collect` stops there, so `"3"` and `"y"` are never parsed. To see every error use `partition` (✓ `2 ok, 2 err`).
6. Implement `From` when the conversion adds nothing and the source type's meaning is obvious in your error (an `Io(io::Error)` variant). Use `map_err` when you need context the source error lacks (which field, which line, which file) or when the same source type is used for several purposes in one function (✓ the `BadField { field, source }` example).
7. ✓ (the `Debug` form, stderr and status 1 are what I observed for a `ParseIntError` result; I used a concrete error type rather than `Box<dyn Error>` in the run) `Error: <Debug of the error>` is printed to **stderr** and the exit status is **1**. Because it is the `Debug` form, any secret held in a field of the error is printed: implement `Debug` by hand to redact it, or keep secrets out of error values.
8. Right: (a) an internal invariant that cannot be violated by any input was violated (a bug; fail loudly), (b) a test assertion; or tooling that should abort on misconfiguration at startup. Wrong: (a) a malformed request from the network; (b) a missing optional file or a failed parse of user input. (Judgment answer.)

</details>

---

## 12. Debugging challenge

A teammate wrote a function that sums a list of numbers read from a file, and tests it:

```rust
fn total(lines: &[&str]) -> i32 { lines.iter().filter_map(|s| s.trim().parse::<i32>().ok()).sum() }
fn main() { println!("{}", total(&["10", "x", " 20 ", "", "3O"])); }
```

✓ It compiles without warnings and prints `30`. The input has three bad lines (`"x"`, the empty string, and `"3O"` with a letter O instead of zero). The report says the total is "30, which looks fine". What is wrong, and how do you fix it so bad input is noticed?

<details markdown="1">
<summary>Solution</summary>

`filter_map(|s| ... .ok())` converts every `Err` into `None` and drops it, so **malformed lines vanish without a trace**; the sum is just smaller. A financial total that silently ignores lines is worse than one that crashes.

Return a `Result` and stop at the first bad line:

```rust
use std::num::ParseIntError;
fn total(lines: &[&str]) -> Result<i32, ParseIntError> {
    let mut sum = 0i32;
    for s in lines { sum += s.trim().parse::<i32>()?; }
    Ok(sum)
}
fn total2(lines: &[&str]) -> Result<i32, ParseIntError> {
    lines.iter().map(|s| s.trim().parse::<i32>()).sum()
}
```

✓ `total(&["10", "x", " 20 "])` returns `Err(ParseIntError { kind: InvalidDigit })`, `total(&["10", " 20 "])` returns `Ok(30)`, and `total2(&["1", "2"])` returns `Ok(3)`. If the program *should* tolerate bad lines, say so in the API (return the count skipped, or collect the errors with `partition`) instead of hiding it. Also note the remaining hazard: `sum += ...` can overflow `i32` (a panic in debug, a wrap in release; Lesson 04); `checked_add` with an error variant is the robust version.

</details>

---

## 13. Practical challenge: `cfglab`

Create a library crate `cfglab` (edition 2024) that parses a small `key = value` configuration:

1. `ConfigError` (`#[non_exhaustive]` enum, `Debug`, `PartialEq`, `Eq`, `Display`, `Error` with `source()`), with variants `Syntax { line }`, `DuplicateKey { key, line }`, `MissingKey(&'static str)`, `UnknownKey { key, line }`, `InvalidInt { key, source: ParseIntError }`, `InvalidBool { key, value }`, `OutOfRange { key, min, max, got }`. **No `From<ParseIntError>`**: attach the key with `map_err`.
2. `Config { name, port, workers, verbose }` and `parse(&str) -> Result<Config, ConfigError>`: skip blank lines and `#` comments; trim; keys `name` (required, non-empty), `port` (required, 1..=65535), `workers` (optional, 1..=1024, default 4), `verbose` (optional, `true`/`false`, default false); report unknown and duplicate keys with their 1-based line numbers. Use `ok_or`, `?`, `transpose`, and checked conversions (`u16::try_from`); no `unwrap`/`expect`/indexing in library code.
3. `first_even_square(&[i32]) -> Option<i32>` with `find` and `and_then(checked_mul)`, and `parse_all(&[&str]) -> Result<Vec<i32>, ParseIntError>` with `collect`.
4. `report(&dyn Error) -> String` walking the `source()` chain, and `summary(&str) -> Result<String, Box<dyn Error>>` using `?` with a boxed error plus a string error via `.into()`.
5. Tests for minimal and full configs, every error variant (including parse overflow and negative numbers), range edges (0, 65535, 70000), message text, the error chain, the combinator functions, and `downcast_ref` on the boxed error; plus a doctest. It must pass `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, and `cargo test --locked` (run `cargo generate-lockfile` once first).

<details markdown="1">
<summary>Solution</summary>

**`src/lib.rs`**

```rust
//! cfglab: parsing a small `key = value` configuration with typed, contextual errors.
//!
//! ```
//! let cfg = cfglab::parse("name = demo\nport = 8080\n")?;
//! assert_eq!((cfg.name.as_str(), cfg.port, cfg.workers, cfg.verbose), ("demo", 8080, 4, false));
//! # Ok::<(), cfglab::ConfigError>(())
//! ```

use std::collections::HashMap;
use std::error::Error;
use std::fmt;
use std::num::ParseIntError;

/// Default number of workers when the key is absent.
pub const DEFAULT_WORKERS: usize = 4;

/// Everything that can go wrong while reading a configuration.
///
/// Note there is no `From<ParseIntError>`: a bare `ParseIntError` says nothing about *which*
/// key was bad, so each call site attaches that context with `map_err`.
#[derive(Debug, PartialEq, Eq)]
#[non_exhaustive]
pub enum ConfigError {
    /// A line without `=`. `line` is 1-based.
    Syntax { line: usize },
    /// The same key appeared twice.
    DuplicateKey { key: String, line: usize },
    /// A required key is absent.
    MissingKey(&'static str),
    /// A key that is not part of the schema.
    UnknownKey { key: String, line: usize },
    /// A number could not be parsed.
    InvalidInt {
        key: &'static str,
        source: ParseIntError,
    },
    /// A boolean was not `true` or `false`.
    InvalidBool { key: &'static str, value: String },
    /// A number was valid but outside the allowed range.
    OutOfRange {
        key: &'static str,
        min: u64,
        max: u64,
        got: u64,
    },
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ConfigError::Syntax { line } => write!(f, "line {line}: expected `key = value`"),
            ConfigError::DuplicateKey { key, line } => {
                write!(f, "line {line}: duplicate key {key:?}")
            }
            ConfigError::MissingKey(key) => write!(f, "missing required key {key:?}"),
            ConfigError::UnknownKey { key, line } => write!(f, "line {line}: unknown key {key:?}"),
            ConfigError::InvalidInt { key, .. } => write!(f, "key {key:?} is not a valid number"),
            ConfigError::InvalidBool { key, value } => {
                write!(f, "key {key:?} must be true or false, got {value:?}")
            }
            ConfigError::OutOfRange { key, min, max, got } => {
                write!(f, "key {key:?} must be between {min} and {max}, got {got}")
            }
        }
    }
}

impl Error for ConfigError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            ConfigError::InvalidInt { source, .. } => Some(source),
            _ => None,
        }
    }
}

/// A validated configuration.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Config {
    pub name: String,
    pub port: u16,
    pub workers: usize,
    pub verbose: bool,
}

fn parse_number(key: &'static str, raw: &str, min: u64, max: u64) -> Result<u64, ConfigError> {
    let n: u64 = raw
        .parse()
        .map_err(|source| ConfigError::InvalidInt { key, source })?;
    if (min..=max).contains(&n) {
        Ok(n)
    } else {
        Err(ConfigError::OutOfRange {
            key,
            min,
            max,
            got: n,
        })
    }
}

fn parse_bool(key: &'static str, raw: &str) -> Result<bool, ConfigError> {
    match raw {
        "true" => Ok(true),
        "false" => Ok(false),
        other => Err(ConfigError::InvalidBool {
            key,
            value: other.to_owned(),
        }),
    }
}

/// Parses `key = value` lines. Blank lines and `#` comments are skipped. Keys: `name`
/// (required, non-empty), `port` (required, 1..=65535), `workers` (optional, 1..=1024,
/// default 4), `verbose` (optional, default false).
pub fn parse(text: &str) -> Result<Config, ConfigError> {
    let mut seen: HashMap<&str, &str> = HashMap::new();
    for (i, raw) in text.lines().enumerate() {
        let line = raw.trim();
        if line.is_empty() || line.starts_with('#') {
            continue;
        }
        let (key, value) = line
            .split_once('=')
            .ok_or(ConfigError::Syntax { line: i + 1 })?;
        let (key, value) = (key.trim(), value.trim());
        if !matches!(key, "name" | "port" | "workers" | "verbose") {
            return Err(ConfigError::UnknownKey {
                key: key.to_owned(),
                line: i + 1,
            });
        }
        if seen.insert(key, value).is_some() {
            return Err(ConfigError::DuplicateKey {
                key: key.to_owned(),
                line: i + 1,
            });
        }
    }

    let name = seen
        .get("name")
        .copied()
        .filter(|n| !n.is_empty())
        .ok_or(ConfigError::MissingKey("name"))?;
    let port = parse_number(
        "port",
        seen.get("port").ok_or(ConfigError::MissingKey("port"))?,
        1,
        65_535,
    )?;
    let workers = match seen.get("workers") {
        Some(raw) => parse_number("workers", raw, 1, 1_024)?,
        None => DEFAULT_WORKERS as u64,
    };
    let verbose = seen
        .get("verbose")
        .map(|raw| parse_bool("verbose", raw))
        .transpose()?
        .unwrap_or(false);

    Ok(Config {
        name: name.to_owned(),
        // The ranges above guarantee these conversions succeed; `try_from` keeps that checked.
        port: u16::try_from(port).map_err(|_| ConfigError::OutOfRange {
            key: "port",
            min: 1,
            max: 65_535,
            got: port,
        })?,
        workers: usize::try_from(workers).map_err(|_| ConfigError::OutOfRange {
            key: "workers",
            min: 1,
            max: 1_024,
            got: workers,
        })?,
        verbose,
    })
}

/// The first even number in `xs`, squared. `None` if there is no even number or the square
/// overflows `i32`.
pub fn first_even_square(xs: &[i32]) -> Option<i32> {
    xs.iter()
        .copied()
        .find(|x| x % 2 == 0)
        .and_then(|x| x.checked_mul(x))
}

/// Parses every string, stopping at the first failure.
pub fn parse_all(items: &[&str]) -> Result<Vec<i32>, ParseIntError> {
    items.iter().map(|s| s.trim().parse::<i32>()).collect()
}

/// Renders an error and its chain of causes as `outer: inner: innermost`.
pub fn report(err: &dyn Error) -> String {
    let mut out = err.to_string();
    let mut cur = err.source();
    while let Some(inner) = cur {
        out.push_str(": ");
        out.push_str(&inner.to_string());
        cur = inner.source();
    }
    out
}

/// Loads a config and returns a one-line summary, using `?` with a boxed error so any
/// error type can propagate.
pub fn summary(text: &str) -> Result<String, Box<dyn Error>> {
    let cfg = parse(text)?;
    if cfg.verbose && cfg.workers > 64 {
        return Err("verbose mode with more than 64 workers is not supported".into());
    }
    Ok(format!(
        "{} on port {} with {} workers",
        cfg.name, cfg.port, cfg.workers
    ))
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_minimal_and_full() {
        let c = parse("name=a\nport=1").unwrap();
        assert_eq!(
            c,
            Config {
                name: "a".into(),
                port: 1,
                workers: DEFAULT_WORKERS,
                verbose: false
            }
        );
        let c =
            parse("# hi\n\n name = svc \n port = 65535\nworkers = 8\nverbose = true\n").unwrap();
        assert_eq!(
            c,
            Config {
                name: "svc".into(),
                port: 65535,
                workers: 8,
                verbose: true
            }
        );
    }

    #[test]
    fn missing_keys() {
        assert_eq!(parse(""), Err(ConfigError::MissingKey("name")));
        assert_eq!(parse("name=a"), Err(ConfigError::MissingKey("port")));
        assert_eq!(parse("name=\nport=1"), Err(ConfigError::MissingKey("name")));
    }

    #[test]
    fn syntax_duplicates_unknown() {
        assert_eq!(parse("name=a\nport"), Err(ConfigError::Syntax { line: 2 }));
        assert_eq!(
            parse("name=a\nname=b"),
            Err(ConfigError::DuplicateKey {
                key: "name".into(),
                line: 2
            })
        );
        assert_eq!(
            parse("name=a\nport=1\ncolour=red"),
            Err(ConfigError::UnknownKey {
                key: "colour".into(),
                line: 3
            })
        );
    }

    #[test]
    fn numbers_and_ranges() {
        assert!(matches!(
            parse("name=a\nport=abc"),
            Err(ConfigError::InvalidInt { key: "port", .. })
        ));
        assert!(matches!(
            parse("name=a\nport=-1"),
            Err(ConfigError::InvalidInt { key: "port", .. })
        ));
        assert!(matches!(
            parse("name=a\nport=99999999999999999999999"),
            Err(ConfigError::InvalidInt { .. })
        ));
        assert_eq!(
            parse("name=a\nport=0"),
            Err(ConfigError::OutOfRange {
                key: "port",
                min: 1,
                max: 65_535,
                got: 0
            })
        );
        assert_eq!(
            parse("name=a\nport=70000"),
            Err(ConfigError::OutOfRange {
                key: "port",
                min: 1,
                max: 65_535,
                got: 70_000
            })
        );
        assert!(matches!(
            parse("name=a\nport=1\nworkers=0"),
            Err(ConfigError::OutOfRange { key: "workers", .. })
        ));
    }

    #[test]
    fn bools() {
        assert_eq!(
            parse("name=a\nport=1\nverbose=yes"),
            Err(ConfigError::InvalidBool {
                key: "verbose",
                value: "yes".into()
            })
        );
        assert!(!parse("name=a\nport=1\nverbose=false").unwrap().verbose);
    }

    #[test]
    fn error_messages_and_chain() {
        let e = parse("name=a\nport=abc").unwrap_err();
        assert_eq!(e.to_string(), "key \"port\" is not a valid number");
        assert_eq!(
            report(&e),
            "key \"port\" is not a valid number: invalid digit found in string"
        );
        let e = parse("name=a").unwrap_err();
        assert_eq!(report(&e), "missing required key \"port\"");
        assert!(e.source().is_none());
    }

    #[test]
    fn option_combinators() {
        assert_eq!(first_even_square(&[1, 3, 4, 6]), Some(16));
        assert_eq!(first_even_square(&[1, 3]), None);
        assert_eq!(first_even_square(&[]), None);
        assert_eq!(first_even_square(&[i32::MAX - 1]), None); // square overflows
    }

    #[test]
    fn collect_stops_at_first_error() {
        assert_eq!(parse_all(&["1", " 2 ", "3"]), Ok(vec![1, 2, 3]));
        assert!(parse_all(&["1", "x", "y"]).is_err());
        assert_eq!(parse_all(&[]), Ok(vec![]));
    }

    #[test]
    fn boxed_errors_propagate_any_error() {
        assert_eq!(
            summary("name=a\nport=80").unwrap(),
            "a on port 80 with 4 workers"
        );
        let e = summary("name=a").unwrap_err();
        assert!(e.downcast_ref::<ConfigError>().is_some());
        let e = summary("name=a\nport=80\nworkers=100\nverbose=true").unwrap_err();
        assert!(e.downcast_ref::<ConfigError>().is_none());
        assert_eq!(
            e.to_string(),
            "verbose mode with more than 64 workers is not supported"
        );
    }
}
```

✓ With this exact file: `cargo fmt --check` clean, `cargo clippy --all-targets --locked -- -D warnings` clean, `cargo test --locked` → 9 unit tests and 1 doctest passed.

**Design notes.**

- **Two-phase parse.** Phase 1 walks the lines, validating shape (`split_once('=').ok_or(Syntax)?`), the key set, and duplicates (`HashMap::insert` returns the old value, so `is_some()` detects a duplicate). Phase 2 converts the collected values into typed fields. Splitting them keeps each function short and each error attributable to a line or a key.
- **Context via `map_err`.** `parse_number` attaches the key to a bare `ParseIntError` (`InvalidInt { key, source }`); `source()` returns the underlying error so `report` prints `key "port" is not a valid number: invalid digit found in string`. `Display` mentions only this level, as recommended.
- **`transpose` for optional-but-validated keys.** `seen.get("verbose").map(|raw| parse_bool(..)).transpose()?.unwrap_or(false)` turns `Option<Result<bool, E>>` into `Result<Option<bool>, E>`, so a missing key defaults and a present-but-bad key is an error.
- **Checked narrowing.** `u16::try_from(port)` and `usize::try_from(workers)` are redundant given the range checks, but they keep the guarantee local to the conversion instead of relying on a cast (`as` would silently truncate, Lesson 04); the error branch is unreachable today and exists so a future range change cannot corrupt a value.
- **`#[non_exhaustive]`** (Lesson 13) lets the crate add error kinds without a semver break.
- **`summary` returns `Box<dyn Error>`** so the `ConfigError` (via `?`) and a plain message (via `.into()`) coexist; the tests use `downcast_ref` to show which one you got. This is the application-style signature; `parse` itself is the library-style one.
- Panics in this crate: none in library code; the tests use `unwrap`/`unwrap_err`, which is acceptable in tests.

</details>

---

## 14. Interview questions

<details markdown="1">
<summary>Answers</summary>

1. **How does Rust handle errors without exceptions?** Fallible functions return `Result<T, E>` (or `Option<T>`); the caller must handle or propagate it. Panics exist for bugs, not for expected failures.
2. **What does `?` do?** On `Ok(v)`/`Some(v)` it yields `v`; on `Err(e)`/`None` it returns early from the function, converting the error with `From::from(e)`.
3. **Why can't I use `?` in `main` that returns `()`?** `?` needs the function's return type to implement the `Try`-family traits (`Result`, `Option`, ...): E0277. Make `main` return `Result<(), E>`.
4. **What happens when `main` returns `Err`?** The error is printed to stderr with its `Debug` format (`Error: ...`) and the process exits with status 1 (`ExitCode` gives control of the status).
5. **`Option` vs `Result`?** `Option` says a value may be absent with no reason; `Result` says an operation failed and carries why. Convert with `ok_or` / `ok()`.
6. **`unwrap_or` vs `unwrap_or_else`?** The first evaluates its argument eagerly, always; the second takes a closure that runs only when needed.
7. **When is `unwrap`/`expect` acceptable?** Tests, examples, or where an earlier check proves success (document why with `expect`); never on external input.
8. **What does `collect::<Result<Vec<_>, _>>()` do on an error?** Stops at the first `Err` and returns it; later items aren't evaluated.
9. **`Box<dyn Error>` vs an enum?** Boxed: simple, anything converts in, but callers can't match kinds without `downcast`. Enum: callers can match; use for libraries.
10. **What is `Error::source` for?** Linking an error to its underlying cause so reporters can print the chain. `Display` should describe only the current level.
11. **When use `map_err` rather than `From`?** When you need to attach context the source error lacks, or the same source type has several meanings in one function.
12. **Why is ignoring a `Result` a warning?** `Result` is `#[must_use]`; ignoring it silently drops a failure. Use `let _ =` deliberately.
13. **Why is `filter_map(.. .ok())` dangerous?** It discards errors, so bad input silently shrinks the result.
14. **How large is `Option<&T>`?** The size of a pointer (null niche, guaranteed); `Option<u32>` is 8 bytes (✓ on this target) since `u32` has no spare bit patterns.
15. **What exit status does a panic give?** 101 in the Linux runs I did (✓; I did not test other platforms); it unwinds by default (running destructors) unless `panic = "abort"`.

</details>

---

## 15. Key takeaways

- Failure is part of the type: `Option<T>` for "maybe absent", `Result<T, E>` for "failed because...". Panics are for bugs.
- `?` returns early on `Err`/`None` and converts the error with `From`. It needs the function to return a compatible type (E0277 otherwise), and `Option` and `Result` don't mix without an explicit `ok_or`/`ok`.
- Make error types enums with `Display` + `Error` (+ `source()` for causes); use `Box<dyn Error>` at application edges; use `map_err` when context matters and `From` when the conversion is lossless.
- Prefer lazy `_else` forms when the fallback has a cost; use `collect::<Result<..>>()` for all-or-nothing, `partition` to see everything, and avoid `filter_map(.ok())` unless dropping is the intent.
- Never `unwrap` external input; a reachable panic is a denial-of-service bug. Panic exits with status 101, `main` returning `Err` with status 1.
- `Result<T, E>` is as big as its larger payload: keep errors small or boxed. All of this is zero-cost compared with hand-written error codes.
- Validate after parsing, fail closed, keep state consistent on error, and keep secrets out of `Debug` output.

**Connections.** Lesson 13's enums and validating constructors are now returning `Result`; Lesson 06's `parse`/`FromStr`/`let else` are the source of most errors you propagate; Lesson 08's unwinding explains what a panic does; Lesson 09's niches explain the sizes. Next: Lesson 15 (newtypes, typestate, and making invalid states unrepresentable) uses these errors in constructors and builds compile-time state machines. Lesson 17 (iterators) goes deeper into `collect` and adaptors; Lesson 26 compares `thiserror`/`anyhow` and designs error types for libraries and services; Lessons 27–32 handle errors across threads and `await` points; Lesson 39 tests error paths; Lessons 51–52 classify retryable errors in RPC and web3 services. See the [roadmap](00_roadmap.md) and the [course index](index.md).

---

## 16. References

- *The Rust Programming Language*, ch. 9 "Error Handling": <https://doc.rust-lang.org/book/ch09-00-error-handling.html>
- Rust standard library documentation: `Option`, `Result`, `std::error::Error`, `std::process::ExitCode`: <https://doc.rust-lang.org/std/option/enum.Option.html>, <https://doc.rust-lang.org/std/result/enum.Result.html>, <https://doc.rust-lang.org/std/error/trait.Error.html>, <https://doc.rust-lang.org/std/process/struct.ExitCode.html>
- *The Rust Reference*, "The question mark operator" and "Panic": <https://doc.rust-lang.org/reference/expressions/operator-expr.html#the-question-mark-operator>, <https://doc.rust-lang.org/reference/panic.html>
- *Rust API Guidelines*, "Error types are meaningful and well-behaved" (C-GOOD-ERR): <https://rust-lang.github.io/api-guidelines/interoperability.html#error-types-are-meaningful-and-well-behaved-c-good-err>
- Rust error code index: E0277 (trait bound / `?` conversions), E0271: <https://doc.rust-lang.org/error_codes/E0277.html>, <https://doc.rust-lang.org/error_codes/E0271.html>
- Rust standard library documentation: `Iterator::collect` and `FromIterator for Result`, `Iterator::partition`, `Option::transpose`: <https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.collect>, <https://doc.rust-lang.org/std/result/enum.Result.html#impl-FromIterator%3CResult%3CA,+E%3E%3E-for-Result%3CV,+E%3E>
- Clippy lint list: `result_large_err`, `unwrap_used`, `or_fun_call`: <https://rust-lang.github.io/rust-clippy/master/index.html>
- *Rust by Example*, "Error handling": <https://doc.rust-lang.org/rust-by-example/error.html>

The URLs above are the standard documentation locations; I did not open every one during this session, so if a link has moved, search the docs site for the page title. Unchecked from memory: the exact default threshold and lint group of the Clippy lints named above, and the anchor fragment in the `FromIterator for Result` link.
