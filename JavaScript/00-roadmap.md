# JavaScript Mastery Curriculum — Roadmap & Index

This is the master index for your full rebuild of JavaScript, from internals to senior-level system thinking. Each chapter ships as its own file so you can read, reference, and revisit independently. This file is the map — it shows how every chapter depends on and feeds into the others, so the language stops feeling like a pile of trivia and starts feeling like one coherent machine.

## How the chapters connect

```
FOUNDATIONS (engines, parsing, execution model)
        │
        ▼
LANGUAGE FUNDAMENTALS (scope, hoisting, TDZ, types, coercion)
        │
        ├──────────────┬───────────────┐
        ▼              ▼               ▼
    FUNCTIONS       OBJECTS        MEMORY MODEL
   (closures,     (prototypes,    (stack/heap, GC,
    this, HOFs)    descriptors)    WeakMap/WeakSet)
        │              │               │
        └──────┬───────┴───────┬───────┘
               ▼               ▼
           CLASSES        ASYNC JS
        (sugar over      (event loop,
         prototypes)      promises, async/await)
               │               │
               └───────┬───────┘
                       ▼
              MODULES & ERROR HANDLING
                       │
                       ▼
             ADVANCED / METAPROGRAMMING
          (Symbols, Proxy, Reflect, iterators)
                       │
                       ▼
                  PERFORMANCE
          (memoization, event loop tuning,
           GC-aware code, Big O in practice)
                       │
                       ▼
            MODERN JS & INTERVIEW SYNTHESIS
```

**The core insight the whole curriculum builds toward:** almost every "weird" JS behavior — hoisting, `this`, coercion, closures, the event loop — is a direct, logical consequence of two facts: (1) JS was written in 10 days and had to stay backward-compatible forever, and (2) the engine's execution model is single-threaded with a very specific set of phases (parse → compile → execute, with a run-to-completion event loop). Once those two facts are real to you, nothing in the language is "magic" anymore.

## Chapter list (delivered as separate files)

1. **01-foundations.md** ✅ — History, ECMAScript, engines (V8/SpiderMonkey/JSC), parsing, compilation vs interpretation, JIT, runtime vs engine, browser vs Node, the execution model
2. **02-language-fundamentals.md** — variables, scope, hoisting, TDZ, data types, primitives vs objects, boxing, coercion, equality, truthy/falsy, operators
3. **03-functions.md** — declarations/expressions/arrows, `this`, `bind`/`call`/`apply`, closures, lexical scope, HOFs, currying, composition, recursion
4. **04-objects-and-prototypes.md** — creation patterns, property descriptors, getters/setters, enumerability, prototype chain, `Object.create`, `Reflect`, `Proxy`
5. **05-classes.md** — constructor functions, ES6 classes, inheritance, polymorphism, encapsulation, private fields, statics
6. **06-async-javascript.md** — call stack, Web APIs, task queue vs microtask queue, event loop, Promises, async/await, generators, async generators, `fetch`, `AbortController`
7. **07-memory.md** — stack vs heap, references, garbage collection algorithms, memory leaks, WeakMap/WeakSet
8. **08-modules.md** — CommonJS vs ESM, dynamic imports, tree shaking
9. **09-error-handling.md** — try/catch/finally, throw, error types, custom errors
10. **10-advanced-metaprogramming.md** — Symbols, iterables/iterators, generators as iterators, Proxy/Reflect deep dive, decorators proposal
11. **11-performance.md** — event delegation, debounce/throttle, memoization, lazy loading, code splitting, Big O in real code
12. **12-modern-js-and-interview-synthesis.md** — ES6→ES2025 feature tour, cross-topic FAANG-style scenarios, "explain what happens" debugging drills, final integrated roadmap review

Each chapter follows the same template: intuitive explanation → formal explanation → why it exists → internals → examples (easy → hard) → mental model → common mistakes/misconceptions → edge cases → best practices → performance notes → when NOT to use it → interview questions (beginner/mid/senior/FAANG-style) → exercises/challenges.

## How we'll proceed

Say **"next"** and I'll produce Chapter 2, and so on — each as its own file. You can also jump around ("do Chapter 6 next," or "go deeper on closures before we move on") and I'll adjust. Chapter 1 is attached now.
