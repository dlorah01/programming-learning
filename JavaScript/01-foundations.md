---
layout: default
title: "Chapter 1 — Foundations"
---

# Chapter 1 — Foundations

> Depends on: nothing (this is bedrock).
> Feeds into: everything. Hoisting, closures, the event loop, and coercion all make sense once you internalize how JS actually runs.

---

## 1.1 History of JavaScript (why the language is shaped the way it is)

**Intuitive version:** JavaScript was designed in 10 days by Brendan Eich at Netscape in 1995, under the mandate that it "look like Java" (marketing), be easy for non-programmers to use in HTML pages, and never break existing web pages once shipped. Almost every "weird" part of JS — `typeof null === "object"`, loose equality, automatic semicolon insertion, `var` hoisting — is a scar from that rushed origin plus 30 years of *never being allowed to remove anything*, because the web has no version flag: your browser today still runs code written in 1996.

**Formal version:** JavaScript is a multi-paradigm (imperative, functional, prototype-based OOP), dynamically-typed, single-threaded (per realm), garbage-collected, ECMAScript-conformant scripting language.

**Why it exists:** Netscape needed a lightweight scripting language to make web pages interactive, competing against Microsoft's VBScript. Sun Microsystems' Java was the “serious” language for applets; JavaScript was positioned as its scripting sidekick (hence the confusing name — the languages are unrelated).

**Timeline that matters for interviews:**
- 1995 — Mocha/LiveScript/JavaScript, 10-day design by Brendan Eich
- 1996 — Microsoft ships JScript (reverse-engineered), starts the browser wars
- 1997 — Standardized as **ECMAScript** (ES1) via ECMA International, because Netscape didn't own the trademark "JavaScript" (Sun did)
- 1999 — ES3: regex, try/catch, better string handling — this became the de facto web standard for a decade
- 2008 — ES4 abandoned (too ambitious: static types, classes, modules — politically split the committee)
- 2009 — **ES5**: strict mode, JSON support, `Array` methods (`map`/`filter`/`reduce`), `Object.defineProperty`
- 2015 — **ES6/ES2015**: the big one — `let`/`const`, arrow functions, classes, promises, modules, destructuring, template literals, generators
- 2016–present — **yearly release cadence** (ES2016, ES2017...), each adding a small, finalized set of features via the TC39 proposal process (Stage 0 → 4)

**Common misconception:** "JavaScript and Java are related." They are not — the name was a marketing decision. Also common: "ECMAScript and JavaScript are different languages." They aren't — ECMAScript is the *specification*, JavaScript (along with JScript, ActionScript) is an *implementation* of it.

**Interview questions:**
- *Beginner:* What's the difference between JavaScript and ECMAScript?
- *Mid:* Why does JS still support `var` and old quirks like `==`? Why not just remove them?
- *Senior:* Explain why ES4 failed and what that tells you about how TC39 operates today (consensus-based, Stage 0–4 process, "don't break the web").

---

## 1.2 ECMAScript & TC39

**Intuitive version:** ECMAScript is the rulebook; JavaScript engines are different students all trying to follow the same rulebook correctly (with engine-specific extensions, like Node's `process` or the browser's `window`).

**Formal version:** ECMA-262 is the specification maintained by **TC39**, a committee of browser vendors, framework authors, and invited experts. Proposals move through 5 stages:
- **Stage 0** — strawperson (any idea)
- **Stage 1** — proposal (problem is described, championed by a TC39 member)
- **Stage 2** — draft (syntax finalized)
- **Stage 3** — candidate (spec-complete, needs real-world implementation feedback)
- **Stage 4** — finished, ships in the next yearly ECMAScript edition

**Why it matters practically:** when you see a feature and wonder "can I use this in production," check its stage. Stage 3 features often already work behind flags or via Babel; Stage 4 is safe everywhere modern.

**Real-world example:** the `Array.prototype.at()` method, optional chaining (`?.`), and nullish coalescing (`??`) all went through this exact pipeline before landing in ES2020/ES2022.

**Interview questions:**
- *Mid:* What does it mean for a proposal to be "Stage 3"? Would you use it in production?
- *Senior:* How does TC39's consensus model (any member can block) shape how conservatively JS evolves compared to a BDFL-run language like Python?

---

## 1.3 JavaScript Engines (V8, SpiderMonkey, JavaScriptCore)

**Intuitive version:** An engine is a program written in C++ (or Rust, increasingly) whose entire job is: take JS source text, and make it *actually run* — fast. Different browsers ship different engines, the way different card games can use different physical decks as long as the rules match.

**Formal version:** A JS engine implements the ECMAScript spec: it parses source into an AST, compiles/interprets it into some executable form (bytecode and/or machine code), and manages memory (heap, GC) and execution (call stack).

| Engine | Made by | Used in |
|---|---|---|
| **V8** | Google | Chrome, Edge, Node.js, Deno |
| **SpiderMonkey** | Mozilla | Firefox (the *original* JS engine, 1995) |
| **JavaScriptCore (Nitro)** | Apple | Safari, all iOS browsers (WebKit) |

**Why multiple engines exist:** competition drove massive performance gains in the 2008–2012 "browser wars" (V8's release in Chrome specifically triggered this) — and no single vendor controls the web, by design.

**How V8 works internally (the one you'll be asked about most):**
1. **Parser** produces an AST (Abstract Syntax Tree) from source text.
2. **Ignition** (V8's interpreter) compiles the AST to bytecode and starts executing immediately — fast startup, no wait for optimization.
3. V8 profiles the running code (which functions are "hot" — called often, with stable argument types).
4. **TurboFan** (V8's optimizing JIT compiler) recompiles hot functions into highly optimized machine code, using assumptions about types ("this function always receives numbers").
5. If those assumptions break later (someone passes a string instead) — a **deoptimization** ("bailout") occurs: V8 throws away the optimized code and falls back to the interpreter, then re-profiles.

**Mental model:** think of Ignition as a diligent but slow intern who does the work correctly every time, and TurboFan as an expert who watches the intern, learns the pattern, and starts doing the work in a highly specialized fast way — but if the input ever violates the pattern they assumed, they have to stop and hand it back to the intern.

**Real-world performance implication:** this is *why* "monomorphic" functions (always called with the same shape/type of argument) are faster than "polymorphic" ones. It's why frameworks and perf guides tell you to avoid changing an object's shape after creation, or mixing types in arrays.

```js
// Monomorphic — fast, TurboFan can specialize
function add(a, b) { return a + b; }
add(1, 2); add(3, 4); add(5, 6);

// Polymorphic/megamorphic — forces deopt-prone generic code
add(1, 2); add("a", "b"); add({}, []);
```

**Common mistakes:** assuming "the engine will optimize it anyway" for hot paths with unstable types; assuming all engines behave identically (they don't — GC pause behavior, JIT thresholds, and even `Array` method performance differ across V8/SpiderMonkey/JSC).

**Interview questions:**
- *Beginner:* Name two JS engines and where each is used.
- *Mid:* What is JIT compilation and why does it make JS both an "interpreted" and "compiled" language, depending on who you ask?
- *Senior:* Explain monomorphic vs polymorphic function calls and why this matters for V8 performance.
- *FAANG-style:* Why might a function that's fast in isolation become slow once integrated into a larger app? (Answer touches: inline caching invalidation, megamorphic call sites, deoptimization from unexpected types.)

---

## 1.4 Parsing, Compilation, and Interpretation

**Intuitive version:** Before your code runs even once, the engine has to *read* it into a structure it can understand — like a compiler reading a recipe before cooking anything.

**Formal version — the pipeline:**
1. **Tokenizing/Lexing** — raw text → tokens (`const`, `x`, `=`, `5`, `;`)
2. **Parsing** — tokens → **AST** (Abstract Syntax Tree), a nested structure representing the program's grammar
3. **Compilation** — AST → bytecode (Ignition) — note: this already makes V8 a "compiler," which surprises people who think JS is "purely interpreted"
4. **Execution** — bytecode runs on a virtual machine; hot paths get JIT-compiled to native machine code

**Is JavaScript "interpreted" or "compiled"?** Trick question, common in interviews. The honest answer: **JS is not defined by the spec as either** — that's an implementation detail. Modern engines are *both*: they interpret first (fast startup) and JIT-compile hot code (fast steady-state execution). This hybrid approach is called **JIT (Just-In-Time) compilation**.

**Why this design exists:** Pure interpretation is too slow for computation-heavy code (loops, math). Pure ahead-of-time compilation (like C) would mean waiting to compile the *entire* page's JS before anything runs — bad for a web page that needs to feel instant. JIT gives you the best of both: instant start, fast hot loops.

**Two-pass nature and hoisting connection:** Before execution, engines do a **creation phase** where they scan the current scope for function and variable declarations and allocate memory for them ahead of running any line-by-line code. This is *why* hoisting exists — it's not "moving code to the top," it's that declarations are registered in memory before execution begins. (Full depth in Chapter 2.)

```js
console.log(typeof greet); // "function" — already known before this line runs
function greet() { return "hi"; }
```

**Common misconceptions:**
- "JS source is compiled to machine code once at build/deploy time" — false, except in AOT tools like some bundlers/edge runtimes; standard engines compile at runtime, lazily, per function, based on actual usage.
- "Interpreted languages are always slower than compiled ones" — true in principle for naive interpreters, but JIT-compiled JS in V8 can approach C-like speeds for numeric hot loops.

**Interview questions:**
- *Beginner:* What is an AST?
- *Mid:* Is JavaScript compiled or interpreted? Defend your answer.
- *Senior:* Walk through what happens between hitting Enter on `node app.js` and your first `console.log` printing.

---

## 1.5 Runtime vs. Engine

**Intuitive version:** The **engine** is the brain that understands JS syntax and executes it. The **runtime** is the engine *plus* everything around it that lets JS actually do useful things — talk to a DOM, read a file, make an HTTP request — none of which are part of the ECMAScript spec itself.

**Formal version:** ECMAScript defines the *language* (syntax, types, `Array`, `Promise`, etc.) but not I/O. `setTimeout`, `fetch`, `document`, `fs`, `process` are **host environment APIs**, provided by the runtime (browser or Node), not the language spec.

**Why this separation exists:** It lets the same language run in wildly different environments — a browser tab, a server, an embedded IoT device, a CLI tool — each host providing only the APIs that make sense for it.

```
┌─────────────────────────────────────────┐
│              RUNTIME                     │
│  ┌───────────┐   ┌────────────────────┐ │
│  │  JS ENGINE│   │   Host APIs          │ │
│  │ (V8, etc.)│   │ (DOM / fs / fetch /  │ │
│  │  parses & │   │  setTimeout / etc.)  │ │
│  │  executes │   │                       │ │
│  └───────────┘   └────────────────────┘ │
│         Event Loop glues them together    │
└─────────────────────────────────────────┘
```

**Interview questions:**
- *Mid:* Is `setTimeout` part of JavaScript? Prove it.
- *Senior:* Why does `document` exist in the browser but not in Node, while `Array.prototype.map` exists in both?

---

## 1.6 Browser vs. Node.js

**Intuitive version:** Same engine-family concept (both commonly use V8), completely different job descriptions. A browser's job is to render untrusted, sandboxed web pages safely and interactively. Node's job is to let JS run as a general-purpose, trusted server/CLI language with filesystem and network access.

| | Browser | Node.js |
|---|---|---|
| Engine | V8 (Chrome/Edge), SpiderMonkey (Firefox), JSC (Safari) | V8 |
| Global object | `window` | `global` (`globalThis` works in both) |
| Modules (historically) | none native until ESM in browsers (`<script type="module">`) | CommonJS (`require`) by default, ESM supported |
| DOM access | yes | no |
| Filesystem access | no (sandboxed) | yes (`fs` module) |
| Security model | strict sandbox, same-origin policy | full OS-level trust (whatever the user running Node has) |

**Why it matters practically:** code using `window` breaks in Node; code using `require`/`fs` breaks in a browser bundle without polyfills. This is a very common source of "works on my machine" bugs when isomorphic (universal) code is shared between frontend and backend.

**Interview questions:**
- *Beginner:* Can Node.js access the DOM? Why or why not?
- *Mid:* What is `globalThis` and why was it introduced (ES2020)?
- *Senior:* You're writing an npm package meant to run in both browser and Node. What design precautions do you take?

---

## 1.7 The JavaScript Execution Model (the concept everything else in this curriculum sits on top of)

**Intuitive version:** JavaScript runs on a single thread, executing one thing at a time, using a **call stack** to track "what function is currently running and what called it." Anything that takes time (a timer, a network request) is handed off to the *runtime*, not the engine, so the single thread is never blocked waiting — this is the seed of the entire event loop model (full depth in Chapter 6).

**Formal version:** JS has a **run-to-completion** execution model: once a function starts executing, it runs entirely before any other JS code can interleave (no preemptive multitasking within a single realm). Concurrency is achieved via an event loop that schedules callbacks between synchronous executions, not via OS threads.

**Why this design exists:** the DOM is not thread-safe. If two threads could mutate a webpage simultaneously, you'd get race conditions on visible UI state constantly. A single-threaded, run-to-completion model trades raw parallelism for predictability — you never have to reason about another piece of JS interrupting yours mid-function.

**Mental model:** Picture a single chef (the call stack) in a kitchen who can only do one task at a time and finishes each task before starting the next. Orders that take a long time (a delivery, an oven timer) get handed off to a *waiter* (the runtime/Web APIs) who comes back and taps the chef on the shoulder only *after* the chef finishes whatever they're currently doing — that tap is a queued callback.

```js
console.log("1: sync start");
setTimeout(() => console.log("3: from setTimeout"), 0);
console.log("2: sync end");
// Output: 1, 2, 3 — even with a 0ms delay, because the callback
// waits for the call stack to be empty, no matter how short the timer.
```

**Common mistake:** believing `setTimeout(fn, 0)` runs "immediately." It runs *after* all currently queued synchronous code and (as you'll see in Ch. 6) after all queued microtasks.

**Interview questions:**
- *Beginner:* Is JavaScript single-threaded? What does that mean in practice?
- *Mid:* Why doesn't a `setTimeout(fn, 0)` execute immediately?
- *Senior:* Explain "run-to-completion" and why it's a deliberate tradeoff, not a limitation.
- *FAANG-style ("why is this happening?"):* Given a `while(true){}` loop with no `await` or yield inside it, why does the entire browser tab freeze, including scrolling and other timers? (Answer: single thread — nothing, not even rendering or other callbacks, can interleave with a function that never returns control to the stack.)

---

## Practical exercises — Chapter 1

1. **Explain out loud** (no notes) the full pipeline from `node app.js` to your first line of output. Aim for engine + runtime + parsing + execution model in one coherent 60-second answer.
2. **Debugging exercise:** a teammate says "I made this function faster by manually inlining it, but in production it's now *slower* than before." List 3 V8-specific reasons this could happen (hint: inline caching, monomorphism, deoptimization).
3. **Small challenge:** without running it, predict the output order:
   ```js
   console.log("A");
   setTimeout(() => console.log("B"), 0);
   console.log("C");
   ```
   Then explain *why* using the call stack / Web API / queue model — not just "I know the answer."
4. **Advanced challenge:** research and summarize, in your own words, the difference between how V8's Ignition+TurboFan pipeline works versus SpiderMonkey's tiered (Baseline Interpreter → Baseline JIT → Ion) pipeline. What's the shared philosophy across both?

---

**Next:** Chapter 2 — Language Fundamentals (scope, hoisting, TDZ, types, coercion) builds directly on the "creation phase before execution" idea from §1.4. Say **"next"** when ready.
