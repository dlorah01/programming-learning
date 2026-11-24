---
layout: default
title: "Lesson 60 — Technical interviews and capstone projects"
---

# Lesson 60 — Technical interviews and capstone projects

**Module 19 · Professional mastery (second and last lesson; last lesson of the course)** · Status: generated · Edition used in examples: **2024** · External dependencies: **none** (1 package in `Cargo.lock`, the crate itself)

Verified while writing this lesson with `rustc 1.97.0 (2d8144b78 2026-07-07)`, `cargo 1.97.0`, `clippy 0.1.97` and `rustfmt 1.9.0-stable`, on x86-64 Linux (glibc 2.39), 2 cores. ✓ marks outputs and measurements I actually ran; unmarked statements are documented behavior, general experience, or opinion, and I say which.

**What this lesson is, and is not.** Two things remain after fifty-nine lessons: showing what you know under pressure, and building something that proves it. The first half is about **technical interviews**: a method that works on any problem, then nine problem families worked in Rust with tests that run (an LRU cache, intervals, graphs, binary search, sliding windows, dynamic programming, heaps, tries), and the Rust-specific questions interviewers ask. The second half is about **capstone projects**: five project briefs with scope, milestones and acceptance criteria, a portfolio checklist, and how to review your own work. What I did **not** do: sit or conduct a real interview, measure what any employer values, or survey the hiring market. Everything about hiring practice is experience-based opinion and is labeled so; the code is the only part I can promise.

Environment notes: crates.io and GitHub were reachable from the sandbox, but documentation sites were not. Known answers for edit distance, the longest increasing subsequence, minimum coins, shortest paths and integer square roots were computed by a **separate Python program** (`p60/py/ref.py`), not by the code under test; the other tests compare against brute-force or naive models written inside the test file.

Builds on [Lesson 16](16_collections.md) (collections), [Lesson 17](17_iterators.md) (iterators), [Lesson 07](07_ownership_and_moves.md), [Lesson 10](10_references_and_borrowing.md) and [Lesson 11](11_lifetimes.md) (ownership), [Lesson 26](26_error_design.md), [Lesson 39](39_testing.md), [Lesson 40](40_property_testing_fuzzing.md), [Lesson 41](41_benchmarking_profiling.md), [Lesson 42](42_performance_engineering.md), [Lesson 43](43_observability_ci_supply_chain.md) and, for the system-design part of an interview, [Lesson 59](59_architecture_system_design.md). This is the last lesson; there is no next one.

---

## 1. Why this lesson exists

Interviews reward a *process* more than a trick, and the same few mistakes end most of them: coding before understanding the question, an off-by-one in a loop bound, an edge case never mentioned, an answer whose complexity cannot be stated. Each claim below has an experiment in the crate.

| Claim | Experiment (all ✓ in the crate) |
|---|---|
| An LRU cache is O(1) with a map and a linked list, and in Rust the list is a `Vec` of indices | `lru_matches_a_naive_model_on_random_operations` compares 2000 random operations at five capacities against a `Vec` model (§4.1) |
| Intervals: sort, then one pass; "touching" and "back to back" are different questions | `merge` joins `(1,4),(4,5)`; `max_overlap` of `(0,10),(10,20)` is 1 (§4.2) |
| A topological order must not offer a node before all its prerequisites | a node needing 0 and 3 is not offered after 0 alone (§4.3, found by mutation) |
| Dijkstra needs a "stale entry" check, and a zero-weight cycle makes the wrong comparison loop forever | a `<=` mutant hangs; the real one is checked against Bellman-Ford on 200 random graphs (§4.3) |
| `(lo + hi) / 2` is a bug | ✓ with a range near `u32::MAX` it panics; `lo + (hi - lo) / 2` does not (§4.4, §15) |
| The sliding window must not move backwards | `longest_unique("abba")` is 2, not 3; `"tmmzuxt"` is 5 (§4.5) |
| Greedy fails for coin change | with coins `[1,3,4]` and amount 6, greedy takes 3 coins and the DP takes 2 (§4.6) |
| A heap of size *k* selects in O(n log k) | `top_k_frequent` matches a full sort on 300 random inputs (§4.7) |
| Testing against a slow, obviously correct oracle finds what examples miss | 11 of the tests are oracle comparisons (§5) |

The crate has **38 tests**, passing in 8 full runs; `cargo clippy --all-targets -- -D warnings` is clean; and 85 mutants (§17) were run against it.

## 2. The landscape (from memory)

None of this section was checked against current sources.

- **Typical stages.** A recruiter screen; one or two technical screens (a coding problem in a shared editor, or a take-home); a "loop" of several interviews (coding, system design, a deep dive into past work, behavioral). Rust roles often add a code-reading or code-review exercise, a question about ownership and concurrency, and a discussion of a project you built.
- **What is being judged**, in my experience: whether you clarify before coding, whether your solution is correct on edge cases, whether you can state its complexity, whether you test it, how you react to hints, and whether you communicate while you work. Speed matters less than most candidates fear and clarity matters more.
- **Rust-specific hiring** (opinion): teams hiring Rust engineers tend to care about ownership fluency, error handling, concurrency reasoning, and the judgment to avoid `unsafe` and over-engineering. A candidate who can explain *why* the borrow checker rejects something is more convincing than one who memorized the fix.
- **Allowed tools vary.** Some interviews let you compile; many give a plain editor. Know the standard library well enough to write without autocomplete, and ask what is permitted.

## 3. A method that works on any problem

1. **Restate and clarify.** Input size and types, duplicates, negative values, empty input, sorted or not, memory limits, what to return when there is no answer. Ask; do not assume.
2. **Examples.** Work one normal example by hand, then one edge example (empty, one element, all equal, maximum values). Say them aloud; they become your tests.
3. **Brute force first.** State the obvious solution and its complexity. It is your fallback, your oracle in §5, and proof you understood the question.
4. **Find the bottleneck, then the structure.** Repeated lookup → hash map. Repeated minimum → heap. Sorted input → binary search or two pointers. Overlapping sub-problems → dynamic programming. Relationships → graph.
5. **Say the plan before the code**, including complexity. Get a nod.
6. **Write clean code**, small functions, names that explain. Handle the edge cases you listed.
7. **Test by hand**, tracing a normal and an edge example through the code. Then say what you would test next.
8. **Complexity and trade-offs.** Time, space, what breaks at 10× the input, what you would change for a streaming input.

In Rust, add one more habit: decide **ownership and error behavior** explicitly. Does the function borrow or take ownership? Return `Option`, `Result` or panic? The crate's functions return `Option` where "no answer" is natural (`kth_largest`, `min_coins`) and document every precondition, because "what happens on empty input" is the first follow-up question you will get.

## 4. Nine problem families, worked

Each subsection states the problem, the idea, the code, the boundary cases, and the tests (all ✓).

### 4.1 LRU cache: a map plus a list, without `Rc<RefCell<..>>`

*Design a cache with `get` and `put` in O(1) that evicts the least recently used entry when full.* The classic answer is a hash map from key to a node of a doubly linked list. In Rust a pointer-linked list fights the borrow checker ([Lesson 23](23_box_rc_arc.md), [Lesson 25](25_weak_pin.md)); the clean answer stores nodes in a `Vec` and links them by **index**, so there is no `unsafe`, no reference counting, and eviction reuses the evicted node's slot.

```rust
pub struct Lru<K, V> {
    cap: usize,
    map: HashMap<K, usize>,
    nodes: Vec<Node<K, V>>,
    head: usize, // most recently used
    tail: usize, // least recently used
}

fn push_front(&mut self, i: usize) {
    self.nodes[i].prev = NIL;
    self.nodes[i].next = self.head;
    if self.head != NIL {
        self.nodes[self.head].prev = i;
    }
    self.head = i;
    if self.tail == NIL {
        self.tail = i;
    }
}
```

```rust
/// Inserts or replaces. Returns the entry evicted to make room, if any.
pub fn put(&mut self, key: K, value: V) -> Option<(K, V)> {
    if self.cap == 0 {
        return Some((key, value));
    }
    if let Some(&i) = self.map.get(&key) {
        self.nodes[i].value = value;
        self.unlink(i);
        self.push_front(i);
        return None;
    }
    if self.map.len() < self.cap {
        let i = self.nodes.len();
        self.nodes.push(Node {
            key: key.clone(),
            value,
            prev: NIL,
            next: NIL,
        });
        self.map.insert(key, i);
        self.push_front(i);
        return None;
    }
    // full: reuse the least recently used slot
    let i = self.tail;
    self.unlink(i);
    let old_key = std::mem::replace(&mut self.nodes[i].key, key.clone());
    let old_value = std::mem::replace(&mut self.nodes[i].value, value);
    self.map.remove(&old_key);
    self.map.insert(key, i);
    self.push_front(i);
    Some((old_key, old_value))
}
```

Boundary cases the tests pin: capacity 0 stores nothing and returns the offered entry (`lru_capacity_zero_and_one`); replacing an existing key refreshes it and does not evict (`lru_replacing_a_value_refreshes_without_growing`); `peek` does not change the order (`lru_peek_does_not_refresh`); and a model test applies 2000 random `put`/`get` operations at capacities 1 to 5 to both the cache and a `Vec` where the freshest item is last, comparing every return value and the length (`lru_matches_a_naive_model_on_random_operations`). Follow-ups to expect: thread safety (wrap in a `Mutex`; or shard by key), a TTL per entry (store an expiry; check on `get`), an LFU variant (frequency buckets).

### 4.2 Intervals: sort, then sweep

*Merge overlapping intervals.* Sort by start, then extend the last output interval or start a new one. The decision that interviewers probe is the comparison: `s <= last.end` merges touching intervals (`(1,4)` and `(4,5)` become `(1,5)`), `s < last.end` would not (a mutant that changes it is caught).

```rust
/// Merges overlapping or touching intervals; the result is sorted and disjoint. O(n log n).
pub fn merge(mut v: Vec<(i64, i64)>) -> Vec<(i64, i64)> {
    v.sort_unstable();
    let mut out: Vec<(i64, i64)> = Vec::with_capacity(v.len());
    for (s, e) in v {
        match out.last_mut() {
            Some(last) if s <= last.1 => last.1 = last.1.max(e),
            _ => out.push((s, e)),
        }
    }
    out
}
```

*How many rooms are needed for a set of meetings?* Turn each meeting into two events (+1 at the start, -1 at the end), sort, sweep, and take the maximum running count. With half-open meetings an end and a start at the same time must not overlap, so at equal times the **end sorts first** (`-1 < +1` in the tuple order): `max_overlap(&[(0,10),(10,20)])` is 1, and a mutant that reverses the order is caught.

```rust
/// The most meetings that are in progress at the same moment. Meetings are half-open
/// `[start, end)`, so one ending at 10 and one starting at 10 do not overlap. O(n log n).
pub fn max_overlap(meetings: &[(i64, i64)]) -> usize {
    let mut events: Vec<(i64, i32)> = Vec::with_capacity(meetings.len() * 2);
    for &(s, e) in meetings {
        if s < e {
            events.push((s, 1));
            events.push((e, -1));
        }
    }
    // at equal times, ends (-1) sort before starts (+1)
    events.sort_unstable();
    let (mut cur, mut best) = (0i32, 0i32);
    for (_, d) in events {
        cur += d;
        best = best.max(cur);
    }
    best as usize
}
```

✓ `max_overlap_matches_brute_force` checks 300 random meeting sets against "count the meetings covering each integer time". One mutant, accepting empty meetings `(5,5)` as events, is equivalent: its `-1` sorts before its `+1`, so the running count dips and returns and never exceeds the true maximum.

### 4.3 Graphs: topological order, shortest paths, union-find

**Topological sort** (*schedule tasks with prerequisites*): repeatedly take a node with no unmet prerequisites. Kahn's algorithm counts incoming edges and releases a node when the count reaches zero. Using a `BTreeSet` as the ready set makes the result deterministic (smallest available first), which makes it testable; a plain queue is fine when any valid order will do. A cycle leaves nodes unreleased, so the output is shorter than `n`.

```rust
/// A topological order (smallest node first among the available ones), or `None` if there is a cycle.
/// `edges` are `(from, to)`; nodes outside `0..n` are ignored. O((n + e) log n).
pub fn topo_sort(n: usize, edges: &[(usize, usize)]) -> Option<Vec<usize>> {
    let mut indeg = vec![0usize; n];
    let mut adj = vec![Vec::new(); n];
    for &(a, b) in edges {
        if a < n && b < n {
            adj[a].push(b);
            indeg[b] += 1;
        }
    }
    let mut ready: BTreeSet<usize> = (0..n).filter(|&i| indeg[i] == 0).collect();
    let mut out = Vec::with_capacity(n);
    while let Some(&u) = ready.iter().next() {
        ready.remove(&u);
        out.push(u);
        for &v in &adj[u] {
            indeg[v] -= 1;
            if indeg[v] == 0 {
                ready.insert(v);
            }
        }
    }
    if out.len() == n { Some(out) } else { None }
}
```

✓ The tests: a chain, a cycle, a self loop (a cycle), no edges, edges to missing nodes ignored, and "smallest available first" (`topo_sort_orders_and_detects_cycles`); and 200 random acyclic graphs where the result must put every edge's source before its target (`topo_sort_result_respects_every_edge`). **The mutation run found a gap here**: a mutant that released a node when its count dropped to 1 instead of 0 survived. In every random graph the edges ran from a smaller to a larger number, so the smallest-first rule picked the prerequisite before the early-released node anyway, and the error never showed. The test `a_node_with_two_prerequisites_waits_for_both` (edges `0→2` and `3→2`, expected `[0, 1, 3, 2]`) closes it. The lesson for interviews: the example you pick to test by hand should be the one with *two* prerequisites, not the chain.

**Shortest paths with non-negative weights** (Dijkstra): a min-heap of `(distance, node)`; pop the closest unsettled node; skip **stale** entries (the same node pushed earlier with a longer distance); relax its edges. In Rust, `BinaryHeap` is a max-heap, so wrap the key in `Reverse`.

```rust
/// Shortest distances from `src` over non-negative weights; `None` for unreachable nodes.
/// `edges` are `(from, to, weight)`. O((n + e) log n).
pub fn dijkstra(n: usize, edges: &[(usize, usize, u64)], src: usize) -> Vec<Option<u64>> {
    let mut adj = vec![Vec::new(); n];
    for &(a, b, w) in edges {
        if a < n && b < n {
            adj[a].push((b, w));
        }
    }
    let mut dist: Vec<Option<u64>> = vec![None; n];
    if src >= n {
        return dist;
    }
    let mut heap = BinaryHeap::new();
    dist[src] = Some(0);
    heap.push(Reverse((0u64, src)));
    while let Some(Reverse((d, u))) = heap.pop() {
        if dist[u].is_some_and(|best| d > best) {
            continue; // a stale heap entry
        }
        for &(v, w) in &adj[u] {
            let nd = d.saturating_add(w);
            if dist[v].is_none_or(|cur| nd < cur) {
                dist[v] = Some(nd);
                heap.push(Reverse((nd, v)));
            }
        }
    }
    dist
}
```

✓ The known answer from Python for the graph `0→1 (4), 0→2 (1), 2→1 (2), 1→3 (1), 2→3 (5)` is distances `[0, 3, 1, 4, unreachable]` (`dijkstra_known_answer`); `dijkstra_matches_bellman_ford` compares 200 random graphs against a Bellman-Ford relaxation loop; edge cases include zero-weight edges, parallel edges, a directed edge the wrong way, an out-of-range source (`a_source_equal_to_the_node_count_is_out_of_range`) and edges to missing nodes (`an_edge_to_a_missing_node_is_ignored`, both added after mutation). Two mutants are instructive: changing the relaxation test from `nd < cur` to `nd <= cur` makes the program **loop forever** on a zero-weight cycle (the suite hangs; I count a hang as detected), and `wrapping_add` instead of `saturating_add` is caught by the test with a `u64::MAX` weight. Say the limits aloud: it is wrong for negative weights (use Bellman-Ford), and `O((n + e) log n)` with a binary heap.

**Union-find** (*connected components, cycle detection in an undirected graph, Kruskal*): each set is a tree; `find` follows parents to the root and compresses the path; `union` attaches the smaller tree under the larger.

```rust
impl UnionFind {
    pub fn new(n: usize) -> UnionFind {
        UnionFind {
            parent: (0..n).collect(),
            size: vec![1; n],
            sets: n,
        }
    }
    pub fn find(&mut self, x: usize) -> usize {
        let mut root = x;
        while self.parent[root] != root {
            root = self.parent[root];
        }
        let mut cur = x;
        while self.parent[cur] != root {
            let next = self.parent[cur];
            self.parent[cur] = root;
            cur = next;
        }
        root
    }
    /// Joins the sets of `a` and `b`; false if they were already one.
    pub fn union(&mut self, a: usize, b: usize) -> bool {
        let (mut ra, mut rb) = (self.find(a), self.find(b));
        if ra == rb {
            return false;
        }
        if self.size[ra] < self.size[rb] {
            std::mem::swap(&mut ra, &mut rb);
        }
        self.parent[rb] = ra;
        self.size[ra] += self.size[rb];
        self.sets -= 1;
        true
    }
    pub fn sets(&self) -> usize {
        self.sets
    }
    pub fn size_of(&mut self, x: usize) -> usize {
        let r = self.find(x);
        self.size[r]
    }
}
```

✓ `union_find_matches_a_label_model` applies 200 random unions and checks the set count and every element's set size against a relabeling model. Two mutants survive and are **equivalent in behavior**: removing path compression and flipping union-by-size change only the *speed* (the depth of the trees), never an answer, so no functional test can see them. A benchmark could, which is a useful thing to say in an interview when asked "how would you test the optimization?".

### 4.4 Binary search and its traps

*Find the first position where a sorted array is at least `target`.* The loop `while lo < hi` with `hi = mid` and `lo = mid + 1` is the form that is easy to get right: the invariant is "the answer is in `lo..=hi`", and each step shrinks the range.

```rust
/// The first index `i` with `v[i] >= target` (`v.len()` if none); `v` must be sorted. O(log n).
pub fn lower_bound(v: &[i64], target: i64) -> usize {
    let (mut lo, mut hi) = (0usize, v.len());
    while lo < hi {
        let mid = lo + (hi - lo) / 2; // not (lo + hi) / 2: that can overflow
        if v[mid] < target {
            lo = mid + 1;
        } else {
            hi = mid;
        }
    }
    lo
}
```

Traps, each tested: the midpoint `lo + (hi - lo) / 2` instead of `(lo + hi) / 2` (the sum can overflow; see §15, where a real overflow is demonstrated); `<` versus `<=` (the first element `>=` target versus the first `>`; `lower_bound_cases` checks duplicates `[1,2,2,2,5,9]`: `lower_bound(2) = 1`); the empty slice; a target above everything (returns `len`). ✓ `lower_bound_matches_partition_point` compares 300 random arrays and every target from -1 to 11 with the standard library's `partition_point`. Mutants that change `<` to `<=`, the loop condition to `<=` or the midpoint all fail (the loop-condition and midpoint mutants make the search loop forever).

**Rotated sorted array** (*search in `[4,5,6,7,0,1,2]`*): at each step one half is sorted; decide whether the target lies in the sorted half.

```rust
/// Index of `target` in a sorted array of distinct values that was rotated at an unknown pivot.
/// O(log n).
pub fn search_rotated(v: &[i64], target: i64) -> Option<usize> {
    if v.is_empty() {
        return None;
    }
    let (mut lo, mut hi) = (0usize, v.len() - 1);
    while lo <= hi {
        let mid = lo + (hi - lo) / 2;
        if v[mid] == target {
            return Some(mid);
        }
        if v[lo] <= v[mid] {
            // the left half is sorted
            if v[lo] <= target && target < v[mid] {
                if mid == 0 {
                    return None;
                }
                hi = mid - 1;
            } else {
                lo = mid + 1;
            }
        } else if v[mid] < target && target <= v[hi] {
            lo = mid + 1;
        } else {
            if mid == 0 {
                return None;
            }
            hi = mid - 1;
        }
    }
    None
}
```

✓ `rotated_search_every_rotation` tries **every rotation of arrays of length 0 to 11** and every element, and four absent values. This is the right way to test this problem: its bugs live in the boundaries between "sorted half" cases, and exhausting small cases is cheaper than reasoning about them. The `<=` in `v[lo] <= v[mid]` matters when `lo == mid` (a one-element left half): the mutant that writes `<` there is caught by this exhaustive test, and I did not find a smaller test that catches it, which is the argument for exhausting small cases.

*Integer square root without floating point*: binary search on the answer. ✓ `isqrt` agrees with Python's `math.isqrt` on `[0, 1, 2, 3, 4, 15, 16, 17, 10^12, 2^64 - 1]`; the bound `min(n, u32::MAX)` keeps `mid * mid` from overflowing a `u64`, and the invariant tests check `isqrt(r²) = r` and `isqrt(r² - 1) = r - 1` for several `r`, including 65 535, 65 536 and 4 294 967 295.

### 4.5 Strings: the sliding window

*Longest substring without repeating characters.* Keep a window `[start, i]` with no repeats and a map from byte to its last position. When the byte at `i` was last seen **inside** the window, move `start` past it.

```rust
/// Length of the longest substring with no repeated byte. O(n).
pub fn longest_unique(s: &str) -> usize {
    let mut last: HashMap<u8, usize> = HashMap::new();
    let (mut start, mut best) = (0usize, 0usize);
    for (i, b) in s.bytes().enumerate() {
        if let Some(&j) = last.get(&b)
            && j >= start
        {
            start = j + 1;
        }
        last.insert(b, i);
        best = best.max(i + 1 - start);
    }
    best
}
```

The classic bug is moving `start` backwards: in `"abba"`, when the final `a` arrives its last position (0) is *before* the window start (2), so it must be ignored. ✓ `longest_unique("abba")` is 2 and `("tmmzuxt")` is 5; the `j >= start` mutant is caught. The random test compares 300 short strings over a 4-letter alphabet against the O(n³) brute force. Also here: `balanced` (a stack; return false on a mismatch *and* if the stack is non-empty at the end: `"(("`), and `anagrams` (count array; `"aab"` versus `"abb"` has the same letters but different counts). One mutant (skipping the length pre-check) is equivalent: different lengths cannot give all-zero counts.

### 4.6 Dynamic programming

*Edit distance*, *minimum coins*, and *longest increasing subsequence* cover the shapes you meet most. The method is always: define the state in words, write the recurrence, fix the base cases, choose the order, then see whether a row suffices.

```rust
/// Levenshtein distance (insert, delete, substitute) between two byte strings.
/// O(a·b) time, O(b) space.
pub fn edit_distance(a: &str, b: &str) -> usize {
    let (a, b) = (a.as_bytes(), b.as_bytes());
    let mut prev: Vec<usize> = (0..=b.len()).collect();
    for i in 1..=a.len() {
        let mut cur = vec![i; b.len() + 1];
        for j in 1..=b.len() {
            let sub = prev[j - 1] + usize::from(a[i - 1] != b[j - 1]);
            cur[j] = sub.min(prev[j] + 1).min(cur[j - 1] + 1);
        }
        prev = cur;
    }
    prev[b.len()]
}
```

State: `prev[j]` is the distance between the first `i-1` bytes of `a` and the first `j` of `b`; each cell is the minimum of substitution (or match), deletion and insertion. Space is one row. ✓ Python's answers: `kitten → sitting` is 3, `intention → execution` is 5, `"" → "abc"` is 3. `edit_distance_is_a_metric_on_random_strings` checks properties instead of examples: symmetry, the triangle inequality, at least the length difference, at most the longer length.

```rust
/// The fewest coins that sum to `amount`, or `None` if impossible. O(amount · coins).
pub fn min_coins(coins: &[u64], amount: u64) -> Option<u64> {
    let amount = usize::try_from(amount).ok()?;
    let mut best: Vec<Option<u64>> = vec![None; amount + 1];
    best[0] = Some(0);
    for a in 1..=amount {
        for &c in coins {
            let Ok(c) = usize::try_from(c) else { continue };
            if c == 0 || c > a {
                continue;
            }
            if let Some(prev) = best[a - c]
                && best[a].is_none_or(|cur| prev + 1 < cur)
            {
                best[a] = Some(prev + 1);
            }
        }
    }
    best[amount]
}
```

✓ The greedy trap: coins `[1,3,4]`, amount 6, the DP finds 2 (3+3) where "largest coin first" would take 4+1+1. Python's answers: 11 with `[1,2,5]` is 3; 3 with `[2]` is impossible; 6249 with `[186,419,83,408]` is 20; 63 with US coins is 6. A zero coin is skipped (otherwise a subtraction of zero would loop on the same state).

```rust
/// Length of the longest strictly increasing subsequence. O(n log n).
pub fn lis(v: &[i64]) -> usize {
    let mut tails: Vec<i64> = Vec::new();
    for &x in v {
        let i = tails.partition_point(|&t| t < x);
        if i == tails.len() {
            tails.push(x);
        } else {
            tails[i] = x;
        }
    }
    tails.len()
}
```

*Longest strictly increasing subsequence in O(n log n):* `tails[k]` is the smallest possible last element of an increasing subsequence of length `k+1`; each new value replaces the first tail that is `>=` it (or extends). `partition_point(|&t| t < x)` is exactly that search, and using `<=` instead would count equal values and answer the *non-decreasing* version (a mutant that does exactly that is caught by `[7,7,7]`, which must be 1). ✓ Python: `[10,9,2,5,3,7,101,18]` gives 4; `[0,1,0,3,2,3]` gives 4; `[7,7,7]` gives 1; the test also compares 300 random inputs to the quadratic DP. Note that `tails` is **not** the subsequence itself, a frequent wrong claim.

### 4.7 Heaps: selection

*The k most frequent items.* Count with a map, then keep a min-heap of size `k`: push each candidate, pop the worst when the size exceeds `k`. O(n log k) instead of sorting everything. Ties need a rule the interviewer can see (here, the smaller item first), so the output is deterministic.

```rust
/// The `k` most frequent items, most frequent first; ties broken by the smaller item.
/// O(n log k).
pub fn top_k_frequent(items: &[i64], k: usize) -> Vec<i64> {
    let mut counts: HashMap<i64, usize> = HashMap::new();
    for &x in items {
        *counts.entry(x).or_insert(0) += 1;
    }
    // a min-heap of the best k so far: the root is the worst of them
    let mut heap: BinaryHeap<(Reverse<usize>, i64)> = BinaryHeap::new();
    for (&x, &c) in &counts {
        heap.push((Reverse(c), x));
        if heap.len() > k {
            heap.pop();
        }
    }
    let mut out: Vec<(usize, i64)> = heap.into_iter().map(|(Reverse(c), x)| (c, x)).collect();
    out.sort_unstable_by(|a, b| b.0.cmp(&a.0).then(a.1.cmp(&b.1)));
    out.into_iter().map(|(_, x)| x).collect()
}

/// The `k`-th largest value (1-based), or `None` if there are fewer than `k` values. O(n log k).
pub fn kth_largest(v: &[i64], k: usize) -> Option<i64> {
    if k == 0 || k > v.len() {
        return None;
    }
    let mut heap = BinaryHeap::new();
    for &x in v {
        heap.push(Reverse(x));
        if heap.len() > k {
            heap.pop();
        }
    }
    heap.peek().map(|r| r.0)
}
```

✓ `top_k_frequent_matches_sorting` compares 300 random inputs and values of `k` from 0 to 4 against sorting the whole count table. Two mutants are equivalent and the reasons are worth knowing: doubling every count (`+= 2`) changes no ranking, and dropping the explicit `k == 0` test in `kth_largest` still returns `None` because the heap empties itself. `kth_largest` uses the same size-`k` min-heap: the root is the `k`-th largest.

### 4.8 Tries

*Autocomplete, prefix counting.* Each node has children by byte, an "end of word" flag, and a count of words passing through. The count is the part people forget; with it, `count_prefix` is a walk with no traversal of the subtree.

```rust
impl Trie {
    pub fn new() -> Trie {
        Trie::default()
    }

    /// Adds a word. Adding a word twice counts once.
    pub fn insert(&mut self, word: &str) {
        if self.contains(word) {
            return;
        }
        let mut n = &mut self.root;
        n.through += 1;
        for b in word.bytes() {
            n = n.kids.entry(b).or_default();
            n.through += 1;
        }
        n.end = true;
    }

    fn walk(&self, s: &str) -> Option<&Node> {
        let mut n = &self.root;
        for b in s.bytes() {
            n = n.kids.get(&b)?;
        }
        Some(n)
    }

    pub fn contains(&self, word: &str) -> bool {
        self.walk(word).is_some_and(|n| n.end)
    }

    /// How many distinct words start with `prefix` (the empty prefix counts all).
    pub fn count_prefix(&self, prefix: &str) -> usize {
        self.walk(prefix).map_or(0, |n| n.through)
    }

    /// All words starting with `prefix`, in byte order.
    pub fn words_with_prefix(&self, prefix: &str) -> Vec<String> {
        let mut out = Vec::new();
        if let Some(n) = self.walk(prefix) {
            let mut buf = prefix.as_bytes().to_vec();
            collect(n, &mut buf, &mut out);
        }
        out
    }
}
```

✓ `trie_basics` covers a prefix that is also a word (`app` inside `apple`), a duplicate insert (counted once), the empty prefix, a missing prefix, and ordered output; `trie_empty_word` pins the empty string as a word; `trie_matches_a_sorted_set` compares 200 random inserts and prefix queries with a `BTreeSet` filtered by `starts_with`. A note on bytes: the trie works on bytes, so non-ASCII text is handled byte by byte, and `words_with_prefix` rebuilds strings with `from_utf8_lossy`, which is exact for words that were inserted as `&str` and prefixes that fall on character boundaries.

### 4.9 Two more habits from the exercises

*Two-sum* (Exercise 1) is the standard "one pass with a map of what I have seen" and its trap is the element paired with itself. *Kadane* (Exercise 2) shows how an "empty subarray" assumption changes the answer on all-negative input. Both come with the same discipline: say the invariant ("`cur` is the best sum of a run ending here").

## 5. Testing under interview conditions

You rarely have a test framework in an interview, but you always have a brain and a few minutes.

1. **Trace the normal example** through your code, writing the variable values.
2. **Trace the edge example** (empty, one, all equal).
3. **Name the invariant** of each loop in one sentence. If you cannot, the loop is probably wrong.
4. **Offer the oracle.** "I would test it against the brute force on random small inputs." This is the technique the crate uses: ✓ 11 of the 38 tests compare a fast algorithm to a slow, obviously correct one on hundreds of small random inputs (LRU, overlap, topo order, Dijkstra, union-find, `lower_bound`, longest-unique, LIS, top-k, trie, plus the metric properties of edit distance). A tiny linear congruential generator in the test file (`Lcg`) keeps them deterministic and dependency-free ([Lesson 40](40_property_testing_fuzzing.md) shows `proptest` for the real thing).
5. **Exhaust small cases** when the logic is boundary-heavy (the rotated-array test).

The mutation run is the same idea turned on the tests: of 85 small edits, the ones that survived told me which of *my* tests were weak (§17).

## 6. The Rust questions behind the algorithm

Interviewers for Rust roles usually branch from a coding problem into the language. The short answers below are the outline of what a good answer contains; the lessons that teach each are linked.

- **"Why can't I return a reference to a local?"** The value is dropped at the end of the function; the reference would dangle. Return the owned value, or take the storage as a parameter ([Lesson 11](11_lifetimes.md)).
- **"`&str` versus `String`; `&[T]` versus `Vec<T>`?"** Borrowed view versus owner. Accept the borrowed form in parameters (all the functions in the crate take `&str` and `&[i64]`), return the owned form when you create data ([Lesson 06](06_strings_conversions_patterns.md)).
- **"`Option` versus `Result` versus panic?"** `Option` when absence is normal (`kth_largest` of too few elements), `Result` when the caller can act on why, panic for bugs and broken invariants ([Lesson 26](26_error_design.md)).
- **"What does `Send` mean? `Sync`?"** `T: Send` means a value can move to another thread; `T: Sync` means `&T` can be shared across threads ([Lesson 22](22_gats_send_sync_phantom.md)). `Rc` is neither; `Arc<Mutex<T>>` is the shared-mutable pattern ([Lesson 24](24_interior_mutability.md)).
- **"Why is this `HashMap` iteration order different between runs?"** The default hasher is randomly seeded to resist collision attacks; do not depend on the order, and sort or use `BTreeMap` for deterministic output (the crate does, in `topo_sort`, `top_k_frequent`'s tie-break and `group_anagrams`) ([Lesson 16](16_collections.md)).
- **"What does `async` do to a function?"** It returns a future that does nothing until polled; it is cooperative, so a blocking call stalls the executor thread ([Lesson 30](30_futures_async.md), [Lesson 31](31_tokio.md)).
- **"When is `unsafe` justified?"** When the compiler cannot see an invariant you can prove and document, and then in the smallest block with a safety comment ([Lesson 36](36_unsafe_raw_pointers.md)). A good interview answer includes "and I would test it under Miri".
- **"How would you make this faster?"** Measure first ([Lesson 41](41_benchmarking_profiling.md)), then the algorithm, then allocations and memory layout ([Lesson 42](42_performance_engineering.md)).

## 7. System design and behavioral rounds

**System design** is [Lesson 59](59_architecture_system_design.md) in a conversation: requirements, estimates (Little's law, servers needed), the simplest design, then where it breaks, with the reliability machinery (rate limits, timeouts, retries with budgets, breakers, idempotency, shedding, consistent hashing, SLOs). Practice by picking a service you use and writing one page: the numbers, the design, the three likeliest failures and what the user sees.

**Behavioral and deep-dive rounds** (opinion): prepare three stories about your own work in the shape *situation, what you decided and why, what happened, what you would change*. Choose stories where something went wrong. Be exact about what *you* did versus the team. The capstone projects below are meant to give you those stories.

## 8. Capstone projects

A capstone is a project you can defend line by line. Pick **one** and finish it, rather than starting three. Each brief has a **scope**, **milestones**, **acceptance criteria** and **the lessons it exercises**. A finished small project beats an ambitious unfinished one.

### 8.1 Capstone A — a command-line tool (about 2 weekends)

*A log analyzer, a personal-finance importer, a markdown link checker, or similar.* **Scope:** reads files or standard input, parses with `clap`, handles errors with context, prints a clear report, supports `--json`. **Milestones:** (1) parse and model the data with types; (2) core logic as a library crate with unit tests; (3) the CLI with integration tests that run the binary; (4) error messages a stranger can act on; (5) a README with examples. **Acceptance:** `cargo test`, `clippy -D warnings` and `fmt --check` pass in CI; exit codes are meaningful; a 100 MB input does not exhaust memory (streams, not slurps); a benchmark or measurement for the hot path. **Exercises:** Lessons 03, 14, 17, 26, 39, 44.

### 8.2 Capstone B — an HTTP service with a database (about 3 weekends)

*A URL shortener, a bookmark service, a paste bin.* **Scope:** REST endpoints, persistent storage, authentication, configuration from the environment, structured logs. **Milestones:** (1) the domain as a library with no I/O; (2) storage behind a trait, with an in-memory and a real implementation; (3) the HTTP layer; (4) auth and input validation; (5) rate limiting and timeouts using the ideas of [Lesson 59](59_architecture_system_design.md); (6) a container image and a health check. **Acceptance:** the same test suite runs against both storage implementations; load test numbers recorded with the machine described; `429` and `503` paths tested; secrets never logged. **Exercises:** Lessons 31, 45, 46, 47, 43, 59.

### 8.3 Capstone C — a concurrent job queue (about 3 weekends)

*An in-process or networked work queue with retries.* **Scope:** workers, a bounded queue, at-least-once delivery, a retry policy with backoff and a budget, graceful shutdown. **Milestones:** (1) a single-threaded model with a fake clock and tests (copy the style of `reliab`); (2) worker threads or tasks; (3) cancellation and shutdown that drains; (4) persistence so jobs survive a restart; (5) metrics. **Acceptance:** no job lost or duplicated in a test that kills workers at random points; shutdown completes within a stated bound; the queue is bounded and refuses with a clear error when full. **Exercises:** Lessons 27 to 32, 52, 59.

### 8.4 Capstone D — a small storage engine (about 4 weekends)

*An append-only key-value store with an in-memory index and compaction.* **Scope:** `put`, `get`, `delete`, crash safety, a log format with checksums. **Milestones:** (1) the record format and its encoder/decoder, tested against hand-computed bytes; (2) append and index; (3) recovery that stops at the first corrupt record; (4) compaction; (5) a property test that compares the store with a `HashMap` under random operations *including restarts*. **Acceptance:** a truncated or corrupted file never panics and never returns wrong data; the model test passes with thousands of operations; a benchmark for put/get. **Exercises:** Lessons 16, 36 (only if you need it), 37, 40, 41, 42.

### 8.5 Capstone E — a blockchain-facing service (about 4 weekends)

*An indexer or a relayer for a chain of your choice.* **Scope:** follows a chain, survives reorganizations, never double-processes, exposes a query API. **Milestones:** (1) a model chain you control, with reorganizations, for tests ([Lesson 58](58_cross_chain_relayers.md)); (2) the indexer with rollback; (3) idempotent handling keyed by a stable id; (4) the real client, behind a trait; (5) retries with a budget and a circuit breaker ([Lesson 59](59_architecture_system_design.md)). **Acceptance:** a reorganization test with depth greater than the finality assumption is documented as a limit, not hidden; replaying the same blocks twice changes nothing. **Exercises:** Lessons 48 to 52, and one of 53, 56, 57.

### 8.6 How to scope and finish

- Write the acceptance criteria *first* and put them in the README. If you cannot test a criterion, rewrite it.
- Build the smallest vertical slice (input to output) in the first weekend. Everything after is widening.
- Cut features, never tests. A smaller project with real tests is more convincing.
- Keep a `DECISIONS.md` of five-line records ([Lesson 59](59_architecture_system_design.md), §3.4). They become your interview answers.
- Finish: tag a release, make the README honest about what is not done.

## 9. Making the work visible

A reviewer reads a repository for about a minute before deciding whether to read more (opinion). Make the minute count:

1. **README** with one sentence of purpose, a 10-line quick start that works, one screenshot or sample output, a section called "Limits" and a section called "How it is tested".
2. **Tests that run in CI** with the badge. `cargo fmt --check`, `cargo clippy -- -D warnings`, `cargo test`, and, where it matters, `cargo audit` or `cargo deny` ([Lesson 43](43_observability_ci_supply_chain.md)).
3. **A benchmark with the machine written down**, as this course did: ranges over runs, not one number.
4. **Small commits with explanations.** The history is part of the work.
5. **No dead code, no commented-out blocks, no `TODO` without an issue.**
6. **A license and a `Cargo.toml` with accurate metadata** if it is a library.

## 10. Cost ✓

Release build, x86-64 Linux, 2 cores, `black_box` around results, three runs of `examples/bench.rs` after the final source. Ranges are min to max of the three:

| Operation | per operation (3 runs) |
|---|---|
| LRU `put` + `get` at capacity 1000, keys cycling over 1500 | 22.6 to 28.8 ns |
| `lower_bound` in 1 million sorted `i64` (random targets) | 237.9 to 265.1 ns |
| `edit_distance` of two 270-byte strings (72 900 cells) | 189 289 to 260 857 ns in total (about 2.6 to 3.6 ns per cell) |
| `dijkstra`, 10 000 nodes and 50 000 edges (whole run) | 1.80 to 2.50 ms |
| trie insert, 50 000 words (includes a `contains` check first) | 136.6 to 184.0 ns |
| trie `contains`, 50 000 words | 52.7 to 69.9 ns |

Read these as *shape*. `lower_bound` takes about 250 ns, far more than the 20 steps of arithmetic would suggest, because an 8 MB array does not fit in the cache and each step is a likely cache miss; the same search on a small array would be an order of magnitude faster (not measured here). The LRU's 25 ns is two hash lookups and a few index writes. The numbers say that the asymptotic story holds and that memory behavior sets the constants, which is the honest answer to "why is my O(log n) slow?".

## 11. Rust notes

- **Indices instead of pointers** (the LRU's `prev`/`next` are `usize` with a `NIL` sentinel) is the standard Rust answer to linked structures: safe, cache-friendlier than boxed nodes, and trivially `Send`. The price: stale indices are possible if you ever free slots, which this cache avoids by reusing the evicted slot.
- **`BinaryHeap` is a max-heap.** `Reverse(x)` flips it; tuples compare lexicographically, which is how `(Reverse(count), item)` encodes "fewest count first, then largest item", so `pop` removes the worst candidate.
- **`let` chains** (edition 2024) let you write `if let Some(x) = a && cond { .. }` instead of a nested `if`; clippy in this toolchain asks for it (`collapsible_if`), and the crate uses them.
- **`Option::is_none_or`** and **`is_some_and`** read better than `map_or(true, ..)`.
- **`slice::partition_point`** is binary search for "the first element where the predicate is false"; know it, but also be able to write it by hand.
- **Checked and saturating arithmetic** appear wherever the input can be adversarial (`lcm`, `climb`, `product_except_self` in the exercises). An interview answer that says "this can overflow; I would return `None`" earns more than silence.
- **Do not fight the borrow checker in an interview.** If a reference-heavy design is not compiling, switch to indices or clone once and say you would optimize later.

## 12. Common mistakes

1. **Coding before clarifying**, then discovering the input can be empty.
2. **No brute force**, so no oracle and no fallback.
3. **Skipping the complexity statement.**
4. **Off-by-one in a binary search**, usually the mix of `<`/`<=` and `mid`/`mid + 1`.
5. **`(lo + hi) / 2`** on large indices.
6. **Moving a sliding window's start backwards.**
7. **Greedy where the problem needs DP** (coin change with `[1,3,4]`).
8. **Forgetting the stale-entry check in Dijkstra**, or using it on negative weights.
9. **Testing only the happy example**, and only the chain, not the node with two prerequisites.
10. **Relying on `HashMap` order** in output.
11. **Silent overflow** in sums, products and midpoints.
12. **Fighting the borrow checker** instead of choosing a simpler ownership design.
13. **Unfinished capstones**: three half-built projects instead of one honest one.
14. **A README that overstates**: claims with no tests behind them.
15. **Memorizing solutions** without the invariant; the first follow-up question breaks it.

## 13. Honest limits

- **I did not interview anyone or get interviewed.** The method in §3, the stages in §2 and the advice in §7 to §9 are opinion from general experience and could be wrong for a particular company.
- **The problems are classics, and the code is a teaching model.** I did not check them against any online judge, and I did not read any company's question bank.
- **The tests show agreement with oracles on small random inputs and hand-chosen edges**, not correctness for all inputs. The mutation result (§17) shows these tests detect 85 specific edits; eight mutants are equivalent or performance-only, five more are detected only because the suite never finishes, and the table says why for each.
- **The benchmark numbers** are one machine, three runs, and the large-array search is dominated by cache behavior that I explained but did not measure separately.
- **Hiring trends, salaries, what a given employer values** are outside what I verified.

## 14. Exercises

Each solution is in the `exp` crate with tests (10 tests, 8 full runs). Answers are collapsed; try first.

**Exercise 1.** Return indices `(i, j)`, `i < j`, with `v[i] + v[j] == target`, the pair with the smallest `j`; `None` if none. Sums that would overflow never match.

<details markdown="1">
<summary>Answer</summary>

```rust
   //! Exercise 1: two numbers that add up to a target.
   use std::collections::HashMap;

   /// Indices `(i, j)` with `i < j` and `v[i] + v[j] == target`, the pair with the smallest `j`
   /// (then the smallest `i`); `None` if there is none. Sums that would overflow never match. O(n).
   pub fn two_sum(v: &[i64], target: i64) -> Option<(usize, usize)> {
       let mut seen: HashMap<i64, usize> = HashMap::new();
       for (j, &x) in v.iter().enumerate() {
           if let Some(need) = target.checked_sub(x)
               && let Some(&i) = seen.get(&need)
           {
               return Some((i, j));
           }
           seen.entry(x).or_insert(j); // keep the first index of a repeated value
       }
       None
   }
```

One pass with a map of values already seen. Storing only the *first* index of a repeated value is what makes `[3, 3]` with target 6 work and `[3]` with target 6 not (an element is not paired with itself: the lookup happens before the insert). `checked_sub` handles `target - x` overflowing. ✓ Compared with a double loop on 300 random inputs.
</details>

**Exercise 2.** Maximum sum of a non-empty contiguous run (Kadane); `None` for an empty slice or overflow.

<details markdown="1">
<summary>Answer</summary>

```rust
   //! Exercise 2: the best sum of a non-empty contiguous run (Kadane).

   /// The maximum sum over non-empty subarrays, or `None` for an empty slice. Overflow gives `None`.
   /// O(n), O(1) space.
   pub fn max_subarray(v: &[i64]) -> Option<i64> {
       let (&first, rest) = v.split_first()?;
       let (mut cur, mut best) = (first, first);
       for &x in rest {
           cur = if cur > 0 { cur.checked_add(x)? } else { x };
           best = best.max(cur);
       }
       Some(best)
   }
```

Invariant: `cur` is the best sum of a run ending at the current element: extend it if it is positive, otherwise start fresh. ✓ `[-3, -1, -2]` gives `-1`: the answer must be non-empty, so "take nothing" is not allowed.
</details>

**Exercise 3.** Group words that are anagrams of each other, groups ordered by their sorted-letters key.

<details markdown="1">
<summary>Answer</summary>

```rust
   //! Exercise 3: group words that are anagrams of each other.
   use std::collections::BTreeMap;

   /// Groups in order of their sorted-letters key; words inside a group keep their input order.
   pub fn group_anagrams(words: &[&str]) -> Vec<Vec<String>> {
       let mut groups: BTreeMap<Vec<u8>, Vec<String>> = BTreeMap::new();
       for w in words {
           let mut key = w.as_bytes().to_vec();
           key.sort_unstable();
           groups.entry(key).or_default().push((*w).to_string());
       }
       groups.into_values().collect()
   }
```

A `BTreeMap` keyed by the sorted bytes gives a deterministic order; a `HashMap` would give a different order on every run. ✓ `["eat","tea","tan","ate","nat","bat"]` gives `[bat] [eat tea ate] [tan nat]`.
</details>

**Exercise 4.** Count islands of `true` cells connected up, down, left and right; rows may have different lengths.

<details markdown="1">
<summary>Answer</summary>

```rust
   //! Exercise 4: count islands of `true` cells connected up, down, left and right.

   /// Number of connected groups of `true`. Rows may have different lengths (a ragged grid).
   pub fn islands(grid: &[Vec<bool>]) -> usize {
       let mut seen: Vec<Vec<bool>> = grid.iter().map(|r| vec![false; r.len()]).collect();
       let mut count = 0;
       for r in 0..grid.len() {
           for c in 0..grid[r].len() {
               if !grid[r][c] || seen[r][c] {
                   continue;
               }
               count += 1;
               let mut stack = vec![(r, c)];
               seen[r][c] = true;
               while let Some((y, x)) = stack.pop() {
                   let mut visit = |ny: usize, nx: usize| {
                       if ny < grid.len() && nx < grid[ny].len() && grid[ny][nx] && !seen[ny][nx] {
                           seen[ny][nx] = true;
                           stack.push((ny, nx));
                       }
                   };
                   visit(y + 1, x);
                   visit(y, x + 1);
                   if y > 0 {
                       visit(y - 1, x);
                   }
                   if x > 0 {
                       visit(y, x - 1);
                   }
               }
           }
       }
       count
   }
```

An explicit stack instead of recursion avoids a stack overflow on a large single island. `y + 1` and `x + 1` cannot underflow; the decrements are guarded. ✓ Diagonals do not connect (`"101","010","101"` has 5 islands).
</details>

**Exercise 5.** `gcd` and an overflow-safe `lcm`.

<details markdown="1">
<summary>Answer</summary>

```rust
   //! Exercise 5: greatest common divisor and least common multiple.

   /// Euclid's algorithm. `gcd(0, 0)` is 0.
   pub fn gcd(mut a: u64, mut b: u64) -> u64 {
       while b != 0 {
           (a, b) = (b, a % b);
       }
       a
   }

   /// The least common multiple, or `None` on overflow. `lcm(0, x)` is 0.
   pub fn lcm(a: u64, b: u64) -> Option<u64> {
       if a == 0 || b == 0 {
           return Some(0);
       }
       (a / gcd(a, b)).checked_mul(b) // divide first: a * b can overflow even when the lcm does not
   }
```

Dividing before multiplying matters: ✓ `lcm(u64::MAX, u64::MAX)` is `u64::MAX`, although `a * b` overflows.
</details>

**Exercise 6.** Count the ways to climb `n` stairs taking 1 or 2 steps; `None` if the count does not fit in a `u128`.

<details markdown="1">
<summary>Answer</summary>

```rust
   //! Exercise 6: ways to climb `n` stairs taking 1 or 2 steps at a time.

   /// The number of ways, or `None` if it does not fit in a `u128`. O(n), O(1) space.
   pub fn climb(n: u32) -> Option<u128> {
       let (mut a, mut b) = (1u128, 1u128); // ways(0), ways(1)
       for _ in 1..n {
           (a, b) = (b, a.checked_add(b)?);
       }
       Some(if n == 0 { a } else { b })
   }
```

It is the Fibonacci sequence shifted by one. ✓ `climb(50)` is 20 365 011 074; `climb(185)` fits and `climb(186)` does not.
</details>

**Exercise 7.** For each position, the product of all the other elements, without division; `None` on overflow.

<details markdown="1">
<summary>Answer</summary>

```rust
   //! Exercise 7: for each position, the product of all the other elements, without division.

   /// `out[i]` is the product of every element except `v[i]`; `None` on overflow. O(n).
   pub fn product_except_self(v: &[i64]) -> Option<Vec<i64>> {
       let n = v.len();
       let mut out = vec![1i64; n];
       let mut left = 1i64;
       for i in 0..n {
           out[i] = left;
           if i + 1 < n {
               left = left.checked_mul(v[i])?;
           }
       }
       let mut right = 1i64;
       for i in (0..n).rev() {
           out[i] = out[i].checked_mul(right)?;
           if i > 0 {
               right = right.checked_mul(v[i])?;
           }
       }
       Some(out)
   }
```

Two passes: prefix products, then suffix products. Division would fail with zeros (`[0, 2, 3]`). ✓ `[0, 0, 3]` gives all zeros; the last multiplication of the left pass is skipped so that an unused product cannot overflow.
</details>

**Exercise 8.** Rotate a square matrix 90° clockwise in place; refuse a non-square one without changing it.

<details markdown="1">
<summary>Answer</summary>

```rust
   //! Exercise 8: rotate a square matrix 90 degrees clockwise in place.

   /// Rotates `m` (n rows of n values) clockwise; returns false (and changes nothing) if `m` is not square.
   pub fn rotate_clockwise(m: &mut [Vec<i32>]) -> bool {
       let n = m.len();
       if m.iter().any(|r| r.len() != n) {
           return false;
       }
       // indices, not iterators: the swap touches `m[i][j]` and `m[j][i]` (two rows) at once
       #[allow(clippy::needless_range_loop)]
       for i in 0..n {
           for j in i + 1..n {
               let t = m[i][j];
               m[i][j] = m[j][i];
               m[j][i] = t;
           }
       }
       for row in m.iter_mut() {
           row.reverse();
       }
       true
   }
```

Transpose, then reverse each row. ✓ Four rotations restore the original. The clippy allow is deliberate and commented: the swap reads `m[i][j]` and `m[j][i]` (two rows) at once, which an iterator over rows cannot express without splitting the slice.
</details>

## 15. Debugging challenge

This function is in a library that searches over a very large index space. It is tested on ranges of a few hundred and works. In production, searches near the top of the range panic. What is wrong?

```rust
/// The faulty program.
pub mod buggy {
    /// The first value in `lo..hi` for which `pred` is true (`pred` is false, then true, never back);
    /// `hi` if there is none.
    pub fn first_true(mut lo: u32, mut hi: u32, pred: impl Fn(u32) -> bool) -> u32 {
        while lo < hi {
            let mid = (lo + hi) / 2;
            if pred(mid) {
                hi = mid;
            } else {
                lo = mid + 1;
            }
        }
        lo
    }
}
```

<details markdown="1">
<summary>Solution</summary>

`lo + hi` overflows `u32` when both are large (`4_294_967_285 + 4_294_967_295`). In a debug build that is a panic; in a release build it wraps and the search silently looks in the wrong place. ✓ The test calls the buggy version with `lo = u32::MAX - 10` and `hi = u32::MAX` and observes the panic; the fixed version finds the right answer for the same inputs, and for the full range `0..u32::MAX`.

```rust
/// The corrected program.
pub mod fixed {
    /// The first value in `lo..hi` for which `pred` is true (`pred` is false, then true, never back);
    /// `hi` if there is none.
    pub fn first_true(mut lo: u32, mut hi: u32, pred: impl Fn(u32) -> bool) -> u32 {
        while lo < hi {
            // the difference cannot overflow; the sum of two large indices can
            let mid = lo + (hi - lo) / 2;
            if pred(mid) {
                hi = mid;
            } else {
                lo = mid + 1;
            }
        }
        lo
    }
}
```

`hi - lo` cannot overflow because `lo < hi`, and `lo + (hi - lo) / 2` is always in range. It is the same fix as the `lower_bound` midpoint in §4.4, and a famous bug that sat in a widely used standard-library binary search for years (from memory).
</details>

## 16. Interview questions

Answers are collapsed.

1. **Walk me through your method for an unfamiliar problem.**
<details markdown="1"><summary>Answer</summary>

Clarify inputs and edge cases, work examples, state the brute force and its complexity, find the bottleneck, name the structure that removes it, say the plan, code in small functions, trace examples, then give complexity and trade-offs.
</details>

2. **Design an LRU cache.**
<details markdown="1"><summary>Answer</summary>

A hash map from key to a node in a doubly linked list, most recent at the head. `get` moves the node to the head; `put` inserts at the head and, when full, removes the tail. Both O(1). In Rust, store nodes in a `Vec` and link by index, reusing the evicted slot.
</details>

3. **Why is `(lo + hi) / 2` a bug and what replaces it?**
<details markdown="1"><summary>Answer</summary>

The sum can overflow. Use `lo + (hi - lo) / 2`, which stays in range because `lo <= hi`.
</details>

4. **How do you detect a cycle in a directed graph?**
<details markdown="1"><summary>Answer</summary>

Kahn's algorithm: if the topological order has fewer nodes than the graph, there is a cycle. Or a depth-first search with three colors (unvisited, in progress, done): an edge to an in-progress node is a cycle.
</details>

5. **When does Dijkstra fail?**
<details markdown="1"><summary>Answer</summary>

With negative edge weights, because a settled node's distance can later be improved. Use Bellman-Ford (O(n·e)) or, for DAGs, relax in topological order.
</details>

6. **Merge overlapping intervals and give the complexity.**
<details markdown="1"><summary>Answer</summary>

Sort by start, then sweep, extending the last interval when the next start is at most its end. O(n log n) for the sort, O(n) for the sweep.
</details>

7. **Greedy or dynamic programming for coin change?**
<details markdown="1"><summary>Answer</summary>

Greedy is correct only for special coin systems (such as US coins). For arbitrary coins, such as `[1, 3, 4]` and amount 6, greedy gives 3 coins and the optimum is 2. Use DP over amounts: O(amount · coins).
</details>

8. **How do you select the k largest of n items?**
<details markdown="1"><summary>Answer</summary>

A min-heap of size k: O(n log k) time, O(k) space. Sorting is O(n log n). Quickselect gives expected O(n) but mutates and has a worst case; mention it.
</details>

9. **Union-find: what do path compression and union by size give you?**
<details markdown="1"><summary>Answer</summary>

Nearly constant amortized time per operation (inverse Ackermann). Neither changes any answer, only speed, so tests of behavior cannot detect their absence.
</details>

10. **How would you test a function you just wrote, with no framework?**
<details markdown="1"><summary>Answer</summary>

Trace a normal and an edge example, name each loop's invariant, and propose a brute-force oracle compared on random small inputs, plus exhaustive small cases for boundary-heavy logic.
</details>

11. **Why use `BTreeMap` or sorting in output code?**
<details markdown="1"><summary>Answer</summary>

`HashMap` iteration order is unspecified and randomized per run; deterministic output is easier to test and to reproduce.
</details>

12. **Explain `Send` and `Sync` and give a type that lacks each.**
<details markdown="1"><summary>Answer</summary>

`Send`: ownership can move to another thread. `Sync`: `&T` can be shared between threads. `Rc<T>` is neither. `Cell<T>` and `RefCell<T>` are `Send` (if `T` is) but not `Sync`. `MutexGuard` is `Sync` (if `T` is) but not `Send`.
</details>

13. **A function returns `Option` in one place and panics in another. How do you decide?**
<details markdown="1"><summary>Answer</summary>

Absence the caller should expect is `Option`; failure the caller can handle differently by cause is `Result`; a violated invariant (a bug) is a panic. Document preconditions either way.
</details>

14. **Tell me about a project you built and what you would change.**
<details markdown="1"><summary>Answer</summary>

Use the shape: situation, decision and the alternatives you rejected, result with a number, and one thing you would do differently. Pick a project with a real failure and say exactly what you did yourself.
</details>

15. **What is your approach to a take-home or capstone project?**
<details markdown="1"><summary>Answer</summary>

Write acceptance criteria first, build a thin end-to-end slice, add tests as you go, cut features rather than tests, document limits honestly, and leave a README that lets a stranger run it in a few minutes.
</details>

## 17. Practical challenge: `drill`

The crate below is the whole lesson. Build it with `cargo test`; the benchmark is `cargo run --release --example bench`. Layout: `Cargo.toml`, `src/{lib,lru,intervals,graph,search,strings,dp,topk,trie}.rs`, `tests/{basic_tests,gap_tests}.rs`, `examples/bench.rs`.

### `Cargo.toml`

```toml
[package]
name = "drill"
version = "0.1.0"
edition = "2024"
publish = false

[dependencies]

[profile.dev.package."*"]
opt-level = 3
```

### `src/lib.rs`

```rust
//! Classic interview problems, written the way you would want to defend them: stated
//! preconditions, explicit edge cases, no hidden panics, and a complexity you can name.
pub mod dp;
pub mod graph;
pub mod intervals;
pub mod lru;
pub mod search;
pub mod strings;
pub mod topk;
pub mod trie;
```

### `src/lru.rs`

```rust
//! An LRU cache with O(1) `get` and `put`: a hash map into a doubly linked list stored in a `Vec`
//! (indices instead of pointers, so no `unsafe` and no `Rc<RefCell<..>>`).
use std::collections::HashMap;
use std::hash::Hash;

const NIL: usize = usize::MAX;

struct Node<K, V> {
    key: K,
    value: V,
    prev: usize,
    next: usize,
}

pub struct Lru<K, V> {
    cap: usize,
    map: HashMap<K, usize>,
    nodes: Vec<Node<K, V>>,
    head: usize, // most recently used
    tail: usize, // least recently used
}

impl<K: Hash + Eq + Clone, V> Lru<K, V> {
    /// A cache holding at most `cap` entries. A capacity of 0 stores nothing.
    pub fn new(cap: usize) -> Lru<K, V> {
        Lru {
            cap,
            map: HashMap::new(),
            nodes: Vec::new(),
            head: NIL,
            tail: NIL,
        }
    }

    pub fn len(&self) -> usize {
        self.map.len()
    }

    pub fn is_empty(&self) -> bool {
        self.map.is_empty()
    }

    fn unlink(&mut self, i: usize) {
        let (p, n) = (self.nodes[i].prev, self.nodes[i].next);
        if p != NIL {
            self.nodes[p].next = n;
        } else {
            self.head = n;
        }
        if n != NIL {
            self.nodes[n].prev = p;
        } else {
            self.tail = p;
        }
    }

    fn push_front(&mut self, i: usize) {
        self.nodes[i].prev = NIL;
        self.nodes[i].next = self.head;
        if self.head != NIL {
            self.nodes[self.head].prev = i;
        }
        self.head = i;
        if self.tail == NIL {
            self.tail = i;
        }
    }

    /// Reads a value and marks it most recently used.
    pub fn get(&mut self, key: &K) -> Option<&V> {
        let i = *self.map.get(key)?;
        self.unlink(i);
        self.push_front(i);
        Some(&self.nodes[i].value)
    }

    /// Reads without changing the order.
    pub fn peek(&self, key: &K) -> Option<&V> {
        self.map.get(key).map(|&i| &self.nodes[i].value)
    }

    /// Inserts or replaces. Returns the entry evicted to make room, if any.
    pub fn put(&mut self, key: K, value: V) -> Option<(K, V)> {
        if self.cap == 0 {
            return Some((key, value));
        }
        if let Some(&i) = self.map.get(&key) {
            self.nodes[i].value = value;
            self.unlink(i);
            self.push_front(i);
            return None;
        }
        if self.map.len() < self.cap {
            let i = self.nodes.len();
            self.nodes.push(Node {
                key: key.clone(),
                value,
                prev: NIL,
                next: NIL,
            });
            self.map.insert(key, i);
            self.push_front(i);
            return None;
        }
        // full: reuse the least recently used slot
        let i = self.tail;
        self.unlink(i);
        let old_key = std::mem::replace(&mut self.nodes[i].key, key.clone());
        let old_value = std::mem::replace(&mut self.nodes[i].value, value);
        self.map.remove(&old_key);
        self.map.insert(key, i);
        self.push_front(i);
        Some((old_key, old_value))
    }
}
```

### `src/intervals.rs`

```rust
//! Interval problems. An interval is `(start, end)` with `start <= end`, closed at both ends.

/// Merges overlapping or touching intervals; the result is sorted and disjoint. O(n log n).
pub fn merge(mut v: Vec<(i64, i64)>) -> Vec<(i64, i64)> {
    v.sort_unstable();
    let mut out: Vec<(i64, i64)> = Vec::with_capacity(v.len());
    for (s, e) in v {
        match out.last_mut() {
            Some(last) if s <= last.1 => last.1 = last.1.max(e),
            _ => out.push((s, e)),
        }
    }
    out
}

/// The most meetings that are in progress at the same moment. Meetings are half-open
/// `[start, end)`, so one ending at 10 and one starting at 10 do not overlap. O(n log n).
pub fn max_overlap(meetings: &[(i64, i64)]) -> usize {
    let mut events: Vec<(i64, i32)> = Vec::with_capacity(meetings.len() * 2);
    for &(s, e) in meetings {
        if s < e {
            events.push((s, 1));
            events.push((e, -1));
        }
    }
    // at equal times, ends (-1) sort before starts (+1)
    events.sort_unstable();
    let (mut cur, mut best) = (0i32, 0i32);
    for (_, d) in events {
        cur += d;
        best = best.max(cur);
    }
    best as usize
}
```

### `src/graph.rs`

```rust
//! Graph problems on nodes numbered `0..n`.
use std::cmp::Reverse;
use std::collections::{BTreeSet, BinaryHeap};

/// A topological order (smallest node first among the available ones), or `None` if there is a cycle.
/// `edges` are `(from, to)`; nodes outside `0..n` are ignored. O((n + e) log n).
pub fn topo_sort(n: usize, edges: &[(usize, usize)]) -> Option<Vec<usize>> {
    let mut indeg = vec![0usize; n];
    let mut adj = vec![Vec::new(); n];
    for &(a, b) in edges {
        if a < n && b < n {
            adj[a].push(b);
            indeg[b] += 1;
        }
    }
    let mut ready: BTreeSet<usize> = (0..n).filter(|&i| indeg[i] == 0).collect();
    let mut out = Vec::with_capacity(n);
    while let Some(&u) = ready.iter().next() {
        ready.remove(&u);
        out.push(u);
        for &v in &adj[u] {
            indeg[v] -= 1;
            if indeg[v] == 0 {
                ready.insert(v);
            }
        }
    }
    if out.len() == n { Some(out) } else { None }
}

/// Shortest distances from `src` over non-negative weights; `None` for unreachable nodes.
/// `edges` are `(from, to, weight)`. O((n + e) log n).
pub fn dijkstra(n: usize, edges: &[(usize, usize, u64)], src: usize) -> Vec<Option<u64>> {
    let mut adj = vec![Vec::new(); n];
    for &(a, b, w) in edges {
        if a < n && b < n {
            adj[a].push((b, w));
        }
    }
    let mut dist: Vec<Option<u64>> = vec![None; n];
    if src >= n {
        return dist;
    }
    let mut heap = BinaryHeap::new();
    dist[src] = Some(0);
    heap.push(Reverse((0u64, src)));
    while let Some(Reverse((d, u))) = heap.pop() {
        if dist[u].is_some_and(|best| d > best) {
            continue; // a stale heap entry
        }
        for &(v, w) in &adj[u] {
            let nd = d.saturating_add(w);
            if dist[v].is_none_or(|cur| nd < cur) {
                dist[v] = Some(nd);
                heap.push(Reverse((nd, v)));
            }
        }
    }
    dist
}

/// Disjoint sets with path compression and union by size.
pub struct UnionFind {
    parent: Vec<usize>,
    size: Vec<usize>,
    sets: usize,
}

impl UnionFind {
    pub fn new(n: usize) -> UnionFind {
        UnionFind {
            parent: (0..n).collect(),
            size: vec![1; n],
            sets: n,
        }
    }
    pub fn find(&mut self, x: usize) -> usize {
        let mut root = x;
        while self.parent[root] != root {
            root = self.parent[root];
        }
        let mut cur = x;
        while self.parent[cur] != root {
            let next = self.parent[cur];
            self.parent[cur] = root;
            cur = next;
        }
        root
    }
    /// Joins the sets of `a` and `b`; false if they were already one.
    pub fn union(&mut self, a: usize, b: usize) -> bool {
        let (mut ra, mut rb) = (self.find(a), self.find(b));
        if ra == rb {
            return false;
        }
        if self.size[ra] < self.size[rb] {
            std::mem::swap(&mut ra, &mut rb);
        }
        self.parent[rb] = ra;
        self.size[ra] += self.size[rb];
        self.sets -= 1;
        true
    }
    pub fn sets(&self) -> usize {
        self.sets
    }
    pub fn size_of(&mut self, x: usize) -> usize {
        let r = self.find(x);
        self.size[r]
    }
}
```

### `src/search.rs`

```rust
//! Binary search and its traps.

/// The first index `i` with `v[i] >= target` (`v.len()` if none); `v` must be sorted. O(log n).
pub fn lower_bound(v: &[i64], target: i64) -> usize {
    let (mut lo, mut hi) = (0usize, v.len());
    while lo < hi {
        let mid = lo + (hi - lo) / 2; // not (lo + hi) / 2: that can overflow
        if v[mid] < target {
            lo = mid + 1;
        } else {
            hi = mid;
        }
    }
    lo
}

/// Index of `target` in a sorted array of distinct values that was rotated at an unknown pivot.
/// O(log n).
pub fn search_rotated(v: &[i64], target: i64) -> Option<usize> {
    if v.is_empty() {
        return None;
    }
    let (mut lo, mut hi) = (0usize, v.len() - 1);
    while lo <= hi {
        let mid = lo + (hi - lo) / 2;
        if v[mid] == target {
            return Some(mid);
        }
        if v[lo] <= v[mid] {
            // the left half is sorted
            if v[lo] <= target && target < v[mid] {
                if mid == 0 {
                    return None;
                }
                hi = mid - 1;
            } else {
                lo = mid + 1;
            }
        } else if v[mid] < target && target <= v[hi] {
            lo = mid + 1;
        } else {
            if mid == 0 {
                return None;
            }
            hi = mid - 1;
        }
    }
    None
}

/// The integer square root: the largest `r` with `r * r <= n`. O(log n), no floating point.
pub fn isqrt(n: u64) -> u64 {
    let (mut lo, mut hi) = (0u64, n.min(u64::from(u32::MAX)));
    while lo < hi {
        let mid = lo + (hi - lo).div_ceil(2);
        if mid * mid <= n {
            lo = mid;
        } else {
            hi = mid - 1;
        }
    }
    lo
}
```

### `src/strings.rs`

```rust
//! String problems on bytes (ASCII); non-ASCII input is handled byte by byte.
use std::collections::HashMap;

/// Length of the longest substring with no repeated byte. O(n).
pub fn longest_unique(s: &str) -> usize {
    let mut last: HashMap<u8, usize> = HashMap::new();
    let (mut start, mut best) = (0usize, 0usize);
    for (i, b) in s.bytes().enumerate() {
        if let Some(&j) = last.get(&b)
            && j >= start
        {
            start = j + 1;
        }
        last.insert(b, i);
        best = best.max(i + 1 - start);
    }
    best
}

/// Are the brackets `()[]{}` balanced and properly nested? Other characters are ignored. O(n).
pub fn balanced(s: &str) -> bool {
    let mut stack = Vec::new();
    for c in s.chars() {
        match c {
            '(' | '[' | '{' => stack.push(c),
            ')' | ']' | '}' => {
                let want = match c {
                    ')' => '(',
                    ']' => '[',
                    _ => '{',
                };
                if stack.pop() != Some(want) {
                    return false;
                }
            }
            _ => {}
        }
    }
    stack.is_empty()
}

/// Are the two strings anagrams (same bytes, any order)? O(n).
pub fn anagrams(a: &str, b: &str) -> bool {
    if a.len() != b.len() {
        return false;
    }
    let mut counts = [0i32; 256];
    for x in a.bytes() {
        counts[x as usize] += 1;
    }
    for x in b.bytes() {
        counts[x as usize] -= 1;
    }
    counts.iter().all(|&c| c == 0)
}
```

### `src/dp.rs`

```rust
//! Dynamic programming.

/// Levenshtein distance (insert, delete, substitute) between two byte strings.
/// O(a·b) time, O(b) space.
pub fn edit_distance(a: &str, b: &str) -> usize {
    let (a, b) = (a.as_bytes(), b.as_bytes());
    let mut prev: Vec<usize> = (0..=b.len()).collect();
    for i in 1..=a.len() {
        let mut cur = vec![i; b.len() + 1];
        for j in 1..=b.len() {
            let sub = prev[j - 1] + usize::from(a[i - 1] != b[j - 1]);
            cur[j] = sub.min(prev[j] + 1).min(cur[j - 1] + 1);
        }
        prev = cur;
    }
    prev[b.len()]
}

/// The fewest coins that sum to `amount`, or `None` if impossible. O(amount · coins).
pub fn min_coins(coins: &[u64], amount: u64) -> Option<u64> {
    let amount = usize::try_from(amount).ok()?;
    let mut best: Vec<Option<u64>> = vec![None; amount + 1];
    best[0] = Some(0);
    for a in 1..=amount {
        for &c in coins {
            let Ok(c) = usize::try_from(c) else { continue };
            if c == 0 || c > a {
                continue;
            }
            if let Some(prev) = best[a - c]
                && best[a].is_none_or(|cur| prev + 1 < cur)
            {
                best[a] = Some(prev + 1);
            }
        }
    }
    best[amount]
}

/// Length of the longest strictly increasing subsequence. O(n log n).
pub fn lis(v: &[i64]) -> usize {
    let mut tails: Vec<i64> = Vec::new();
    for &x in v {
        let i = tails.partition_point(|&t| t < x);
        if i == tails.len() {
            tails.push(x);
        } else {
            tails[i] = x;
        }
    }
    tails.len()
}
```

### `src/topk.rs`

```rust
//! Selection with heaps.
use std::cmp::Reverse;
use std::collections::{BinaryHeap, HashMap};

/// The `k` most frequent items, most frequent first; ties broken by the smaller item.
/// O(n log k).
pub fn top_k_frequent(items: &[i64], k: usize) -> Vec<i64> {
    let mut counts: HashMap<i64, usize> = HashMap::new();
    for &x in items {
        *counts.entry(x).or_insert(0) += 1;
    }
    // a min-heap of the best k so far: the root is the worst of them
    let mut heap: BinaryHeap<(Reverse<usize>, i64)> = BinaryHeap::new();
    for (&x, &c) in &counts {
        heap.push((Reverse(c), x));
        if heap.len() > k {
            heap.pop();
        }
    }
    let mut out: Vec<(usize, i64)> = heap.into_iter().map(|(Reverse(c), x)| (c, x)).collect();
    out.sort_unstable_by(|a, b| b.0.cmp(&a.0).then(a.1.cmp(&b.1)));
    out.into_iter().map(|(_, x)| x).collect()
}

/// The `k`-th largest value (1-based), or `None` if there are fewer than `k` values. O(n log k).
pub fn kth_largest(v: &[i64], k: usize) -> Option<i64> {
    if k == 0 || k > v.len() {
        return None;
    }
    let mut heap = BinaryHeap::new();
    for &x in v {
        heap.push(Reverse(x));
        if heap.len() > k {
            heap.pop();
        }
    }
    heap.peek().map(|r| r.0)
}
```

### `src/trie.rs`

```rust
//! A trie over bytes.
use std::collections::BTreeMap;

#[derive(Default)]
struct Node {
    kids: BTreeMap<u8, Node>,
    end: bool,
    through: usize, // how many inserted words pass through this node
}

#[derive(Default)]
pub struct Trie {
    root: Node,
}

impl Trie {
    pub fn new() -> Trie {
        Trie::default()
    }

    /// Adds a word. Adding a word twice counts once.
    pub fn insert(&mut self, word: &str) {
        if self.contains(word) {
            return;
        }
        let mut n = &mut self.root;
        n.through += 1;
        for b in word.bytes() {
            n = n.kids.entry(b).or_default();
            n.through += 1;
        }
        n.end = true;
    }

    fn walk(&self, s: &str) -> Option<&Node> {
        let mut n = &self.root;
        for b in s.bytes() {
            n = n.kids.get(&b)?;
        }
        Some(n)
    }

    pub fn contains(&self, word: &str) -> bool {
        self.walk(word).is_some_and(|n| n.end)
    }

    /// How many distinct words start with `prefix` (the empty prefix counts all).
    pub fn count_prefix(&self, prefix: &str) -> usize {
        self.walk(prefix).map_or(0, |n| n.through)
    }

    /// All words starting with `prefix`, in byte order.
    pub fn words_with_prefix(&self, prefix: &str) -> Vec<String> {
        let mut out = Vec::new();
        if let Some(n) = self.walk(prefix) {
            let mut buf = prefix.as_bytes().to_vec();
            collect(n, &mut buf, &mut out);
        }
        out
    }
}

fn collect(n: &Node, buf: &mut Vec<u8>, out: &mut Vec<String>) {
    if n.end {
        out.push(String::from_utf8_lossy(buf).into_owned());
    }
    for (&b, kid) in &n.kids {
        buf.push(b);
        collect(kid, buf, out);
        buf.pop();
    }
}
```

### `tests/basic_tests.rs`

```rust
use drill::dp::*;
use drill::graph::*;
use drill::intervals::*;
use drill::lru::Lru;
use drill::search::*;
use drill::strings::*;
use drill::topk::*;
use drill::trie::Trie;

/// A tiny deterministic generator for brute-force comparisons (no dependencies).
struct Lcg(u64);
impl Lcg {
    fn next(&mut self, bound: u64) -> u64 {
        self.0 = self
            .0
            .wrapping_mul(6364136223846793005)
            .wrapping_add(1442695040888963407);
        (self.0 >> 33) % bound
    }
}

// ---- LRU ----

#[test]
fn lru_evicts_the_least_recently_used() {
    let mut c = Lru::new(2);
    assert_eq!(c.put("a", 1), None);
    assert_eq!(c.put("b", 2), None);
    assert_eq!(c.get(&"a"), Some(&1)); // a is now the freshest
    assert_eq!(c.put("c", 3), Some(("b", 2)));
    assert_eq!(c.peek(&"b"), None);
    assert_eq!(c.peek(&"a"), Some(&1));
    assert_eq!(c.peek(&"c"), Some(&3));
    assert_eq!(c.len(), 2);
}

#[test]
fn lru_replacing_a_value_refreshes_without_growing() {
    let mut c = Lru::new(2);
    c.put(1, "x");
    c.put(2, "y");
    assert_eq!(c.put(1, "z"), None);
    assert_eq!(c.len(), 2);
    assert_eq!(c.put(3, "w"), Some((2, "y"))); // 1 was refreshed by the replace
    assert_eq!(c.peek(&1), Some(&"z"));
}

#[test]
fn lru_peek_does_not_refresh() {
    let mut c = Lru::new(2);
    c.put(1, 1);
    c.put(2, 2);
    assert_eq!(c.peek(&1), Some(&1));
    assert_eq!(c.put(3, 3), Some((1, 1)));
}

#[test]
fn lru_capacity_zero_and_one() {
    let mut z = Lru::new(0);
    assert_eq!(z.put(1, 1), Some((1, 1)));
    assert!(z.is_empty());
    assert_eq!(z.get(&1), None);
    let mut o = Lru::new(1);
    o.put(1, 1);
    assert_eq!(o.put(2, 2), Some((1, 1)));
    assert_eq!(o.get(&2), Some(&2));
    assert_eq!(o.get(&1), None);
}

#[test]
fn lru_matches_a_naive_model_on_random_operations() {
    let mut rng = Lcg(7);
    for cap in 1..6usize {
        let mut c: Lru<u64, u64> = Lru::new(cap);
        let mut model: Vec<(u64, u64)> = Vec::new(); // most recent last
        for step in 0..2000u64 {
            let k = rng.next(8);
            if rng.next(2) == 0 {
                let evicted = c.put(k, step);
                let mut want = None;
                if let Some(p) = model.iter().position(|e| e.0 == k) {
                    model.remove(p);
                } else if model.len() == cap {
                    want = Some(model.remove(0));
                }
                model.push((k, step));
                assert_eq!(evicted, want, "cap {cap} step {step}");
            } else {
                let got = c.get(&k).copied();
                let want = model.iter().position(|e| e.0 == k).map(|p| {
                    let e = model.remove(p);
                    model.push(e);
                    e.1
                });
                assert_eq!(got, want, "cap {cap} step {step}");
            }
            assert_eq!(c.len(), model.len());
        }
    }
}

// ---- intervals ----

#[test]
fn merge_examples() {
    assert_eq!(
        merge(vec![(1, 3), (2, 6), (8, 10), (15, 18)]),
        vec![(1, 6), (8, 10), (15, 18)]
    );
    assert_eq!(merge(vec![(1, 4), (4, 5)]), vec![(1, 5)]); // touching merges
    assert_eq!(merge(vec![(5, 6), (1, 10)]), vec![(1, 10)]);
    assert_eq!(merge(vec![(1, 10), (2, 3)]), vec![(1, 10)]); // contained
    assert_eq!(merge(vec![]), vec![]);
    assert_eq!(merge(vec![(3, 3)]), vec![(3, 3)]);
    assert_eq!(merge(vec![(1, 2), (4, 5)]), vec![(1, 2), (4, 5)]);
    assert_eq!(
        merge(vec![(i64::MIN, 0), (0, i64::MAX)]),
        vec![(i64::MIN, i64::MAX)]
    );
}

#[test]
fn max_overlap_examples() {
    assert_eq!(max_overlap(&[(0, 30), (5, 10), (15, 20)]), 2);
    assert_eq!(max_overlap(&[(7, 10), (2, 4)]), 1);
    assert_eq!(max_overlap(&[(0, 10), (10, 20)]), 1); // back to back
    assert_eq!(max_overlap(&[]), 0);
    assert_eq!(max_overlap(&[(5, 5), (5, 5)]), 0); // empty meetings
    assert_eq!(max_overlap(&[(1, 5), (1, 5), (1, 5)]), 3);
    assert_eq!(max_overlap(&[(1, 5), (5, 9), (3, 7)]), 2);
}

#[test]
fn max_overlap_matches_brute_force() {
    let mut rng = Lcg(11);
    for _ in 0..300 {
        let n = rng.next(8) as usize;
        let m: Vec<(i64, i64)> = (0..n)
            .map(|_| {
                let s = rng.next(20) as i64;
                (s, s + rng.next(6) as i64)
            })
            .collect();
        let brute = (0..30i64)
            .map(|t| m.iter().filter(|&&(s, e)| s <= t && t < e).count())
            .max()
            .unwrap_or(0);
        assert_eq!(max_overlap(&m), brute, "{m:?}");
    }
}

// ---- graph ----

#[test]
fn topo_sort_orders_and_detects_cycles() {
    assert_eq!(
        topo_sort(4, &[(0, 1), (1, 2), (0, 3)]),
        Some(vec![0, 1, 2, 3])
    );
    assert_eq!(topo_sort(3, &[(2, 1), (1, 0)]), Some(vec![2, 1, 0]));
    assert_eq!(topo_sort(3, &[(0, 1), (1, 2), (2, 0)]), None);
    assert_eq!(topo_sort(1, &[(0, 0)]), None); // a self loop is a cycle
    assert_eq!(topo_sort(0, &[]), Some(vec![]));
    assert_eq!(topo_sort(3, &[]), Some(vec![0, 1, 2]));
    assert_eq!(topo_sort(2, &[(0, 7), (9, 1)]), Some(vec![0, 1])); // out-of-range edges are ignored
    assert_eq!(topo_sort(4, &[(1, 0), (3, 2)]), Some(vec![1, 0, 3, 2])); // smallest available first
}

#[test]
fn topo_sort_result_respects_every_edge() {
    let mut rng = Lcg(3);
    for _ in 0..200 {
        let n = 2 + rng.next(8) as usize;
        let edges: Vec<(usize, usize)> = (0..rng.next(12))
            .filter_map(|_| {
                let (a, b) = (rng.next(n as u64) as usize, rng.next(n as u64) as usize);
                (a < b).then_some((a, b)) // only forward edges: always acyclic
            })
            .collect();
        let order = topo_sort(n, &edges).expect("acyclic");
        let pos: Vec<usize> = (0..n)
            .map(|v| order.iter().position(|&x| x == v).unwrap())
            .collect();
        assert!(edges.iter().all(|&(a, b)| pos[a] < pos[b]));
        assert_eq!(order.len(), n);
    }
}

#[test]
fn dijkstra_known_answer() {
    // py/ref.py: [0, 3, 1, 4, None]
    let e = [(0, 1, 4), (0, 2, 1), (2, 1, 2), (1, 3, 1), (2, 3, 5)];
    assert_eq!(
        dijkstra(5, &e, 0),
        vec![Some(0), Some(3), Some(1), Some(4), None]
    );
}

#[test]
fn dijkstra_edge_cases() {
    assert_eq!(dijkstra(3, &[], 0), vec![Some(0), None, None]);
    assert_eq!(dijkstra(3, &[], 9), vec![None, None, None]);
    assert_eq!(dijkstra(2, &[(0, 1, 0)], 0), vec![Some(0), Some(0)]); // zero-weight edge
    assert_eq!(dijkstra(2, &[(1, 0, 5)], 0), vec![Some(0), None]); // directed
    assert_eq!(
        dijkstra(3, &[(0, 1, u64::MAX), (1, 2, 5)], 0)[2],
        Some(u64::MAX)
    ); // saturates
    assert_eq!(
        dijkstra(2, &[(0, 1, 9), (0, 1, 3), (0, 1, 7)], 0)[1],
        Some(3)
    ); // parallel edges
}

#[test]
fn dijkstra_matches_bellman_ford() {
    let mut rng = Lcg(5);
    for _ in 0..200 {
        let n = 1 + rng.next(7) as usize;
        let e: Vec<(usize, usize, u64)> = (0..rng.next(15))
            .map(|_| {
                (
                    rng.next(n as u64) as usize,
                    rng.next(n as u64) as usize,
                    rng.next(10),
                )
            })
            .collect();
        let mut d: Vec<Option<u64>> = vec![None; n];
        d[0] = Some(0);
        for _ in 0..n {
            for &(a, b, w) in &e {
                if let Some(x) = d[a]
                    && d[b].is_none_or(|c| x + w < c)
                {
                    d[b] = Some(x + w);
                }
            }
        }
        assert_eq!(dijkstra(n, &e, 0), d);
    }
}

#[test]
fn union_find_basics() {
    let mut u = UnionFind::new(6);
    assert_eq!(u.sets(), 6);
    assert!(u.union(0, 1));
    assert!(u.union(2, 3));
    assert!(!u.union(1, 0));
    assert_eq!(u.sets(), 4);
    assert!(u.union(1, 3));
    assert_eq!(u.sets(), 3);
    assert_eq!(u.find(0), u.find(2));
    assert_ne!(u.find(0), u.find(4));
    assert_eq!(u.size_of(3), 4);
    assert_eq!(u.size_of(5), 1);
}

#[test]
fn union_find_matches_a_label_model() {
    let mut rng = Lcg(9);
    let n = 12usize;
    let mut u = UnionFind::new(n);
    let mut label: Vec<usize> = (0..n).collect();
    for _ in 0..200 {
        let (a, b) = (rng.next(n as u64) as usize, rng.next(n as u64) as usize);
        let merged = label[a] != label[b];
        assert_eq!(u.union(a, b), merged);
        if merged {
            let (from, to) = (label[b], label[a]);
            for l in label.iter_mut() {
                if *l == from {
                    *l = to;
                }
            }
        }
        let distinct: std::collections::HashSet<_> = label.iter().collect();
        assert_eq!(u.sets(), distinct.len());
        for x in 0..n {
            assert_eq!(
                u.size_of(x),
                label.iter().filter(|&&l| l == label[x]).count()
            );
        }
    }
}

// ---- search ----

#[test]
fn lower_bound_cases() {
    let v = [1, 2, 2, 2, 5, 9];
    assert_eq!(lower_bound(&v, 2), 1);
    assert_eq!(lower_bound(&v, 3), 4);
    assert_eq!(lower_bound(&v, 0), 0);
    assert_eq!(lower_bound(&v, 10), 6);
    assert_eq!(lower_bound(&v, 9), 5);
    assert_eq!(lower_bound(&[], 1), 0);
    assert_eq!(lower_bound(&[i64::MAX], i64::MAX), 0);
}

#[test]
fn lower_bound_matches_partition_point() {
    let mut rng = Lcg(13);
    for _ in 0..300 {
        let mut v: Vec<i64> = (0..rng.next(12)).map(|_| rng.next(10) as i64).collect();
        v.sort_unstable();
        for t in -1..12 {
            assert_eq!(lower_bound(&v, t), v.partition_point(|&x| x < t));
        }
    }
}

#[test]
fn rotated_search_every_rotation() {
    for n in 0..12usize {
        let base: Vec<i64> = (0..n as i64).map(|x| x * 2).collect();
        for r in 0..n.max(1) {
            let mut v = base.clone();
            v.rotate_left(r);
            for (i, &x) in v.iter().enumerate() {
                assert_eq!(search_rotated(&v, x), Some(i), "{v:?} {x}");
            }
            for miss in [-1i64, 1, 3, 100] {
                assert_eq!(search_rotated(&v, miss), None, "{v:?} {miss}");
            }
        }
    }
}

#[test]
fn isqrt_known_answers() {
    // py/ref.py
    let want = [
        (0u64, 0u64),
        (1, 1),
        (2, 1),
        (3, 1),
        (4, 2),
        (15, 3),
        (16, 4),
        (17, 4),
        (1_000_000_000_000, 1_000_000),
        (u64::MAX, 4_294_967_295),
    ];
    for (n, r) in want {
        assert_eq!(isqrt(n), r, "{n}");
    }
}

#[test]
fn isqrt_invariant_around_squares() {
    for r in [1u64, 2, 3, 1000, 65_535, 65_536, 4_294_967_295] {
        let sq = r * r;
        assert_eq!(isqrt(sq), r);
        assert_eq!(isqrt(sq - 1), r - 1);
        if sq < u64::MAX - 2 * r {
            assert_eq!(isqrt(sq + 2 * r), r);
        }
    }
}

// ---- strings ----

#[test]
fn longest_unique_cases() {
    assert_eq!(longest_unique("abcabcbb"), 3);
    assert_eq!(longest_unique("bbbbb"), 1);
    assert_eq!(longest_unique("pwwkew"), 3);
    assert_eq!(longest_unique(""), 0);
    assert_eq!(longest_unique("abba"), 2); // the stale index of 'a' must not move the window back
    assert_eq!(longest_unique("tmmzuxt"), 5);
    assert_eq!(longest_unique("abcdef"), 6);
    assert_eq!(longest_unique("dvdf"), 3);
}

#[test]
fn longest_unique_matches_brute_force() {
    let mut rng = Lcg(17);
    for _ in 0..300 {
        let s: String = (0..rng.next(12))
            .map(|_| (b'a' + rng.next(4) as u8) as char)
            .collect();
        let b = s.as_bytes();
        let mut best = 0;
        for i in 0..b.len() {
            for j in i..b.len() {
                let w = &b[i..=j];
                let mut seen = std::collections::HashSet::new();
                if w.iter().all(|x| seen.insert(*x)) {
                    best = best.max(w.len());
                }
            }
        }
        assert_eq!(longest_unique(&s), best, "{s}");
    }
}

#[test]
fn balanced_cases() {
    for ok in ["", "()", "()[]{}", "{[()]}", "a(b)c", "((()))"] {
        assert!(balanced(ok), "{ok}");
    }
    for bad in ["(", ")", "(]", "([)]", "{[}", "())", "((", "]["] {
        assert!(!balanced(bad), "{bad}");
    }
}

#[test]
fn anagram_cases() {
    assert!(anagrams("listen", "silent"));
    assert!(anagrams("", ""));
    assert!(!anagrams("rat", "car"));
    assert!(!anagrams("aab", "abb")); // same length, same set, different counts
    assert!(!anagrams("a", "ab"));
    assert!(anagrams("héllo", "olléh"));
}

// ---- dp ----

#[test]
fn edit_distance_known_answers() {
    // py/ref.py: 3, 5, 3
    assert_eq!(edit_distance("kitten", "sitting"), 3);
    assert_eq!(edit_distance("intention", "execution"), 5);
    assert_eq!(edit_distance("", "abc"), 3);
    assert_eq!(edit_distance("abc", ""), 3);
    assert_eq!(edit_distance("", ""), 0);
    assert_eq!(edit_distance("same", "same"), 0);
    assert_eq!(edit_distance("a", "b"), 1);
    assert_eq!(edit_distance("ab", "ba"), 2);
}

#[test]
fn edit_distance_is_a_metric_on_random_strings() {
    let mut rng = Lcg(19);
    let gen_s = |rng: &mut Lcg| -> String {
        (0..rng.next(7))
            .map(|_| (b'a' + rng.next(3) as u8) as char)
            .collect()
    };
    for _ in 0..200 {
        let (a, b, c) = (gen_s(&mut rng), gen_s(&mut rng), gen_s(&mut rng));
        assert_eq!(edit_distance(&a, &b), edit_distance(&b, &a));
        assert!(edit_distance(&a, &c) <= edit_distance(&a, &b) + edit_distance(&b, &c));
        assert!(edit_distance(&a, &b) >= a.len().abs_diff(b.len()));
        assert!(edit_distance(&a, &b) <= a.len().max(b.len()));
    }
}

#[test]
fn min_coins_known_answers() {
    // py/ref.py: 3, None, 20, 6
    assert_eq!(min_coins(&[1, 2, 5], 11), Some(3));
    assert_eq!(min_coins(&[2], 3), None);
    assert_eq!(min_coins(&[186, 419, 83, 408], 6249), Some(20));
    assert_eq!(min_coins(&[1, 5, 10, 25], 63), Some(6));
    assert_eq!(min_coins(&[5], 0), Some(0));
    assert_eq!(min_coins(&[], 5), None);
    assert_eq!(min_coins(&[0, 3], 6), Some(2)); // a zero coin is ignored, not an infinite loop
    assert_eq!(min_coins(&[3, 4], 6), Some(2));
    assert_eq!(min_coins(&[1, 3, 4], 6), Some(2)); // greedy would use 3 coins (4+1+1)
}

#[test]
fn lis_known_answers() {
    // py/ref.py: 4, 4, 1
    assert_eq!(lis(&[10, 9, 2, 5, 3, 7, 101, 18]), 4);
    assert_eq!(lis(&[0, 1, 0, 3, 2, 3]), 4);
    assert_eq!(lis(&[7, 7, 7]), 1); // strictly increasing
    assert_eq!(lis(&[]), 0);
    assert_eq!(lis(&[1, 2, 3, 4]), 4);
    assert_eq!(lis(&[4, 3, 2, 1]), 1);
}

#[test]
fn lis_matches_quadratic_dp() {
    let mut rng = Lcg(23);
    for _ in 0..300 {
        let v: Vec<i64> = (0..rng.next(12)).map(|_| rng.next(8) as i64).collect();
        let mut best = vec![1usize; v.len()];
        for i in 0..v.len() {
            for j in 0..i {
                if v[j] < v[i] {
                    best[i] = best[i].max(best[j] + 1);
                }
            }
        }
        assert_eq!(lis(&v), best.into_iter().max().unwrap_or(0), "{v:?}");
    }
}

// ---- topk ----

#[test]
fn top_k_frequent_cases() {
    assert_eq!(top_k_frequent(&[1, 1, 1, 2, 2, 3], 2), vec![1, 2]);
    assert_eq!(top_k_frequent(&[5, 5, 7, 7, 9], 2), vec![5, 7]); // tie broken by the smaller item
    assert_eq!(top_k_frequent(&[4], 1), vec![4]);
    assert_eq!(top_k_frequent(&[1, 2, 3], 5), vec![1, 2, 3]);
    assert_eq!(top_k_frequent(&[1, 2], 0), Vec::<i64>::new());
    assert_eq!(top_k_frequent(&[], 3), Vec::<i64>::new());
    assert_eq!(top_k_frequent(&[3, 3, 2, 2, 1, 1], 2), vec![1, 2]);
}

#[test]
fn top_k_frequent_matches_sorting() {
    let mut rng = Lcg(29);
    for _ in 0..300 {
        let v: Vec<i64> = (0..rng.next(20)).map(|_| rng.next(6) as i64).collect();
        let k = rng.next(5) as usize;
        let mut counts = std::collections::BTreeMap::new();
        for &x in &v {
            *counts.entry(x).or_insert(0usize) += 1;
        }
        let mut all: Vec<(usize, i64)> = counts.into_iter().map(|(x, c)| (c, x)).collect();
        all.sort_by(|a, b| b.0.cmp(&a.0).then(a.1.cmp(&b.1)));
        let want: Vec<i64> = all.into_iter().take(k).map(|(_, x)| x).collect();
        assert_eq!(top_k_frequent(&v, k), want, "{v:?} k={k}");
    }
}

#[test]
fn kth_largest_cases() {
    assert_eq!(kth_largest(&[3, 2, 1, 5, 6, 4], 2), Some(5));
    assert_eq!(kth_largest(&[3, 2, 3, 1, 2, 4, 5, 5, 6], 4), Some(4));
    assert_eq!(kth_largest(&[1], 1), Some(1));
    assert_eq!(kth_largest(&[1, 2], 3), None);
    assert_eq!(kth_largest(&[1, 2], 0), None);
    assert_eq!(kth_largest(&[5, 5, 5], 3), Some(5));
    assert_eq!(kth_largest(&[1, 2, 3], 3), Some(1));
}

// ---- trie ----

#[test]
fn trie_basics() {
    let mut t = Trie::new();
    for w in ["apple", "app", "apply", "banana", "app"] {
        t.insert(w);
    }
    assert!(t.contains("app") && t.contains("apple") && !t.contains("appl"));
    assert!(!t.contains("") && !t.contains("applesauce"));
    assert_eq!(t.count_prefix("app"), 3); // the duplicate "app" counted once
    assert_eq!(t.count_prefix("appl"), 2);
    assert_eq!(t.count_prefix(""), 4);
    assert_eq!(t.count_prefix("x"), 0);
    assert_eq!(t.words_with_prefix("app"), vec!["app", "apple", "apply"]);
    assert_eq!(t.words_with_prefix("b"), vec!["banana"]);
    assert!(t.words_with_prefix("z").is_empty());
}

#[test]
fn trie_empty_word() {
    let mut t = Trie::new();
    t.insert("");
    assert!(t.contains(""));
    assert_eq!(t.count_prefix(""), 1);
    assert_eq!(t.words_with_prefix(""), vec![String::new()]);
    t.insert("");
    assert_eq!(t.count_prefix(""), 1);
}

#[test]
fn trie_matches_a_sorted_set() {
    let mut rng = Lcg(31);
    let mut t = Trie::new();
    let mut set = std::collections::BTreeSet::new();
    for _ in 0..200 {
        let w: String = (0..rng.next(5))
            .map(|_| (b'a' + rng.next(3) as u8) as char)
            .collect();
        t.insert(&w);
        set.insert(w);
        let p: String = (0..rng.next(3))
            .map(|_| (b'a' + rng.next(3) as u8) as char)
            .collect();
        let want: Vec<String> = set.iter().filter(|s| s.starts_with(&p)).cloned().collect();
        assert_eq!(t.words_with_prefix(&p), want);
        assert_eq!(t.count_prefix(&p), want.len());
    }
}
```

### `tests/gap_tests.rs`

```rust
//! Tests added after the first mutation run showed what the ordinary tests did not pin down.
use drill::graph::{dijkstra, topo_sort};

#[test]
fn a_node_with_two_prerequisites_waits_for_both() {
    // 2 needs 0 and 3. When 0 is done, 2 still has one prerequisite left and must not be offered yet.
    assert_eq!(topo_sort(4, &[(0, 2), (3, 2)]), Some(vec![0, 1, 3, 2]));
}

#[test]
fn a_source_equal_to_the_node_count_is_out_of_range() {
    assert_eq!(dijkstra(3, &[], 3), vec![None, None, None]);
}

#[test]
fn an_edge_to_a_missing_node_is_ignored() {
    assert_eq!(
        dijkstra(2, &[(0, 5, 1), (0, 1, 2)], 0),
        vec![Some(0), Some(2)]
    );
    assert_eq!(dijkstra(2, &[(5, 1, 1)], 0), vec![Some(0), None]);
}
```

### `examples/bench.rs`

```rust
use drill::dp::edit_distance;
use drill::graph::dijkstra;
use drill::lru::Lru;
use drill::search::lower_bound;
use drill::trie::Trie;
use std::hint::black_box;
use std::time::Instant;

fn time<F: FnMut()>(name: &str, n: u64, mut f: F) {
    let t = Instant::now();
    f();
    println!(
        "{name:<34} {:>10.1} ns/op",
        t.elapsed().as_nanos() as f64 / n as f64
    );
}

fn main() {
    let n = 1_000_000u64;
    let mut c: Lru<u64, u64> = Lru::new(1000);
    time("lru put+get (cap 1000)", n * 2, || {
        for i in 0..n {
            c.put(i % 1500, i);
            black_box(c.get(&((i * 7) % 1500)));
        }
    });
    let v: Vec<i64> = (0..1_000_000).collect();
    time("lower_bound (1M elements)", n, || {
        for i in 0..n {
            black_box(lower_bound(&v, (i * 7919 % 1_000_000) as i64));
        }
    });
    let a = "intention".repeat(30);
    let b = "execution".repeat(30);
    time("edit_distance (270x270)", 1, || {
        black_box(edit_distance(&a, &b));
    });
    let nodes = 10_000usize;
    let edges: Vec<(usize, usize, u64)> = (0..nodes * 5)
        .map(|i| (i % nodes, (i * 31 + 7) % nodes, (i % 13) as u64 + 1))
        .collect();
    time("dijkstra (10k nodes, 50k edges)", 1, || {
        black_box(dijkstra(nodes, &edges, 0));
    });
    let mut t = Trie::new();
    let words: Vec<String> = (0..50_000).map(|i| format!("word{i}")).collect();
    time("trie insert (50k words)", 50_000, || {
        for w in &words {
            t.insert(w);
        }
    });
    time("trie contains", 50_000, || {
        for w in &words {
            black_box(t.contains(w));
        }
    });
}
```

### `py/ref.py` (independent known answers)

```python
# independent reference answers for known inputs
import heapq, itertools
def edit(a,b):
    d=list(range(len(b)+1))
    for i,x in enumerate(a,1):
        n=[i]
        for j,y in enumerate(b,1):
            n.append(min(d[j-1]+(x!=y),d[j]+1,n[j-1]+1))
        d=n
    return d[-1]
print("edit kitten sitting",edit("kitten","sitting"),"intention execution",edit("intention","execution"),"'' abc",edit("","abc"))
def lis(v):
    best=[1]*len(v)
    for i in range(len(v)):
        for j in range(i):
            if v[j]<v[i]: best[i]=max(best[i],best[j]+1)
    return max(best,default=0)
print("lis",lis([10,9,2,5,3,7,101,18]),lis([0,1,0,3,2,3]),lis([7,7,7]))
def coins(cs,a):
    INF=10**9;d=[0]+[INF]*a
    for x in range(1,a+1):
        for c in cs:
            if 0<c<=x: d[x]=min(d[x],d[x-c]+1)
    return d[a] if d[a]<INF else None
print("coins",coins([1,2,5],11),coins([2],3),coins([186,419,83,408],6249),coins([1,5,10,25],63))
def dij(n,E,s):
    d=[None]*n;d[s]=0;h=[(0,s)]
    while h:
        x,u=heapq.heappop(h)
        if d[u]<x:continue
        for a,b,w in E:
            if a==u and (d[b] is None or x+w<d[b]): d[b]=x+w;heapq.heappush(h,(x+w,b))
    return d
print("dij",dij(5,[(0,1,4),(0,2,1),(2,1,2),(1,3,1),(2,3,5)],0))
import math
print("isqrt",[math.isqrt(n) for n in [0,1,2,3,4,15,16,17,10**12,2**64-1]])
```

### Checks and mutation testing

**Mutation check ✓.** I made 85 small source edits (a "mutant") across eight source modules, each in its own copy of the crate, and ran the whole test suite against each. **72 were caught by a failing test, 5 are detected by a hang, and 8 are equivalent (M17, M32, M36, M55, M63, M65, M73, M76); none fails to compile and none survives unexplained.** **What the passes found.** I ran the mutants against the 35 tests of `basic_tests.rs`, in several batches. Two mistakes of mine in the process: I started a second batch while the first was still running, and a hanging mutant makes the harness kill every test process on the machine, so some results were contaminated (a mutant reported as caught by a test that had nothing to do with it); I noticed the nonsensical test names, re-ran every mutant whose result I distrusted, **alone**, and the table is built from those clean runs. In the clean results **three were real gaps**, each closed with one test in `gap_tests.rs`: a node with two prerequisites released after only one (`topo_sort`: the random graphs only had edges from smaller to larger numbers, which hid it), a Dijkstra source equal to the node count (I had tested one far out of range, not the first one just outside), and an edge to a missing node in Dijkstra (I had tested it only for `topo_sort`). Eight mutants are equivalent or change only speed, each with the reason below; 5 are detected only because the suite never finishes, which is a valid detection but not a pleasant one (a test timeout in CI would turn it into a clean failure). The pattern in the real gaps is that I tested a property in one function and forgot the same property in its sibling. Reading the survivors as "which twin did I forget?" is faster than reading them as "which line is untested". The column "change" is generated from the edit itself (the part that differs, with a little context, whitespace collapsed); `…` marks omitted text. The tests ran in the dev profile with optimized dependencies.

| # | module | change | result |
|---|---|---|---|
| M1 | `lru` | `…self.cap == 0` → `…self.cap == 1` | caught by `lru_capacity_zero_and_one` |
| M2 | `lru` | `…self.map.len() <` → `…self.map.len() <=` | caught by `lru_capacity_zero_and_one` |
| M3 | `lru` | `let i = self.tail` → `let i = self.head` | caught by `lru_evicts_the_least_recently_used` |
| M4 | `lru` | `self.nodes[i].value = value; self.unlink(i); self.push_front(i);` → `self.nodes[i].value = value;` | caught by `lru_matches_a_naive_model_on_random_operations` |
| M5 | `lru` | `…*self.map.get(key)?; self.unlink(i); self.push_front(i);` → `…*self.map.get(key)?;` | caught by `lru_evicts_the_least_recently_used` |
| M6 | `lru` | `self.map.remove(&old_key);` → `∅` | caught by `lru_capacity_zero_and_one` |
| M7 | `lru` | `…self.nodes[p].next = n; } else { self.head = n; }` → `…self.nodes[p].next = n; }` | caught by `lru_matches_a_naive_model_on_random_operations` |
| M8 | `lru` | `…self.nodes[n].prev = p; } else { self.tail = p; }` → `…self.nodes[n].prev = p; }` | caught by `lru_evicts_the_least_recently_used` |
| M9 | `lru` | `if self.tail == NIL { self.tail = i; }` → `∅` | caught by `lru_capacity_zero_and_one` |
| M10 | `lru` | `self.nodes[self.head].prev = i;` → `∅` | caught by `lru_evicts_the_least_recently_used` |
| M11 | `lru` | `return Some((key, value))` → `return None` | caught by `lru_capacity_zero_and_one` |
| M12 | `lru` | `…&self.nodes[i` → `…&self.nodes[0` | caught by `lru_evicts_the_least_recently_used` |
| M13 | `lru` | `self.nodes[i].prev = NIL; self.nodes[i].` → `self.nodes[i].` | caught by `lru_matches_a_naive_model_on_random_operations` |
| M14 | `intervals` | `…last) if s <=` → `…last) if s <` | caught by `merge_examples` |
| M15 | `intervals` | `last.1 = last.1.max(e)` → `last.1 = e` | caught by `merge_examples` |
| M16 | `intervals` | `v.sort_unstable(); ` → `∅` | caught by `merge_examples` |
| M17 | `intervals` | `if s <` → `if s <=` | **equivalent**: an empty meeting `(5,5)` adds a `-1` and then a `+1` at the same time (ends sort first), so the running count dips and returns and never exceeds the true maximum |
| M18 | `intervals` | `events.sort_unstable(` → `events.sort_unstable_by(\|a, b\| b.cmp(a)` | caught by `max_overlap_examples` |
| M19 | `intervals` | `best = best.max(cur)` → `best = cur` | caught by `max_overlap_examples` |
| M20 | `intervals` | `events.push((e, -` → `events.push((e, ` | caught by `max_overlap_matches_brute_force` |
| M21 | `graph` | `if a < n &&` → `if a < n \|\|` | caught by `topo_sort_orders_and_detects_cycles` |
| M22 | `graph` | `…BTreeSet<usize> = (0..n).filter(\|&i\| indeg[i] == 0).collect` → `…BTreeSet<usize> = BTreeSet::new` | caught by `topo_sort_orders_and_detects_cycles` |
| M23 | `graph` | `if indeg[v] == 0` → `if indeg[v] <= 1` | caught by `a_node_with_two_prerequisites_waits_for_both` |
| M24 | `graph` | `if out.len() == n { Some(out) } else { None }` → `Some(out)` | caught by `topo_sort_orders_and_detects_cycles` |
| M25 | `graph` | `…ready.iter().next` → `…ready.iter().next_back` | caught by `topo_sort_orders_and_detects_cycles` |
| M26 | `graph` | `…dist[u].is_some_and(\|best\| d >` → `…dist[u].is_some_and(\|best\| d >=` | caught by `dijkstra_known_answer` |
| M27 | `graph` | `let nd = d.saturat` → `let nd = d.wrapp` | caught by `dijkstra_edge_cases` |
| M28 | `graph` | `dist[v].is_none_or(\|cur\| nd <` → `dist[v].is_none_or(\|cur\| nd <=` | detected: the test suite never finishes (an infinite loop), killed after the time limit |
| M29 | `graph` | `…Reverse((nd, v` → `…Reverse((nd, u` | caught by `dijkstra_edge_cases` |
| M30 | `graph` | `if src >=` → `if src >` | caught by `a_source_equal_to_the_node_count_is_out_of_range` |
| M31 | `graph` | `dist[src] = Some(0` → `dist[src] = Some(1` | caught by `dijkstra_edge_cases` |
| M32 | `graph` | `…self.size[ra] <` → `…self.size[ra] >` | **equivalent in behavior**: union by size only keeps the trees shallow; set counts, sizes and membership are the same either way, so only a benchmark could see it |
| M33 | `graph` | `self.size[ra] += self.size[rb]` → `self.size[ra] += 1` | caught by `union_find_basics` |
| M34 | `graph` | `self.sets -= 1` → `self.sets -= 2` | caught by `union_find_basics` |
| M35 | `graph` | `if ra == rb { return false; }` → `∅` | caught by `union_find_basics` |
| M36 | `graph` | `self.parent[cur] = root; ` → `∅` | **equivalent in behavior**: path compression only shortens later walks; the roots found are the same, so only a benchmark could see it |
| M37 | `graph` | `if a < n &&` → `if a < n \|\|` | caught by `an_edge_to_a_missing_node_is_ignored` |
| M38 | `search` | `if v[mid] <` → `if v[mid] <=` | caught by `lower_bound_cases` |
| M39 | `search` | `…)); while lo <` → `…)); while lo <=` | detected: the test suite never finishes (an infinite loop), killed after the time limit |
| M40 | `search` | `let mid = lo + (hi - lo) / 2` → `let mid = (lo + hi) / 2 + 1` | detected: the test suite never finishes (an infinite loop), killed after the time limit |
| M41 | `search` | `if v[lo] <=` → `if v[lo] <` | caught by `rotated_search_every_rotation` |
| M42 | `search` | `if v[lo] <=` → `if v[lo] <` | caught by `rotated_search_every_rotation` |
| M43 | `search` | `…target && target <=` → `…target && target <` | caught by `rotated_search_every_rotation` |
| M44 | `search` | `if v.is_empty() { return None; }` → `∅` | caught by `rotated_search_every_rotation` |
| M45 | `search` | `…lo + (hi - lo).div_ceil(2)` → `…lo + (hi - lo) / 2` | detected: the test suite never finishes (an infinite loop), killed after the time limit |
| M46 | `search` | `if mid * mid <=` → `if mid * mid <` | caught by `isqrt_invariant_around_squares` |
| M47 | `search` | `n.min(u64::from(u32::MAX))` → `n` | caught by `isqrt_invariant_around_squares` |
| M48 | `search` | `lo = mid + 1` → `lo = mid` | detected: the test suite never finishes (an infinite loop), killed after the time limit |
| M49 | `strings` | `&& j >=` → `&& j >` | caught by `longest_unique_cases` |
| M50 | `strings` | `start = j + 1` → `start = j` | caught by `longest_unique_cases` |
| M51 | `strings` | `best = best.max(i + 1 - start)` → `best = i + 1 - start` | caught by `longest_unique_cases` |
| M52 | `strings` | `…stack.pop() !` → `…stack.pop() =` | caught by `balanced_cases` |
| M53 | `strings` | `stack.is_empty()` → `true` | caught by `balanced_cases` |
| M54 | `strings` | `')' => '(` → `')' => '[` | caught by `balanced_cases` |
| M55 | `strings` | `if a.len() != b.len() { return false; }` → `∅` | **equivalent**: strings of different lengths cannot produce all-zero byte counts, so the length test is an early exit that changes no answer |
| M56 | `strings` | `counts.iter().all` → `counts.iter().any` | caught by `anagram_cases` |
| M57 | `strings` | `counts[x as usize] -` → `counts[x as usize] +` | caught by `anagram_cases` |
| M58 | `dp` | `usize::from(a[i - 1] != b[j - 1])` → `0` | caught by `edit_distance_known_answers` |
| M59 | `dp` | `…prev[j] + 1).min(cur[j - 1] + 1)` → `…prev[j] + 1)` | caught by `edit_distance_is_a_metric_on_random_strings` |
| M60 | `dp` | `.min(cur[j - 1] + 1)` → `∅` | caught by `edit_distance_known_answers` |
| M61 | `dp` | `…Vec<usize> = (0..=b.len()).collect()` → `…Vec<usize> = vec![0; b.len() + 1]` | caught by `edit_distance_known_answers` |
| M62 | `dp` | `…mut cur = vec![i` → `…mut cur = vec![0` | caught by `edit_distance_known_answers` |
| M63 | `dp` | `if c == 0 \|\| c ` → `if c ` | **equivalent**: with a zero coin, `best[a - 0]` is `best[a]` itself, and the "improvement" test `prev + 1 < cur` can never hold, so the explicit skip changes no answer (it documents intent and avoids a pointless pass) |
| M64 | `dp` | `best[0] = Some(0)` → `best[0] = None` | caught by `min_coins_known_answers` |
| M65 | `dp` | `prev + 1 <` → `prev + 1 <=` | **equivalent**: `<=` only re-assigns a value equal to the current one |
| M66 | `dp` | `…tails.partition_point(\|&t\| t <` → `…tails.partition_point(\|&t\| t <=` | caught by `lis_matches_quadratic_dp` |
| M67 | `dp` | `tails[i] = x;` → `∅` | caught by `lis_matches_quadratic_dp` |
| M68 | `dp` | `…c == 0 \|\| c >` → `…c == 0 \|\| c >=` | caught by `min_coins_known_answers` |
| M69 | `topk` | `…heap.len() >` → `…heap.len() >=` | caught by `top_k_frequent_cases` |
| M70 | `topk` | `b.0.cmp(&a.0).then(a.1.cmp(&b` → `b.0.cmp(&a.0).then(b.1.cmp(&a` | caught by `top_k_frequent_matches_sorting` |
| M71 | `topk` | `b.0.cmp(&a` → `a.0.cmp(&b` | caught by `top_k_frequent_cases` |
| M72 | `topk` | `…Reverse(c), x` → `…Reverse(c), 0` | caught by `top_k_frequent_cases` |
| M73 | `topk` | `if k == 0 \|\| k ` → `if k ` | **equivalent**: with `k == 0` the size-`k` heap pops every item and `peek` returns `None`, which is also what the explicit test returns |
| M74 | `topk` | `…k == 0 \|\| k >` → `…k == 0 \|\| k >=` | caught by `kth_largest_cases` |
| M75 | `topk` | `heap.push(Reverse(` → `heap.push(Reverse(-` | caught by `kth_largest_cases` |
| M76 | `topk` | `…x).or_insert(0) += 1` → `…x).or_insert(0) += 2` | **equivalent**: doubling every count changes no ranking, so the selected items and their order are the same |
| M77 | `trie` | `if self.contains(word) { return; }` → `∅` | caught by `trie_empty_word` |
| M78 | `trie` | `n.end = true;` → `∅` | caught by `trie_empty_word` |
| M79 | `trie` | `…word).is_some_and(\|n\| n.end` → `…word).is_some(` | caught by `trie_basics` |
| M80 | `trie` | `n.through += 1; ` → `∅` | caught by `trie_basics` |
| M81 | `trie` | `…b).or_default(); n.through += 1;` → `…b).or_default();` | caught by `trie_basics` |
| M82 | `trie` | `…prefix).map_or(0` → `…prefix).map_or(1` | caught by `trie_basics` |
| M83 | `trie` | `buf.pop();` → `∅` | caught by `trie_basics` |
| M84 | `trie` | `if ` → `if !` | caught by `trie_basics` |
| M85 | `trie` | `let mut buf = prefix.as_bytes().to_vec` → `let mut buf = Vec::new` | caught by `trie_basics` |

What the table does not show: it tests *these* edits, not every possible bug; the equivalent mutants could hide a gap if my reasoning about them is wrong; and passing every mutant says nothing about behavior on very large inputs, under concurrency, or in an interview.

### Design notes

- **Every function states its complexity and its edge behavior** in the doc comment, because those are the two follow-up questions.
- **No dependencies and no `unsafe`.** Everything is the standard library.
- **Deterministic output** wherever a `HashMap` could leak its order.
- **Oracles in the tests.** The random comparisons use a 3-line generator so that the suite needs nothing external.

## Key takeaways

- A method beats a trick: clarify, examples, brute force, structure, plan, code, trace, complexity.
- Know the families: map, heap, binary search, sliding window, DP, graph, trie, sweep; for each, the invariant and the trap.
- Test against a slow oracle on small random inputs, and exhaust small cases when the logic is all boundaries.
- In Rust, choose ownership and failure behavior explicitly; prefer indices to pointers in linked structures; make output deterministic.
- A capstone you finished, with tests, honest limits and a README, is worth more than three you did not.
- Mutation testing result: 72 of 85 mutants were caught by a failing test, 5 by a hang and 8 are equivalent; the 3 real gaps were all "forgot the same check in the sibling function", and a contaminated batch (my own harness mistake) was re-run clean.

**Connections.** This lesson uses nearly everything: ownership and collections (Lessons 07 to 25), error design (26), testing, property testing and benchmarking (39 to 42), CI and supply chain (43), services (44 to 47), crypto and chains (48 to 58), and the reliability and design habits of Lesson 59. There is no next lesson; the roadmap's "Coverage check" lists where every requested area lives.

## Where to go from here

Pick one capstone and finish it. Read other people's Rust: the standard library's source for `slice::binary_search` and `BinaryHeap`, and a crate you use every week. Review a pull request in an open-source project you rely on. Keep a log of bugs you found and what test would have caught them. The skill that lasts is not knowing the answer; it is building the habit of proving it.

## References

Everything here is from memory; I could not reach documentation sites from the sandbox. Check current versions before relying on a detail.

- *The Rust Programming Language* and *Rust by Example* (the standard introductory texts); the standard library documentation for `BinaryHeap`, `BTreeMap`, `slice::partition_point`, `Option::is_none_or`.
- Cormen, Leiserson, Rivest and Stein, *Introduction to Algorithms* (Dijkstra, union-find, dynamic programming, topological sort).
- Bentley, *Programming Pearls* (binary search and the habit of testing small cases); the 2006 account of the midpoint-overflow bug in a widely used binary search (Joshua Bloch, on the Google Research blog, from memory).
- The Rustonomicon for `unsafe` questions; Miri for undefined-behavior checks.
- Karger-style and other design references: see [Lesson 59](59_architecture_system_design.md).

**Unchecked or from memory.** All of §2 and §7 to §9 (hiring and portfolio practice); the references and the description of the midpoint bug's history; the claim about the cache behavior behind the `lower_bound` timing (explained, not separately measured); statements about how other languages' and companies' interviews work. Not tested: any real interview, any online judge, any concurrent use of the structures in the crate.
