# Python Mastery Course — Module 9: Collections

---

## 9.1 `list` — Dynamic Array Internals

**Technical model:** CPython's `list` is a dynamic array (like a JS `Array`, or conceptually a C++ `std::vector`) — a contiguous block of pointers to `PyObject`s, **not** a linked list (a genuinely common misconception worth explicitly correcting). This is why indexing is O(1): `lst[i]` is a direct pointer-array offset computation, not a traversal.

**Amortized growth:** when a `list` outgrows its allocated backing array, CPython over-allocates (roughly ~12.5% extra capacity, following a specific growth pattern in `listobject.c`) rather than growing by exactly one slot — this is the same amortized-doubling-family strategy JS engines use for arrays and C++ uses for `std::vector`. The consequence: `.append()` is **O(1) amortized**, not because every single append is cheap, but because the expensive O(n) reallocation-and-copy happens rarely enough that the *average* cost per append stays constant.

**Big-O cheat sheet you should have memorized cold:**

| Operation | Complexity | Why |
|---|---|---|
| `lst[i]` (index) | O(1) | direct array offset |
| `lst.append(x)` | O(1) amortized | see above |
| `lst.pop()` (no arg, end) | O(1) | just shrink from the end |
| `lst.pop(0)` (front) | **O(n)** | every remaining element must shift left one slot |
| `lst.insert(0, x)` | **O(n)** | same shifting cost |
| `x in lst` | O(n) | linear scan, no shortcuts |
| `lst.sort()` | O(n log n) | Timsort — see below |
| `len(lst)` | O(1) | length is cached, not counted |

**The `.pop(0)`/`.insert(0, ...)` trap is a genuine, very common interview and code-review flag:** using a `list` as a queue (repeatedly popping/inserting at the front) is silently O(n) per operation — O(n²) total for n operations — a real production performance bug pattern. The fix is `collections.deque` (§9.3), and "why shouldn't you use a list as a queue" is a fair, common mid-level question.

**Sorting — Timsort, worth knowing by name:** `list.sort()` / `sorted()` use **Timsort**, a hybrid merge-sort/insertion-sort algorithm (invented by Tim Peters — the same person who wrote the Zen of Python) specifically designed to exploit already-sorted or partially-sorted runs in real-world data, which is extremely common in practice (nearly-sorted logs, already-sorted sub-lists) — genuinely faster than a naive O(n log n) sort on realistic inputs, while still guaranteeing O(n log n) worst case. It's also **stable** (equal elements retain their relative order) — a real, sometimes-relied-upon guarantee, e.g. sorting by one key then a secondary key in two passes depends on stability to work correctly.

---

## 9.2 `tuple` — Not Just "an Immutable List"

Beyond immutability (Module 1), tuples have genuinely different use-case connotations in idiomatic Python: a `list` is conventionally a **homogeneous sequence of arbitrary length** (all elements same "kind," length not semantically meaningful); a `tuple` is conventionally a **fixed-size, possibly heterogeneous record** (`("Ada", 1815, "mathematician")` — think "lightweight struct," not "array"). This connotation is why function return values that bundle a few different things together (`return name, count` — implicitly a tuple) read naturally, while `return [name, count]` would read oddly to an experienced Python developer — it signals the wrong semantic category.

**Performance note:** tuples are marginally faster to construct and slightly more memory-compact than lists of the same length, precisely because their fixed size lets CPython skip the list's over-allocation machinery entirely — a genuine, small, real-world reason to prefer a tuple for genuinely fixed-shape data, beyond just "immutability."

---

## 9.3 `dict` — Hash Table Internals, and Why Python 3.7+ Dicts Are Ordered

**Technical model:** CPython's `dict` is a hash table — keys are hashed (`hash(key)`, requiring `__hash__`, which is why unhashable types like plain `list`/`dict`/`set` can't be dict keys — direct callback to Module 1's mutability/hashability discussion) and the hash determines a bucket/slot for O(1) average-case lookup, insert, and delete.

**Big-O:** `d[key]`, `d[key] = v`, `key in d`, `del d[key]` are all **O(1) average case**, **O(n) worst case** (pathological hash collisions — practically irrelevant for normal usage, worth knowing exists for a thorough answer, not worth worrying about day to day).

**Why dicts are insertion-ordered since 3.7 (a genuine, celebrated language change, not always true):** this was originally a CPython 3.6 *implementation detail* (a side effect of a new, more memory-compact internal representation splitting the hash table from a separate dense array preserving insertion order) that became an official, guaranteed *language* feature in 3.7. Before 3.6, dict order was explicitly unspecified and did vary — this is exactly why `collections.OrderedDict` existed as a separate type in the first place (§9.4) — it's now largely legacy, since a plain `dict` gives you the same ordering guarantee natively.

**JS comparison:** JS objects have their own, different, historically messy key-ordering rules (integer-like keys sort numerically first, everything else in insertion order) — Python's rule is simpler and uniform: always pure insertion order, no special-casing by key type.

**`dict.keys()`, `.values()`, `.items()` return *views*, not lists — a real, sometimes-tested distinction:**

```python
d = {"a": 1, "b": 2}
keys = d.keys()
d["c"] = 3
print(keys)   # dict_keys(['a', 'b', 'c']) — the view reflects the LIVE dict, not a snapshot at call time
```

This is efficient (no copy made) but is a genuine gotcha if you mutate a dict while iterating one of these views — `RuntimeError: dictionary changed size during iteration` — a common, real bug; the fix is iterating over `list(d.items())` (an explicit, deliberate snapshot copy) when you need to mutate during iteration.

---

## 9.4 The `collections` Module — Specialized Tools, Know When Each Wins

**`collections.deque` — a genuine double-ended queue, O(1) append/pop from *both* ends:**

```python
from collections import deque
dq = deque([1, 2, 3])
dq.appendleft(0)     # O(1) — this is the list.insert(0,...) fix from §9.1
dq.append(4)          # O(1)
dq.popleft()           # O(1)
```

Internally a doubly-linked list of fixed-size blocks (not a single Python-level linked list of individual nodes — that would be far slower; it's block-based for cache efficiency), trading away `list`'s O(1) random indexing (`dq[500]` is O(n) on a deque) for O(1) operations at both ends. **Use `deque` for queues/sliding-window/BFS problems; use `list` when you need random access or only ever touch the end.**

**`collections.defaultdict` — eliminates the "check if key exists, initialize if not" boilerplate:**

```python
from collections import defaultdict

# WITHOUT defaultdict — the common, slightly clunky idiom
groups = {}
for word in words:
    key = len(word)
    groups.setdefault(key, []).append(word)     # setdefault is the one-line non-defaultdict fix

# WITH defaultdict — cleaner for repeated grouping/counting patterns
groups = defaultdict(list)
for word in words:
    groups[len(word)].append(word)     # missing key auto-creates via list() — the factory — no KeyError
```

The constructor argument is a zero-arg **factory callable** (`list`, `int`, `set`, or your own function) invoked automatically the first time a missing key is accessed — genuinely idiomatic for grouping/counting/graph-adjacency-list construction, an extremely common real-world pattern.

**`collections.Counter` — a `dict` subclass purpose-built for counting, with real convenience methods beyond a `defaultdict(int)`:**

```python
from collections import Counter
c = Counter("mississippi")
print(c)                      # Counter({'i': 4, 's': 4, 'p': 2, 'm': 1})
print(c.most_common(2))        # [('i', 4), ('s', 4)] — genuinely useful, not trivially reimplemented
print(c["z"])                  # 0 — missing keys default to 0, no KeyError, unlike plain dict
c.update("test")               # adds counts rather than overwriting — a real, distinct convenience
```

`most_common(n)` alone (an efficient heap-based top-n selection under the hood, not a naive full sort) is reason enough to reach for `Counter` over a hand-rolled `defaultdict(int)` for any "count and rank" task — a very common interview-question building block (anagram checks, frequency analysis).

**`heapq` — a binary min-heap built directly on top of a plain `list` (not a separate type):**

```python
import heapq
nums = [5, 1, 8, 2, 9]
heapq.heapify(nums)          # O(n) — rearranges the list IN PLACE into heap order
heapq.heappush(nums, 3)       # O(log n)
smallest = heapq.heappop(nums) # O(log n) — always removes/returns the current minimum
print(heapq.nsmallest(3, nums))  # efficient top-k, O(n log k), not a full O(n log n) sort
```

`heapq` only gives you a **min-heap** natively — for a max-heap, the standard, genuinely idiomatic trick is negating values on the way in and out (`heapq.heappush(heap, -value)`), which surprises people expecting a `maxheap=True` flag that doesn't exist. `heapq` is the workhorse behind Dijkstra's algorithm, top-k/streaming-median problems, and task-scheduling-by-priority — extremely common in algorithmic interviews specifically.

---

## 9.5 `set` and `frozenset` — Hash-Table-Backed, Same Hashability Rules as `dict` Keys

Same hash-table internals and O(1) average-case membership testing as `dict` (in fact, a `set` is essentially "a dict with only keys, no values," internally). The main real-world value proposition: **`x in some_set` is O(1) average**, vs. **`x in some_list` at O(n)** — a genuinely common, high-value optimization: if you're repeatedly checking membership against a collection, converting it to a `set` once up front is a standard, expected move, not a micro-optimization nitpick, once the collection is nontrivially large or the check happens in a loop.

```python
allowed_ids = set(load_allowed_ids())   # O(n) once, up front
for record in huge_stream:
    if record.id in allowed_ids:         # O(1) per check instead of O(n) per check
        process(record)
```

Set algebra (`|` union, `&` intersection, `-` difference, `^` symmetric difference) has no direct JS built-in equivalent at all (JS's `Set` lacks these operators natively; you'd hand-roll them or use a library) — genuinely useful, idiomatic, and worth reaching for over manual loops when the problem is naturally set-shaped (deduplication, "find common elements," "find what's missing").

---

## 9.6 Common Mistakes / Interview Traps — Consolidated

- Using `list.pop(0)`/`list.insert(0, x)` in a loop, silently making an algorithm O(n²) instead of O(n) — reach for `deque` instead.
- Mutating a dict while iterating its `.keys()`/`.items()` view directly, hitting `RuntimeError`.
- Repeatedly checking `x in some_list` inside a loop over large data instead of converting to a `set` once.
- Forgetting `heapq` is min-heap-only and needing the negation trick for max-heap behavior.
- Choosing `list` for fixed-shape heterogeneous data where a `tuple` (or better, a `dataclass`/`namedtuple`) would communicate intent more clearly.
- Assuming `dict` order was always guaranteed — it's a 3.7+ language guarantee, not something that was always true (relevant if you ever touch legacy 2.7/3.5 code).

---

## 9.7 Exercises

**Predict the output:**
```python
d = {}
d.setdefault("a", []).append(1)
d.setdefault("a", []).append(2)
print(d)
```

**Core:** Given a large stream of log-line dicts, write an efficient function returning the top-5 most frequent `status_code` values using `Counter`, and explain its complexity vs. a hand-rolled `defaultdict(int)` + manual sort.

**Debugging exercise:** A function processes a queue by repeatedly doing `item = queue.pop(0)` inside a `while queue:` loop, and profiling shows it's the dominant cost in a hot path processing 100k items. Diagnose and fix using the correct data structure, explaining the complexity change precisely.

**Interview — junior:** "What's the time complexity of checking `x in my_list` vs `x in my_set`? Why the difference?"

**Interview — mid:** "Explain, precisely, why Python dict keys must be hashable, and why that means a `list` can never be a dict key but a `tuple` sometimes can and sometimes can't." (Ties directly back to Module 1's mutability/hashability discussion — a good check of cross-module retention.)

**Interview — senior:** "Design the data structures for a real-time leaderboard that needs frequent 'get top 10' queries and frequent score updates for arbitrary players. Compare a sorted list, a `heapq`, and a balanced structure, and justify a choice given the actual read/write frequency tradeoffs."

**Advanced / FAANG-style:** "Implement an LRU cache using `dict` + `deque` (or `OrderedDict`'s `move_to_end`), achieving O(1) get and put. Explain precisely which dict/deque properties from this module make O(1) possible, and where a naive list-based approach would degrade to O(n)."

---

*Next: Module 10 — Error Handling & Context Managers: exceptions as control flow, custom exception hierarchies, `try`/`except`/`else`/`finally` precisely, and `with`/context managers built from first principles (the `__enter__`/`__exit__` protocol, and `contextlib`).*
