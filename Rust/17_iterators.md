---
title: "Lesson 17 — Iterators and adapters"
---

# Lesson 17 — Iterators and adapters

**Module 6 · Collections and functional programming** · Status: generated · Edition used in examples: **2024** · External dependencies: none

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux. As before, ✓ marks outputs and diagnostics I actually ran; unmarked statements are documented behavior I did not run. Everything is **stable Rust**. Programs were compiled without optimizations unless stated.

Builds on [Lesson 16](16_collections.md) (collections, `collect`, `entry`), [Lesson 14](14_option_result_errors.md) (`Option`, `Result`, `?`), [Lesson 10](10_references_and_borrowing.md) (borrow rules), [Lesson 11](11_lifetimes.md) (lifetimes in structs and return types) and [Lesson 06](06_strings_conversions_patterns.md) (`chars`, `split`, slices). Closures are used throughout and are covered properly in Lesson 18; here you only need "`|x| expr` is an anonymous function that can use variables around it".

---

## 1. What an iterator is, and why Rust is built around them

An **iterator** is anything that can hand out a sequence of values one at a time. In Rust that is exactly one trait with one required method:

```rust
trait Iterator {
    type Item;                                  // what it yields
    fn next(&mut self) -> Option<Self::Item>;   // Some(item) or None when finished
    // ...dozens of provided methods (map, filter, sum, collect, ...) built on next()
}
```

Everything else (`map`, `filter`, `sum`, `collect`, `zip`, ...) is a *provided method* written in terms of `next`. If you implement `next`, you get all of them for free. The design has three consequences that this whole lesson explores:

1. **Iterators are lazy.** Building one does no work; work happens only when something calls `next`.
2. **Iterators are values with state.** An iterator remembers where it is (that is why `next` takes `&mut self`), and an adapter like `map` is just a struct wrapping another iterator.
3. **Chains are zero-cost abstractions** in the Rust sense: the generic code is monomorphized and inlined, so `v.iter().map(f).filter(g).sum()` normally compiles to the same loop you would write by hand. (I show a small ✓ check of this in Section 10; it is not a general guarantee.)

Why the design: `for` loops, bounds-check-free traversal, and the "iterator invalidation" bugs of other languages (Lesson 16) all fit one model. An iterator that borrows a collection holds a borrow (Lesson 10), so the compiler prevents you from modifying the collection while iterating it.

---

## 2. Laziness, `next`, and `for`

```rust
fn main() {
    let v = vec![1, 2, 3, 4];
    let it = v.iter().map(|x| { println!("map {x}"); x * 10 });
    println!("built, nothing ran");
    let out: Vec<i32> = it.collect();
    println!("{out:?}");
    let first = v.iter().map(|x| { println!("visit {x}"); x * 2 }).find(|&y| y > 2);
    println!("{first:?}");
}
```

✓ Output:

```text
built, nothing ran
map 1
map 2
map 3
map 4
[10, 20, 30, 40]
visit 1
visit 2
Some(4)
```

- Nothing printed while the adapter chain was built; the `map` closure ran only when `collect` pulled values.
- `find` **short-circuits**: it stopped at the second element (`visit 2`, result `Some(4)`) and never touched 3 or 4. Because the chain is pull-based, later stages decide how much of the earlier stages' work is done.

Since adapters do nothing until consumed, forgetting to consume one is a bug the compiler warns about ✓:

```rust
fn main() {
    let v = vec![1, 2, 3];
    v.iter().map(|x| x + 1);
}
```

✓ `warning: unused `Map` that must be used` with the note "iterators are lazy and do nothing unless consumed". Treat that warning as an error in your head (and in CI, Lesson 43).

Calling `next` by hand shows the protocol ✓:

```rust
fn main() {
    let v = vec![10, 20];
    let mut it = v.iter();
    println!("{:?} {:?} {:?} {:?}", it.next(), it.next(), it.next(), it.next());
    let mut c = 0;
    let mut counter = std::iter::from_fn(|| { c += 1; if c <= 3 { Some(c) } else { None } });
    println!("{:?}", counter.by_ref().collect::<Vec<_>>());
    println!("{:?}", counter.next());
}
```

✓ Output `Some(10) Some(20) None None`, then `[1, 2, 3]` and `None`. After the first `None` an iterator is *allowed* to return `Some` again (the trait does not forbid it), which is why `fuse()` exists (Section 5.5); the standard collections' iterators keep returning `None`.

### 2.1 What a `for` loop really is

`for x in expr { body }` is sugar for: call `IntoIterator::into_iter(expr)` once, then loop calling `next` until `None`:

```rust
fn main() {
    let v = vec![1, 2, 3];
    let mut it = IntoIterator::into_iter(&v);
    let mut sum = 0;
    loop {
        match it.next() { Some(x) => sum += x, None => break }
    }
    println!("{sum}");
    let mut total = 0;
    for x in [10, 20] { total += x; }   // arrays iterate by value (2021 edition and later)
    println!("{total}");
}
```

✓ Output `6` and `30`. `IntoIterator` is the trait that says "I can be turned into an iterator", and every `Iterator` is also an `IntoIterator` (it returns itself). That is why `for x in v.iter().rev()` and `for x in &v` both work.

### 2.2 Three ways to iterate a collection

A collection offers three iterators, differing in what `Item` is (Lesson 10's borrow kinds):

| Call | Also written | `Item` for `Vec<String>` | Effect on the collection |
|---|---|---|---|
| `v.iter()` | `for x in &v` | `&String` | borrowed, read-only |
| `v.iter_mut()` | `for x in &mut v` | `&mut String` | borrowed mutably, can edit in place |
| `v.into_iter()` | `for x in v` | `String` | **consumes** `v` (moves the elements out) |

```rust
fn main() {
    let mut v = vec![String::from("a"), String::from("b")];
    for s in &v { print!("{} ", s.len()); }
    for s in &mut v { s.push('!'); }
    for s in v.iter().map(String::as_str) { print!("{s} "); }
    println!();
    let owned: Vec<String> = v.into_iter().collect();
    println!("{owned:?}");
}
```

✓ Output `1 1 a! b! ` then `["a!", "b!"]`. After `for s in v` the vector has been moved:

```rust
fn main() {
    let v = vec![String::from("a")];
    for s in v { println!("{s}"); }
    println!("{}", v.len());
}
```

✓ `error[E0382]: borrow of moved value: `v``, with the note that `into_iter` takes ownership (Lesson 07). Use `for s in &v` to keep using `v`.

---

## 3. Adapters: building pipelines

An **adapter** (also "iterator adaptor") takes an iterator and returns a new iterator; it is lazy. Here are the ones you will use most ✓:

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5, 6];
    let evens: Vec<i32> = v.iter().copied().filter(|x| x % 2 == 0).collect();
    println!("{evens:?}");
    println!("{:?}", v.iter().map(|x| x * x).sum::<i32>());
    println!("{:?}", v.iter().enumerate().filter(|(i, _)| i % 2 == 0).map(|(_, x)| *x).collect::<Vec<_>>());
    println!("{:?}", v.iter().zip(v.iter().skip(1)).map(|(a, b)| b - a).collect::<Vec<_>>());
    println!("{:?}", v.iter().take_while(|&&x| x < 4).collect::<Vec<_>>());
    println!("{:?}", v.iter().skip_while(|&&x| x < 4).collect::<Vec<_>>());
    println!("{:?}", v.iter().step_by(2).collect::<Vec<_>>());
    println!("{:?}", v.iter().rev().take(2).collect::<Vec<_>>());
    println!("{:?}", [[1, 2], [3, 4]].iter().flatten().collect::<Vec<_>>());
    println!("{:?}", ["a b", "c"].iter().flat_map(|s| s.split(' ')).collect::<Vec<_>>());
    println!("{:?}", (1..4).chain(7..9).collect::<Vec<_>>());
    println!("{:?}", v.iter().scan(0, |acc, &x| { *acc += x; Some(*acc) }).collect::<Vec<_>>());
    println!("{:?}", v.iter().filter_map(|&x| if x > 4 { Some(x * 2) } else { None }).collect::<Vec<_>>());
    println!("{:?}", v.iter().map_while(|&x| (x < 3).then_some(x)).collect::<Vec<_>>());
}
```

✓ Output:

```text
[2, 4, 6]
91
[1, 3, 5]
[1, 1, 1, 1, 1]
[1, 2, 3]
[4, 5, 6]
[1, 3, 5]
[6, 5]
[1, 2, 3, 4]
["a", "b", "c"]
[1, 2, 3, 7, 8]
[1, 3, 6, 10, 15, 21]
[10, 12]
[1, 2]
```

| Adapter | What it does |
|---|---|
| `map(f)` | transform each item |
| `filter(p)` | keep items where `p` is true (`p` receives `&Item`) |
| `filter_map(f)` | `map` then drop the `None`s (one step) |
| `enumerate()` | pairs each item with its index `(0, x)` |
| `zip(other)` | pairs items from two iterators; **stops at the shorter one** ✓ |
| `chain(other)` | all of the first, then all of the second |
| `take(n)` / `skip(n)` | first `n` / all but the first `n` |
| `take_while(p)` / `skip_while(p)` / `map_while(f)` | stop / skip / stop-and-map while a condition holds |
| `step_by(n)` | every `n`th item |
| `rev()` | reverse (needs a double-ended iterator, Section 7) |
| `flatten()` / `flat_map(f)` | turn an iterator of iterables into one flat iterator |
| `scan(init, f)` | a `map` with mutable state carried across items (running totals) |
| `peekable()` | adds `peek`, to look at the next item without consuming it |
| `fuse()` | after the first `None`, always `None` |
| `cycle()` | repeat forever (needs `Clone`) |
| `inspect(f)` | run `f` on each item for debugging, passing it through |
| `copied()` / `cloned()` | turn `&T` items into `T` (needs `Copy` / `Clone`) |
| `by_ref()` | borrow the iterator so a consumer does not take it by value |

**Item types and the `&` patterns.** `v.iter()` yields `&i32`, so closures see references: `filter` takes `&Item`, i.e. `&&i32` (hence `|&&x|`), while `map` takes `&i32` (hence `|&x|` or `*x`). When the extra `&`s are noisy, add `.copied()` (for `Copy` items) or `.cloned()` first, then the closures see plain `i32`. Only copy or clone when it is cheap; `.cloned()` on `String`s allocates.

### 3.1 Ranges and sources

Ranges are iterators, and the `std::iter` module makes more ✓:

```rust
fn main() {
    println!("{:?}", (1..5).collect::<Vec<_>>());
    println!("{:?}", (1..=5).rev().collect::<Vec<_>>());
    println!("{:?}", (0..10).step_by(3).collect::<Vec<_>>());
    println!("{} {}", (5..1).count(), (1..=1).count());
    println!("{:?}", ('a'..='e').collect::<String>());
    println!("{:?}", (1u64..=20).product::<u64>());
    println!("{:?}", (1..4).size_hint());
    println!("{:?}", (1..).take(3).collect::<Vec<_>>());
    println!("{:?}", std::iter::repeat("ab").take(3).collect::<String>());
    println!("{:?}", std::iter::once(1).chain(std::iter::empty()).collect::<Vec<i32>>());
    println!("{:?}", std::iter::successors(Some(1u32), |&x| x.checked_mul(10)).count());
}
```

✓ Output:

```text
[1, 2, 3, 4]
[5, 4, 3, 2, 1]
[0, 3, 6, 9]
0 1
"abcde"
2432902008176640000
(3, Some(3))
[1, 2, 3]
"ababab"
[1]
10
```

`(5..1)` is an *empty* range (it does not count down; use `(1..5).rev()`). `(1..)` is infinite, so you must bound it with `take`, `take_while`, `zip` or a short-circuiting consumer. `successors(first, f)` generates a sequence from a seed, ending when `f` returns `None` (here powers of ten until `u32` would overflow: 10 values). `from_fn` (Section 2) generates from a closure with state.

---

## 4. Consumers: turning an iterator into a value

A **consumer** (or "consuming adapter") calls `next` until it has an answer. These run the pipeline ✓:

```rust
fn main() {
    let v = vec![3, 1, 4, 1, 5, 9, 2, 6];
    println!("{:?} {:?}", v.iter().max(), v.iter().min());
    println!("{:?}", v.iter().max_by_key(|&&x| (x as i32 - 5).abs()));
    println!("{} {}", v.iter().any(|&x| x > 8), v.iter().all(|&x| x > 0));
    println!("{:?} {:?}", v.iter().find(|&&x| x > 4), v.iter().position(|&x| x == 5));
    println!("{:?}", v.iter().fold(String::new(), |mut s, x| { s.push_str(&x.to_string()); s }));
    println!("{:?}", v.iter().copied().reduce(|a, b| a.max(b)));
    println!("{}", v.iter().product::<i32>());
    println!("{:?}", v.iter().copied().partition::<Vec<i32>, _>(|x| x % 2 == 0));
    let (a, b): (Vec<i32>, Vec<char>) = [(1, 'a'), (2, 'b')].into_iter().unzip();
    println!("{a:?} {b:?}");
    println!("{:?} {:?} {}", v.iter().last(), v.iter().nth(2), v.iter().count());
    let empty: Vec<i32> = vec![];
    println!("{:?} {} {:?}", empty.iter().max(), empty.iter().sum::<i32>(), empty.iter().copied().reduce(|a, b| a + b));
}
```

✓ Output:

```text
Some(9) Some(1)
Some(9)
true true
Some(5) Some(4)
"31415926"
Some(9)
6480
([4, 2, 6], [3, 1, 1, 5, 9])
[1, 2] ['a', 'b']
Some(6) Some(4) 8
None 0 None
```

Things to know:

- `fold(init, f)` is the general consumer: it threads an accumulator through the items. `sum`, `product`, `count`, `max`, ... are all special cases. `reduce` is `fold` with the first item as the starting value, so it returns `None` for an empty iterator ✓ (`sum` of nothing is the identity `0`, but there is no sensible "max of nothing", so `max` returns `Option`).
- `max_by_key(f)` returns the item whose key is largest; for ties among equals, `max` family returns the **last** maximum and `min` returns the **first** minimum (documented).
- `any`, `all`, `find`, `position` **short-circuit**.
- `partition` splits into two collections; `unzip` splits an iterator of pairs.
- Consumers that take `self` by value (most do) consume the iterator; use `by_ref()` to keep using it afterwards (Section 5.4).

### 4.1 Floats are not `Ord`, so `max` and `min` need help

```rust
fn main() {
    let v = [2.5, -1.0, 7.25, 3.0];
    let m = v.iter().copied().fold(f64::NEG_INFINITY, f64::max);
    let mx = v.iter().copied().max_by(|a, b| a.total_cmp(b));
    let mn = v.iter().min_by(|a, b| a.partial_cmp(b).unwrap());
    println!("{m} {mx:?} {mn:?}");
    println!("{}", v.iter().sum::<f64>() / v.len() as f64);
    let bad = v.iter().max();
}
```

✓ The last line gives `error[E0277]: the trait bound `f64: Ord` is not satisfied` (Lesson 16: `NaN` has no order). Without the `bad` line ✓ the output is `7.25 Some(7.25) Some(-1.0)` and the mean `2.9375`. Prefer `total_cmp` over `partial_cmp(..).unwrap()`, which panics on `NaN`.

---

## 5. `collect`, `sum` and the "type annotations needed" family

### 5.1 `collect` builds any `FromIterator`

`collect` is generic over its **return type**; the type you ask for decides what is built ✓:

```rust
use std::collections::{BTreeMap, BTreeSet, HashMap, HashSet, VecDeque};
fn main() {
    let words = ["b", "a", "b"];
    let s: String = words.iter().copied().collect();
    let v: Vec<&str> = words.iter().copied().collect();
    let set: BTreeSet<&str> = words.iter().copied().collect();
    let m: BTreeMap<&str, usize> = words.iter().enumerate().map(|(i, w)| (*w, i)).collect();
    let d: VecDeque<_> = (1..4).collect();
    let h: HashSet<i32> = (1..4).collect();
    let hm: HashMap<_, _> = (1..3).zip("xy".chars()).collect();
    println!("{s} {v:?} {set:?} {m:?} {d:?} {} {}", h.len(), hm.len());
}
```

✓ Output `bab ["b", "a", "b"] {"a", "b"} {"a": 1, "b": 2} [1, 2, 3] 3 2`. Note the `BTreeMap`: key `"b"` appeared twice and the later value (2) won, as with repeated `insert`.

If the compiler cannot work out the target type, it asks ✓:

```rust
fn main() {
    let v = (1..4).collect();
    println!("{}", 1);
}
```

✓ `error[E0283]: type annotations needed`, "the type must implement `FromIterator<i32>`", with the suggestion `let v: Vec<_> = ...`. Annotate the binding (`let v: Vec<_>`), or use the turbofish (`.collect::<Vec<_>>()`). `sum` and `product` need the same:

```rust
fn main() {
    let v = vec![1, 2, 3];
    let s = v.iter().sum();
    println!("{}", s);
}
```

✓ `E0283` again; write `let s: i32 = ...` or `sum::<i32>()`. (On older compilers this family reported `E0282`; on 1.97 I saw `E0283` for all three.)

A subtle one: `partition` with a `&`-item iterator and an underscore ✓:

```rust
fn main() {
    let v = vec![3, 1, 4];
    let (a, b) = v.iter().partition::<Vec<_>, _>(|&&x| x % 2 == 0);
    println!("{a:?} {b:?}");
}
```

✓ `error[E0283]: type annotations needed`, "multiple `impl`s satisfying `Vec<_>: Extend<&i32>` found" (a `Vec<i32>` can be extended from `&i32` when `i32: Copy`, and a `Vec<&i32>` from `&i32` too). Say which you mean ✓:

```rust
fn main() {
    let v = vec![3, 1, 4];
    let (a, b): (Vec<&i32>, Vec<&i32>) = v.iter().partition(|&&x| x % 2 == 0);
    let (c, d): (Vec<i32>, Vec<i32>) = v.iter().copied().partition(|x| x % 2 == 0);
    println!("{a:?} {b:?} {c:?} {d:?}");
}
```

✓ Output `[4] [3, 1] [4] [3, 1]`.

### 5.2 Collecting `Result` and `Option`: the "all or nothing" pattern

`Result<C, E>` and `Option<C>` also implement `FromIterator`, which turns an iterator of fallible items into a fallible collection that **stops at the first failure** (documented) ✓:

```rust
fn main() {
    let r: Result<Vec<i32>, _> = ["1", "2", "x", "4"].iter().map(|s| s.parse::<i32>()).collect();
    println!("{r:?}");
    let o: Option<Vec<i32>> = [Some(1), Some(2)].into_iter().collect();
    let o2: Option<Vec<i32>> = [Some(1), None].into_iter().collect();
    println!("{o:?} {o2:?}");
    let total: Result<i32, std::num::ParseIntError> = ["1", "2"].iter().map(|s| s.parse::<i32>()).sum();
    println!("{total:?}");
    let ok: Vec<i32> = ["1", "x", "3"].iter().filter_map(|s| s.parse().ok()).collect();
    println!("{ok:?}");
    let (good, bad): (Vec<_>, Vec<_>) = ["1", "x", "3"].iter().map(|s| s.parse::<i32>()).partition(|r| r.is_ok());
    println!("{} {}", good.len(), bad.len());
}
```

✓ Output:

```text
Err(ParseIntError { kind: InvalidDigit })
Some([1, 2]) None
Ok(3)
[1, 3]
2 1
```

Choose deliberately (Lesson 14): `collect::<Result<_, _>>()` for "reject bad input", `filter_map(|s| s.parse().ok())` to *silently skip* bad items (dangerous if silence hides a bug), `partition` to keep both sides and report the failures. `sum` and `product` also accept `Result`/`Option` items.

Also note how `collect` sizes the output: it uses the iterator's `size_hint` to reserve capacity, so collecting an exact-size iterator allocates once ✓ (Lesson 16: `(0..10).collect::<Vec<_>>()` had capacity 10).

### 5.3 `sum` can overflow

```rust
fn main() {
    let v: Vec<u8> = vec![200, 100];
    let s: u8 = v.iter().sum();
    println!("{s}");
}
```

✓ In a debug build: `panicked ... attempt to add with overflow` (from `core::iter::traits::accum`). In a release build, overflow checks are off by default and the sum would wrap (Lesson 04); I show that in the debugging challenge (✓ it printed `44`). Widen first, or use a checked fold ✓:

```rust
fn main() {
    let r = [1u8, 2, 3].iter().try_fold(0u8, |acc, &x| acc.checked_add(x));
    let r2 = [200u8, 100].iter().try_fold(0u8, |acc, &x| acc.checked_add(x));
    println!("{r:?} {r2:?}");
    let sum: u32 = [200u8, 100].iter().map(|&x| u32::from(x)).sum();
    println!("{sum}");
}
```

✓ Output `Some(6) None`, then `300`. `try_fold` is `fold` that stops early when the closure returns `None` or `Err` (it works with any type that implements the `Try` machinery behind `?`: `Option`, `Result`, ...). Use it whenever an accumulation can fail: money, sizes, counters coming from untrusted data.

### 5.4 `by_ref`, `take_while` and consumed items

Consumers take the iterator by value, so a second use needs `by_ref()` ✓:

```rust
fn main() {
    let mut it = (1..10).into_iter();
    let head: Vec<_> = it.by_ref().take(3).collect();
    let rest: Vec<_> = it.collect();
    println!("{head:?} {rest:?}");
    let mut it2 = 1..10;
    let a: Vec<_> = it2.by_ref().take_while(|&x| x < 4).collect();
    println!("{a:?} {:?}", it2.next());
}
```

✓ Output `[1, 2, 3] [4, 5, 6, 7, 8, 9]` and `[1, 2, 3] Some(5)`. The second line is the famous gotcha: `take_while` has to *read* the first failing item (4) to learn that it should stop, so that item is consumed and lost; the iterator resumes at 5. If you need to look without consuming, use `peekable()` (below).

### 5.5 More small ones ✓

```rust
fn main() {
    let mut cyc = [1, 2].iter().cycle();
    println!("{:?}", (0..5).map(|_| *cyc.next().unwrap()).collect::<Vec<_>>());
    println!("{:?}", [1, 2, 3].iter().zip(["a", "b"]).collect::<Vec<_>>());
    let mut f = [1].iter().fuse();
    println!("{:?} {:?} {:?}", f.next(), f.next(), f.next());
}
```

✓ Output `[1, 2, 1, 2, 1]`, `[(1, "a"), (2, "b")]` (zip stopped at the shorter side; the `3` was dropped), `Some(1) None None`.

---

## 6. Closures that borrow vs the borrow checker

The closure passed to an adapter may capture variables, and those captures follow Lesson 10's rules. You cannot mutate a collection you are iterating:

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    let doubled: Vec<i32> = v.iter().map(|x| { v.push(*x); x * 2 }).collect();
    println!("{doubled:?}");
}
```

✓ `error[E0502]: cannot borrow `v` as mutable because it is also borrowed as immutable`. Mutating something *else* from the closure is fine and common ✓:

```rust
fn main() {
    let v = vec![1, 2, 3];
    let mut log = Vec::new();
    let total: i32 = v.iter().map(|x| { log.push(*x); x * 2 }).sum();
    println!("{total} {log:?}");
}
```

✓ Output `12 [1, 2, 3]`. Side effects inside `map` are legal but easy to misread, especially given laziness (the effect happens only when and if the item is pulled). Prefer `for` or `for_each` for effects, `map` for pure transformations.

`sort_by_cached_key` is the iterator-flavored sort for expensive keys: it computes each key once (`["a", "bb", "ccc"]` ✓ from sorting by length).

---

## 7. `DoubleEndedIterator`, `ExactSizeIterator`, `size_hint`

Some iterators know more than "give me the next one":

- **`DoubleEndedIterator`** adds `next_back`, taking items from the other end. It makes `rev()` possible. Slices, ranges, `chars()` and `BTreeMap` iterators have it ✓ (`"héllo".chars().rev()` is `"olléh"`; `enumerate().rev()` works because slices are also exact-size).
- **`ExactSizeIterator`** adds `len()`: it promises that `size_hint` is exact.
- **`size_hint() -> (lower, Option<upper>)`** is a *hint* used for pre-allocation (`collect`, `extend`). It must be correct as a bound for safe code to be well-behaved, but a wrong hint is a logic bug, not memory unsafety (only the `unsafe` `TrustedLen` machinery in the standard library relies on it).

A custom iterator can implement all of them ✓:

```rust
struct Span { lo: u32, hi: u32 }
impl Iterator for Span {
    type Item = u32;
    fn next(&mut self) -> Option<u32> { if self.lo >= self.hi { None } else { self.lo += 1; Some(self.lo - 1) } }
    fn size_hint(&self) -> (usize, Option<usize>) { let n = (self.hi - self.lo) as usize; (n, Some(n)) }
}
impl DoubleEndedIterator for Span {
    fn next_back(&mut self) -> Option<u32> { if self.lo >= self.hi { None } else { self.hi -= 1; Some(self.hi) } }
}
impl ExactSizeIterator for Span {}
fn main() {
    let s = Span { lo: 1, hi: 5 };
    println!("{} {:?}", s.len(), s.rev().collect::<Vec<_>>());
    let mut s = Span { lo: 0, hi: 4 };
    println!("{:?} {:?} {:?}", s.next(), s.next_back(), s.collect::<Vec<_>>());
    println!("{:?}", Span { lo: 0, hi: 3 }.size_hint());
}
```

✓ Output `4 [4, 3, 2, 1]`, `Some(0) Some(3) [1, 2]`, `(3, Some(3))`. The front and back cursors meet in the middle: after taking 0 from the front and 3 from the back, only `[1, 2]` remain.

---

## 8. Writing your own iterators

### 8.1 A struct plus `next`

```rust
struct Countdown(u32);
impl Iterator for Countdown {
    type Item = u32;
    fn next(&mut self) -> Option<u32> {
        if self.0 == 0 { None } else { self.0 -= 1; Some(self.0 + 1) }
    }
}
struct Fib { a: u64, b: u64 }
impl Iterator for Fib {
    type Item = u64;
    fn next(&mut self) -> Option<u64> {
        let r = self.a;
        let n = self.a.checked_add(self.b)?;   // stop instead of overflowing
        self.a = self.b; self.b = n;
        Some(r)
    }
}
fn main() {
    println!("{:?}", Countdown(3).collect::<Vec<_>>());
    println!("{}", Countdown(4).map(|x| x * 2).sum::<u32>());
    println!("{:?}", Fib { a: 0, b: 1 }.take(10).collect::<Vec<_>>());
    for (i, x) in Countdown(2).enumerate() { print!("{i}:{x} "); }
    println!();
}
```

✓ Output `[3, 2, 1]`, `20`, `[0, 1, 1, 2, 3, 5, 8, 13, 21, 34]`, `0:2 1:1`. Notice that implementing `next` alone gave `map`, `sum`, `take`, `enumerate` and the rest. (This `Fib` also stops one term early because it checks the *next* sum before returning the current term; its `count()` was 92 in my run. Whether that is acceptable is a spec question; the point is to make overflow end the sequence rather than panic or wrap. `successors` in exercise 3 yields 93 terms.)

### 8.2 `IntoIterator`, `FromIterator`, `Extend` for your own collection

To make your own type work in `for` loops and with `collect`, implement the conversion traits ✓:

```rust
struct Bag { items: Vec<String> }
impl IntoIterator for Bag {
    type Item = String;
    type IntoIter = std::vec::IntoIter<String>;
    fn into_iter(self) -> Self::IntoIter { self.items.into_iter() }
}
impl<'a> IntoIterator for &'a Bag {
    type Item = &'a String;
    type IntoIter = std::slice::Iter<'a, String>;
    fn into_iter(self) -> Self::IntoIter { self.items.iter() }
}
impl FromIterator<String> for Bag {
    fn from_iter<I: IntoIterator<Item = String>>(it: I) -> Self { Bag { items: it.into_iter().collect() } }
}
impl Extend<String> for Bag {
    fn extend<I: IntoIterator<Item = String>>(&mut self, it: I) { self.items.extend(it) }
}
fn main() {
    let mut b: Bag = ["x", "y"].iter().map(|s| s.to_string()).collect();
    b.extend(["z".to_string()]);
    for s in &b { print!("{s} "); }
    println!();
    let lens: Vec<usize> = (&b).into_iter().map(|s| s.len()).collect();
    println!("{lens:?}");
    for s in b { print!("{s}! "); }
    println!();
}
```

✓ Output `x y z `, `[1, 1, 1]`, `x! y! z! `. The pattern is always three impls: by value (`Bag`), by shared reference (`&Bag`), and, if you want in-place edits, by `&mut Bag`. Delegate to the inner collection's iterator and name its type in `IntoIter` (or return `impl Iterator` from an inherent method when you do not want to commit to a type).

### 8.3 Iterators that borrow: lifetimes

An iterator may hand out references into data it borrows. The lifetime goes on the struct and on `Item` (Lesson 11) ✓:

```rust
struct Words<'a> { rest: &'a str }
impl<'a> Iterator for Words<'a> {
    type Item = &'a str;
    fn next(&mut self) -> Option<&'a str> {
        let s = self.rest.trim_start();
        if s.is_empty() { return None; }
        let end = s.find(char::is_whitespace).unwrap_or(s.len());
        let (w, rest) = s.split_at(end);
        self.rest = rest;
        Some(w)
    }
}
fn main() {
    let text = String::from("  the quick  brown fox ");
    let ws: Vec<&str> = Words { rest: &text }.collect();
    println!("{ws:?}");
    println!("{}", ws.iter().map(|w| w.len()).max().unwrap_or(0));
}
```

✓ Output `["the", "quick", "brown", "fox"]` and `5`. The items borrow from `text` for `'a`, not from the iterator, so they stay valid after the iterator is gone and you can collect them all at once. That works because `Item = &'a str` is tied to the *source* lifetime.

### 8.4 The limit: an iterator cannot lend out `&mut` windows of itself

If each item must borrow from the iterator itself (for example overlapping mutable windows into a buffer the iterator owns), `Iterator` cannot express it, because `Item` cannot depend on the lifetime of the `&mut self` borrow in `next`:

```rust
struct Windows<'a> { buf: &'a mut [i32], pos: usize }
impl<'a> Iterator for Windows<'a> {
    type Item = &'a mut [i32];
    fn next(&mut self) -> Option<Self::Item> {
        if self.pos + 2 > self.buf.len() { return None; }
        let w = &mut self.buf[self.pos..self.pos + 2];
        self.pos += 1;
        Some(w)
    }
}
fn main() {}
```

✓ `error: lifetime may not live long enough` ("method was supposed to return data with lifetime `'a` but it is returning data with lifetime `'1`"; no error code). This is the same aliasing rule as Lesson 10: two live `&mut` windows that overlap would alias. A "lending iterator" (an iterator whose items borrow from the iterator) needs generic associated types (GATs, Lesson 22) and is not in the standard library at the time of writing (from memory; the unstable situation may have changed). In practice use `chunks_mut`/`split_at_mut` for non-overlapping pieces, or an index loop for overlapping ones.

### 8.5 A `Peekable` lexer

`peekable()` lets you look at the next item before deciding whether to consume it, which is exactly what tokenizers and parsers need ✓:

```rust
#[derive(Debug, PartialEq)]
enum Tok { Num(u32), Plus, Star }
fn lex(s: &str) -> Result<Vec<Tok>, String> {
    let mut out = Vec::new();
    let mut it = s.chars().peekable();
    while let Some(&c) = it.peek() {
        match c {
            ' ' => { it.next(); }
            '+' => { out.push(Tok::Plus); it.next(); }
            '*' => { out.push(Tok::Star); it.next(); }
            '0'..='9' => {
                let mut n = 0u32;
                while let Some(d) = it.next_if(|c| c.is_ascii_digit()) {
                    n = n.checked_mul(10).and_then(|n| n.checked_add(d.to_digit(10).unwrap())).ok_or("overflow")?;
                }
                out.push(Tok::Num(n));
            }
            other => return Err(format!("bad char {other:?}")),
        }
    }
    Ok(out)
}
fn main() {
    println!("{:?}", lex("12 + 3*45"));
    println!("{:?}", lex("1 - 2"));
    println!("{:?}", lex("99999999999"));
}
```

✓ Output:

```text
Ok([Num(12), Plus, Num(3), Star, Num(45)])
Err("bad char '-'")
Err("overflow")
```

`next_if(pred)` consumes the next item only when the predicate holds. The `.unwrap()` is safe because `is_ascii_digit` was just checked; the `checked_*` calls turn overflow into an error instead of a panic. The practical challenge turns this into a proper `Iterator` of `Result`s.

---

## 9. Returning iterators from functions

You cannot name the type of `v.iter().filter(..)` (it contains a closure type), so return it as `impl Iterator` (static dispatch, one concrete type) or `Box<dyn Iterator>` (dynamic dispatch, any type) ✓:

```rust
fn evens(v: &[i32]) -> impl Iterator<Item = i32> + '_ { v.iter().copied().filter(|x| x % 2 == 0) }
fn evens_2024(v: &[i32]) -> impl Iterator<Item = &i32> { v.iter().filter(|x| **x % 2 == 0) }
fn boxed(flag: bool) -> Box<dyn Iterator<Item = u32>> {
    if flag { Box::new(0..3) } else { Box::new([10, 20].into_iter()) }
}
fn main() {
    let v = vec![1, 2, 3, 4];
    println!("{:?} {:?}", evens(&v).collect::<Vec<_>>(), evens_2024(&v).collect::<Vec<_>>());
    println!("{:?} {:?}", boxed(true).collect::<Vec<_>>(), boxed(false).collect::<Vec<_>>());
}
```

✓ Output `[2, 4] [2, 4]` and `[0, 1, 2] [10, 20]`. In edition 2024 a return-position `impl Trait` captures all in-scope lifetimes automatically (Lesson 11), so `+ '_` is optional here; in edition 2021 it was required to avoid `E0700`.

`impl Iterator` means *one* concrete type, chosen by the function body. Returning two different adapter types from `if`/`else` fails ✓:

```rust
fn bad(flag: bool) -> impl Iterator<Item = u32> {
    if flag { (0..3).map(|x| x) } else { (5..8).map(|x| x + 1) }
}
fn main() { let _ = bad(true); }
```

✓ `error[E0308]: `if` and `else` have incompatible types`, "expected closure, found a different closure" and "no two closures, even if identical, have the same type". Fixes ✓: return `Box<dyn Iterator<Item = u32>>` (a heap allocation and a virtual call per `next`), or restructure so both branches have the same type (compute the parameters in the branch and build one iterator):

```rust
fn good_box(flag: bool) -> Box<dyn Iterator<Item = u32>> {
    if flag { Box::new((0..3).map(|x| x)) } else { Box::new((5..8).map(|x| x + 1)) }
}
fn good_same(flag: bool) -> impl Iterator<Item = u32> {
    let (lo, hi, add) = if flag { (0, 3, 0) } else { (5, 8, 1) };
    (lo..hi).map(move |x| x + add)
}
fn main() { println!("{:?} {:?}", good_box(false).collect::<Vec<_>>(), good_same(true).collect::<Vec<_>>()); }
```

✓ Output `[6, 7, 8] [0, 1, 2]`.

---

## 10. How it compiles, and what it costs

An adapter is an ordinary struct that holds the previous iterator and (if it takes one) a closure. Its `next` calls the inner `next`. So the type of a chain nests: `v.iter().map(f).filter(g)` is a `Filter<Map<Iter<T>, F>, G>`. Because non-capturing closures are zero-sized (Lesson 09), the whole chain is often just the inner iterator's size ✓:

```rust
use std::mem::size_of_val;
fn main() {
    let v = vec![1u32, 2, 3];
    let it = v.iter().map(|x| x * 2).filter(|x| x % 3 != 0);
    println!("{}", size_of_val(&it));
    let k = 5u32;
    let it2 = v.iter().map(move |x| x * k);
    println!("{}", size_of_val(&it2));
    let it3 = v.iter().enumerate().skip(1).take(2);
    println!("{}", size_of_val(&it3));
    println!("{}", size_of_val(&v.iter()));
    println!("{}", size_of_val(&(0..10u32)));
}
```

✓ Output `16`, `24`, `40`, `16`, `8`. The slice iterator is 16 bytes (a start and an end pointer); two non-capturing closures add nothing (16); a closure capturing a `u32` adds 4 plus padding (24); `enumerate`, `skip` and `take` each add a counter (3 × 8 = 24, total 40); a `Range<u32>` is two `u32`s (8). These sizes are for this toolchain and are not guaranteed.

Because every adapter and closure type is known at compile time, the compiler monomorphizes and inlines the chain into one loop; with optimizations it often removes the bounds checks that an index-based loop would perform. A small check (compiled with `-O`, x86-64) of two functions that sum `x * 2` for `x % 3 != 0` over a `&[u32]`, one with an iterator chain and one with an indexed `for` loop ✓:

```rust
#[inline(never)]
pub fn sum_iter(v: &[u32]) -> u32 { v.iter().map(|x| x * 2).filter(|x| x % 3 != 0).sum() }
#[inline(never)]
pub fn sum_loop(v: &[u32]) -> u32 {
    let mut s = 0u32;
    for i in 0..v.len() { let x = v[i] * 2; if x % 3 != 0 { s += x; } }
    s
}
fn main() {
    let v: Vec<u32> = (0..100).collect();
    println!("{} {}", sum_iter(&v), sum_loop(&v));
}
```

✓ Both return `6534`. In the `--emit asm -O` output each function body was **66 instructions**, with 37 using SSE (`xmm`) registers (so both were auto-vectorized) and no calls to panic or bounds-check helpers. I counted instructions; I did not diff them line by line. This is one small example on one compiler version, not a promise about every chain: iterator code is *usually* as fast as the hand loop, and sometimes faster (no bounds checks), but inspect or benchmark hot paths (Lesson 41) instead of assuming.

Practical performance rules:

- Avoid collecting into an intermediate `Vec` just to iterate again; keep the chain lazy until the final consumer.
- Prefer `iter()` over `into_iter()` plus `clone()`; prefer `copied()` for `Copy` types.
- `chars().rev()`, `windows`, `chunks` and `zip` are cheap; `last()` on a non-double-ended iterator walks the whole thing; `nth(n)` is O(n) unless the iterator overrides it (slices do).
- `rev()` on a `Vec` iterator costs nothing; `rev()` on a `Chain` or `Filter` works but is only available when the pieces are double-ended.
- `Box<dyn Iterator>` costs an allocation and an indirect call per item; use `impl Iterator` when you can.
- `flat_map` and `flatten` are fine, but `Option::iter()`/`into_iter()` inside `flat_map` is a common way to write `filter_map`.
- The `collect` target decides the allocation strategy; `collect::<String>()` of `char`s reserves from the size hint.
- Debug builds do not inline as aggressively, so iterator-heavy code can be much slower with `cargo run` than `cargo run --release`. Benchmark release builds only.

---

## 11. Common mistakes and edge cases

- **Forgetting to consume** an adapter (`map` without `collect`/`for_each`): nothing runs ✓ (`unused Map` warning).
- **Type annotations needed** for `collect`/`sum`/`product`/`partition` (E0283 ✓). Use `let x: T` or turbofish.
- **Mutating the collection while iterating** it (E0502 ✓). Collect indices first, `retain`, `drain`, or build a new collection.
- **Using a collection after `for x in v`** (E0382 ✓). Iterate `&v`.
- **Reference-level mismatches** (`&&i32`): add `.copied()`/`.cloned()` or match with `|&&x|`.
- **`take_while` swallows one item** ✓; use `peekable`+`next_if` or `map_while` when you must not lose it.
- **`sum` overflow**: panics in debug, wraps in release ✓. Widen or use `try_fold`/`checked_*`.
- **`max`/`min` on floats** (E0277 ✓): use `total_cmp` or `fold(f64::max)`; note `fold(f64::NEG_INFINITY, f64::max)` ignores a `NaN` item (documented behavior of `f64::max`), which may hide bad data.
- **`filter_map(|s| s.parse().ok())`** silently drops errors; if bad input is a bug, collect into `Result`.
- **Infinite iterators** (`repeat`, `cycle`, `(1..)`) with `collect`, `count`, `last` or `sum`: never terminates or exhausts memory. Always bound them.
- **Side effects in `map`** run lazily, possibly never, possibly more than once if you clone the iterator.
- **Cloning an iterator clones its state**, including captured data; `cycle` requires it.
- **`dedup`-like logic with `windows`** panics on short inputs if you index `w[1]` assuming length; `windows(2)` on an empty or one-element slice yields nothing (safe), but `chunks_exact` drops the remainder.
- **Returning different adapter types** from branches (E0308 ✓). Use `Box<dyn Iterator>` or one shape.
- **Relying on `size_hint`** for correctness: it is only a hint for capacity.

---

## 12. Security and reliability

- **Unbounded input produces unbounded output.** `lines()`, `collect()` and `read_to_end` on attacker-controlled data fill memory. Bound with `take(n)`, check lengths before reserving, and process streams incrementally instead of collecting them.
- **Infinite or very long iterators** in request handlers are a denial-of-service vector. Put an explicit cap on repetition counts you accept from callers.
- **Silent data loss.** `filter_map(.ok())`, `flatten` over `Result`s, and `zip` with unequal lengths all discard data without telling you ✓ (zip dropped the `3`). In validation, parsing or financial code, prefer `collect::<Result<_, _>>()` and assert equal lengths.
- **Arithmetic in accumulators.** `sum`/`product` overflow panics (debug) or wraps (release) ✓. A wrapped total in a balance or fee calculation is a security bug; use `checked_*`/`try_fold`/wider types (Lesson 04).
- **Panics inside closures** (`unwrap`, indexing) abort the whole pipeline; with untrusted inputs return `Result` or use `get`.
- **Parsers that are iterators** should be total: every input, including invalid UTF-8-derived text, odd Unicode and huge numbers, must end in a value or an error, never a panic or an infinite loop. The practical challenge's lexer slices only at `char_indices` boundaries, uses `checked_*` for numbers, stops after the first error, and was exercised on a handful of odd Unicode inputs (not a fuzz test; real fuzzing is Lesson 40).
- **Order nondeterminism** from iterating a `HashMap` leaks into your output (Lesson 16): sort or use `BTreeMap` before anything that is signed, hashed or compared.
- Rust's guarantees (no data races, no use-after-free through iterator invalidation) do not stop logic bugs such as off-by-one windows, silent drops, or unbounded collection.

---

## 13. Exercises

Try each before opening the answer. They are not marked complete unless you tell me you did them.

1. **(Easy)** What does this print, and in what order? `let it = v.iter().map(|x| { println!("map {x}"); x * 10 }); println!("built"); let out: Vec<i32> = it.collect();` for `v = [1, 2, 3, 4]`.
2. **(Easy)** Fix the `E0283` in `let (a, b) = v.iter().partition::<Vec<_>, _>(|&&x| x % 2 == 0);`.
3. **(Medium)** With `std::iter::successors`, generate the Fibonacci numbers below 100, and compute the sum of the even Fibonacci numbers below 4,000,000.
4. **(Medium)** Write run-length encoding: `rle("aaabcc") == [('a', 3), ('b', 1), ('c', 2)]`, first with a loop, then as an iterator using `peekable`.
5. **(Medium)** Given `&[&str]`, (a) sum them as `i64`, failing on the first bad item; (b) sum only the valid ones and report how many were invalid.
6. **(Hard)** Transpose a rectangular `&[Vec<T>]` with iterators, returning `None` for a ragged or empty input.
7. **(Hard)** Make `Span { lo, hi }` yield `lo..hi` and also work with `.rev()` and `.len()`. Which traits do you implement?
8. **(Hard)** Write a function that returns *either* `0..3` mapped through `x` *or* `5..8` mapped through `x + 1`, as `impl Iterator<Item = u32>`. Why does the obvious version fail, and what are two fixes?

<details markdown="1">
<summary>Answers</summary>

1. ✓ `built` first (nothing runs while building), then `map 1`, `map 2`, `map 3`, `map 4` as `collect` pulls each item, giving `[10, 20, 30, 40]`.
2. ✓ The `_` inside `Vec<_>` is ambiguous because both `Vec<&i32>` and `Vec<i32>` can be extended from `&i32`. Either name the type, `let (a, b): (Vec<&i32>, Vec<&i32>) = v.iter().partition(|&&x| x % 2 == 0);`, or copy the items first: `let (a, b): (Vec<i32>, Vec<i32>) = v.iter().copied().partition(|x| x % 2 == 0);`. Both printed `[4] [3, 1]` for `[3, 1, 4]`.
3. ✓ (run)
   ```rust
   fn fib_iter() -> impl Iterator<Item = u64> + Clone {
       std::iter::successors(Some((0u64, 1u64)), |&(a, b)| Some((b, a.checked_add(b)?))).map(|(a, _)| a)
   }
   // fib_iter().take_while(|&x| x < 100).collect::<Vec<_>>()
   //   == [0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89]
   // fib_iter().take_while(|&x| x < 4_000_000).filter(|x| x % 2 == 0).sum::<u64>() == 4613732
   ```
   The state is a pair `(a, b)`; `checked_add(b)?` inside the closure ends the sequence on overflow instead of panicking. `fib_iter().count()` is 93 on this machine (the terms that fit in a `u64`).
4. ✓ (both versions run and agree on `"aaabcc"`, the empty string and `"ééa"`)
   ```rust
   fn rle(s: &str) -> Vec<(char, usize)> {
       let mut out: Vec<(char, usize)> = Vec::new();
       for c in s.chars() {
           match out.last_mut() {
               Some((last, n)) if *last == c => *n += 1,
               _ => out.push((c, 1)),
           }
       }
       out
   }
   fn rle_iter(s: &str) -> Vec<(char, usize)> {
       let mut it = s.chars().peekable();
       std::iter::from_fn(|| {
           let c = it.next()?;
           let mut n = 1;
           while it.next_if_eq(&c).is_some() { n += 1; }
           Some((c, n))
       }).collect()
   }
   ```
   `rle("aaabcc")` is `[('a', 3), ('b', 1), ('c', 2)]`.
5. ✓ (run)
   ```rust
   fn sum_all(items: &[&str]) -> Result<i64, std::num::ParseIntError> {
       items.iter().map(|s| s.trim().parse::<i64>()).sum()
   }
   fn sum_valid(items: &[&str]) -> (i64, usize) {
       let (ok, bad): (Vec<_>, Vec<_>) = items.iter().map(|s| s.trim().parse::<i64>()).partition(Result::is_ok);
       (ok.into_iter().map(Result::unwrap).sum(), bad.len())
   }
   ```
   `sum_all(&["1", " 2 ", "3"])` is `Ok(6)`; `sum_all(&["1", "x", "y"])` is `Err(ParseIntError { kind: InvalidDigit })` (the first failure); `sum_valid(&["1", "x", "3"])` is `(4, 1)`. The `unwrap` is safe because only `Ok` values were partitioned in (a `filter_map(Result::ok)` plus a separate error count is the other common shape). `sum::<i64>()` can still overflow on huge inputs; use `try_fold` with `checked_add` if that matters.
6. ✓ (run: `[[1,2,3],[4,5,6]]` becomes `[[1,4],[2,5],[3,6]]`; a ragged input gives `None`; an empty outer slice gives `None`)
   ```rust
   fn transpose<T: Clone>(m: &[Vec<T>]) -> Option<Vec<Vec<T>>> {
       let cols = m.first()?.len();
       if m.iter().any(|r| r.len() != cols) { return None; }
       Some((0..cols).map(|c| m.iter().map(|row| row[c].clone()).collect()).collect())
   }
   ```
   The ragged check guarantees `row[c]` is in bounds. A matrix of one empty row transposes to `Some([])` in my run. Whether an empty input should be `None` or an empty matrix is a design decision; here it is `None` because there is no first row to take the width from.
7. ✓ (run) Implement `Iterator` (with `next` and an exact `size_hint`), `DoubleEndedIterator` (`next_back` decrements `hi`), and the marker `ExactSizeIterator` (it needs the exact `size_hint` and gives `len()`). See Section 7: `Span { lo: 1, hi: 5 }` had `len() == 4` and `.rev()` gave `[4, 3, 2, 1]`.
8. ✓ Each closure has its own type, so the two `Map<Range<u32>, closure>` types differ and `impl Iterator` (one concrete type) can't be both (`E0308`). Fix 1: `Box<dyn Iterator<Item = u32>>` and `Box::new(..)` each branch. Fix 2: compute the parameters in the branch and build one iterator, `let (lo, hi, add) = if flag { (0, 3, 0) } else { (5, 8, 1) }; (lo..hi).map(move |x| x + add)`. Another is an `Either`-style enum implementing `Iterator` (external crates provide one; not shown here).

</details>

---

## 14. Debugging challenge

A pricing script should print every price, count them, and total them:

```rust
fn main() {
    let prices: Vec<u8> = vec![200, 100];
    let mut count = 0;
    prices.iter().map(|p| { count += 1; println!("{p}"); });
    let total: u8 = prices.iter().sum();
    println!("{count} {total}");
}
```

Run in a debug build ✓ it prints nothing and panics; in a release build ✓ it prints `0 44`. Find **both** bugs (and note that the compiler warned about one of them).

<details markdown="1">
<summary>Solution</summary>

**Bug 1: the `map` never runs.** `map` is lazy, and nothing consumes the iterator, so `count` stays `0` and nothing is printed. The compiler tells you: ✓ `warning: unused `Map` that must be used` (and a second warning about `Iterator::map` discarding values). Use a `for` loop (or `for_each`) for side effects, and keep `map` for pure transformations.

**Bug 2: `u8` overflow in `sum`.** `200 + 100` is 300, which does not fit in `u8`. In a debug build ✓ it panics with `attempt to add with overflow`; in a release build overflow checks are off by default, so the result silently wraps to `300 - 256 = 44` ✓. A silently wrong total is the dangerous one. Widen before summing, or accumulate with checks:

```rust
fn main() {
    let prices: Vec<u8> = vec![200, 100];
    let mut count = 0;
    for p in &prices { count += 1; println!("{p}"); }
    let total: u32 = prices.iter().map(|&p| u32::from(p)).sum();
    let checked: Option<u8> = prices.iter().try_fold(0u8, |a, &p| a.checked_add(p));
    println!("{count} {total} {checked:?}");
}
```

✓ Output `200`, `100`, then `2 300 None`: both prices printed, the count is right, the widened total is correct, and the checked `u8` accumulation reports the overflow as `None` instead of wrapping. Better still: choose a type that can hold the domain (money as `u64` cents), not `u8`.

</details>

---

## 15. Practical challenge: `iterlab`

Create a library crate `iterlab` (edition 2024) with:

1. An extension trait `IterExt` (implemented for every `Iterator`) with:
   - `chunked(NonZeroUsize) -> Chunked<Self>` yielding `Vec`s of up to `n` items, with a correct `size_hint`, which must not pre-allocate `n` slots when `n` is huge.
   - `dedup_consecutive() -> DedupConsecutive<Self>` collapsing runs of equal items; items need `PartialEq` but **not** `Clone`; it must be safe to call after the inner iterator ends.
   - `interleave(other) -> Interleave<Self, J::IntoIter>` alternating between two iterators, then draining whichever remains.
2. A `Lexer<'a>` that is an `Iterator<Item = Result<Token<'a>, LexError>>` over a tiny arithmetic language: `u64` numbers, identifiers (borrowing from the input), `+ - * / ( )`, whitespace skipped. Errors carry **byte offsets**; after the first error the iterator yields nothing more; numbers use `checked_*` arithmetic; non-ASCII input must never panic. `lex_all(&str) -> Result<Vec<Token>, LexError>` is one `collect`. `LexError` implements `Display` and `Error`.
3. Tests: chunk sizes and `size_hint`; the huge-chunk case; dedup keeps non-adjacent repeats and works on non-`Clone` items; interleave with uneven lengths and empty inputs; **laziness** (an adapter pulls nothing until `next`, and one chunk pulls exactly one chunk); every token kind; Unicode identifiers; error offsets after multi-byte characters; the maximum `u64` and one past it; nothing after an error; and no panics on a handful of odd Unicode strings. A usage doctest.
4. It must pass `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, and `cargo test --locked` (run `cargo generate-lockfile` once first).

<details markdown="1">
<summary>Solution</summary>

**`src/lib.rs`**

```rust
//! Custom iterator adapters, an extension trait, and a lexer that is an iterator.
//!
//! ```
//! use iterlab::{lex_all, IterExt, Token};
//! use std::num::NonZeroUsize;
//!
//! let n = NonZeroUsize::new(2).unwrap();
//! let chunks: Vec<Vec<i32>> = (1..=5).chunked(n).collect();
//! assert_eq!(chunks, vec![vec![1, 2], vec![3, 4], vec![5]]);
//!
//! let runs: Vec<i32> = [1, 1, 2, 2, 2, 1].into_iter().dedup_consecutive().collect();
//! assert_eq!(runs, vec![1, 2, 1]);
//!
//! assert_eq!(lex_all("1 + x").unwrap(), vec![Token::Num(1), Token::Plus, Token::Ident("x")]);
//! ```

use std::fmt;
use std::iter::{Fuse, Peekable};
use std::num::NonZeroUsize;
use std::str::CharIndices;

/// Adapter returned by [`IterExt::chunked`]: yields `Vec`s of up to `size` items.
#[derive(Debug, Clone)]
pub struct Chunked<I> {
    iter: I,
    size: usize,
}

impl<I: Iterator> Iterator for Chunked<I> {
    type Item = Vec<I::Item>;

    fn next(&mut self) -> Option<Self::Item> {
        // `Take`'s size hint is capped at `size`, so `collect` never reserves more than that.
        let chunk: Vec<I::Item> = self.iter.by_ref().take(self.size).collect();
        if chunk.is_empty() { None } else { Some(chunk) }
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        let (lo, hi) = self.iter.size_hint();
        (lo.div_ceil(self.size), hi.map(|h| h.div_ceil(self.size)))
    }
}

/// Adapter returned by [`IterExt::dedup_consecutive`]: collapses runs of equal items.
#[derive(Debug, Clone)]
pub struct DedupConsecutive<I: Iterator> {
    iter: Fuse<I>,
    pending: Option<I::Item>,
}

impl<I: Iterator> Iterator for DedupConsecutive<I>
where
    I::Item: PartialEq,
{
    type Item = I::Item;

    fn next(&mut self) -> Option<Self::Item> {
        let first = self.pending.take().or_else(|| self.iter.next())?;
        loop {
            match self.iter.next() {
                Some(x) if x == first => continue,
                Some(x) => {
                    self.pending = Some(x);
                    return Some(first);
                }
                None => return Some(first),
            }
        }
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        let (_, hi) = self.iter.size_hint();
        let pending = usize::from(self.pending.is_some());
        // At least one item remains if anything is pending; at most everything is distinct.
        (pending, hi.and_then(|h| h.checked_add(pending)))
    }
}

/// Adapter returned by [`IterExt::interleave`]: alternates between two iterators,
/// then continues with whichever one still has items.
#[derive(Debug, Clone)]
pub struct Interleave<A, B> {
    a: Fuse<A>,
    b: Fuse<B>,
    take_a: bool,
}

impl<A, B> Iterator for Interleave<A, B>
where
    A: Iterator,
    B: Iterator<Item = A::Item>,
{
    type Item = A::Item;

    fn next(&mut self) -> Option<Self::Item> {
        self.take_a = !self.take_a;
        if self.take_a {
            self.a.next().or_else(|| self.b.next())
        } else {
            self.b.next().or_else(|| self.a.next())
        }
    }
}

/// Extension methods available on every iterator.
pub trait IterExt: Iterator + Sized {
    /// Groups items into `Vec`s of at most `size` (the last may be shorter).
    fn chunked(self, size: NonZeroUsize) -> Chunked<Self> {
        Chunked {
            iter: self,
            size: size.get(),
        }
    }

    /// Collapses consecutive equal items into one.
    fn dedup_consecutive(self) -> DedupConsecutive<Self>
    where
        Self::Item: PartialEq,
    {
        DedupConsecutive {
            iter: self.fuse(),
            pending: None,
        }
    }

    /// Alternates items from `self` and `other`.
    fn interleave<J>(self, other: J) -> Interleave<Self, J::IntoIter>
    where
        J: IntoIterator<Item = Self::Item>,
    {
        Interleave {
            a: self.fuse(),
            b: other.into_iter().fuse(),
            take_a: false,
        }
    }
}

impl<I: Iterator> IterExt for I {}

/// A token of a tiny arithmetic language. Identifiers borrow from the source text.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Token<'a> {
    Num(u64),
    Ident(&'a str),
    Plus,
    Minus,
    Star,
    Slash,
    LParen,
    RParen,
}

/// Why lexing stopped. `at` is a byte offset into the source.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum LexError {
    UnexpectedChar { ch: char, at: usize },
    NumberTooLarge { at: usize },
}

impl fmt::Display for LexError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            LexError::UnexpectedChar { ch, at } => write!(f, "unexpected {ch:?} at byte {at}"),
            LexError::NumberTooLarge { at } => {
                write!(f, "number starting at byte {at} is too large")
            }
        }
    }
}

impl std::error::Error for LexError {}

/// A lexer that is an iterator of `Result<Token, LexError>`.
/// After the first error it yields nothing more.
#[derive(Debug, Clone)]
pub struct Lexer<'a> {
    src: &'a str,
    chars: Peekable<CharIndices<'a>>,
    done: bool,
}

impl<'a> Lexer<'a> {
    /// Creates a lexer over `src`.
    pub fn new(src: &'a str) -> Self {
        Lexer {
            src,
            chars: src.char_indices().peekable(),
            done: false,
        }
    }

    fn fail(&mut self, err: LexError) -> Option<Result<Token<'a>, LexError>> {
        self.done = true;
        Some(Err(err))
    }
}

impl<'a> Iterator for Lexer<'a> {
    type Item = Result<Token<'a>, LexError>;

    fn next(&mut self) -> Option<Self::Item> {
        if self.done {
            return None;
        }
        while self.chars.next_if(|&(_, c)| c.is_whitespace()).is_some() {}
        let (start, c) = self.chars.next()?;
        let token = match c {
            '+' => Token::Plus,
            '-' => Token::Minus,
            '*' => Token::Star,
            '/' => Token::Slash,
            '(' => Token::LParen,
            ')' => Token::RParen,
            '0'..='9' => {
                let mut n = u64::from(c as u8 - b'0');
                while let Some((_, d)) = self.chars.next_if(|&(_, d)| d.is_ascii_digit()) {
                    let digit = u64::from(d as u8 - b'0');
                    match n.checked_mul(10).and_then(|n| n.checked_add(digit)) {
                        Some(next) => n = next,
                        None => return self.fail(LexError::NumberTooLarge { at: start }),
                    }
                }
                Token::Num(n)
            }
            c if c.is_alphabetic() || c == '_' => {
                let mut end = start + c.len_utf8();
                while let Some((j, d)) = self
                    .chars
                    .next_if(|&(_, d)| d.is_alphanumeric() || d == '_')
                {
                    end = j + d.len_utf8();
                }
                Token::Ident(&self.src[start..end])
            }
            other => {
                return self.fail(LexError::UnexpectedChar {
                    ch: other,
                    at: start,
                });
            }
        };
        Some(Ok(token))
    }
}

/// Lexes the whole input, stopping at the first error.
pub fn lex_all(src: &str) -> Result<Vec<Token<'_>>, LexError> {
    Lexer::new(src).collect()
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::cell::Cell;

    fn n(x: usize) -> NonZeroUsize {
        NonZeroUsize::new(x).unwrap()
    }

    #[test]
    fn chunked_groups_and_sizes() {
        let v: Vec<Vec<i32>> = (1..=7).chunked(n(3)).collect();
        assert_eq!(v, vec![vec![1, 2, 3], vec![4, 5, 6], vec![7]]);
        assert_eq!((0..0).chunked(n(3)).count(), 0);
        assert_eq!((1..=7).chunked(n(3)).size_hint(), (3, Some(3)));
        assert_eq!((1..=6).chunked(n(3)).size_hint(), (2, Some(2)));
        // A huge chunk size must not preallocate that much memory.
        let big: Vec<Vec<u8>> = [1u8, 2].into_iter().chunked(n(usize::MAX)).collect();
        assert_eq!(big, vec![vec![1, 2]]);
    }

    #[test]
    fn dedup_consecutive_keeps_non_adjacent_repeats() {
        let v: Vec<_> = [1, 1, 2, 2, 2, 1, 3, 3]
            .into_iter()
            .dedup_consecutive()
            .collect();
        assert_eq!(v, vec![1, 2, 1, 3]);
        assert_eq!(std::iter::empty::<i32>().dedup_consecutive().next(), None);
        assert_eq!(["a"].into_iter().dedup_consecutive().count(), 1);
        // Works for items that are not Clone.
        #[derive(PartialEq, Debug)]
        struct NoClone(u8);
        let v: Vec<_> = vec![NoClone(1), NoClone(1), NoClone(2)]
            .into_iter()
            .dedup_consecutive()
            .collect();
        assert_eq!(v, vec![NoClone(1), NoClone(2)]);
    }

    #[test]
    fn interleave_alternates_then_drains() {
        let v: Vec<_> = [1, 3, 5].into_iter().interleave([2, 4]).collect();
        assert_eq!(v, vec![1, 2, 3, 4, 5]);
        let v: Vec<_> = [1].into_iter().interleave([2, 4, 6]).collect();
        assert_eq!(v, vec![1, 2, 4, 6]);
        assert_eq!(
            Vec::<i32>::new().into_iter().interleave(Vec::new()).count(),
            0
        );
    }

    #[test]
    fn adapters_are_lazy() {
        let pulled = Cell::new(0);
        let source = (1..=100).inspect(|_| pulled.set(pulled.get() + 1));
        let mut chunks = source.chunked(n(4));
        assert_eq!(pulled.get(), 0, "building the adapter pulls nothing");
        assert_eq!(chunks.next(), Some(vec![1, 2, 3, 4]));
        assert_eq!(pulled.get(), 4, "one chunk pulls exactly one chunk");
    }

    #[test]
    fn lexer_tokens_and_borrowed_identifiers() {
        let toks = lex_all("12 + foo_1*(x - 3)/ é2").unwrap();
        assert_eq!(
            toks,
            vec![
                Token::Num(12),
                Token::Plus,
                Token::Ident("foo_1"),
                Token::Star,
                Token::LParen,
                Token::Ident("x"),
                Token::Minus,
                Token::Num(3),
                Token::RParen,
                Token::Slash,
                Token::Ident("é2"),
            ]
        );
        assert_eq!(lex_all("").unwrap(), vec![]);
        assert_eq!(lex_all("   ").unwrap(), vec![]);
    }

    #[test]
    fn lexer_errors_carry_byte_offsets_and_stop() {
        assert_eq!(
            lex_all("1 + $"),
            Err(LexError::UnexpectedChar { ch: '$', at: 4 })
        );
        assert_eq!(
            lex_all("é + #"),
            Err(LexError::UnexpectedChar { ch: '#', at: 5 })
        );
        assert_eq!(
            lex_all("99999999999999999999"),
            Err(LexError::NumberTooLarge { at: 0 })
        );
        assert_eq!(
            lex_all("18446744073709551615"),
            Ok(vec![Token::Num(u64::MAX)])
        );
        let mut lx = Lexer::new("1 ? 2");
        assert_eq!(lx.next(), Some(Ok(Token::Num(1))));
        assert!(matches!(lx.next(), Some(Err(_))));
        assert_eq!(lx.next(), None, "nothing after an error");
        let msg = lex_all("?").unwrap_err().to_string();
        assert_eq!(msg, "unexpected '?' at byte 0");
    }

    #[test]
    fn lexer_never_panics_on_arbitrary_text() {
        for s in ["\u{0}", "🙂", "1🙂", "a\u{301}b", "٣", "x٣"] {
            let _ = Lexer::new(s).count();
        }
    }
}
```

✓ With this exact file: `cargo fmt --check` clean, `cargo clippy --all-targets --locked -- -D warnings` clean, `cargo test --locked` → 7 unit tests and 1 doctest passed.

**Design notes.**

- **Adapters are structs that wrap an iterator.** `Chunked`, `DedupConsecutive` and `Interleave` each hold their inner iterator(s) and implement `next`; the extension trait adds a method to every iterator through a blanket `impl<I: Iterator> IterExt for I`. That is exactly how `std`'s own `map`/`filter` are built.
- **`Chunked::next`** uses `by_ref().take(size).collect()`. `Take`'s size hint is capped at `size`, so `collect` reserves at most what the inner iterator reports it has: the test with `usize::MAX` chunks over two items does not try to allocate an enormous buffer. (Calling `Vec::with_capacity(size)` instead would be a denial-of-service bug.) Its `size_hint` divides the inner hint with `div_ceil`.
- **`DedupConsecutive`** stores the next *different* item in `pending` so no `Clone` is needed: it reads ahead until the value changes, returns the first of the run, and keeps the one that ended it. The inner iterator is wrapped in `Fuse` because it may be called again after returning `None` (it is, when `pending` is empty and the iterator is exhausted).
- **`Interleave`** flips a flag each call and falls back to the other iterator with `or_else` when the preferred side is exhausted, so uneven lengths drain naturally. Both sides are `Fuse`d so an exhausted side keeps answering `None`.
- **The lexer is an iterator**, so `lex_all` is just `collect::<Result<Vec<_>, _>>()` (Section 5.2), and callers can also stream tokens without allocating. `Token::Ident(&'a str)` borrows from the source (Section 8.3); the slice bounds come from `char_indices` plus `len_utf8`, so they always fall on character boundaries.
- **Totality.** Numbers use `checked_mul`/`checked_add`; an unknown character or overflow sets `done`, returns one `Err`, and every later `next` returns `None`. `u64::MAX` lexes and one past it is `NumberTooLarge`. The odd-Unicode test only runs a few inputs; it is not fuzzing.
- **`size_hint` for `Lexer`** is the default `(0, None)`, which is honest because the number of tokens is unknown.
- **Not a full language.** There is no parser, no float/negative literal support, no spans beyond a start offset, and error messages are minimal.

</details>

---

## 16. Interview questions

<details markdown="1">
<summary>Answers</summary>

1. **What is the `Iterator` trait?** One associated type `Item` and one required method `next(&mut self) -> Option<Self::Item>`; every adapter and consumer is a provided method built on `next`.
2. **What does it mean that iterators are lazy?** Building an adapter chain does no work; elements are produced only when a consumer (`collect`, `sum`, `for`, ...) calls `next`. Unused adapters trigger an `unused_must_use` warning.
3. **How does a `for` loop work?** It calls `IntoIterator::into_iter` on the expression once, then loops calling `next` until `None`.
4. **`iter()` vs `iter_mut()` vs `into_iter()`?** Shared references, mutable references, and owned values (consuming the collection), respectively; `for x in &v`, `&mut v` and `v` map to them.
5. **`map` vs `for_each`?** `map` is a lazy adapter that returns an iterator and must be consumed; `for_each` is a consumer for side effects. Use `for` for effects when you may need `break`/`?`.
6. **How does `collect` know what to build?** It is generic over the return type `B: FromIterator<Item>`, so the annotation or turbofish decides (`Vec`, `HashMap`, `String`, `Result<Vec<_>, _>`, ...).
7. **What does `collect::<Result<Vec<_>, _>>()` do?** Collects items if all are `Ok`, but returns the first `Err` and stops pulling more.
8. **`filter_map(|x| x.parse().ok())` vs collecting a `Result`?** The first silently skips failures; the second fails the whole operation. Pick based on whether bad data is acceptable.
9. **Why can `take_while` lose an element?** It must consume the first failing element to discover that it should stop, and that element is not returned. Use `peekable` + `next_if` or `map_while` plus care.
10. **What is `by_ref` for?** Consumers take the iterator by value; `by_ref` lends it by `&mut` so you can keep using it afterwards.
11. **What is a double-ended iterator?** One that implements `next_back` so you can iterate from both ends; required by `rev()`.
12. **What's `size_hint` for?** A `(lower, Option<upper>)` estimate of remaining items, used to pre-allocate in `collect`/`extend`; `ExactSizeIterator` promises it is exact.
13. **How do you return an iterator from a function?** `impl Iterator<Item = T>` for a single concrete type (static dispatch), or `Box<dyn Iterator<Item = T>>` when branches return different types (dynamic dispatch, heap allocation).
14. **Why is iterator code usually as fast as a hand-written loop?** Adapters are structs, closures are zero-sized or small types, everything is monomorphized and inlined, and bounds checks often disappear. Measure hot paths rather than assuming.
15. **Why can't `Iterator` lend out `&mut` windows of its own buffer?** `Item` cannot depend on the lifetime of the `&mut self` borrow in `next`, so items cannot borrow from the iterator; that needs a lending iterator (GATs).
16. **How do you make your type usable in `for` and with `collect`?** Implement `IntoIterator` (for `T`, `&T`, `&mut T` as appropriate), `FromIterator`, and optionally `Extend`.
17. **How can `sum` be a security bug?** Overflow panics in debug and wraps in release, so a total can silently become wrong. Use wider types, `checked_add`/`try_fold`.
18. **What happens with `(1..).collect::<Vec<_>>()`?** It never ends and exhausts memory: infinite iterators need a bound (`take`, `take_while`, `zip`) before an unbounded consumer.

</details>

---

## 17. Key takeaways

- An iterator is any type with `next() -> Option<Item>`; adapters are structs that wrap iterators, consumers drive them. Implement `next` and you get the whole toolbox.
- Iterators are lazy and pull-based: nothing happens until a consumer runs, and short-circuiting consumers (`find`, `any`, `take`) stop early. Unused adapters warn.
- `for` is sugar for `IntoIterator::into_iter` plus `next`. Choose `iter()`, `iter_mut()` or `into_iter()` by whether you borrow, edit, or consume; consuming moves the collection.
- `collect` is driven by the target type; annotate it (`E0283` otherwise). `collect::<Result<_, _>>()` and `try_fold` express "stop at the first failure"; `filter_map(.ok())` hides failures.
- Know the gotchas: `take_while` eats one item, `zip` truncates, `sum` can overflow, floats are not `Ord`, infinite iterators need bounds, and you cannot mutate a collection you are iterating.
- Return iterators with `impl Iterator` (one type) or `Box<dyn Iterator>` (several); an iterator whose items borrow from itself needs a lending iterator, not `Iterator`.
- Chains are normally compiled to tight loops, as in the one ✓ check I ran, but verify hot paths.

**Connections.** Lesson 16's collections are the sources and sinks (`collect`, `extend`, `windows`, `chunks`); Lessons 10 and 11 explain every borrow error and lifetime annotation here; Lesson 14's `Result`/`Option` combinators and `?` appear inside pipelines; Lesson 15's newtypes can implement `IntoIterator` and `FromIterator`. Next: Lesson 18 (closures, `Fn`/`FnMut`/`FnOnce`, which are the `f` in every adapter) and Lesson 19 (generics and trait bounds, which explain signatures like `fn map<B, F: FnMut(Self::Item) -> B>(self, f: F) -> Map<Self, F>`). Lesson 21 covers `dyn Trait` (`Box<dyn Iterator>`), Lesson 22 covers GATs and lending iterators, Lesson 30 covers the async analogue (`Stream`), Lesson 41 covers measuring iterator performance, and Lesson 40 covers fuzzing parsers like the lexer. See the [roadmap](00_roadmap.md) and the [course index](index.md).

---

## 18. References

- *The Rust Standard Library*, `std::iter` module documentation (the trait, adapters, `FromIterator`, `IntoIterator`, `DoubleEndedIterator`, `ExactSizeIterator`, `successors`, `from_fn`): <https://doc.rust-lang.org/std/iter/index.html>
- `std::iter::Iterator` (method-by-method documentation, including the notes on `take_while`, `max`/`min` ties, `sum` overflow, and `collect` into `Result`): <https://doc.rust-lang.org/std/iter/trait.Iterator.html>
- `std::iter::Peekable` (`peek`, `next_if`, `next_if_eq`): <https://doc.rust-lang.org/std/iter/struct.Peekable.html>
- *The Rust Programming Language*, ch. 13.2 "Processing a Series of Items with Iterators" and ch. 13.4 "Comparing Performance: Loops vs. Iterators": <https://doc.rust-lang.org/book/ch13-02-iterators.html>, <https://doc.rust-lang.org/book/ch13-04-performance.html>
- *The Rust Reference*, "Iterator loops" (`for` expressions): <https://doc.rust-lang.org/reference/expressions/loop-expr.html#iterator-loops>
- `f64::total_cmp`: <https://doc.rust-lang.org/std/primitive.f64.html#method.total_cmp>
- *Rust by Example*, "Iterators": <https://doc.rust-lang.org/rust-by-example/trait/iter.html>
- Rust error code index: E0283 (type annotations needed), E0382, E0502, E0308, E0277: <https://doc.rust-lang.org/error_codes/E0283.html>, <https://doc.rust-lang.org/error_codes/E0502.html>, <https://doc.rust-lang.org/error_codes/E0308.html>
- Rust RFC 1522 "Minimal `impl Trait`" (return-position `impl Trait`) and RFC 3498 "Lifetime capture rules 2024": <https://rust-lang.github.io/rfcs/1522-conservative-impl-trait.html>, <https://rust-lang.github.io/rfcs/3498-lifetime-capture-rules-2024.html>

The URLs above are the standard documentation locations; I did not open every one during this session (the sandbox cannot reach them), so if a link has moved, search the docs site for the page title. Unchecked from memory: the RFC numbers and URLs, the Rust Book chapter numbers, the statement that the standard library has no stable lending iterator at the time of writing, the "max returns the last / min the first" tie rule (documented, not run), and the behavior that `collect::<Result<_, _>>()` stops pulling at the first error (documented, not instrumented). The ✓ assembly note counts instructions in one `-O` build; I did not compare the two bodies line by line.
