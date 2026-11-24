---
title: "Lesson 42 — Performance engineering: cache locality, SIMD, allocations, binary size"
---

# Lesson 42 — Performance engineering: cache locality, SIMD, allocations, binary size

**Module 13 · Testing, performance and tooling** · Status: generated · Edition used in examples: **2024** · External dependencies: `proptest` (dev-dependency only)

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable` and `proptest 1.12.0`, on x86-64 Linux, 2 cores, Intel Xeon @ 2.10 GHz. As before, ✓ marks outputs and measurements I actually ran; unmarked statements are documented behavior I did not run. **Every timing is from one shared, virtualized machine, taken with a hand-written harness (minimum or median of several runs), and I give the range seen across two or three runs wherever the runs differed.** Lesson 41 showed that identical code can vary by several percent here; some numbers below varied more. What the machine reports about its caches (`lscpu`): L1 data 96 KiB across 2 cores (48 KiB each), L2 4 MiB across 2 cores (2 MiB each), L3 260 MiB (a virtualized figure I do not trust). Not available, so described from memory only: `perf` and hardware counters (cache-miss counts), `cargo asm`, `cargo bloat`, `std::simd` (nightly only), AArch64 or any other CPU architecture, and 32-bit targets. I could not open the documentation sites, so rules stated without ✓ are from memory; the closing paragraph lists them.

Builds on [Lesson 41](41_benchmarking_profiling.md) (how to measure, `black_box`, allocation counting, Cargo profile knobs), [Lesson 40](40_property_testing_fuzzing.md) (properties that prove optimized code equals simple code), [Lesson 39](39_testing.md) (mutation checks), [Lesson 36](36_unsafe_raw_pointers.md) (`unsafe` blocks and `SAFETY:` comments, used here for intrinsics and for a counting allocator), [Lesson 16](16_collections.md) (`Vec`, capacity and growth), [Lesson 09](09_memory_layout.md) (size, alignment and padding, which §4.1 extends) and [Lesson 08](08_stack_heap_drop.md) (stack, heap and allocation). Lesson 43 is next; it appears here only as plain text.

---

## 1. Why this lesson exists

Lesson 41 taught how to measure. This lesson is about what usually turns out to matter, in rough order of how often it is the answer:

1. **Memory.** A modern CPU executes a few instructions per nanosecond but a trip to main memory costs on the order of a hundred nanoseconds (✓ 155 to 162 ns per dependent load in §2). How your data is laid out and in what order you touch it often matters more than any instruction-level trick.
2. **Doing several things per instruction.** SIMD (vector) instructions process 16, 32 or 64 bytes at a time. The compiler does some of this automatically; knowing when it can and cannot is the skill.
3. **Allocation.** Every allocation costs time and every allocation in a hot loop is a candidate to remove (Lesson 41 counted 20,010 → 15).
4. **Size.** A smaller binary starts faster, fits in smaller devices and has less to ship; it is a different goal and sometimes a trade against speed.

The running example is a crate, `tunelab`, with a simple reference implementation and one or more optimized variants for each theme, all tested to give the same answers. Everything marked ✓ comes from it or from the small programs shown in the text.

---

## 2. The memory hierarchy, measured ✓

Memory is a ladder of caches: small and fast near the core (L1, L2), larger and slower further out (L3), then main memory. Data moves between levels in fixed-size **cache lines** (64 bytes on x86-64 CPUs, from memory; I did not query it here). Reading one byte brings its whole line; reading the next byte on that line is then almost free.

To see the ladder, make every load **depend on the previous one**, so the CPU cannot overlap them and each hop costs the full latency of whichever level holds the data: follow `next[at]` repeatedly through a table of `u32`. Two tables of the same size: `cycle_sequential` (hop to the next index; the hardware prefetcher guesses this) and `cycle_random` (a random order through all the entries; no pattern to guess).

✓ Nanoseconds per hop (minimum of 5 timed runs of 4,000,000 hops; ranges are across three runs):

| Table size | Sequential | Random |
|---:|---:|---:|
| 4 KiB | 1.4 | 1.4 |
| 32 KiB | 1.4 to 1.5 | 1.5 |
| 256 KiB | 1.3 to 1.5 | 4.0 to 4.3 |
| 2 MiB | 1.3 to 1.6 | 16.2 to 17.2 |
| 16 MiB | 1.3 to 1.6 | 56.6 to 70.7 |
| 128 MiB | 1.3 to 1.7 | 154.8 to 162.0 |

Reading it:

- **Sequential access is flat at about 1.3 to 1.7 ns per hop for every size.** The prefetcher sees the pattern and the data arrives before it is needed.
- **Random access steps up each time the table outgrows a level.** Up to 32 KiB it fits the 48 KiB L1 (same as sequential). At 256 KiB it no longer fits L1 (about 4 ns). At 2 MiB it is at or past the per-core L2 (about 16 ns). At 16 MiB and 128 MiB it is going to L3 and then to memory (57 to 71 ns, then about 155 to 162 ns). The 128 MiB random access is about **110 times slower per hop** than the sequential one.
- I did not isolate every contributor. For example, part of the jump at large sizes may be address-translation (TLB) misses with 4 KiB pages (`getconf PAGESIZE` printed 4096), not only cache misses; separating them needs hardware counters I did not have. The boundaries I named come from the sizes `lscpu` reports for this machine and need not match yours.
- The same table rewritten in a program with no dependency between loads would be faster, since the CPU can have many loads in flight at once. The chase isolates **latency**; real programs sit somewhere between latency-bound and bandwidth-bound.

---

## 3. Access order: walk memory the way it is laid out

A `Vec<u32>` holding an `n` by `n` matrix in **row-major** order stores element `(r, c)` at index `r * n + c`: the elements of a row are adjacent. Summing along rows reads memory in order. Summing down columns jumps `n * 4` bytes each step, touching a new cache line each time.

`sum_row_major` and `sum_col_major` return the same number (a tested property) and differ only in loop order.

✓ Minimum time of 9 runs, per range over three runs of the measurement program:

| Side `n` | Matrix size | Row-major | Column-major | Column / row |
|---:|---:|---:|---:|---:|
| 256 | 256 KiB | 10.8 to 14.2 µs | 47 to 62 µs | 3.3× to 5.6× |
| 1024 | 4 MiB | 239 to 250 µs | 7.7 to 9.0 ms | 32× to 37× |
| 4096 | 64 MiB | 3.2 to 4.7 ms | 141 to 147 ms | 30× to 46× |

Same instructions, same answer, a factor of **30 or more** from loop order alone once the matrix no longer fits in cache. Even at 256 KiB (L2-resident) it is 3 to 6 times. The spread across runs is wide (the 4096 row-major time varied by 45%), so quote the order of magnitude, not the digits.

Rules of thumb that follow:

- Make the **innermost loop** run along the contiguous dimension.
- Iterate with iterators or slices rather than index arithmetic where you can; the compiler can more easily prove bounds and vectorize (§5).
- If you need both orders, **blocking/tiling** (process the matrix in small square tiles that fit in cache) is the standard fix; I did not implement or measure it here.

---

## 4. Layout: array of structs vs struct of arrays

### 4.1 What a struct occupies ✓

```rust
use std::mem::{align_of, size_of};

#[allow(dead_code)]
struct Reordered {
    a: u8,
    b: u64,
    c: u8,
}

#[allow(dead_code)]
#[repr(C)]
struct CLayout {
    a: u8,
    b: u64,
    c: u8,
}

#[allow(dead_code)]
#[repr(C)]
struct CSorted {
    b: u64,
    a: u8,
    c: u8,
}

fn main() {
    println!("Reordered (default repr): size {} align {}", size_of::<Reordered>(), align_of::<Reordered>());
    println!("CLayout   (repr(C), u8,u64,u8): size {}", size_of::<CLayout>());
    println!("CSorted   (repr(C), u64,u8,u8): size {}", size_of::<CSorted>());
    println!("Option<Box<u64>> {} Box<u64> {} Option<u64> {} Option<&u8> {}", size_of::<Option<Box<u64>>>(), size_of::<Box<u64>>(), size_of::<Option<u64>>(), size_of::<Option<&u8>>());
    println!("String {} Vec<u8> {} &str {} Box<str> {}", size_of::<String>(), size_of::<Vec<u8>>(), size_of::<&str>(), size_of::<Box<str>>());
}
```

✓ Output: `Reordered` size 16, align 8; `CLayout` size **24**; `CSorted` size **16**; `Option<Box<u64>>` 8 (same as `Box<u64>`: the null pointer encodes `None`, a **niche**), `Option<u64>` 16 (a tag plus padding), `Option<&u8>` 8; `String` and `Vec<u8>` 24, `&str` and `Box<str>` 16.

Meaning: each field must be aligned to its own size, so C layout inserts padding (`u8` then `u64` wastes 7 bytes, and the struct is padded at the end to a multiple of its alignment). Plain `struct` in Rust has **no guaranteed layout**: the compiler is free to reorder fields to reduce padding, which is why `Reordered` is 16 and not 24. Use `#[repr(C)]` only when you need a defined layout (FFI, Lesson 37), and then order fields from largest alignment to smallest yourself.

### 4.2 Why layout matters for speed ✓

`Particle` in `tunelab` is a 64-byte record (✓ `size_of::<Particle>() == 64`, asserted by a test): position, velocity, an id, a `u32` weight and 4 bytes of padding. The query "sum of all weights" needs 4 of its 64 bytes.

- **Array of structs (AoS):** `Vec<Particle>`. The weights are 64 bytes apart; scanning them reads every cache line of the whole array: 64 MB for 1,000,000 particles to use 4 MB of information.
- **Struct of arrays (SoA):** `ParticlesSoa` with a separate `Vec<u32>` for weights. The query reads only the 4 MB weight array, and the sum is a plain sum of contiguous integers (vectorizable, §5).

✓ 1,000,000 particles, minimum of 15 runs: **AoS 2.38 to 2.56 ms; SoA 0.21 to 0.25 ms; ratio 10.3× to 11.3×** (three runs). The ratio is close to the 16 to 1 byte ratio (64 bytes read per 4 needed), less because some time is fixed overhead and the AoS scan is bandwidth-bound while the SoA one is partly compute-bound.

When AoS wins: a loop that uses most of the fields of each record (all of position and velocity per step) is faster with the record in one place, and adding or removing whole records is simpler. SoA wins when a pass touches **a few fields of many records**. The choice is a measurement, not a rule; I measured only the weight-sum query, not an update of all fields.

---

## 5. SIMD and auto-vectorization

### 5.1 What the compiler does for you ✓

A SIMD ("single instruction, multiple data") instruction applies one operation to a whole vector of values. On `x86_64` the baseline target guarantees **SSE2** (128-bit registers: four `f32`, four `u32`, sixteen bytes). LLVM's **auto-vectorizer** rewrites simple loops to use these when it can prove the result is unchanged. For floating point, it often cannot:

```rust
#[inline(never)]
pub fn sum_seq(xs: &[f32]) -> f32 {
    let mut s = 0.0f32;
    for &x in xs {
        s += x;
    }
    s
}

#[inline(never)]
pub fn sum_int(xs: &[u32]) -> u32 {
    xs.iter().fold(0u32, |a, &x| a.wrapping_add(x))
}

fn main() {
    let xs: Vec<f32> = (0..1000).map(|i| i as f32).collect();
    let us: Vec<u32> = (0..1000).collect();
    println!("{} {}", sum_seq(std::hint::black_box(&xs)), sum_int(std::hint::black_box(&us)));
}
```

✓ I compiled this with `rustc -O --emit asm` and counted instructions in each function. `sum_seq` contains **9 `addss`** (scalar single-precision adds) and no `addps` (the packed form). `sum_int` contains **5 `paddd`** (packed 32-bit integer adds). With `-C target-cpu=x86-64-v3` (AVX2 allowed), `sum_int` has 13 `vpaddd`.

Why: floating-point addition is **not associative** (`(a + b) + c` can differ from `a + (b + c)` in the last bits), and Rust, like C without `-ffast-math`, requires the exact left-to-right result. Vectorizing the sum means adding in a different order. Integer addition is associative (wrapping included), so the compiler may vectorize it freely. Rust has no fast-math flag for stable code; to allow the reordering you must write it down yourself.

### 5.2 Writing the order down ✓

`sum_f32_lanes` keeps eight independent accumulators (lane *i* sums elements *i*, *i + 8*, ...) and adds the lanes at the end. That is a *different but fixed* order, so the compiler may use packed adds.

✓ Asm of `sum_f32_lanes`: **10 `addps`**. ✓ Time to sum 16,384 `f32`s (minimum of 31 runs, three runs): sequential **8.2 to 10.0 µs (0.50 to 0.61 ns per element)**, lanes **1.09 to 1.29 µs (0.07 to 0.08 ns)**, i.e. about **7.7× faster**; `sum_u32_wrapping` took 0.79 to 1.06 µs.

The price: the answer changes. ✓ For this data the sequential sum printed `8065.5366` and the lane sum `8065.536`. A tested example (`float_sums_are_not_associative`) makes it stark: `16777216.0` followed by fifteen `1.0`s sums to `16777216.0` sequentially (each `+1` is lost, since 2²⁴ + 1 is not representable in `f32`) and to `16777230.0` with lanes (seven lanes collect their ones before meeting the big value). Neither is "wrong"; the lane version is often closer to the true sum, but it is not the same, and anything that compares results exactly (tests, consensus-critical code, reproducible science) must know which order it has.

### 5.3 Helping the compiler

- Prefer **iterators and slices** over index loops: no bounds check per element, and `chunks_exact` gives the compiler fixed-size blocks.
- Keep loop bodies simple: no early exits, no calls the optimizer cannot see through, no data-dependent branches that change what is loaded.
- Use integer or reorder-friendly forms (as above) for reductions.
- Check, do not assume: look for packed instructions in the output of `--emit asm` (or `cargo asm`, from memory, not run), and measure.

Vectorized does not mean fast: ✓ the compiler **did** vectorize `count_byte_scalar` (`filter(..).count()`; the asm has `pcmpeqb` and `paddq`) and it still ran at only **2.4 to 2.9 GiB/s** (§5.4), because widening each byte's comparison result to 64-bit counters does a lot of work per byte.

### 5.4 Three ways to count a byte ✓

`count_byte` counts occurrences of one byte in a slice:

- `count_byte_scalar`: the obvious `iter().filter(..).count()`.
- `count_byte_swar`: "SIMD within a register": eight bytes at a time in an ordinary `u64`, using an exact zero-byte test. Safe, portable, no intrinsics.
- `count_byte_sse2`: explicit intrinsics from `std::arch::x86_64`: compare 16 bytes, extract a bit mask, count its ones. `unsafe`, with a `SAFETY:` comment; x86-64 only.

All three agree on every length around the chunk sizes (0 to 69) and on arbitrary bytes (property tests). ✓ Throughput on 1 MiB, minimum of 31 runs:

| Build | scalar | SWAR | SSE2 intrinsics |
|---|---:|---:|---:|
| default x86-64 target (SSE2) | 2.4 to 2.9 GiB/s | 8.5 to 9.7 GiB/s | 8.4 to 13.2 GiB/s |
| `-C target-cpu=x86-64-v3` (AVX2) | 9.9 GiB/s | 30.7 GiB/s | 35.2 GiB/s |

Two things stand out. (1) **The portable SWAR version is about 3.5 to 4× faster than the obvious code and close to the intrinsics version**, with no `unsafe`. (2) **Compiling for a newer CPU level made all three about 3× faster** (the compiler used 256-bit vectors); the intrinsics function got faster too because the same SSE2 intrinsics are re-encoded for AVX. The two AVX2 runs agreed to within a few thousandths; the baseline runs varied more (SSE2: 8.4 GiB/s in one run, 11.7 in another, 12.8 to 13.2 in three more), so treat the SSE2-versus-SWAR ordering in the baseline row as unresolved.

### 5.5 Choosing the CPU level

| Approach | Gain | Cost |
|---|---|---|
| Default target (`x86_64` baseline, SSE2) | none extra | runs on every x86-64 CPU |
| `-C target-cpu=x86-64-v3` or `native` at build time | ✓ about 3× here | the binary may crash with "illegal instruction" on older CPUs. ✓ `native` also failed to run under valgrind here (AVX-512 not emulated by this version) |
| **Runtime detection**: `is_x86_feature_detected!("avx2")` and a function compiled with `#[target_feature(enable = "avx2")]` | fast path when available, one portable binary | `unsafe` call (you promise the feature exists), a branch per call, and you maintain two paths |
| `std::simd` (portable SIMD) | one source for many targets | nightly only at the time of writing (from memory; I did not check the tracking status), so not usable on stable |
| A vetted crate (for example `memchr` for byte search; not run here) | tested, dispatching implementations | a dependency |

The runtime-detection pattern, compiled and run ✓ (`avx2 detected at run time: true` on this machine, results equal to the portable version):

```rust
#[cfg(target_arch = "x86_64")]
fn sum_bytes(data: &[u8]) -> u64 {
    if is_x86_feature_detected!("avx2") {
        // SAFETY: we just checked that this CPU supports AVX2.
        unsafe { sum_bytes_avx2(data) }
    } else {
        sum_bytes_portable(data)
    }
}

#[cfg(not(target_arch = "x86_64"))]
fn sum_bytes(data: &[u8]) -> u64 {
    sum_bytes_portable(data)
}

fn sum_bytes_portable(data: &[u8]) -> u64 {
    data.iter().map(|&b| u64::from(b)).sum()
}

/// The same loop, but compiled *as if* AVX2 were available, so the compiler
/// may use 256-bit vectors inside this function only.
#[cfg(target_arch = "x86_64")]
#[target_feature(enable = "avx2")]
unsafe fn sum_bytes_avx2(data: &[u8]) -> u64 {
    data.iter().map(|&b| u64::from(b)).sum()
}

fn main() {
    let data: Vec<u8> = (0..1000u32).map(|i| (i % 251) as u8).collect();
    assert_eq!(sum_bytes(&data), sum_bytes_portable(&data));
    println!("sum = {}", sum_bytes(&data));
    #[cfg(target_arch = "x86_64")]
    println!("avx2 detected at run time: {}", is_x86_feature_detected!("avx2"));
}
```

The function is `unsafe` because calling a `target_feature` function on a CPU without the feature is undefined behavior; the `SAFETY:` comment must point to the check that makes the call sound. I did not benchmark this particular function.

Rules for explicit SIMD: **have a scalar reference version and test equivalence** (`tunelab` does, with properties and every length near the chunk size); keep the `unsafe` block as small as possible; handle the tail (the last `len % 16` bytes) separately; and measure, because the compiler's own vectorization of a simple loop may already be close.

---

## 6. Allocation reuse ✓

`join_u32_alloc` builds a `String` per number and a `Vec<String>`, then joins. `join_u32_into(values, &mut out)` writes into a caller-supplied `String` with `write!`, after `out.clear()`. `clear` sets the length to zero but **keeps the capacity**, so later calls only allocate if the output is longer than any before.

✓ Allocation counts (a counting `GlobalAlloc` in the test binary, as in Lesson 41; 1,000 numbers): the allocating version made **1,002 allocations**; the reusing version made **10** on its first call (the buffer doubling its way up) and **0 on each of 10 later calls**, asserted in `tests/alloc.rs`. ✓ Time (minimum of 31 runs, five runs): allocating **24.6 to 28.3 µs**, reusing **12.5 to 20.5 µs**, a ratio of **1.4× to 2.1×** (the reuse timing was the unstable one).

Design guidance:

- Give hot functions a `&mut` output parameter (or a method that appends to a caller's buffer) in addition to a convenient version that returns a new value. The standard library does this (`read_line(&mut String)`, `Vec::extend`, `fmt::Write`).
- `Vec::with_capacity`/`String::with_capacity` when the size is known or can be estimated: avoids the doubling reallocations (Lesson 41 ✓: dropping a capacity hint from a map made 22 allocations instead of 15).
- Reuse buffers across loop iterations: `buf.clear()`, not `buf = Vec::new()` (✓ a mutant that replaced `clear()` with `*out = String::new()` was caught by the allocation test).
- **Reliability caveat:** a reused buffer retains the largest capacity it ever reached. One enormous input can pin a lot of memory for the life of the program; for long-running services consider `shrink_to` after outliers, or cap the size you keep.
- Do not allocate inside the loop when a stack array or a slice of an existing buffer will do (borrow, as in `top_n_fast`'s `&str` keys).
- Avoid `format!`/`to_string()` in hot paths when `write!` into a reusable buffer works (✓ a mutant in §14 that used `to_string` per number kept the output identical and allocated every time).

---

## 7. Binary size ✓

Size matters for embedded targets (Lesson 38), WebAssembly, containers, and startup. I built a small word-counting program (a `HashMap`, `String`s, `println!`) with the settings below and measured the size of the resulting file. These sizes are for this program and toolchain on Linux x86-64.

| Configuration | File size (bytes) |
|---|---:|
| `cargo build` (dev profile) | 5,079,584 |
| `cargo build --release`, `opt-level = 0` | 585,400 |
| `cargo build --release`, default | 495,408 |
| + `strip = true` | 381,600 |
| + `lto = "fat"` | 346,872 |
| + `codegen-units = 1` | 346,504 |
| + `panic = "abort"` | 341,584 |
| `opt-level = "s"` with all of the above | 322,504 |
| `opt-level = "z"` with all of the above | 320,440 |

Reading it:

- Release versus dev is a factor of about **10** here, mostly debug information and unoptimized code.
- **Stripping symbols** removed 23% from the default release binary (495,408 → 381,600) at no runtime cost.
- LTO saved another 9% (381,600 → 346,872). `codegen-units = 1` saved almost nothing on top, and `panic = "abort"` saved 1.4% (346,504 → 341,584).
- Optimizing for size (`s` or `z`) saved a further 5 to 6%, and Lesson 41 ✓ measured the speed cost of `z` (43% to 93% more instructions on that workload). Whether that is a good trade depends on the product.
- What remains is mostly the standard library (the formatting machinery and hash map code), which is why real savings below this point require `#![no_std]` (Lesson 38: ✓ a stripped freestanding binary was 9,864 bytes against 4,358,728 for std hello at `-O`) or dropping features such as `println!` and panic formatting.

Other levers, not measured here (from memory): avoiding heavy generic instantiation (every `T` is a separate copy of the code), `cargo bloat` to see which functions are large, trimming dependency features, `opt-level = "s"` for most crates with `opt-level = 3` for a hot one, and `wasm-opt` for WebAssembly. Debug information and symbol names can leak internal names and paths; stripping them is also a small hygiene step for distributed binaries.

---

## 8. A method

1. **Know the question.** Throughput, latency, memory, size, or startup? The levers differ.
2. **Measure first** (Lesson 41): a release build, a realistic input, spread not a single number, deterministic proxies where wall-clock is noisy.
3. **Check the algorithm and data structure**, then **the memory behavior** (order, layout, allocation), then **vectorization**, then **flags**.
4. **Keep the simple version** as a reference and test equality (properties, Lesson 40). Optimized code that is wrong is a bug.
5. **Change one thing, re-measure, keep the result only if it helped.** Several ideas in this lesson (SIMD, SoA) only help when the data is large enough or the access pattern matches.
6. **Write down what you measured and on what machine.** Numbers do not transfer between machines.

---

## 9. Performance, reliability, security

- **`unsafe` for speed needs the same discipline as any `unsafe`** (Lesson 36): a minimal block, a `SAFETY:` comment, a safe wrapper, and tests (✓ here: every length around the 16-byte chunk, arbitrary bytes, and mutants that break the comparison and the tail handling). A wrong tail loop that reads past the end of a slice is memory unsafety, not just a wrong count.
- **Reproducibility:** reordering floating-point arithmetic changes results (✓ §5.2). If results must be bit-identical across machines or versions, document the order and test it.
- **Portability:** `-C target-cpu=native` binaries can fail on other machines with an illegal-instruction error; use runtime detection or a fixed level (such as x86-64-v2) for anything you distribute.
- **Denial of service:** performance cliffs are attack surface. An input shape that defeats the cache or the hasher (Lesson 41) can be crafted. Bound input sizes and allocation (Lesson 40).
- **Memory retention:** reused buffers can hold on to peak capacity (§6).
- **Timing side channels:** for secret-dependent code (cryptography), "faster on some inputs" is a vulnerability; branches and early exits that depend on secrets leak (I do not cover constant-time programming in this lesson).
- **Benchmarks lie by omission:** a micro-benchmark with a hot cache says nothing about the program that runs it once after a cold start.

---

## 10. Common mistakes

1. Optimizing without measuring, or measuring a debug build (Lesson 41).
2. Looping down columns of a row-major array (✓ 30× or worse).
3. Using AoS for a query that touches one field of millions of records (✓ about 10×), or SoA for code that always uses every field of one record.
4. Assuming the compiler will vectorize a floating-point reduction (✓ it does not), or assuming vectorized code is fast (✓ 2.4 to 2.9 GiB/s scalar versus 8.5 to 9.7 with SWAR).
5. Reaching for `unsafe` intrinsics before trying safe forms (iterator chunks, SWAR).
6. Using `-C target-cpu=native` in a build that ships to other machines.
7. Allocating inside a hot loop when a reusable buffer would do (✓ 1,002 allocations versus 0).
8. Forgetting that reused buffers keep their peak capacity.
9. Comparing floating-point results from different summation orders with `==`.
10. Treating binary size and speed as the same goal (`opt-level = "z"` is smaller and slower).

---

## 11. Exercises

Answers are collapsed. Try each before opening it.

1. **(Warm-up)** Using the pointer-chase table, explain in two sentences why the random access time rises in steps while sequential access stays flat.
2. **(Warm-up)** Why can LLVM vectorize `sum_u32_wrapping` but not `sum_f32_sequential`?
3. **(Easy)** Write `column_sums(data: &[u32], n: usize) -> Vec<u64>` for an `n` by `n` row-major matrix that returns the sum of each column but walks memory in row order.
4. **(Medium)** `Particle` is 64 bytes. You change `weight` from `u32` to `u64` and add a `u64` field. What happens to `size_of::<Particle>()` and to the AoS weight query?
5. **(Medium)** When would you prefer AoS to SoA? Give two situations.
6. **(Medium-hard)** In the runtime-detection example, why is `sum_bytes_avx2` an `unsafe fn`, and what must the `SAFETY:` comment at the call site justify?
7. **(Hard)** You must shrink a command-line tool's binary from 495 KB toward 320 KB without hurting speed on its main workload. Order the steps from the table and say which you would test for speed.
8. **(Hard)** Design the API for a hot `render` function that is called a million times a second so that it does not allocate, and say what can go wrong with the design.

<details markdown="1">
<summary>Answers</summary>

1. Sequential access is predicted by the hardware prefetcher, so data arrives before it is needed and every size costs about the same (✓ 1.3 to 1.7 ns). Random access has no pattern to prefetch and every hop waits for the memory level that holds the data, so the time steps up each time the table outgrows L1, L2 and L3 (✓ about 1.5, 4, 16, 57 to 71, then 155 to 162 ns).
2. Integer addition is associative, so splitting the sum across vector lanes and combining at the end gives the same result; the compiler may reorder it (✓ `paddd` in the asm). Floating-point addition is not associative, so the compiler must keep the left-to-right order, which is a dependency chain it cannot split (✓ only `addss` in the asm).
3. Accumulate into one row-sized output while scanning rows in order:

   ```rust
   fn column_sums(data: &[u32], n: usize) -> Vec<u64> {
       assert_eq!(data.len(), n * n, "expected an n by n matrix");
       let mut sums = vec![0u64; n];
       for row in data.chunks_exact(n.max(1)).take(n) {
           for (s, &x) in sums.iter_mut().zip(row) {
               *s += u64::from(x);
           }
       }
       sums
   }

   fn main() {
       let data: Vec<u32> = (0..9).collect(); // 3x3: 0 1 2 / 3 4 5 / 6 7 8
       assert_eq!(column_sums(&data, 3), vec![9, 12, 15]);
       assert_eq!(column_sums(&[], 0), Vec::<u64>::new());
       println!("ok");
   }
   ```

   The inner loop touches `sums` and one row, both contiguous; the column-by-column version would jump `n * 4` bytes per step (✓ 30× or worse at 4096).
4. Remove the old 4-byte padding field first, since it no longer applies. The fields are `pos` 24 + `vel` 24 + `id` 8 + `weight` 8 + the new `u64` 8 = **72 bytes**, already a multiple of the alignment 8. The record no longer fits in one 64-byte cache line (and records now straddle lines), and the AoS weight query reads 72 bytes per particle to use 8, so it touches about 12% more memory per particle than before; a SoA `Vec<u64>` of weights costs 8 bytes per particle. (Check with `size_of` and a test, as `tunelab` does for 64 bytes.)
5. (a) A loop that uses most fields of each record, such as a physics step updating position from velocity, gets all it needs from one or two cache lines. (b) Code that inserts, removes or moves whole records (sorting by key, queues) is simpler and cheaper with one contiguous record per item. Also, when the data set is small enough to stay in cache, the difference vanishes, so measure.
6. A function with `#[target_feature(enable = "avx2")]` may contain instructions that cause an illegal-instruction fault (undefined behavior in Rust's model) on a CPU without AVX2, and the compiler cannot check that for a call. The `SAFETY:` comment must say that the call happens only after `is_x86_feature_detected!("avx2")` returned true on this same CPU (✓ the check is the line above the call).
7. Free first: `strip = true` (✓ 495,408 → 381,600, no runtime cost). Then LTO (✓ → 346,872; it also tends to speed code up, Lesson 41 ✓ about 10% fewer instructions in that workload). `codegen-units = 1` (✓ almost nothing here, longer builds). `panic = "abort"` (✓ → 341,584, but changes panic behavior). Last, `opt-level = "s"` or `"z"` (✓ → 322,504 / 320,440), the only step that may hurt speed (Lesson 41 ✓ `z` cost 43% to 93% more instructions), so benchmark the main workload with it and with `s`; the other steps need a speed check only to confirm nothing regressed.
8. Let the caller own the buffer: `fn render(&self, out: &mut String)` that does not `clear()` (the caller decides) or does (document it), so a loop can reuse one `String` (✓ 10 allocations on the first call, 0 afterwards). Provide a convenience `fn render_to_string(&self) -> String` that creates a buffer and calls the first. Things that can go wrong: forgetting that `clear()` keeps capacity so memory is pinned after a huge output (shrink after outliers); sharing the buffer between threads (use one per thread, or pass it in); and appending to a dirty buffer when the caller forgot to clear. Test the zero-allocation property with a counting allocator (✓ `reused_buffer_stops_allocating_after_warm_up`).

</details>

---

## 12. Debugging challenge

A developer vectorized a float sum with eight lanes and wrote this test to be sure it matches the original. It fails on most runs even though "the lanes are just a faster way to add". Find the real problem and fix the test (not the function).

```rust
fn sum_f32_sequential(xs: &[f32]) -> f32 {
    let mut s = 0.0f32;
    for &x in xs {
        s += x;
    }
    s
}

fn sum_f32_lanes(xs: &[f32]) -> f32 {
    let mut lanes = [0.0f32; 8];
    let mut chunks = xs.chunks_exact(8);
    for chunk in &mut chunks {
        for (acc, &x) in lanes.iter_mut().zip(chunk) {
            *acc += x;
        }
    }
    let mut s = 0.0f32;
    for l in lanes {
        s += l;
    }
    for &x in chunks.remainder() {
        s += x;
    }
    s
}

fn main() {
    // A deterministic stream of floats in [0, 1).
    let mut state = 12345u64;
    let xs: Vec<f32> = (0..10_000)
        .map(|_| {
            state = state.wrapping_mul(6364136223846793005).wrapping_add(1442695040888963407);
            (state >> 40) as f32 / 16_777_216.0
        })
        .collect();
    assert_eq!(sum_f32_lanes(&xs), sum_f32_sequential(&xs));
    println!("ok");
}
```

<details markdown="1">
<summary>Solution</summary>

Nothing is wrong with `sum_f32_lanes`. The test is wrong: it demands **bit-for-bit equality** between two sums that add the numbers in different orders, and floating-point addition is not associative (§5.2). Each addition rounds, so the two orders accumulate different rounding errors; with 10,000 values in [0, 1) the results differ in the last digits. (✓ Running the failing program: `left: 5029.318`, `right: 5029.3247`. The same effect appears in the lesson's own data: `8065.5366` against `8065.536`.)

Fix the test, according to what you actually promise:

- If you promise a *close* result, compare with a tolerance that scales with the data, such as the sum of absolute values times a few units of machine epsilon times the number of terms:

```rust
fn sum_f32_sequential(xs: &[f32]) -> f32 {
    let mut s = 0.0f32;
    for &x in xs {
        s += x;
    }
    s
}

fn sum_f32_lanes(xs: &[f32]) -> f32 {
    let mut lanes = [0.0f32; 8];
    let mut chunks = xs.chunks_exact(8);
    for chunk in &mut chunks {
        for (acc, &x) in lanes.iter_mut().zip(chunk) {
            *acc += x;
        }
    }
    let mut s = 0.0f32;
    for l in lanes {
        s += l;
    }
    for &x in chunks.remainder() {
        s += x;
    }
    s
}

fn main() {
    let mut state = 12345u64;
    let xs: Vec<f32> = (0..10_000)
        .map(|_| {
            state = state.wrapping_mul(6364136223846793005).wrapping_add(1442695040888963407);
            (state >> 40) as f32 / 16_777_216.0
        })
        .collect();
    let a = f64::from(sum_f32_lanes(&xs));
    let b = f64::from(sum_f32_sequential(&xs));
    let magnitude: f64 = xs.iter().map(|&x| f64::from(x.abs())).sum();
    let bound = magnitude * f64::from(f32::EPSILON) * xs.len() as f64;
    assert!((a - b).abs() <= bound, "{a} vs {b}, allowed {bound}");
    println!("ok: {a} vs {b}");
}
```

- If you want an *exact* test, use data whose sums are exact in any order: integer-valued floats below 2²⁴, as `tunelab`'s property test does (`lane_sum_is_close_to_sequential_sum`), and keep a separate test that documents the difference (`float_sums_are_not_associative`).
- If the result must be bit-identical everywhere, do not change the order: keep the sequential sum, or specify the lane order as part of the contract.

</details>

---

## 13. Interview questions

1. *Why can iterating a 2-D array by columns be dramatically slower than by rows?*
2. *What is the difference between latency-bound and bandwidth-bound code?*
3. *AoS versus SoA: what is the trade-off?*
4. *Why does the Rust compiler refuse to auto-vectorize a floating-point sum, and what can you do about it?*
5. *How do you use a CPU feature such as AVX2 safely in a library that must run on any x86-64 CPU?*
6. *What does `Vec::clear` do to capacity, and why does it matter?*
7. *How would you reduce a Rust binary's size, and what does each step cost?*
8. *What is `#[repr(C)]` for, and what does the default Rust representation allow the compiler to do?*

<details markdown="1">
<summary>Answers</summary>

1. Row-major layout puts a row's elements next to each other; walking down a column touches a different cache line on every step, so most of each loaded line is wasted and the prefetcher cannot help (✓ 30× to 46× slower for a 64 MiB matrix).
2. Latency-bound code waits for each memory access before it can issue the next (pointer chasing: ✓ 155 ns per hop at 128 MiB); bandwidth-bound code issues many independent accesses and is limited by how many bytes per second the memory system delivers (a large sequential scan). Different fixes: shorten dependency chains and improve locality for the first; reduce bytes moved for the second (SoA, smaller types).
3. AoS keeps each record together, good when most fields are used together and for moving whole records. SoA keeps each field contiguous, good when a pass uses few fields of many records and for vectorization (✓ about 10× for a weight sum over 1M particles). The cost of SoA is more complex updates of whole records and more bookkeeping.
4. Floating-point addition is not associative, so reordering into vector lanes could change the result; Rust requires exact semantics and has no stable fast-math mode. Write the order yourself (several accumulators), accept the different rounding, and test with a tolerance or exactly-representable data (✓ the sums differed: `8065.5366` vs `8065.536`; 7.7× faster).
5. Detect the feature at run time (`is_x86_feature_detected!`), put the fast code in a `#[target_feature(enable = "…")] unsafe fn`, call it only when the check passed (with a `SAFETY:` comment), and keep a portable fallback plus a test that both agree.
6. `clear` sets the length to zero and keeps the allocation, so refilling does not allocate again (✓ 0 allocations on later calls). The flip side is that the buffer keeps its peak capacity, which can pin memory after one huge input.
7. Strip symbols (✓ −23%), LTO (✓ −9%), `codegen-units = 1` and `panic = "abort"` (✓ small gains, behavior change for the latter), `opt-level = "s"`/`"z"` (✓ about −5% more, costs speed), prune dependencies and generics, and for extreme cases use `no_std` (Lesson 38). Always measure speed afterwards.
8. `#[repr(C)]` fixes the field order and padding the way C does, which FFI and some binary formats need (✓ `u8, u64, u8` is 24 bytes; sorting the fields makes it 16). The default representation has no guaranteed layout, so the compiler may reorder fields to reduce padding (✓ 16 bytes for the same fields) and use niches such as the null pointer for `Option<Box<T>>` (✓ 8 bytes).

</details>

---

## 14. Practical challenge: `tunelab`

Build the library crate `tunelab` with its tests and measurement driver. Requirements:

1. A flat row-major `Matrix` with `sum_row_major` and `sum_col_major`; a 64-byte `Particle`, its SoA form and weight-sum queries; pointer-chase tables (`cycle_sequential`, `cycle_random`) and `chase`.
2. `sum_f32_sequential`, `sum_f32_lanes` and `sum_u32_wrapping`; `count_byte_scalar`, `count_byte_swar`, and an x86-64 `count_byte_sse2` with a `SAFETY:` comment, plus `count_byte` that picks the best for the target.
3. `join_u32_alloc` and `join_u32_into` (clears but keeps capacity).
4. Tests: known values and boundaries (every length 0 to 69 for the byte counters); property tests that the variants agree; a test that float sums are not associative; allocation-count tests with a counting allocator (reused buffer: zero allocations after warm-up); a `size_of::<Particle>() == 64` test.
5. A measurement driver (`examples/measure.rs`) with a minimum/median harness.
6. Mutation checks, reporting what survives and why.

<details markdown="1">
<summary>Solution</summary>

`Cargo.toml`:

```toml
[package]
name = "tunelab"
version = "0.1.0"
edition = "2024"

[dependencies]

[dev-dependencies]
proptest = "1"

# Line tables help profilers and `--emit asm` readers; they do not change codegen.
[profile.release]
debug = "line-tables-only"
```

`src/lib.rs`:

````rust
//! `tunelab`: small, separately testable examples of the four themes of
//! performance engineering: **memory layout and cache behavior**, **SIMD and
//! auto-vectorization**, **allocation reuse**, and (via the profile settings
//! in the lesson) **binary size**.
//!
//! Every optimized function has a simple reference version that returns the
//! same answer, and the tests check that they agree.
//!
//! ```
//! use tunelab::{count_byte, count_byte_scalar};
//!
//! let data = b"banana bandana";
//! assert_eq!(count_byte(data, b'a'), 6);
//! assert_eq!(count_byte(data, b'a'), count_byte_scalar(data, b'a'));
//! ```

#![deny(missing_docs)]

use std::fmt::Write as _;

// ---------------------------------------------------------------------------
// 1. Memory layout and access order
// ---------------------------------------------------------------------------

/// A square matrix of `u32` stored in one flat `Vec` in row-major order.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Matrix {
    n: usize,
    data: Vec<u32>,
}

impl Matrix {
    /// Creates an `n` by `n` matrix whose element `(r, c)` is
    /// `(r * n + c) as u32` (wrapping), so every element is distinct.
    pub fn new_sequential(n: usize) -> Self {
        let data = (0..n * n).map(|i| i as u32).collect();
        Matrix { n, data }
    }

    /// The side length.
    pub fn side(&self) -> usize {
        self.n
    }

    /// Element at row `r`, column `c`. Panics if out of range.
    pub fn get(&self, r: usize, c: usize) -> u32 {
        assert!(r < self.n && c < self.n, "index out of range");
        self.data[r * self.n + c]
    }
}

/// Sums every element, walking **along rows** (consecutive addresses).
pub fn sum_row_major(m: &Matrix) -> u64 {
    let mut total = 0u64;
    for r in 0..m.n {
        for c in 0..m.n {
            total += u64::from(m.data[r * m.n + c]);
        }
    }
    total
}

/// Sums every element, walking **down columns** (stride of `n * 4` bytes).
/// Same answer as [`sum_row_major`], different memory access pattern.
pub fn sum_col_major(m: &Matrix) -> u64 {
    let mut total = 0u64;
    for c in 0..m.n {
        for r in 0..m.n {
            total += u64::from(m.data[r * m.n + c]);
        }
    }
    total
}

/// An "array of structs" record: 64 bytes, one cache line, of which a
/// query that only needs `weight` uses 4.
#[derive(Debug, Clone, Copy, PartialEq)]
#[repr(C)]
pub struct Particle {
    /// Position.
    pub pos: [f64; 3],
    /// Velocity.
    pub vel: [f64; 3],
    /// Identifier.
    pub id: u64,
    /// The only field the benchmark query reads.
    pub weight: u32,
    /// Padding that keeps the record at 64 bytes.
    pub pad: [u8; 4],
}

/// "Struct of arrays": each field in its own contiguous vector.
#[derive(Debug, Clone, Default, PartialEq)]
pub struct ParticlesSoa {
    /// Positions.
    pub pos: Vec<[f64; 3]>,
    /// Velocities.
    pub vel: Vec<[f64; 3]>,
    /// Identifiers.
    pub id: Vec<u64>,
    /// Weights.
    pub weight: Vec<u32>,
}

/// Builds `n` deterministic particles.
pub fn make_particles(n: usize) -> Vec<Particle> {
    (0..n)
        .map(|i| Particle {
            pos: [i as f64, 0.0, 1.0],
            vel: [0.5, 0.25, 0.0],
            id: i as u64,
            weight: (i as u32).wrapping_mul(2_654_435_761) >> 20,
            pad: [0; 4],
        })
        .collect()
}

/// Converts the array-of-structs form to struct-of-arrays.
pub fn to_soa(ps: &[Particle]) -> ParticlesSoa {
    let mut soa = ParticlesSoa::default();
    for p in ps {
        soa.pos.push(p.pos);
        soa.vel.push(p.vel);
        soa.id.push(p.id);
        soa.weight.push(p.weight);
    }
    soa
}

/// Sum of all weights from the array-of-structs layout.
pub fn total_weight_aos(ps: &[Particle]) -> u64 {
    ps.iter().map(|p| u64::from(p.weight)).sum()
}

/// Sum of all weights from the struct-of-arrays layout.
pub fn total_weight_soa(soa: &ParticlesSoa) -> u64 {
    soa.weight.iter().map(|&w| u64::from(w)).sum()
}

/// Builds a permutation of `0..n` that is a **single cycle** visiting the
/// indexes in order (`i -> i + 1`, last -> 0): the friendly case for a cache.
pub fn cycle_sequential(n: usize) -> Vec<u32> {
    (0..n).map(|i| ((i + 1) % n) as u32).collect()
}

/// Builds a permutation of `0..n` that is a single cycle in a pseudo-random
/// order: the hostile case for a cache.
pub fn cycle_random(n: usize, seed: u64) -> Vec<u32> {
    let mut order: Vec<u32> = (0..n as u32).collect();
    let mut state = seed | 1;
    let mut next = move || {
        state ^= state >> 12;
        state ^= state << 25;
        state ^= state >> 27;
        state.wrapping_mul(0x2545_F491_4F6C_DD1D)
    };
    // Fisher-Yates shuffle of the visiting order.
    for i in (1..n).rev() {
        let j = (next() >> 33) as usize % (i + 1);
        order.swap(i, j);
    }
    // `order` is the visiting sequence of one big cycle, whatever the shuffle
    // produced; convert it to a next-table (each index points at its successor).
    let mut next_of = vec![0u32; n];
    for w in 0..n {
        next_of[order[w] as usize] = order[(w + 1) % n];
    }
    next_of
}

/// Follows `next` for `steps` hops starting at 0 and returns where it ends.
/// Each hop depends on the previous one, so the CPU cannot overlap them: the
/// time per hop is the latency of the memory level the table lives in.
pub fn chase(next: &[u32], steps: usize) -> u32 {
    let mut at = 0u32;
    for _ in 0..steps {
        at = next[at as usize];
    }
    at
}

// ---------------------------------------------------------------------------
// 2. Auto-vectorization and explicit SIMD
// ---------------------------------------------------------------------------

/// Sums `f32`s strictly left to right. Floating-point addition is not
/// associative, so the compiler may **not** reorder this into vector adds.
pub fn sum_f32_sequential(xs: &[f32]) -> f32 {
    let mut s = 0.0f32;
    for &x in xs {
        s += x;
    }
    s
}

/// Sums `f32`s with eight independent accumulators. The *result can differ in
/// the last bits* from [`sum_f32_sequential`] (different rounding order), but
/// the loop is vectorizable because the order is now written down.
pub fn sum_f32_lanes(xs: &[f32]) -> f32 {
    let mut lanes = [0.0f32; 8];
    let mut chunks = xs.chunks_exact(8);
    for chunk in &mut chunks {
        for (acc, &x) in lanes.iter_mut().zip(chunk) {
            *acc += x;
        }
    }
    let mut s = 0.0f32;
    for l in lanes {
        s += l;
    }
    for &x in chunks.remainder() {
        s += x;
    }
    s
}

/// Sums `u32`s with wrapping arithmetic. Integer addition *is* associative
/// (including wrapping), so the compiler can vectorize this on its own.
pub fn sum_u32_wrapping(xs: &[u32]) -> u32 {
    xs.iter().fold(0u32, |a, &x| a.wrapping_add(x))
}

/// Counts occurrences of `needle`: the obvious version.
pub fn count_byte_scalar(haystack: &[u8], needle: u8) -> usize {
    haystack.iter().filter(|&&b| b == needle).count()
}

const LO7: u64 = 0x7f7f_7f7f_7f7f_7f7f;
const ONES: u64 = 0x0101_0101_0101_0101;

/// Counts occurrences of `needle` eight bytes at a time using ordinary `u64`
/// arithmetic ("SIMD within a register"). Portable, safe, no intrinsics.
pub fn count_byte_swar(haystack: &[u8], needle: u8) -> usize {
    let pattern = u64::from(needle).wrapping_mul(ONES);
    let mut chunks = haystack.chunks_exact(8);
    let mut count = 0usize;
    for chunk in &mut chunks {
        let word = u64::from_le_bytes(chunk.try_into().expect("chunks_exact(8)"));
        // Bytes equal to the needle become zero bytes.
        let x = word ^ pattern;
        // Exact zero-byte detector: the high bit of a byte is set in `t`
        // if and only if that byte of `x` is zero (no false positives).
        let t = !(((x & LO7).wrapping_add(LO7)) | x | LO7);
        count += t.count_ones() as usize;
    }
    count + count_byte_scalar(chunks.remainder(), needle)
}

/// Counts occurrences of `needle` with explicit SSE2 intrinsics (sixteen bytes
/// per step). SSE2 is part of the baseline `x86_64` target, so no runtime
/// feature detection is needed. Only available on `x86_64`.
#[cfg(target_arch = "x86_64")]
pub fn count_byte_sse2(haystack: &[u8], needle: u8) -> usize {
    use std::arch::x86_64::{
        __m128i, _mm_cmpeq_epi8, _mm_loadu_si128, _mm_movemask_epi8, _mm_set1_epi8,
    };
    let mut chunks = haystack.chunks_exact(16);
    let mut count = 0usize;
    // SAFETY: SSE2 is always available on x86_64. Each chunk is exactly 16
    // bytes, and `_mm_loadu_si128` has no alignment requirement.
    unsafe {
        let pattern = _mm_set1_epi8(needle as i8);
        for chunk in &mut chunks {
            let v = _mm_loadu_si128(chunk.as_ptr().cast::<__m128i>());
            let mask = _mm_movemask_epi8(_mm_cmpeq_epi8(v, pattern));
            count += (mask as u32).count_ones() as usize;
        }
    }
    count + count_byte_scalar(chunks.remainder(), needle)
}

/// Counts occurrences of `needle` using the best implementation available for
/// the compilation target.
pub fn count_byte(haystack: &[u8], needle: u8) -> usize {
    #[cfg(target_arch = "x86_64")]
    {
        count_byte_sse2(haystack, needle)
    }
    #[cfg(not(target_arch = "x86_64"))]
    {
        count_byte_swar(haystack, needle)
    }
}

// ---------------------------------------------------------------------------
// 3. Allocation reuse
// ---------------------------------------------------------------------------

/// Joins numbers with commas, allocating a `String` per number plus the result.
pub fn join_u32_alloc(values: &[u32]) -> String {
    let parts: Vec<String> = values.iter().map(|v| v.to_string()).collect();
    parts.join(",")
}

/// Joins numbers with commas into a caller-supplied buffer. The buffer is
/// cleared first but keeps its capacity, so repeated calls allocate only when
/// a result is longer than any previous one.
pub fn join_u32_into(values: &[u32], out: &mut String) {
    out.clear();
    for (i, v) in values.iter().enumerate() {
        if i > 0 {
            out.push(',');
        }
        write!(out, "{v}").expect("writing to a String cannot fail");
    }
}
````

`tests/behavior.rs`:

```rust
use proptest::prelude::*;
use tunelab::*;

#[test]
fn matrix_sums_agree_and_match_the_closed_form() {
    for n in [0usize, 1, 2, 7, 64] {
        let m = Matrix::new_sequential(n);
        let expected: u64 = (0..(n * n) as u64).sum();
        assert_eq!(sum_row_major(&m), expected, "n = {n}");
        assert_eq!(sum_col_major(&m), expected, "n = {n}");
    }
    let m = Matrix::new_sequential(3);
    assert_eq!(m.side(), 3);
    assert_eq!(m.get(2, 1), 7);
}

#[test]
#[should_panic(expected = "index out of range")]
fn matrix_get_checks_the_column_too() {
    // Column 3 of a 3x3 matrix would alias row 1, column 0 without the check.
    Matrix::new_sequential(3).get(0, 3);
}

#[test]
fn particle_record_is_one_cache_line() {
    assert_eq!(std::mem::size_of::<Particle>(), 64);
    assert_eq!(std::mem::size_of::<[f64; 3]>(), 24);
}

#[test]
fn aos_and_soa_agree() {
    for n in [0usize, 1, 5, 1000] {
        let ps = make_particles(n);
        let soa = to_soa(&ps);
        assert_eq!(soa.weight.len(), n);
        assert_eq!(soa.id.len(), n);
        assert_eq!(total_weight_aos(&ps), total_weight_soa(&soa));
    }
    let ps = make_particles(3);
    let soa = to_soa(&ps);
    assert_eq!(soa.pos[2], ps[2].pos);
    assert_eq!(soa.vel[1], ps[1].vel);
    assert_eq!(soa.id[2], 2);
}

fn is_single_cycle(next: &[u32]) -> bool {
    let n = next.len();
    let mut at = 0usize;
    for step in 1..=n {
        at = next[at] as usize;
        if at == 0 {
            return step == n;
        }
    }
    false
}

#[test]
fn both_tables_are_single_cycles() {
    for n in [1usize, 2, 3, 10, 1000] {
        assert!(is_single_cycle(&cycle_sequential(n)), "sequential {n}");
        assert!(is_single_cycle(&cycle_random(n, 7)), "random {n}");
    }
    assert_ne!(cycle_random(100, 1), cycle_random(100, 2));
    assert_eq!(cycle_random(100, 1), cycle_random(100, 1));
}

#[test]
fn chase_returns_to_the_start_after_one_lap() {
    for n in [1usize, 5, 64] {
        assert_eq!(chase(&cycle_sequential(n), n), 0);
        assert_eq!(chase(&cycle_random(n, 3), n), 0);
        assert_eq!(chase(&cycle_sequential(n), 0), 0);
    }
    assert_eq!(chase(&cycle_sequential(10), 3), 3);
}

#[test]
fn float_sums_are_not_associative() {
    // 2^24 + 1 is not representable in f32, so the order of additions matters.
    let big = 16_777_216.0f32;
    let mut xs = vec![big];
    xs.extend(std::iter::repeat_n(1.0f32, 15));
    let seq = sum_f32_sequential(&xs);
    let lanes = sum_f32_lanes(&xs);
    assert_eq!(seq, big, "each +1.0 is lost one at a time");
    assert_eq!(
        lanes,
        big + 14.0,
        "seven lanes collect their ones before meeting the big value"
    );
    assert_ne!(seq, lanes);
}

#[test]
fn float_sums_agree_on_exactly_representable_values() {
    let xs: Vec<f32> = (0..1000).map(|i| (i % 7) as f32).collect();
    assert_eq!(sum_f32_sequential(&xs), sum_f32_lanes(&xs));
    assert_eq!(sum_f32_lanes(&[]), 0.0);
    assert_eq!(sum_f32_lanes(&[1.5, 2.5, 3.0]), 7.0);
}

#[test]
fn integer_sum_wraps() {
    assert_eq!(sum_u32_wrapping(&[u32::MAX, 2, 3]), 4);
    assert_eq!(sum_u32_wrapping(&[]), 0);
}

#[test]
fn count_byte_known_values_and_boundaries() {
    let all_a = vec![b'a'; 100];
    for f in [
        count_byte_scalar,
        count_byte_swar,
        count_byte_sse2,
        count_byte,
    ] {
        assert_eq!(f(b"", b'x'), 0);
        assert_eq!(f(b"x", b'x'), 1);
        assert_eq!(f(&all_a, b'a'), 100);
        assert_eq!(f(&all_a, b'b'), 0);
        assert_eq!(f(b"banana bandana", b'a'), 6);
        // High-bit bytes and the needle 0 and 0xFF must work too.
        assert_eq!(
            f(
                &[0xFF, 0x00, 0x80, 0xFF, 0x7F, 0x00, 0xFF, 0x01, 0xFF],
                0xFF
            ),
            4
        );
        assert_eq!(f(&[0u8; 17], 0), 17);
    }
}

#[test]
fn count_byte_every_length_around_the_chunk_sizes() {
    for len in 0..70usize {
        let data: Vec<u8> = (0..len).map(|i| if i % 3 == 0 { 7 } else { 9 }).collect();
        let expected = data.iter().filter(|&&b| b == 7).count();
        assert_eq!(count_byte_scalar(&data, 7), expected, "scalar {len}");
        assert_eq!(count_byte_swar(&data, 7), expected, "swar {len}");
        assert_eq!(count_byte_sse2(&data, 7), expected, "sse2 {len}");
    }
}

#[test]
fn joined_numbers_match() {
    assert_eq!(join_u32_alloc(&[]), "");
    assert_eq!(join_u32_alloc(&[5]), "5");
    assert_eq!(join_u32_alloc(&[1, 20, 300]), "1,20,300");
    let mut buf = String::from("stale contents that must disappear");
    join_u32_into(&[1, 20, 300], &mut buf);
    assert_eq!(buf, "1,20,300");
    join_u32_into(&[], &mut buf);
    assert_eq!(buf, "");
}

proptest! {
    #[test]
    fn count_byte_variants_agree(data in proptest::collection::vec(any::<u8>(), 0..300), needle in any::<u8>()) {
        let expected = count_byte_scalar(&data, needle);
        prop_assert_eq!(count_byte_swar(&data, needle), expected);
        prop_assert_eq!(count_byte_sse2(&data, needle), expected);
        prop_assert_eq!(count_byte(&data, needle), expected);
    }

    #[test]
    fn count_byte_agree_on_few_distinct_values(data in proptest::collection::vec(0u8..3, 0..300), needle in 0u8..3) {
        let expected = count_byte_scalar(&data, needle);
        prop_assert_eq!(count_byte_swar(&data, needle), expected);
        prop_assert_eq!(count_byte_sse2(&data, needle), expected);
    }

    #[test]
    fn join_variants_agree(values in proptest::collection::vec(any::<u32>(), 0..40)) {
        let mut buf = String::from("junk");
        join_u32_into(&values, &mut buf);
        prop_assert_eq!(buf, join_u32_alloc(&values));
    }

    #[test]
    fn lane_sum_is_close_to_sequential_sum(values in proptest::collection::vec(-1000i32..1000, 0..200)) {
        let xs: Vec<f32> = values.iter().map(|&v| v as f32).collect();
        // Integer-valued floats below 2^24 add exactly, so the order cannot matter.
        prop_assert_eq!(sum_f32_lanes(&xs), sum_f32_sequential(&xs));
    }

    #[test]
    fn matrix_orders_agree(n in 0usize..40) {
        let m = Matrix::new_sequential(n);
        prop_assert_eq!(sum_row_major(&m), sum_col_major(&m));
    }
}
```

`tests/alloc.rs`:

```rust
//! Allocation reuse is a countable property, so it is a test.

use std::alloc::{GlobalAlloc, Layout, System};
use std::cell::Cell;
use tunelab::{join_u32_alloc, join_u32_into};

struct Counting;

thread_local! {
    static ALLOCS: Cell<usize> = const { Cell::new(0) };
}

// SAFETY: every method forwards to `System` unchanged; the thread-local
// counter is a const-initialized `Cell`, so touching it cannot allocate.
unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        ALLOCS.with(|c| c.set(c.get() + 1));
        // SAFETY: same contract as the caller's.
        unsafe { System.alloc(layout) }
    }

    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        // SAFETY: same contract as the caller's.
        unsafe { System.dealloc(ptr, layout) }
    }

    unsafe fn realloc(&self, ptr: *mut u8, layout: Layout, new_size: usize) -> *mut u8 {
        ALLOCS.with(|c| c.set(c.get() + 1));
        // SAFETY: same contract as the caller's.
        unsafe { System.realloc(ptr, layout, new_size) }
    }
}

#[global_allocator]
static GLOBAL: Counting = Counting;

fn count_allocs<T>(f: impl FnOnce() -> T) -> (T, usize) {
    let before = ALLOCS.with(Cell::get);
    let out = f();
    (out, ALLOCS.with(Cell::get) - before)
}

#[test]
fn allocating_join_allocates_per_number() {
    let values: Vec<u32> = (0..1000).collect();
    let (s, n) = count_allocs(|| join_u32_alloc(&values));
    assert_eq!(s.len(), 3889);
    eprintln!("allocating join: {n} allocations");
    assert!(n >= 1000, "one String per number at least: {n}");
}

#[test]
fn reused_buffer_stops_allocating_after_warm_up() {
    let values: Vec<u32> = (0..1000).collect();
    let mut buf = String::new();
    let (_, first) = count_allocs(|| join_u32_into(&values, &mut buf));
    eprintln!("first call: {first} allocations");
    assert!(first >= 1, "the first call has to grow the buffer");
    assert!(first <= 16, "doubling growth is logarithmic: {first}");
    for _ in 0..10 {
        let (_, again) = count_allocs(|| join_u32_into(&values, &mut buf));
        assert_eq!(again, 0, "a warmed-up buffer must not allocate");
    }
    assert_eq!(buf, join_u32_alloc(&values));
}

#[test]
fn counter_sees_a_known_number_of_allocations() {
    let (_v, n) = count_allocs(|| Vec::<u8>::with_capacity(10));
    assert_eq!(n, 1);
    let (_, none) = count_allocs(|| 1 + 1);
    assert_eq!(none, 0);
}
```

`examples/measure.rs`:

```rust
//! Hand-rolled timing driver: `measure <matrix|layout|chase|simd|bytes|join|all>`.
//! Reports the minimum and median of several runs. Release build only.
use std::hint::black_box;
use std::time::Instant;
use tunelab::*;

fn bench<R>(samples: usize, mut f: impl FnMut() -> R) -> (f64, f64) {
    black_box(f()); // warm-up
    let mut v: Vec<f64> = (0..samples)
        .map(|_| {
            let t = Instant::now();
            let r = f();
            let ns = t.elapsed().as_nanos() as f64;
            black_box(r);
            ns
        })
        .collect();
    v.sort_by(f64::total_cmp);
    (v[0], v[v.len() / 2])
}

fn matrix() {
    for n in [256usize, 1024, 4096] {
        let m = Matrix::new_sequential(n);
        let (rmin, rmed) = bench(9, || sum_row_major(black_box(&m)));
        let (cmin, cmed) = bench(9, || sum_col_major(black_box(&m)));
        println!(
            "matrix n={n:<5} ({:>6} KiB): row min {:>11.0} ns med {:>11.0} | col min {:>11.0} ns med {:>11.0} | col/row (min) {:.2}x",
            n * n * 4 / 1024,
            rmin,
            rmed,
            cmin,
            cmed,
            cmin / rmin
        );
    }
}

fn layout() {
    let n = 1_000_000;
    let ps = make_particles(n);
    let soa = to_soa(&ps);
    let (amin, amed) = bench(15, || total_weight_aos(black_box(&ps)));
    let (smin, smed) = bench(15, || total_weight_soa(black_box(&soa)));
    println!(
        "weights of {n} particles: AoS min {amin:.0} ns med {amed:.0} | SoA min {smin:.0} ns med {smed:.0} | AoS/SoA (min) {:.2}x",
        amin / smin
    );
}

fn chase_exp() {
    let steps = 4_000_000;
    for kib in [4usize, 32, 256, 2048, 16384, 131072] {
        let n = kib * 1024 / 4;
        let seq = cycle_sequential(n);
        let rnd = cycle_random(n, 11);
        let (smin, _) = bench(5, || chase(black_box(&seq), steps));
        let (rmin, _) = bench(5, || chase(black_box(&rnd), steps));
        println!(
            "chase table {kib:>7} KiB: sequential {:>6.2} ns/hop | random {:>6.2} ns/hop",
            smin / steps as f64,
            rmin / steps as f64
        );
    }
}

fn simd() {
    let n = 16 * 1024;
    let xs: Vec<f32> = (0..n).map(|i| (i % 1000) as f32 * 0.001).collect();
    let us: Vec<u32> = (0..n as u32).collect();
    let (a, _) = bench(31, || sum_f32_sequential(black_box(&xs)));
    let (b, _) = bench(31, || sum_f32_lanes(black_box(&xs)));
    let (c, _) = bench(31, || sum_u32_wrapping(black_box(&us)));
    println!(
        "sum of {n} elements: f32 sequential {a:.0} ns ({:.2} ns/elem) | f32 8 lanes {b:.0} ns ({:.2}) | u32 wrapping {c:.0} ns ({:.2})",
        a / n as f64,
        b / n as f64,
        c / n as f64
    );
    println!(
        "  f32 results: sequential {} lanes {} (differ: {})",
        sum_f32_sequential(&xs),
        sum_f32_lanes(&xs),
        sum_f32_sequential(&xs) != sum_f32_lanes(&xs)
    );
}

fn bytes() {
    let n = 1 << 20;
    let data: Vec<u8> = (0..n as usize)
        .map(|i| (i.wrapping_mul(2_654_435_761usize) >> 24) as u8 % 16)
        .collect();
    let expected = count_byte_scalar(&data, 3);
    assert_eq!(count_byte_swar(&data, 3), expected);
    assert_eq!(count_byte_sse2(&data, 3), expected);
    let (a, _) = bench(31, || count_byte_scalar(black_box(&data), 3));
    let (b, _) = bench(31, || count_byte_swar(black_box(&data), 3));
    let (c, _) = bench(31, || count_byte_sse2(black_box(&data), 3));
    println!(
        "count_byte over 1 MiB (count {expected}): scalar {a:.0} ns ({:.2} GiB/s) | swar {b:.0} ns ({:.2} GiB/s) | sse2 {c:.0} ns ({:.2} GiB/s)",
        n as f64 / a,
        n as f64 / b,
        n as f64 / c
    );
}

fn join() {
    let values: Vec<u32> = (0..1000).map(|i| i * 7919).collect();
    let (a, _) = bench(31, || join_u32_alloc(black_box(&values)));
    let mut buf = String::new();
    let (b, _) = bench(31, || {
        join_u32_into(black_box(&values), &mut buf);
        buf.len()
    });
    println!(
        "join 1000 numbers: allocating {a:.0} ns | reused buffer {b:.0} ns | ratio {:.2}x",
        a / b
    );
}

fn main() {
    let which = std::env::args().nth(1).unwrap_or_else(|| "all".into());
    let all = which == "all";
    if all || which == "matrix" {
        matrix();
    }
    if all || which == "layout" {
        layout();
    }
    if all || which == "chase" {
        chase_exp();
    }
    if all || which == "simd" {
        simd();
    }
    if all || which == "bytes" {
        bytes();
    }
    if all || which == "join" {
        join();
    }
}
```

Commands (from the crate root):

```text
cargo test
cargo build --release --example measure
./target/release/examples/measure            # all experiments (or: matrix | layout | chase | simd | bytes | join)
RUSTFLAGS="-C target-cpu=x86-64-v3" cargo build --release --example measure   # AVX2 build (this CPU supports it)
cargo rustc --release --lib -- --emit asm    # look for addps / paddd / pcmpeqb in target/release/deps/*.s
```

✓ Checks I ran: `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, `cargo test --locked` (3 allocation tests, 17 behavior tests including 5 property tests, 1 doctest; all pass) and the measurements quoted in the lesson. `RUSTDOCFLAGS="-D warnings" cargo doc --no-deps` also passes.

**Mutation checks** (each mutant on a copy of the crate, full test suite):

| # | Mutation | Result |
|---|---|---|
| T1 | `Matrix::get` forgets to check the column | **caught** (the `should_panic` test: column 3 of a 3×3 matrix would silently alias another element) |
| T2 | `sum_col_major` skips the last column | **caught** (two tests) |
| T3 | `to_soa` does not copy the weights | **caught** |
| T4 | Fisher-Yates shuffle (`% (i + 1)`) becomes `% i` (Sattolo form) in `cycle_random` | **survived: equivalent mutant.** The shuffle only decides the visiting order, and any permutation of the visiting order, converted to a next-table, is one big cycle |
| T5 | SWAR zero-byte test loses its `\| x` term | **caught** (bytes with the high bit set, such as `0x7F` against needle `0xFF`, look like zero bytes without it) |
| T6 | SWAR tail (`len % 8` bytes) dropped | **caught** (four tests) |
| T7 | `count_byte_sse2` tail dropped | **caught** (four tests) |
| T8 | SSE2 compares against zero instead of the needle | **caught** (four tests) |
| T9 | `sum_f32_lanes` tail dropped | **caught** |
| T10 | `chunks_exact(8)` becomes `chunks_exact(4)` in `sum_f32_lanes` | **caught only by `float_sums_are_not_associative`**: the sums of exactly-representable values still agree, only the *rounding order* changes, so only the test that pins the order notices |
| T11 | `join_u32_into` does not clear | **caught** (three tests) |
| T12 | separator condition `i > 0` becomes `i > 1` | **caught** (three tests) |
| T13 | `sum_u32_wrapping` uses `+` instead of `wrapping_add` | **caught** (overflow panic in a debug build; **in a release build the same mutant would wrap silently and this test could not see it**, so run the suite in debug) |
| T14 | `chase` takes one hop too many | **caught** |
| T15 | `write!` replaced by `push_str(&v.to_string())` (same output, one allocation per number) | **caught only by the allocation test** |
| T16 | `clear()` replaced by `*out = String::new()` (same output, allocates every call) | **caught only by the allocation test** |
| T18 | `Matrix::new_sequential` adds 1 to each element | **caught** |

Two further mutants I tried (T17: change the padding of `Particle` to 8 bytes; T8 as first written: use a comparison intrinsic that was not imported) did not compile and are not counted. Totals: 17 valid mutants, 16 caught, 1 equivalent (T4). Three observations. (a) T15 and T16 are the lesson's theme in miniature: **performance properties (zero allocations, reuse) have their own test** because output-equality tests cannot see them. (b) T10 shows that a test about *what differs on purpose* (rounding order) can be the only guard of a design choice. (c) T13 is a reminder that overflow checks depend on the build profile.

**Design notes.**

- Every optimized function has a simple reference in the same crate, and the tests compare them on boundary lengths and on arbitrary inputs.
- `count_byte_sse2` is `pub` and safe: all `unsafe` is inside, the chunks are exactly 16 bytes, and loads are unaligned (`_mm_loadu_si128`), so there is no alignment requirement. SSE2 is part of the `x86_64` baseline, so no runtime detection is required.
- `count_byte` falls back to the SWAR version on non-x86-64 targets; I only compiled for x86-64, so the `cfg(not(target_arch = "x86_64"))` branch was never built.
- `cycle_random` converts a shuffled visiting order into a next-table, so the table is a single cycle by construction (tested by following it).
- Not handled: blocked matrix transposition, AVX2/AVX-512 intrinsics, AArch64 NEON, alignment-aware allocation, non-square matrices, hardware counters, and a statistical benchmark harness (Lesson 41's Criterion would be the next step for these benchmarks).

</details>

---

## 15. Key takeaways

- **Memory first.** A dependent load costs about 1.5 ns in L1 and about 155 to 162 ns from main memory on this machine (✓ a 100-fold range), and a sequential scan hides almost all of it (✓ flat 1.3 to 1.7 ns).
- Walk data in the order it is stored (✓ 30× or more for column-major over a 64 MiB matrix) and store together what is used together (✓ SoA about 10× for a one-field query).
- The compiler vectorizes integer reductions and cannot reorder floating-point ones (✓ `paddd` versus `addss`); writing the order down gives about 7.7× at the price of different rounding.
- Vectorized is not automatically fast, and the portable safe SWAR version came within a small factor of the intrinsics (✓ 8.5 to 9.7 against 8.4 to 13.2 GiB/s) while the scalar code was 2.4 to 2.9. Compiling for a newer CPU level gave about 3× more (✓) at the cost of portability, so prefer runtime detection for libraries.
- Reuse buffers: ✓ 1,002 allocations became 10 once and then 0; keep a reference version and an allocation test; remember reused buffers retain their peak capacity.
- Size: stripping, LTO and `panic = "abort"` are cheap (✓ 495,408 → 341,584 bytes); `s`/`z` trade speed for the last few percent; below that, `no_std` is a different program.
- Performance claims are measurements on a machine: I report ranges because the runs differed.

**Connections.** [Lesson 41](41_benchmarking_profiling.md) supplied the measurement discipline and the counting allocator; [Lesson 40](40_property_testing_fuzzing.md)'s properties prove each fast variant equals its reference; [Lesson 39](39_testing.md)'s mutation method found the performance-only gaps (T15, T16); [Lesson 38](38_no_std_embedded_miri.md) is where binary size becomes the main constraint; [Lesson 37](37_maybeuninit_ffi.md) and [Lesson 36](36_unsafe_raw_pointers.md) cover the `repr(C)` and `unsafe` rules used in §4 and §5; [Lesson 16](16_collections.md) covers `Vec` capacity. Next: Lesson 43 covers observability, CI, and dependency security.

---

## 16. References

- The Rust Performance Book (data layout, type sizes, allocation, build configuration, inlining): <https://nnethercote.github.io/perf-book/>
- The Rustonomicon, "Data Layout" (representations, `repr(C)`, padding, niches): <https://doc.rust-lang.org/nomicon/data.html>
- `std::arch` (intrinsics) and `is_x86_feature_detected!`: <https://doc.rust-lang.org/std/arch/index.html>, <https://doc.rust-lang.org/std/macro.is_x86_feature_detected.html>
- The Rust Reference, "Target features" (`#[target_feature]`): <https://doc.rust-lang.org/reference/attributes/codegen.html>
- Cargo reference, "Profiles" (`strip`, `lto`, `opt-level`, `panic`): <https://doc.rust-lang.org/cargo/reference/profiles.html>
- `min-sized-rust` (a community guide to shrinking binaries): <https://github.com/johnthagen/min-sized-rust>
- Ulrich Drepper, "What Every Programmer Should Know About Memory" (the classic article on caches; title from memory).
- The `memchr` crate (a production byte search with runtime-dispatched SIMD; not run here): <https://crates.io/crates/memchr>

The URLs above are the standard documentation locations; I did not open them during this session (the sandbox cannot reach the documentation sites), so if a link has moved, search for the page title. Unchecked or from memory: that cache lines are 64 bytes on x86-64 (I did not query it); the hardware-prefetcher explanation of the flat sequential line and the TLB contribution to the large-table random line (the timings are measured, the causes are not isolated); the details of how LLVM's auto-vectorizer decides (the instruction counts are measured); that stable Rust has no fast-math mode; that `std::simd` is nightly-only at the time of writing; that `cargo asm`, `cargo bloat` and `wasm-opt` exist and what they do (none run); the claim that a `target_feature` function called on a CPU without the feature is undefined behavior; the statement that niches make `Option<Box<T>>` pointer-sized in general (I measured the specific sizes printed in §4.1); the guidance on `shrink_to` for retained capacity; every item in "other levers" in §7; that `memchr` dispatches at run time; and the remarks about constant-time code. The measurements marked ✓ are from one shared virtual machine; ranges quoted are across two or three runs of the same binary, and the first-run SSE2 figure of 8.4 GiB/s versus 12.8 to 13.2 later is an example of run-to-run variation I could not explain. Binary sizes are for one small program and this toolchain. Exercise answers 4, 5, 7 and 8 combine ✓-run facts with reasoning from memory; the code blocks in answer 3 and in the debugging challenge were compiled and run. Everything marked ✓ was run on this machine's `rustc 1.97.0`.
