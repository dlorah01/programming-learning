---
layout: default
title: "Chapter 8 — Advanced Types II: Conditional Types, `infer`, Recursion"
---

# Chapter 8 — Advanced Types II: Conditional Types, `infer`, Recursion

## 8.1 Conditional Types

**Intuition.** A conditional type is an `if/else` for types: "if `A` is assignable to `B`, resolve to type `X`, otherwise resolve to type `Y`." It's how TS expresses type-level branching logic.

**Technical explanation.**
```ts
type IsString<T> = T extends string ? "yes" : "no";
type A = IsString<"hi">;   // "yes"
type B = IsString<42>;      // "no"
```
The `extends` here means "is assignable to" (same meaning as generic constraints, Ch. 6.4 — not class inheritance).

**Internal behavior.** The checker evaluates `T extends U ? X : Y` by checking assignability of `T` to `U` using the same machinery as any other assignability check (Ch. 3.4). When `T` is a **generic, not-yet-resolved** type parameter (inside another generic function/type), the conditional can't be immediately evaluated — it stays "deferred" until `T` is substituted with something concrete at the outer instantiation site. This deferral is essential: it's what lets conditional types be *composed* inside larger generic utilities without prematurely collapsing.

**Compiler behavior / inference.** Conditional types can be **chained** (a type-level `if/else if/else` ladder):
```ts
type TypeName<T> =
  T extends string ? "string" :
  T extends number ? "number" :
  T extends boolean ? "boolean" :
  T extends undefined ? "undefined" :
  T extends Function ? "function" :
  "object";
```
Each branch tests in order; the first match wins — structurally similar to the overload-resolution "first match" rule from Ch. 4.3, a nice pattern-recognition callback for interviews.

**Real-world use cases.** The entire mechanism behind `Exclude`, `Extract`, `NonNullable`, `ReturnType`, `Parameters`, `Awaited` (all Ch. 10) — conditional types are how TS's most commonly used utility types are actually implemented under the hood, not special compiler magic beyond this one construct plus `infer` (§8.2).

**Good practices.** Reach for a conditional type when you need the *shape of the output type to depend on the shape of the input type* — if the output shape is fixed regardless of input, you don't need conditionals, just plain generics (Ch. 6).
**Bad practices.** Building deeply nested conditional-type ladders for logic that would be clearer (and equally correct) as a small set of separate, named type aliases composed together — readability matters for types too.
**Common mistakes.** Forgetting a conditional type on an *unresolved* generic type parameter doesn't evaluate immediately — trying to "test" a generic conditional type's behavior by looking at its unsubstituted form is meaningless; you must instantiate it with a concrete type to see a result.
**Edge case.** Conditional types can reference themselves (recursively) — this is legal and is how recursive utility types (§8.5) work, but is also subject to the checker's recursion-depth safety limit ("Type instantiation is excessively deep and possibly infinite").
**Performance.** Deeply chained/nested conditional types are one of the primary drivers of slow `tsc` builds in advanced type-level code — each branch potentially requires a fresh assignability check, and this compounds across generic instantiations.

### Interview Q&A — §8.1
**Junior:** Q: What does `T extends U ? X : Y` mean? A: A type-level conditional — if `T` is assignable to `U`, the type resolves to `X`; otherwise it resolves to `Y`.
**Mid:** Q: Why doesn't `type IsString<T> = T extends string ? true : false` "evaluate" when you look at its raw declaration? A: Because `T` is an unresolved generic type parameter at the declaration site — the conditional is deferred until the type is instantiated with a concrete type argument, at which point the checker can actually perform the assignability check and pick a branch.
**Senior:** Q: Why are deeply chained conditional types described as a common compile-time performance concern? A: Each branch requires the checker to perform a real assignability check (itself potentially expensive for large/structural types, Ch. 3.1), and these checks compound across every generic instantiation site that uses the conditional type — in large codebases with many call sites and complex input types, this can measurably slow the overall build, which is why senior engineers profile and simplify hot, widely-used conditional-type utilities.

---

## 8.2 `infer`

**Intuition.** `infer` lets you **capture** a piece of a type you're pattern-matching against inside a conditional type, and bind it to a new type variable you can use in the "true" branch — it's type-level destructuring combined with pattern matching.

**Technical explanation.**
```ts
type ElementType<T> = T extends (infer U)[] ? U : never;
type X = ElementType<number[]>; // number
type Y = ElementType<string>;    // never — string isn't an array pattern
```
`infer U` says "whatever type fills this position in a matching array-shaped `T`, call it `U` and let me use it if the match succeeds."

**Internal behavior.** During the assignability check that a conditional type performs, when the checker encounters an `infer` position, it treats that spot as a "hole" to solve for — much like generic argument inference at a function call (Ch. 6.1), but applied structurally within the pattern being matched, rather than from function call arguments. If the match succeeds, the inferred type is bound and available in the true branch; if multiple candidate positions could bind the same `infer` variable (e.g., a union scenario), the checker computes either a union or intersection of candidates depending on the position's variance (roughly: covariant positions like return types union candidates, contravariant positions like function parameters intersect them — directly tied to Ch. 3.6's variance rules).

**Compiler behavior / inference — canonical real-world examples:**
```ts
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never;
type MyParameters<T> = T extends (...args: infer P) => any ? P : never;
type UnwrapPromise<T> = T extends Promise<infer U> ? U : T;
```
These are (close to) the actual implementations of the built-in `ReturnType`, `Parameters`, and part of `Awaited` (Ch. 10) — `infer` is precisely the mechanism that makes them possible.

**Real-world use cases.** Extracting function return/parameter types for wrapper/decorator utilities, unwrapping `Promise<T>` for async utility types, extracting array element types, extracting the first/rest of a tuple (Ch. 6.6's `Head`/`Tail`), parsing structure out of template literal types (Ch. 9).

**Good practices.** Name `infer` variables meaningfully (`infer ElementType` reads far better than `infer U` in non-trivial utilities) — type-level code benefits from the same naming discipline as value-level code.
**Bad practices.** Stacking many `infer` positions in a single conditional without intermediate named types when the pattern is genuinely complex — hurts both readability and error messages.
**Common mistakes.** Expecting `infer` to work outside a conditional type's `extends` clause — it's *only* valid syntax there; it's not a general-purpose type-level "capture" operator usable anywhere.
**Edge case.** Multiple `infer` occurrences of the *same* variable name at different structural positions constrain each other — the checker attempts to find a single type consistent with all occurrences, which can produce a union or an error depending on how conflicting the positions are, an advanced/tricky area worth testing explicitly rather than assuming.
**Performance.** Same cost profile as conditional types generally — `infer`-heavy pattern matching over large/recursive structures compounds with the recursive-type cost concerns in §8.5.

### Interview Q&A — §8.2
**Junior:** Q: What does `infer U` do inside `T extends SomeShape<infer U> ? U : never`? A: Captures whatever type fills that structural position in `T` (if `T` matches the shape), binding it to `U` for use in the true branch.
**Mid:** Q: Can `infer` be used outside a conditional type's `extends` clause? A: No — `infer` is only valid syntax within the `extends` clause of a conditional type; it's a pattern-matching construct specific to that context, not a general type-capture keyword.
**Senior:** Q: How does variance affect what type `infer` produces when the same `infer` variable appears in multiple positions, e.g., in a function type with multiple parameters of the same inferred type? A: The checker considers the position's variance (Ch. 3.5/3.6) — covariant positions (like multiple possible return types across a union) tend to combine inferred candidates into a union, while contravariant positions (like function parameters) tend to combine into an intersection, mirroring the general rule that consuming positions want the narrowest-safe (intersected) type and producing positions want the widest-safe (unioned) type.
**FAANG/tricky:** Q: Write a conditional type that extracts the resolved type of a `Promise`, but only unwraps *one* level (doesn't recursively unwrap nested promises) — then explain why real `Awaited<T>` needs to be recursive instead. A: `type UnwrapOnce<T> = T extends Promise<infer U> ? U : T;` — this only peels one layer, so `UnwrapOnce<Promise<Promise<number>>>` gives `Promise<number>`, not `number`. Real `Awaited<T>` (Ch. 10) must recursively re-apply itself to the inferred `U` (`T extends Promise<infer U> ? Awaited<U> : T`) to correctly model how `await` on a promise-of-a-promise fully unwraps in real JS semantics — directly motivating recursive conditional types (§8.5).

**Predict the inferred type / debugging exercise**
```ts
type FirstArg<T> = T extends (arg: infer A, ...rest: any[]) => any ? A : never;
type X = FirstArg<(name: string, age: number) => void>; // string
type Y = FirstArg<() => void>;                            // never — no first arg to infer
```

---

## 8.3 Distributive Conditional Types

**Intuition.** When a conditional type is applied to a **union**, and the type being tested is a "naked" (bare, unwrapped) generic type parameter, TS doesn't test the whole union at once — it tests **each member separately** and unions the results. This is the single most-surprising, most-tested behavior in this entire course.

**Technical explanation.**
```ts
type ToArray<T> = T extends unknown ? T[] : never;
type Result = ToArray<string | number>; // string[] | number[]  — NOT (string | number)[]
```
This happens because `T` in `T extends unknown ? T[] : never` is a bare, unmodified type parameter directly in the `extends` clause's left side — this specific syntactic shape is what triggers distribution.

**Internal behavior.** The checker's distribution rule: when the checked type (`T` in `T extends U ? X : Y`) is literally a naked type parameter and is instantiated with a union type, the compiler expands the conditional into "apply this conditional to each union member individually, then union the results" — mechanically equivalent to writing out `(A extends U ? X : Y) | (B extends U ? X : Y) | ...` for `T = A | B | ...`.

**Compiler behavior — how to opt out of distribution.** Wrapping `T` in a tuple (or any non-naked structural position) on both sides of `extends` suppresses distribution:
```ts
type ToArrayNonDist<T> = [T] extends [unknown] ? T[] : never;
type Result2 = ToArrayNonDist<string | number>; // (string | number)[] — whole union tested at once
```
This `[T] extends [U]` idiom is the standard, universally-recognized way to deliberately disable distribution when you want the union treated as one unit — a genuinely essential piece of idiomatic advanced TS knowledge.

**Real-world use cases.** This is *exactly* how `Exclude<T, U>` and `Extract<T, U>` (Ch. 10) work — `Exclude<T, U> = T extends U ? never : T` relies entirely on distribution to filter a union member-by-member, mapping excluded members to `never` (which then vanishes from the resulting union, since `A | never` simplifies to `A`).

**Good practices.** When writing a conditional-type utility meant to operate on unions member-by-member (filtering, per-member transformation), rely on (and document) the naked-type-parameter distribution behavior deliberately. When you specifically need the union treated as a single unit, use the `[T] extends [U]` non-distributive idiom explicitly and comment why.
**Bad practices.** Writing distributive conditional-type utilities without realizing distribution is happening, then being confused why a union input produces a union of transformed results instead of one transformed union.
**Common mistakes.** The single most common "gotcha" in this entire course: expecting `T extends X ? A : B` to test the whole union `T` as one thing when `T` is instantiated with a union — it doesn't, by default, whenever `T` is a naked type parameter.
**Edge case.** Distribution also affects `never`: since `never` is (formally) treated as the empty union, a distributive conditional type applied to `never` distributes over *zero* members and immediately resolves to `never` itself, regardless of what the branches would otherwise produce — this is precisely why `Exclude<never, X>` is `never`, and is a genuinely tricky, real interview question.
**Performance.** Distribution over large unions means the conditional is effectively evaluated once *per union member* — for very large unions (hundreds of literal members, e.g. from template literal type combinations, Ch. 9), this multiplies the cost of whatever work the conditional does, a real, citable compile-time concern.

### Interview Q&A — §8.3
**Junior:** Q: Given `type Wrap<T> = T extends unknown ? T[] : never;`, what does `Wrap<string | number>` produce? A: `string[] | number[]` — the conditional distributes over each union member separately rather than treating `string | number` as a single unit.
**Mid:** Q: What specific syntactic condition triggers distributive behavior in a conditional type? A: The type being tested on the left of `extends` must be a **naked, unwrapped generic type parameter** instantiated with a union — if it's wrapped in any structural context (like a tuple `[T]`), distribution is suppressed and the whole union is tested as one unit.
**Senior:** Q: How would you write `Exclude<T, U>` from scratch, and explain exactly why distribution is essential to its correctness? A: `type MyExclude<T, U> = T extends U ? never : T;` — when `T` is instantiated with a union, distribution tests each member individually against `U`; members that match `U` map to `never` (which effectively disappears from the resulting union, since unioning with `never` is a no-op), while non-matching members pass through unchanged — without distribution, the entire union `T` would be tested against `U` as one unit, either entirely matching or entirely not, which couldn't produce the intended per-member filtering behavior at all.
**FAANG/tricky:** Q: What does `Exclude<never, string>` evaluate to, and why? A: `never` — because a distributive conditional type applied to `never` (the empty union) distributes over zero members, producing the empty union (`never`) as the result regardless of what the true/false branches would otherwise compute; this is a direct, sometimes-surprising consequence of treating `never` as literally "a union of nothing" within the distribution mechanism.

**Predict/debug — capstone distribution exercise**
```ts
type NonNullableCustom<T> = T extends null | undefined ? never : T;
type Result = NonNullableCustom<string | number | null | undefined>;
// Result: string | number
// Trace: distributes over string, number, null, undefined individually;
// null and undefined each match the left branch → never (vanish);
// string and number don't match → pass through unchanged.
```

---

## 8.4 Built-in / Standard Conditional-Type Idioms (bridge to Ch. 10)

A quick reference of the canonical hand-rollable versions, since these are extremely common interview "implement X from scratch" questions:
```ts
type MyExclude<T, U> = T extends U ? never : T;
type MyExtract<T, U> = T extends U ? T : never;
type MyNonNullable<T> = T extends null | undefined ? never : T;
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never;
type MyParameters<T> = T extends (...args: infer P) => any ? P : never;
type MyInstanceType<T> = T extends new (...args: any[]) => infer R ? R : never;
type MyAwaited<T> = T extends Promise<infer U> ? MyAwaited<U> : T; // recursive — see §8.5
```

### Interview Q&A — §8.4
**Mid/Senior:** Q: Implement `Extract<T, U>` from scratch and explain the difference from `Exclude`. A: `type MyExtract<T, U> = T extends U ? T : never;` — the branches are simply swapped relative to `Exclude`: `Extract` keeps members that *do* match `U` (mapping non-matches to `never`), while `Exclude` keeps members that *don't* match. Both rely on the same distributive mechanism (§8.3).

---

## 8.5 Recursive Types

**Intuition.** Some real-world shapes are inherently self-referential — a JSON value can contain other JSON values, a tree node contains other tree nodes, a nested array can contain further nested arrays. Recursive types let a type alias reference itself (directly or through a chain) to model this precisely.

**Technical explanation.**
```ts
type Json = string | number | boolean | null | Json[] | { [key: string]: Json };

type DeepReadonly<T> = T extends object
  ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
  : T;

type Awaited2<T> = T extends Promise<infer U> ? Awaited2<U> : T; // recursive infer, from §8.4
```

**Internal behavior.** The checker resolves recursive types **lazily** — it doesn't try to fully "expand" a recursive type infinitely up front (which would be impossible); instead, it resolves one level at a time as needed, exactly mirroring the lazy, on-demand resolution strategy from Ch. 1.3's general checker behavior. To prevent genuinely infinite/runaway recursion (e.g., a mistakenly-unbounded recursive type applied to a type that never "bottoms out"), TS enforces a **hard recursion-depth limit**, producing the well-known `error TS2589: Type instantiation is excessively deep and possibly infinite` when exceeded.

**Compiler behavior / inference.** Recursive conditional types (like `Awaited`) need a genuine **base case** (a branch that doesn't recurse further) to terminate — exactly like recursive functions at the value level; get the base case wrong (or omit it) and you'll hit the depth-limit error on any real usage.

**Real-world use cases.** JSON-shaped types, tree/AST node types, deeply-nested partial/readonly utility types (`DeepPartial<T>`, `DeepReadonly<T>` — extremely common in real-world state-management and config-merging code), recursive `Awaited<T>` (the actual built-in, Ch. 10), recursive tuple-processing utilities (Ch. 6.6's `Head`/`Tail` used repeatedly to walk an entire tuple).

**Good practices.** Always design a recursive type with an explicit, reachable base case; test recursive utility types against deeply nested real-world data shapes (not just shallow examples) to catch depth-limit issues before they surface in a teammate's actual code.
**Bad practices.** Writing "deep" utility types without considering termination — a recursive type that never reaches a base case for some valid input will either infinite-loop the checker's recursion guard (hard error) or, worse, silently behave incorrectly for edge-case inputs.
**Common mistakes.** Applying a `Deep*` utility type to an object containing functions, class instances, or other "should not be recursed into" values without excluding them — naive `DeepReadonly` implementations, for example, can accidentally try to recurse into function types (which have no meaningful "properties" to make readonly) or built-ins like `Date`/`Map`, producing nonsensical or broken results; production-grade deep utilities explicitly special-case these.
**Edge case.** The exact recursion depth limit is an implementation detail that has changed across TS versions (generally increased over time as the checker has been optimized) — don't hard-code assumptions about "how deep is too deep" into your mental model; if you're hitting the limit in reasonable real-world code, that's usually a sign to simplify the recursive type's structure, not a sign to seek a workaround for infinite depth.
**Performance.** Recursive types are, alongside deeply-chained conditional types, one of the single biggest real-world `tsc` performance concerns for advanced type-level libraries — every additional recursion level compounds the checker's work, and this is the most commonly cited reason for "why did adding this utility type slow down our whole build."

### Interview Q&A — §8.5
**Junior:** Q: Can a TypeScript type alias reference itself? A: Yes — recursive type aliases are legal and are the standard way to model self-referential shapes like JSON values or tree structures.
**Mid:** Q: What error do you get if a recursive type never terminates (has no reachable base case) for some input, and why does it happen? A: `TS2589: Type instantiation is excessively deep and possibly infinite` — the checker has a hard recursion-depth limit specifically to prevent truly infinite recursive expansion from hanging the compiler; hitting it usually means either a genuinely infinite input structure or, more commonly, a missing/incorrect base case in the recursive type.
**Senior:** Q: Why must `Awaited<T>` be implemented recursively rather than as a single-level conditional type? A: Because real JS `await` semantics fully unwrap nested promises (`await` on a `Promise<Promise<number>>` ultimately yields `number`, not `Promise<number>`) — a single-level `T extends Promise<infer U> ? U : T` only peels one layer, so it would incorrectly leave a nested promise type unresolved; the recursive form re-applies itself to the inferred inner type until it stops matching `Promise<...>`, correctly modeling full unwrapping.
**FAANG/tricky:** Q: Why does a naive `DeepReadonly<T>` implementation often misbehave on real-world data containing `Date`, `Map`, class instances, or functions? A: A naive version (`T extends object ? { readonly [K in keyof T]: DeepReadonly<T[K]> } : T`) treats *any* object-like type uniformly, including built-ins like `Date`/`Map`/functions, which have internal structure or behavior not meant to be re-mapped as plain data properties — recursing into them can produce nonsensical types (e.g., trying to make a `Date`'s internal methods "readonly" in ways that don't correspond to anything meaningful) or simply strip away their special/callable nature; production implementations explicitly special-case these built-ins (checking for `T extends Function`, `T extends Date`, etc., before falling through to the generic object-recursion branch).

**Predict/debug — capstone recursive-type exercise**
```ts
type DeepPartial<T> = T extends object
  ? T extends Function
    ? T
    : { [K in keyof T]?: DeepPartial<T[K]> }
  : T;

interface Config {
  server: { port: number; host: string };
  onError: (err: Error) => void;
}
type PartialConfig = DeepPartial<Config>;
// { server?: { port?: number; host?: string }; onError?: (err: Error) => void }
// Note: onError itself becomes optional, but its own signature is preserved
// unchanged (not recursed into) because of the `T extends Function ? T : ...` guard.
```

---

## Practical Exercises — Chapter 8

1. **Implement from memory:** Write `Exclude`, `Extract`, `NonNullable`, `ReturnType`, and `Parameters` from scratch, without looking at Ch. 10 or `lib.es5.d.ts` — then compare against the real built-ins.
2. **Distribution trap:** Predict, then verify, the result of applying a distributive conditional type to `never`, to a single non-union type, and to a 4-member union — write out the "expanded" mental-model form for each.
3. **Non-distributive rewrite:** Take a distributive conditional type and rewrite it using the `[T] extends [U]` idiom to compare the two behaviors side-by-side on the same union input.
4. **Recursive JSON type:** Write a `Json` recursive type (as in §8.5) and a `flatten<T extends Json>` utility type that (conceptually) describes flattening nested arrays — focus on correct base cases.
5. **Debug a depth-limit error:** Deliberately write a recursive type missing its base case, trigger `TS2589`, then fix it — document exactly what change fixed it and why.

---

**Next:** Chapter 9 — Advanced Types III: template literal types and branded/nominal types. Say **"next"** to continue.
