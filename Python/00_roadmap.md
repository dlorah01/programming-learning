---
layout: default
title: "Roadmap"
---

# Python Mastery Course — Roadmap

This file is the map of the whole course: what each module covers, why it sits where it does in the sequence, and how the modules build on each other. Use it to navigate, to decide where to jump in if you want to skip around, and as a checklist of what "complete" looks like.

**Format reminder, consistent across every module file:** each module covers its topics with intuitive + technical explanation, internal/CPython implementation detail, why the feature exists historically, a direct JS/TS comparison, progressively harder examples, common mistakes and interview traps, performance/memory considerations where relevant, a debugging exercise, and a tiered set of interview questions (junior → mid → senior → FAANG-style). Module 19 is the exception — it's a pure Q&A reference with full answers included, meant for self-testing rather than teaching new material.

---

## How the Course Is Sequenced

The order isn't arbitrary — each block depends on the one before it:

- **Modules 1–3 (Foundations)** establish how Python actually executes and what its basic values are. Everything later assumes this.
- **Modules 4–7 (Control Flow → Functions → Closures → Generators)** build up the language's execution/data-flow model layer by layer — each one leans on the last (e.g., closures need you to understand scope from control flow; decorators need closures; generators formalize the iteration protocol first previewed in control flow).
- **Modules 8–9 (OOP → Descriptors/Metaclasses)** go from "how to write a class" to "what a class actually is," mechanically. Module 8 is explicitly optional-depth — advanced material for reading framework internals and senior interviews, not day-to-day necessity.
- **Module 10 (Collections)** is where data-structure choice starts directly affecting the Big-O of code you write — it's positioned right after OOP because `dict`/`set`/`list` internals reuse the hashability rules from Module 1 and the mutability model throughout.
- **Modules 11–12 (Errors/Context Managers → Functional Tools)** round out core language fluency — how to fail safely and how to process data declaratively.
- **Module 13 (Modules/Packaging)** is a deliberate pivot from "language" to "project" — how code is organized and shared once it's no longer a single script.
- **Modules 14–16 (Concurrency → Memory → Performance)** are the systems-level trio — how Python actually behaves under load, under memory pressure, and under profiling. These three modules constantly cross-reference each other (the GIL explanation in 13 depends on refcounting from 14; performance fixes in 15 depend on both).
- **Module 17 (Type Hints)** sits after the systems trio deliberately — by this point you have enough of the language's real behavior in view to understand precisely what static typing does and doesn't guarantee.
- **Modules 18–19 (Testing → Production Engineering)** are the "shipping it" layer — how professional Python codebases are verified and operated.
- **Module 20 (Interview Sprint)** is cumulative retrieval practice — no new material, full answers included, meant to be revisited repeatedly rather than read once.

---

## Module-by-Module Summary

**Module 1 — Foundations**
Python's philosophy (Zen of Python, "one obvious way"), CPython vs. PyPy/other implementations, the compile-to-bytecode pipeline (`dis` module), the no-hoisting execution model, and the version timeline (3.6–3.13) with what's actually load-bearing at each release. *Sets up:* the "no JIT" fact that recurs in Modules 6, 13, 15; the no-hoisting fact that resolves confusion throughout.

**Module 2 — Names, Objects, References, Identity vs Equality, Mutability**
Why Python variables are labels, not boxes. `is` vs `==`, the small-int caching trap, the full mutability taxonomy, why "pass by reference/value" is the wrong question, the mutable-default-argument bug, shallow vs. deep copy. *The single most load-bearing module in the course* — referenced directly by Modules 4, 5, 7, 9, 14.

**Module 3 — Numbers, Strings, Booleans, None, Truthiness**
Arbitrary-precision `int`, floor division vs. truncation, string immutability and the `join`-vs-`+=` performance trap, `bool` as an `int` subclass, `None` as Python's single null value, and container truthiness (the sharpest divergence from JS).

**Module 4 — Control Flow, Comprehensions, `match`/`case`, Iteration**
No block scoping, iterator-based `for` loops (`enumerate`/`zip` over manual indexing), why comprehensions beat `map`/`filter`+`lambda` on both style and real speed, `match`/`case`'s structural power and no-fallthrough behavior, and the formal preview of the iterator protocol completed in Module 6.

**Module 5 — Functions Deep Dive**
Real keyword arguments (not JS's destructuring trick), positional-only/keyword-only parameters (`/` and `*`) as API-design tools, `*args`/`**kwargs` and the forwarding pattern that powers decorators, default-argument evaluation timing, and why CPython has no tail-call optimization.

**Module 6 — Closures & Decorators**
`nonlocal` and Python's explicit-scoping philosophy, the classic loop-variable late-binding bug, `lambda`'s deliberate single-expression restriction, and decorators built from first principles through `functools.wraps` and parameterized/stacked decorators.

**Module 7 — Generators & Iterators**
The formal `__iter__`/`__next__` protocol, iterable-vs-iterator distinction, why generators exist (laziness/memory, not raw speed), the bracket-vs-parens performance signal, `yield from`'s real purpose, and `send`/`throw` as the conceptual ancestor of `async`/`await` (paid off in Module 13).

**Module 8 — OOP & the Python Data Model**
The "everything is dunder dispatch" worldview — why `+`, `len()`, `in`, `for`, `bool()` all work the way they do. `__new__` vs `__init__`, MRO/C3 linearization, `dataclasses`, why Python starts with plain attributes and upgrades to `@property` later, and `__slots__` as a real memory tool.

**Module 9 — Descriptors & Metaclasses**
`__getattr__` vs `__getattribute__`, the descriptor protocol (what `@property` is actually built on, including the data/non-data descriptor distinction), and metaclasses as customizing class *creation* itself — with the honest senior take that `__init_subclass__` usually beats a full metaclass. *Advanced/optional depth.*

**Module 10 — Collections**
`list` as a dynamic array with amortized growth (and the O(n) `pop(0)` trap), Timsort, why dicts are ordered since 3.7, hash-table internals tied to Module 1's hashability rules, and the practical `deque`/`defaultdict`/`Counter`/`heapq` toolkit with real Big-O reasoning.

**Module 11 — Error Handling & Context Managers**
EAFP vs. LBYL, all four `try` clauses precisely (including the `finally`-with-`return` trap), custom exception hierarchies, `Exception` vs `BaseException`, and `with`/context managers from `__enter__`/`__exit__` through the `@contextmanager` shortcut.

**Module 12 — Functional Tools**
Why `reduce` was demoted out of builtins, `itertools`'s combinatorics (`permutations`/`combinations`/`product`), the real `groupby` consecutive-keys-only trap, and `functools.lru_cache`/`partial` as production-grade tools with real gotchas.

**Module 13 — Modules, Packaging, and Environments**
The `sys.modules`/`sys.path` import mechanics and circular imports, `__init__.py`'s real role, `if __name__ == "__main__"`, why Python needs explicit virtual environments unlike Node's automatic `node_modules`, and `pip`/`poetry`/`uv` compared.

**Module 14 — Concurrency: GIL, Threading, Multiprocessing, Asyncio**
What the GIL actually locks and doesn't, why race conditions still happen despite it, `threading` vs `multiprocessing` vs `asyncio` mapped onto I/O-bound vs CPU-bound work, the blocking-call-inside-`async def` trap, and concurrency vs. parallelism stated precisely.

**Module 15 — Memory: Reference Counting, GC, Weak References**
Deterministic refcounting and why it makes context managers feel natural, the cycle-detecting GC as a necessary backstop, `weakref`/`WeakValueDictionary` for caches, and optimization techniques tying back to `__slots__`, generators, and NumPy's boxed-vs-unboxed distinction.

**Module 16 — Performance & Profiling**
Measure-first discipline, `timeit` vs. noisy manual timing, `cProfile` workflow, spotting accidental O(n²) patterns from Module 9's table, and vectorization as the real production answer to "pure Python is slow."

**Module 17 — Type Hints & Static Analysis**
Type hints as fully erased/unenforced, `Protocol` as structural typing mirroring Python's duck-typing philosophy, `TypedDict`/`Literal` as near-1:1 TS analogs, and an honest comparison of what Python's type system can't guarantee vs. TypeScript's compile-gate.

**Module 18 — Testing**
Why `pytest` won out over `unittest`, fixtures as dependency injection with scope tradeoffs, the `Mock()`-silently-accepts-typos gotcha and `create_autospec` as the fix, and what a genuinely good test suite looks like.

**Module 19 — Production Engineering**
Why `logging` beats `print()` (including the lazy `%s`-formatting detail), the `src/` layout and the packaging-bug reasoning behind it, `Ruff`/`Black` mapped onto ESLint/Prettier, and a realistic staged CI pipeline.

**Module 20 — Interview Sprint (Full Q&A Reference)**
Cumulative, answer-complete retrieval practice across every module — junior rapid-fire, mid-level mechanism questions, senior design questions, FAANG-style deep-systems synthesis, and a full code-review exercise with a corrected code sketch. Meant to be revisited repeatedly, not read once.

---

## Suggested Ways to Use This Course

- **Linear, first pass:** work Modules 1 → 20 in order — this is how the dependency chain above is designed to be read.
- **Pre-interview review:** jump straight to Module 20, use it to self-test, and dive back into whichever earlier module exposes a gap.
- **Reference lookup:** once you've been through it once, treat each module as a standalone reference doc — they're each self-contained enough to reread individually when a specific topic comes up in real work.
- **Cross-module threads worth tracing on a second pass**, since they show up repeatedly and reward being seen as one continuous idea rather than separate facts: *mutability/hashability* (Modules 2, 5, 8, 10, 12, 15), *laziness* (Modules 4, 7, 16), *the no-JIT/boxed-object performance story* (Modules 1, 15, 16), and *explicit-over-implicit as a recurring design choice* (Modules 1, 6, 11, 17).

---

*All 21 files (this roadmap plus Modules 1–20) live in your outputs folder. Let me know if you'd like a condensed one-page cheat sheet next, or want to go deeper on any specific module.*
