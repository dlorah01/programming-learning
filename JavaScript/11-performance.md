# Chapter 11 — Performance

> Depends on: Ch.1 §1.3 (engine internals — monomorphism), Ch.3 §3.4 (closures power debounce/throttle/memoize), Ch.6 (event loop timing), Ch.7 (GC-aware caching), Ch.8 (code splitting via dynamic import).
> Feeds into: Ch.12 (synthesizing everything into real system design answers).

---

## 11.1 Event Delegation

**Intuitive version:** Instead of attaching a separate event listener to every single child element (expensive, and breaks for dynamically-added elements), attach ONE listener to a shared parent and use event bubbling to figure out which child was actually interacted with.

**Formal version:** DOM events **bubble** — they fire on the target element, then propagate upward through every ancestor. Delegation exploits this: listen on the ancestor, inspect `event.target` to determine the actual originating element.

```js
// BAD: N listeners for N list items, and new items added later have NO listener at all
document.querySelectorAll("li").forEach(li => li.addEventListener("click", handleClick));

// GOOD: ONE listener, works for items added to the DOM later too, since bubbling is dynamic
document.querySelector("ul").addEventListener("click", event => {
  if (event.target.tagName === "LI") handleClick(event);
});
```

**Why it matters practically:** reduces memory usage (one listener vs. thousands) and automatically handles dynamically-added elements without needing to re-attach listeners — a very common real-world win for large lists, tables, or infinite-scroll UIs.

**Interview questions:**
- *Mid:* What is event delegation and what DOM behavior does it rely on?
- *Senior:* Why does a naively-attached per-item listener fail for elements added to the DOM after page load, and how does delegation fix this automatically?

---

## 11.2 Debouncing

**Intuitive version:** Debouncing delays running a function until a burst of calls has *stopped* for a specified period — useful when you only care about the *final* state after rapid-fire events (typing, window resize).

**Formal version:**
```js
function debounce(fn, delay) {
  let timer;
  return (...args) => {
    clearTimeout(timer);                       // cancel any pending call
    timer = setTimeout(() => fn(...args), delay); // schedule a fresh one
  };
}
const debouncedSearch = debounce(query => api.search(query), 300);
input.addEventListener("input", e => debouncedSearch(e.target.value));
// only fires the API call 300ms after the user STOPS typing, not on every keystroke
```

**Real-world use cases:** search-as-you-type, resize handlers, autosave, form validation on input.

**Interview questions:**
- *Mid:* Implement `debounce` from scratch (shown above) — extremely common live-coding question, already previewed in Ch.3.
- *Senior:* Give a real bug you'd hit from NOT debouncing a search input (excessive API calls, race conditions between out-of-order responses — ties to Ch.6's `AbortController` discussion).

---

## 11.3 Throttling

**Intuitive version:** Throttling ensures a function runs **at most once** per specified interval, no matter how many times it's triggered — useful when you want *regular, periodic* updates during a continuous stream of events (scroll, mousemove), not just the final one.

**Formal version:**
```js
function throttle(fn, interval) {
  let lastCall = 0;
  return (...args) => {
    const now = Date.now();
    if (now - lastCall >= interval) {
      lastCall = now;
      fn(...args);
    }
  };
}
window.addEventListener("scroll", throttle(updateScrollIndicator, 100));
// updateScrollIndicator runs at most once every 100ms, giving smooth periodic updates
```

**Debounce vs. throttle — the distinction interviewers specifically probe:**

| | Debounce | Throttle |
|---|---|---|
| Fires | Once, after activity STOPS | Repeatedly, at a fixed max rate, DURING activity |
| Use case | Search input, autosave | Scroll position tracking, mousemove-based drag, rate-limiting API calls |

**Interview questions:**
- *Mid:* Implement `throttle` from scratch and explain, precisely, how it differs from `debounce`.
- *Senior/FAANG:* Given an infinite-scroll feature, would you use debounce or throttle for the scroll listener that checks "are we near the bottom"? Justify it. (Throttle — you want periodic checks *during* continuous scrolling, not only once scrolling fully stops.)

---

## 11.4 Memoization

**Intuitive version:** Caching a function's results by its input arguments so repeated calls with the same inputs return instantly from cache instead of recomputing.

**Formal version:**
```js
function memoize(fn) {
  const cache = new Map();
  return (...args) => {
    const key = JSON.stringify(args); // simple but imperfect key strategy — see caveats below
    if (cache.has(key)) return cache.get(key);
    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
}
const slowFib = n => (n <= 1 ? n : slowFib(n - 1) + slowFib(n - 2));
const fastFib = memoize(function fib(n) { return n <= 1 ? n : fastFib(n - 1) + fastFib(n - 2); });
// naive slowFib(40) takes seconds (exponential, ~2^40 calls); memoized version is near-instant (linear)
```

**Trade-off (must state in any senior answer):** memoization trades **memory for speed** — an unbounded cache on a long-running server process is itself a memory leak risk (ties directly to Ch.7). Production memoization should use a bounded cache (LRU eviction) or a `WeakMap` when keyed by objects (Ch.7 §7.4).

**`JSON.stringify` key caveat:** fails for non-serializable arguments (functions, `undefined` inside objects, circular references) and can produce different strings for objects with the same keys in different orders (`{a:1,b:2}` vs `{b:2,a:1}`) — production memoization libraries use more robust key strategies.

**Interview questions:**
- *Mid:* Implement `memoize` from scratch.
- *Senior:* What's the risk of memoizing without any cache eviction strategy in a long-running server process?
- *FAANG-style:* Compare the time complexity of naive recursive Fibonacci (exponential, O(2^n)) vs. memoized Fibonacci (linear, O(n)) — explain exactly why memoization changes the complexity class, not just the constant factor.

---

## 11.5 Lazy Loading & Code Splitting

**Intuitive version:** Don't load/compute what the user doesn't need yet — defer it until it's actually required. This spans images (`loading="lazy"`), routes (dynamic `import()`, Ch.8 §8.4), and expensive computations.

```html
<img src="huge-photo.jpg" loading="lazy" /> <!-- browser defers loading until near viewport -->
```
```js
// Route-based code splitting (React example, but the underlying mechanism is Ch.8's dynamic import)
const SettingsPage = React.lazy(() => import("./SettingsPage"));
```

**Why it matters:** reduces initial bundle size / Time To Interactive — a huge real-world performance lever, especially on slow networks/devices. This directly connects to Ch.8's tree-shaking and dynamic-import material — lazy loading is tree shaking's runtime counterpart (deferring, rather than eliminating, unused-right-now code).

**Interview questions:**
- *Mid:* How does route-based code splitting improve initial load time?
- *Senior:* What's the trade-off of lazy loading (e.g., a brief loading spinner/flash when a lazily-loaded chunk is first requested), and how do you mitigate it (preloading on hover/intent, suspense fallbacks)?

---

## 11.6 Big O Considerations in Real JS Code

**Intuitive version:** Algorithmic complexity isn't just a whiteboard abstraction — common JS array/object methods have real, differing costs that matter in hot paths.

**Common complexity gotchas specific to JS (frequently tested):**
```js
arr.includes(x);           // O(n) — linear scan
arr.indexOf(x);              // O(n) — linear scan
set.has(x);                   // O(1) average — Set uses hashing internally
obj[key];                      // O(1) average — object property access is hash-based

arr.unshift(x);               // O(n) — must re-index EVERY existing element
arr.push(x);                    // O(1) amortized — adds to the end, no re-indexing

// A classic accidental O(n²): repeatedly using includes() inside a loop over another array
function intersect(a, b) {
  return a.filter(x => b.includes(x)); // O(n * m) — for each of n items, scan all of m
}
// Fix: convert b to a Set first for O(1) lookups -> overall O(n + m)
function intersectFast(a, b) {
  const setB = new Set(b);
  return a.filter(x => setB.has(x));
}
```

**Why this is a favorite senior/FAANG interview angle:** it tests whether a candidate reaches for "correct" code (`includes` in a loop) vs. "correct AND efficient" code (`Set`-based lookup) — a very realistic, common code-review-level distinction, not just abstract algorithms trivia.

**Interview questions:**
- *Mid:* What's the time complexity of `Array.prototype.includes`? Of `Set.prototype.has`?
- *Senior/FAANG:* Given the `intersect` function above operating on two large arrays, identify the complexity problem and fix it — a genuinely common real interview question, testing both Big O intuition and JS-specific data structure knowledge.

---

## Practical exercises — Chapter 11

1. Implement `throttle` and `debounce` from memory, side by side, and write a one-sentence rule for choosing between them.
2. **Debugging exercise:** a production app's memoization cache (using the naive `memoize` from §11.4) causes a slow memory leak over days of uptime on a long-running server process. Redesign it with an LRU eviction policy (bounded size).
3. **Small challenge:** rewrite `intersectFast` from §11.6 to also preserve the *count* of duplicate matches correctly, still in better than O(n·m).
4. **Advanced challenge:** design a virtualized list rendering strategy (only render DOM nodes for currently-visible rows out of a 100,000-item list) — explain how this combines ideas from event delegation, lazy loading, and Big O awareness into one real system.

---

**Next:** Chapter 12 — Modern JS & Interview Synthesis (ES6→present feature tour, cross-topic FAANG scenarios, debugging drills, and the final integrated roadmap). This is the capstone chapter. Say **"next"** when ready.
