---
title: "Lesson 26 — Error design: panic vs Result, custom errors, sources, anyhow, thiserror"
---

# Lesson 26 — Error design: panic vs `Result`, custom errors, sources, `anyhow`, `thiserror`

**Module 9 · Error handling** · Status: generated · Edition used in examples: **2024** · External dependencies: `thiserror 2.0.21`, `anyhow 1.0.104` (resolved from crates.io on the day of writing; **the first lesson in this course that uses external crates**)

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux. As before, ✓ marks outputs, diagnostics and measurements I actually ran; unmarked statements are documented behavior I did not run. Everything is **stable Rust**. Examples that use `thiserror` or `anyhow` were compiled in a Cargo project with those exact versions; the others with plain `rustc`.

Builds on [Lesson 14](14_option_result_errors.md) (`Option`, `Result`, `?`, the basics of error enums), [Lesson 21](21_trait_objects_dispatch.md) (`Box<dyn Error>`, `dyn Any`/downcasting) and [Lesson 24](24_interior_mutability.md) (a `RefCell` violation is a panic). Logging itself is covered in Lesson 43; this lesson only says *where* to log.

---

## 1. Three kinds of "it didn't work"

| Situation | Tool | Example |
|---|---|---|
| A **bug**: an invariant the programmer believed was true is false | `panic!` (and friends) | index out of bounds, `unreachable!()` reached |
| An **expected failure** the caller can handle or report | `Result<T, E>` | file missing, invalid input, network timeout |
| A value that may legitimately be **absent** | `Option<T>` | first element of an empty list, a map lookup |

Rust has no exceptions for ordinary control flow. Failures that can happen in a correct program are ordinary values in the return type, and the compiler makes you deal with them (`#[must_use]` on `Result`). Panics are for situations where continuing would be wrong or meaningless. Good error design is mostly choosing the right one of these three, then making errors **informative** (what failed, with which input, why) and **structured** (the caller can match on them when it needs to).

---

## 2. Panics

### 2.1 What triggers one ✓

I ran one binary with a command-line switch for each case (debug build; every case exits with status **101**):

| Code | Message printed ✓ |
|---|---|
| `None::<u8>.unwrap()` | ``called `Option::unwrap()` on a `None` value`` |
| `opt.expect("config value `port` must be set")` | ``config value `port` must be set`` |
| `"x".parse::<u8>().unwrap()` | ``called `Result::unwrap()` on an `Err` value: ParseIntError { kind: InvalidDigit }`` |
| `v[7]` with `v.len() == 3` | `index out of bounds: the len is 3 but the index is 7` |
| `10 / zero` (integers) | `attempt to divide by zero` |
| `250u8 * 3` (debug build) | `attempt to multiply with overflow` |
| `unreachable!("state {}", 3)` | `internal error: entered unreachable code: state 3` |
| `todo!()` | `not yet implemented` |
| `assert_eq!(3, 4, "length of {:?}", v)` | ``assertion `left == right` failed: length of [1, 2, 3]`` then `left: 3` and `right: 4` |
| a second `borrow_mut()` on a `RefCell` | `RefCell already borrowed` |

(Each message is preceded by ``thread 'main' (id) panicked at file:line:col:``.) Note the **overflow row is only a panic in a debug build**. The same `250u8 * 3` in an optimized (`-O`) build printed **`238`** ✓: it silently wrapped around (750 mod 256). Overflow checks are on by default in debug builds and off in release builds, unless you enable them in the profile (Lesson 4 introduced `checked_*`, `wrapping_*`, `saturating_*`). **Arithmetic on untrusted numbers must use the explicit methods**, never rely on a debug-build panic (Section 11).

### 2.2 What happens after a panic: unwinding ✓

By default a panic **unwinds** the stack of the current thread: each frame's locals are dropped (so `Drop` implementations run), then the thread ends. Other threads are unaffected; `join()` returns `Err` carrying the panic payload.

```rust
use std::panic::{self, AssertUnwindSafe};
use std::thread;
struct Guard(&'static str);
impl Drop for Guard { fn drop(&mut self) { println!("cleanup {}", self.0); } }
fn payload_text(p: &Box<dyn std::any::Any + Send>) -> String {
    if let Some(s) = p.downcast_ref::<&str>() { format!("&str: {s}") }
    else if let Some(s) = p.downcast_ref::<String>() { format!("String: {s}") }
    else { "unknown payload".to_string() }
}
fn main() {
    let r = panic::catch_unwind(|| { let _g = Guard("inner"); panic!("static message"); });
    println!("{}", payload_text(&r.unwrap_err()));
    let n = 7;
    let r = panic::catch_unwind(|| { panic!("formatted {n}"); });
    println!("{}", payload_text(&r.unwrap_err()));
    let r = panic::catch_unwind(|| { std::panic::panic_any(42u32); });
    println!("{}", payload_text(&r.unwrap_err()));
    let mut v = vec![1];
    let r = panic::catch_unwind(AssertUnwindSafe(|| { v.push(2); panic!("after push"); }));
    println!("{} {:?}", r.is_err(), v);
    let h = thread::spawn(|| { let _g = Guard("thread"); let x: Option<u8> = None; x.unwrap() });
    let j = h.join();
    println!("join is_err={}", j.is_err());
    println!("main continues");
}
```

✓ Standard output (the panic messages go to standard error):

```text
cleanup inner
&str: static message
String: formatted 7
unknown payload
true [1, 2]
cleanup thread
join is_err=true
main continues
```

What it shows:

- `cleanup inner` and `cleanup thread` were printed: **destructors run during unwinding**, which is how locks are released and files closed on panic (and why a `Mutex` becomes poisoned, Lesson 24).
- The payload is `&'static str` for a literal message, `String` for a formatted one, and whatever you give `panic_any` otherwise. You get it back as `Box<dyn Any + Send>` and downcast it (Lesson 21).
- `catch_unwind` requires the closure to be `UnwindSafe`; I used `AssertUnwindSafe` to say "I know `v` may be left half-modified". It *was* modified (`[1, 2]`): catching a panic does not roll anything back.
- A panicking thread does not take down `main`; `join()` hands you the failure.

### 2.3 Where unwinding does not happen

- **`panic = "abort"`** (a profile setting). I compiled the program above with `-C panic=abort` ✓: the first panic **aborted the process** (exit status 134), `catch_unwind` caught nothing, and no `cleanup` line was printed. Services often choose abort to avoid running code in an inconsistent state. If your design relies on catching panics (a worker pool, say), say so in the docs, because a different profile breaks it.
- **A panic while panicking.** If a `Drop` panics while the stack is already unwinding, the runtime gives up ✓:

```rust
struct Bomb;
impl Drop for Bomb { fn drop(&mut self) { panic!("panic in drop"); } }
fn main() {
    let _b = Bomb;
    panic!("first panic");
}
```

✓ Output: `first panic`, then `panic in drop`, then ``panic in a destructor during cleanup`` and ``thread caused non-unwinding panic. aborting.``, exit status 134. Destructors must not panic.

You can observe or change what a panic prints with a hook ✓:

```rust
use std::panic;
fn main() {
    panic::set_hook(Box::new(|info| {
        let loc = info.location().map(|l| format!("{}:{}", l.file(), l.line())).unwrap_or_default();
        let msg = info.payload().downcast_ref::<&str>().copied().unwrap_or("<non-str>");
        println!("[hook] {msg} at {loc}");
    }));
    let _ = panic::catch_unwind(|| panic!("custom hook sees this"));
    let _ = panic::take_hook();
    println!("done");
}
```

✓ Output `[hook] custom hook sees this at c03.rs:8` and `done` (the file name depends on what you called the file). Production programs install a hook to send panics to their logging system, and should keep the default behavior of not hiding them.

### 2.4 When to panic

Panic when the program has a **bug** or reached a state its own logic says is impossible: `unreachable!()`, `assert!` on invariants, `expect("reason")` when you can explain *why* it cannot fail, indexing where you proved the bounds. Panics are also fine in tests, examples and throwaway code. Do **not** panic on conditions that outside input can cause: a malformed file, a short network read, a missing environment variable, a user-supplied number. In a library, a function that panics on bad input makes every caller's program crashable by its own input. Write `expect` messages as *the invariant that makes it safe*: `.expect("regex is a compile-time constant")`, not `.expect("failed")`.

---

## 3. `Option`, `Result` and `?`

Lesson 14 introduced these. A few facts worth re-checking at this level.

**`?` converts the error with `From`.** If the function's error type is not the one produced, you must provide a conversion ✓:

```rust
#[derive(Debug)]
struct MyErr;
fn parse(s: &str) -> Result<i32, std::num::ParseIntError> { s.parse() }
fn f() -> Result<i32, MyErr> {
    let n = parse("5")?;
    Ok(n)
}
fn main() { println!("{:?}", f()); }
```

✓ `error[E0277]: `?` couldn't convert the error to `MyErr``, with the note that `?` "implicitly performs a conversion on the error value using the `From` trait". Fix: `impl From<ParseIntError> for MyErr`, or `.map_err(|_| MyErr)?` at the call site. Mixing `Option` and `Result` is also rejected ✓:

```rust
fn first_char_upper(s: &str) -> Option<char> {
    let c = s.chars().next()?;
    Some(c.to_ascii_uppercase())
}
fn parse_or_none(s: &str) -> Option<i32> {
    let n = s.parse::<i32>()?;
    Some(n)
}
fn main() { println!("{:?} {:?}", first_char_upper("abc"), parse_or_none("1")); }
```

✓ `error[E0277]: the `?` operator can only be used on `Option`s, not `Result`s, in a function that returns `Option``. Convert explicitly: `.ok()?` turns a `Result` into an `Option` (and drops the error, so use it only when the reason does not matter), `.ok_or(e)?` / `.ok_or_else(|| e)?` turns an `Option` into a `Result`.

**Collecting.** Iterators of `Result` can be collected into a `Result` of a collection: it stops at the first error ✓:

```rust
fn main() {
    let ok: Vec<Result<i32, String>> = vec![Ok(1), Ok(2), Ok(3)];
    let bad: Vec<Result<i32, String>> = vec![Ok(1), Err("first".into()), Err("second".into())];
    let a: Result<Vec<i32>, String> = ok.into_iter().collect();
    let mut seen = 0;
    let b: Result<Vec<i32>, String> = bad.into_iter().inspect(|_| seen += 1).collect();
    println!("{a:?} {b:?} seen={seen}");
    let o: Option<Result<i32, std::num::ParseIntError>> = Some("7".parse());
    println!("{:?}", o.transpose());
    let cfg: Option<&str> = None;
    let r: Result<&str, String> = cfg.ok_or_else(|| format!("missing {}", "key"));
    println!("{r:?}");
    let v = ["1", "2", "x", "4"].iter().map(|s| s.parse::<i32>());
    let (good, errs): (Vec<_>, Vec<_>) = v.partition(Result::is_ok);
    println!("{} {}", good.len(), errs.len());
}
```

✓ Output `Ok([1, 2, 3]) Err("first") seen=2`, `Ok(Some(7))`, `Err("missing key")`, `3 1`. `seen=2` shows the short-circuit: the third item was never looked at. `partition(Result::is_ok)` keeps *every* item (3 good, 1 bad, all still wrapped in `Result`) when you want all the failures instead of the first. Prefer `ok_or_else` over `ok_or` when building the error allocates (`format!`), because `ok_or` builds it even on success.

**Main can return a `Result`.** If it returns `Err`, the runtime prints `Error: ` plus the **`Debug`** form of the error and exits with status 1 ✓:

```rust
use std::error::Error;
fn parse(s: &str) -> Result<i32, std::num::ParseIntError> { s.parse() }
fn run() -> Result<i32, Box<dyn Error>> {
    let a = parse("4")?;
    let b = parse("x")?;
    Ok(a + b)
}
fn main() -> Result<(), Box<dyn Error>> {
    let e: Box<dyn Error> = "plain message".into();
    println!("{e} / {e:?}");
    let e2: Box<dyn Error + Send + Sync> = String::from("owned message").into();
    println!("{e2}");
    println!("{}", std::mem::size_of::<Box<dyn Error>>());
    run()?;
    println!("unreachable");
    Ok(())
}
```

✓ Output `plain message / "plain message"`, `owned message`, `16`, then `Error: ParseIntError { kind: InvalidDigit }` on standard error, exit status 1. A `&str` or `String` converts into a `Box<dyn Error>` with `.into()`/`?`, which is convenient for scripts and tests and unstructured (the caller can only print it or downcast). Note what the user sees: *a Debug dump with no context*. The debugging challenge below is exactly this.

To control the exit code, return `ExitCode` instead ✓:

```rust
use std::process::ExitCode;
#[derive(Debug)]
enum E { Usage, Data(String) }
fn real_main() -> Result<(), E> { Err(E::Data("corrupt".into())) }
fn main() -> ExitCode {
    match real_main() {
        Ok(()) => ExitCode::SUCCESS,
        Err(E::Usage) => { eprintln!("usage: tool <file>"); ExitCode::from(2) }
        Err(E::Data(m)) => { eprintln!("data error: {m}"); ExitCode::from(65) }
    }
}
```

✓ Prints `data error: corrupt` and exits with status 65 (the numbers 2 and 65 are only a convention I chose; pick a documented scheme for your tool).

**What errors cost in memory** ✓ (sizes measured on this target):

| Type | `size_of` |
|---|---|
| `std::io::Error` | 8 |
| `Result<u64, io::Error>` | 16 |
| `Result<(), Box<dyn Error>>` | 16 |
| `Result<u32, Infallible>` | 4 |
| `Result<(), ()>` | 1 |
| `Result<u64, String>` | 24 |
| `Result<(), Big>` where `Big` is an enum holding `[u8; 200]` | 201 |

`io::Error` is a single pointer-sized value; an error type that stores a lot inline makes **every** `Result<T, E>` return that big, even on the success path. Box the large variant (or the whole error) if profiling shows it matters (Lesson 41). `Infallible` (an uninhabited type) makes the error arm free: `Result<u32, Infallible>` is just a `u32`.

---

## 4. Designing an error type

### 4.1 By hand: `Display`, `Error`, `source`, `From` ✓

An error type should implement `std::error::Error` (which requires `Debug + Display`), and report its underlying cause through `source()`:

```rust
use std::error::Error;
use std::fmt;
use std::num::ParseIntError;

#[derive(Debug)]
enum ConfigError {
    Missing(&'static str),
    BadPort { raw: String, source: ParseIntError },
    Io(std::io::Error),
}
impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ConfigError::Missing(k) => write!(f, "missing key `{k}`"),
            ConfigError::BadPort { raw, .. } => write!(f, "invalid port {raw:?}"),
            ConfigError::Io(_) => write!(f, "could not read config"),
        }
    }
}
impl Error for ConfigError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            ConfigError::Missing(_) => None,
            ConfigError::BadPort { source, .. } => Some(source),
            ConfigError::Io(e) => Some(e),
        }
    }
}
impl From<std::io::Error> for ConfigError { fn from(e: std::io::Error) -> Self { ConfigError::Io(e) } }

fn port(raw: &str) -> Result<u16, ConfigError> {
    raw.trim().parse().map_err(|source| ConfigError::BadPort { raw: raw.to_string(), source })
}
fn load(path: &str) -> Result<String, ConfigError> { Ok(std::fs::read_to_string(path)?) }

fn report(e: &dyn Error) {
    println!("error: {e}");
    let mut cur = e.source();
    while let Some(s) = cur { println!("  caused by: {s}"); cur = s.source(); }
}
fn main() {
    report(&port("80x").unwrap_err());
    report(&load("/definitely/not/here.toml").unwrap_err());
    report(&ConfigError::Missing("host"));
    println!("{:?}", port("80x").unwrap_err());
    let e: Box<dyn Error> = Box::new(port("x").unwrap_err());
    println!("{}", e.downcast_ref::<ConfigError>().is_some());
    println!("{}", e.is::<std::io::Error>());
}
```

✓ Output:

```text
error: invalid port "80x"
  caused by: invalid digit found in string
error: could not read config
  caused by: No such file or directory (os error 2)
error: missing key `host`
BadPort { raw: "80x", source: ParseIntError { kind: InvalidDigit } }
true
false
```

Conventions this follows (they come from the Rust API guidelines and the standard library's own style, from memory):

1. **`Display` describes only this layer**, in lowercase without a trailing period, and does **not** repeat the source's message. The chain is reported by walking `source()` (the `report` function). If both print the source, the user sees the same text twice.
2. **`source()` returns the underlying error**, so tools can show the whole chain and `downcast_ref` can look for a specific cause.
3. **`From` implementations** make `?` work; give each *kind* of failure its own variant instead of one catch-all "Other(String)".
4. **Carry the data that identifies the failure** (`raw`, a line number, a path), not just a message. The caller can then match on it, and the message can be assembled in one place.
5. **`Box<dyn Error>` loses the type**; you can get it back with `downcast_ref::<ConfigError>()` ✓ (`true`) or test with `is::<T>()` ✓ (`false` for `io::Error`), but a typed error is easier to handle.

### 4.2 `std::io::Error` as a model ✓

`io::Error` combines an OS code, a portable `ErrorKind`, and an optional boxed custom error:

```rust
use std::io::{self, ErrorKind};
use std::error::Error;
fn main() {
    let e = io::Error::new(ErrorKind::NotFound, "no such widget");
    println!("{} | {:?} | {:?}", e, e.kind(), e);
    let e2 = io::Error::other("custom");
    println!("{} {:?}", e2, e2.kind());
    let fs = std::fs::read_to_string("/definitely/not/here").unwrap_err();
    println!("{} | {:?} | raw_os_error={:?}", fs, fs.kind(), fs.raw_os_error());
    println!("{}", fs.source().is_none());
    let inner = io::Error::new(ErrorKind::InvalidData, std::fmt::Error);
    println!("{} source={}", inner, inner.source().is_some());
    println!("{:?}", inner.get_ref().map(|r| r.to_string()));
}
```

✓ Output:

```text
no such widget | NotFound | Custom { kind: NotFound, error: "no such widget" }
custom Other
No such file or directory (os error 2) | NotFound | raw_os_error=Some(2)
true
an error occurred when formatting an argument source=false
Some("an error occurred when formatting an argument")
```

Callers branch on `kind()` (portable) rather than on message text or the raw code. Notice that wrapping an error with `io::Error::new(kind, inner)` makes `Display` print the inner message, while `source()` on the outer `io::Error` returned `None` here: `io::Error` forwards `source()` to the *inner error's* source, not to the inner error itself (what I observed; the inner `fmt::Error` has no source). The inner error is reachable through `get_ref()`. This is a good example of why you should test your chain-walking code on your actual error types.

---

## 5. `thiserror`: the derive for library errors

Writing `Display`/`Error`/`From` by hand is mechanical. The `thiserror` crate (a derive macro, so it generates the same kind of `impl`s you saw above; macros are Lesson 35) removes the boilerplate ✓ (compiled with `thiserror 2.0.21`):

```rust
use thiserror::Error;
use std::error::Error as _;

#[derive(Debug, Error)]
enum ConfigError {
    #[error("missing key `{0}`")]
    Missing(&'static str),
    #[error("invalid port {raw:?}")]
    BadPort { raw: String, #[source] source: std::num::ParseIntError },
    #[error("could not read {path}")]
    Io { path: String, #[source] source: std::io::Error },
    #[error(transparent)]
    Other(#[from] std::fmt::Error),
}
#[derive(Debug, Error)]
#[error("startup failed")]
struct Startup(#[from] ConfigError);

fn main() {
    let e = ConfigError::BadPort { raw: "80x".into(), source: "80x".parse::<u16>().unwrap_err() };
    println!("{e}");
    println!("{:?}", e.source().map(|s| s.to_string()));
    let io = ConfigError::Io { path: "a.toml".into(), source: std::io::Error::other("disk on fire") };
    println!("{io} / {}", io.source().unwrap());
    let s: Startup = ConfigError::Missing("host").into();
    println!("{s} <- {}", s.source().unwrap());
    let t = ConfigError::from(std::fmt::Error);
    println!("{t} | transparent source: {:?}", t.source().map(|x| x.to_string()));
    println!("{}", std::mem::size_of::<ConfigError>());
}
```

✓ Output:

```text
invalid port "80x"
Some("invalid digit found in string")
could not read a.toml / disk on fire
startup failed <- missing key `host`
an error occurred when formatting an argument | transparent source: None
40
```

The attributes:

- `#[error("...")]` writes `Display`; fields are interpolated as `{0}` (tuple) or `{name}` / `{name:?}` (struct variants).
- `#[source]` marks the field returned by `source()`; so does a field **named `source`**, and `#[from]` (which also generates the `From` impl, and implies `#[source]`). In my mutation test, deleting `#[source]` from a field called `source` changed nothing ✓, while deleting it from a field called `last` broke the chain.
- `#[error(transparent)]` forwards both `Display` and `source()` to the inner error (so `t.source()` is the inner error's source: `None` for `fmt::Error`). Use it for "wrapper" variants that add nothing.
- Two `#[from]` variants for the *same* source type cannot coexist, since they would generate two `impl From<ParseIntError> for E`: ✓ `error[E0119]: conflicting implementations of trait `From<ParseIntError>` for type `E``. Then convert manually with `map_err` and attach context (here, which field was parsed).
- It does **not** change your API: the generated code is plain `impl Display`/`impl Error`/`impl From`, so `thiserror` is an implementation detail and does not appear in your public types.

Two `#[from]` variants with different types work together with `?` in the same function ✓:

```rust
use thiserror::Error;
#[derive(Debug, Error)]
enum E { #[error("io: {0}")] Io(#[from] std::io::Error), #[error("parse: {0}")] Parse(#[from] std::num::ParseIntError) }
fn f(s: &str) -> Result<i32, E> {
    let n: i32 = s.parse()?;
    let _ = std::fs::metadata("/nope")?;
    Ok(n)
}
fn main() { println!("{:?}", f("x")); println!("{:?}", f("1").map_err(|e| e.to_string())); }
```

✓ Output `Err(Parse(ParseIntError { kind: InvalidDigit }))` and `Err("io: No such file or directory (os error 2)")`. (Here `Display` includes the source's text with `{0}`: a deliberate exception to convention 1 above, handy for short messages; but then do not also print the chain, or the text repeats.)

---

## 6. `anyhow`: the type for application errors

At the top of an application nobody *matches* on errors; they get reported. `anyhow::Error` is a boxed, type-erased error that carries a chain of context messages ✓ (compiled with `anyhow 1.0.104`):

```rust
use anyhow::{anyhow, bail, ensure, Context, Result};

fn read_port(raw: &str) -> Result<u16> {
    let n: u16 = raw.trim().parse().with_context(|| format!("parsing port {raw:?}"))?;
    ensure!(n >= 1024, "port {n} is privileged");
    if n == 9999 { bail!("port {} is reserved", n); }
    Ok(n)
}
fn start(raw: &str) -> Result<u16> {
    read_port(raw).context("starting server")
}
fn main() {
    for raw in ["8080", "80x", "80", "9999"] {
        match start(raw) {
            Ok(p) => println!("ok {p}"),
            Err(e) => {
                println!("display : {e}");
                println!("alt     : {e:#}");
                println!("debug   :\n{e:?}");
                println!("chain   : {:?}", e.chain().map(|c| c.to_string()).collect::<Vec<_>>());
                println!("root    : {}", e.root_cause());
            }
        }
    }
    let e = anyhow!("code {}", 7);
    println!("{e}");
    let none: Option<u8> = None;
    println!("{:#}", none.context("value was absent").unwrap_err());
    let io: Result<String> = std::fs::read_to_string("/nope").context("reading /nope");
    let err = io.unwrap_err();
    println!("{}", err.downcast_ref::<std::io::Error>().map(|e| format!("{:?}", e.kind())).unwrap_or_default());
    println!("{} {}", std::mem::size_of::<anyhow::Error>(), std::mem::size_of::<Option<anyhow::Error>>());
}
```

✓ Output for the `"80x"` and `"80"` inputs (and the rest) with `RUST_BACKTRACE=0`:

```text
ok 8080
display : starting server
alt     : starting server: parsing port "80x": invalid digit found in string
debug   :
starting server

Caused by:
    0: parsing port "80x"
    1: invalid digit found in string
chain   : ["starting server", "parsing port \"80x\"", "invalid digit found in string"]
root    : invalid digit found in string
display : starting server
alt     : starting server: port 80 is privileged
debug   :
starting server

Caused by:
    port 80 is privileged
chain   : ["starting server", "port 80 is privileged"]
root    : port 80 is privileged
display : starting server
alt     : starting server: port 9999 is reserved
debug   :
starting server

Caused by:
    port 9999 is reserved
chain   : ["starting server", "port 9999 is reserved"]
root    : port 9999 is reserved
code 7
value was absent
NotFound
8 8
```

How to read it:

- `.context(...)` / `.with_context(|| ...)` wrap an error (or turn an `Option::None` into one) with a *what-I-was-doing* message. Use `with_context` when the message needs a `format!`, so the work happens only on the error path.
- `{e}` prints only the outermost message, `{e:#}` prints the whole chain on one line separated by `: `, `{e:?}` prints the outer message plus a numbered `Caused by:` list. A `main` that returns `anyhow::Result<()>` prints the `{e:?}` form on failure.
- `bail!` returns an error immediately; `ensure!` is `assert!` for errors (it returns `Err` instead of panicking); `anyhow!` builds an error value.
- You can still **downcast** to a concrete cause: `downcast_ref::<io::Error>()` found the `NotFound` ✓.
- `anyhow::Error` is **8 bytes** and `Option<anyhow::Error>` is also 8 ✓ (a single non-null pointer), and it is `Send + Sync + 'static` (✓ a `send::<anyhow::Error>()` bound compiles).
- When an `anyhow` error is printed with `RUST_BACKTRACE` set, the `{e:?}` output also gets a `Stack backtrace:` section; I saw it when my shell had backtraces enabled (which is why the examples above were rerun with `RUST_BACKTRACE=0`). Backtraces cost time at error creation when enabled; leave them off on hot error paths in production or accept the cost deliberately.

**Limits of `anyhow::Error`** ✓:

1. It does **not** implement `std::error::Error` (that is how it can offer `From<E: Error>` without a coherence conflict): `fn needs_error<E: std::error::Error>() {}` with `needs_error::<anyhow::Error>()` gives ``error[E0277]: the trait bound `anyhow::Error: std::error::Error` is not satisfied``. So it cannot be a library's public error type that others compose.
2. `?` does **not** convert from `Box<dyn Error + Send + Sync>` ✓: in a function returning `anyhow::Result<()>`, `boxed()?` where `boxed()` returns `Result<(), Box<dyn Error + Send + Sync>>` fails with ``error[E0277]: `?` couldn't convert the error: `dyn std::error::Error + Send + Sync: Sized` is not satisfied`` . The workaround that compiled and ran is `boxed().map_err(anyhow::Error::msg)?`, which keeps the message but not the chain or the original type.

---

## 7. Libraries vs applications

| | **Library** | **Application** |
|---|---|---|
| Who handles errors? | Many unknown callers, who may need to react | A human reading a message, or a log |
| Error type | A **typed** `enum`/`struct` implementing `Error` (use `thiserror` if you like) | `anyhow::Error` (or `Box<dyn Error>`) |
| Should callers `match`? | Yes: variants are API | No: add context and report |
| `#[non_exhaustive]`? | Yes on public enums, so adding a variant is not a breaking change (✓ see below) | Not needed |
| Context | Data in the variants (path, line, key) | `.context("what I was doing")` |
| Third-party errors in the public API | Avoid: they become part of your semver | Fine |

The usual architecture: the library defines its errors; the application's `main` (or each top-level task) uses `anyhow`, adds context as the error travels up, and prints it once. Don't put `anyhow::Error` in a library's public signatures (✓ it is not an `Error`, and callers cannot match on it); don't make a script define six error enums when `anyhow` would do.

**`#[non_exhaustive]` ✓.** Marking a public enum `#[non_exhaustive]` forces downstream `match` expressions to have a wildcard arm. The practical challenge's `ConfigError` has it, and a downstream `match` without `_` fails to compile (✓ `error[E0004]: non-exhaustive patterns: `_` not covered`, noting "`ConfigError` is marked as non-exhaustive"). It is a promise of future variants; use it deliberately.

---

## 8. Strategies that go beyond the type

- **Context at every boundary.** An error should answer *what was being done, to which input, and why it failed*. `invalid digit found in string` alone is the cause but not the story (the debugging challenge).
- **Retryable vs permanent.** A timeout may succeed next time; a refused credential won't. Put that classification in a method on the error (the practical challenge's `Retryable`) so retry code doesn't parse messages. Add exponential backoff (and jitter, in real systems) and always cap attempts. Lesson 52 returns to this for networks.
- **Report all problems when the user is waiting for feedback.** A config validator that stops at the first error makes the user fix and rerun five times; collect them (`parse_all`) while bounding the number you keep (see Section 11).
- **Log once, at the boundary.** If every layer logs the error *and* returns it, one failure appears five times. Either handle it (and log that you did) or return it (and let the top-level report it with its chain). Never log secrets, tokens, or full request bodies in error paths (Lesson 43).
- **Recover on purpose, or fail fast.** Fall back only when the degraded behavior is acceptable and visible (a warning, a metric). Silent fallbacks hide real failures.
- **Clean up on the error path.** RAII (Lesson 8) closes files and releases locks on early `?` returns for free; manual state changes (`self.busy = true`) need a guard type or an explicit rollback.
- **Don't use panics for control flow**, and don't catch panics to continue as if nothing happened unless you discard or rebuild the state they touched.

### 8.1 Converting a panic at a boundary ✓

A long-running worker may want one bad job to become a failed job, not a dead worker. `catch_unwind` at the boundary turns the panic into a value (practical challenge, `boundary::catch`): it returns `Err(PanicError { message })` for `&str`, `String` and other payloads ✓ (tests). This only works with `panic = "unwind"` (✓ Section 2.3), and the data the closure touched may be inconsistent; use it where you then discard that data. Letting a panic cross an `extern "C"` boundary is a separate hazard that Lesson 37 covers.

---

## 9. Common mistakes and edge cases

- **`unwrap()` on input-dependent values** (✓ panics with a bare message). Use `?`, `match`, or `expect` with the invariant.
- **Stringly-typed errors** (`Err("bad line".into())`): callers cannot match, and context is lost (✓ the challenge's output).
- **`Box<dyn Error>` in a library API**: callers can't match on it; they can only downcast.
- **A catch-all `Other(String)` variant** that every failure ends up in.
- **Two `#[from]` for the same type** (✓ E0119): wrap with a variant that says *which* field failed.
- **Duplicating the source in `Display` and printing the chain** (the message appears twice).
- **Dropping the source** (`map_err(|_| MyErr)`): you lose the cause; keep it as `source`.
- **`?` in a function with a different error type and no `From`** (✓ E0277). **`?` on a `Result` in a function returning `Option`** (✓ E0277).
- **Relying on overflow panics** (✓ release build printed `238`).
- **Panicking in `Drop`** (✓ abort during unwinding).
- **`catch_unwind` as a general error mechanism**: it doesn't work under `panic=abort` (✓) and it doesn't roll back state.
- **Matching on error message text** (`e.to_string().contains("not found")`): use `kind()` / variants.
- **`#[non_exhaustive]` forgotten** on a public error enum, so adding a variant later is a breaking change.
- **Returning `anyhow::Error` from a library** (✓ it is not an `Error`).

---

## 10. Performance implications

- A `Result<T, E>` is as big as the bigger of `T` and `E` plus a tag (✓ the table in Section 3): a fat `E` slows down every call, not only failing ones. Box rarely-used large variants; `io::Error` is 8 bytes (✓).
- Constructing an error should be lazy: `ok_or_else`, `with_context`, `map_err(|e| ...)` run their closure only on failure; `ok_or(format!(..))` and `context(format!(..))` allocate even on success.
- `Box<dyn Error>` and `anyhow::Error` allocate when created and use dynamic dispatch to print; that is irrelevant off the hot path and potentially significant if you create millions of errors per second as control flow (don't).
- Backtrace capture (when enabled) is expensive relative to the rest of an error return; know whether your service enables it.
- Panics unwinding through many frames cost far more than returning an `Err`, and are not meant for expected conditions. I did not benchmark either; the claim is the documented design intent, so measure if you doubt it (Lesson 41).

---

## 11. Security and reliability

- **`unwrap` on untrusted data is a remotely triggerable crash.** Any request that makes a library panic is a denial-of-service in a server. Treat parse and conversion of external data as fallible.
- **Integer overflow silently wraps in release builds** (✓ `238`), which can turn a length check into a bypass or a buffer-size miscalculation. Use `checked_*` (return `None`/error on overflow) for sizes, counts, balances and offsets derived from input, or `saturating_*`/`wrapping_*` when that is really the semantics you want. Debug-only panics are a testing aid, not a defense.
- **Error messages are output.** Don't put secrets, tokens, full SQL, internal file paths or stack traces in errors that reach end users or untrusted logs. Separate a user-facing message from the internal chain. Be careful with messages that differ based on whether a user or a resource exists (account enumeration); whether that matters depends on your threat model.
- **Log injection.** Input embedded in an error message may contain newlines or control characters. Use `{:?}` (✓ the lesson's messages quote input with `{raw:?}`) or structured logging fields so attackers can't forge log lines.
- **Unbounded error collection.** `parse_all` keeps one error per bad line; a hostile multi-gigabyte input can make that `Vec` huge. The practical challenge does **not** cap it; in real code stop after N errors and report "and more".
- **Retry storms.** Retrying without caps, backoff or jitter turns a brief outage into a self-inflicted overload; retrying non-idempotent operations can duplicate side effects (payments, transactions). Retry only what is safe to repeat.
- **`catch_unwind` hides state corruption.** If a panic interrupted an update, the data may violate invariants; poisoned mutexes (Lesson 24) are the standard library's version of the same warning.
- **`panic = "abort"`** trades cleanup for a smaller blast radius from inconsistent state; choose it knowingly (✓ it disables `catch_unwind`).
- **Don't claim that Rust's error handling makes a program safe or secure**; it makes failures *visible in types*, which helps, but logic errors, injection and resource exhaustion remain.

---

## 12. Exercises

Try each before opening the answer. They are not marked complete unless you tell me you did them.

1. **(Easy)** What does `250u8 * 3` do in a debug build and in an optimized build?
2. **(Easy)** Which of these panic: `v[7]` (`v.len() == 3`); `"7".parse::<u8>().unwrap()`; `None::<u8>.unwrap()`; `let c = RefCell::new(1); let _a = c.borrow_mut(); let _b = c.borrow_mut();`; `let n: Option<u8> = None; n.unwrap_or(0)`?
3. **(Medium)** `fn f() -> Result<i32, MyErr>` calls `"5".parse::<i32>()?` and fails with E0277. Give two fixes.
4. **(Medium)** What are the three jobs of an error type's `Display`, `Debug`, and `source()`, and what goes wrong if `Display` also prints the source?
5. **(Medium)** When would you choose `Box<dyn Error>`, `thiserror`, and `anyhow`?
6. **(Hard)** Write `fn chain(e: &(dyn Error + 'static)) -> Vec<String>` that returns the message of an error and every cause.
7. **(Hard)** Classify as retryable or permanent, and say what extra information you need: connection reset; HTTP 404; HTTP 503 with `Retry-After`; invalid JSON in the response body; TLS certificate expired.
8. **(Hard)** `parse_all` collects every error. Change its design so that a hostile input cannot make it allocate unbounded memory, without hiding the fact that there were more errors.

<details markdown="1">
<summary>Answers</summary>

1. ✓ Debug: panic `attempt to multiply with overflow`. Optimized: no check, the result wraps (750 mod 256) to **238**. (Overflow checks can be enabled for release in the profile, and `checked_mul` makes the intent explicit.)
2. ✓ Panics: `v[7]` (`index out of bounds: the len is 3 but the index is 7`), `None::<u8>.unwrap()`, and the double `borrow_mut` (`RefCell already borrowed`). Does not panic: `"7".parse::<u8>().unwrap()` (it parses to `7`) and `n.unwrap_or(0)` (returns `0`).
3. ✓ (a) Implement `From<std::num::ParseIntError> for MyErr` so `?` can convert it. (b) Convert at the call site: `"5".parse::<i32>().map_err(|_| MyErr)?` (or `map_err(MyErr::Parse)?` if the variant keeps the cause). Prefer keeping the cause.
4. `Display` is the human message for *this* layer; `Debug` is for developers (derive it; it shows the structure and is what `main -> Result` prints ✓); `source()` links to the underlying cause so tools can walk the chain (✓ `report`). If `Display` also prints the source, a chain-printing reporter shows the cause twice.
5. `Box<dyn Error>`: quick scripts, tests, and prototypes where you just want `?` to work and print. `thiserror`: libraries and any code where callers will match on variants. `anyhow`: application top levels where errors are reported with context and not matched. Libraries typically expose typed errors; applications wrap them with context.
6. `fn chain(e: &(dyn Error + 'static)) -> Vec<String> { let mut out = vec![e.to_string()]; let mut cur = e.source(); while let Some(s) = cur { out.push(s.to_string()); cur = s.source(); } out }`. This is `errlab::error_chain` in the practical challenge ✓ (tested on three-level chains).
7. Reasonable classification (not run, depends on the protocol and your policy): connection reset: retryable (if the operation is idempotent). HTTP 404: permanent (unless you expect eventual consistency, in which case a limited retry). HTTP 503: retryable, honoring `Retry-After` and backoff. Invalid JSON: permanent for this response (a server bug; retrying may help only if a proxy corrupted it). Expired certificate: permanent until someone fixes it, and **never** "retry with verification off". You need to know whether the operation is idempotent (safe to repeat) before retrying anything with side effects.
8. Keep at most N errors (for example 20), count the rest, and return `(Vec<ConfigError>, usize /* omitted */)` or add a `TooManyErrors { shown, omitted }` variant that renders as "and 4 981 more". Also stop scanning early once you hit a hard cap, and bound the input size before parsing. The design lesson is to make the *bound* part of the API instead of an accident.

</details>

---

## 13. Debugging challenge

A teammate's program reports this to the user:

```text
Error: ParseIntError { kind: InvalidDigit }
```

and exits with status 1. Here is the code:

```rust
use std::collections::HashMap;
use std::error::Error;

fn parse_config(text: &str) -> Result<HashMap<String, String>, Box<dyn Error>> {
    let mut map = HashMap::new();
    for line in text.lines() {
        let (k, v) = line.split_once('=').ok_or("bad line")?;
        map.insert(k.trim().to_string(), v.trim().to_string());
    }
    Ok(map)
}

fn port(cfg: &HashMap<String, String>) -> Result<u16, Box<dyn Error>> {
    let raw = cfg.get("port").ok_or("missing key")?;
    Ok(raw.parse()?)
}

fn main() -> Result<(), Box<dyn Error>> {
    let text = "host = example.org\nport = 80x\nmode = fast";
    let cfg = parse_config(text)?;
    println!("port = {}", port(&cfg)?);
    Ok(())
}
```

1. List everything that is wrong with the error reporting (there are four problems). 2. Fix it with a typed error (standard library only). 3. Show a second fix with `anyhow`. 4. For the input `"host = x\nbroken line"` and for `"host = x"`, what should the program say?

<details markdown="1">
<summary>Solution</summary>

**1. The problems.** ✓ Output is `Error: ParseIntError { kind: InvalidDigit }`, exit 1.
(a) **No context**: the message doesn't say it was the *port*, or that the value was `"80x"`. (b) **Stringly-typed errors**: `ok_or("bad line")` and `ok_or("missing key")` tell the user nothing about *which* line or key and give the caller nothing to match. (c) **`Debug` output to the user**: `main` returning `Err` prints `{:?}`, designed for developers. (d) **`Box<dyn Error>` everywhere**: `port()` and `parse_config()` are reusable pieces whose callers cannot react to specific failures.

**2. Typed fix (std only).**

```rust
use std::collections::HashMap;
use std::error::Error;
use std::fmt;
use std::num::ParseIntError;

#[derive(Debug)]
enum ConfigError {
    BadLine { line: usize, text: String },
    MissingKey(&'static str),
    BadValue { key: &'static str, raw: String, source: ParseIntError },
}
impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ConfigError::BadLine { line, text } => write!(f, "line {line}: expected `key = value`, got {text:?}"),
            ConfigError::MissingKey(k) => write!(f, "missing key `{k}`"),
            ConfigError::BadValue { key, raw, .. } => write!(f, "invalid value {raw:?} for `{key}`"),
        }
    }
}
impl Error for ConfigError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self { ConfigError::BadValue { source, .. } => Some(source), _ => None }
    }
}

fn parse_config(text: &str) -> Result<HashMap<String, String>, ConfigError> {
    let mut map = HashMap::new();
    for (i, line) in text.lines().enumerate() {
        let (k, v) = line.split_once('=').ok_or_else(|| ConfigError::BadLine { line: i + 1, text: line.to_string() })?;
        map.insert(k.trim().to_string(), v.trim().to_string());
    }
    Ok(map)
}
fn port(cfg: &HashMap<String, String>) -> Result<u16, ConfigError> {
    let raw = cfg.get("port").ok_or(ConfigError::MissingKey("port"))?;
    raw.parse().map_err(|source| ConfigError::BadValue { key: "port", raw: raw.clone(), source })
}

fn report(e: &(dyn Error + 'static)) -> String {
    let mut out = e.to_string();
    let mut cur = e.source();
    while let Some(s) = cur { out.push_str(&format!(": {s}")); cur = s.source(); }
    out
}
fn run(text: &str) -> Result<u16, ConfigError> { port(&parse_config(text)?) }

fn main() {
    for text in ["host = example.org\nport = 80x\nmode = fast", "host = x\nbroken line", "host = x"] {
        match run(text) {
            Ok(p) => println!("port = {p}"),
            Err(e) => println!("error: {}", report(&e)),
        }
    }
}
```

✓ Output:

```text
error: invalid value "80x" for `port`: invalid digit found in string
error: line 2: expected `key = value`, got "broken line"
error: missing key `port`
```

Each message now answers *what* (which key or line), *which input* (`"80x"`, `"broken line"`), and *why* (the source). The caller can `match` on `ConfigError`. `main` decides the presentation (here a single line built by walking `source()`) instead of leaking `Debug`.

**3. `anyhow` fix.** Appropriate for an application whose `main` just reports:

```rust
use anyhow::{Context, Result, anyhow};
use std::collections::HashMap;

fn parse_config(text: &str) -> Result<HashMap<String, String>> {
    let mut map = HashMap::new();
    for (i, line) in text.lines().enumerate() {
        let (k, v) = line
            .split_once('=')
            .ok_or_else(|| anyhow!("line {}: expected `key = value`, got {line:?}", i + 1))?;
        map.insert(k.trim().to_string(), v.trim().to_string());
    }
    Ok(map)
}

fn port(cfg: &HashMap<String, String>) -> Result<u16> {
    let raw = cfg.get("port").context("missing key `port`")?;
    raw.parse().with_context(|| format!("invalid value {raw:?} for `port`"))
}

fn main() -> Result<()> {
    let text = "host = example.org\nport = 80x\nmode = fast";
    let cfg = parse_config(text).context("loading configuration")?;
    println!("port = {}", port(&cfg).context("loading configuration")?);
    Ok(())
}
```

✓ Output (standard error, exit status 1):

```text
Error: loading configuration

Caused by:
    0: invalid value "80x" for `port`
    1: invalid digit found in string
```

**4.** For `"host = x\nbroken line"`: `error: line 2: expected `key = value`, got "broken line"` (✓ typed version). For `"host = x"`: `error: missing key `port`` (✓). The trade-off: the typed version lets a caller distinguish `BadLine` from `MissingKey` and react; the `anyhow` version is shorter and reads well but a caller can only downcast or print.

</details>

---

## 14. Practical challenge: `errlab`

Create a library crate `errlab` (edition 2024; dependencies `thiserror = "2.0.21"` and `anyhow = "1.0.104"`) with four layers:

1. **`config`** (library style, `thiserror`): `parse(text) -> Result<Config, ConfigError>` and `parse_all(text) -> Result<Config, Vec<ConfigError>>` for `key = value` lines (`#` comments and blank lines ignored). Keys: `host` (required, non-empty), `port` (required, 1..=65535), `retries` (optional, 0..=10, default 3). `ConfigError` is `#[non_exhaustive]` with variants `Syntax{line,text}`, `Missing{key}`, `Unknown{line,key}`, `Duplicate{key,first,second}` and `Invalid{key,raw,#[source] source: ValueError}`; `ValueError` has `NotANumber(#[from] ParseIntError)`, `OutOfRange{min,max,got}` and `Empty`. `parse` returns the first error from `parse_all`.
2. **`retry`**: a trait `Retryable { fn is_retryable(&self) -> bool }`; `retry(max_attempts, op) -> Result<T, RetryError<E>>` where `RetryError` has `Exhausted{attempts,last}` and `Permanent{attempt,source}` (both with sources); stop at once on a permanent error; always run at least once; and `backoff_ms(attempt, base, cap)` that doubles, caps, and never overflows.
3. **`boundary`**: `catch(f) -> Result<R, PanicError>` using `catch_unwind`, extracting `&str`/`String` payloads.
4. **`app`** (application style, `anyhow`): `load(text)` adding the context "loading configuration"; `connect(cfg, fail_times, refuse)` that retries a simulated `NetError` (`Timeout` retryable, `Refused` not) up to `retries + 1` times and adds the context "connecting to host:port"; and `run(...)`.
5. A helper `error_chain(&dyn Error) -> Vec<String>`; tests for every message, the source chain depth, `parse_all` collecting several errors, retry counts for success/exhaustion/permanent, backoff edge cases, panic payload kinds, the `{e:#}` output of `app::run`, downcasting through the `anyhow` context to `ConfigError`, and that all error types are `Send + Sync + 'static`; a usage doctest and a `compile_fail` doctest that a `match` on `ConfigError` without `_` does not compile.
6. It must pass `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, and `cargo test --locked` (run `cargo generate-lockfile` once first, with network access to crates.io).

<details markdown="1">
<summary>Solution</summary>

**`Cargo.toml`** (the relevant part; `cargo add thiserror anyhow` produced exactly these versions, and `Cargo.lock` locked 7 packages)

```toml
[package]
name = "errlab"
version = "0.1.0"
edition = "2024"

[dependencies]
anyhow = "1.0.104"
thiserror = "2.0.21"
```

**`src/lib.rs`**

```rust
//! `errlab`: error design in four layers.
//!
//! * [`config`]: a *library-style* typed error ([`config::ConfigError`])
//!   built with `thiserror`: matchable variants, source chains, line numbers.
//! * [`retry`]: classify errors as retryable or permanent and retry.
//! * [`boundary`]: turn a panic at a boundary into an error value.
//! * [`app`]: *application-style* orchestration with `anyhow` context.

use std::error::Error;

/// Walks `e` and its `source()` chain, returning each message (outermost first).
pub fn error_chain(e: &(dyn Error + 'static)) -> Vec<String> {
    let mut out = vec![e.to_string()];
    let mut cur = e.source();
    while let Some(s) = cur {
        out.push(s.to_string());
        cur = s.source();
    }
    out
}

pub mod config {
    //! Parses `key = value` text. Lines starting with `#` and blank lines are
    //! ignored. Keys: `host` (required), `port` (required, 1..=65535) and
    //! `retries` (optional, 0..=10, default 3).

    use std::collections::HashMap;
    use std::num::ParseIntError;
    use thiserror::Error;

    /// A parsed configuration.
    #[derive(Debug, Clone, PartialEq, Eq)]
    pub struct Config {
        pub host: String,
        pub port: u16,
        pub retries: u8,
    }

    /// What is wrong with a single value.
    #[derive(Debug, Error, PartialEq, Eq)]
    #[non_exhaustive]
    pub enum ValueError {
        #[error("not a number")]
        NotANumber(#[from] ParseIntError),
        #[error("{got} is outside {min}..={max}")]
        OutOfRange { min: u32, max: u32, got: u32 },
        #[error("value is empty")]
        Empty,
    }

    /// Why a configuration could not be loaded.
    ///
    /// The enum is `#[non_exhaustive]`: downstream code must keep a wildcard
    /// arm, so adding a variant later is not a breaking change.
    ///
    /// ```
    /// use errlab::config::{parse, ConfigError};
    /// match parse("host = h") {
    ///     Err(ConfigError::Missing { key }) => assert_eq!(key, "port"),
    ///     Err(_) => unreachable!("other errors"),
    ///     Ok(_) => unreachable!(),
    /// }
    /// ```
    ///
    /// A `match` without a wildcard arm does not compile outside this crate:
    ///
    /// ```compile_fail
    /// use errlab::config::ConfigError;
    /// fn f(e: ConfigError) -> u8 {
    ///     match e {
    ///         ConfigError::Syntax { .. } => 1,
    ///         ConfigError::Missing { .. } => 2,
    ///         ConfigError::Unknown { .. } => 3,
    ///         ConfigError::Duplicate { .. } => 4,
    ///         ConfigError::Invalid { .. } => 5,
    ///     }
    /// }
    /// ```
    #[derive(Debug, Error, PartialEq, Eq)]
    #[non_exhaustive]
    pub enum ConfigError {
        #[error("line {line}: expected `key = value`, got {text:?}")]
        Syntax { line: usize, text: String },
        #[error("missing required key `{key}`")]
        Missing { key: &'static str },
        #[error("line {line}: unknown key `{key}`")]
        Unknown { line: usize, key: String },
        #[error("key `{key}` defined twice (lines {first} and {second})")]
        Duplicate {
            key: String,
            first: usize,
            second: usize,
        },
        #[error("invalid value {raw:?} for `{key}`")]
        Invalid {
            key: &'static str,
            raw: String,
            #[source]
            source: ValueError,
        },
    }

    fn number(key: &'static str, raw: &str, min: u32, max: u32) -> Result<u32, Box<ConfigError>> {
        let invalid = |source: ValueError| {
            Box::new(ConfigError::Invalid {
                key,
                raw: raw.to_string(),
                source,
            })
        };
        if raw.is_empty() {
            return Err(invalid(ValueError::Empty));
        }
        let got: u32 = raw.parse().map_err(|e: ParseIntError| invalid(e.into()))?;
        if !(min..=max).contains(&got) {
            return Err(invalid(ValueError::OutOfRange { min, max, got }));
        }
        Ok(got)
    }

    /// Parses the text and returns the first problem found.
    pub fn parse(text: &str) -> Result<Config, ConfigError> {
        parse_all(text).map_err(|mut errs| errs.swap_remove(0))
    }

    /// Parses the text and returns *every* problem found (never an empty `Vec`).
    pub fn parse_all(text: &str) -> Result<Config, Vec<ConfigError>> {
        let mut errors = Vec::new();
        let mut seen: HashMap<&str, (usize, &str)> = HashMap::new();
        for (i, line) in text.lines().enumerate() {
            let n = i + 1;
            let trimmed = line.trim();
            if trimmed.is_empty() || trimmed.starts_with('#') {
                continue;
            }
            let Some((k, v)) = trimmed.split_once('=') else {
                errors.push(ConfigError::Syntax {
                    line: n,
                    text: line.to_string(),
                });
                continue;
            };
            let (k, v) = (k.trim(), v.trim());
            if !matches!(k, "host" | "port" | "retries") {
                errors.push(ConfigError::Unknown {
                    line: n,
                    key: k.to_string(),
                });
                continue;
            }
            if let Some((first, _)) = seen.get(k) {
                errors.push(ConfigError::Duplicate {
                    key: k.to_string(),
                    first: *first,
                    second: n,
                });
                continue;
            }
            seen.insert(k, (n, v));
        }

        let mut host = None;
        match seen.get("host") {
            None => errors.push(ConfigError::Missing { key: "host" }),
            Some((_, "")) => errors.push(ConfigError::Invalid {
                key: "host",
                raw: String::new(),
                source: ValueError::Empty,
            }),
            Some((_, v)) => host = Some((*v).to_string()),
        }
        let mut port = None;
        match seen.get("port") {
            None => errors.push(ConfigError::Missing { key: "port" }),
            Some((_, v)) => match number("port", v, 1, 65_535) {
                Ok(p) => port = Some(p as u16),
                Err(e) => errors.push(*e),
            },
        }
        let mut retries = 3u8;
        if let Some((_, v)) = seen.get("retries") {
            match number("retries", v, 0, 10) {
                Ok(r) => retries = r as u8,
                Err(e) => errors.push(*e),
            }
        }

        match (host, port, errors.is_empty()) {
            (Some(host), Some(port), true) => Ok(Config {
                host,
                port,
                retries,
            }),
            _ => Err(errors),
        }
    }
}

pub mod retry {
    //! Retrying only what is worth retrying.

    use std::error::Error;
    use thiserror::Error;

    /// Errors that know whether trying again could help.
    pub trait Retryable {
        fn is_retryable(&self) -> bool;
    }

    /// Why [`retry`] gave up.
    #[derive(Debug, Error)]
    pub enum RetryError<E: Error + 'static> {
        #[error("gave up after {attempts} attempts")]
        Exhausted {
            attempts: u32,
            #[source]
            last: E,
        },
        #[error("permanent failure on attempt {attempt}")]
        Permanent {
            attempt: u32,
            #[source]
            source: E,
        },
    }

    /// Calls `op(attempt)` (attempts start at 1) up to `max_attempts` times.
    /// Stops at once on a non-retryable error. `max_attempts == 0` is treated
    /// as 1: the operation always runs at least once (the first attempt
    /// happens before the limit is checked).
    pub fn retry<T, E: Error + Retryable + 'static>(
        max_attempts: u32,
        mut op: impl FnMut(u32) -> Result<T, E>,
    ) -> Result<T, RetryError<E>> {
        let mut attempt = 1;
        loop {
            match op(attempt) {
                Ok(v) => return Ok(v),
                Err(e) if !e.is_retryable() => {
                    return Err(RetryError::Permanent { attempt, source: e });
                }
                Err(e) if attempt >= max_attempts => {
                    return Err(RetryError::Exhausted {
                        attempts: attempt,
                        last: e,
                    });
                }
                Err(_) => attempt += 1,
            }
        }
    }

    /// Exponential backoff in milliseconds: `base`, `2*base`, `4*base`, ...
    /// capped at `cap`, with no overflow for any `attempt` (1-based).
    /// No jitter: add random jitter in real systems to avoid thundering herds.
    pub fn backoff_ms(attempt: u32, base: u64, cap: u64) -> u64 {
        let shift = attempt.saturating_sub(1).min(63);
        base.saturating_mul(1u64 << shift).min(cap)
    }
}

pub mod boundary {
    //! Converting a panic into an error at a boundary (a worker loop, an FFI
    //! entry point). This only works when panics unwind (`panic = "unwind"`,
    //! the default); with `panic = "abort"` the process stops instead.

    use std::panic::{self, UnwindSafe};
    use thiserror::Error;

    #[derive(Debug, Error, PartialEq, Eq)]
    #[error("panic: {message}")]
    pub struct PanicError {
        pub message: String,
    }

    /// Runs `f`, returning `Err` if it panicked.
    ///
    /// Caveat: data that `f` was modifying may be half-updated. Only use this
    /// where you then discard that data or restart the component.
    pub fn catch<R>(f: impl FnOnce() -> R + UnwindSafe) -> Result<R, PanicError> {
        panic::catch_unwind(f).map_err(|payload| {
            let message = if let Some(s) = payload.downcast_ref::<&str>() {
                (*s).to_string()
            } else if let Some(s) = payload.downcast_ref::<String>() {
                s.clone()
            } else {
                "non-string panic payload".to_string()
            };
            PanicError { message }
        })
    }
}

pub mod app {
    //! Application-level code: nobody matches on these errors, they are shown
    //! to a human, so `anyhow` context is the right tool.

    use crate::config::{self, Config};
    use crate::retry::{self, Retryable};
    use anyhow::{Context, Result, ensure};
    use thiserror::Error;

    /// A simulated network failure.
    #[derive(Debug, Error, PartialEq, Eq)]
    pub enum NetError {
        #[error("timed out")]
        Timeout,
        #[error("connection refused")]
        Refused,
    }

    impl Retryable for NetError {
        fn is_retryable(&self) -> bool {
            matches!(self, NetError::Timeout)
        }
    }

    /// Loads the configuration; errors carry a "loading configuration" context.
    pub fn load(text: &str) -> Result<Config> {
        config::parse(text).context("loading configuration")
    }

    /// "Connects" to the configured host. The first `fail_times` attempts time
    /// out; if `refuse` is set the error is a permanent `Refused` instead.
    /// Returns the number of attempts used.
    pub fn connect(cfg: &Config, fail_times: u32, refuse: bool) -> Result<u32> {
        let attempts = u32::from(cfg.retries) + 1;
        let used = retry::retry(attempts, |attempt| {
            if refuse {
                Err(NetError::Refused)
            } else if attempt <= fail_times {
                Err(NetError::Timeout)
            } else {
                Ok(attempt)
            }
        })
        .with_context(|| format!("connecting to {}:{}", cfg.host, cfg.port))?;
        ensure!(used >= 1, "internal error: zero attempts");
        Ok(used)
    }

    /// Load, connect, and describe the outcome in one line.
    pub fn run(text: &str, fail_times: u32, refuse: bool) -> Result<String> {
        let cfg = load(text)?;
        let used = connect(&cfg, fail_times, refuse)?;
        Ok(format!(
            "connected to {}:{} after {used} attempt(s)",
            cfg.host, cfg.port
        ))
    }
}

#[cfg(test)]
mod tests {
    use super::config::*;
    use super::retry::*;
    use super::*;
    use crate::app::{self, NetError};
    use crate::boundary;

    const GOOD: &str = "# demo\nhost = example.org\n\nport = 8080\n";

    #[test]
    fn parses_with_defaults_and_comments() {
        assert_eq!(
            parse(GOOD),
            Ok(Config {
                host: "example.org".into(),
                port: 8080,
                retries: 3
            })
        );
        let c = parse("host=h\nport=1\nretries=0").unwrap();
        assert_eq!((c.port, c.retries), (1, 0));
    }

    #[test]
    fn each_error_variant_has_a_useful_message() {
        let msg = |t: &str| parse(t).unwrap_err().to_string();
        assert_eq!(
            msg("host=h\nbroken"),
            "line 2: expected `key = value`, got \"broken\""
        );
        assert_eq!(msg("host=h"), "missing required key `port`");
        assert_eq!(msg("port=1"), "missing required key `host`");
        assert_eq!(
            msg("host=h\nport=1\ncolor=red"),
            "line 3: unknown key `color`"
        );
        assert_eq!(
            msg("host=a\nhost=b\nport=1"),
            "key `host` defined twice (lines 1 and 2)"
        );
        assert_eq!(msg("host=h\nport=80x"), "invalid value \"80x\" for `port`");
    }

    #[test]
    fn source_chain_ends_at_the_std_error() {
        let e = parse("host=h\nport=80x").unwrap_err();
        assert_eq!(
            error_chain(&e),
            vec![
                "invalid value \"80x\" for `port`",
                "not a number",
                "invalid digit found in string"
            ]
        );
        let e = parse("host=h\nport=0").unwrap_err();
        assert_eq!(
            error_chain(&e),
            vec!["invalid value \"0\" for `port`", "0 is outside 1..=65535"]
        );
        let e = parse("host=h\nport=70000").unwrap_err();
        assert!(matches!(
            e,
            ConfigError::Invalid {
                source: ValueError::OutOfRange { got: 70000, .. },
                ..
            }
        ));
        let e = parse("host=h\nport=").unwrap_err();
        assert_eq!(error_chain(&e)[1], "value is empty");
        assert!(parse("host=\nport=1").is_err());
    }

    #[test]
    fn parse_all_reports_every_problem() {
        let errs = parse_all("oops\nhost=h\ncolor=red\nport=x\nretries=99").unwrap_err();
        assert_eq!(errs.len(), 4, "{errs:?}");
        assert!(matches!(errs[0], ConfigError::Syntax { line: 1, .. }));
        assert!(matches!(errs[1], ConfigError::Unknown { line: 3, .. }));
        assert!(matches!(errs[2], ConfigError::Invalid { key: "port", .. }));
        assert!(matches!(
            errs[3],
            ConfigError::Invalid { key: "retries", .. }
        ));
        // `parse` is the first of them.
        assert_eq!(
            parse("oops\nhost=h\ncolor=red\nport=x\nretries=99"),
            Err(errs.into_iter().next().unwrap())
        );
    }

    #[derive(Debug, Error)]
    enum T {
        #[error("soft")]
        Soft,
        #[error("hard")]
        Hard,
    }
    use thiserror::Error;
    impl Retryable for T {
        fn is_retryable(&self) -> bool {
            matches!(self, T::Soft)
        }
    }

    #[test]
    fn retry_succeeds_exhausts_and_stops_on_permanent() {
        let mut calls = 0;
        let ok = retry(5, |a| {
            calls += 1;
            if a < 3 { Err(T::Soft) } else { Ok(a) }
        });
        assert_eq!(ok.unwrap(), 3);
        assert_eq!(calls, 3);

        let mut calls = 0;
        let e = retry::<(), _>(4, |_| {
            calls += 1;
            Err(T::Soft)
        })
        .unwrap_err();
        assert_eq!(calls, 4);
        assert!(matches!(e, RetryError::Exhausted { attempts: 4, .. }));
        assert_eq!(error_chain(&e), vec!["gave up after 4 attempts", "soft"]);

        let mut calls = 0;
        let e = retry::<(), _>(9, |_| {
            calls += 1;
            Err(T::Hard)
        })
        .unwrap_err();
        assert_eq!(calls, 1);
        assert!(matches!(e, RetryError::Permanent { attempt: 1, .. }));

        let mut calls = 0;
        let _ = retry::<(), _>(0, |_| {
            calls += 1;
            Err(T::Soft)
        });
        assert_eq!(calls, 1, "zero attempts still runs once");
    }

    #[test]
    fn backoff_doubles_caps_and_never_overflows() {
        assert_eq!(backoff_ms(1, 100, 10_000), 100);
        assert_eq!(backoff_ms(2, 100, 10_000), 200);
        assert_eq!(backoff_ms(5, 100, 10_000), 1_600);
        assert_eq!(backoff_ms(20, 100, 10_000), 10_000);
        assert_eq!(backoff_ms(u32::MAX, u64::MAX, 7), 7);
        assert_eq!(backoff_ms(0, 100, 10_000), 100);
    }

    #[test]
    fn panics_become_errors() {
        use std::panic::AssertUnwindSafe;
        assert_eq!(boundary::catch(|| 5), Ok(5));
        let e = boundary::catch(|| -> u8 { panic!("static") }).unwrap_err();
        assert_eq!(e.message, "static");
        let n = 7;
        let e = boundary::catch(|| -> u8 { panic!("formatted {n}") }).unwrap_err();
        assert_eq!(e.to_string(), "panic: formatted 7");
        let e = boundary::catch(|| -> u8 { std::panic::panic_any(1u8) }).unwrap_err();
        assert_eq!(e.message, "non-string panic payload");
        let mut v = vec![1];
        let r = boundary::catch(AssertUnwindSafe(|| {
            v.push(2);
            panic!("half done")
        }));
        assert!(r.is_err());
        assert_eq!(
            v,
            vec![1, 2],
            "state is visible after the panic: it was mutated"
        );
    }

    #[test]
    fn app_errors_read_well_with_alternate_format() {
        let e = app::run("host=h\nport=80x", 0, false).unwrap_err();
        assert_eq!(
            format!("{e:#}"),
            "loading configuration: invalid value \"80x\" for `port`: not a number: invalid digit found in string"
        );
        // The typed error is still reachable through the context chain.
        let cfg_err = e.downcast_ref::<ConfigError>().expect("ConfigError inside");
        assert!(matches!(cfg_err, ConfigError::Invalid { key: "port", .. }));
    }

    #[test]
    fn app_retry_flows() {
        let ok = app::run(GOOD, 2, false).unwrap();
        assert_eq!(ok, "connected to example.org:8080 after 3 attempt(s)");

        // retries=1 allows two attempts; three timeouts exhaust them.
        let e = app::run("host=h\nport=1\nretries=1", 5, false).unwrap_err();
        assert_eq!(
            format!("{e:#}"),
            "connecting to h:1: gave up after 2 attempts: timed out"
        );
        let e = app::run(GOOD, 0, true).unwrap_err();
        assert_eq!(
            format!("{e:#}"),
            "connecting to example.org:8080: permanent failure on attempt 1: connection refused"
        );
        let root = e.root_cause().downcast_ref::<NetError>();
        assert_eq!(root, Some(&NetError::Refused));
    }

    #[test]
    fn auto_traits() {
        fn send_sync<T: Send + Sync + 'static>() {}
        send_sync::<ConfigError>();
        send_sync::<RetryError<NetError>>();
        send_sync::<boundary::PanicError>();
        send_sync::<anyhow::Error>();
    }
}
```

✓ With this exact file: `cargo fmt --check` clean, `cargo clippy --all-targets --locked -- -D warnings` clean, and `cargo test --locked` → 10 unit tests, 1 passing doctest and 1 `compile_fail` doctest passed. In a separate crate that depends on `errlab` I confirmed ✓ that the `compile_fail` program gives `error[E0004]: non-exhaustive patterns: `_` not covered`, with the note that `ConfigError` is marked as non-exhaustive.

Mutation checks (each applied to a copy, then `cargo test --lib`):

| Change | Result ✓ |
|---|---|
| `backoff_ms` multiplies without saturating | `backoff_doubles_caps_and_never_overflows` FAILED |
| `retry` ignores `is_retryable` (treats all errors as retryable) | 2 tests FAILED (`retry_succeeds_exhausts_and_stops_on_permanent`, `app_retry_flows`) |
| `boundary::catch` handles only `&str` payloads | `panics_become_errors` FAILED |
| `parse_all` never detects duplicate keys | `each_error_variant_has_a_useful_message` FAILED |
| `RetryError::Exhausted::last` loses its `#[source]` | 2 tests FAILED (the chain tests) |
| `ConfigError::Invalid::source` loses `#[source]` | **not caught, and cannot be**: a field *named* `source` is the source automatically, so this change is a no-op (equivalent mutant) |
| removing `max_attempts.max(1)` | **not caught, equivalent**: the first attempt always runs before the limit is checked, so the line was redundant, and I deleted it from the code and the doc comment |

**Design notes.**

- **Two layers on purpose.** `config` and `retry` expose *typed* errors for callers to match on; `app` uses `anyhow` because its callers are people. The typed error is still reachable from the `anyhow` value (✓ `downcast_ref::<ConfigError>()` in a test, and `root_cause()` reaches `NetError::Refused`).
- **`Display` per layer, chain for the rest.** `Invalid` prints `invalid value "80x" for `port``; its source `ValueError::NotANumber` prints `not a number`; *its* source is the std `ParseIntError`. Reporters print the chain (`{e:#}` or `error_chain`) instead of each layer repeating its cause.
- **Boxing the temporary error** in the private `number()` helper (`Result<u32, Box<ConfigError>>`) keeps that helper's `Result` small and avoids a clippy lint about large error variants; the public API returns `ConfigError` directly.
- **`retry` is pure control flow.** It does not sleep; `backoff_ms` computes the delay, and the caller decides how to wait (blocking thread, async sleep). No jitter or clock: it would need a random source and time (Lessons 31 and 52).
- **Known gaps.** `parse_all` is unbounded (Section 11); `boundary::catch` cannot help under `panic = "abort"`; `NetError` is simulated, not real I/O; there is no logging (Lesson 43), and the HTTP status classification of Exercise 7 is not implemented.

</details>

---

## 15. Interview questions

<details markdown="1">
<summary>Answers</summary>

1. **When should a function panic and when return `Result`?** Panic for bugs and impossible states (broken invariants); return `Result` for failures that correct programs can encounter, especially anything caused by outside input.
2. **What does a panic do?** Prints a message, then (by default) unwinds the thread's stack, running destructors, and ends the thread; `join()` returns `Err` (✓). With `panic = "abort"` the process aborts instead (✓).
3. **What happens if `Drop` panics during unwinding?** The process aborts (✓ "panic in a destructor during cleanup").
4. **What does `catch_unwind` guarantee?** That the panic stops there. It does not undo changes (✓ `v` stayed `[1, 2]`), doesn't work under `panic=abort` (✓), and needs `UnwindSafe` or an `AssertUnwindSafe` promise.
5. **Why is arithmetic overflow dangerous in Rust?** It panics only with overflow checks enabled (the debug default); release builds wrap (✓ `238`). Use `checked_*` for untrusted values.
6. **How does `?` work?** It returns early with `Err(From::from(e))` (or `None` for `Option`): the error is converted with `From` (✓ E0277 without it), and `Option`/`Result` can't be mixed (✓).
7. **What does `main` returning `Err` print?** `Error: ` and the `Debug` of the error, with exit code 1 (✓). Use `ExitCode` and your own formatting for user-facing tools.
8. **What makes a good error type?** Implements `Error`, a `Display` for its own layer, a `source()` for its cause, `From` for common causes, variants carrying identifying data, `Send + Sync + 'static` when it may cross threads (✓ asserted in a test).
9. **`thiserror` vs `anyhow`?** `thiserror` derives `Display`/`Error`/`From` for your own typed errors (libraries); `anyhow` is an erased error with context chains for application code. `anyhow::Error` is not itself an `Error` (✓ E0277).
10. **Why avoid `Box<dyn Error>` in library APIs?** Callers can't match on it; they can only print or downcast, and you lose a stable, documented set of failure modes.
11. **What does `#[non_exhaustive]` do for an error enum?** Downstream matches need a wildcard arm (✓ E0004 without it), so adding a variant is not a breaking change.
12. **What does `.context()` do and how does `{:#}` differ from `{}`?** It wraps an error with a message and keeps the cause; `{}` prints the outermost message only, `{:#}` the whole chain separated by `: ` (✓).
13. **How do you decide whether to retry?** By classifying the error as transient or permanent, only for idempotent operations, with a cap, backoff and jitter (✓ `Retryable`, `backoff_ms`).
14. **Where should errors be logged?** Once, at the boundary that handles them, with the full chain and no secrets; avoid "log and return" at every layer.
15. **How can error handling become a security problem?** Reachable `unwrap`s (DoS), wrapped overflow, leaking internals or enabling enumeration through messages, log injection, unbounded error collection, retry storms, resuming after a caught panic with corrupted state.
16. **Why does `ok_or_else` beat `ok_or` sometimes?** `ok_or(x)` evaluates `x` eagerly; with an allocating error, that cost is paid on the success path too.
17. **What is `Infallible` for?** An error type with no values: `Result<T, Infallible>` can't be `Err` (✓ `Result<u32, Infallible>` is 4 bytes), used for conversions that never fail.

</details>

---

## 16. Key takeaways

- Three tools: **panic** for bugs, **`Result`** for expected failures, **`Option`** for absence. Never use a panic as an input-error handler.
- A panic unwinds and runs destructors (✓), is isolated per thread (✓), can be caught but not rolled back (✓), aborts under `panic = "abort"` (✓), and aborts if a destructor panics while unwinding (✓).
- Overflow panics only in debug builds; release builds wrap (✓ `238`). Use `checked_*` for untrusted arithmetic.
- `?` converts errors with `From` (✓ E0277 without it) and doesn't mix `Option`/`Result` (✓). `main` returning `Err` prints `Debug` (✓): shape your own output for users.
- A good error type has a layer-specific `Display`, a `source()` chain, `From` conversions, and variants that carry the identifying data. `thiserror` writes that boilerplate for **libraries**; `anyhow` adds context for **applications** and is not an `Error` itself (✓).
- Add context at boundaries, classify retryable vs permanent, collect many errors only with a bound, log once, keep secrets out of messages, and mark public error enums `#[non_exhaustive]` (✓).

**Connections.** Lesson 14 gave you `Result` and `?`; here you design the errors behind them. Lesson 21's `dyn Any` is how payloads and `downcast_ref` work (✓), Lesson 24's poisoned mutex is a panic's after-effect, Lesson 22's `Send + Sync` explains why error types should be `Send + Sync + 'static`. Next: Lesson 27 (threads and channels) uses `JoinHandle::join` results and channel errors; Lesson 33 (API design) revisits `#[non_exhaustive]` and semver; Lesson 35 explains how `thiserror`'s derive works; Lesson 37 covers panics across FFI; Lesson 43 covers logging and tracing; Lesson 52 covers retries and timeouts against real networks. See the [roadmap](00_roadmap.md) and the [course index](index.md).

---

## 17. References

- *The Rust Programming Language*, ch. 9 "Error Handling": <https://doc.rust-lang.org/book/ch09-00-error-handling.html>
- `std::error::Error` (including `source`), `std::result`, `std::option`, `std::process::ExitCode`, `std::io::Error`: <https://doc.rust-lang.org/std/error/trait.Error.html>
- `std::panic` (`catch_unwind`, `set_hook`, `UnwindSafe`) and the profile setting `panic`: <https://doc.rust-lang.org/std/panic/index.html>, <https://doc.rust-lang.org/cargo/reference/profiles.html#panic>
- The Rustonomicon, "Unwinding": <https://doc.rust-lang.org/nomicon/unwinding.html>
- Rust API Guidelines, "Error types are meaningful and well-behaved" (C-GOOD-ERR): <https://rust-lang.github.io/api-guidelines/interoperability.html#error-types-are-meaningful-and-well-behaved-c-good-err>
- `thiserror` documentation: <https://docs.rs/thiserror/2.0.21/thiserror/>; `anyhow` documentation: <https://docs.rs/anyhow/1.0.104/anyhow/>
- Rust error code index: E0004, E0119, E0277: <https://doc.rust-lang.org/error_codes/E0277.html>

The URLs above are the standard documentation locations; I did not open them during this session (the sandbox cannot reach them), so if a link has moved, search the docs site for the page title. The crates themselves *were* downloaded from crates.io by Cargo, which is how I got the version numbers and compiled the examples. Unchecked from memory: the Book chapter number and anchors; the API-guidelines item name and URL; the statement that Display should be lowercase without trailing punctuation (guideline recalled, not re-read); that `extern "C"` panics abort (deferred to Lesson 37); the claim that `thiserror` treats a field named `source` as the source (✓ consistent with my equivalent-mutant result, but the rule is from the crate's documentation as I remember it); that `anyhow::Error` avoids implementing `Error` because of a coherence conflict (the reason is recalled, the missing impl is ✓); `io::Error::source` forwarding behavior (observed, not read in source); the cost claims for backtraces and unwinding (not measured); the exit-code numbers (my own convention); the suggestion that clippy warns about large error variants (recalled, not triggered here); and the HTTP/TLS retry classifications in Exercise 7 (design advice, not run).
