---
layout: default
title: "Module 19: Interview Sprint (Full Q&A Reference)"
---

# Python Mastery Course — Module 19: Interview Sprint (Full Q&A Reference)

This module is formatted as a **self-study reference**: every question is followed immediately by its answer and the technical reasoning behind it, so you can use it to test yourself (cover the answer, try to produce it, then check) or just read it straight through as a cumulative review of Modules 0–18. No new concepts — pure synthesis and retrieval practice.

---

## 19.1 Interview Strategy — How to Actually Answer, Not Just What to Know

**Structure every non-trivial answer the same way, out loud:** (1) state the direct answer/mechanism first, in one or two sentences — don't build up to it; (2) give the concrete example or code that demonstrates it; (3) name the trade-off or "when this matters/doesn't matter" — this third step is what separates a mid-level answer from a senior one, and interviewers are explicitly listening for it.

**Narrate your reasoning, especially when debugging live.** An interviewer watching you silently stare at code for 30 seconds learns nothing; an interviewer hearing "this loop looks O(n), but that `in` check is on a list, so it's actually O(n²) — let me check what data structure would fix that" learns you can diagnose, not just eventually arrive at an answer.

**When you don't know something, say so precisely, then reason toward it.** "I don't know the exact internal mechanism, but based on how reference counting works, I'd guess..." is a genuinely strong answer — much stronger than confidently guessing wrong, and much stronger than a flat "I don't know."

**Always volunteer the trade-off, unprompted, once you've given the "right" answer.** Nearly every question below has a canonical correct-sounding answer *and* a real caveat — stating the caveat unprompted is a reliable senior-level signal.

---

## 19.2 Junior-Level Round — Should Be Instant Recall

**Q1. Difference between `is` and `==`?**
`is` checks identity — whether two names point to the *exact same object* (`id(a) == id(b)`). `==` checks value equality, dispatching to `__eq__`, which for containers means recursive structural comparison. `[1,2] == [1,2]` is `True` (same contents, different objects); `[1,2] is [1,2]` is `False`. Always use `is` for `None`/sentinel checks (`x is None`, never `x == None`) and `==` for everything else. The trap: CPython caches small ints (-5 to 256) and some short strings, so `x = 256; y = 256; x is y` can be `True` by implementation accident — never rely on this.

**Q2. Why is a mutable default argument dangerous?**
Default argument values are evaluated **once**, at `def` execution time, not per call. `def f(bucket=[]):` creates exactly one list object, shared and mutated across every call that doesn't pass its own `bucket`. Fix: `def f(bucket=None): if bucket is None: bucket = []`.

**Q3. What does `bool([])` return, and why? What about `bool([0])`?**
`bool([])` is `False`; `bool([0])` is `True`. Truthiness for containers is based purely on emptiness (`__len__() == 0`), not on the truthiness of their contents — a non-empty list is always truthy even if every element inside it is individually falsy. (Divergence from JS: `[]` is truthy in JS.)

**Q4. Difference between `/` and `//`?**
`/` is true division, always returns a `float`. `//` is **floor** division, not truncation — it rounds toward negative infinity, so `-7 // 2` is `-4`, not `-3`. `%` always returns a result with the same sign as the divisor (`-7 % 2 == 1`), unlike JS's remainder operator, which follows the dividend's sign.

**Q5. What do `*args`/`**kwargs` actually capture?**
`*args` collects extra positional arguments into a `tuple`; `**kwargs` collects extra keyword arguments into a `dict`. The names are pure convention — the `*`/`**` prefixes are the real syntax. At a call site, the same symbols do the opposite job: unpacking (`f(*my_list)`, `f(**my_dict)`) spreads a collection into separate arguments.

**Q6. What is a closure, in one sentence?**
An inner function that retains access to variables from its enclosing function's scope even after the enclosing function has returned, by holding a reference to the enclosing scope's cell — not a copy of the value at creation time, which is exactly why loop-variable closures (`[lambda: i for i in range(3)]`) all see the same final `i`.

**Q7. Difference between a list comprehension and a generator expression?**
Same syntax shape (`[x for x in y]` vs `(x for x in y)`), but a list comprehension eagerly builds the entire list in memory immediately; a generator expression is lazy, producing values one at a time on demand, with no full materialization. Use the generator form when you'll only iterate once and don't need indexing/`len()`/multiple passes — it's a memory win, not generally a speed win (generators pay real per-`next()`-call overhead).

**Q8. What does `__init__` do, and how does it differ from `__new__`?**
`__new__(cls, ...)` actually **creates and returns** the instance (it's implicitly static); `__init__(self, ...)` receives that already-created instance and initializes its state, returning `None`. `SomeClass(...)` calls `__new__` first, then `__init__` on the result. You almost never override `__new__` — it matters for subclassing immutable types (where `__init__` runs too late to set anything) and for patterns like Singletons.

**Q9. Time complexity of `x in a_list` vs `x in a_set`?**
`x in a_list` is O(n) — a linear scan, no shortcuts. `x in a_set` is O(1) average case, because `set` is hash-table-backed (same underlying structure as `dict`, just keys with no values). If you're checking membership repeatedly against the same collection, converting it to a `set` once up front is a standard, expected optimization, not a micro-optimization nitpick.

**Q10. What does `with open(...) as f:` guarantee, mechanically?**
It desugars to `f = open(...).__enter__()` followed by a `try/finally` where `__exit__()` is called unconditionally in the `finally` — so the file is guaranteed closed whether the block completes normally, returns early, or raises an exception. `__exit__` returning `True` would additionally suppress any exception that occurred inside the block (the default, `None`/`False`, lets it propagate after cleanup).

**Q11. What does the GIL prevent, precisely?**
It ensures only one thread executes Python bytecode at a time, even on a multi-core machine — because CPython's reference counting isn't thread-safe without it. It does **not** prevent threads from existing or from providing real concurrency benefit for I/O-bound work (the GIL releases during blocking I/O), and it does **not** make compound operations like `counter += 1` atomic — race conditions are still fully possible and still need explicit `Lock`s.

**Q12. Are Python type hints enforced at runtime?**
No — never, by default. `def f(x: int) -> str:` places zero runtime restriction on what `x` actually is; CPython ignores hints entirely at execution time. They exist purely for static analysis tools (`mypy`, `pyright`) and IDE tooling. This is architecturally similar to TypeScript (also erased at runtime) but culturally far less consistently enforced across the Python ecosystem — there's no build step that blocks shipping mistyped code by default.

**Q13. Why use `logging` instead of `print()` in production code?**
`logging` gives you severity levels (filterable per environment without code changes), multiple simultaneous destinations (stdout, file, remote aggregator), and consistent structured formatting — none of which `print()` provides. `print()` always fires unconditionally with no way to suppress/route it externally.

---

## 19.3 Mid-Level Round — Requires Explaining Mechanism

**Q1. Find and fix the bugs:**
```python
def get_or_create(cache, key, factory=lambda: []):
    if key not in cache:
        cache[key] = factory()
    return cache[key]

def process(items, results={}):
    for item in items:
        bucket = get_or_create(results, item.category)
        bucket.append(item)
    return results
```
**Answer:** `results={}` in `process` is the classic mutable-default bug (Q2 above) — every call without an explicit `results` argument shares and accumulates into the *same* dict forever, across the life of the process. Fix: `results=None`, then `if results is None: results = {}` inside the function. The `factory=lambda: []` default in `get_or_create`, by contrast, is actually **fine** — because it's a *callable* being defaulted, not a mutable *value*; the same lambda object is reused across calls, but calling it (`factory()`) produces a fresh list each time it's invoked, so there's no shared-state bug here. This pairing is a deliberate trap: it tests whether you understand *why* the mutable-default rule applies to defaulted values, not to defaulted callables that produce fresh values.

**Q2. Why does a loop-heavy pure-Python function run ~50x slower than the NumPy equivalent, precisely?**
Two compounding reasons, not one: (1) CPython has no JIT by default — every loop iteration re-dispatches through the bytecode interpreter's opcode-by-opcode evaluation loop, with real per-operation overhead that a compiled/JIT'd language avoids; (2) each element in a plain Python list of numbers is a fully boxed `PyObject` (its own type pointer, refcount, and value, scattered across the heap) versus NumPy's contiguous buffer of raw, unboxed machine doubles. NumPy's vectorized operations move the entire loop into compiled C code operating on that flat buffer, paying neither cost. "NumPy is just faster" is an incomplete answer; naming both mechanisms is the mid/senior-level bar.

**Q3. A loop appending lambdas all return the same final value when called later. Why, and what are two fixes?**
```python
funcs = [lambda: i for i in range(3)]
print([f() for f in funcs])   # [2, 2, 2], not [0, 1, 2]
```
All three lambdas close over the *same* variable `i` by reference, not by value at creation time — `for` loops don't create a new scope per iteration (unlike modern JS `let`), so there's only ever one `i`, which is `2` by the time any lambda actually runs. **Fix 1:** default-argument capture — `lambda i=i: i` — forces evaluation of the current `i` at lambda-creation time, since default values are evaluated once, immediately, per Q2's mechanism from the junior round. **Fix 2:** a helper function returning a fresh closure per call — `def make_fn(i): return lambda: i` — each invocation creates its own independent local `i`.

**Q4. What happens to a generator function's local variables when it's paused at `yield`?**
Unlike a normal function (whose frame is discarded on `return`), a generator's frame is kept alive, suspended, referenced by the generator object itself — all local variables, and the exact bytecode instruction pointer position, are preserved exactly as they were. The next `next()` call resumes execution right after the `yield`, with all state intact. This is genuinely different machinery from a normal function call, which is why suspended generators have a real, non-trivial memory footprint tied to how much local state they're holding, not to how much data they'll eventually produce.

**Q5. Why does defining `__eq__` without `__hash__` break using instances as dict keys?**
By default, `object.__hash__` is identity-based, matching `object.__eq__`'s default identity comparison — hash equality and value equality agree. Once you override `__eq__` to mean structural/value equality, Python automatically sets `__hash__ = None` on that class (making instances unhashable) unless you also explicitly define `__hash__`, because leaving the old identity-based hash in place would violate the fundamental hashing invariant: two objects that compare equal *must* produce the same hash, or dict/set lookups break silently. Fix: implement `__hash__` consistently with your `__eq__` (typically hashing the same fields you compare).

**Q6. `threading` or `multiprocessing` for a task involving a slow network call followed by heavy in-memory computation on the response?**
Neither alone is fully correct — the right answer identifies that the task has two distinct phases with different bottleneck types. The network wait is I/O-bound: the GIL releases during it, so `threading` (or `asyncio`) gives genuine concurrency benefit while waiting. The heavy computation afterward is CPU-bound pure-Python work: threading gives **no** real parallelism there (GIL-bound), so if that computation is substantial, it belongs in a `multiprocessing.Pool` instead. A well-designed pipeline might use async/threaded I/O to gather responses concurrently, then hand the heavy computation off to worker processes — naming this two-phase split, rather than picking one tool for the whole task, is the actual mid-to-senior signal here.

---

## 19.4 Senior-Level Round — Design and Trade-Off Questions

**Q1. Design a caching layer for expensive per-user recommendations, called frequently for the same users, memory-constrained, with a hard 10-minute staleness limit.**
**Answer, with reasoning:** `functools.lru_cache` is the first instinct, but it has two disqualifying gaps here: no built-in TTL (a cached value is only evicted by LRU pressure or `maxsize`, never by *age*), and `maxsize=None` (unbounded) would violate the memory constraint while any fixed `maxsize` risks evicting a hot user's entry based on recency alone rather than genuine staleness. `weakref.WeakValueDictionary` solves memory pressure (entries can be reclaimed under pressure) but still has no TTL concept — a stale-but-still-referenced entry would never expire on its own. The correct design layers these ideas: a bounded-size dict (or LRU structure) storing `(value, timestamp)` pairs, checked on read (`if now - timestamp > 600: recompute`), combined with a size cap and eviction policy for the memory constraint — or, in production, reaching for a purpose-built TTL cache library (or Redis with a native `EXPIRE`) rather than hand-rolling one, once the requirements are this specific. The senior signal is recognizing *why* the two obvious stdlib tools each fall short, not just naming a third-party library.

**Q2. A production async service occasionally hangs completely under load — diagnostic process?**
Start from the symptom shape: "hangs *completely*," not "gets slow," strongly suggests the single-threaded `asyncio` event loop itself is blocked, not merely overloaded — a genuinely overloaded but non-blocked event loop degrades gradually rather than freezing entirely. Diagnostic steps, in order: (1) check for any synchronous/blocking call inside `async def` code — a synchronous DB driver, `requests` instead of an async HTTP client, `time.sleep` instead of `asyncio.sleep`, or even accidental heavy synchronous computation — since any of these stalls *every* concurrent coroutine, not just the one that made the call; (2) enable `asyncio`'s debug mode (`asyncio.run(main(), debug=True)`), which specifically warns about callbacks/coroutines that take too long, surfacing exactly this class of bug; (3) if no blocking call is found, profile for a genuinely CPU-heavy async handler starving the loop's ability to schedule other coroutines. Naming the debug-mode tool specifically, not just "look for blocking calls," is what distinguishes a senior answer from a mid-level one that already knows the trap exists.

**Q3. Propose an incremental type-coverage migration strategy for a large untyped codebase, and state honestly what it will and won't guarantee.**
**Strategy:** start with `mypy`/`pyright` in a lenient/gradual mode (allowing untyped code to pass) rather than `--strict` immediately; type new code and heavily-modified code as a matter of course going forward; prioritize typing the most-imported/most-central modules first, since their types propagate the most inference value outward; ratchet strictness incrementally (module by module, or via a CI check that only fails on *newly introduced* untyped code, never blocking on pre-existing debt all at once). **Honest limits, stated explicitly rather than oversold:** even at full, strict coverage, Python types are erased at runtime and provide zero enforcement outside the checker — any code path the checker didn't actually run (skipped CI, `# type: ignore`, an untyped third-party dependency defaulting to `Any`) can still ship type-unsafe behavior into production. This is a structurally weaker guarantee than TypeScript's compile-gate, and a senior answer says so plainly rather than implying "full type coverage" is equivalent to TS-level safety.

**Q4. Multiple inheritance vs. composition vs. `Protocol` for a plugin system with shared utility behavior but very different core responsibilities?**
There's no single right answer here — the trade-off itself is the point. Multiple inheritance (small, focused mixins providing exactly one capability each, e.g. `LoggableMixin`, `SerializableMixin`) works well when the shared behaviors are genuinely orthogonal and the mixins were designed from the start to be combined — but MRO/C3-linearization complexity grows fast as more mixins combine, and it's a real, documented source of hard-to-debug method resolution surprises at scale. Composition (each plugin *holds* a logger/serializer instance rather than inheriting one) is generally the safer, more testable default, especially as the number of shared behaviors grows, at some cost in call-site verbosity (`self.logger.log(...)` vs. just `self.log(...)`). `Protocol` solves a different problem entirely — it doesn't provide shared *implementation*, only a static guarantee that a plugin's *shape* matches what the system expects (`def run(config: dict) -> Result`), with zero required inheritance relationship, which is genuinely the right tool when plugins are written by separate, decoupled teams/packages that shouldn't need to share a common base class at all. A strong answer picks a combination (e.g., `Protocol` for the plugin contract itself, composition for shared utility behavior) and explains why each piece was chosen for its specific job, rather than defending one tool as universally correct.

---

## 19.5 FAANG-Style / Deep-Systems Round

**Q1. End to end: what happens from `python app.py` to your first business-logic line executing?**
The interpreter starts and initializes core built-in types and the `sys`/`builtins` modules. Your script's source is tokenized, parsed into an AST (via the PEG parser since 3.9), and compiled into a code object — bytecode plus metadata (constants, variable names). That code object executes inside the `__main__` module's namespace, top to bottom, in program order — every `def`/`class`/top-level statement runs exactly once, at this point (no hoisting of any kind — a name only exists once its binding statement has actually executed). Any `import` statement encountered triggers the import system: check `sys.modules` for a cached copy first; if absent, locate the module via `sys.path`, execute *its* top-level code once inside its own fresh namespace, and cache the result — so a module imported from multiple places only ever executes once per process. Throughout all of this, every object created is being reference-counted in real time (Module 14) — assignment, argument passing, and container insertion all adjust `ob_refcnt`, with immediate, deterministic deallocation the instant a count reaches zero, backstopped by a separate periodic cycle-detecting collector for reference cycles refcounting alone can't resolve. By the time your first business-logic line runs, you're executing bytecode instructions one opcode at a time through CPython's C evaluation loop — with no JIT compiling any of it to native machine code along the way, by default.

**Q2. Design a memory-efficient, genuinely parallel pipeline processing a 100GB file, applying a CPU-heavy pure-Python transformation per record, writing results to a database.**
**Read stage:** never load the file into memory at once — iterate it as a generator (`for line in f:`), since file objects are themselves lazy iterators; this keeps read-stage memory roughly constant regardless of file size. **Transform stage:** this is explicitly CPU-bound *pure-Python* work, meaning threading provides no real parallelism here due to the GIL — route batches of records through a `multiprocessing.Pool`, chosen specifically over threading because the bottleneck is computation, not I/O. Batch size matters: too small and pickling/IPC overhead per batch dominates the actual work; too large and you lose responsiveness/load-balancing across workers — this needs empirical tuning, not a guessed constant. **Write stage:** likely I/O-bound (network round-trips to the database) — if the DB driver supports it, an async client with batched writes (`asyncio.gather` over grouped inserts) or a thread pool for a synchronous driver both give real concurrency benefit here, since the GIL releases during the blocking DB call either way. **Memory profile across the whole pipeline:** should stay roughly constant, not proportional to the 100GB input, as long as generators are used consistently at the read stage and results are streamed to the write stage rather than accumulated in one giant in-memory list — the moment any stage does `list(everything)` before proceeding, that guarantee breaks. A comprehensive answer explicitly names all three stages, the specific tool for each, and the reasoning for why that tool fits that stage's bottleneck type — not just "use multiprocessing and generators" as a blanket statement.

**Q3. A junior engineer proposes globally disabling the garbage collector (`gc.disable()`) for a "quick performance win." Evaluate precisely.**
This is not simply wrong — it's workload-dependent, and the correct answer says so explicitly rather than giving a flat yes/no. Reference counting (Module 14.1) handles the overwhelming majority of deallocation on its own, immediately and deterministically, with zero dependency on the cycle-detecting `gc` module; disabling `gc` genuinely does remove real, measurable overhead (periodic scanning cost) for workloads that create few or no reference cycles. The real risk: any genuine reference cycles in the codebase — common in bidirectional object graphs (parent/child references), some closure patterns, and certain framework internals — will now **never** be collected, leaking memory unboundedly for the entire life of the process, since refcounting alone structurally cannot free them (each object in the cycle holds a nonzero refcount from the other, forever). The professional recommendation: never disable it *globally* without first profiling to confirm the cycle collector is a measurable, real bottleneck for this specific workload, and consider scoped disabling (`gc.disable()` around a known cycle-free hot section, followed by `gc.collect()` and re-enabling afterward) over an unconditional global change — a pattern actually used in some real production batch-processing code, but always deliberately and locally, never blindly.

---

## 19.6 Code Review Exercise — Full Answer Key

```python
import time

def fetch_all_users(user_ids, cache={}):
    results = []
    for uid in user_ids:
        if uid in cache:
            results.append(cache[uid])
        else:
            data = requests.get(f"https://api.example.com/users/{uid}").json()
            cache[uid] = data
            results.append(data)
    return results

async def handle_request(user_ids):
    users = fetch_all_users(user_ids)
    print(f"fetched {len(users)} users")
    return users
```

**Issue 1 — Mutable default argument (`cache={}`).** One shared dict object, created once at function-definition time, persists and grows across every call for the life of the process — never explicitly passed by any caller here, so it's silently accumulating unboundedly. Beyond the correctness concern (Module 1/4), this is also an unbounded memory-growth issue (Module 14) with no eviction or TTL.

**Issue 2 — Blocking synchronous call inside an async code path.** `fetch_all_users` uses synchronous `requests.get`, and is called directly (without `await`, and it isn't even genuinely awaitable since it's a plain sync function) from inside `async def handle_request`. This blocks the entire single-threaded event loop for the full duration of every network call, stalling every other concurrent coroutine in the service — the defining real-world `asyncio` production bug (Module 13.4). The `async def` on `handle_request` provides zero actual concurrency benefit as written.

**Issue 3 — `print()` in production request-handling code.** Should be a proper `logging` call with an appropriate level, giving filterable, structured, production-appropriate output instead of an unconditional stdout write (Module 18.1).

**Issue 4 — No error handling around the network call.** A single failed request — `requests.get` raising a connection error, or a non-200 response whose `.json()` call fails — propagates as an unhandled exception all the way up through `handle_request`, with no deliberate handling, logging, or graceful degradation (Module 10).

**Issue 5 (bonus, more subtle) — Sequential fetching of independent requests.** Each `user_id` triggers an independent, unrelated network call, fetched one at a time in a loop — a strong real-world design would fetch them concurrently (an async HTTP client with `asyncio.gather`, or a thread pool) rather than paying the full network latency of each call serially, which compounds badly as `user_ids` grows (Module 13).

**A corrected sketch**, addressing all five:
```python
import logging
import httpx

logger = logging.getLogger(__name__)

async def fetch_all_users(user_ids, cache):
    async def fetch_one(client, uid):
        if uid in cache:
            return cache[uid]
        try:
            resp = await client.get(f"https://api.example.com/users/{uid}")
            resp.raise_for_status()
            data = resp.json()
            cache[uid] = data
            return data
        except httpx.HTTPError:
            logger.error("failed to fetch user %s", uid, exc_info=True)
            return None

    async with httpx.AsyncClient() as client:
        return await asyncio.gather(*(fetch_one(client, uid) for uid in user_ids))

async def handle_request(user_ids, cache):        # cache passed in explicitly, not defaulted
    users = await fetch_all_users(user_ids, cache)
    logger.info("fetched %s users", len(users))
    return users
```

---

## 19.7 Closing — What "Mastery" Actually Looks Like From Here

The material in Modules 0–18, plus this reference, is genuinely complete for interview and production purposes at a senior level — but real mastery comes from two things a course can't do for you: **reading substantial real-world codebases** (`requests`, `flask`, or `httpx` are all genuinely well-written, readable places to start) to see these patterns used in context, and **writing enough production code that the trade-offs in Modules 13–15 and 18 stop being abstract and start being things you've personally been burned by once.** Come back to the questions in this module periodically — the ones that feel easy on a second pass are the ones you've actually internalized; the ones that still require real thought are exactly where to spend more deliberate practice time.

---

*Course complete. All 20 files (Modules 0–19) are saved as individual markdown files, each usable as a standalone reference. Let me know if you'd like a condensed one-page cheat sheet distilling the highest-density facts across every module, or want to go deeper on any specific module with additional worked problems.*
