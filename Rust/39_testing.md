---
layout: default
title: "Lesson 39 — Unit tests, integration tests, and doc tests"
---

# Lesson 39 — Unit tests, integration tests, and doc tests

**Module 13 · Testing, performance and tooling** · Status: generated · Edition used in examples: **2024** · External dependencies: `proptest` (as a *dev*-dependency only)

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux, with `proptest 1.12.0`. As before, ✓ marks outputs, diagnostics and measurements I actually ran; unmarked statements are documented behavior I did not run. Everything is **stable Rust**. These tools were **not installed** here, so I did not run them and describe them from memory only: `cargo nextest`, `cargo llvm-cov`, `cargo tarpaulin`, `cargo mutants`, `cargo fuzz`, Miri (Lesson 38). I could not open the Rust documentation sites, so rules stated without ✓ are from memory; the closing paragraph lists them.

Builds on [Lesson 33](33_modules_api_design_rustdoc.md) (doc comments, `# Examples`, visibility: what is public decides what an integration test can reach), [Lesson 26](26_error_design.md) (error enums are what tests assert on), [Lesson 03](03_cargo_projects.md) (Cargo targets and `dev-dependencies`), [Lesson 38](38_no_std_embedded_miri.md) (the earlier crate's tests and the mutation method), and [Lesson 28](28_atomics_memory_ordering.md) (why shared mutable state in tests is dangerous). Lesson 40 (property testing and fuzzing) is next; it appears here only as plain text.

---

## 1. Why this lesson exists

A test suite is a program whose only job is to be wrong loudly when the code is wrong. The earlier lessons used tests constantly; this one explains how the machinery works, so you can design tests that actually catch bugs rather than tests that merely pass. Three questions drive it:

1. **Where does each kind of test live, and what can it see?** (unit tests: private items; integration tests: the public API; doc tests: the documentation's examples.)
2. **How does Cargo's harness run them?** (in parallel threads, in one process per test *binary*, with captured output). This matters because tests that share state fail in confusing ways.
3. **How do you know the tests are good?** Passing is not evidence. I measured this with **mutation checks** on the practical crate: deliberately break the code and see whether a test notices. Two real gaps turned up (§15).

---

## 2. The three kinds of tests

| Kind | Lives in | Compiled as | Can see | Run by |
|---|---|---|---|---|
| **Unit test** | `#[cfg(test)] mod tests` inside `src/` | part of your crate, with `--test` | private items of the module tree | `cargo test` (the "unittests" binary) |
| **Integration test** | `tests/*.rs` | a separate crate per file, linked against your library | **public API only** | `cargo test` (one binary per file) |
| **Doc test** | ```` ``` ```` blocks in `///` and `//!` comments | a separate tiny crate per example | public API only | `cargo test` (last), or `cargo test --doc` |

Cargo runs them in that order: unit tests, then each integration test file, then doc tests ✓. A failure in an earlier group stops the later groups unless you pass `--no-fail-fast` ✓ (without it, my failing unit tests meant the integration and doc tests did not run at all).

Rules of thumb for choosing:

- A **unit test** checks a small piece, including things you do not want to make public (a private helper, an internal invariant). It is allowed to break when you refactor internals.
- An **integration test** checks what a user of the crate can do. It should keep passing across refactors; it is your guard against accidentally changing the public behavior.
- A **doc test** is documentation that cannot rot: if an example stops compiling or its assertions fail, the build says so. It is also the first thing a user copies, so it must be small and correct.

---

## 3. Unit tests

```rust
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

fn secret() -> u32 {
    42
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn adds() {
        assert_eq!(add(1, 2), 3);
    }

    #[test]
    fn private_is_visible_here() {
        assert_eq!(secret(), 42);
    }
}
```

- `#[cfg(test)]` compiles the module only for `cargo test`, so test code and test-only imports cost nothing in the real build ✓. A helper that only tests use but is declared outside `cfg(test)` triggers `dead_code` in normal builds ✓ (I saw the warning for `secret`).
- `use super::*;` is allowed because a child module can see its parent's private items: that is exactly why unit tests can test private functions.
- A test **passes if it returns normally** and fails if it panics. The `assert!`, `assert_eq!`, `assert_ne!` and `matches!` macros panic with a message.

### 3.1 What a failure looks like ✓

```text
---- tests::prints_and_fails stdout ----
visible only on failure

thread 'tests::prints_and_fails' (255) panicked at src/lib.rs:85:9:
assertion `left == right` failed: custom message 7
  left: 2
 right: 3
```

Two things to notice. First, the harness **captures** `println!` output and shows it only for failing tests; `--nocapture` shows it always ✓. Second, the custom message (`"custom message {}", 7`) goes after the assertion text. Put the *inputs* in the message when a loop produces the cases (`assert_eq!(parse_size(input), Ok(bytes), "input {input:?}")`) so a failure names the row ✓ (the practical crate does).

### 3.2 Tests that return `Result`

```rust
#[test]
fn returns_result() -> Result<(), std::num::ParseIntError> {
    let n: i32 = "12".parse()?;
    assert_eq!(n, 12);
    Ok(())
}
```

`?` works in the test, and an `Err` fails it, printing `Error: ParseIntError { kind: InvalidDigit }` ✓. You cannot combine `Result` with `#[should_panic]`: the compiler says `functions using #[should_panic] must return ()` ✓.

### 3.3 `#[should_panic]`, `#[ignore]`

```rust
#[test]
#[should_panic(expected = "overflow")]
fn overflow_panics() {
    let _ = i32::MAX + std::hint::black_box(1);
}
```

`expected = "..."` requires the panic message to **contain** the text; without it, *any* panic passes, including one from a typo three lines earlier. When the text does not match, the test fails and shows `panic did not contain expected string` ✓. Prefer a `Result`/error-enum assertion over `should_panic` whenever the API can return an error; reserve `should_panic` for contracts that really are panics (indexing, documented preconditions).

`#[ignore = "reason"]` keeps a slow or environment-dependent test out of the default run ✓ (`ignored, slow`). Run only those with `cargo test -- --ignored`, or everything with `--include-ignored` (the latter from memory).

### 3.4 Floating point

`assert_eq!(0.1 + 0.2, 0.3)` fails ✓ with `left: 0.30000000000000004, right: 0.3`. Compare floats with an explicit tolerance: `assert!((a - b).abs() < 1e-9)`, choosing the tolerance from the problem.

---

## 4. How the harness runs your tests

`cargo test` builds each test target as an executable with a built-in harness (`libtest`) and runs it. The facts that matter:

- **Tests run in parallel**, each in its own thread, by default (the number of threads is the number of CPUs). `--test-threads=1` runs them one at a time.
- **A panic only fails that test** (the harness catches it). But a test that calls `std::process::exit` or `abort()`, or crashes with a segmentation fault, takes the whole binary with it. This matters for unsafe and FFI code (Lessons 36 and 37).
- **There is no promised order.** Order in the output is alphabetical, but with threads the execution interleaves.
- **Each test binary is a separate process.** Global state is shared between the tests *of one binary*, never across binaries.

### 4.1 Shared state makes tests order-dependent ✓

I wrote two tests that each assume a global `AtomicUsize` starts at 0 and then set it to 1:

```rust
use std::sync::atomic::{AtomicUsize, Ordering};
pub static COUNTER: AtomicUsize = AtomicUsize::new(0);

#[test]
fn a_expects_zero_then_sets() {
    assert_eq!(COUNTER.load(Ordering::SeqCst), 0, "a saw a dirty counter");
    COUNTER.store(1, Ordering::SeqCst);
}

#[test]
fn b_expects_zero_then_sets() {
    assert_eq!(COUNTER.load(Ordering::SeqCst), 0, "b saw a dirty counter");
    COUNTER.store(1, Ordering::SeqCst);
}
```

Run alone, each passes. Run together, one fails with `b saw a dirty counter` ✓. With `--test-threads=1` it still failed ✓: the problem is not parallelism but **order dependence**, which parallelism only makes random. (In my runs the failing one was `b`; the message depends on which test the scheduler ran second, so do not rely on which one it is.) The cure is never to share mutable state between tests: construct a fresh value in every test, as the practical crate does (§15). If you must touch the process-wide environment (current directory, environment variables, a fixed port, a fixed file path), serialize those tests with a lock or run them in a binary of their own, and use unique temporary paths otherwise. Mutating environment variables is `unsafe` in edition 2024 for exactly this reason (from memory; Lesson 36).

### 4.2 Selecting tests ✓

| Command | Effect |
|---|---|
| `cargo test` | everything: unit, integration, doc |
| `cargo test adds` | tests whose full name contains `adds` (substring match) |
| `cargo test -- --exact tests::adds` | exactly that name |
| `cargo test -- --skip fails --skip prints` | all except names containing these |
| `cargo test -- --list` | print the names, run nothing |
| `cargo test --lib` / `--test bucket` / `--doc` | only the unit tests / one integration file / only doc tests |
| `cargo test -- --ignored` | only `#[ignore]` tests |
| `cargo test -- --nocapture` | show `println!` output of passing tests |
| `cargo test -- --test-threads=1` | one at a time |
| `cargo test --no-fail-fast` | keep going after a failing target |
| `cargo test --no-run` | compile only (prints each `Executable`) |

Everything after `--` goes to the test binary; everything before goes to Cargo. Filtering by a name like `adds` printed `1 passed; ... 7 filtered out` ✓.

---

## 5. Integration tests

Every `.rs` file directly under `tests/` is compiled as **its own crate** that depends on yours, so it sees only your public API ✓ (trying to call a private function there is an error). In `testlab` the files are `tests/size.rs`, `tests/bucket.rs` and `tests/props.rs`; Cargo ran each as a separate executable ✓.

**Shared helpers.** If you put a helper in `tests/common.rs`, Cargo treats it as a test crate and reports `running 0 tests` for it. Put it in **`tests/common/mod.rs`** and write `mod common;` in each test file that needs it ✓ (Cargo only treats files directly in `tests/` as test crates). Every file that declares `mod common;` gets its own copy, and unused helpers produce `dead_code` warnings in the files that do not use them; that is normal.

**Binary crates** (`src/main.rs` only) cannot be imported by integration tests. The usual structure is a thin `main.rs` that calls a library crate (`src/lib.rs`) holding the logic; test the library. For the executable itself, run the built binary as a subprocess with `std::process::Command` and the `CARGO_BIN_EXE_<name>` environment variable that Cargo sets for integration tests (from memory; I did not try it here).

Slow tests that need real resources (network, database, files) belong here, usually behind `#[ignore]` or a feature flag, and use **unique temporary directories** so parallel tests do not collide.

---

## 6. Doc tests

Rustdoc extracts every fenced code block in doc comments, wraps it in a `fn main() { ... }` (unless you wrote your own `main`), compiles it as a **separate crate that depends on yours**, and runs it ✓:

```rust,ignore
/// Adds two numbers.
///
/// ```
/// # let hidden = 10; // a line starting with `# ` is compiled but not shown in the docs
/// assert_eq!(ex::add(hidden, 1), 11);
/// ```
pub fn add(a: i32, b: i32) -> i32 { a + b }
```

Consequences:

- Doc tests see only the **public API** and must `use` what they need (the crate's own name is available).
- A hidden line (`# ` prefix) keeps the example short while still compiling. Use it for setup, not to hide the interesting part.
- Dev-dependencies are available to doc tests (from memory).
- A doc test that uses `?` needs a return type. Without one it fails ✓:

```text
error[E0277]: the `?` operator can only be used in a function that returns `Result` or `Option` ...
```

  The idiom is to end with a hidden line that gives the implicit `main` a `Result` type ✓ (shown here as the doc comment you would write):

```rust,ignore
/// ```
/// let n: i32 = "5".parse()?;
/// assert_eq!(n, 5);
/// # Ok::<(), std::num::ParseIntError>(())
/// ```
pub fn demo() {}
```

### 6.1 Attributes on the fence ✓

| Fence | Meaning |
|---|---|
| ```` ``` ```` | compile and run; must succeed |
| ```` ```no_run ```` | compile, do not run (a `loop {}` or a network call) |
| ```` ```should_panic ```` | compile and run; must panic |
| ```` ```compile_fail ```` | must **fail to compile** (add the error code to be strict: ```` ```compile_fail,E0308 ````) |
| ```` ```ignore ```` | skipped (shown as `ignored`) |
| ```` ```text ```` | not Rust at all (a diagram, output) |

I ran an example of each ✓: output listed `... - compile ... ok` for `no_run`, `... - should panic ... ok`, `... - compile fail ... ok`, and `ignored` for the `ignore` block. Notice that rustdoc compiled the passing examples together in one group and the `compile_fail` and `ignore` ones in another (two `test result:` lines under `Doc-tests`) ✓; I mention it because a `compile_fail` block costs a separate compilation, and a few of them add real time. A `compile_fail` doctest is the right tool for the "this must not compile" guarantees Rust is built on: `testlab` uses one to pin that `ManualClock` is not `Sync` ✓ (`E0277`). For the dedicated, more flexible tool (`trybuild`) see Lesson 35.

`ignore` rots silently, because nothing compiles the block. Prefer `no_run` or a hidden-line setup.

---

## 7. Designing code to be testable

Most hard-to-test code is hard because it **reaches out**: it reads the clock, the environment, the network, or a random number generator directly. The fix is the same each time: **inject the dependency** behind a trait or a parameter, and pass a fake in tests.

`testlab`'s `TokenBucket<C: Clock>` takes its time source as a type parameter. Production code passes a real clock; tests pass `ManualClock`, which only moves when told. The effect:

- No `thread::sleep` in tests. Sleeping makes tests slow and **flaky** (a loaded CI machine sleeps longer than you expect). The whole bucket suite ran in well under a second ✓.
- You can test things real time makes unreachable: a clock that jumps **backwards**, a clock at `u64::MAX`, an elapsed time of 2^63 ms ✓ (all in the crate's tests).

Other techniques, all from memory unless the crate shows them:

- **Pure core, thin shell.** Put the logic in functions that take data and return data; keep I/O at the edges, where a handful of integration tests cover it.
- **Take `impl Read`/`impl Write`** instead of a file path; tests pass a `&[u8]` or a `Vec<u8>`.
- **Traits with fakes** (hand-written, usually 10 lines) beat mocking frameworks for most code. A mocking crate (`mockall`) helps when a trait is big, at the price of macros and compile time.
- **Seed randomness** and print the seed on failure, so a failing run can be replayed.
- **Don't test the framework.** A test that re-implements the function under test is worthless (it has the same bug); assert against **independent** facts: known vectors, hand-computed tables, an oracle that is obviously correct, or an invariant.

---

## 8. Property-based testing

A normal test checks examples you thought of. A **property test** states a rule that must hold for *all* inputs, generates many random inputs, and, on failure, **shrinks** the failing input to a minimal one. `proptest` is the common crate (add it under `[dev-dependencies]`):

```rust,ignore
use proptest::prelude::*;

proptest! {
    #[test]
    fn format_then_parse_is_the_identity(n in any::<u64>()) {
        prop_assert_eq!(parse_size(&format_size(n)), Ok(n));
    }

    #[test]
    fn parse_never_panics_on_arbitrary_text(s in ".*") {
        let _ = parse_size(&s);
    }
}
```

Good properties, in rough order of how often they find bugs:

1. **Round trip**: `decode(encode(x)) == x`.
2. **Never panics / never crashes** on arbitrary input (for parsers and decoders; this is what finds missing bounds checks).
3. **Invariants** after any sequence of operations (the bucket never grants more than `capacity + rate * elapsed`; its `available()` never exceeds the capacity ✓ in `props.rs`).
4. **Oracle**: compare to a slow, obviously correct implementation (or to the standard library).
5. **Exactness of hints**: "if it says retry after `t` ms, then `t - 1` fails and `t` succeeds" ✓.

What a failure looks like ✓ (this came from a mutation: I changed `"KB"` to 1,024 and `proptest` found it):

```text
Test failed: assertion failed: `(left == right)`
  left: `Ok(1024)`,
 right: `Ok(1000)` at tests/props.rs:27.
minimal failing input: n = 1, unit = 1
```

It shrank a random failing case to `n = 1, unit = 1`: the smallest number with the unit `KB`. Observations:

- By default it runs 256 cases per property (from memory; I did not count) and honors the `PROPTEST_CASES` environment variable: `PROPTEST_CASES=2000` ran the six properties in 0.22 s ✓.
- It saves failing cases to a `proptest-regressions/` file next to the test and replays them first next time; commit that file. In integration tests it printed a warning that it `failed to find lib.rs or main.rs` for the default persistence location ✓ and still saved to `tests/props.proptest-regressions`; configuring persistence explicitly (`ProptestConfig`) avoids the warning (from memory).
- Properties find *classes* of bugs, not all bugs; they are a complement to the example tests, not a replacement. The format-then-parse property alone would not have caught `format_size` choosing the wrong unit (a mutant I ran: both functions stayed mutually consistent) ✓; an explicit table did.
- Randomized tests must stay **reproducible**: never seed from the clock in a way you cannot replay.

Fuzzing (`cargo fuzz`, from memory) goes further for parsers and unsafe code: coverage-guided mutation of inputs, run for minutes or hours rather than milliseconds. It is not installed here.

---

## 9. Testing special cases

- **Panics and `unsafe`.** `std::panic::catch_unwind` lets a test check that something panics and continue ✓ (it worked in my experiment). For unsafe code, run the tests under Miri (Lesson 38) and valgrind or sanitizers, and add **drop-counting** tests with a type whose `Drop` increments a counter (Lesson 37's `array_from_fn_lite`).
- **Concurrency.** Tests that "work on my machine" are the norm; schedule-dependent bugs need tools (`loom`, from memory) or stress loops. Never use `sleep` to wait for another thread; wait on a channel, a `join`, or a condition.
- **Async.** Use the runtime's test macro (`#[tokio::test]`, Lesson 31) or a paused clock; same advice: do not sleep for real.
- **Error values.** Assert on the **variant** (`assert_eq!(parse_size("5kb"), Err(SizeError::UnknownUnit))`), not on `to_string()`, except in one test that pins the message text if users see it ✓.
- **Boundaries.** Test `0`, `1`, `MAX`, `MAX - 1`, `MAX + 1`, empty, one element, and the exact off-by-one around every limit. Most real parser and counter bugs live exactly there ✓ (the u64-boundary and 2^63 tests below exist because of this).
- **Snapshot tests** (`insta`, from memory) store expected output in a file; good for big structured output, bad if reviewers approve snapshots without reading them.

---

## 10. Performance, reliability, security

- **Speed.** Tests are code you run constantly. Keep unit tests under milliseconds; `cargo test` compiles each integration file as its own crate and links it, so *many* files cost link time (from memory the common remedy is a single `tests/it/main.rs` with modules; I did not measure it). `cargo nextest` (from memory, not installed) runs each test in its own process, which isolates global state and is often faster on big workspaces.
- **Flakiness is a bug.** A test that fails 1 time in 100 teaches people to ignore red builds. Causes: real time, shared state, ports, ordering of hash-map iteration, thread scheduling. Remove the cause; do not add retries.
- **Security-relevant tests.** For anything that parses untrusted input, test: truncation at every length, a length field pointing past the end, huge counts, invalid UTF-8, and arithmetic overflow at the extremes (`"18446744073709551616"` ✓). Property tests with arbitrary text assert "no panic"; fuzzing assesses deeper.
- **Do not put secrets in tests**, and do not call the real network in unit tests.
- **Coverage ≠ quality.** A line can be executed without any assertion about its result. I could not run a coverage tool here; instead I used mutation checks (§15), which measure whether the tests *notice* a change.

---

## 11. Common mistakes

1. Tests that depend on each other or on global state (✓ the counter example).
2. `#[should_panic]` without `expected`, so any panic passes.
3. `thread::sleep` to "wait for" something; use fakes, channels or joins.
4. Re-computing the expected value with the same logic as the code.
5. Asserting only the happy path; forgetting the error variants and boundaries.
6. `assert_eq!` on floats without a tolerance (✓).
7. Putting shared helpers in `tests/common.rs` (it becomes an empty test crate) instead of `tests/common/mod.rs`.
8. Doc tests with `?` and no return type (✓ `E0277`); `ignore`d doc tests that rot.
9. Testing private details in integration tests by making things `pub` "for tests"; use a unit test instead.
10. Treating a green suite as proof; surviving mutants (✓ M11 below) show what it missed.
11. Huge doc examples; keep them to the point and hide setup with `# `.
12. Flaky randomized tests that cannot be replayed (always keep the seed or the regression file).

---

## 12. Exercises

Answers are collapsed. Try each before opening it.

1. **(Warm-up)** Name the three places tests can live, and say for each what it can see.
2. **(Warm-up)** Why does `tests/common.rs` produce a test run with zero tests, and what is the fix?
3. **(Easy)** Write a unit test module for `fn clamp_percent(x: i32) -> i32` (returns 0 for negatives, 100 for above 100) covering the boundaries.
4. **(Medium)** Write a doc test for `fn first_word(s: &str) -> Option<&str>` that uses a hidden line and demonstrates the `None` case.
5. **(Medium)** `#[should_panic]` test passes even though the function panics for the wrong reason. How do you tighten it?
6. **(Medium-hard)** Refactor `fn is_expired(created: SystemTime) -> bool` (it calls `SystemTime::now()`) so it can be tested without sleeping.
7. **(Hard)** State a round-trip property and a "never panics" property for a function pair `encode(&[u8]) -> String` / `decode(&str) -> Result<Vec<u8>, E>`, and say what each would catch.
8. **(Hard)** Your suite passes and coverage is 100%. A mutation (`>=` changed to `>`) survives. What does that tell you, and what do you add?

<details markdown="1">
<summary>Answers</summary>

1. Unit tests (`#[cfg(test)] mod tests` in `src/`): private and public items of the crate. Integration tests (`tests/*.rs`): only the public API, each file is its own crate. Doc tests (in `///` and `//!`): only the public API, each example is its own tiny crate that depends on yours.
2. Cargo treats every `.rs` file directly under `tests/` as a test crate, and `common.rs` has no `#[test]` functions, so it reports 0 tests. Put the helpers in `tests/common/mod.rs` (a subdirectory is not a test target) and declare `mod common;` in the tests that use them ✓.
3. Test both sides of each boundary:

   ```rust
   pub fn clamp_percent(x: i32) -> i32 {
       x.clamp(0, 100)
   }

   #[cfg(test)]
   mod tests {
       use super::*;

       #[test]
       fn boundaries() {
           let cases = [
               (i32::MIN, 0),
               (-1, 0),
               (0, 0),
               (1, 1),
               (99, 99),
               (100, 100),
               (101, 100),
               (i32::MAX, 100),
           ];
           for (input, expected) in cases {
               assert_eq!(clamp_percent(input), expected, "input {input}");
           }
       }
   }
   ```
4. Document it with a runnable example:

   ````rust,ignore
   /// Returns the first whitespace-separated word.
   ///
   /// ```
   /// # use ex::first_word;
   /// assert_eq!(first_word("hello world"), Some("hello"));
   /// assert_eq!(first_word("   "), None);
   /// ```
   pub fn first_word(s: &str) -> Option<&str> {
       s.split_whitespace().next()
   }
   ````

   (In a real crate the hidden line would be `# use your_crate::first_word;`. The doctest runs only if the function is public.)
5. Add `expected = "..."` with a fragment of the *intended* message, so a different panic (an unrelated `unwrap` on `None`, an index error) fails the test ✓ (`panic did not contain expected string`). Better, if the API can return an error, assert on the `Err` variant instead.
6. Inject the time source, and keep a thin wrapper that uses the real clock:

   ```rust
   use std::time::{Duration, SystemTime};

   pub fn is_expired_at(created: SystemTime, now: SystemTime, ttl: Duration) -> bool {
       now.duration_since(created).is_ok_and(|age| age > ttl)
   }

   pub fn is_expired(created: SystemTime, ttl: Duration) -> bool {
       is_expired_at(created, SystemTime::now(), ttl)
   }

   #[cfg(test)]
   mod tests {
       use super::*;

       #[test]
       fn expiry_boundaries() {
           let t0 = SystemTime::UNIX_EPOCH + Duration::from_secs(1000);
           let ttl = Duration::from_secs(60);
           assert!(!is_expired_at(t0, t0, ttl));
           assert!(!is_expired_at(t0, t0 + ttl, ttl), "exactly the ttl is not expired");
           assert!(is_expired_at(t0, t0 + ttl + Duration::from_nanos(1), ttl));
           assert!(!is_expired_at(t0, t0 - Duration::from_secs(5), ttl), "clock went backwards");
       }
   }
   ```

   A `Clock` trait, as in `testlab`, is the same idea for a long-lived object.
7. Round trip: `decode(&encode(&bytes)) == Ok(bytes)` for every `Vec<u8>`; it catches asymmetric bugs (padding, off-by-one on lengths, endianness) but not a bug that is symmetric in both functions (both wrong the same way). Never panics: `decode(&arbitrary_string)` must return `Ok` or `Err`, never panic; it catches missing bounds checks, `unwrap`s, and overflow on malformed input. Add known vectors (hand-computed or from a specification) to catch the symmetric bugs.
8. The suite executes the line but never checks the behavior exactly at the boundary, so a wrong comparison goes unnoticed (✓ the same pattern as M9 in this lesson's table). Coverage measures executed lines, not asserted behavior. Add tests for the values on both sides of the boundary (`limit - 1`, `limit`, `limit + 1`) and assert the results; then re-run the mutation.

</details>

---

## 13. Debugging challenge

A team adds two tests for a global request counter. Each passes when run alone with `cargo test one_request` or `cargo test two_requests`, but `cargo test` fails intermittently and always with a message about the *wrong count*.

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

static REQUESTS: AtomicUsize = AtomicUsize::new(0);

pub fn handle_request() -> usize {
    REQUESTS.fetch_add(1, Ordering::SeqCst) + 1
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn one_request() {
        assert_eq!(handle_request(), 1);
    }

    #[test]
    fn two_requests() {
        assert_eq!(handle_request(), 1);
        assert_eq!(handle_request(), 2);
    }
}
```

✓ The same shape (two tests each assuming a shared global starts at zero) failed with `b saw a dirty counter` when run together and passed individually, and kept failing with `--test-threads=1`. Explain, and fix.

<details markdown="1">
<summary>Solution</summary>

Both tests use the same process-wide `static`. The harness runs all tests of a binary in **one process**, so the second test to run sees the count the first one left behind. With threads the order is random (so the failure is intermittent); with `--test-threads=1` the order is fixed but the dependence is still there (✓ it still failed). Passing individually proves nothing about independence.

Fix: make the state a *value* owned by each test, not a global:

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

pub struct Counter {
    requests: AtomicUsize,
}

impl Counter {
    pub const fn new() -> Counter {
        Counter { requests: AtomicUsize::new(0) }
    }

    pub fn handle_request(&self) -> usize {
        self.requests.fetch_add(1, Ordering::SeqCst) + 1
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn one_request() {
        assert_eq!(Counter::new().handle_request(), 1);
    }

    #[test]
    fn two_requests() {
        let c = Counter::new();
        assert_eq!(c.handle_request(), 1);
        assert_eq!(c.handle_request(), 2);
    }
}
```

The production program can still keep one `static COUNTER: Counter = Counter::new();` (the constructor is `const`); only the tests stop sharing it. If the global is unavoidable (an environment variable, a process-wide registry), serialize the tests that touch it with a `static Mutex<()>` lock, or move them into their own integration-test file, which gets its own process.

</details>

---

## 14. Interview questions

1. *Where do unit, integration and doc tests live, and what can each access?*
2. *How does Cargo run tests, and why can shared global state cause failures?*
3. *How do you share helper code between integration tests?*
4. *How do you test code that depends on the current time or on randomness?*
5. *What does `#[should_panic(expected = ...)]` add, and when would you avoid `should_panic` entirely?*
6. *What are the doc-test fence attributes, and what happens to a doc test that uses `?`?*
7. *What is property-based testing and what is shrinking? Give two properties for a parser.*
8. *The suite is green and coverage is high. How do you check that the tests are any good?*

<details markdown="1">
<summary>Answers</summary>

1. Unit tests: `#[cfg(test)] mod` inside `src/`, can access private items. Integration tests: files in `tests/`, each its own crate, public API only. Doc tests: code blocks in doc comments, each compiled as a separate crate, public API only ✓.
2. Each test binary is one process; its tests run on parallel threads with output captured. They share statics, the environment and the file system, so tests that mutate them become order-dependent (✓ the counter example failed even with one thread). The remedy is per-test state, injected dependencies, or serialization.
3. Put them in `tests/common/mod.rs` and add `mod common;` to each test file; a file `tests/common.rs` would be run as a test crate with zero tests ✓.
4. Inject the dependency: a `Clock` trait (✓ `ManualClock`) or a time parameter, or a seeded RNG whose seed is printed on failure. Never sleep.
5. `expected` requires the panic message to contain given text, so a panic for a different reason fails the test ✓. Avoid `should_panic` when the API can return an error: assert on the `Err` variant; also `should_panic` cannot be combined with a `Result`-returning test ✓.
6. ```` ``` ```` (run), `no_run`, `should_panic`, `compile_fail[,Exxxx]`, `ignore`, `text` ✓. A doc test with `?` fails with `E0277` unless it ends with a hidden line such as `# Ok::<(), SomeError>(())` ✓.
7. A property test checks a rule against many generated inputs; on failure it shrinks the input to a minimal counterexample (✓ `n = 1, unit = 1`). For a parser: `parse(&format(x)) == x` (round trip), and `parse(&arbitrary_string)` never panics.
8. Mutation testing: change the code in small ways (flip a comparison, remove a call) and check that some test fails. Surviving mutants are either equivalent changes or real gaps (✓ I found a gap with `wrapping_mul` instead of `saturating_mul`, and added a test). Also review boundaries, run property tests and, for unsafe code, Miri/valgrind.

</details>

---

## 15. Practical challenge: `testlab`

Build a library crate `testlab` with two components and test them at every level. Requirements:

1. `size`: `parse_size(&str) -> Result<u64, SizeError>` for `"512"`, `"10KiB"`, `"2 MB"` (units `B KB KiB MB MiB GB GiB`, case-sensitive, spaces allowed between number and unit), with errors for empty input, bad number, unknown unit and overflow (no panics, exact `u64` boundaries); `format_size(u64) -> String` using the largest exact binary unit.
2. `bucket`: a token bucket generic over a `Clock` trait, with `ManualClock` for tests; `try_acquire(n)` returns `Denied::{TooLarge, Never, RetryAfterMs(t)}` with an **exact** retry hint; refill capped at capacity; a clock going backwards earns nothing; no overflow for huge elapsed times.
3. Unit tests for private helpers; integration tests with a shared `tests/common/mod.rs`; doc tests (including a `compile_fail` one); property tests with `proptest`.
4. `#![deny(missing_docs)]`; fmt, clippy `-D warnings`, rustdoc `-D warnings`.
5. Mutation checks on copies; report survivors honestly and fix gaps.

<details markdown="1">
<summary>Solution (spoilers: full source)</summary>

`Cargo.toml`:

```toml
[package]
name = "testlab"
version = "0.1.0"
edition = "2024"

[dependencies]

[dev-dependencies]
proptest = "1"
```

`src/lib.rs`:

```rust
//! `testlab`: two small components written to be tested at every level.
//!
//! * [`size`]: parse and format byte sizes such as `"10KiB"`.
//! * [`bucket`]: a token-bucket rate limiter driven by an injectable [`Clock`],
//!   so tests control time instead of sleeping.
//!
//! ```
//! use testlab::{format_size, parse_size};
//! assert_eq!(parse_size("2 KiB"), Ok(2048));
//! assert_eq!(format_size(3 * 1024 * 1024), "3MiB");
//! ```

#![deny(missing_docs)]

pub mod bucket;
pub mod size;

pub use bucket::{Clock, Denied, ManualClock, TokenBucket};
pub use size::{SizeError, format_size, parse_size};
```

`src/size.rs`:

```rust
//! Parsing and formatting of byte sizes.

use core::fmt;

/// Why [`parse_size`] failed.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[non_exhaustive]
pub enum SizeError {
    /// The input is empty or has no digits.
    Empty,
    /// The number part is missing or not a decimal number.
    BadNumber,
    /// The unit is not one of `B KB KiB MB MiB GB GiB`.
    UnknownUnit,
    /// The result does not fit in a `u64`.
    Overflow,
}

impl fmt::Display for SizeError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.write_str(match self {
            SizeError::Empty => "empty size",
            SizeError::BadNumber => "invalid number",
            SizeError::UnknownUnit => "unknown unit",
            SizeError::Overflow => "size does not fit in 64 bits",
        })
    }
}

impl std::error::Error for SizeError {}

/// Bytes per unit for the supported unit names (case-sensitive).
fn unit_factor(unit: &str) -> Option<u64> {
    Some(match unit {
        "" | "B" => 1,
        "KB" => 1_000,
        "KiB" => 1 << 10,
        "MB" => 1_000_000,
        "MiB" => 1 << 20,
        "GB" => 1_000_000_000,
        "GiB" => 1 << 30,
        _ => return None,
    })
}

/// Parses a size such as `"512"`, `"10KiB"` or `"2 MB"` into bytes.
///
/// A bare number means bytes. Spaces between the number and the unit are
/// allowed; leading and trailing whitespace is not.
///
/// ```
/// use testlab::{SizeError, parse_size};
/// assert_eq!(parse_size("512"), Ok(512));
/// assert_eq!(parse_size("10KiB"), Ok(10 * 1024));
/// assert_eq!(parse_size("2 MB"), Ok(2_000_000));
/// assert_eq!(parse_size("5 parsecs"), Err(SizeError::UnknownUnit));
/// ```
pub fn parse_size(input: &str) -> Result<u64, SizeError> {
    if input.is_empty() {
        return Err(SizeError::Empty);
    }
    let digits_end = input
        .find(|c: char| !c.is_ascii_digit())
        .unwrap_or(input.len());
    let (number, rest) = input.split_at(digits_end);
    if number.is_empty() {
        return Err(SizeError::BadNumber);
    }
    let n: u64 = number.parse().map_err(|_| SizeError::Overflow)?;
    let unit = rest.trim_start_matches(' ');
    let factor = unit_factor(unit).ok_or(SizeError::UnknownUnit)?;
    n.checked_mul(factor).ok_or(SizeError::Overflow)
}

/// Formats `bytes` using the largest binary unit that divides it exactly.
///
/// ```
/// use testlab::format_size;
/// assert_eq!(format_size(0), "0B");
/// assert_eq!(format_size(1536), "1536B"); // not a whole KiB
/// assert_eq!(format_size(2048), "2KiB");
/// ```
pub fn format_size(bytes: u64) -> String {
    for (shift, unit) in [(30, "GiB"), (20, "MiB"), (10, "KiB")] {
        let factor = 1u64 << shift;
        if bytes != 0 && bytes.is_multiple_of(factor) {
            return format!("{}{unit}", bytes / factor);
        }
    }
    format!("{bytes}B")
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn unit_factor_knows_every_unit_and_nothing_else() {
        let table = [
            ("", 1),
            ("B", 1),
            ("KB", 1_000),
            ("KiB", 1_024),
            ("MB", 1_000_000),
            ("MiB", 1_048_576),
            ("GB", 1_000_000_000),
            ("GiB", 1_073_741_824),
        ];
        for (name, factor) in table {
            assert_eq!(unit_factor(name), Some(factor), "unit {name:?}");
        }
        for bad in ["kb", "KIB", "b", "TB", " B", "B "] {
            assert_eq!(unit_factor(bad), None, "unit {bad:?}");
        }
    }
}
```

`src/bucket.rs`:

```rust
//! A token-bucket rate limiter with an injectable clock.

use core::cell::Cell;
use core::fmt;

/// A source of monotonic milliseconds.
pub trait Clock {
    /// Milliseconds since an arbitrary fixed origin.
    fn now_ms(&self) -> u64;
}

impl<C: Clock + ?Sized> Clock for &C {
    fn now_ms(&self) -> u64 {
        (**self).now_ms()
    }
}

/// A clock that only moves when told to; for tests and simulations.
///
/// It is not `Sync` (it uses a [`Cell`]):
///
/// ```compile_fail,E0277
/// fn needs_sync<T: Sync>(_: &T) {}
/// needs_sync(&testlab::ManualClock::new(0));
/// ```
#[derive(Debug)]
pub struct ManualClock {
    now: Cell<u64>,
}

impl ManualClock {
    /// Creates a clock reading `start` milliseconds.
    pub fn new(start: u64) -> ManualClock {
        ManualClock {
            now: Cell::new(start),
        }
    }

    /// Moves the clock forward (saturating at `u64::MAX`).
    pub fn advance(&self, ms: u64) {
        self.now.set(self.now.get().saturating_add(ms));
    }

    /// Sets the clock to an absolute reading, even a smaller one.
    pub fn set(&self, ms: u64) {
        self.now.set(ms);
    }
}

impl Clock for ManualClock {
    fn now_ms(&self) -> u64 {
        self.now.get()
    }
}

/// Why a request was refused.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[non_exhaustive]
pub enum Denied {
    /// More tokens were requested than the bucket can ever hold.
    TooLarge,
    /// The bucket does not refill, so the request can never succeed.
    Never,
    /// Not enough tokens yet; retry after this many milliseconds.
    RetryAfterMs(u64),
}

impl fmt::Display for Denied {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Denied::TooLarge => f.write_str("request exceeds bucket capacity"),
            Denied::Never => f.write_str("bucket never refills"),
            Denied::RetryAfterMs(ms) => write!(f, "retry after {ms} ms"),
        }
    }
}

impl std::error::Error for Denied {}

/// Milli-tokens per token: tokens are tracked in thousandths so that a rate
/// of `r` tokens per second adds exactly `r` milli-tokens per millisecond.
const MILLI: u64 = 1_000;

/// A token bucket holding up to `capacity` tokens, refilled at
/// `per_second` tokens per second.
///
/// ```
/// use testlab::{Denied, ManualClock, TokenBucket};
///
/// let clock = ManualClock::new(0);
/// let mut bucket = TokenBucket::new(&clock, 3, 1); // 3 tokens, +1 per second
/// assert_eq!(bucket.try_acquire(3), Ok(()));
/// assert_eq!(bucket.try_acquire(1), Err(Denied::RetryAfterMs(1000)));
/// clock.advance(1000);
/// assert_eq!(bucket.try_acquire(1), Ok(()));
/// ```
#[derive(Debug)]
pub struct TokenBucket<C: Clock> {
    clock: C,
    capacity: u32,
    per_second: u32,
    /// Current tokens in thousandths. Invariant: `<= capacity * MILLI`.
    milli_tokens: u64,
    /// Clock reading at the last refill.
    last_ms: u64,
}

impl<C: Clock> TokenBucket<C> {
    /// Creates a full bucket.
    pub fn new(clock: C, capacity: u32, per_second: u32) -> TokenBucket<C> {
        let last_ms = clock.now_ms();
        TokenBucket {
            clock,
            capacity,
            per_second,
            milli_tokens: u64::from(capacity) * MILLI,
            last_ms,
        }
    }

    /// Adds the tokens earned since the last refill. A clock that moved
    /// backwards earns nothing and does not rewind the reference point.
    fn refill(&mut self) {
        let now = self.clock.now_ms();
        if now <= self.last_ms {
            return;
        }
        let elapsed = now - self.last_ms;
        let earned = elapsed.saturating_mul(u64::from(self.per_second));
        let max = u64::from(self.capacity) * MILLI;
        self.milli_tokens = self.milli_tokens.saturating_add(earned).min(max);
        self.last_ms = now;
    }

    /// Whole tokens currently available (after refilling).
    pub fn available(&mut self) -> u32 {
        self.refill();
        (self.milli_tokens / MILLI) as u32
    }

    /// Takes `n` tokens if available.
    pub fn try_acquire(&mut self, n: u32) -> Result<(), Denied> {
        if n > self.capacity {
            return Err(Denied::TooLarge);
        }
        self.refill();
        let need = u64::from(n) * MILLI;
        if self.milli_tokens >= need {
            self.milli_tokens -= need;
            return Ok(());
        }
        if self.per_second == 0 {
            return Err(Denied::Never);
        }
        let missing = need - self.milli_tokens;
        let rate = u64::from(self.per_second);
        Err(Denied::RetryAfterMs(missing.div_ceil(rate)))
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn refill_never_exceeds_capacity_even_with_huge_elapsed_time() {
        let clock = ManualClock::new(0);
        let mut b = TokenBucket::new(&clock, 5, u32::MAX);
        b.try_acquire(5).unwrap();
        clock.set(u64::MAX);
        b.refill();
        assert_eq!(b.milli_tokens, 5 * MILLI);
    }

    #[test]
    fn refill_does_not_wrap_when_elapsed_times_rate_overflows() {
        // 2^63 ms * 2 tokens/s is exactly 2^64 milli-tokens: a wrapping
        // multiplication would give 0 and the bucket would stay empty.
        let clock = ManualClock::new(0);
        let mut b = TokenBucket::new(&clock, 5, 2);
        b.try_acquire(5).unwrap();
        clock.set(1 << 63);
        b.refill();
        assert_eq!(b.milli_tokens, 5 * MILLI);
    }

    #[test]
    fn backwards_clock_does_not_rewind_the_reference_point() {
        let clock = ManualClock::new(1000);
        let mut b = TokenBucket::new(&clock, 2, 1);
        b.try_acquire(2).unwrap();
        clock.set(500);
        b.refill();
        assert_eq!(b.last_ms, 1000);
        clock.set(1500);
        b.refill();
        assert_eq!(b.milli_tokens, 500);
    }
}
```

`tests/common/mod.rs`:

```rust
//! Helpers shared by the integration tests. Living in `common/mod.rs` (not
//! `common.rs`) stops Cargo from treating it as a test crate of its own.

use testlab::{ManualClock, TokenBucket};

/// A bucket of capacity 10 refilling 2 tokens per second, on a clock at 0.
pub fn standard_bucket(clock: &ManualClock) -> TokenBucket<&ManualClock> {
    TokenBucket::new(clock, 10, 2)
}
```

`tests/size.rs`:

```rust
use testlab::{SizeError, format_size, parse_size};

#[test]
fn parses_every_unit() {
    let cases = [
        ("0", 0),
        ("7", 7),
        ("7B", 7),
        ("1KB", 1_000),
        ("1KiB", 1_024),
        ("1MB", 1_000_000),
        ("1MiB", 1_048_576),
        ("1GB", 1_000_000_000),
        ("1GiB", 1_073_741_824),
        ("3  KiB", 3_072),
        ("007", 7),
    ];
    for (input, bytes) in cases {
        assert_eq!(parse_size(input), Ok(bytes), "input {input:?}");
    }
}

#[test]
fn rejects_bad_input_with_the_right_error() {
    let cases = [
        ("", SizeError::Empty),
        ("KiB", SizeError::BadNumber),
        (" 5", SizeError::BadNumber),
        ("-5", SizeError::BadNumber),
        ("5kb", SizeError::UnknownUnit),
        ("5 TB", SizeError::UnknownUnit),
        ("5B ", SizeError::UnknownUnit),
        ("5\tKiB", SizeError::UnknownUnit),
        ("5.5KB", SizeError::UnknownUnit),
        ("99999999999999999999", SizeError::Overflow),
        ("18446744073709551615GB", SizeError::Overflow),
        ("17179869184GiB", SizeError::Overflow),
    ];
    for (input, err) in cases {
        assert_eq!(parse_size(input), Err(err), "input {input:?}");
    }
}

#[test]
fn boundaries_of_u64_are_exact() {
    assert_eq!(parse_size("18446744073709551615"), Ok(u64::MAX));
    assert_eq!(parse_size("18446744073709551616"), Err(SizeError::Overflow));
    assert_eq!(parse_size("17179869183GiB"), Ok(17_179_869_183 * (1 << 30)));
}

#[test]
fn formats_with_the_largest_exact_binary_unit() {
    let cases = [
        (0, "0B"),
        (1, "1B"),
        (1023, "1023B"),
        (1024, "1KiB"),
        (1536, "1536B"),
        (2048, "2KiB"),
        (1 << 20, "1MiB"),
        (3 << 20, "3MiB"),
        ((1 << 20) + 1024, "1025KiB"),
        (1 << 30, "1GiB"),
        (5 << 30, "5GiB"),
        (u64::MAX, "18446744073709551615B"),
    ];
    for (bytes, text) in cases {
        assert_eq!(format_size(bytes), text);
    }
}

#[test]
fn error_messages_are_stable() {
    assert_eq!(SizeError::Empty.to_string(), "empty size");
    assert_eq!(SizeError::BadNumber.to_string(), "invalid number");
    assert_eq!(SizeError::UnknownUnit.to_string(), "unknown unit");
    assert_eq!(
        SizeError::Overflow.to_string(),
        "size does not fit in 64 bits"
    );
}
```

`tests/bucket.rs`:

```rust
mod common;

use common::standard_bucket;
use testlab::{Denied, ManualClock, TokenBucket};

#[test]
fn starts_full_and_drains() {
    let clock = ManualClock::new(0);
    let mut b = standard_bucket(&clock);
    assert_eq!(b.available(), 10);
    assert_eq!(b.try_acquire(4), Ok(()));
    assert_eq!(b.available(), 6);
    assert_eq!(b.try_acquire(6), Ok(()));
    assert_eq!(b.available(), 0);
}

#[test]
fn reports_exact_retry_delay_rounded_up() {
    let clock = ManualClock::new(0);
    let mut b = standard_bucket(&clock); // +2 tokens per second = 1 per 500 ms
    b.try_acquire(10).unwrap();
    assert_eq!(b.try_acquire(1), Err(Denied::RetryAfterMs(500)));
    clock.advance(499);
    assert_eq!(b.try_acquire(1), Err(Denied::RetryAfterMs(1)));
    clock.advance(1);
    assert_eq!(b.try_acquire(1), Ok(()));
    assert_eq!(b.try_acquire(3), Err(Denied::RetryAfterMs(1500)));
}

#[test]
fn refill_is_capped_at_capacity() {
    let clock = ManualClock::new(0);
    let mut b = standard_bucket(&clock);
    b.try_acquire(10).unwrap();
    clock.advance(1_000_000);
    assert_eq!(b.available(), 10);
}

#[test]
fn fractional_tokens_accumulate_across_calls() {
    let clock = ManualClock::new(0);
    let mut b = standard_bucket(&clock);
    b.try_acquire(10).unwrap();
    for _ in 0..4 {
        clock.advance(250); // half a token each time
        let _ = b.available();
    }
    assert_eq!(b.available(), 2, "4 x 250 ms at 2 tokens/s = 2 tokens");
}

#[test]
fn too_large_requests_are_refused_without_consuming() {
    let clock = ManualClock::new(0);
    let mut b = standard_bucket(&clock);
    assert_eq!(b.try_acquire(11), Err(Denied::TooLarge));
    assert_eq!(b.available(), 10);
    assert_eq!(b.try_acquire(10), Ok(()), "exactly the capacity is allowed");
}

#[test]
fn zero_token_request_always_succeeds() {
    let clock = ManualClock::new(0);
    let mut b = TokenBucket::new(&clock, 0, 0);
    assert_eq!(b.try_acquire(0), Ok(()));
    assert_eq!(b.try_acquire(1), Err(Denied::TooLarge));
}

#[test]
fn a_bucket_that_never_refills_says_so() {
    let clock = ManualClock::new(0);
    let mut b = TokenBucket::new(&clock, 2, 0);
    assert_eq!(b.try_acquire(2), Ok(()));
    assert_eq!(b.try_acquire(1), Err(Denied::Never));
    clock.advance(1_000_000);
    assert_eq!(b.try_acquire(1), Err(Denied::Never));
}

#[test]
fn a_clock_that_goes_backwards_earns_nothing() {
    let clock = ManualClock::new(10_000);
    let mut b = standard_bucket(&clock);
    b.try_acquire(10).unwrap();
    clock.set(0);
    assert_eq!(b.available(), 0);
    clock.set(10_500);
    assert_eq!(
        b.available(),
        1,
        "only the 500 ms since the old reference point count"
    );
}

#[test]
fn works_with_an_owned_clock_and_a_custom_clock() {
    struct Fixed(u64);
    impl testlab::Clock for Fixed {
        fn now_ms(&self) -> u64 {
            self.0
        }
    }
    let mut b = TokenBucket::new(Fixed(5), 1, 1);
    assert_eq!(b.try_acquire(1), Ok(()));
    assert_eq!(b.try_acquire(1), Err(Denied::RetryAfterMs(1000)));
}

#[test]
fn denied_messages() {
    assert_eq!(
        Denied::TooLarge.to_string(),
        "request exceeds bucket capacity"
    );
    assert_eq!(Denied::Never.to_string(), "bucket never refills");
    assert_eq!(Denied::RetryAfterMs(7).to_string(), "retry after 7 ms");
}

#[test]
fn manual_clock_saturates() {
    let c = ManualClock::new(u64::MAX - 1);
    c.advance(10);
    assert_eq!(testlab::Clock::now_ms(&c), u64::MAX);
}
```

`tests/props.rs`:

```rust
//! Property tests: assertions that must hold for *every* input, checked on
//! many generated inputs. A failing case is shrunk to a minimal one.

use proptest::prelude::*;
use testlab::{Denied, ManualClock, TokenBucket, format_size, parse_size};

proptest! {
    #[test]
    fn format_then_parse_is_the_identity(n in any::<u64>()) {
        prop_assert_eq!(parse_size(&format_size(n)), Ok(n));
    }

    #[test]
    fn parse_never_panics_on_arbitrary_text(s in ".*") {
        let _ = parse_size(&s);
    }

    #[test]
    fn plain_decimal_numbers_parse_to_themselves(n in any::<u64>()) {
        prop_assert_eq!(parse_size(&n.to_string()), Ok(n));
    }

    #[test]
    fn multiplying_by_a_unit_matches_checked_math(n in 0u64..1_000_000, unit in 0usize..7) {
        let (name, factor) = [("B", 1u64), ("KB", 1_000), ("KiB", 1_024), ("MB", 1_000_000),
                              ("MiB", 1_048_576), ("GB", 1_000_000_000), ("GiB", 1_073_741_824)][unit];
        prop_assert_eq!(parse_size(&format!("{n}{name}")), Ok(n * factor));
    }

    /// Over any schedule of requests and time steps, the bucket never hands
    /// out more than `capacity + rate * elapsed_seconds` tokens.
    #[test]
    fn never_grants_more_than_capacity_plus_refill(
        capacity in 0u32..20,
        rate in 0u32..10,
        steps in proptest::collection::vec((0u64..2_000, 0u32..25), 0..60),
    ) {
        let clock = ManualClock::new(0);
        let mut bucket = TokenBucket::new(&clock, capacity, rate);
        let mut granted: u64 = 0;
        for (advance_ms, want) in steps {
            clock.advance(advance_ms);
            if bucket.try_acquire(want).is_ok() {
                granted += u64::from(want);
            }
            let elapsed = testlab::Clock::now_ms(&clock);
            let limit_milli = u64::from(capacity) * 1000 + elapsed * u64::from(rate);
            prop_assert!(granted * 1000 <= limit_milli, "granted {granted}, limit {limit_milli}/1000");
            prop_assert!(bucket.available() <= capacity);
        }
    }

    /// If the bucket says "retry after t ms", waiting exactly t ms must succeed
    /// and waiting one millisecond less must not.
    #[test]
    fn retry_hint_is_exact(capacity in 1u32..20, rate in 1u32..10, drain in 0u32..20, want in 1u32..20) {
        let clock = ManualClock::new(0);
        let mut bucket = TokenBucket::new(&clock, capacity, rate);
        let _ = bucket.try_acquire(drain.min(capacity));
        if let Err(Denied::RetryAfterMs(t)) = bucket.try_acquire(want) {
            prop_assert!(t >= 1);
            clock.advance(t - 1);
            prop_assert!(bucket.try_acquire(want).is_err(), "succeeded early at t-1");
            clock.advance(1);
            prop_assert_eq!(bucket.try_acquire(want), Ok(()));
        }
    }
}
```

**Verification ✓.** `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, `cargo test --locked` and `RUSTDOCFLAGS="-D warnings" cargo doc --no-deps` pass: 4 unit tests, 11 bucket tests, 5 size tests, 6 property tests, and 5 doc tests (4 runnable examples plus one `compile_fail,E0277`). The six properties also passed with `PROPTEST_CASES=2000` ✓. **Not run:** coverage tools, `cargo nextest`, `cargo mutants`, fuzzing, Miri, other targets. The crate is safe Rust only.

**Mutation checks ✓** (each applied to a copy with its own target directory; "caught" means at least one test target reported `FAILED`):

| # | Mutation | Result |
|---|---|---|
| M1 | `"KB"` factor 1,000 → 1,024 | **caught** by the unit table, `tests/size.rs`, and the property test (shrunk to `n = 1, unit = 1`) |
| M2 | `checked_mul` → `wrapping_mul` in `parse_size` | **caught** (overflow cases) |
| M3 | `format_size` tries `KiB` before `GiB` | **caught** by the explicit table; the round-trip property alone would still pass (both directions stay consistent) |
| M4 | the empty-number check removed | **caught** (`"KiB"` must be `BadNumber`) |
| M5 | `trim_start_matches(' ')` → `trim_start()` (tabs accepted) | **caught**, but only *after* I added the `"5\tKiB"` row. My first attempt at this mutant had a typo in its pattern, so I never ran it against the original tests; I added the row because I reasoned that `trim_start` is broader, and the mutant then failed it. I cannot claim the original suite would have missed it |
| M6 | refill guard `now <= last` → `now < last` | **survived**: **equivalent mutant**. With `now == last` the elapsed time is 0, so nothing is earned and `last_ms` keeps its value |
| M7 | refill no longer capped at capacity | **caught** |
| M8 | `n > capacity` → `n >= capacity` | **caught** |
| M9 | `milli_tokens >= need` → `>` | **caught** (the exact-fit boundary tests) |
| M10 | `div_ceil` → truncating `/` in the retry hint | **caught** |
| M11 | `saturating_mul` → `wrapping_mul` for `elapsed * rate` | **survived my first suite** (real gap: the one test with a huge elapsed time wrapped to another large number and was capped anyway). I added `refill_does_not_wrap_when_elapsed_times_rate_overflows` (2^63 ms at 2 tokens/s wraps to exactly 0), then the mutant was **caught** |
| M12 | `last_ms` not updated after a refill | **caught** |
| M13 | the `Never` branch removed | **caught** (it would divide by zero) |
| M14 | `available()` returns the fractional part (`%` instead of `/`) | **caught** |
| M15 | `ManualClock::advance` wraps instead of saturating | **caught** |
| M16 | initial `last_ms` fixed to 0 instead of the clock's reading | **survived**: **equivalent mutant**. The bucket starts full, so the first refill caps at the same value whichever reference point it uses |
| M17 | refill ignores the rate (`earned = elapsed`) | **caught** |
| M18 | acquiring empties the bucket (`= 0` instead of `-= need`) | **caught** |

Totals: 18 mutants. 16 are caught by the final suite (M5 and M11 only after I added a test); 2 survive and are equivalent mutants (M6, M16). Of the first-run survivors, only M11 was a real test gap, and I closed it. Lessons: (a) the explicit table found what the round-trip property missed (M3); (b) the property test found, in one run, what an unlucky example set could miss (M1); (c) a "huge input" test must be chosen so the wrong implementation gives a *different* answer, not just a large one (M11).

**Design notes.**

- Time is a **type parameter**, not a global, so tests never sleep and can make time jump backwards or to `u64::MAX`. `impl<C: Clock + ?Sized> Clock for &C` lets a bucket borrow the clock the test keeps controlling.
- Tokens are counted in thousandths, so a rate of `r` tokens per second adds exactly `r` milli-tokens per millisecond: no floating point, exact retry hints (`missing.div_ceil(rate)`).
- `parse_size` finds the digit run, parses it (an overflowing run is `Overflow`), and multiplies with `checked_mul`; it never indexes, so no input can panic ✓ (also a property).
- The unit tests in `src/` cover only what integration tests cannot reach (`unit_factor`, the private `refill` and its fields).
- Not handled: fractional sizes (`"1.5KiB"`), thread-safe sharing of a bucket (`ManualClock` uses a `Cell`; a real shared limiter needs atomics or a lock), long-running clocks near `u64::MAX` milliseconds beyond the saturation tests, fuzzing, and a real `SystemClock` (omitted to keep the crate dependency- and OS-free).

</details>

---

## 16. Key takeaways

- Three homes for tests: unit (private access), integration (public API, one crate per file), doc tests (examples that cannot rot). `cargo test` runs them in that order and stops at the first failing group unless `--no-fail-fast` (✓).
- The harness runs tests in parallel threads in one process per binary; shared state makes tests order-dependent (✓ failed even with one thread). Give every test its own state.
- Shared integration helpers go in `tests/common/mod.rs` (✓). Doc tests are separate crates: hide setup with `# `, give `?` a return type, and use `no_run`, `should_panic`, `compile_fail` deliberately (✓).
- Make code testable by injecting dependencies (a `Clock`), not by sleeping or exposing internals.
- Assert exact behavior at boundaries; assert on error variants; use independent expected values.
- Property tests state rules over all inputs and shrink failures (✓ `n = 1, unit = 1`); they complement, not replace, example tests.
- A green suite is not proof. Mutation checks found a real gap (✓ M11) and an equivalent pair (M6, M16); coverage numbers would not have.

**Connections.** [Lesson 33](33_modules_api_design_rustdoc.md) gave `# Examples` and visibility, which decide what doc and integration tests can see; [Lesson 26](26_error_design.md)'s error enums are what these tests assert on; [Lesson 28](28_atomics_memory_ordering.md) explains why the global counter in the debugging challenge is a data-race-free but order-dependent hazard; [Lesson 35](35_proc_macros.md) introduced `trybuild` for `compile_fail`-style testing of macros; [Lessons 36 to 38](36_unsafe_raw_pointers.md) used drop counting, valgrind and mutation checks for `unsafe` code. Next: Lesson 40 goes deeper on property testing and fuzzing, the techniques that generate the inputs you did not think of.

---

## 17. References

- The Rust Book, chapter 11 "Writing Automated Tests" (unit, integration, `should_panic`, `ignore`, test organization): <https://doc.rust-lang.org/book/>
- The Cargo Book, "Cargo Targets" and "cargo test" (test targets, `--no-fail-fast`, `--doc`, the `tests/` directory layout): <https://doc.rust-lang.org/cargo/>
- The rustdoc book, "Documentation tests" (hidden lines, `no_run`, `should_panic`, `compile_fail`, `ignore`, `?` in doc tests): <https://doc.rust-lang.org/rustdoc/write-documentation/documentation-tests.html>
- `std::test` and the `libtest` command-line options (`--list`, `--exact`, `--skip`, `--nocapture`, `--test-threads`, `--ignored`): run `cargo test -- --help`.
- The `proptest` book and crate documentation (strategies, shrinking, `ProptestConfig`, regression files): <https://proptest-rs.github.io/proptest/>
- Rust API Guidelines, "Documentation" checklist (examples for every public item): <https://rust-lang.github.io/api-guidelines/>
- `cargo-nextest` (not run): <https://nexte.st/>; `cargo-llvm-cov` (not run): <https://github.com/taiki-e/cargo-llvm-cov>; `cargo-mutants` (not run): <https://mutants.rs/>; `cargo-fuzz` (not run): <https://rust-fuzz.github.io/book/>

The URLs above are the standard documentation locations; I did not open them during this session (the sandbox cannot reach the documentation sites), so if a link has moved, search for the page title. Unchecked or from memory: that unit and integration tests are built with `--test` and `libtest` (I only observed the behavior); the default of 256 proptest cases (I observed `PROPTEST_CASES` but did not count cases); that doc tests can use dev-dependencies; `CARGO_BIN_EXE_<name>`; `--include-ignored`; that mutating environment variables is `unsafe` in edition 2024; the claim that many integration-test files cost extra link time and the single-`it/main.rs` remedy (not measured); how `cargo nextest` runs tests; every statement about `mockall`, `insta`, `loom`, `trybuild`'s details, `cargo fuzz`, `cargo mutants`, coverage tools (none installed); and how `proptest` persists failures by default beyond the single warning I saw. Exercise answers 4, 6 and 7 compile but are not part of the crate; I compiled their code blocks separately (see the verification list in my working notes), and the reasoning in answers 7 and 8 is from memory. Everything marked ✓ was run on this machine's `rustc 1.97.0` with `proptest 1.12.0`.
