# Chapter 7 — Advanced Types I: `keyof`, `typeof`, Indexed Access, Mapped Types

This chapter is about **type-level introspection and transformation** — operators that let you *derive* new types from existing ones instead of writing everything by hand. This is the toolkit that makes utility types (Ch. 10) possible.

## 7.1 `keyof`

**Intuition.** `keyof T` asks the type system "what are the valid property names of `T`?" and gives you back a union of those names as string/number/symbol literal types.

**Technical explanation.**
```ts
interface User { id: string; age: number; }
type UserKeys = keyof User; // "id" | "age"
```

**Internal behavior.** The checker computes `keyof T` by enumerating `T`'s known member names (including inherited ones, and numeric index signature keys as `number`, string index signatures as `string`) and producing the literal-type union. This is a genuine type-level *computation*, not a simple lookup — it's recomputed whenever `T` changes (e.g., `keyof T` inside a generic stays abstract/unresolved until `T` is substituted with something concrete).

**Compiler behavior / inference.** `keyof any` is `string | number | symbol` (all possible property key types); `keyof {}` is `never` (no keys). Index signatures affect `keyof`: `keyof { [key: string]: unknown }` is `string | number` (JS quirk: numeric keys are coerced to strings, so a string index signature also permits numeric access — TS models this by including both).

**Real-world use cases.** Generic property accessors (`pluck` from Ch. 6.4), building `Record<keyof T, ...>`-shaped derived types, exhaustively iterating/validating known property names at the type level (e.g., ensuring a validation-rules object covers every field of a form type).

**Good practices.** Use `keyof typeof someConstObject` (see §7.2) to derive a literal union from a real runtime object instead of hand-writing a parallel union — a single source of truth, same principle as the `as const` pattern from Ch. 2.7.
**Bad practices.** Hand-duplicating a union of property names that could instead be derived via `keyof` — invites drift when the object type changes and the union isn't updated.
**Common mistakes.** Forgetting `keyof` on a generic `T` (unconstrained) is close to useless until `T` is constrained/substituted — `function f<T>(key: keyof T)` is fine and correctly generic, but assuming you can do much with `keyof T` *inside* the function body beyond basic indexed access requires care (Ch 6.4's `pluck` pattern).
**Edge case.** `keyof` on a union type produces the **intersection** of each member's keys, not the union of all keys — a frequently-tested surprise: `keyof (A | B)` = `keyof A & keyof B`, because only keys guaranteed to exist on *every* possible member are safe to access without narrowing (consistent with Ch. 5.4's union member-access rule).
**Performance.** Cheap for typical interfaces; can be a real cost when applied to enormous generated/derived types (e.g., `keyof` of a huge auto-generated API schema type) combined repeatedly in mapped/conditional types.

### Interview Q&A — §7.1
**Junior:** Q: What does `keyof User` produce for `interface User { id: string; age: number }`? A: The union of literal types `"id" | "age"`.
**Mid:** Q: What is `keyof (A | B)` in terms of `keyof A` and `keyof B`? A: The **intersection** `keyof A & keyof B` — only property names guaranteed to exist on every member of the union, consistent with the rule that unions only expose operations/members common to all constituents.
**Senior:** Q: Why does `keyof (A | B)` use intersection rather than union, and how does this connect to safe member access on unions generally? A: A value typed `A | B` could actually be either at runtime, so only a key present on *both* is guaranteed safe to access without first narrowing to a specific member — using the union of keys instead would let you reference a key that doesn't exist on whichever variant you actually have, which would be unsound; this mirrors exactly why unions only permit common-to-all member access (Ch. 5.4).

**Predict the inferred type**
```ts
interface A { x: number; y: number; }
interface B { y: number; z: number; }
type K = keyof (A | B); // "y" — only the common key
```

---

## 7.2 `typeof` (type-level operator, not the JS runtime operator)

**Intuition.** `typeof` in a *type position* asks "what type did the checker already infer for this existing value?" — it lets you derive a type from a value instead of writing the type by hand, keeping the two permanently in sync.

**Technical explanation.**
```ts
const config = { retries: 3, timeout: 1000 };
type Config = typeof config; // { retries: number; timeout: number }

function add(a: number, b: number) { return a + b; }
type AddFn = typeof add; // (a: number, b: number) => number
```
Note: this is a **compile-time-only** construct, completely distinct from JS's runtime `typeof` operator (which returns a string like `"number"` at runtime) — same keyword, different world, disambiguated purely by syntactic position (type position vs expression position). A very common early-career confusion worth stating explicitly.

**Internal behavior.** `typeof x` in a type position simply asks the checker for the type it already computed for `x` at that declaration — no new computation, just a reference to existing inference results. This is why it composes so well with `as const` (Ch. 2.7) and `keyof` — `keyof typeof config` derives a literal-union type of `config`'s own actual property names, directly from the runtime object.

**Compiler behavior / inference.** `typeof` can reference variables, functions, classes (`typeof SomeClass` gives you the *constructor* type, not the instance type — another common gotcha, covered further in Ch. 10's `InstanceType`), and namespaces.

**Real-world use cases.** Deriving types from configuration objects, deriving action-type unions from Redux action-creator maps, deriving prop types from default-props objects, avoiding maintaining a type and a value in parallel (the single-source-of-truth principle recurring throughout this course).

**Good practices.** Whenever you notice you're maintaining a type and a runtime value that describe "the same thing" in parallel, ask whether `typeof` (possibly combined with `as const`) can derive one from the other instead.
**Bad practices.** Writing `typeof someMutableLetVariable` and expecting a narrow/precise type — remember Ch. 2.8's widening rules apply first; `typeof` reflects whatever type was already inferred/widened, it doesn't retroactively narrow anything.
**Common mistakes.** Confusing `typeof SomeClass` (the constructor/static side type) with `SomeClass` used as a type (the instance side) — `typeof SomeClass` is what you'd use to type a variable meant to *hold the class itself* (e.g., a factory parameter `type Factory = new () => SomeClass` vs `typeof SomeClass`).
**Edge case.** `typeof` combined with `ReturnType`/`Parameters` (Ch. 10) on functions whose types you don't want to hand-duplicate is an extremely common real-world pattern for wrapping/adapting third-party functions.
**Performance.** Negligible — a direct reference to already-computed type information, not new structural work.

### Interview Q&A — §7.2
**Junior:** Q: Is `typeof` in `type Config = typeof config` the same as JS's runtime `typeof` operator? A: No — same keyword, but in a type position it asks the compiler for the statically inferred type of an existing value/declaration; JS's runtime `typeof` returns a string describing a value's runtime category. They're disambiguated by syntactic position.
**Mid:** Q: What's the difference between `typeof SomeClass` and using `SomeClass` directly as a type? A: `SomeClass` as a type refers to the *instance* shape (what `new SomeClass()` produces); `typeof SomeClass` refers to the *constructor/static* side — the type of the class value itself, useful when a parameter needs to accept the class/constructor, not an instance of it.
**Senior:** Q: What's the value of the `keyof typeof someObject` pattern over hand-writing a matching union type? A: It derives the literal-key union directly from a real runtime object's actual keys, so the type can never silently drift out of sync with the object as it evolves — adding/removing/renaming a key in the object automatically updates every derived type, whereas a hand-written parallel union requires manual maintenance and is a real, common source of type/runtime mismatch bugs.

**Predict the inferred type**
```ts
const routes = { home: "/", about: "/about", contact: "/contact" } as const;
type Route = typeof routes[keyof typeof routes]; // "/" | "/about" | "/contact"
```

---

## 7.3 The `in` Operator (in mapped-type position)

**Intuition.** In a mapped type, `in` means "for each key in this set, produce a corresponding property" — it's a type-level `for...in` loop over a union of keys, generating one property per iteration.

**Technical explanation.**
```ts
type Flags = { [K in "read" | "write" | "execute"]: boolean };
// { read: boolean; write: boolean; execute: boolean }
```
Distinct from the *value-level* `in` operator used for narrowing (`"prop" in obj`, Ch. 2.9) — same keyword, two different jobs (one iterates keys to build a type, the other checks membership to narrow a value), again disambiguated by position/context.

**Internal behavior.** The checker treats `[K in U]` as: for each literal type `K` that's a member of union `U`, synthesize one property named `K` with whatever type expression follows the colon (which can itself reference `K`, enabling per-key-dependent value types — the heart of mapped types, §7.5).

**Real-world use cases.** The mechanism underlying essentially every "transform all properties of T" utility type (`Partial`, `Required`, `Readonly`, `Pick`, `Record` — all built with `in`-based mapped types under the hood, Ch. 10).

**Common mistakes.** Confusing narrowing `in` (`"key" in obj`, a runtime/value check usable for narrowing) with mapped-type `in` (`[K in U]`, a compile-time-only key-iteration construct) — they share a keyword but are unrelated mechanisms in unrelated positions.

### Interview Q&A — §7.3
**Junior:** Q: What does `{ [K in "a" | "b"]: number }` produce as a type? A: `{ a: number; b: number }` — one property per member of the key union.
**Mid:** Q: Is the `in` used in `{ [K in keyof T]: ... }` the same mechanism as the `in` used in `if ("key" in obj)`? A: No — they share a keyword but serve entirely different purposes: one is a compile-time mapped-type key iterator (builds a new object type), the other is a runtime property-existence check usable for narrowing a value's type within a conditional branch.

---

## 7.4 Indexed Access Types

**Intuition.** Just as `obj["key"]` retrieves a *value* at runtime, `T["key"]` retrieves a *type* at compile time — "give me the type of this specific property of this type."

**Technical explanation.**
```ts
interface User { id: string; address: { city: string; zip: string }; }
type UserId = User["id"];           // string
type City = User["address"]["city"]; // string — chains, just like real property access
type AllValues = User[keyof User];   // string | { city: string; zip: string } — every property's type, unioned
```

**Internal behavior.** `T[K]` is computed by looking up the member named (by) `K` in `T`'s member list and returning its type; when `K` is a union of literal keys (as in `T[keyof T]`), the checker computes the indexed access for *each* member of the union and returns the union of results — this is a **distributive**-flavored computation conceptually related to (but distinct from) the distributive conditional types covered in Ch. 8.3.

**Compiler behavior / inference.** Indexed access also works on arrays/tuples: `type ElementType = SomeArray[number]` (the pattern from Ch. 2.7's `typeof arr[number]`) — using the special `number` index type to get "the type of any element," and on tuples specifically, `SomeTuple[0]` gets the precise type at that exact position.

**Real-world use cases.** Extracting a nested property's type without re-declaring it (avoids duplication/drift, same "single source of truth" theme as `typeof`), deriving element types from array/tuple constants, building generic accessor return types (`T[K]` in the `pluck` pattern from Ch. 6.4).

**Good practices.** Prefer `T["propName"]` / `T[K]` over manually re-typing a nested shape when the source type already exists — same drift-avoidance principle as `typeof`.
**Bad practices.** Deeply chaining indexed access across many levels (`T["a"]["b"]["c"]["d"]`) without intermediate named aliases — hurts readability and error message clarity; consider extracting intermediate type aliases for deep chains.
**Common mistakes.** Trying `T[SomeNonExistentKey]` — a compile error (the key must actually exist on `T`, or be constrained via a generic `K extends keyof T` as in Ch. 6.4) — indexed access is not forgiving of typos, which is exactly its safety value.
**Edge case.** Indexed access on a union of object types with the same key name computes to the union of that key's type across all members — consistent with how `keyof` on unions and general union member access behave (Ch. 5.4/7.1) — internal consistency worth recognizing as one coherent design, not several unrelated rules.
**Performance.** Cheap for direct lookups; chained/distributed indexed access over large unions can compound cost similarly to other union-heavy operations.

### Interview Q&A — §7.4
**Junior:** Q: What does `User["id"]` give you as a type, given `interface User { id: string }`? A: `string` — the type of the `id` property.
**Mid:** Q: What does `SomeArray[number]` mean as a type expression, and what's it commonly used for? A: It retrieves "the type of any element" of an array/tuple type (using the special `number` index type rather than a specific literal index) — commonly combined with `as const` on a runtime array to derive a literal union type from its actual elements (Ch. 2.7's pattern).
**Senior:** Q: How does `T[keyof T]` behave when `T` has properties of different types, and why? A: It computes the type at each key in `keyof T` and unions all the results together — giving you "the type of any property value on `T`," which is genuinely useful for building generic value-extraction utilities, and is internally consistent with how indexed access distributes over key unions generally.

---

## 7.5 Mapped Types

**Intuition.** A mapped type takes an existing type's keys and produces a *new* object type by applying some transformation to each property — "for every key in T, do X to its value type" — this is the generalized, reusable version of the ad hoc `in`-based examples in §7.3, and it's how virtually every built-in utility type (Ch. 10) is actually implemented.

**Technical explanation.**
```ts
type MyPartial<T> = { [K in keyof T]?: T[K] };
type MyReadonly<T> = { readonly [K in keyof T]: T[K] };
type MyRecord<K extends string | number | symbol, V> = { [P in K]: V };
```
Modifiers `?` and `readonly` can be added (as above) or explicitly **removed** with a `-` prefix:
```ts
type MyRequired<T> = { [K in keyof T]-?: T[K] };   // strips optionality
type MyMutable<T> = { -readonly [K in keyof T]: T[K] }; // strips readonly
```

**Internal behavior.** The checker iterates `keyof T` (or whatever key-union expression is given), and for each key `K`, synthesizes a property using the mapped type's value expression (which typically references `T[K]`, the original property's type, possibly further transformed) and applies any modifier changes — a genuine type-level transformation pipeline, computed once per instantiation with a concrete `T`.

**Key remapping (`as` clause, modern TS).**
```ts
type Getters<T> = { [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K] };
interface Person { name: string; age: number; }
type PersonGetters = Getters<Person>; // { getName: () => string; getAge: () => number }
```
The `as` clause lets you **rename** keys during mapping (here combined with template literal types, Ch. 9) — a powerful, relatively recent (TS 4.1+) capability that dramatically expands what mapped types alone can express, without needing separate conditional-type gymnastics for key renaming.

**Compiler behavior / inference.** Mapped types **preserve** the original modifiers by default unless you explicitly add/remove them — `{ [K in keyof T]: T[K] }` (the "homomorphic" identity mapped type) actually preserves `T`'s own `readonly`/`?` modifiers per-property, rather than resetting them; this "homomorphic mapped type" behavior is specifically why `Partial<T>`/`Readonly<T>` correctly interact with already-partial/already-readonly source types instead of clobbering existing modifiers unexpectedly.

**Real-world use cases.** Every built-in utility type (Ch. 10), DTO transformation types (turning a full entity type into an "update payload" type with all-optional fields), generating getter/setter method-map types from a data shape, deep-transformation utilities (`DeepReadonly<T>`, `DeepPartial<T>` — recursive mapped types, previewed in Ch. 8.5).

**Good practices.** Learn to read/write the four canonical building blocks (`Partial`, `Required`, `Readonly`, `Record`-style) from scratch — they cover the large majority of real-world mapped-type needs, and understanding them by hand makes reading unfamiliar library-defined mapped types far less intimidating.
**Bad practices.** Reaching for a hand-rolled mapped type when a built-in utility type (Ch. 10) already does exactly what's needed — reinvents a well-tested wheel and hurts readability for other engineers who recognize the standard names.
**Common mistakes.** Forgetting the homomorphic-preservation behavior and being surprised that `{ [K in keyof T]: T[K] }` isn't just a no-op-looking identity but actually *preserves* per-property modifiers precisely because it's written in this specific homomorphic form (`keyof T` directly, not some derived/renamed key set) — non-homomorphic mapped types (e.g., ones using a fixed key union unrelated to an actual `keyof T`) don't get this modifier-preservation behavior.
**Edge case.** Combining an `as` key-remapping clause with a conditional expression inside it lets you conditionally **drop** keys entirely from the resulting type (mapping a key to `never` in the `as` clause causes TS to omit that property altogether) — a genuinely elegant pattern used to build custom `Omit`-like or filter-by-value-type utilities.
**Performance.** Mapped types over very large object types (hundreds of properties, or deeply recursive mapped types applied to nested structures) are one of the more common real-world compile-time cost centers in large codebases with auto-generated API types — a legitimate, frequently-cited performance concern at senior/staff level.

### Interview Q&A — §7.5
**Junior:** Q: What does `{ [K in keyof T]?: T[K] }` do? A: Produces a new type with the same keys as `T`, but every property made optional — this is essentially how the built-in `Partial<T>` utility type works.
**Mid:** Q: How do you remove `readonly` from every property of a type using a mapped type? A: `{ -readonly [K in keyof T]: T[K] }` — the `-` prefix explicitly strips the modifier, as opposed to omitting it (which would just leave existing modifiers as-is due to homomorphic preservation).
**Senior:** Q: What does "homomorphic mapped type" mean, and why does it matter practically? A: A mapped type is homomorphic when it maps directly `over keyof T` (preserving T's structure/shape one-to-one), which causes TS to automatically carry over each property's existing modifiers (`readonly`, `?`) unless explicitly overridden — this is why `Partial<Readonly<T>>` behaves sensibly (still readonly, now also optional) instead of the mapping silently resetting modifiers; non-homomorphic mapped types (with a key set not directly derived from a single `keyof T`) don't get this automatic preservation and must handle modifiers explicitly.
**FAANG/tricky:** Q: How can a mapped type with an `as` clause be used to *remove* keys conditionally, not just rename them? A: By mapping a key to `never` inside the `as` clause conditionally (e.g., `[K in keyof T as T[K] extends Function ? never : K]: T[K]`) — TS omits any property whose remapped key evaluates to `never` entirely from the resulting type, effectively implementing a type-level filter (e.g., "keep only non-function properties") without needing a separate `Omit`/`Pick` pass.

**Predict the inferred type / coding challenge**
```ts
type NonFunctionKeys<T> = { [K in keyof T as T[K] extends (...args: any[]) => any ? never : K]: T[K] };
interface Service { name: string; start(): void; stop(): void; }
type Data = NonFunctionKeys<Service>; // { name: string } — methods filtered out entirely
```

---

## Practical Exercises — Chapter 7

1. **From scratch:** Implement `MyPick<T, K extends keyof T>`, `MyOmit<T, K extends keyof T>`, `MyRecord<K extends string | number | symbol, V>` using only `keyof`, indexed access, and mapped types — no peeking at Ch. 10 first.
2. **Single source of truth drill:** Take a config object literal in a real project, and derive its type via `typeof`, then derive a literal union of its keys via `keyof typeof`, replacing any hand-written parallel type/union you find.
3. **Key remapping challenge:** Write a `Getters<T>` mapped type (as shown in §7.5) and a companion `Setters<T>` type producing `set${Capitalize<K>}(value: T[K]): void` signatures.
4. **Filter-by-value-type:** Write a mapped type that keeps only the properties of `T` whose value type extends `string`, dropping all others, using the `as ... ? never : K` pattern.
5. **Debug:** Given `type X = SomeUnionOfObjects[keyof SomeUnionOfObjects]`, predict the resulting type for a specific 2-3-member union you construct, then verify.

---

**Next:** Chapter 8 — Advanced Types II: conditional types, `infer`, distributive conditional types, recursive types. Say **"next"** to continue.
