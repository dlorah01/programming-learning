---
layout: default
title: "Chapter 10 — Advanced Topics & Metaprogramming"
---

# Chapter 10 — Advanced Topics & Metaprogramming

> Depends on: Ch.4 §4.6–4.7 (Reflect/Proxy), Ch.6 §6.5 (generators).
> Feeds into: Ch.11 (lazy evaluation via iterators/generators is a performance technique), real-world library/framework internals.

---

## 10.1 Symbols

**Intuitive version:** A `Symbol` is a primitive value that's guaranteed to be unique — even two symbols created with the identical description are never equal. They exist to create object keys that can't accidentally collide with string keys, ever.

**Formal version:**
```js
const s1 = Symbol("id");
const s2 = Symbol("id");
s1 === s2; // false — always unique, even with the same description

const obj = { [s1]: "value" };
obj[s1]; // "value"
Object.keys(obj);        // [] — symbols are NOT included in normal enumeration
JSON.stringify(obj);       // "{}" — symbols are skipped entirely
```

**Why Symbols exist:** to safely add metadata/behavior to objects (including built-ins you don't own, like `Array.prototype`) without any risk of clashing with existing or future string property names. This is exactly how JS itself extends built-in protocols non-invasively — see `Symbol.iterator` below.

**Well-known symbols (the practical part interviews focus on):**
```js
Symbol.iterator;    // defines how an object behaves in for...of, spread, destructuring
Symbol.asyncIterator; // the async version, used by for await...of
Symbol.toPrimitive;   // customizes an object's coercion behavior (Ch.2 §2.7)
Symbol.hasInstance;    // customizes `instanceof` behavior
```

```js
const obj = {
  value: 42,
  [Symbol.toPrimitive](hint) {
    if (hint === "number") return this.value;
    if (hint === "string") return `Value(${this.value})`;
    return "default";
  },
};
+obj;             // 42 — hint "number"
`${obj}`;          // "Value(42)" — hint "string"
```

**Interview questions:**
- *Mid:* Why doesn't `Object.keys` include Symbol-keyed properties?
- *Senior:* Name a well-known Symbol and explain what protocol it powers.

---

## 10.2 The Iterable & Iterator Protocols

**Intuitive version:** These are the two "interfaces" that make `for...of`, spread (`...`), and destructuring work on *any* object, not just arrays — including your own custom data structures.

**Formal version:**
- **Iterator protocol:** an object with a `.next()` method returning `{ value, done }`.
- **Iterable protocol:** an object with a `[Symbol.iterator]` method that returns an iterator.

```js
// Built-in example: arrays are iterable because Array.prototype[Symbol.iterator] exists
const arr = [1, 2, 3];
const it = arr[Symbol.iterator]();
it.next(); // { value: 1, done: false }
it.next(); // { value: 2, done: false }
it.next(); // { value: 3, done: false }
it.next(); // { value: undefined, done: true }

// Making a CUSTOM object iterable — this is what unlocks for...of, spread, destructuring on it
class Range {
  constructor(start, end) { this.start = start; this.end = end; }
  [Symbol.iterator]() {
    let current = this.start, end = this.end;
    return {
      next() {
        return current <= end
          ? { value: current++, done: false }
          : { value: undefined, done: true };
      },
    };
  }
}
[...new Range(1, 5)]; // [1, 2, 3, 4, 5] — spread works because Range implements the protocol
for (const n of new Range(1, 3)) console.log(n); // 1, 2, 3
```

**Why this matters practically:** `for...of`, spread, destructuring, `Array.from`, and `Promise.all` (when given a non-array iterable) all rely on this exact protocol — implement it once on your custom class and it works everywhere those language features expect an iterable, for free.

**Interview questions:**
- *Mid:* What's the difference between the iterable protocol and the iterator protocol?
- *Senior:* Implement a custom iterable class (like `Range` above) from scratch — very common live-coding question.

---

## 10.3 Generators as Iterators (much simpler than hand-writing the protocol)

**Intuitive version:** Generators (Ch.6 §6.5) automatically implement the iterator protocol — `.next()`/`{value, done}` is exactly what a generator's returned object gives you for free — so they're by far the easiest way to make something iterable.

```js
class Range {
  constructor(start, end) { this.start = start; this.end = end; }
  *[Symbol.iterator]() {           // generator method — MUCH less boilerplate than §10.2's version
    for (let i = this.start; i <= this.end; i++) yield i;
  }
}
[...new Range(1, 5)]; // [1, 2, 3, 4, 5] — same result, far less code
```

**Why prefer generators for this:** hand-writing the iterator protocol (§10.2) requires manually tracking state (`current`) across calls; a generator function does this automatically via its own execution-pause mechanism — dramatically less error-prone.

**Interview questions:**
- *Senior:* Rewrite the manual `Range` iterator (§10.2) using a generator method, and explain why it's less error-prone.

---

## 10.4 Meta-Programming: Deeper `Proxy`/`Reflect` Patterns

**Intuitive version (recap of Ch.4 §4.6–4.7, extended):** meta-programming means writing code that manipulates the *behavior* of other code/objects generically, rather than manipulating specific values directly.

**Pattern — a validation layer via `Proxy` (real-world use, e.g. API response shape enforcement):**
```js
function createValidatedObject(schema) {
  return new Proxy({}, {
    set(target, prop, value) {
      if (schema[prop] && typeof value !== schema[prop]) {
        throw new TypeError(`${String(prop)} must be a ${schema[prop]}`);
      }
      return Reflect.set(target, prop, value);
    },
  });
}
const user = createValidatedObject({ name: "string", age: "number" });
user.name = "Ana";  // fine
user.age = "old";    // TypeError — caught at the moment of assignment, not later
```

**Pattern — auto-memoizing function wrapper via `Proxy`'s `apply` trap:**
```js
function memoizeProxy(fn) {
  const cache = new Map();
  return new Proxy(fn, {
    apply(target, thisArg, args) {
      const key = JSON.stringify(args);
      if (cache.has(key)) return cache.get(key);
      const result = Reflect.apply(target, thisArg, args);
      cache.set(key, result);
      return result;
    },
  });
}
```

**Interview questions:**
- *Senior/FAANG:* Design a `Proxy`-based read-only "deep freeze" wrapper that throws when any nested property is mutated (a common senior take-home/whiteboard exercise, testing recursive `Proxy` wrapping on nested `get` results).

---

## 10.5 Decorators (Stage 3 Proposal / Now Standardized in Recent ECMAScript)

**Intuitive version:** Decorators let you attach reusable, declarative behavior to a class or its members using `@decoratorName` syntax — without manually rewriting the class body each time (logging, memoization, validation, dependency injection, etc.).

**Formal version (current, Stage 3-derived syntax):**
```js
function logged(originalMethod, context) {
  const methodName = String(context.name);
  return function (...args) {
    console.log(`Calling ${methodName} with`, args);
    return originalMethod.call(this, ...args);
  };
}

class Calculator {
  @logged
  add(a, b) { return a + b; }
}
new Calculator().add(2, 3); // logs "Calling add with [2, 3]", then returns 5
```

**Why they exist / historical context:** decorators were heavily popularized in JS by TypeScript and frameworks like Angular years *before* being an official part of the language, using an earlier, incompatible experimental proposal — the standardized version (which reached Stage 3 and is landing in engines/TS as of recent releases) has a different underlying design. Interview-worthy nuance: "I've seen `@Component` in Angular" and "the standardized decorators proposal" are not guaranteed to be the exact same mechanism — Angular's predates and differs from the finalized TC39 design.

**When NOT to use them:** for one-off logic that doesn't repeat across multiple classes/methods — decorators add indirection that isn't worth it unless the same cross-cutting behavior (logging, caching, validation) is genuinely reused.

**Interview questions:**
- *Mid:* What problem do decorators solve that plain higher-order functions/wrapper methods don't as cleanly?
- *Senior:* Why might "the decorators you've seen in Angular" differ from "the standardized decorators proposal"? (Historical/versioning nuance — good for showing awareness of the language's evolution, tying back to Ch.1's TC39 process.)

---

## Practical exercises — Chapter 10

1. Implement `[Symbol.iterator]` manually (no generator) for a `LinkedList` class so it supports `for...of` and spread.
2. Rewrite exercise 1 using a generator method instead — compare the line count and clarity.
3. **Small challenge:** implement a `Proxy`-based `readOnly(obj)` wrapper that throws a `TypeError` on any attempted `set`, `deleteProperty`, or `defineProperty`.
4. **Advanced challenge:** implement the recursive deep-freeze `Proxy` described in §10.4's interview question — nested objects returned from `get` should also be wrapped, so mutation attempts fail at any depth, not just the top level.

---

**Next:** Chapter 11 — Performance (event delegation, debounce/throttle, memoization, lazy loading, code splitting, Big O in real code). Say **"next"** when ready.
