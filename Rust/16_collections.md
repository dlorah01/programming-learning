---
layout: default
title: "Lesson 16 — Collections: Vec, VecDeque, HashMap, BTreeMap, and friends"
---

# Lesson 16 — Collections: `Vec`, `VecDeque`, `HashMap`, `BTreeMap`, and friends

**Module 6 · Collections and functional programming** · Status: generated · Edition used in examples: **2024** · External dependencies: none

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97`, `rustfmt 1.9.0-stable`, on x86-64 Linux. As before, ✓ marks outputs and diagnostics I actually ran; unmarked statements are documented behavior I did not run. Everything is **stable Rust**. Programs were compiled without optimizations unless stated.

Builds on [Lesson 10](10_references_and_borrowing.md) (borrow rules, iterator invalidation), [Lesson 08](08_stack_heap_drop.md) (heap allocation, `Drop`), [Lesson 09](09_memory_layout.md) (`size_of`, fat pointers), [Lesson 06](06_strings_conversions_patterns.md) (`String` is a `Vec<u8>` with a UTF-8 invariant, slices, patterns) and [Lesson 15](15_newtype_typestate.md) (newtypes, derives).

---

## 1. Choosing a collection is choosing costs

Every collection answers the same question ("where do I keep many values?") with a different trade-off in memory layout, lookup speed, ordering and mutation cost. The standard library (`std::collections` plus `Vec`) has a small set; know what each one is *made of* and the choice mostly makes itself:

| Collection | Made of | Good at | Bad at |
|---|---|---|---|
| `Vec<T>` | one contiguous heap buffer | indexing, appending, iterating, cache locality | inserting/removing in the middle or front |
| `VecDeque<T>` | ring buffer (contiguous, wraps around) | push/pop at both ends | slices that may be split in two parts |
| `HashMap<K, V>` / `HashSet<T>` | hash table | average O(1) lookup, insert, remove | no useful order; keys need `Hash + Eq` |
| `BTreeMap<K, V>` / `BTreeSet<T>` | balanced tree with wide nodes | sorted iteration, range queries, `min`/`max` | lookup is O(log n) and less cache-friendly than a hash probe; keys need `Ord` |
| `BinaryHeap<T>` | binary heap in a `Vec` | repeatedly taking the largest item | no sorted iteration, no lookup of arbitrary items |
| `LinkedList<T>` | doubly-linked nodes | almost nothing (see below) | everything else |

The costs per operation are documented in the `std::collections` module docs ("Performance" section); I summarize them below and do not repeat the whole table. **Default to `Vec`.** Reach for something else when you can name the operation that `Vec` does badly.

---

## 2. `Vec<T>`

### 2.1 What a `Vec` is

A `Vec<T>` is three words: a pointer to a heap buffer, a **capacity** (how many elements the buffer can hold) and a **length** (how many are initialized). Lesson 08 showed this layout; here are the sizes ✓ and how the buffer grows:

```rust
use std::mem::size_of;
fn main() {
    println!("{} {} {}", size_of::<Vec<u8>>(), size_of::<Vec<u64>>(), size_of::<&[u8]>());
    let mut v: Vec<u32> = Vec::new();
    println!("len={} cap={}", v.len(), v.capacity());
    let mut last = v.capacity();
    for i in 0..40 {
        v.push(i);
        if v.capacity() != last { println!("push #{} -> cap {}", i + 1, v.capacity()); last = v.capacity(); }
    }
    let w: Vec<u32> = Vec::with_capacity(100);
    println!("{} {}", w.len(), w.capacity());
    let z: Vec<()> = vec![(); 5];
    println!("{} {}", z.len(), z.capacity() == usize::MAX);
}
```

✓ Output:

```text
24 24 16
len=0 cap=0
push #1 -> cap 4
push #5 -> cap 8
push #9 -> cap 16
push #17 -> cap 32
push #33 -> cap 64
0 100
5 true
```

What this shows:

- `Vec<u8>` and `Vec<u64>` are both 24 bytes (pointer, capacity, length); the *slice reference* `&[u8]` is 16 (pointer, length), a fat pointer (Lesson 09).
- `Vec::new()` **does not allocate** (capacity 0). The first `push` allocates; here capacity went 4, 8, 16, 32, 64. That is *amortized doubling*: occasionally a push is O(n) (allocate, copy, free), but averaged over many pushes each is O(1). The exact growth factor and the minimum capacity are implementation details that the documentation does not promise (the docs say only that capacity grows "amortized"); I observed this for `u32` on this toolchain, and the minimum is different for other element sizes.
- `Vec::with_capacity(100)` allocates once, up front. If you know (or can estimate) the final size, this avoids all the reallocations. The length is still 0.
- A `Vec` of zero-sized types never allocates, and reports capacity `usize::MAX` ✓.

**Reallocation moves the buffer**, so a pointer taken before a `push` can dangle after it. Safe Rust makes that impossible to misuse with the borrow checker; you can observe it with a raw pointer comparison:

```rust
fn main() {
    let mut v = vec![1u32, 2, 3];
    let p0 = v.as_ptr();
    v.push(4); // may reallocate
    println!("{}", p0 == v.as_ptr());
    let mut w = Vec::with_capacity(8);
    w.push(1u32);
    let q0 = w.as_ptr();
    for i in 0..7 { w.push(i); }
    println!("{}", q0 == w.as_ptr());
}
```

✓ Output `true` and `true`. The first is `true` here only by luck of the allocator (it extended the allocation in place); the language does not promise either result, which is the point. The second is `true` for a stronger reason: the capacity was never exceeded, so no reallocation was needed.

And this is the compile-time guard against the first case:

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    let first = &v[0];
    v.push(4);
    println!("{first}");
}
```

✓ `error[E0502]: cannot borrow `v` as mutable because it is also borrowed as immutable`. That is exactly the bug (a reference into a buffer that `push` may move) that C++ calls iterator invalidation; Rust refuses to compile it (Lesson 10). The same error appears for the loop version:

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    for x in &v { if *x == 2 { v.push(4); } }
}
```

✓ `E0502`. Fix by collecting changes first and applying them after the loop, by iterating over indices when you only need positions, or by using `retain`/`drain` for removal.

### 2.2 Core operations and their costs

```rust
fn main() {
    let mut v = vec!['a', 'b', 'c', 'd', 'e'];
    let r = v.remove(1);            // O(n): shifts the tail left
    println!("{r} {v:?}");
    let s = v.swap_remove(0);       // O(1): moves the LAST element into the hole
    println!("{s} {v:?}");
    v.insert(1, 'z');               // O(n): shifts the tail right
    println!("{v:?}");
    v.truncate(2);                  // drops the extra elements
    println!("{v:?}");
    let d: Vec<char> = v.drain(..).collect();
    println!("{d:?} {v:?}");
    let mut n = vec![1, 2, 3, 4, 5, 6];
    n.retain(|x| x % 2 == 0);       // one pass, keeps the matching elements
    println!("{n:?}");
    let mut m = vec![3, 1, 1, 2, 2, 3];
    m.sort(); m.dedup();            // dedup removes CONSECUTIVE duplicates only
    println!("{m:?}");
    let mut t = vec![1, 2, 3, 4, 5];
    let tail = t.split_off(3);
    println!("{t:?} {tail:?}");
    t.extend_from_slice(&tail);
    t.rotate_left(2);
    println!("{t:?}");
}
```

✓ Output:

```text
b ['a', 'c', 'd', 'e']
a ['e', 'c', 'd']
['e', 'z', 'c', 'd']
['e', 'z']
['e', 'z'] []
[2, 4, 6]
[1, 2, 3]
[1, 2, 3] [4, 5]
[3, 4, 5, 1, 2]
```

Costs to remember (documented): `push`/`pop` at the end are amortized O(1); `insert`/`remove` at index *i* are O(n − i) because elements shift; `swap_remove` is O(1) but **does not preserve order**; `contains` and `position` are O(n); `sort` is O(n log n); `binary_search` is O(log n) **on sorted data only**.

`dedup` removes only adjacent duplicates, which is why you sort first (or use a `HashSet`, Section 3.4, when you need to keep the original order).

`truncate`, `remove`, `clear` and dropping the `Vec` run each removed element's destructor. Lesson 08's drop rules, observed ✓:

```rust
struct D(&'static str);
impl Drop for D { fn drop(&mut self) { println!("drop {}", self.0); } }
fn main() {
    let mut v = vec![D("a"), D("b"), D("c")];
    v.truncate(2);
    println!("truncated");
    v.remove(0);
    println!("removed");
    drop(v);
    println!("end");
}
```

✓ Output `drop c`, `truncated`, `drop a`, `removed`, `drop b`, `end`: `truncate` drops the cut elements immediately; `remove(0)` returns the element, and since the result is not used it is dropped at the end of that statement; dropping the `Vec` drops what remains, in order.

**Capacity and memory.** Removing elements does not shrink the buffer; call `shrink_to_fit` (or `into_boxed_slice`) when a long-lived `Vec` has become much smaller than it was ✓:

```rust
fn main() {
    let mut v: Vec<i32> = (0..1000).collect();
    v.truncate(10);
    println!("{} {}", v.len(), v.capacity() >= 1000);
    v.shrink_to_fit();
    println!("{}", v.capacity());
    let b: Box<[i32]> = v.into_boxed_slice();
    println!("{}", b.len());
}
```

✓ Output `10 true`, `10`, `10`. (`shrink_to_fit` is a request the allocator may round; here the capacity became 10.) `reserve(n)` guarantees room for `n` more elements and `reserve_exact` avoids over-allocating; use them before a loop that pushes a known number of items.

### 2.3 Indexing: panic or `Option`

`v[i]` panics if `i >= len`; `v.get(i)` returns `Option<&T>`. Range indexing panics too ✓:

```rust
fn main() {
    let v = vec![1, 2, 3];
    println!("{:?} {:?}", v.get(1), v.get(9));
    let i = std::env::args().count() + 9;
    println!("{}", v[i]);
}
```

✓ Prints `Some(2) None`, then `thread 'main' panicked ... index out of bounds: the len is 3 but the index is 10` (exit status 101). Slicing `&v[3..9]` on a 7-element vector panics with `range end index 9 out of range for slice of length 7` ✓. Use `get(range)` when the index comes from outside your control.

Indexing is `Index::index`, which returns a reference. You cannot move a non-`Copy` element out of a `Vec` by indexing:

```rust
fn main() {
    let v = vec![String::from("a"), String::from("b")];
    let s = v[0];
    println!("{s}");
}
```

✓ `error[E0507]: cannot move out of index of `Vec<String>``: moving would leave a hole in the vector. Take the value out in a way that leaves something valid behind ✓:

```rust
fn main() {
    let mut v = vec![String::from("a"), String::from("b")];
    let a = std::mem::take(&mut v[0]);     // leaves "" behind
    let b = v.swap_remove(1);              // moves the last element out
    let c = v[0].clone();                  // or just clone
    println!("{a:?} {b:?} {c:?} {v:?}");
    for s in vec![String::from("x"), String::from("y")] { print!("{s} "); }  // consuming iteration moves each
    println!();
}
```

✓ Output `"a" "b" "" [""]` and `x y`. Iterating a `Vec` by value (`for s in vec`) consumes it and moves each element out; `for s in &vec` borrows.

### 2.4 Slices, windows and chunks

Most read-only algorithms should accept `&[T]`, not `&Vec<T>` (Lesson 10's deref coercion makes both callable with a `Vec`). Slices have a rich API:

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5, 6, 7];
    println!("{:?}", v.chunks(3).collect::<Vec<_>>());
    println!("{:?}", v.windows(3).map(|w| w.iter().sum::<i32>()).collect::<Vec<_>>());
    let (a, b) = v.split_at(2);
    println!("{a:?} {b:?}");
    println!("{:?} {:?}", v.first(), v.last());
    if let [first, .., last] = v.as_slice() { println!("{first} {last}"); }
    println!("{}", v.contains(&5));
    println!("{:?}", v.iter().position(|&x| x == 5));
}
```

✓ Output:

```text
[[1, 2, 3], [4, 5, 6], [7]]
[6, 9, 12, 15, 18]
[1, 2] [3, 4, 5, 6, 7]
Some(1) Some(7)
1 7
true
Some(4)
```

`chunks(3)` splits into non-overlapping pieces (the last may be shorter); `windows(3)` yields every overlapping run of 3 (here 5 of them for 7 elements). Slice patterns (`[first, .., last]`, Lesson 06) destructure by shape.

### 2.5 Sorting and searching

```rust
fn main() {
    let mut f = vec![2.5, -1.0, 9.0, 0.0];
    f.sort_by(|a, b| a.partial_cmp(b).unwrap());
    println!("{f:?}");
    f.sort_by(f64::total_cmp);
    println!("{f:?}");
    let mut w = vec!["pear", "fig", "apple", "kiwi"];
    w.sort_by_key(|s| s.len());
    println!("{w:?}");             // stable: "pear" stays before "kiwi"
    w.sort_unstable();
    println!("{w:?}");
    let s = [1, 3, 5, 7, 9];
    println!("{:?} {:?}", s.binary_search(&7), s.binary_search(&4));
}
```

✓ Output:

```text
[-1.0, 0.0, 2.5, 9.0]
[-1.0, 0.0, 2.5, 9.0]
["fig", "pear", "kiwi", "apple"]
["apple", "fig", "kiwi", "pear"]
Ok(3) Err(2)
```

- `sort` and `sort_by_key` are **stable** (equal elements keep their order; they allocate scratch space); `sort_unstable*` is usually faster and uses no extra memory but may reorder equal elements. Prefer unstable when equal elements are indistinguishable.
- `f64` is not `Ord` because of `NaN`, so `f.sort()` doesn't compile ✓ (`error[E0277]: the trait bound `{float}: Ord` is not satisfied`). `partial_cmp(...).unwrap()` works until a `NaN` shows up, and then it **panics** ✓ (`called Option::unwrap() on a None value` for `[1.0, NaN]`). `f64::total_cmp` defines a total order (negative zero before positive zero, `NaN` last) and never panics.
- `binary_search` returns `Ok(index)` if found, or `Err(insertion_point)` that keeps the slice sorted ✓. If the slice isn't sorted the result is unspecified (but not unsafe).

---

## 3. Hash-based collections: `HashMap` and `HashSet`

### 3.1 Basics

```rust
use std::collections::HashMap;
fn main() {
    let mut m: HashMap<String, u32> = HashMap::new();
    m.insert("a".to_string(), 1);
    println!("{:?}", m.insert("a".to_string(), 2));      // returns the OLD value
    println!("{:?} {:?}", m.get("a"), m.get("zz"));
    println!("{}", m["a"]);                                // panics if missing
    let text = "the cat the dog the end";
    let mut counts: HashMap<&str, usize> = HashMap::new();
    for w in text.split_whitespace() { *counts.entry(w).or_insert(0) += 1; }
    let mut pairs: Vec<_> = counts.iter().collect();
    pairs.sort();
    println!("{pairs:?}");
    m.entry("b".into()).or_insert_with(|| 10);
    m.entry("b".into()).and_modify(|v| *v += 1).or_insert(0);
    println!("{:?}", m.get("b"));
    println!("{:?} {}", m.remove("a"), m.len());
    let _ = m["missing"];
}
```

✓ Output:

```text
Some(1)
Some(2) None
2
[("cat", 1), ("dog", 1), ("end", 1), ("the", 3)]
Some(11)
Some(2) 1
```

and then the program panics with `no entry found for key` on `m["missing"]`. Key points:

- `insert` returns `Option<V>`: the previous value if the key was present.
- `get` returns `Option<&V>`; `m[key]` panics when the key is absent. Prefer `get` unless a missing key is a bug.
- You can look up a `String` key with a `&str` (`m.get("a")`): lookups are generic over `Q` where `K: Borrow<Q>`, so no allocation is needed to search. The same works for `Vec<u8>` keys looked up with `&[u8]` ✓ (`Some(5)`).
- **The `entry` API** does "find, or insert, then give me a mutable reference" with a single hash computation and no borrow-checker fight. `*counts.entry(w).or_insert(0) += 1` is the idiomatic counter. `or_default()` uses `Default::default()` (great for `Vec` values: `map.entry(k).or_default().push(x)`).

### 3.2 Key requirements: `Hash + Eq`

A key type must implement `Hash` and `Eq`, and the two must agree: **if `a == b` then `hash(a) == hash(b)`**. Derive both (Lesson 13) and you get this for free. `f64` has neither `Eq` nor `Hash`, again because of `NaN`:

```rust
use std::collections::HashMap;
#[derive(Debug, PartialEq)]
struct P(f64);
fn main() {
    let mut m = HashMap::new();
    m.insert(P(1.0), 1);
}
```

✓ Two errors, both `E0277`: ``the trait bound `P: Eq` is not satisfied`` and ``the trait bound `P: Hash` is not satisfied``. Same for `BTreeMap<f64, _>` ✓ (``the trait bound `f64: Ord` is not satisfied``). Use an integer (e.g. cents), a string, or a newtype that supplies a total order (exercise 5).

A derived key works as you'd expect, and equal keys collapse ✓:

```rust
use std::collections::{HashMap, HashSet, BTreeMap};
#[derive(Debug, Clone, PartialEq, Eq, Hash, PartialOrd, Ord)]
struct Key { ns: u8, id: u32 }
fn main() {
    let mut m = HashMap::new();
    m.insert(Key { ns: 1, id: 7 }, "a");
    m.insert(Key { ns: 1, id: 7 }, "b");
    println!("{} {:?}", m.len(), m.get(&Key { ns: 1, id: 7 }));
    let mut b = BTreeMap::new();
    b.insert(Key { ns: 2, id: 1 }, 'x'); b.insert(Key { ns: 1, id: 9 }, 'y'); b.insert(Key { ns: 1, id: 2 }, 'z');
    println!("{:?}", b.keys().collect::<Vec<_>>());
    let s: HashSet<i32> = [1, 2, 3].into_iter().collect();
    let t: HashSet<i32> = [2, 3, 4].into_iter().collect();
    let mut i: Vec<_> = s.intersection(&t).copied().collect(); i.sort();
    let mut u: Vec<_> = s.union(&t).copied().collect(); u.sort();
    let mut d: Vec<_> = s.difference(&t).copied().collect(); d.sort();
    println!("{i:?} {u:?} {d:?}");
}
```

✓ Output `1 Some("b")`, then `[Key { ns: 1, id: 2 }, Key { ns: 1, id: 9 }, Key { ns: 2, id: 1 }]` (derived `Ord` compares fields in order, Lesson 13), then `[2, 3] [1, 2, 3, 4] [1]`. Set algebra (`union`, `intersection`, `difference`, `symmetric_difference`) returns lazy iterators; collect them and sort if you need a deterministic order.

**Do not mutate a key in a way that changes its hash or equality** while it is in the map (possible only through interior mutability, Lesson 24). The map will not find it again; this is a logic error, not undefined behavior.

### 3.3 Iteration order is unspecified

```rust
use std::collections::HashMap;
fn main() {
    let a: HashMap<i32, i32> = (0..8).map(|i| (i, i)).collect();
    let b: HashMap<i32, i32> = (0..8).map(|i| (i, i)).collect();
    let ka: Vec<_> = a.keys().copied().collect();
    let kb: Vec<_> = b.keys().copied().collect();
    println!("{}", ka == kb);
    let mut sorted = ka.clone(); sorted.sort();
    println!("{sorted:?}");
}
```

✓ Output `false` and `[0, 1, 2, 3, 4, 5, 6, 7]`: two maps built from the same data, in the same run, iterated in **different orders**, because each `HashMap` gets its own random hash seed. (It can print `true` on other runs by chance; the point is that you cannot depend on either.) If output must be reproducible (tests, reports, anything signed or hashed), sort the keys or use a `BTreeMap`. Equality ignores order: two maps with the same pairs are `==` regardless of insertion order ✓ (`a == b` is `true` in my test with different insertion order).

### 3.4 `HashSet` patterns

```rust
use std::collections::{HashMap, HashSet};
fn main() {
    let a: HashMap<i32, i32> = [(1, 1), (2, 2)].into_iter().collect();
    let b: HashMap<i32, i32> = [(2, 2), (1, 1)].into_iter().collect();
    println!("{}", a == b);
    let s: HashSet<_> = [3, 1, 2].into_iter().collect();
    let t: HashSet<_> = [1, 2, 3].into_iter().collect();
    println!("{} {}", s == t, s.is_subset(&t));
    let mut seen = HashSet::new();
    let dups: Vec<_> = [1, 2, 1, 3, 2].into_iter().filter(|x| !seen.insert(*x)).collect();
    println!("{dups:?}");
}
```

✓ Output `true`, `true true`, `[1, 2]`. `HashSet::insert` returns `true` if the value was new, which makes "first time I've seen this" a one-liner (also used to remove duplicates while keeping the first occurrence, exercise 6).

### 3.5 The hasher, speed, and denial of service

`HashMap` is implemented as a SwissTable-style open-addressing table (the `hashbrown` crate, vendored into the standard library). By default it hashes keys with **SipHash 1-3 using a random key per map** (`RandomState`). That default is chosen to resist *hash-flooding* attacks, where an attacker supplies many keys that collide to degrade your O(1) lookups into O(n) and burn your CPU. It is somewhat slower than non-cryptographic hashes, which is the trade-off.

You can swap the hasher through the third type parameter, for example for trusted integer keys ✓:

```rust
use std::collections::HashMap;
use std::hash::{BuildHasherDefault, DefaultHasher, Hash, Hasher, BuildHasher, RandomState};
#[derive(Default)]
struct Fnv(u64);
impl Hasher for Fnv {
    fn finish(&self) -> u64 { self.0 }
    fn write(&mut self, bytes: &[u8]) {
        if self.0 == 0 { self.0 = 0xcbf29ce484222325; }
        for b in bytes { self.0 ^= *b as u64; self.0 = self.0.wrapping_mul(0x100000001b3); }
    }
}
fn main() {
    let mut m: HashMap<&str, i32, BuildHasherDefault<Fnv>> = HashMap::default();
    m.insert("a", 1); m.insert("b", 2);
    println!("{:?} {}", m.get("a"), m.len());
    let s1 = RandomState::new(); let s2 = RandomState::new();
    let h = |s: &RandomState| { let mut h = s.build_hasher(); 42u32.hash(&mut h); h.finish() };
    println!("{}", h(&s1) == h(&s1));
    println!("{}", h(&s1) == h(&s2));
    let mut d1 = DefaultHasher::new(); let mut d2 = DefaultHasher::new();
    1u8.hash(&mut d1); 1u8.hash(&mut d2);
    println!("{}", d1.finish() == d2.finish());
}
```

✓ Output `Some(1) 2`, `true`, `false`, `true`. The same `RandomState` hashes the same value identically; two `RandomState`s hash it differently (that is the per-map randomness), so a hash is not a stable identifier across maps or runs. Only use a faster hasher (the `FxHash`/`ahash` families in external crates, or a small one like the toy FNV above) when keys are **not attacker-controlled**; a toy hasher like this one is for demonstration, not production. And never use `DefaultHasher` output as a persistent or cryptographic hash: its algorithm is not specified to stay the same across Rust releases (Lesson 48 covers real hash functions).

Performance practice: `HashMap::with_capacity(n)` avoids rehashing as the map grows ✓ (a map created with capacity 100 held 100 inserts without changing capacity); `entry` avoids hashing twice; borrowed lookups avoid allocating keys.

### 3.6 Two mutable references into one map

The borrow checker sees `get_mut` as borrowing the whole map:

```rust
use std::collections::HashMap;
fn main() {
    let mut m: HashMap<&str, i32> = HashMap::new();
    m.insert("a", 1); m.insert("b", 2);
    let a = m.get_mut("a").unwrap();
    let b = m.get_mut("b").unwrap();
    *a += *b;
}
```

✓ `error[E0499]: cannot borrow `m` as mutable more than once at a time`. It cannot know that "a" and "b" are different entries. Options: copy a value out first (`let b = m["b"]; *m.get_mut("a").unwrap() += b;`), restructure, or use `get_disjoint_mut`, which checks at run time that the keys are distinct ✓:

```rust
use std::collections::HashMap;
fn main() {
    let mut m: HashMap<&str, i32> = HashMap::new();
    m.insert("a", 1); m.insert("b", 2);
    if let [Some(a), Some(b)] = m.get_disjoint_mut(["a", "b"]) { std::mem::swap(a, b); }
    let mut keys: Vec<_> = m.iter().collect(); keys.sort();
    println!("{keys:?}");
    println!("{:?}", m.get_disjoint_mut(["a", "a"]).len());
}
```

✓ Output `[("a", 2), ("b", 1)]`, then a panic `duplicate keys found` for `["a", "a"]`. (`get_disjoint_mut` is stable on 1.97; I believe it was stabilized in 1.86, from memory.)

### 3.7 `get`, then `insert` in a function that returns the reference

A famous annoyance: look up, return the reference if found, otherwise insert and return that.

```rust
use std::collections::HashMap;
fn get_or_make(m: &mut HashMap<u32, String>, k: u32) -> &String {
    if let Some(v) = m.get(&k) { return v; }
    m.insert(k, "new".into());
    &m[&k]
}
fn main() { let mut m = HashMap::new(); println!("{}", get_or_make(&mut m, 1)); }
```

✓ `error[E0502]: cannot borrow `*m` as mutable because it is also borrowed as immutable`. This is the conditional-return-of-a-borrow case from Lesson 11 (the current borrow checker cannot see that the early-returned borrow is dead on the other path). The fix is the `entry` API ✓, which needs only one borrow:

```rust
use std::collections::HashMap;
fn get_or_make(m: &mut HashMap<u32, String>, k: u32) -> &String {
    m.entry(k).or_insert_with(|| "new".into())
}
fn main() { let mut m = HashMap::new(); println!("{}", get_or_make(&mut m, 1)); println!("{}", get_or_make(&mut m, 1)); }
```

✓ Output `new` twice. Careful when generalizing: the *same* get-then-insert pattern, when the reference is **not** returned out of the function, compiled fine for me ✓ (`match m.get(&1) { Some(v) => v, None => { m.insert(1, ...); m.get(&1).unwrap() } }` followed by a use of the result printed `x`). So the problem is specifically the returned borrow. Still, `entry` is the clearer idiom either way.

---

## 4. Ordered collections: `BTreeMap` and `BTreeSet`

A `BTreeMap` keeps keys sorted. It is a B-tree: each node holds several keys in a small array, so there are far fewer pointer hops and better cache behavior than a binary search tree, and it needs only `Ord`, not `Hash`. (The node width is an internal detail; I recall it being about a dozen keys per node, from memory.)

```rust
use std::collections::{BTreeMap, BTreeSet};
use std::ops::Bound::{Excluded, Included, Unbounded};
fn main() {
    let mut m = BTreeMap::new();
    for (k, v) in [(30, "c"), (10, "a"), (20, "b"), (40, "d")] { m.insert(k, v); }
    println!("{m:?}");
    println!("{:?} {:?}", m.first_key_value(), m.last_key_value());
    println!("{:?}", m.range(15..=30).collect::<Vec<_>>());
    println!("{:?}", m.range((Excluded(10), Unbounded)).map(|(k, _)| *k).collect::<Vec<_>>());
    println!("{:?}", m.range(..25).next_back());        // greatest key below 25
    println!("{:?}", m.range((Included(25), Unbounded)).next()); // smallest key at or above 25
    println!("{:?}", m.pop_first());
    let s: BTreeSet<&str> = ["pear", "apple", "fig"].into_iter().collect();
    println!("{s:?}");
    let bad = m.range(30..10).count();
    println!("{bad}");
}
```

✓ Output:

```text
{10: "a", 20: "b", 30: "c", 40: "d"}
Some((10, "a")) Some((40, "d"))
[(20, "b"), (30, "c")]
[20, 30, 40]
Some((20, "b"))
Some((30, "c"))
Some((10, "a"))
{"apple", "fig", "pear"}
```

and then a panic `range start is greater than range end in BTreeMap` for `range(30..10)`. Things a `HashMap` cannot do and a `BTreeMap` can: ordered iteration, `first/last`, `pop_first/pop_last`, and `range` queries ("all keys between a and b", "the predecessor of x", "the successor of x"). That makes it the right structure for time-indexed data, price ladders, interval lookups and any report that must come out in a deterministic order ✓ (`{"a": 1, ...}` iteration order was sorted; `retain` and `==` work as expected).

Costs (documented): lookup, insert, remove are O(log n); iteration over all items is O(n) in key order; `range` is O(log n + items returned).

---

## 5. `VecDeque` and `BinaryHeap`

### 5.1 `VecDeque`: a queue that is fast at both ends

A `VecDeque<T>` is a ring buffer: contiguous memory with a head index that wraps around, so `push_front`/`pop_front` are O(1) (a `Vec` needs O(n) to insert at the front). Indexing is still O(1).

```rust
use std::collections::VecDeque;
fn main() {
    let mut q = VecDeque::new();
    q.push_back(1); q.push_back(2); q.push_front(0);
    println!("{q:?} {:?} {:?}", q.front(), q.back());
    println!("{:?} {:?}", q.pop_front(), q.pop_back());
    let mut r: VecDeque<i32> = (1..=5).collect();
    r.rotate_left(2);
    println!("{r:?}");
    let (a, b) = r.as_slices();
    println!("{} {}", a.len() + b.len(), r.len());
    r.make_contiguous().sort();
    println!("{r:?} {}", r[2]);
    let v: Vec<i32> = r.into();
    println!("{v:?}");
}
```

✓ Output:

```text
[0, 1, 2] Some(0) Some(2)
Some(0) Some(2)
[3, 4, 5, 1, 2]
5 5
[1, 2, 3, 4, 5] 3
[1, 2, 3, 4, 5]
```

Because of the wrap-around, the contents may be in **two** slices (`as_slices`); `make_contiguous` rearranges them into one so that slice methods like `sort` can run. Use it for FIFO queues, BFS frontiers, sliding windows and bounded history buffers. (A rough timing of 20,000 `insert(0, _)` calls on a `Vec` against 20,000 `push_front` calls on a `VecDeque` showed the `VecDeque` faster in a debug build; that was a one-off check, not a benchmark, and I give no numbers. Lesson 41 shows how to measure properly.)

### 5.2 `BinaryHeap`: a priority queue

`BinaryHeap<T>` is a **max-heap**: `pop` returns the *largest* item, in O(log n); `peek` is O(1); `push` is O(log n) worst case. Wrap items in `std::cmp::Reverse` for a min-heap. Tuples compare lexicographically, so `(priority, payload)` works if the payload is `Ord`:

```rust
use std::collections::BinaryHeap;
use std::cmp::Reverse;
fn main() {
    let mut h = BinaryHeap::new();
    for x in [5, 1, 8, 3] { h.push(x); }
    println!("{:?} {}", h.peek(), h.len());
    let mut out = vec![]; while let Some(x) = h.pop() { out.push(x); }
    println!("{out:?}");
    let mut mh = BinaryHeap::new();
    for x in [5, 1, 8, 3] { mh.push(Reverse(x)); }
    println!("{:?}", mh.pop().map(|Reverse(x)| x));
    let h2: BinaryHeap<_> = vec![2, 9, 4].into();
    println!("{:?}", h2.into_sorted_vec());
    let mut t = BinaryHeap::new();
    t.push((2, "b")); t.push((9, "z")); t.push((9, "a"));
    println!("{:?}", t.pop());
}
```

✓ Output `Some(8) 4`, `[8, 5, 3, 1]`, `Some(1)`, `[2, 4, 9]`, `Some((9, "z"))`. The heap's *iteration* order is not sorted (only `pop` order is), so use `into_sorted_vec` to get ascending order. Classic uses: Dijkstra and A* (with `Reverse` on the distance), task schedulers, merging sorted streams, and top-k (the practical challenge).

### 5.3 `LinkedList`

`LinkedList` exists but is rarely the right answer: every element is a separate allocation, traversal chases pointers (poor cache behavior), and the O(1) splice and insert operations require a cursor API that is still unstable at the time of writing (from memory). In almost every case a `Vec` or `VecDeque` is faster. Lesson 12 built a linked list to teach ownership, not because you should use one.

---

## 6. Memory sizes and layouts

Sizes on this machine ✓:

```rust
use std::collections::*;
use std::mem::size_of;
fn main() {
    println!("HashMap<u32,u32> {}", size_of::<HashMap<u32, u32>>());
    println!("HashSet<u32> {}", size_of::<HashSet<u32>>());
    println!("BTreeMap<u32,u32> {}", size_of::<BTreeMap<u32, u32>>());
    println!("VecDeque<u32> {}", size_of::<VecDeque<u32>>());
    println!("BinaryHeap<u32> {}", size_of::<BinaryHeap<u32>>());
    println!("Vec<u32> {}", size_of::<Vec<u32>>());
    println!("Box<[u32]> {}", size_of::<Box<[u32]>>());
    println!("Option<Vec<u32>> {}", size_of::<Option<Vec<u32>>>());
}
```

✓ Output:

```text
HashMap<u32,u32> 48
HashSet<u32> 48
BTreeMap<u32,u32> 24
VecDeque<u32> 32
BinaryHeap<u32> 24
Vec<u32> 24
Box<[u32]> 16
Option<Vec<u32>> 24
```

These are the sizes of the *handles* on the stack; the elements live on the heap. Layout of the standard library types is not guaranteed and may change between releases, so never write code that depends on these numbers. Observations: a `BinaryHeap` is a wrapped `Vec` (24); `Box<[T]>` has no capacity field (16), which is why `into_boxed_slice` can save memory for data you'll never grow; `Option<Vec<T>>` is the same size as `Vec<T>` because the non-null pointer provides a niche (Lesson 09). A `HashMap` also carries its hasher state (the random seed), hence 48 here.

A `Vec<Vec<T>>` is a vector of separately allocated rows, so each row is its own heap block; `vec![vec![0u8; 3]; 2]` clones the inner vector, so rows are independent ✓ (changing `g2[0][1]` did not affect the original `grid[0]`). For dense numeric grids, a flat `Vec<T>` with index arithmetic (`row * width + col`) is more cache-friendly and one allocation.

---

## 7. Choosing, and the cost cheat-sheet

Documented asymptotics (n = number of elements, "amortized" for growth):

| Operation | `Vec` | `VecDeque` | `HashMap` | `BTreeMap` | `BinaryHeap` |
|---|---|---|---|---|---|
| Index by position | O(1) | O(1) | n/a | n/a | n/a |
| Lookup by key/contains | O(n) | O(n) | O(1) average | O(log n) | O(n) |
| Insert/push at end | O(1) amortized | O(1) amortized | O(1) average | O(log n) | O(log n) |
| Insert/remove at front | O(n) | O(1) | n/a | n/a | n/a |
| Insert/remove in middle | O(n) | O(n) | n/a | n/a | n/a |
| Min/max | O(n) | O(n) | O(n) | O(log n) | max O(1) |
| Sorted iteration | sort first | sort first | sort first | O(n) | `into_sorted_vec` |
| Range query | binary search if sorted | no | no | O(log n + k) | no |

Practical rules:

1. Start with `Vec`. Small `Vec`s with a linear search beat maps surprisingly often because of cache locality (measure before you assume; Lesson 41).
2. Need lookup by key, order irrelevant: `HashMap`. Need the keys in order or ranges: `BTreeMap`.
3. Need a queue: `VecDeque`. Need "largest/smallest next": `BinaryHeap`.
4. Need uniqueness: `HashSet`/`BTreeSet`. Need reproducible output: sort or use the `BTree*` types.
5. Avoid `LinkedList`.
6. Pre-size with `with_capacity` when you know the size; `shrink_to_fit` for long-lived leftovers.

---

## 8. Common mistakes and edge cases

- **Holding a reference across a mutation** (`E0502`, Section 2.1). Collect changes first, or index.
- **Indexing with `[]` on untrusted input**: panic. Use `get`.
- **`remove(0)` in a loop**: O(n²). Use `VecDeque`, `drain`, or `retain`.
- **`sort()` on `f64`** doesn't compile; `partial_cmp().unwrap()` panics on `NaN`. Use `total_cmp`.
- **Assuming `HashMap` order**: unspecified and different per map (✓). Sort before printing, hashing, or comparing output.
- **Using `m[key]` where `get` is right**: panics on a missing key.
- **Double lookup** (`if !m.contains_key(&k) { m.insert(k, v); }`): two hashes. Use `entry`.
- **Cloning keys just to look them up**: use borrowed lookups (`&str` for `String` keys).
- **Mutating a key's hash/equality while it is in the map** (interior mutability): lost entries.
- **`dedup` without sorting**: only removes adjacent duplicates.
- **`vec![x; n]` requires `Clone`**; `vec![Vec::new(); n]` makes independent empty vectors.
- **Capacity vs length**: `with_capacity(n)` gives length 0; indexing `v[0]` still panics.
- **Unbounded growth** of a cache, queue or map fed by outside input (see Security).

---

## 9. Performance implications

- Amortized growth means `push` is cheap on average but has occasional O(n) spikes (copy). In latency-sensitive code, reserve capacity ahead of time.
- `Vec` is contiguous: iteration is the fastest traversal Rust offers, because of cache lines and prefetching. A `Vec<Struct>` of arrays-of-structs vs a struct of `Vec`s ("struct of arrays") changes cache behavior; Lesson 42 covers it.
- `HashMap` lookups hash the key and probe a table; with the default SipHash, hashing long keys costs more than a custom hasher would. The cost model is "O(1)" but with a nontrivial constant.
- `BTreeMap` has O(log n) operations but with wide nodes; for small maps or sorted-order workloads it can beat `HashMap`.
- Each `String`/`Vec` key or value is its own heap allocation. Millions of tiny allocations dominate runtime; consider interning, arenas (Lesson 12), or `Box<str>`.
- Clone-heavy code (cloning a `Vec` or `HashMap` to satisfy the borrow checker) copies all the data. Prefer borrowing or `Rc`/`Arc` (Lesson 23) when sharing.
- I did not include benchmark numbers in this lesson. Any claim of "X is faster than Y" here is about asymptotics or a qualitative one-off, not a measured result.

---

## 10. Security and reliability

- **Unbounded collections are a denial-of-service risk.** If an attacker (or just a bug) can make you push to a `Vec`, insert into a `HashMap`, or enqueue into a channel without limit, memory grows until the process is killed. Cap sizes, apply backpressure (Lesson 32), and bound caches (the LRU in the practical challenge has a fixed capacity).
- **Hash flooding.** Keep the default `RandomState` for any map keyed by attacker-influenced data (network input, user names). Only use fast non-cryptographic hashers for trusted keys.
- **`with_capacity(n)` with an attacker-supplied `n`** allocates that much immediately and can abort the process on overflow or exhaustion. Validate sizes before reserving. (`try_reserve` returns an error instead of aborting.)
- **Panics on index/key misuse** (`v[i]`, `m[k]`, `BTreeMap::range` with a reversed range) are a remote crash if the input is untrusted. Use `get` and checked arithmetic, and validate ranges.
- **Nondeterministic iteration order** is a correctness bug in anything that must be reproducible: signing, hashing a set of items, consensus-critical state (Lessons 48 to 57). Use ordered collections or sort explicitly.
- **Sorting with an inconsistent comparator** (not a total order, e.g. `partial_cmp().unwrap()` on `NaN`, or a comparison that isn't transitive) can panic or give an unspecified order in recent standard libraries; the result is still memory safe.
- **Timing and key-dependent lookups**: ordinary maps are not constant-time; do not use them to compare secrets (Lesson 48).
- Safe Rust prevents use-after-free and data races on these collections; it does not prevent logic errors such as unbounded growth, stale cache entries, or wrong keys.

---

## 11. Exercises

Try each before opening the answer. They are not marked complete unless you tell me you did them.

1. **(Easy)** For `let mut v: Vec<u32> = Vec::new();` pushed 40 times, at which pushes did the capacity change on this toolchain, and is that behavior guaranteed?
2. **(Easy)** Fix: `let first = &v[0]; v.push(4); println!("{first}");`
3. **(Medium)** Write `top_words(text: &str, n: usize) -> Vec<(String, usize)>` returning the `n` most frequent words (case-insensitive), ties broken alphabetically.
4. **(Medium)** Write `two_sum(nums: &[i64], target: i64) -> Option<(usize, usize)>` in O(n) using a map. Handle overflow without panicking.
5. **(Medium)** Why does `BTreeMap<f64, _>` not compile, and how can you make a map keyed by `f64` work?
6. **(Medium)** Remove duplicates from a slice while keeping the first occurrence of each item and the original order.
7. **(Hard)** Sliding window maximum in O(n): for each window of size `k`, the maximum, using a `VecDeque`.
8. **(Hard)** For each of these, pick a collection and justify it: (a) a job scheduler that always runs the highest-priority job next, (b) a "last 100 events" log, (c) an index from timestamp to event supporting "events in [t1, t2)", (d) counting distinct user ids in a stream.

<details markdown="1">
<summary>Answers</summary>

1. ✓ Capacity went `0 -> 4` at push 1, `4 -> 8` at push 5, `8 -> 16` at push 9, `16 -> 32` at push 17, and `32 -> 64` at push 33. It is **not guaranteed**: the documentation promises amortized growth, not the factor or the minimum, and the starting capacity depends on the element size.
2. The reference `first` is live across the `push`, which may reallocate (✓ `E0502`). Either finish with `first` before pushing (`println!("{}", v[0]); v.push(4);`), or copy the value out (`let first = v[0];` for a `Copy` type, or `.clone()`), or take the value to push before borrowing.
3. ✓ (run: `top_words("b a B c a b d", 3)` gave `[("b", 3), ("a", 2), ("c", 1)]`)
   ```rust
   use std::collections::HashMap;
   fn top_words(text: &str, n: usize) -> Vec<(String, usize)> {
       let mut counts: HashMap<String, usize> = HashMap::new();
       for w in text.split_whitespace() { *counts.entry(w.to_lowercase()).or_default() += 1; }
       let mut v: Vec<_> = counts.into_iter().collect();
       v.sort_by(|a, b| b.1.cmp(&a.1).then_with(|| a.0.cmp(&b.0)));
       v.truncate(n);
       v
   }
   ```
   Sorting by count descending, then by word, makes the output deterministic even though the `HashMap` iteration order is not. (For huge inputs with small `n`, a `TopK` over `(count, Reverse(word))` avoids sorting everything.)
4. ✓ (run)
   ```rust
   use std::collections::HashMap;
   fn two_sum(nums: &[i64], target: i64) -> Option<(usize, usize)> {
       let mut seen: HashMap<i64, usize> = HashMap::with_capacity(nums.len());
       for (i, &n) in nums.iter().enumerate() {
           if let Some(complement) = target.checked_sub(n)
               && let Some(&j) = seen.get(&complement)
           {
               return Some((j, i));
           }
           seen.insert(n, i);
       }
       None
   }
   ```
   `two_sum(&[2, 7, 11, 15], 9)` is `Some((0, 1))`, `two_sum(&[3, 3], 6)` is `Some((0, 1))` (the check happens before inserting, so an element is never paired with itself), `two_sum(&[1, 2], 10)` is `None`, and `two_sum(&[i64::MIN, 5], 5)` is `None` without an overflow panic because of `checked_sub`. The `if let ... && let ...` chain needs edition 2024 (let chains, stabilized in 1.88, from memory; it compiled on 1.97 ✓).
5. `f64` is not `Ord` (`NaN` is unordered), so `BTreeMap<f64, _>` fails with ✓ `E0277: the trait bound `f64: Ord` is not satisfied`. Wrap it in a newtype (Lesson 15) that defines a total order with `f64::total_cmp`:
   ```rust
   use std::cmp::Ordering;
   #[derive(Debug, Clone, Copy)]
   struct OrdF64(f64);
   impl PartialEq for OrdF64 { fn eq(&self, o: &Self) -> bool { self.cmp(o) == Ordering::Equal } }
   impl Eq for OrdF64 {}
   impl PartialOrd for OrdF64 { fn partial_cmp(&self, o: &Self) -> Option<Ordering> { Some(self.cmp(o)) } }
   impl Ord for OrdF64 { fn cmp(&self, o: &Self) -> Ordering { self.0.total_cmp(&o.0) } }
   ```
   ✓ Inserting `2.5, -1.0, NaN, 0.0, -0.0` into a `BTreeMap<OrdF64, _>` iterated as `-1.0, -0.0, 0.0, 2.5, NaN`: `-0.0` and `0.0` are **distinct** keys under `total_cmp`, and `NaN` sorts last. For money, prefer integers (cents) instead of floats.
6. ✓ (run: `[3, 1, 3, 2, 1]` becomes `[3, 1, 2]`)
   ```rust
   use std::collections::HashSet;
   fn dedup_stable<T: std::hash::Hash + Eq + Clone>(v: &[T]) -> Vec<T> {
       let mut seen = HashSet::new();
       v.iter().filter(|x| seen.insert((*x).clone())).cloned().collect()
   }
   ```
   `insert` returns `true` only the first time. This is O(n) and keeps order; `sort` + `dedup` is O(n log n) and does not.
7. ✓ (run: `sliding_max(&[1, 3, -1, -3, 5, 3, 6, 7], 3)` gave `[3, 3, 5, 5, 6, 7]`; `k = 0` gives `[]`)
   ```rust
   use std::collections::VecDeque;
   fn sliding_max(v: &[i32], k: usize) -> Vec<i32> {
       let mut out = Vec::new();
       let mut dq: VecDeque<usize> = VecDeque::new(); // indices; values decreasing
       for (i, &x) in v.iter().enumerate() {
           while dq.back().is_some_and(|&j| v[j] <= x) { dq.pop_back(); }
           dq.push_back(i);
           if dq.front().is_some_and(|&j| j + k <= i) { dq.pop_front(); }
           if i + 1 >= k && k > 0 { out.push(v[dq[0]]); }
       }
       out
   }
   ```
   The deque holds indices whose values are in decreasing order, so the front is always the maximum of the window. Each index is pushed and popped at most once, so the whole thing is O(n).
8. (a) `BinaryHeap` (or `BinaryHeap<Reverse<_>>` if lower number means higher priority): `pop` gives the next job in O(log n). If you also need to cancel or re-prioritize arbitrary jobs, a `BTreeMap` or an indexed heap is needed. (b) `VecDeque` with a length cap: `push_back`, and `pop_front` when `len > 100`. (c) `BTreeMap<Timestamp, Event>` (or `BTreeMap<Timestamp, Vec<Event>>` for equal timestamps): `range(t1..t2)` answers the query in O(log n + k). (d) `HashSet<UserId>`, `len()` at the end: O(1) per insert. If the stream is too big to hold, you would need a probabilistic structure (e.g. HyperLogLog, an external crate; not covered here). These are design answers, not compiled.

</details>

---

## 12. Debugging challenge

A caching helper should return the cached value or create it, handing back a reference:

```rust
use std::collections::HashMap;
fn get_or_make(m: &mut HashMap<u32, String>, k: u32) -> &String {
    if let Some(v) = m.get(&k) { return v; }
    m.insert(k, "new".into());
    &m[&k]
}
fn main() { let mut m = HashMap::new(); println!("{}", get_or_make(&mut m, 1)); }
```

✓ It fails with `error[E0502]: cannot borrow `*m` as mutable because it is also borrowed as immutable`. The author says "the `return v` path ends the borrow, so the `insert` is fine!" Why does the compiler disagree, and what is the fix?

<details markdown="1">
<summary>Solution</summary>

The reference `v` is returned from the function, so its lifetime must be as long as the one in the return type (`'_` tied to `m`). The borrow checker therefore treats the `m.get(&k)` borrow as lasting for that whole lifetime *on every path*, including the path where `get` returned `None` and execution continues to `m.insert`. It cannot express "this borrow lasts only if we return it" (this is the conditional-return limitation discussed in Lesson 11). So `m.insert` conflicts with a borrow that is considered still live.

Fix: use the `entry` API, which takes the single mutable borrow up front and gives back a reference into the map in both cases:

```rust
fn get_or_make(m: &mut HashMap<u32, String>, k: u32) -> &String {
    m.entry(k).or_insert_with(|| "new".into())
}
```

✓ Calling it twice prints `new` twice, and the second call finds the entry already there. `entry` also hashes `k` once instead of up to three times (`get`, `insert`, `[]`). Other workarounds: check `contains_key` first and then `insert` and index (two to three hashes, but borrow-correct), or return `Option`/clone the value instead of a reference.

Also note (✓ tested) that when the reference is *not* returned from the function, the same get-then-insert shape compiled on 1.97, so don't over-generalize this error: it is the returned borrow that makes it fail.

</details>

---

## 13. Practical challenge: `cachelab`

Create a library crate `cachelab` (edition 2024) with:

1. `LruCache<K, V>` with a fixed `NonZeroUsize` capacity: `put` (returns the evicted entry, if any; updating an existing key never evicts), `get` (refreshes recency), `peek` (does not), `contains_key`, `remove`, `len`/`is_empty`/`capacity`, and `iter_mru` (most recent first). Lookups must accept borrowed keys (`&str` for `String`). Use a `HashMap` plus a `BTreeMap` keyed by a use counter so every operation is O(1) average or O(log n).
2. `TopK<T: Ord>`: stream items in, keep the `k` largest using a `BinaryHeap<Reverse<T>>`; `k == 0` keeps nothing; `into_sorted_desc`.
3. `word_counts(text) -> BTreeMap<String, usize>` (lowercased alphanumeric words, deterministic order) and `group_by(items, key_fn) -> BTreeMap<K, Vec<T>>`.
4. Tests: eviction order; `get` vs `peek`; updating a key; removal and recency order; capacity 1; borrowed lookup; a **model-based test** comparing the cache to a naive `Vec` implementation over thousands of pseudo-random operations; `TopK` against sort-and-truncate; deterministic counting and grouping. A usage doctest.
5. It must pass `cargo fmt --check`, `cargo clippy --all-targets --locked -- -D warnings`, and `cargo test --locked` (run `cargo generate-lockfile` once first).

<details markdown="1">
<summary>Solution</summary>

**`src/lib.rs`**

```rust
//! Small collection-based utilities: an LRU cache (`HashMap` + `BTreeMap`),
//! a streaming top-k (`BinaryHeap`), and two deterministic grouping helpers.
//!
//! ```
//! use cachelab::{LruCache, TopK};
//! use std::num::NonZeroUsize;
//!
//! let mut cache = LruCache::new(NonZeroUsize::new(2).unwrap());
//! cache.put("a", 1);
//! cache.put("b", 2);
//! cache.get("a"); // "a" is now the most recently used
//! let evicted = cache.put("c", 3);
//! assert_eq!(evicted, Some(("b", 2)));
//!
//! let mut top = TopK::new(2);
//! for x in [5, 1, 9, 3] {
//!     top.push(x);
//! }
//! assert_eq!(top.into_sorted_desc(), vec![9, 5]);
//! ```

use std::borrow::Borrow;
use std::cmp::Reverse;
use std::collections::{BTreeMap, BinaryHeap, HashMap};
use std::hash::Hash;
use std::num::NonZeroUsize;

/// A least-recently-used cache with O(1) average lookup and O(log n) recency updates.
///
/// `map` holds each value with the "tick" of its last use; `order` maps each tick back
/// to its key, so the smallest tick is always the least recently used entry.
/// Invariant: `map` and `order` always have the same length, and every `(key, tick)`
/// pair in one appears in the other. The tick is a `u64` that increases on every use;
/// overflowing it would take centuries at a billion operations per second.
#[derive(Debug)]
pub struct LruCache<K, V> {
    capacity: NonZeroUsize,
    tick: u64,
    map: HashMap<K, (V, u64)>,
    order: BTreeMap<u64, K>,
}

impl<K: Hash + Eq + Clone, V> LruCache<K, V> {
    /// Creates an empty cache holding at most `capacity` entries.
    pub fn new(capacity: NonZeroUsize) -> Self {
        LruCache {
            capacity,
            tick: 0,
            map: HashMap::with_capacity(capacity.get()),
            order: BTreeMap::new(),
        }
    }

    /// Number of entries currently stored.
    pub fn len(&self) -> usize {
        self.map.len()
    }

    /// True if the cache has no entries.
    pub fn is_empty(&self) -> bool {
        self.map.is_empty()
    }

    /// Maximum number of entries.
    pub fn capacity(&self) -> usize {
        self.capacity.get()
    }

    /// Inserts or updates `key`, making it the most recently used.
    ///
    /// Returns the entry that was evicted to make room, if any. Updating an
    /// existing key never evicts and returns `None`.
    pub fn put(&mut self, key: K, value: V) -> Option<(K, V)> {
        self.tick += 1;
        let tick = self.tick;
        if let Some(entry) = self.map.get_mut(&key) {
            self.order.remove(&entry.1);
            entry.0 = value;
            entry.1 = tick;
            self.order.insert(tick, key);
            return None;
        }
        let mut evicted = None;
        if self.map.len() == self.capacity.get()
            && let Some((_, old_key)) = self.order.pop_first()
        {
            evicted = self.map.remove(&old_key).map(|(v, _)| (old_key, v));
        }
        self.order.insert(tick, key.clone());
        self.map.insert(key, (value, tick));
        evicted
    }

    /// Looks up `key` and marks it as the most recently used.
    pub fn get<Q>(&mut self, key: &Q) -> Option<&V>
    where
        K: Borrow<Q>,
        Q: Hash + Eq + ?Sized,
    {
        let entry = self.map.get_mut(key)?;
        let owned_key = self
            .order
            .remove(&entry.1)
            .expect("order and map are kept in sync");
        self.tick += 1;
        entry.1 = self.tick;
        self.order.insert(self.tick, owned_key);
        Some(&entry.0)
    }

    /// Looks up `key` without changing its recency.
    pub fn peek<Q>(&self, key: &Q) -> Option<&V>
    where
        K: Borrow<Q>,
        Q: Hash + Eq + ?Sized,
    {
        self.map.get(key).map(|(v, _)| v)
    }

    /// True if `key` is present (does not change recency).
    pub fn contains_key<Q>(&self, key: &Q) -> bool
    where
        K: Borrow<Q>,
        Q: Hash + Eq + ?Sized,
    {
        self.map.contains_key(key)
    }

    /// Removes `key`, returning its value.
    pub fn remove<Q>(&mut self, key: &Q) -> Option<V>
    where
        K: Borrow<Q>,
        Q: Hash + Eq + ?Sized,
    {
        let (value, tick) = self.map.remove(key)?;
        self.order.remove(&tick);
        Some(value)
    }

    /// Iterates from the most recently used entry to the least recently used.
    pub fn iter_mru(&self) -> impl Iterator<Item = (&K, &V)> {
        self.order
            .values()
            .rev()
            .filter_map(|k| self.map.get(k).map(|(v, _)| (k, v)))
    }
}

/// Keeps the `k` largest items seen so far using a min-heap of size `k`.
///
/// Memory is O(k) and each `push` is O(log k), so it works on streams that do
/// not fit in memory. `k == 0` keeps nothing.
#[derive(Debug)]
pub struct TopK<T: Ord> {
    k: usize,
    heap: BinaryHeap<Reverse<T>>,
}

impl<T: Ord> TopK<T> {
    /// Creates a collector for the `k` largest items.
    pub fn new(k: usize) -> Self {
        TopK {
            k,
            heap: BinaryHeap::with_capacity(k),
        }
    }

    /// Offers an item; it is kept only if it is among the `k` largest so far.
    pub fn push(&mut self, item: T) {
        if self.k == 0 {
            return;
        }
        if self.heap.len() < self.k {
            self.heap.push(Reverse(item));
        } else if let Some(mut smallest) = self.heap.peek_mut()
            && item > smallest.0
        {
            *smallest = Reverse(item);
        }
    }

    /// Number of items currently kept (at most `k`).
    pub fn len(&self) -> usize {
        self.heap.len()
    }

    /// True if nothing is kept.
    pub fn is_empty(&self) -> bool {
        self.heap.is_empty()
    }

    /// Consumes the collector and returns the kept items, largest first.
    pub fn into_sorted_desc(self) -> Vec<T> {
        // Ascending order of `Reverse<T>` is descending order of `T`.
        self.heap
            .into_sorted_vec()
            .into_iter()
            .map(|r| r.0)
            .collect()
    }
}

/// Counts words (maximal runs of alphanumeric characters, lowercased).
///
/// A `BTreeMap` makes the result order deterministic, which matters for tests,
/// reports and anything that must be reproducible.
pub fn word_counts(text: &str) -> BTreeMap<String, usize> {
    let mut counts = BTreeMap::new();
    for word in text
        .split(|c: char| !c.is_alphanumeric())
        .filter(|w| !w.is_empty())
    {
        *counts.entry(word.to_lowercase()).or_insert(0) += 1;
    }
    counts
}

/// Groups items by a key, keeping each group in input order and the groups sorted by key.
pub fn group_by<T, K: Ord>(
    items: impl IntoIterator<Item = T>,
    key: impl Fn(&T) -> K,
) -> BTreeMap<K, Vec<T>> {
    let mut groups: BTreeMap<K, Vec<T>> = BTreeMap::new();
    for item in items {
        groups.entry(key(&item)).or_default().push(item);
    }
    groups
}

#[cfg(test)]
mod tests {
    use super::*;

    fn cap(n: usize) -> NonZeroUsize {
        NonZeroUsize::new(n).unwrap()
    }

    #[test]
    fn evicts_least_recently_used() {
        let mut c = LruCache::new(cap(2));
        assert_eq!(c.put(1, "a"), None);
        assert_eq!(c.put(2, "b"), None);
        assert_eq!(c.put(3, "c"), Some((1, "a")));
        assert!(!c.contains_key(&1));
        assert_eq!(c.len(), 2);
    }

    #[test]
    fn get_refreshes_but_peek_does_not() {
        let mut c = LruCache::new(cap(2));
        c.put("a", 1);
        c.put("b", 2);
        assert_eq!(c.peek("a"), Some(&1)); // does not refresh
        assert_eq!(c.put("c", 3), Some(("a", 1)));
        let mut c = LruCache::new(cap(2));
        c.put("a", 1);
        c.put("b", 2);
        assert_eq!(c.get("a"), Some(&1)); // refreshes
        assert_eq!(c.put("c", 3), Some(("b", 2)));
    }

    #[test]
    fn updating_existing_key_does_not_evict() {
        let mut c = LruCache::new(cap(2));
        c.put("a", 1);
        c.put("b", 2);
        assert_eq!(c.put("a", 10), None);
        assert_eq!(c.len(), 2);
        assert_eq!(c.peek("a"), Some(&10));
        // "a" is now most recent, so "b" goes first.
        assert_eq!(c.put("c", 3), Some(("b", 2)));
    }

    #[test]
    fn remove_and_mru_order() {
        let mut c = LruCache::new(cap(3));
        c.put(1, 'a');
        c.put(2, 'b');
        c.put(3, 'c');
        c.get(&1);
        let order: Vec<_> = c.iter_mru().map(|(k, _)| *k).collect();
        assert_eq!(order, vec![1, 3, 2]);
        assert_eq!(c.remove(&3), Some('c'));
        assert_eq!(c.remove(&3), None);
        let order: Vec<_> = c.iter_mru().map(|(k, _)| *k).collect();
        assert_eq!(order, vec![1, 2]);
    }

    #[test]
    fn capacity_one_and_borrowed_lookup() {
        let mut c: LruCache<String, i32> = LruCache::new(cap(1));
        c.put("x".to_string(), 1);
        assert_eq!(c.get("x"), Some(&1)); // &str lookup of a String key
        assert_eq!(c.put("y".to_string(), 2), Some(("x".to_string(), 1)));
        assert_eq!(c.capacity(), 1);
        assert!(!c.is_empty());
    }

    /// Compare against a naive Vec model on a deterministic pseudo-random workload.
    #[test]
    fn matches_naive_model() {
        let mut state = 12345u64;
        let mut next = move || {
            state = state
                .wrapping_mul(6364136223846793005)
                .wrapping_add(1442695040888963407);
            (state >> 33) as u32
        };
        let capacity = 4;
        let mut c = LruCache::new(cap(capacity));
        let mut model: Vec<(u32, u32)> = Vec::new(); // front = least recent
        for _ in 0..5000 {
            let key = next() % 8;
            match next() % 3 {
                0 => {
                    let val = next();
                    let evicted = c.put(key, val);
                    let mut expected = None;
                    if let Some(pos) = model.iter().position(|e| e.0 == key) {
                        model.remove(pos);
                    } else if model.len() == capacity {
                        expected = Some(model.remove(0));
                    }
                    model.push((key, val));
                    assert_eq!(evicted, expected);
                }
                1 => {
                    let got = c.get(&key).copied();
                    let want = model.iter().position(|e| e.0 == key).map(|pos| {
                        let e = model.remove(pos);
                        model.push(e);
                        e.1
                    });
                    assert_eq!(got, want);
                }
                _ => {
                    let got = c.remove(&key);
                    let want = model
                        .iter()
                        .position(|e| e.0 == key)
                        .map(|pos| model.remove(pos).1);
                    assert_eq!(got, want);
                }
            }
            assert_eq!(c.len(), model.len());
            assert_eq!(c.map.len(), c.order.len());
        }
    }

    #[test]
    fn top_k_basics() {
        let mut t = TopK::new(3);
        for x in [5, 1, 9, 3, 9, 7] {
            t.push(x);
        }
        assert_eq!(t.len(), 3);
        assert_eq!(t.into_sorted_desc(), vec![9, 9, 7]);

        let mut few = TopK::new(5);
        few.push(2);
        few.push(1);
        assert_eq!(few.into_sorted_desc(), vec![2, 1]);

        let mut none = TopK::new(0);
        none.push(1);
        assert!(none.is_empty());
        assert!(none.into_sorted_desc().is_empty());
    }

    #[test]
    fn top_k_matches_sort_and_truncate() {
        let mut state = 99u64;
        let data: Vec<u32> = (0..1000)
            .map(|_| {
                state = state.wrapping_mul(6364136223846793005).wrapping_add(1);
                (state >> 40) as u32 % 100
            })
            .collect();
        let mut t = TopK::new(10);
        for &x in &data {
            t.push(x);
        }
        let mut expected = data.clone();
        expected.sort_unstable_by(|a, b| b.cmp(a));
        expected.truncate(10);
        assert_eq!(t.into_sorted_desc(), expected);
    }

    #[test]
    fn counts_and_groups_are_deterministic() {
        let c = word_counts("The cat, the DOG; the end. Cat!");
        let pairs: Vec<_> = c.iter().map(|(k, v)| (k.as_str(), *v)).collect();
        assert_eq!(pairs, vec![("cat", 2), ("dog", 1), ("end", 1), ("the", 3)]);
        assert!(word_counts("  ,, ").is_empty());

        let g = group_by(["apple", "avocado", "banana"], |w| w.chars().next());
        assert_eq!(g[&Some('a')], vec!["apple", "avocado"]);
        assert_eq!(g.len(), 2);
    }
}
```

✓ With this exact file: `cargo fmt --check` clean, `cargo clippy --all-targets --locked -- -D warnings` clean, `cargo test --locked` → 9 unit tests and 1 doctest passed. (Clippy's `collapsible_if` lint asked for let chains in two places on the first attempt; I applied them, which is why the code uses `&& let`; that syntax needs edition 2024.)

**Design notes.**

- **Two structures, one invariant.** `map: HashMap<K, (V, tick)>` finds a value in O(1) average; `order: BTreeMap<tick, K>` keeps keys sorted by last use, so the *smallest* tick is the least recently used and `pop_first` evicts it in O(log n). Every mutation updates both, and the model test asserts `map.len() == order.len()` after each of 5,000 operations.
- **Why not a `VecDeque` for recency?** Moving an arbitrary key to the back is O(n) (find it, remove it). The tick-keyed `BTreeMap` makes that O(log n). (The classic O(1) LRU uses a linked list threaded through the map; in safe Rust that needs indices or `unsafe`, and is Lesson 12 territory.)
- **`get` without cloning the key.** `order.remove(&tick)` returns the owned key, which is re-inserted under the new tick, so the borrowed lookup type `Q` never needs `ToOwned`. The `expect` documents the invariant; it can only fire if the code itself is wrong, never because of caller input.
- **`put` clones the key once** on insertion because both maps need an owned copy (hence `K: Clone`).
- **Bounded memory.** The capacity is a `NonZeroUsize` (Lesson 15), so a zero-capacity cache is unrepresentable; the map is pre-sized with `with_capacity`. An attacker cannot make this cache grow beyond `capacity`.
- **`TopK`** keeps a min-heap of at most `k` items; `peek_mut` lets it replace the smallest in place and re-sift on drop, instead of `pop` then `push`. Memory is O(k).
- **Determinism.** `word_counts` and `group_by` return `BTreeMap`s so their output order never depends on a hash seed.
- **Not production code.** No thread safety (use a `Mutex`, Lesson 29), no TTL, no metrics, and a `u64` tick that is assumed not to overflow.

</details>

---

## 14. Interview questions

<details markdown="1">
<summary>Answers</summary>

1. **How does `Vec` grow?** It stores pointer, capacity and length. When `push` would exceed capacity it allocates a larger buffer (geometric growth), moves the elements and frees the old one; that makes `push` amortized O(1). The exact factor is not specified.
2. **`Vec::new()` vs `Vec::with_capacity(n)`?** `new` does not allocate; `with_capacity(n)` allocates room for `n` elements and leaves the length 0. Reserve when you know the size to avoid reallocations.
3. **Why can't you hold `&v[0]` while calling `v.push(...)`?** `push` may reallocate and move the buffer, leaving the reference dangling; the borrow checker rejects it (`E0502`). It's the compile-time form of C++'s iterator invalidation.
4. **`remove` vs `swap_remove`?** `remove(i)` is O(n) and preserves order; `swap_remove(i)` is O(1) but moves the last element into the gap, changing order.
5. **When use `VecDeque` over `Vec`?** When you need O(1) insertion/removal at both ends (queues, deques, sliding windows). It's a ring buffer; its contents may span two slices.
6. **`HashMap` vs `BTreeMap`?** `HashMap`: average O(1), needs `Hash + Eq`, unordered. `BTreeMap`: O(log n), needs `Ord`, sorted iteration, range queries, `first/last`.
7. **What does a `HashMap` key need, and what's the contract between `Hash` and `Eq`?** `Hash + Eq`; if `a == b` then `hash(a) == hash(b)`. `f64` can't be a key because it isn't `Eq`/`Hash` (`NaN`).
8. **Why is `HashMap` iteration order random?** Each map has a randomly seeded hasher (`RandomState`), and the table layout is an implementation detail. Don't rely on it; sort or use `BTreeMap`.
9. **Why SipHash by default?** To resist hash-flooding DoS from attacker-chosen colliding keys, at some speed cost. Swap in a faster hasher only for trusted keys.
10. **What is the `entry` API for?** To look up and insert/update with one hash and one borrow: `*m.entry(k).or_insert(0) += 1`, `.or_default().push(x)`, `.and_modify(..).or_insert(..)`. It also solves get-then-insert borrow problems.
11. **`m[k]` vs `m.get(k)`?** `[]` panics if the key is missing; `get` returns `Option<&V>`.
12. **How do you sort a `Vec<f64>`?** `v.sort_by(f64::total_cmp)`. `sort()` doesn't compile because `f64` isn't `Ord`; `partial_cmp().unwrap()` panics on `NaN`.
13. **Stable vs unstable sort?** Stable keeps the relative order of equal elements and may allocate; unstable (`sort_unstable`) may reorder equals, is usually faster, and works in place.
14. **How would you implement a min-heap with `BinaryHeap`?** Wrap items in `std::cmp::Reverse`, since `BinaryHeap` is a max-heap.
15. **How do you get the top-k of a large stream?** A size-k min-heap: for each item, if the heap has fewer than k push it, else replace the smallest if the new item is larger. O(n log k) time, O(k) memory.
16. **Why prefer `&[T]` over `&Vec<T>` in parameters?** It accepts arrays, slices and vectors (deref coercion) and states that you don't need to grow or reallocate.
17. **Why is `LinkedList` rarely used?** Per-node allocation and pointer chasing hurt cache performance; `Vec`/`VecDeque` are faster for nearly all workloads.
18. **What security issues arise with collections?** Unbounded growth, attacker-controlled `with_capacity`, hash flooding, panics on bad indices, and nondeterministic iteration order in anything that must be reproducible.

</details>

---

## 15. Key takeaways

- Default to `Vec`; know what each collection is made of and the operation it is bad at. `HashMap` for keyed lookup, `BTreeMap` for order and ranges, `VecDeque` for queues, `BinaryHeap` for priority.
- A `Vec` is (pointer, capacity, length); `push` is amortized O(1) via geometric growth, reallocation moves the buffer, and the borrow checker turns "reference into a growing buffer" into `E0502`. Growth factors are not guaranteed.
- Pre-size with `with_capacity`/`reserve`, shrink with `shrink_to_fit`, and use `get` instead of `[]` on untrusted indices. `swap_remove`, `retain`, `drain` and `VecDeque` avoid O(n²) patterns.
- Hash keys need `Hash + Eq` and agree with each other; floats aren't keys without a total-order wrapper. Iteration order of a `HashMap` is unspecified, so sort or use a `BTreeMap` when order matters.
- The `entry` API is the idiomatic way to count, group and cache; it also fixes get-then-insert borrow errors when a reference is returned.
- The default hasher is chosen for DoS resistance; replace it only for trusted keys.
- Bound every collection that outside input can grow, and validate sizes before reserving.

**Connections.** Lesson 06 (strings and slices), Lesson 08 (heap and `Drop`), Lesson 10 (the borrow errors here) and Lesson 11 (the returned-borrow limitation) are the foundation; Lesson 13's derives (`Hash`, `Ord`) make types usable as keys; Lesson 15's newtypes supply total orders and non-zero capacities. Next: Lesson 17 (iterators and adapters: `map`, `filter`, `collect`, `windows` and `chunks` revisited as iterator chains) and Lesson 18 (closures, used in `retain`, `sort_by_key`, `entry().or_insert_with`). Lesson 19 explains the generic bounds (`K: Hash + Eq`, `Q: ?Sized`, `Borrow`) used here; Lesson 23 covers `Rc`/`Arc` for sharing values stored in collections; Lesson 29 for sharing a map across threads (`Mutex<HashMap>`); Lesson 32 for backpressure and bounded channels; Lesson 41 for measuring collection performance; Lesson 42 for data layout; Lessons 48 to 57 for deterministic ordering in blockchain state. See the [roadmap](00_roadmap.md) and the [course index](index.md).

---

## 16. References

- *The Rust Standard Library*, `std::collections` module documentation, including "When Should I Use Which Collection?" and the performance tables: <https://doc.rust-lang.org/std/collections/index.html>
- `std::vec::Vec` (capacity and growth, `Vec` and `swap_remove`, `drain`, `retain`), `std::collections::HashMap` (the `entry` API, hashing, `get_disjoint_mut`), `std::collections::BTreeMap` (`range`), `std::collections::VecDeque`, `std::collections::BinaryHeap`: <https://doc.rust-lang.org/std/vec/struct.Vec.html>, <https://doc.rust-lang.org/std/collections/struct.HashMap.html>, <https://doc.rust-lang.org/std/collections/struct.BTreeMap.html>, <https://doc.rust-lang.org/std/collections/struct.VecDeque.html>, <https://doc.rust-lang.org/std/collections/struct.BinaryHeap.html>
- *The Rust Programming Language*, ch. 8 "Common Collections": <https://doc.rust-lang.org/book/ch08-00-common-collections.html>
- *The Rustonomicon*, "Implementing Vec": <https://doc.rust-lang.org/nomicon/vec/vec.html>
- `std::hash::{Hash, Hasher, BuildHasher}` and `std::hash::RandomState`: <https://doc.rust-lang.org/std/hash/index.html>
- `f64::total_cmp`: <https://doc.rust-lang.org/std/primitive.f64.html#method.total_cmp>
- The `hashbrown` crate (SwissTable implementation used by `HashMap`): <https://github.com/rust-lang/hashbrown>
- Rust error codes E0502, E0499, E0507, E0277: <https://doc.rust-lang.org/error_codes/E0502.html>, <https://doc.rust-lang.org/error_codes/E0499.html>, <https://doc.rust-lang.org/error_codes/E0507.html>
- Crosby and Wallach, "Denial of Service via Algorithmic Complexity Attacks" (hash flooding background): <https://www.usenix.org/legacy/events/sec03/tech/full_papers/crosby/crosby.pdf>

The URLs above are the standard documentation locations; I did not open every one during this session (the sandbox cannot reach them), so if a link has moved, search the docs site for the page title. Unchecked from memory: that `get_disjoint_mut` was stabilized in 1.86 and let chains in 1.88 (both compile on 1.97 ✓), the approximate B-tree node width, that the `LinkedList` cursor API is still unstable, the SipHash 1-3 detail and the `hashbrown` SwissTable implementation, the exact URL of the Crosby and Wallach paper, that recent standard-library sorts may panic on an inconsistent comparator, and `try_reserve` (documented API, not run).
