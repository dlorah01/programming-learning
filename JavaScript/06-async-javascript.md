---
layout: default
title: "Chapter 6 — Asynchronous JavaScript"
---

# Chapter 6 — Asynchronous JavaScript

> Depends on: Ch.1 §1.7 (single-threaded, run-to-completion execution model), Ch.3 (closures — callbacks are just functions with captured scope).
> Feeds into: Ch.9 (error handling in async code), Ch.11 (performance patterns like debouncing rely on the event loop).

This is the single most-tested topic in mid/senior JS interviews. Nail this chapter and a huge fraction of "why is this happening" questions become trivial.

---

## 6.1 The Call Stack, Web APIs, and Queues — the Full Picture

**Intuitive version (recap + expansion of Ch.1 §1.7):** JS itself only has one thread and one call stack. Anything that takes time — timers, network requests, file I/O — is delegated to the **runtime** (browser Web APIs or Node's libuv), which runs it outside the JS thread, and hands a callback back to be queued once it's done. The **event loop** is the mechanism that decides when queued callbacks are allowed to run — only ever when the call stack is completely empty.

**Formal version — the four pieces:**
1. **Call Stack** — where currently-executing JS lives, one frame per active function call.
2. **Web APIs / Node APIs** — the runtime's own threads/mechanisms (timers, network stack, filesystem) that do the actual waiting, outside the JS engine.
3. **Callback Queue (a.k.a. Macrotask Queue)** — holds callbacks from `setTimeout`, `setInterval`, DOM events, I/O.
4. **Microtask Queue** — holds callbacks from Promises (`.then`/`.catch`/`.finally`), `queueMicrotask`, and (in Node) `process.nextTick` (which is actually even higher priority than regular microtasks).

**The event loop's actual algorithm (memorize this — it's asked constantly):**
```
loop forever:
  1. Run the next macrotask (if the call stack is empty) — e.g. one setTimeout callback
  2. After that macrotask finishes, drain the ENTIRE microtask queue completely
     (including any new microtasks queued by microtasks that just ran)
  3. (browser only) possibly render a frame
  4. go back to step 1
```

**The critical rule that trips everyone up:** the microtask queue is **fully drained** (not just one item) before the next macrotask runs, and even before rendering.

```js
console.log("1: sync");

setTimeout(() => console.log("2: macrotask"), 0);

Promise.resolve().then(() => console.log("3: microtask"));

console.log("4: sync");

// Output: 1, 4, 3, 2
// All sync code first (1, 4), THEN the entire microtask queue (3), THEN the next macrotask (2)
```

**Mental model (extends Ch.1's chef analogy):** the chef (call stack) finishes all current orders. Before taking the *next* order ticket from the counter (macrotask queue), the chef first clears out an entire tray of "quick follow-up notes" (microtasks) that piled up on the counter — even if new follow-up notes keep appearing while clearing that tray, they ALL get done before the next big ticket is taken.

**Interview questions:**
- *Beginner:* What's the difference between the call stack and the callback queue?
- *Mid:* Predict the output of the code above and explain each step.
- *Senior/FAANG:* Given nested promises and setTimeouts, trace the exact execution order (see §6.2 for a harder version of this).

---

## 6.2 Microtasks vs. Macrotasks — Going Deeper

**Priority order (highest to lowest):**
1. Currently executing synchronous code (call stack)
2. `process.nextTick` (Node only — even higher priority than regular microtasks)
3. Microtasks (Promise callbacks, `queueMicrotask`)
4. Macrotasks (`setTimeout`, `setInterval`, I/O, UI events)

**A genuinely hard trace (classic FAANG whiteboard question):**
```js
console.log("A");

setTimeout(() => console.log("B"), 0);

Promise.resolve()
  .then(() => {
    console.log("C");
    return Promise.resolve();
  })
  .then(() => console.log("D"));

Promise.resolve().then(() => console.log("E"));

console.log("F");

// Output: A, F, C, E, D, B
// A, F: sync code runs first.
// Then microtask queue drains: first .then callback (C) runs, scheduling a NEW microtask
// for its own continuation — but E (already queued before C ran) is next in line ahead of D,
// because D was only queued once C's returned promise resolved (an extra microtask tick later).
// Only after ALL microtasks are exhausted does the macrotask (B) run.
```
Walking through *why* E comes before D (not just knowing the answer) is what separates a strong candidate from someone who memorized the output.

**Interview questions:**
- *Senior/FAANG:* Trace the exact output of the example above, explaining each microtask tick.
- *Senior:* Why does Node's `process.nextTick` run even before regular Promise microtasks?

---

## 6.3 Promises

**Intuitive version:** A `Promise` is an object representing a value that isn't available yet but will be, eventually — either successfully (**resolved**) or with a failure (**rejected**). It replaces the old "callback hell" pattern with something chainable and composable.

**Formal version:** a Promise has three states — **pending**, **fulfilled**, **rejected** — and is **settled** exactly once, permanently (state can't change again after settling).

```js
function wait(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}
wait(1000).then(() => console.log("done"));
```

**Why Promises exist:** deeply nested callbacks ("callback hell" / "pyramid of doom") were hard to read, hard to compose, and had inconsistent error handling (every callback needed its own error-first check). Promises standardize both composition and error propagation (errors bubble down the chain to the nearest `.catch`, like exceptions).

```js
// Callback hell (pre-ES6 style)
getUser(id, user => {
  getPosts(user.id, posts => {
    getComments(posts[0].id, comments => {
      console.log(comments); // deeply nested, error handling repeated at every level
    });
  });
});

// Promise chain — flat, composable, single error handler
getUser(id)
  .then(user => getPosts(user.id))
  .then(posts => getComments(posts[0].id))
  .then(comments => console.log(comments))
  .catch(err => console.error(err)); // catches errors from ANY step in the chain
```

**Combinators — know exactly what each does (frequently confused in interviews):**
```js
Promise.all([p1, p2, p3]);        // waits for ALL to resolve; rejects immediately if ANY rejects
Promise.allSettled([p1, p2, p3]); // waits for ALL to settle (success or fail), never short-circuits, gives status for each
Promise.race([p1, p2, p3]);        // settles as soon as the FIRST promise settles (resolve OR reject)
Promise.any([p1, p2, p3]);         // settles as soon as the FIRST one RESOLVES; rejects only if ALL reject (AggregateError)
```

**Common mistakes:**
- Forgetting to `return` inside a `.then` — breaks chaining, the next `.then` gets `undefined` instead of the awaited value.
- Using `Promise.all` when you actually want partial results even if some fail — should be `allSettled`.
- Creating an unnecessary "Promise constructor antipattern" — wrapping an already-promise-returning function in `new Promise()` needlessly.

```js
// Antipattern
function getData() {
  return new Promise((resolve, reject) => {
    fetchSomething().then(resolve).catch(reject); // pointless — fetchSomething() already returns a promise!
  });
}
// Fix: just return it directly
function getData() { return fetchSomething(); }
```

**Interview questions:**
- *Beginner:* What are the three states of a Promise?
- *Mid:* What's the difference between `Promise.all` and `Promise.allSettled`? Give a real scenario for each.
- *Senior:* What's the "Promise constructor antipattern" and why is it a smell?
- *FAANG-style:* Implement a simplified `Promise.all` from scratch:
  ```js
  function myPromiseAll(promises) {
    return new Promise((resolve, reject) => {
      const results = [];
      let completed = 0;
      if (promises.length === 0) return resolve([]);
      promises.forEach((p, i) => {
        Promise.resolve(p).then(value => {
          results[i] = value;
          if (++completed === promises.length) resolve(results);
        }, reject); // any single rejection rejects the whole thing immediately
      });
    });
  }
  ```

---

## 6.4 `async`/`await`

**Intuitive version:** `async`/`await` (ES2017) is syntax sugar that lets you write promise-based code that *looks* synchronous, without changing anything about the underlying event loop.

**Formal version:** an `async function` always returns a Promise (wrapping the return value, or the thrown error as a rejection). `await` pauses the function's execution (without blocking the thread — other code still runs) until the awaited promise settles.

```js
async function getUserData(id) {
  try {
    const user = await getUser(id);       // pauses HERE, but the rest of the program keeps running
    const posts = await getPosts(user.id);
    return posts;
  } catch (err) {
    console.error(err); // replaces .catch() — regular try/catch works because await "unwraps" rejections into throws
  }
}
```

**Crucial mechanism (senior-level, often mis-explained):** `await` doesn't block the thread. It suspends the *async function's* execution and schedules its continuation as a microtask once the awaited value resolves — meanwhile, the call stack is free and other synchronous code, other microtasks, and macrotasks continue to run normally.

**Sequential vs. concurrent awaiting — a very common real-world performance bug:**
```js
// SLOW — sequential, each await blocks the next from starting (total time = sum of both)
const a = await fetchA(); // 1000ms
const b = await fetchB(); // 1000ms
// total: ~2000ms

// FAST — concurrent, both start immediately, run in parallel (total time = max of both)
const [a, b] = await Promise.all([fetchA(), fetchB()]);
// total: ~1000ms
```
This is one of the most impactful real-world performance mistakes engineers make — awaiting independent operations sequentially when they could run concurrently.

**Interview questions:**
- *Beginner:* What does an `async function` always return?
- *Mid:* Does `await` block the JS thread? Explain precisely what it does instead.
- *Senior/FAANG:* Given two independent API calls, why is awaiting them sequentially slower than `Promise.all`, and rewrite the code to fix it.
- *Debugging exercise:* a `for` loop with `await fetch(...)` inside runs requests one at a time instead of in parallel — explain why, and fix it using `Promise.all` with `.map`.

---

## 6.5 Generators

**Intuitive version:** A generator function (`function*`) can pause its own execution at `yield` points and resume later, producing a sequence of values over time instead of all at once.

**Formal version:** calling a generator function doesn't run its body — it returns an **iterator** (Ch.10 for full protocol details). Each call to `.next()` runs the function until the next `yield`, returning `{ value, done }`.

```js
function* idGenerator() {
  let id = 1;
  while (true) yield id++;
}
const gen = idGenerator();
gen.next(); // { value: 1, done: false }
gen.next(); // { value: 2, done: false }
```

**Why generators matter for async JS (historical link):** before native `async`/`await` existed, generators + a "runner" function (libraries like `co`) were the community's way of writing pause-and-resume async code — `async`/`await` is essentially generators with a built-in runner baked into the language. Understanding this lineage is a great "how did the language evolve" senior answer.

**Interview questions:**
- *Mid:* What does calling a generator function return, and when does its body actually start executing?
- *Senior:* How do generators relate historically to `async`/`await`?

---

## 6.6 `fetch` and `AbortController`

**Intuitive version:** `fetch` is the modern, Promise-based way to make HTTP requests (replacing `XMLHttpRequest`). `AbortController` lets you cancel an in-flight `fetch` (or other abortable operation).

**Formal version:**
```js
const controller = new AbortController();
fetch(url, { signal: controller.signal })
  .then(res => res.json())
  .catch(err => {
    if (err.name === "AbortError") console.log("Request was cancelled");
  });

setTimeout(() => controller.abort(), 5000); // cancel if it takes longer than 5s
```

**Common mistake:** forgetting that `fetch` does **not** reject on HTTP error statuses (404, 500) — only on network failure. You must check `response.ok` manually.
```js
const res = await fetch(url);
if (!res.ok) throw new Error(`HTTP ${res.status}`); // fetch alone won't throw for a 404!
```

**Real-world use case for `AbortController`:** cancelling a search-as-you-type API request when the user types a new character before the previous request finishes (avoiding race conditions where an old, slow response overwrites a newer, faster one).

**Interview questions:**
- *Mid:* Does `fetch` reject on a 404 response? Why or why not?
- *Senior:* Design a debounced search input that cancels the previous in-flight request when a new keystroke happens, using `AbortController`.

---

## 6.7 Async Generators & `for await...of`

**Intuitive version:** Combines generators and promises — a function that can `yield` values that are themselves asynchronous, consumed one at a time with `for await...of`.

```js
async function* streamPages(url) {
  let nextUrl = url;
  while (nextUrl) {
    const res = await fetch(nextUrl);
    const data = await res.json();
    yield data.items;
    nextUrl = data.nextUrl;
  }
}

for await (const page of streamPages(startUrl)) {
  console.log(page); // processes each page as it arrives, without loading everything into memory at once
}
```

**Real-world use case:** paginated API consumption, streaming large datasets, reading Node streams — anywhere you want to process async data incrementally rather than waiting for the entire dataset.

**Interview questions:**
- *Senior:* When would you reach for an async generator instead of just awaiting `Promise.all` on every page upfront? (Answer: memory efficiency for large/unbounded datasets, and processing results as they arrive rather than waiting for everything.)

---

## Practical exercises — Chapter 6

1. Without running it, trace the exact output order of the §6.2 example, explaining each microtask tick out loud.
2. **Debugging exercise:** a loop does `items.forEach(async item => await process(item))` expecting sequential processing, but everything appears to run concurrently and the code after the loop runs too early. Explain why `forEach` doesn't await async callbacks, and rewrite it correctly with a `for...of` loop or `Promise.all`.
3. **Small challenge:** implement `myPromiseAll` (shown above) from memory, without looking back.
4. **Advanced challenge:** implement a `retry(fn, times, delayMs)` utility that retries a failing async function up to `times` times with a delay between attempts, using `async`/`await` and a loop.

---

**Next:** Chapter 7 — Memory (stack vs. heap, garbage collection, memory leaks, WeakMap/WeakSet) — this explains exactly what closures (Ch.3) and long-lived Promise chains are doing to your app's memory footprint. Say **"next"** when ready.
