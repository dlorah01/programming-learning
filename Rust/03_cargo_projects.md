---
layout: default
title: "Lesson 03 — Cargo projects: crates, modules, dependencies, features, workspaces, profiles, CI"
---

# Lesson 03 — Cargo projects: crates, modules, dependencies, features, workspaces, profiles, CI

**Module 1 · Foundations and toolchain** · Status: generated · Edition used in examples: **2024** · External dependency used: `itoa 1.0.18` (fetched from crates.io during verification)

Verified while writing this lesson with `rustc 1.97.0`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux. As in Lesson 02, ✓ marks commands and outputs I actually ran; unmarked statements are documented Cargo behavior I did not run. Edition 2024 needs Rust **1.85.0 or newer**.

Builds on [Lesson 02](02_toolchain.md) (Cargo basics, profiles, quality gate) and [Lesson 01](01_foundations.md) (editions, overflow semantics).

---

## 1. Vocabulary: package, crate, target, module, workspace

The words are used loosely in the wild, so fix them first.

| Term | Meaning |
|---|---|
| **Package** | What one `Cargo.toml` describes: metadata, dependencies, features, and one or more crates. The unit you version and publish |
| **Crate** | A *compilation unit*: one `rustc` invocation builds one crate, either a **binary** or a **library**. Its root file is `src/main.rs` or `src/lib.rs` |
| **Target** | A buildable artifact of a package: library, binary, example, integration test, benchmark |
| **Module** | A namespace *inside* a crate and the unit of **privacy** |
| **Workspace** | A set of packages that share one `Cargo.lock`, one `target/` directory, and common settings |
| **Registry** | A package index and download host; the default is crates.io |

On crates.io the word "crate" really means *package*. In compiler terms a crate is the compilation unit. Context decides.

### 1.1 Package layout (Cargo auto-discovers targets)

| Path | Target |
|---|---|
| `src/main.rs` | the package's default binary (named after the package) |
| `src/lib.rs` | the package's library (a package has **at most one** library) |
| `src/bin/*.rs` | additional binaries |
| `examples/*.rs` | examples (`cargo run --example name`) |
| `tests/*.rs` | integration tests, each compiled as its own crate (Lesson 39) |
| `benches/*.rs` | benchmarks (Lesson 41) |

A package may have both `lib.rs` and `main.rs`: the binary then depends on the library through the package name. Hyphens in a package name become underscores in code (`my-crate` is imported as `my_crate`).

When a package has several binaries, `cargo run` must be told which (✓, observed):

```text
error: `cargo run` could not determine which binary to run. Use the `--bin` option to specify a binary, or the `default-run` manifest key.
available binaries: app, extra
```

Fix it with `cargo run --bin app`, or with `default-run = "app"` under `[package]` (✓ worked in the workspace of Section 7).

---

## 2. Modules and visibility

### 2.1 The module tree

A crate is a tree of modules rooted at the **crate root**. `mod name;` is a **declaration**: it tells the compiler "include a module here". It is *not* an import. A `.rs` file that no `mod` declaration reaches is not compiled at all, which is a classic source of "my code does nothing" confusion.

Where a declared module's code lives:

- `mod area { ... }` inline, or
- `mod area;` in the crate root, loaded from `src/area.rs` (or the older style `src/area/mod.rs`);
- a submodule `mod checks;` declared inside `src/area.rs` loads from `src/area/checks.rs` (✓ this exact layout compiled in Section 7).

### 2.2 Paths and `use`

```rust
mod outer {
    pub fn f() -> &'static str {
        "outer::f"
    }

    pub mod inner {
        pub fn g() -> String {
            format!("{} -> {}", super::f(), self::h())
        }

        fn h() -> &'static str {
            "inner::h"
        }

        pub fn from_root() -> &'static str {
            crate::top()
        }
    }
}

fn top() -> &'static str {
    "top"
}

use outer::inner::{self, g as call_g};

fn main() {
    println!("{}", call_g());
    println!("{}", inner::from_root());
}
```

Output (✓): `outer::f -> inner::h`, then `top`.

- `crate::` starts at the crate root; `self::` is the current module; `super::` is the parent.
- `use` brings a name into scope: nested groups `{self, g as call_g}`, renames with `as`. Avoid glob imports (`use x::*`) outside tests and preludes; they hide where names come from.
- `pub use path::Item;` **re-exports** an item, letting you shape a public API independently of your internal file layout (✓ used in Section 7: `pub use area::rect_area;` lets users write `shapes::rect_area`).

### 2.3 Privacy is the encapsulation boundary

Everything is **private by default**. Visibility forms:

| Syntax | Visible to |
|---|---|
| *(nothing)* | the current module and its descendants |
| `pub` | anywhere the item's path is reachable |
| `pub(crate)` | the whole crate |
| `pub(super)` | the parent module |
| `pub(in path)` | a named ancestor module |

Rule of thumb: code can see the private items of its **ancestors**, but not of its **siblings or children**. Privacy applies to **struct fields** too, which is how a type protects its invariants:

```rust
mod bank {
    pub struct Account {
        pub owner: String,
        balance: u64, // private: only code inside `bank` can touch it
    }

    impl Account {
        pub fn open(owner: &str) -> Account {
            Account {
                owner: owner.to_owned(),
                balance: 0,
            }
        }

        pub fn deposit(&mut self, amount: u64) -> Option<u64> {
            self.balance = self.balance.checked_add(amount)?;
            Some(self.balance)
        }
    }

    mod audit {
        // Visible in the parent module `bank`, but not outside it.
        pub(super) fn log(msg: &str) {
            println!("audit: {msg}");
        }
    }

    pub fn close(account: Account) {
        audit::log(&format!("closing {}", account.owner));
    }
}

use bank::{Account, close};

fn main() {
    let mut a = Account::open("ana");
    println!("{:?}", a.deposit(5));
    close(a);
}
```

Output (✓): `Some(5)`, then `audit: closing ana`. Because `balance` is private, the only way to change it is `deposit`, which uses `checked_add` (Lesson 01): no outside code can break the "balance never overflows" invariant.

Incorrect:

```rust
mod bank {
    pub struct Account {
        pub owner: String,
        balance: u64,
    }

    fn audit() {}

    pub fn open() {
        audit();
    }
}

fn main() {
    bank::open();
    bank::audit(); // error: private function
    let _a = bank::Account {
        owner: String::new(),
        balance: 1, // error: private field
    };
}
```

rustc first reports `error[E0603]: function `audit` is private` (✓). Only **after** you fix that does it report `error[E0451]: field `balance` of struct `Account` is private` (✓ I removed the first error and recompiled). Errors arrive in **phases**: a later phase may not run until an earlier one succeeds, so a clean list of errors is not the whole list.

**Design rationale.** Module-level privacy lets you decide the public surface of a library separately from its implementation, and lets you change internals without breaking users. Treat everything `pub` as a promise (Section 3.5, SemVer).

---

## 3. Dependencies

### 3.1 Declaring them

```toml
[dependencies]
itoa = "1.0.18"                                  # from crates.io
shapes = { path = "../shapes" }                  # a local package
# some_lib = { git = "https://...", rev = "abc123" }   # from Git, pinned to a commit

[dev-dependencies]    # only for tests, examples, benchmarks; not inherited by dependents
[build-dependencies]  # only for build.rs build scripts
```

`cargo add itoa` (✓) updated the manifest and printed what it did, including the crate's features and a note about MSRV-aware resolution:

```text
      Adding itoa v1.0.18 to dependencies
             Features:
             - no-panic
    Updating crates.io index
     Locking 1 package to latest Rust 1.97.0 compatible version
```

### 3.2 Version requirements

A bare version is a **caret** requirement (Cargo book): compatible updates are allowed, breaking ones are not.

| Written | Means |
|---|---|
| `"1.0.18"` or `"^1.0.18"` | `>=1.0.18, <2.0.0` |
| `"0.2.3"` | `>=0.2.3, <0.3.0` (for `0.x`, the *minor* number is the breaking one) |
| `"~1.0.18"` | `>=1.0.18, <1.1.0` |
| `"=1.0.18"` | exactly that version |
| `"*"` | anything; avoid it |

### 3.3 `Cargo.toml` says what is *allowed*; `Cargo.lock` says what was *chosen*

The lockfile records the exact resolved version of every package. Observed (✓):

- After the build, `Cargo.lock` held `itoa` `1.0.18` and the registry source.
- `cargo update itoa --precise 1.0.15` was **refused**: the lockfile cannot hold a version outside the manifest's requirement (`^1.0.18`). The message began: `candidate versions found which didn't match: 1.0.15`.
- Plain `cargo update` re-resolved within the requirements and changed nothing (`Locking 0 packages`).
- `cargo tree` prints the resolved graph (✓):

```text
netcheck v0.1.0 (/path/to/netcheck)
└── itoa v1.0.18
```

- After adding a new dependency line to `Cargo.toml` without updating the lockfile, `cargo build --locked` **failed** (✓):

```text
error: cannot update the lock file .../Cargo.lock because --locked was passed to prevent this
```

- With **no** `Cargo.lock` at all, `--locked` fails too (✓ I hit this rebuilding the challenge workspace from a fresh directory): `error: cannot create the lock file .../Cargo.lock because --locked was passed to prevent this`. Create it once with `cargo generate-lockfile` (or any normal build) and commit it.

Policy: commit `Cargo.lock` for applications. For libraries it is often committed too, but **only the top-level lockfile is used** when someone builds a project that depends on your library, so it does not constrain your users. In CI use `--locked` so a forgotten lockfile update fails loudly instead of silently changing what you test.

### 3.4 Choosing dependencies is a trust decision

Each dependency is code you ship and code that runs at build time (Lesson 02). Before adding one: is it maintained, how many transitive dependencies does it pull (`cargo tree`), what license, does it use `unsafe` heavily, can you do the job with `std`? Lesson 43 covers auditing tools.

### 3.5 SemVer in Rust

Versions are `MAJOR.MINOR.PATCH`. For your own crates: removing or renaming a `pub` item, changing a public signature, making a public field private, removing a feature, or adding a required method to a public trait is a **breaking change**. `#[non_exhaustive]` on public structs and enums lets you add fields and variants later without breaking users, but it must be there from the start: adding it to an existing public item is itself a breaking change (per the Cargo book). The Cargo book's SemVer-compatibility chapter lists the rules; tools can check them automatically (Lesson 43).

---

## 4. Features

A **feature** is a named, compile-time option of a package. Features enable optional code and optional dependencies, so users pay (in compile time, binary size and attack surface) only for what they use.

```toml
[features]
default = []              # features enabled unless the user opts out
fast-int = ["dep:itoa"]   # enabling `fast-int` pulls in the optional dependency

[dependencies]
itoa = { version = "1.0", optional = true }
```

```rust
pub const fn backend() -> &'static str {
    if cfg!(feature = "fast-int") {
        "itoa"
    } else {
        "std"
    }
}

pub fn format_int(n: u32) -> String {
    #[cfg(feature = "fast-int")]
    {
        let mut buf = itoa::Buffer::new();
        buf.format(n).to_owned()
    }
    #[cfg(not(feature = "fast-int"))]
    {
        n.to_string()
    }
}
```

- `#[cfg(feature = "x")]` includes or removes **code** at compile time; `cfg!(feature = "x")` is the same test as a `bool` expression (both branches must still compile).
- `optional = true` creates an implicit feature of the same name; using `dep:` anywhere in `[features]` suppresses that implicit feature so you can name features independently of dependencies (needs Rust 1.60+).
- Flags: `--features a,b`, `--no-default-features`, `--all-features`; in a workspace `--features pkg/feature` targets one member. A dependent can write `shapes = { path = "...", features = ["fast-int"], default-features = false }`.

### 4.1 The additive rule and feature unification

Cargo's documented design: **features must be additive**. Enabling a feature must never remove functionality, and any combination should be safe. The reason is **unification**: when several packages depend on the same crate, Cargo builds it **once** with the **union** of all features anyone requested. Features belong to the package that defines them, and `default-features = false` does not guarantee defaults are off if something else in the graph leaves them on. Do not model mutually exclusive choices as features; if you must, add a `compile_error!` to detect the conflict. (For `no_std`, Cargo recommends a `std` feature, not a `no_std` feature; Lesson 38.)

### 4.2 The trap, reproduced (✓)

In the workspace of Section 7, `app` depends on `shapes` with `features = ["fast-int"]`. Suppose `shapes` had this test:

```rust
#[test]
fn backend_is_std_by_default() {
    assert_eq!(backend(), "std");
}
```

Observed:

| Command | Result |
|---|---|
| `cargo test -p shapes` | **passes** (feature off, `shapes` built alone) |
| `cargo test --workspace` | **fails**: `left: "itoa"`, `right: "std"` (feature on, requested by `app` and unified) |

`cargo tree -e features -p app` shows the request that causes it (✓):

```text
app v0.1.0 (...)
├── shapes feature "default"
│   └── shapes v0.1.0 (...)
│       └── itoa feature "default"
│           └── itoa v1.0.18
└── shapes feature "fast-int"
    └── shapes v0.1.0 (...) (*)
```

The fix is to make tests **state which feature configuration they assume**:

```rust
#[test]
#[cfg(not(feature = "fast-int"))]
fn backend_is_std_without_the_feature() {
    assert_eq!(backend(), "std");
}

#[test]
#[cfg(feature = "fast-int")]
fn backend_is_itoa_with_the_feature() {
    assert_eq!(backend(), "itoa");
}
```

Verified (✓): `cargo test -p shapes`, `cargo test -p shapes --features fast-int`, `cargo test --workspace` and `cargo test --workspace --all-features` all pass, each running the test matching its configuration. This is why CI exercises several feature combinations (Section 8).

### 4.3 The resolver

The **resolver version** controls how features (and MSRV) are unified; it is global to a workspace, and only the top-level manifest's value counts:

| Resolver | Default for | Behavior (Cargo reference) |
|---|---|---|
| `"1"` | no edition-based default | unifies features regardless of where they are requested |
| `"2"` | edition 2021 | no unification across target-specific dependencies not being built, between build-dependencies / proc-macros and normal dependencies, or from dev-dependencies unless they are being built |
| `"3"` | edition 2024 (Rust 1.84+) | same feature rules as 2; additionally prefers dependency versions compatible with your `rust-version` |

Observed (✓): in the workspace with `rust-version = "1.85"` Cargo printed `Locking 1 package to latest Rust 1.85 compatible version`, the MSRV-aware behavior. A **virtual workspace** (a root manifest with no `[package]`) has no edition to inherit from, so omit `resolver` and Cargo warns (✓):

```text
warning: virtual workspace defaulting to `resolver = "1"` despite one or more workspace members being on edition 2024 which implies `resolver = "3"`
  = note: to use the edition 2024 resolver, specify `workspace.resolver = "3"` in the workspace root's manifest
```

Always set `resolver = "3"` explicitly in a virtual workspace root.

---

## 5. Build profiles beyond the defaults

Lesson 02 listed the defaults. You customize them under `[profile.<name>]` in `Cargo.toml`.

| Setting | Values | Trade-off |
|---|---|---|
| `opt-level` | `0`–`3`, `"s"`, `"z"` | speed vs size vs compile time |
| `lto` | `false`, `"thin"`, `true` (fat), `"off"` | cross-crate optimization; longer link time |
| `codegen-units` | integer | fewer units: more optimization, less parallelism |
| `panic` | `"unwind"`, `"abort"` | `abort` shrinks code but panics cannot be caught and destructors do not run on panic; tests, benchmarks, build scripts and proc macros ignore this setting (Cargo book) |
| `strip` | `"none"`, `"debuginfo"`, `"symbols"` (or `true`) | smaller binary, harder to debug |
| `debug` | `false`, `true`, `"line-tables-only"`, `0`–`2` | debuggability vs size |

### 5.1 Measured effect on one tiny binary (✓)

Release build of `app` from Section 7 (it calls `shapes` and `itoa`), one setting changed at a time via `cargo build --release --config 'profile.release.<key>=<value>'`:

| Configuration | Binary size (bytes) |
|---|---|
| release defaults | 446,904 |
| `lto = true` | 378,448 |
| `lto = true`, `codegen-units = 1` | 378,448 |
| `strip = true` | 343,816 |
| `panic = "abort"` | 444,512 |
| `opt-level = "z"` | 447,768 |
| all five combined | 289,464 |

Read this table carefully:

- It is **one trivial program on one platform**. Most of the size is the standard library, so percentages here do not transfer to your project.
- `opt-level = "z"` alone came out **slightly larger** than the default. Settings interact; "size mode" is not automatically smaller. **Measure your own binary.**
- These are sizes, not speeds. Optimizations that shrink code can make it slower and vice versa; Lesson 41 teaches how to measure speed.

### 5.2 Custom profiles and placement rules

A custom profile must say what it inherits from (✓ built to `target/release-small/` and ran):

```toml
[profile.release-small]
inherits = "release"
opt-level = "z"
lto = true
codegen-units = 1
strip = true
panic = "abort"
```

Build with `cargo build --profile release-small`. Profiles are honored **only in the workspace root manifest**; in a member they are ignored with a warning (✓):

```text
warning: profiles for the non root package will be ignored, specify profiles at the workspace root:
```

You can also override per dependency, for example optimizing dependencies even in `dev` with `[profile.dev.package."*"] opt-level = 2` (documented, not run here).

---

## 6. Build scripts and publishing (overview)

- A **build script** `build.rs` runs *before* compiling the package; it can generate code, link C libraries, or probe the environment. It is arbitrary code executed at build time, as are procedural macros (Lesson 02, Section 13). Lessons 35 and 37 return to both.
- To publish to crates.io a package needs metadata (`description`, `license`, ...), uses SemVer, and every dependency must be available from the registry (a path dependency also needs a `version` to be publishable). Published versions can be **yanked** (discouraged for new use) but never deleted. For internal packages set `publish = false` in `[package]` so an accidental `cargo publish` fails. Nothing in this lesson publishes anything.

---

## 7. Workspaces

A **workspace** groups packages that evolve together. Benefits: one `Cargo.lock` (consistent dependency versions), one shared `target/` (build each dependency once), shared settings, and commands that act on all members.

Observed (✓ via `cargo metadata`): members `shapes` and `app`; one `Cargo.lock` at the root and none inside the members; the `target/` directory at the root.

### 7.1 Root manifest

```toml
[workspace]
resolver = "3"
members = ["crates/shapes", "crates/app"]

[workspace.package]            # values members can inherit
version = "0.1.0"
edition = "2024"
rust-version = "1.85"

[workspace.dependencies]       # one place to pin dependency versions
itoa = "1.0"
shapes = { path = "crates/shapes" }
```

Members inherit with `version.workspace = true`, `edition.workspace = true`, and `itoa = { workspace = true }`; a member can add features on top (`features = ["fast-int"]`).

### 7.2 Commands

| Command | Effect |
|---|---|
| `cargo build --workspace` ✓ | build every member |
| `cargo test --workspace` ✓ | test every member (features unify, Section 4.2) |
| `cargo test -p shapes` ✓ | one member |
| `cargo test --workspace --exclude app` | all but one |
| `cargo run -p app --bin extra` ✓ | a specific binary of a member |

### 7.3 When to split into crates, and when not

Split when: a part has a **genuinely separate API** or audience; a heavy optional dependency should not burden everyone; you need a **procedural-macro crate** (it must be its own crate); independent crates compile in parallel; or you want enforced boundaries (a crate cannot reach into another's private items). Do **not** split early: every crate adds manifests, version coordination, and cross-crate friction. Start with modules (Section 2); promote a module to a crate when a concrete reason appears.

---

## 8. CI basics

The goal is the same gate you built in Lesson 02, extended for workspaces and features. Locally (✓ this exact script passed on the workspace):

```sh
#!/bin/sh
# Local copy of the CI pipeline: stop at the first failing stage.
set -eu

cargo fmt --all --check
cargo clippy --workspace --all-targets --all-features --locked -- -D warnings
cargo test --workspace --locked
cargo test --workspace --all-features --locked
cargo check -p shapes --no-default-features --locked
cargo build --workspace --release --locked

echo "ci passed"
```

Why each flag:

- `--workspace`: every member, not only the current directory's package.
- `--all-targets`: library, binaries, tests, examples and benchmarks, so a lint in a test file cannot hide.
- `--locked`: fail if `Cargo.lock` would change (Section 3.3).
- `--all-features` plus a `--no-default-features` check: exercise the extremes. For many independent features, a *feature powerset* tool (such as `cargo-hack`) is the thorough option; I did not run it here.
- `-D warnings`: turn lints into failures in CI (Lesson 02, Section 5).

The same pipeline as a GitHub Actions workflow in `.github/workflows/ci.yml` (✓ parses as valid YAML; the `rustup toolchain install` step could not be run in my sandbox because it blocks the download host, so confirm it on your first CI run):

```yaml
name: ci

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read

env:
  CARGO_TERM_COLOR: always

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install toolchain
        run: rustup toolchain install stable --profile minimal --component clippy --component rustfmt
      - name: Format
        run: cargo fmt --all --check
      - name: Lint
        run: cargo clippy --workspace --all-targets --all-features --locked -- -D warnings
      - name: Test (default features)
        run: cargo test --workspace --locked
      - name: Test (all features)
        run: cargo test --workspace --all-features --locked
      - name: Check (no default features)
        run: cargo check -p shapes --no-default-features --locked
      - name: Release build
        run: cargo build --workspace --release --locked
```

Notes:

- I could not verify the current major version of `actions/checkout` (my attempt to read its release page was blocked), so check it, and consider **pinning third-party actions to a full commit SHA** and keeping `permissions` minimal, as shown.
- Add a second job that builds with your **MSRV** (`rustup toolchain install 1.85.0`, then `cargo +1.85.0 check --workspace --locked`) so `rust-version` is a tested promise, not a hope (documented approach; not run here).
- Cache `target/` and the registry with a caching action keyed on `Cargo.lock` to speed up runs; evaluate the options for your repository.
- `cargo update` in CI changes what you test; run it deliberately (for example in a scheduled job that opens a pull request), not on every build.

---

## 9. Common mistakes and edge cases

1. **Creating a `.rs` file and forgetting `mod name;`.** Nothing compiles it.
2. **Making everything `pub`** to silence privacy errors. Prefer `pub(crate)`/`pub(super)`, and `pub use` for the intended API.
3. **Wildcard dependency versions** (`"*"`) and unpinned `git` dependencies.
4. **Not committing `Cargo.lock` for an application**, or not using `--locked` in CI.
5. **Treating features as mutually exclusive switches**, then hitting unification surprises (Section 4.2).
6. **Testing only the default feature set.**
7. **Omitting `resolver = "3"`** in a virtual workspace (warning in Section 4.3).
8. **Putting `[profile.*]` in a member manifest** (ignored with a warning).
9. **Tuning profiles by folklore.** `opt-level = "z"` made my sample binary slightly larger, not smaller (Section 5.1).
10. **A path dependency with no `version`** when you later try to publish.
11. **Forgetting `publish = false`** on internal packages.
12. **Assuming one error list is complete**: privacy errors appear in phases (Section 2.3).

---

## 10. Performance implications

- **Compile time:** splitting into crates lets Cargo build independent crates in parallel and recompile only what changed; features keep unused code and dependencies out of the build. Over-splitting adds overhead. I did not time any build here, so measure your own.
- **Binary size and speed:** governed by the profile (Section 5), by features, and by dependencies. `lto`, `codegen-units = 1` and `opt-level` trade **compile time** for **runtime or size** gains that vary by program. Use measurements, as in Section 5.1, never rules of thumb.
- **Build artifacts:** a workspace shares one `target/`, so each dependency is compiled once per profile and feature set; different feature sets on the same crate can mean extra compilation.

---

## 11. Security and reliability

- **Supply chain.** Every dependency (and its transitive tree) is trusted code, plus build scripts and proc macros that run at build time. Review additions, prefer fewer and well-maintained crates, and watch for typo-squatted names. Tools for advisories and policy (`cargo audit`, `cargo deny`) arrive in Lesson 43.
- **Reproducibility.** Commit `Cargo.lock` for applications, use `--locked` in CI, pin git dependencies to a `rev`, and pin the toolchain (Lesson 02).
- **Features expand the attack surface.** Optional dependencies compile only when enabled; enabling `--all-features` in CI tests them, but your *users* choose which are active, so every combination is a configuration you ship.
- **Privacy is a safety tool.** Keep invariant-carrying fields private (Section 2.3); expose operations, not state.
- **`publish = false`** prevents accidental publication of internal or proprietary code.
- **CI hygiene.** Minimal `permissions`, pinned actions, and no secrets exposed to pull requests from forks.

---

## 12. Exercises

Do them in scratch directories. Mark them done yourself.

1. **Vocabulary.** For the workspace in Section 13, list its packages, crates, and targets. (`cargo metadata --no-deps` and `cargo tree` help.)
2. **Privacy.** Reproduce the `bank` example. Predict the two error codes and *their order* before compiling. Then expose `balance` through a `pub fn balance(&self) -> u64` and explain why that is safer than `pub balance`.
3. **Modules and files.** Create `src/area.rs` and `src/area/checks.rs`. Delete the `mod checks;` line and show that an error inside `checks.rs` stops being reported. Why?
4. **Dependencies.** `cargo add` a crate of your choice. Inspect `Cargo.toml`, `Cargo.lock` and `cargo tree`. Try `cargo update <crate> --precise <version outside the requirement>` and explain the refusal.
5. **Locking.** Add a dependency line by hand and run `cargo build --locked`. Read the error and fix it.
6. **Features.** Add a feature `strict` to `shapes` that makes `rect_area` also return `None` when the area exceeds 1000. Write tests for both configurations and run all four combinations from Section 4.2.
7. **Profiles.** Reproduce the size table of Section 5.1 for your own binary. Which single setting helped most? Did any make it larger?
8. **Workspace growth.** Add a third member `report` that depends on `shapes` through `workspace = true`. Confirm there is still exactly one `Cargo.lock`.
9. **CI.** Write the `ci.sh` of Section 8 for your workspace and break it three different ways (format, lint, a test that depends on a feature). Which stage catches each?

---

## 13. Practical challenge

**Build a two-crate workspace.** Requirements:

1. Root: a *virtual* workspace with `resolver = "3"`, `[workspace.package]` (version, edition 2024, `rust-version = "1.85"`) and `[workspace.dependencies]` (`itoa`, and `shapes` as a path dependency).
2. `shapes` (library): a public module `area` with a **private** submodule `checks` (in its own file) exposing `pub(super) fn nonzero`; a public `rect_area(w, h) -> Option<u32>` that rejects zero sides and overflow; a re-export `shapes::rect_area`; an optional dependency `itoa` behind a `fast-int` feature; `backend()` and `format_int()` working with either feature state; tests that are correct in **both** feature states.
3. `app` (binary package): two binaries (`app`, `extra`) and `default-run = "app"`; it depends on `shapes` with `fast-int` enabled and prints `backend=itoa area=42`.
4. A `release-small` profile at the root, built with `--profile release-small`.
5. `ci.sh` (Section 8) passing, plus the workflow file. `ci.sh` uses `--locked`, so run `cargo generate-lockfile` once first and commit the resulting `Cargo.lock`.

Predict before running: what does `cargo test -p shapes` run, and what does `cargo test --workspace` run, with respect to the backend test?

<details markdown="1">
<summary>Solution (verified: every file below was built and `ci.sh` printed `ci passed` on cargo 1.97.0)</summary>

Layout:

```text
ws/
├── Cargo.toml
├── ci.sh
├── .github/workflows/ci.yml        (as in Section 8)
└── crates/
    ├── shapes/
    │   ├── Cargo.toml
    │   └── src/{lib.rs, area.rs, area/checks.rs}
    └── app/
        ├── Cargo.toml
        └── src/{main.rs, bin/extra.rs}
```

**`Cargo.toml`**

```toml
[workspace]
resolver = "3"
members = ["crates/shapes", "crates/app"]

[workspace.package]
version = "0.1.0"
edition = "2024"
rust-version = "1.85"

[workspace.dependencies]
itoa = "1.0"
shapes = { path = "crates/shapes" }

[profile.release-small]
inherits = "release"
opt-level = "z"
lto = true
codegen-units = 1
strip = true
panic = "abort"
```

**`crates/shapes/Cargo.toml`**

```toml
[package]
name = "shapes"
version.workspace = true
edition.workspace = true
rust-version.workspace = true

[features]
default = []
fast-int = ["dep:itoa"]

[dependencies]
itoa = { workspace = true, optional = true }
```

**`crates/shapes/src/lib.rs`**

```rust
//! Tiny geometry library used to explore Cargo.

pub mod area;

pub use area::rect_area;

/// Which integer-formatting backend this build uses.
pub const fn backend() -> &'static str {
    if cfg!(feature = "fast-int") {
        "itoa"
    } else {
        "std"
    }
}

/// Formats an integer using the selected backend.
pub fn format_int(n: u32) -> String {
    #[cfg(feature = "fast-int")]
    {
        let mut buf = itoa::Buffer::new();
        buf.format(n).to_owned()
    }
    #[cfg(not(feature = "fast-int"))]
    {
        n.to_string()
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn formats_the_same_with_either_backend() {
        assert_eq!(format_int(42), "42");
    }

    #[test]
    #[cfg(not(feature = "fast-int"))]
    fn backend_is_std_without_the_feature() {
        assert_eq!(backend(), "std");
    }

    #[test]
    #[cfg(feature = "fast-int")]
    fn backend_is_itoa_with_the_feature() {
        assert_eq!(backend(), "itoa");
    }
}
```

**`crates/shapes/src/area.rs`**

```rust
mod checks;

/// Area of a rectangle, or `None` for a zero side or on overflow.
pub fn rect_area(w: u32, h: u32) -> Option<u32> {
    checks::nonzero(w, h)?;
    w.checked_mul(h)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn multiplies() {
        assert_eq!(rect_area(6, 7), Some(42));
    }

    #[test]
    fn rejects_zero_and_overflow() {
        assert_eq!(rect_area(0, 5), None);
        assert_eq!(rect_area(u32::MAX, 2), None);
    }
}
```

**`crates/shapes/src/area/checks.rs`**

```rust
/// Visible only inside the parent module `area`.
pub(super) fn nonzero(w: u32, h: u32) -> Option<()> {
    if w == 0 || h == 0 { None } else { Some(()) }
}
```

**`crates/app/Cargo.toml`**

```toml
[package]
name = "app"
version.workspace = true
edition.workspace = true
rust-version.workspace = true
default-run = "app"

[dependencies]
shapes = { workspace = true, features = ["fast-int"] }
```

**`crates/app/src/main.rs`**

```rust
use shapes::{backend, format_int, rect_area};

fn main() {
    let area = rect_area(6, 7).map_or_else(|| "overflow".to_owned(), format_int);
    println!("backend={} area={area}", backend());
}
```

**`crates/app/src/bin/extra.rs`**

```rust
fn main() {
    println!("extra binary");
}
```

**`ci.sh`**

```sh
#!/bin/sh
# Local copy of the CI pipeline: stop at the first failing stage.
set -eu

cargo fmt --all --check
cargo clippy --workspace --all-targets --all-features --locked -- -D warnings
cargo test --workspace --locked
cargo test --workspace --all-features --locked
cargo check -p shapes --no-default-features --locked
cargo build --workspace --release --locked

echo "ci passed"
```

Run `cargo generate-lockfile` once before `ci.sh` and commit `Cargo.lock` (Section 3.3). Observed (✓): `cargo run` prints `backend=itoa area=42` (thanks to `default-run`); `cargo run --bin extra -p app` prints `extra binary`; `sh ci.sh` ends with `ci passed`. **Answer to the prediction:** `cargo test -p shapes` builds `shapes` alone, so the *without-the-feature* test runs; `cargo test --workspace` unifies `app`'s request, so the *with-the-feature* test runs (4 tests pass in each). `cargo build --profile release-small -p app --bin app` produced `target/release-small/app` of **289,464 bytes**, versus **446,904** for the default release build of the same binary (this program only; Section 5.1).

</details>

---

## 14. Debugging challenge

This workspace's tests pass for one command and fail for another. **Predict which**, explain *exactly why*, and fix it **without deleting the test and without removing the feature from `viewer`**.

```text
--- Cargo.toml ---
[workspace]
resolver = "3"
members = ["logger", "viewer"]

--- logger/Cargo.toml ---
[package]
name = "logger"
version = "0.1.0"
edition = "2024"

[features]
verbose = []

--- logger/src/lib.rs ---
pub fn level() -> &'static str {
    if cfg!(feature = "verbose") {
        "verbose"
    } else {
        "quiet"
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn level_is_quiet() {
        assert_eq!(level(), "quiet");
    }
}

--- viewer/Cargo.toml ---
[package]
name = "viewer"
version = "0.1.0"
edition = "2024"

[dependencies]
logger = { path = "../logger", features = ["verbose"] }

--- viewer/src/main.rs ---
fn main() {
    println!("{}", logger::level());
}
```

<details markdown="1">
<summary>Solution (open after you try)</summary>

Observed (✓):

| Command | Result |
|---|---|
| `cargo test -p logger` | passes: `logger` is built alone, feature off |
| `cargo test --workspace` | **fails**: `left: "verbose"`, `right: "quiet"` |
| `cargo test -p logger --features verbose` | fails for the same reason |
| `cargo test --workspace --exclude viewer` | passes |

`viewer` asks for `logger`'s `verbose` feature. When both packages are built in one invocation, Cargo builds `logger` **once** with the **union** of requested features, so `verbose` is on, and the assertion `level() == "quiet"` is false. Feature **unification**, not a flaky test: the test encoded an assumption about the feature state that the build does not guarantee.

Fix: make each test declare its configuration (verified: all four commands pass, each running the matching test):

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[cfg(not(feature = "verbose"))]
    fn level_is_quiet_without_the_feature() {
        assert_eq!(level(), "quiet");
    }

    #[test]
    #[cfg(feature = "verbose")]
    fn level_is_verbose_with_the_feature() {
        assert_eq!(level(), "verbose");
    }
}
```

Excluding `viewer` only hides the symptom. The real lesson: features are additive, library tests must be correct for *every* combination they can be built with, and CI should run more than one (Section 8).

</details>

---

## 15. Interview questions

<details markdown="1">
<summary>1. What is the difference between a package, a crate, and a module?</summary>

A package is what a `Cargo.toml` describes (versioned, publishable, containing one or more crates). A crate is a compilation unit, a binary or a library, with a root file. A module is a namespace and privacy boundary inside a crate. A package has at most one library crate and any number of binary crates.

</details>

<details markdown="1">
<summary>2. What does private mean in Rust, and how do you widen visibility precisely?</summary>

By default an item is visible to the module that defines it and its descendants. `pub` makes it reachable from anywhere its path is visible; `pub(crate)`, `pub(super)` and `pub(in path)` widen it only as far as the crate, parent, or a named ancestor. Fields are private independently of the struct, which is how types protect invariants.

</details>

<details markdown="1">
<summary>3. What does `serde = "1.2.3"` allow, and what about `"0.2.3"`?</summary>

A bare version is a caret requirement: `>=1.2.3, <2.0.0` for the first. For `0.x` versions the minor number is the breaking one, so `"0.2.3"` allows `>=0.2.3, <0.3.0`.

</details>

<details markdown="1">
<summary>4. What is feature unification, and why must features be additive?</summary>

When several packages depend on the same crate, Cargo builds it once with the union of all requested features. If a feature removed or changed behavior, enabling it for one dependent could break another. So features must only add functionality, and tests must be written for every configuration they can be built under.

</details>

<details markdown="1">
<summary>5. Why use `--locked` in CI, and what does `Cargo.lock` do for libraries?</summary>

`--locked` makes the build fail if the lockfile would change, so CI tests exactly what is committed. For a library, the lockfile documents the versions you tested with but is ignored when others depend on the library; only the top-level project's lockfile counts.

</details>

<details markdown="1">
<summary>6. Why set `resolver = "3"` in a virtual workspace?</summary>

The resolver is a workspace-wide setting taken from the top-level manifest, and a virtual manifest has no package edition to infer it from; Cargo warns and defaults to `"1"`. Resolver 3 (edition 2024) also prefers dependency versions compatible with your `rust-version`.

</details>

<details markdown="1">
<summary>7. Name three profile settings that trade compile time for runtime or size, and a caveat for each.</summary>

`lto` (cross-crate optimization; slower linking), `codegen-units` (fewer units optimize better but compile less in parallel), and `panic = "abort"` (smaller code, but panics cannot be caught and destructors are skipped on panic). The general caveat: the effect depends on the program, as the measured table shows, so measure before adopting.

</details>

---

## 16. Key takeaways

- **Package** (manifest) ⊃ **crate** (compilation unit) ⊃ **modules** (namespaces and privacy). A workspace shares a lockfile and `target/`.
- `mod x;` declares; `use` imports; `pub use` re-exports. Privacy is the tool for protecting invariants; widen it only as far as needed.
- `Cargo.toml` states allowed versions, `Cargo.lock` the chosen ones; use `--locked` in CI, commit the lockfile for applications.
- Features are additive and unified across the build graph: write tests that declare their feature assumptions and run several combinations in CI.
- Profiles are tunable but non-obvious: measure; `opt-level = "z"` is not automatically smaller.
- Set `resolver = "3"` in virtual workspaces; keep profiles at the workspace root.
- Dependencies, build scripts and proc macros are a trust decision; CI should be reproducible, least-privilege, and exercise more than the default configuration.

**Connections.** Lesson 01's `checked_*` arithmetic protects the `Account` invariant; Lesson 02's gate grows into the workspace CI here. Next: Lesson 04 starts Module 2 (variables, mutability, shadowing, primitives, numeric behavior). Cargo resurfaces in Lesson 33 (public API design and `rustdoc`), Lesson 35 (proc-macro crates), Lesson 39 (tests), and Lesson 43 (CI and supply-chain security). See the [roadmap](00_roadmap.md) and the [course index](index.md).

---

## 17. References

- *The Cargo Book*, specifying dependencies: <https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html>
- *The Cargo Book*, features: <https://doc.rust-lang.org/cargo/reference/features.html>
- *The Cargo Book*, the dependency resolver (resolver versions, feature unification): <https://doc.rust-lang.org/cargo/reference/resolver.html>
- *The Cargo Book*, workspaces: <https://doc.rust-lang.org/cargo/reference/workspaces.html>
- *The Cargo Book*, profiles: <https://doc.rust-lang.org/cargo/reference/profiles.html>
- *The Cargo Book*, package layout and target auto-discovery: <https://doc.rust-lang.org/cargo/guide/project-layout.html>
- *The Cargo Book*, SemVer compatibility: <https://doc.rust-lang.org/cargo/reference/semver.html>
- *The Cargo Book*, `rust-version`: <https://doc.rust-lang.org/cargo/reference/rust-version.html>
- *The Cargo Book*, build scripts: <https://doc.rust-lang.org/cargo/reference/build-scripts.html>
- *The Cargo Book*, publishing on crates.io: <https://doc.rust-lang.org/cargo/reference/publishing.html>
- *The Rust Programming Language*, ch. 7 "Managing Growing Projects with Packages, Crates, and Modules": <https://doc.rust-lang.org/book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html>
- *The Rust Reference*, visibility and privacy: <https://doc.rust-lang.org/reference/visibility-and-privacy.html>
- GitHub Docs, security hardening for GitHub Actions: <https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions>
