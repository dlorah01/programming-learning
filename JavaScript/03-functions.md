# Chapter 3 — Functions

> Depends on: Ch.2 §2.2 (scope chain), §2.3 (hoisting).
> Feeds into: Ch.4 (methods are functions attached to objects), Ch.5 (classes are sugar over functions + prototypes), Ch.6 (callbacks/promises are just functions passed around).

---

## 3.1 Function Declarations vs. Function Expressions vs. Arrow Functions

**Intuitive version:** Three ways to write "a reusable block of code," each with different hoisting behavior, different `this` binding, and different intended uses.

**Formal version:**
```js
function declaration() {}           // Function Declaration — fully hoisted (Ch.2 §2.3)
const expr = function () {};         // Function Expression — not hoisted (var name is, value isn't)
const named = function inner() {};   // Named Function Expression — name only visible inside itself (useful for recursion/stack traces)
const arrow = () => {};              // Arrow Function — no own `this`, no own `arguments`, cannot be a constructor
```

**Why arrow functions exist (ES6):** the single biggest pain point in pre-ES6 JS was `this` losing its intended value inside callbacks (`setTimeout`, array methods, event handlers), forcing everyone to write `const self = this;` or `.bind(this)` everywhere. Arrow functions solve this by **not having their own `this` at all** — they lexically inherit it from the enclosing scope, exactly like a normal variable would.

```js
// Pre-ES6 pain:
function Timer() {
  this.seconds = 0;
  setInterval(function () {
    this.seconds++; // `this` is NOT Timer here — it's the global object (or undefined in strict mode)
  }, 1000);
}

// Fixed with arrow:
function Timer() {
  this.seconds = 0;
  setInterval(() => {
    this.seconds++; // `this` is lexically inherited from Timer's `this` — correct
  }, 1000);
}
```

**Key differences table:**

| | Declaration | Expression | Arrow |
|---|---|---|---|
| Hoisted with body | Yes | No | No |
| Has own `this` | Yes (dynamic) | Yes (dynamic) | No (lexical) |
| Has own `arguments` | Yes | Yes | No |
| Usable as constructor (`new`) | Yes | Yes | No — throws |
| Has `prototype` property | Yes | Yes | No |

**When NOT to use arrow functions:** object methods that need `this` to refer to the object, and any function that needs to be a constructor or use `arguments`.
```js
const obj = {
  value: 42,
  getValue: () => this.value, // BUG: `this` is NOT obj — it's whatever enclosing scope has (often undefined/window)
  getValueCorrect() { return this.value; } // shorthand method — `this` correctly bound to obj at call time
};
```

**Interview questions:**
- *Beginner:* Name three ways to define a function in JS.
- *Mid:* Why do arrow functions not have their own `this`?
- *Senior/FAANG:* Given the `Timer` example above, explain exactly what `this` resolves to in the non-arrow version, in both non-strict and strict mode.

---

## 3.2 `this` — the Most Misunderstood Keyword in JS

**Intuitive version:** `this` is not fixed by *where* a function is written (unlike scope) — it's determined by **how the function is called**. Same function, called differently, gets a different `this`. Arrow functions are the one exception (§3.1).

**Formal version — the 4 (5) binding rules, in precedence order:**

1. **`new` binding** — `new Foo()` — `this` is the newly created object.
2. **Explicit binding** — `fn.call(obj)`, `fn.apply(obj)`, `fn.bind(obj)()` — `this` is whatever you pass.
3. **Implicit binding** — `obj.method()` — `this` is `obj` (whatever is left of the dot at call time).
4. **Default binding** — plain `fn()` call — `this` is `undefined` in strict mode, or the global object in sloppy mode.
5. **Arrow functions** — ignore all of the above; inherit `this` lexically from the enclosing scope.

```js
function show() { console.log(this.name); }

const a = { name: "A", show };
const b = { name: "B", show };

a.show();               // "A" — implicit binding
b.show();                // "B" — implicit binding
const detached = a.show;
detached();               // undefined / TypeError (strict) — default binding, lost the object!
detached.call(b);         // "B" — explicit binding overrides
const bound = a.show.bind(b);
bound();                   // "B" — bind is permanent, cannot be overridden even by later .call()
```

**Classic real-world bug — losing `this` when passing a method as a callback:**
```js
class Counter {
  count = 0;
  increment() { this.count++; }
}
const c = new Counter();
button.addEventListener("click", c.increment); // BUG: called as plain fn, `this` is undefined
button.addEventListener("click", () => c.increment()); // fix: arrow preserves `this` via closure
button.addEventListener("click", c.increment.bind(c));  // fix: explicit bind
```

**Mental model:** `this` is like a pronoun ("I", "me") — its meaning depends entirely on who is speaking (how the function was invoked) at that moment, not where the sentence (function) was written. Arrow functions are the exception: they always refer back to whoever was speaking in the surrounding context, like an unchangeable quote.

**Interview questions:**
- *Beginner:* What determines the value of `this` in a regular function?
- *Mid:* Rank `new`, `bind`, implicit, and default binding by precedence and prove it with an example.
- *Senior/FAANG:* Why does destructuring a method off an object (`const { method } = obj`) break `this`, and what are three ways to fix it?

---

## 3.3 `call`, `apply`, `bind`

**Intuitive version:** Three tools for explicitly controlling what `this` a function runs with.

**Formal version:**
```js
fn.call(thisArg, arg1, arg2);     // invokes immediately, args listed individually
fn.apply(thisArg, [arg1, arg2]);  // invokes immediately, args as an array
const bound = fn.bind(thisArg, arg1); // returns a NEW function, permanently bound, not invoked yet
```

**Why they exist:** before arrow functions and classes, these were the primary mechanisms for explicit `this` control and for function borrowing (using one object's method on another object without copying code).

```js
function greet(greeting) { return `${greeting}, ${this.name}`; }
const person = { name: "Ana" };
greet.call(person, "Hi");     // "Hi, Ana"
greet.apply(person, ["Hi"]);  // "Hi, Ana"  (same, array form)
const greetAna = greet.bind(person);
greetAna("Hi");                 // "Hi, Ana"
```

**Real-world pattern — function borrowing:**
```js
function sum() {
  return Array.prototype.reduce.call(arguments, (a, b) => a + b, 0);
  // borrowing Array's reduce on the array-like `arguments` object
}
```

**Performance note:** `apply` with a spread of large arrays used to have call-stack-size limits and slight perf overhead vs `call`; in modern engines the difference is negligible for most code — don't over-optimize this without profiling.

**Interview questions:**
- *Beginner:* What's the difference between `call` and `apply`?
- *Mid:* Why does `bind` return a new function instead of invoking immediately?
- *Senior:* Implement your own `myBind` from scratch (classic FAANG live-coding question):
  ```js
  Function.prototype.myBind = function (thisArg, ...boundArgs) {
    const fn = this;
    return function (...args) {
      return fn.apply(thisArg, [...boundArgs, ...args]);
    };
  };
  ```

---

## 3.4 Closures

**Intuitive version:** A closure is a function that "remembers" the variables from the scope it was created in, even after that outer scope has finished executing. It's the scope chain (Ch.2 §2.2) made persistent.

**Formal version:** When a function is created, it retains a reference to its lexical environment. As long as any inner function still references an outer variable, that variable is kept alive in memory (not garbage collected) even after the outer function returns — this is a closure.

```js
function makeCounter() {
  let count = 0;              // this variable would normally die when makeCounter() returns
  return function () {
    count++;                   // but this returned function "closes over" it, keeping it alive
    return count;
  };
}
const counter = makeCounter();
counter(); // 1
counter(); // 2
counter(); // 3 — count persists between calls, private to this counter instance
```

**Why closures exist:** they're a direct, unavoidable consequence of lexical scoping (Ch.2) combined with functions being first-class values that can be returned/passed around. JS didn't "add" closures as a feature — they fall out naturally once you have first-class functions + lexical scope + garbage collection that respects live references.

**Real-world uses:** data privacy/encapsulation (before private class fields existed), memoization (Ch.11), currying (§3.6), event handler state, the module pattern, debounce/throttle implementations (Ch.11).

**Classic interview trap — closures in loops (same root cause as Ch.2's `var`/`let` example):**
```js
function attachHandlers() {
  const buttons = document.querySelectorAll("button");
  for (var i = 0; i < buttons.length; i++) {
    buttons[i].addEventListener("click", () => console.log(i));
  }
  // Every handler logs the SAME final value of i — one shared closure variable
}
// Fix: use `let` (new binding per iteration) or an IIFE to create a new scope per iteration
```

**Common misconception:** "closures copy the variable's value at creation time." False — closures capture a *live reference* to the variable, not a snapshot. This is exactly why the loop bug above happens.

**Memory implication (ties to Ch.7):** closures are a common source of memory leaks — if a long-lived closure holds a reference to a large object that's no longer needed, that object can't be garbage collected.

**Interview questions:**
- *Beginner:* What is a closure? Give a simple example.
- *Mid:* Why does the `var` loop example log the same number for every handler?
- *Senior:* Do closures capture values or references? Prove it with code.
- *FAANG-style:* Implement a `once(fn)` function that ensures `fn` only ever runs one time, using a closure:
  ```js
  function once(fn) {
    let called = false, result;
    return (...args) => {
      if (!called) { called = true; result = fn(...args); }
      return result;
    };
  }
  ```

---

## 3.5 Higher-Order Functions & Callbacks

**Intuitive version:** A higher-order function (HOF) either takes a function as an argument, returns a function, or both. `map`, `filter`, `reduce`, `setTimeout`, and `addEventListener` are all HOFs.

**Formal version:** possible because functions are **first-class citizens** in JS — they can be assigned to variables, passed as arguments, returned from other functions, and stored in data structures, just like any other value.

```js
const double = x => x * 2;
const isEven = x => x % 2 === 0;

[1,2,3,4,5].map(double).filter(isEven); // [2, 4, 6, 8, 10] -> filter -> [2,4,6,8,10] all even actually
```

**`reduce` as the "universal" HOF (senior-level insight):** `map` and `filter` can both be implemented in terms of `reduce` — a favorite "prove you understand the abstraction" interview question:
```js
const map = (arr, fn) => arr.reduce((acc, x) => [...acc, fn(x)], []);
const filter = (arr, fn) => arr.reduce((acc, x) => fn(x) ? [...acc, x] : acc, []);
```

**Interview questions:**
- *Beginner:* What makes a function "higher-order"?
- *Mid:* Implement `Array.prototype.map` from scratch without using any built-in iteration method other than a loop.
- *Senior:* Implement `filter` using only `reduce`.

---

## 3.6 Currying, Partial Application, and Composition

**Intuitive version:** Currying transforms a function that takes multiple arguments into a sequence of functions that each take one argument. Partial application pre-fills *some* arguments now, leaving the rest for later. Composition chains functions so the output of one feeds into the next.

**Formal version:**
```js
// Normal
function add3(a, b, c) { return a + b + c; }

// Curried
const curriedAdd3 = a => b => c => a + b + c;
curriedAdd3(1)(2)(3); // 6

// Partial application (fixes some args, not necessarily one at a time)
function partial(fn, ...fixedArgs) {
  return (...rest) => fn(...fixedArgs, ...rest);
}
const add5 = partial(add3, 2, 3);
add5(10); // 15

// Generic curry (handles any arity)
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) return fn(...args);
    return (...more) => curried(...args, ...more);
  };
}
```

**Why they exist / when to use:** currying enables clean function reuse and configuration — e.g., `const double = multiply(2)` reads like a specialized, reusable function rather than a generic one you re-supply arguments to every time. It's central to functional-programming-style JS (Ramda, lodash/fp, Redux middleware signatures like `store => next => action => {}` are curried functions).

**Composition:**
```js
const compose = (...fns) => x => fns.reduceRight((acc, fn) => fn(acc), x);
const pipe = (...fns) => x => fns.reduce((acc, fn) => fn(acc), x);

const shout = s => s.toUpperCase() + "!";
const exclaim = s => s + "!!!";
pipe(shout, exclaim)("hello"); // "HELLO!!!!"
```

**When NOT to use heavy currying:** in typical imperative/OOP codebases, over-currying hurts readability and debuggability for teams unfamiliar with FP style — use judiciously, not everywhere.

**Interview questions:**
- *Mid:* What's the difference between currying and partial application?
- *Senior:* Implement a generic `curry` that works for any function arity (shown above) — very common FAANG live-coding question.
- *FAANG-style:* Implement `pipe`/`compose` and explain the difference in evaluation order between them.

---

## 3.7 Recursion

**Intuitive version:** A function that calls itself to break a problem into smaller identical subproblems, until a base case stops the recursion.

**Formal version:** every recursive call adds a new frame to the **call stack** (Ch.1 §1.7). Without a correctly reached base case, this grows unbounded → **"Maximum call stack size exceeded"** (stack overflow).

```js
function factorial(n) {
  if (n <= 1) return 1;       // base case
  return n * factorial(n - 1); // recursive case
}
```

**Tail Call Optimization (TCO) — important nuance:** ES6 spec *allows* engines to optimize tail-position recursive calls to avoid growing the stack, but **V8 has never shipped this** — so in practice, deep recursion in Node/Chrome can still overflow the stack even in "tail call" form. Know this for interviews: it's a commonly cited "spec says X, but real engines do Y" gotcha.

```js
// "Tail-recursive" form — spec-legal for TCO, but V8 does NOT optimize this
function factorialTCO(n, acc = 1) {
  if (n <= 1) return acc;
  return factorialTCO(n - 1, n * acc); // tail position, but still overflows in V8 for large n
}
```

**Interview questions:**
- *Beginner:* What is a base case and why is it required?
- *Mid:* What causes "Maximum call stack size exceeded"?
- *Senior/FAANG:* Does V8 implement tail call optimization? What does that mean for writing deeply recursive code in production Node apps? (Answer: convert to iteration, or use trampolining, for anything with unbounded depth.)

---

## Practical exercises — Chapter 3

1. Predict the `this` value in each case, then verify:
   ```js
   const obj = {
     name: "X",
     regular: function () { return this.name; },
     arrow: () => this.name,
   };
   const { regular, arrow } = obj;
   console.log(obj.regular(), obj.arrow(), regular(), arrow());
   ```
2. **Debugging exercise:** a React-style class component has `this.handleClick` passed directly to an event listener and `this` is `undefined` inside it at runtime. Give two idiomatic fixes.
3. **Small challenge:** implement `debounce(fn, delay)` using a closure (preview of Ch.11, but the core mechanism is pure closures):
   ```js
   function debounce(fn, delay) {
     let timer;
     return (...args) => {
       clearTimeout(timer);
       timer = setTimeout(() => fn(...args), delay);
     };
   }
   ```
4. **Advanced challenge:** implement `curry` (as above) AND a `memoize(fn)` HOF that caches results by stringified arguments, then combine them: a curried, memoized `add(a)(b)(c)`.

---

**Next:** Chapter 4 — Objects & Prototypes (property descriptors, the prototype chain, `Object.create`, `Proxy`/`Reflect`). This is where "everything in JS is an object" gets its real mechanical explanation. Say **"next"** when ready.
