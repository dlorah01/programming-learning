---
layout: default
title: "Lesson 02 — Toolchain: rustup, rustc, Cargo, Clippy, rustfmt"
---

# Lesson 02 — Toolchain: rustup, rustc, Cargo, Clippy, rustfmt

**Module 1 · Foundations and toolchain** · Status: generated · Edition used in examples: **2024** · Dependencies: none (`std` only)

Verified while writing this lesson with: `rustup 1.28.2`, `rustc 1.97.0 (2d8144b78 2026-07-07)` (LLVM 22.1.6), `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux. Commands marked ✓ were actually run; unmarked commands are documented behavior I did not run here (they need network access or modify the installation). Cargo project examples need edition 2024, so Rust **1.85.0 or newer**.

Builds on [Lesson 01](01_foundations.md): editions, stable vs nightly, debug vs release overflow behavior.

---

## 1. The toolchain map

| Tool | Role | Lives where |
|---|---|---|
| **rustup** | Toolchain manager: installs and switches compilers, components, targets | Outside any toolchain |
| **rustc** | The compiler (Lesson 01 pipeline) | Toolchain |
| **cargo** | Build system, package manager, test/doc runner, front end to everything below | Toolchain |
| **rust-std** | The standard library, **one build per target** | Toolchain component |
| **clippy** | Linter (`cargo clippy`) | Toolchain component |
| **rustfmt** | Code formatter (`cargo fmt`) | Toolchain component |
| **rustdoc** | Documentation generator (`cargo doc`, Lesson 33) | Toolchain |
| **rust-analyzer** | Editor language server; usually installed via your editor | Separate |

**Why a separate manager?** Rust ships a new stable release every six weeks (Lesson 01) and supports many compilation targets. rustup makes it cheap to hold several compiler versions side by side, add targets such as WebAssembly, and pin a project to a version. Cargo and the crates.io registry predate the 1.0 release, so the ecosystem had one standard build and package workflow from the start, which is why almost every Rust project looks alike on disk.

Check what you have (✓):

```text
$ rustup --version        # rustup 1.28.2 (e4f3ad6f8 2025-04-28)
$ rustup show
Default host: x86_64-unknown-linux-gnu
installed toolchains
  stable-x86_64-unknown-linux-gnu (active, default)
$ rustup component list --installed
cargo-x86_64-unknown-linux-gnu
clippy-x86_64-unknown-linux-gnu
rust-std-x86_64-unknown-linux-gnu
rustc-x86_64-unknown-linux-gnu
rustfmt-x86_64-unknown-linux-gnu
```

A **toolchain** is a named bundle: a channel or version plus the host triple, e.g. `stable-x86_64-unknown-linux-gnu`. A **component** is a piece of it. A **profile** is a preset group of components: per the rustup documentation, `minimal` (rustc, rust-std, cargo), `default` (minimal plus rust-docs, rustfmt, clippy; what rustup uses unless you change it), and `complete` (everything; documented as something to avoid). The machine I used has a hand-picked set, so yours may also list `rust-docs`.

---

## 2. rustup in practice

| Task | Command |
|---|---|
| Update installed toolchains | `rustup update` |
| Install a specific version / channel | `rustup toolchain install 1.85.0` · `rustup toolchain install nightly` |
| Change the global default | `rustup default stable` |
| Use a toolchain for one command | `cargo +nightly build` · `cargo +1.85.0 test` |
| Per-directory override (stored by rustup) | `rustup override set 1.85.0` |
| Add components | `rustup component add clippy rustfmt` |
| Add a compilation target | `rustup target add wasm32-unknown-unknown` |
| Show the active toolchain and why | `rustup show` ✓ · `rustup show active-toolchain` ✓ |
| Run any command under a toolchain | `rustup run nightly rustc --version` |

### 2.1 Which toolchain is used? (precedence)

From the rustup book, highest priority first:

1. `+toolchain` on the command line (`cargo +beta …`)
2. the `RUSTUP_TOOLCHAIN` environment variable
3. a directory override (`rustup override set`)
4. a **`rust-toolchain.toml`** file (found by searching up the directory tree)
5. the default toolchain (`rustup default`)

`rustup show` tells you *why* a toolchain is active. Observed (✓) with a `rust-toolchain.toml` present:

```text
active toolchain
name: stable-x86_64-unknown-linux-gnu
active because: overridden by '.../hello_cargo/rust-toolchain.toml'
```

### 2.2 Target triples

A target is named `arch-vendor-os[-env]`; our host is `x86_64-unknown-linux-gnu` (x86-64 CPU, unknown vendor, Linux, GNU C library). Compiling for another target needs its `rust-std` (`rustup target add`) and usually a linker for it. This compiler knows **322** targets (✓ `rustc --print target-list | wc -l`); support levels (Tier 1, 2, 3) are listed in the rustc book's platform-support page and decide what is tested and guaranteed to build. Cross-compilation details return in Lessons 37 and 47.

---

## 3. rustc used directly

You met `rustc` in Lesson 01. Cargo calls it for you, but you need to read its options. Verified to exist on 1.97.0 (✓ `rustc --help`, `rustc -C help`):

| Option | Meaning |
|---|---|
| `--edition 2024` | Language edition (bare `rustc` defaults to 2015) |
| `-O` | Same as `-C opt-level=3` |
| `-C opt-level=0..3,s,z` | Optimization level; `s`/`z` favor size |
| `-g` / `-C debuginfo=` | Debug information (`-g` = `-C debuginfo=2`) |
| `-C overflow-checks=on\|off` | Integer overflow checks (Lesson 01) |
| `-C debug-assertions=on\|off` | Enables `debug_assert!` and `cfg(debug_assertions)` |
| `-C panic=unwind\|abort` | Panic strategy |
| `-C lto=` | Link-time optimization |
| `-C codegen-units=N` | Parallelism vs optimization trade-off |
| `-C strip=none\|debuginfo\|symbols` | Strip symbols from the output |
| `-C target-cpu=…` | Tune for a CPU; `native` produces binaries that may not run on other machines |
| `--crate-type bin\|lib\|rlib\|dylib\|cdylib\|staticlib\|proc-macro` | What to build |
| `--emit mir\|llvm-ir\|asm\|…` | Intermediate outputs (Lesson 01) |
| `--print cfg\|sysroot\|target-list\|…` | Print compiler information |
| `--target <triple>` | Cross-compile |
| `--test` | Build a test harness |
| `--cfg name` | Set a conditional-compilation flag |
| `-W` / `-A` / `-D` / `-F <lint>` | Warn / allow / deny / forbid a lint |
| `--cap-lints <level>` | Cap lint levels (Cargo uses this for dependencies) |
| `--explain E0382` | Long explanation of an error code ✓ |

Useful observations (✓):

```text
$ rustc -vV
rustc 1.97.0 (2d8144b78 2026-07-07)
binary: rustc
commit-hash: 2d8144b7880597b6e6d3dfd63a9a9efae3f533d3
commit-date: 2026-07-07
host: x86_64-unknown-linux-gnu
release: 1.97.0
LLVM version: 22.1.6

$ rustc --print cfg           # excerpt of the default (unoptimized) build
debug_assertions
panic="unwind"
target_arch="x86_64"
target_endian="little"
target_env="gnu"
target_family="unix"
```

`debug_assertions` is present in the default build and **absent with `-O`** (✓ I counted it in both outputs). That is the same switch behind the debug-versus-release behavior you saw in Lesson 01.

---

## 4. Cargo essentials

Cargo is the tool you will actually type. It reads `Cargo.toml`, resolves dependencies, calls `rustc` with the right flags, and caches results.

### 4.1 A new project (✓)

```text
$ cargo new hello_cargo
    Creating binary (application) `hello_cargo` package
$ cd hello_cargo
$ cargo run
   Compiling hello_cargo v0.1.0 (/path/to/hello_cargo)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.27s   # your timing will differ
     Running `target/debug/hello_cargo`
Hello, world!
```

Generated files:

| Path | Purpose |
|---|---|
| `Cargo.toml` | The **manifest**: package metadata, edition, dependencies, profiles, features |
| `src/main.rs` | Entry point of a **binary** crate (`cargo new --lib` creates `src/lib.rs` instead) |
| `.gitignore` | Contains `/target` |
| `.git/` | `cargo new` initializes a Git repository (with no commits) when you are not already inside one |
| `Cargo.lock` | Appears after the first build: the exact resolved dependency versions |
| `target/` | All build output; **never commit it** |

The generated manifest (✓):

```toml
[package]
name = "hello_cargo"
version = "0.1.0"
edition = "2024"

[dependencies]
```

`Cargo.lock` begins with a header marking it as generated and "not intended for manual editing" (✓). Rule of thumb: commit it for applications so builds are reproducible; the policy for libraries is discussed in Lesson 03.

### 4.2 The commands you will use daily

| Command | What it does |
|---|---|
| `cargo check` ✓ | Parse, expand macros, type-check and borrow-check, then **stop before code generation**. Fastest feedback loop |
| `cargo build` ✓ | Compile in the `dev` profile into `target/debug/` |
| `cargo build --release` ✓ | Compile in the `release` profile into `target/release/` |
| `cargo run` ✓ | Build, then run the binary |
| `cargo test` ✓ | Build and run tests (Lesson 39) |
| `cargo clippy` ✓ | Lint (Section 6) |
| `cargo fmt` ✓ | Format (Section 7) |
| `cargo fix` ✓ | Apply compiler-suggested fixes; migrate editions (Section 9) |
| `cargo doc` | Build documentation (Lesson 33) |
| `cargo clean` | Delete `target/` |
| `cargo --list` ✓ | List every available subcommand and alias (`b`, `c`, `d`, …) |

**`check` vs `build`.** `cargo check` skips code generation, the most expensive phase, so use it constantly while editing. One caveat: a small class of errors surfaces only after monomorphization or during code generation, so `check` passing does not strictly guarantee `build` passing. Treat it as the fast first gate, not the last.

### 4.3 Dev and release profiles

Cargo's two built-in profiles (documented defaults, from the Cargo book):

| Setting | `dev` | `release` |
|---|---|---|
| `opt-level` | 0 | 3 |
| `debug` | true (full debug info) | false |
| `debug-assertions` | true | false |
| `overflow-checks` | **true** | **false** |
| `incremental` | true | false |
| `codegen-units` | 256 | 16 |
| `lto` | false | false |
| `panic` | `unwind` | `unwind` |

This table is the *source* of Lesson 01's overflow behavior: `dev` checks overflow, `release` does not. Cargo's own banner confirms the mapping (✓): ``Finished `dev` profile [unoptimized + debuginfo]`` versus ``Finished `release` profile [optimized]``.

Observed binary sizes for the same tiny program on this machine (✓): **4,409,800 bytes** (`target/debug`) versus **445,128 bytes** (`target/release`). Neither was stripped. This is a size observation for one trivial program on one platform, **not** a performance measurement; do not generalize the ratio.

You can customize profiles in `Cargo.toml` (`[profile.release]`), covered in Lesson 03.

---

## 5. How to read compiler diagnostics

Real output from the Lesson 01 move example (✓, `rustc --edition 2024`):

```text
error[E0382]: borrow of moved value: `s`
 --> ex03_move_bad.rs:4:16
  |
2 |     let s = String::from("hi");
  |         - move occurs because `s` has type `String`, which does not implement the `Copy` trait
3 |     let t = s; // ownership of the heap buffer moves to `t`
  |             - value moved here
4 |     println!("{s} {t}"); // error: `s` was moved
  |                ^ value borrowed here after move
  |
help: consider cloning the value if the performance cost is acceptable
  |
3 |     let t = s.clone(); // ownership of the heap buffer moves to `t`
  |              ++++++++

error: aborting due to 1 previous error

For more information about this error, try `rustc --explain E0382`.
```

Anatomy:

- **Header**: `level[code]: message`. Levels: `error`, `warning`, `note`, `help`.
- **`-->`**: file, line, column of the primary location.
- **Labels**: `^` marks the primary span; `-` marks secondary spans that *tell the story* (where it was declared, where it moved, where it was used).
- **`help:`**: a suggested fix, shown as a diff (`++++++++` = inserted text).
- **Footer**: `rustc --explain E0382` prints a long explanation with examples.

How to work with them:

1. Fix the **first** error first; later ones often cascade from it.
2. Read the labels, not just the headline: they reconstruct the sequence of events.
3. Treat `help:` as a hint, not a verdict. Here it suggests `.clone()`, with the honest qualifier "if the performance cost is acceptable". Reflexively cloning to satisfy the borrow checker hides design questions (Lesson 01, common mistake 2).
4. Use `rustc --explain <code>` when a code is new to you.

**Lint levels.** Lints are `allow`, `warn`, `deny` or `forbid` (cannot be overridden later). Warnings do not stop a build (✓, an unused variable produced `warning: unused variable` and the build still finished). Enforce cleanliness in **CI** (`-D warnings`) rather than with `#![deny(warnings)]` in source: a newer compiler can add lints, and a hard-coded deny would then break anyone building your crate with that compiler.

---

## 6. Clippy

Clippy is a collection of lints for common mistakes, non-idiomatic code, and performance traps. It runs the compiler front end, so `cargo clippy` also reports ordinary rustc warnings.

Observed (✓) across the small programs used while writing this lesson. In each case plain `cargo build` produced **no** corresponding warning:

| Lint | Message | Why it matters |
|---|---|---|
| `needless_range_loop` | the loop variable `i` is only used to index `values` | Prefer iterators; they cannot go out of bounds and often optimize better |
| `len_zero` | length comparison to zero | `is_empty()` states intent and can be cheaper for some collections |
| `clone_on_copy` | using `clone` on type `u8` which implements `Copy` | Copying is implicit for `Copy` types |
| `useless_vec` | useless use of `vec!` | A fixed-size array avoids a heap allocation |
| `const_is_empty` | this expression always evaluates to false | `is_empty()` on a fixed-size array is a constant; the check is dead code |

The debugging challenge in Section 15 triggers the first two; the others came from variants I tried while verifying it. That is itself a useful lesson: Clippy reacts to *how* code is shaped, so a small restructuring can add or remove a lint.

**Groups and default levels** (Clippy documentation): `correctness` is deny by default; `suspicious`, `style`, `complexity` and `perf` warn; `pedantic`, `restriction`, `nursery` and `cargo` are allow by default. The docs advise against enabling `restriction` wholesale because those lints can conflict with each other; pick individual ones.

**Configuring** (documented):

```rust
#![warn(clippy::pedantic)]          // enable a group for the crate

#[allow(clippy::too_many_lines)]    // silence one lint on one item
fn long_function() { /* ... */ }
```

```toml
# Cargo.toml (requires Rust 1.74+)
[lints.clippy]
pedantic = "warn"
unwrap_used = "deny"
```

Lint thresholds go in `clippy.toml`. Prefer `#[expect(lint, reason = "...")]` (stable since Rust 1.81) over `#[allow]` when you must suppress something: `expect` itself warns if the suppression becomes unnecessary, so stale suppressions do not accumulate.

**CI form** (✓ exit code observed): `cargo clippy --all-targets -- -D warnings` exits **non-zero (101)** when any lint fires. Clippy is advice, not truth: it has false positives, and `pedantic` lints are deliberately opinionated.

---

## 7. rustfmt

`cargo fmt` rewrites code to the standard style, ending formatting debates. `cargo fmt --check` changes nothing and fails if the code is not formatted (✓ exit code **1**), printing a diff:

```text
Diff in src/main.rs:5:
         total += v[i];
     }
-    if v.len() == 0 { println!("empty"); }
+    if v.len() == 0 {
+        println!("empty");
+    }
```

Configuration lives in `rustfmt.toml` or `.rustfmt.toml` at the project root, e.g. `max_width = 80` (the default width is 100, ✓ `rustfmt --print-config default`). Notes:

- Some options are **unstable** and honored only by a nightly rustfmt; stick to stable options for portable projects.
- The default style is codified in the *Rust Style Guide*; changing it makes your code diff noisily against everyone else's.
- Run it on every commit (editor on-save or a pre-commit hook) and enforce it in CI with `--check`.

---

## 8. Pinning the toolchain: `rust-toolchain.toml`

Place it at the project root so everyone (and CI) uses the same compiler:

```toml
[toolchain]
channel = "stable"
components = ["clippy", "rustfmt"]
```

Supported keys (rustup book): `channel` (e.g. `stable`, `beta`, `nightly`, a version like `1.85.0`, or a dated release), `profile` (`minimal`, `default`, `complete`), `components`, `targets`, and `path` (a custom toolchain, mutually exclusive with `channel`). Observed (✓): with this file present, `rustup show` reported the toolchain as "overridden by '…/rust-toolchain.toml'".

**Failure mode (✓).** A typo in `channel` is treated as a *custom toolchain name*:

```toml
[toolchain]
channel = "stabel"
```

```text
error: custom toolchain 'stabel' specified in override file '…/rust-toolchain.toml' is not installed
```

Read it carefully: the message is about a toolchain that does not exist, not about a missing install of stable.

| Choice | Benefit | Cost |
|---|---|---|
| `channel = "stable"` | Always current; security and compiler fixes arrive automatically | New releases add lints and may change diagnostics; a `-D warnings` CI can start failing without any code change |
| `channel = "1.97.0"` (exact) | Reproducible builds and CI | You must bump it deliberately, and you do not get compiler fixes until you do |

**Two different version settings; do not confuse them:**

- `rust-toolchain.toml` says *which compiler to run for this repository*.
- `rust-version = "1.85"` in `Cargo.toml` (the **MSRV**, minimum supported Rust version) says *which compiler versions your crate promises to support*. Cargo checks it and refuses to build on an older compiler. Libraries should set and test it; Lesson 03 covers the details.

---

## 9. Migrating editions: `cargo fix --edition`

Workflow, with what I observed (✓) when migrating a fresh edition-2021 project:

```text
$ cargo fix --edition --allow-no-vcs
   Migrating Cargo.toml from 2021 edition to 2024
    Checking mig v0.1.0 (...)
   Migrating src/main.rs from 2021 edition to 2024
    Finished `dev` profile [unoptimized + debuginfo] target(s) ...
```

**Important observation (✓):** after this command, `Cargo.toml` **still said `edition = "2021"`**. The command rewrites *source code* so it means the same thing under the new edition; **you** then change the edition by hand:

1. Optionally run `cargo update` and test first: the Edition Guide notes that some proc-macros and build-time code generators can misbehave with a newer edition until updated.
2. Commit your work. In a repository with uncommitted changes, Cargo refuses (✓): ``error: the working directory of this package has uncommitted changes, and `cargo fix` can potentially perform destructive changes``. You can override it with `--allow-dirty` (or `--allow-no-vcs` outside version control), but a clean commit makes the migration diff reviewable. After committing, the same command ran fine (✓).
3. `cargo fix --edition`
4. Edit `Cargo.toml`: `edition = "2024"`
5. `cargo check`, `cargo test`, then `cargo fmt` (the new edition can change formatting rules).
6. Review the diff; anything the tool cannot fix is left as a warning for a human to resolve.

---

## 10. A local quality gate (preview of CI)

Four stages, ordered cheapest and fastest first so you fail early:

```text
cargo fmt --check                             # style; no compilation
cargo clippy --all-targets -- -D warnings     # compiler front end + lints, tests included
cargo test                                    # behavior
cargo build --release                         # the optimized build actually compiles
```

Lesson 03 adds features and workspaces (`--all-features`, `--workspace`); Lesson 43 turns this into real CI with dependency auditing.

---

## 11. Common mistakes and edge cases

1. **Using `cargo build` for every edit.** Use `cargo check`; build only when you need a binary.
2. **Judging performance or binary size from `target/debug`.** Debug builds do not optimize (Lesson 01).
3. **Forgetting `--release`** before running a timing or load test.
4. **Committing `target/`.** `cargo new` already ignores it.
5. **Silencing everything**: `#![allow(warnings)]` or `#![allow(clippy::all)]` at the crate root throws away the tool's value. Suppress one lint, on one item, with a reason.
6. **`-C target-cpu=native` for software you distribute.** The binary may crash with an illegal instruction on older CPUs.
7. **Making `nightly` your global default** "to be safe". Stable is the default for good reasons; use `cargo +nightly` for the exceptions (Lesson 01).
8. **`rustup update` mid-project** without checking whether your CI and `rust-version` agree.
9. **Assuming `cargo check` proves the build works** (Section 4.2 caveat).
10. **Typos in `rust-toolchain.toml`** (Section 8).
11. **Forcing `cargo fix --allow-dirty` on a dirty tree**, losing track of which edits were yours.

---

## 12. Performance implications

- `cargo check` skips code generation, the most expensive phase, so it should give faster feedback than `build`; I recorded no timings, so measure it on your own project.
- `incremental = true` in `dev` recompiles only what changed; it is `false` in `release`, which favors the best optimization over rebuild speed.
- `codegen-units = 256` in `dev` maximizes parallelism and rebuild speed, while `16` in `release` gives the optimizer more context. Lowering it further, or enabling `lto`, can improve speed or size at the cost of compile time; measure before and after (Lesson 41).
- `target/` grows large across profiles, targets and toolchain versions; `cargo clean` reclaims it.

---

## 13. Security and reliability

- **Building code is running code.** `cargo build` and `cargo check` execute **build scripts** (`build.rs`) and **procedural macros** from your dependency tree with your user's privileges. Never build an untrusted crate on a machine holding secrets, and review new dependencies (Lessons 35, 43).
- **Reproducibility is a security property.** Commit `Cargo.lock` for applications, pin the toolchain, and use `--locked` in CI (Cargo then errors instead of silently updating the lockfile; flag verified to exist ✓).
- **Install from official sources.** Get rustup from the official site and let it fetch toolchains over HTTPS rather than from third-party mirrors you have not vetted.
- **Warnings are early alarms.** Overflow, unused `Result`s, and unreachable code often show up as lints first.
- **Pinned vs floating toolchains** is a trade-off between reproducibility and receiving compiler fixes (Section 8); decide it consciously.

---

## 14. Exercises

Do them in a scratch directory. Mark them done yourself.

1. **Inventory.** Run `rustup show`, `rustc -vV`, and `rustup component list --installed`. Note your host triple, compiler commit hash, and LLVM version. Is `rust-docs` installed?
2. **Anatomy.** Run `cargo new tour`, then `cargo run`. List every file and directory that exists before and after the first run, and explain each one.
3. **Profiles.** Build with `cargo build` and `cargo build --release`. Find both binaries, compare sizes with `ls -l`, and explain the line "`dev` profile [unoptimized + debuginfo]".
4. **Warnings.** Add an unused variable and read the warning. Then add a loop of the form `for i in 0..v.len() { … v[i] … }` and compare `cargo build` with `cargo clippy` output.
5. **Formatting.** Break the formatting of your code, run `cargo fmt --check`, and record the exit code (`echo $?`). Then run `cargo fmt`.
6. **Pinning.** Add a `rust-toolchain.toml` with `channel = "stable"`, verify with `rustup show active-toolchain`, then introduce the `stabel` typo and read the error.
7. **Cfg.** Compare `rustc --print cfg` with `rustc -O --print cfg`. Which `cfg` disappears, and what does it control?
8. **Migration.** Create a project with `cargo new --edition 2021 mig`, run `cargo fix --edition --allow-no-vcs`, and check whether `Cargo.toml` changed. Then migrate it by hand.

---

## 15. Debugging challenge

This program compiles without warnings and prints `6`, but your CI runs `cargo clippy -- -D warnings` and fails. **Predict which lints fire**, then fix the code **without changing its output** so that both `cargo fmt --check` and `cargo clippy -- -D warnings` pass.

```rust
fn summarize(values: &[i32]) {
    let mut total = 0;
    for i in 0..values.len() {
        total += values[i];
    }
    if values.len() == 0 {
        println!("empty");
    }
    println!("{total}");
}

fn main() {
    let v = vec![1, 2, 3];
    summarize(&v);
}
```

<details markdown="1">
<summary>Solution (open after you try)</summary>

Clippy reports (✓): `needless_range_loop` ("the loop variable `i` is only used to index `values`") and `len_zero` ("length comparison to zero"). `cargo build` is silent because neither is a compiler warning.

Fixed (verified: `fmt --check` exit 0, `clippy -D warnings` exit 0, output `6`):

```rust
fn summarize(values: &[i32]) {
    let total: i32 = values.iter().sum();
    if values.is_empty() {
        println!("empty");
    }
    println!("{total}");
}

fn main() {
    let v = [1, 2, 3];
    summarize(&v);
}
```

Why the changes are better, not just quieter: `iter().sum()` cannot index out of bounds and states the intent; `is_empty()` states intent; passing a fixed-size array avoids a heap allocation because `summarize` only needs a slice (Lesson 05). Note the `: i32` annotation: `sum` can produce several numeric types, so inference needs help.

A trap I hit while verifying this: my first fixed version called `is_empty()` on a *fixed-size array* inside `main`, and Clippy then reported `const_is_empty` ("this expression always evaluates to false") because the length is known at compile time. Moving the check into a function that takes a slice avoids that.

</details>

---

## 16. Interview questions

<details markdown="1">
<summary>1. What is the difference between rustup, rustc, and cargo?</summary>

rustup manages toolchains (versions, components, targets). rustc compiles crates. Cargo is the build system and package manager that invokes rustc with the right flags, resolves dependencies, and runs tests, docs, and tooling. You usually run `cargo`; Cargo runs the `rustc` of the toolchain rustup selected.

</details>

<details markdown="1">
<summary>2. Why does `cargo check` exist if `cargo build` already tells me about errors?</summary>

`check` stops after type checking and borrow checking, skipping code generation, so feedback is faster. Caveat: a few errors surface only during monomorphization or codegen, so a clean `check` does not strictly guarantee a clean `build`.

</details>

<details markdown="1">
<summary>3. Which behaviors differ between dev and release, and why does it matter?</summary>

By default `dev` has `opt-level = 0`, debug info, `debug-assertions` and `overflow-checks` on, and incremental compilation; `release` has `opt-level = 3`, no debug info, assertions and overflow checks off, and non-incremental builds. It matters because performance, size and even arithmetic behavior (panic vs wrap) differ, so tests, benchmarks and production must use the intended profile.

</details>

<details markdown="1">
<summary>4. How do `rust-toolchain.toml` and `rust-version` differ?</summary>

`rust-toolchain.toml` selects the compiler rustup uses for that repository (a project/CI decision). `rust-version` (MSRV) in `Cargo.toml` declares the oldest compiler your crate promises to support, and Cargo errors on older ones. A library typically pins a recent toolchain for development while declaring a lower MSRV and testing against it.

</details>

<details markdown="1">
<summary>5. Why run Clippy with `-D warnings` in CI but avoid `#![deny(warnings)]` in the source?</summary>

CI is a controlled environment where you choose the toolchain, so failing on any lint is safe and keeps the main branch clean. A hard-coded `deny(warnings)` also affects anyone compiling the crate with a different (usually newer) compiler, where new lints would turn into build failures for downstream users.

</details>

<details markdown="1">
<summary>6. What can run on your machine when you merely run `cargo build` on a project?</summary>

Build scripts (`build.rs`) and procedural macros from the project and all its dependencies, with your privileges. Building an untrusted crate is code execution. Mitigations: review dependencies, build in a sandbox or container, avoid secrets on the build machine, and audit the dependency tree (Lesson 43).

</details>

<details markdown="1">
<summary>7. What does `Cargo.lock` do and what does `--locked` add?</summary>

`Cargo.lock` records the exact resolved version of every dependency so builds are reproducible. `--locked` makes Cargo fail if the lockfile would need updating, instead of silently changing it; CI typically uses it to catch a forgotten lockfile update.

</details>

---

## 17. Practical challenge

**Build a local quality gate.** Create a library crate `gate` that:

1. uses edition 2024 and declares `rust-version = "1.85"`;
2. has a `rust-toolchain.toml` pinning `stable` with `clippy` and `rustfmt`;
3. has a `rustfmt.toml` setting `max_width = 80`;
4. contains `pub fn add_saturating(a: u8, b: u8) -> u8` (clamps at `u8::MAX`) with two unit tests;
5. has an executable `check.sh` that runs the four-stage gate from Section 10 and prints `gate passed` only if every stage succeeds.

Then **break it three different ways** and confirm each break is caught by a *different* stage: bad formatting, a Clippy lint, and a failing test. Predict the stage before you run.

<details markdown="1">
<summary>Solution (verified on rustc 1.97.0 / cargo 1.97.0)</summary>

`Cargo.toml`:

```toml
[package]
name = "gate"
version = "0.1.0"
edition = "2024"
rust-version = "1.85"

[dependencies]
```

`rust-toolchain.toml`:

```toml
[toolchain]
channel = "stable"
components = ["clippy", "rustfmt"]
```

`rustfmt.toml`:

```toml
max_width = 80
```

`src/lib.rs`:

```rust
/// Adds two bytes, clamping at `u8::MAX` instead of overflowing.
pub fn add_saturating(a: u8, b: u8) -> u8 {
    a.saturating_add(b)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn clamps_at_max() {
        assert_eq!(add_saturating(200, 100), u8::MAX);
    }

    #[test]
    fn adds_normally() {
        assert_eq!(add_saturating(1, 2), 3);
    }
}
```

`check.sh` (run with `sh check.sh`):

```sh
#!/bin/sh
# Local quality gate: stop at the first failing stage.
set -eu

cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test
cargo build --release

echo "gate passed"
```

Clean run: both tests pass and the script prints `gate passed` (✓).

The three breaks, each run through `check.sh` (✓):

| Break | Change | Failing stage | Observed |
|---|---|---|---|
| Formatting | Write the body on one line: `pub fn add_saturating(a: u8, b: u8) -> u8 { a.saturating_add(b) }` | `cargo fmt --check` | Diff printed, exit code **1** |
| Lint | Add `let c = b.clone();` and use `c` | `cargo clippy … -D warnings` | `error: using `clone` on type `u8` which implements the `Copy` trait` (`clippy::clone_on_copy`), exit code **101** |
| Behavior | Replace `saturating_add` with `wrapping_add` | `cargo test` | `assertion `left == right` failed`, `left: 44`, `right: 255` (200 + 100 = 300, which wraps to 44), exit code **101** |

Why the stages are ordered this way: formatting needs no compilation, so it is the cheapest to fail on; Clippy needs the front end but not execution; tests need compilation *and* execution; the release build is last because it is the slowest and the previous stages already caught most problems. The third break is a Lesson 01 callback: `wrapping_add` and `saturating_add` have different contracts, and only a test notices.

</details>

---

## 18. Key takeaways

- **rustup** picks the toolchain, **cargo** drives the build, **rustc** compiles; Clippy and rustfmt are components you should run constantly.
- Toolchain precedence: `+toolchain` > `RUSTUP_TOOLCHAIN` > directory override > `rust-toolchain.toml` > default. `rustup show` tells you which one won.
- `cargo check` for speed, `cargo build --release` for real measurements; `dev` and `release` differ in more than speed, including overflow behavior.
- Read diagnostics top-down: header, `-->`, labels, `help`, then `rustc --explain`. Treat suggestions as hints.
- Enforce quality in CI (`fmt --check`, `clippy -D warnings`, tests), not with blanket `allow`s or `deny(warnings)` in source.
- Pin the toolchain deliberately; do not confuse `rust-toolchain.toml` with `rust-version`.
- `cargo fix --edition` migrates code but **does not** change the edition in `Cargo.toml`.
- Building code runs code (build scripts, proc macros): dependencies are a trust decision.

**Connections.** Lesson 01 gave you editions, channels and the debug/release overflow difference; here the profile table shows where that behavior is configured. Next, Lesson 03 covers crates, modules, workspaces, dependencies, features, custom build profiles and CI basics; Lesson 33 returns to `rustdoc`; Lesson 39 to `cargo test`; Lesson 43 to CI and supply-chain security. See the [roadmap](00_roadmap.md) and the [course index](index.md).

---

## 19. References

- *The Rust Programming Language*, ch. 1 "Getting Started" (installation, Hello Cargo): <https://doc.rust-lang.org/book/ch01-00-getting-started.html>
- *The rustup book*, overrides and `rust-toolchain.toml`: <https://rust-lang.github.io/rustup/overrides.html>
- *The rustup book*, profiles: <https://rust-lang.github.io/rustup/concepts/profiles.html>
- *The Cargo Book*: <https://doc.rust-lang.org/cargo/>
- *The Cargo Book*, profiles (default values): <https://doc.rust-lang.org/cargo/reference/profiles.html>
- *The Cargo Book*, `rust-version`: <https://doc.rust-lang.org/cargo/reference/rust-version.html>
- *The rustc Book* (command-line arguments, lints, platform support): <https://doc.rust-lang.org/rustc/>
- *The Clippy Book*: <https://doc.rust-lang.org/clippy/>
- Clippy lint index: <https://rust-lang.github.io/rust-clippy/stable/index.html>
- *rustfmt* configuration: <https://rust-lang.github.io/rustfmt/>
- *The Rust Style Guide*: <https://doc.rust-lang.org/style-guide/>
- *The Edition Guide*, transitioning an existing project: <https://doc.rust-lang.org/edition-guide/editions/transitioning-an-existing-project-to-a-new-edition.html>
