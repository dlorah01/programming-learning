---
layout: default
title: "Chapter 3 — The Type System: Structural Typing, Assignability, Variance"
---

# Chapter 3 — The Type System: Structural Typing, Assignability, Variance

This chapter is the load-bearing wall of everything else in this course. Generics, conditional types, function overloads, class design — all of it is downstream of "when does the checker consider type A assignable to type B." Take this chapter slowly.

## 3.1 Structural Typing

**Intuition.** TS doesn't ask "what is this thing called / where does it come from" — it asks "does this thing have the right shape." Two unrelated types, declared in different files with different names, are interchangeable if their members line up. This is fundamentally different from Java/C#, where you'd need explicit inheritance or interface implementation.

**Technical explanation.** A type `A` is assignable to type `B` if `A`'s members are a **superset-or-equal** of `B`'s required members, with each member itself being assignable per the same rules (recursively). Name, declaration site, and "intent" are irrelevant.

```ts
interface Point { x: number; y: number; }
class Vector { constructor(public x: number, public y: number) {} }

function log(p: Point) { console.log(p.x, p.y); }
log(new Vector(1, 2)); // OK — Vector structurally satisfies Point, no relationship declared
log({ x: 1, y: 2, z: 3 }); // also OK, unless it's a fresh literal (excess property check, Ch.2)
```

**Internal behavior.** The checker's `isTypeAssignableTo` / `isTypeRelatedTo` machinery walks both types' member lists and recursively checks each corresponding property, method signature, index signature, and call/construct signature. This is genuinely expensive in the worst case (it's why deeply recursive generic structural comparisons can blow up compile time) but is memoized/cached per type-pair within a compilation.

**Compiler behavior.** Structural comparison is *relational*, not identity-based — there's no "type ID" being compared. Two independently-written `interface`s with identical members are 100% interchangeable everywhere, with zero declared relationship.

**Type inference angle.** This is also *why* inference can flow so freely between unrelated-looking constructs — a returned object literal, a class instance, a `Pick<T,K>` result, and a hand-written interface can all satisfy the same target type without any of them "knowing" about each other.

**Real-world use cases.** Mocking/testing (pass a plain object literal wherever a class instance is expected, as long as the shape matches — huge for unit tests without heavy DI frameworks), duck-typed dependency injection, adapting third-party library return types to your own domain interfaces without wrapper classes.

**Good practices.** Design interfaces around the **minimum shape actually needed** by a function ("accept the narrowest interface, return the widest/most specific type") — structural typing rewards this because callers don't need to explicitly implement anything, just conform.
**Bad practices.** Relying on structural typing to accidentally treat semantically different things as interchangeable just because they share field names (`{ id: string }` for both `UserId` and `ProductId` — structurally identical, semantically catastrophic if swapped). This is exactly the problem branded types solve (Ch. 9).
**Common mistakes.** Assuming two objects are "the same type" because a `console.log` shows the same shape — TS cares about the *declared/inferred* type, and structural compatibility can still surprise you when optional properties or method parameter variance are involved (see §3.5–3.7).
**Edge case.** **Empty interfaces/object types (`{}`)** are structurally satisfied by almost anything (any non-nullish value) — a classic gotcha: `{}` doesn't mean "empty object," it means "any value that isn't `null`/`undefined`," because structurally, having *zero* required members is trivially satisfied by objects, arrays, functions, even primitives (`5` structurally has "at least zero properties").
**Performance.** Structural comparison cost is the dominant driver of TS compile time in large codebases with big/deep interfaces — this is the mechanical reason "your build got slow after adding this huge generic type" is such a common senior-level complaint (Ch. 8/13 revisit this with conditional types).

### Interview Q&A — §3.1
**Junior:** Q: What is structural typing? A: A type system where compatibility is determined by the actual shape/members of a type, not by its name or explicit declared relationships.
**Mid:** Q: If two unrelated classes `A` and `B` happen to have identical public properties, are instances interchangeable in TS? A: Yes — TS only checks shape compatibility, not class identity or inheritance, so an `A` instance is assignable wherever a `B`-shaped type is expected (and vice versa).
**Senior:** Q: Why does `{}` accept almost any value, and what's the practical danger of accidentally typing something `{}`? A: `{}` describes "an object type with zero required members," which is structurally satisfied by virtually everything except `null`/`undefined` — including primitives and functions. The danger: if you meant "an empty, specific object" you'll get a type that provides essentially no safety, silently accepting wrong values (a real source of "why did TS let me pass a number here" bugs).
**FAANG/tricky:** Q: Why is deep structural comparison a genuine, unavoidable compile-time cost driver, and what design choices mitigate it? A: Because assignability requires recursively comparing every member (and nested members) between two types with no shortcut via identity, the cost is proportional to the *shape complexity being compared*, not the number of lines of code. Mitigations: flatter interfaces over deeply nested ones, avoiding deeply recursive/conditional generic types on hot paths, `skipLibCheck` to avoid re-verifying dependency shapes, and splitting huge union/intersection types into named aliases (which enables the checker's caching by identity of the alias, somewhat reducing repeated full structural re-derivation).

**Predict the type / gotcha**
```ts
function acceptsAnything(x: {}) { }
acceptsAnything(5);          // OK
acceptsAnything("str");      // OK
acceptsAnything(null);       // Error under strictNullChecks
acceptsAnything(undefined);  // Error under strictNullChecks
```

---

## 3.2 Nominal Typing (and how TS fakes it)

**Intuition.** Nominal typing is "same shape isn't enough — you must be *declared* as this type" (Java's `class`/`interface` `implements`, C#, Rust). TS is structural by default and has **no built-in nominal typing**, but senior engineers routinely need it (e.g., don't let a `UserId` and `OrderId`, both `string`s, be interchanged) — so the ecosystem invented a workaround: **branded (a.k.a. nominal-emulated) types**.

**Technical explanation.**
```ts
type UserId = string & { readonly __brand: "UserId" };
type OrderId = string & { readonly __brand: "OrderId" };

function getUser(id: UserId) { /* ... */ }
declare const oid: OrderId;
getUser(oid); // Error — brands don't match, even though both are ultimately `string`
```
The `__brand` property doesn't exist at runtime (you construct these values via a function that just does a type assertion) — it's a **compile-time-only tag** that breaks structural interchangeability on purpose.

**Internal behavior.** Because the intersection type `string & { __brand: "UserId" }` structurally requires a property that no real `string` runtime value has, ordinary strings are not assignable to it — you're forced through an explicit constructor/assertion function, which is exactly the point: it creates an intentional, compiler-enforced "smart constructor" chokepoint.

**Compiler behavior / inference.** There's no special "brand" keyword — this is a pure application of structural typing rules (intersections + impossible-to-naturally-satisfy properties) to *simulate* nominal behavior. TS never sees "UserId" as special; it's just another structural shape that happens to be very hard to satisfy accidentally.

**Real-world use cases.** IDs of different entity types, units of measure (`Meters` vs `Feet`, both `number`), validated-vs-unvalidated data (`RawInput` vs `SanitizedInput`), currency amounts tagged by currency.

**Good practices.** Brand primitive types at domain boundaries where mixing them is a realistic, costly bug (money, IDs, units). Provide a single smart-constructor function per branded type (`function toUserId(s: string): UserId`) so validation logic has exactly one home.
**Bad practices.** Branding everything reflexively — adds friction for little gain when confusion between two strings is implausible or low-stakes.
**Common mistakes.** Forgetting the brand field must be non-existent on real values (using a plausible/realistic property name for the brand risks accidental real-world collision, however unlikely) — convention is an unexported, oddly-named symbol or string literal tag.
**Edge case.** Branding via a `unique symbol` field instead of a string literal tag is marginally stronger (truly cannot be replicated by accident from outside the module, since `unique symbol` types can only originate from one declaration) — a senior-level refinement over the plain string-tag brand pattern.
**Performance.** Zero runtime cost — it's purely a structural-typing trick, fully erased.

### Interview Q&A — §3.2
**Junior:** Q: Does TypeScript support nominal typing natively? A: No — TS is structurally typed by default; nominal-like behavior has to be simulated.
**Mid:** Q: How do "branded types" fake nominal typing in a structural system? A: By intersecting a base type with an otherwise-unsatisfiable marker property (e.g., `& { __brand: "X" }`), so only values explicitly constructed/asserted through a controlled function satisfy the type, blocking accidental structural interchange between two otherwise-identical primitive-based types.
**Senior:** Q: What's the practical difference between branding with a string-literal tag field vs a `unique symbol` field, and when would you choose the stronger one? A: A string-literal-tag brand can, in principle, be replicated by anyone who knows the exact tag string (via an assertion), so it's a convention-enforced boundary, not an absolute one; a `unique symbol` brand can only be produced by referencing the one module-scoped symbol declaration, making it effectively impossible to forge from outside — choose it for high-stakes domains (money, auth tokens) where even a determined/careless developer shouldn't be able to bypass it with a casual `as`.

**Predict/debug**
```ts
type Meters = number & { readonly __unit: "m" };
function toMeters(n: number): Meters { return n as Meters; }
const distance: Meters = toMeters(5);
const raw: number = distance; // OK — Meters is structurally a number (widening direction is fine)
const bad: Meters = 5;        // Error — plain number isn't a Meters without going through toMeters
```

---

## 3.3 Duck Typing

**Intuition.** "If it walks like a duck and quacks like a duck, treat it as a duck." This is the informal, dynamic-language version of the same idea structural typing formalizes statically — the term predates TS and applies to any language (including plain JS) where behavior/shape, not declared type, determines usability.

**Technical explanation.** In JS (no static types), duck typing happens at *runtime*: code just calls `.quack()` on whatever it's given and finds out at call time whether that works. TS's structural typing is essentially **duck typing checked at compile time instead of discovered at runtime** — same philosophy, different phase.

**Internal/compiler behavior.** No separate mechanism from §3.1 — this is a terminology distinction interviewers like to probe (do you understand that "structural typing" is TS's static formalization of the dynamic "duck typing" philosophy JS already had) more than a separate feature.

**Real-world use cases / good practices.** Recognizing that TS's structural rules are *not* an arbitrary design choice — they're the natural fit for a language layered on top of JS's inherently duck-typed runtime; fighting this (trying to force nominal-only patterns everywhere) works against the grain of both TS and JS.

### Interview Q&A — §3.3
**Junior:** Q: What's "duck typing"? A: The principle that an object's suitability for use is determined by whether it has the needed methods/properties, not by its declared type or class.
**Mid/Senior/FAANG:** Q: How does TS's structural typing relate to JS's duck typing? A: Structural typing is duck typing moved to compile time — TS statically verifies "does this shape have what's needed" instead of JS discovering it dynamically at the moment of use; TS was deliberately designed this way *because* it sits on top of an inherently duck-typed runtime, so a nominal type system would constantly fight against idiomatic JS patterns (object literals, mixins, plain-object mocking).

---

## 3.4 Type Compatibility & Assignability

**Intuition.** "Compatibility" is really just "can I use an A where a B is expected" — TS calls this **assignability**, and it's the single relation every other feature (function calls, generic inference, unions, overload resolution) reduces to internally.

**Technical explanation — the assignability rule set, roughly:**
- Primitive `A` assignable to primitive `B` iff `A` is the same or a literal subtype of `B`.
- Object `A` assignable to object `B` iff every required member of `B` exists (compatibly) on `A` — **width subtyping**: more properties is fine, fewer is not.
- Function `A` assignable to function `B` iff `A`'s parameters are compatible with `B`'s (contravariantly, mostly — §3.6) and `A`'s return type is assignable to `B`'s return type (covariantly).
- `A` assignable to `B` if `A` is a union and *every* member of the union is assignable to `B`; `A` assignable to `B` if `B` is a union and `A` is assignable to *at least one* member.

**Internal behavior.** This is implemented via a recursive relation checker (`checkTypeRelatedTo` internally) with three modes: identical, assignable (the practical day-to-day one), and subtype-of (used more narrowly, e.g. for certain inference scenarios). Assignability is deliberately looser/more pragmatic than a mathematically "sound" subtype relation in a few places — TS explicitly favors usability over 100% soundness (documented TS design goal), which explains several "gotchas" throughout this course (numeric enum openness, bivariant method params, `any`'s escape hatch).

**Compiler behavior / inference.** Assignability checks run constantly and invisibly: every function call checks argument-to-parameter assignability, every `return` checks return-value-to-declared-return-type, every variable initializer checks value-to-annotation.

**Real-world use cases.** Every single line of "does this compile" in TS is an assignability question — understanding this relation *is* understanding how to debug type errors quickly (ask: "what's actually being assigned to what, and which specific member/param/return breaks it").

**Good practices.** When debugging a confusing type error, mentally reduce it to "X is not assignable to Y because [specific member/signature]" — TS's error messages literally do this recursively; read the innermost cause, not just the outermost type names.
**Bad practices.** Reflexively reaching for `as`/`any` the moment an assignability error appears instead of reading *why* — usually the error is correctly catching a real mismatch.
**Common mistakes.** Assuming assignability is symmetric — it's almost never symmetric (`Cat` assignable to `Animal` doesn't imply `Animal` assignable to `Cat`); confusing "these two types are related" with "these two types are the same."
**Edge case.** TS documents itself as intentionally **unsound** in specific, bounded ways for ergonomics (array covariance, bivariant method parameters, `any`) — this is a real, citable design decision, not an accident, and is a great senior-interview talking point.
**Performance.** Assignability checking is the single most frequently executed operation in the checker — its implementation efficiency (caching per type-pair, structural short-circuiting) is central to `tsc` performance overall.

### Interview Q&A — §3.4
**Junior:** Q: What does "type A is assignable to type B" mean in practice? A: A value of type A can be used anywhere a value of type B is expected, without a compile error.
**Mid:** Q: Is assignability symmetric — if A is assignable to B, is B always assignable to A? A: No, generally not — assignability models "A is usable as B" (often because A is more specific/has more members), which is a one-directional (subtype-like) relation, not equivalence.
**Senior:** Q: TS is often described as an "intentionally unsound" type system. What does that mean and give two concrete examples. A: TS knowingly permits certain type-safety violations for practical ergonomics rather than mathematical soundness — e.g., array/generic covariance allowing `Cat[]` to be used as `Animal[]` even though pushing a `Dog` into it through the `Animal[]` view would break the underlying `Cat[]` invariant, and bivariant method-parameter checking (§3.7) allowing narrower-parameter override methods that a fully sound contravariant check would reject.

**Debugging exercise**
```ts
interface Handler { handle(event: { type: string }): void; }
const h: Handler = {
  handle(event: { type: "click" }) { /* ... */ } // does this compile?
};
```
→ Compiles under default (method-shorthand) bivariant checking — a narrower parameter type is accepted for **methods**, which would be rejected for a plain function-typed property (`handle: (event: {type:string}) => void`) under `strictFunctionTypes`. This exact asymmetry is covered in §3.7.

---

## 3.5 Variance — Overview

**Intuition.** Variance answers: "If `Cat` is a subtype of `Animal`, what's the relationship between `Container<Cat>` and `Container<Animal>`?" This question only exists because of **generic/parameterized types** (arrays, functions, `Promise<T>`, your own generic classes) — variance is about how subtyping "propagates" through a type constructor.

**Technical explanation — the four variance kinds:**
| Kind | Rule | Example |
|---|---|---|
| **Covariant** | `Sub<A>` is a subtype of `Sub<B>` when `A` is a subtype of `B` (relationship preserved) | Return types, array element types (in TS) |
| **Contravariant** | `Sub<A>` is a subtype of `Sub<B>` when `B` is a subtype of `A` (relationship flipped) | Function parameter types (soundly) |
| **Invariant** | `Sub<A>` and `Sub<B>` are related only if `A` and `B` are exactly the same | Mutable array element type, strictly (not TS's default choice — see below) |
| **Bivariant** | `Sub<A>` and `Sub<B>` are considered compatible in *either* direction (unsound, permissive) | TS method parameters (default, for ergonomics) |

**Visual mental model:**
```
Animal
  ▲
  │  subtype relationship
  │
 Cat

Covariant container:      Producer<Cat>  is-a  Producer<Animal>   (arrow direction preserved)
Contravariant container:  Consumer<Animal> is-a  Consumer<Cat>    (arrow direction flipped)
Invariant container:      Box<Cat> unrelated to Box<Animal>       (no arrow at all)
```
Intuition for *why* the direction flips for consumers: if you need something that can **accept** any `Animal`, a function that accepts any `Cat` is NOT good enough (it'd reject a `Dog`) — but a function that accepts *any object at all* (wider than `Animal`) is more than good enough. Consuming positions want **wider-or-equal**, i.e. contravariant. Producing positions want **narrower-or-equal**, i.e. covariant.

**Real-world use cases.** This governs whether overriding a method with a narrower parameter type is safe (it isn't, generally), whether a `(cat: Cat) => void` callback can be used where `(animal: Animal) => void` is expected (no, unsoundly-in-reverse would be needed), and whether `Promise<Cat>` can be used as `Promise<Animal>` (yes — `Promise` is covariant in its type parameter, matches intuition of "a promise that produces X").

### Interview Q&A — §3.5
**Junior:** Q: In plain words, what is variance? A: The rule for how the subtype relationship between two types affects the subtype relationship between generic/parameterized versions of those types (e.g., does `Cat[]` relate to `Animal[]` the same way `Cat` relates to `Animal`).
**Mid:** Q: Why is it intuitive that function parameters should be contravariant while return types are covariant? A: A function is safe to substitute if it can handle at least everything the original could receive (accepting a *wider* parameter type is safe — contravariant) and produces at least what's expected (returning a *narrower*/more specific type is safe — covariant) — this mirrors the Liskov Substitution Principle applied position-by-position.
**Senior:** Q: Give a concrete unsound example enabled by covariant arrays. A:
```ts
const cats: Cat[] = [new Cat()];
const animals: Animal[] = cats; // allowed — array covariance
animals.push(new Dog());        // compiles — Dog is an Animal
cats[1].meow();                 // runtime crash — cats[1] is actually a Dog
```
TS allows this for ergonomic reasons (arrays would be nearly unusable generically if strictly invariant), fully aware it's unsound.

---

## 3.6 Covariance & Contravariance in TS Specifically

**Technical explanation.**
- **Return types are covariant** (always, everywhere) — a function returning `Cat` is assignable to a function-type expecting a return of `Animal`, because callers only ever *consume* the return value, and any `Cat` is usable as an `Animal`.
- **Function parameters are contravariant under `strictFunctionTypes`** for standalone function types (`type Fn = (x: T) => void` syntax, and arrow-function-typed properties) — but see §3.7 for the method exception.
- **Arrays/generic type parameters are covariant by default** in TS (unsound, as shown above) — this is a deliberate, documented pragmatic choice; TS does not offer a way to declare true invariance for your own generics without manual workarounds.

**Internal behavior.** `strictFunctionTypes` specifically toggles whether the checker performs proper contravariant checking on function-type parameters (sound) vs the older bivariant check (unsound but more permissive) — it does **not** affect method-shorthand syntax at all (§3.7), which is a very deliberate, very-frequently-tested exception.

**Compiler behavior / inference.**
```ts
// strictFunctionTypes: true
type Handler = (e: Animal) => void;
const catHandler: Handler = (c: Cat) => {};   // Error — Cat handler can't handle a Dog passed as Animal
const wideHandler: Handler = (a: Animal | string) => {}; // OK — accepts at least as much as required
```

**Real-world use cases.** Event handler typing, callback-based APIs, Comparator/Predicate function parameters (`(a: T, b: T) => number`).

**Good practices.** Keep `strictFunctionTypes` on (it's part of `strict`) — the unsound alternative silently permits real bugs in event-handling code.
**Bad practices.** Widening a callback parameter type just to make an assignability error go away without checking whether the callback is actually safe to receive the wider type.
**Common mistakes.** Expecting standalone function-type variance rules to also apply to class/interface **methods** — they don't (§3.7), a distinction many working engineers never learn explicitly.
**Edge case.** Because TS arrays are covariant, mutating methods (`push`) on an array typed via a wider reference are a real unsoundness surface — mitigated only by discipline (`readonly T[]` where possible) not by the type system itself.
**Performance.** N/A — purely a soundness/design topic, not a compile-speed one.

### Interview Q&A — §3.6
**Junior:** Q: Are function return types covariant or contravariant in TS? A: Covariant — a function returning a more specific type can be used where a function returning a more general type is expected.
**Mid:** Q: What does `strictFunctionTypes` change? A: It makes function *parameter* checking properly contravariant (sound) for standalone function types, instead of the looser bivariant check.
**Senior:** Q: Why does `strictFunctionTypes` explicitly exclude method syntax? A: Because bivariant method-parameter checking, while technically unsound, matches extremely common and largely safe OOP override patterns (e.g., overriding a method to accept a more specific parameter type in a subclass) that show up constantly in real class hierarchies; the TS team judged that strictly enforcing contravariance there would reject too much legitimate, low-risk code relative to the bugs it would actually catch — a deliberate ergonomics-vs-soundness tradeoff (elaborated in §3.7).

---

## 3.7 Bivariance (and the Method vs Function-Property Distinction)

**Intuition.** This is the most-asked "gotcha" variance question in TS interviews: **identical-looking code behaves differently checked as a method vs. as a function-typed property**, purely due to syntax.

**Technical explanation.**
```ts
interface Animal { name: string; }
interface Cat extends Animal { meow(): void; }

interface UsingMethod  { handle(a: Animal): void; }       // method shorthand
interface UsingProperty { handle: (a: Animal) => void; }  // function-typed property

const m: UsingMethod = { handle(c: Cat) { c.meow(); } };     // OK — bivariant, even under strict
const p: UsingProperty = { handle(c: Cat) { c.meow(); } };   // Error under strictFunctionTypes — contravariant
```

**Internal behavior.** This is a hard-coded special case in the checker: method-shorthand-declared signatures (`handle(a: Animal): void` inside an interface/type/class) are deliberately checked **bivariantly** (either direction accepted) regardless of `strictFunctionTypes`, while property-syntax function types (`handle: (a: Animal) => void`) get the strict contravariant treatment. This isn't an accident or oversight — it's documented TS behavior, motivated by how common and how *usually*-safe this pattern is in real class hierarchies (overriding a handler to accept a narrower/more specific type is the everyday OOP override pattern, and strictly banning it would reject a huge amount of otherwise-reasonable code for a comparatively rare real bug).

**Compiler behavior.** This means the exact same runtime shape, described with two different (structurally supposedly equivalent!) syntaxes, gets **different compile-time strictness** — a genuinely surprising, high-signal senior/FAANG interview fact.

**Real-world use cases.** Explains why many OOP-style handler hierarchies (Express middleware-like patterns, event emitter subclasses) "just work" with narrower-parameter overrides without fighting the type checker, even under `strict`.

**Good practices.** Know this distinction exists so you can *choose* it deliberately — use method syntax when you intentionally want the more permissive override behavior (matches OOP intuition), use property syntax when you need full soundness (e.g., a plain callback-accepting API where accidental unsoundness would be a real production risk).
**Bad practices.** Being unaware of the distinction and being surprised when refactoring a method into a property (or vice versa) silently changes what compiles.
**Common mistakes.** Assuming `strictFunctionTypes` makes *everything* contravariant — it explicitly, by design, does not touch method syntax.
**Edge case.** This also affects generic class method overriding checks and is part of why some class hierarchies that "feel wrong" type-check fine — always worth explicitly testing if you're building library-grade class hierarchies where this matters.
**Performance.** N/A.

### Interview Q&A — §3.7
**Junior:** Q: Does `strictFunctionTypes` make method parameters in interfaces strictly contravariant? A: No — method-shorthand signatures are always checked bivariantly, regardless of `strictFunctionTypes`; only function-typed properties get strict contravariant checking.
**Mid:** Q: Rewrite `interface X { run(a: Animal): void }` as a property instead of a method, and explain what compile-time behavior changes. A: `interface X { run: (a: Animal) => void }` — now overriding/assigning an implementation with a narrower parameter (`run: (c: Cat) => void`) becomes a compile error under `strictFunctionTypes`, whereas the method form would accept it.
**Senior/FAANG:** Q: Why did the TypeScript team deliberately choose unsound bivariance for methods instead of applying the same sound contravariant rule everywhere? A: Empirically, an enormous amount of real-world OOP code relies on overriding methods with narrower parameter types (a very common, usually-safe pattern in practice, e.g. specialized event handlers), and enforcing strict contravariance there would produce a large number of forced-`any`/assertion workarounds for comparatively few actual bugs caught — a calculated ergonomics tradeoff the team has explicitly documented, prioritizing "usable type system for real JS/OOP code" over 100% theoretical soundness.

**Predict/debug — capstone variance exercise**
```ts
// strictFunctionTypes: true
class Animal {}
class Cat extends Animal { meow() {} }
class Dog extends Animal { bark() {} }

class Shelter {
  handle(a: Animal) { console.log("handling", a); }
}
class CatShelter extends Shelter {
  handle(c: Cat) { c.meow(); } // does this compile? is it safe?
}

const shelters: Shelter[] = [new CatShelter()];
shelters[0].handle(new Dog()); // if the above compiled, what happens here at runtime?
```
→ **Compiles** (method bivariance), and **crashes at runtime** (`c.meow is not a function` because a `Dog` was passed) — the textbook demonstration of why method bivariance is a documented, deliberate unsoundness, not a bug, and exactly the kind of scenario a FAANG interviewer wants you to trace through out loud.

---

## Practical Exercises — Chapter 3

1. **Structural proof:** Write two completely unrelated interfaces with identical members, and a function that accepts one — show a value of the other type-checks when passed in. Then add one extra required member to only one interface and show the call now fails, explaining exactly why via width subtyping.
2. **Build a branded type:** Design `Email` and `Username` branded types (both underlying `string`), with smart constructors that validate format before returning the branded value. Show that a raw string cannot be passed where `Email` is expected.
3. **Variance trace:** Given a `Comparator<T> = (a: T, b: T) => number` type, explain (without running code) whether `Comparator<Animal>` is assignable to `Comparator<Cat>`, or the reverse, and why — tie the answer explicitly to contravariance.
4. **Bivariance hunt:** Take an existing class hierarchy in a real codebase (or write one), convert a method to a property function type, and note every place `strictFunctionTypes` now produces a new error that method syntax was silently allowing.

---

## Chapter 3 Summary — Mental Model

```
                     ASSIGNABILITY (the master relation)
                              │
        ┌─────────────────────┼─────────────────────┐
   structural comparison   width subtyping      variance rules
   (member-by-member)    (extra props OK,      (how subtyping propagates
                          missing props not)     through generic params)
                                                        │
                                    ┌───────────────────┼───────────────────┐
                              covariant             contravariant       bivariant
                          (return types,          (function params,    (method params —
                           array elements —        strictFunctionTypes)  ALWAYS, by design,
                           TS: unsound for                               regardless of flags)
                           mutable arrays)

Nominal typing doesn't exist natively → simulated via branded types
   (structural rules + an unsatisfiable marker member = de facto nominal boundary)

Duck typing (JS, runtime) ⟷ Structural typing (TS, compile-time)
   — same philosophy, different phase of enforcement
```

Everything from here forward — generics (Ch. 6), conditional types (Ch. 8), overloads (Ch. 4) — is this chapter's rules applied to increasingly parameterized, increasingly abstract shapes. If assignability and variance feel solid, the rest of the type system will feel like *composition*, not new rules.

---

**Next:** Chapter 4 — Functions: function types, overloads, optional/default/rest parameters, `this` typing. Say **"next"** to continue.
