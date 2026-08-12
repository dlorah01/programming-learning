---
layout: default
title: "Chapter 16 — Capstone: The Complete Mental Model & Mock Interview"
---
# Chapter 16 — Capstone: The Complete Mental Model & Mock Interview

## 16.1 The Complete Mental Model

Everything in this course reduces to a small number of load-bearing ideas. If you internalize these, you can derive almost everything else on the spot, even for questions you've never seen before.

```
┌──────────────────────────────────────────────────────────────────────────┐
│  0. FOUNDATION: TypeScript = JavaScript + a type-checking phase that      │
│     is fully erased before runtime (with rare, deliberate exceptions:     │
│     enums, namespaces-with-values, class fields, decorator metadata).     │
│     The compiler pipeline: Scan → Parse (AST) → Bind (Symbols) →          │
│     Check (types) → Emit (erase + downlevel). Checking and emitting are   │
│     independent — errors don't block output unless configured to.        │
└──────────────────────────────────────────────────────────────────────────┘
                                     │
        ┌────────────────────────────┼────────────────────────────┐
        ▼                            ▼                            ▼
┌───────────────┐          ┌──────────────────┐          ┌──────────────────┐
│ 1. THE TYPE    │          │ 2. ASSIGNABILITY  │          │ 3. INFERENCE &    │
│    UNIVERSE    │          │    (the master     │          │    NARROWING       │
│                │          │    relation)        │          │    (types change    │
│ every value's  │          │                     │          │    per program      │
│ type sits      │          │ structural, member- │          │    point via CFA)   │
│ between:       │          │ by-member           │          │                     │
│  unknown (top) │          │ comparison; width    │          │ bottom-up vs        │
│  and never     │          │ subtyping; governs    │          │ top-down/           │
│  (bottom)      │          │ EVERY function call,   │          │ contextual          │
│                │          │ assignment, return    │          │ inference;          │
│ literal types  │          │                       │          │ widening on         │
│ widen on       │          │ VARIANCE = how it     │          │ let/const;          │
│ let, stay      │          │ propagates through     │          │ narrowing via       │
│ narrow on      │          │ generics:              │          │ CFA (typeof,        │
│ const (unless  │          │  - covariant (return)  │          │ instanceof, in,     │
│ mutated)       │          │  - contravariant       │          │ discriminants,      │
│                │          │    (params, strict)     │          │ guards, asserts);   │
│ as const       │          │  - bivariant (methods,  │          │ exhaustiveness via   │
│ freezes deep   │          │    always, by design)   │          │ never                │
└───────┬────────┘          └──────────┬─────────────┘          └──────────┬──────────┘
        │                              │                                    │
        └──────────────┬────────────────┴──────────────┬────────────────────┘
                        ▼                                ▼
            ┌──────────────────────┐         ┌────────────────────────────┐
            │ 4. COMPOSITION         │         │ 5. TYPE-LEVEL PROGRAMMING    │
            │                        │         │                              │
            │ interface (mergeable,  │         │ keyof / typeof / indexed      │
            │ extends, checked       │         │ access = INTROSPECTION        │
            │ upfront) vs type       │         │ mapped types ([K in U])       │
            │ (closed, can alias      │         │   = TRANSFORMATION            │
            │ anything)               │         │ conditional types              │
            │                        │         │   (T extends U ? X : Y)        │
            │ & intersection          │         │   = BRANCHING (deferred        │
            │  (computed, can        │         │   until T is concrete)         │
            │  silently → never)      │         │ infer = PATTERN CAPTURE        │
            │ | union                │         │ distribution = implicit         │
            │  (only common-to-all    │         │   ITERATION over naked         │
            │  ops without narrowing) │         │   union type params            │
            │                        │         │ recursive types = LOOPING       │
            │ generics = type-level   │         │   (needs a base case)          │
            │ variables, linking      │         │ template literals =             │
            │ positions together      │         │   STRING-LEVEL computation      │
            └───────────┬────────────┘         └──────────────┬──────────────┘
                        │                                       │
                        └───────────────────┬───────────────────┘
                                             ▼
                              ┌────────────────────────────────┐
                              │ 6. EVERYTHING ELSE IS BUILT       │
                              │    FROM THESE SIX IDEAS:           │
                              │                                    │
                              │ • Every utility type (Ch.10) =      │
                              │   a named composition of #4 + #5    │
                              │ • Branded types (Ch.3/9) = #4's      │
                              │   intersection + an unsatisfiable    │
                              │   marker, faking nominal typing      │
                              │   atop a structural system (#2)      │
                              │ • Declaration merging (Ch.5/12) =    │
                              │   the BINDER combining multiple      │
                              │   declarations into one Symbol       │
                              │ • Modules/.d.ts (Ch.11) = how #0-5   │
                              │   cross file/package boundaries      │
                              │ • Frameworks (Ch.15) = #1-5 applied  │
                              │   to real domain problems            │
                              └────────────────────────────────┘
```

**The one-sentence version, if you only remember one thing:**
*TypeScript layers a structural, gradually-narrowable, fully-erased type system on top of JavaScript, where almost every advanced feature is one of six primitives (assignability, variance, inference/narrowing via control-flow analysis, composition via unions/intersections/generics, or type-level introspection/transformation/branching) combined in a new way — and the compiler is honest about being deliberately, documented-ly unsound in a handful of specific places for the sake of real-world JS ergonomics.*

**The "deliberately unsound" list — worth having memorized as a set:**
1. Array/generic covariance (mutable `Cat[]` usable as `Animal[]`, Ch. 3.5–3.6)
2. Bivariant method parameters (Ch. 3.7) — always, regardless of `strictFunctionTypes`
3. Numeric enums accept any `number` (Ch. 2.6)
4. `any` breaks the entire subtype lattice on purpose (Ch. 2.1/3.4)
5. Excess property checking is a heuristic, bypassable via an intermediate variable (Ch. 2.3)
6. User-defined type guards are trusted, not verified (Ch. 13.3)
7. `as` assertions perform no validation at all (Ch. 3.4/14.1)

Knowing this list cold — and being able to explain *why* each exists (ergonomics vs. soundness tradeoff) rather than just that it exists — is a genuinely reliable signal of senior-level depth in an interview.

---

## 16.2 Full Mixed Mock Interview

Answer each before checking the answer beneath it. Mix of levels, mixed order — like a real interview.

---

**Q1 (Junior).** What's the difference between `interface` and `type` in one sentence?
> `interface` supports declaration merging and is used for object/callable/constructable shapes; `type` can alias anything (unions, tuples, primitives, computed types) but can't be redeclared/merged.

**Q2 (Mid).** Why does `let x = 5` have type `number` but `const x = 5` has type `5`?
> `let` allows reassignment, so TS widens to the general type to permit future compatible values; `const` guarantees the binding never changes, so the precise literal type is preserved.

**Q3 (Senior).** Explain why `Omit<A | B, "key">` can silently produce a wrong/merged type instead of per-member omission.
> `Omit` is implemented via `Pick<T, Exclude<keyof T, K>>`, and `keyof` on a union computes the *intersection* of each member's keys, not the union — so `Omit` applied directly to a union only "sees" keys common to every member, losing per-member distinctness. Fix: `DistributiveOmit<T,K> = T extends unknown ? Omit<T,K> : never`, forcing distribution over each member first.

**Q4 (FAANG/tricky).** What does `Exclude<never, string>` evaluate to, and why?
> `never` — a distributive conditional type applied to `never` (the empty union) distributes over zero members, so the result is the empty union (`never`) regardless of what the branches compute.

**Q5 (Junior).** What does `unknown` let you do that `any` doesn't prevent?
> `unknown` forces narrowing before any operation is allowed on the value; `any` disables checking entirely, silently allowing any operation, even unsafe ones.

**Q6 (Mid).** Why does a numeric `enum` produce real JavaScript at runtime, while an `interface` produces nothing?
> Regular (non-`const`) enums compile to an actual runtime object with a bidirectional value↔name mapping, because that mapping is genuinely useful/expected JS behavior; interfaces are purely compile-time structural contracts with no runtime existence at all.

**Q7 (Senior).** A class method override with a narrower parameter type compiles under `strict: true`. Why doesn't `strictFunctionTypes` catch this?
> `strictFunctionTypes` only applies sound, contravariant parameter checking to *standalone function types* (arrow-type aliases, function-typed properties) — method-shorthand signatures are deliberately, always checked bivariantly regardless of the flag, a documented ergonomics-over-soundness tradeoff for common OOP override patterns.

**Q8 (FAANG).** Trace through: why does this compile but crash at runtime?
```ts
class Shelter { handle(a: Animal) {} }
class CatShelter extends Shelter { handle(c: Cat) { c.meow(); } }
const shelters: Shelter[] = [new CatShelter()];
shelters[0].handle(new Dog());
```
> Method bivariance (Q7) lets `CatShelter.handle` narrow its parameter to `Cat` without error. At runtime, `shelters[0]` is a real `CatShelter` instance; calling `.handle(new Dog())` through the `Shelter[]`-typed array invokes `CatShelter.handle` with an actual `Dog`, and `dog.meow()` doesn't exist — a `TypeError` at runtime, demonstrating the documented unsoundness concretely.

**Q9 (Junior).** What's the difference between `?:` (optional parameter) and `= value` (default parameter) inside the function body?
> Inside the body, an optional parameter's type includes `undefined` (must be narrowed before use as the base type); a default parameter's type excludes `undefined`, because the compiler knows the default guarantees a value by the time the body executes.

**Q10 (Mid).** What does `keyof (A | B)` evaluate to, in terms of `keyof A` and `keyof B`?
> `keyof A & keyof B` — the intersection, not the union — because only keys guaranteed to exist on every possible union member are safe to access without first narrowing to a specific member.

**Q11 (Senior).** Why is `readonly T[]` a "free" safety improvement to add to a function parameter?
> A mutable `T[]` is freely assignable to a `readonly T[]`-typed parameter (arrays are covariant, and read-only access is always safe from a wider-capability type), so it costs callers nothing while compile-time-enforcing that the function won't mutate their array — pure upside with no downside for legitimate callers.

**Q12 (FAANG).** Implement `DeepReadonly<T>` and explain the one guard clause that prevents it from misbehaving on real-world data.
```ts
type DeepReadonly<T> = T extends Function ? T :
  T extends object ? { readonly [K in keyof T]: DeepReadonly<T[K]> } : T;
```
> Without the `T extends Function ? T : ...` guard, the recursion would try to treat function types as plain data objects to be mapped over, which is meaningless (functions don't have enumerable "properties" in the relevant sense) and can produce broken/nonsensical types; production-grade deep utilities also typically special-case `Date`, `Map`, `Set`, and other built-ins with the same reasoning.

**Q13 (Junior).** Does `tsc` still produce a `.js` file if there are type errors?
> Yes, by default — unless `noEmitOnError: true` is set, type-checking and emitting are independent phases, and diagnostics don't block emit.

**Q14 (Mid).** Why can't a `static` method access a generic class's own instance type parameter `T`?
> Static members belong to the class itself, independent of any specific instantiation — there's no concrete `T` bound for them to refer to, since `T` is only bound per-instance at construction time.

**Q15 (Senior).** Explain why module augmentation (`declare module "express" {...}`) can compile successfully but have zero actual effect anywhere.
> Declaration merging (Ch. 5.2/11.5) is fundamentally Symbol-based — it requires the augmenting declaration to resolve to the *exact same* module Symbol the library itself uses internally. If the module specifier resolves through a different path (a re-export barrel, mismatched package structure), TS creates a new, unrelated ambient module declaration instead of merging with the real one — syntactically valid, semantically inert.

**Q16 (FAANG/tricky).** Why does `Function.prototype.bind`'s type definition need variadic tuple types rather than a plain `T[]` rest parameter?
> `bind` must track exactly which specific parameter types are pre-filled (by the arguments passed to `bind` itself) and which remain, position by position, to compute the correctly-typed remaining function signature — collapsing all parameters into one unioned array type would destroy exactly the positional precision needed for that computation.

**Q17 (Junior).** What does the `?? ` nullish coalescing operator's presence in a codebase suggest about `strictNullChecks`?
> It suggests `strictNullChecks` is likely enabled (or at least that the team cares about explicit null/undefined handling) — `??` is specifically useful for providing defaults only when a value is `null`/`undefined` (as opposed to `||`, which also triggers on other falsy values like `0` or `""`), a distinction that matters most when the type system is actually tracking nullability precisely.

**Q18 (Mid).** Why does `satisfies` preserve more type information than an equivalent type annotation?
> An annotation (`const x: T = value`) makes `x`'s type *become* `T`, widening away any more specific literal/structural information the value's own bottom-up inference would have preserved; `satisfies` validates `value` against `T` (same error-catching power) but leaves `x`'s actual type as whatever was naturally inferred from `value` itself.

**Q19 (Senior).** Why is a `Result<T, E>` return type considered a stronger correctness guarantee than relying on `try`/`catch` with good documentation?
> `Result<T,E>` makes the possibility of failure part of the function's *type signature*, and the union's structure forces callers to narrow before accessing the success value (Ch. 5.4/13.4) — the compiler enforces handling. A thrown exception is entirely invisible to the type system; nothing prevents a caller from simply forgetting to wrap a call in `try`/`catch`, and no compiler error results from that omission, no matter how well-documented the possible exception is.

**Q20 (FAANG, capstone).** Walk through, end to end, what happens when the compiler processes `const nums = [1, 2, 3].map(n => n * 2);` — name every stage/mechanism from this course that's involved.
> **Scanner/Parser** (Ch. 1.3) tokenize and build the AST for the array literal, method call, and arrow function. **Binder** creates Symbols for `nums` and resolves `map` against `Array<number>.prototype.map`'s declared generic signature. **Bottom-up inference** (Ch. 13.1) computes `[1,2,3]`'s type as `number[]` (each literal widens per Ch. 2.8's rules for a `const`-but-mutable-array context — actually here, since it's an array literal not a bare `const x = 1`, the elements individually infer as `number`, unioned/homogeneous — no per-element widening surprise since they're already all `number`). **Contextual typing** (Ch. 4.1/13.1) pushes `Array<number>.map`'s callback parameter type down into the arrow function, typing `n` as `number` without explicit annotation. **Generic inference** (Ch. 6.1) resolves `map<U>`'s type parameter `U` from the callback's return type (`n * 2` → `number`), so `U = number`. The **checker** verifies the whole chain's assignability (Ch. 3.4) at each step. Finally `nums`'s declared type is `number[]`, and the **emitter** (Ch. 1.3) strips all of this type information entirely, producing plain `[1, 2, 3].map(n => n * 2)` as the actual runtime JS.

---

## 16.3 Closing: How to Keep Going

You now have the full conceptual map. From here, depth comes from **doing**, not reading further:
- Re-read a complex library's `.d.ts` file (React's, Express's, or a smaller focused library) and identify every mechanism from this course by name as you encounter it.
- Deliberately break things: remove a `strict` flag, weaken a type, introduce the "deliberately unsound" list's patterns on purpose, and observe exactly what stops being caught.
- Implement a small `Result<T,E>` or validation library from scratch, purely as a type-design exercise.
- Revisit Chapter 8/9 whenever you hit an unfamiliar library type definition full of `infer`/conditional types — it will now be legible instead of intimidating.

The roadmap file (`00-roadmap.md`) is fully checked off. This curriculum is complete — everything from "why does TypeScript exist" to tracing a single line of code through the entire compiler pipeline is now covered, cross-referenced, and drilled with interview material at every level.
