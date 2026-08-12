---
layout: default
title: "Module 14: Memory — Reference Counting, GC, Weak References"
---

# Python Mastery Course — Module 14: Memory — Reference Counting, GC, Weak References

---

## 14.1 Reference Counting — CPython's Primary Memory-Management Mechanism

**The core mechanism, precisely:** every `PyObject` in CPython carries an internal `ob_refcnt` field — a count of how many references currently point to it. Binding a name to an object, appending it to a list, storing it as a dict value, passing it as an argument — all increment the refcount; a name going out of scope, being rebound, an object being removed from a container — all decrement it. **The instant an object's refcount hits zero, it is deallocated immediately** — this is deterministic, synchronous, and happens exactly at that point in program execution, not at some later, unpredictable garbage-collection pause.

```python
import sys

x = []
print(sys.getrefcount(x))   # typically 2 — one for `x`, one for the temporary argument reference
                              # getrefcount itself creates a temporary reference, hence +1

y = x
print(sys.getrefcount(x))   # 3 — now two real names (x, y) plus the temporary
del y
print(sys.getrefcount(x))   # back to 2
```

**Why this matters, practically and for interviews — it's a genuine, significant architectural difference from JS/V8, Java/JVM, and most GC'd languages:** V8, the JVM, and most modern managed runtimes use purely **tracing** garbage collectors (mark-and-sweep or generational variants) with **no** deterministic, immediate reclamation — an object becomes eligible for collection when unreachable, but *when* the collector actually runs and frees it is not something your code can predict or rely on. CPython's refcounting gives you something closer to deterministic destruction (much like C++ RAII/`std::shared_ptr`) for the vast majority of objects — this is precisely *why* `with open(...) as f:` (Module 10) reliably closes files exactly when you'd expect, and it's part of why context managers feel so natural and load-bearing in Python specifically, compared to languages where "when does this get cleaned up" is inherently fuzzier.

**The performance cost, worth knowing precisely:** every single reference operation — assignment, function call, container insert — pays a refcount increment/decrement, even for immutable, "just reading" operations. This is a genuine, real overhead baked into every operation in CPython, and it's part of *why* the free-threaded/no-GIL build (Module 13.1) is such a hard engineering problem — refcounting needs to become thread-safe (atomic increments/decrements) everywhere without a GIL protecting it, which has real performance costs of its own that the CPython team has spent years mitigating.

---

## 14.2 Cycle-Detecting Garbage Collection — Why Refcounting Alone Isn't Enough

**The problem refcounting can never solve on its own — reference cycles:**

```python
class Node:
    def __init__(self):
        self.parent = None
        self.child = None

a = Node()
b = Node()
a.child = b
b.parent = a
del a
del b
# a and b now reference EACH OTHER, and nothing external references either —
# but each object's refcount is still 1 (from the other), never reaching 0!
# Pure refcounting would leak this pair forever.
```

This is exactly why CPython has a **second, separate garbage collector** — a generational, tracing cycle detector (the `gc` module) that runs periodically, specifically to find and collect groups of objects that reference each other but are unreachable from any live root (a global, a local variable, a stack frame). Refcounting handles the overwhelming majority of deallocation immediately and deterministically; the cycle collector exists purely as a backstop for the specific pathological case refcounting structurally cannot solve.

**Generational design, briefly, why it exists:** the empirical observation (shared with most tracing collectors, including V8's) is that most objects die young — so the `gc` module divides tracked objects into generations (0, 1, 2), scanning the youngest generation most frequently (cheap, since it's usually small) and older generations progressively less often (objects that have survived several scans are statistically likely to keep surviving, so re-scanning them constantly would waste work). You can introspect and tune this:

```python
import gc
print(gc.get_threshold())   # e.g. (700, 10, 10) — collection trigger thresholds per generation
gc.collect()                  # force a full collection cycle manually — rarely needed in practice
```

**Practical, senior-level guidance:** manually calling `gc.collect()` or disabling the collector (`gc.disable()`) is a genuine, occasionally-legitimate performance technique for specific workloads (e.g., a batch job that creates huge numbers of short-lived, cycle-free objects and wants to skip the cycle collector's overhead entirely, re-enabling/collecting at defined checkpoints) — but it's an advanced, workload-specific optimization, not a default habit; reaching for it without profiling first is a real anti-pattern.

**Only container-capable types can participate in cycles, worth knowing precisely:** simple, "leaf" objects like `int`, `str`, `float` can never be part of a reference cycle (they can't hold references to other objects at all) — only types that can contain other objects (custom class instances, `list`, `dict`, closures capturing cells) are tracked by the cycle collector in the first place; this is why the classic cycle example always involves objects referencing each other, never plain numbers/strings.

---

## 14.3 Weak References — Referencing Without Incrementing the Refcount

```python
import weakref

class Cache:
    pass

obj = Cache()
weak = weakref.ref(obj)
print(weak())          # <Cache object at ...> — call it like a function to get the actual object
del obj
print(weak())           # None — the object was actually collected; the weak reference doesn't keep it alive
```

**Why this exists — a genuine, real production tool, not a curiosity:** a normal reference keeps its target alive (contributes to the refcount); a `weakref` lets you hold a reference *without* preventing garbage collection — essential for caches, observer/listener registries, and parent-child object graphs where you specifically want to avoid creating a reference cycle or accidentally keeping large objects alive forever just because a cache happens to still reference them.

**`weakref.WeakValueDictionary` / `WeakKeyDictionary` — the practical, most commonly reached-for form:**

```python
cache = weakref.WeakValueDictionary()
cache["key"] = SomeExpensiveObject()
# if nothing else holds a strong reference to that object, it can be garbage collected,
# and it will automatically, silently disappear from the cache — no manual eviction logic needed
```

Genuinely idiomatic for memoization/caching patterns where you want cached results to be reclaimable under memory pressure rather than pinned in memory forever — a real, meaningful difference from a plain `dict`-based cache, which would keep every cached value alive indefinitely by design (this is precisely why `functools.lru_cache`, Module 11.3, has a `maxsize` — a bounded-size eviction strategy addressing the same underlying "don't let a cache grow forever" concern from a different angle).

---

## 14.4 Concrete Memory-Optimization Techniques — Tying the Whole Module Together

- **`__slots__` (Module 7.6)** — the single highest-leverage, most broadly applicable technique for large collections of small objects; eliminates the per-instance `__dict__` overhead entirely.
- **Generators over lists (Module 6.5)** — avoid materializing full sequences in memory when a single lazy pass suffices.
- **`array.array` / NumPy arrays over `list` for large homogeneous numeric data** — a plain Python `list` of a million floats stores a million separate, individually-boxed `PyObject` `float` instances (each with its own refcount, type pointer, and value — real overhead per element); a NumPy array stores a single contiguous C buffer of raw doubles, dramatically more memory-compact and cache-friendly — a genuinely major, common real-world memory (and performance) win for numeric-heavy code, worth knowing as the default answer to "how do I make this list of a million numbers use less memory."
- **String interning awareness (Module 1.2)** — don't rely on it, but understand it's happening for some short strings/identifiers as a memory-saving CPython implementation detail.
- **Small-int caching (Module 1.2)** — same category, an implementation detail that quietly reduces memory for common small integer values, never something to depend on.
- **Weak references / bounded caches (§14.3, `lru_cache(maxsize=...)`)** — prevent unbounded cache growth from becoming a memory leak in long-running processes.

**Profiling before optimizing — the actual professional workflow, worth stating explicitly here since Module 15 covers tooling in depth:** `sys.getsizeof()` for a single object's shallow size, and libraries like `tracemalloc` (stdlib, tracks allocations with a traceback of where memory was allocated) or `memory_profiler`/`objgraph` for real investigation — guessing at memory optimizations without measuring first is a real, common anti-pattern; `__slots__`/generators/arrays are worth defaulting to for large-scale collections, but genuine memory debugging always starts with actual measurement, not intuition.

---

## 14.5 Common Mistakes / Interview Traps — Consolidated

- Claiming Python "doesn't have garbage collection, just reference counting" — incomplete; the cycle collector is a real, necessary second mechanism (§14.2).
- Assuming reference cycles are rare/theoretical — parent-child object graphs, doubly-linked structures, and closures capturing surrounding objects are genuinely common real-world cycle sources.
- Building a cache with a plain `dict` and being surprised it grows unbounded and never releases memory, when `WeakValueDictionary` or a bounded `lru_cache` would have been the correct tool.
- Storing millions of numbers in a plain `list` instead of `array.array`/NumPy and being surprised by memory usage, without realizing each element is a fully boxed, individually-refcounted object.
- Reaching for `gc.disable()`/manual `gc.collect()` tuning without first profiling to confirm the cycle collector is actually a measurable bottleneck.

---

## 14.6 Exercises

**Predict the output:**
```python
import gc

class Node:
    def __init__(self):
        self.ref = None

gc.disable()
a = Node()
b = Node()
a.ref = b
b.ref = a
del a, b
print(gc.collect())   # what does this return, and why is it nonzero despite refcounting alone
                        # being unable to free a and b?
```

**Core:** Implement a simple observer pattern (`Subject` holding a list of `Observer` callbacks) using `weakref` so that observers can be garbage collected normally once nothing else references them, without the `Subject` keeping them alive forever, and demonstrate the difference vs. a plain-list version.

**Debugging exercise:** A long-running service's memory usage grows steadily over days despite no obvious leaks in the "business logic." A `tracemalloc` snapshot shows thousands of small custom `TreeNode` objects with `.parent`/`.children` references never being freed. Diagnose the likely cause precisely and propose a fix using what this module covers.

**Interview — junior:** "How does CPython know when to free an object's memory? What's the very first thing that happens?"

**Interview — mid:** "Why can't reference counting alone handle every case? Give a concrete example of an object graph it fails to collect, and explain what additional mechanism CPython uses to catch it."

**Interview — senior:** "You're building an in-memory cache for expensive computed results in a long-running service. Compare a plain `dict`, `functools.lru_cache(maxsize=N)`, and `weakref.WeakValueDictionary` for this use case, and justify a choice given that some cached objects are large and memory pressure is a real production concern."

**Advanced / FAANG-style:** "Explain precisely why the CPython free-threaded (no-GIL) build is a genuinely hard engineering problem specifically because of reference counting, connecting this back to what you learned about the GIL in Module 13 and refcounting here."

---

*Next: Module 15 — Performance & Profiling: Big-O in practice, `timeit`/`cProfile`/`line_profiler`, vectorization, and the concrete, evidence-based workflow for making Python code faster.*
