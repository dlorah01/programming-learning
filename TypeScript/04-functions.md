---
layout: default
title: "Chapter 4 — Functions"
---

# Chapter 4 — Functions

## 4.1 Function Types

**Intuition.** A function type describes a *contract*: what goes in, what comes out — nothing about implementation. It's the same structural-typing philosophy from Ch. 3 applied to "callable shapes" instead of "property shapes."

**Technical explanation.** Three equivalent-ish ways to write a function type:
```ts
type Add1 = (a: number, b: number) => number;              // arrow-style type alias
interface Add2 { (a: number, b: number): number; }          // call signature in interface
type Add3 = { (a: number, b: number): number };             // call signature in object type literal
```
All three describe "a callable that takes two numbers, returns a number." A function's actual runtime type is `Function`-shaped (it's an object with a `call`/`apply`/`bind`/`length`/`name`, plus your parameters), but for typing purposes what matters is the **call signature**.

**Internal behavior.** Function types are represented in the checker as object types with a special call-signature member. This is why a function *value* can also have additional properties (`fn.foo = 'bar'`) and be typed to reflect that — a callable object type can combine a call signature with regular property members:
```ts
interface Fetcher { (url: string): Promise<Response>; cache: Map<string, Response>; }
```

**Compiler behavior / inference.** Return types and parameter types are inferred from the function body/expression when not annotated. **Contextual typing** is the key mechanism for callback parameters: when a function literal is passed where a specific function type is expected, its parameters get their types "from context" without you writing them.
```ts
const nums = [1, 2, 3];
nums.map(n => n * 2); // `n` inferred as number — from Array<number>.map's signature, not from `n`'s usage
```

**Real-world use cases.** Typing callback-heavy APIs (event handlers, middleware, functional-composition utilities), typing higher-order functions (functions that return functions).

**Good practices.** Prefer named function type aliases for repeated shapes (`type Middleware = (req: Request, res: Response, next: NextFunction) => void`) instead of inlining the same signature everywhere — improves error messages and refactor safety.
**Bad practices.** Annotating callback parameter types manually when contextual typing already infers them correctly — adds noise and risks drifting out of sync if the outer signature changes.
**Common mistakes.** Forgetting that a *standalone* function type's parameters are contravariant (Ch. 3.6) while method-shorthand call signatures are bivariant — this exact distinction applies to interface call signatures too, not just named methods.
**Edge case.** Function types support **overload signatures** only in the interface/call-signature form, not in the arrow-type-alias form directly (a genuinely non-obvious syntactic limitation — §4.3 shows the actual overload syntax).
**Performance.** Negligible on its own.

### Interview Q&A — §4.1
**Junior:** Q: What is contextual typing? A: When TS infers a function expression's parameter types automatically from the expected type at its usage site (e.g., a callback's expected signature), instead of requiring explicit annotations.
**Mid:** Q: Can a value be both callable and have properties in TS? A: Yes — an object type can declare a call signature alongside regular members, matching real JS where functions are also objects (`fn.someProp = x`).
**Senior:** Q: Why does contextual typing matter so much for developer ergonomics in callback-heavy JS APIs? A: Without it, every array method callback, every event handler, every promise `.then()` would require redundant explicit parameter annotations matching the outer API's signature — contextual typing lets the checker push the expected type "down" into the literal being written, which is what makes idiomatic functional JS patterns (`.map`, `.filter`, `.reduce`) feel type-safe without any extra syntax.

**Predict the inferred type**
```ts
const handlers: Record<string, (x: number) => number> = {
  double: x => x * 2,   // x: number (contextually typed from Record's value type)
};
```

---

## 4.2 Optional & Default Parameters

**Intuition.** Optional parameters (`?`) say "caller may omit this"; default parameters (`= value`) say "if omitted, use this value" — related but distinct: every default parameter is *effectively* optional to call, but not every optional parameter has a default.

**Technical explanation.**
```ts
function greet(name: string, title?: string) { }         // title: string | undefined, optional
function greet2(name: string, title: string = "Mx") { }  // title: string inside the body (default fills gaps)
```

**Internal behavior.** Inside the function body, an optional parameter's type is `T | undefined` (you must narrow before using it as `T`); a default parameter's type inside the body is just `T` — the checker knows the default value guarantees a non-`undefined` value by the time the body runs, even though *callers* may omit the argument (contextually, default params behave like optional params from the caller's side, but like guaranteed values from the callee's side).

**Compiler behavior / inference.** Default parameter types are inferred from the default value if not annotated: `function f(x = 5)` infers `x: number`. Optional parameters **must come after required parameters** (a hard syntactic/structural rule — `function f(a?: string, b: string)` is invalid).

**Real-world use cases.** Config-object parameters with sensible defaults, optional callback arguments (`Array.prototype.map`'s `index`/`array` params).

**Good practices.** Prefer default parameters over `param ?? defaultValue` inside the body when the default is a simple constant — keeps intent visible in the signature itself (better for autocomplete/docs).
**Bad practices.** Mixing many optional/boolean parameters positionally (`function create(name: string, active?: boolean, admin?: boolean, verified?: boolean)`) — becomes unreadable and error-prone at call sites; prefer a single options object.
**Common mistakes.** Treating an optional parameter as always safely `T` without narrowing — a classic `undefined` runtime bug slipping past a hasty read of the signature.
**Edge case.** `exactOptionalPropertyTypes` (Ch. 1) doesn't apply to function *parameters* the same way it applies to object *properties* — optional parameters called with an explicit `undefined` argument are always fine (`f(undefined)` matches `f(x?: T)`), unlike the object-property distinction between "absent" and "present-but-undefined."
**Performance.** N/A.

### Interview Q&A — §4.2
**Junior:** Q: What's the type of an optional parameter `x?: string` inside the function body? A: `string | undefined`.
**Mid:** Q: Why must optional parameters come after required ones? A: Positional calling would otherwise be ambiguous — the compiler (and callers) couldn't tell whether an omitted middle argument means "skip this one" or "shift everything left," so optional/default parameters are restricted to trailing positions (rest parameters are the sole further exception, always last).
**Senior:** Q: Inside a function body, why is a default parameter's type narrower (non-undefined) than an equivalent optional parameter's type, even though both can be omitted by the caller? A: A default parameter's initializer guarantees that by the time the function body executes, the parameter has been assigned either the caller's value or the default — so from the body's perspective, `undefined` is never actually observable, and the checker soundly narrows it; an optional parameter has no such guarantee, so `undefined` remains a real possibility the body must handle.

**Predict/debug**
```ts
function f(a: number, b?: number) { return a + b; } // Error: b is possibly undefined
function g(a: number, b: number = 0) { return a + b; } // OK
```

---

## 4.3 Overloads

**Intuition.** Sometimes one function genuinely behaves differently — different return types, different valid argument combinations — depending on *how* it's called. Overloads let you describe several distinct valid call "shapes" for the same function name, giving each its own precise types, while a single real implementation handles them all at runtime.

**Technical explanation.**
```ts
function parse(input: string): object;
function parse(input: string, reviver: (k: string, v: unknown) => unknown): object;
function parse(input: string, reviver?: (k: string, v: unknown) => unknown): object {
  // implementation — the "implementation signature," not itself part of the public overload set
  return JSON.parse(input, reviver);
}
```
Callers only ever see the overload *signatures* (the ones without a body); the implementation signature must be general enough to satisfy all of them but is invisible to call-site type checking.

**Internal behavior.** When resolving a call, the checker tries each overload signature **top to bottom**, in declaration order, and uses the **first one that matches** — not the "best" or "most specific" match, just the first syntactically compatible one. This is a frequently-tested gotcha: overload order matters, and a too-general overload placed first will "shadow" a more specific one placed after it.

**Compiler behavior / inference.** Overload resolution happens before generic inference in some cases and interacts subtly with it — overloaded generic functions are one of the genuinely hard corners of the type system (library authors, e.g. lodash/RxJS type definitions, lean on this heavily). Return type displayed to the caller is whichever overload matched, not a union of all possible returns.

**Real-world use cases.** `document.createElement(tagName)` (returns a specific `HTMLXElement` subtype per known tag name, falls back to generic `HTMLElement` for unknown strings) is the canonical example — modeled with dozens of overloads in `lib.dom.d.ts`. Also common in query builders, event emitters (`on(event: "error", cb: (err: Error) => void): this; on(event: string, cb: (...args: any[]) => void): this;`).

**Good practices.** Order overloads from **most specific to most general** (mirrors the "first match wins" resolution). Prefer a union-typed single signature over overloads whenever the relationship between input and output can be expressed with conditional types or a simple union — overloads should be a last resort for genuinely irregular APIs, not a default tool.
**Bad practices.** Writing overloads where a single generic signature with conditional return types would be clearer and safer (overloads don't enforce any actual *correlation* between which overload was "meant" and what the implementation does — it's on you to keep them honest).
**Common mistakes.** Putting a general/loose overload before a specific one, silently shadowing the specific one for all calls that also happen to structurally match the general one.
**Edge case.** The implementation signature is **not callable from outside** — even though it's the "real" function, external callers only see the declared overloads; if none of the declared overloads match a call, you get an error even if the implementation signature technically could have handled it.
**Performance.** Overload resolution adds a linear-in-overload-count cost per call site during checking — usually negligible unless a library declares dozens of overloads on a hot generic function.

### Interview Q&A — §4.3
**Junior:** Q: What are function overloads used for? A: Describing multiple valid call signatures (different parameter/return type combinations) for a single function name, so each call site gets precise types based on how it's called.
**Mid:** Q: In what order does TS try overload signatures when resolving a call? A: Top to bottom, in declaration order — it uses the first structurally matching signature, not necessarily the most specific or "best" one.
**Senior:** Q: Why is overload order such a common source of subtle bugs in library type definitions? A: Because resolution is first-match, not best-match; if a more general overload is declared before a more specific one, any call that satisfies both will resolve to the general (often less precise, sometimes outright wrong-return-type) overload, silently — there's no error, just a worse type than intended, which can go unnoticed for a long time.
**FAANG/tricky:** Q: Can external code ever call a function using its implementation signature directly, bypassing the declared overloads? A: No — the implementation signature is only used internally to type-check the function body against all declared overloads; it is invisible to callers. A call must match one of the explicit overload signatures or it's an error, even if the implementation would have handled it fine at runtime.

**Predict/debug — overload ordering bug**
```ts
function process(x: string | number): string;
function process(x: number): number;   // unreachable in practice for number-only calls!
function process(x: any): any { return x; }

const r = process(5); // r: string — NOT number, because the first overload (string|number) matches first
```
→ This is exactly the "general-before-specific" ordering bug; swapping the two overload declarations fixes it.

---

## 4.4 Rest Parameters

**Intuition.** Rest parameters collect "everything else" passed positionally into a single typed array — the typed version of JS's `...args`.

**Technical explanation.**
```ts
function sum(...nums: number[]): number { return nums.reduce((a, b) => a + b, 0); }
function log(message: string, ...args: unknown[]): void { }
```
Rest parameters must be **last**, and their type is always an array (or tuple, for variadic-tuple precision — Ch. 6) type.

**Internal behavior.** At the call site, the checker packs any trailing arguments beyond the fixed/named parameters into the declared array/tuple type and checks each against the element type. Combined with generics and variadic tuple types (Ch. 6), rest parameters are how TS types genuinely variadic APIs (`Function.prototype.bind`, `Promise.all`, Redux's `combineReducers`) with per-argument precision instead of collapsing everything to `any[]`.

**Compiler behavior / inference.** `function f(...args: number[])` — calling `f(1,2,3)` infers each argument checked against `number`. With a labeled tuple rest type (`function f(...args: [string, number])`), you get exact positional typing instead of a homogeneous array — a fixed-arity function expressed via rest syntax.

**Real-world use cases.** Variadic math/utility functions (`Math.max`-like helpers), logging functions, `bind`/`call`/`apply`-style APIs, event emitter `emit(event, ...args)`.

**Good practices.** Use a rest parameter's array type as narrow as truthfully possible (`number[]`, not `any[]`); use variadic tuples (Ch. 6) when different positions truly need different types.
**Bad practices.** Typing a rest parameter `...args: any[]` out of laziness when the actual argument shapes are well-known and finite.
**Common mistakes.** Forgetting only one rest parameter is allowed, and it must be the last parameter — can't have named parameters after it.
**Edge case.** Spreading a `readonly` tuple into a rest-parameter call requires the tuple's readonly-ness to be compatible; a `readonly [string, number]` array can generally be spread into `f(a: string, b: number)` fine, but spreading into a *mutable*-typed rest parameter (`...args: [string, number]`, mutable) can sometimes trip variance-related errors depending on TS version — worth testing explicitly in real code rather than assuming.
**Performance.** N/A meaningfully.

### Interview Q&A — §4.4
**Junior:** Q: What type does `...args: number[]` give `args` inside the function? A: `number[]` — a real array containing all extra positional arguments.
**Mid:** Q: Can you have a parameter after a rest parameter? A: No — a rest parameter must be the last parameter in the list.
**Senior:** Q: How do rest parameters combine with variadic tuple types to type something like `bind`? A: Instead of a plain `T[]` rest type, you use a generic tuple type parameter (`...args: A extends unknown[]`) that captures the *exact* sequence of argument types positionally, letting TS preserve individual types per position through the rest parameter rather than collapsing them into one unioned array type — this is what lets `bind`'s type definition correctly compute "the original parameters minus the ones already bound" (fully explored in Ch. 6.6).

---

## 4.5 `this` Typing

**Intuition.** In JS, `this` is dynamically determined by *how* a function is called, not where it's defined — a notorious footgun. TS lets you pin down what `this` is *supposed* to be for a given function, and will error if you call it in a way that would violate that.

**Technical explanation.**
```ts
function reportSize(this: { width: number; height: number }) {
  console.log(this.width * this.height);
}
const shape = { width: 2, height: 3, reportSize };
shape.reportSize();     // OK — `this` matches
reportSize();            // Error — `this` is undefined/global, not the required shape
const detached = shape.reportSize;
detached();               // Error — calling without the right receiver
```
The `this` parameter is a **fake parameter** — it's declared first in the parameter list purely for typing purposes and is completely erased at emit; it doesn't count toward the function's real arity/`length`.

**Internal behavior.** The checker treats a `this` parameter specially during call-site checking: for a plain function call `f()`, `this` is checked against the global/undefined context; for a method call `obj.f()`, `this` is checked against `typeof obj`. Arrow functions **do not have their own `this`** at all (lexical `this`, inherited from the enclosing scope) — so arrow functions cannot declare a `this` parameter; TS enforces this as a real error if you try.

**Compiler behavior / inference.** `noImplicitThis` (part of `strict`) requires `this` to have a resolvable, non-implicit-`any` type in any function that uses it without an explicit `this` parameter or without being an obvious method of a typed object.

**Real-world use cases.** Typing plugin/mixin APIs where a function is meant to be attached to and called as a method of various host objects (jQuery-plugin-style patterns, class mixins, Vue 2's `this`-based component options API — a textbook real-world use of explicit `this` typing).
Polymorphic `this` return types (`ThisType<T>` and the `this` return pattern) enable fluent/chainable builder APIs:
```ts
class Builder {
  private parts: string[] = [];
  add(part: string): this { this.parts.push(part); return this; }
}
class SpecialBuilder extends Builder {
  addSpecial(): this { this.parts_hack(); return this; }
}
new SpecialBuilder().add("a").addSpecial(); // `this` return type keeps subclass chaining safe
```
Using `this` as a return type (rather than the literal class name) is what makes chainable methods work correctly through subclassing — an important, frequently-missed senior-level detail.

**Good practices.** Use `this: T` parameters for standalone functions meant to be used as methods; use `this` as a return type annotation for fluent builder APIs on classes designed to be subclassed.
**Bad practices.** Relying on implicit `this` inference in a standalone function and being surprised when it's `undefined` at runtime (strict mode) — enable `noImplicitThis` and let the compiler force you to be explicit.
**Common mistakes.** Detaching a method from its object (`const fn = obj.method; fn();`) and losing the correct `this` binding — a runtime JS gotcha that a `this`-typed function signature will actually catch at compile time, which is a genuinely valuable, underused pattern.
**Edge case.** Arrow function class fields (`field = () => { ... }`) permanently bind `this` to the instance at construction time (via closure, not prototype method dispatch) — this sidesteps the whole `this`-detachment problem entirely, at the cost of one function instance per object instance instead of a shared prototype method (a real memory/perf tradeoff for very large numbers of instances).
**Performance.** Negligible for typing itself; the arrow-field-vs-prototype-method choice above is the actual runtime perf-relevant decision.

### Interview Q&A — §4.5
**Junior:** Q: What does declaring `this: SomeType` as a function's first parameter do? A: Tells the compiler what `this` must be for calls to this function to type-check; it's erased at runtime and doesn't count as a real argument.
**Mid:** Q: Why can't arrow functions declare a `this` parameter? A: Arrow functions don't have their own `this` binding — they lexically inherit `this` from their enclosing scope, so there's nothing for a `this` parameter to override; TS flags this as an error if attempted.
**Senior:** Q: Why should chainable builder methods return `this` rather than the builder's own class name as the return type? A: Returning the literal class type breaks correctly for subclasses — a subclass method chained after a base class method would get back the base class's type, losing access to the subclass's own methods in the chain. Returning the special `this` type makes the return type polymorphic — it resolves to whatever the actual calling subclass is, preserving full chainability through inheritance.
**FAANG/tricky:** Q: Given `const fn = obj.method; fn();` produces a runtime bug in plain JS due to lost `this` binding, how (if at all) can TS catch this at compile time? A: Only if `method` was declared with an explicit `this: T` parameter type — then calling the detached `fn()` without an appropriate receiver fails the `this`-compatibility check at the call site, since a bare call has an incompatible (undefined/global) `this` context. Without an explicit `this` parameter, TS has no information to catch the detachment, and the bug only surfaces at runtime — a strong argument for annotating `this` on any function designed to be used as a method.

**Predict/debug**
```ts
class Counter {
  count = 0;
  increment = () => { this.count++; };      // arrow field: `this` always bound correctly
  incrementBad() { this.count++; }          // prototype method: `this` depends on call site
}
const c = new Counter();
const inc1 = c.increment;
const inc2 = c.incrementBad;
inc1(); // works — this.count becomes 1, arrow field closed over instance `this`
inc2(); // runtime error/undefined behavior — `this` is not `c` anymore when detached
```

---

## Practical Exercises — Chapter 4

1. **Overload design:** Write overloads for a `createElement(tag)` mini-function that returns a more specific type for `"input"`/`"button"`/`"div"` and a generic fallback type for any other string — then deliberately reorder them to demonstrate the shadowing bug from §4.3, and fix it.
2. **Options-object refactor:** Take a function with 4 positional optional/boolean parameters and refactor it to a single typed options-object parameter with defaults, explaining the readability/safety tradeoffs at each call site.
3. **`this`-safety exercise:** Write a plugin-style function meant to be attached to arbitrary objects as a method, give it a proper `this` parameter type, and show both a valid and an invalid call that TS correctly accepts/rejects.
4. **Variadic rest function:** Type a `pipe(...fns)` function (composes functions left-to-right) as precisely as you can with plain generics/rest parameters, and note where you'd need Ch. 6's variadic tuples for full per-position accuracy.

---

**Next:** Chapter 5 — Objects: interfaces vs type aliases, declaration merging, extension, unions & intersections. Say **"next"** to continue.
