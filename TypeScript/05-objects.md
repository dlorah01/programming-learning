# Chapter 5 — Objects: Interfaces, Type Aliases, Unions & Intersections

## 5.1 Interfaces vs Type Aliases

**Intuition.** Both name a shape. The difference isn't "what they can describe" (mostly overlapping) — it's *how they behave as declarations*: interfaces are open/extensible and merge; type aliases are closed, single-assignment bindings that can name anything (not just object shapes).

**Technical explanation.**
```ts
interface User { id: string; name: string; }
type UserT = { id: string; name: string; };
```
For plain object shapes, these are structurally identical and interchangeable. The differences appear at the edges:

| Capability | `interface` | `type` |
|---|---|---|
| Object/shape description | ✅ | ✅ |
| Union types | ❌ | ✅ (`type A = X \| Y`) |
| Primitive/tuple/function aliasing | ❌ (no `interface = string`) | ✅ |
| Declaration merging (same name, multiple declarations combine) | ✅ | ❌ (error: duplicate identifier) |
| `extends` (explicit, checked subtyping intent) | ✅ | via `&` intersection (different mechanism, similar effect) |
| Can be implemented by a class (`implements`) | ✅ | ✅ (object-shaped ones) |
| Mapped types, conditional types | ❌ | ✅ |

**Internal behavior.** Interfaces are resolved by the **binder** as multiple declarations merging into a single Symbol with a combined member list (this merging is a first-class binder feature, not a checker trick). Type aliases are a single Symbol pointing at one type expression — there is no merging mechanism for them at all, by design; redeclaring a `type` with the same name is simply a duplicate-identifier error.

**Compiler behavior.** Because interfaces merge, `lib.dom.d.ts`-style global augmentation and library "module augmentation" (Ch. 11/12) fundamentally rely on interfaces being mergeable — this is the single biggest structural reason interfaces still matter even though type aliases can express almost everything else. Error messages also differ slightly: interface-based type errors tend to show the interface's name; type-alias errors sometimes expand the full structural shape inline (can be more or less readable depending on complexity).

**Type inference.** No difference in how either is *inferred against* — assignability rules (Ch. 3) treat an `interface`-typed value and a structurally identical `type`-typed value as fully interchangeable.

**Real-world use cases.**
- **Interfaces:** public library API surfaces meant to be extended/augmented by consumers, class contracts (`implements`), anywhere merge-based extensibility is a feature, not a bug.
- **Type aliases:** unions, mapped/conditional types, function types, tuples, anything that isn't a plain extensible object contract, utility-type-heavy code.

**Good practices.** A common, defensible senior convention: **use `interface` for public object-shaped contracts (especially anything a library consumer might extend), use `type` for everything else** (unions, computed/derived types, function signatures, tuples). Some teams simplify further to "always `type` unless you specifically need merging" — both are legitimate, just be consistent within a codebase.
**Bad practices.** Mixing conventions randomly within the same codebase without a stated rule — makes it harder to predict "can I augment this" at a glance.
**Common mistakes.** Trying to declare a union or a tuple with `interface` syntax (not possible — `interface Status { "a" | "b" }` isn't valid); trying to re-open/add to a `type` alias the way you can an `interface` (not possible, must use `&` intersection to compose a *new* type instead).
**Edge case.** Declaration merging isn't just "same name, adds properties" — it has real rules about property conflicts (must be identical types) and can merge an `interface` with a `namespace` or a `class` (Ch. 12 covers this fully) — interfaces are part of a broader merging system, not an isolated quirk.
**Performance.** For pure object shapes, no meaningful difference. Very large union/mapped/conditional `type` aliases can be more expensive to check than an equivalent flat `interface`, purely because of what they're expressing (a union/computed type), not because `type` itself is inherently slower.

### Interview Q&A — §5.1
**Junior:** Q: Give one concrete thing `interface` can do that `type` can't. A: Declaration merging — multiple `interface` declarations with the same name automatically combine into one; redeclaring a `type` alias with the same name is an error.
**Mid:** Q: Give one concrete thing `type` can do that `interface` can't. A: Alias a union, a tuple, a primitive, or a mapped/conditional type — `interface` can only describe object/callable/constructable shapes.
**Senior:** Q: Why do most public library type definitions (e.g., DOM types, Express's `Request`) use `interface` rather than `type` for their core objects? A: So consumers can **augment** them via declaration merging (module augmentation, Ch. 12) — e.g., adding custom properties to `Express.Request` or `Window` in your own app without modifying the library's source — a capability that's structurally impossible with `type` aliases, which have no merging mechanism at all.
**FAANG/tricky:** Q: If interfaces and type aliases are "mostly interchangeable" for object shapes, why does this distinction matter enough to be a recurring interview topic? A: Because the difference isn't really about expressive power for simple cases — it's about **extensibility as a design decision**. Choosing `interface` for a public type is an implicit promise "consumers may merge into this"; choosing `type` is an implicit promise "this is closed, compose around it with `&` instead." Getting this wrong in a public API either accidentally invites fragile external augmentation or accidentally blocks a legitimate, commonly-needed extension pattern.

**Predict/debug**
```ts
interface Config { timeout: number; }
interface Config { retries: number; } // merges! not an error
const c: Config = { timeout: 100, retries: 3 }; // must satisfy BOTH declarations combined

type Config2 = { timeout: number; };
type Config2 = { retries: number; }; // Error: Duplicate identifier 'Config2'
```

---

## 5.2 Declaration Merging (deep dive)

**Intuition.** Declaration merging is the compiler's way of letting several separate declarations — possibly in different files — combine into a single logical entity, as if you'd written them together. It's the mechanism that makes ambient global typing and library augmentation possible at all.

**Technical explanation — what can merge:**
- **Interface + interface** → combined members (shown above).
- **Namespace + namespace** → combined exported members.
- **Namespace + class** → the namespace's exports become static-like members alongside the class (a pattern historically used before ES module statics were idiomatic).
- **Namespace + function** → adds properties to a function (models real JS patterns like `myFunc.defaultOptions = {...}`).
- **Namespace + enum** → adds extra static-like members to an enum.
- **Interface + class is NOT direct merging** — but a class's *instance shape* can be structurally described/extended by a same-named interface in some ambient scenarios (an edge case, not a primary pattern).

**Internal behavior.** The binder, while processing the whole program's files, collects every declaration sharing a name+kind into one Symbol with multiple "declaration" entries; the checker then computes the *effective* type/members by combining all of them. This is why merge order across files doesn't matter (unlike JS's runtime execution order) — it's a purely declarative combination.

**Compiler behavior / conflict rules.** Merged interface members with the same property name must have **identical types** (not just compatible/assignable — identical) or it's an error. Call/construct signatures instead accumulate as overloads (later-declared, in general, take priority for overload resolution ordering, mirroring §4.3's first-match rule, though exact ordering across merged declarations has version-specific nuance worth verifying against current docs for anything load-bearing).

**Real-world use cases.** Extending global types (`declare global { interface Window { myGlobal: string } }`), augmenting a third-party library's types (`declare module "express" { interface Request { user?: User } }` — the standard way to add `req.user` typing for auth middleware), plugin systems.

**Good practices.** Use declaration merging deliberately and sparingly for genuine augmentation needs (library extension points), not as a general-purpose "add more stuff later" habit within your own first-party code (there, just edit the original interface).
**Bad practices.** Relying on merge-based global augmentation for values that should really be passed explicitly (implicit global state hiding as "just extend the Window type" is a code smell, not just a typing pattern).
**Common mistakes.** Declaring a merged interface member with a slightly different (not identical) type than an existing declaration and being confused by the resulting error — remember: identical types required for property merges.
**Edge case.** Module augmentation (`declare module "some-lib" {...}`) only works correctly if it's structured to actually merge with the *same* underlying Symbol the library itself created — getting the module path/import structure subtly wrong creates a *new*, unrelated global-scope declaration instead of actually augmenting the library (a real, hard-to-diagnose gotcha covered fully in Ch. 12).
**Performance.** Merged interfaces with very large combined member counts hit the same structural-comparison cost profile as any large interface (Ch. 3) — no special extra cost from the merging mechanism itself.

### Interview Q&A — §5.2
**Junior:** Q: What is declaration merging? A: TS combining multiple declarations that share the same name (interfaces, namespaces, certain namespace+function/class/enum pairs) into one logical entity with unified members.
**Mid:** Q: What happens if two merged interfaces declare the same property with two different types? A: A compile error — merged property types must be identical, not just assignable/compatible.
**Senior:** Q: Walk through, mechanically, how adding `req.user` typing to Express's `Request` via module augmentation actually works. A: You write `declare module "express" { interface Request { user?: MyUser; } }` in an ambient `.d.ts` file included in your program; because Express's own types declare `Request` as an `interface` inside that same module's declared shape, your declaration merges with theirs at the binder level — every place `Request` is used throughout your program (including inside Express's own middleware types) sees the combined shape, without ever touching Express's source.

---

## 5.3 Extension (`extends`) vs Intersection (`&`)

**Intuition.** Both combine shapes, but `extends` is a **declared, checked subtyping relationship** ("I am explicitly building on top of X, and the compiler verifies compatibility as I do"), while `&` is a **computed combination** ("give me a new type that is simultaneously all of these").

**Technical explanation.**
```ts
interface Animal { name: string; }
interface Dog extends Animal { breed: string; }              // extension

type AnimalT = { name: string };
type DogT = AnimalT & { breed: string };                       // intersection
```
For this simple case, `Dog` and `DogT` end up structurally identical. Differences emerge with conflicting members: `extends` on an interface with an incompatible override is a **compile-time error at the declaration site** (you're told immediately that your extension is invalid); an intersection of two conflicting types instead **computes** the combination, and if a property's types are incompatible primitives, the result silently becomes `never` for that property (no immediate error at the type's declaration — the surprise shows up later, when you try to actually use/construct a value of that type and can't).

**Internal behavior.** `extends` triggers the checker to explicitly verify that each new/overridden member in the child interface is assignable-compatible with the corresponding member in the parent (or that a totally new member doesn't conflict) — a proactive check. `&` is lazier: it just computes a new type whose members are the union of both types' member names, with each member's type being the **intersection** of the corresponding types from each side — including intersecting to `never` silently if they're fundamentally incompatible (`string & number` → `never`).

**Compiler behavior / inference.**
```ts
interface A { x: string; }
interface B extends A { x: number; } // Error immediately: incompatible override

type AT = { x: string };
type BT = AT & { x: number };        // No error here — BT.x has type `string & number` = never
declare const b: BT;
b.x; // type: never — usable nowhere, but the error was deferred, not upfront
```

**Real-world use cases.** `extends` for genuine "is-a" hierarchies and public interface contracts meant to be layered predictably; `&` for ad hoc composition of independent trait-like types (mixing in capabilities: `type Timestamped = { createdAt: Date }; type Loggable = { log(): void }; type Entity = BaseEntity & Timestamped & Loggable`).

**Good practices.** Prefer `extends` when you want the compiler to *immediately* flag incompatible layering as you write it — this is a real, meaningful safety advantage over intersections for hierarchical designs. Reserve `&` for combining genuinely independent, non-conflicting trait types.
**Bad practices.** Building deep hierarchies purely with `&` chains where conflicting members silently degrade to `never` instead of erroring — a real, non-obvious footgun in large composed types.
**Common mistakes.** Not noticing an intersection has produced a `never` member until much later, then being confused why "nothing can satisfy this type."
**Edge case.** Interfaces can `extends` multiple interfaces at once (`interface C extends A, B {}`), and — unlike `&` — the checker verifies compatibility across all of them upfront at declaration time; multiple inheritance-style extension is fully supported for interfaces (TS has no single-inheritance restriction for interfaces the way classes have for `extends`, though classes can `implements` multiple interfaces).
**Performance.** Intersections of many large types can be genuinely expensive to fully resolve/display (the checker sometimes needs to flatten/normalize the combination), and IDE tooltips showing a deeply intersected type can become unreadable "type soup" — a real, common senior complaint that's a strong argument for extracting named intermediate `type`/`interface` layers instead of nesting raw `&` chains.

### Interview Q&A — §5.3
**Junior:** Q: What's the basic difference between `interface B extends A` and `type B = A & {...}`? A: `extends` is a declared relationship the compiler checks upfront for compatibility; `&` computes a combined type after the fact, which can silently produce `never` for conflicting members instead of erroring immediately.
**Mid:** Q: What happens if you intersect two types with a conflicting property of incompatible primitive types? A: The property's resulting type becomes `never` (the intersection of two incompatible types) — no immediate error, but the overall type becomes impossible to construct validly for that member.
**Senior:** Q: Why might a senior engineer deliberately choose `extends` over `&` even when both would type-check the "happy path" identically? A: `extends` gives fail-fast feedback exactly where an incompatible override is introduced, which is much more useful in a large, evolving codebase than discovering a silent `never` collapse deep inside a composed type much later, often far from the actual point where the conflicting declarations were combined — better locality of error for maintainability.

**Debugging exercise**
```ts
type WithId = { id: string };
type WithNumericId = { id: number };
type Broken = WithId & WithNumericId;
const x: Broken = { id: "abc" }; // Error — id: never, nothing satisfies it
```
→ Diagnose: intersection of incompatible `id` types collapsed to `never`; fix by resolving which `id` type is actually correct, or renaming one before combining.

---

## 5.4 Unions

**Intuition.** A union (`A | B`) says "this value is *one of* these possibilities, and I don't yet know which" — the type-level equivalent of "it could be this or that."

**Technical explanation.** `type Result = Success | Failure;` — a value of type `Result` is guaranteed to be assignable to *at least one* member, and only operations valid on **every** member are allowed without narrowing first (the intersection of available operations, not the union — a very common early-learner confusion).

**Internal behavior.** Structurally, TS represents a union as a genuine set of constituent types; accessing a member on a union value requires the checker to verify that member exists (compatibly) on *all* constituents, or you must narrow first (Ch. 2.9) to a specific constituent (or subset) before accessing member-specific properties. This is precisely why discriminated unions (a shared literal "tag" property present, differently valued, on every member) are so powerful — the tag is always safely accessible on the full union, and narrowing on it precisely selects one constituent.

**Compiler behavior / inference.** TS automatically **widens to a union** when a value could plausibly be more than one type at inference time (mixed array literals from Ch. 2, ternary expressions with different branch types, function return type inference across multiple `return` statements with different types).

```ts
function f(x: boolean) {
  return x ? "yes" : 1; // inferred return type: string | number
}
```

**Real-world use cases.** API response modeling (`Success | Error`), state machines, Redux action unions, anywhere "one of several known variants" describes the domain honestly (see Ch. 2.9's exhaustiveness pattern — the primary payoff of well-modeled unions).

**Good practices.** Prefer modeling real domain alternatives as unions (ideally discriminated) instead of one big object with many optional fields (Ch. 2.9) — unions make invalid states genuinely unrepresentable, which is one of the highest-leverage type-safety techniques in the whole language.
**Bad practices.** Using `any`/overly wide types to avoid dealing with a union's need for narrowing — throws away the actual safety the union was meant to provide.
**Common mistakes.** Trying to access a member that only exists on *some* union constituents without narrowing first — a very common, very early TS error message (`Property 'x' does not exist on type 'A | B'`).
**Edge case.** Unions **distribute** through conditional types in a special way (`T extends U ? X : Y` applied to a union `T` checks each member separately, Ch. 8) — a deep, frequently-tested advanced-type behavior that only makes sense once union fundamentals are solid.
**Performance.** Very large unions (hundreds/thousands of literal members, especially from generated types or template literal type combinations, Ch. 9) can meaningfully slow the checker — assignability against a huge union effectively needs to check against many members.

### Interview Q&A — §5.4
**Junior:** Q: If `x: string | number`, can you call `x.toUpperCase()` directly? A: No — that method only exists on `string`, not `number`; you must narrow `x` (e.g. `typeof x === "string"`) first.
**Mid:** Q: Why does TS only allow operations valid on *every* member of a union without narrowing, rather than *any* member? A: Because the actual runtime value could be any one of the constituents — allowing an operation that's only valid on some members would be unsound, since you can't statically guarantee which member you actually have without narrowing.
**Senior:** Q: Why is "make invalid states unrepresentable" via discriminated unions considered a higher-leverage practice than validating invariants at runtime with lots of `if` checks? A: A well-designed discriminated union structurally prevents the invalid combination from ever being constructed in the first place — there's no code path, no test case, no runtime check needed for "what if status is success but error is also set," because that shape simply cannot be built and passed the type checker. Runtime validation only catches invalid states *after* the fact, at each individual check site, and can be forgotten in some code paths; the type system enforces it everywhere, always, for free.

**Predict the inferred type**
```ts
const val = Math.random() > 0.5 ? { kind: "a" as const, x: 1 } : { kind: "b" as const, y: 2 };
// val: { kind: "a"; x: number } | { kind: "b"; y: number }  — a clean discriminated union
```

---

## 5.5 Intersections (usage patterns beyond §5.3)

**Intuition.** Beyond hierarchy-building, intersections are the primary tool for **mixing in** independent capabilities/traits onto a base type without inheritance.

**Technical explanation.**
```ts
type Timestamped = { createdAt: Date; updatedAt: Date };
type SoftDeletable = { deletedAt: Date | null };
type Entity<T> = T & Timestamped & SoftDeletable;

type User = Entity<{ id: string; name: string }>;
// User: { id: string; name: string } & Timestamped & SoftDeletable, flattened structurally
```

**Real-world use cases.** ORMs/entity modeling (mixing timestamp/audit fields into domain entities), enhancing third-party prop types in React (`type EnhancedProps = LibraryProps & { onCustomEvent: () => void }`), combining multiple trait-like capability types for mixins (Ch. 12 covers the class-mixin runtime pattern this pairs with).

**Good practices.** Keep each "trait" type small, single-purpose, and independently named — intersections compose best when each side is conceptually orthogonal (no overlapping property names, ideally), keeping the §5.3 `never`-collapse risk low by construction.
**Bad practices.** Intersecting many overlapping, loosely-related types "because it compiles" without checking for silent conflicts.
**Common mistakes.** Forgetting function-type intersections behave like **overload sets**, not merged single signatures — `type F = ((x: string) => void) & ((x: number) => void)` creates a callable usable with either a `string` or a `number` argument (like two overloads), not a function requiring both simultaneously — a genuinely non-obvious, interview-relevant behavior.
**Edge case.** The function-intersection-as-overloads behavior above is exactly how some advanced typing patterns (certain generic utility signatures, callable+constructable hybrid types) are built — worth recognizing when you see it in library type definitions.
**Performance.** Same considerations as §5.3 — deep/wide intersections can produce hard-to-read, costly-to-normalize composite types; prefer named intermediate aliases.

### Interview Q&A — §5.5
**Junior:** Q: What's a practical use for `&` beyond building an object hierarchy? A: Mixing independent trait-like types together (e.g., adding audit fields like `createdAt`/`updatedAt` onto multiple unrelated entity types) without needing inheritance.
**Senior/FAANG:** Q: What does `((x: string) => void) & ((x: number) => void)` actually mean as a type, and how is that different from what you might intuitively guess? A: It's *not* "a function that needs both a string and a number simultaneously" — intersecting function types produces something callable in **either** way, behaving like an overloaded function accepting a `string` OR a `number` (whichever matches at the call site), because a value satisfying an intersection of callable types must satisfy each call signature independently, and a real function value can indeed be called either way; this is a common surprise since object-type intersections behave completely differently (merged single shape) from function-type intersections (overload-like alternation).

---

## Practical Exercises — Chapter 5

1. **Convention exercise:** Take a small existing module and rewrite every `type` alias that's a plain object shape as an `interface`, and vice versa where it isn't a plain shape — write down, for each, whether the change gained or lost any real capability.
2. **Augmentation drill:** Practice module augmentation by adding a custom property to a third-party (or hypothetical) library's exported interface via `declare module`, and verify it's visible everywhere that type is used in your code.
3. **Extends vs intersect trap:** Deliberately construct a `&`-based type with a silent `never`-collapsed member, find it via a confusing downstream error, then refactor it to use `extends` so the same conflict is caught immediately at declaration time instead.
4. **Union modeling:** Take a real "all-optional-fields" state object from a project you know, and redesign it as a discriminated union with an exhaustiveness-checked handler.

---

## Chapter 5 Summary — Mental Model

```
              OBJECT / SHAPE DECLARATION
                        │
        ┌────────────────────────────────┐
   interface                          type alias
   (mergeable, extensible,           (single binding, can name
    checked `extends`)                anything: unions, tuples,
        │                             functions, computed types)
        │
   declaration merging
   (binder-level combination:
    interface+interface,
    namespace+{class,function,enum})


              COMBINING TYPES
                        │
        ┌────────────────────────────────┐
      extends (interfaces)             &  (intersection, any types)
   checked upfront, fails fast      computed after the fact,
   on incompatible overrides        can silently collapse to `never`
                                     on conflicting object members;
                                     ALTERNATES (overload-like) for
                                     function-type intersections


              ALTERNATIVES
                        │
                   |  (union)
        "one of several known shapes" —
        only common-to-all operations allowed
        without narrowing; pairs with
        discriminated-union + exhaustiveness
        checking (Ch. 2.9) for the highest-
        leverage domain-modeling pattern in TS
```

---

**Next:** Chapter 6 — Generics: generic functions/interfaces/classes, constraints, defaults, variadic tuples. Say **"next"** to continue.
