---
layout: default
title: "Exercise & Answer Key — Full Reference"
---

# Exercise & Answer Key — Full Reference

This compiles every "Practical exercises" section from Chapters 1–11, plus the Chapter 12 self-assessment, with full model answers. Every answer includes not just the result/code but the **mechanism** behind it — the goal is that you could explain each one to an interviewer without reciting memorized lines, because you understand *why* it's true.

Use it two ways:
1. **As a test:** cover the answer, attempt the exercise cold, then check yourself against both the result AND the reasoning.
2. **As a checklist:** the exercises themselves ARE the list of concepts you need to be fluent in — if any explanation surprises you, that's exactly where to re-read the source chapter (linked by §-section).

---

## Chapter 1 — Foundations

**1. Explain the full pipeline from `node app.js` to first output.**
> Node starts → V8 initializes → source is tokenized (lexed) into tokens, then parsed into an **AST** (Abstract Syntax Tree) representing the program's grammar. Before any line executes, V8 runs a **creation phase** over each scope, registering `var`/`function`/`let`/`const` declarations in memory ahead of time — this is the mechanical root of hoisting (Ch.2 §2.3). Then **Ignition** (V8's interpreter) compiles the AST to bytecode and starts executing it line by line immediately — deliberately prioritizing fast startup over raw execution speed. As the program runs, V8's profiler tracks which functions get called repeatedly with stable argument *shapes/types* ("hot" functions). Those get handed to **TurboFan**, the optimizing JIT compiler, which generates specialized machine code based on the type assumptions it observed. If a later call violates those assumptions (e.g., a function profiled as always receiving numbers suddenly gets a string), V8 **deoptimizes** — throws away the optimized code and falls back to Ignition, then reprofiles. Throughout, Node's host APIs (via libuv) handle anything asynchronous, and the **event loop** (Ch.6 §6.1) decides when queued callbacks run relative to this synchronous execution. Your first `console.log` prints the moment the interpreter's linear execution reaches that statement.
>
> *Why this matters:* this single pipeline is the mechanical justification for hoisting, for why JS is neither "purely interpreted" nor "purely compiled," and for why monomorphic code is faster (§1.3) — it's not three separate facts, it's one pipeline viewed from three angles.

**2. Three V8-specific reasons manual inlining could make code slower in production.**
> (a) **Polymorphism at the call site.** The original small function, called from many places, may have been **monomorphic** at each individual call site (always receiving the same shape of argument there), letting TurboFan generate a specialized fast path per site. Inlining it manually into one large function can merge previously-separate call sites into one, which now sees varied argument types — making it **polymorphic or megamorphic**, which forces V8 to fall back to slower, generic property/type lookups instead of specialized inline caches.
> (b) **Inlining-budget heuristics.** V8's own automatic inliner has size/complexity limits on what it will inline; a manually bloated function can exceed thresholds that would have let V8 make a *better* automatic inlining decision on the original, smaller function.
> (c) **Deoptimization risk surface.** A larger, manually-merged function has more code paths and more chances for one branch to violate a type assumption TurboFan made — triggering a deopt for the *entire* function rather than just a small, isolated piece.
> *Why:* this connects §1.3's inline-caching/monomorphism material to a realistic code-review scenario — "faster in isolation, slower in production" is almost always a call-site-shape problem, not a raw-instruction-count problem.

**3. Predict output order.**
```js
console.log("A"); setTimeout(() => console.log("B"), 0); console.log("C");
```
> **A, C, B.** Mechanism: `setTimeout`'s callback isn't run by the JS engine directly — it's handed off to the runtime's timer facility (a Web API / libuv timer), which, once the delay elapses, places the callback on the **macrotask queue**. The event loop only pulls from that queue when the **call stack is completely empty** (Ch.1 §1.7's run-to-completion rule). Since `console.log("C")` is still synchronous code sitting on the stack when the timer fires, it always finishes first — the `0ms` delay only guarantees "as soon as possible after the stack clears and any microtasks drain," never "immediately" or "before other sync code."

**4. V8 (Ignition→TurboFan) vs. SpiderMonkey (Baseline Interpreter→Baseline JIT→Ion) — shared philosophy.**
> Both use **tiered execution**: an initial fast-to-produce, slower-to-run tier (bytecode interpretation) for instant startup, followed by progressively more aggressive JIT tiers applied only to code proven "hot" by runtime profiling — because compiling *everything* ahead-of-time to maximally optimized code would make page load unacceptably slow, while interpreting *everything* forever would make hot loops unacceptably slow. The shared underlying principle: **spend optimization effort proportionally to how much a piece of code will actually run**, and always keep a safe, slower fallback (Ignition / Baseline) available for when the fast tier's type assumptions turn out wrong.

---

## Chapter 2 — Language Fundamentals

**1. TDZ shadowing trace.**
```js
let x = "outer";
function test() { console.log(x); let x = "inner"; }
test();
```
> **Throws `ReferenceError: Cannot access 'x' before initialization'`.**
> *Why:* during the creation phase of `test`'s function scope, the engine scans the entire function body and finds the `let x` declaration, registering the name `x` in that scope **immediately** — but leaves it in the **Temporal Dead Zone**, uninitialized, until the actual `let x = "inner"` line executes (Ch.2 §2.3–2.4). Because JS resolves `x` via the **nearest enclosing scope** in the scope chain (Ch.2 §2.2), and `test`'s own scope now has an `x` binding (even though it's not yet initialized), the lookup never even reaches the outer `x = "outer"`. It stops at the inner, TDZ-bound `x` and throws. This is the exact mechanism proving that TDZ isn't just "you can't use `let` before its line" — it's that the *name itself* shadows outer scopes for the entire containing block, dead-zone and all.

**2. `if (user.age || 0)` bug — explain and fix.**
> *Mechanism:* `||` evaluates its left operand's **truthiness** (Ch.2 §2.8) and falls through to the right operand for **any** falsy value — and `0` is one of the 8 falsy values. So a genuinely valid `age: 0` is indistinguishable, to `||`, from `age: null` or `age: undefined` — all three trigger the fallback. Fix: use `??`, which checks specifically for `null`/`undefined` (via the internal nullish-check, not general truthiness) and leaves every other value — including `0`, `""`, and `false` — untouched: `user.age ?? 0`.

**3. Implement `looseEquals` for numbers/strings/booleans/null/undefined.**
```js
function looseEquals(a, b) {
  if (a === null || a === undefined) return b === null || b === undefined;
  if (b === null || b === undefined) return false;
  if (typeof a === typeof b) return a === b;
  return Number(a) === Number(b); // covers number/string/boolean cross-type coercion
}
```
> *Why it's structured this way:* it mirrors the actual **Abstract Equality Comparison Algorithm** JS itself uses — `null`/`undefined` are handled as a special mutually-exclusive pair first (they equal each other and NOTHING else, not even `0` or `""`), same-type values compare directly with no coercion needed, and only genuinely mixed types fall through to numeric coercion (`ToNumber`), which is the common conversion target for string/boolean/number comparisons in the real spec.

**4. Five surprising `==` results, with the exact rule.**
> - `"" == 0` → `true` — `ToNumber("")` is `0` (empty string parses as numeric zero).
> - `[] == 0` → `true` — arrays coerce via `ToPrimitive`, which tries `valueOf()` first (returns the array itself, rejected since it's not primitive), then falls back to `toString()` → `""`, which then goes through the same `ToNumber("") === 0` path as above.
> - `"\t\n " == 0` → `true` — whitespace-only strings still parse to numeric `0` under `ToNumber`'s trimming rules.
> - `null == 0` → `false` — this is the one exception to "coerce toward numbers": the spec **hard-codes** that `null`/`undefined` are loosely equal *only* to each other and to nothing else, bypassing numeric coercion entirely, specifically to avoid nonsensical results like `null == 0` being true.
> - `[1] == 1` → `true` — `[1]` → `ToPrimitive` → `valueOf()` rejected → `toString()` → `"1"` → `ToNumber("1")` → `1`.
> *Common thread:* almost every "weird" `==` result traces back to the **same two-step process** — non-primitives get funneled through `ToPrimitive` (usually landing on their string form) before any numeric comparison happens.

---

## Chapter 3 — Functions

**1. `this` trace.**
```js
const obj = { name: "X", regular: function () { return this.name; }, arrow: () => this.name };
const { regular, arrow } = obj;
console.log(obj.regular(), obj.arrow(), regular(), arrow());
```
> `"X" undefined undefined undefined`.
> - `obj.regular()` → **implicit binding** (Ch.3 §3.2 rule 3): the object left of the dot at call time (`obj`) becomes `this` → `"X"`.
> - `obj.arrow()` → arrow functions have **no own `this`**; they lexically inherit whatever `this` was in scope at the moment the arrow was *defined* — here, the top level of the module/script, not `obj` — so implicit binding never applies, regardless of how it's called → `undefined`.
> - `regular()` (destructured off `obj`, then called plain) → the object context is lost entirely at the moment of destructuring; calling it as a bare `regular()` triggers **default binding** (rule 4), which is `undefined` in strict mode → `undefined`.
> - `arrow()` → same as `obj.arrow()` — its `this` was fixed lexically at definition and is completely unaffected by how or where it's later called → `undefined`.
> *Key insight this proves:* regular functions determine `this` from the **call site** (how you invoke them); arrow functions determine it from the **definition site** (where they're written) — these are fundamentally different resolution mechanisms, not just a stylistic difference.

**2. Fix `this.handleClick` losing context in a class component.**
> *Why it breaks:* `addEventListener("click", c.increment)` passes the **function value itself**, detached from `c` — when the browser later invokes it as a plain callback (`fn(event)`), that's a default-binding call (Ch.3 §3.2 rule 4), so `this` is `undefined`, not `c`.
> Fix 1 — class field arrow function: `handleClick = () => { ... }` — because it's an arrow, it captures `this` lexically from the surrounding constructor scope at the moment the instance is built, permanently, regardless of how it's later invoked.
> Fix 2 — explicit `bind` in the constructor: `this.handleClick = this.handleClick.bind(this)` — `bind` (Ch.3 §3.3) returns a *new* function with `this` permanently locked to the given value, which no later `call`/`apply`/implicit-binding attempt can override.

**3. Implement `debounce`.**
```js
function debounce(fn, delay) {
  let timer;
  return (...args) => { clearTimeout(timer); timer = setTimeout(() => fn(...args), delay); };
}
```
> *Mechanism:* the returned function is a **closure** (Ch.3 §3.4) over the `timer` variable — each invocation cancels any pending timer and schedules a fresh one. Because `timer` is captured by reference and shared across every call to the debounced function, rapid repeated calls keep resetting the same countdown instead of accumulating separate independent timers — `fn` only actually fires once activity has paused for the full `delay`.

**4. Curried, memoized `add(a)(b)(c)`.**
```js
function curry(fn) {
  return function curried(...args) {
    return args.length >= fn.length ? fn(...args) : (...more) => curried(...args, ...more);
  };
}
function memoize(fn) {
  const cache = new Map();
  return (...args) => {
    const key = JSON.stringify(args);
    if (!cache.has(key)) cache.set(key, fn(...args));
    return cache.get(key);
  };
}
const add = curry(memoize((a, b, c) => a + b + c));
add(1)(2)(3); // 6, cached on subsequent identical calls
```
> *Why `curry` works:* `fn.length` reports the function's declared arity (number of named parameters) — `curried` keeps accumulating arguments via closures until it has at least that many, at which point it finally invokes the original function. Each partial call (`curried(...args, ...more)`) is itself a new closure remembering everything accumulated so far — this is the same closure mechanism from §3.4, just chained. `memoize` wraps the *innermost* function so that once all three arguments are known, repeated identical full calls hit the cache instead of recomputing.

---

## Chapter 4 — Objects & Prototypes

**1. Shared-array-on-prototype bug — fix.**
> *Why it happens:* `Bad.prototype.items = []` creates **one single array object** in memory, and every instance's `[[Prototype]]` link points to the *same* `Bad.prototype` object (Ch.4 §4.4). Since `items` is never redefined as an *own* property on any instance, every property access for `.items` walks the prototype chain and resolves to that one shared array — so `.push()` on any instance mutates the one array all instances see.
> Fix — move the initialization into the constructor, so each instance gets its own **own property**, shadowing the prototype and never touching the shared reference:
```js
function Fixed() { this.items = []; } // own property per instance, set during construction, not shared via prototype
```

**2. Getter-name-collision stack overflow — fix.**
> *Why it happens:* `get value() { return this.value; }` — reading `this.value` from *inside the getter for `value`* triggers the exact same getter again (property lookup doesn't know or care that it's already "inside" a getter call), recursing infinitely until the call stack overflows (Ch.3 §3.7's stack-overflow mechanism, triggered here by a naming mistake rather than genuine recursion).
> Fix — use a differently-named backing field so the getter/setter and the storage location are distinct properties:
```js
const fixed = { _value: 10, get value() { return this._value; }, set value(v) { this._value = v; } };
```

**3. `Object.freeze`-like helper via `defineProperty`.**
```js
function myFreeze(obj) {
  Object.keys(obj).forEach(key => {
    Object.defineProperty(obj, key, { writable: false, configurable: false });
  });
  return obj;
}
```
> *Mechanism:* every property has a descriptor (Ch.4 §4.2) controlling whether it can be reassigned (`writable`) or have its descriptor changed/be deleted (`configurable`). Setting both to `false` for every own key replicates the essential behavior of `Object.freeze` — note this is intentionally **shallow**, exactly like the real `Object.freeze`, since it only touches the object's own top-level property descriptors, not any nested objects those properties might reference (this is exactly why Ch.12's self-assessment question 7 exists).

**4. `Proxy`-based observable object.**
```js
function observable(target, onChange) {
  return new Proxy(target, {
    set(obj, key, value) {
      const old = obj[key];
      const result = Reflect.set(obj, key, value);
      if (old !== value) onChange(key, old, value);
      return result;
    },
  });
}
```
> *Mechanism:* the `set` **trap** (Ch.4 §4.7) intercepts every property assignment on the proxy, before it reaches the underlying object — letting you run arbitrary logic (here, firing a callback) around the write. `Reflect.set` (Ch.4 §4.6) performs the actual default assignment, so you're augmenting normal behavior rather than reimplementing it from scratch. This is a simplified version of the exact mechanism Vue 3's reactivity system uses — one general `set` trap, rather than needing to know each property's name ahead of time the way `Object.defineProperty`-based reactivity (Vue 2) required.

---

## Chapter 5 — Classes

**1. Implement `new` from scratch.**
```js
function myNew(Constructor, ...args) {
  const obj = Object.create(Constructor.prototype);
  const result = Constructor.apply(obj, args);
  return (typeof result === "object" && result !== null) ? result : obj;
}
```
> *Why each line matters:* `Object.create(Constructor.prototype)` performs steps 1–2 of `new` (Ch.5 §5.1) — create a bare object and link its `[[Prototype]]` to the constructor's `.prototype`, which is what makes `instanceof` and inherited methods work later. `Constructor.apply(obj, args)` performs step 3 — runs the constructor body with `this` explicitly bound to the new object (Ch.3 §3.3's `apply` mechanism). The final conditional performs step 4 — the real `new` only uses the constructor's own `return` value if it's an object; if the constructor returns a primitive (or nothing), the freshly created `obj` is used instead — a rule the real spec includes specifically so that implicit `return`s (or forgetting to return) don't silently break every class-based object ever created.

**2. Fix "must call super constructor before accessing 'this'".**
> *Why it happens:* in a subclass, `this` is not created by the subclass's own constructor — it's created by the **base class's constructor** and only becomes available once `super(...)` runs (Ch.5 §5.3). Referencing `this` (or calling a method that internally uses `this`) before that point references something that, mechanically, doesn't exist yet.
> Fix: reorder so all `this`-touching code comes strictly after `super(...)`:
```js
class Dog extends Animal {
  constructor(name, breed) {
    super(name);       // this now exists
    this.breed = breed; // safe here, and only here onward
  }
}
```

**3. Closure-based `Account` → `#private` fields.**
```js
class Account {
  #balance = 0;
  deposit(amt) { this.#balance += amt; }
  getBalance() { return this.#balance; }
}
```
> *Why this is a real upgrade, not just syntax:* the closure-based version's privacy came from `balance` living in a function scope that no external code could reference (Ch.3's scope chain) — genuinely private, but verbose and requiring a factory-function pattern instead of `class`. `#balance` achieves the same **language-enforced** privacy (unreachable via bracket notation, `Object.keys`, `JSON.stringify`, or `Reflect.ownKeys` — Ch.5 §5.5) while keeping normal `class` ergonomics: inheritance, `instanceof`, and standard method syntax all still work normally.

**4. `Shape` hierarchy with polymorphism + static factory.**
```js
class Shape { area() { return 0; } }
class Circle extends Shape { constructor(r) { super(); this.r = r; } area() { return Math.PI * this.r ** 2; } }
class Rectangle extends Shape { constructor(w, h) { super(); this.w = w; this.h = h; } area() { return this.w * this.h; } }
class ShapeFactory {
  static create(type, ...args) {
    if (type === "circle") return new Circle(...args);
    if (type === "rectangle") return new Rectangle(...args);
    throw new Error("Unknown shape");
  }
}
```
> *Why this demonstrates polymorphism specifically:* calling `.area()` on any `Shape` subclass instance doesn't require the caller to know or check *which* subclass it is — the prototype chain lookup (Ch.4 §4.4) automatically finds each instance's own, closest `area()` override before ever reaching `Shape.prototype.area`. This is **dynamic dispatch through the prototype chain**, not an `if/switch` on a type field — the mechanism is identical to Ch.5 §5.4's core explanation. The static factory is a separate concept: `ShapeFactory.create` lives on the factory class itself (Ch.5 §5.6), not on instances, and centralizes construction logic so calling code never needs to import/know about `Circle`/`Rectangle` directly.

---

## Chapter 6 — Asynchronous JavaScript

**1. Trace the §6.2 hard microtask example.**
```js
console.log("A");
setTimeout(() => console.log("B"), 0);
Promise.resolve().then(() => { console.log("C"); return Promise.resolve(); }).then(() => console.log("D"));
Promise.resolve().then(() => console.log("E"));
console.log("F");
```
> Output: **A, F, C, E, D, B.**
> *Step by step:* the call stack runs all synchronous code first — `A` and `F` print immediately, while both `Promise.resolve().then(...)` calls merely **schedule** their callbacks on the microtask queue without running them yet (Ch.6 §6.1's algorithm: sync code always finishes completely first).
> Once the stack is empty, the event loop drains the **entire** microtask queue before touching any macrotask. The queue at this point holds, in order: [C's callback, E's callback]. C runs first, printing `"C"`, and its `return Promise.resolve()` schedules a **new** microtask for D's callback — but that new microtask goes to the *back* of the current queue. E's callback was already sitting ahead of it (queued before C ever ran), so E runs next, printing `"E"`. Only after E is done does D's newly-scheduled continuation run, printing `"D"`.
> Only once the microtask queue is **completely** empty (no more C/D/E-style work left) does the event loop finally pull the macrotask — `setTimeout`'s callback — off the macrotask queue, printing `"B"` last.
> *The one rule this whole trace hinges on:* microtasks scheduled *during* microtask draining still get drained in the same pass, but they go to the back of the line — so ordering among microtasks depends on exactly *when* each was scheduled, not just which `.then()` appears first in the source.

**2. Fix `forEach` with `await` inside (doesn't actually await).**
```js
// Sequential (one at a time, each waits for the previous):
for (const item of items) { await process(item); }
// Concurrent (all start immediately, run in parallel):
await Promise.all(items.map(item => process(item)));
```
> *Why `forEach` breaks this:* `Array.prototype.forEach` calls its callback for every element and completely ignores whatever the callback returns — it has no concept of promises or `await` at all (Ch.6 §6.4). An `async` callback passed to `forEach` still returns a promise, but `forEach` throws that return value away and moves on to the next iteration immediately, so all the async callbacks effectively fire back-to-back with no actual waiting between them, and any code written *after* the `forEach` call runs before any of them finish. A plain `for...of` loop, by contrast, genuinely pauses the surrounding `async function` at each `await`, because `await` is a language-level suspension point tied to that specific function's execution — not something `forEach`'s internal iteration logic participates in at all.

**3. Implement `myPromiseAll`.**
```js
function myPromiseAll(promises) {
  return new Promise((resolve, reject) => {
    const results = []; let completed = 0;
    if (promises.length === 0) return resolve([]);
    promises.forEach((p, i) => {
      Promise.resolve(p).then(v => { results[i] = v; if (++completed === promises.length) resolve(results); }, reject);
    });
  });
}
```
> *Why it's built this way:* each input is wrapped in `Promise.resolve(p)` so the function works even if given plain (non-promise) values, mirroring the real `Promise.all`'s contract. Results are written into `results[i]` by **index**, not push order — because promises can settle in any order, and the contract of `Promise.all` guarantees the output array matches the *input* order, not completion order. The second argument to `.then` (the rejection handler) immediately calls the outer `reject`, which is why `Promise.all` **short-circuits**: the very first rejection anywhere immediately settles the whole combined promise, even while other promises are still pending — this is the mechanical reason `Promise.all` behaves differently from `Promise.allSettled` (Ch.6 §6.3).

**4. Implement `retry(fn, times, delayMs)`.**
```js
async function retry(fn, times, delayMs) {
  for (let i = 0; i < times; i++) {
    try { return await fn(); }
    catch (err) { if (i === times - 1) throw err; await new Promise(r => setTimeout(r, delayMs)); }
  }
}
```
> *Mechanism:* because `await` converts a rejected promise into a thrown exception inside an `async function` (Ch.6 §6.4 / Ch.9 §9.4), a failing `fn()` call is caught by the ordinary `try/catch` on each loop iteration rather than needing a `.catch()` chain. On the final allowed attempt, the error is deliberately re-thrown instead of retried again, propagating failure to the caller (Ch.9 §9.3's "don't silently swallow" principle). Between retries, `await new Promise(r => setTimeout(r, delayMs))` is a common idiom for "pause this async function for `delayMs` without blocking the thread" — it creates a promise that resolves only once the timer fires, and `await` suspends the function (not the thread) until then.

---

## Chapter 7 — Memory

**1. Why circular references don't leak in modern engines.**
> *Reference counting's failure mode:* under naive reference counting, an object is freed the instant its incoming-reference count hits zero. If object A holds a reference to B and B holds a reference back to A, each has a count of at least 1 **from the other**, even after every *external* reference to both is gone — the count never reaches zero, so neither is ever freed, even though nothing outside the pair can reach either of them anymore.
> *Why mark-and-sweep avoids this entirely:* mark-and-sweep (Ch.7 §7.2) doesn't count references at all — it starts from a fixed set of **roots** (globals, the current call stack, live closures) and walks every reachable reference outward, marking whatever it finds. A cycle that's unreachable from any root is simply never marked during that walk, regardless of how many references exist *within* the cycle — so both objects are correctly identified as garbage and swept, with the cycle being completely irrelevant to the algorithm's correctness.

**2. Fix the forgotten-interval leak.**
```js
function startPolling(largeData) {
  const id = setInterval(() => console.log(largeData.length), 1000);
  return () => clearInterval(id); // caller MUST invoke this to release the closure/timer
}
const stop = startPolling(data);
// later: stop();
```
> *Why the original leaked:* the interval's callback is a closure (Ch.3 §3.4) that captures `largeData` — as long as the interval is active, the runtime holds a live reference to that callback, which in turn keeps `largeData` reachable from a root (the active timer registry), so it can never be garbage collected, no matter how large or how long unused. `clearInterval` removes the timer's registration entirely, which drops the last reference to the callback (and therefore to `largeData`) — only then does it become unreachable and eligible for collection.

**3. WeakMap-based memoization for object-argument functions.**
```js
function memoizeByObject(fn) {
  const cache = new WeakMap();
  return (obj) => {
    if (cache.has(obj)) return cache.get(obj);
    const result = fn(obj);
    cache.set(obj, result);
    return result;
  };
}
```
> *Why `WeakMap` specifically:* a regular `Map` would hold a **strong** reference to every `obj` key ever passed in, keeping each one (and everything it references) alive in memory for the entire lifetime of the cache — even long after the calling code has no other reference to that object (Ch.7 §7.4). `WeakMap` holds only a **weak** reference: once `obj` becomes unreachable from anywhere else in the program, the garbage collector is free to reclaim it, and its cache entry is automatically and silently removed along with it — giving you caching without turning the cache into a de facto memory leak.

**4. DevTools heap snapshot diagnostic process.**
> Take a baseline heap snapshot → perform the suspected leaking action repeatedly (e.g., open/close the same modal N times) → take a second snapshot → use DevTools' snapshot **comparison** view, which shows the delta in retained object counts per constructor between the two snapshots — a steadily growing count for a constructor tied to the repeated action (e.g., `Modal` instances, or their internal listener closures) is the smoking gun. Specifically filter for **"Detached"** DOM nodes, since a node removed from the visible page but still referenced by leftover JS state shows up exactly this way — DevTools literally labels it "detached." From there, use the snapshot's **retainer tree** (what object is holding a reference to the leaking one) to trace backward to the specific closure, event listener, or cache entry responsible, which turns a vague "it feels slower over time" bug report into a specific line of code to fix.

---

## Chapter 8 — Modules

**1. ESM live binding vs. CommonJS primitive export.**
> *ESM:* `export let count` doesn't export a copy of the value — it exports a **live binding**, essentially a reference to the module's own internal variable slot (Ch.8 §8.3). When `increment()` later changes `count` inside the exporting module, every importer sees the updated value immediately, because they're not holding a snapshot — they're looking at the same underlying binding.
> *CommonJS:* `module.exports = { count }` builds a plain object at the moment that line runs, copying whatever `count`'s value was *at that instant* into a new property. If the exporting module later does `count = 5` (reassigning its own local variable), nothing about the already-built `module.exports` object changes — the importer's copy is frozen in time from export. (Note the important nuance: if the exporting module instead *mutates* an exported *object's* property, like `module.exports.settings.x = 5`, importers WOULD see that change — because objects are reference types, Ch.2 §2.5. It's specifically reassignment of primitives that CommonJS fails to propagate live.)
> *Root cause of the difference:* ESM's live-binding behavior is only possible because `import`/`export` are resolved through the module system's own static binding mechanism at parse time, not through ordinary object property copying — a direct consequence of ESM being a first-class language feature rather than a library-level convention built on regular objects (Ch.8 §8.3).

**2. 2MB bundle growth from one icon import — diagnose.**
> Likely causes, in order of likelihood: the icon library ships as **CommonJS only**, so the bundler can't statically prove which exports are unused (Ch.8 §8.5 — tree shaking requires ESM's static, non-dynamic `import`/`export` structure) and conservatively includes the whole module graph; or the library has **side-effecting top-level code** (e.g., auto-registering every icon globally on import), which a bundler must preserve even if you only explicitly import one named export, since removing it could change observable behavior; or the package's `package.json` simply doesn't declare `"sideEffects": false`, so the bundler defaults to the safe-but-conservative assumption that nothing can be removed.
> Diagnosis: run a bundle analyzer (`source-map-explorer`, `webpack-bundle-analyzer`) to visually confirm the entire icon library (not just the one icon) landed in the output. Fix: switch to an ESM-native build of the library, import the specific icon's own submodule path directly if the library supports it, or add/verify `"sideEffects": false` in the library's `package.json` if you control it.

**3. Convert CommonJS to ESM.**
```js
// Before (CJS)
function add(a, b) { return a + b; }
module.exports = { add };
module.exports.default = function multiply(a, b) { return a * b; };

// After (ESM)
export function add(a, b) { return a + b; }
export default function multiply(a, b) { return a * b; }
```
> *Why the mapping works this way:* CommonJS has no native concept of "default" vs. "named" exports — it's just properties on one `module.exports` object, so a `.default` property was a common convention to *simulate* what ESM does natively. ESM's `export default` is a genuine, separate binding type from named `export`s (Ch.8 §8.3), which is why the converted version can express the same two exports (`add` as named, `multiply` as default) using dedicated syntax instead of an object-property convention.

**4. Route-based code splitting strategy.**
> Eagerly bundle: the app shell, router, and the single most-likely-first route — this minimizes what has to download before the user sees *anything* useful, which is the actual metric ("Time To Interactive") this strategy optimizes for.
> Lazily load everything else via dynamic `import()` (Ch.8 §8.4): each additional route becomes its own separate chunk, fetched only when the user actually navigates there — the bundler recognizes the `import()` call syntax specifically and automatically splits that module's code (and its unique dependencies) out of the main bundle.
> Optional refinement: **preload** likely-next routes (e.g., on link hover, or via `<link rel="prefetch">`) so the network request for that chunk starts *before* the user actually clicks, hiding the latency of the lazy load behind their decision-making time rather than making them wait after clicking. This whole strategy is the runtime analog of Ch.8 §8.5's tree shaking — instead of eliminating unused code entirely, it defers loading code that's "unused right now."

---

## Chapter 9 — Error Handling

**1. `try` returns `"A"`, `finally` returns `"B"` — result and danger.**
> Returns **`"B"`.**
> *Mechanism:* when `try` hits its `return "A"`, the engine notes that a return is pending but doesn't actually exit the function yet — it must first run `finally`, since `finally` is guaranteed to execute no matter how `try` exits (Ch.9 §9.1). If `finally` itself contains a `return`, that **new** return statement completely overrides the pending one from `try` — the function ultimately returns `"B"`, and the `"A"` return is discarded entirely as if it never happened.
> *Why this is dangerous, specifically:* this same override behavior applies not just to a competing `return`, but to a **pending thrown exception** too — if `try` throws an error and `finally` contains a `return`, the exception is silently swallowed and never propagates, with nothing in the code visibly indicating that an error was lost. This is why a `return` inside `finally` is considered a serious anti-pattern in code review, not just a style nitpick.

**2. Fix the missing-`await` bug.**
```js
async function fixed() {
  try {
    const user = await fetchUser(id); // await added — now errors ARE caught here
    console.log(user);
  } catch (err) { console.error(err); }
}
```
> *Why the original broke:* without `await`, `fetchUser(id).then(...)` creates a promise chain that is **entirely disconnected** from the surrounding `async function`'s own control flow and its `try/catch` (Ch.9 §9.4) — the `async function` itself doesn't know that promise exists, so it can't catch anything it rejects with. `await` is what actually links a promise's eventual rejection to the enclosing function's synchronous-looking `try/catch`, by converting that rejection into a real thrown exception at the point of the `await` expression.

**3. `AppError` hierarchy.**
```js
class AppError extends Error { constructor(msg) { super(msg); this.name = this.constructor.name; } }
class NetworkError extends AppError {}
class ValidationError extends AppError { constructor(msg, field) { super(msg); this.field = field; } }

try { /* ... */ }
catch (err) {
  if (err instanceof ValidationError) console.log("field:", err.field);
  else if (err instanceof NetworkError) console.log("retry the request");
  else throw err;
}
```
> *Why extend `Error` at all levels, and why `this.name = this.constructor.name`:* extending the built-in `Error` class (Ch.9 §9.3) means every custom error automatically gets a real `.stack` trace captured at construction — essential for debugging, and something a plain thrown object would never have. Setting `this.name` to `this.constructor.name` dynamically means each subclass's stack trace and `.toString()` correctly show `"NetworkError"` or `"ValidationError"` rather than the generic inherited `"Error"`, without needing to hardcode the name separately in every subclass.
> *Why `instanceof` beats string-matching `err.message`:* it uses the actual prototype chain (Ch.4 §4.4) to check type, which is exact and can't accidentally match on unrelated errors that happen to contain similar wording — and the final `else throw err` re-throws anything genuinely unexpected rather than silently swallowing it (Ch.9 §9.3's "don't catch what you don't understand" principle).

**4. `safeAsync(fn)` returning `[error, result]`.**
```js
function safeAsync(fn) {
  return async (...args) => {
    try { return [null, await fn(...args)]; }
    catch (err) { return [err, null]; }
  };
}
```
> *Mechanism:* this wraps the `await`-triggers-a-catchable-exception mechanism from Ch.9 §9.4 once, generically, for any async function — instead of every call site needing its own `try/catch`, callers get a plain tuple back and can check `if (error) { ... }` the same way Go-style error handling works. The trade-off worth stating explicitly in an interview: this pattern trades the language's built-in propagation (an uncaught error normally bubbles until something handles it) for an explicit, opt-in check at every call site — good for making failure paths visible and impossible to silently ignore, but it means forgetting to check the returned `error` is now a *logic* bug rather than something the runtime would ever flag on its own.

---

## Chapter 10 — Advanced/Metaprogramming

**1 & 2. `LinkedList` iterable — manual vs. generator.**
```js
class LinkedList {
  #head = null;
  add(value) {
    const node = { value, next: null };
    if (!this.#head) this.#head = node;
    else { let cur = this.#head; while (cur.next) cur = cur.next; cur.next = node; }
    return this;
  }
  // Manual iterator protocol implementation:
  [Symbol.iterator]() {
    let current = this.#head;
    return {
      next() {
        if (!current) return { value: undefined, done: true };
        const value = current.value;
        current = current.next;
        return { value, done: false };
      },
    };
  }
  // Generator version — same external behavior, far less manual state tracking:
  *[Symbol.iterator]() { let cur = this.#head; while (cur) { yield cur.value; cur = cur.next; } }
}
```
> *Why the manual version needs its own closure variable (`current`):* the iterator protocol (Ch.10 §10.2) requires a `.next()` method that remembers, **between separate calls**, exactly where iteration left off — there's no other built-in mechanism for that, so you must manually maintain a mutable `current` pointer captured by the returned object's closure.
> *Why the generator version is strictly simpler:* a generator function (Ch.6 §6.5 / Ch.10 §10.3) automatically preserves its entire execution state — including the value of `cur` — across `yield` pauses, using the engine's own suspension mechanism instead of a hand-rolled closure variable. This is the exact reason generators are the recommended real-world approach for implementing custom iterables: they eliminate an entire category of manual state-tracking bugs.

**3. `readOnly(obj)` Proxy.**
```js
function readOnly(obj) {
  return new Proxy(obj, {
    set() { throw new TypeError("Object is read-only"); },
    deleteProperty() { throw new TypeError("Object is read-only"); },
    defineProperty() { throw new TypeError("Object is read-only"); },
  });
}
```
> *Why three separate traps are needed:* each fundamental object operation — assignment, `delete`, and `Object.defineProperty` — is intercepted by its own distinct trap (Ch.4 §4.7 / Ch.10 §10.4); there's no single "block all mutation" trap, because `Proxy`'s design mirrors the actual distinct internal operations the spec defines. Leaving any one of the three traps out would leave a mutation path open (e.g., omitting `deleteProperty` would still let `delete obj.key` succeed even though `obj.key = x` correctly throws).

**4. Recursive deep-freeze `Proxy`.**
```js
function deepFreeze(obj) {
  return new Proxy(obj, {
    get(target, prop) {
      const value = Reflect.get(target, prop);
      return (typeof value === "object" && value !== null) ? deepFreeze(value) : value;
    },
    set() { throw new TypeError("Cannot mutate frozen object"); },
  });
}
```
> *Why wrapping happens inside the `get` trap, not upfront:* eagerly wrapping every nested object recursively at creation time would require walking the entire object graph immediately (expensive, and impossible for circular/very large structures) — instead, this **lazily** wraps a nested object only the moment it's actually accessed, returning a fresh `deepFreeze`-wrapped proxy for that nested value each time. This means mutation protection extends to arbitrary depth automatically, without ever needing to know the shape of the object ahead of time — a direct, practical illustration of why `Proxy` traps operate generically per-operation rather than requiring upfront knowledge of an object's structure (Ch.4 §4.7's core theme).

---

## Chapter 11 — Performance

**1. Debounce vs. throttle — one-sentence rule.**
> Use **debounce** when only the *final* state after a burst of activity matters (search input, autosave, resize-then-recalculate) — it deliberately delays execution and resets that delay on every new call, so rapid-fire triggers collapse into one. Use **throttle** when you need *periodic* updates *during* continuous activity (scroll position tracking, drag handling) — it deliberately allows execution at a capped, regular rate rather than waiting for activity to fully stop, because waiting would mean the UI never updates while the user is still actively scrolling/dragging.

**2. Redesign naive `memoize` with LRU eviction.**
```js
function memoizeLRU(fn, maxSize = 100) {
  const cache = new Map();
  return (...args) => {
    const key = JSON.stringify(args);
    if (cache.has(key)) {
      const value = cache.get(key);
      cache.delete(key); cache.set(key, value); // re-insert to mark as most-recently-used
      return value;
    }
    const result = fn(...args);
    if (cache.size >= maxSize) cache.delete(cache.keys().next().value); // evict the oldest (least-recently-used) entry
    cache.set(key, result);
    return result;
  };
}
```
> *Why this fixes the leak:* the naive version from Ch.11 §11.4 grows its cache **unboundedly** — every unique argument combination ever seen adds a permanent entry, which on a long-running server process is functionally the same failure mode as Ch.7's memory leaks (accidentally-retained reachable memory that's never needed again). This version exploits `Map`'s **guaranteed insertion order** — deleting then re-setting a key moves it to the end of iteration order, so the *first* key returned by `.keys()` is always the least-recently-used one, giving true LRU eviction in O(1) per operation without a separate tracking structure.

**3. `intersectFast` preserving duplicate counts, better than O(n·m).**
```js
function intersectWithCounts(a, b) {
  const countsB = new Map();
  for (const x of b) countsB.set(x, (countsB.get(x) || 0) + 1);
  const result = [];
  for (const x of a) {
    if (countsB.get(x) > 0) { result.push(x); countsB.set(x, countsB.get(x) - 1); }
  }
  return result; // O(n + m)
}
```
> *Why this beats naive `includes()`-in-a-loop:* the original problem (Ch.11 §11.6) was that `b.includes(x)` performs an **O(m) linear scan** for every one of the `n` elements in `a`, giving O(n·m) overall. Pre-building a `Map` of counts from `b` costs O(m) once, and each lookup (`countsB.get(x)`) is O(1) average (hash-based, Ch.11 §11.6), so the total cost becomes O(n + m) — the algorithmic complexity class itself changes, not just the constant factor. Using a `Map` of **counts** rather than a `Set` of presence is what correctly preserves duplicate-handling: each match decrements the remaining count for that value, so a value appearing twice in `b` can match at most twice in `a`, exactly mirroring correct multiset intersection semantics.

**4. Virtualized list rendering strategy.**
> Compute the currently visible index range from the container's `scrollTop` and a known (or estimated) row height, then render **only** the DOM nodes for that range plus a small buffer — never the full dataset. Use a spacer element (or CSS padding) representing the total height of all *unrendered* rows above and below the visible window, so the scrollbar's size and position remain correct even though most rows don't physically exist as DOM nodes. Attach one **throttled** scroll listener (Ch.11 §11.3 — throttle, not debounce, because you need periodic updates *during* continuous scrolling, not only once it stops) that recalculates the visible range and re-renders only the delta.
> *Why this combines three separate chapter ideas into one system:* it applies event delegation's core principle (do the minimum work per event, Ch.11 §11.1) to scroll events specifically; it applies Big-O awareness (Ch.11 §11.6) by keeping DOM node count proportional to *viewport size*, not *dataset size* — turning an O(n) render cost into an O(visible rows) cost regardless of how large the underlying list grows; and it uses throttling (§11.3) to bound how often the (relatively expensive) recalculation work runs, rather than on every single scroll-event firing.

---

## Chapter 12 — Self-Assessment Answers (§12.5)

1. **`typeof null === "object"`:** a bug preserved from the original 1995 implementation — values were internally tagged with a type ID, and the "object" tag happened to share its numeric encoding with the null pointer representation. It can't be fixed today without breaking any existing code that (knowingly or not) depends on this exact behavior — a direct consequence of Ch.1's permanent-backward-compatibility rule, not a logical design choice anyone would make today.
2. **`var` vs `let` in loops:** `var` is function-scoped, so a `for (var i...)` loop has exactly **one** `i` binding shared across all iterations — every closure created inside the loop captures a reference to that same single variable, so by the time any callback runs, they all see its final post-loop value. `let` is block-scoped and the spec specifically creates a **fresh binding per iteration** for `for`-loop `let` declarations, so each closure captures its own independent snapshot.
3. **`this` in regular vs. arrow functions:** regular functions resolve `this` dynamically based on the **call site** — how the function is actually invoked (implicit/explicit/default/`new` binding, ranked by precedence in Ch.3 §3.2). Arrow functions have no `this`-binding mechanism of their own at all; they resolve `this` exactly like any other lexically-scoped variable, inheriting it permanently from whatever scope enclosed them at the moment they were defined — this was added specifically to eliminate the extremely common bug of `this` unexpectedly changing inside a passed-around callback.
4. **Interleaved trace:** all currently-queued synchronous code runs to completion first (the call stack must be empty) → then the **entire** microtask queue drains, including any new microtasks scheduled by microtasks that ran during this same drain → only once no microtasks remain does the event loop pull and run the next single macrotask, then the cycle repeats.
5. **Circular references and GC:** mark-and-sweep collectors determine garbage by walking **reachability from roots**, not by counting incoming references — an unreachable cycle is still correctly identified as unreachable (nothing outside the cycle points to it) and gets swept, regardless of how many references the cycle's members hold on each other internally.
6. **Tree shaking ESM vs. CJS:** ESM requires `import`/`export` to be static, top-level declarations with literal (non-computed) specifiers — this lets a bundler compute the complete, exact dependency/usage graph purely by reading the code, without executing any of it. CommonJS's `require()` is a normal function call that can be fully dynamic and conditional (`require(someComputedPath)`), so a bundler can't safely prove which parts of a module are unused without actually running it — so it conservatively keeps everything.
7. **`Object.freeze` is shallow:** it only locks the descriptors of the object's own, immediate top-level properties (`writable`/`configurable` set to false) — it has no awareness of, and doesn't recurse into, any nested object or array that those properties happen to reference, which remains just as mutable as if it had never been frozen at all.
8. **Debounce vs. throttle:** debounce collapses a burst of calls into exactly one, fired only after activity has fully stopped for the specified delay; throttle allows execution to repeat, but caps it to at most once per fixed interval, continuing to fire periodically throughout ongoing activity rather than waiting for it to end.
9. **No TCO in V8:** although the ECMAScript spec explicitly permits engines to implement proper tail-call optimization (reusing the current stack frame instead of pushing a new one for a call in tail position), V8 has never shipped this optimization — meaning even code written in a deliberately tail-recursive style can still overflow the call stack on deep recursion in real Chrome/Node environments, despite being spec-legal. Production code with unbounded or very deep recursion depth should be rewritten iteratively or via trampolining rather than relying on the spec's permission alone.
10. **`new Foo()` → prototype chain lookup:** `new` creates a fresh object, links its internal `[[Prototype]]` to `Foo.prototype`, and runs `Foo`'s body with `this` bound to that new object (Ch.5 §5.1's four steps). Later, calling `instance.method()` first checks whether `method` exists as an **own** property directly on `instance` — if not found, it walks the `[[Prototype]]` chain (`Foo.prototype`, then whatever `Foo.prototype`'s own `[[Prototype]]` points to, typically `Object.prototype`, then `null`), returning the first match found, or `undefined`/a `TypeError` if the chain is exhausted with nothing found.

---

## How to use this going forward

- Re-attempt any exercise where your first instinct didn't match the model answer — and specifically check whether you got the **result** right but the **mechanism** wrong, since that's the gap that actually matters in a senior-level interview.
- The **mechanisms**, not the specific code or specific outputs, are what generalize: e.g., for Ch.6 exercise #1, the value isn't memorizing "A, F, C, E, D, B" — it's being able to derive that order from the microtask-queue-draining rule for *any* differently-shaped example you haven't seen before.
- If you want, I can turn any subset of these into a timed, randomized quiz format, or generate fresh (unseen) variants of the trickiest ones — new microtask-ordering traces, new `this`-binding puzzles, new Big-O bundle-size scenarios — specifically designed so memorizing this answer key won't help, only genuine understanding of the mechanism will.
