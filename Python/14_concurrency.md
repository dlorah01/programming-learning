---
layout: default
title: "Module 13: Concurrency — GIL, Threading, Multiprocessing, Asyncio"
---

# Python Mastery Course — Module 13: Concurrency — GIL, Threading, Multiprocessing, Asyncio

---

## 13.1 The GIL — What It Actually Is, Precisely

**The claim to get exactly right (a very commonly garbled interview answer):** the Global Interpreter Lock is a single mutex, internal to CPython, that ensures **only one thread executes Python bytecode at a time**, even on a multi-core machine, even with multiple OS-level threads created. It exists specifically because CPython's memory management — reference counting (Module 14) — is **not thread-safe** by default; without the GIL, two threads simultaneously incrementing/decrementing the same object's refcount could race and corrupt it, causing memory leaks or premature frees/crashes. The GIL is a deliberate, historically pragmatic trade: it made the CPython interpreter itself simple and fast for the single-threaded case (the overwhelmingly common case in the 1990s when this design was made), at the cost of true multi-core parallelism for pure-Python code.

**What the GIL does NOT prevent — a genuine, common misconception to correct precisely:** it does not prevent you from *creating* multiple threads, and it does not prevent concurrency generally — it specifically prevents **CPU-bound Python bytecode from running in true parallel across cores**. Threads still provide real, genuine concurrency benefit for **I/O-bound** work, because the GIL is explicitly **released** during blocking I/O operations (network calls, file reads, `time.sleep`) and during many C-extension operations (NumPy's heavy-lifting C code releases the GIL internally, which is exactly why NumPy/pandas can still get real multi-core speedups despite the GIL — the actual number-crunching happens outside the GIL's reach).

**The concrete rule that resolves most "should I use threading or multiprocessing" questions instantly:**

| Workload | Right tool | Why |
|---|---|---|
| I/O-bound (network calls, file/db reads, waiting on external services) | `threading` or `asyncio` | GIL is released during the wait; multiple threads/coroutines genuinely overlap useful work while waiting |
| CPU-bound (heavy computation, tight pure-Python loops) | `multiprocessing` | Only real escape from the GIL — separate OS processes, each with its own interpreter and GIL, genuinely parallel across cores |
| CPU-bound, but the heavy lifting is inside NumPy/pandas/a C extension | `threading` can actually work fine here too | the GIL is released inside the C code doing the real work |

**JS/Node comparison, a genuinely useful framing:** Node's single-threaded event loop is conceptually closer to Python's GIL-constrained situation than people initially assume — Node also can't run your JS callback code across multiple cores without explicitly spinning up Worker Threads or child processes, for very similar underlying reasons (JS's memory model also assumes single-threaded execution of your code by default). The real difference is that Node's I/O model (the event loop + libuv) is baked in from the start as the *default* way you write async code, while Python's evolved: `threading` (real OS threads, GIL-limited) came first historically, and `asyncio`'s single-threaded cooperative event loop — Python's actual Node-style answer — arrived much later (3.4+, with `async`/`await` syntax formalized in 3.5) as a *deliberate alternative* to threading for I/O-bound concurrency, not a replacement for it.

**Historical/future context, worth knowing precisely:** Python 3.13 introduced an **experimental, opt-in "free-threaded" build** (PEP 703, no-GIL) — a genuinely major, years-in-the-making CPython engineering effort — but it is not the default build as of the versions you'll be deploying, and enabling it currently costs meaningful single-threaded performance and breaks GIL-reliant C extensions until they're updated. Mention this if asked "is the GIL going away" — the honest, current answer is "there's real, serious movement, but it's not the production default yet," not a flat "no" or an overclaimed "yes, solved."

---

## 13.2 `threading` — Real OS Threads, GIL-Limited

```python
import threading, requests

def fetch(url, results, i):
    results[i] = requests.get(url).status_code

urls = ["https://example.com"] * 5
results = [None] * 5
threads = [threading.Thread(target=fetch, args=(url, results, i)) for i, url in enumerate(urls)]
for t in threads: t.start()
for t in threads: t.join()     # blocks until all threads complete
print(results)
```

This genuinely overlaps 5 network waits — real wall-clock speedup over doing them sequentially — because the GIL releases during each blocking `requests.get()` call, letting another thread run while the first waits on the network.

**Race conditions — still fully possible despite the GIL, a genuine, common misconception to correct:** the GIL prevents bytecode-level corruption of a *single* operation, but does **not** make compound operations atomic. `counter += 1` is multiple bytecode operations (load, add, store) — a thread switch can happen *between* them:

```python
counter = 0
def increment():
    global counter
    for _ in range(100_000):
        counter += 1      # NOT atomic — a thread switch mid-increment can lose updates

threads = [threading.Thread(target=increment) for _ in range(4)]
for t in threads: t.start()
for t in threads: t.join()
print(counter)   # frequently LESS than 400,000 — a real, reproducible race condition
```

**The fix — `threading.Lock`, same concept as any other language's mutex:**

```python
lock = threading.Lock()
def increment_safe():
    global counter
    for _ in range(100_000):
        with lock:              # the context-manager protocol from Module 10, applied to real concurrency
            counter += 1
```

"The GIL means I don't need to worry about thread safety" is a genuinely common, genuinely wrong belief — a fair, common interview trap specifically because it sounds plausible.

---

## 13.3 `multiprocessing` — Real Parallelism, Separate Processes

```python
from multiprocessing import Pool

def cpu_heavy(n):
    return sum(i * i for i in range(n))

if __name__ == "__main__":          # REQUIRED on some platforms — see below
    with Pool(processes=4) as pool:
        results = pool.map(cpu_heavy, [10_000_000] * 4)   # genuinely runs on 4 separate cores
```

Each worker is a **separate OS process** with its own Python interpreter, its own GIL, its own memory space — true parallelism across cores for CPU-bound work, at the real cost of: (1) **no shared memory by default** — data passed to/from worker processes is *pickled* (serialized) and sent over an IPC channel, real overhead that can dominate for small, fast tasks; (2) genuinely higher startup cost per worker than a thread; (3) needing explicit tools (`multiprocessing.Value`, `Manager`, shared memory buffers) for any actual shared state, since processes don't share Python objects the way threads do.

**Why `if __name__ == "__main__":` (Module 12.3) is often flagged as *required* here, specifically on Windows and with the `spawn` start method (the default on macOS since 3.8):** creating a new process involves **re-importing** the main script fresh in the child process (since there's no `fork`-based memory copy in `spawn` mode) — without the guard, the child process would recursively try to spawn its own pool of children, infinitely. This is a genuine, very common `multiprocessing` beginner bug, and directly reuses Module 12's `__name__` mechanics in a new, concrete context.

**When it's worth the overhead — the real judgment call:** multiprocessing wins when per-task computation is substantial relative to the pickling/IPC cost (large batch numeric computation, image processing per-file, independent heavy simulations) — for many small, fast tasks, the process/IPC overhead can genuinely make multiprocessing *slower* than a plain single-threaded loop, a real, common performance-tuning gotcha worth testing empirically rather than assuming.

---

## 13.4 `asyncio` — Cooperative Single-Threaded Concurrency

**Intuition, direct JS bridge:** this is Python's actual Node-style answer — a single-threaded event loop, cooperative multitasking, `async def`/`await` syntax that will look immediately familiar if you know JS's `async`/`await`. The core difference from JS: Python's cooperative scheduling is **fully explicit** — a coroutine only yields control at an `await` point, never implicitly/preemptively (identical philosophy to JS's event loop, genuinely, not a divergence here — both are single-threaded and cooperative).

```python
import asyncio

async def fetch(url):
    print(f"starting {url}")
    await asyncio.sleep(1)     # non-blocking "sleep" — yields control back to the event loop
    print(f"done {url}")
    return f"result for {url}"

async def main():
    results = await asyncio.gather(     # runs all three CONCURRENTLY, not sequentially
        fetch("a"), fetch("b"), fetch("c")
    )
    print(results)

asyncio.run(main())    # total time ~1 second, not ~3 — genuine concurrency, single thread
```

**Mechanically, connecting directly to Module 6's generators:** an `async def` function is, at a deep implementation level, a specialized coroutine object whose suspension/resumption machinery is directly descended from generators' `yield`-based suspension (Module 6.6's `send`/`throw` mechanics) — `await` on a coroutine or Future essentially says "suspend me here, hand control back to the event loop, resume me when this result is ready," architecturally the same shape as Module 6.4's `yield from` delegating into a sub-generator, formalized into dedicated syntax with a dedicated event-loop scheduler running everything.

**The single most important rule for using `asyncio` correctly, and a real, common production bug:** `await` only actually yields control at genuinely `await`-able points — calling a **blocking**, non-async function (a synchronous `requests.get()`, a synchronous heavy computation, `time.sleep()` instead of `asyncio.sleep()`) *inside* an `async def` function **blocks the entire single-threaded event loop**, stalling every other concurrent coroutine, silently defeating the entire point:

```python
async def bad_fetch(url):
    return requests.get(url)      # BLOCKS the whole event loop — no other coroutine can run
                                    # meanwhile, even though this LOOKS like it's inside async code

async def good_fetch(url, session):
    async with session.get(url) as resp:     # aiohttp — a genuinely async-native HTTP client
        return await resp.text()
```

This is *the* defining "does this person actually understand asyncio" interview/code-review checkpoint — using a synchronous, blocking library inside `async def` code is a very common, very real production performance bug, because it looks correct (it runs, it returns a value) while silently eliminating all the concurrency you were trying to gain.

**When to use `asyncio` vs. `threading` for I/O-bound work — a genuine, fair design question:** both work for I/O concurrency; `asyncio` generally scales to far more concurrent operations with less overhead (thousands of concurrent coroutines are cheap; thousands of OS threads are not — real memory/scheduling cost per thread), but requires your entire I/O-touching dependency chain to be async-native (async DB drivers, async HTTP clients) to avoid the blocking-call trap above — `threading` is often the pragmatic choice when you're integrating with existing synchronous libraries you don't control and can't easily replace.

---

## 13.5 Concurrency vs. Parallelism — The Precise Distinction, Worth Stating Cleanly

**Concurrency**: structuring a program to *deal with* multiple things happening in overlapping time periods — doesn't require multiple cores, doesn't even require multiple threads (`asyncio` achieves genuine concurrency on a single thread/core). **Parallelism**: multiple things *literally* executing at the same instant, requiring multiple cores/processors. `threading` in CPython gives you concurrency for I/O-bound work but **not** true parallelism for CPU-bound work (the GIL, §13.1); `multiprocessing` gives you genuine parallelism; `asyncio` gives you concurrency, explicitly not parallelism, by design, on one thread. Being able to state this distinction crisply, and correctly place each of Python's three concurrency tools on it, is one of the most reliable senior-level differentiators in this entire module.

---

## 13.6 Common Mistakes / Interview Traps — Consolidated

- Claiming the GIL prevents race conditions entirely — it doesn't; compound operations like `+=` still need explicit locking (§13.2).
- Believing threading gives real parallel speedup for CPU-bound pure-Python work — it doesn't, use `multiprocessing`.
- Calling a blocking/synchronous function inside `async def` code, silently stalling the entire event loop (§13.4) — the single most common real asyncio production bug.
- Forgetting `if __name__ == "__main__":` with `multiprocessing` on `spawn`-based platforms, causing runaway recursive process creation.
- Assuming multiprocessing is always faster for CPU-bound work without considering pickling/IPC overhead for small, fast, numerous tasks.
- Conflating "concurrent" and "parallel" as interchangeable terms in an interview answer.

---

## 13.7 Exercises

**Predict the output:**
```python
import asyncio
async def task(n):
    await asyncio.sleep(n)
    print(f"task {n} done")

async def main():
    await asyncio.gather(task(2), task(1), task(3))

asyncio.run(main())
```
*(What order do the print statements appear in, and roughly how long does the whole thing take?)*

**Core:** Write a `threading`-based version and an `asyncio`-based version of a function that fetches 10 URLs concurrently, and explain which you'd choose for a service also using a synchronous ORM you don't control.

**Debugging exercise:** An `asyncio`-based web service's response times degrade badly under load despite using `async def` handlers throughout. Profiling shows a `time.sleep(0.1)` call buried in a "quick rate-limiting hack" inside one handler. Explain precisely why this single line degrades the *entire* service, not just that handler's requests.

**Interview — junior:** "What does the GIL actually prevent, precisely? What does it not prevent?"

**Interview — mid:** "You need to process 1,000 large images with a CPU-heavy filter. Would you use `threading` or `multiprocessing`, and why? What changes if the 'CPU-heavy filter' is actually a call into a NumPy/OpenCV function?"

**Interview — senior:** "Design the concurrency model for a service that needs to: (a) make hundreds of concurrent outbound API calls, (b) run a CPU-heavy validation step on each response, and (c) write results to a database. Justify which of `threading`/`multiprocessing`/`asyncio` (or a combination) fits each stage."

**Advanced / FAANG-style:** "Explain, precisely, why `counter += 1` performed by four threads in a loop can produce a final count less than the true total, given the GIL — walk through the actual bytecode-level interleaving that causes lost updates, referencing what you learned about bytecode in Module 0."

---

*Next: Module 14 — Memory: reference counting, cycle-detecting garbage collection, weak references, and concrete memory-optimization techniques (this connects directly back to `__slots__` from Module 7 and object identity from Module 1).*
