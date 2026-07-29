# Chapter 4 — Objects & Prototypes

> Depends on: Ch.2 §2.5 (objects vs primitives), Ch.3 (functions as first-class values, `this`).
> Feeds into: Ch.5 (classes are sugar over exactly this machinery), Ch.10 (Proxy/Reflect deep dive).

---

## 4.1 Object Creation Patterns

**Intuitive version:** There's more than one way to make an object, and each way sets up a different prototype chain (§4.4) — which affects what methods are "inherited" for free.

**Formal version — the main patterns:**
```js
const obj1 = {};                         // object literal — prototype is Object.prototype
const obj2 = new Object();               // equivalent, more verbose
const obj3 = Object.create(protoObj);    // explicit prototype — full control
const obj4 = Object.create(null);        // NO prototype at all — a true "dictionary" object

function Person(name) { this.name = name; } // constructor function pattern (pre-ES6)
const obj5 = new Person("Ana");
```

**Why `Object.create(null)` matters (senior-level detail):** objects created this way have no inherited methods at all — no `toString`, no `hasOwnProperty`. This is the correct choice for objects used purely as hash maps/dictionaries with attacker-controllable keys, to avoid **prototype pollution** attacks (where a key like `"__proto__"` or `"constructor"` could otherwise collide with inherited machinery).

```js
const dict = Object.create(null);
dict["toString"] = "uh oh, but safe here";
console.log(dict.toString); // "uh oh, but safe here" — no inherited method to collide with
```

**Interview questions:**
- *Mid:* What's the difference between `{}` and `Object.create(null)`?
- *Senior:* Why might you use `Object.create(null)` for a cache/dictionary object in production code?

---

## 4.2 Property Descriptors

**Intuitive version:** Every property on an object isn't just a "key: value" — it also has hidden metadata controlling whether it can be changed, deleted, or shown in loops. Object literals set sensible defaults for all of this that you rarely think about — until you need finer control.

**Formal version:** every property has a descriptor with these flags (defaults shown for literal properties):
```js
{
  value: <the value>,
  writable: true,      // can the value be reassigned?
  enumerable: true,     // does it show up in for...in / Object.keys / JSON.stringify?
  configurable: true,   // can the descriptor itself be changed, or the property deleted?
}
```

```js
const obj = {};
Object.defineProperty(obj, "id", {
  value: 42,
  writable: false,     // read-only
  enumerable: false,    // hidden from Object.keys/for-in/JSON.stringify
  configurable: false,   // can't delete or redefine
});

obj.id = 100;             // silently fails in sloppy mode, throws in strict mode
console.log(Object.keys(obj)); // [] — id is hidden
delete obj.id;              // fails
```

**Why this exists:** JS needs a way to build robust, protected APIs (built-in methods like `Array.prototype.push` are non-enumerable, for example — otherwise `for...in` on an array would loop over method names too) without adding a separate "private" keyword to the entire language early on.

**Real-world example:** this is exactly why `for...in` on an array doesn't show `push`, `map`, etc. — they're defined as non-enumerable on `Array.prototype`.

**Interview questions:**
- *Mid:* What are the four attributes in a property descriptor?
- *Senior:* Why doesn't `for...in` iterate over array methods like `.push`?
- *FAANG-style:* Implement a `freezeProperty(obj, key)` helper using `Object.defineProperty` that makes a property read-only but still enumerable.

---

## 4.3 Getters, Setters, and Enumerability

**Intuitive version:** Getters/setters let a property *look* like a plain value from the outside while actually running code (validation, computed values, logging) whenever it's read or written.

**Formal version:**
```js
const person = {
  firstName: "Ada",
  lastName: "Lovelace",
  get fullName() { return `${this.firstName} ${this.lastName}`; },
  set fullName(value) { [this.firstName, this.lastName] = value.split(" "); },
};
person.fullName;              // "Ada Lovelace" — looks like a property, runs code
person.fullName = "Grace Hopper"; // triggers setter, splits into first/last
```

**Why they exist:** to enable encapsulation — validation, derived/computed properties, and side effects — while keeping the *calling* syntax identical to plain property access (no `getFullName()` verbosity needed).

**Common mistake:** infinite recursion by naming the getter/setter the same as the backing field:
```js
const bad = {
  get value() { return this.value; }, // infinite recursion! calls itself
};
```
Fix: use a differently-named backing field (`_value` convention, or a closure/private field).

**Interview questions:**
- *Mid:* What's the practical benefit of a getter over a regular method like `getFullName()`?
- *Senior:* Why does `get value() { return this.value }` cause infinite recursion, and how do you fix it?

---

## 4.4 The Prototype Chain

**Intuitive version:** Every object has a hidden link to another object (its "prototype"). When you access a property that doesn't exist directly on an object, JS automatically looks up that chain of links until it finds it (or reaches the end, `null`). This is how objects "inherit" methods without copying them.

**Formal version:** every object has an internal `[[Prototype]]` slot (exposed via `Object.getPrototypeOf(obj)` or the legacy accessor `obj.__proto__`). Property lookup walks this chain. Functions additionally have a `.prototype` **property** (not the same thing as `[[Prototype]]`!) which becomes the `[[Prototype]]` of objects created via `new SomeFunction()`.

```js
function Animal(name) { this.name = name; }
Animal.prototype.speak = function () { return `${this.name} makes a sound.`; };

const dog = new Animal("Rex");
dog.speak(); // "Rex makes a sound." — not found on dog itself, found via [[Prototype]] chain

console.log(dog.__proto__ === Animal.prototype); // true
console.log(Object.getPrototypeOf(dog) === Animal.prototype); // true (preferred, standard API)
console.log(dog.hasOwnProperty("name"));   // true — own property
console.log(dog.hasOwnProperty("speak"));  // false — inherited via chain
```

**`prototype` vs `__proto__` — the #1 source of confusion (must be crystal clear for interviews):**
- `Function.prototype` — a **property** that exists only on functions; it's the object that will become the `[[Prototype]]` of instances created with `new`.
- `obj.__proto__` — a (legacy, but still widely supported) **accessor** exposing an object's actual `[[Prototype]]` link. Modern code should use `Object.getPrototypeOf`/`Object.setPrototypeOf` instead.

**The chain terminates at `Object.prototype`, whose own `[[Prototype]]` is `null`:**
```
dog → Animal.prototype → Object.prototype → null
```

**Mental model:** think of it like corporate escalation — you (the object) get asked a question (property access). If you don't personally know the answer, you escalate to your manager (`[[Prototype]]`), who escalates further up the chain, until someone knows the answer or you reach the CEO (`Object.prototype`) with nowhere further to escalate (`null`).

**Why prototypal inheritance exists instead of classical (copy-based) inheritance:** it's memory-efficient (one shared method lives once on the prototype, not copied onto every instance) and dynamically flexible (you can change a prototype's method and every existing instance immediately sees the update, since lookup happens live).

```js
Animal.prototype.speak = function () { return `${this.name} says hi!`; };
dog.speak(); // "Rex says hi!" — updated live, dog wasn't recreated
```

**`Object.create` and manual prototypal inheritance (pre-class syntax):**
```js
const animalProto = {
  speak() { return `${this.name} makes a sound.`; },
};
const dog2 = Object.create(animalProto);
dog2.name = "Buddy";
dog2.speak(); // "Buddy makes a sound."
```

**Common mistakes:**
- Confusing `.prototype` (function property) with `.__proto__`/`[[Prototype]]` (any object's actual link).
- Mutating shared prototype state expecting per-instance behavior (e.g., putting an array on `.prototype` — every instance shares the SAME array).
```js
function Bad() {}
Bad.prototype.items = []; // shared across ALL instances!
const b1 = new Bad(), b2 = new Bad();
b1.items.push("x");
console.log(b2.items); // ["x"] — leaked across instances, classic bug
```

**Interview questions:**
- *Beginner:* What is the prototype chain?
- *Mid:* What's the difference between `Function.prototype` and `obj.__proto__`?
- *Senior:* Why is putting a mutable object/array directly on a constructor's `.prototype` almost always a bug?
- *FAANG-style:* Implement `Object.create` from scratch:
  ```js
  function myCreate(proto) {
    function F() {}
    F.prototype = proto;
    return new F();
  }
  ```

---

## 4.5 `Object.assign` and Shallow vs. Deep Copy

**Intuitive version:** `Object.assign(target, ...sources)` copies **own enumerable** properties from source objects into a target — but only one level deep.

```js
const a = { nested: { x: 1 } };
const b = Object.assign({}, a);
b.nested.x = 99;
console.log(a.nested.x); // 99 — shallow copy shares the nested object!
```

**Modern alternative:** the spread operator (`{ ...a }`) does the same shallow-copy job with cleaner syntax; `structuredClone(obj)` (widely available since 2022) does a true deep clone natively, handling circular references — preferred over the old `JSON.parse(JSON.stringify(obj))` deep-clone hack, which silently drops functions, `undefined`, `Symbol`, and breaks on circular refs.

**Interview questions:**
- *Mid:* Why does mutating a nested object after `Object.assign` affect the original?
- *Senior:* What are the problems with using `JSON.parse(JSON.stringify(x))` for deep cloning, and what should you use instead?

---

## 4.6 `Reflect`

**Intuitive version:** `Reflect` is a built-in object bundling the "internal" object operations (get, set, delete, define property, etc.) as clean, consistent functions — essentially a tidy, function-based API for things you could already do with operators/statements.

**Formal version (ES6):** `Reflect` methods mirror the internal MOP (Meta-Object Protocol) operations (`Reflect.get`, `.set`, `.has`, `.deleteProperty`, `.ownKeys`, `.defineProperty`, `.getPrototypeOf`, etc.), each returning consistent, predictable results (e.g., `Reflect.deleteProperty` returns `true`/`false` instead of sometimes throwing).

**Why it exists:** primarily to pair with `Proxy` (§4.7) — every `Proxy` trap has a matching `Reflect` method, so you can easily forward default behavior from inside a custom trap.

```js
Reflect.has(obj, "key");    // like `"key" in obj`, but as a function
Reflect.ownKeys(obj);       // like Object.keys but includes non-enumerable and Symbol keys
```

**Interview questions:**
- *Mid:* What problem does `Reflect` solve that plain operators don't?
- *Senior:* Why is `Reflect` almost always used together with `Proxy`?

---

## 4.7 `Proxy` — Metaprogramming Object Behavior

**Intuitive version:** A `Proxy` wraps an object and lets you intercept fundamental operations on it — reading a property, writing one, checking `in`, deleting — and run custom logic instead of (or in addition to) the default behavior.

**Formal version:** `new Proxy(target, handler)` — `handler` contains **traps** (`get`, `set`, `has`, `deleteProperty`, `apply`, `construct`, etc.), each intercepting the corresponding operation.

```js
const user = { name: "Ana", age: 30 };
const logged = new Proxy(user, {
  get(target, prop, receiver) {
    console.log(`Reading ${String(prop)}`);
    return Reflect.get(target, prop, receiver); // forward default behavior
  },
  set(target, prop, value) {
    if (prop === "age" && typeof value !== "number") {
      throw new TypeError("age must be a number");
    }
    return Reflect.set(target, prop, value);
  },
});
logged.name;        // logs "Reading name", returns "Ana"
logged.age = "old"; // throws TypeError — custom validation
```

**Real-world uses:** Vue 3's reactivity system is built on `Proxy` (replacing Vue 2's `Object.defineProperty`-based approach — a great "why did the framework change its internals" interview topic); API validation layers; auto-vivifying nested objects; negative array indexing polyfills; logging/debugging wrappers.

**Why `Proxy` was needed even though `Object.defineProperty` already existed:** `defineProperty` only intercepts *existing, known* property names one at a time — it can't intercept property **addition** of new, previously-unknown keys, or operations like `delete`/`in`/function calls. `Proxy` intercepts the operation itself, generically, for any key, including ones added later. This is exactly why Vue migrated from `defineProperty` to `Proxy` for its reactivity system — the old approach couldn't detect new properties added after object creation, or array index/length changes, without workarounds.

**Interview questions:**
- *Mid:* What is a `Proxy` trap? Name three.
- *Senior:* Why couldn't `Object.defineProperty` alone implement full reactive tracking of arbitrary property additions? Why does `Proxy` solve this?
- *FAANG-style:* Implement a `Proxy`-based negative-indexing array wrapper: `arr[-1]` returns the last element.
  ```js
  function negativeArray(arr) {
    return new Proxy(arr, {
      get(target, prop) {
        if (typeof prop === "string" && /^-\d+$/.test(prop)) {
          return target[target.length + Number(prop)];
        }
        return Reflect.get(target, prop);
      },
    });
  }
  ```

---

## Practical exercises — Chapter 4

1. Predict and explain: given `Bad.prototype.items = []` from §4.4, why does mutating one instance's `items` affect all others? Fix the constructor to avoid this.
2. **Debugging exercise:** a getter named the same as its backing field causes a stack overflow. Identify the bug and fix it using a private backing field.
3. **Small challenge:** implement `Object.freeze`-like behavior manually using `Object.defineProperty` in a loop over all keys, setting `writable: false, configurable: false`.
4. **Advanced challenge:** build a `Proxy`-based "observable" object that calls a provided `onChange(key, oldValue, newValue)` callback whenever any property changes — the conceptual core of how reactive frameworks like Vue/MobX work.

---

**Next:** Chapter 5 — Classes (constructor functions vs. ES6 classes, inheritance, polymorphism, private fields, statics) — this is the payoff chapter where prototypes stop being "internal machinery" and become the syntax you use daily. Say **"next"** when ready.
