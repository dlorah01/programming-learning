---
layout: default
title: "Chapter 5 — Classes"
---

# Chapter 5 — Classes

> Depends on: Ch.4 (prototype chain, `Object.create`) — classes are syntax sugar over exactly that machinery.
> Feeds into: Ch.10 (decorators proposal builds on class syntax), real-world OOP design in application code.

---

## 5.1 Constructor Functions (the pre-ES6 way — still essential to understand)

**Intuitive version:** Before `class` existed (ES6, 2015), "classes" in JS were just regular functions used with `new`, combined with manually attaching shared methods to `.prototype`.

**Formal version:**
```js
function Animal(name) {
  this.name = name; // `new` creates a fresh object, binds `this` to it
}
Animal.prototype.speak = function () {
  return `${this.name} makes a sound.`;
};

const dog = new Animal("Rex");
```

**What `new` actually does, step by step (a very common interview question — implement it yourself):**
1. Creates a new empty object.
2. Sets that object's `[[Prototype]]` to `Constructor.prototype`.
3. Calls the constructor function with `this` bound to the new object.
4. If the constructor doesn't explicitly return an object, returns the new object automatically.

```js
function myNew(Constructor, ...args) {
  const obj = Object.create(Constructor.prototype); // steps 1 + 2
  const result = Constructor.apply(obj, args);        // step 3
  return (typeof result === "object" && result !== null) ? result : obj; // step 4
}
```

**Interview questions:**
- *Senior/FAANG:* Implement `new` from scratch without using the `new` keyword (shown above) — extremely common at top-tier companies.
- *Mid:* What happens if a constructor function explicitly `return`s a primitive vs. an object?

---

## 5.2 ES6 Classes — Syntax Sugar, Not a New Model

**Intuitive version:** `class` syntax (ES6) looks like classical OOP from Java/C++, but under the hood it's **exactly** the constructor-function + prototype pattern from §5.1 — just cleaner syntax. This is one of the single most important facts to internalize about JS: there is no separate "class-based" runtime model bolted on; it's the same prototypal system, dressed up.

```js
class Animal {
  constructor(name) { this.name = name; }
  speak() { return `${this.name} makes a sound.`; }
}
console.log(typeof Animal);                          // "function" — classes ARE functions
console.log(Animal.prototype.speak);                    // the method lives on .prototype, same as before
console.log(new Animal("Rex") instanceof Animal);        // true, same prototype-chain mechanism
```

**Key differences from constructor functions (real, not just cosmetic):**
- Class bodies **always run in strict mode**, even without `"use strict"`.
- Class declarations are **not hoisted** the way function declarations are — they land in the TDZ (Ch.2 §2.4) until evaluated.
- Calling a class **without** `new` throws a `TypeError` (constructor functions silently allow it, causing subtle `this`-is-global bugs).
- Methods defined in a class body are **non-enumerable** by default (matching built-ins' behavior); methods manually attached to `.prototype` the old way are enumerable by default.

```js
class Foo {}
Foo(); // TypeError: Class constructor Foo cannot be invoked without 'new'

function Bar() {}
Bar(); // silently runs, `this` is undefined (strict) or global (sloppy) — a classic old-JS footgun class syntax fixes
```

**Why ES6 classes exist:** not to change the object model (prototypal inheritance stays exactly as-is) but to give developers coming from classical-OOP languages familiar, less error-prone syntax, and to close footguns like calling without `new` — all while remaining 100% backward-compatible with the existing prototype system (Ch.1's constraint again).

**Interview questions:**
- *Beginner:* Are ES6 classes a new inheritance model or sugar over prototypes?
- *Mid:* What happens if you call a class without `new`? Why is this an improvement over constructor functions?
- *Senior:* Why are class body methods non-enumerable by default, and why does that matter for `for...in`/`JSON.stringify`?

---

## 5.3 Inheritance — `extends` and `super`

**Intuitive version:** `extends` links one class's prototype chain to another's, so instances of the subclass inherit methods from the parent. `super` gives you access to the parent's constructor/methods from inside the child.

**Formal version:**
```js
class Animal {
  constructor(name) { this.name = name; }
  speak() { return `${this.name} makes a sound.`; }
}
class Dog extends Animal {
  constructor(name, breed) {
    super(name);        // MUST call before using `this` — calls Animal's constructor
    this.breed = breed;
  }
  speak() {              // overriding — polymorphism (§5.4)
    return `${super.speak()} Specifically, a bark!`; // super.method() calls the parent's version
  }
}
const rex = new Dog("Rex", "Labrador");
rex.speak(); // "Rex makes a sound. Specifically, a bark!"
```

**Under the hood:** `extends` sets `Dog.prototype.__proto__ = Animal.prototype` (linking the chains) AND `Dog.__proto__ = Animal` (so static methods are inherited too). `super(name)` in the constructor calls `Animal.call(this, name)`, essentially.

**Critical rule — `this` is not available before `super()` in a subclass constructor:**
```js
class Dog extends Animal {
  constructor(name) {
    console.log(this); // ReferenceError: must call super() first
    super(name);
  }
}
```
**Why this rule exists:** in a subclass, `this` isn't created by the subclass itself — it's created by the base class's constructor (via `super()`), then handed down. Referencing `this` before that call happens means referencing something that doesn't exist yet.

**Interview questions:**
- *Mid:* What does `super()` do, and why must it be called before using `this` in a derived class?
- *Senior:* Explain, mechanically, what `extends` does to the prototype chain of both the class and its `.prototype`.

---

## 5.4 Polymorphism

**Intuitive version:** Different classes can implement the same method name differently, and calling code doesn't need to know which specific class it's dealing with — it just calls `.speak()` and gets the right behavior.

**Formal version:** achieved via method overriding (subclass redefines a method inherited from its parent) — dynamic dispatch happens naturally through the prototype chain lookup (Ch.4 §4.4): the engine finds the *closest* matching method up the chain at call time.

```js
class Shape { area() { return 0; } }
class Circle extends Shape { constructor(r) { super(); this.r = r; } area() { return Math.PI * this.r ** 2; } }
class Square extends Shape { constructor(s) { super(); this.s = s; } area() { return this.s ** 2; } }

[new Circle(2), new Square(3)].forEach(shape => console.log(shape.area()));
// each calls ITS OWN area() — polymorphic dispatch, no if/else on type needed
```

**Interview questions:**
- *Mid:* What is polymorphism and how does JS achieve it without explicit interfaces?
- *Senior:* Why is polymorphic dispatch generally preferable to a big `if/switch` on a `type` field?

---

## 5.5 Encapsulation & Private Class Fields

**Intuitive version:** Encapsulation hides internal state so outside code can't directly mess with it. JS lacked *true* privacy for object properties for over 20 years (only conventions like `_underscorePrefix`, or closures, simulated it) — until ES2022's real private fields.

**Formal version — the evolution:**
```js
// 1) Convention only (NOT real privacy — still fully accessible)
class Old { constructor() { this._balance = 0; } }

// 2) Closure-based privacy (real, but verbose, pre-class-fields)
function Account() {
  let balance = 0; // truly inaccessible from outside — no reference exists
  return { deposit: amt => balance += amt, getBalance: () => balance };
}

// 3) ES2022 private fields (# syntax) — real privacy, clean syntax
class Account3 {
  #balance = 0; // truly private — inaccessible and even invisible outside the class
  deposit(amt) { this.#balance += amt; }
  getBalance() { return this.#balance; }
}
const acc = new Account3();
acc.#balance; // SyntaxError — not even a runtime error, the parser rejects it outside the class
```

**Why `#` syntax specifically (not just a naming convention) exists:** engines needed a way to guarantee privacy at the **language level**, enforceable even against `Object.keys`, `JSON.stringify`, `Reflect.ownKeys`, and `Proxy` traps — a mere naming convention (`_balance`) can't prevent any of that. `#fields` are not even reachable via bracket notation or reflection; the parser itself rejects access outside the class body.

**Private methods and static private fields also exist:**
```js
class Account4 {
  #balance = 0;
  #validate(amt) { if (amt < 0) throw new Error("negative"); } // private method
  static #instanceCount = 0; // private static field
  deposit(amt) { this.#validate(amt); this.#balance += amt; }
}
```

**Interview questions:**
- *Beginner:* How did developers simulate private properties before ES2022?
- *Mid:* Why is `_balance` (underscore convention) not real encapsulation?
- *Senior:* Why can't `Object.keys`/`JSON.stringify`/`Proxy` see `#private` fields, while they *can* see closure-based "private" state indirectly influence public methods but not directly enumerate it either? Compare the two approaches' guarantees.

---

## 5.6 Static Methods and Properties

**Intuitive version:** Static members belong to the class itself, not to instances — utility/factory functions and shared state that doesn't need "an instance" to make sense.

**Formal version:**
```js
class MathUtils {
  static PI = 3.14159;
  static square(x) { return x * x; }
}
MathUtils.square(4); // 16 — called on the class, not an instance
const m = new MathUtils();
m.square; // undefined — NOT inherited by instances, only by the class/subclasses
```

**Real-world use:** factory methods (`Array.from`, `Object.create`), singleton patterns, utility namespaces, tracking counts across all instances (`static #instanceCount`, incremented in the constructor).

**Interview questions:**
- *Beginner:* Can an instance call a static method directly? (No.)
- *Mid:* Give a real-world use case for a static factory method over a public constructor.

---

## Practical exercises — Chapter 5

1. Implement `new` from scratch (§5.1) without ever writing the `new` keyword, and test it against a real class.
2. **Debugging exercise:** a subclass constructor throws `ReferenceError: must call super constructor before accessing 'this'`. Identify exactly which line is wrong and fix it.
3. **Small challenge:** convert the closure-based `Account` (§5.5, version 2) into an ES2022 `#private`-field class, preserving identical external behavior.
4. **Advanced challenge:** build a small class hierarchy (`Shape` → `Circle`, `Rectangle`, `Triangle`) demonstrating polymorphism, with a static factory `Shape.create(type, ...args)` that returns the correct subclass instance — a common "design a mini system" interview exercise.

---

**Next:** Chapter 6 — Asynchronous JavaScript (event loop, microtasks vs. macrotasks, Promises, async/await, generators). This is the chapter most FAANG interviews spend the most time on — and it all traces back to the single-threaded execution model from Ch.1 §1.7. Say **"next"** when ready.
