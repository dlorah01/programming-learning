---
layout: default
title: "Chapter 6 — Generics"
---

# Chapter 6 — Generics

## 6.1 The Core Idea: Type Parameters

**Intuition.** A generic is a type-level *variable* — a placeholder that gets filled in with a real type at each use site, letting you write one definition that works precisely for many different types without collapsing to `any` and losing all safety. Think of a generic function as a factory that stamps out a specialized version of itself per call.

**Technical explanation.**
```ts
function identity<T>(x: T): T { return x; }
identity(5);        // T = number, returns number
identity("hi");      // T = string, returns string
```
`T` is a **type parameter** — it's not a specific type, it's a name bound to whatever type is supplied (explicitly or inferred) at each call. Compare to `any`: `function identity(x: any): any` loses the relationship between input and output entirely — callers get `any` back regardless of what they passed in. Generics *preserve* that relationship.

**Internal behavior.** The checker treats a type parameter as an opaque, unknown type *within* the generic's own definition (you can't assume `T` has any particular member unless constrained, §6.4) but performs **type parameter inference** at each call site — essentially solving "what type, substituted for `T`, makes this call's arguments compatible with the declared parameter types" — then substitutes that solved type everywhere `T` appears in the return type and rest of the signature.

**Compiler behavior / inference.** Generic inference typically works **argument → parameter type**, matching structurally to solve for `T`. When multiple arguments constrain `T` differently (e.g., `function pair<T>(a: T, b: T)`), TS computes the **best common supertype** (often producing a union if arguments are unrelated) unless a constraint narrows the possibilities.
```ts
function pair<T>(a: T, b: T): [T, T] { return [a, b]; }
pair(1, "a"); // T inferred as string | number — best common type across both args
```

**Real-world use cases.** Generic containers (`Array<T>`, `Promise<T>`, `Map<K,V>`), generic utility functions (`identity`, `first`, `groupBy`), generic API client wrappers (`fetchJson<T>(url): Promise<T>`), library-grade reusable components.

**Good practices.** Only introduce a type parameter when it's actually *used* more than once in the signature (linking input and output, or two inputs together) — a type parameter used exactly once provides no more safety than the concrete type it's standing in for and is usually a smell (covered as a common mistake below).
**Bad practices.** "Generic-washing" — sprinkling `<T>` on functions where `T` never constrains a relationship, purely to look more sophisticated.
**Common mistakes.** Writing `function wrap<T>(x: T): T[] { return [x]; }` when `x: unknown` combined with `unknown[]` would be equally safe if the relationship weren't actually needed — the tell for "you need a generic" is: *does a type appearing at one position need to be the same type as a type appearing at another position (or the return)?* If yes, generic. If no, you probably want a plain (possibly `unknown`) type.
**Edge case.** A type parameter that's used only once in a **return type**, with no relationship to any input (`function make<T>(): T[]`), is legitimate and common (the type is supplied explicitly by the caller, `make<User>()`, since there's nothing to infer it from) — the "used more than once" heuristic is about avoiding *pointless* parameters, not a hard rule.
**Performance.** Generic inference itself is fast per call; the cost grows with how *deeply* the type parameter is used in complex conditional/mapped types elsewhere in the signature (Ch. 8) — plain generic functions like `identity` are essentially free.

### Interview Q&A — §6.1
**Junior:** Q: What problem do generics solve that `any` doesn't? A: Generics preserve the relationship between input and output types (e.g., "the function returns exactly what type was passed in"), while `any` discards all type information, losing that safety.
**Mid:** Q: In `function pair<T>(a: T, b: T)`, what type does TS infer if you call `pair(1, "a")`? A: `T` is inferred as `string | number` — the best common type covering both arguments, since there's no shared narrower type between an unrelated `number` and `string` without a constraint.
**Senior:** Q: What's the practical test for "does this function actually need a generic, or should it just use `unknown`"? A: Ask whether a type appearing in one position (parameter or return) needs to be *provably the same* as a type in another position. If the function's safety guarantee depends on that identity being preserved across positions, it needs a real type parameter; if each position's type is independently irrelevant to the others, a generic adds ceremony without adding safety over `unknown`.

**Predict the inferred type**
```ts
function first<T>(arr: T[]): T | undefined { return arr[0]; }
const x = first([1, 2, 3]);      // x: number | undefined
const y = first(["a", "b"]);     // y: string | undefined
const z = first([]);             // z: unknown ... actually: T inferred as `unknown` here since nothing constrains it
```

---

## 6.2 Generic Interfaces & Type Aliases

**Intuition.** Just as a function can be parameterized over a type, so can a *shape* — "a box that holds a T," "a result that's either a T on success or an error," without committing to what T is until it's used.

**Technical explanation.**
```ts
interface Box<T> { value: T; }
type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E };

const b: Box<number> = { value: 5 };
const r: Result<User> = { ok: true, value: someUser }; // E defaults to Error
```

**Internal behavior.** A generic interface/type alias is really a **type-level function** from type arguments to a concrete type — internally the checker keeps the generic definition abstract and performs substitution (instantiation) at each usage with concrete type arguments, exactly like generic function calls, just applied to type positions instead of value positions.

**Compiler behavior / inference.** Unlike generic functions, generic interfaces/type aliases usually can't infer their type arguments from "usage" the same automatic way — you generally supply them explicitly (`Box<number>`) or they're inferred from a *surrounding* context (e.g., assigning an object literal to a variable annotated `Box<number>`).

**Real-world use cases.** `Result<T, E>`/`Either<L, R>`-style error-handling types (a very common senior-level pattern replacing exceptions for expected failure cases), generic API response wrappers (`ApiResponse<T>`), generic repository/store interfaces (`Repository<T>`).

**Good practices.** Use a `Result<T, E>` pattern for *expected*, recoverable failures (validation errors, not-found) instead of throwing — makes failure handling a compile-time-enforced part of the type, not an implicit possibility hidden in a `try/catch` a caller might forget.
**Bad practices.** Overusing single-type-parameter generic wrappers (`Wrapper<T> = { value: T }`) that add no behavior over the bare `T` — pure ceremony if there's no actual added semantic (error handling, laziness, etc.).
**Common mistakes.** Forgetting default type parameters (`E = Error`) need to come after any non-defaulted parameters, mirroring optional-parameter ordering rules from Ch. 4.
**Edge case.** Recursive generic type aliases (a type alias referencing itself with different type arguments) are legal and are the backbone of tree-shaped/JSON-shaped generic types (Ch. 8.5 covers recursive types fully) — e.g. `type Nested<T> = T | Nested<T>[]`.
**Performance.** Deeply nested generic interface instantiations (generics-of-generics-of-generics) are a real, non-trivial compile-time cost — each layer of substitution the checker performs adds work, and this compounds with conditional/mapped types (Ch. 8).

### Interview Q&A — §6.2
**Junior:** Q: What does `interface Box<T> { value: T }` let you express that a non-generic `interface Box { value: unknown }` doesn't? A: That the *specific* type of `value` is known and preserved per usage (`Box<number>`, `Box<string>`, etc.), rather than being erased to `unknown` for every consumer.
**Mid:** Q: Why is `Result<T, E = Error>` often preferred over throwing exceptions for expected failure cases? A: It makes the possibility of failure an explicit, type-checked part of the function's contract — callers are forced (via narrowing on `ok`) to handle both the success and failure branches, whereas a thrown exception's type is invisible to the type system and easy to forget to catch.
**Senior:** Q: Why can't generic interfaces infer their type arguments from a call the same automatic way generic functions can? A: There's no "call" with argument expressions to structurally match against — an interface is instantiated either explicitly (`Box<number>`) or via contextual typing from an assignment/parameter position expecting that specific generic instantiation; without such context, the checker has nothing analogous to function-call argument matching to solve `T` from.

---

## 6.3 Generic Classes

**Intuition.** Same idea as generic interfaces, but for classes — the class's methods and properties can all reference a shared type parameter fixed once per instance (`new Stack<number>()` vs `new Stack<string>()` are different specialized "flavors" of the same class blueprint).

**Technical explanation.**
```ts
class Stack<T> {
  private items: T[] = [];
  push(item: T): void { this.items.push(item); }
  pop(): T | undefined { return this.items.pop(); }
}
const numStack = new Stack<number>();
const inferredStack = new Stack<string>(); // or often inferred from first push/constructor arg
```

**Internal behavior.** Each class's own type parameter is scoped to the whole class body — instance methods, properties, and (with care) even static members can reference it, though **static members cannot reference the class's own instance type parameter** (a real, frequently-tested rule: statics belong to the class itself, not to any particular instantiation, so they have no `T` to refer to — a separate, independent generic would be needed on a static method itself if genuinely required).

**Compiler behavior / inference.** Constructor argument types can drive inference of the class's type parameter (`class Box<T> { constructor(public value: T) {} }; new Box(5)` infers `Box<number>` without explicit `<number>`), same mechanism as generic function inference applied to the constructor signature.

**Real-world use cases.** Generic data structures (Stack, Queue, LinkedList, Tree), generic repository/service base classes, generic event emitters (`EventEmitter<EventMap>`), generic state containers.

**Good practices.** Prefer inferring the type parameter from constructor arguments when natural, rather than forcing every caller to write it explicitly — reduces boilerplate while keeping full safety.
**Bad practices.** Adding a class type parameter "just in case" future flexibility is needed, when the class is only ever instantiated with one concrete type in practice — unnecessary abstraction.
**Common mistakes.** Trying to reference the class's instance type parameter `T` inside a `static` method — a compile error, since statics exist independent of any specific instantiation.
**Edge case.** A generic class can itself implement a generic interface, propagating the same type parameter through (`class ArrayStack<T> implements Stack<T> { ... }`) — a common pattern for defining multiple concrete implementations of one generic contract.
**Performance.** No special cost beyond ordinary generic instantiation, though very large generic class hierarchies (generic subclasses of generic base classes) accumulate the same substitution costs as deeply nested generic interfaces.

### Interview Q&A — §6.3
**Junior:** Q: Can a `static` method access the class's own generic type parameter `T`? A: No — static members exist independently of any particular instantiation of the class, so there's no concrete `T` for them to refer to.
**Mid:** Q: How does `new Box(5)` infer `Box<number>` without writing `new Box<number>(5)` explicitly? A: TS applies the same generic-argument-inference mechanism used for function calls to the constructor's parameter types — matching the passed argument `5` against the constructor parameter typed `T` and solving `T = number`.
**Senior:** Q: When would you deliberately give a `static` method its own, independent generic type parameter on a generic class? A: When a static (factory-like) method needs its own type-safety relationship unrelated to the class's own instance type parameter — e.g., a static `Box.fromArray<U>(arr: U[]): Box<U>[]` — the static method declares and infers its own `U`, entirely separate from and unrelated to the class's instance-level `T`.

---

## 6.4 Generic Constraints (`extends`)

**Intuition.** A bare `T` can be *anything*, so the compiler won't let you assume it has any particular member. A constraint (`T extends SomeShape`) narrows the universe of acceptable types down to "anything with at least this shape," letting you safely use those members inside the generic code.

**Technical explanation.**
```ts
function getLength<T extends { length: number }>(x: T): number { return x.length; }
getLength("hello");       // OK — string has .length
getLength([1,2,3]);       // OK — array has .length
getLength(5);              // Error — number has no .length
```
Note: `extends` here is reused terminology from Ch. 3/5 but means "is assignable to" in the constraint sense, not class inheritance — a source of early confusion worth naming explicitly.

**Internal behavior.** Within the generic function/class body, the checker treats `T` as having *exactly* the constraint's shape (no more, no less) for the purposes of member access — even though the *actual* type substituted at a call site might have more members, the body can't assume that, only what the constraint guarantees. This is why you sometimes need `keyof T` (Ch. 7) rather than assuming specific property names beyond the constraint.

**Compiler behavior / inference.** Constraints also narrow what TS infers `T` *as* at call sites in some scenarios (multiple constrained arguments can pull inference toward the constraint's shape rather than a wildly wide union) and are checked at the call boundary — passing an argument that doesn't satisfy the constraint is a compile error before you even reach the function body's logic.

**Real-world use cases.** `keyof`-based property accessors (`function get<T, K extends keyof T>(obj: T, key: K): T[K]`, the canonical generic-constraint interview example, revisited fully in Ch. 7), constraining generic repository types to have an `id` field, constraining event types to have a `type` discriminant.

**Good practices.** Constrain as tightly as the function actually needs — a constraint is a *promise* about what the body can rely on, so keep it minimal (don't over-constrain and accidentally exclude valid callers) but sufficient (don't under-constrain and be forced into unsafe casts inside the body).
**Bad practices.** Constraining to a huge, specific interface when only one or two members are actually used — unnecessarily restricts what callers can pass; prefer a minimal structural constraint (`{ id: string }`) over a large concrete type when only `id` matters.
**Common mistakes.** Forgetting that a constraint doesn't change what the *caller* sees as the return type — `getLength`'s return type is still `number` regardless of whether you passed a `string` or `array`; constraints affect what's usable *inside* the function, not automatically what flows *out* (unless you separately reference `T` in the return type too).
**Edge case.** You can constrain one type parameter using another (`function pluck<T, K extends keyof T>(obj: T, key: K): T[K]`) — a two-type-parameter pattern that's extremely common in utility-typed codebases and directly sets up Ch. 7's indexed access types.
**Performance.** Constraint checking adds a bounded, per-call-site cost (checking the argument against the constraint) — negligible in isolation, but constraint checks compound with the rest of the generic inference/substitution machinery in deeply generic code.

### Interview Q&A — §6.4
**Junior:** Q: What does `T extends { length: number }` mean in a generic function signature? A: It restricts `T` to only types that have (at least) a `length: number` member, letting the function body safely access `.length` on any value of type `T`.
**Mid:** Q: Does a generic constraint affect what the function's *return type* is inferred as? A: Not automatically — a constraint only governs what's safe to use *inside* the function body; if the return type isn't separately expressed in terms of `T` (or a related type parameter), the constraint alone doesn't propagate anything extra to the caller.
**Senior:** Q: Explain the `pluck<T, K extends keyof T>(obj: T, key: K): T[K]` pattern — why two type parameters instead of one? A: `T` captures the exact shape of the object being passed, and `K extends keyof T` constrains the second parameter to only valid keys *of that specific `T`* — critically, `keyof T` is evaluated per call site once `T` is known, so this correctly rejects `pluck(obj, "nonExistentKey")` for the actual passed-in object type, not against some generic fixed key set; `T[K]` (indexed access, Ch. 7) then gives the precise return type for that specific key.

**Predict/debug**
```ts
function merge<T extends object, U extends object>(a: T, b: U): T & U {
  return { ...a, ...b };
}
const result = merge({ name: "Alice" }, { age: 30 });
// result: { name: string } & { age: number } — effectively { name: string; age: number }
```

---

## 6.5 Default Generic Parameters

**Intuition.** Just like default function parameters, a default type parameter says "if the caller doesn't specify this type argument, use this one" — reduces boilerplate for the common case while still allowing full override.

**Technical explanation.**
```ts
interface ApiResponse<T = unknown> { data: T; status: number; }
type EventHandler<E = Event> = (event: E) => void;

const r: ApiResponse = { data: "anything", status: 200 }; // T defaults to unknown
const r2: ApiResponse<User> = { data: someUser, status: 200 }; // T explicitly overridden
```

**Internal behavior.** Default type parameters are substituted exactly like default function parameter values — if a type argument isn't supplied (and can't be inferred from context), the default expression is used in its place; defaults can themselves reference earlier type parameters in the same declaration (`type Pair<T, U = T> = [T, U]`), a genuinely useful, sometimes-overlooked capability.

**Compiler behavior / inference.** Type parameter defaults, like optional/default function parameters, must be declared **after** any non-defaulted type parameters in the parameter list — same ordering rule as Ch. 4.2, applied at the type level.

**Real-world use cases.** Library APIs with a sensible common-case default but full flexibility for advanced users (`useState<T = undefined>()`-style patterns, generic HTTP client response wrappers defaulting to `unknown` payload types).

**Good practices.** Default to the *safest* reasonable type (`unknown` rather than `any`) when a generic parameter is genuinely optional — preserves safety for callers who don't specify it.
**Bad practices.** Defaulting to `any` for convenience — silently reintroduces all of `any`'s unsoundness for anyone who forgets to specify the type argument.
**Common mistakes.** Assuming a default type parameter is *inferred* from usage the way a default function parameter's value is computed at call time — it isn't; if TS's inference from arguments/context produces a real result, that wins; the default is only used when inference produces nothing usable and no explicit argument was given.
**Edge case.** A default parameter referencing a previous type parameter (`Pair<T, U = T>`) lets you express "same type unless overridden" ergonomically — genuinely useful for APIs like `Record<K, V = K>`-style utilities (not real built-ins, but a common custom-utility pattern).
**Performance.** Negligible — defaults are resolved once per instantiation, same cost class as explicit type arguments.

### Interview Q&A — §6.5
**Junior:** Q: What does `interface Box<T = string>` mean? A: If a caller writes `Box` without a type argument, `T` defaults to `string`; callers can still override it explicitly (`Box<number>`).
**Mid:** Q: Must default type parameters come before or after non-defaulted ones? A: After — same ordering constraint as default/optional function parameters; you can't have a non-defaulted type parameter following a defaulted one.
**Senior:** Q: What's a legitimate use for a default type parameter that references an earlier type parameter in the same declaration? A: Expressing "this second type defaults to being the same as the first, unless the caller wants something different" — e.g. `type Pair<T, U = T> = [T, U]` lets `Pair<number>` mean `[number, number]` while still permitting `Pair<number, string>` for a genuinely mixed pair, minimizing boilerplate for the common (same-type) case.

---

## 6.6 Variadic Generics (Variadic Tuple Types)

**Intuition.** Sometimes a function's argument list isn't fixed-arity *or* uniformly-typed — it's an arbitrary-length sequence where **each position matters individually** (like `bind`, `concat`, `pipe`). Variadic tuple types let a generic type parameter capture "a whole tuple of unknown length and per-position types," and manipulate it (prepend, append, split) as a unit.

**Technical explanation.**
```ts
type Head<T extends unknown[]> = T extends [infer H, ...unknown[]] ? H : never;
type Tail<T extends unknown[]> = T extends [unknown, ...infer R] ? R : never;

function prepend<T extends unknown[], V>(arr: [...T], value: V): [V, ...T] {
  return [value, ...arr];
}
const result = prepend([1, 2, 3] as const, "x"); // ["x", 1, 2, 3] — precisely typed per position
```
The `...T` spread inside a tuple type position (both as a type parameter constraint and in the return type) is what makes this "variadic" — `T` itself stands for an entire tuple, not a single type.

**Internal behavior.** The checker treats a rest-position type parameter (`T extends unknown[]`, used with `...T` in a tuple type) specially: it can infer `T` as a **whole tuple type** (preserving each position's specific type) from a spread argument, rather than collapsing everything into one unioned array element type the way a plain `T[]` rest parameter would (Ch. 4.4). This inference behavior (`infer` combined with rest-tuple patterns, fully explored in Ch. 8's conditional types) is genuinely one of the more advanced corners of the type system.

**Compiler behavior / inference.** This is exactly how modern `lib.es5.d.ts`/`lib.esnext.d.ts` type `Function.prototype.bind`, `Array.prototype.concat`, and `Promise.all` — each is typed with variadic tuple generics to preserve full per-argument precision instead of degrading to `any[]`.

**Real-world use cases.** Typing `pipe`/`compose` function-composition utilities precisely, typing `bind`-like partial application helpers, typing SQL-query-builder or route-builder APIs where argument count and per-position type both matter, typing tuple manipulation utilities (`Head`, `Tail`, `Last`, `Concat`) used as building blocks for more advanced generic library code.

**Good practices.** Reach for variadic tuples only when per-position precision is genuinely valuable to callers (bind/compose-style APIs); for simple "list of same-typed things," a plain `T[]` rest parameter remains simpler and sufficient (Ch. 4.4) — don't reach for variadic tuple complexity by default.
**Bad practices.** Over-engineering a simple homogeneous rest parameter into an unnecessary variadic tuple type, adding complexity with no real per-position safety benefit.
**Common mistakes.** Forgetting the `[...T]` spread syntax in a parameter position vs. just `T` — dropping the spread changes the meaning from "spread this tuple's elements as individual parameters" to "one single parameter whose type happens to be a tuple," a subtle, easy-to-get-wrong distinction.
**Edge case.** Variadic tuples combine with `infer` (Ch. 8) to let you write generic "peel off the first/last element" utilities (`Head<T>`, `Tail<T>` above) — these are the actual building blocks used inside many advanced type-level libraries (e.g., type-safe route parameter extraction, type-safe curry implementations).
**Performance.** Deep recursive variadic-tuple manipulation (recursively peeling elements one at a time for long tuples) can hit real compile-time and even TS's recursion-depth limits for sufficiently long tuples — a genuine, documented scaling limit worth knowing about rather than assuming infinite headroom.

### Interview Q&A — §6.6
**Junior:** Q: What's the basic difference between `...args: T[]` and a variadic tuple type parameter `T extends unknown[]` used as `...args: T`? A: `...args: T[]` collapses all rest arguments into one array with a single (possibly unioned) element type; a variadic tuple parameter preserves each argument's individual, per-position type as a tuple.
**Mid:** Q: Why can't a plain `T[]` rest parameter type `Function.prototype.bind` precisely? A: `bind` needs to know exactly which specific parameter types are being pre-filled and which remain, position by position, to compute the correctly-typed remaining function — collapsing all arguments into one unioned array type would lose exactly the positional precision needed to compute that "remaining parameters" type.
**Senior:** Q: What's a real compile-time cost or limitation of variadic-tuple-heavy generic code? A: Recursive peeling of long tuples (as in `Head`/`Tail`-based utilities applied element-by-element) scales with tuple length and can hit noticeable compile-time slowdowns or TS's built-in recursion-depth guard ("Type instantiation is excessively deep") for sufficiently long argument lists — a real, citable scaling limit of type-level programming in TS (elaborated further in Ch. 8.5 on recursive types).

**Predict/debug**
```ts
function pipe<T, U, V>(f: (x: T) => U, g: (x: U) => V): (x: T) => V {
  return x => g(f(x));
}
// This 2-function version is straightforward generics.
// A fully variadic `pipe(...fns)` supporting arbitrary length with full
// per-step type-safety requires recursive variadic tuple typing — try
// sketching the type signature before checking Ch. 8/9 for the full solution.
```

---

## Practical Exercises — Chapter 6

1. **Generic vs `unknown` audit:** Take 3 functions from a real project using `<T>` and, for each, determine whether the type parameter is actually load-bearing (appears meaningfully more than once) or could be simplified to `unknown`/a concrete type.
2. **Build a `Result<T, E>`:** Implement a small `Result<T, E = Error>` type with `map`, `flatMap`/`andThen`, and `unwrapOr` helper functions, fully generic and type-safe, no `any`.
3. **Constrain and pluck:** Implement `pluck<T, K extends keyof T>(obj: T, key: K): T[K]` from scratch and write 3 call sites: one valid, one that should fail (`key` not in `T`), one where TS infers `T` from an object literal argument.
4. **Generic class:** Build a generic `Stack<T>` with `push`, `pop`, `peek`, `isEmpty`, and a static factory method `Stack.of<U>(...items: U[]): Stack<U>` — note why the static method needs its own type parameter.
5. **Variadic warm-up:** Write `Head<T>` and `Tail<T>` conditional-type utilities for tuples (peek ahead to Ch. 8 syntax if needed) and use them to implement a `Last<T>` utility.

---

**Next:** Chapter 7 — Advanced Types Part 1: `keyof`, `typeof`, `in`, indexed access, mapped types. Say **"next"** to continue.
