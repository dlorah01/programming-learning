---
layout: default
title: "Chapter 7 — Memory"
---

# Chapter 7 — Memory

> Depends on: Ch.2 §2.5 (primitives copy by value, objects by reference), Ch.3 §3.4 (closures keep variables alive).
> Feeds into: Ch.11 (performance — GC pauses, memoization cache growth).

---

## 7.1 Stack vs. Heap

**Intuitive version:** The **stack** is a small, fast, orderly region of memory for simple, fixed-size data and function call bookkeeping. The **heap** is a large, less structured region for complex, variable-sized data — objects, arrays, functions, closures.

**Formal version:** primitive values are typically stored directly in the stack frame (or inlined by the engine); objects are allocated in the heap, and the stack/variable only holds a **reference (pointer)** to that heap location. This directly explains Ch.2's "primitives copy by value, objects copy by reference" rule — you're not copying the object itself, you're copying the pointer to it.

```
STACK (per function call, LIFO, fast, size known at compile time)
┌─────────────────┐
│ frame: main()     │  x = 5 (primitive, inline)
│                    │  obj = 0x1A2B (pointer to heap)
├─────────────────┤
│ frame: foo()       │  ...
└─────────────────┘

HEAP (large, unordered, garbage-collected)
┌─────────────────────────────┐
│ 0x1A2B: { name: "Ana", ... }   │
└─────────────────────────────┘
```

**Why this split exists:** the stack's fixed-size, last-in-first-out structure makes allocation/deallocation essentially free (just move a pointer) — but it can only hold data whose size is known ahead of time and that has a clear, short lifetime (a function call). The heap trades that speed for flexibility: arbitrary-sized, arbitrarily long-lived data, at the cost of needing a garbage collector to reclaim it.

**Interview questions:**
- *Beginner:* Where are primitive values typically stored vs. objects?
- *Mid:* Explain why `function mutate(obj) { obj.x = 1 }` affects the caller's object, but `function mutate(num) { num = 1 }` doesn't affect the caller's number.

---

## 7.2 Garbage Collection

**Intuitive version:** JS automatically frees memory that's no longer reachable from your running code — you never manually call `free()` like in C.

**Formal version — the core algorithm modern engines use: Mark-and-Sweep.**
1. Starting from a set of **roots** (global object, currently executing functions' local variables, closures still referenced), the GC walks every reachable reference, **marking** each object it finds.
2. After the walk, anything **not marked** is considered garbage — unreachable — and is **swept** (its memory reclaimed).

**Why "reachability," not "reference counting," is the model:** an older, simpler GC algorithm (**reference counting** — track how many references point to each object, free it when the count hits 0) has a fatal flaw: **circular references** never hit zero, causing leaks.
```js
function leak() {
  const a = {};
  const b = {};
  a.ref = b;
  b.ref = a; // circular reference
} // under naive reference counting, neither a nor b would ever be freed, even though
  // NOTHING outside this function can reach them anymore once it returns
```
Mark-and-sweep solves this correctly: after `leak()` returns, neither `a` nor `b` is reachable from any root, so both get marked as garbage and swept — regardless of the fact that they reference each other. **V8 uses a generational, mark-and-sweep-based collector** (young/old generation splits, since most objects die young — this is called the "generational hypothesis").

**Generational GC (senior-level detail):** V8 splits the heap into a small **young generation** (new objects, collected frequently with a fast algorithm called Scavenge) and a larger **old generation** (long-lived objects that survived several young-gen collections, collected less often with a more thorough mark-sweep-compact pass). This exploits the empirical observation that most objects are short-lived (e.g., a temporary object inside a function call), so it's efficient to check the small young space often and the large old space rarely.

**Interview questions:**
- *Mid:* What is "reachability" and how does it differ from simple reference counting?
- *Senior:* Why does circular reference NOT cause a memory leak in modern JS engines?
- *Senior/FAANG:* Explain the generational hypothesis and why V8 splits its heap into young and old generations.

---

## 7.3 Memory Leaks in JavaScript (despite having a GC!)

**Intuitive version:** "Automatic garbage collection" doesn't mean "impossible to leak memory" — it means the GC will always correctly free anything *unreachable*. If your code accidentally keeps something reachable that it doesn't actually need anymore, the GC has no way to know that and will never free it. Leaks in JS are almost always "accidental reachability," not a GC bug.

**Formal version — the four classic categories:**

**1. Accidental global variables:**
```js
function leaky() { accidentalGlobal = new Array(1e6).fill("x"); } // no declaration — Ch.2 §2.2 implicit global
// lives for the entire page/process lifetime, never eligible for GC
```

**2. Forgotten timers/intervals holding closures:**
```js
function startPolling(largeData) {
  setInterval(() => {
    console.log(largeData.length); // closure keeps largeData alive FOREVER, even if no longer needed
  }, 1000);
  // no clearInterval ever called — the interval (and its closure) never gets cleaned up
}
```

**3. Detached DOM references:**
```js
let detachedNode;
function cacheNode() {
  const el = document.getElementById("large-list");
  detachedNode = el; // cached reference
  el.remove();          // removed from the DOM...
  // ...but `detachedNode` still references it, so the entire subtree stays in memory,
  // "detached" from the visible page but NOT garbage collected
}
```

**4. Long-lived closures capturing more than they need (ties directly to Ch.3 §3.4):**
```js
function setup() {
  const hugeArray = new Array(1e6).fill("data");
  const smallValue = 42;
  return function () {
    return smallValue; // only uses smallValue...
    // ...but if hugeArray is in the SAME closure scope, some engines keep the whole
    // scope alive, not just the specific variable used — depends on engine optimization
  };
}
```

**Mental model:** think of the GC as a diligent librarian who will remove any book from the shelf the instant nobody holds a claim ticket for it — but if you keep a claim ticket in your pocket for a book you forgot about, the librarian has no way to know you don't actually want it anymore. The "leak" is always on the holding side, not the librarian's side.

**Best practices to avoid leaks:** always pair `setInterval`/`addEventListener` with corresponding `clearInterval`/`removeEventListener` when a component/module is torn down; avoid unintentional global variables (use strict mode, Ch.2); null out large cached references you no longer need; use WeakMap/WeakSet for caches keyed by objects (§7.4).

**Interview questions:**
- *Mid:* Name three common causes of memory leaks in JS despite having automatic GC.
- *Senior:* Explain "detached DOM nodes" and why DevTools' heap snapshot tool is used to diagnose them.
- *FAANG-style debugging:* given a single-page app that gets progressively slower the longer it's left open, walk through your diagnostic process (heap snapshots, comparing two snapshots over time, looking for detached nodes / growing retained size on specific constructors).

---

## 7.4 `WeakMap` and `WeakSet`

**Intuitive version:** Regular `Map`/`Set` hold **strong** references to their keys/values — meaning even if nothing else in your program references an object, storing it in a `Map` keeps it alive forever. `WeakMap`/`WeakSet` hold **weak** references — they don't prevent garbage collection, so entries automatically disappear once their key object is no longer referenced anywhere else.

**Formal version:**
```js
let obj = { id: 1 };
const map = new Map();
map.set(obj, "metadata");
obj = null;
// the object is STILL alive — Map holds a strong reference, this is a leak if unintended

let obj2 = { id: 2 };
const weakMap = new WeakMap();
weakMap.set(obj2, "metadata");
obj2 = null;
// the object becomes eligible for GC — WeakMap does NOT keep it alive
```

**Key restrictions (both by design, tied directly to the "weak" semantics):**
- Keys must be objects (or, since ES2023, registered Symbols) — never primitives, since primitives aren't garbage-collected heap entities in the same way.
- **Not iterable** — no `.keys()`, `.forEach()`, no way to list entries. This is deliberate: if you could enumerate entries, you could accidentally keep a reference to a key alive just by iterating, defeating the entire purpose. It also means the exact GC timing (which is unpredictable) never becomes observable behavior.

**Real-world use case:** associating private/extra metadata with an object without preventing that object from being garbage collected when the rest of the program is done with it — e.g., a WeakMap-based cache of computed results per DOM node, that automatically cleans itself up when a node is removed and no longer referenced.

```js
const cache = new WeakMap();
function computeExpensive(el) {
  if (cache.has(el)) return cache.get(el);
  const result = expensiveComputation(el);
  cache.set(el, result);
  return result;
}
// if `el` is later removed from the DOM and has no other references, the cache entry
// is automatically freed too — a regular Map would leak these forever
```

**Interview questions:**
- *Mid:* Why can't you iterate over a `WeakMap`?
- *Senior:* Give a real scenario where using `Map` instead of `WeakMap` for a cache causes a memory leak, and explain the fix.

---

## Practical exercises — Chapter 7

1. Explain why the circular-reference example in §7.2 does NOT leak in a modern JS engine, but WOULD leak under naive reference counting.
2. **Debugging exercise:** given the "forgotten interval" leak example (§7.3, case 2), rewrite `startPolling` to properly clean up, returning a `stop()` function the caller must invoke.
3. **Small challenge:** implement a WeakMap-based memoization cache for a function that takes a single object argument, so cached results are automatically cleaned up when the input object is no longer referenced elsewhere.
4. **Advanced challenge:** describe, step by step, how you would use Chrome DevTools' Memory tab to confirm a suspected leak in a long-running single-page app (heap snapshot comparison, detached node search, allocation timeline) — a very common senior/staff-level practical interview question.

---

**Next:** Chapter 8 — Modules (CommonJS vs. ES Modules, dynamic imports, tree shaking). Say **"next"** when ready.
