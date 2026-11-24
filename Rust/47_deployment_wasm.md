---
layout: default
title: "Lesson 47 — Deployment and WebAssembly"
---
{% raw %}

# Lesson 47 — Deployment and WebAssembly

**Module 14 · Application development (last lesson of the module)** · Status: generated · Edition used in examples: **2024** · External dependencies: `signal-hook 0.3.18` (the service), `wasmi 0.51.5` (a WebAssembly interpreter used as a test host), `wasm-bindgen 0.2.130` and its CLI (one section)

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97` and `rustfmt 1.9.0-stable`, on x86-64 Linux (glibc 2.39), 2 cores, Node.js 22.22.0, Docker Engine 29.8.2. As before, ✓ marks outputs and measurements I actually ran; unmarked statements are documented behavior I did not run.

**Three things about how this machine differed from yours, stated up front.**

1. `rustup target add wasm32-unknown-unknown` **failed here** (the download host `static.rust-lang.org` was unreachable from the sandbox). So I built the WebAssembly standard library myself from the Rust 1.97.0 source with `RUSTC_BOOTSTRAP=1 cargo build -Zbuild-std=std,panic_abort`. That variable and flag are an unsupported escape hatch for unstable features; **you should not need them**: on a normal machine `rustup target add wasm32-unknown-unknown` is all it takes, and sizes could differ slightly from mine because the official prebuilt standard library is built differently. For WASI I also copied the C library objects from `wasi-sdk 27` into the toolchain directory.
2. Docker's daemon was not running; I started `dockerd` by hand (storage driver `vfs`, cgroup v1, no bridge network, containers on `--network host`). I could build and run images only from `FROM scratch` (a registry was not needed). I did not use Kubernetes, a cloud, a registry, or any other CPU architecture or operating system.
3. I could not open the documentation sites; rules stated without ✓ are from memory and are listed in the closing paragraph.

**Not tested at all:** `musl` targets, cross-compiling to other targets, Windows/macOS, Kubernetes, `wasmtime`, the WebAssembly component model, wasm threads and SIMD, `wasm-opt`/Binaryen (not installed), browsers (only Node), GitHub Actions or any CI service, TLS/HTTPS termination, metrics systems. Timings are single measurements on one shared virtual machine. **Nothing here makes a deployment secure**; hardening flags and sandboxes reduce exposure, and the lesson says what each one does and does not do.

Builds on [Lesson 46](46_databases_config_auth_jobs.md) (configuration, secrets, graceful pool shutdown, jobs), [Lesson 45](45_http_apis.md) (HTTP, timeouts, limits), [Lesson 44](44_cli_serialization.md) (exit codes, `--version`), [Lesson 43](43_observability_ci_supply_chain.md) (logging, CI, `cargo-audit`, `cargo-deny`, reproducible builds), [Lesson 42](42_performance_engineering.md) (release profiles, LTO, `panic = "abort"`), [Lesson 41](41_benchmarking_profiling.md) (measuring), [Lesson 38](38_no_std_embedded_miri.md) (`no_std`, Miri), [Lesson 37](37_maybeuninit_ffi.md) (the C ABI, `extern "C"`), [Lesson 36](36_unsafe_raw_pointers.md) (raw pointers), [Lesson 27](27_threads_channels.md) (threads) and [Lesson 28](28_atomics_memory_ordering.md) (atomics). Lesson 48 is next; it appears here only as plain text.

---

## 1. Why this lesson exists

Writing the code is half the job. The other half is getting it to run somewhere you do not control, keeping it running, and stopping it without losing work. This lesson covers the two shapes that "somewhere else" takes for a Rust program:

- **A native service** running under a process manager or in a container: how to build it small and reproducibly, how to configure it, how to tell the platform it is alive and ready, and how to shut it down gracefully when the platform sends `SIGTERM`.
- **A WebAssembly module** running inside a host (a browser, Node.js, or a Rust program embedding a runtime): a different compilation target with its own rules, a very narrow interface made only of numbers, and a sandbox that is only as good as the host's limits.

Both are about **boundaries**, as the last four lessons have been: the process boundary (signals, exit codes, ports, environment) and the module boundary (imports, exports, linear memory). The running examples are two small crates: `shiplab` (a std-only HTTP service with health checks, strict configuration and a graceful drain; 28 tests) and `wasmlab` (pure functions exported over the plain WebAssembly C ABI; 13 native tests plus 6 that run the compiled `.wasm`). All sources are in the practical challenge.

---

## 2. Part A — Shipping a native service

### 2.1 Release profiles: what each knob buys ✓

The same crate (`shipd`, seven packages in `Cargo.lock`), built into a fresh target directory each time (cold build, two cores, dependencies included), changing one setting at a time:

| Build | Binary size | Build time |
|---|---:|---:|
| debug (`cargo build`) | 5,861,464 B | 1.9 s |
| release: `opt-level = 3`, no LTO, 16 codegen units, `panic = "unwind"`, `strip = "none"` | 4,475,096 B | 2.4 s |
| + `strip = "symbols"` | 422,392 B | 2.6 s |
| + `panic = "abort"` | 407,136 B | 2.3 s |
| + `lto = "fat"`, `codegen-units = 1` | 370,648 B | 6.1 s |
| + `opt-level = "s"` | 348,976 B | 5.5 s |
| + `opt-level = "z"` (instead of `"s"`) | 349,168 B | 5.4 s |

(The second row is the combination of settings I wrote out, not necessarily Cargo's own default for `strip`.) What each setting does:

- **`strip`** removes symbol tables and debug information. It is by far the largest effect here (4.4 MB to 422 KB), and it costs nothing at run time; the price is unreadable stack traces. Keep an unstripped build (or separate debug files) for crash analysis.
- **`panic = "abort"`** removes the unwinding machinery: smaller, slightly faster, but a panic now ends the process (so `catch_unwind` cannot recover; Lesson 42), and destructors do not run during a panic.
- **`lto` and `codegen-units = 1`** let the compiler optimize across crates and see the whole program at once: smaller and often faster, but ✓ the build took 6.1 s instead of 2.3 s on this tiny program, and grows quickly with project size.
- **`opt-level`**: `3` is fastest code, `"s"` and `"z"` optimize for size. ✓ In this single measurement `z` was not smaller than `s` (349,168 vs 348,976); the usual advice that `z` is smaller is a tendency, not a law. Measure; and remember the smaller build can be slower (Lesson 42).

The profile `shiplab` ships with is in its `Cargo.toml` (`opt-level = 3`, `lto = "fat"`, `codegen-units = 1`, `panic = "abort"`, `strip = true`). Choosing `abort` here is a deliberate decision: the service is meant to be restarted by its supervisor, and each connection's thread is isolated enough that a crashed process is acceptable. If a bug in one request handler should not kill the others, keep `unwind` and catch panics at the thread boundary (Lesson 46 does this for jobs).

### 2.2 Build information and locked builds ✓

- Build binaries with `cargo build --release --locked`: it fails instead of silently updating `Cargo.lock`. Commit the lockfile for applications.
- `build.rs` in `shiplab` records the compiler version and target and `/version` serves them. ✓ Output from the container run: `{"version":"0.1.0","rustc":"rustc 1.97.0 (2d8144b78 2026-07-07)","target":"x86_64-unknown-linux-gnu"}`. A deployed binary should be able to say what it is. (I did not embed a Git commit: that needs `git` in the build environment and breaks reproducibility if done carelessly.)
- `rust-version = "1.85"` documents the oldest compiler the code is meant to build with. I only tested 1.97.0.

### 2.3 Linking: dynamic, static, and what a container needs ✓

The default Linux build links dynamically against the system C library. ✓ `ldd` on the stripped release binary:

```text
linux-vdso.so.1
libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1
libc.so.6     => /lib/x86_64-linux-gnu/libc.so.6
/lib64/ld-linux-x86-64.so.2
```

Such a binary runs only where compatible shared libraries exist, which is why container images based on a full distribution are common. Statically linking the C library removes that dependency:

```text
RUSTFLAGS="-C target-feature=+crt-static" \
  cargo build --release --locked --target x86_64-unknown-linux-gnu
```

✓ Result: `ldd` prints `statically linked`, `file` says `static-pie linked`, and the size went from 370,648 B to **1,336,736 B** (the C library is now inside). Notes: the `--target` flag is required for `+crt-static` to apply cleanly; static glibc has known limitations (name-service lookups, `dlopen`; from memory), and `musl` is the other common route to a fully static binary (not installable here, so not tested). A static binary is exactly what lets the image in §4 be `FROM scratch`.

### 2.4 Reproducible builds ✓

If two people build the same source, do they get the same bytes? That makes a binary verifiable. Experiment with `shipd` (stripped release build):

| Build | SHA-256 (first 16 hex) |
|---|---|
| copy of the source in directory A | `c03d602728928565` |
| the same source in a different, deeper directory B | `c03d602728928565` |
| same source, `CARGO_HOME` set to `/tmp/w47/ch1` | `c6822f78d9ee8936` |
| same source, `CARGO_HOME` set to `/tmp/w47/ch2` | `80c3d185b26a7e57` |
| as above with `--remap-path-prefix` for the project and `CARGO_HOME` (ch1) | `7b58d1512673baed` |
| same remap with `CARGO_HOME` = ch2 | `7b58d1512673baed` |

So: the project's own directory did **not** change the result (Cargo passes workspace paths relatively), but the location of the dependency sources **did**, because their absolute paths end up in the binary (✓ `strings` found two `/root/.cargo` paths). `--remap-path-prefix=<real>=<fixed>` removed the difference. Honest limits: one compiler, one operating system, one set of flags; I did not test different machines, different compiler versions (which do change output), or crates whose build scripts embed timestamps or run C compilers.

### 2.5 Cross-compilation and supply chain (from memory)

`cargo build --target <triple>` builds for another platform; you need the target's standard library (`rustup target add`), usually a linker for it (set in `.cargo/config.toml`), and a C toolchain if any dependency compiles C (Lesson 46's SQLite does). Tools such as `cross` (containers) and `cargo-zigbuild` (uses Zig's linker) automate this. I did not run any of them. Lesson 43 covers what belongs in CI: `cargo build --locked`, `cargo test`, `cargo clippy -D warnings`, `cargo audit` and `cargo deny` (both are installed on this machine, but I did not run them for this lesson's crates). `shiplab` has one dependency, `signal-hook`, so its `Cargo.lock` has **7 packages**.

---

## 3. The service: configuration, health, and graceful shutdown

### 3.1 Configuration: the twelve-factor way, strictly ✓

`shiplab::config::Config::from_vars` takes `(name, value)` pairs (the process passes `std::env::vars()`; tests pass vectors, so no global state). Defaults, then overrides:

| Variable | Default | Valid range |
|---|---|---|
| `SHIPLAB_ADDR` | `127.0.0.1:8080` | a socket address |
| `SHIPLAB_DRAIN_MS` | 10,000 | 0 to 600,000 |
| `SHIPLAB_MAX_CONNS` | 64 | 1 to 10,000 |
| `SHIPLAB_READ_TIMEOUT_MS` | 5,000 | 1 to 600,000 |

Rules, each with a test: names without the `SHIPLAB_` prefix are ignored; a misspelled `SHIPLAB_*` name is an **error** (a setting that silently does nothing is the worst kind); every range is checked at both ends; surrounding spaces are trimmed; the error names the variable and **never echoes the value** (✓ a test puts `hunter2secret` in a numeric variable and checks the message does not contain it, as in Lesson 46); the default listen address is loopback, so an operator must opt in to exposing the service (`0.0.0.0` inside a container, where the container's port mapping is the real exposure control).

### 3.2 Exit codes are an interface ✓

| Code | Meaning | A supervisor should |
|---:|---|---|
| 0 | clean stop, including a finished drain | not restart |
| 2 | invalid configuration | not restart in a loop (retrying cannot fix it) |
| 3 | drain deadline passed with work still running | investigate |
| 4 | could not bind the socket | retry later or alert |

✓ Tests start the real binary and check codes 0, 2, 3 and 4 and what goes to stderr. A process that crashes from a signal has no code at all: Docker reports `128 + signal` (137 for `SIGKILL`, 143 for `SIGTERM`; both ✓ observed below). Logs go to **stderr** in `key=value` lines and stdout carries only the one line `listening addr=...` that the tests read; a container platform collects stdout/stderr, so a service should not write log files itself.

### 3.3 Liveness, readiness, and the two probes ✓

- `/healthz` is **liveness**: "the process is up and its main loop answers". It always returns 200 (✓ even while draining).
- `/readyz` is **readiness**: "send me new traffic". 200 normally; **503 once shutdown has begun**, so a load balancer stops routing to this instance while it finishes what it has. (A readiness check that also tested the database, a queue or a downstream service would need care: a dependency outage can make every instance "not ready" at once. `shiplab` has no dependencies to check.)
- `/version` reports build information; `/slow?ms=N` simulates a long request and is capped at 5,000 ms so a client cannot park a thread forever.

### 3.4 Limits at the edge ✓

Same ideas as Lesson 45, in a std-only server: the request line is at most 8,192 bytes and the whole head at most 16,384 (✓ tests pin both limits exactly: accepted at the limit, refused one byte over, including an unterminated head); a connection cap (`MAX_CONNS`; the next client gets `503 busy`; ✓ slots are released when connections close); a read timeout so a silent client is dropped (✓ with a 150 ms setting the connection was closed after between 120 ms and 3 s); one request per connection (`Connection: close`), which keeps the accept loop and the drain logic simple at the cost of connection setup per request. Each connection gets a thread; a bound on the number of connections is what bounds the number of threads.

### 3.5 Graceful shutdown ✓

A container platform or `systemctl stop` sends `SIGTERM`, waits, and then sends `SIGKILL`. Handling the first correctly means: stop taking new work, finish what is in flight, exit with the right code, all before the kill. The algorithm in `server::serve`:

```rust
loop {
    if shutdown.load(SeqCst) && drain_started.is_none() {
        drain_started = Some(Instant::now());
        draining.store(true, SeqCst);                 // /readyz now says 503
    }
    if let Some(t0) = drain_started {
        if in_flight.load(SeqCst) == 0       { return exit::OK; }
        if t0.elapsed() >= cfg.drain         { return exit::DRAIN_TIMEOUT; }
    }
    match listener.accept() { /* nonblocking: spawn a thread, or sleep 2 ms */ }
}
```

- **The signal handler only sets a flag.** `signal_hook::flag::register(SIGTERM, flag)` stores into an `AtomicBool` and returns; the main loop notices. (A signal handler may only do a very small set of async-signal-safe things; from memory: allocation and locking are not among them. A flag is the standard way around this; std itself offers no signal API.) `SIGINT` (Ctrl-C) is registered too.
- **Counting in-flight work with a guard.** Each accepted connection increments `in_flight` and carries an `InFlight` value whose `Drop` decrements it, so the count is right however the handler ends (normal return, early return, timeout, a client reset).
- **The listener stays open while draining**, so `/readyz` can answer 503 and a probe sees the change (✓ test: while a 700 ms request runs, after `SIGTERM`, `/readyz` returns 503 and `/healthz` 200, the slow request still completes with `done`, and the process exits 0).
- **There is a deadline.** If work does not finish within `SHIPLAB_DRAIN_MS` the service gives up with code 3 (✓ a connection that never sends anything plus `DRAIN_MS=200` exited with 3 after between 180 ms and 3 s). **The platform's kill timeout must be longer than the drain deadline**, or the platform's `SIGKILL` wins and the exit code is 137 (§4).
- Not in `shiplab`: a delay between flipping readiness and closing the listener (so every load balancer notices first), keep-alive connections, and refusing new work with `Connection: close` hints. In real deployments, how long to keep serving while "not ready" depends on the load balancer's probe interval.

---

## 4. Containers ✓

A container is a Linux process with isolated namespaces and resource limits; the image is a file tree plus metadata. Because `shipd` is a static binary, its image can be nearly empty:

```dockerfile
FROM scratch
COPY shipd /shipd
ENV SHIPLAB_ADDR=0.0.0.0:8080
EXPOSE 8080
USER 65532:65532
ENTRYPOINT ["/shipd"]
```

✓ `docker build` produced an image of **1.34 MB**. It runs, answers `/healthz` and `/version` and logs to `docker logs`. Choices in those six lines:

- **`FROM scratch`**: nothing but the binary: no shell, no package manager, no CA certificates, no time-zone data (from memory), so also nothing for an attacker to use and nothing for you to `exec` into. Services that make TLS requests need CA certificates copied in; those that need a shell for debugging need a different base. I only tested the `scratch` case.
- **`USER 65532:65532`**: a non-root numeric user. ✓ `docker inspect` shows it.
- **Exec form** `ENTRYPOINT ["/shipd"]`: the binary is PID 1 and receives signals directly. The *shell form* (`ENTRYPOINT /shipd`) would start `/bin/sh -c /shipd`, and the shell does not forward `SIGTERM` (from memory; not tested here, and `scratch` has no shell anyway).
- A multi-stage build (compile in a `rust` image, copy the binary into the final image) is the usual way to avoid shipping the toolchain (from memory; I built the binary outside Docker, so I did not test a multi-stage build).

✓ Running it hardened worked: `docker run --read-only --cap-drop ALL --security-opt no-new-privileges --memory 32m --pids-limit 64 ...` and `/readyz` answered `ready`. These flags remove the ability to write to the filesystem, drop all Linux capabilities, forbid privilege gain, and cap memory and process count. They narrow what a compromised process could do; they do not make the application correct or secure.

### 4.1 PID 1 and the stop experiment ✓

On Linux, the first process in a container is PID 1, and the kernel gives PID 1 **no default signal handling**: a signal it has not installed a handler for is ignored. So a program that does not handle `SIGTERM` cannot be stopped politely when it runs as PID 1. Measured with `docker stop`:

| Process | `docker stop` took | Exit code | What happened |
|---|---:|---:|---|
| `shipd`, with a 1.5 s `/slow` request in flight | **1.14 s** | **0** | drained: the request finished (`done` received), log shows `exit_code=0` |
| a 4-line C program (`for(;;) pause();`) with no handler, `-t 3` | 3.09 s | **137** | `SIGTERM` ignored; Docker waited the whole timeout, then `SIGKILL` |
| the same program with `docker run --init` | 0.076 s | **143** | Docker's init process (PID 1) forwarded `SIGTERM`; the default action ended the program |

Lessons: install a handler (as `shipd` does) **or** run an init process; the default stop timeout is 10 s (from memory; I passed `-t` explicitly); exit code 137 in a restart loop usually means "killed after not stopping in time" (it can also mean the kernel killed the process for exceeding its memory limit; I did not test that case).

---

## 5. Process managers and orchestrators

### 5.1 systemd (syntax ✓, behavior from memory)

A unit file for the same binary, with the stop behavior spelled out:

```ini
[Unit]
Description=shipd example service
After=network-online.target
Wants=network-online.target

[Service]
Type=exec
ExecStart=/usr/local/bin/shipd
Environment=SHIPLAB_ADDR=127.0.0.1:8080
Environment=SHIPLAB_DRAIN_MS=10000
Restart=on-failure
RestartSec=2
# SIGTERM first (the default); wait longer than the drain deadline before SIGKILL.
KillSignal=SIGTERM
TimeoutStopSec=15
DynamicUser=yes
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
CapabilityBoundingSet=
MemoryMax=128M

[Install]
WantedBy=multi-user.target
```

✓ `systemd-analyze verify` accepted it (exit code 0; I verified the same file with `ExecStart` pointing at the binary in a temporary directory, because the tool checks that the executable exists), and when I misspelled one key (`KillSignall`) it reported ``Unknown key name 'KillSignall' ... ignoring``, which is exactly the kind of typo that silently loses a setting. systemd was not running here, so **I did not start the service**; what follows is from memory: `TimeoutStopSec` must exceed the drain deadline (here 15 s vs 10 s) or systemd kills the process; `Restart=on-failure` restarts on non-zero exit codes (and `RestartPreventExitStatus=2` would stop a restart loop caused by bad configuration); `DynamicUser`, `ProtectSystem`, `NoNewPrivileges`, an empty capability set and `MemoryMax` are the unit-file counterparts of the container flags above.

### 5.2 Kubernetes (from memory, not run)

The same ideas appear as: a **liveness probe** on `/healthz` (restart if it fails), a **readiness probe** on `/readyz` (remove from the load balancer if it fails), a **startup probe** for slow starts, `terminationGracePeriodSeconds` (the kill timeout; must exceed the drain), and rolling updates that start new pods before stopping old ones. A request that arrives *after* the pod is told to stop can still reach it, because endpoint removal and `SIGTERM` are not strictly synchronized, which is why the drain keeps answering instead of closing the listener at once. None of this was tested.

---

## 6. Part B — WebAssembly

### 6.1 What WebAssembly is

A **WebAssembly module** is a portable, validated, sandboxed binary program for a small virtual stack machine. Key facts:

- The only value types are numbers: `i32`, `i64`, `f32`, `f64` (plus some reference types). There are no strings, structs or slices at the boundary.
- A module has **imports** (functions, memory, tables it needs from its host) and **exports** (what the host may call or read). It can do nothing outside what its imports give it: no files, clock or network unless the host provides them. That is the capability model, and it is why the sandbox is useful.
- It owns a **linear memory**: one byte array measured in 64 KiB pages, growable. Pointers in the module are offsets into that array. The host can read and write it too.
- It is **validated** before running (types, stack discipline), and out-of-bounds memory access, division by zero or reaching `unreachable` **traps** (ends the call with an error) instead of corrupting the host.
- Rust targets: **`wasm32-unknown-unknown`** (no operating system at all; this lesson's main target), **`wasm32-wasip1`** (WASI "preview 1": a standard set of imports for files, arguments, environment, clock; ✓ used in §6.10) and, from memory, a `wasm32-wasip2` target for the component model (not used). On wasm32, `usize` and pointers are **32 bits** (§6.6).

Hosts in this lesson: Node.js (V8) and `wasmi`, a WebAssembly interpreter written in Rust, used inside Rust tests.

### 6.2 The smallest module ✓

```rust
// src/lib.rs, with `crate-type = ["cdylib"]` in Cargo.toml
#[unsafe(no_mangle)]
pub extern "C" fn add(a: i32, b: i32) -> i32 { a.wrapping_add(b) }
```

`cargo build --release --target wasm32-unknown-unknown` (plus `opt-level = "s"`, `lto`, `panic = "abort"`, `strip`) gave a `.wasm` file of **335 bytes** ✓. `#[unsafe(no_mangle)]` (edition 2024 spelling) keeps the symbol name `add`, and `extern "C"` fixes the calling convention; `cdylib` tells the compiler to produce a standalone library. Everything beyond that single function in a real module is the price of features you use.

### 6.3 What size costs ✓

`wasmlab` has an allocator, string handling, a sieve and panics. Same source, different settings (single builds, my locally built standard library; **sizes with the official prebuilt one may differ**):

| Build | `.wasm` size |
|---|---:|
| `add` only (above) | 335 B |
| `wasmlab`, `opt-level = "s"`, LTO, abort, strip (the profile in its `Cargo.toml`) | 20,305 B |
| `opt-level = 3` | 18,146 B |
| `opt-level = "z"` | 24,640 B |
| no LTO, 16 codegen units | 20,424 B |
| `strip = false` (debug names kept) | 31,113 B |
| `opt-level = 0` | 165,071 B |
| + feature `fmt` (one `format!("value={n} hex={n:#x}")`) | 22,024 B |
| + feature `hostlog` (one import, one `format!`) | 22,306 B |

Readings: **a handful of `format!`/allocation/panic paths account for the jump from 335 B to 20 KB**; one `format!` added about 1.7 KB (✓ 22,024 − 20,305); stripping names saved about 10 KB; and again `opt-level = 3` was the smallest in this crate while `"z"` was the largest, so measure instead of assuming. I did not run `wasm-opt` (Binaryen), which commonly shrinks modules further (from memory).

### 6.4 The ABI: numbers only, memory for everything else ✓

A host cannot pass a `&str`. The protocol in `wasmlab` (every Rust-to-JavaScript glue generator reinvents it):

1. The host calls **`alloc(len)`** → an address in the module's memory.
2. The host writes the bytes there (`new Uint8Array(memory.buffer).set(bytes, ptr)`).
3. The host calls **`slugify(ptr, len)`**. The result is a **new** buffer, returned as one `i64`: `(ptr << 32) | len`.
4. The host reads `len` bytes at `ptr`, **copies them out**, and calls **`dealloc(ptr, len)`** for the result and for its input. Whoever allocates must be told who frees: here the module allocates everything, the host frees everything it was given, and `dealloc` must receive the **same length** that `alloc` used.

Two gotchas I reproduced ✓ (Node 22):

- **`i64` arrives as a signed `BigInt`.** FNV-1a of `"foobar"` is the unsigned `0x85944171f73967e8`, but the first version of my script printed `0x-7a6bbe8e08c69818` (the same 64 bits as a signed value). The fix is `BigInt.asUintN(64, value)`, applied to the packed `(ptr, len)` too, otherwise a pointer above 2³¹ gives a negative number. (JavaScript `Number` cannot hold 64-bit integers exactly, which is why the API uses `BigInt`.)
- **`memory.buffer` is replaced when memory grows, and old views are detached.** After `primes(50_000_000)` grew the memory from 34 pages to 797 (50 MiB), a `Uint8Array` created earlier had `length` **0**, while a fresh view had 52,232,192 bytes. Never cache a view across a call that might allocate; fetch `new Uint8Array(x.memory.buffer)` each time (the host script does).

Native tests can drive the same `extern "C"` functions with real pointers (✓ `tests/abi.rs`), but only the length of a result is meaningful there: on a 64-bit host a pointer does not fit the 32-bit half of the packed value. The real boundary is tested in §6.8.

### 6.5 Panics and traps ✓

With `panic = "abort"` (what this target uses; unwinding on wasm needs unstable features, from memory), a panic becomes the `unreachable` instruction, and the host sees an error. ✓ In Node: `x.divide(1, 0)` throws `RuntimeError: unreachable`; the instance still works afterwards (`add(1,1)` returned 2), though in general **the module's state is undefined after a trap in the middle of a mutation**: discard the instance when in doubt. ✓ In `wasmi`: the call returns an `Err` whose text contains `unreachable`.

A related native-side surprise ✓: a panic *inside an `extern "C"` function* does **not** unwind into the caller; (from memory: since Rust 1.81) it aborts the process with ``panic in a function that cannot unwind``. My first native test `catch_unwind(|| divide(1, 0))` crashed the whole test binary (SIGABRT). So ABI functions must either never panic or be tested where traps are catchable (the wasm host); the mutation run (§11) hits this twice, and the harness had to count a test-process abort as detection.

### 6.6 `usize` is 32 bits: a bug that native tests cannot see ✓

The sieve in my first version of `prime_count` did `let mut j = i * i;` with `i: usize`. On a 64-bit machine `i * i` for `i` up to 10⁶ fits easily. On `wasm32`, `usize` is 32 bits, so `i * i` overflows once `i > 65,535`. Release builds do not check overflow, so the value silently wraps and the loop marks the wrong numbers as composite:

```text
native:  prime_count(1_000_000) = 78498   (correct)    cargo test: passed
wasm:    primes(1000000)        = 78493   (wrong)      Node, release build
wasm (debug profile): primes(1000000) throws RuntimeError: unreachable   (overflow check)
```

✓ All three observed. The fix is checked arithmetic (`i.checked_mul(i)`; on overflow there are no multiples ≤ n to mark) and a size cap for the allocation (`MAX_SIEVE`, ✓ tested). The lesson is general: **code that compiles for two targets must be tested on both**. §11 shows the mutation that reintroduces the bug (`wrapping_mul`): all 13 native tests still pass; only the test that runs the compiled module fails. This is the debugging challenge in §9.

### 6.7 Hosting: Node and `wasmi` ✓

**Node.js** (`js/host.mjs` in the challenge): `new WebAssembly.Module(bytes)`, `WebAssembly.Module.exports/imports` to inspect it (✓ for `wasmlab`: exports `add alloc dealloc divide fnv1a memory primes slugify`, imports `[]`), `new WebAssembly.Instance(module, imports)`.

**`wasmi`** lets a Rust program load and run a module, which is how you would embed plugins or run untrusted computations (✓ 6 tests in `tests/wasm_host.rs`, ignored by default because they need the compiled `.wasm`; the file's header shows the two commands):

- **Fuel** bounds the work: with 1,000,000 units, `primes(1000)` ran, and `primes(100_000_000)` failed with an out-of-fuel error instead of running for seconds.
- **A memory limit** (`StoreLimitsBuilder::memory_size(8 MiB)`) turned `primes(50_000_000)` into an error, while `primes(100_000)` (9,592 primes) still worked.
- **Missing imports fail at link time**: the `hostlog` build imports `env.host_log`; instantiating it with an empty linker failed with an error naming `host_log`. A module cannot call a capability the host did not provide, which is the sandbox boundary in practice. (To provide it, the host registers a function under `("env", "host_log")` in the linker.)
- Strings crossed the boundary through memory exactly as in §6.4 (`Memory::write` and `Memory::read`), and the results equal the native `fnv1a_64` and `slugify_str`.

A sandbox limits *what* a module can reach, not *how much* it can consume: without fuel or a time and memory limit, a hostile module can still spin or allocate. Interpreters such as `wasmi` are slower than JIT compilers (from memory; I did not measure `wasmi`), which is a reason runtimes with compilers exist (`wasmtime`, not tested here).

### 6.8 Test on the target ✓

The structure that fits:

- Pure logic (`slugify_str`, `fnv1a_64`, `prime_count`, `pack`/`unpack`) is tested natively, fast, with ordinary `cargo test` (13 tests).
- The exported ABI is tested **through the compiled module** (6 `#[ignore]`d tests in `wasmi`, plus the Node script), because that is where 32-bit pointers, traps and the memory protocol live.
- In CI you would build the wasm artifact first and run the ignored tests with `-- --ignored`. (From memory; the other common tool, `wasm-bindgen-test`, runs tests in a browser or Node; it is installed with the CLI but I did not use it.)

### 6.9 `wasm-bindgen`: generated glue ✓

Hand-written ABI code is easy to get wrong (the two gotchas above). `wasm-bindgen` generates the glue from annotations:

```rust
#[wasm_bindgen(js_name = slugText)]
pub fn slug_text(text: &str) -> Option<String> { /* ... */ }
```

✓ Built with the same toolchain, then `wasm-bindgen --target nodejs --out-dir out_node ...` (and `--target web`), and called from Node: `m.slugText("Hello, Wörld!")` returned `"hello-w-rld"`, `m.slugText("日本語")` returned `undefined`, `m.fnvText("foobar")` was a `bigint` equal to `0x85944171f73967e8` (already unsigned), `m.primeCount(1000000)` returned 78498. What I observed besides:

- **Sizes**: the compiler's output was 39,229 B; after `wasm-bindgen` the module was **21,452 B** plus **3,870 B** of JavaScript glue (`nodejs` target) or **7,417 B** (`web` target), and `.d.ts` type declarations (a TypeScript description of the API).
- **Versions must match**: `wasm-bindgen = "=0.2.130"` in `Cargo.toml` and CLI 0.2.130 (the library and the tool that post-processes its output must be the same version; the `=` pins it).
- **Exports of dependencies leak**: `wblab` depends on `wasmlab`, whose `#[unsafe(no_mangle)]` functions are *also* exported by the final module (✓ the export list included `add alloc dealloc divide fnv1a primes slugify` next to my three functions), and a name clash made `wasm-bindgen` fail with ``the name `slugify` is exported by multiple crates``. In a real design you would put the C-ABI exports behind a Cargo feature so a library crate used as a dependency does not export symbols nobody asked for. (`wasmlab` as written does not do that; that is a limitation of the example.)
- With `strip = true`, `wasm-bindgen` warned that the `target_features` section was missing; I used `strip = "debuginfo"` for that crate.

### 6.10 WASI: files and arguments through capabilities ✓

`wasm32-wasip1` gives `std` a real (capability-based) system interface. A small program that reads arguments, an environment variable, standard input, a file inside a directory and a file outside it, and the clock, built to a **115,730 B** module and run in Node's WASI (`new WASI({ preopens: { "/sandbox": dir }, args, env })`) printed:

```text
imports: wasi_snapshot_preview1
args=["wasihello", "--flag"] NAME=ada stdin_bytes=11
read: hello from the host dir
read /etc/hostname: NotFound
clock ok: true
exit code: 0
```

The guest could read the directory the host *pre-opened* and nothing else: `/etc/hostname` (which exists on this machine) was `NotFound`. Node's WASI support was flagged as experimental (it printed an `ExperimentalWarning` which I silenced), and, from memory, is documented as not a hardened security boundary. For untrusted code, use a runtime built for it and add resource limits; this was a demonstration of the capability model, not a security test.

### 6.11 Is it fast? ✓ (two loops only)

| Work | Native (release) | wasm in Node/V8 |
|---|---|---|
| `primes(10_000_000)` (sieve, 10 MB) | 59.3 to 71.0 ms | 57.6 to 58.8 ms |
| FNV-1a over 20 × 1 MiB | 29.7 to 33.5 ms | 30.2 to 34.9 ms |

(Five runs each, sorted, one machine.) For these two loops, native code and V8's compiled wasm took about the same time. That is **not** a general result: both loops are limited by memory bandwidth or by a dependent multiply chain, and I measured neither start-up and compilation time, the cost of crossing the boundary many times, nor code that uses SIMD or threads. Copying data in and out of linear memory is real work that a pure-compute benchmark hides.

### 6.12 Security notes

- Sandboxing: a module cannot touch host memory or call out except through its imports, and out-of-bounds accesses trap. That is a strong property, and it depends on the **host** implementing the boundary correctly and applying limits (§6.7: fuel, memory).
- Inside the module, memory is one flat array: a bug in `unsafe` Rust (or in C compiled to wasm) can still corrupt the module's own data and, from memory, can be exploitable within it, because the usual hardware protections are absent. Memory safety of safe Rust still applies; the `unsafe` ABI functions are where review effort belongs (each has a `# Safety` section and `// SAFETY:` comments).
- Treat all input from the host as untrusted *inside* the module too (✓ `slugify` validates UTF-8 and returns `NONE`; the sieve refuses sizes above the cap).
- Nothing here guarantees security. Supply chain concerns of Lesson 43 apply to `wasm-bindgen` and every other build-time tool.

---

## 7. Common mistakes

1. Shipping a `debug` build (✓ 5.9 MB, much slower) or an unstripped release (✓ 4.5 MB).
2. A service that ignores `SIGTERM` as PID 1 (✓ 3 s of waiting and exit 137), or a shell-form `ENTRYPOINT` that hides the process from signals.
3. A kill timeout shorter than the drain deadline, so the graceful path never finishes.
4. Using the same endpoint for liveness and readiness; or a readiness check that fails for everyone when a shared dependency is down.
5. Config that silently ignores typos, or that prints secrets in errors (✓ strict names, values never echoed).
6. Running as root, with a writable root filesystem and all capabilities, when none are needed.
7. Assuming a "reproducible" build because you built twice on one machine (✓ `CARGO_HOME` location changed the hash).
8. Testing wasm-bound code only natively (✓ the `usize` bug) or only in debug builds (✓ the debug build trapped where the release build silently returned a wrong value).
9. Treating `i64` results as unsigned in JavaScript (✓ negative `BigInt`), or keeping a `Uint8Array` across calls that may grow memory (✓ length 0).
10. Panicking in `extern "C"` functions (✓ the process aborts natively; in wasm it traps), and leaving them undocumented.
11. Exporting more than intended from a `cdylib` (✓ dependency exports leaked).
12. Running untrusted wasm without fuel and memory limits.

---

## 8. Exercises

Answers are collapsed. Try each before opening it.

1. **(Warm-up)** Which of `strip`, `panic = "abort"`, `lto` and `opt-level = 3` shrink the binary, which cost build time, and which change behavior you might rely on?
2. **(Warm-up)** A service answers 200 on `/readyz` forever but its database is down. Which problem is that, and what is the risk of "fixing" it by making `/readyz` check the database?
3. **(Easy)** Write `parse_port(&str) -> Result<u16, &'static str>`: decimal digits only (after trimming), `1..=65535`, with the errors `"not a number"` and `"out of range"`, and no overflow panic for huge inputs.
4. **(Medium)** Write `wait_for_idle(&AtomicUsize, Duration) -> bool` that returns `true` when the counter reaches zero within the deadline and `false` otherwise, polling without spinning. Test it with a thread.
5. **(Medium)** Export `reverse(ptr, len) -> u64` from a `wasm32-unknown-unknown` crate (it reverses the characters of UTF-8 text and returns a new `(ptr << 32) | len` buffer, or 0 for invalid input) and write the Node.js side, avoiding both pitfalls from §6.4.
6. **(Medium-hard)** `x.fnv1a(ptr, len)` in your JavaScript prints negative numbers for some inputs. Explain exactly why, and give two fixes (one in JavaScript, one in Rust).
7. **(Hard)** Design a zero-downtime rolling update: in what order do the old and new instances become ready, stop being ready, drain and exit? How long must the platform's kill timeout be, and what happens to a request that outlives the drain deadline?
8. **(Hard)** You must run a customer-supplied computation as a WebAssembly module inside your Rust service. List the controls you would apply and what each prevents.

<details markdown="1">
<summary>Answers</summary>

1. `strip` shrinks a lot (✓ 4.4 MB to 422 KB), is free at run time, and changes nothing but your ability to read crash traces. `panic = "abort"` shrinks a little (✓ 422 to 407 KB) and **changes behavior**: panics end the process and `catch_unwind` stops working. `lto = "fat"` with `codegen-units = 1` shrinks (✓ 407 to 371 KB) and often speeds up, at a cost in build time (✓ 2.3 s to 6.1 s). `opt-level = 3` maximizes speed, not size (the size-oriented levels are `"s"` and `"z"`; ✓ in my measurement `"z"` was not smaller than `"s"`).
2. Readiness (and, loosely, dependency health) is the question: the instance claims to accept traffic it cannot serve, so users see errors while the load balancer keeps sending them. Making `/readyz` check the database fixes that instance, but if all instances share the database, an outage makes **all** of them unready at once, and the load balancer has nowhere to send traffic (or restarts/cascades make things worse). Common compromises: check only what is local to the instance, or degrade gracefully and report dependency state separately (metrics, a different endpoint) rather than removing every instance. (General practice from memory.)
3. A possible answer (✓ compiled and run):

   ```rust
   fn parse_port(text: &str) -> Result<u16, &'static str> {
       let t = text.trim();
       if t.is_empty() || !t.bytes().all(|b| b.is_ascii_digit()) {
           return Err("not a number");
       }
       match t.parse::<u32>() {
           Ok(n) if (1..=65_535).contains(&n) => Ok(n as u16),
           _ => Err("out of range"), // 0, 65536.., and digits too long for u32
       }
   }

   fn main() {
       assert_eq!(parse_port("8080"), Ok(8080));
       assert_eq!(parse_port(" 1 "), Ok(1));
       assert_eq!(parse_port("65535"), Ok(65535));
       assert_eq!(parse_port("0"), Err("out of range"));
       assert_eq!(parse_port("65536"), Err("out of range"));
       assert_eq!(parse_port("99999999999999999999"), Err("out of range"));
       assert_eq!(parse_port("+80"), Err("not a number")); // `parse` alone would accept "+80"
       assert_eq!(parse_port("-1"), Err("not a number"));
       assert_eq!(parse_port(""), Err("not a number"));
       assert_eq!(parse_port("80a"), Err("not a number"));
   }
   ```

   Checking for digits first matters: `"+80".parse::<u32>()` succeeds. Parsing as `u32` and then range-checking keeps overflowing digit strings in the "out of range" bucket without a panic.
4. A possible answer (✓ compiled and run):

   ```rust
   use std::sync::atomic::{AtomicUsize, Ordering};
   use std::sync::Arc;
   use std::time::{Duration, Instant};

   fn wait_for_idle(in_flight: &AtomicUsize, deadline: Duration) -> bool {
       let start = Instant::now();
       loop {
           if in_flight.load(Ordering::SeqCst) == 0 {
               return true;
           }
           if start.elapsed() >= deadline {
               return false;
           }
           std::thread::sleep(Duration::from_millis(2));
       }
   }

   fn main() {
       let n = Arc::new(AtomicUsize::new(1));
       let worker = {
           let n = n.clone();
           std::thread::spawn(move || {
               std::thread::sleep(Duration::from_millis(50));
               n.fetch_sub(1, Ordering::SeqCst);
           })
       };
       assert!(wait_for_idle(&n, Duration::from_secs(2)));
       worker.join().unwrap();
       let stuck = AtomicUsize::new(1);
       let t = Instant::now();
       assert!(!wait_for_idle(&stuck, Duration::from_millis(40)));
       assert!(t.elapsed() >= Duration::from_millis(40));
       assert!(wait_for_idle(&AtomicUsize::new(0), Duration::ZERO)); // idle already: true
   }
   ```

   The idle check comes first so an already-idle system returns `true` even with a zero deadline. Sleeping 2 ms between checks avoids a busy loop; a condition variable or channel would wake exactly when the count hits zero, at the price of more machinery (Lesson 29).
5. A possible answer (✓ built with `wasm32-unknown-unknown` and run in Node; it printed `["olléh",null,"😀ba"]`). Rust:

   ```rust
   // crate-type = ["cdylib"]
   #[unsafe(no_mangle)]
   pub extern "C" fn alloc(len: usize) -> *mut u8 {
       Box::into_raw(vec![0u8; len].into_boxed_slice()).cast()
   }

   /// # Safety
   /// `ptr`/`len` must be exactly a block handed out by this module, not yet freed.
   #[unsafe(no_mangle)]
   pub unsafe extern "C" fn dealloc(ptr: *mut u8, len: usize) {
       // SAFETY: by the contract above.
       drop(unsafe { Box::from_raw(std::ptr::slice_from_raw_parts_mut(ptr, len)) });
   }

   /// # Safety
   /// `ptr..ptr+len` must be readable memory of this module.
   #[unsafe(no_mangle)]
   pub unsafe extern "C" fn reverse(ptr: *const u8, len: usize) -> u64 {
       // SAFETY: guaranteed by the caller.
       let bytes = unsafe { std::slice::from_raw_parts(ptr, len) };
       let Ok(text) = std::str::from_utf8(bytes) else { return 0 };
       let out: String = text.chars().rev().collect();
       if out.is_empty() {
           return 0;
       }
       let boxed = out.into_bytes().into_boxed_slice();
       let n = boxed.len() as u64;
       let p = Box::into_raw(boxed).cast::<u8>() as usize as u64;
       (p << 32) | n
   }
   ```

   JavaScript:

   ```js
   import { readFileSync } from "node:fs";
   const { exports: x } = new WebAssembly.Instance(
     new WebAssembly.Module(readFileSync(process.argv[2])), {});
   const enc = new TextEncoder(), dec = new TextDecoder();
   const view = () => new Uint8Array(x.memory.buffer); // never cache this across calls

   function reverse(text) {
     const input = enc.encode(text);
     const ptr = x.alloc(input.length);
     view().set(input, ptr);
     const packed = BigInt.asUintN(64, x.reverse(ptr, input.length)); // i64 arrives signed
     x.dealloc(ptr, input.length);
     if (packed === 0n) return null;
     const outPtr = Number(packed >> 32n), outLen = Number(packed & 0xffffffffn);
     const out = dec.decode(view().slice(outPtr, outPtr + outLen)); // copy before freeing
     x.dealloc(outPtr, outLen);
     return out;
   }
   ```

   The two pitfalls: unsigned reinterpretation of the `i64`, and a fresh `view()` after each call that can allocate.
6. WebAssembly's `i64` maps to a JavaScript `BigInt`, and the value comes back **signed**: FNV-1a results have the top bit set about half the time, so they appear negative (✓ `0x85944171f73967e8` printed as `-0x7a6bbe8e08c69818`). JavaScript fix: `BigInt.asUintN(64, value)`. Rust-side fixes: return the value in two `u32` halves (or write it into a buffer the host reads), or use `wasm-bindgen`, whose generated glue returns an unsigned `bigint` for `u64` (✓ observed). The same bug hits the packed `(ptr << 32) | len` result whenever the pointer is 2³¹ or larger.
7. One safe order: (1) start new instances; (2) wait until each reports **ready** (the readiness probe, not just "process started"); (3) only then tell the old instances to stop: they flip `/readyz` to 503 and the load balancer stops sending new requests (this takes up to a probe interval, so keep serving during that window); (4) old instances finish in-flight requests (the drain) and exit 0. The platform's kill timeout must exceed `probe interval + longest legitimate request` (here: greater than `SHIPLAB_DRAIN_MS`, with margin), or the platform's `SIGKILL` cuts requests off (exit 137). A request that outlives the drain deadline is abandoned: the service gives up (✓ code 3) and the client sees a reset connection; clients should retry idempotent requests (Lesson 45). General reasoning from memory; only the single-instance behavior is ✓.
8. Controls (based on what ✓ I ran with `wasmi`, plus general practice): give the module **no imports** you do not need, so it has no ambient files, network or clock (✓ link failure for missing imports); set a **memory limit** (✓ an 8 MiB cap turned a large request into an error); set a **fuel** or time limit per call (✓ out-of-fuel instead of a long run); cap **input and output sizes** at the boundary and validate everything crossing it (✓ UTF-8 validation, size cap on the sieve); run each customer's module in its **own instance** and discard it after a trap (state may be corrupt); run the host in a restricted process (container flags from §4) because the runtime itself can have bugs; and log traps. None of these, alone or together, is a guarantee; a determined attacker targets the host runtime and what you import.

</details>

---

## 9. Debugging challenge

A teammate wrote this sieve. `cargo test` passes natively, and the function is exported to a browser as `primes`. Users report "the count is slightly off for big numbers". On the release `.wasm`, `primes(1000000)` returns `78493` instead of `78498`, while a **debug** build of the same module throws `RuntimeError: unreachable`. Explain both symptoms and fix the function.

```rust
// crate-type = ["cdylib", "rlib"], built for wasm32-unknown-unknown
pub fn prime_count(n: u32) -> u32 {
    if n < 2 {
        return 0;
    }
    let n = n as usize;
    let mut composite = vec![false; n + 1];
    let mut count = 0;
    for i in 2..=n {
        if !composite[i] {
            count += 1;
            let mut j = i * i;
            while j <= n {
                composite[j] = true;
                j += i;
            }
        }
    }
    count
}

#[unsafe(no_mangle)]
pub extern "C" fn primes(n: u32) -> u32 {
    prime_count(n)
}

#[cfg(test)]
mod tests {
    use super::prime_count;

    #[test]
    fn known_counts() {
        assert_eq!(prime_count(100), 25);
        assert_eq!(prime_count(1_000_000), 78_498);
    }
}
```

<details markdown="1">
<summary>Solution</summary>

**Cause.** On `wasm32`, `usize` is 32 bits. For `i > 65,535`, `i * i` is larger than `u32::MAX`. Overflow behaviour depends on the profile: the **release** build compiles without overflow checks, so the product silently wraps to a small number; `j` then starts far below `i * i`, and the inner loop marks numbers as composite that are not multiples of `i` greater than `i` (for example `i` itself or smaller primes' products), removing real primes from the count: 78,493 (✓ observed). The **debug** build has overflow checks, which turn the overflow into a panic, which on wasm is a trap: `RuntimeError: unreachable` (✓ observed). On the 64-bit machine running `cargo test`, `usize` is 64 bits and `1_000_000 * 1_000_000` fits, so the test passes: the test never exercised the failing arithmetic.

**Fix.** Use checked arithmetic, and say what overflow means: if `i * i` does not fit, there are no multiples of `i` that are `<= n` and not yet marked, so skip it; also guard the increment, and bound the allocation (the module's memory is limited, and `n` comes from the host):

```rust
pub const MAX_SIEVE: u32 = 200_000_000;

pub fn prime_count(n: u32) -> u32 {
    if n > MAX_SIEVE {
        return u32::MAX; // refuse instead of trying to allocate gigabytes
    }
    if n < 2 {
        return 0;
    }
    let n = n as usize;
    let mut composite = vec![false; n + 1];
    let mut count = 0;
    for i in 2..=n {
        if !composite[i] {
            count += 1;
            let Some(mut j) = i.checked_mul(i) else {
                continue; // i * i does not fit: no multiples <= n to mark
            };
            while j <= n {
                composite[j] = true;
                match j.checked_add(i) {
                    Some(next) => j = next,
                    None => break,
                }
            }
        }
    }
    count
}
```

(✓ This is the version in `wasmlab`: `primes(1000000)` returns 78498 in Node and in `wasmi`; native tests pass.) **Test on the target.** The regression test that matters runs the compiled module: `tests/wasm_host.rs::numeric_exports_work_in_the_interpreter` asserts 78,498. In §11's mutation run, replacing the fix with `i.wrapping_mul(i)` passes all 13 native tests and fails exactly that one. Alternatively keep the arithmetic in a type that is the same width everywhere (`u64`) or use `u32` explicitly and `checked_*`, so the code means the same thing on 32 and 64 bits. Also run the **release** configuration of the module in tests, because the debug configuration crashed loudly here while the release one lied quietly.

</details>

---

## 10. Interview questions

1. *Which release-profile settings would you use for a service, and what does `panic = "abort"` change?*
2. *What does `docker stop` do, and why can a Rust service as PID 1 take 10 s to stop?*
3. *Liveness vs readiness: what does each probe mean, and what goes wrong if they are mixed up?*
4. *How do you shut a service down gracefully?*
5. *What makes a build reproducible, and what broke it in your experiment?*
6. *What are the three kinds of things in a WebAssembly module's interface, and what can a module do without imports?*
7. *How do you pass a string between JavaScript and a Rust-compiled module?*
8. *What is different about `usize` on `wasm32`, and how could that hide a bug from your native tests?*
9. *What happens when Rust code in a wasm module panics? And in an `extern "C"` function natively?*
10. *What does the WebAssembly sandbox protect, and what must the host still do?*

<details markdown="1">
<summary>Answers</summary>

1. Typically `opt-level = 3`, `lto = "fat"`, `codegen-units = 1`, `strip = true`, and `panic = "abort"` when a supervisor restarts the process (✓ 5.9 MB debug vs 349 to 371 KB). `abort` removes unwinding: smaller and slightly faster, but a panic ends the process, destructors do not run on the way out, and `catch_unwind` can no longer recover.
2. It sends `SIGTERM` to PID 1 in the container, waits for the stop timeout (10 s by default, from memory; ✓ I passed `-t`), then `SIGKILL`. PID 1 has no default signal action, so a program that installed no handler ignores `SIGTERM` (✓ 3.09 s, exit 137). Fix: install a handler (✓ 1.14 s, exit 0 while finishing a request), or use `--init` (✓ 0.076 s, exit 143).
3. Liveness: "should the platform restart me?" Readiness: "should the platform route traffic to me?" If liveness reflected a dependency outage, a database blip would restart every instance; if readiness stays 200 while draining, requests keep arriving at a process that is shutting down (✓ `shipd` flips `/readyz` to 503 on `SIGTERM` and keeps `/healthz` at 200).
4. Install a signal handler that sets an atomic flag; on the flag, stop being ready, keep serving what is in flight (count it with an RAII guard), exit 0 when the count is zero, with a deadline that exits with a distinct non-zero code; make the platform's kill timeout longer than the deadline (✓ tests for each path).
5. The same inputs produce the same bytes: same compiler, same flags, same sources, no embedded timestamps or absolute paths. ✓ Moving the project directory changed nothing, but a different `CARGO_HOME` changed the hash because the dependency sources' absolute paths are embedded; `--remap-path-prefix` made the two builds identical.
6. Imports (what it needs from the host), exports (functions, memory, globals the host may use), and its own linear memory. Without imports it can compute and use its memory, nothing else: no files, no clock, no network (✓ `wasmlab` has `imports: []`).
7. Via linear memory: the host asks the module to allocate, writes UTF-8 bytes at the returned address, passes `(ptr, len)`, then reads the result `(ptr, len)`, copies it out and frees both blocks. Generators such as `wasm-bindgen` produce this glue (✓ and hide the `BigInt` signedness and view-detachment pitfalls).
8. It is 32 bits, so arithmetic that fits in 64 bits natively can overflow (✓ `i * i` in a sieve). Release builds wrap silently and debug builds trap, so the bug can look like either "wrong answer" or "crash"; native tests pass in both. Test the compiled module, in the release configuration.
9. In wasm with `panic = "abort"`, a panic becomes an `unreachable` trap and the host gets an error (✓ `RuntimeError: unreachable`; ✓ `wasmi` returns an `Err`). Natively, a panic that would unwind out of an `extern "C"` function aborts the process (✓ SIGABRT, "panic in a function that cannot unwind").
10. It keeps the guest from touching anything the host did not hand it (memory outside its linear memory, files, network) and turns illegal operations into traps. The host must still limit CPU (fuel/timeouts) and memory (✓ both enforced in `wasmi`), validate what crosses the boundary, keep imports minimal, isolate instances, and keep its runtime up to date. The sandbox is only as strong as the host that implements it.

</details>

---

## 11. Practical challenge: `shiplab` and `wasmlab`

Build two crates.

**`shiplab`** (a binary `shipd` plus a library):

1. Configuration from `SHIPLAB_*` variables with defaults and ranges; misspelled names are errors; values are never echoed.
2. A std-only HTTP/1.1 server: bounded request line and head, `/healthz`, `/readyz`, `/version`, `/slow`, 404/405/400/431, a connection cap and a read timeout.
3. Graceful shutdown on `SIGTERM` and `SIGINT`: readiness flips to 503, in-flight requests finish, exit 0; a deadline exits 3; exit 2 for bad configuration and 4 for bind errors.
4. Tests that start the real binary and send real signals.
5. A `FROM scratch` Dockerfile for a static build, and a systemd unit.

**`wasmlab`** (a `cdylib` + `rlib`):

1. Pure functions: an FNV-1a hash, a `slugify`, a prime counter with a size cap, and `pack`/`unpack` for a `(ptr, len)` pair in an `i64`.
2. An exported C ABI with `alloc`/`dealloc`, a hash and a string function that returns a new buffer, a function that traps, and (behind features) one that uses `format!` and one that calls a host import.
3. Native tests for the logic and the ABI, a leak test with a counting allocator, tests that run the compiled `.wasm` in `wasmi` (fuel, memory limit, traps, link failure), and a Node.js host script.

<details markdown="1">
<summary>Solution</summary>

### `shiplab`

`Cargo.toml`:

```toml
[package]
name = "shiplab"
version = "0.1.0"
edition = "2024"
rust-version = "1.85"
publish = false
description = "A tiny service used to study configuration, health checks and graceful shutdown"

[dependencies]
signal-hook = "0.3"

[profile.release]
opt-level = 3
lto = "fat"
codegen-units = 1
panic = "abort"
strip = true
```

`build.rs`:

```rust
//! Records the compiler version and target so `/version` can report them.
use std::process::Command;

fn main() {
    let rustc = std::env::var("RUSTC").unwrap_or_else(|_| "rustc".into());
    let version = Command::new(rustc)
        .arg("--version")
        .output()
        .ok()
        .and_then(|o| String::from_utf8(o.stdout).ok())
        .map_or_else(|| "unknown".to_string(), |s| s.trim().to_string());
    println!("cargo:rustc-env=SHIPLAB_RUSTC={version}");
    println!(
        "cargo:rustc-env=SHIPLAB_TARGET={}",
        std::env::var("TARGET").unwrap_or_default()
    );
    println!("cargo:rerun-if-changed=build.rs");
}
```

`src/lib.rs`:

```rust
//! `shiplab`: a small HTTP service built to study how a Rust program is
//! configured, health-checked, stopped and shipped.
//!
//! The pieces are deliberately separate so each can be tested alone:
//! [`config`] reads settings, [`http`] parses requests, [`server`] runs the
//! accept loop and the graceful drain.
#![deny(missing_docs)]

pub mod config;
pub mod http;
pub mod server;

/// The crate version, as reported by `/version`.
pub const VERSION: &str = env!("CARGO_PKG_VERSION");
/// The compiler that built this binary (recorded by `build.rs`).
pub const RUSTC: &str = env!("SHIPLAB_RUSTC");
/// The target triple this binary was built for (recorded by `build.rs`).
pub const TARGET: &str = env!("SHIPLAB_TARGET");

/// Process exit codes, documented so an orchestrator can tell failures apart.
pub mod exit {
    /// Clean stop (including a completed graceful drain).
    pub const OK: i32 = 0;
    /// Invalid configuration.
    pub const CONFIG: i32 = 2;
    /// The drain deadline passed with requests still running.
    pub const DRAIN_TIMEOUT: i32 = 3;
    /// Could not bind the listening socket.
    pub const BIND: i32 = 4;
}
```

`src/config.rs`:

````rust
//! Configuration from environment variables (twelve-factor style).

use std::fmt;
use std::net::SocketAddr;
use std::time::Duration;

/// Runtime settings.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Config {
    /// Address to listen on (`SHIPLAB_ADDR`, default `127.0.0.1:8080`).
    pub addr: SocketAddr,
    /// How long a graceful drain may take (`SHIPLAB_DRAIN_MS`, default 10000).
    pub drain: Duration,
    /// Maximum simultaneous connections (`SHIPLAB_MAX_CONNS`, default 64).
    pub max_conns: usize,
    /// Per-connection read timeout (`SHIPLAB_READ_TIMEOUT_MS`, default 5000).
    pub read_timeout: Duration,
}

/// A configuration error that names the variable but never echoes its value.
#[derive(Debug, PartialEq, Eq)]
pub struct ConfigError {
    /// The offending variable.
    pub var: &'static str,
    /// What was wrong with it.
    pub problem: &'static str,
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "invalid {}: {}", self.var, self.problem)
    }
}

impl std::error::Error for ConfigError {}

const PREFIX: &str = "SHIPLAB_";
const KNOWN: [&str; 4] = ["ADDR", "DRAIN_MS", "MAX_CONNS", "READ_TIMEOUT_MS"];

impl Default for Config {
    fn default() -> Self {
        Config {
            addr: SocketAddr::from(([127, 0, 0, 1], 8080)),
            drain: Duration::from_millis(10_000),
            max_conns: 64,
            read_timeout: Duration::from_millis(5_000),
        }
    }
}

fn num<T: TryFrom<u64>>(
    var: &'static str,
    value: &str,
    lo: u64,
    hi: u64,
) -> Result<T, ConfigError> {
    let n: u64 = value.trim().parse().map_err(|_| ConfigError {
        var,
        problem: "not a non-negative integer",
    })?;
    if !(lo..=hi).contains(&n) {
        return Err(ConfigError {
            var,
            problem: "out of range",
        });
    }
    T::try_from(n).map_err(|_| ConfigError {
        var,
        problem: "out of range",
    })
}

impl Config {
    /// Builds a configuration from `(name, value)` pairs, such as
    /// `std::env::vars()`. Names without the `SHIPLAB_` prefix are ignored;
    /// unknown prefixed names are errors, so a typo cannot be silent.
    pub fn from_vars<I>(vars: I) -> Result<Config, ConfigError>
    where
        I: IntoIterator<Item = (String, String)>,
    {
        let mut cfg = Config::default();
        for (name, value) in vars {
            let Some(key) = name.strip_prefix(PREFIX) else {
                continue;
            };
            match key {
                "ADDR" => {
                    cfg.addr = value.trim().parse().map_err(|_| ConfigError {
                        var: "SHIPLAB_ADDR",
                        problem: "not a socket address like 127.0.0.1:8080",
                    })?;
                }
                "DRAIN_MS" => {
                    cfg.drain = Duration::from_millis(num("SHIPLAB_DRAIN_MS", &value, 0, 600_000)?)
                }
                "MAX_CONNS" => cfg.max_conns = num("SHIPLAB_MAX_CONNS", &value, 1, 10_000)?,
                "READ_TIMEOUT_MS" => {
                    cfg.read_timeout =
                        Duration::from_millis(num("SHIPLAB_READ_TIMEOUT_MS", &value, 1, 600_000)?)
                }
                _ => {
                    debug_assert!(!KNOWN.contains(&key));
                    return Err(ConfigError {
                        var: "SHIPLAB_*",
                        problem: "unknown variable (check the spelling)",
                    });
                }
            }
        }
        Ok(cfg)
    }
}
````

`src/http.rs`:

````rust
//! A deliberately small HTTP/1.1 request parser with hard limits.

/// Longest accepted request line, in bytes.
pub const MAX_REQUEST_LINE: usize = 8 * 1024;
/// Largest accepted request head (line plus headers), in bytes.
pub const MAX_HEAD: usize = 16 * 1024;

/// A parsed request line.
#[derive(Debug, PartialEq, Eq)]
pub struct Request<'a> {
    /// The method, e.g. `GET`.
    pub method: &'a str,
    /// The path without the query string.
    pub path: &'a str,
    /// The raw query string (no leading `?`), possibly empty.
    pub query: &'a str,
}

/// Why a request was rejected.
#[derive(Debug, PartialEq, Eq)]
pub enum ParseError {
    /// The head is not valid UTF-8, or the request line is malformed.
    Malformed,
    /// The request line or head exceeds the limits.
    TooLarge,
}

/// Parses the request line from the bytes read so far.
///
/// Returns `Ok(None)` while the head is incomplete (no blank line yet).
pub fn parse(buf: &[u8]) -> Result<Option<Request<'_>>, ParseError> {
    let line_end = buf.windows(2).position(|w| w == b"\r\n");
    match line_end {
        Some(l) if l > MAX_REQUEST_LINE => return Err(ParseError::TooLarge),
        None if buf.len() > MAX_REQUEST_LINE => return Err(ParseError::TooLarge),
        _ => {}
    }
    let Some(blank) = buf.windows(4).position(|w| w == b"\r\n\r\n") else {
        if buf.len() > MAX_HEAD {
            return Err(ParseError::TooLarge);
        }
        return Ok(None);
    };
    if blank + 4 > MAX_HEAD {
        return Err(ParseError::TooLarge);
    }
    let line = &buf[..line_end.unwrap_or(blank)];
    let line = std::str::from_utf8(line).map_err(|_| ParseError::Malformed)?;
    let mut parts = line.split(' ');
    let (Some(method), Some(target), Some(version), None) =
        (parts.next(), parts.next(), parts.next(), parts.next())
    else {
        return Err(ParseError::Malformed);
    };
    if method.is_empty()
        || !method.bytes().all(|b| b.is_ascii_uppercase())
        || !target.starts_with('/')
        || !(version == "HTTP/1.1" || version == "HTTP/1.0")
    {
        return Err(ParseError::Malformed);
    }
    let (path, query) = target.split_once('?').unwrap_or((target, ""));
    Ok(Some(Request {
        method,
        path,
        query,
    }))
}

/// Returns the value of `key` in a `a=1&b=2` query string.
pub fn query_param<'a>(query: &'a str, key: &str) -> Option<&'a str> {
    query
        .split('&')
        .filter_map(|kv| kv.split_once('='))
        .find(|(k, _)| *k == key)
        .map(|(_, v)| v)
}
````

`src/server.rs`:

````rust
//! The accept loop, the router and the graceful drain.

use std::io::{ErrorKind, Read, Write};
use std::net::{TcpListener, TcpStream};
use std::sync::Arc;
use std::sync::atomic::{AtomicBool, AtomicUsize, Ordering};
use std::time::{Duration, Instant};

use crate::config::Config;
use crate::http::{self, ParseError, Request};
use crate::{RUSTC, TARGET, VERSION, exit};

/// Longest delay `/slow` will honor, so a client cannot park a thread forever.
pub const MAX_SLOW_MS: u64 = 5_000;

/// What the router decided to answer.
#[derive(Debug, PartialEq, Eq)]
pub struct Reply {
    /// Status code.
    pub status: u16,
    /// Response body.
    pub body: String,
}

impl Reply {
    fn new(status: u16, body: impl Into<String>) -> Reply {
        Reply {
            status,
            body: body.into(),
        }
    }
}

fn reason(status: u16) -> &'static str {
    match status {
        200 => "OK",
        400 => "Bad Request",
        404 => "Not Found",
        405 => "Method Not Allowed",
        431 => "Request Header Fields Too Large",
        _ => "Service Unavailable",
    }
}

/// The delay `/slow` will use for a query string: missing or invalid is 0,
/// anything above [`MAX_SLOW_MS`] is capped.
pub fn slow_ms(query: &str) -> u64 {
    http::query_param(query, "ms")
        .and_then(|v| v.parse::<u64>().ok())
        .unwrap_or(0)
        .min(MAX_SLOW_MS)
}

/// Maps a request to a reply. `draining` is true once shutdown has started.
///
/// * `/healthz` is *liveness*: the process is up. Always 200.
/// * `/readyz` is *readiness*: 200 normally, 503 while draining, so a load
///   balancer stops sending new traffic.
/// * `/version` reports build information.
/// * `/slow?ms=N` waits N milliseconds (capped), to simulate a long request.
pub fn route(req: &Request<'_>, draining: bool) -> Reply {
    if req.method != "GET" {
        return Reply::new(405, "method not allowed\n");
    }
    match req.path {
        "/healthz" => Reply::new(200, "ok\n"),
        "/readyz" if draining => Reply::new(503, "draining\n"),
        "/readyz" => Reply::new(200, "ready\n"),
        "/version" => Reply::new(
            200,
            format!(
                "{{\"version\":\"{VERSION}\",\"rustc\":\"{RUSTC}\",\"target\":\"{TARGET}\"}}\n"
            ),
        ),
        "/slow" => {
            std::thread::sleep(Duration::from_millis(slow_ms(req.query)));
            Reply::new(200, "done\n")
        }
        _ => Reply::new(404, "not found\n"),
    }
}

fn write_reply(stream: &mut TcpStream, reply: &Reply) {
    let head = format!(
        "HTTP/1.1 {} {}\r\nContent-Type: text/plain\r\nContent-Length: {}\r\nConnection: close\r\n\r\n",
        reply.status,
        reason(reply.status),
        reply.body.len()
    );
    let _ = stream.write_all(head.as_bytes());
    let _ = stream.write_all(reply.body.as_bytes());
    let _ = stream.flush();
}

/// Decrements the in-flight counter when a connection ends, however it ends.
struct InFlight(Arc<AtomicUsize>);

impl Drop for InFlight {
    fn drop(&mut self) {
        self.0.fetch_sub(1, Ordering::SeqCst);
    }
}

fn handle(mut stream: TcpStream, cfg: &Config, draining: &AtomicBool, _guard: InFlight) {
    let _ = stream.set_nonblocking(false);
    let _ = stream.set_read_timeout(Some(cfg.read_timeout));
    let mut buf = Vec::with_capacity(512);
    let mut chunk = [0u8; 1024];
    loop {
        match http::parse(&buf) {
            Ok(Some(req)) => {
                let reply = route(&req, draining.load(Ordering::SeqCst));
                write_reply(&mut stream, &reply);
                return;
            }
            Ok(None) => {}
            Err(ParseError::TooLarge) => {
                return write_reply(&mut stream, &Reply::new(431, "too large\n"));
            }
            Err(ParseError::Malformed) => {
                return write_reply(&mut stream, &Reply::new(400, "bad request\n"));
            }
        }
        match stream.read(&mut chunk) {
            Ok(0) | Err(_) => return, // closed, timed out or reset: just drop it
            Ok(n) => buf.extend_from_slice(&chunk[..n]),
        }
    }
}

/// Serves until `shutdown` becomes true, then drains.
///
/// Draining keeps accepting (so `/readyz` can answer 503) but returns once no
/// connection is in flight. Returns [`exit::OK`] on a clean drain or
/// [`exit::DRAIN_TIMEOUT`] if `cfg.drain` elapsed first.
pub fn serve(listener: &TcpListener, cfg: &Config, shutdown: &AtomicBool) -> i32 {
    listener.set_nonblocking(true).expect("set_nonblocking");
    let in_flight = Arc::new(AtomicUsize::new(0));
    let draining = Arc::new(AtomicBool::new(false));
    let mut drain_started: Option<Instant> = None;
    loop {
        if shutdown.load(Ordering::SeqCst) && drain_started.is_none() {
            drain_started = Some(Instant::now());
            draining.store(true, Ordering::SeqCst);
        }
        if let Some(t0) = drain_started {
            if in_flight.load(Ordering::SeqCst) == 0 {
                return exit::OK;
            }
            if t0.elapsed() >= cfg.drain {
                return exit::DRAIN_TIMEOUT;
            }
        }
        match listener.accept() {
            Ok((mut stream, _)) => {
                if in_flight.load(Ordering::SeqCst) >= cfg.max_conns {
                    write_reply(&mut stream, &Reply::new(503, "busy\n"));
                    continue;
                }
                in_flight.fetch_add(1, Ordering::SeqCst);
                let guard = InFlight(in_flight.clone());
                let (cfg, draining) = (cfg.clone(), draining.clone());
                std::thread::spawn(move || handle(stream, &cfg, &draining, guard));
            }
            Err(e) if e.kind() == ErrorKind::WouldBlock => {
                std::thread::sleep(Duration::from_millis(2));
            }
            Err(_) => std::thread::sleep(Duration::from_millis(10)),
        }
    }
}
````

`src/bin/shipd.rs`:

```rust
//! The `shipd` binary: configuration, signals, logging and exit codes.

use std::io::Write;
use std::net::TcpListener;
use std::sync::Arc;
use std::sync::atomic::AtomicBool;

use shiplab::{config::Config, exit, server};

fn main() {
    let cfg = match Config::from_vars(std::env::vars()) {
        Ok(c) => c,
        Err(e) => {
            eprintln!("level=error msg=\"{e}\"");
            std::process::exit(exit::CONFIG);
        }
    };
    let listener = match TcpListener::bind(cfg.addr) {
        Ok(l) => l,
        Err(e) => {
            eprintln!("level=error msg=\"cannot bind\" kind={:?}", e.kind());
            std::process::exit(exit::BIND);
        }
    };
    let shutdown = Arc::new(AtomicBool::new(false));
    for sig in [signal_hook::consts::SIGTERM, signal_hook::consts::SIGINT] {
        signal_hook::flag::register(sig, shutdown.clone()).expect("register signal handler");
    }
    let addr = listener.local_addr().expect("local_addr");
    println!("listening addr={addr} version={}", shiplab::VERSION);
    std::io::stdout().flush().ok();
    let code = server::serve(&listener, &cfg, &shutdown);
    eprintln!("level=info msg=\"stopped\" exit_code={code}");
    std::process::exit(code);
}
```

`tests/common/mod.rs`:

````rust
//! Helpers that start the real `shipd` binary and talk to it over TCP.
#![allow(dead_code)]

use std::io::{BufRead, BufReader, Read, Write};
use std::net::{SocketAddr, TcpStream};
use std::process::{Child, Command, Stdio};
use std::time::{Duration, Instant};

pub struct Daemon {
    pub child: Child,
    pub addr: SocketAddr,
}

impl Daemon {
    /// Starts `shipd` on an OS-chosen port with the given extra variables.
    pub fn start(extra: &[(&str, &str)]) -> Daemon {
        let mut cmd = Command::new(env!("CARGO_BIN_EXE_shipd"));
        cmd.env_clear()
            .env("SHIPLAB_ADDR", "127.0.0.1:0")
            .stdout(Stdio::piped())
            .stderr(Stdio::piped());
        for (k, v) in extra {
            cmd.env(k, v);
        }
        let mut child = cmd.spawn().expect("spawn shipd");
        let mut line = String::new();
        BufReader::new(child.stdout.as_mut().unwrap())
            .read_line(&mut line)
            .expect("read the listening line");
        let addr = line
            .split_whitespace()
            .find_map(|w| w.strip_prefix("addr="))
            .unwrap_or_else(|| panic!("no addr in {line:?}"))
            .parse()
            .expect("socket address");
        Daemon { child, addr }
    }

    pub fn pid(&self) -> u32 {
        self.child.id()
    }

    pub fn signal(&self, name: &str) {
        let ok = Command::new("kill")
            .args([&format!("-{name}"), &self.pid().to_string()])
            .status()
            .unwrap()
            .success();
        assert!(ok, "kill -{name} failed");
    }

    /// Waits up to `limit` for the process to exit and returns its code.
    pub fn wait_exit(&mut self, limit: Duration) -> Option<i32> {
        let t0 = Instant::now();
        while t0.elapsed() < limit {
            if let Some(st) = self.child.try_wait().unwrap() {
                return st.code();
            }
            std::thread::sleep(Duration::from_millis(5));
        }
        None
    }

    pub fn stderr(&mut self) -> String {
        let mut s = String::new();
        if let Some(e) = self.child.stderr.as_mut() {
            e.read_to_string(&mut s).ok();
        }
        s
    }
}

impl Drop for Daemon {
    fn drop(&mut self) {
        let _ = self.child.kill();
        let _ = self.child.wait();
    }
}

/// Sends raw bytes and returns `(status, body)`.
pub fn raw(addr: SocketAddr, request: &[u8]) -> (u16, String) {
    let mut s = TcpStream::connect(addr).expect("connect");
    s.set_read_timeout(Some(Duration::from_secs(10))).unwrap();
    s.write_all(request).unwrap();
    let mut out = String::new();
    s.read_to_string(&mut out).ok();
    parse_response(&out)
}

pub fn get(addr: SocketAddr, path: &str) -> (u16, String) {
    raw(
        addr,
        format!("GET {path} HTTP/1.1\r\nHost: x\r\n\r\n").as_bytes(),
    )
}

/// Sends raw bytes and returns the complete response text.
pub fn raw_text(addr: SocketAddr, request: &[u8]) -> String {
    let mut s = TcpStream::connect(addr).expect("connect");
    s.set_read_timeout(Some(Duration::from_secs(10))).unwrap();
    s.write_all(request).unwrap();
    let mut out = String::new();
    s.read_to_string(&mut out).ok();
    out
}

pub fn parse_response(text: &str) -> (u16, String) {
    // Every response must announce exactly the body length it sends.
    if let Some((head, body)) = text.split_once("\r\n\r\n") {
        let declared: usize = head
            .lines()
            .find_map(|l| l.strip_prefix("Content-Length: "))
            .and_then(|v| v.parse().ok())
            .expect("Content-Length header");
        assert_eq!(declared, body.len(), "Content-Length must match the body");
    }
    let status = text
        .split_whitespace()
        .nth(1)
        .and_then(|s| s.parse().ok())
        .unwrap_or(0);
    let body = text
        .split_once("\r\n\r\n")
        .map_or("", |(_, b)| b)
        .to_string();
    (status, body)
}
````

`tests/units.rs`:

````rust
use std::time::Duration;

use shiplab::config::{Config, ConfigError};
use shiplab::http::{self, MAX_HEAD, MAX_REQUEST_LINE, ParseError, Request};
use shiplab::server::{MAX_SLOW_MS, Reply, route, slow_ms};

fn vars(pairs: &[(&str, &str)]) -> Vec<(String, String)> {
    pairs
        .iter()
        .map(|(k, v)| (k.to_string(), v.to_string()))
        .collect()
}

fn cfg(pairs: &[(&str, &str)]) -> Result<Config, ConfigError> {
    Config::from_vars(vars(pairs))
}

// ---------- config ----------

#[test]
fn defaults_apply_when_nothing_is_set() {
    let c = cfg(&[("PATH", "/usr/bin"), ("HOME", "/root")]).unwrap();
    assert_eq!(c, Config::default());
    assert_eq!(c.addr.to_string(), "127.0.0.1:8080");
    assert_eq!(c.drain, Duration::from_millis(10_000));
    assert_eq!(c.max_conns, 64);
    assert_eq!(c.read_timeout, Duration::from_millis(5_000));
}

#[test]
fn every_variable_is_wired_to_its_own_field() {
    let c = cfg(&[
        ("SHIPLAB_ADDR", "0.0.0.0:9000"),
        ("SHIPLAB_DRAIN_MS", "1234"),
        ("SHIPLAB_MAX_CONNS", "17"),
        ("SHIPLAB_READ_TIMEOUT_MS", "777"),
    ])
    .unwrap();
    assert_eq!(c.addr.to_string(), "0.0.0.0:9000");
    assert_eq!(c.drain, Duration::from_millis(1234));
    assert_eq!(c.max_conns, 17);
    assert_eq!(c.read_timeout, Duration::from_millis(777));
}

#[test]
fn unprefixed_names_are_ignored_but_misspelled_prefixed_names_are_errors() {
    assert!(cfg(&[("MAX_CONNS", "not a number"), ("XSHIPLAB_ADDR", "bad")]).is_ok());
    let e = cfg(&[("SHIPLAB_DRAIN", "5")]).unwrap_err();
    assert_eq!(e.var, "SHIPLAB_*");
    assert!(e.to_string().contains("unknown variable"));
}

#[test]
fn errors_name_the_variable_and_never_echo_the_value() {
    let e = cfg(&[("SHIPLAB_MAX_CONNS", "hunter2secret")]).unwrap_err();
    assert_eq!(e.var, "SHIPLAB_MAX_CONNS");
    let text = e.to_string();
    assert!(text.contains("SHIPLAB_MAX_CONNS") && !text.contains("hunter2secret"));
    let e = cfg(&[("SHIPLAB_ADDR", "hunter2secret")]).unwrap_err();
    assert!(!e.to_string().contains("hunter2secret"));
    assert!(cfg(&[("SHIPLAB_DRAIN_MS", "-1")]).is_err());
    assert!(cfg(&[("SHIPLAB_DRAIN_MS", "")]).is_err());
    assert!(cfg(&[("SHIPLAB_DRAIN_MS", "1.5")]).is_err());
}

#[test]
fn every_range_is_enforced_at_both_ends() {
    let ok = |k: &str, v: &str| cfg(&[(k, v)]).is_ok();
    assert!(ok("SHIPLAB_DRAIN_MS", "0") && ok("SHIPLAB_DRAIN_MS", "600000"));
    assert!(!ok("SHIPLAB_DRAIN_MS", "600001"));
    assert!(!ok("SHIPLAB_MAX_CONNS", "0") && ok("SHIPLAB_MAX_CONNS", "1"));
    assert!(ok("SHIPLAB_MAX_CONNS", "10000") && !ok("SHIPLAB_MAX_CONNS", "10001"));
    assert!(!ok("SHIPLAB_READ_TIMEOUT_MS", "0") && ok("SHIPLAB_READ_TIMEOUT_MS", "1"));
    assert!(ok("SHIPLAB_READ_TIMEOUT_MS", "600000"));
    assert!(!ok("SHIPLAB_READ_TIMEOUT_MS", "600001"));
    assert!(
        ok("SHIPLAB_MAX_CONNS", " 5 "),
        "surrounding spaces are trimmed"
    );
}

#[test]
fn later_values_override_earlier_ones_for_the_same_name() {
    let c = cfg(&[("SHIPLAB_MAX_CONNS", "3"), ("SHIPLAB_MAX_CONNS", "9")]).unwrap();
    assert_eq!(c.max_conns, 9);
}

// ---------- http parsing ----------

fn parsed(buf: &[u8]) -> Request<'_> {
    http::parse(buf).unwrap().expect("complete")
}

#[test]
fn a_complete_request_line_is_split_into_parts() {
    let r = parsed(b"GET /slow?ms=30&x=1 HTTP/1.1\r\nHost: a\r\n\r\n");
    assert_eq!(
        r,
        Request {
            method: "GET",
            path: "/slow",
            query: "ms=30&x=1"
        }
    );
    let r = parsed(b"POST /a HTTP/1.0\r\n\r\n");
    assert_eq!((r.method, r.path, r.query), ("POST", "/a", ""));
}

#[test]
fn an_incomplete_head_asks_for_more_bytes() {
    assert_eq!(http::parse(b""), Ok(None));
    assert_eq!(http::parse(b"GET / HTTP/1.1\r\nHost: a\r\n"), Ok(None));
    assert_eq!(http::parse(b"GET / HTTP/1.1\r\n\r"), Ok(None));
}

#[test]
fn malformed_request_lines_are_rejected() {
    for bad in [
        &b"get / HTTP/1.1\r\n\r\n"[..],
        b"GET x HTTP/1.1\r\n\r\n",
        b"GET / HTTP/2\r\n\r\n",
        b"GET /\r\n\r\n",
        b"GET / HTTP/1.1 extra\r\n\r\n",
        b" / HTTP/1.1\r\n\r\n",
        b"GET /\xff HTTP/1.1\r\n\r\n",
        b"\r\n\r\n",
    ] {
        assert_eq!(http::parse(bad), Err(ParseError::Malformed), "{bad:?}");
    }
}

fn request_with_line_len(line_len: usize) -> Vec<u8> {
    // "GET /" + padding + " HTTP/1.1" has exactly `line_len` bytes.
    let fixed = "GET / HTTP/1.1".len();
    let mut v = b"GET /".to_vec();
    v.extend(std::iter::repeat_n(b'a', line_len - fixed));
    v.extend_from_slice(b" HTTP/1.1\r\n\r\n");
    v
}

#[test]
fn the_request_line_limit_is_exact() {
    assert!(http::parse(&request_with_line_len(MAX_REQUEST_LINE)).is_ok());
    assert_eq!(
        http::parse(&request_with_line_len(MAX_REQUEST_LINE + 1)),
        Err(ParseError::TooLarge)
    );
    // No line terminator yet, but already over the limit: do not wait for more.
    let long = vec![b'a'; MAX_REQUEST_LINE + 1];
    assert_eq!(http::parse(&long), Err(ParseError::TooLarge));
    assert_eq!(http::parse(&long[..MAX_REQUEST_LINE]), Ok(None));
}

#[test]
fn the_head_limit_is_exact() {
    let mut head = b"GET / HTTP/1.1\r\n".to_vec();
    // pad with header bytes until head (including the blank line) is MAX_HEAD
    let pad = MAX_HEAD - head.len() - 4 - "X: ".len();
    head.extend_from_slice(b"X: ");
    head.extend(std::iter::repeat_n(b'v', pad));
    head.extend_from_slice(b"\r\n\r\n");
    assert_eq!(head.len(), MAX_HEAD);
    assert!(http::parse(&head).unwrap().is_some());
    head.insert(20, b'v');
    assert_eq!(head.len(), MAX_HEAD + 1);
    assert_eq!(http::parse(&head), Err(ParseError::TooLarge));
    // an unterminated head over the limit is also refused
    let open = [b"GET / HTTP/1.1\r\nX: ".to_vec(), vec![b'v'; MAX_HEAD]].concat();
    assert_eq!(http::parse(&open), Err(ParseError::TooLarge));
}

#[test]
fn query_parameters_are_looked_up_by_exact_key() {
    assert_eq!(http::query_param("ms=5&a=b", "a"), Some("b"));
    assert_eq!(http::query_param("ms=5&a=b", "ms"), Some("5"));
    assert_eq!(http::query_param("ms=5", "m"), None);
    assert_eq!(http::query_param("", "ms"), None);
    assert_eq!(http::query_param("flag&ms=7", "ms"), Some("7"));
}

// ---------- routing ----------

fn req<'a>(method: &'a str, path: &'a str, query: &'a str) -> Request<'a> {
    Request {
        method,
        path,
        query,
    }
}

#[test]
fn routes_answer_the_documented_statuses() {
    let get = |p| route(&req("GET", p, ""), false);
    assert_eq!(
        get("/healthz"),
        Reply {
            status: 200,
            body: "ok\n".into()
        }
    );
    assert_eq!(
        get("/readyz"),
        Reply {
            status: 200,
            body: "ready\n".into()
        }
    );
    assert_eq!(get("/nope").status, 404);
    assert_eq!(get("/healthz/").status, 404);
    assert_eq!(route(&req("POST", "/healthz", ""), false).status, 405);
    assert_eq!(route(&req("GET", "/slow", "ms=1"), false).body, "done\n");
}

#[test]
fn readiness_flips_to_503_while_draining_but_liveness_does_not() {
    let r = route(&req("GET", "/readyz", ""), true);
    assert_eq!((r.status, r.body.as_str()), (503, "draining\n"));
    assert_eq!(route(&req("GET", "/healthz", ""), true).status, 200);
}

#[test]
fn version_reports_build_information() {
    let r = route(&req("GET", "/version", ""), false);
    assert_eq!(r.status, 200);
    assert!(
        r.body
            .contains(&format!("\"version\":\"{}\"", shiplab::VERSION))
    );
    assert!(r.body.contains("rustc 1."), "{}", r.body);
    assert!(
        r.body.contains("x86_64") || r.body.contains("aarch64"),
        "{}",
        r.body
    );
}

#[test]
fn slow_delay_is_parsed_defaulted_and_capped() {
    assert_eq!(slow_ms(""), 0);
    assert_eq!(slow_ms("ms=250"), 250);
    assert_eq!(slow_ms("ms=abc"), 0);
    assert_eq!(slow_ms("ms=-5"), 0);
    assert_eq!(MAX_SLOW_MS, 5_000, "the documented cap");
    assert_eq!(slow_ms(&format!("ms={MAX_SLOW_MS}")), MAX_SLOW_MS);
    assert_eq!(slow_ms(&format!("ms={}", MAX_SLOW_MS + 1)), MAX_SLOW_MS);
    assert_eq!(
        slow_ms("ms=99999999999999999999"),
        0,
        "overflow is not a huge sleep"
    );
}

#[test]
fn an_unterminated_head_is_only_refused_beyond_the_limit() {
    let prefix = b"GET / HTTP/1.1\r\nX: ".to_vec();
    let mut buf = prefix.clone();
    buf.extend(std::iter::repeat_n(b'v', MAX_HEAD - prefix.len()));
    assert_eq!(buf.len(), MAX_HEAD);
    assert_eq!(
        http::parse(&buf),
        Ok(None),
        "exactly at the limit: keep reading"
    );
    buf.push(b'v');
    assert_eq!(http::parse(&buf), Err(ParseError::TooLarge));
}
````

`tests/service.rs`:

````rust
mod common;

use std::io::{Read, Write};
use std::net::TcpStream;
use std::time::{Duration, Instant};

use common::{Daemon, get, raw, raw_text};

#[test]
fn health_readiness_version_and_errors_over_real_http() {
    let d = Daemon::start(&[]);
    assert_eq!(get(d.addr, "/healthz"), (200, "ok\n".into()));
    assert_eq!(get(d.addr, "/readyz"), (200, "ready\n".into()));
    let (st, body) = get(d.addr, "/version");
    assert_eq!(st, 200);
    assert!(body.contains(shiplab::VERSION) && body.contains("rustc 1."));
    assert_eq!(get(d.addr, "/missing").0, 404);
    assert_eq!(raw(d.addr, b"POST /healthz HTTP/1.1\r\n\r\n").0, 405);
    assert_eq!(raw(d.addr, b"nonsense\r\n\r\n").0, 400);
    let big = format!("GET /{} HTTP/1.1\r\n\r\n", "a".repeat(9000));
    assert_eq!(raw(d.addr, big.as_bytes()).0, 431);
}

#[test]
fn a_request_split_across_packets_is_reassembled() {
    let d = Daemon::start(&[]);
    let mut s = TcpStream::connect(d.addr).unwrap();
    for part in [&b"GET /heal"[..], b"thz HTTP/1.1\r\nHo", b"st: x\r\n\r\n"] {
        s.write_all(part).unwrap();
        s.flush().unwrap();
        std::thread::sleep(Duration::from_millis(30));
    }
    let mut out = String::new();
    s.read_to_string(&mut out).unwrap();
    assert!(out.starts_with("HTTP/1.1 200 OK"), "{out}");
}

#[test]
fn connections_beyond_the_limit_get_503_and_recover_afterwards() {
    let d = Daemon::start(&[("SHIPLAB_MAX_CONNS", "2")]);
    let hold1 = TcpStream::connect(d.addr).unwrap();
    let hold2 = TcpStream::connect(d.addr).unwrap();
    std::thread::sleep(Duration::from_millis(100));
    assert_eq!(get(d.addr, "/healthz").0, 503);
    drop((hold1, hold2));
    std::thread::sleep(Duration::from_millis(150));
    assert_eq!(get(d.addr, "/healthz").0, 200, "slots are released");
}

#[test]
fn a_silent_client_is_dropped_after_the_read_timeout() {
    let d = Daemon::start(&[("SHIPLAB_READ_TIMEOUT_MS", "150")]);
    let mut s = TcpStream::connect(d.addr).unwrap();
    s.set_read_timeout(Some(Duration::from_secs(5))).unwrap();
    let t0 = Instant::now();
    let mut buf = [0u8; 8];
    let n = s.read(&mut buf).unwrap();
    let waited = t0.elapsed();
    assert_eq!(n, 0, "the server closed the connection");
    assert!(
        waited >= Duration::from_millis(120) && waited < Duration::from_secs(3),
        "{waited:?}"
    );
}

#[test]
fn an_idle_server_exits_zero_quickly_on_sigterm_and_on_sigint() {
    for sig in ["TERM", "INT"] {
        let mut d = Daemon::start(&[]);
        d.signal(sig);
        let t0 = Instant::now();
        assert_eq!(d.wait_exit(Duration::from_secs(5)), Some(0), "SIG{sig}");
        assert!(t0.elapsed() < Duration::from_secs(2));
        assert!(d.stderr().contains("exit_code=0"));
    }
}

#[test]
fn sigterm_lets_an_in_flight_request_finish_and_flips_readiness() {
    let mut d = Daemon::start(&[("SHIPLAB_DRAIN_MS", "5000")]);
    let addr = d.addr;
    let slow = std::thread::spawn(move || get(addr, "/slow?ms=700"));
    std::thread::sleep(Duration::from_millis(200)); // the request is being served
    d.signal("TERM");
    std::thread::sleep(Duration::from_millis(100));
    assert_eq!(get(addr, "/readyz"), (503, "draining\n".into()));
    assert_eq!(
        get(addr, "/healthz").0,
        200,
        "liveness stays up while draining"
    );
    assert_eq!(
        slow.join().unwrap(),
        (200, "done\n".into()),
        "the slow request completed"
    );
    assert_eq!(d.wait_exit(Duration::from_secs(5)), Some(0));
}

#[test]
fn a_stuck_connection_makes_the_drain_time_out_with_code_3() {
    let mut d = Daemon::start(&[
        ("SHIPLAB_DRAIN_MS", "200"),
        ("SHIPLAB_READ_TIMEOUT_MS", "60000"),
    ]);
    let _stuck = TcpStream::connect(d.addr).unwrap(); // sends nothing
    std::thread::sleep(Duration::from_millis(100));
    d.signal("TERM");
    let t0 = Instant::now();
    assert_eq!(d.wait_exit(Duration::from_secs(5)), Some(3));
    let took = t0.elapsed();
    assert!(
        took >= Duration::from_millis(180) && took < Duration::from_secs(3),
        "{took:?}"
    );
    assert!(d.stderr().contains("exit_code=3"));
}

#[test]
fn a_zero_drain_deadline_gives_up_at_once_when_work_is_running() {
    let mut d = Daemon::start(&[("SHIPLAB_DRAIN_MS", "0")]);
    let _stuck = TcpStream::connect(d.addr).unwrap();
    std::thread::sleep(Duration::from_millis(100));
    d.signal("TERM");
    assert_eq!(d.wait_exit(Duration::from_secs(5)), Some(3));
}

#[test]
fn bad_configuration_exits_2_naming_the_variable_but_not_the_value() {
    let out = std::process::Command::new(env!("CARGO_BIN_EXE_shipd"))
        .env_clear()
        .env("SHIPLAB_MAX_CONNS", "s3cr3t-value")
        .output()
        .unwrap();
    assert_eq!(out.status.code(), Some(2));
    let err = String::from_utf8_lossy(&out.stderr);
    assert!(
        err.contains("SHIPLAB_MAX_CONNS") && !err.contains("s3cr3t-value"),
        "{err}"
    );
    assert!(out.stdout.is_empty(), "nothing is announced on stdout");
}

#[test]
fn binding_a_busy_port_exits_4() {
    let d = Daemon::start(&[]);
    let out = std::process::Command::new(env!("CARGO_BIN_EXE_shipd"))
        .env_clear()
        .env("SHIPLAB_ADDR", d.addr.to_string())
        .output()
        .unwrap();
    assert_eq!(out.status.code(), Some(4));
    assert!(String::from_utf8_lossy(&out.stderr).contains("cannot bind"));
}

#[test]
fn status_lines_carry_the_standard_reason_phrases_and_headers() {
    let d = Daemon::start(&[("SHIPLAB_MAX_CONNS", "1")]);
    let status_line = |req: &[u8]| {
        let t = raw_text(d.addr, req);
        assert!(t.contains("Content-Type: text/plain\r\n"), "{t}");
        assert!(t.contains("Connection: close\r\n"), "{t}");
        t.lines().next().unwrap().to_string()
    };
    assert_eq!(
        status_line(b"GET /healthz HTTP/1.1\r\n\r\n"),
        "HTTP/1.1 200 OK"
    );
    assert_eq!(
        status_line(b"GET /x HTTP/1.1\r\n\r\n"),
        "HTTP/1.1 404 Not Found"
    );
    assert_eq!(
        status_line(b"PUT /x HTTP/1.1\r\n\r\n"),
        "HTTP/1.1 405 Method Not Allowed"
    );
    assert_eq!(status_line(b"junk\r\n\r\n"), "HTTP/1.1 400 Bad Request");
    let big = format!("GET /{} HTTP/1.1\r\n\r\n", "a".repeat(9000));
    assert_eq!(
        status_line(big.as_bytes()),
        "HTTP/1.1 431 Request Header Fields Too Large"
    );
    let _hold = std::net::TcpStream::connect(d.addr).unwrap();
    std::thread::sleep(Duration::from_millis(100));
    assert_eq!(
        status_line(b"GET /healthz HTTP/1.1\r\n\r\n"),
        "HTTP/1.1 503 Service Unavailable"
    );
}
````

`Dockerfile` (next to a static `shipd` built as in §2.3), and the commands I used:

```dockerfile
FROM scratch
COPY shipd /shipd
ENV SHIPLAB_ADDR=0.0.0.0:8080
EXPOSE 8080
USER 65532:65532
ENTRYPOINT ["/shipd"]
```

```text
docker build -t shiplab:test .
docker run -d --name s1 --network host -e SHIPLAB_ADDR=127.0.0.1:18080 shiplab:test
docker stop s1        # waits for the drain; check `docker inspect s1 --format '{{.State.ExitCode}}'`
```

The systemd unit is in §5.1.

### `wasmlab`

`Cargo.toml`:

```toml
[package]
name = "wasmlab"
version = "0.1.0"
edition = "2024"
rust-version = "1.85"
publish = false
description = "A small library exported over the plain WebAssembly C ABI"

[lib]
crate-type = ["cdylib", "rlib"]

[features]
# Adds an export that uses `format!`, to measure what formatting costs in size.
fmt = []
# Adds an import (`env.host_log`) and an export that calls it; wasm32 only.
hostlog = []

[dev-dependencies]
wasmi = "0.51"

[profile.release]
opt-level = "s"
lto = true
codegen-units = 1
panic = "abort"
strip = true
```

`src/lib.rs`:

````rust
//! Pure functions plus a tiny memory ABI so a host (a browser, Node, or an
//! embedding runtime) can call them across the WebAssembly boundary.
//!
//! The boundary only carries numbers. Anything bigger travels through the
//! module's linear memory: the host asks us for a buffer with [`alloc`],
//! writes bytes into it, passes `(ptr, len)`, and frees what we return with
//! [`dealloc`].
#![deny(missing_docs)]

/// The "no result" value returned by [`slugify`] on invalid input.
pub const NONE: u64 = 0;

/// Longest slug [`slugify_str`] produces, in bytes.
pub const MAX_SLUG: usize = 64;

/// Packs a pointer and a length into one `u64` (pointer in the high half).
///
/// WebAssembly (without the multi-value feature in the C ABI) returns one
/// number, so a `(ptr, len)` pair is folded into an `i64`; JavaScript sees it
/// as a `BigInt`.
pub fn pack(ptr: u32, len: u32) -> u64 {
    (u64::from(ptr) << 32) | u64::from(len)
}

/// Splits a value made by [`pack`].
pub fn unpack(v: u64) -> (u32, u32) {
    ((v >> 32) as u32, (v & 0xffff_ffff) as u32)
}

/// 64-bit FNV-1a hash (a simple, non-cryptographic hash; do not use it for
/// anything security related).
pub fn fnv1a_64(bytes: &[u8]) -> u64 {
    let mut h: u64 = 0xcbf2_9ce4_8422_2325;
    for &b in bytes {
        h ^= u64::from(b);
        h = h.wrapping_mul(0x0000_0100_0000_01b3);
    }
    h
}

/// Turns text into a URL slug: ASCII letters and digits are lower-cased and
/// kept, every other run of characters becomes one `-`, leading and trailing
/// dashes are removed, and the result is cut at [`MAX_SLUG`] bytes (without
/// leaving a trailing dash).
pub fn slugify_str(text: &str) -> String {
    let mut out = String::new();
    let mut pending_dash = false;
    for c in text.chars() {
        if c.is_ascii_alphanumeric() {
            if pending_dash && !out.is_empty() {
                out.push('-');
            }
            pending_dash = false;
            out.push(c.to_ascii_lowercase());
        } else {
            pending_dash = true;
        }
        if out.len() >= MAX_SLUG {
            break;
        }
    }
    out.truncate(MAX_SLUG);
    // Cutting can leave a dash at the end (a dash was added, then the letter
    // after it was cut off); a slug never ends with one.
    while out.ends_with('-') {
        out.pop();
    }
    out
}

/// Largest `n` that [`prime_count`] will sieve (about 200 MB of memory).
pub const MAX_SIEVE: u32 = 200_000_000;

/// Whether [`prime_count`] will sieve `n` (rather than refuse it).
pub fn sieve_allowed(n: u32) -> bool {
    n <= MAX_SIEVE
}

/// Counts the primes `<= n` with a sieve of Eratosthenes (allocates `n + 1`
/// bytes, which makes the module grow its memory for large `n`).
///
/// Returns `u32::MAX` if `n > `[`MAX_SIEVE`], instead of trying to allocate
/// gigabytes.
///
/// All index arithmetic is checked: on `wasm32`, `usize` is 32 bits wide, so
/// `i * i` overflows once `i > 65 535`, long before it would on a 64-bit host.
pub fn prime_count(n: u32) -> u32 {
    if !sieve_allowed(n) {
        return u32::MAX;
    }
    if n < 2 {
        return 0;
    }
    let n = n as usize;
    let mut composite = vec![false; n + 1];
    let mut count = 0;
    for i in 2..=n {
        if !composite[i] {
            count += 1;
            let Some(mut j) = i.checked_mul(i) else {
                continue; // i * i does not fit: no multiples <= n to mark
            };
            while j <= n {
                composite[j] = true;
                match j.checked_add(i) {
                    Some(next) => j = next,
                    None => break,
                }
            }
        }
    }
    count
}

// ---------------------------------------------------------------------------
// The exported ABI. Plain `extern "C"` functions with numeric arguments.
// ---------------------------------------------------------------------------

/// Adds two numbers with wrapping arithmetic (no overflow trap).
#[unsafe(no_mangle)]
pub extern "C" fn add(a: i32, b: i32) -> i32 {
    a.wrapping_add(b)
}

/// Integer division that **panics** when `b == 0`. Compiled with
/// `panic = "abort"` for WebAssembly, the panic becomes a trap.
#[unsafe(no_mangle)]
pub extern "C" fn divide(a: i32, b: i32) -> i32 {
    assert!(b != 0, "division by zero");
    a.wrapping_div(b)
}

/// Returns the number of primes `<= n`. See [`prime_count`].
#[unsafe(no_mangle)]
pub extern "C" fn primes(n: u32) -> u32 {
    prime_count(n)
}

/// Allocates `len` bytes in this module's memory and returns the address.
/// The caller owns the block and must return it with [`dealloc`] using the
/// same `len`. Returns a non-null dangling pointer when `len == 0`.
#[unsafe(no_mangle)]
pub extern "C" fn alloc(len: usize) -> *mut u8 {
    let boxed: Box<[u8]> = vec![0u8; len].into_boxed_slice();
    Box::into_raw(boxed).cast::<u8>()
}

/// Frees a block obtained from [`alloc`] or returned by [`slugify`].
///
/// # Safety
/// `ptr` and `len` must be exactly a pointer and length previously handed out
/// by this module and not yet freed.
#[unsafe(no_mangle)]
pub unsafe extern "C" fn dealloc(ptr: *mut u8, len: usize) {
    // SAFETY: by the contract above this is the original boxed slice.
    drop(unsafe { Box::from_raw(std::ptr::slice_from_raw_parts_mut(ptr, len)) });
}

/// FNV-1a of `len` bytes at `ptr`.
///
/// # Safety
/// `ptr..ptr+len` must be readable memory inside this module.
#[unsafe(no_mangle)]
pub unsafe extern "C" fn fnv1a(ptr: *const u8, len: usize) -> u64 {
    // SAFETY: guaranteed by the caller.
    fnv1a_64(unsafe { std::slice::from_raw_parts(ptr, len) })
}

/// Slugifies the UTF-8 text at `ptr..ptr+len`.
///
/// Returns [`pack`]`(ptr, len)` of a new buffer the caller must free with
/// [`dealloc`], or [`NONE`] if the input is not valid UTF-8 or the slug is
/// empty.
///
/// # Safety
/// `ptr..ptr+len` must be readable memory inside this module.
#[unsafe(no_mangle)]
pub unsafe extern "C" fn slugify(ptr: *const u8, len: usize) -> u64 {
    // SAFETY: guaranteed by the caller.
    let bytes = unsafe { std::slice::from_raw_parts(ptr, len) };
    let Ok(text) = std::str::from_utf8(bytes) else {
        return NONE;
    };
    let slug = slugify_str(text);
    if slug.is_empty() {
        return NONE;
    }
    let boxed = slug.into_bytes().into_boxed_slice();
    let n = boxed.len() as u32;
    let p = Box::into_raw(boxed).cast::<u8>();
    pack(p as usize as u32, n)
}

/// Writes the decimal text of `n` (via `format!`) and returns it like
/// [`slugify`] does. Exists only to measure the size cost of formatting.
#[cfg(feature = "fmt")]
#[unsafe(no_mangle)]
pub extern "C" fn describe(n: u64) -> u64 {
    let boxed = format!("value={n} hex={n:#x}")
        .into_bytes()
        .into_boxed_slice();
    let len = boxed.len() as u32;
    pack(Box::into_raw(boxed).cast::<u8>() as usize as u32, len)
}

#[cfg(all(feature = "hostlog", target_arch = "wasm32"))]
#[link(wasm_import_module = "env")]
unsafe extern "C" {
    /// Provided by the host: receives a UTF-8 message.
    fn host_log(ptr: *const u8, len: usize);
}

/// Calls the host's `env.host_log` with `"hello, <name>"`.
///
/// # Safety
/// `ptr..ptr+len` must be readable UTF-8 memory inside this module.
#[cfg(all(feature = "hostlog", target_arch = "wasm32"))]
#[unsafe(no_mangle)]
pub unsafe extern "C" fn greet(ptr: *const u8, len: usize) {
    // SAFETY: guaranteed by the caller.
    let name = String::from_utf8_lossy(unsafe { std::slice::from_raw_parts(ptr, len) });
    let msg = format!("hello, {name}");
    // SAFETY: the host promises to read `len` bytes at `ptr` during the call.
    unsafe { host_log(msg.as_ptr(), msg.len()) };
}
````

`tests/logic.rs`:

````rust
use wasmlab::{
    MAX_SIEVE, MAX_SLUG, NONE, fnv1a_64, pack, prime_count, sieve_allowed, slugify_str, unpack,
};

#[test]
fn fnv1a_matches_published_vectors() {
    // Reference values of the 64-bit FNV-1a algorithm.
    assert_eq!(fnv1a_64(b""), 0xcbf2_9ce4_8422_2325);
    assert_eq!(fnv1a_64(b"a"), 0xaf63_dc4c_8601_ec8c);
    assert_eq!(fnv1a_64(b"foobar"), 0x8594_4171_f739_67e8);
}

#[test]
fn pack_and_unpack_round_trip_at_the_edges() {
    for (p, l) in [
        (0, 0),
        (1, 0),
        (0, 1),
        (u32::MAX, u32::MAX),
        (65536, 12),
        (12, 65536),
    ] {
        assert_eq!(unpack(pack(p, l)), (p, l));
    }
    assert_eq!(pack(1, 2), 0x0000_0001_0000_0002);
    assert_eq!(unpack(NONE), (0, 0));
}

#[test]
fn slugify_table() {
    let cases = [
        ("Hello, World!", "hello-world"),
        ("  Rust & WebAssembly  ", "rust-webassembly"),
        ("already-a-slug", "already-a-slug"),
        ("Tabs\tand\nnewlines", "tabs-and-newlines"),
        ("A--B__C", "a-b-c"),
        ("ÁÉÍ café", "caf"),
        ("日本語", ""),
        ("", ""),
        ("---", ""),
        ("a", "a"),
        ("x1 y2", "x1-y2"),
        ("-lead", "lead"),
        ("trail-", "trail"),
    ];
    for (input, want) in cases {
        assert_eq!(slugify_str(input), want, "input {input:?}");
    }
}

#[test]
fn slug_length_is_capped_without_a_trailing_dash() {
    let long = "a".repeat(100);
    assert_eq!(slugify_str(&long).len(), MAX_SLUG);
    let words = "ab ".repeat(40);
    let s = slugify_str(&words);
    assert!(
        s.len() <= MAX_SLUG && !s.ends_with('-'),
        "{s:?} ({})",
        s.len()
    );
    // exactly at the cap
    assert_eq!(slugify_str(&"b".repeat(MAX_SLUG)).len(), MAX_SLUG);
    assert_eq!(slugify_str(&"b".repeat(MAX_SLUG - 1)).len(), MAX_SLUG - 1);
    // a dash that would land exactly on the cap is dropped
    let edge = format!("{} z", "c".repeat(MAX_SLUG - 1));
    assert_eq!(slugify_str(&edge), "c".repeat(MAX_SLUG - 1));
}

#[test]
fn prime_counts_match_known_values() {
    let known = [
        (0, 0),
        (1, 0),
        (2, 1),
        (3, 2),
        (4, 2),
        (10, 4),
        (100, 25),
        (1_000, 168),
        (1_000_000, 78_498),
    ];
    for (n, want) in known {
        assert_eq!(prime_count(n), want, "n = {n}");
    }
}

#[test]
fn oversized_sieve_requests_are_refused_not_attempted() {
    assert_eq!(prime_count(MAX_SIEVE + 1), u32::MAX);
    assert_eq!(prime_count(u32::MAX), u32::MAX);
}

#[test]
fn the_documented_limits_are_pinned_and_the_sieve_boundary_is_exact() {
    assert_eq!(MAX_SLUG, 64);
    assert_eq!(MAX_SIEVE, 200_000_000);
    assert!(sieve_allowed(0) && sieve_allowed(MAX_SIEVE));
    assert!(!sieve_allowed(MAX_SIEVE + 1) && !sieve_allowed(u32::MAX));
}
````

`tests/abi.rs`:

````rust
//! The exported functions exercised natively through raw pointers, exactly as
//! a host would drive them through linear memory.

use wasmlab::{NONE, add, alloc, dealloc, divide, fnv1a, primes, slugify, unpack};

fn put(bytes: &[u8]) -> (*mut u8, usize) {
    let p = alloc(bytes.len());
    // SAFETY: `alloc` returned `bytes.len()` writable bytes.
    unsafe { std::ptr::copy_nonoverlapping(bytes.as_ptr(), p, bytes.len()) };
    (p, bytes.len())
}

#[test]
fn numeric_exports() {
    assert_eq!(add(2, 3), 5);
    assert_eq!(add(i32::MAX, 1), i32::MIN, "wrapping, not a trap");
    assert_eq!(primes(100), 25);
    assert_eq!(divide(7, 2), 3);
    assert_eq!(divide(i32::MIN, -1), i32::MIN, "wrapping division");
}

#[test]
fn hash_through_memory() {
    let (p, n) = put(b"foobar");
    // SAFETY: p..p+n came from `alloc` and is initialized.
    let h = unsafe { fnv1a(p, n) };
    assert_eq!(h, 0x8594_4171_f739_67e8);
    // SAFETY: same block, freed once.
    unsafe { dealloc(p, n) };
}

#[test]
fn slugify_returns_a_buffer_the_caller_frees() {
    let (p, n) = put("Hello, Wörld 2026!".as_bytes());
    // SAFETY: the block is valid and initialized.
    let packed = unsafe { slugify(p, n) };
    assert_ne!(packed, NONE);
    let (rp, rl) = unpack(packed);
    // On a 64-bit host the pointer does not fit 32 bits, so this check only
    // makes sense on wasm32; natively we verify the length, which is exact.
    assert_eq!(rl as usize, "hello-w-rld-2026".len());
    let _ = rp;
    // SAFETY: input block freed once. (The native pointer was truncated by
    // `pack`, so the result block is deliberately not reconstructed here.)
    unsafe { dealloc(p, n) };
}

#[test]
fn invalid_utf8_and_empty_slugs_give_none() {
    let (p, n) = put(&[0xff, 0xfe, b'a']);
    // SAFETY: valid block.
    assert_eq!(unsafe { slugify(p, n) }, NONE);
    // SAFETY: freed once.
    unsafe { dealloc(p, n) };
    let (p, n) = put("!!!".as_bytes());
    // SAFETY: valid block.
    assert_eq!(unsafe { slugify(p, n) }, NONE);
    // SAFETY: freed once.
    unsafe { dealloc(p, n) };
    // SAFETY: a zero-length read through a dangling-but-aligned pointer is valid.
    assert_eq!(unsafe { slugify(alloc(0), 0) }, NONE);
}

#[test]
fn zero_length_alloc_is_usable() {
    let p = alloc(0);
    assert!(!p.is_null());
    // SAFETY: matches the alloc above.
    unsafe { dealloc(p, 0) };
}
````

`tests/leak.rs`:

````rust
//! A counting allocator proves that `alloc`/`dealloc` are balanced.
//!
//! The counter is per thread, so the test harness's own allocations on other
//! threads cannot disturb it.
use std::alloc::{GlobalAlloc, Layout, System};
use std::cell::Cell;

use wasmlab::{alloc, dealloc};

thread_local! {
    static LIVE: Cell<isize> = const { Cell::new(0) };
}

struct Counting;

// SAFETY: forwards every call to the system allocator unchanged.
unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        LIVE.with(|c| c.set(c.get() + layout.size() as isize));
        // SAFETY: same contract as the caller's.
        unsafe { System.alloc(layout) }
    }
    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        LIVE.with(|c| c.set(c.get() - layout.size() as isize));
        // SAFETY: same contract as the caller's.
        unsafe { System.dealloc(ptr, layout) }
    }
}

#[global_allocator]
static A: Counting = Counting;

#[test]
fn every_block_handed_out_by_alloc_is_returned_by_dealloc() {
    let before = LIVE.with(Cell::get);
    for len in [0usize, 1, 7, 100, 4096] {
        let p = alloc(len);
        // SAFETY: `p`/`len` come straight from `alloc` and are freed once.
        unsafe { dealloc(p, len) };
    }
    assert_eq!(LIVE.with(Cell::get), before);
    // and the counter really sees allocations
    let p = alloc(100);
    assert!(LIVE.with(Cell::get) >= before + 100);
    // SAFETY: freed once.
    unsafe { dealloc(p, 100) };
}
````

`tests/wasm_host.rs`:

````rust
//! Runs the *compiled* module inside the `wasmi` interpreter, as an embedding
//! host would. These tests are `#[ignore]`d because they need the `.wasm`
//! file; build it first, then run:
//!
//! ```text
//! cargo build --release --target wasm32-unknown-unknown
//! WASMLAB_WASM=target/wasm32-unknown-unknown/release/wasmlab.wasm \
//!     cargo test --test wasm_host -- --ignored
//! ```
use wasmi::{Engine, Instance, Linker, Memory, Module, Store, StoreLimits, StoreLimitsBuilder};

struct Host {
    limits: StoreLimits,
}

fn load(fuel: Option<u64>, max_memory: Option<usize>) -> (Store<Host>, Instance) {
    let path = std::env::var("WASMLAB_WASM").expect("set WASMLAB_WASM to the .wasm path");
    let bytes = std::fs::read(&path).unwrap_or_else(|e| panic!("{path}: {e}"));
    let mut config = wasmi::Config::default();
    config.consume_fuel(fuel.is_some());
    let engine = Engine::new(&config);
    let module = Module::new(&engine, &bytes[..]).expect("valid module");
    let mut limits = StoreLimitsBuilder::new();
    if let Some(m) = max_memory {
        limits = limits.memory_size(m);
    }
    let mut store = Store::new(
        &engine,
        Host {
            limits: limits.build(),
        },
    );
    store.limiter(|h| &mut h.limits);
    if let Some(f) = fuel {
        store.set_fuel(f).unwrap();
    }
    let instance = Linker::new(&engine)
        .instantiate_and_start(&mut store, &module)
        .expect("instantiate");
    (store, instance)
}

fn mem(store: &Store<Host>, i: &Instance) -> Memory {
    i.get_memory(store, "memory").expect("exported memory")
}

#[test]
#[ignore = "needs the compiled .wasm (see the module docs)"]
fn numeric_exports_work_in_the_interpreter() {
    let (mut s, i) = load(None, None);
    let add = i.get_typed_func::<(i32, i32), i32>(&s, "add").unwrap();
    assert_eq!(add.call(&mut s, (2, 3)).unwrap(), 5);
    assert_eq!(add.call(&mut s, (i32::MAX, 1)).unwrap(), i32::MIN);
    let primes = i.get_typed_func::<u32, u32>(&s, "primes").unwrap();
    // On wasm32 `usize` is 32 bits: an unchecked `i * i` in the sieve gave
    // 78 493 here instead of 78 498 before it was fixed.
    assert_eq!(primes.call(&mut s, 1_000_000).unwrap(), 78_498);
    assert_eq!(primes.call(&mut s, 200_000_001).unwrap(), u32::MAX);
}

#[test]
#[ignore = "needs the compiled .wasm (see the module docs)"]
fn strings_cross_the_boundary_through_linear_memory() {
    let (mut s, i) = load(None, None);
    let alloc = i.get_typed_func::<u32, u32>(&s, "alloc").unwrap();
    let dealloc = i.get_typed_func::<(u32, u32), ()>(&s, "dealloc").unwrap();
    let slugify = i.get_typed_func::<(u32, u32), u64>(&s, "slugify").unwrap();
    let fnv = i.get_typed_func::<(u32, u32), u64>(&s, "fnv1a").unwrap();
    let m = mem(&s, &i);

    let input = "Hello, Wörld 2026!".as_bytes();
    let p = alloc.call(&mut s, input.len() as u32).unwrap();
    m.write(&mut s, p as usize, input).unwrap();
    let packed = slugify.call(&mut s, (p, input.len() as u32)).unwrap();
    let (rp, rl) = wasmlab::unpack(packed);
    let mut out = vec![0u8; rl as usize];
    m.read(&s, rp as usize, &mut out).unwrap();
    assert_eq!(String::from_utf8(out).unwrap(), "hello-w-rld-2026");
    dealloc.call(&mut s, (rp, rl)).unwrap();
    assert_eq!(
        fnv.call(&mut s, (p, input.len() as u32)).unwrap(),
        wasmlab::fnv1a_64(input)
    );
    dealloc.call(&mut s, (p, input.len() as u32)).unwrap();

    // invalid UTF-8 and empty slugs report NONE
    let bad = [0xff, 0xfe, b'a'];
    let p = alloc.call(&mut s, 3).unwrap();
    m.write(&mut s, p as usize, &bad).unwrap();
    assert_eq!(slugify.call(&mut s, (p, 3)).unwrap(), wasmlab::NONE);
}

#[test]
#[ignore = "needs the compiled .wasm (see the module docs)"]
fn a_panic_is_a_trap_and_the_host_survives() {
    let (mut s, i) = load(None, None);
    let div = i.get_typed_func::<(i32, i32), i32>(&s, "divide").unwrap();
    assert_eq!(div.call(&mut s, (7, 2)).unwrap(), 3);
    let err = div.call(&mut s, (1, 0)).unwrap_err();
    assert!(err.to_string().contains("unreachable"), "{err}");
}

#[test]
#[ignore = "needs the compiled .wasm (see the module docs)"]
fn fuel_limits_how_much_work_a_guest_may_do() {
    let (mut s, i) = load(Some(1_000_000), None);
    let primes = i.get_typed_func::<u32, u32>(&s, "primes").unwrap();
    assert_eq!(primes.call(&mut s, 1_000).unwrap(), 168);
    s.set_fuel(1_000_000).unwrap();
    let err = primes.call(&mut s, 100_000_000).unwrap_err();
    assert!(err.to_string().to_lowercase().contains("fuel"), "{err}");
}

#[test]
#[ignore = "needs the compiled .wasm (see the module docs)"]
fn a_memory_cap_turns_a_large_request_into_a_trap() {
    let (mut s, i) = load(None, Some(8 * 1024 * 1024));
    let primes = i.get_typed_func::<u32, u32>(&s, "primes").unwrap();
    assert_eq!(primes.call(&mut s, 100_000).unwrap(), 9_592);
    assert!(primes.call(&mut s, 50_000_000).is_err());
}

#[test]
#[ignore = "needs the compiled .wasm (see the module docs)"]
fn a_module_that_imports_something_the_host_did_not_provide_fails_to_link() {
    let path = std::env::var("WASMLAB_HOSTLOG_WASM").expect("set WASMLAB_HOSTLOG_WASM");
    let engine = Engine::default();
    let module = Module::new(&engine, &std::fs::read(path).unwrap()[..]).unwrap();
    let mut store = Store::new(&engine, ());
    let err = Linker::<()>::new(&engine)
        .instantiate_and_start(&mut store, &module)
        .unwrap_err();
    assert!(err.to_string().contains("host_log"), "{err}");
}
````

`examples/bench.rs`:

````rust
//! Native timing of the same work the Node host times on WebAssembly.
use std::time::Instant;

use wasmlab::{fnv1a_64, prime_count};

fn main() {
    let mut t = Vec::new();
    let mut last = 0;
    for _ in 0..5 {
        let s = Instant::now();
        last = std::hint::black_box(prime_count(std::hint::black_box(10_000_000)));
        t.push(s.elapsed().as_secs_f64() * 1e3);
    }
    t.sort_by(|a, b| a.partial_cmp(b).unwrap());
    println!(
        "primes(10_000_000) = {last}; native ms (sorted): {}",
        t.iter()
            .map(|v| format!("{v:.1}"))
            .collect::<Vec<_>>()
            .join(" ")
    );
    let data = vec![0xabu8; 1 << 20];
    let mut t = Vec::new();
    for _ in 0..5 {
        let s = Instant::now();
        let mut h = 0;
        for _ in 0..20 {
            h ^= std::hint::black_box(fnv1a_64(std::hint::black_box(&data)));
        }
        std::hint::black_box(h);
        t.push(s.elapsed().as_secs_f64() * 1e3);
    }
    t.sort_by(|a, b| a.partial_cmp(b).unwrap());
    println!(
        "fnv1a 20 x 1 MiB native ms (sorted): {}",
        t.iter()
            .map(|v| format!("{v:.1}"))
            .collect::<Vec<_>>()
            .join(" ")
    );
}
````

`js/host.mjs`:

````js
// Drives wasmlab.wasm from Node.js. Usage: node js/host.mjs path/to/wasmlab.wasm
import { readFileSync } from "node:fs";

const bytes = readFileSync(process.argv[2]);
const mod = new WebAssembly.Module(bytes);
console.log("exports:", WebAssembly.Module.exports(mod).map((e) => `${e.name}:${e.kind}`).sort().join(" "));
console.log("imports:", JSON.stringify(WebAssembly.Module.imports(mod)));

const { exports: x } = new WebAssembly.Instance(mod, {});
const enc = new TextEncoder();
const dec = new TextDecoder();

// Always re-read x.memory.buffer: it is replaced when the memory grows.
const view = () => new Uint8Array(x.memory.buffer);

function withBytes(data, f) {
  const ptr = x.alloc(data.length);
  view().set(data, ptr);
  try {
    return f(ptr, data.length);
  } finally {
    x.dealloc(ptr, data.length);
  }
}
// i64 results arrive as *signed* BigInts: reinterpret as unsigned.
const u64 = (v) => BigInt.asUintN(64, v);
const fnv = (s) => u64(withBytes(enc.encode(s), (p, n) => x.fnv1a(p, n)));
function slug(s) {
  return withBytes(enc.encode(s), (p, n) => {
    const packed = u64(x.slugify(p, n)); // BigInt: (ptr << 32) | len
    if (packed === 0n) return null;
    const ptr = Number(packed >> 32n);
    const len = Number(packed & 0xffffffffn);
    const out = dec.decode(view().slice(ptr, ptr + len)); // copy out first
    x.dealloc(ptr, len);
    return out;
  });
}

console.log("add(2,3) =", x.add(2, 3), " add(2147483647,1) =", x.add(2147483647, 1));
console.log("typeof fnv1a result:", typeof fnv(""), " fnv(foobar) = 0x" + fnv("foobar").toString(16));
console.log("fnv('') ===", fnv("").toString(16));
console.log("slug('Hello, World!') =", JSON.stringify(slug("Hello, World!")));
console.log("slug('日本語') =", slug("日本語"));
console.log("slug(invalid utf-8) =", withBytes(new Uint8Array([0xff, 0xfe, 0x61]), (p, n) => x.slugify(p, n)));
console.log("primes(1000000) =", x.primes(1_000_000));

// A trap: the panic becomes `unreachable`, and the JS exception is a RuntimeError.
try {
  x.divide(1, 0);
} catch (e) {
  console.log("divide(1,0) throws:", e.constructor.name + ":", e.message);
}
// The module is still usable after a trap in a *pure* function, but any state
// the aborted call was in the middle of changing is undefined; re-instantiate in real code.
console.log("after trap, add(1,1) =", x.add(1, 1));

// Memory growth detaches old views.
const before = x.memory.buffer.byteLength;
const stale = view();
x.primes(50_000_000);
const after = x.memory.buffer.byteLength;
console.log(`memory: ${before / 65536} pages -> ${after / 65536} pages (${(after / 1048576).toFixed(0)} MiB)`);
console.log("stale view length after growth:", stale.length, " fresh view length:", view().length);

// Timing (one run each, this machine).
function time(f, n = 5) {
  const t = [];
  for (let i = 0; i < n; i++) {
    const s = process.hrtime.bigint();
    f();
    t.push(Number(process.hrtime.bigint() - s) / 1e6);
  }
  return t.sort((a, b) => a - b);
}
const t = time(() => x.primes(10_000_000));
console.log("primes(10_000_000) wasm in Node/V8, ms (sorted, 5 runs):", t.map((v) => v.toFixed(1)).join(" "));

// fnv1a over 20 x 1 MiB, copying the data in once.
const big = new Uint8Array(1 << 20).fill(0xab);
const ptr = x.alloc(big.length);
view().set(big, ptr);
const t2 = time(() => { for (let i = 0; i < 20; i++) x.fnv1a(ptr, big.length); });
console.log("fnv1a 20 x 1 MiB wasm in Node/V8, ms (sorted, 5 runs):", t2.map((v) => v.toFixed(1)).join(" "));
x.dealloc(ptr, big.length);
````

Building and running the wasm tests (on a normal machine, after `rustup target add wasm32-unknown-unknown`; I had to add `RUSTC_BOOTSTRAP=1` and `-Zbuild-std=std,panic_abort` for the reasons given at the top, which you should **not** need):

```text
cargo build --release --target wasm32-unknown-unknown
cargo build --release --target wasm32-unknown-unknown --features hostlog   # separate target dir if you want to keep both
WASMLAB_WASM=target/wasm32-unknown-unknown/release/wasmlab.wasm \
WASMLAB_HOSTLOG_WASM=<path to the hostlog build> \
    cargo test --test wasm_host -- --ignored
node js/host.mjs target/wasm32-unknown-unknown/release/wasmlab.wasm
```

The `wasm-bindgen` variant (`wblab`, section 6.9) is three functions with `#[wasm_bindgen(js_name = ...)]` over `wasmlab`'s pure functions; `wasm-bindgen = "=0.2.130"` and the CLI of the same version.

### Checks and mutation testing

✓ `cargo fmt`; `cargo clippy --all-targets -- -D warnings` clean for both crates; `shiplab`: 28 tests (units 17, service 11) and 12 consecutive full runs without a failure; `wasmlab`: 13 native tests (logic 7, ABI 5, leak 1), 6 `wasmi` tests that pass against the built module, and 5 consecutive native runs without a failure.

**Mutation check ✓.** I made 58 one-line changes (a "mutant"), 38 in `shiplab` and 20 in `wasmlab`, each in its own copy of the crate, and ran the whole test suite against each. **55 were caught, 3 are equivalent (M26, M50, M57), none survive.** Two mutants survived the *first* run and showed real gaps; I added a test for each and re-ran them: M15 (nothing pinned the limit for a head that has not ended yet) and M56 (a `dealloc` that leaks: nothing noticed until I added a counting allocator). Mutants that did not compile (one of mine, `7 + num(...)`, had a type error) were dropped, not counted. Two of the mutants (M52, M53) were first reported as "survived" by my script, because the test binary **aborted** (overflow inside an `extern "C"` function is a panic that cannot unwind, §6.5) instead of printing `FAILED`; I fixed the classifier to count an aborted test process as detection and re-ran them. M58 is the important one: it reintroduces the 32-bit overflow of §6.6, and it is caught **only** by the test that runs the compiled module. The tests ran in the dev profile.

| # | crate | change | result |
|---|---|---|---|
| M1 | `shiplab` | default port 8081 instead of 8080 | caught by `defaults_apply_when_nothing_is_set` |
| M2 | `shiplab` | default drain 5000 ms instead of 10000 | caught by `defaults_apply_when_nothing_is_set` |
| M3 | `shiplab` | default connection cap 32 instead of 64 | caught by `defaults_apply_when_nothing_is_set` |
| M4 | `shiplab` | default read timeout 3000 ms instead of 5000 | caught by `defaults_apply_when_nothing_is_set` |
| M5 | `shiplab` | `DRAIN_MS` upper bound 600001 | caught by `every_range_is_enforced_at_both_ends` |
| M6 | `shiplab` | `MAX_CONNS` lower bound 0 | caught by `every_range_is_enforced_at_both_ends` |
| M7 | `shiplab` | `READ_TIMEOUT_MS` lower bound 0 | caught by `every_range_is_enforced_at_both_ends` |
| M8 | `shiplab` | `MAX_CONNS` upper bound 10001 | caught by `every_range_is_enforced_at_both_ends` |
| M9 | `shiplab` | prefix `SHIPLAB` without the underscore | caught by `a_request_split_across_packets_is_reassembled` |
| M10 | `shiplab` | range check exclusive at the top (`lo..hi`) | caught by `every_range_is_enforced_at_both_ends` |
| M11 | `shiplab` | value not trimmed before parsing | caught by `every_range_is_enforced_at_both_ends` |
| M12 | `shiplab` | request line limit `>=` | caught by `the_request_line_limit_is_exact` |
| M13 | `shiplab` | unterminated line limit `>=` | caught by `the_request_line_limit_is_exact` |
| M14 | `shiplab` | head limit `>=` (terminated head) | caught by `the_head_limit_is_exact` |
| M15 | `shiplab` | head limit `>=` (unterminated head) | caught by `an_unterminated_head_is_only_refused_beyond_the_limit` (test added after the first run left H4 alive) |
| M16 | `shiplab` | lower-case methods accepted | caught by `malformed_request_lines_are_rejected` |
| M17 | `shiplab` | target without leading `/` accepted | caught by `malformed_request_lines_are_rejected` |
| M18 | `shiplab` | `HTTP/1.0` no longer accepted | caught by `a_complete_request_line_is_split_into_parts` |
| M19 | `shiplab` | a path without `?` gets the path as its query | caught by `a_complete_request_line_is_split_into_parts` |
| M20 | `shiplab` | query key matched by prefix | caught by `query_parameters_are_looked_up_by_exact_key` |
| M21 | `shiplab` | non-GET methods served | caught by `health_readiness_version_and_errors_over_real_http` |
| M22 | `shiplab` | `/readyz` stays 200 while draining | caught by `sigterm_lets_an_in_flight_request_finish_and_flips_readiness` |
| M23 | `shiplab` | `/healthz` returns 503 while draining | caught by `sigterm_lets_an_in_flight_request_finish_and_flips_readiness` |
| M24 | `shiplab` | `/slow` cap 50000 ms | caught by `slow_delay_is_parsed_defaulted_and_capped` |
| M25 | `shiplab` | connection cap off by one (`>`) | caught by `connections_beyond_the_limit_get_503_and_recover_afterwards` |
| M26 | `shiplab` | drain deadline `>` instead of `>=` | **equivalent**: with a zero deadline `elapsed() > 0` is true as soon as any time has passed |
| M27 | `shiplab` | drain timeout returns exit code 0 | caught by `a_zero_drain_deadline_gives_up_at_once_when_work_is_running` |
| M28 | `shiplab` | drain finishes with one request still in flight | caught by `a_stuck_connection_makes_the_drain_time_out_with_code_3` |
| M29 | `shiplab` | in-flight counter never decremented | caught by `connections_beyond_the_limit_get_503_and_recover_afterwards` |
| M30 | `shiplab` | read timeout not set | caught by `a_silent_client_is_dropped_after_the_read_timeout` |
| M31 | `shiplab` | draining flag never set | caught by `sigterm_lets_an_in_flight_request_finish_and_flips_readiness` |
| M32 | `shiplab` | `Content-Length` one too large | caught by `health_readiness_version_and_errors_over_real_http` |
| M33 | `shiplab` | reason phrase "Not Fond" | caught by `status_lines_carry_the_standard_reason_phrases_and_headers` |
| M34 | `shiplab` | `Connection: keep-alive` header while closing | caught by `status_lines_carry_the_standard_reason_phrases_and_headers` |
| M35 | `shiplab` | config error exits 1 instead of 2 | caught by `bad_configuration_exits_2_naming_the_variable_but_not_the_value` |
| M36 | `shiplab` | `SIGINT` handler not installed | caught by `an_idle_server_exits_zero_quickly_on_sigterm_and_on_sigint` |
| M37 | `shiplab` | `SIGTERM` handler not installed | caught by `a_stuck_connection_makes_the_drain_time_out_with_code_3` |
| M38 | `shiplab` | `DRAIN_TIMEOUT` exit code 5 | caught by `a_zero_drain_deadline_gives_up_at_once_when_work_is_running` |
| M39 | `wasmlab` | `pack` shifts by 16 | caught by `slugify_returns_a_buffer_the_caller_frees` |
| M40 | `wasmlab` | `unpack` masks 16 bits | caught by `pack_and_unpack_round_trip_at_the_edges` |
| M41 | `wasmlab` | FNV offset basis off by one | caught by `hash_through_memory` |
| M42 | `wasmlab` | FNV prime off by one | caught by `hash_through_memory` |
| M43 | `wasmlab` | FNV uses `|=` instead of `^=` | caught by `hash_through_memory` |
| M44 | `wasmlab` | slug not lower-cased | caught by `slugify_table` |
| M45 | `wasmlab` | leading dash allowed | caught by `slugify_table` |
| M46 | `wasmlab` | `MAX_SLUG` 63 | caught by `the_documented_limits_are_pinned_and_the_sieve_boundary_is_exact` |
| M47 | `wasmlab` | trailing-dash trim removed | caught by `slug_length_is_capped_without_a_trailing_dash` |
| M48 | `wasmlab` | sieve limit `<` (excludes the maximum) | caught by `the_documented_limits_are_pinned_and_the_sieve_boundary_is_exact` |
| M49 | `wasmlab` | `n < 3` early return | caught by `prime_counts_match_known_values` |
| M50 | `wasmlab` | sieve starts marking at `2 * i` instead of `i * i` | **equivalent**: starting at `2 * i` marks the same numbers (only slower) |
| M51 | `wasmlab` | counts primes twice | caught by `numeric_exports` |
| M52 | `wasmlab` | `add` uses `+` instead of `wrapping_add` | caught (the test process aborted: overflow check inside an `extern "C"` function) |
| M53 | `wasmlab` | `divide` uses `/` instead of `wrapping_div` | caught (the test process aborted: overflow inside an `extern "C"` function) |
| M54 | `wasmlab` | empty slug no longer returns `NONE` | caught by `invalid_utf8_and_empty_slugs_give_none` |
| M55 | `wasmlab` | invalid UTF-8 repaired with `from_utf8_lossy` | caught by `invalid_utf8_and_empty_slugs_give_none` |
| M56 | `wasmlab` | `dealloc` leaks the block | caught by `every_block_handed_out_by_alloc_is_returned_by_dealloc` (test added after the first run left W18 alive) |
| M57 | `wasmlab` | `divide` without the `assert!` | **equivalent in effect**: no native test calls `divide(_, 0)`; in wasm the division itself traps, so the host still sees an error either way (the compiled-module test checks for an error from `divide(1, 0)`) |
| M58 | `wasmlab` | sieve uses `wrapping_mul(i, i)` (the 32-bit bug) | caught `numeric_exports_work_in_the_interpreter` (compiled-module test only; **all 13 native tests passed**) |

What the table does not show: it tests *these* 58 edits, not every possible bug; the signal and drain tests rely on timing margins, so a heavily loaded machine could make them flaky (I saw no failure in 12 runs); and passing every mutant says nothing about security.

**Design notes.**

- `shiplab` keeps policy (ranges, limits, exit codes, route table) in small functions that tests call directly, and tests the process behavior (signals, ports, exit codes) by starting the real binary. Time-based behavior uses generous bounds (for example 120 ms to 3 s for a 150 ms timeout) so a loaded machine does not make tests flaky.
- Thread-per-connection with a connection cap is the simplest model that makes the drain logic visible. A production server would use an async runtime (Lesson 31) or an existing HTTP library (Lesson 45); the drain algorithm (flag, readiness flip, in-flight count, deadline) is the same.
- `wasmlab`'s ABI functions are thin: they validate, call a pure function, and convert. All behavior worth testing is in the pure functions, so native tests cover it; the ABI is covered where it executes.
- Not handled: TLS, keep-alive and pipelining, request bodies, structured (JSON) logs and metrics, a delay before closing the listener, multi-stage Docker builds, Kubernetes manifests, signing and provenance of artifacts, `wasm-opt`, wasm threads, the component model, running wasm in a browser, and exporting a smaller API surface from `wasmlab` when it is used as a dependency.

</details>

---

## 12. Key takeaways

- A release binary is a set of decisions: `strip` and LTO shrink it (✓ 5.9 MB debug to 349 to 371 KB), `panic = "abort"` changes behavior, and every choice should be measured (✓ `z` was not smaller than `s`).
- A static binary makes a tiny image possible (✓ 1.34 MB `FROM scratch`), run as a non-root user, read-only, with capabilities dropped; none of that replaces a correct application.
- Configure strictly, fail with distinct exit codes, and log to stderr; expose liveness and readiness as different things (✓).
- Shutdown is a protocol: a handler that sets a flag, readiness flipping first, in-flight work counted by a guard, a deadline, and a platform timeout that is longer than the deadline (✓ 28 tests; ✓ `docker stop` 1.14 s with exit 0, versus 3.09 s and 137 without a handler as PID 1).
- Reproducibility needs fixed inputs: even the location of `CARGO_HOME` changed the hash until paths were remapped (✓).
- WebAssembly is a different target with a numbers-only boundary: strings go through linear memory with an explicit ownership protocol, `i64` becomes a signed `BigInt`, memory views detach on growth, and `usize` is 32 bits (✓ every one reproduced).
- Test where the code runs: the sieve bug passed 13 native tests and failed only in the compiled module; the debug build trapped while the release build lied (✓).
- A sandbox constrains capabilities, not consumption: add fuel and memory limits, minimal imports and size caps (✓ `wasmi`), and do not mistake any of it for a guarantee.
- Mutation testing found real gaps again (✓ an unterminated-head boundary, a leak that no test noticed) and showed where native tests stop being enough.

**Connections.** [Lesson 46](46_databases_config_auth_jobs.md) supplied the configuration and secret-handling habits and the graceful pool shutdown that §3.5 generalizes to a whole service; [Lesson 45](45_http_apis.md) is the HTTP layer whose limits and timeouts `shiplab` repeats in miniature; [Lesson 44](44_cli_serialization.md) covers exit codes and version output; [Lesson 43](43_observability_ci_supply_chain.md) covers CI, logging and dependency policy for what is built here; [Lessons 41 and 42](42_performance_engineering.md) explain the profile settings and how to measure them; [Lessons 36 and 37](37_maybeuninit_ffi.md) explain the raw pointers and `extern "C"` that the wasm ABI uses; [Lesson 38](38_no_std_embedded_miri.md) is the other place where "no operating system" targets appear; [Lessons 27 and 28](27_threads_channels.md) explain the threads and atomics under the drain. This closes Module 14; Lesson 48 begins the cryptography and distributed-systems module with hashes, signatures and key management.

---

## 13. References

- The Cargo book: profiles (`opt-level`, `lto`, `codegen-units`, `panic`, `strip`), `--locked`, build scripts: <https://doc.rust-lang.org/cargo/reference/profiles.html>
- The rustc book: platform support, `-C target-feature=+crt-static`, `--remap-path-prefix`, targets `wasm32-unknown-unknown` and `wasm32-wasip1`: <https://doc.rust-lang.org/rustc/>
- The Rustonomicon and the Rust reference on `extern "C"` and unwinding: <https://doc.rust-lang.org/nomicon/>
- `signal-hook` documentation: <https://docs.rs/signal-hook>
- Docker documentation: `docker stop`, `--init`, `ENTRYPOINT` exec vs shell form, `--read-only`, `--cap-drop`: <https://docs.docker.com/>
- systemd: `systemd.service(5)`, `systemd.kill(5)`, `systemd-analyze(1)`: <https://www.freedesktop.org/software/systemd/man/>
- Kubernetes documentation: probes, termination of Pods: <https://kubernetes.io/docs/>
- The Twelve-Factor App (config, logs, disposability): <https://12factor.net/>
- WebAssembly specification and the MDN WebAssembly guide (`WebAssembly.Module`, `Memory`, `BigInt` and `i64`): <https://webassembly.github.io/spec/>, <https://developer.mozilla.org/docs/WebAssembly>
- The Rust and WebAssembly book, `wasm-bindgen` guide: <https://rustwasm.github.io/docs/book/>, <https://rustwasm.github.io/docs/wasm-bindgen/>
- WASI: <https://wasi.dev/>; Node.js `node:wasi` documentation: <https://nodejs.org/api/wasi.html>
- `wasmi` documentation: <https://docs.rs/wasmi>

The URLs above are the standard documentation locations; I did not open them during this session (the sandbox cannot reach the documentation sites), so if a link has moved, search for the page title. **Unchecked or from memory:** Cargo's own default value for `strip` in release builds; that `wasm-opt` commonly reduces module size further; that static glibc has limitations for name lookups and `dlopen`, and that `musl` is the common alternative; the Docker default stop timeout of 10 s; shell-form `ENTRYPOINT` and signal forwarding; that `scratch` images lack CA certificates and time-zone data; multi-stage Docker builds; the behavior of systemd directives (`Restart`, `RestartPreventExitStatus`, `TimeoutStopSec`, sandboxing options), since the unit was only syntax-checked; everything about Kubernetes probes, termination and rolling updates; what a signal handler may safely do; `cross` and `cargo-zigbuild`; what `wasm32-wasip2` and the component model are; that interpreters are slower than compiling runtimes; that Node's WASI is not a hardened security boundary; that unsafe memory bugs inside a wasm module can be exploitable within it; what `wasm-bindgen-test` does; zero-downtime rollout details in exercise 7; and CI practice. **Environment caveats:** the wasm32 standard library was built locally from source with unstable-feature flags because `rustup` could not download it, `wasi-libc` objects came from `wasi-sdk 27`, and Docker ran with a manually started daemon, `vfs` storage and host networking, so sizes and container timings may differ from yours. Measurements marked ✓ are single runs (or ranges of five runs) on one shared virtual machine; the timings of the two loops in §6.11 say nothing about other workloads. The code blocks in exercises 3, 4 and 5 and in the debugging challenge's original and fixed sieve were compiled and run (the exercise 5 module on `wasm32-unknown-unknown` in Node 22); the container, signal, drain, reproducibility, profile, wasm size, Node, `wasmi`, `wasm-bindgen` and WASI results marked ✓ were observed with `rustc 1.97.0`, `signal-hook 0.3.18`, `wasmi 0.51.5`, `wasm-bindgen-cli 0.2.130`, Node 22.22.0 and Docker 29.8.2.
{% endraw %}
