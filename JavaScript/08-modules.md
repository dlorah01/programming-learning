# Chapter 8 — Modules

> Depends on: Ch.1 §1.6 (browser vs. Node have historically had different module systems), Ch.6 (dynamic `import()` returns a Promise).
> Feeds into: Ch.11 (tree shaking is a core bundle-size performance technique).

---

## 8.1 Why Modules Exist at All

**Intuitive version:** Before modules, every script on a page shared **one single global scope** — every `<script>` tag's variables and functions collided in the same namespace. Modules give each file its own private scope by default, with an explicit mechanism to share things between files.

**Formal version:** a module is a file with its own top-level scope; nothing declared in it is visible elsewhere unless explicitly exported and imported.

```html
<!-- Pre-module era: script1.js and script2.js share ONE global scope -->
<script src="script1.js"></script> <!-- declares `let x = 1` -->
<script src="script2.js"></script> <!-- `let x = 2` here -> SyntaxError: Identifier 'x' has already been declared -->
```

**Why this mattered enough to add two competing systems (CommonJS then ESM):** as JS moved from "sprinkle a bit of interactivity on a page" to "build entire large applications" (Node.js in 2009, large SPAs later), global-scope collisions and unclear dependency ordering became unmanageable at scale.

---

## 8.2 CommonJS (CJS) — Node's Original Module System

**Intuitive version:** Node needed a module system in 2009, years before ES Modules existed in the spec (2015) or had real engine support (much later). CommonJS was the pragmatic solution: `require()` to import, `module.exports` to export.

**Formal version:**
```js
// math.js
function add(a, b) { return a + b; }
module.exports = { add };

// app.js
const { add } = require("./math.js");
```

**Key mechanical properties (frequently tested):**
- **Synchronous** — `require()` blocks until the required file is fully loaded and executed. Fine for local disk reads (Node), unworkable for the browser's network-based loading.
- **Loaded and executed once, then cached** — subsequent `require()` calls for the same file return the same cached `module.exports` object, not a re-execution.
- **Dynamic by nature** — `require(someVariable)` is completely valid; the module to load can be computed at runtime.
- **Exports are copied values at require time** (with an important nuance: if you export an object, you get a reference to that live object, so later mutations to it ARE visible — but reassigning the exported binding itself is not reflected, unlike ESM's live bindings, §8.3).

**Interview questions:**
- *Beginner:* What function does CommonJS use to import a module?
- *Mid:* Why is `require()` synchronous, and why would that be a problem in a browser?

---

## 8.3 ES Modules (ESM) — the Language-Native Standard

**Intuitive version:** ES2015 added `import`/`export` directly into the JavaScript language spec itself (not a library convention like CommonJS) — designed from the start to support both synchronous-feeling static analysis AND asynchronous loading (crucial for the browser, where files come over the network).

**Formal version:**
```js
// math.mjs
export function add(a, b) { return a + b; }
export const PI = 3.14159;
export default function multiply(a, b) { return a * b; } // one default export per module

// app.mjs
import multiply, { add, PI } from "./math.mjs";
```

**Key mechanical properties (the differences from CommonJS are a favorite interview topic):**

| | CommonJS | ES Modules |
|---|---|---|
| Loading | Synchronous | Asynchronous (supports both static and dynamic loading) |
| When resolved | Runtime | **Statically analyzed at parse time**, before execution — enables tree shaking |
| Bindings | Copied value (with object-reference caveat) | **Live bindings** — a reference to the actual variable, not a copy |
| `this` at top level | `module.exports` (an object) | `undefined` |
| Conditional imports | Trivial (`require` is just a function call) | Not directly — must use dynamic `import()` (§8.4) |

**Live bindings — the most important, most-tested mechanical difference:**
```js
// counter.mjs
export let count = 0;
export function increment() { count++; }

// app.mjs
import { count, increment } from "./counter.mjs";
console.log(count); // 0
increment();
console.log(count); // 1 — updated! ESM imports are LIVE references to the exporting module's binding,
                        // not a one-time copied snapshot (which is what CommonJS effectively gives you
                        // for primitive exports)
```

**Why static analysis matters (this is the entire reason tree shaking is possible, §8.4):** because `import`/`export` statements must appear at the top level with literal (non-computed) specifiers, a bundler can determine the *entire* dependency graph and exactly which exports are used, **without ever executing any code** — impossible with CommonJS's fully dynamic, runtime `require(computedPath)`.

**Interview questions:**
- *Mid:* What's a "live binding" and how does it differ from CommonJS's copied exports?
- *Senior:* Why can bundlers tree-shake ES Modules but not (easily) CommonJS modules?
- *Senior/FAANG:* Why is `import` required to be at the top level (not inside an `if` block), unlike `require`?

---

## 8.4 Dynamic `import()`

**Intuitive version:** Sometimes you genuinely need to load a module conditionally or lazily — ESM's static-only `import` can't do that, so the language added a separate, function-like `import()` that returns a Promise.

**Formal version:**
```js
button.addEventListener("click", async () => {
  const { openModal } = await import("./modal.js"); // loaded only when actually needed
  openModal();
});
```

**Why it exists:** enables **code splitting** — bundlers (Webpack, Vite, Rollup) recognize `import()` calls and automatically split that module into a separate chunk, loaded on demand rather than bundled into the initial page load. This directly powers route-based lazy loading in modern frontend frameworks (Ch.11).

**Interview questions:**
- *Mid:* What does dynamic `import()` return?
- *Senior:* How does dynamic `import()` enable code splitting, and why does that matter for initial page load performance?

---

## 8.5 Tree Shaking

**Intuitive version:** "Tree shaking" is a bundler optimization that removes exported code you never actually import/use, so your final bundle only ships what's needed — like shaking a tree so only the dead leaves (unused code) fall off.

**Formal version:** relies entirely on ESM's static structure (§8.3) — because a bundler can determine, without running any code, exactly which named exports are imported anywhere in the dependency graph, it can safely delete the rest during the production build.

```js
// utils.js — exports 50 functions
export function usedFunction() { /* ... */ }
export function neverUsedFunction() { /* ... */ } // ...49 more like this

// app.js
import { usedFunction } from "./utils.js"; // only this one import
// A tree-shaking bundler ships ONLY usedFunction (and its dependencies) in the final bundle,
// entirely eliminating neverUsedFunction and everything else
```

**Why this doesn't reliably work with CommonJS:** `module.exports` is just a regular JS object, potentially built up dynamically (`if (condition) { module.exports.extra = fn; }`) — a bundler can't safely prove which parts are "dead" without actually running the code, so it conservatively keeps everything.

**Real-world implication:** this is why library authors are strongly encouraged to publish ESM builds ("this package is tree-shakeable") — importing one function from a CommonJS-only utility library can accidentally pull the *entire* library into your bundle.

**Side effects break tree shaking (a very common real-world gotcha):**
```js
// polyfills.js
import "./patch-array-prototype"; // a side-effecting import with no named exports used
// bundlers must assume side-effecting modules run for a REASON and can't be safely removed,
// even if nothing is explicitly imported from them — unless package.json declares "sideEffects": false
```

**Interview questions:**
- *Mid:* What is tree shaking and what module system does it depend on?
- *Senior:* Why can a bundler safely tree-shake ESM but not CommonJS?
- *FAANG-style:* A teammate imports one function from a large utility library and notices the bundle size barely changes. What are the likely causes, and how would you diagnose it? (Answer: library is CommonJS-only, or has side effects preventing shaking, or `package.json` doesn't declare `"sideEffects": false` — diagnosed via a bundle analyzer like `webpack-bundle-analyzer` or `source-map-explorer`.)

---

## Practical exercises — Chapter 8

1. Explain, precisely, why the ESM live-binding example (`count`/`increment`) would behave differently if `counter.mjs` were rewritten as a CommonJS module exporting a primitive `count` directly.
2. **Debugging exercise:** a teammate imports a single icon component from an icon library, but the production bundle grows by 2MB. Walk through your diagnostic steps and likely root causes.
3. **Small challenge:** convert a small CommonJS module (with `require`/`module.exports`) to ES Modules, preserving identical exported behavior, including a default export.
4. **Advanced challenge:** design a route-based code-splitting strategy for a multi-page SPA using dynamic `import()`, explaining which chunks load eagerly vs. lazily and why.

---

**Next:** Chapter 9 — Error Handling (try/catch/finally, custom error types, async error propagation). Say **"next"** when ready.
