---
title: "Rust Course — Roadmap"
---

# Rust Course — Roadmap

**Goal:** genuine independent engineering ability in Rust — from absolute foundations to systems programming, concurrency, and blockchain infrastructure — not merely finishing a list of topics.

## Current position

| | |
|---|---|
| **Last generated lesson** | [Lesson 60 — Technical interviews and capstone projects](60_interviews_capstone.md) |
| **Next lesson** | None: all 60 lessons are generated |
| **Lessons generated** | 60 of 60 |
| **Course edition / toolchain used in examples** | Rust edition 2024, verified with `rustc 1.97.0 (2d8144b78 2026-07-07)` |

The course is complete; there is no further lesson to advance to.

## Status legend

- **Generated**: the lesson file exists.
- **Planned**: not yet written; the filename is the intended one and may be adjusted if the curriculum is expanded.
- **Exercises**: marked only when *you* say you completed them (or ask me to mark them). No exercise is marked complete yet.

## Curriculum

### Module 1 — Foundations and toolchain

| # | Lesson | File | Status | Exercises |
|---|---|---|---|---|
| 01 | Foundations: what Rust is, design philosophy, compiled and statically typed languages, memory safety, zero-cost abstractions, editions, stable vs nightly | [01_foundations.md](01_foundations.md) | Generated | Not completed |
| 02 | Toolchain: rustup, rustc, Cargo, Clippy, rustfmt, compiler options, rust-toolchain.toml | [02_toolchain.md](02_toolchain.md) | Generated | Not completed |
| 03 | Cargo projects: crates, modules, workspaces, dependencies, features, build profiles, CI basics | [03_cargo_projects.md](03_cargo_projects.md) | Generated | Not completed |

### Module 2 — Syntax and fundamentals

| # | Lesson | File | Status | Exercises |
|---|---|---|---|---|
| 04 | Variables, mutability, shadowing, primitives, numeric behavior, definite initialization | [04_variables_and_types.md](04_variables_and_types.md) | Generated | Not completed |
| 05 | Functions, expressions vs statements, control flow, tuples, arrays, slices | [05_functions_and_control_flow.md](05_functions_and_control_flow.md) | Generated | Not completed |
| 06 | Strings, text and bytes, conversions (`From`/`TryFrom`/`as`), pattern matching | [06_strings_conversions_patterns.md](06_strings_conversions_patterns.md) | Generated | Not completed |

### Module 3 — Ownership and memory

| # | Lesson | File | Status | Exercises |
|---|---|---|---|---|
| 07 | Ownership, moves, `Copy`, `Clone` | [07_ownership_and_moves.md](07_ownership_and_moves.md) | Generated | Not completed |
| 08 | Stack and heap, `Drop`, RAII, allocation and deallocation | [08_stack_heap_drop.md](08_stack_heap_drop.md) | Generated | Not completed |
| 09 | Memory layout: size, alignment, padding, niches, `repr` | [09_memory_layout.md](09_memory_layout.md) | Generated | Not completed |

### Module 4 — Borrowing and lifetimes

| # | Lesson | File | Status | Exercises |
|---|---|---|---|---|
| 10 | References, shared and mutable borrowing, aliasing, reborrowing | [10_references_and_borrowing.md](10_references_and_borrowing.md) | Generated | Not completed |
| 11 | Lifetime annotations, elision, lifetime bounds | [11_lifetimes.md](11_lifetimes.md) | Generated | Not completed |
| 12 | Variance, higher-ranked trait bounds, reading borrow-checker diagnostics, hard borrow problems | [12_variance_hrtb_borrowck.md](12_variance_hrtb_borrowck.md) | Generated | Not completed |

### Module 5 — Data modeling

| # | Lesson | File | Status | Exercises |
|---|---|---|---|---|
| 13 | Structs, enums, methods, associated functions | [13_structs_enums_methods.md](13_structs_enums_methods.md) | Generated | Not completed |
| 14 | `Option`, `Result`, the `?` operator, error propagation | [14_option_result_errors.md](14_option_result_errors.md) | Generated | Not completed |
| 15 | Newtypes, typestate, state machines | [15_newtype_typestate.md](15_newtype_typestate.md) | Generated | Not completed |

### Module 6 — Collections and functional programming

| # | Lesson | File | Status | Exercises |
|---|---|---|---|---|
| 16 | `Vec`, `String`, `HashMap`, `BTreeMap`, collection performance | [16_collections.md](16_collections.md) | Generated | Not completed |
| 17 | Iterators and adapters | [17_iterators.md](17_iterators.md) | Generated | Not completed |
| 18 | Closures, `Fn` / `FnMut` / `FnOnce` | [18_closures.md](18_closures.md) | Generated | Not completed |

### Module 7 — Generics and traits

| # | Lesson | File | Status | Exercises |
|---|---|---|---|---|
| 19 | Generics, trait bounds, monomorphization | [19_generics_and_bounds.md](19_generics_and_bounds.md) | Generated | Not completed |
| 20 | Associated types, blanket impls, coherence, orphan rules | [20_associated_types_coherence.md](20_associated_types_coherence.md) | Generated | Not completed |
| 21 | Trait objects, static vs dynamic dispatch, object safety | [21_trait_objects_dispatch.md](21_trait_objects_dispatch.md) | Generated | Not completed |
| 22 | GATs, `Send`, `Sync`, `PhantomData`, variance in practice | [22_gats_send_sync_phantom.md](22_gats_send_sync_phantom.md) | Generated | Not completed |

### Module 8 — Smart pointers

| # | Lesson | File | Status | Exercises |
|---|---|---|---|---|
| 23 | `Box`, `Rc`, `Arc` | [23_box_rc_arc.md](23_box_rc_arc.md) | Generated | Not completed |
| 24 | Interior mutability: `Cell`, `RefCell`, `Mutex`, `RwLock` | [24_interior_mutability.md](24_interior_mutability.md) | Generated | Not completed |
| 25 | `Weak`, reference cycles, `Pin` | [25_weak_pin.md](25_weak_pin.md) | Generated | Not completed |

### Module 9 — Error handling

| # | Lesson | File | Status | Exercises |
|---|---|---|---|---|
| 26 | Error design: panic vs `Result`, custom errors, sources, `anyhow`, `thiserror`, logging, recovery | [26_error_design.md](26_error_design.md) | Generated | Not completed |

### Module 10 — Concurrency and async

| # | Lesson | File | Status | Exercises |
|---|---|---|---|---|
| 27 | Threads, scoped threads, channels | [27_threads_channels.md](27_threads_channels.md) | Generated | Not completed |
| 28 | Atomics and memory ordering | [28_atomics_memory_ordering.md](28_atomics_memory_ordering.md) | Generated | Not completed |
| 29 | Locks, races, deadlocks | [29_locks_deadlocks.md](29_locks_deadlocks.md) | Generated | Not completed |
| 30 | Futures, `async`/`await`, executors | [30_futures_async.md](30_futures_async.md) | Generated | Not completed |
| 31 | Tokio in practice | [31_tokio.md](31_tokio.md) | Generated | Not completed |
| 32 | Cancellation, streams, backpressure | [32_cancellation_streams_backpressure.md](32_cancellation_streams_backpressure.md) | Generated | Not completed |

### Module 11 — Modules and metaprogramming

| # | Lesson | File | Status | Exercises |
|---|---|---|---|---|
| 33 | Modules, visibility, API design, rustdoc | [33_modules_api_design_rustdoc.md](33_modules_api_design_rustdoc.md) | Generated | Not completed |
| 34 | Declarative macros, hygiene | [34_declarative_macros.md](34_declarative_macros.md) | Generated | Not completed |
| 35 | Procedural macros, derives, attributes | [35_proc_macros.md](35_proc_macros.md) | Generated | Not completed |

### Module 12 — Unsafe Rust and systems programming

| # | Lesson | File | Status | Exercises |
|---|---|---|---|---|
| 36 | Unsafe contracts, raw pointers, undefined behavior | [36_unsafe_raw_pointers.md](36_unsafe_raw_pointers.md) | Generated | Not completed |
| 37 | `MaybeUninit`, FFI, `repr(C)`, C interoperability | [37_maybeuninit_ffi.md](37_maybeuninit_ffi.md) | Generated | Not completed |
| 38 | `no_std`, embedded basics, Miri | [38_no_std_embedded_miri.md](38_no_std_embedded_miri.md) | Generated | Not completed |

### Module 13 — Testing and performance

| # | Lesson | File | Status | Exercises |
|---|---|---|---|---|
| 39 | Unit and integration tests, doc tests | [39_testing.md](39_testing.md) | Generated | Not completed |
| 40 | Property testing and fuzzing | [40_property_testing_fuzzing.md](40_property_testing_fuzzing.md) | Generated | Not completed |
| 41 | Benchmarking and profiling | [41_benchmarking_profiling.md](41_benchmarking_profiling.md) | Generated | Not completed |
| 42 | Performance engineering: cache locality, SIMD, allocations, binary size | [42_performance_engineering.md](42_performance_engineering.md) | Generated | Not completed |
| 43 | Observability, CI, dependency security | [43_observability_ci_supply_chain.md](43_observability_ci_supply_chain.md) | Generated | Not completed |

### Module 14 — Application development

| # | Lesson | File | Status | Exercises |
|---|---|---|---|---|
| 44 | CLI tools and serialization | [44_cli_serialization.md](44_cli_serialization.md) | Generated | Not completed |
| 45 | HTTP clients and servers, APIs | [45_http_apis.md](45_http_apis.md) | Generated | Not completed |
| 46 | Databases, configuration, authentication, background jobs | [46_databases_config_auth_jobs.md](46_databases_config_auth_jobs.md) | Generated | Not completed |
| 47 | Deployment and WebAssembly | [47_deployment_wasm.md](47_deployment_wasm.md) | Generated | Not completed |

### Module 15 — Cryptography and distributed systems

| # | Lesson | File | Status | Exercises |
|---|---|---|---|---|
| 48 | Hashes, signatures, key management | [48_crypto_primitives.md](48_crypto_primitives.md) | Generated | Not completed |
| 49 | Merkle trees, transaction models, consensus, trust assumptions | [49_merkle_transactions_consensus.md](49_merkle_transactions_consensus.md) | Generated | Not completed |
| 50 | Networking and deterministic execution | [50_networking_determinism.md](50_networking_determinism.md) | Generated | Not completed |

### Module 16 — Blockchain and Web3 infrastructure (chain-agnostic)

| # | Lesson | File | Status | Exercises |
|---|---|---|---|---|
| 51 | RPC, nodes, indexers, transaction monitoring, serialization | [51_rpc_nodes_indexers.md](51_rpc_nodes_indexers.md) | Generated | Not completed |
| 52 | Off-chain services, cryptographic integrations, fault tolerance | [52_web3_reliability.md](52_web3_reliability.md) | Generated | Not completed |

### Module 17 — Solana

| # | Lesson | File | Status | Exercises |
|---|---|---|---|---|
| 53 | Solana execution model: accounts, programs, instructions, PDAs, CPI | [53_solana_model.md](53_solana_model.md) | Generated | Not completed |
| 54 | Anchor, token programs, account validation | [54_anchor.md](54_anchor.md) | Generated | Not completed |
| 55 | Program testing, deployment, upgrades, security auditing | [55_solana_security_testing_deploy.md](55_solana_security_testing_deploy.md) | Generated | Not completed |

### Module 18 — Other blockchain ecosystems

| # | Lesson | File | Status | Exercises |
|---|---|---|---|---|
| 56 | Substrate/Polkadot concepts and Rust clients | [56_substrate_polkadot.md](56_substrate_polkadot.md) | Generated | Not completed |
| 57 | EVM interaction from Rust | [57_evm_rust.md](57_evm_rust.md) | Generated | Not completed |
| 58 | Cross-chain infrastructure, relayers, indexers | [58_cross_chain_relayers.md](58_cross_chain_relayers.md) | Generated | Not completed |

### Module 19 — Professional mastery

| # | Lesson | File | Status | Exercises |
|---|---|---|---|---|
| 59 | Architecture, code review, system design, production reliability | [59_architecture_system_design.md](59_architecture_system_design.md) | Generated | Not completed |
| 60 | Technical interviews and capstone projects | [60_interviews_capstone.md](60_interviews_capstone.md) | Generated | Not completed |

## Coverage check against the requested areas

Every requested area is mapped: foundations (01), toolchain (02–03), syntax (04–06), ownership and memory (07–09), borrowing and lifetimes (10–12), data modeling (13–15), collections and functional programming (16–18), generics and traits (19–22), smart pointers (23–25), error handling (14, 26), concurrency and async (27–32), modules and metaprogramming (33–35), unsafe and systems (36–38), testing and performance (39–43), applications and WebAssembly (44–47), cryptography and distributed systems (48–50), blockchain infrastructure (51–52), Solana (53–55), other ecosystems (56–58), professional mastery (59–60).

## Conventions and notes

- **Core language first.** Lessons 01–47 teach Rust itself and general engineering. Blockchain lessons (51+) are kept separate so that framework or chain-specific behavior is never mistaken for core Rust, and so that no single chain's execution model is implied to apply to all chains.
- **Verification.** Code in each lesson is compiled and run before delivery, and the toolchain version is stated. Benchmark numbers are never invented.
- **Stable by default.** Experimental or nightly-only features are labelled explicitly.
- **Roadmap changes.** The curriculum may be expanded where needed; any change is recorded here.
- **Navigation.** Back to the [course index](index.md).
