# Chapter 2 — Language Fundamentals

> Depends on: Ch.1 §1.4 (creation phase before execution) and §1.7 (execution model).
> Feeds into: Ch.3 (closures rely on scope), Ch.4 (property access relies on coercion of keys), Ch.6 (async code still obeys scope rules).

---

## 2.1 Variables: `var`, `let`, `const`

**Intuitive version:** `var` is the original, loosely-scoped way to declare a variable — it leaks out of blocks and gets "hoisted" as `undefined`. `let`/`const` (ES6) are the fixed version: properly block-scoped, and they don't quietly initialize to `undefined` before you reach them.

**Formal version:** `var` declarations are **function-scoped** (or global-scoped if outside any function). `let`/`const` are **block-scoped** (any `{}` — `if`, `for`, bare blocks). `const` additionally forbids **reassignment** of the binding (not mutation of the value it points to).

**Why `let`/`const` exist:** `var`'s function-scoping caused a specific, extremely common bug class — variables "leaking" out of loops/conditionals and being accidentally shared or overwritten. ES6 fixed this without removing `var` (backward compatibility, per Ch.1).

```js
if (true) {
  var a = 1;
  let b = 2;
}
console.log(a); // 1 — leaked out of the block
console.log(b); // ReferenceError — b never existed outside the block
```

**`const` mutability trap (very common misconception):** `const` freezes the *binding*, not the *value*.

```js
const arr = [1, 2, 3];
arr.push(4);     // fine — mutating the array, not reassigning arr
arr = [5, 6];     // TypeError — reassignment
```

**Classic interview trap — `var` in loops:**

```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// Logs: 3, 3, 3 — one shared `i`, all callbacks see its final value

for (let j = 0; j < 3; j++) {
  setTimeout(() => console.log(j), 0);
}
// Logs: 0, 1, 2 — `let` creates a NEW binding per iteration
```
This single example is one of the most-asked JS interview questions in existence. The "why" (explained fully): `let` in a `for` loop creates a fresh lexical binding for each iteration, so each closure captures its own `j`. `var` has one binding for the entire loop, so all closures share the same final value.

**Best practice:** default to `const`; use `let` only when reassignment is genuinely needed; avoid `var` in new code entirely — but you must still understand it deeply, because you'll read it in legacy code and it's a top interview topic.

**Interview questions:**
- *Beginner:* What's the difference between `let` and `const`?
- *Mid:* Why does `const arr = []; arr.push(1)` not throw?
- *Senior/FAANG:* Explain, mechanism by mechanism, why the `var` vs `let` loop example above produces different output.

---

## 2.2 Scope, the Scope Chain, and Lexical Scoping

**Intuitive version:** Scope is "where can this name be seen from." JavaScript decides scope by *where code is written* (lexical/static scoping), not by *how it's called* (dynamic scoping) — so you can always figure out a variable's scope just by reading the source, without running it.

**Formal version:** Every function/block creates a new **lexical environment**, which holds a reference to its **outer environment**. Looking up a variable walks this chain outward until found (or `ReferenceError` if never found). This chain is fixed at *definition* time, not call time.

```js
const x = "outer";
function readX() { console.log(x); }
function wrapper() {
  const x = "inner";
  readX(); // logs "outer" — NOT "inner"
}
wrapper();
```
`readX` is lexically defined next to the outer `x`, so its scope chain points there — regardless of the fact that it's *called* from inside `wrapper`, where a different `x` exists. This is the proof that JS is lexically, not dynamically, scoped.

**Mental model:** scope chain = a stack of transparent sheets. Each function/block gets a new sheet. Looking up a name means looking at your sheet, then the sheet behind it, then the one behind that, until you hit a name or run out of sheets (global scope). The sheets are stacked based on *where you drew them*, not who's currently looking.

**Global scope pitfall:** in non-strict mode, assigning to an undeclared variable creates an *implicit global*:
```js
function leak() { accidental = "oops"; } // no let/const/var
leak();
console.log(accidental); // "oops" — leaked onto global object
```
`"use strict"` (automatic in ES modules and classes) turns this into a `ReferenceError` instead — a good example of why strict mode exists.

**Interview questions:**
- *Beginner:* What is the scope chain?
- *Mid:* What's the difference between lexical and dynamic scoping? Is JS which one?
- *Senior:* Why does strict mode throw on undeclared assignment, and why wasn't this the default from day one? (Ties back to Ch.1's backward-compatibility constraint.)

---

## 2.3 Hoisting

**Intuitive version:** "Hoisting" is the popular (slightly misleading) name for what Ch.1 §1.4 called the **creation phase**: before any code in a scope runs, the engine scans it and registers all `var`/`function`/`let`/`const` declarations in memory *first*. Nothing physically moves — it just means declarations are "known" before execution reaches them.

**Formal version:** during the creation phase of an execution context, the engine:
1. Creates bindings for every `var` in the scope, initialized to `undefined`.
2. Fully hoists `function` declarations — both name AND body are available immediately.
3. Creates bindings for `let`/`const`, but leaves them **uninitialized** (see TDZ, §2.4) — accessing them before their declaration line throws.

```js
console.log(a);        // undefined (var hoisted, not yet assigned)
console.log(typeof f); // "function" (function declarations fully hoisted)
console.log(b);        // ReferenceError (TDZ)

var a = 1;
function f() {}
let b = 2;
```

**Function expressions are NOT hoisted the same way** — only the `var` name is hoisted (as `undefined`), the function *value* is assigned at the line it appears:
```js
console.log(typeof g); // "undefined" — g is just a hoisted var, unassigned yet
var g = function () {};
```

**Why hoisting exists:** it's a side effect of engines needing to know, ahead of time, how much memory to allocate and what names exist in a scope — necessary for the two-pass parse/execute model from Ch.1. It also lets mutually-recursive function declarations call each other regardless of declaration order.

**Interview questions:**
- *Beginner:* What does "hoisting" mean?
- *Mid:* Why does `console.log(typeof f)` print `"function"` for a function declaration but `"undefined"` for a function expression assigned to a `var`?
- *Senior:* Explain hoisting in terms of execution context creation phases, not as "moving code to the top" (a common oversimplification that breaks down under scrutiny).

---

## 2.4 The Temporal Dead Zone (TDZ)

**Intuitive version:** Between the start of a scope and the line where a `let`/`const` is actually declared, that variable technically "exists" (the engine knows its name) but is radioactive to touch — hence "dead zone."

**Formal version:** `let`/`const`/`class` bindings are hoisted to the top of their block but remain in an **uninitialized** binding state until their declaration statement executes. Any read/write attempt in that window throws a `ReferenceError`.

```js
{
  console.log(x); // ReferenceError: Cannot access 'x' before initialization
  let x = 5;
}
```

**Why TDZ exists:** it's a deliberate strictness improvement over `var`'s silent `undefined` — the committee decided that accessing a variable before its declaration is *almost always a bug*, and TDZ turns it into a loud error instead of a silent wrong value.

**Tricky edge case (common FAANG gotcha):**
```js
let x = 1;
{
  console.log(x); // ReferenceError, NOT 1!
  let x = 2;
}
```
Even though an outer `x` exists, the inner block's own `let x` shadows it for the *entire block*, including before its declaration line — this is TDZ shadowing, and it trips up even experienced engineers.

**Interview questions:**
- *Mid:* What is the Temporal Dead Zone and why does it exist?
- *Senior/FAANG:* Predict the output of the shadowing example above and explain exactly why, referencing hoisting + TDZ together.

---

## 2.5 Data Types: Primitives vs. Objects

**Intuitive version:** JS has two categories of values: **primitives** (simple, immutable, copied by value) and **objects** (complex, mutable, copied/passed by reference).

**Formal version — the 7 primitive types + 1 special:**
`string`, `number`, `bigint`, `boolean`, `undefined`, `symbol`, `null` — plus `object` (which covers objects, arrays, functions, dates, etc. — everything non-primitive).

**Why the split exists:** primitives model simple, comparable-by-value data efficiently (stored directly, often on the stack or inline); objects model complex, shared, mutable structures (stored on the heap, accessed via reference) — see Ch.7 for the full memory model.

```js
let a = { count: 1 };
let b = a;
b.count = 2;
console.log(a.count); // 2 — same object, two references

let x = 5;
let y = x;
y = 10;
console.log(x); // 5 — primitives copy by value
```

**`typeof null === "object"` — the most famous bug in the language:** this is a bug from the original 1995 implementation (values were tagged with a type ID, and the "object" tag happened to be `0`, same as the null pointer representation) that TC39 could never fix without breaking the web (Ch.1's backward-compatibility rule). It is not logical — it's historical.

**`typeof` cheat sheet (memorize for interviews):**
```js
typeof undefined       // "undefined"
typeof null            // "object"  (the famous bug)
typeof 42               // "number"
typeof 42n              // "bigint"
typeof "hi"             // "string"
typeof true             // "boolean"
typeof Symbol()         // "symbol"
typeof {}                // "object"
typeof []                // "object"  (arrays are objects!)
typeof function(){}      // "function" (technically an object, but typeof special-cases it)
```

**Interview questions:**
- *Beginner:* List all primitive types in JS.
- *Mid:* Why is `typeof null === "object"`?
- *Senior:* Why do primitives copy by value and objects by reference — what does this imply for function arguments? (Pass a primitive → function gets a copy; pass an object → function gets a reference to the same object, so mutations are visible outside.)

---

## 2.6 Boxing (Primitives Behaving Like Objects)

**Intuitive version:** If primitives are simple and don't have methods, why does `"hi".toUpperCase()` work? Because the engine temporarily "boxes" the primitive into a wrapper object, lets you call the method, then discards the wrapper.

**Formal version:** `String`, `Number`, and `Boolean` are constructor functions that create wrapper objects. When you access a property on a primitive, the engine performs **auto-boxing**: creates a temporary wrapper object, performs the property lookup on it, then discards it.

```js
const s = "hello";
console.log(s.length);       // 5 — auto-boxed temporarily
console.log(typeof s);        // still "string" — the boxing is transient

const boxed = new String("hello");
console.log(typeof boxed);    // "object" — a REAL wrapper object, not a primitive
console.log(boxed === "hello"); // false — object identity, not value equality
```

**Common mistake:** manually using `new String()`/`new Number()`/`new Boolean()` — this creates a real object, which breaks equality checks and `typeof`, and is essentially always a bug in real code. Best practice: never use these constructors with `new`.

**Interview questions:**
- *Mid:* How does `"abc".length` work if strings are primitives with no properties?
- *Senior:* What's the difference between `"5" == new Number(5)` and `"5" === new Number(5)`? (First is `true` via coercion of the object to a primitive; second is `false` — different types, `object` vs `string`.)

---

## 2.7 Type Coercion

**Intuitive version:** JS will often try to convert values between types automatically rather than throwing an error — this is "coercion," and it's the source of most "JS is weird" memes.

**Formal version:** Coercion happens via internal `ToPrimitive`, `ToString`, `ToNumber`, `ToBoolean` operations, triggered implicitly by operators (`+`, `==`, template literals, `if` conditions) or explicitly (`String(x)`, `Number(x)`, `!!x`).

**The `+` operator's special rule (must-know):** if *either* operand is a string, `+` does string concatenation; otherwise numeric addition.
```js
1 + "1"     // "11" — string concat wins
1 + 1        // 2
1 + true     // 2  — true coerces to 1
1 + null     // 1  — null coerces to 0
1 + undefined // NaN — undefined coerces to NaN
[] + []      // ""  — both arrays -> "" via ToPrimitive
[] + {}      // "[object Object]" — array -> "", object -> "[object Object]"
```

**Why `[] + []` is `""`:** arrays/objects get converted via `ToPrimitive`, which tries `valueOf()` first (returns the object itself for arrays, which is not a primitive, so it's rejected), then falls back to `toString()`. `[].toString()` is `""`, so `"" + "" = ""`.

**`==` vs `===` (the single most-asked JS interview topic):**
- `===` (strict equality): no coercion, types must match.
- `==` (loose equality): coerces operands to a common type first, following the **Abstract Equality Comparison Algorithm**.

```js
0 == false     // true  (false -> 0)
0 == ""        // true  ("" -> 0)
0 == "0"       // true  ("0" -> 0)
false == "0"   // true  (false -> 0, "0" -> 0)
null == undefined // true (special case, ONLY equal to each other with ==)
null === undefined // false
NaN == NaN      // false — NaN is never equal to anything, even itself
```

**Best practice:** always use `===`/`!==` unless you have a specific, documented reason for `==` (the only common legitimate use: `x == null` to check for both `null` and `undefined` in one comparison).

**Interview questions:**
- *Beginner:* What's the difference between `==` and `===`?
- *Mid:* Why is `[] + []` an empty string?
- *Senior/FAANG:* Without running it, determine `[] == ![]`. (Answer: `true`. `![]` is `false` (arrays are truthy, so `!` gives `false`). Then `[] == false` triggers coercion: `[]` → `""` → `0`, and `false` → `0`. `0 == 0` → `true`. This is a classic "explain the weirdness" interview question.)

---

## 2.8 Truthy / Falsy

**Intuitive version:** In boolean contexts (`if`, `&&`, `||`, `!`), every value is treated as either truthy or falsy — and there are only **8 falsy values**, everything else is truthy.

**The complete falsy list (memorize this exactly):**
```
false, 0, -0, 0n, "", null, undefined, NaN
```
Everything else — including `"0"`, `"false"`, `[]`, `{}`, `function(){}` — is **truthy**. This surprises almost everyone the first time: an empty array and empty object are truthy.

```js
if ([]) console.log("truthy!");  // runs — [] is truthy
if ({}) console.log("truthy!");  // runs — {} is truthy
if ("0") console.log("truthy!"); // runs — non-empty string is truthy
```

**Interview questions:**
- *Beginner:* List all 8 falsy values.
- *Mid:* Is `[]` truthy or falsy? Why does this surprise people coming from Python (where `[]` is falsy)?

---

## 2.9 Operators & Expressions vs. Statements

**Intuitive version:** An **expression** produces a value (`2 + 2`, `foo()`, `a ? b : c`). A **statement** performs an action and doesn't itself evaluate to a usable value (`if (...) {}`, `for (...) {}`, `let x = 5;`). This distinction explains why some syntax is legal in one place and not another.

**Why it matters practically:** arrow function implicit-return bodies must be expressions:
```js
const f = () => { return 5; }; // statement body — needs `return`
const g = () => 5;              // expression body — implicit return, no braces
const h = () => { a: 1 };       // NOT an object! {} here is parsed as a block
                                  // statement with a label `a:`, not an object literal
const i = () => ({ a: 1 });     // parens force it to be parsed as an expression
```
That last pair (`h` vs `i`) is a genuinely common real-world bug and a favorite interview trap.

**Key operators worth internalizing precisely:**
- `??` (nullish coalescing, ES2020) — only falls through on `null`/`undefined`, unlike `||` which falls through on *any* falsy value:
  ```js
  const count = 0;
  count || 10   // 10 — WRONG if 0 is a valid value you wanted to keep
  count ?? 10   // 0  — correct, only null/undefined trigger the fallback
  ```
- `?.` (optional chaining, ES2020) — short-circuits to `undefined` instead of throwing on `null`/`undefined` access:
  ```js
  const city = user?.address?.city; // no throw even if user or address is null
  ```

**Interview questions:**
- *Beginner:* What's the difference between a statement and an expression?
- *Mid:* Why does `() => { a: 1 }` not return an object, and how do you fix it?
- *Senior:* Explain a real bug you'd get from using `||` instead of `??` for default values, and why `??` was added instead of just fixing `||`. (Can't fix `||` — backward compatibility, Ch.1 theme again.)

---

## Practical exercises — Chapter 2

1. Predict the output, then explain via scope chain + TDZ:
   ```js
   let x = "outer";
   function test() {
     console.log(x);
     let x = "inner";
   }
   test();
   ```
2. **Debugging exercise:** A teammate's code does `if (user.age || 0)` to provide a default age of 0, but ages of `0` are being silently dropped elsewhere. Explain the coercion bug and rewrite it correctly.
3. **Small challenge:** implement a `looseEquals(a, b)` function that replicates `==` behavior for numbers, strings, booleans, `null`, and `undefined` only (no objects) — to prove you understand the abstract equality algorithm, not just its outputs.
4. **Advanced challenge:** Without a reference, list and explain **five** genuinely surprising `==` coercion results (beyond the ones shown above), and for each, state the *exact* ToPrimitive/ToNumber rule responsible.

---

**Next:** Chapter 3 — Functions (declarations, arrows, `this`, `bind`/`call`/`apply`, closures, HOFs, currying). This is where the scope chain from §2.2 becomes the engine behind one of JS's most powerful features: closures. Say **"next"** when ready.
