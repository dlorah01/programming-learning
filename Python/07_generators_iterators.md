---
layout: default
title: "Module 6: Generators & Iterators"
---

# Python Mastery Course — Module 6: Generators & Iterators

---

## 6.1 The Iterator Protocol, Formalized (Module 3 Preview Made Precise)

Two dunder methods, two roles:

- **Iterable**: implements `__iter__(self)`, which returns an **iterator**. (A `list` is iterable but is *not itself* an iterator — see below.)
- **Iterator**: implements `__next__(self)`, which returns the next value or raises `StopIteration` when exhausted, *and* implements `__iter__(self)` returning `self` (so an iterator is trivially also iterable — this self-returning `__iter__` is what lets you use an iterator anywhere a `for` loop expects an iterable).

```python
nums = [1, 2, 3]
it = iter(nums)            # calls nums.__iter__() — returns a NEW list_iterator object
print(next(it))              # 1
print(next(it))              # 2
print(next(it))              # 3
print(next(it))              # StopIteration raised
```

**The crucial, frequently-tested distinction: a `list` is iterable but not an iterator.** Calling `iter()` on the same list twice gives you two *independent* iterator objects, each with its own position — this is why you can nest two `for` loops over the same list simultaneously without them interfering. An iterator, by contrast, is **stateful and single-use** — once exhausted, it stays exhausted:

```python
gen = (x for x in range(3))    # generator expression — an iterator, see §6.3
print(list(gen))                # [0, 1, 2]
print(list(gen))                # [] — already exhausted, permanently. No reset.
```

**JS comparison:** structurally identical to `Symbol.iterator` / the JS iterator protocol (`{ next() { return {value, done} } }`) — if you understand JS generators/iterators, the shape is already familiar; Python's `StopIteration`-as-exception vs. JS's `{done: true}` sentinel object is the main surface-level difference, and it's a direct instance of Module 0's "explicit is better than implicit, errors shouldn't pass silently" — Python treats exhaustion as an exceptional control-flow event the loop machinery specifically catches, rather than a data value you could accidentally forget to check.

---

## 6.2 Generators — Functions That Pause

**Intuition:** a generator function looks like a normal function but contains `yield`; calling it doesn't run the body at all — it returns a **generator object** (an iterator, per §6.1, built automatically for you). Each call to `next()` on it runs the function body until the next `yield`, pauses there (preserving all local state — variables, the exact instruction pointer position), and resumes exactly there on the next `next()` call.

```python
def count_up_to(n):
    i = 1
    while i <= n:
        yield i
        i += 1

gen = count_up_to(3)
print(gen)          # <generator object count_up_to at 0x...> — note: body has NOT run yet
print(next(gen))     # 1 — runs until first yield
print(next(gen))     # 2
print(next(gen))     # 3
print(next(gen))     # StopIteration
```

**The internal mechanism, technically:** a generator function's frame is *not* discarded when it hits `yield` the way a normal function's frame is discarded on `return` — CPython keeps the frame object alive, suspended, referenced by the generator object itself. This is genuinely different machinery from a normal function call, and it's *why* generators have memory cost proportional to their suspended state, not to the data they'll eventually produce — the entire point covered next.

**Why generators exist — the core value proposition, and the most important idea in this module: laziness.**

```python
def all_lines_eager(filename):
    with open(filename) as f:
        return f.readlines()        # loads the ENTIRE file into memory as a list, all at once

def all_lines_lazy(filename):
    with open(filename) as f:
        for line in f:                # file objects are themselves iterators — one line at a time
            yield line                 # produces exactly one line, on demand, no matter the file size
```

`all_lines_lazy` can process a 500GB log file with roughly constant memory overhead — the generator produces one line at a time, and if the consumer only needs the first match and then `break`s, the rest of the file is never even read. `all_lines_eager` must materialize everything up front. This is the single biggest practical reason to reach for a generator: **you get the composability of "just iterate over it with `for`" without paying the memory cost of building the whole sequence first.**

---

## 6.3 Generator Expressions — Comprehension Syntax, Lazy Semantics

```python
squares_list = [x * x for x in range(1_000_000)]     # eager — builds a full million-element list NOW
squares_gen  = (x * x for x in range(1_000_000))     # lazy — builds nothing yet; ~constant memory
```

Same syntax as a list comprehension (Module 3), parens instead of brackets, but semantically it's sugar for a generator function — nothing computes until you iterate. This is the idiomatic choice whenever you're going to consume a sequence exactly once, in order, and don't need random access/indexing/`len()`/multiple passes:

```python
total = sum(x * x for x in range(1_000_000))    # note: no extra parens needed as the sole call arg
                                                    # never materializes the full list of squares in memory
```

**Real interview/production distinction:** `sum([x*x for x in range(n)])` and `sum(x*x for x in range(n))` produce the *identical result* but have different memory profiles — the first builds a full intermediate list just to immediately throw it away; the second never does. A senior reviewer will flag the bracket version in a hot path over large `n` as an unnecessary allocation. This single parenthesis-vs-bracket choice is a genuine, common code-review-level performance signal — not a stylistic footnote.

---

## 6.4 `yield from` — Delegating to a Sub-Iterator

```python
def inner():
    yield 1
    yield 2

def outer():
    yield from inner()    # equivalent to: for x in inner(): yield x  — but also forwards
    yield 3                 # send()/throw()/return values correctly, which the manual loop doesn't

print(list(outer()))   # [1, 2, 3]
```

**Why it exists beyond being sugar for a loop:** `yield from` correctly plumbs `.send()` and `.throw()` calls (§6.6) through to the sub-generator, and correctly propagates the sub-generator's `return` value as the *result* of the `yield from` expression itself — a manual `for x in inner(): yield x` loop does neither of these correctly. This matters for recursively flattening nested generator structures (e.g. a generator-based recursive tree traversal) and is the direct conceptual ancestor of `await` in `async def` coroutines (Module 13) — `await` is, at a deep implementation level, doing something structurally very close to `yield from` under the hood, which is a genuinely useful mental bridge once you reach async.

---

## 6.5 Memory & Performance: Generators vs. Lists — When To Use Which

| Use a list when... | Use a generator when... |
|---|---|
| You need random access (`seq[5]`), `len()`, slicing | You only need one forward pass |
| You'll iterate multiple times | You'll iterate exactly once |
| The full sequence is small / fits comfortably in memory | The sequence is huge, infinite, or expensive-per-item to produce |
| You need to pass it to something requiring a concrete sequence (e.g. `json.dumps`, most sorting) | You're building a data-processing pipeline (filter → transform → filter, chained lazily) |

**Performance nuance that surprises people:** generators aren't "faster" in a raw-throughput sense — iterating a pre-built list is typically *faster per element* than a generator, because a generator pays real per-`next()`-call overhead (resuming a suspended frame has genuine cost) whereas list iteration is close to a raw memory scan. The actual win is **memory**, and **avoiding wasted work** when a consumer might stop early (`break`, `next(gen)` once, `any()`/`all()` short-circuiting) — generators let you not compute values nobody ends up consuming. Know this trade-off precisely; "generators are just faster" is a common, imprecise answer that a strong interviewer will push back on.

---

## 6.6 `send()` and `throw()` — Generators as Two-Way Channels (Know These Exist)

```python
def echo():
    while True:
        received = yield          # yield can be an EXPRESSION that receives a value via .send()
        print(f"got: {received}")

gen = echo()
next(gen)            # must "prime" the generator to the first yield before sending
gen.send("hello")     # prints "got: hello"
gen.throw(ValueError, "boom")   # raises ValueError INSIDE the generator, at the paused yield point
```

This is niche in everyday application code (you'll rarely hand-write `send`-driven generators) but is the exact mechanism `asyncio` coroutines were originally built on before `async`/`await` became dedicated syntax — worth recognizing conceptually now so Module 13 doesn't feel like new magic, just a formalized version of this.

---

## 6.7 Common Mistakes / Interview Traps — Consolidated

- Trying to iterate an already-exhausted generator a second time and being confused why it silently produces nothing (no error — just an empty iteration, easy to miss in a bug report).
- Passing `[x for x in huge_range]` to a function like `sum()`/`any()`/`all()`/`max()` that only needs one pass — the bracket-vs-parens distinction (§6.3) is a real, checkable performance smell.
- Assuming generators are unconditionally "faster" rather than "more memory-efficient, with per-call overhead" (§6.5).
- Forgetting to "prime" a `send()`-driven generator with an initial `next()` call before the first `.send()`.
- Confusing `yield from inner()` with a plain `for`-loop-and-yield when `send`/`throw`/return-value propagation actually matters.

---

## 6.8 Exercises

> **Run a snippet locally.** Copy any code block below and feed it straight to Python from your clipboard — on macOS: `pbpaste | python3 -` — or run `python3 -`, paste, and press Ctrl-D. Predict the output first, *then* run it. A few snippets reference a helper you're asked to write, or leave an input undefined — save those to a scratch file (`python3 scratch.py`) and fill in the blank first.

**Predict the output:**
```python
def gen():
    print("start")
    yield 1
    print("middle")
    yield 2
    print("end")

g = gen()
print("created")
print(next(g))
print(next(g))
```

**Core:** Write a generator `chunked(iterable, size)` that yields successive lists of length `size` from any iterable, without ever materializing the whole input in memory at once.

**Debugging exercise:** A function processes a 10GB file by doing `lines = list(open(path))` then iterating `lines` twice (once to validate, once to process) — it crashes with a memory error in production but worked fine on the developer's small test file. Diagnose, and redesign using generators — including how you'd handle needing "two passes" over data that's now lazy and single-use.

**Interview — junior:** "What's the difference between a list comprehension and a generator expression? When would each be the wrong choice?"

**Interview — mid:** "Explain, precisely, why `iter([1,2,3])` called twice gives you two independent iterators, but calling `next()` twice on the *same* iterator object does not reset."

**Interview — senior:** "Design a lazy ETL pipeline — read records from a huge lazy source, filter, transform, and write output — using only generators chained together (no intermediate lists). Explain what happens to memory usage as the pipeline runs, and where, if anywhere, you'd deliberately break laziness (e.g., for something that genuinely needs all records at once, like sorting)."

**Advanced / FAANG-style:** "Explain what `yield from` propagates that a plain `for x in sub: yield x` loop does not, and connect this precisely to how you'd expect `await` to behave for a coroutine that awaits another coroutine." (Direct, deliberate bridge into Module 13.)

---

## 6.9 Exercise Solutions

Try each exercise cold first, then check your reasoning here. Keep a short personal note on anything you got wrong — that running log is the highest-value review material as the course goes on.

**Q — Predict the output:**
```python
def gen():
    print("start")
    yield 1
    print("middle")
    yield 2
    print("end")

g = gen()
print("created")
print(next(g))
print(next(g))
```
**A:** `created`, then `start`, then `1`, then `middle`, then `2`. Calling `gen()` doesn't execute any of the function body — it just creates a suspended generator object, so `"created"` prints before anything inside `gen` runs. The first `next(g)` resumes execution from the top, printing `"start"`, then pauses at the first `yield`, handing back `1` (printed by the outer `print`). The second `next(g)` resumes right after that `yield`, printing `"middle"`, then pauses at the second `yield`, handing back `2`.

**Q — Core:** Write a generator `chunked(iterable, size)` that yields successive lists of length `size` from any iterable, without materializing the whole input in memory at once.
**A:**
```python
def chunked(iterable, size):
    chunk = []
    for item in iterable:
        chunk.append(item)
        if len(chunk) == size:
            yield chunk
            chunk = []
    if chunk:
        yield chunk
```

**Q — Debugging:** A function processes a 10GB file by doing `lines = list(open(path))` then iterating `lines` twice — it crashes with a memory error in production but worked fine on a small test file. Diagnose and redesign.
**A:** `list(open(path))` forces the *entire* file into memory as a list of lines at once — fine for a small test file, catastrophic at 10GB. Redesign around generators, and address the "need two passes" requirement by simply opening the file twice — file objects are lazy, sequential iterators, so `for line in open(path): validate(line)` followed by a second, separate `for line in open(path): process(line)` keeps memory roughly constant across both passes, at the cost of reading the file from disk twice. If the two passes' logic can reasonably be combined, a single pass that validates and processes each line together avoids even that second read.

**Q — Interview (junior):** "What's the difference between a list comprehension and a generator expression? When would each be the wrong choice?"
**A:** Same syntax shape, different brackets — a list comprehension eagerly builds the entire list in memory immediately; a generator expression lazily produces one value at a time on demand. A generator expression is the wrong choice when you need `len()`, indexing, or to iterate more than once (it supports exactly one forward pass, then it's exhausted). A list comprehension is the wrong choice when the sequence is very large and you only need a single pass — you'd be paying for memory you don't need.

**Q — Interview (mid):** "Explain, precisely, why `iter([1,2,3])` called twice gives you two independent iterators, but calling `next()` twice on the same iterator object does not reset."
**A:** `iter(some_list)` calls `some_list.__iter__()`, which explicitly constructs and returns a brand-new iterator object each time it's called — the list itself holds no iteration-position state; it's iterable, not itself an iterator. So two separate `iter()` calls on the same list genuinely produce two independent objects, each tracking its own position. `next()`, by contrast, is called *on a specific iterator object* — it advances that object's own internal position and has no mechanism to "reset," since nothing in the iterator protocol defines a rewind operation; once an iterator is created, its position only ever moves forward.

**Q — Interview (senior):** "Design a lazy ETL pipeline — read, filter, transform, write — using only generators chained together. Explain what happens to memory usage as the pipeline runs, and where you'd deliberately break laziness."
**A:**
```python
def read_records(path):
    for line in open(path):
        yield parse(line)

def filter_valid(records):
    for r in records:
        if is_valid(r):
            yield r

def transform(records):
    for r in records:
        yield apply_transform(r)

for record in transform(filter_valid(read_records(path))):
    write(record)
```
Memory stays roughly constant throughout — at any given moment, only a small number of records (one per stage, roughly) are actually "in flight," since each stage pulls from the previous one only as the final `for` loop demands the next value. Laziness should be deliberately broken only where an operation genuinely requires the full dataset at once — for example, if a later requirement needed the records sorted by some field, `sorted()` would need to materialize the entire sequence in memory at that specific point, which should be an isolated, intentional exception rather than something that silently happens throughout the pipeline.

**Q — Advanced:** "Explain what `yield from` propagates that a plain `for x in sub: yield x` loop does not, and connect this to how you'd expect `await` to behave for a coroutine that awaits another coroutine."
**A:** `yield from sub_generator` correctly forwards `.send()` and `.throw()` calls made on the *outer* generator down into the sub-generator, and it propagates the sub-generator's eventual `return` value as the result of the `yield from` expression itself. A manual `for x in sub: yield x` loop does neither — it only forwards the yielded values, with no channel back down for sent values/exceptions and no way to capture a return value at all. This maps directly onto `await`: when a coroutine awaits another coroutine, values/exceptions and the final return value are expected to propagate correctly through that chain of awaits, exactly the same shape as `yield from` delegating into a sub-generator — which is architecturally close to how `await` is actually implemented under the hood.

---

*Next: Module 7 — OOP & the Python Data Model: classes, inheritance vs. composition, dataclasses, properties, and the full "everything is dunder-method dispatch" worldview that explains why `+`, `len()`, `in`, `for`, and `bool()` all work the way they do.*
