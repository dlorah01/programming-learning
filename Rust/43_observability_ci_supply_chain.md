---
layout: default
title: "Lesson 43 — Observability, CI, dependency security"
---

# Lesson 43 — Observability, CI, dependency security

**Module 13 · Testing, performance and tooling** · Status: generated · Edition used in examples: **2024** · External dependencies: `tracing 0.1.44` (library), `tracing-subscriber 0.3.23` (tests and examples only)

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, `cargo-deny 0.20.2`, `cargo-audit 0.22.2`, on x86-64 Linux, 2 cores. As before, ✓ marks outputs and measurements I actually ran; unmarked statements are documented behavior I did not run. Things I could **not** run here and therefore describe from memory only: GitHub Actions itself (the workflow file in §8 is syntax-checked as YAML but was never executed by GitHub), the minimum-supported-Rust-version build (`rustup` could not install a second toolchain in this sandbox, so only the error message for an unmet `rust-version` was verified), `cargo vet`, `cargo geiger`, `cargo crev`, code coverage tools, OpenTelemetry exporters, and `tracing` features other than the ones used. I could not open the documentation sites, so rules stated without ✓ are from memory; the closing paragraph lists them. The advisory database and crates.io were reachable from the sandbox, so the dependency checks in §9 are real runs against the advisory database *as it stood when I ran them*; the database changes daily, so your results will differ.

Builds on [Lesson 41](41_benchmarking_profiling.md) (measuring overhead honestly), [Lesson 39](39_testing.md) (tests, mutation checks), [Lesson 28](28_atomics_memory_ordering.md) (the `Relaxed` counters), [Lesson 27](27_threads_channels.md) (scoped threads), [Lesson 26](26_error_design.md) (error types that explain themselves), [Lesson 35](35_proc_macros.md) and [Lesson 37](37_maybeuninit_ffi.md) (code that runs at build time: proc macros and build scripts, relevant to §9), and [Lesson 33](33_modules_api_design_rustdoc.md) (documented public API). Lesson 44 is next; it appears here only as plain text.

---

## 1. Why this lesson exists

A program that passes its tests can still be impossible to operate or unsafe to ship. Three habits close that gap:

1. **Observability.** When the program misbehaves in production you cannot attach a debugger. You can only read what it chose to record. Logs, metrics and traces decide whether an incident takes five minutes or five days, and a careless log line can itself be the incident (a leaked password, a forged log entry, a million lines per second).
2. **Continuous integration.** The checks you ran by hand in Lessons 1 to 42 (`fmt`, `clippy`, `test`, `doc`) must run on every change, on a clean machine, the same way every time.
3. **Dependency security.** A typical Rust program is mostly other people's code. Rust's memory safety does not protect you from a dependency that is malicious, abandoned, wrongly licensed, or has a known vulnerability. (Rust reduces one class of bugs; it does not make a program secure.)

The running example is a small crate, `obslab`: a job processor that logs structured events, never logs its credential, counts outcomes, and retries. It ships with the files a project needs for the other two habits: a `ci.sh` that runs the CI steps locally, a `deny.toml` dependency policy and a workflow file. All ✓ results come from it.

---

## 2. Logs, metrics, traces

| Signal | Answers | Shape | Cost |
|---|---|---|---|
| **Log** (event) | "What happened in this one case, and why?" | a record: time, level, message, fields | per event; can be large |
| **Metric** | "How often / how much / how fast, overall?" | a number that changes (counter, gauge, histogram) | tiny and constant per update |
| **Trace** (span) | "Where did this request spend its time, across functions or services?" | a tree of timed spans with ids | per span |

Use metrics to *notice* a problem (error rate rose), traces to *locate* it (which step is slow), logs to *explain* it (the exact error and inputs for one request). This lesson implements the first two with `tracing` and plain counters; spans are used as context for log events. Exporting traces to a collector (OpenTelemetry) is outside what I ran.

### 2.1 `log` versus `tracing` (from memory)

The `log` crate is a facade: libraries call `log::info!`, and the application picks one logger. `tracing` is a newer, richer facade from the same ecosystem with the same macros, plus **structured fields** (typed key-value pairs, not text pasted into a message) and **spans** (a period of time with fields that applies to every event inside it). `tracing` can also emit `log` records for compatibility (a feature flag, not used here). Libraries should depend only on the facade (`tracing`, as `obslab` does) and leave subscriber setup to the binary; `tracing-subscriber` appears only under `[dev-dependencies]` so that a user of the library does not pay for it.

---

## 3. Structured events and spans with `tracing` ✓

```rust
// obslab::process, abridged
#[instrument(skip(job, metrics), fields(job.id = job.id, user = %job.user))]
pub fn process(job: &Job, metrics: &Metrics) -> Result<u64, JobError> {
    metrics.started.fetch_add(1, Ordering::Relaxed);
    info!(items = job.payload.len(), "job started");
    // ...
    error!(error = %e, "job failed");
}
```

- `info!(items = 3, "job started")` records the level, a **message**, and a typed **field** `items`. A machine can filter on `items`; no regex over prose is needed.
- `#[instrument]` wraps the function in a **span** named after it. `skip(job, metrics)` stops it recording every argument automatically (the job contains a credential; the metrics are not interesting) and `fields(...)` adds exactly the ones we choose: the job id and user.
- The `%` sigil means "record with `Display`"; `?` means "record with `Debug`"; a bare value uses its native `Value` implementation (numbers, bools, `&str`).

✓ What a subscriber wrote for a successful job (plain text, no timestamps, captured in a test):

```text
 INFO process{job.id=7 user=ada}: obslab: job started items=3
 INFO process{job.id=7 user=ada}: obslab: job finished sum=6
```

Every event inside the span automatically carries `process{job.id=7 user=ada}`: you never pass the id to inner functions just to log it. A failure is one `ERROR` line with `error=payload is empty`.

**Design guidance.**

- One event per fact; put variable data in fields, not in the message string, so the message is a stable, searchable constant (`"job failed"`, not `format!("job {id} failed")`).
- Levels are a contract with the operator: `ERROR` = needs attention (an operation failed), `WARN` = unexpected but handled (a retry), `INFO` = normal milestones, `DEBUG`/`TRACE` = developer detail, off by default. ✓ in `obslab`, a successful run emits nothing at `warn` or `error`.
- Log an error **once**, where it is handled, not at every layer it passes through (otherwise one failure produces five lines and the counts lie).
- Library code emits events; it does not configure output, read `RUST_LOG`, or initialize a global subscriber. The binary does that once at start-up.

### 3.1 Filtering ✓

A subscriber decides which events are recorded. `tracing-subscriber`'s `EnvFilter` uses the `RUST_LOG` syntax: a comma-separated list of `target=level` directives, where the target is usually the crate or module path.

| Filter string | Effect (✓ checked in tests) |
|---|---|
| `info` | `job started` and `job finished` appear; the per-item `debug!` lines do not |
| `debug` | the two per-item `adding` lines also appear |
| `warn` | nothing, on a successful job |
| `off` | nothing |
| `warn,obslab=info` | everything from `obslab` at `info`, other crates only `warn` and above |
| `info,obslab=error` | `obslab`'s info events silenced |

### 3.2 A gotcha that cost me a flaky test ✓

`tracing` caches, for each log statement ("callsite"), whether any subscriber is interested, so a disabled statement is nearly free (§5). Tests that install **different** scoped subscribers (`tracing::subscriber::with_default`) in parallel threads can race on that cache. In this crate's test file, with eleven tests each installing its own filter, ✓ the suite failed in 2 of the first 7 runs (the `debug` filter test saw only `INFO` lines), then passed 40 of 40 runs without any change and 40 of 40 with the fix, so the failure is intermittent and I cannot give you a rate. The fix is to make the tests that install subscribers run one at a time (a `static Mutex<()>` taken at the start of each test, with `PoisonError::into_inner` so one failed test does not poison the rest). It is a good example of a bug that shows up as a *test* flake: when a logging test fails only sometimes, suspect shared global state before suspecting the code under test.

---

## 4. Testing what you log ✓

Logs are an interface: operators and alerts depend on them, so test them like one. The pattern used in `obslab`:

1. A `MakeWriter` that appends to a shared `Arc<Mutex<Vec<u8>>>`.
2. `tracing_subscriber::fmt().with_env_filter(..).with_writer(capture).with_ansi(false).without_time().finish()`: plain text and no timestamps, so assertions are deterministic.
3. `tracing::subscriber::with_default(subscriber, || { ... })` to scope it to one closure.
4. Assertions on the captured text: the message exists, the field values are there, the line count is exactly what you expect (`assert_eq!(out.lines().count(), 2)` catches an accidental extra or duplicated event).

What the eleven tests in `tests/logging.rs` pin down: start and finish events with their fields; `ERROR` with the error text; filters by level and by target; the retry warnings (`attempt=1 max=5 error=boom 1`), and "giving up" exactly once; metrics counts, including under four threads at once; and the redaction guarantee of §6.

---

## 5. What does logging cost? ✓

`examples/overhead.rs` runs the same loop with a `debug!` event in four situations (release build, 2,000,000 iterations per sample, minimum of 7 samples, nanoseconds per iteration; three runs):

| Situation | ns per event |
|---|---:|
| no log statement at all (the loop only) | 0.26 to 0.30 |
| `debug!` and **no subscriber installed** | 0.36 to 0.43 |
| `debug!` and a subscriber **at INFO** (the event is filtered out) | 0.36 to 0.43 |
| `debug!` **recorded and formatted** into an `io::sink` | 285 to 317 |

(medians of the last row: 326 to 349.) So a disabled statement costs about **0.1 ns more than no statement** (one load and a predictable branch), and an enabled one costs about **three hundred nanoseconds**, a thousandfold difference, and that is formatting into a sink with no I/O. Consequences:

- Leaving `debug!` and `trace!` statements in hot code is cheap *while they are off*. You do not have to delete them before shipping.
- Turning them on in production at high rates is not cheap: 1,000,000 events per second at ~300 ns is 30% of a core, before any real I/O.
- **Do not compute expensive values eagerly for a log field.** The arguments of a disabled macro are not evaluated (from memory: the macro checks first), but a value you computed on the line above the macro is. Put the work inside the macro call, or guard with `tracing::enabled!` (from memory).
- Compile-time level caps (`tracing`'s `max_level_*` and `release_max_level_*` features, from memory; not tested) remove statements entirely.
- Real I/O, a lock around stdout, or JSON encoding costs more than this sink measurement. Production subscribers usually write through a non-blocking writer (`tracing-appender`, from memory); I did not measure one.

---

## 6. Never log a secret ✓

The most common logging security bug is not exotic: someone logs a whole struct with `{:?}` and a credential comes along. Review and discipline fail eventually; the type system can help. `obslab::Redacted<T>`:

```rust
pub struct Redacted<T>(T);

impl<T> Redacted<T> {
    pub fn new(value: T) -> Self { Redacted(value) }
    /// Returns the secret. The name is deliberately loud.
    pub fn expose(&self) -> &T { &self.0 }
}

impl<T> fmt::Debug for Redacted<T> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result { f.write_str("[redacted]") }
}
// and the same for Display
```

`Job` holds `token: Redacted<String>` and derives `Debug`. ✓ Test `the_token_never_appears_even_when_the_whole_job_is_debug_printed`: with the filter at `trace`, `tracing::info!(job = ?j, ...)` and `process(&j, ..)` produce output containing `[redacted]` and **not** the token, for both a valid token and a wrong one (a wrong token is the more dangerous case: users mistype their real password into the token field).

More ways secrets leak (from experience and memory; only the first two are exercised here):

- **`#[derive(Debug)]` on a struct with a secret field** (above). Prefer a wrapper over a hand-written `Debug` for the whole struct, so a new field cannot be forgotten.
- **`#[instrument]` without `skip`**: it records every argument with `Debug`. ✓ A mutant that dropped `job` from `skip(...)` was caught because the span suddenly carried extra fields.
- Secrets in URLs (`?token=...`, basic-auth `user:pass@host`): log the path, not the query string.
- Secrets in error messages (`"login failed for password X"`), panics and backtraces, environment dumps, crash reports.
- Personal data (names, emails, addresses) is regulated in many places: log identifiers, not content. I give no legal advice.
- Remember logs are copied: to files, aggregation services, support tickets. Assume anything logged is readable by everyone who can read logs, for as long as you retain them.

Also note `expose()` is a deliberate escape hatch: wherever it appears, a reviewer should ask "where does this value go?". Automated checks can search for it (`grep expose`).

### 6.1 Log forging ✓

If untrusted text reaches a log message, it can contain a newline, and a line-oriented log then contains a **fake record**. `examples/inject.rs` logs `"bob\n INFO login ok user=admin"` four ways. ✓ Output of the default `fmt` subscriber:

```text
 INFO display field:   user = bob
 INFO login ok user=admin
 INFO Display (%) field user=bob
 INFO login ok user=admin
 INFO Debug (?) field user="bob\n INFO login ok user=admin"
 INFO plain &str field user="bob\n INFO login ok user=admin"
```

Interpolating the text into the message (line 1) or recording it with `%` (`Display`, line 2) writes the newline raw, so a second, forged `INFO login ok user=admin` line appears. Recording with `?` (`Debug`) or as a bare `&str` field escapes it (`\n` shown literally and the value quoted). So: **record untrusted strings as fields, with `?` or bare**, not with `%` and never inside the message; or use a JSON formatter, which escapes by construction (from memory; not run). The same applies to carriage returns and ANSI escape sequences that can fool a terminal.

---

## 7. Metrics: counters ✓

`obslab::Metrics` has three `AtomicU64` counters (`started`, `succeeded`, `failed`) updated with `fetch_add(1, Ordering::Relaxed)` and read by `snapshot()`:

- **`Relaxed` is enough** because each counter is an independent number; no other memory is published through it (Lesson 28). ✓ Four threads, 250 jobs each: the totals are exactly `started: 1000, succeeded: 1000, failed: 0`.
- `snapshot()` reads three atomics one after the other: they are not a transaction, so `started - succeeded - failed` can be briefly nonzero while jobs are in flight. That is fine for monitoring; it would not be fine for accounting.
- Monotonic counters are better than "current rate" numbers: the monitoring system computes rates and survives missed scrapes.
- **Cardinality**: a metric labelled by `user_id` creates one time series per user and can overwhelm a monitoring system (from memory). Labels should come from small fixed sets (status, route, error kind).
- Histograms (latency distributions) and gauges (queue depth) are the other basic types; I implemented neither. In practice you would use a metrics crate that exposes them in a standard format (from memory; none used here).
- Do not put a lock on the hot path of a metric update; atomics (or per-thread shards, for extreme rates) keep it at a few nanoseconds (Lesson 41's discipline: measure before claiming).

---

## 8. Continuous integration

The goal: the checks a maintainer would run before merging run automatically on every change, on a clean machine, and block the merge on failure. The commands are ordinary Cargo commands; ✓ `obslab/ci.sh` runs them in order and stops at the first failure:

```text
== format            cargo fmt --all -- --check
== lints             cargo clippy --all-targets --locked -- -D warnings
== tests             cargo test --locked
== docs              cargo doc --no-deps --locked
== release build     cargo build --release --locked
== duplicate crates  cargo tree --locked -d --target all   (fails if any crate appears in two versions)
all checks passed
```

✓ It passes on `obslab` and took about 31 seconds on this machine (including compiling the release build of the dependencies). The environment variables `RUSTFLAGS="-D warnings"` and `RUSTDOCFLAGS="-D warnings"` turn every compiler and rustdoc warning into an error.

### 8.1 What each flag is for

- **`--locked`**: refuses to build if `Cargo.lock` would have to change. ✓ I added a dependency to `Cargo.toml` without updating the lock: `cargo build --locked` stopped with `error: cannot update the lock file ... because --locked was passed to prevent this`. Without it, CI would silently pick newer dependency versions than the ones you tested.
- **`cargo fmt --check`** exits non-zero on any formatting difference instead of rewriting files.
- **`-D warnings`**: keeps the codebase warning-free. The risk is that a **new compiler release adds a lint** and breaks CI without any change in your code. Common mitigations (from memory): pin the toolchain in `rust-toolchain.toml` and update it deliberately; apply `-D warnings` only in the lint job, not to every build; or run `clippy -- -D warnings` on a pinned version while building with the plain command elsewhere. Also note that `RUSTFLAGS` changes cause a rebuild of dependencies and use a different cache key.
- **Duplicate-crate check**: two versions of one crate in the tree mean extra compile time and binary size (and sometimes subtle type mismatches). ✓ `obslab` has none (`cargo tree -d` prints `warning: nothing to print`).
- **Doc build**: broken intra-doc links and invalid code in doc comments fail CI, and doctests run under `cargo test`.

### 8.2 The minimum supported Rust version (MSRV)

`rust-version = "1.85"` in `Cargo.toml` is a promise about the oldest compiler that builds the crate. ✓ Cargo enforces it: setting `rust-version = "1.99"` made `cargo build` fail with `error: rustc 1.97.0 is not supported by the following package: obslab@0.1.0 requires rustc 1.99`. ✓ With an MSRV set, the resolver is MSRV-aware: the lock file was generated with `Locking 19 packages to latest Rust 1.85 compatible versions`, i.e. it avoided dependency versions that need a newer compiler (this is the default resolver behavior of edition 2024, from memory). **Declaring** an MSRV is not **testing** it: the `msrv` job in the workflow below builds with exactly that toolchain so the promise cannot silently break. I could not run that job here (only one toolchain installed).

### 8.3 A workflow file (not run)

`.github/workflows/ci.yml`, in the practical challenge, has three jobs: `check` (fmt, clippy, test, doc), `msrv` (build on the declared minimum version), and `deny` (§9). Properties worth copying, all from memory because I could not run GitHub Actions:

- `permissions: contents: read` at the top: the default token should be able to do nothing but read.
- `on: push` to the main branch and `pull_request`, so a pull request is tested before it merges.
- Every third-party action is a piece of **code you run with access to your repository**. Pinning to a full commit hash instead of a movable tag such as `@v4` prevents a compromised tag from changing what you run; I used tags in the sample for readability, which is the weaker choice.
- Never expose secrets to jobs triggered by untrusted pull requests; avoid `pull_request_target` with a checkout of the pull request's code (a classic privilege-escalation pattern).
- Cache the Cargo registry and `target/` (a cache action) to keep runs fast; the cache key should include `Cargo.lock`.
- A matrix over stable, beta and the MSRV, a scheduled run that catches new compiler lints early, `cargo test --doc`, miri for `unsafe` crates (Lesson 38) and a coverage report are common additions I did not run.

---

## 9. Dependency security

### 9.1 What you actually depend on ✓

`obslab` asks for one dependency, `tracing`. ✓ `cargo tree --locked` shows the real graph: `tracing` pulls in `tracing-core`, `pin-project-lite`, `once_cell`, and the proc macro `tracing-attributes`, which brings `syn`, `quote`, `proc-macro2` and `unicode-ident`. For the library alone that is **9 packages**; the tests add `tracing-subscriber` and **8 more** (regex engine, sharded slab, thread-locals, ...), for 17 in total on this platform. `Cargo.lock` lists 19 checksummed packages (the lock file records the full resolution, including packages that are not needed on this platform or feature set; I did not investigate which ones). ✓ `cargo tree -i regex-syntax` shows why a package is present (inverted tree): here only through `tracing-subscriber`, a dev-dependency, so users of the library never build it.

Every package in that tree is code that runs **with your program's privileges**, and some of it runs **at build time**: proc macros (Lesson 35) and `build.rs` scripts (Lesson 37) execute arbitrary code on the developer's or CI machine when you compile. `cargo build` is therefore an act of trust.

### 9.2 The lock file is a security feature ✓

`Cargo.lock` pins each dependency to an exact version **and a checksum of its source**. ✓ I changed one hex digit of the recorded checksum for `once_cell` and ran `cargo build --locked`: it refused with `error: checksum for once_cell v1.21.4 changed between lock files` and listed the possible causes (corrupt lock file, a mirror returning different content, a corrupt source). So a registry or mirror that serves different bytes for the same version is detected. Rules that follow:

- **Commit `Cargo.lock` for binaries** (applications, services, tools). For libraries it is also committed in many projects for reproducible CI, though downstream users resolve their own (from memory).
- **Build with `--locked`** in CI and release builds.
- Review lock-file changes in pull requests like code: a new package or a version jump is a decision. `cargo update --dry-run` shows what an update would change ✓ (`Locking 0 packages to latest Rust 1.85 compatible versions`).
- `cargo vendor` copies every locked dependency into a directory you can inspect, commit and build offline. ✓ For `obslab` it wrote 19 crate directories, 23 MB. It is how you audit exactly what you ship and how you build in a network-isolated environment.

### 9.3 `cargo-deny` ✓

`cargo deny check` enforces a written policy, `deny.toml`, over the whole dependency graph, with four checks:

| Check | Fails when | ✓ What I ran |
|---|---|---|
| **advisories** | a package version has a published security advisory (RustSec database), is yanked, or is flagged unmaintained | a scratch crate that depends on `time = "=0.1.45"` failed with `error[vulnerability]: Potential segfault in the time crate`, ID `RUSTSEC-2020-0071`, and the advisory text (affected functions set environment variables without synchronization; the fix is to move to `time` 0.2.23 or later, or 0.3) |
| **licenses** | a package's license is not on your allow-list | with the default empty allow-list the very first run failed (`error[rejected]: failed to satisfy license requirements` for `unicode-ident`'s `(MIT OR Apache-2.0) AND Unicode-3.0`); after allowing `MIT`, `Apache-2.0`, `Unicode-3.0` it passed; removing `Unicode-3.0` made it fail again |
| **bans** | a banned crate appears, or (as configured) two versions of one crate, or a wildcard version requirement `*` | adding `{ crate = "syn" }` to `deny` failed with `error[banned]: crate 'syn = 2.0.119' is explicitly banned` and the chain of packages that pull it in |
| **sources** | a package comes from an unknown registry or git repository | passed; unknown sources are denied by the policy |

✓ The final policy on `obslab` prints `advisories ok, bans ok, licenses ok, sources ok`. Notes:

- A *vulnerability* means "somebody reported a problem in this version", not "your program is exploitable": the affected function may be unused. It is a prompt to investigate or upgrade, and an ignore entry should carry a reason and a review date.
- A clean report means "no **known** problems": new advisories are published for code that did not change. Run the check **on a schedule** (for example weekly) as well as on every change, or you will not see a new advisory until the next commit.
- The advisory data came from the network at run time; an offline CI cannot check advisories.
- `cargo-audit` is the other widely used tool for the same database. ✓ `cargo audit` loaded 1,422 advisories and scanned `obslab`'s lock file (20 crate entries) with no findings; on the `time = "=0.1.45"` scratch crate it reported `RUSTSEC-2020-0071`, `Solution: Upgrade to >=0.2.23` and `error: 1 vulnerability found!`. `cargo-deny` adds the license, ban and source checks on top of the advisories.
- `cargo-deny` and `cargo-audit` check **known** problems and policy. They do not read the code. They would not have caught a malicious new package that nobody has reported yet.

### 9.4 What no scanner covers, and what helps

- **Typosquatting and name confusion**: a package named like a popular one (`serde_json` vs `serde-json`), or an unfamiliar maintainer. Check the repository link, the download count and the maintainer before adding a dependency.
- **Compromised maintainer or release**: a legitimate crate publishes a malicious version. Delay: do not auto-merge brand-new versions; read changelogs and diffs for sensitive packages.
- **Abandoned crates**: no fix will ever come. The advisories check can flag some as unmaintained; also look at the date of the last release.
- **Build-time code**: proc macros and `build.rs` run unsandboxed. Build in an isolated CI environment with no secrets mounted, and prefer fewer, smaller dependencies.
- **Unreviewed `unsafe` in dependencies**: tools such as `cargo geiger` count it, and review systems such as `cargo vet` and `cargo crev` record who has audited which version (from memory; none run here).
- **Your own releases**: protect the account that publishes (two-factor authentication), prefer short-lived tokens or trusted publishing to long-lived secrets in CI, and sign or checksum release artifacts. `cargo package --list` ✓ shows exactly which files would be published (`Cargo.lock`, `Cargo.toml`, `Cargo.toml.orig`, the sources...); it also warned `manifest has no description, license, license-file, documentation, homepage or repository`, a reminder that `publish = false` was the right setting for this teaching crate.
- **Fewer dependencies is the cheapest control.** Every package you do not add is a package nobody can compromise. Disable default features you do not use (`default-features = false`), as `tracing-subscriber` is in this crate's `Cargo.toml`.

---

## 10. Performance, reliability, security: summary

- **Observability is a feature with a budget**: ✓ ~300 ns per recorded event; ~0.1 ns per disabled one. Keep hot paths at `debug`/`trace` and off by default.
- **Reliability**: log failures once where they are handled; count outcomes (✓ exactly under concurrency); make retries visible (✓ one `warn` per failed attempt, one `error` when giving up).
- **Security**: secrets by construction (✓ `Redacted`), untrusted text as escaped fields (✓ log forging demo), `--locked` builds with checksums (✓), a dependency policy enforced by a tool (✓), and scheduled re-checks because the advisory database changes.
- **Honest limits**: scanners find known problems; logs can still contain data you did not think of; none of this makes a system secure by itself.

---

## 11. Common mistakes

1. Logging a whole struct with `{:?}` when it contains a credential.
2. `#[instrument]` with no `skip` on a function whose arguments include secrets or large buffers.
3. Pasting variable data into the message (`format!`), which defeats filtering and aggregation.
4. Recording untrusted strings with `%` or in the message (✓ forged log lines).
5. Logging the same error at every layer.
6. Initializing a global subscriber inside a library.
7. Tests that install different subscribers in parallel without serialization (✓ intermittent failure).
8. Expensive work outside the log macro that runs even when the level is disabled.
9. CI without `--locked`, so it tests different versions than you did.
10. Treating "no advisories found" as "safe", or running the check only when code changes.
11. Declaring an MSRV and never building with it.
12. Third-party CI actions referenced by movable tags with broad token permissions.

---

## 12. Exercises

Answers are collapsed. Try each before opening it.

1. **(Warm-up)** Which signal (log, metric, trace) would you use to: (a) alert when more than 5% of requests fail; (b) find which of five internal steps made one request take 8 seconds; (c) see the exact error text for one failed request?
2. **(Warm-up)** In the overhead table, why is the cost of a `debug!` statement with a subscriber at INFO about the same as with no subscriber at all?
3. **(Easy)** Write a `Redacted<T>` wrapper (standard library only) whose `Debug` and `Display` never show the value, and show that a derived `Debug` struct containing it prints `[redacted]`.
4. **(Medium)** Why can the `level_filters_decide_what_is_recorded` test fail intermittently when tests run in parallel, and what is the fix?
5. **(Medium)** Write `escape_for_log(&str) -> String` that makes untrusted text safe to place in a one-line log record.
6. **(Medium-hard)** Your CI sets `RUSTFLAGS="-D warnings"` for every job. Name two ways this can hurt you and one safer arrangement.
7. **(Hard)** `cargo deny check` is green. A new release of a dependency, published an hour ago, contains a malicious `build.rs`. Does the check protect you? What would?
8. **(Hard)** Design the dependency-security part of a release process for a command-line tool published to crates.io.

<details markdown="1">
<summary>Answers</summary>

1. (a) A **metric**: a counter of requests and a counter of failures; the monitoring system computes the ratio and alerts. (b) A **trace**: a span per step shows each step's duration for that request. (c) A **log** event with the error as a field, found by filtering on the request id.
2. Because the macro first checks, with a cheap cached test, whether any subscriber wants a `debug` event at this callsite. In both cases the answer is "no", so no arguments are evaluated and nothing is formatted (✓ 0.36 to 0.43 ns both, about 0.1 ns above the loop with no statement). The cost appears only when an interested subscriber records and formats the event (✓ 285 to 317 ns).
3. A possible answer:

   ```rust
   use std::fmt;

   struct Redacted<T>(T);

   impl<T> Redacted<T> {
       fn expose(&self) -> &T {
           &self.0
       }
   }

   impl<T> fmt::Debug for Redacted<T> {
       fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
           f.write_str("[redacted]")
       }
   }

   impl<T> fmt::Display for Redacted<T> {
       fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
           f.write_str("[redacted]")
       }
   }

   #[derive(Debug)]
   struct Login {
       user: String,
       password: Redacted<String>,
   }

   fn main() {
       let l = Login { user: "ada".into(), password: Redacted("hunter2".into()) };
       let shown = format!("{l:?}");
       assert_eq!(shown, r#"Login { user: "ada", password: [redacted] }"#);
       assert!(!shown.contains("hunter2"));
       assert_eq!(format!("{}", l.password), "[redacted]");
       assert_eq!(l.password.expose(), "hunter2");
       assert_eq!(l.user, "ada");
       println!("{shown}");
   }
   ```

   The derive on `Login` is safe because the field's own `Debug` is the redacting one.
4. Each log statement caches whether any subscriber is interested. Tests that install different scoped subscribers (one at `info`, one at `debug`, one `off`) at the same moment on different threads can race on that cache, so the `debug` test sometimes sees only `INFO` lines (✓ 2 failures in the first 7 runs). The fix is to serialize the tests that install subscribers with a shared `Mutex` (✓ 40 of 40 runs passed afterwards). Another option is a single test that sets the filters one after another, or a dedicated test binary per filter.
5. One approach is to escape every control character:

   ```rust
   fn escape_for_log(s: &str) -> String {
       let mut out = String::with_capacity(s.len());
       for c in s.chars() {
           if c.is_control() {
               out.extend(c.escape_default());
           } else {
               out.push(c);
           }
       }
       out
   }

   fn main() {
       let input = "bob\n INFO login ok user=admin\r\x1b[31m";
       let safe = escape_for_log(input);
       assert!(!safe.contains('\n') && !safe.contains('\r') && !safe.contains('\x1b'));
       assert_eq!(safe, "bob\\n INFO login ok user=admin\\r\\u{1b}[31m");
       assert_eq!(escape_for_log("plain text"), "plain text");
       println!("{safe}");
   }
   ```

   In `tracing`, recording the value with `?` or as a bare `&str` field already does this (✓ the demo), so prefer that to writing your own.
6. (a) A new compiler release can introduce a lint and turn every build red without any change from you. (b) `RUSTFLAGS` changes force a full rebuild and use a different cache key, slowing CI, and (from memory) flags set globally can also apply to build scripts and proc macros. Safer: pin the toolchain with `rust-toolchain.toml`, update it deliberately in its own pull request, and apply `-D warnings` only in the clippy job (`cargo clippy -- -D warnings`) while other jobs build normally.
7. No. `cargo deny` checks known advisories, licenses, bans and sources; a one-hour-old malicious release has no advisory, and the license and source can look perfectly normal. Mitigations: `--locked` builds so a new release is not picked up until someone updates the lock file on purpose; review lock-file changes and diffs of sensitive dependencies (`cargo vendor` makes diffs easy); a cool-down policy before adopting brand-new versions; build in an isolated CI job with no secrets, because `build.rs` and proc macros run with the build's privileges; and use audit records such as `cargo vet` where available.
8. For example: commit `Cargo.lock`; build and test with `--locked`; enforce `deny.toml` on every pull request and on a weekly schedule; review dependency updates as separate pull requests with lock-file diffs, optionally with a delay; keep dependencies few and default features off; build releases in a clean CI environment with no long-lived secrets (short-lived or trusted-publishing tokens, two-factor authentication on the publishing account); package with `cargo package --list` to see exactly what is published; publish checksums or signatures of release binaries; and have a documented response for when an advisory lands (who is notified, how a fix release ships).

</details>

---

## 13. Debugging challenge

A reviewer finds that production logs contain customers' API keys. The code below simulates the logging with `println!`. Find every leak and fix them without removing the useful information from the logs.

```rust
#[derive(Debug)]
struct Request {
    user: String,
    api_key: String,
    url: String,
}

fn handle(r: &Request) {
    println!("INFO handling request: {:?}", r);
    println!("INFO GET {}", r.url);
    if r.api_key.len() < 8 {
        println!("WARN bad key '{}' for user {}", r.api_key, r.user);
    }
}

fn main() {
    handle(&Request {
        user: "ada".into(),
        api_key: "k_live_12345".into(),
        url: "https://api.example.com/v1/items?page=2&token=k_live_12345".into(),
    });
    handle(&Request {
        user: "bob".into(),
        api_key: "short".into(),
        url: "https://api.example.com/v1/items".into(),
    });
}
```

<details markdown="1">
<summary>Solution</summary>

Four leaks: (1) the derived `Debug` prints `api_key` (and the whole URL, with its token); (2) the URL contains the key in the query string (`token=...`); (3) the warning prints the bad key itself (a mistyped key is often a near-miss of a real one); (4) the user name is printed raw (a minor injection issue, §6.1: escape it). Fix: wrap the key in a redacting type, log the URL without its query string (the path is what an operator needs), and say *that* the key is bad, not what it is:

```rust
use std::fmt;

struct Redacted<T>(T);

impl<T> Redacted<T> {
    fn expose(&self) -> &T {
        &self.0
    }
}

impl<T> fmt::Debug for Redacted<T> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.write_str("[redacted]")
    }
}

struct Request {
    user: String,
    api_key: Redacted<String>,
    url: String,
}

/// Everything before the first `?` or `#`: no query string, no fragment.
fn without_query(url: &str) -> &str {
    url.split(['?', '#']).next().unwrap_or(url)
}

fn handle(r: &Request) -> Vec<String> {
    // An allow-list of fields, not `{:?}` of the whole struct: the URL (with
    // its query string) must not be printed either.
    let mut lines = vec![format!(
        "INFO handling request: user={:?} key={:?} path={}",
        r.user,
        r.api_key,
        without_query(&r.url)
    )];
    if r.api_key.expose().len() < 8 {
        lines.push(format!("WARN bad key (length {}) for user {:?}", r.api_key.expose().len(), r.user));
    }
    lines
}

fn main() {
    let a = handle(&Request {
        user: "ada".into(),
        api_key: Redacted("k_live_12345".into()),
        url: "https://api.example.com/v1/items?page=2&token=k_live_12345".into(),
    });
    let b = handle(&Request {
        user: "bob\nINFO forged".into(),
        api_key: Redacted("short".into()),
        url: "https://api.example.com/v1/items".into(),
    });
    for line in a.iter().chain(&b) {
        assert!(!line.contains("k_live_12345"), "{line}");
        assert!(!line.contains("short'"), "{line}");
        assert!(!line.contains('\n'), "{line}");
        println!("{line}");
    }
}
```

The log lines keep what operators need (who, which path, that a key was too short and how short) and lose what attackers need. With `tracing`, use `#[instrument(skip(request))]` plus explicit `fields(...)`, and record the user name as a field with `?`.

</details>

---

## 14. Interview questions

1. *What is the difference between logs, metrics and traces? When do you use each?*
2. *Why should a library not initialize a logger?*
3. *What is a structured log, and why is it better than a formatted string?*
4. *How do you keep a secret out of logs in Rust?*
5. *What is log injection, and how do you prevent it?*
6. *What should a Rust CI pipeline run on every change?*
7. *What does `Cargo.lock` protect against, and what does `--locked` do?*
8. *What does `cargo-deny` check, and what can it not tell you?*

<details markdown="1">
<summary>Answers</summary>

1. Logs record individual events with context (explain one case); metrics are cheap numbers aggregated over time (detect a problem and alert); traces are timed trees of spans across steps or services (locate where time goes). Metrics notice, traces locate, logs explain.
2. Output configuration is the application's decision (format, destination, level, rate). If a library installs a global logger it conflicts with the application's choice and can only be done once per process. A library emits events through a facade (`tracing`, `log`) and does nothing else.
3. Fields are typed key-value pairs attached to a constant message, so tools can filter, group and count by field without parsing prose; a formatted string hides the data inside the text (✓ `items=3` as a field, `"job started"` constant).
4. By construction: wrap secrets in a type whose `Debug` and `Display` print a placeholder (✓ `Redacted`), `skip` secret arguments in `#[instrument]`, keep secrets out of URLs and error messages, and treat any `expose()` call as a review point. Discipline alone fails.
5. Untrusted text containing a newline (or terminal escape sequences) can forge or hide records. Record untrusted values as escaped fields (`?` or bare `&str` in `tracing`: ✓ escaped; `%` and message interpolation: ✓ raw newline written), or use a structured (JSON) formatter that escapes.
6. At least: formatting check, clippy with warnings denied, the full test suite (including doctests), a documentation build, and a locked release build; plus dependency policy checks and a build on the declared minimum Rust version. Run with `--locked` on a clean machine, and on a schedule too.
7. The lock file pins exact versions and source checksums, so builds are reproducible and a tampered or substituted download is detected (✓ a one-digit checksum change was refused). `--locked` makes Cargo fail instead of updating the lock file, so CI tests exactly what was reviewed.
8. Four checks: known security advisories (and yanked or unmaintained packages), allowed licenses, banned or duplicate crates, and permitted sources. It reports only what is **known** and encoded in the policy: it does not read the code, and cannot detect an unreported malicious release or a vulnerability that does not yet have an advisory.

</details>

---

## 15. Practical challenge: `obslab` and its CI and policy files

Build the library crate `obslab` and its project files. Requirements:

1. `Redacted<T>` (never prints the value), `Job` with a redacted token, `JobError` implementing `Display` and `Error`, `Metrics`/`Snapshot` with relaxed atomic counters, `process` (instrumented span; start, finish and failure events; counters) and `retry` (a `warn` per retried failure, an `error` when giving up, `attempts == 0` treated as one attempt).
2. Tests that capture log output and check: events and span fields, filters by level and target, that the token never appears (even when the whole job is debug-printed), exact metrics (also across four threads), retry behavior, overflow handling.
3. An overhead example, a log-injection example, and a `ci.sh` that mirrors CI.
4. A `deny.toml` policy (advisories, licenses, bans, sources) that passes on the crate, and a GitHub Actions workflow with `check`, `msrv` and `deny` jobs.
5. Run your own mutation checks, and say which survive and why.

<details markdown="1">
<summary>Solution</summary>

`Cargo.toml`:

```toml
[package]
name = "obslab"
version = "0.1.0"
edition = "2024"
rust-version = "1.85"
publish = false

[dependencies]
tracing = "0.1"

[dev-dependencies]
tracing-subscriber = { version = "0.3", default-features = false, features = ["fmt", "env-filter", "std"] }
```

`src/lib.rs`:

````rust
//! `obslab`: observability for a tiny job processor.
//!
//! Three ideas, each small and testable:
//!
//! * **Structured events and spans** with the `tracing` crate: every record
//!   has a level, a message and typed fields, and spans carry context.
//! * **Redaction by construction**: secrets are wrapped in [`Redacted`], whose
//!   `Debug` and `Display` never print the value, so logging a whole struct
//!   cannot leak it.
//! * **Metrics as counters** ([`Metrics`]): cheap, lock-free totals that a
//!   monitoring system can scrape.
//!
//! ```
//! use obslab::Redacted;
//!
//! let token = Redacted::new("s3cret".to_string());
//! assert_eq!(format!("{token:?}"), "[redacted]");
//! assert_eq!(token.expose(), "s3cret");
//! ```

#![deny(missing_docs)]

use std::fmt;
use std::sync::atomic::{AtomicU64, Ordering};
use tracing::{debug, error, info, instrument, warn};

/// A value that must never appear in logs, error messages or debug output.
#[derive(Clone, PartialEq, Eq)]
pub struct Redacted<T>(T);

impl<T> Redacted<T> {
    /// Wraps a secret.
    pub fn new(value: T) -> Self {
        Redacted(value)
    }

    /// Returns the secret. The name is deliberately loud: every call site is
    /// a place a reviewer should look at.
    pub fn expose(&self) -> &T {
        &self.0
    }
}

impl<T> fmt::Debug for Redacted<T> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.write_str("[redacted]")
    }
}

impl<T> fmt::Display for Redacted<T> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.write_str("[redacted]")
    }
}

/// A unit of work.
#[derive(Debug, Clone)]
pub struct Job {
    /// Identifier, safe to log.
    pub id: u64,
    /// Account name, safe to log.
    pub user: String,
    /// Credential used to authorize the job. Never logged.
    pub token: Redacted<String>,
    /// Numbers to add up.
    pub payload: Vec<u32>,
}

/// Why a job failed.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum JobError {
    /// The payload had no items.
    EmptyPayload,
    /// The sum does not fit in a `u64`.
    Overflow,
    /// The token was rejected.
    Unauthorized,
}

impl fmt::Display for JobError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            JobError::EmptyPayload => f.write_str("payload is empty"),
            JobError::Overflow => f.write_str("sum overflows u64"),
            JobError::Unauthorized => f.write_str("token rejected"),
        }
    }
}

impl std::error::Error for JobError {}

/// Lock-free counters for monitoring.
#[derive(Debug, Default)]
pub struct Metrics {
    started: AtomicU64,
    succeeded: AtomicU64,
    failed: AtomicU64,
}

/// A point-in-time copy of the counters.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Snapshot {
    /// Jobs begun.
    pub started: u64,
    /// Jobs that returned `Ok`.
    pub succeeded: u64,
    /// Jobs that returned `Err`.
    pub failed: u64,
}

impl Metrics {
    /// Creates zeroed counters.
    pub fn new() -> Self {
        Self::default()
    }

    /// Reads the counters. Each is read atomically; the three together are
    /// not a consistent transaction, which is fine for monitoring.
    pub fn snapshot(&self) -> Snapshot {
        Snapshot {
            started: self.started.load(Ordering::Relaxed),
            succeeded: self.succeeded.load(Ordering::Relaxed),
            failed: self.failed.load(Ordering::Relaxed),
        }
    }
}

/// The token this demo accepts.
const VALID_TOKEN: &str = "let-me-in";

/// Runs one job. The span `process` carries the job id and user for every
/// event inside it; the job itself (which holds the token) is skipped.
#[instrument(skip(job, metrics), fields(job.id = job.id, user = %job.user))]
pub fn process(job: &Job, metrics: &Metrics) -> Result<u64, JobError> {
    metrics.started.fetch_add(1, Ordering::Relaxed);
    info!(items = job.payload.len(), "job started");
    let result = run(job);
    match &result {
        Ok(sum) => {
            metrics.succeeded.fetch_add(1, Ordering::Relaxed);
            info!(sum, "job finished");
        }
        Err(e) => {
            metrics.failed.fetch_add(1, Ordering::Relaxed);
            error!(error = %e, "job failed");
        }
    }
    result
}

/// Adds without wrapping, reporting [`JobError::Overflow`] instead.
pub fn add(sum: u64, x: u64) -> Result<u64, JobError> {
    sum.checked_add(x).ok_or(JobError::Overflow)
}

fn run(job: &Job) -> Result<u64, JobError> {
    if job.token.expose() != VALID_TOKEN {
        return Err(JobError::Unauthorized);
    }
    if job.payload.is_empty() {
        return Err(JobError::EmptyPayload);
    }
    let mut sum = 0u64;
    for (i, &x) in job.payload.iter().enumerate() {
        debug!(index = i, value = x, "adding");
        sum = add(sum, u64::from(x))?;
    }
    Ok(sum)
}

/// Calls `op` up to `attempts` times, logging a warning for each failure
/// that will be retried and returning the last error if all attempts fail.
/// `attempts == 0` is treated as one attempt.
pub fn retry<T, E: fmt::Display>(
    attempts: u32,
    mut op: impl FnMut(u32) -> Result<T, E>,
) -> Result<T, E> {
    let attempts = attempts.max(1);
    let mut attempt = 1;
    loop {
        match op(attempt) {
            Ok(v) => return Ok(v),
            Err(e) if attempt < attempts => {
                warn!(attempt, max = attempts, error = %e, "attempt failed, retrying");
                attempt += 1;
            }
            Err(e) => {
                error!(attempt, error = %e, "giving up");
                return Err(e);
            }
        }
    }
}
````

`tests/logging.rs`:

```rust
use obslab::{Job, JobError, Metrics, Redacted, Snapshot, add, process, retry};
use std::io;
use std::sync::{Arc, Mutex, MutexGuard, PoisonError};
use tracing_subscriber::fmt::MakeWriter;

/// Collects everything the subscriber writes so a test can inspect it.
#[derive(Clone, Default)]
struct Capture(Arc<Mutex<Vec<u8>>>);

struct CaptureWriter(Arc<Mutex<Vec<u8>>>);

impl io::Write for CaptureWriter {
    fn write(&mut self, buf: &[u8]) -> io::Result<usize> {
        self.0.lock().expect("capture lock").extend_from_slice(buf);
        Ok(buf.len())
    }
    fn flush(&mut self) -> io::Result<()> {
        Ok(())
    }
}

impl<'a> MakeWriter<'a> for Capture {
    type Writer = CaptureWriter;
    fn make_writer(&'a self) -> Self::Writer {
        CaptureWriter(Arc::clone(&self.0))
    }
}

impl Capture {
    fn text(&self) -> String {
        String::from_utf8(self.0.lock().expect("capture lock").clone()).expect("utf-8 logs")
    }
}

static SERIAL: Mutex<()> = Mutex::new(());

/// `tracing` caches, per call site, whether any subscriber is interested.
/// Tests that install different filters at the same moment can race on that
/// cache and miss events (it failed 1 run in 6 before this lock existed), so
/// every test in this file runs one at a time.
fn serial() -> MutexGuard<'static, ()> {
    SERIAL.lock().unwrap_or_else(PoisonError::into_inner)
}

/// Runs `f` with a subscriber that writes plain text (no time, no colors)
/// into a buffer, filtered by `filter` (same syntax as `RUST_LOG`).
fn captured(filter: &str, f: impl FnOnce()) -> String {
    let cap = Capture::default();
    let subscriber = tracing_subscriber::fmt()
        .with_env_filter(filter)
        .with_writer(cap.clone())
        .with_ansi(false)
        .without_time()
        .finish();
    tracing::subscriber::with_default(subscriber, f);
    cap.text()
}

fn job(id: u64, token: &str, payload: Vec<u32>) -> Job {
    Job {
        id,
        user: "ada".into(),
        token: Redacted::new(token.into()),
        payload,
    }
}

#[test]
fn successful_job_logs_start_and_finish_with_span_fields() {
    let _guard = serial();
    let m = Metrics::new();
    let out = captured("info", || {
        assert_eq!(process(&job(7, "let-me-in", vec![1, 2, 3]), &m), Ok(6));
    });
    assert!(out.contains("job started"), "{out}");
    assert!(out.contains("job finished"), "{out}");
    assert!(out.contains("items=3"), "{out}");
    assert!(out.contains("sum=6"), "{out}");
    // The span's fields are printed in front of every event inside it.
    assert!(out.contains("process{job.id=7 user=ada}"), "{out}");
    assert_eq!(out.lines().count(), 2, "{out}");
}

#[test]
fn failures_log_at_error_level_with_the_error_text() {
    let _guard = serial();
    let m = Metrics::new();
    let out = captured("info", || {
        assert_eq!(
            process(&job(8, "let-me-in", vec![]), &m),
            Err(JobError::EmptyPayload)
        );
    });
    assert!(out.contains("ERROR"), "{out}");
    assert!(out.contains("error=payload is empty"), "{out}");
}

#[test]
fn the_token_never_appears_even_when_the_whole_job_is_debug_printed() {
    let _guard = serial();
    let m = Metrics::new();
    let j = job(9, "let-me-in", vec![1]);
    let out = captured("trace", || {
        tracing::info!(job = ?j, "dumping the whole job");
        let _ = process(&j, &m);
    });
    assert!(out.contains("[redacted]"), "{out}");
    assert!(!out.contains("let-me-in"), "{out}");
    let bad = job(10, "wrong-token-123", vec![1]);
    let out = captured("trace", || {
        assert_eq!(process(&bad, &m), Err(JobError::Unauthorized));
    });
    assert!(!out.contains("wrong-token-123"), "{out}");
    assert_eq!(format!("{}", bad.token), "[redacted]");
    assert_eq!(format!("{:?}", bad.token), "[redacted]");
    assert_eq!(bad.token.expose(), "wrong-token-123");
}

#[test]
fn level_filters_decide_what_is_recorded() {
    let _guard = serial();
    let m = Metrics::new();
    let j = job(1, "let-me-in", vec![5, 6]);
    let info = captured("info", || {
        let _ = process(&j, &m);
    });
    let debug = captured("debug", || {
        let _ = process(&j, &m);
    });
    let warn = captured("warn", || {
        let _ = process(&j, &m);
    });
    let off = captured("off", || {
        let _ = process(&j, &m);
    });
    assert!(!info.contains("adding"), "{info}");
    assert_eq!(debug.matches("adding").count(), 2, "{debug}");
    assert!(
        warn.is_empty(),
        "no warn or error events on success: {warn}"
    );
    assert!(off.is_empty());
}

#[test]
fn target_filters_can_silence_one_crate() {
    let _guard = serial();
    let m = Metrics::new();
    let j = job(1, "let-me-in", vec![5]);
    let out = captured("warn,obslab=info", || {
        let _ = process(&j, &m);
    });
    assert!(out.contains("job started"), "{out}");
    let out = captured("info,obslab=error", || {
        let _ = process(&j, &m);
    });
    assert!(out.is_empty(), "{out}");
}

#[test]
fn metrics_count_every_outcome() {
    let _guard = serial();
    let m = Metrics::new();
    assert_eq!(
        m.snapshot(),
        Snapshot {
            started: 0,
            succeeded: 0,
            failed: 0
        }
    );
    captured("off", || {
        assert!(process(&job(1, "let-me-in", vec![1]), &m).is_ok());
        assert!(process(&job(2, "let-me-in", vec![]), &m).is_err());
        assert!(process(&job(3, "bad", vec![1]), &m).is_err());
        assert_eq!(
            process(&job(4, "let-me-in", vec![u32::MAX; 3]), &m),
            Ok(3 * u64::from(u32::MAX))
        );
    });
    assert_eq!(
        m.snapshot(),
        Snapshot {
            started: 4,
            succeeded: 2,
            failed: 2
        }
    );
}

#[test]
fn metrics_are_correct_under_concurrency() {
    let _guard = serial();
    let m = Metrics::new();
    std::thread::scope(|s| {
        for t in 0..4u64 {
            let m = &m;
            s.spawn(move || {
                for i in 0..250u64 {
                    let j = job(t * 1000 + i, "let-me-in", vec![1, 2]);
                    assert_eq!(process(&j, m), Ok(3));
                }
            });
        }
    });
    assert_eq!(
        m.snapshot(),
        Snapshot {
            started: 1000,
            succeeded: 1000,
            failed: 0
        }
    );
}

#[test]
fn retry_logs_each_failed_attempt_and_stops_on_success() {
    let _guard = serial();
    let mut seen = Vec::new();
    let out = captured("info", || {
        let r: Result<&str, String> = retry(5, |n| {
            seen.push(n);
            if n < 3 {
                Err(format!("boom {n}"))
            } else {
                Ok("done")
            }
        });
        assert_eq!(r, Ok("done"));
    });
    assert_eq!(seen, vec![1, 2, 3]);
    assert_eq!(out.matches("attempt failed, retrying").count(), 2, "{out}");
    assert!(out.contains("attempt=1 max=5 error=boom 1"), "{out}");
    assert!(!out.contains("giving up"));
}

#[test]
fn retry_gives_up_after_the_limit_and_zero_means_one_attempt() {
    let _guard = serial();
    let mut calls = 0;
    let out = captured("info", || {
        let r: Result<(), String> = retry(3, |n| {
            calls += 1;
            Err(format!("nope {n}"))
        });
        assert_eq!(r, Err("nope 3".to_string()));
    });
    assert_eq!(calls, 3);
    assert_eq!(out.matches("attempt failed, retrying").count(), 2, "{out}");
    assert_eq!(out.matches("giving up").count(), 1, "{out}");
    let mut calls = 0;
    let r: Result<(), String> = retry(0, |_| {
        calls += 1;
        Err("x".into())
    });
    assert!(r.is_err());
    assert_eq!(calls, 1);
}

#[test]
fn job_errors_display_for_humans() {
    let _guard = serial();
    assert_eq!(JobError::EmptyPayload.to_string(), "payload is empty");
    assert_eq!(JobError::Overflow.to_string(), "sum overflows u64");
    assert_eq!(JobError::Unauthorized.to_string(), "token rejected");
    let boxed: Box<dyn std::error::Error> = Box::new(JobError::Overflow);
    assert_eq!(boxed.to_string(), "sum overflows u64");
}

#[test]
fn overflow_is_reported_not_wrapped() {
    let _guard = serial();
    assert_eq!(add(1, 2), Ok(3));
    assert_eq!(add(u64::MAX - 1, 1), Ok(u64::MAX));
    assert_eq!(add(u64::MAX, 1), Err(JobError::Overflow));
    assert_eq!(add(u64::MAX, u64::MAX), Err(JobError::Overflow));
    // A payload of maximal u32s still fits in a u64.
    let m = Metrics::new();
    captured("off", || {
        let r = process(&job(1, "let-me-in", vec![u32::MAX, u32::MAX]), &m);
        assert_eq!(r, Ok(2 * u64::from(u32::MAX)));
    });
}
```

`examples/overhead.rs`:

```rust
//! How much does a log statement cost when it is disabled, and when it is on?
//! Release build only: `cargo run --release --example overhead`.
use std::hint::black_box;
use std::time::Instant;
use tracing::{Level, debug};

const N: u64 = 2_000_000;

fn emit(n: u64) -> u64 {
    let mut acc = 0u64;
    for i in 0..n {
        acc = acc.wrapping_add(black_box(i));
        debug!(i, "tick");
    }
    acc
}

fn time(label: &str, f: impl Fn() -> u64) {
    let mut v: Vec<f64> = (0..7)
        .map(|_| {
            let t = Instant::now();
            black_box(f());
            t.elapsed().as_nanos() as f64 / N as f64
        })
        .collect();
    v.sort_by(f64::total_cmp);
    println!(
        "{label:<46} min {:>7.2} ns/event  median {:>7.2}",
        v[0], v[3]
    );
}

fn main() {
    // Baseline: the same loop with no log statement at all.
    time("no log statement", || {
        let mut acc = 0u64;
        for i in 0..N {
            acc = acc.wrapping_add(black_box(i));
        }
        acc
    });
    // 1. No subscriber installed anywhere.
    time("debug! with no subscriber", || emit(N));
    // 2. A subscriber that only wants INFO and above.
    let info = tracing_subscriber::fmt()
        .with_max_level(Level::INFO)
        .with_writer(std::io::sink)
        .finish();
    tracing::subscriber::with_default(info, || {
        time("debug! filtered out (subscriber at INFO)", || emit(N));
    });
    // 3. A subscriber that records DEBUG and formats into a sink.
    let debug_sub = tracing_subscriber::fmt()
        .with_max_level(Level::DEBUG)
        .with_writer(std::io::sink)
        .with_ansi(false)
        .finish();
    tracing::subscriber::with_default(debug_sub, || {
        time("debug! recorded and formatted (sink)", || emit(N));
    });
}
```

`examples/inject.rs`:

```rust
//! What happens when untrusted text with a newline goes into a log field?
use tracing::info;

fn main() {
    tracing_subscriber::fmt()
        .with_ansi(false)
        .without_time()
        .with_target(false)
        .init();
    let name = "bob\n INFO login ok user=admin";
    info!("display field:   user = {}", name);
    info!(user = %name, "Display (%) field");
    info!(user = ?name, "Debug (?) field");
    info!(user = name, "plain &str field");
}
```

`ci.sh`:

```bash
#!/usr/bin/env bash
# Run the same checks CI runs, in the same order. Stops at the first failure.
set -euo pipefail
export RUSTFLAGS="-D warnings"
export RUSTDOCFLAGS="-D warnings"
export CARGO_TERM_COLOR=never

step() { printf '\n== %s\n' "$*"; }

step "format";            cargo fmt --all -- --check
step "lints";             cargo clippy --all-targets --locked -- -D warnings
step "tests";             cargo test --locked
step "docs";              cargo doc --no-deps --locked
step "release build";     cargo build --release --locked
step "duplicate crates";  cargo tree --locked -d --target all | tee /dev/stderr | { ! grep -q 'v[0-9]'; }
echo; echo "all checks passed"
```

`deny.toml`:

```toml
# Policy for dependency checks: `cargo deny check`.

[graph]
all-features = true

[advisories]
# Fail on known vulnerabilities, unmaintained crates in our graph and yanked
# versions. Exceptions must be listed here with a reason and a review date.
yanked = "deny"
ignore = []

[licenses]
# Only these SPDX licenses may appear in the dependency graph.
allow = ["MIT", "Apache-2.0", "Unicode-3.0"]
confidence-threshold = 0.9

[licenses.private]
# This crate is not published, so it needs no license of its own.
ignore = true

[bans]
multiple-versions = "deny"
wildcards = "deny"
deny = []

[sources]
unknown-registry = "deny"
unknown-git = "deny"
allow-registry = ["https://github.com/rust-lang/crates.io-index"]
allow-git = []
```

`.github/workflows/ci.yml` (parsed as YAML ✓; never executed by GitHub, so its action names and versions are from memory):

```yaml
name: ci

on:
  push:
    branches: [main]
  pull_request:

# Least privilege: the default token can only read the repository.
permissions:
  contents: read

env:
  CARGO_TERM_COLOR: always
  RUSTFLAGS: -D warnings
  RUSTDOCFLAGS: -D warnings

jobs:
  check:
    name: fmt, clippy, test, doc
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt, clippy
      - uses: Swatinem/rust-cache@v2
      - run: cargo fmt --all -- --check
      - run: cargo clippy --all-targets --locked -- -D warnings
      - run: cargo test --locked
      - run: cargo doc --no-deps --locked

  msrv:
    name: minimum supported Rust version
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      # Keep this version equal to `rust-version` in Cargo.toml.
      - uses: dtolnay/rust-toolchain@1.85.0
      - run: cargo build --locked

  deny:
    name: dependency policy
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: EmbarkStudios/cargo-deny-action@v2
```

✓ Checks I ran: `./ci.sh` (fmt, clippy with `-D warnings`, 11 tests plus 1 doctest, docs, release build, duplicate-crate check: all pass), `cargo deny check` (all four sections ok), and the experiments quoted in the lesson.

**Mutation checks** (each mutant on a copy of the crate, full test suite):

| # | Mutation | Result |
|---|---|---|
| O1 | `Redacted`'s `Debug` text changed | **caught** (redaction test and doctest) |
| O2 | `Redacted`'s `Display` text changed | **caught** |
| O3 | `#[instrument(skip(job, metrics))]` loses `job` from `skip` | **caught** (the span then carries an extra field) |
| O4 | `started` counter not incremented | **caught** (two tests) |
| O5 | success increments `failed` instead of `succeeded` | **caught** (two tests) |
| O6 | failure logged at `warn` instead of `error` | **caught** |
| O7 | token comparison inverted | **caught** (many tests) |
| O8 | empty-payload check removed | **caught** |
| O9 | `add` wraps instead of reporting overflow | **caught** (only by the direct `add` test: a real overflow needs more than 4 billion items) |
| O10 | `retry` makes one attempt too many | **caught** |
| O11 | `attempts.max(1)` removed from `retry` | **survived: equivalent mutant.** With `attempts == 0` the first failure already satisfies "not less than attempts", so the function gives up after one attempt exactly as before; the line only documents the intent |
| O12 | retry warning demoted to `debug` | **caught** |
| O13 | `succeeded` incremented by 2 | **caught** |
| O14 | per-item `debug!` removed | **caught** (the level-filter test) |
| O15 | `items` field removed from the start event | **caught** |
| O16 | `snapshot` reports `succeeded` in the `failed` slot | **caught only by the concurrency test**: in the sequential test succeeded and failed are both 2, so swapping them is invisible; a lesson in choosing test data with distinct values |
| O17 | an extra `info!` that logs the token with `expose()` | **caught** (the redaction test and the exact-line-count test) |
| O18 | `sum` reported as `sum + 1` in the finish event | **caught** |
| O19 | "giving up" error event removed | **caught** |

Totals: 19 mutants, 18 caught, 1 equivalent (O11). O17 is the important one for this lesson: a developer adding a single careless log line is exactly the failure the redaction test exists for.

**Design notes.**

- `process` skips the whole `Job` in its span and names the safe fields explicitly (allow-list instead of deny-list).
- The `Redacted` wrapper has no `Clone`-free guarantee against a determined developer (`expose()` exists), but it makes the safe path the default and the unsafe path greppable.
- Counters use `Relaxed` because they publish nothing else (Lesson 28); `snapshot` is explicitly not a transaction.
- `tracing-subscriber` is a dev-dependency with `default-features = false` and only the features the tests use.
- Not handled: JSON output, a non-blocking writer, trace export, metrics histograms, log sampling or rate limiting, per-request ids propagated across threads (a span does not follow work onto another thread by itself; from memory), log rotation and retention, and a complete secret-scanning step in CI.

</details>

---

## 16. Key takeaways

- Choose the signal by the question: metrics notice, traces locate, logs explain.
- Structured fields beat formatted strings, library code emits and the binary configures, and each error is logged once, where it is handled.
- Logging has a price you can measure: ✓ about 0.1 ns when disabled, 285 to 317 ns when recorded and formatted. Cheap to leave in, expensive to turn on at high rates.
- Make safe logging the default by construction (✓ `Redacted`; `skip` in `#[instrument]`), record untrusted text as escaped fields (✓ `%` and message interpolation forged a log line), and test your logs, serializing tests that install subscribers (✓ an intermittent failure otherwise).
- CI runs the same commands you do, with `--locked`, from a clean state, and tests the MSRV you declare. `-D warnings` is a trade-off, not a free win.
- Dependencies are code you run, sometimes at build time. The lock file's checksums detect substituted sources (✓), `cargo deny` enforces a policy over advisories, licenses, bans and sources (✓), and none of it finds what is not yet known: schedule the checks, review updates, keep the tree small.
- Rust's guarantees stop at the edge of memory safety; supply-chain and operational security are engineering work.

**Connections.** [Lesson 41](41_benchmarking_profiling.md) supplied the overhead-measurement method; [Lesson 39](39_testing.md) and [Lesson 40](40_property_testing_fuzzing.md) supply the testing habits (mutation checks found the weak test data in O16); [Lesson 28](28_atomics_memory_ordering.md) justifies the counters; [Lesson 35](35_proc_macros.md) and [Lesson 37](37_maybeuninit_ffi.md) explain why `cargo build` runs foreign code; [Lesson 26](26_error_design.md) is where error types earn their `Display`. Next: Lesson 44 covers command-line tools and serialization.

---

## 17. References

- `tracing` crate documentation and the `tracing-subscriber` crate (`fmt`, `EnvFilter`, `MakeWriter`): <https://docs.rs/tracing>, <https://docs.rs/tracing-subscriber>
- `log` crate (the facade): <https://docs.rs/log>
- The Cargo Book: "Cargo.lock", `--locked`, `rust-version`, `cargo tree`, `cargo vendor`, "Resolver versions": <https://doc.rust-lang.org/cargo/>
- `cargo-deny` book (advisories, licenses, bans, sources): <https://embarkstudios.github.io/cargo-deny/>
- RustSec advisory database and `cargo-audit`: <https://rustsec.org/>
- Advisory RUSTSEC-2020-0071 (`time`): <https://rustsec.org/advisories/RUSTSEC-2020-0071>
- GitHub Actions documentation, "Security hardening for GitHub Actions": <https://docs.github.com/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions>
- `cargo-vet` and `cargo-crev` (audit records), `cargo-geiger` (counts `unsafe`): <https://mozilla.github.io/cargo-vet/>, <https://github.com/crev-dev/cargo-crev>, <https://github.com/geiger-rs/cargo-geiger>
- OWASP, "Logging Cheat Sheet" (what to log, what never to log, log injection): <https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html>

The URLs above are the standard documentation locations; I did not open them during this session (the sandbox cannot reach the documentation sites; only crates.io and the advisory database were reachable through the tools themselves), so if a link has moved, search for the page title. Unchecked or from memory: the description of `log` versus `tracing` and the `log` compatibility feature; `tracing-appender`, `tracing::enabled!`, the `max_level_*` and `release_max_level_*` features and JSON formatters (none used here); that disabled-macro arguments are not evaluated (the timing is consistent with it, I did not read the macro); all metrics advice about cardinality, histograms and gauges; every statement about GitHub Actions, including the names and versions of the actions in the workflow, `pull_request_target`, token permissions, caches, matrices, and pinning by commit hash; that the MSRV-aware resolver is the default for edition 2024 (I observed the message, not the rule); whether libraries should commit `Cargo.lock`; why the lock file lists 19 packages while the host-platform tree shows 17; that `cargo vendor` is the way to audit and build offline in general (I ran it once); the descriptions of `cargo vet`, `cargo crev`, `cargo geiger` and trusted publishing; the claim typosquatting guidance; the legal remark about personal data (no legal advice is intended); and that a span does not follow work onto another thread. The measurements marked ✓ are from one shared virtual machine and, for the overhead table, three runs of one binary. The advisory and license results reflect the advisory database and the dependency versions at the time I ran them. Exercise answers 2, 4, 6, 7 and 8 combine ✓-run facts with reasoning from memory; the code blocks in answers 3 and 5 and in the debugging challenge were compiled and run. Everything marked ✓ was run on this machine's `rustc 1.97.0` with `tracing 0.1.44`, `tracing-subscriber 0.3.23` and `cargo-deny 0.20.2`.
