---
layout: default
title: "Rust: from foundations to systems and blockchain infrastructure"
---

# Rust: from foundations to systems and blockchain infrastructure

A lesson-by-lesson course that teaches Rust from absolute fundamentals to professional mastery: the language and its type system, ownership and lifetimes, concurrency and async, unsafe and systems programming, performance, applications, cryptography, and blockchain infrastructure (Solana, Substrate/Polkadot-related ecosystems, EVM tooling).

The aim is independent engineering ability: understanding *why* the language works as it does, how the compiler reasons about your program, and how to design reliable production systems.

## How to use this course

1. Read the lessons in order. Each one states its prerequisites and builds on the previous ones.
2. Run every example yourself. Examples state the Rust edition (2024) and dependencies, and were verified with a real toolchain.
3. Do the exercises and the debugging challenge *before* opening the collapsed solutions.
4. Follow your progress in the [roadmap](00_roadmap.md).

## Start here

- [Course roadmap and lesson statuses](00_roadmap.md)
- [Lesson 01 — Foundations](01_foundations.md)
- [Lesson 02 — Toolchain](02_toolchain.md)
- [Lesson 03 — Cargo projects](03_cargo_projects.md)
- [Lesson 04 — Variables and types](04_variables_and_types.md)
- [Lesson 05 — Functions and control flow](05_functions_and_control_flow.md)
- [Lesson 06 — Strings, conversions, patterns](06_strings_conversions_patterns.md)
- [Lesson 07 — Ownership and moves](07_ownership_and_moves.md)
- [Lesson 08 — Stack, heap, and `Drop`](08_stack_heap_drop.md)
- [Lesson 09 — Memory layout](09_memory_layout.md)
- [Lesson 10 — References and borrowing](10_references_and_borrowing.md)
- [Lesson 11 — Lifetimes](11_lifetimes.md)
- [Lesson 12 — Variance, higher-ranked bounds, borrow-checker diagnostics](12_variance_hrtb_borrowck.md)
- [Lesson 13 — Structs, enums, methods](13_structs_enums_methods.md)
- [Lesson 14 — Option, Result, the ? operator, error propagation](14_option_result_errors.md)
- [Lesson 15 — Newtypes, typestate, and state machines](15_newtype_typestate.md)
- [Lesson 16 — Collections: Vec, VecDeque, HashMap, BTreeMap, and friends](16_collections.md)
- [Lesson 17 — Iterators and adapters](17_iterators.md)
- [Lesson 18 — Closures: Fn, FnMut, FnOnce](18_closures.md)
- [Lesson 19 — Generics, trait bounds, monomorphization](19_generics_and_bounds.md)
- [Lesson 20 — Associated types, blanket impls, coherence, orphan rules](20_associated_types_coherence.md)
- [Lesson 21 — Trait objects, static vs dynamic dispatch, dyn compatibility](21_trait_objects_dispatch.md)
- [Lesson 22 — GATs, Send, Sync, PhantomData, variance in practice](22_gats_send_sync_phantom.md)
- [Lesson 23 — Box, Rc, and Arc: owning pointers](23_box_rc_arc.md)
- [Lesson 24 — Interior mutability: Cell, RefCell, Mutex, RwLock](24_interior_mutability.md)
- [Lesson 25 — Weak, reference cycles, and Pin](25_weak_pin.md)
- [Lesson 26 — Error design: panic vs Result, custom errors, sources, anyhow, thiserror](26_error_design.md)
- [Lesson 27 — Threads, scoped threads, and channels](27_threads_channels.md)
- [Lesson 28 — Atomics and memory ordering](28_atomics_memory_ordering.md)
- [Lesson 29 — Locks, races, deadlocks](29_locks_deadlocks.md)
- [Lesson 30 — Futures, async/await, and executors](30_futures_async.md)
- [Lesson 31 — Tokio in practice](31_tokio.md)
- [Lesson 32 — Cancellation, streams, and backpressure](32_cancellation_streams_backpressure.md)
- [Lesson 33 — Modules, visibility, API design, and rustdoc](33_modules_api_design_rustdoc.md)
- [Lesson 34 — Declarative macros and hygiene](34_declarative_macros.md)
- [Lesson 35 — Procedural macros: derives, attributes, and function-like macros](35_proc_macros.md)
- [Lesson 36 — Unsafe contracts, raw pointers, and undefined behavior](36_unsafe_raw_pointers.md)
- [Lesson 37 — `MaybeUninit`, FFI, `repr(C)`, C interoperability](37_maybeuninit_ffi.md)
- [Lesson 38 — `no_std`, embedded basics, Miri](38_no_std_embedded_miri.md)
- [Lesson 39 — Unit tests, integration tests, and doc tests](39_testing.md)
- [Lesson 40 — Property testing and fuzzing](40_property_testing_fuzzing.md)
- [Lesson 41 — Benchmarking and profiling](41_benchmarking_profiling.md)
- [Lesson 42 — Performance engineering: cache locality, SIMD, allocations, binary size](42_performance_engineering.md)
- [Lesson 43 — Observability, CI, dependency security](43_observability_ci_supply_chain.md)
- [Lesson 44 — CLI tools and serialization](44_cli_serialization.md)
- [Lesson 45 — HTTP clients and servers, APIs](45_http_apis.md)
- [Lesson 46 — Databases, configuration, authentication, background jobs](46_databases_config_auth_jobs.md)
- [Lesson 47 — Deployment and WebAssembly](47_deployment_wasm.md)
- [Lesson 48 — Hashes, signatures, key management](48_crypto_primitives.md)
- [Lesson 49 — Merkle trees, transaction models, consensus, trust assumptions](49_merkle_transactions_consensus.md)
- [Lesson 50 — Networking and deterministic execution](50_networking_determinism.md)
- [Lesson 51 — RPC, nodes, indexers, transaction monitoring, serialization](51_rpc_nodes_indexers.md)
- [Lesson 52 — Off-chain services, cryptographic integrations, fault tolerance](52_web3_reliability.md)
- [Lesson 53 — Solana execution model: accounts, programs, instructions, PDAs, CPI](53_solana_model.md)
- [Lesson 54 — Anchor, token programs, account validation](54_anchor.md)
- [Lesson 55 — Program testing, deployment, upgrades, security auditing](55_solana_security_testing_deploy.md)
- [Lesson 56 — Substrate/Polkadot concepts and Rust clients](56_substrate_polkadot.md)
- [Lesson 57 — EVM interaction from Rust](57_evm_rust.md)
- [Lesson 58 — Cross-chain infrastructure, relayers and indexers](58_cross_chain_relayers.md)
- [Lesson 59 — Architecture, code review, system design, production reliability](59_architecture_system_design.md)
- [Lesson 60 — Technical interviews and capstone projects](60_interviews_capstone.md)

## Lessons available

| # | Lesson |
|---|---|
| 01 | [Foundations: what Rust is and why it works this way](01_foundations.md) |
| 02 | [Toolchain: rustup, rustc, Cargo, Clippy, rustfmt](02_toolchain.md) |
| 03 | [Cargo projects: crates, modules, dependencies, features, workspaces, profiles, CI](03_cargo_projects.md) |
| 04 | [Variables, mutability, shadowing, primitives, and numeric behavior](04_variables_and_types.md) |
| 05 | [Functions, expressions, control flow, tuples, arrays, and slices](05_functions_and_control_flow.md) |
| 06 | [Strings, text and bytes, conversions, and pattern matching](06_strings_conversions_patterns.md) |
| 07 | [Ownership, moves, `Copy`, and `Clone`](07_ownership_and_moves.md) |
| 08 | [Stack and heap, `Drop`, RAII, allocation and deallocation](08_stack_heap_drop.md) |
| 09 | [Memory layout: size, alignment, padding, niches, and `repr`](09_memory_layout.md) |
| 10 | [References and borrowing: shared, mutable, aliasing, reborrowing](10_references_and_borrowing.md) |
| 11 | [Lifetimes: annotations, elision, structs that borrow, and `'static`](11_lifetimes.md) |
| 12 | [Variance, higher-ranked bounds, reading borrow-checker diagnostics, hard borrow problems](12_variance_hrtb_borrowck.md) |
| 13 | [Structs, enums, methods, and associated functions](13_structs_enums_methods.md) |
| 14 | [Option, Result, the `?` operator, and error propagation](14_option_result_errors.md) |
| 15 | [Newtypes, typestate, and state machines](15_newtype_typestate.md) |
| 16 | [Collections: Vec, VecDeque, HashMap, BTreeMap, and friends](16_collections.md) |
| 17 | [Iterators and adapters](17_iterators.md) |
| 18 | [Closures: Fn, FnMut, FnOnce](18_closures.md) |
| 19 | [Generics, trait bounds, monomorphization](19_generics_and_bounds.md) |
| 20 | [Associated types, blanket impls, coherence, orphan rules](20_associated_types_coherence.md) |
| 21 | [Trait objects, static vs dynamic dispatch, dyn compatibility](21_trait_objects_dispatch.md) |
| 22 | [GATs, Send, Sync, PhantomData, variance in practice](22_gats_send_sync_phantom.md) |
| 23 | [Box, Rc, and Arc: owning pointers](23_box_rc_arc.md) |
| 24 | [Interior mutability: Cell, RefCell, Mutex, RwLock](24_interior_mutability.md) |
| 25 | [Weak, reference cycles, and Pin](25_weak_pin.md) |
| 26 | [Error design: panic vs Result, custom errors, sources, anyhow, thiserror](26_error_design.md) |
| 27 | [Threads, scoped threads, and channels](27_threads_channels.md) |
| 28 | [Atomics and memory ordering](28_atomics_memory_ordering.md) |
| 29 | [Locks, races, deadlocks](29_locks_deadlocks.md) |
| 30 | [Futures, async/await, and executors](30_futures_async.md) |
| 31 | [Tokio in practice](31_tokio.md) |
| 32 | [Cancellation, streams, and backpressure](32_cancellation_streams_backpressure.md) |
| 33 | [Modules, visibility, API design, and rustdoc](33_modules_api_design_rustdoc.md) |
| 34 | [Declarative macros and hygiene](34_declarative_macros.md) |
| 35 | [Procedural macros: derives, attributes, and function-like macros](35_proc_macros.md) |
| 36 | [Unsafe contracts, raw pointers, and undefined behavior](36_unsafe_raw_pointers.md) |
| 37 | [`MaybeUninit`, FFI, `repr(C)`, C interoperability](37_maybeuninit_ffi.md) |
| 38 | [`no_std`, embedded basics, Miri](38_no_std_embedded_miri.md) |
| 39 | [Unit tests, integration tests, and doc tests](39_testing.md) |
| 40 | [Property testing and fuzzing](40_property_testing_fuzzing.md) |
| 41 | [Benchmarking and profiling](41_benchmarking_profiling.md) |
| 42 | [Performance engineering: cache locality, SIMD, allocations, binary size](42_performance_engineering.md) |
| 43 | [Observability, CI, dependency security](43_observability_ci_supply_chain.md) |
| 44 | [CLI tools and serialization](44_cli_serialization.md) |
| 45 | [HTTP clients and servers, APIs](45_http_apis.md) |
| 46 | [Databases, configuration, authentication, background jobs](46_databases_config_auth_jobs.md) |
| 47 | [Deployment and WebAssembly](47_deployment_wasm.md) |
| 48 | [Hashes, signatures, key management](48_crypto_primitives.md) |
| 49 | [Merkle trees, transaction models, consensus, trust assumptions](49_merkle_transactions_consensus.md) |
| 50 | [Networking and deterministic execution](50_networking_determinism.md) |
| 51 | [RPC, nodes, indexers, transaction monitoring, serialization](51_rpc_nodes_indexers.md) |
| 52 | [Off-chain services, cryptographic integrations, fault tolerance](52_web3_reliability.md) |
| 53 | [Solana execution model: accounts, programs, instructions, PDAs, CPI](53_solana_model.md) |
| 54 | [Anchor, token programs, account validation](54_anchor.md) |
| 55 | [Program testing, deployment, upgrades, security auditing](55_solana_security_testing_deploy.md) |
| 56 | [Substrate/Polkadot concepts and Rust clients](56_substrate_polkadot.md) |
| 57 | [EVM interaction from Rust](57_evm_rust.md) |
| 58 | [Cross-chain infrastructure, relayers, indexers](58_cross_chain_relayers.md) |
| 59 | [Architecture, code review, system design, production reliability](59_architecture_system_design.md) |
| 60 | [Technical interviews and capstone projects](60_interviews_capstone.md) |

## Module overview

| Module | Lessons | Topics |
|---|---|---|
| 1. Foundations and toolchain | 01–03 | Design philosophy, memory safety, editions; rustup, Cargo, Clippy, rustfmt; workspaces, features, profiles |
| 2. Syntax and fundamentals | 04–06 | Variables, numerics, functions, control flow, slices, strings, conversions, patterns |
| 3. Ownership and memory | 07–09 | Moves, `Copy`/`Clone`, stack and heap, `Drop`, RAII, memory layout |
| 4. Borrowing and lifetimes | 10–12 | References, aliasing, lifetimes, variance, higher-ranked bounds, borrow-checker diagnostics |
| 5. Data modeling | 13–15 | Structs, enums, `Option`/`Result`, newtypes, typestate |
| 6. Collections and functional style | 16–18 | `Vec`, `HashMap`, `BTreeMap`, iterators, closures |
| 7. Generics and traits | 19–22 | Bounds, associated types, coherence, trait objects, GATs, `Send`/`Sync`, `PhantomData` |
| 8. Smart pointers | 23–25 | `Box`, `Rc`, `Arc`, `Cell`, `RefCell`, `Mutex`, `Weak`, `Pin` |
| 9. Error handling | 26 | Panics vs errors, custom errors, `anyhow`, `thiserror` |
| 10. Concurrency and async | 27–32 | Threads, atomics, locks, futures, Tokio, cancellation, backpressure |
| 11. Modules and metaprogramming | 33–35 | API design, rustdoc, declarative and procedural macros |
| 12. Unsafe and systems | 36–38 | Raw pointers, UB, `MaybeUninit`, FFI, `no_std`, Miri |
| 13. Testing and performance | 39–43 | Tests, fuzzing, benchmarking, profiling, SIMD, observability, supply-chain security |
| 14. Application development | 44–47 | CLIs, serialization, HTTP, databases, deployment, WebAssembly |
| 15. Cryptography and distributed systems | 48–50 | Hashes, signatures, Merkle trees, consensus, networking, determinism |
| 16. Blockchain infrastructure | 51–52 | RPC, nodes, indexers, monitoring, fault tolerance |
| 17. Solana | 53–55 | Accounts, programs, PDAs, CPI, Anchor, testing, upgrades, auditing |
| 18. Other ecosystems | 56–58 | Substrate/Polkadot, EVM from Rust, relayers and cross-chain |
| 19. Professional mastery | 59–60 | Architecture, review, system design, interviews, capstones |

Lessons are linked from this page as they are published; the roadmap lists planned file names for the rest.
