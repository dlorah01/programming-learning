---
layout: default
title: "Lesson 40 — Property testing and fuzzing"
---

# Lesson 40 — Property testing and fuzzing

**Module 13 · Testing, performance and tooling** · Status: generated · Edition used in examples: **2024** · External dependencies: `proptest` (dev-dependency); `libfuzzer-sys` (in the separate `fuzz/` crate)

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, `proptest 1.12.0`, `cargo-fuzz 0.13.2` and `libfuzzer-sys 0.4.13`, on x86-64 Linux. As before, ✓ marks outputs, diagnostics and measurements I actually ran; unmarked statements are documented behavior I did not run. One caveat about the fuzzing runs: `cargo fuzz` normally requires the **nightly** toolchain, and nightly is not installed here, so I ran it on the stable compiler with the environment variable `RUSTC_BOOTSTRAP=1`, an **unsupported** trick that lets stable accept nightly-only flags. The results are real libFuzzer runs, but do not copy that setting into a project; use `rustup toolchain install nightly` and `cargo +nightly fuzz run`. Not installed or not run: Miri (Lesson 38), `cargo mutants`, OSS-Fuzz, AFL, `honggfuzz`, `quickcheck`, `arbitrary`-based structure-aware fuzzing (described from memory only). I could not open the documentation sites, so rules stated without ✓ are from memory; the closing paragraph lists them.

Builds on [Lesson 39](39_testing.md) (test harness, `proptest` basics, mutation checks), [Lesson 26](26_error_design.md) (error enums that decoders return), [Lesson 36](36_unsafe_raw_pointers.md) and [Lesson 37](37_maybeuninit_ffi.md) (why unsafe code needs more than example tests), [Lesson 38](38_no_std_embedded_miri.md) (the frame decoder and the "no input may panic" rule), and [Lesson 16](16_collections.md) (the `HashMap` and `VecDeque` inside the cache). Lesson 41 (benchmarking and profiling) is next; it appears here only as plain text.

---

## 1. Why this lesson exists

An example test says "for *this* input the output is *that*". The bugs that hurt are in the inputs nobody wrote down: the 11-byte message, the key that collides, the sequence of operations nobody tried. Two techniques generate those inputs for you:

- **Property-based testing** states a rule ("decoding what you encoded gives back the original") and checks it on many generated inputs; when it fails it **shrinks** the input to a small counterexample. Fast (milliseconds), runs with `cargo test`, ideal for logic.
- **Fuzzing** feeds a program a huge number of mutated byte strings, guided by **code coverage**: inputs that reach new code are kept and mutated further. Slow (minutes to days), runs as a separate process, ideal for parsers, decoders, and unsafe code that handle untrusted bytes.

This lesson's running example is a crate (`proplab`) of components that face hostile input on purpose: a variable-length integer decoder, a run-length decoder that must not be turned into a memory bomb, and a cache. I planted a bug in a copy of the decoder and checked which technique finds it, and how fast. The surprising answer (§5) is the main lesson: **a property test is only as good as its generator**.

---

## 2. Three ways to generate inputs

| | Example tests | Property tests | Fuzzing |
|---|---|---|---|
| Inputs come from | you | a random generator you describe (a *strategy*) | mutation of a corpus, steered by coverage |
| Checks | exact expected output | a rule over all inputs | "does not crash", plus any assertions you put in the target |
| Typical run | microseconds | 256 cases by default in about 0.1 s | millions of executions per minute (✓ 9.3 million in 91 s on this machine) |
| Failure report | the failing assertion | a **minimal** counterexample (shrinking) | a crashing input file; can be minimized with `tmin` |
| Strength | documents the format, pins edge cases | finds logic bugs and boundary errors | finds crashes, panics, hangs, memory errors on structured-but-hostile bytes |
| Weakness | only what you thought of | blind to narrow input shapes unless the strategy is biased (§5) | needs a harness; weak at deep semantic checks unless you assert them |

They are complementary. A good pipeline: examples for the specification, properties for the rules, a fuzz target for every untrusted-input entry point, and the crash files fuzzers produce turned into permanent example tests.

---

## 3. Finding properties

"Write a property" is the hard part. These shapes cover most real cases (the practical crate uses all of them ✓):

1. **Round trip.** `decode(encode(x)) == x`. Finds asymmetries (padding, off-by-one lengths, endianness).
2. **Totality.** `parse(arbitrary_bytes)` returns `Ok` or `Err` and never panics, hangs, or allocates without bound.
3. **Canonical form.** If `decode(bytes)` succeeds, `encode(decode(bytes))` equals the bytes that were consumed. Finds decoders that accept several spellings of one value (a classic source of signature-malleability and parser-differential bugs).
4. **Oracle / model.** Compare to a slow, obviously-correct implementation (§6) or to the standard library.
5. **Invariants.** After any sequence of operations, a rule still holds (`len <= capacity`).
6. **Idempotence and commutativity.** `normalize(normalize(x)) == normalize(x)`; the order of independent operations does not matter.
7. **Metamorphic.** Transform the input in a known way and the output must change in a known way (`sort(reverse(v)) == sort(v)`).
8. **Limits.** The output never exceeds the caller's limit, however hostile the input (the RLE decoder ✓).

**A rule of thumb for the *wrong* property:** a property that the broken code could satisfy trivially. "The output of `sort` is sorted" is satisfied by a `sort` that returns an empty vector. Pair it with "and is a permutation of the input" (exercise 4).

---

## 4. `proptest` in practice

You have seen the basics in Lesson 39. The parts that matter for writing good generators:

**Strategies** describe how to make a value *and* how to shrink it. Building blocks ✓ (all used in `tests/props.rs`):

| Strategy | Meaning |
|---|---|
| `any::<T>()` | a value of any type that implements `Arbitrary`, drawn broadly |
| `0u8..4`, `1..=10` | a value in a range (shrinks toward the low end) |
| `Just(x)` | always `x` |
| `proptest::collection::vec(s, 0..20)` | a `Vec` of elements from `s`, with a length in the range |
| `proptest::option::of(s)` | `None` or `Some` from `s` |
| `s.prop_map(f)` | transform generated values (shrinking still works on the source) |
| `prop_oneof![3 => a, 1 => b]` | pick a strategy by weight |
| `prop_compose!` | name a strategy built from other strategies |
| `prop_assume!(cond)` | discard this case if `cond` is false (do not overuse; too many discards fail the test) |

**Assertions.** Inside `proptest!` use `prop_assert!`/`prop_assert_eq!` so a failure is reported through the framework and the input can be shrunk. A plain `assert!` also fails the case (it panics, and proptest catches it), but `prop_assert!` is the conventional spelling.

**Counts and reproducibility.** The default is 256 cases per property (from memory); `PROPTEST_CASES=100000` raises it ✓ (a 100,000-case run took about 2.3 s on the decoder property in §5). Failing cases are saved to `proptest-regressions/` or `<test file>.proptest-regressions` and **replayed first** on the next run, so a failure that appeared once is deterministic afterwards ✓ (the saved line for the example in §5 was `shrinks to v = [3, 3]`). Commit that file.

**Shrinking** ✓. When a case fails, proptest repeatedly simplifies it (shorter vectors, smaller numbers) while it still fails, and reports the minimal input. That is what turns "a 19-element vector of random bytes" into `[3, 3]`.

**Cost control.** Properties run in debug builds by default, 256 times. Keep each case fast, and put the really heavy searches (huge case counts) behind an environment variable or `#[ignore]` so `cargo test` stays quick.

---

## 5. The generator decides what you find

### 5.1 A narrow bug that uniform data misses ✓

```rust
// needs proptest
use proptest::prelude::*;

// A buggy function: sorts, but removes duplicates only when len > 3.
fn buggy_dedup(mut v: Vec<u8>) -> Vec<u8> {
    v.sort();
    if v.len() > 3 {
        v.dedup();
    }
    v
}

fn no_dupes(v: &[u8]) -> bool {
    v.windows(2).all(|w| w[0] < w[1])
}

proptest! {
    #[test]
    fn dedup_has_no_duplicates(v in proptest::collection::vec(any::<u8>(), 0..20)) {
        prop_assert!(no_dupes(&buggy_dedup(v)));
    }
}
```

✓ **This test passed.** The bug is real: any vector of length 2 or 3 with a repeated element comes back with the duplicate. But with elements drawn from all 256 byte values, a short vector with a duplicate is rare (roughly one case in a few hundred), so 256 random cases usually miss it. Changing only the element strategy to a small alphabet, `vec(0u8..4, 0..20)`, made it fail immediately ✓ and the saved regression line read `shrinks to v = [3, 3]`.

The lesson is not "use small ranges". It is: **ask what input shape triggers the bug, and make sure the strategy produces that shape often.** Small alphabets create collisions; bounded sizes create boundaries; `prop_oneof!` mixes uniform values with `0`, `1`, `MAX`, powers of two (`interesting_u64()` in the practical crate).

### 5.2 A bug behind a chain of conditions ✓

I copied `proplab` and deleted one line from the varint decoder, the check that rejects an 11th byte (`if i == MAX_LEN { return Err(Overflow) }`). Without it, a very long run of continuation bytes reaches `low << (7 * i)` with `i >= 10`, and in a debug build that is `attempt to shift left with overflow` ✓ (a panic; in a release build it silently wraps, which is worse).

Results on that buggy copy ✓:

| Technique | Result |
|---|---|
| Example test `decode_u64(&[0x80; 11])` | **caught** (I wrote it knowing the bug; it is the kind of case you add *after* a fuzzer or review finds the class) |
| Property "decode of arbitrary bytes is total and canonical", bytes uniform in `0..=255`, 256 cases | **passed**, 6 runs out of 6 |
| The same property with `PROPTEST_CASES=100000` | **passed** (2.3 s) |
| Property "survives long continuation chains": bytes biased to `0x80..=0xFF` | **failed** in each of 3 runs; shrunk to eleven `128` bytes and `tail = None`, found after 216 successful cases |
| `cargo fuzz` (libFuzzer) with an assertion-free-except-panics target | **found a crashing input** within the 120 s budget; the last progress line before the crash was at execution 6,437 |

Why did 100,000 uniform cases miss it? I measured that only **21 of 100,000** uniform inputs have 11 or more leading bytes with the high bit set. And that is not enough: the byte at index 9 must also have its low seven bits equal to 0 or 1, otherwise the (still present) check `i == 9 && low > 1` rejects the input as `Overflow` before the bug is reached. That multiplies the odds by roughly 2/128. Expected hits in 100,000 cases: well under one. The failing shape (a chain of eleven continuation bytes whose tenth byte is `0x80` or `0x81`) is a path through the code, and uniform random bytes essentially never walk it.

The fuzzer found it because it **uses coverage as feedback**: an input that gets one byte further into the loop reaches new code, so it is kept in the corpus and mutated further; ten bytes later, the shift overflow is one mutation away. Random generation has no such memory.

The takeaways are practical:

1. For parsers and decoders, write both the uniform property *and* targeted generators (long chains of continuation bytes, boundary lengths, maximum values).
2. Run a fuzzer on every function that takes untrusted bytes. It finds paths your generators did not imagine.
3. When the fuzzer finds something, **add the minimized input as an example test** (the `[0x80; 11]` test) and add or bias a property so the class stays covered.

---

## 6. Model-based testing

For stateful code the input is a **sequence of operations**. The technique: generate a random sequence, apply it to the real implementation *and* to a deliberately simple model, and assert after every step that the visible behavior agrees.

The cache in the practical crate stores a `HashMap` plus a `VecDeque` of keys for recency; the model is a plain `Vec<(key, value)>` where position is recency and every operation is a linear scan: slow and obviously right. The property:

```rust,ignore
#[derive(Debug, Clone)]
enum Op {
    Put(u8, u16),
    Get(u8),
}

proptest! {
    #[test]
    fn lru_behaves_like_the_model(cap in 0usize..5, ops in ops()) {
        let mut real: Lru<u8, u16> = Lru::new(cap);
        let mut model = Model { cap, entries: Vec::new() };
        for op in ops {
            match op {
                Op::Put(k, v) => prop_assert_eq!(real.put(k, v), model.put(k, v)),
                Op::Get(k) => prop_assert_eq!(real.get(&k).copied(), model.get(k)),
            }
            prop_assert!(real.len() <= cap);
            let keys: Vec<u8> = model.entries.iter().map(|e| e.0).collect();
            prop_assert_eq!(real.keys_lru_to_mru(), keys);
        }
    }
}
```

Design choices that matter (all in the crate ✓): **few distinct keys** (`0..6`) so hits, updates and evictions all happen constantly; capacity includes **zero**; the generator mixes operations by weight; shrinking removes whole operations, so a failure is reported as the shortest sequence that still disagrees. All five cache mutants I tried (§14) were caught by this one property.

---

## 7. Fuzzing

### 7.1 How a coverage-guided fuzzer works

1. The code under test is compiled with **instrumentation** that records which branches executed.
2. The fuzzer keeps a **corpus** of inputs. It picks one, mutates it (flip a bit, insert a byte, copy a part, cross over with another input, insert a "magic" value it noticed in comparisons), and runs the target.
3. If the run reaches **new coverage**, the new input joins the corpus. If the process crashes (panic, abort, sanitizer error), hangs, or exceeds a memory limit, the input is saved as an artifact.
4. This repeats millions of times. The corpus grows and shrinks (it also *minimizes* redundant entries).

In Rust, `cargo fuzz` wraps **libFuzzer** (from LLVM), and by default builds with AddressSanitizer, so memory errors in `unsafe` code are crashes too. A panic is a crash because the fuzz harness aborts on it.

### 7.2 The layout ✓

`cargo fuzz init` creates a separate crate next to yours:

```text
fuzz/
  Cargo.toml            # depends on libfuzzer-sys and on your crate by path
  fuzz_targets/
    decode.rs           # one binary per target
  corpus/decode/        # created on first run: the interesting inputs found
  artifacts/decode/     # created when something crashes
```

```toml
[package]
name = "proplab-fuzz"
version = "0.0.0"
publish = false
edition = "2024"

[package.metadata]
cargo-fuzz = true

[dependencies]
libfuzzer-sys = "0.4"

[dependencies.proplab]
path = ".."

[[bin]]
name = "decode"
path = "fuzz_targets/decode.rs"
test = false
doc = false
bench = false
```

```rust
#![no_main]

use libfuzzer_sys::fuzz_target;
use proplab::{rle, varint};

fuzz_target!(|data: &[u8]| {
    // 1. The varint decoder must be total; when it accepts, it is canonical.
    if let Ok((n, used)) = varint::decode_u64(data) {
        let mut again = Vec::new();
        varint::encode_u64(n, &mut again);
        assert_eq!(&again[..], &data[..used]);
    }
    // 2. The RLE decoder must be total and honor its limit.
    if let Ok(out) = rle::decode(data, 1 << 16) {
        assert!(out.len() <= 1 << 16);
    }
});
```

The `fuzz_target!` closure receives arbitrary bytes. It must be **fast, deterministic and stateless** (no global state, no I/O, no randomness), and it should assert every property that is cheap to check, not only "no crash": here, that an accepted varint is canonical and that the RLE output respects its limit.

Commands (from the crate root):

```text
cargo +nightly fuzz run decode                       # run until a crash or you stop it
cargo +nightly fuzz run decode -- -max_total_time=60 # libFuzzer flag: stop after 60 s
cargo +nightly fuzz tmin decode fuzz/artifacts/decode/crash-...   # minimize a crash
cargo +nightly fuzz run decode fuzz/artifacts/decode/crash-...    # re-run one input
```

I ran the same commands without `+nightly`, with `RUSTC_BOOTSTRAP=1` (see the note at the top).

### 7.3 What the runs looked like ✓

**On the buggy copy** the fuzzer printed progress lines such as `#3 NEW cov: 51 ft: 51 corp: 2/3b ...` (`cov` counts covered code edges; `corp` is the corpus size), and then:

```text
thread '<unnamed>' panicked at .../src/varint.rs:64:19:
attempt to shift left with overflow
==3547== ERROR: libFuzzer: deadly signal
```

It saved the crashing input under `fuzz/artifacts/decode/`. `cargo fuzz tmin` shrank it to **11 bytes**:

```text
[246, 255, 255, 255, 255, 255, 255, 255, 238, 128, 214]
```

(The minimizer keeps the byte count and simplifies bytes; proptest's shrunk case was eleven `128`s. Different tools, same shape: eleven bytes with the high bit set until the end. The minimized input is not necessarily the "nicest".)

**On the correct crate** I ran the fuzzer for 90 seconds ✓: `Done 9345755 runs in 91 second(s)`, `cov: 110 ft: 303 corp: 116/3857b`, about 103,000 executions per second (one measurement on two cores; not a benchmark), no crash. A clean run proves nothing about absence of bugs; it only says this fuzzer did not find one in this time. Coverage plateaued early (`cov: 110` for most of the run), which suggests the target is small; with a larger parser you would watch `cov` and `corp` climb for much longer.

### 7.4 Good fuzz targets

- **Fuzz at the boundary of untrusted input**, not deep inside. Fuzz `decode(bytes)`, not `helper(i32)`.
- **Assert, don't just survive.** Round-trip, canonical-form and limit checks turn the fuzzer into a bug finder for *logic*, not just crashes.
- **Differential fuzzing** (from memory): run two implementations (yours vs a reference library, or optimized vs naive) on the same input and assert they agree. This is how many serious parser discrepancies are found.
- **Structure-aware fuzzing** with the `arbitrary` crate (from memory) lets the fuzzer's bytes be interpreted as a structured value (e.g. `Vec<Op>` for the cache), which is good when random bytes would almost never parse.
- **Seed the corpus** with valid examples (real files, protocol captures); mutation then explores near valid inputs. Dictionaries of keywords (from memory, a libFuzzer feature) help text formats.
- **Keep the corpus** in version control or a cache, so each CI run starts from the previous state, and turn fixed crashes into example tests.
- **Bound everything**: input length (`-max_len`), memory (`-rss_limit_mb`), time per execution (`-timeout`), so a hang or an allocation bomb is reported rather than stalling (flags from memory).

### 7.5 Resource limits are a security feature ✓

`rle::decode` takes `max_out` and checks `count > room` **before** allocating. The test `rle_a_tiny_hostile_input_is_refused_not_expanded` encodes "2^63 copies of `x`" in ten bytes and expects `OutputLimit`. Without the check, the decoder would try to allocate petabytes (or loop for years). The fuzz target uses a limit of 64 KiB, so even a bug here would show up as a rejected run, not a hung machine. Whenever a format contains a length or count field, apply a caller-chosen limit before you trust it.

---

## 8. Where each tool fits

| Question | Tool |
|---|---|
| "Does my round-trip / invariant hold broadly?" | property test (`proptest`) |
| "Does my stateful object behave like a simple model?" | model-based property test |
| "Can any byte string crash or hang this parser?" | fuzzing, plus a totality property for fast feedback |
| "Is there memory unsafety in my `unsafe` code?" | fuzzing under a sanitizer, Miri (Lesson 38), valgrind; plus drop-counting tests (Lesson 37) |
| "Is this concurrent code correct?" | schedule-exploring tools such as `loom` (from memory), not fuzzing |
| "Did I handle this one regression?" | an example test, always |

In CI (from memory): run property tests with the normal suite; run each fuzz target for a short bounded time per change (and for hours on a schedule), with the corpus cached; many open-source projects also enroll in continuous fuzzing services (OSS-Fuzz).

---

## 9. Performance, reliability, security

- **Speed.** My small decoder target ran at about 100,000 executions per second ✓; a target that does I/O or allocates large buffers per run is typically far slower (from memory) and finds correspondingly less. Keep targets lean.
- **Determinism.** Randomized tests must be reproducible: proptest persists failures, libFuzzer writes the crashing input to a file. Never use wall-clock seeds without printing them.
- **Reliability.** A property or fuzz failure is not "flaky": it is a real input. Convert it to an example test and fix the bug.
- **Security.** Fuzzing is a front-line defense for code that parses untrusted data (network protocols, file formats, transaction decoders in blockchain clients). Rust prevents memory unsafety in safe code but **not** panics, infinite loops, quadratic behavior, huge allocations, or logic errors such as accepting two encodings of one value. Those are exactly the bugs these tools find, and in a consensus system a decoder panic or a non-canonical encoding can be a network-wide failure. Fuzzing does not prove security.
- **Cost of false confidence.** A fuzzer run that finds nothing, or a green property suite, are statements about the inputs tried. Track coverage, vary generators, and keep mutation checks (§14).

---

## 10. Common mistakes

1. Properties that the broken code also satisfies (`sorted` without `permutation`).
2. A uniform generator for a bug that needs a particular shape (✓ the passing `buggy_dedup` test; ✓ 100,000 uniform decoder cases).
3. Using `prop_assume!` to throw away most inputs instead of constructing valid ones (use `prop_map`/`prop_compose!`).
4. Re-implementing the function under test as the expected value.
5. Non-deterministic properties (time, global state, `HashMap` iteration order).
6. Huge case counts in the default `cargo test` run.
7. Not committing the regression file, so a failure cannot be replayed on another machine.
8. A fuzz target with I/O, global state or unbounded memory; or one that catches panics (`catch_unwind`) and hides the crash.
9. Fuzzing without assertions, so only crashes are found.
10. Forgetting to turn a fixed crash into a permanent example test.
11. Treating a clean fuzz run as proof of absence.
12. Running a fuzz harness on release code with `overflow-checks` off and expecting arithmetic bugs (✓ the shift overflow is a panic only with overflow checks; `cargo fuzz` builds with debug assertions enabled by default, from memory, and I observed the panic).

---

## 11. Exercises

Answers are collapsed. Try each before opening it.

1. **(Warm-up)** Give one example test and one property for `fn reverse<T>(v: &mut Vec<T>)`.
2. **(Warm-up)** Name five common shapes of property.
3. **(Easy)** Write a `proptest` that checks `reverse` twice is the identity and that the length is preserved.
4. **(Medium)** `fn my_sort(v: &mut Vec<i32>)`: write a property set that cannot be satisfied by a function that returns an empty vector.
5. **(Medium)** A property about `fn merge(a: Vec<u8>, b: Vec<u8>) -> Vec<u8>` (merging two sorted lists) keeps passing, but the code has a bug when the lists share elements. What is wrong with the generator, and how do you fix it?
6. **(Medium-hard)** Outline a model-based test for a bounded stack with `push` (fails when full), `pop` and `peek`.
7. **(Hard)** Explain, without hand-waving, why libFuzzer found the 11-byte varint bug in thousands of executions when 100,000 uniform random cases did not.
8. **(Hard)** Write a `fuzz_target!` for `fn parse(&[u8]) -> Result<Msg, ParseError>` with `fn encode(&Msg) -> Vec<u8>`, asserting something stronger than "no crash".

<details markdown="1">
<summary>Answers</summary>

1. Example: reversing `[1, 2, 3]` gives `[3, 2, 1]`. Property: for every vector `v`, reversing twice gives `v` back (and the length never changes).
2. Round trip; totality (never panics); canonical form; oracle/model comparison; invariants; idempotence or commutativity; metamorphic relations; limits. (Any five.)
3. Generate any vector, reverse a copy twice, and compare:

   ```rust
   // needs proptest
   use proptest::prelude::*;

   fn reverse<T>(v: &mut Vec<T>) {
       v.reverse();
   }

   proptest! {
       #[test]
       fn reverse_twice_is_identity(original in proptest::collection::vec(any::<i32>(), 0..50)) {
           let mut v = original.clone();
           reverse(&mut v);
           prop_assert_eq!(v.len(), original.len());
           reverse(&mut v);
           prop_assert_eq!(v, original);
       }
   }
   ```
4. Sortedness alone is satisfied by an empty result, so also require the output to be a permutation of the input, ideally by comparing with the standard library's sort:
   ```rust
   // needs proptest
   use proptest::prelude::*;

   fn my_sort(v: &mut Vec<i32>) {
       v.sort_unstable();
   }

   proptest! {
       #[test]
       fn sort_is_sorted_and_a_permutation(input in proptest::collection::vec(-5i32..5, 0..40)) {
           let mut out = input.clone();
           my_sort(&mut out);
           prop_assert!(out.windows(2).all(|w| w[0] <= w[1]), "not sorted: {out:?}");
           let mut expected = input.clone();
           expected.sort();
           prop_assert_eq!(out, expected);
       }
   }
   ```
   (The small value range makes duplicates common, so equal-element handling is exercised.)
5. If both lists are generated from `any::<u8>()` with lengths up to 20, the chance that they share a value is high but the shape "equal elements at the same relative position, especially at the boundary between lists" is rare, and if the lists are not forced to be sorted the property may be vacuous. Fix: generate sorted lists from a small alphabet (`vec(0u8..4, 0..10)` then `.prop_map(|mut v| { v.sort(); v })`) so shared elements are the norm, and compare with an oracle (`concat` then `sort`).
6. Model: a `Vec<T>` plus the capacity. Operations: `Push(x)`, `Pop`, `Peek`. For each generated sequence, apply it to the real stack and the model; after every step assert that results agree (`push` fails exactly when `len == cap`; `pop`/`peek` return the model's last element or `None`) and that `len` agrees. Generate small capacities including 0 and 1 so that "full" and "empty" occur constantly.
7. Hitting the bug needs eleven consecutive bytes with the high bit set *and* the byte at index 9 to be `0x80` or `0x81`. A uniformly random byte string satisfies the first part with probability about 2^-11 and the second with about 2/128, and it also needs a length of 11 or more. I measured 21 of 100,000 uniform inputs with the first part and the second part makes the combination very rare. A coverage-guided fuzzer does not need the whole shape at once: each time a mutation gets one byte further into the loop it reaches code that was not covered before, so the input is kept and becomes the starting point for the next mutation; the target is reached by accumulating small wins (✓ found by about execution 6,400).
8. Parse, re-encode, and compare:

   ```rust,ignore
   fuzz_target!(|data: &[u8]| {
       if let Ok(msg) = parse(data) {
           // Anything accepted must survive a round trip...
           let bytes = encode(&msg);
           let again = parse(&bytes).expect("encoder output must parse");
           assert_eq!(again, msg);
           // ...and, if the format is canonical, re-encoding must match the input.
           assert_eq!(bytes, data);
       }
   });
   ```
   Drop the last assertion if the format accepts several spellings on purpose; if it should be canonical, that assertion finds decoders that are too lenient. Add a size limit parameter to `parse` so the fuzzer cannot be stalled by an allocation bomb.

</details>

---

## 12. Debugging challenge

A developer writes a de-duplicating sort and a property test. The test is green on every run, in CI and locally, even though a user reports that `[3, 3]` comes back unchanged.

```rust
// needs proptest
use proptest::prelude::*;

fn sort_dedup(mut v: Vec<u8>) -> Vec<u8> {
    v.sort();
    if v.len() > 3 {
        v.dedup();
    }
    v
}

fn strictly_increasing(v: &[u8]) -> bool {
    v.windows(2).all(|w| w[0] < w[1])
}

proptest! {
    #[test]
    fn output_is_strictly_increasing(v in proptest::collection::vec(any::<u8>(), 0..20)) {
        prop_assert!(strictly_increasing(&sort_dedup(v)));
    }
}
```

✓ The same shape passed at the default 256 cases. After changing only the element strategy to `0u8..4`, it failed and proptest reported `shrinks to v = [3, 3]`. Explain, and fix.

<details markdown="1">
<summary>Solution</summary>

There are two bugs. The code bug: `dedup` only runs when `v.len() > 3`, so short vectors with duplicates (length 2 or 3) are returned unchanged. The test bug: the generator draws every element from all 256 byte values, so a short vector that contains a repeated element is very rare (a few percent of the short cases at most, and the length must also be exactly 2 or 3). With 256 cases the test usually never sees one, and it passes even though the property it states is false.

Fix the code (`v.dedup()` unconditionally) **and** fix the test so it exercises the failing shape:

```rust
// needs proptest
use proptest::prelude::*;

fn sort_dedup(mut v: Vec<u8>) -> Vec<u8> {
    v.sort();
    v.dedup();
    v
}

fn strictly_increasing(v: &[u8]) -> bool {
    v.windows(2).all(|w| w[0] < w[1])
}

proptest! {
    // A small alphabet makes duplicates the common case, and lengths start at 0.
    #[test]
    fn output_is_strictly_increasing(v in proptest::collection::vec(0u8..4, 0..20)) {
        prop_assert!(strictly_increasing(&sort_dedup(v)));
    }

    // And keep a uniform version too, plus a stronger oracle-based property.
    #[test]
    fn matches_a_set(v in proptest::collection::vec(any::<u8>(), 0..20)) {
        let expected: Vec<u8> = v.iter().copied().collect::<std::collections::BTreeSet<_>>().into_iter().collect();
        prop_assert_eq!(sort_dedup(v), expected);
    }
}
```

The general rule: when a property test passes, ask "what would the smallest failing input look like, and would this generator ever produce it?" A quick way to check is to temporarily break the code on purpose and see whether the test notices (a mutation check, Lesson 39).

</details>

---

## 13. Interview questions

1. *What is property-based testing, and what is shrinking?*
2. *Give four kinds of properties you would write for an encoder/decoder pair.*
3. *Why can a property test pass while the code is wrong?*
4. *How does a coverage-guided fuzzer differ from a random-input test?*
5. *What should a good fuzz target look like?*
6. *How do you test a stateful data structure with properties?*
7. *Where does fuzzing fit in a CI pipeline, and what do you do with a crash it finds?*
8. *What bugs does Rust's type system not protect you from, that fuzzing finds?*

<details markdown="1">
<summary>Answers</summary>

1. Instead of fixed examples you state a rule that must hold for all inputs and check it on generated ones; when a case fails, shrinking simplifies it step by step to a minimal counterexample (✓ `[3, 3]`, eleven `128`s).
2. Round trip (`decode(encode(x)) == x`), totality (decoding arbitrary bytes never panics), canonical form (`encode(decode(b))` equals the consumed bytes), limits (output never exceeds the caller's limit), and the encoded length equals a closed-form formula (✓ `encoded_len`).
3. Because the generator may never produce the failing shape (✓ uniform bytes and short duplicate vectors; ✓ 100,000 uniform decoder cases missed an 11-byte input), or the property is too weak (sortedness without permutation). Bias the strategy toward boundaries and collisions and add an oracle.
4. A fuzzer instruments the code, keeps inputs that reach new coverage, and mutates those; it accumulates progress through nested conditions that random generation cannot reach (✓ libFuzzer reached the bug in thousands of executions). A random test has no feedback.
5. Small, fast, deterministic, no global state or I/O; fed the untrusted-input entry point; asserts round-trip/canonical/limit invariants, not just "no crash"; bounded input and memory; seeded with valid examples; its crash artifacts are kept as regression tests.
6. Model-based testing: generate sequences of operations, apply them to the real object and to a trivial model, and compare results and observable state after each step (✓ the cache against a `Vec`).
7. Short bounded runs on each change with a cached corpus, longer scheduled runs (hours), possibly continuous fuzzing. A crash is minimized (`tmin`), added as a permanent example test, the bug is fixed, and a property or generator is adjusted so that class of inputs stays covered.
8. Panics (index out of range, overflow, `unwrap`), infinite loops and quadratic blowups, unbounded allocations, logic errors such as accepting non-canonical encodings or disagreeing with a reference parser, and memory errors in `unsafe` code or FFI.

</details>

---

## 14. Practical challenge: `proplab`

Build a library crate `proplab` and test it with examples, properties and a fuzz target. Requirements:

1. `varint`: LEB128-style `u64` encoding. `decode_u64(&[u8]) -> Result<(u64, usize), VarintError>` accepts only the shortest form, rejects values over 64 bits, never panics. `encoded_len(n)` in closed form.
2. `rle`: `encode(&[u8])` as `(varint run length, byte)` pairs; `decode(&[u8], max_out)` with errors for a truncated run, a zero run, a bad varint, and an output that would exceed `max_out` (checked **before** allocating).
3. `lru::Lru<K, V>`: bounded cache; `get` and `put` count as use; `put` returns the evicted entry (or the rejected one when the capacity is 0).
4. Example tests that pin exact bytes and boundaries; property tests with biased generators for each component, including a model-based test for the cache; a fuzz target asserting canonical form and the limit.
5. `#![deny(missing_docs)]`; fmt, clippy `-D warnings`, rustdoc `-D warnings`.
6. Mutation checks on copies, recording which kind of test caught each mutant.

<details markdown="1">
<summary>Solution (spoilers: full source)</summary>

`Cargo.toml`:

```toml
[package]
name = "proplab"
version = "0.1.0"
edition = "2024"

[dependencies]

[dev-dependencies]
proptest = "1"
```

`src/lib.rs`:

```rust
//! `proplab`: three small components designed to be tested with properties
//! and fuzzing rather than only with hand-picked examples.
//!
//! * [`varint`]: LEB128-style variable-length `u64` encoding (canonical form only).
//! * [`rle`]: run-length encoding whose decoder takes an output limit, so a
//!   tiny hostile input cannot expand into gigabytes.
//! * [`lru`]: a fixed-capacity least-recently-used cache.
//!
//! ```
//! use proplab::{lru::Lru, rle, varint};
//!
//! let mut buf = Vec::new();
//! varint::encode_u64(300, &mut buf);
//! assert_eq!(buf, [0xAC, 0x02]);
//! assert_eq!(varint::decode_u64(&buf), Ok((300, 2)));
//!
//! let packed = rle::encode(b"aaab");
//! assert_eq!(rle::decode(&packed, 100).unwrap(), b"aaab");
//!
//! let mut cache = Lru::new(1);
//! cache.put("a", 1);
//! assert_eq!(cache.put("b", 2), Some(("a", 1))); // "a" was evicted
//! ```

#![deny(missing_docs)]

pub mod lru;
pub mod rle;
pub mod varint;
```

`src/varint.rs`:

```rust
//! Variable-length encoding of `u64`: seven data bits per byte, least
//! significant group first, high bit set on every byte except the last.
//!
//! Only the **canonical** (shortest) encoding is accepted when decoding, so
//! every number has exactly one valid byte string.

use core::fmt;

/// The longest valid encoding of a `u64`.
pub const MAX_LEN: usize = 10;

/// Why decoding failed.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[non_exhaustive]
pub enum VarintError {
    /// The input ended before the last byte (high bit clear) was seen.
    Truncated,
    /// The value does not fit in 64 bits.
    Overflow,
    /// The value is correct but not encoded in the shortest form.
    NonCanonical,
}

impl fmt::Display for VarintError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.write_str(match self {
            VarintError::Truncated => "varint is truncated",
            VarintError::Overflow => "varint does not fit in 64 bits",
            VarintError::NonCanonical => "varint is not in shortest form",
        })
    }
}

impl std::error::Error for VarintError {}

/// Appends the encoding of `n` to `out`.
pub fn encode_u64(mut n: u64, out: &mut Vec<u8>) {
    loop {
        let low = (n & 0x7F) as u8;
        n >>= 7;
        if n == 0 {
            out.push(low);
            return;
        }
        out.push(low | 0x80);
    }
}

/// The number of bytes `encode_u64(n, ..)` appends.
pub fn encoded_len(n: u64) -> usize {
    let bits = 64 - (n | 1).leading_zeros() as usize;
    bits.div_ceil(7)
}

/// Decodes one value from the start of `input`; returns it and the number of
/// bytes used.
pub fn decode_u64(input: &[u8]) -> Result<(u64, usize), VarintError> {
    let mut result: u64 = 0;
    for (i, &byte) in input.iter().enumerate() {
        if i == MAX_LEN {
            return Err(VarintError::Overflow);
        }
        let low = u64::from(byte & 0x7F);
        if i == MAX_LEN - 1 && low > 1 {
            return Err(VarintError::Overflow);
        }
        result |= low << (7 * i);
        if byte & 0x80 == 0 {
            if i > 0 && low == 0 {
                return Err(VarintError::NonCanonical);
            }
            return Ok((result, i + 1));
        }
    }
    Err(VarintError::Truncated)
}
```

`src/rle.rs`:

```rust
//! Run-length encoding: a sequence of `(run length as a varint, byte)` pairs.

use crate::varint::{self, VarintError};
use core::fmt;

/// Why decoding failed.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[non_exhaustive]
pub enum RleError {
    /// A run length could not be decoded.
    Varint(VarintError),
    /// The input ended between a run length and its byte.
    Truncated,
    /// A run length of zero (the encoder never produces one).
    ZeroRun,
    /// The output would exceed the caller's limit.
    OutputLimit,
}

impl fmt::Display for RleError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            RleError::Varint(e) => write!(f, "bad run length: {e}"),
            RleError::Truncated => f.write_str("input ends inside a run"),
            RleError::ZeroRun => f.write_str("run length of zero"),
            RleError::OutputLimit => f.write_str("decoded output would exceed the limit"),
        }
    }
}

impl std::error::Error for RleError {}

impl From<VarintError> for RleError {
    fn from(e: VarintError) -> RleError {
        RleError::Varint(e)
    }
}

/// Encodes `input` as `(length, byte)` pairs, one per maximal run.
pub fn encode(input: &[u8]) -> Vec<u8> {
    let mut out = Vec::new();
    let mut i = 0;
    while i < input.len() {
        let byte = input[i];
        let run = input[i..].iter().take_while(|&&b| b == byte).count();
        varint::encode_u64(run as u64, &mut out);
        out.push(byte);
        i += run;
    }
    out
}

/// Decodes `input`, refusing to produce more than `max_out` bytes.
///
/// The limit is checked **before** any memory is allocated for a run, so a
/// few bytes claiming a huge run cost nothing.
pub fn decode(input: &[u8], max_out: usize) -> Result<Vec<u8>, RleError> {
    let mut out: Vec<u8> = Vec::new();
    let mut pos = 0;
    while pos < input.len() {
        let (count, used) = varint::decode_u64(&input[pos..])?;
        pos += used;
        let byte = *input.get(pos).ok_or(RleError::Truncated)?;
        pos += 1;
        if count == 0 {
            return Err(RleError::ZeroRun);
        }
        let room = max_out - out.len();
        let count = usize::try_from(count).map_err(|_| RleError::OutputLimit)?;
        if count > room {
            return Err(RleError::OutputLimit);
        }
        out.resize(out.len() + count, byte);
    }
    Ok(out)
}
```

`src/lru.rs`:

```rust
//! A fixed-capacity least-recently-used cache.

use std::collections::{HashMap, VecDeque};
use std::hash::Hash;

/// A cache holding at most `capacity` entries; inserting into a full cache
/// evicts the least recently used one. Both `get` and `put` count as a use.
#[derive(Debug)]
pub struct Lru<K, V> {
    capacity: usize,
    map: HashMap<K, V>,
    /// Keys from least recently used (front) to most recently used (back).
    /// Invariant: contains exactly the keys of `map`, each once.
    order: VecDeque<K>,
}

impl<K: Hash + Eq + Clone, V> Lru<K, V> {
    /// Creates an empty cache. A capacity of zero stores nothing.
    pub fn new(capacity: usize) -> Lru<K, V> {
        Lru {
            capacity,
            map: HashMap::new(),
            order: VecDeque::new(),
        }
    }

    /// Number of entries.
    pub fn len(&self) -> usize {
        self.map.len()
    }

    /// Whether the cache is empty.
    pub fn is_empty(&self) -> bool {
        self.map.is_empty()
    }

    /// Moves `key` (which must be present) to the most-recently-used end.
    fn touch(&mut self, key: &K) {
        if let Some(pos) = self.order.iter().position(|k| k == key)
            && let Some(k) = self.order.remove(pos)
        {
            self.order.push_back(k);
        }
    }

    /// Looks up `key` and marks it most recently used.
    pub fn get(&mut self, key: &K) -> Option<&V> {
        if !self.map.contains_key(key) {
            return None;
        }
        self.touch(key);
        self.map.get(key)
    }

    /// Inserts or updates `key`, marking it most recently used.
    ///
    /// Returns the entry that was evicted to make room, if any. With a
    /// capacity of zero the new entry itself is returned.
    pub fn put(&mut self, key: K, value: V) -> Option<(K, V)> {
        if self.capacity == 0 {
            return Some((key, value));
        }
        if let Some(slot) = self.map.get_mut(&key) {
            *slot = value;
            self.touch(&key);
            return None;
        }
        let mut evicted = None;
        if self.map.len() == self.capacity
            && let Some(old) = self.order.pop_front()
            && let Some(v) = self.map.remove(&old)
        {
            evicted = Some((old, v));
        }
        self.order.push_back(key.clone());
        self.map.insert(key, value);
        evicted
    }

    /// Whether `key` is present; does not count as a use.
    pub fn contains(&self, key: &K) -> bool {
        self.map.contains_key(key)
    }

    /// The keys from least to most recently used.
    pub fn keys_lru_to_mru(&self) -> Vec<K> {
        self.order.iter().cloned().collect()
    }
}
```

`tests/examples.rs`:

```rust
//! Example-based tests: the cases a human thought of, including the exact
//! byte strings of the format.

use proplab::lru::Lru;
use proplab::rle::{self, RleError};
use proplab::varint::{self, VarintError, decode_u64, encode_u64, encoded_len};

fn enc(n: u64) -> Vec<u8> {
    let mut v = Vec::new();
    encode_u64(n, &mut v);
    v
}

#[test]
fn varint_known_encodings() {
    assert_eq!(enc(0), [0x00]);
    assert_eq!(enc(1), [0x01]);
    assert_eq!(enc(127), [0x7F]);
    assert_eq!(enc(128), [0x80, 0x01]);
    assert_eq!(enc(300), [0xAC, 0x02]);
    assert_eq!(enc(16_383), [0xFF, 0x7F]);
    assert_eq!(enc(16_384), [0x80, 0x80, 0x01]);
    let max = enc(u64::MAX);
    assert_eq!(max.len(), varint::MAX_LEN);
    assert_eq!(max[9], 0x01);
}

#[test]
fn varint_decode_reports_the_bytes_used() {
    assert_eq!(decode_u64(&[0x05, 0xFF, 0xFF]), Ok((5, 1)));
    assert_eq!(decode_u64(&[0xAC, 0x02, 0x99]), Ok((300, 2)));
}

#[test]
fn varint_rejects_bad_input() {
    assert_eq!(decode_u64(&[]), Err(VarintError::Truncated));
    assert_eq!(decode_u64(&[0x80]), Err(VarintError::Truncated));
    assert_eq!(decode_u64(&[0x80, 0x00]), Err(VarintError::NonCanonical));
    assert_eq!(
        decode_u64(&[0xFF, 0x80, 0x00]),
        Err(VarintError::NonCanonical)
    );
    assert_eq!(decode_u64(&[0x80; 10]), Err(VarintError::Truncated));
    assert_eq!(decode_u64(&[0x80; 11]), Err(VarintError::Overflow));
    let mut too_big = vec![0xFF; 9];
    too_big.push(0x02);
    assert_eq!(decode_u64(&too_big), Err(VarintError::Overflow));
    let mut max = vec![0xFF; 9];
    max.push(0x01);
    assert_eq!(decode_u64(&max), Ok((u64::MAX, 10)));
}

#[test]
fn varint_encoded_len_matches_the_encoder() {
    for n in [
        0,
        1,
        127,
        128,
        16_383,
        16_384,
        u64::from(u32::MAX),
        u64::MAX,
    ] {
        assert_eq!(encoded_len(n), enc(n).len(), "n = {n}");
    }
}

#[test]
fn rle_known_encodings() {
    assert_eq!(rle::encode(b""), b"");
    assert_eq!(rle::encode(b"a"), [1, b'a']);
    assert_eq!(rle::encode(b"aaab"), [3, b'a', 1, b'b']);
    let long = vec![7u8; 300];
    assert_eq!(rle::encode(&long), [0xAC, 0x02, 7]);
}

#[test]
fn rle_decode_enforces_the_output_limit_exactly() {
    let packed = rle::encode(&[9u8; 10]);
    assert_eq!(rle::decode(&packed, 10).unwrap().len(), 10);
    assert_eq!(rle::decode(&packed, 9), Err(RleError::OutputLimit));
    assert_eq!(rle::decode(&[], 0), Ok(vec![]));
}

#[test]
fn rle_a_tiny_hostile_input_is_refused_not_expanded() {
    // "2^63 copies of 'x'" in 10 bytes.
    let mut hostile = Vec::new();
    varint::encode_u64(1 << 63, &mut hostile);
    hostile.push(b'x');
    assert!(hostile.len() <= 11);
    assert_eq!(rle::decode(&hostile, 1 << 20), Err(RleError::OutputLimit));
}

#[test]
fn rle_decode_errors() {
    assert_eq!(rle::decode(&[3], 10), Err(RleError::Truncated));
    assert_eq!(rle::decode(&[0, b'a'], 10), Err(RleError::ZeroRun));
    assert_eq!(
        rle::decode(&[0x80], 10),
        Err(RleError::Varint(VarintError::Truncated))
    );
    assert_eq!(rle::decode(&[2, b'a', 2], 10), Err(RleError::Truncated));
}

#[test]
fn rle_accepts_adjacent_runs_of_the_same_byte() {
    assert_eq!(rle::decode(&[1, b'a', 2, b'a'], 10).unwrap(), b"aaa");
}

#[test]
fn error_messages() {
    assert_eq!(VarintError::Truncated.to_string(), "varint is truncated");
    assert_eq!(
        RleError::Varint(VarintError::Overflow).to_string(),
        "bad run length: varint does not fit in 64 bits"
    );
    assert_eq!(RleError::ZeroRun.to_string(), "run length of zero");
}

#[test]
fn lru_evicts_the_least_recently_used() {
    let mut c = Lru::new(2);
    assert_eq!(c.put('a', 1), None);
    assert_eq!(c.put('b', 2), None);
    assert_eq!(c.get(&'a'), Some(&1)); // a is now most recent
    assert_eq!(c.put('c', 3), Some(('b', 2)));
    assert!(c.contains(&'a') && c.contains(&'c') && !c.contains(&'b'));
    assert_eq!(c.keys_lru_to_mru(), ['a', 'c']);
}

#[test]
fn lru_update_refreshes_without_evicting() {
    let mut c = Lru::new(2);
    c.put(1, "x");
    c.put(2, "y");
    assert_eq!(c.put(1, "z"), None);
    assert_eq!(c.len(), 2);
    assert_eq!(c.keys_lru_to_mru(), [2, 1]);
    assert_eq!(c.get(&1), Some(&"z"));
}

#[test]
fn lru_capacity_zero_stores_nothing() {
    let mut c = Lru::new(0);
    assert_eq!(c.put(1, 1), Some((1, 1)));
    assert!(c.is_empty());
    assert_eq!(c.get(&1), None);
}

#[test]
fn lru_get_miss_does_not_change_order() {
    let mut c = Lru::new(3);
    c.put(1, ());
    c.put(2, ());
    assert_eq!(c.get(&9), None);
    assert_eq!(c.keys_lru_to_mru(), [1, 2]);
}
```

`tests/props.rs`:

```rust
//! Property tests: rules that must hold for every input, a model-based test
//! for the cache, and decoders fed arbitrary bytes.

use proplab::lru::Lru;
use proplab::rle;
use proplab::varint::{self, decode_u64, encode_u64, encoded_len};
use proptest::prelude::*;

/// Numbers biased toward the interesting boundaries as well as uniform ones.
fn interesting_u64() -> impl Strategy<Value = u64> {
    prop_oneof![
        4 => any::<u64>(),
        2 => 0u64..300,
        2 => (0u32..64).prop_map(|s| 1u64 << s),
        2 => (0u32..64).prop_map(|s| (1u64 << s).wrapping_sub(1)),
        1 => Just(u64::MAX),
    ]
}

proptest! {
    #[test]
    fn varint_round_trips_and_is_shortest(n in interesting_u64()) {
        let mut buf = Vec::new();
        encode_u64(n, &mut buf);
        prop_assert_eq!(buf.len(), encoded_len(n));
        prop_assert!(buf.len() <= varint::MAX_LEN);
        prop_assert_eq!(decode_u64(&buf), Ok((n, buf.len())));
    }

    /// Whatever the bytes are, decoding must not panic; and if it succeeds,
    /// re-encoding the value must give back exactly the bytes that were used
    /// (that is what "canonical" means).
    #[test]
    fn varint_decode_of_arbitrary_bytes_is_total_and_canonical(
        bytes in proptest::collection::vec(any::<u8>(), 0..16)
    ) {
        if let Ok((n, used)) = decode_u64(&bytes) {
            let mut again = Vec::new();
            encode_u64(n, &mut again);
            prop_assert_eq!(&again[..], &bytes[..used]);
        }
    }

    /// Bytes with the continuation bit set are rare in uniform data, so also
    /// generate inputs made mostly of them.
    #[test]
    fn varint_decode_survives_long_continuation_chains(
        mut bytes in proptest::collection::vec(0x80u8..=0xFF, 0..20),
        tail in proptest::option::of(0u8..0x80),
    ) {
        bytes.extend(tail);
        let _ = decode_u64(&bytes);
    }

    #[test]
    fn rle_round_trips_for_data_with_runs(data in proptest::collection::vec(0u8..3, 0..200)) {
        let packed = rle::encode(&data);
        prop_assert_eq!(rle::decode(&packed, data.len()), Ok(data.clone()));
        if !data.is_empty() {
            prop_assert!(rle::decode(&packed, data.len() - 1).is_err());
        }
    }

    #[test]
    fn rle_round_trips_for_arbitrary_data(data in proptest::collection::vec(any::<u8>(), 0..200)) {
        let packed = rle::encode(&data);
        prop_assert_eq!(rle::decode(&packed, usize::MAX), Ok(data));
    }

    #[test]
    fn rle_decode_of_arbitrary_bytes_respects_the_limit(
        bytes in proptest::collection::vec(any::<u8>(), 0..64),
        limit in 0usize..5000,
    ) {
        if let Ok(out) = rle::decode(&bytes, limit) {
            prop_assert!(out.len() <= limit);
            // Whatever was accepted must survive a trip through the encoder.
            prop_assert_eq!(rle::decode(&rle::encode(&out), limit), Ok(out));
        }
    }
}

#[derive(Debug, Clone)]
enum Op {
    Put(u8, u16),
    Get(u8),
}

fn ops() -> impl Strategy<Value = Vec<Op>> {
    // Few distinct keys, so that hits, updates and evictions all happen.
    let op = prop_oneof![
        3 => (0u8..6, any::<u16>()).prop_map(|(k, v)| Op::Put(k, v)),
        2 => (0u8..6).prop_map(Op::Get),
    ];
    proptest::collection::vec(op, 0..60)
}

/// The obviously-correct model: a `Vec` kept in recency order.
struct Model {
    cap: usize,
    entries: Vec<(u8, u16)>, // front = least recently used
}

impl Model {
    fn get(&mut self, k: u8) -> Option<u16> {
        let pos = self.entries.iter().position(|e| e.0 == k)?;
        let e = self.entries.remove(pos);
        self.entries.push(e);
        Some(e.1)
    }

    fn put(&mut self, k: u8, v: u16) -> Option<(u8, u16)> {
        if self.cap == 0 {
            return Some((k, v));
        }
        if let Some(pos) = self.entries.iter().position(|e| e.0 == k) {
            self.entries.remove(pos);
            self.entries.push((k, v));
            return None;
        }
        let evicted = (self.entries.len() == self.cap).then(|| self.entries.remove(0));
        self.entries.push((k, v));
        evicted
    }
}

proptest! {
    #[test]
    fn lru_behaves_like_the_model(cap in 0usize..5, ops in ops()) {
        let mut real: Lru<u8, u16> = Lru::new(cap);
        let mut model = Model { cap, entries: Vec::new() };
        for op in ops {
            match op {
                Op::Put(k, v) => prop_assert_eq!(real.put(k, v), model.put(k, v)),
                Op::Get(k) => prop_assert_eq!(real.get(&k).copied(), model.get(k)),
            }
            prop_assert!(real.len() <= cap);
            let keys: Vec<u8> = model.entries.iter().map(|e| e.0).collect();
            prop_assert_eq!(real.keys_lru_to_mru(), keys);
        }
    }
}
```

`fuzz/Cargo.toml` and `fuzz/fuzz_targets/decode.rs` are the two files listed in §7.2.

**Verification ✓.** `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, `cargo test --locked` and `RUSTDOCFLAGS="-D warnings" cargo doc --no-deps` pass: 14 example tests, 7 property tests, and 1 doctest. The fuzz target ran for 90 seconds on the correct crate (9,345,755 executions, no crash; `RUSTC_BOOTSTRAP=1`, see the note at the top) and found the planted bug in a copy of the crate (§5.2). **Not run:** Miri, `cargo mutants`, long fuzzing campaigns, structure-aware fuzzing, other targets. The crate is safe Rust only.

**Mutation checks ✓** (each applied to a copy with its own target directory; "caught by" lists the kinds of test that failed: **E** = example tests, **P** = property tests, **D** = doctest):

| # | Mutation | Result |
|---|---|---|
| V1 | encoder shifts by 6 instead of 7 | **caught** (E, P, D) |
| V2 | the "more than 10 bytes" check removed (the planted bug of §5.2) | **caught** (E: `[0x80; 11]`; P: the biased continuation-chain property) |
| V3 | tenth byte may carry value 2 (`low > 2`) | **caught** (E only) |
| V4 | non-canonical check fires for a single `0x00` byte too | **caught** (E, P) |
| V5 | non-canonical check removed | **caught by E only**. The canonical-form property did **not** fail in this run: it needs a random input that starts with a continuation byte followed by a zero byte, which is plausible but not guaranteed in 256 cases. It is a property that finds the bug *sometimes* |
| V6 | `encoded_len` uses `/ 7` instead of `div_ceil(7)` | **caught** (E, P) |
| V7 | `encoded_len` drops the `\| 1` (wrong for 0) | **caught** (E only) |
| R1 | RLE encoder advances by 1 instead of by the run length | **caught** (E, P, D) |
| R2 | output limit check `>` becomes `>=` | **caught** (E, P) |
| R3 | zero-length run accepted | **caught** (E only) |
| R4 | `input[pos]` instead of a checked `get` (panics on truncation) | **caught** (E, P) |
| R5 | `usize::try_from(count)` replaced by `count as usize` | **survived**: **equivalent mutant on this 64-bit target** (`u64` and `usize` have the same width, so the conversion never fails). On a 32-bit target it would be a real bug, and I did not test one |
| L1 | `touch` never moves a key | **caught** (E, P) |
| L2 | updating an existing key does not refresh it | **caught** (E, P) |
| L3 | evicts the most recently used instead of the least | **caught** (E, P) |
| L4 | capacity-zero special case removed | **caught** (E, P) |
| L5 | `get` does not refresh | **caught** (E, P) |
| L6 | `get` skips the early `contains_key` check | **survived**: **equivalent mutant** (`touch` on a missing key does nothing and `map.get` returns `None` anyway) |

Totals: 18 mutants, 16 caught, 2 equivalent (R5 on 64-bit, L6). Three observations. (a) The property tests caught every cache mutant that the examples caught, and for mutants such as R2, V4 and V6 the failing property gave a minimal input without my writing that case. (b) Four mutants (V3, V7, R3, and V5) were caught **only** by hand-written examples: properties are not a replacement for pinned edge cases (the tenth byte, the value 0, the zero run). (c) V2, the bug the uniform property could not find, was caught by an example I wrote *because I already knew the bug*, which is exactly why the fuzzer matters for bugs nobody has thought of yet.

**Design notes.**

- `decode_u64` returns the byte count so a caller can parse a stream of values; it rejects, rather than normalizes, non-canonical input, so one value has exactly one valid encoding (the property that the fuzz target asserts).
- `rle::decode` computes `room = max_out - out.len()` (never underflows because `out.len() <= max_out` is an invariant) and compares the run length to it before `resize`. The exact boundary (`limit` accepted, `limit - 1` refused) is an example test; the "never exceeds the limit for arbitrary bytes" rule is a property and a fuzz assertion.
- The cache uses `HashMap` plus `VecDeque` and an `O(n)` `touch`; it is correct, not fast. The invariant "`order` contains exactly the keys of `map`, once each" is what the model test checks indirectly through `keys_lru_to_mru`.
- The generators: `interesting_u64()` mixes uniform values with small numbers, powers of two, `2^k - 1` and `MAX`; the continuation-chain property builds inputs mostly of `0x80..=0xFF` bytes; the cache test uses 6 keys and capacities 0 to 4.
- Not handled: streaming decode of input split across reads, `no_std` use, hashing-based DoS considerations for the cache keys, concurrency (the cache is single-threaded), structure-aware fuzzing with `arbitrary`, and any performance measurement.

</details>

---

## 15. Key takeaways

- Properties state rules over all inputs and shrink failures to minimal cases (✓ `[3, 3]`); the best shapes are round trip, totality, canonical form, model comparison, invariants and limits.
- **The generator decides what you can find.** A green property with a uniform generator proved nothing about a short-duplicate bug (✓) or about an 11-byte decoder bug (✓ missed at 100,000 cases). Bias toward boundaries, collisions and long chains; pair with oracles.
- Model-based tests catch stateful bugs by comparing against a trivially correct model (✓ 5 of 5 cache mutants).
- Coverage-guided fuzzing finds narrow paths by accumulating small wins (✓ found the planted bug in about six thousand executions; 9.3 million clean executions in 91 s on the correct crate). It is the right tool for untrusted bytes and `unsafe` code; run it under a sanitizer and assert invariants, not only "no crash".
- Always apply a caller-chosen limit before trusting a length field (✓ the 10-byte "2^63 copies" input is refused).
- Turn every fuzz or property failure into a permanent example test; hand-written examples still catch things properties miss (✓ V3, V7, R3).
- None of this is proof. Mutation checks show what the suite would miss; two equivalent mutants and one platform-dependent one (R5) were reported as such.

**Connections.** [Lesson 39](39_testing.md) gave the harness and mutation method this lesson relies on; [Lesson 38](38_no_std_embedded_miri.md) introduced the frame decoder's "no input may panic" rule that the fuzz target enforces; [Lesson 36](36_unsafe_raw_pointers.md) and [Lesson 37](37_maybeuninit_ffi.md) explain why unsafe code needs sanitizers plus fuzzing; [Lesson 26](26_error_design.md) shaped the error enums; [Lesson 16](16_collections.md) covers the `HashMap` and `VecDeque` used by the cache. Next: Lesson 41 covers benchmarking and profiling, where the question changes from "is it correct?" to "is it fast, and how do we know?".

---

## 16. References

- The `proptest` book (strategies, `prop_oneof!`, `prop_compose!`, shrinking, persistence, configuration): <https://proptest-rs.github.io/proptest/>
- The Rust Fuzz Book (`cargo fuzz`, targets, corpora, `tmin`, structure-aware fuzzing with `arbitrary`): <https://rust-fuzz.github.io/book/>
- libFuzzer documentation (options such as `-max_total_time`, `-max_len`, `-rss_limit_mb`, `-timeout`, dictionaries): <https://llvm.org/docs/LibFuzzer.html>
- `libfuzzer-sys` and `cargo-fuzz` on crates.io: <https://crates.io/crates/libfuzzer-sys>, <https://crates.io/crates/cargo-fuzz>
- The `arbitrary` crate (structure-aware inputs; not run here): <https://crates.io/crates/arbitrary>
- OSS-Fuzz (continuous fuzzing for open-source projects; not used here): <https://google.github.io/oss-fuzz/>
- John Hughes, "QuickCheck Testing for Fun and Profit" and the QuickCheck line of work, the origin of property-based testing (paper title from memory).
- The Rust Book, chapter 11 "Writing Automated Tests", for the harness these tests run in: <https://doc.rust-lang.org/book/>

The URLs above are the standard documentation locations; I did not open them during this session (the sandbox cannot reach the documentation sites), so if a link has moved, search for the page title. Unchecked or from memory: that `cargo fuzz` requires a nightly toolchain and that building with `RUSTC_BOOTSTRAP=1` on stable is unsupported (I only observed that the latter worked); the 256-case default of `proptest`; the exact shrinking algorithm; that `cargo fuzz` builds with AddressSanitizer and debug assertions by default (I observed the panic, not the build flags); every libFuzzer flag other than `-max_total_time` (all others are from memory); dictionaries, structure-aware fuzzing with `arbitrary`, differential fuzzing practice, `loom`, OSS-Fuzz, AFL and honggfuzz; the CI recommendations; the claim that a decoder panic or non-canonical encoding can be a network-wide failure in a consensus system (a general statement, not tested here); and the QuickCheck paper reference. The measurement that 21 of 100,000 uniform inputs have eleven leading continuation bytes is from one run, and my probability estimate for the second condition (about 2/128) is a back-of-envelope calculation, not a measurement. Exercise answers 5, 6, 7 and 8 combine ✓-run facts with reasoning from memory, and 3, 4 and 12's code compiled separately (see my working notes). Everything marked ✓ was run on this machine's `rustc 1.97.0` with `proptest 1.12.0` and `cargo-fuzz 0.13.2`.
