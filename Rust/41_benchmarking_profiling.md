---
title: "Lesson 41 — Benchmarking and profiling"
---

# Lesson 41 — Benchmarking and profiling

**Module 13 · Testing, performance and tooling** · Status: generated · Edition used in examples: **2024** · External dependencies: `criterion 0.5` and `proptest` (both dev-dependencies only)

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, `criterion 0.5.1`, `proptest 1.12.0` and `valgrind 3.22.0` (`callgrind`, `callgrind_annotate`), on x86-64 Linux with 2 cores (Intel Xeon @ 2.10 GHz). As before, ✓ marks outputs and measurements I actually ran; unmarked statements are documented behavior I did not run. **Every timing in this lesson is a single measurement session on one shared, virtualized machine**; absolute numbers will differ on yours, and I show a case (§4) where two runs of *identical* code differ by several percent. Not available here, so described from memory only: `perf` (and therefore `perf stat`/`perf record`), `cargo flamegraph`, `hyperfine`, `heaptrack`, and DHAT. I could not open the documentation sites, so rules stated without ✓ are from memory; the closing paragraph lists them.

Builds on [Lesson 39](39_testing.md) (the harness, mutation checks), [Lesson 40](40_property_testing_fuzzing.md) (properties used here to prove the fast versions equal the simple one), [Lesson 16](16_collections.md) (`HashMap`, hashing), [Lesson 36](36_unsafe_raw_pointers.md) (the `unsafe impl GlobalAlloc` that counts allocations), and [Lesson 33](33_modules_api_design_rustdoc.md) (documented, tested public API). Lesson 42 is next; it appears here only as plain text.

---

## 1. Why this lesson exists

"Rust is fast" is a property of the language's *ceiling*, not of the program you wrote. The first version of most programs is correct and slow in boring ways: an allocation per item, a hasher chosen for safety rather than speed, a debug build. Fixing that requires three skills, in this order:

1. **Measure honestly.** A benchmark that measures the wrong thing is worse than none, because it produces confident numbers (§2, §3).
2. **Find where the cost is.** Profiling tells you *which* code and *which kind* of cost (instructions, allocations, cache misses), so you do not optimize what does not matter (§5, §6).
3. **Change, then re-measure.** Algorithm and allocation changes beat compiler flags; flags are a last, cheap step (§7, §8).

The running example is a crate, `perflab`: one task ("the *n* most frequent words in a text") implemented three ways that return **identical** results and differ only in cost. All numbers below come from it.

---

## 2. What a timing actually measures

### 2.1 The optimizer is part of your benchmark ✓

`Instant::now()` around a call looks innocent. This program times a loop that sums squares, three ways, in a release build:

```rust
use std::hint::black_box;
use std::time::Instant;

fn sum_to(n: u64) -> u64 {
    let mut s = 0u64;
    for i in 0..n {
        s = s.wrapping_add(i.wrapping_mul(i));
    }
    s
}

fn time<F: FnMut() -> u64>(label: &str, mut f: F) {
    let mut samples = Vec::new();
    for _ in 0..7 {
        let t = Instant::now();
        let r = f();
        let dt = t.elapsed();
        black_box(r);
        samples.push(dt.as_nanos());
    }
    samples.sort();
    println!("{label:<34} min {:>6} ns   median {:>6} ns", samples[0], samples[3]);
}

fn main() {
    const N: u64 = 50_000_000;
    time("constant input, result used", || sum_to(N));
    time("black_box(input)", || sum_to(black_box(N)));
    time("result ignored, constant input", || {
        let _ = sum_to(N);
        0
    });
}
```

✓ Output (`rustc -O`): `constant input, result used` min 40 ns, median 46 ns; `black_box(input)` min 32 ns, median 33 ns; `result ignored` min 44 ns, median 46 ns. Fifty million iterations in 33 to 46 nanoseconds is physically impossible (that would be over a billion iterations per nanosecond). The loop did not run. LLVM turns "sum of `i*i`" into a closed-form expression, and I confirmed it does so **even when the bound is hidden behind `black_box`**: `black_box` stops the compiler from *knowing the value* of `n`, but it does not stop it from replacing the loop with a formula that works for any `n`. The same experiment measured the cost of the clock itself: ✓ an empty `Instant::now()` plus `elapsed()` took a **median of 29 ns** (minimum 27 ns, maximum 6,317 ns in 1,000 tries). Anything near that size is measuring the clock.

Lessons:

- **A number that is too good to be true usually is.** Compute the implied rate (iterations per nanosecond, bytes per cycle) and check it against what the hardware can do.
- `std::hint::black_box(x)` is an identity function the optimizer must treat as opaque. It is a *hint* (the documentation says it is best-effort); use it on inputs *and* on results, and still sanity check the rate.
- Prefer a workload whose cost the optimizer cannot rewrite. Here is the same harness around a loop with a data dependency that has no closed form:

```rust
use std::hint::black_box;
use std::time::Instant;

fn mix(n: u64) -> u64 {
    let mut s = 0x9E37_79B9_7F4A_7C15u64;
    for i in 0..n {
        s ^= i;
        s = s.rotate_left(13).wrapping_mul(0x2545_F491_4F6C_DD1D);
    }
    s
}

fn main() {
    for n in [1_000_000u64, 10_000_000, 100_000_000] {
        let mut v = Vec::new();
        for _ in 0..7 {
            let t = Instant::now();
            let r = mix(black_box(n));
            let dt = t.elapsed();
            black_box(r);
            v.push(dt.as_nanos());
        }
        v.sort();
        println!("n = {n}: median {} ns ({:.2} ns/iter)", v[3], v[3] as f64 / n as f64);
    }
}
```

✓ Output: medians of 1,801,748 ns, 17,977,566 ns and 177,783,096 ns, i.e. **1.80, 1.80 and 1.78 ns per iteration**. Time that scales linearly with the input, and a per-iteration cost of a few cycles at 2.1 GHz, is what real work looks like. (Min and max over the seven samples were within about 5% of the median for each size on this run.)

### 2.2 Checklist for a trustworthy measurement

| Rule | Why |
|---|---|
| Release profile (`cargo bench`/`cargo run --release`) | ✓ the same function costs 4.2× more instructions at `opt-level = 0` (§7) |
| Hide inputs and consume results (`black_box`) | stops constant folding and dead-code removal, but not every rewrite (§2.1) |
| Separate **setup** from the measured work | building the input text cost more instructions than the naive algorithm itself (§6) |
| Many samples, report spread, not one number | ✓ two runs of identical code differed by about 5% (§4) |
| Same input in every variant; check outputs are equal | otherwise you compare different tasks (`perflab` asserts equality in tests) |
| Realistic sizes and shapes | cache behavior and allocator behavior change with size (§4) |
| Fix the environment: power state, other load, CPU pinning | virtualized or shared machines add noise you cannot remove |
| Re-measure after *every* change | intuition about performance is wrong often enough to matter |

---

## 3. The task, and three implementations

Words are maximal runs of alphabetic characters, compared case-insensitively. The result is the `n` most frequent words, sorted by count descending and then word ascending (so ties do not depend on hash order).

1. **`top_n_naive`**: `text.split(..)`, then `word.to_lowercase()` for every occurrence (an allocation each), counted in a `HashMap<String, usize>`.
2. **`top_n_fast`**: for ASCII text, lowercase the *whole text once*, count `HashMap<&str, usize>` that **borrows** from it, pre-size the map, and copy only the `n` winning words into `String`s. Non-ASCII text falls back to version 1, so results are always identical.
3. **`top_n_fnv`**: version 2 with a cheap hasher, 64-bit FNV-1a, instead of the standard library's default.

Three details deserve a sentence each:

- `best_n` uses `select_nth_unstable_by` to partition the `n` best entries to the front in expected linear time, then sorts only those. Sorting all distinct words would be `O(d log d)` for no benefit when `n` is small.
- The default `HashMap` hasher is SipHash 1-3 with a random per-map key. It is slower than FNV on short keys but resists **hash flooding** (an attacker choosing keys that all collide). FNV-1a is not collision-resistant; §8 returns to this.
- Equality of the three variants is a *test*, not an assumption (§14): an optimization that changes the answer is a bug.

---

## 4. Benchmarking with Criterion ✓

`cargo bench` can run Rust's built-in `#[bench]` only on nightly (from memory), so on stable the standard choice is **Criterion**. It runs a warm-up, collects many samples, and reports a confidence interval, plus a comparison against the previous run.

```toml
[dev-dependencies]
criterion = { version = "0.5", default-features = false, features = ["cargo_bench_support"] }

[[bench]]
name = "wordfreq"
harness = false          # Criterion supplies its own main
```

The benchmark file is in the practical challenge (§14). Its shape: one group, three sizes (1,000, 20,000 and 200,000 words over a 2,000-word vocabulary), each variant benchmarked on the same text with `Throughput::Bytes` so Criterion also reports MiB/s, and `black_box` on the input.

✓ Results of one run (`--warm-up-time 1 --measurement-time 3`; the middle number of Criterion's `[low estimate high]` interval):

| words | naive | fast | fnv | fast vs naive | fnv vs naive |
|---:|---:|---:|---:|---:|---:|
| 1,000 | 108.0 µs | 55.57 µs | 27.69 µs | 1.94× | 3.90× |
| 20,000 | 1.369 ms | 753.7 µs | 540.3 µs | 1.82× | 2.53× |
| 200,000 | 11.05 ms | 6.206 ms | 4.475 ms | 1.78× | 2.47× |

Reading the table honestly:

- The *ordering* (naive > fast > fnv) and the rough factors (about 1.8× and about 2.5×) were stable across the runs I did. The third digit was not.
- ✓ An earlier run in the same session, on identical code and workload, gave 1.29 ms / 712.7 µs / 515.6 µs at 20,000 words, i.e. about 5% faster than the table. The machine is shared and virtualized, and Criterion's own "change" line reported differences such as `+6.7%` with `p < 0.05` between two runs of **unchanged** code. A "significant" difference of a few percent between runs is therefore not evidence of a regression; it is evidence of noise, which Criterion can estimate within a run but cannot see across runs on a drifting machine.
- The 1,000-word case favors FNV more (3.9×) than the large ones (2.5×). With a small input, the fixed per-call costs and hashing are a larger share; with a large one, memory traffic and sorting matter more. Never extrapolate from one size.
- The throughput figures (fnv about 183 to 227 MiB/s) are for this task and text shape; they say nothing about other workloads.

---

## 5. Deterministic proxies: instruction counts and allocation counts

Wall-clock time is noisy. Two other measurements are **repeatable** and answer different questions.

### 5.1 Instruction counts with `callgrind` ✓

`valgrind --tool=callgrind` runs the program on a simulated CPU and counts instructions executed (`Ir`) per function and line. The count is the same on every run (no noise), slow to collect (a large constant factor), and **not time**: it ignores cache misses, branch mispredictions and memory latency, and all instructions count equally.

I wrote a small driver (`examples/profile.rs`, in §14): `profile <naive|fast|fnv|none> [words]`, where `none` only builds the input. Subtracting the `none` run from each variant gives the cost of the algorithm alone.

✓ At 20,000 words, default release profile: baseline (just generating the text) **12,472,737** instructions; net cost: naive **13,970,030**, fast **8,360,839** (1.67× fewer), fnv **5,649,422** (2.47× fewer). Compare with the time ratios above (1.82× and 2.53×): the instruction ratio is close but not equal to the time ratio. Fewer allocations and fewer cache misses can buy more (or less) than the instruction count suggests, so use the count to confirm a *direction* and to gate regressions, not to predict milliseconds.

### 5.2 Allocation counts in a test ✓

Allocation is a cost you can count exactly with a few lines: wrap `System` in a `GlobalAlloc` that increments a counter. Because the counter lives in the **test** binary, ordinary `cargo test` can assert it. The implementation is `unsafe` (Lesson 36), forwards every call unchanged, and uses `thread_local!` `Cell`s so that tests running in parallel threads do not see each other's allocations and so that touching the counter never allocates.

✓ Results (20,000 words over 500 distinct words): **naive 20,010 allocations, fast 15, fnv 15**, and ✓ the fast version's count stays at 15 when the text grows tenfold (the naive one grows with the number of words). These are asserted in `tests/alloc.rs`: the naive version allocates at least once per word, the fast ones stay at or below 16, and the count of the fast version does not grow with the input. A performance property turned into a regression test that never flakes.

---

## 6. Profiling: where does the cost live?

### 6.1 Two families of profilers

- **Sampling profilers** (`perf record`, and tools built on it such as `cargo flamegraph`, from memory): interrupt the program many times per second and record the call stack. Low overhead, work on release builds, see real cache and branch behavior, but statistical: short functions and short runs are poorly resolved. They need **frame information** to build stacks; the usual Rust recipe (from memory) is to keep optimizations and add debug line tables, which is what `debug = "line-tables-only"` in `[profile.release]` does here, plus sometimes `-C force-frame-pointers=yes` for frame-pointer unwinding. *I could not run `perf` or `cargo flamegraph` in this sandbox.*
- **Instrumenting/simulating profilers** (`callgrind`, ✓ used here): exact counts, deterministic, much slower, simulate rather than measure the real CPU.
- **Allocation profilers** (`heaptrack`, DHAT, or the counting allocator of §5.2): who allocates, how much, how often. Not run here except the counter.

Pick by question: *"where is the time going?"* → sampling; *"did this change reduce work, deterministically?"* → callgrind or counters; *"who allocates?"* → allocation profiler.

### 6.2 Reading a callgrind profile ✓

`callgrind_annotate` lists functions by exclusive `Ir`. ✓ For `naive` (whole program, 26,442,767 instructions), the top entries were allocator code (`_int_free` 8.3%, `malloc` 6.4%, `realloc` 5.8%, `free` 4.2%, `_int_realloc` 4.4%), SipHash (`core::hash::sip::Hasher::write` 5.4%, plus 3% in the `RandomState` hasher glue) and `str::to_lowercase` 5.1%. For `fast` (20,833,576 total): the allocator entries roughly halve in absolute terms (`malloc` 1,681,094 → 841,298; `free` 1,120,716 → 560,848; `_int_free` 2,199,340 → 1,081,657), SipHash is unchanged (1,430,883 → 1,403,922, the same number of hashes), and `hashbrown`'s `rustc_entry` appears. What remains of the allocator cost is consistent with **building the input** (I did not break it out of this listing).

Two traps this shows:

1. **The profile contains your setup.** `perflab::synthetic_text` was the largest single entry in the `fast` profile (2,018,443 instructions, 9.7%). If I profiled only that, I would "optimize" the text generator. Always profile with the setup excluded or subtracted (as done with the `none` baseline).
2. **Allocator time is spread over several functions.** No single `malloc` line tells you the damage; add up the family (`malloc`, `free`, `realloc`, `_int_*`) or use an allocation counter. After the change to borrowed `&str` keys, ✓ the allocation count dropped from 20,010 to 15, which is the single biggest structural improvement in this lesson.

### 6.3 A method

1. State the question and the workload (input shape and size) first.
2. Reproduce the slowness with a release build that keeps line tables.
3. Get a baseline number *and its noise*.
4. Profile; list the top costs by category (allocation, hashing, copying, compute, waiting).
5. Change **one** thing; re-run the equality tests; re-measure.
6. Stop when the remaining cost is not worth the added complexity.

---

## 7. Compiler and Cargo profile knobs ✓

Settings live under `[profile.release]` (or are overridden per run with `CARGO_PROFILE_RELEASE_<KEY>`). I measured the net instruction cost of each variant (20,000 words, baseline subtracted) under ten configurations:

| Setting | naive | fast | fnv |
|---|---:|---:|---:|
| default release (`opt-level = 3`) | 13,970,030 | 8,360,839 | 5,649,422 |
| `opt-level = 0` | 58,829,268 | 55,383,539 | 45,618,954 |
| `opt-level = 1` | 16,484,493 | 11,141,385 | 8,360,676 |
| `opt-level = 2` | 14,172,433 | 8,544,420 | 5,637,493 |
| `opt-level = 3` (explicit) | 13,971,145 | 8,335,681 | 5,649,394 |
| `opt-level = "z"` (size) | 20,035,986 | 14,876,840 | 10,922,628 |
| `lto = "fat"` | 12,533,598 | 7,752,791 | 4,949,451 |
| `codegen-units = 1` | 13,310,379 | 7,897,635 | 4,994,449 |
| `lto = "fat"` + `codegen-units = 1` | 12,566,764 | 7,807,221 | 4,953,710 |
| `panic = "abort"` | 13,984,647 | 8,357,829 | 5,671,222 |

How to read it:

- **`opt-level = 0` is 4.2× (naive) to 8.1× (fnv) more instructions.** Never judge performance from a debug build. (The abstraction layers of iterators and `HashMap` only collapse when optimized.)
- Going from `2` to `3` changed almost nothing for this workload. `opt-level = "z"` (optimize for size) costs 43% to 93% more instructions here.
- `lto = "fat"` saved about 10% for `naive` and 12% for `fnv`; `codegen-units = 1` saved about 5% to 12%; both together were no better than LTO alone on this program. The effect depends on the code and how much crosses crate boundaries; measure yours.
- `panic = "abort"` changed the count by under 0.5% here (it can still matter for code size and for code with many landing pads).
- **The best knob is smaller than the best algorithm change:** the cheapest flag change saved about 10%, while switching `naive` to `fast` saved 40% and to `fnv` saved 60% in instructions.

Costs of the knobs: `lto = "fat"` and `codegen-units = 1` increase compile time (I did not time it); `opt-level = 3` can grow code size. A failed experiment I discarded: building with `-C target-cpu=native` produced a binary that valgrind could not run (this CPU has AVX-512, which this valgrind version does not emulate), so I have no number for it. `target-cpu=native` also makes the binary non-portable to older CPUs, which matters for anything you distribute.

---

## 8. What to change, in order

1. **Algorithm and data structure.** `select_nth_unstable_by` instead of sorting everything; a map instead of a scan.
2. **Allocation.** Borrow instead of copying (`&str` keys), lowercase once instead of per word, pre-size (`with_capacity`). ✓ 20,010 → 15 allocations. The capacity hint matters even for tiny maps: removing it made the fast version allocate 22 times instead of 15 (✓ a mutant, §14), because the map regrows as it fills.
3. **Hashing.** ✓ FNV-1a was 1.4× faster than the default hasher in the fast version at 20,000 words (753.7 µs → 540.3 µs). The price is real: **FNV-1a is not collision-resistant, so any map keyed by attacker-controlled input can be flooded into quadratic behavior** (a denial of service). The default `RandomState` exists to prevent that. Use a fast non-keyed hasher only for trusted keys (compiler-internal tables, small fixed vocabularies); if you need speed *and* safety, look at keyed fast hashers (such as `ahash`, from memory; not run here).
4. **Data layout and cache behavior.** Contiguous `Vec`s beat pointer chasing (Lesson 16); measure with a sampling profiler or hardware counters, which I could not run here.
5. **Parallelism** (the concurrency lessons): only after single-thread costs are understood.
6. **Compiler knobs** (§7), last: cheap, small, sometimes free.

---

## 9. Performance, reliability, security

- **Benchmark regression gates:** gating CI on wall-clock time is flaky on shared runners (§4). Gate on **deterministic** measures: instruction counts, allocation counts (✓ `tests/alloc.rs`), binary size. Keep wall-clock benchmarks for trend tracking on a quiet machine.
- **Correctness first:** optimized code is new code. ✓ `perflab` has property tests asserting that `top_n_fast` and `top_n_fnv` equal `top_n_naive` for arbitrary Unicode text, including non-ASCII input (the Unicode example has a word with a final sigma, which `str::to_lowercase` treats specially, from memory; the non-ASCII fallback exists because the ASCII shortcut cannot reproduce that).
- **Security:** performance changes can open attack surface. A cheap hasher on untrusted keys is a vulnerability, not an optimization. `unsafe` "speedups" need Lesson 36's discipline. Rust's memory safety does not make an algorithmically unbounded loop safe: an input-controlled worst case is a denial of service.
- **Reliability:** do not leave benchmark-only `black_box` or counting allocators in production code. The counting `GlobalAlloc` is only in a test binary, so it cannot affect the library.
- **Profile builds are not release builds:** adding `debug = "line-tables-only"` changes binary size (not the generated machine code in the cases I looked at, but measure if it matters to you), and instrumenting profilers change timing.

---

## 10. Common mistakes

1. Benchmarking a debug build (✓ 4.2× to 8.1× the instructions).
2. Trusting a number that implies impossible throughput (§2.1).
3. Timing setup together with the work (✓ the text generator cost more than the naive algorithm, §6.2).
4. Comparing variants on different inputs, or without checking they return the same answer.
5. Reading one run as truth, or a "significant" few-percent change between runs as a regression (§4).
6. Optimizing the first function in a profile without checking whether it belongs to the code under test.
7. Using a non-collision-resistant hasher with untrusted keys.
8. Treating instruction counts as time, or time as instruction counts (§5.1).
9. Reaching for flags (`lto`, `codegen-units`, `target-cpu`) before fixing allocation and algorithm.
10. Forgetting that a micro-benchmark's winner may lose in the real program (cache, allocator, and input shape differ).

---

## 11. Exercises

Answers are collapsed. Try each before opening it.

1. **(Warm-up)** Why must performance be judged on a release build? Give the size of the effect measured in this lesson.
2. **(Warm-up)** What does `std::hint::black_box` guarantee, and what did §2.1 show it does *not* do?
3. **(Easy)** Write a tiny harness `median_ns(f)` that runs a closure 7 times with `Instant` and returns the median, using `black_box` on the result.
4. **(Medium)** Why does the allocation counter in `tests/alloc.rs` use `thread_local!` counters instead of global atomics?
5. **(Medium)** Criterion printed "change: +6.7% (p < 0.05)" for a benchmark whose code did not change. What are two possible explanations, and what would you do next?
6. **(Medium-hard)** `top_n_fnv` is 1.4× faster than `top_n_fast`. A colleague wants to use FNV-1a for the `HashMap` of a public web service's per-user request counters, keyed by a request header. What do you say?
7. **(Hard)** Instruction counts said 1.67× and wall-clock time said 1.82× for `fast` vs `naive`. Give two reasons the two ratios can differ.
8. **(Hard)** Design a CI performance check for `top_n_fast` that is stable on shared runners.

<details markdown="1">
<summary>Answers</summary>

1. Optimizations (inlining, loop transformations, removing iterator and `HashMap` abstraction layers) are what makes Rust fast; a debug build skips most of them. ✓ In §7, `opt-level = 0` ran 58.8 M instructions for the naive variant against 14.0 M at the default release setting (4.2×), and 45.6 M against 5.6 M for the FNV variant (8.1×).
2. It returns its argument and the optimizer must treat the value as opaque (an optimization barrier for the *value*), on a best-effort basis. It does not stop the compiler from replacing a whole loop with a closed form: ✓ `sum_to(black_box(50_000_000))` still ran in about 33 ns. It also does not make a benchmark representative; it only removes one way to cheat.
3. A possible answer:

   ```rust
   use std::hint::black_box;
   use std::time::Instant;

   fn median_ns<R>(mut f: impl FnMut() -> R) -> u128 {
       let mut samples: Vec<u128> = (0..7)
           .map(|_| {
               let t = Instant::now();
               let r = f();
               let dt = t.elapsed().as_nanos();
               black_box(r);
               dt
           })
           .collect();
       samples.sort_unstable();
       samples[3]
   }

   fn main() {
       let v: Vec<u64> = (0..1_000_000).collect();
       let ns = median_ns(|| black_box(&v).iter().fold(0u64, |a, x| a ^ x.rotate_left(7)));
       println!("{ns} ns");
   }
   ```

   It is still crude: no warm-up, only seven samples, no outlier analysis. That is why Criterion exists.
4. Tests run in parallel threads by default. A global counter would add the allocations of *other* tests running at the same moment, making assertions such as "at most 16 allocations" flaky. A per-thread counter measures only the current thread. The counters are `const`-initialized `Cell`s so reading and updating them cannot itself allocate (which would recurse into the allocator).
5. (a) Noise: on a shared or virtualized machine, frequency, neighbors and cache state drift between runs; ✓ I saw exactly this (about 5% between two runs of identical code). (b) The baseline Criterion compared against was recorded under different conditions. Next: re-run a few times, close other work, compare the confidence intervals (not just the point estimate), and, if the change matters, confirm with a deterministic measure (instruction count) before concluding anything.
6. Say no for untrusted keys. FNV-1a has no per-process secret and is trivial to construct collisions for, so an attacker who controls the header can make many keys land in the same bucket and turn constant-time lookups into linear scans (hash flooding). Keep the default `RandomState`, or use a keyed hasher designed for this. If the counters are a measured bottleneck, first reduce allocations and key size; a 1.4× hasher gain on a small share of the request path is rarely worth a vulnerability.
7. Examples: instructions are not equal in cost (the naive variant executes many `malloc`/`free` calls whose memory access patterns cause cache misses, which cost more cycles than their instruction count suggests); the time includes memory effects, branch mispredictions and clock noise that callgrind does not simulate; the baseline subtraction I applied to the instruction counts has no equivalent in the Criterion run, which times only the call but runs with a warmed cache. Any two of these (the first is the most likely).
8. Gate on deterministic quantities: a test that asserts allocation count (exactly or within a small bound, as `tests/alloc.rs` does) and an instruction-count budget measured with `callgrind` (a fixed threshold with a margin, rerun when you intentionally change the algorithm). Keep wall-clock Criterion runs as a non-blocking report on a dedicated machine, comparing against a stored baseline, and require a regression to repeat before acting. Always keep the equality property tests, since a "faster" change that returns a different answer must fail.

</details>

---

## 12. Debugging challenge

A developer reports "our `sum_to` takes 33 nanoseconds for 50 million iterations, 10,000 times faster than C!" and wants to ship a "performance win". Find everything wrong with the measurement and fix it.

```rust
use std::hint::black_box;
use std::time::Instant;

fn sum_to(n: u64) -> u64 {
    let mut s = 0u64;
    for i in 0..n {
        s = s.wrapping_add(i.wrapping_mul(i));
    }
    s
}

fn main() {
    let t = Instant::now();
    let r = sum_to(black_box(50_000_000));
    let ns = t.elapsed().as_nanos();
    black_box(r);
    println!("{ns} ns for 50M iterations = {:.3} ns/iter", ns as f64 / 50_000_000.0);
}
```

<details markdown="1">
<summary>Solution</summary>

Problems:

1. **The loop did not run.** 33 ns is about the cost of reading the clock twice (✓ an empty `Instant::now()` + `elapsed()` has a median of 29 ns on this machine). The implied 0.0000007 ns/iteration is impossible. LLVM turns the sum into a closed form; ✓ `black_box(n)` hid the *value* but not the *formula*.
2. **One sample.** No warm-up, no spread; a single reading could not distinguish a real change from noise (✓ about 5% drift between runs).
3. **The function measured is not the one that ships** if the real code has a different loop; the benchmark checks a toy.
4. It is only valid in a **release** build, and the report does not say which it was.

Fix: use a workload with a data dependency the optimizer cannot collapse, hide the input and consume the result, take several samples, report the median and the implied rate, and cross-check with an instruction count if the number looks surprising:

```rust
use std::hint::black_box;
use std::time::Instant;

fn mix(n: u64) -> u64 {
    let mut s = 0x9E37_79B9_7F4A_7C15u64;
    for i in 0..n {
        s ^= i;
        s = s.rotate_left(13).wrapping_mul(0x2545_F491_4F6C_DD1D);
    }
    s
}

fn main() {
    let n = 50_000_000u64;
    let mut v: Vec<u128> = (0..7)
        .map(|_| {
            let t = Instant::now();
            let r = mix(black_box(n));
            let dt = t.elapsed().as_nanos();
            black_box(r);
            dt
        })
        .collect();
    v.sort_unstable();
    println!("median {} ns = {:.2} ns/iter", v[3], v[3] as f64 / n as f64);
}
```

✓ The same shape of loop measured about 1.8 ns per iteration, which is a believable few cycles per iteration.

</details>

---

## 13. Interview questions

1. *Why do you benchmark in release mode, and what else can make a benchmark lie?*
2. *What does `black_box` do, and what can it not do?*
3. *What is the difference between a sampling and an instrumenting profiler?*
4. *When would you use instruction counts instead of wall-clock time?*
5. *The program is slow and allocates heavily. What do you do, in order?*
6. *What are the trade-offs of `lto`, `codegen-units = 1`, `panic = "abort"` and `opt-level = "z"`?*
7. *Why is a non-keyed hash function dangerous in a server?*
8. *How would you stop a performance regression from reaching production without flaky CI?*

<details markdown="1">
<summary>Answers</summary>

1. Debug builds skip optimizations (✓ 4.2× to 8.1× the instructions in this lesson). Other ways to lie: dead-code elimination or constant folding (✓ a 50-million-iteration loop timed at 33 ns), including setup in the timing, unrepresentative input size or shape, a single sample, a noisy machine, comparing variants that compute different things.
2. It is an identity function the optimizer treats as opaque, used to hide inputs and force results to be "used". It is a best-effort hint and cannot stop whole-loop rewrites (✓), so sanity check the implied rate.
3. A sampling profiler interrupts a running program and records stacks: low overhead, statistical, real hardware behavior. An instrumenting or simulating profiler (`callgrind`) counts exactly what executes: deterministic and precise, but slow and not real timing.
4. When you need a repeatable number (CI gates, small differences smaller than the noise, shared machines). Instruction counts ignore cache and branch effects, so they confirm direction, not magnitude (✓ 1.67× vs 1.82×).
5. Measure first; profile the right workload with setup excluded; fix algorithmic issues, then allocations (borrow, pre-size, reuse buffers; ✓ 20,010 → 15), then hashing and data layout, then parallelism, and only then compiler flags; re-measure and re-run correctness tests at each step.
6. `lto` and `codegen-units = 1` can improve speed (✓ about 5% to 12% fewer instructions here) at the cost of longer compiles; `panic = "abort"` shrinks code and removes unwinding but changes panic behavior (no destructors run on panic, `catch_unwind` stops working); `opt-level = "z"` favors size over speed (✓ 43% to 93% more instructions here).
7. An attacker who knows the hash function can craft keys that collide, turning `O(1)` operations into `O(n)` and exhausting the CPU (hash flooding). The default `RandomState` uses a random key per map to prevent that; FNV-1a does not. It is acceptable only for trusted keys.
8. Deterministic gates (allocation-count tests, instruction-count budgets, binary size) in CI; wall-clock benchmarks as a non-blocking trend report on dedicated hardware; equality/property tests to guard correctness; require a repeated regression before reacting.

</details>

---

## 14. Practical challenge: `perflab`

Build the library crate `perflab` and its measurement tooling. Requirements:

1. `top_n_naive`, `top_n_fast` (ASCII fast path with borrowed keys, pre-sized map, one lowercase pass, non-ASCII fallback) and `top_n_fnv` (same with a hand-written `Hasher`). Ties broken alphabetically. All three return identical results.
2. A deterministic `synthetic_text(words, vocab, seed)` generator for realistic benchmarking input, with a test that it has the shape it claims.
3. Tests: example tests that pin exact results and ordering; property tests proving the three variants equal on arbitrary text; a test of the hasher against published FNV-1a vectors; allocation-count tests with a counting global allocator.
4. A Criterion benchmark at three sizes with throughput, and a profiling driver example.
5. Run your own mutation checks (as in Lesson 39) and say which survive and why.

<details markdown="1">
<summary>Solution</summary>

`Cargo.toml`:

```toml
[package]
name = "perflab"
version = "0.1.0"
edition = "2024"

[dependencies]

[dev-dependencies]
criterion = { version = "0.5", default-features = false, features = ["cargo_bench_support"] }
proptest = "1"

[[bench]]
name = "wordfreq"
harness = false

# Keep optimizations, add line tables so profilers can name source lines.
[profile.release]
debug = "line-tables-only"
```

`src/lib.rs`:

````rust
//! `perflab`: one task, "the `n` most frequent words in a text", implemented
//! three ways so that benchmarking and profiling have something real to
//! compare. All variants return **identical** results; only their cost
//! differs.
//!
//! Words are maximal runs of alphabetic characters, compared case-insensitively.
//! Results are sorted by count (descending), then by word (ascending).
//!
//! ```
//! use perflab::{top_n_fast, top_n_naive};
//!
//! let text = "the cat and The dog and THE bird";
//! let top = top_n_fast(text, 2);
//! assert_eq!(top, vec![("the".to_string(), 3), ("and".to_string(), 2)]);
//! assert_eq!(top, top_n_naive(text, 2));
//! ```

#![deny(missing_docs)]

use std::collections::HashMap;
use std::hash::{BuildHasherDefault, Hasher};

/// A deterministic pseudo-random text generator for tests and benchmarks:
/// `words` words drawn (skewed toward small indexes) from `vocab` distinct
/// lowercase words, with some words capitalized, separated by spaces and
/// punctuation.
pub fn synthetic_text(words: usize, vocab: usize, seed: u64) -> String {
    let mut state = seed | 1;
    let mut next = move || {
        // xorshift64*
        state ^= state >> 12;
        state ^= state << 25;
        state ^= state >> 27;
        state.wrapping_mul(0x2545_F491_4F6C_DD1D)
    };
    let mut out = String::with_capacity(words * 7);
    for i in 0..words {
        // Product of two draws skews the index toward 0 (a few common words).
        let a = (next() >> 33) as usize % vocab.max(1);
        let b = (next() >> 33) as usize % vocab.max(1);
        let idx = a * b / vocab.max(1);
        // An alphabetic-only word: "w", the index in base 26, then "x".
        let mut word = String::from("w");
        let mut k = idx;
        loop {
            word.push(char::from(b'a' + (k % 26) as u8));
            k /= 26;
            if k == 0 {
                break;
            }
        }
        word.push('x');
        if next() % 8 == 0 {
            let mut chars = word.chars();
            if let Some(first) = chars.next() {
                out.extend(first.to_uppercase());
                out.push_str(chars.as_str());
            }
        } else {
            out.push_str(&word);
        }
        out.push_str(if i % 11 == 10 { ". " } else { " " });
    }
    out
}

/// Orders entries by count (descending), then word (ascending).
fn cmp_entries<S: AsRef<str>>(a: &(S, usize), b: &(S, usize)) -> std::cmp::Ordering {
    b.1.cmp(&a.1).then_with(|| a.0.as_ref().cmp(b.0.as_ref()))
}

/// Keeps the best `n` entries without sorting all of them.
fn best_n<S: AsRef<str>>(mut entries: Vec<(S, usize)>, n: usize) -> Vec<(S, usize)> {
    if n == 0 {
        return Vec::new();
    }
    if n < entries.len() {
        entries.select_nth_unstable_by(n - 1, cmp_entries);
        entries.truncate(n);
    }
    entries.sort_unstable_by(cmp_entries);
    entries
}

/// The straightforward version: allocates a lowercase `String` for every
/// word occurrence and another one for every map insertion.
pub fn top_n_naive(text: &str, n: usize) -> Vec<(String, usize)> {
    let mut counts: HashMap<String, usize> = HashMap::new();
    for word in text
        .split(|c: char| !c.is_alphabetic())
        .filter(|w| !w.is_empty())
    {
        *counts.entry(word.to_lowercase()).or_insert(0) += 1;
    }
    best_n(counts.into_iter().collect(), n)
}

/// ASCII fast path: lowercases the text once, counts **borrowed** words, and
/// copies only the `n` winners. Non-ASCII text falls back to the general
/// version, so results are always identical to [`top_n_naive`].
pub fn top_n_fast(text: &str, n: usize) -> Vec<(String, usize)> {
    if !text.is_ascii() {
        return top_n_naive(text, n);
    }
    let lowered = text.to_ascii_lowercase();
    let mut counts: HashMap<&str, usize> = HashMap::with_capacity(256);
    for word in lowered
        .split(|c: char| !c.is_ascii_alphabetic())
        .filter(|w| !w.is_empty())
    {
        *counts.entry(word).or_insert(0) += 1;
    }
    let best = best_n(counts.into_iter().collect(), n);
    best.into_iter().map(|(w, c)| (w.to_owned(), c)).collect()
}

/// 64-bit FNV-1a: very cheap for short keys, **not** collision-resistant.
/// Do not use it with attacker-controlled keys (hash flooding).
#[derive(Default, Clone, Copy)]
pub struct Fnv1a(u64);

impl Hasher for Fnv1a {
    fn finish(&self) -> u64 {
        self.0
    }

    fn write(&mut self, bytes: &[u8]) {
        let mut h = if self.0 == 0 {
            0xcbf2_9ce4_8422_2325
        } else {
            self.0
        };
        for &b in bytes {
            h ^= u64::from(b);
            h = h.wrapping_mul(0x0000_0100_0000_01b3);
        }
        self.0 = h;
    }
}

/// Like [`top_n_fast`] but with the cheap [`Fnv1a`] hasher instead of the
/// standard library's DoS-resistant default. Faster on short words; only
/// appropriate when the text is trusted.
pub fn top_n_fnv(text: &str, n: usize) -> Vec<(String, usize)> {
    if !text.is_ascii() {
        return top_n_naive(text, n);
    }
    let lowered = text.to_ascii_lowercase();
    let mut counts: HashMap<&str, usize, BuildHasherDefault<Fnv1a>> =
        HashMap::with_capacity_and_hasher(256, BuildHasherDefault::default());
    for word in lowered
        .split(|c: char| !c.is_ascii_alphabetic())
        .filter(|w| !w.is_empty())
    {
        *counts.entry(word).or_insert(0) += 1;
    }
    let best = best_n(counts.into_iter().collect(), n);
    best.into_iter().map(|(w, c)| (w.to_owned(), c)).collect()
}
````

`tests/equivalence.rs`:

```rust
//! The optimized versions must return exactly what the simple one returns.

use perflab::{top_n_fast, top_n_fnv, top_n_naive};
use proptest::prelude::*;

#[test]
fn known_results_and_ordering_rules() {
    let text = "b a B a c, c; C!";
    // a: 2, b: 2, c: 3. Ties broken alphabetically.
    let all = vec![
        ("c".to_string(), 3),
        ("a".to_string(), 2),
        ("b".to_string(), 2),
    ];
    assert_eq!(top_n_naive(text, 10), all);
    assert_eq!(top_n_fast(text, 10), all);
    assert_eq!(top_n_fnv(text, 10), all);
    assert_eq!(top_n_fast(text, 2), all[..2]);
    assert_eq!(top_n_fast(text, 0), Vec::new());
    assert_eq!(top_n_fast("", 5), Vec::new());
    assert_eq!(top_n_fast("123 !!! 456", 5), Vec::new());
}

#[test]
fn ties_are_resolved_by_word_not_by_hash_order() {
    let text = "zeta alpha mu beta";
    let expected: Vec<String> = ["alpha", "beta", "mu"].map(String::from).to_vec();
    for f in [top_n_naive, top_n_fast, top_n_fnv] {
        let got: Vec<String> = f(text, 3).into_iter().map(|(w, _)| w).collect();
        assert_eq!(got, expected);
    }
}

#[test]
fn unicode_text_takes_the_general_path_and_still_agrees() {
    let text = "Éé ÉÉ straße STRASSE ǅ ǆ ΣΑΣ σας";
    let expected = top_n_naive(text, 10);
    assert_eq!(top_n_fast(text, 10), expected);
    assert_eq!(top_n_fnv(text, 10), expected);
    assert!(expected.iter().any(|(w, c)| w == "éé" && *c == 2));
}

#[test]
fn synthetic_text_has_the_requested_shape() {
    let text = perflab::synthetic_text(3_000, 200, 9);
    let all = top_n_naive(&text, usize::MAX);
    let total: usize = all.iter().map(|(_, c)| c).sum();
    assert_eq!(total, 3_000, "every generated word is a countable word");
    assert!(
        all.len() <= 200,
        "at most `vocab` distinct words, got {}",
        all.len()
    );
    assert!(all.len() > 50, "and a decent variety, got {}", all.len());
    assert_eq!(
        perflab::synthetic_text(100, 10, 5),
        perflab::synthetic_text(100, 10, 5)
    );
    assert_ne!(
        perflab::synthetic_text(100, 10, 5),
        perflab::synthetic_text(100, 10, 6)
    );
}

#[test]
fn n_larger_than_the_number_of_distinct_words_returns_all() {
    let got = top_n_fast("x y x", 100);
    assert_eq!(got, vec![("x".to_string(), 2), ("y".to_string(), 1)]);
}

proptest! {
    #[test]
    fn variants_agree_on_arbitrary_text(text in ".{0,200}", n in 0usize..8) {
        let expected = top_n_naive(&text, n);
        prop_assert_eq!(top_n_fast(&text, n), expected.clone());
        prop_assert_eq!(top_n_fnv(&text, n), expected);
    }

    #[test]
    fn variants_agree_on_ascii_text_with_few_words(
        text in "[a-cA-C ,.]{0,120}", n in 0usize..5
    ) {
        let expected = top_n_naive(&text, n);
        prop_assert_eq!(top_n_fast(&text, n), expected.clone());
        prop_assert_eq!(top_n_fnv(&text, n), expected);
    }

    #[test]
    fn counts_are_sorted_descending_and_bounded(
        text in "[a-d ]{0,80}", n in 0usize..6
    ) {
        let got = top_n_fast(&text, n);
        prop_assert!(got.len() <= n);
        prop_assert!(got.windows(2).all(|w| w[0].1 > w[1].1 || (w[0].1 == w[1].1 && w[0].0 < w[1].0)));
    }
}

#[test]
fn fnv_hasher_matches_published_fnv1a_64_vectors() {
    use perflab::Fnv1a;
    use std::hash::Hasher;
    let hash = |chunks: &[&[u8]]| {
        let mut h = Fnv1a::default();
        for c in chunks {
            h.write(c);
        }
        h.finish()
    };
    assert_eq!(hash(&[b"a"]), 0xaf63_dc4c_8601_ec8c);
    assert_eq!(hash(&[b"foobar"]), 0x8594_4171_f739_67e8);
    // Splitting the input across several write calls must not change the hash.
    assert_eq!(hash(&[b"foo", b"bar"]), hash(&[b"foobar"]));
}
```

`tests/alloc.rs`:

```rust
//! Allocation counts are a deterministic proxy for one kind of cost, so they
//! can be asserted in ordinary tests, unlike wall-clock time.

use perflab::{synthetic_text, top_n_fast, top_n_fnv, top_n_naive};
use std::alloc::{GlobalAlloc, Layout, System};
use std::cell::Cell;

struct Counting;

thread_local! {
    static ALLOCS: Cell<usize> = const { Cell::new(0) };
    static BYTES: Cell<usize> = const { Cell::new(0) };
}

// SAFETY: every method forwards to `System` unchanged; the thread-local
// counters are const-initialized `Cell`s, so touching them cannot allocate.
unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        ALLOCS.with(|c| c.set(c.get() + 1));
        BYTES.with(|c| c.set(c.get() + layout.size()));
        // SAFETY: same contract as the caller's.
        unsafe { System.alloc(layout) }
    }

    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        // SAFETY: same contract as the caller's.
        unsafe { System.dealloc(ptr, layout) }
    }

    unsafe fn realloc(&self, ptr: *mut u8, layout: Layout, new_size: usize) -> *mut u8 {
        ALLOCS.with(|c| c.set(c.get() + 1));
        BYTES.with(|c| c.set(c.get() + new_size));
        // SAFETY: same contract as the caller's.
        unsafe { System.realloc(ptr, layout, new_size) }
    }
}

#[global_allocator]
static GLOBAL: Counting = Counting;

/// Runs `f` and returns its result with the number of allocations the
/// **current thread** made while it ran.
fn count_allocs<T>(f: impl FnOnce() -> T) -> (T, usize) {
    let before = ALLOCS.with(Cell::get);
    let out = f();
    (out, ALLOCS.with(Cell::get) - before)
}

#[test]
fn the_counter_sees_a_known_number_of_allocations() {
    let (_, n) = count_allocs(|| {
        let v: Vec<u8> = Vec::with_capacity(10);
        let s = String::from("hello");
        std::hint::black_box((&v, &s));
    });
    assert_eq!(n, 2);
    let (_, none) = count_allocs(|| 1 + 1);
    assert_eq!(none, 0);
}

#[test]
fn all_variants_agree_on_a_realistic_text() {
    let text = synthetic_text(5_000, 300, 7);
    let expected = top_n_naive(&text, 10);
    assert_eq!(top_n_fast(&text, 10), expected);
    assert_eq!(top_n_fnv(&text, 10), expected);
    assert_eq!(expected.len(), 10);
}

#[test]
fn the_naive_version_allocates_per_word_and_the_fast_one_does_not() {
    let text = synthetic_text(20_000, 500, 1);
    let (a, naive) = count_allocs(|| top_n_naive(&text, 10));
    let (b, fast) = count_allocs(|| top_n_fast(&text, 10));
    let (c, fnv) = count_allocs(|| top_n_fnv(&text, 10));
    eprintln!("allocations: naive {naive}, fast {fast}, fnv {fnv}");
    assert_eq!(a, b);
    assert_eq!(a, c);
    assert!(
        naive >= 20_000,
        "naive allocates at least once per word: {naive}"
    );
    // With the capacity hint the map never regrows for this vocabulary:
    // lowered copy, map table, entry vector and result strings only.
    assert!(fast <= 16, "fast should stay near its fixed set: {fast}");
    assert!(fast * 50 < naive, "fast {fast} vs naive {naive}");
    assert!(fnv * 50 < naive, "fnv {fnv} vs naive {naive}");
}

#[test]
fn fast_version_allocation_count_does_not_grow_with_the_input() {
    let small = synthetic_text(10_000, 500, 3);
    let large = synthetic_text(100_000, 500, 3);
    let (_, a_small) = count_allocs(|| top_n_fast(&small, 10));
    let (_, a_large) = count_allocs(|| top_n_fast(&large, 10));
    eprintln!("fast: {a_small} -> {a_large} allocations for 10x the words");
    // Same vocabulary, 10x the words: the number of allocations stays flat.
    assert!(a_large <= a_small + 4, "small {a_small}, large {a_large}");
    let (_, n_small) = count_allocs(|| top_n_naive(&small, 10));
    let (_, n_large) = count_allocs(|| top_n_naive(&large, 10));
    assert!(
        n_large > 5 * n_small,
        "naive should scale with input: {n_small} -> {n_large}"
    );
}
```

`benches/wordfreq.rs`:

```rust
use criterion::{BenchmarkId, Criterion, Throughput, black_box, criterion_group, criterion_main};
use perflab::{synthetic_text, top_n_fast, top_n_fnv, top_n_naive};

fn bench(c: &mut Criterion) {
    let mut group = c.benchmark_group("top_n");
    for words in [1_000usize, 20_000, 200_000] {
        let text = synthetic_text(words, 2_000, 42);
        group.throughput(Throughput::Bytes(text.len() as u64));
        group.bench_with_input(BenchmarkId::new("naive", words), &text, |b, t| {
            b.iter(|| top_n_naive(black_box(t), 10))
        });
        group.bench_with_input(BenchmarkId::new("fast", words), &text, |b, t| {
            b.iter(|| top_n_fast(black_box(t), 10))
        });
        group.bench_with_input(BenchmarkId::new("fnv", words), &text, |b, t| {
            b.iter(|| top_n_fnv(black_box(t), 10))
        });
    }
    group.finish();
}

criterion_group!(benches, bench);
criterion_main!(benches);
```

`examples/profile.rs`:

```rust
//! A small driver for profilers: `profile <naive|fast|fnv|none> [words]`.
use perflab::{synthetic_text, top_n_fast, top_n_fnv, top_n_naive};
use std::hint::black_box;

fn main() {
    let mut args = std::env::args().skip(1);
    let variant = args.next().unwrap_or_else(|| "fast".to_string());
    let words: usize = args.next().and_then(|s| s.parse().ok()).unwrap_or(20_000);
    let text = synthetic_text(words, 2_000, 42);
    let result = match variant.as_str() {
        "naive" => top_n_naive(black_box(&text), 10),
        "fast" => top_n_fast(black_box(&text), 10),
        "fnv" => top_n_fnv(black_box(&text), 10),
        "none" => Vec::new(), // baseline: only builds the input text
        other => {
            eprintln!("unknown variant {other}");
            std::process::exit(2);
        }
    };
    println!("{variant}: top word {:?}", result.first());
}
```

Commands (from the crate root):

```text
cargo test                                  # 4 + 9 + 1 tests (doctest included)
cargo bench                                 # Criterion; results in target/criterion/
cargo build --release --example profile
valgrind --tool=callgrind --callgrind-out-file=out.callgrind \
    target/release/examples/profile fast 20000
callgrind_annotate out.callgrind
```

✓ Checks I ran: `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, `cargo test --locked` (4 allocation tests, 9 equivalence tests including the three property tests, 1 doctest; all pass) and `cargo doc --no-deps` with `RUSTDOCFLAGS="-D warnings"`.

**Mutation checks** (each mutant on a copy of the crate, full test suite, `E` = equivalence tests, `A` = allocation tests):

| # | Mutation | Result |
|---|---|---|
| P1 | `n < entries.len()` becomes `n <= entries.len()` in `best_n` | **survived: equivalent mutant.** When `n == len`, `select_nth_unstable_by(len - 1, ..)` is valid and `truncate(len)` does nothing; the output is the same, only slightly more work |
| P2 | tie-break reversed (word descending) | **caught** (E) |
| P3 | the `n == 0` early return removed | **caught** (E: underflow in `n - 1`) |
| P4 | non-ASCII fallback disabled in `top_n_fast` | **caught** (E: the Unicode example and the property test) |
| P5 | lowercase pass removed (`to_owned` instead of `to_ascii_lowercase`) | **caught** (E, A) |
| P6 | the naive counter adds 2 instead of 1 | **caught** (E, A) |
| P7 | FNV prime replaced by 1 | **caught only after I added the published-vector test** (see below) |
| P8 | FNV offset basis applied on every `write` call instead of only the first | **survived the original suite; caught only after I added the published-vector test.** My reading (not measured): `str`'s `Hash` impl ends with a separate one-byte write, so every key would end up with the same hash; the map still returns correct answers, just slowly |
| P9 | map pre-sizing (`with_capacity(256)`) removed | **caught only after I added the `fast <= 16` assertion** |
| P10 | `entries.truncate(n)` removed | **caught** (E, A) |
| P11 | empty-word filter disabled in `top_n_fnv` | **caught** (E, A) |

Totals: 11 mutants, 10 caught, 1 equivalent (P1). The first run of the suite let **three mutants survive**: P7 and P8 (the hasher still produced *consistent* hashes, just poor ones, and consistency is all the map needs, so output-equality tests could never see a bad hash function) and P9 (removing a capacity hint changes only performance, so no output test notices; ✓ the fast version then made 22 allocations instead of 15). I closed them with a test against the published FNV-1a 64-bit vectors (`"a"` → `0xaf63dc4c8601ec8c`, `"foobar"` → `0x85944171f73967e8`, and chunked writes equal one write) and by tightening the allocation test to `fast <= 16`. That is the lesson's theme in miniature: **performance properties need their own tests**, because correctness tests are blind to them.

**Design notes.**

- The three functions share `best_n`, so ordering is defined in one place (`cmp_entries`: count descending, then word ascending). Equality tests therefore compare results whose tie order does not depend on `HashMap` iteration order.
- `top_n_fast` and `top_n_fnv` return `String`s only for the `n` winners, so the cost of copying is `O(n)` rather than `O(distinct words)`.
- The `Fnv1a` hasher uses `self.0 == 0` to mean "not started", because `BuildHasherDefault` builds hashers with `Default`. Consequently a hash state that happens to equal 0 mid-stream would be treated as unstarted by the next `write`. The result is still deterministic per key sequence, so the map stays correct, but it is a wart; a production hasher would store the offset basis in `Default`.
- The counting allocator is `unsafe`, justified in `SAFETY:` comments, and exists only in the test binary.
- Not handled: Unicode word segmentation (words are alphabetic runs, not linguistic words), streaming input, parallel counting, memory-mapped files, non-`Send` considerations, a hasher with a keyed random seed, and any measurement of compile time or binary size.

</details>

---

## 15. Key takeaways

- A benchmark is an experiment: it can be wrong in ways the number does not reveal. Compute the implied rate, check the outputs are equal, and keep setup out of the measurement (✓ 33 ns for 50 million iterations was a removed loop; the clock itself costs about 29 ns).
- Report spread, not a single number: ✓ identical code differed by about 5% between runs on this machine, so few-percent "changes" are not findings.
- Use deterministic proxies where wall-clock time is noisy: ✓ instruction counts (naive 13.97 M, fast 8.36 M, fnv 5.65 M net) and allocation counts (✓ 20,010 → 15), and test the latter in plain `cargo test`.
- Profile the code under test, not your setup (✓ the input generator was the biggest entry in the `fast` profile).
- Change algorithms and allocation before flags (✓ the best flag saved about 10%, the structural change 40% to 60% of instructions), and never judge a debug build (✓ 4.2× to 8.1×).
- A faster hasher is a security decision (hash flooding), not just a speed knob.
- Performance properties need their own tests; output-equality tests were blind to three of the eleven mutants (✓ P7, P8, P9).

**Connections.** [Lesson 40](40_property_testing_fuzzing.md)'s properties and generators prove the optimized versions equal the simple one; [Lesson 39](39_testing.md)'s mutation method found the performance-only gaps; [Lesson 36](36_unsafe_raw_pointers.md) explains the `unsafe impl GlobalAlloc` used to count allocations; [Lesson 16](16_collections.md) covers `HashMap`, SipHash and capacity; [Lesson 33](33_modules_api_design_rustdoc.md) gives the documented, tested public API shape. Next: Lesson 42 continues Module 13.

---

## 16. References

- Criterion.rs book (benchmark groups, throughput, baselines, command-line options): <https://bheisler.github.io/criterion.rs/book/>
- The Rust Performance Book (profiling, allocation, hashing, build configuration): <https://nnethercote.github.io/perf-book/>
- Cargo reference, "Profiles" (`opt-level`, `lto`, `codegen-units`, `panic`, `debug`): <https://doc.rust-lang.org/cargo/reference/profiles.html>
- `std::hint::black_box`: <https://doc.rust-lang.org/std/hint/fn.black_box.html>
- Valgrind manual, Callgrind: <https://valgrind.org/docs/manual/cl-manual.html>
- `std::alloc::GlobalAlloc`: <https://doc.rust-lang.org/std/alloc/trait.GlobalAlloc.html>
- FNV hash description and test vectors: <http://www.isthe.com/chongo/tech/comp/fnv/>
- Linux `perf` documentation and `cargo flamegraph` on crates.io: <https://perf.wiki.kernel.org/>, <https://crates.io/crates/flamegraph>

The URLs above are the standard documentation locations; I did not open them during this session (the sandbox cannot reach the documentation sites), so if a link has moved, search for the page title. Unchecked or from memory: that built-in `#[bench]` is nightly-only; that `black_box` is documented as best-effort; every description of `perf`, `cargo flamegraph`, `hyperfine`, `heaptrack` and DHAT (none installed, none run), including the frame-pointer advice; that `bench` inherits release profile settings; Criterion's baseline and command-line options other than `--warm-up-time` and `--measurement-time`; the description of SipHash 1-3 and `RandomState` as the default (the callgrind output did show `Sip13Rounds` and `RandomState` symbols); `ahash` as an example of a keyed fast hasher; the claim that `panic = "abort"` disables `catch_unwind` and destructors on panic; the FNV test vectors (I computed them independently in Python and the crate's test passes, but did not open the reference page); and the statement that code size or compile-time costs of `lto` and `codegen-units = 1` (I did not time compilation). The measurements marked ✓ are single sessions on one shared virtual machine: Criterion medians come from one run of `--warm-up-time 1 --measurement-time 3`, instruction counts come from one callgrind run each (deterministic, but only for this input and these exact toolchain and path settings), and the 5% run-to-run difference compares two sessions. Exercise answers 3, 4, 7 and 8 combine ✓-run facts with reasoning from memory. Everything marked ✓ was run on this machine's `rustc 1.97.0` with `criterion 0.5.1`, `proptest 1.12.0` and `valgrind 3.22.0`.
