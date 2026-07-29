# Chapter 2 — Types: Primitives to Narrowing

## 2.1 Primitive Types

**Intuition.** These mirror JS's own primitive values exactly — TS just gives each a name you can write down: `string`, `number`, `boolean`, `bigint`, `symbol`, `undefined`, `null`, plus the special `void` and `never`.

**Technical explanation.**
- `string`, `number`, `boolean`, `bigint`, `symbol` — correspond 1:1 to `typeof` results.
- `undefined`, `null` — both a value *and* a type of that single value. Under `strictNullChecks`, they're not part of every other type by default.
- `void` — the *absence of a meaningful return value* (function return position only, semantically). Not a real JS value — it's a TS-only convention meaning "I return undefined but don't rely on it."
- `never` — the type of a value that **cannot exist**: a function that always throws, an infinite loop, or the empty branch after exhaustive narrowing.
- `object` (lowercase) — any non-primitive (excludes `string`/`number`/`boolean`/etc., includes arrays/functions/objects). Rarely what you want — usually `Record<string, unknown>` or a specific shape is more useful.
- `unknown` — the type-safe counterpart to `any`: anything is assignable *to* `unknown`, but `unknown` is assignable to *nothing* until narrowed.
- `any` — opts a value **out of the type system entirely**; bidirectionally assignable to/from everything, propagates virally through anything it touches.

**Internal behavior.** The checker represents primitives as intrinsic types (singletons internally — there's one canonical `string` type object the checker reuses everywhere, not one per occurrence). `never` is the **bottom type** (subtype of everything, assignable to anything, nothing is assignable to it except `never` itself) and `unknown` is the **top type** (supertype of everything). `any` breaks the lattice entirely — it's simultaneously top and bottom, which is exactly why it's unsound: assigning `any` to `string` or `string` to `any` are both allowed, unlike proper subtyping.

**Compiler behavior.** `void` in a function return position permits a function returning `undefined` OR "whose return value should be ignored" (a `void`-returning callback type can actually accept a function that returns something else — a deliberate looseness for callback ergonomics: `arr.forEach(x => x.push(1))` — `push` returns a number, `forEach` wants a `void`-returning callback, TS allows it because it discards the value).

**Type inference.**
```ts
let a = 5;              // number (widened, see §2.8)
const b = 5;             // 5 (literal)
function f() { return; } // return type void
function g(): never { throw new Error(); } // must be explicit or inferred from always-throw analysis
```

**Real-world use cases.** `unknown` for values from untyped boundaries (JSON.parse, catch clauses, third-party libraries without types) that you must narrow before use — the disciplined alternative to `any`. `never` for exhaustiveness checks (§2.9) and for modeling functions that never return (process exit, assertion helpers).

**Good practices.** Default to `unknown` over `any` at any I/O boundary. Use `void` for callback parameter types when the return value is intentionally ignored.
**Bad practices.** Using `object` when you mean "a specific shape" — it only guarantees non-primitive, nothing about structure.
**Common mistakes.** Believing `void` means "returns nothing" strictly — it's "return value not guaranteed usable," which is subtly different (see forEach example above). Confusing `null`/`undefined` semantics (`undefined` = "never set"; `null` = "intentionally empty") and mixing them inconsistently across a codebase.
**Edge case.** `any` defeats `strictNullChecks` locally — `let x: any; x.foo.bar` compiles even though `x` could be `null`. This is one reason `any` is more dangerous than just "loose typing" — it silences *every* other check too.
**Performance.** Primitives are cheap for the checker (no structural comparison needed, just identity/subtype checks against intrinsics) — this is part of why "type-heavy but primitive-shaped" code checks fast, while deeply nested object/generic types are the actual cost driver.

### Interview Q&A — §2.1
**Junior:** Q: What's the difference between `unknown` and `any`? A: `any` disables type checking entirely; `unknown` is type-safe — you must narrow it (e.g. `typeof`, `instanceof`) before performing any operation on it.
**Mid:** Q: What is `never` used for? A: Representing values that can't occur — functions that always throw/loop forever, and the fallthrough case after exhaustively narrowing a union (used for exhaustiveness checks).
**Senior:** Q: Why is `any` described as "bidirectional" while `unknown` and `never` are not? A: `any` is assignable to and from virtually every type, breaking the normal subtype lattice (it acts as both top and bottom), which is precisely why it disables checking; `unknown` is only a supertype (top) — things flow into it, not out — and `never` is only a subtype (bottom) — it flows into everything, nothing flows into it.
**FAANG/tricky:** Q: Why does `Array.prototype.forEach`'s callback type accept a function returning `number`, even though it's typed to expect a function returning `void`? A: TS specifically allows a wider-return function where `void` is expected in "callback contravariant position" as an ergonomic exception (not a general soundness rule) — the value is discarded, so returning *something* is safe, but writing `(): void => 5` directly as a variable's own declared type would behave differently in stricter contexts; it's specifically about matching callback *parameter* positions.

**Predict the inferred type**
```ts
function fail(): never { throw new Error("x"); }
function log(msg: string): void { console.log(msg); }
const x = log("hi"); // x: void
let y: unknown = 5;
// y.toFixed(2); // error — must narrow first
```

---

## 2.2 Literal Types

**Intuition.** Just as `5` is a `number`, TS lets `5` *itself* be a type — the set containing exactly one value. Literal types are how TS models exact values, not just categories of values.

**Technical explanation.** String, number, boolean, and (with template literals, Ch. 9) even computed string patterns can be literal types: `"GET"`, `42`, `true`. A literal type is a subtype of its base type — `"GET"` is assignable to `string`, not vice versa.

**Internal behavior.** The checker tracks the *most specific* type at the point of declaration but **widens** it the moment the value could be reassigned (`let`) — literal types are only preserved where the value is provably immutable at that binding (`const`, or explicitly annotated). This is the mechanism behind `as const` (§2.7).

**Compiler behavior / inference.**
```ts
const method = "GET";        // type: "GET"
let method2 = "GET";         // type: string (widened — could later be reassigned to any string)
function send(method: "GET" | "POST") { /* ... */ }
send(method);   // OK — "GET" assignable to "GET" | "POST"
send(method2);  // Error — string not assignable to "GET" | "POST"
```

**Real-world use cases.** Discriminated unions (Ch. 5), string-literal-based config APIs (`"left" | "center" | "right"`), HTTP method typing, Redux-style action `type` fields.

**Good practices.** Prefer union-of-literals over `enum` for many use cases (Ch. 2.6 compares them) — plain string literals are zero-runtime-cost and interop trivially with plain strings/JSON.
**Bad practices.** Widely typing free-form string params as `string` when a known-finite set of literals would catch typos at compile time.
**Common mistakes.** Passing a `let`-declared variable where a literal type is expected and being surprised by a widening error — the fix is `as const` or an explicit literal type annotation on the `let`.
**Edge case.** Object literal properties widen individually unless the whole object is `as const` — `{ method: "GET" }` infers `{ method: string }`, not `{ method: "GET" }`, *even though* it was declared with `const` — because `const` only freezes the *binding*, not nested property mutability (§2.8 covers this precisely).
**Performance.** Large literal unions (hundreds of string literals) are still cheap to check membership against (set-like), but exploding literal unions via type-level products (e.g. all combinations of two 50-item unions = 2500 members) can become genuinely slow — a real senior-level gotcha (Ch. 9 template literals).

### Interview Q&A — §2.2
**Junior:** Q: What type does `const dir = "up"` have? A: The literal type `"up"`, not `string`.
**Mid:** Q: Why does `let dir = "up"` widen to `string` but `const dir = "up"` doesn't? A: `let` allows reassignment, so TS widens to the general type to permit any future string; `const` guarantees the binding never changes, so TS keeps the precise literal.
**Senior:** Q: Given `const cfg = { method: "GET" }`, why is `cfg.method` typed `string` rather than `"GET"`, and how would you fix it two different ways? A: `const` only prevents reassigning `cfg` itself, not mutating `cfg.method`, so TS conservatively widens nested literal properties to their base type. Fixes: `as const` on the whole object, or an explicit type annotation `{ method: "GET" as const }` / `{ method: "GET" } satisfies {method: "GET"}`.

**Predict the inferred type**
```ts
function pick<T extends string>(x: T): T { return x; }
const r = pick("hello"); // r: "hello" (generic inference preserves literal — different from plain assignment widening!)
```
*(This is a genuinely tricky one — generic type parameter inference from a literal argument keeps it literal, because `T` is inferred as the narrowest type that fits, unlike a bare `let`/`const` variable assignment context.)*

---

## 2.3 Object Types

**Intuition.** An object type is a *shape contract*: "anything with at least these properties, of these types, qualifies" — TS doesn't care about the object's class, name, or origin, only its shape (this is structural typing, formalized in Ch. 3).

**Technical explanation.**
```ts
type Point = { x: number; y: number };
interface Point2 { x: number; y: number; readonly id?: string; }
```
Modifiers: `?` (optional), `readonly` (no reassignment through this type's view — doesn't freeze the underlying object, just this typed access path).

**Internal behavior.** Object types are represented internally as a set of member Symbols (property name → type + flags for optional/readonly) plus (optionally) index signatures and call/construct signatures. Structural compatibility is computed by comparing member sets, not by identity or declared name (Ch. 3 goes deep here).

**Compiler behavior — excess property checks.** Object *literals* assigned directly get an extra check literals-only: any property not in the target type is flagged, even if structurally a superset would normally be fine.
```ts
function draw(p: Point) {}
draw({ x: 1, y: 2, z: 3 }); // Error: 'z' does not exist — excess property check
const obj = { x: 1, y: 2, z: 3 };
draw(obj); // OK! obj is not a literal at the call site, so only structural (superset) compatibility applies
```
This is one of the most-asked "gotcha" interview questions in TS.

**Type inference.** Object literal types are inferred member-by-member, each property's type independently widened (see §2.8), producing a "fresh" object type — freshness is exactly what triggers excess property checking; once assigned to a variable and reused, the type loses that "freshness" flag.

**Real-world use cases.** DTOs, API payload shapes, config objects, React props.

**Good practices.** Prefer `readonly` on props/DTOs you don't intend to mutate — cheap safety. Use optional (`?`) only for genuinely-optional data, not as a lazy way to avoid initializing something.
**Bad practices.** Overusing index signatures (`{ [key: string]: any }`) when the actual key set is known and finite — throws away all the safety a concrete shape would give you.
**Common mistakes.** Expecting `readonly` to deep-freeze nested objects — it's shallow; `readonly` on `{ nested: { a: number } }` still lets you do `obj.nested.a = 5`.
**Edge case.** Excess property checks are bypassed by an intermediate variable (shown above) *and* by spread (`draw({ ...obj, z: 3 })` is also exempted by some versions/patterns) — this is a real soundness gap senior engineers should know about explicitly, not just intuit.
**Performance.** Wide object types (hundreds of properties) are checked via structural member comparison — cost roughly scales with member count on both sides for each assignability check; this rarely matters until you're at generated-schema scale (e.g., huge auto-generated API types).

### Interview Q&A — §2.3
**Junior:** Q: What does `readonly` do on an object type property? A: Prevents reassignment of that property through that typed reference (compile-time only, not a runtime freeze).
**Mid:** Q: Why does TS error on an excess property in an object literal argument but not on the same object stored in a variable first? A: Excess property checking only applies to "fresh" object literals at their point of creation — once assigned to a variable, the type is treated with normal structural (width-subtyping) rules, which allow extra properties.
**Senior:** Q: Is `readonly` in TS equivalent to `Object.freeze` at runtime? A: No — `readonly` is purely a compile-time view restriction on that specific typed reference; the underlying object is fully mutable at runtime and mutable through any other (non-readonly-typed) reference to the same object. `Object.freeze` is a real runtime guarantee.
**FAANG/tricky:** Q: Why is the excess property check considered a deliberate "escape-hatch-aware" heuristic rather than a soundness feature? A: Structural typing is fundamentally about width subtyping (an object with extra props should be assignable to a narrower-shaped type — this is core soundness, "more props is fine"). Excess property checking is actually a *usability* heuristic layered on top specifically for object literals, to catch likely typos (`{ nmae: "x" }`) — TS knowingly narrows the otherwise-sound structural rule at literal-creation sites because that's where typos are statistically most likely and most useful to catch.

**Debugging challenge**
```ts
interface Config { timeout: number; }
function setup(c: Config) {}
const userConfig = { timeout: 5000, retries: 3 };
setup(userConfig); // compiles! why doesn't this feel wrong?
```
→ Structural width subtyping: `userConfig` has all of `Config`'s required members; extra members are fine for non-literal structural assignability. Only *literal* arguments get excess-property-checked.

---

## 2.4 Arrays

**Intuition.** An array type says "a resizable, homogeneous(-ish) sequence" — `number[]` means "any number of numbers, in order, indexable by position."

**Technical explanation.** `number[]` ≡ `Array<number>` (identical, just syntax sugar). Readonly variant: `readonly number[]` ≡ `ReadonlyArray<number>` — blocks mutating methods (`push`, `sort`, `splice`, index assignment) at the type level only.

**Internal behavior.** Arrays are structurally just objects with numeric index signatures + the `Array.prototype` method signatures (from `lib.es5.d.ts` etc.) — there's no special "array-ness" beyond that structural shape (which is also why array-like objects can sometimes structurally satisfy array types accidentally).

**Compiler behavior / inference.**
```ts
const a = [1, 2, 3];        // number[]
const b = [1, "two"];       // (string | number)[]
const c: readonly number[] = [1, 2, 3];
c.push(4); // Error — push doesn't exist on readonly number[]
```
Mixed-literal arrays infer a **union** element type — not `any`, a real (possibly wide) union.

**Real-world use cases.** Lists of homogeneous domain entities; `readonly T[]` for function parameters you promise not to mutate (a strong, cheap API contract signal).

**Good practices.** Accept `readonly T[]` in function signatures whenever you don't need to mutate the input — it widens what callers can pass (a mutable array *is* assignable to a readonly-typed parameter, but not vice versa — variance in your favor, Ch. 3).
**Bad practices.** Using `any[]` for "an array of mixed stuff I don't want to think about" instead of a proper union or tuple.
**Common mistakes.** Assuming `readonly number[]` prevents the *original* array from ever being mutated — it only restricts mutation through that particular typed binding; another reference without `readonly` can still mutate the same underlying array.
**Edge case.** `noUncheckedIndexedAccess` (Ch.1) changes `arr[i]` from `T` to `T | undefined` — without it, out-of-bounds access is a silent, unsound `undefined` typed as `T`.
**Performance.** No special cost beyond generic structural array-of-T checking; large union element types from heterogeneous array literals can compound if propagated into generic inference downstream.

### Interview Q&A — §2.4
**Junior:** Q: What's the difference between `number[]` and `Array<number>`? A: None semantically — different syntax for the same type.
**Mid:** Q: What does `readonly number[]` actually prevent? A: Compile-time use of mutating methods/index assignment through that specific typed reference; it's not a runtime freeze and doesn't stop mutation via other non-readonly references to the same array.
**Senior:** Q: Why is it generally safe/idiomatic to accept `readonly T[]` in a function parameter even if callers usually pass mutable arrays? A: A mutable `T[]` is a subtype of `readonly T[]` (you can always use a more-capable array where a less-capable/read-only view is expected — this is sound covariance for read-only access), so it costs callers nothing while communicating and enforcing "this function won't mutate your array."

**Predict the inferred type**
```ts
const mixed = [1, "a", true]; // (string | number | boolean)[]
const empty = [];             // any[] (widens further to any[] without contextual typing — later pushes don't retroactively narrow it well; a classic gotcha)
```

---

## 2.5 Tuples

**Intuition.** A tuple is an array with a fixed, known length and a specific type *per position* — "the first slot is always a string, the second always a number," unlike a regular array where every slot is the same type.

**Technical explanation.**
```ts
type Pair = [string, number];
const p: Pair = ["age", 30];
type Named = [name: string, age: number]; // labeled tuple elements (readability only, no runtime effect)
type Variadic = [first: string, ...rest: number[]]; // rest elements (Ch.6 generic variadic tuples build on this)
type OptionalTuple = [string, number?]; // optional trailing element
```

**Internal behavior.** Tuples are still arrays at runtime (real JS arrays) — the fixed-length/per-position typing is a **compile-time-only overlay**. The checker tracks a `length` literal type too, which is what enables exhaustive-position checks and is why destructuring a tuple gives precise per-variable types.

**Compiler behavior / inference.** Array literals do **not** infer as tuples by default — they infer as arrays with a union element type, unless you provide a tuple type annotation or use `as const`:
```ts
function f(): [string, number] { return ["x", 1]; } // OK, return-type contextual typing makes it a tuple
let t = ["x", 1];        // (string | number)[] — NOT a tuple by default
const t2 = ["x", 1] as const; // readonly ["x", 1] — precise tuple, literal types, readonly
```

**Real-world use cases.** React's `useState` returns a tuple (`[value, setter]`) specifically so destructuring gets exact types per position and you can name them freely — this is the canonical real-world tuple use case worth knowing cold for interviews. Also: coordinate pairs, CSV row modeling, function argument-list types (`Parameters<T>` returns a tuple).

**Good practices.** Use tuples when position carries fixed meaning and count is fixed/small; otherwise prefer a named object (`{x, y}` beats `[number, number]` for readability once it's not a well-known convention like coordinates).
**Bad practices.** Using tuples for "a list of things" where order/position isn't semantically meaningful — that's what arrays are for.
**Common mistakes.** Expecting array literals to auto-infer as tuples — forgetting `as const` or an explicit annotation, then being confused why destructuring gives wider unioned types.
**Edge case.** Variadic tuples (`[string, ...number[]]`) combined with generic inference are the backbone of how TS types `Function.prototype.bind`, `Array.prototype.concat`, and similar variadic-argument APIs precisely (Ch. 6 goes deep).
**Performance.** Negligible on its own; becomes relevant combined with heavy generic tuple manipulation (recursive tuple-processing types), covered in Ch. 8–9.

### Interview Q&A — §2.5
**Junior:** Q: How is a tuple different from a regular array type at runtime? A: It isn't — tuples are ordinary JS arrays at runtime; the fixed-length, per-position typing exists only at compile time.
**Mid:** Q: Why does `let t = ["x", 1]` not infer as `[string, number]`? A: Without a tuple type annotation, contextual type, or `as const`, TS defaults array literal inference to a mutable array with a unioned element type, since that's the more general (safer default) shape.
**Senior:** Q: Why does React's `useState` return a tuple instead of an object like `{ value, setValue }`? A: A tuple lets consumers destructure with **arbitrary local names** (`const [count, setCount] = useState(0)`) while still getting precise, position-specific types — an object would force consumers to either use fixed property names or manually rename via destructuring aliasing (`{ value: count }`), which is more verbose for something used dozens of times per component.

**Predict the inferred type**
```ts
function useState<T>(initial: T): [T, (v: T) => void] { /* ... */ return [initial, () => {}]; }
const [count, setCount] = useState(0); // count: number, setCount: (v: number) => void
```

---

## 2.6 Enums

**Intuition.** An enum names a fixed set of related constants — "the days of the week," "the states an order can be in" — giving them a shared type and (for numeric enums) a bidirectional value↔name mapping.

**Technical explanation.**
```ts
enum Direction { Up, Down, Left, Right }        // numeric, auto-incremented 0,1,2,3
enum Status { Active = "ACTIVE", Done = "DONE" } // string enum — no reverse mapping
const enum Fast { A, B }                         // const enum — inlined, no object emitted
```

**Internal behavior — this is the one place §1.2's "types are fully erased" claim has a real exception.** A regular `enum` compiles to an **actual runtime object**:
```js
// enum Direction { Up, Down }  compiles roughly to:
var Direction;
(function (Direction) {
  Direction[Direction["Up"] = 0] = "Up";
  Direction[Direction["Down"] = 1] = "Down";
})(Direction || (Direction = {}));
```
This creates the bidirectional map (`Direction.Up === 0` and `Direction[0] === "Up"`) — **only for numeric enums**; string enums don't get reverse mapping (would be lossy/ambiguous otherwise) and compile to a simpler flat object.

`const enum` is different: it requires whole-program knowledge (the checker inlines the literal value directly at every usage site, emitting **zero runtime object** at all) — which is exactly why it's incompatible with `isolatedModules`/single-file transpilers (§1.4).

**Compiler behavior / inference.** Enum members are literal types themselves; a `Status` variable's type is the enum type, structurally compatible with... itself, mostly (numeric enums are *nominally-flavored* — see edge case below, one of the few nominal-ish corners of TS).

**Real-world use cases.** Historically common for status/state fields; increasingly, many senior teams prefer **union-of-string-literals** instead (`type Status = "ACTIVE" | "DONE"`) because it's zero-runtime-cost, trivially serializes to/from JSON, and has no nominal quirks.

**Good practices.** If you use enums, prefer string enums (self-documenting values, no accidental numeric reordering bugs) or `const enum` in single-project (non-`isolatedModules`) setups where inlining is safe. Many senior TS style guides (including parts of the TS team's own guidance) now recommend literal unions over enums by default.
**Bad practices.** Numeric enums where insertion order matters for external persistence (adding a member in the middle silently renumbers everything after it — a serialization/DB-compatibility landmine).
**Common mistakes.** Forgetting `const enum` can't be used with `isolatedModules`, Babel, or in some bundler setups — breaks builds for reasons unrelated to your code's correctness.
**Edge case.** Numeric enums are structurally *loose* in one direction (any `number` is assignable to a numeric enum type! — a real unsoundness gap: `let d: Direction = 999;` compiles), while string enums are properly nominal-like (only exact enum members are assignable, not arbitrary strings) — this asymmetry is a classic tricky-interview fact.
**Performance.** Regular enums add real runtime code/bytes (the IIFE object above) shipped to every bundle that imports them, however small; `const enum` and literal unions add zero bytes.

### Interview Q&A — §2.6
**Junior:** Q: What's the difference between a string enum and a numeric enum? A: Numeric enums auto-assign incrementing numbers and get a reverse (value→name) mapping; string enums require explicit values and have no reverse mapping.
**Mid:** Q: Does an `enum` exist at runtime? A: Yes (unless it's a `const enum`) — regular enums compile to a real JS object; this is one of the few TS constructs that isn't purely erased.
**Senior:** Q: Why do many senior TS style guides now prefer string literal unions over enums? A: Enums add real runtime code (bundle size), have JSON-(de)serialization friction (numeric enums serialize as numbers, easy to misinterpret; the reverse mapping pollutes `Object.keys`), and numeric enums have an unsound gap where any arbitrary number is assignable to the enum type — literal unions are zero-cost, strictly checked, and interop trivially with plain JSON.
**FAANG/tricky:** Q: Why can't `const enum` be used safely with Babel or `isolatedModules`? A: `const enum` requires the compiler to resolve and inline the literal value of every member at *every usage site* across the whole program — a cross-file, type-aware transform. Single-file transpilers (Babel/etc.) process each file in isolation with no knowledge of other files' enum declarations, so they cannot perform this inlining and would either error or (worse) silently emit incorrect/undefined references.

**Predict the output**
```ts
enum Direction { Up, Down }
let d: Direction = 999; // does this compile?
```
→ **Yes, it compiles** — numeric enums are unsoundly open to any `number` value, a well-known TS gotcha.

---

## 2.7 Const Assertions (`as const`)

**Intuition.** `as const` tells the compiler: "treat every literal in this expression as frozen and exact — don't widen anything, and make it deeply readonly." It's the tool that closes the gap described in §2.2/2.5 where object/array literals normally widen.

**Technical explanation.**
```ts
let a = "hello" as const;         // type: "hello" (not string)
const point = { x: 1, y: 2 } as const; // type: { readonly x: 1; readonly y: 2 }
const tuple = [1, "a"] as const;  // type: readonly [1, "a"]
```

**Internal behavior.** `as const` is a special assertion recognized by the checker specifically (not a generic type assertion mechanism reused) — it recursively applies: (a) literal-type inference instead of base-type widening for every primitive, (b) `readonly` on every array/object member, (c) array literals become **readonly tuples** rather than mutable arrays.

**Compiler behavior / inference.** Because the result is `readonly`, this pairs naturally with using the object as an exhaustive, type-safe "enum-like" constant set:
```ts
const Directions = ["Up", "Down", "Left", "Right"] as const;
type Direction = typeof Directions[number]; // "Up" | "Down" | "Left" | "Right"
```
This `typeof X[number]` pattern — "derive a union type from a runtime array of literals" — is one of the most useful, most-tested-in-interviews idioms in modern TS, because it keeps a single source of truth (the runtime array) instead of maintaining a parallel type and array by hand.

**Real-world use cases.** Deriving literal union types from real runtime arrays/config objects (single source of truth); Redux action creators (`{ type: "ADD" as const, payload }` so the union of actions discriminates correctly, Ch. 5); freezing config objects.

**Good practices.** Use the `as const` + `typeof X[number]` pattern instead of manually duplicating a literal union alongside a runtime array — one source of truth, structurally impossible to drift out of sync.
**Bad practices.** Applying `as const` and then still trying to mutate the value (compiles to an error, but shows a misunderstanding of intent) — if you need mutation, `as const` is the wrong tool.
**Common mistakes.** Forgetting `as const` is shallow-*looking* but is actually **deep** (unlike `readonly` on a hand-written type, which is shallow by default) — nested objects/arrays inside an `as const` expression are recursively frozen too, which surprises people expecting shallow behavior to match plain `readonly`.
**Edge case.** `as const` on an object with a function property doesn't do anything meaningful to the function's type (functions aren't literal-narrowable the same way) — it only affects primitive/array/object literal members.
**Performance.** None meaningful — purely a compile-time inference directive.

### Interview Q&A — §2.7
**Junior:** Q: What does `as const` do to `let x = "a" as const`? A: Keeps `x`'s type as the literal `"a"` instead of widening to `string`.
**Mid:** Q: How would you derive a union type `"a" | "b" | "c"` from a runtime array `["a", "b", "c"]` without writing the union by hand? A: `const arr = ["a","b","c"] as const; type U = typeof arr[number];`
**Senior:** Q: Is `as const`'s readonly effect shallow or deep, and how does that differ from a manually-written `readonly` property? A: `as const` is deep/recursive — every nested array/object literal inside the expression becomes `readonly` and every primitive becomes a literal type. A manually written `readonly` modifier on a type is shallow by default — it only protects the one property it's written on, not nested contents, unless you explicitly wrap nested types too (or use a recursive `DeepReadonly<T>` utility, Ch. 8).

**Predict the inferred type**
```ts
const config = { retries: 3, methods: ["GET", "POST"] } as const;
// type: { readonly retries: 3; readonly methods: readonly ["GET", "POST"] }
```

---

## 2.8 Type Widening

**Intuition.** Widening is the checker's default caution: "if I don't know this binding is fixed forever, I'll generalize the type so future reassignments aren't wrongly rejected." It's the mechanism behind almost every "why isn't my type as specific as I expected" question in this chapter.

**Technical explanation.** Two distinct widening phenomena:
1. **Literal widening**: `5` → `number`, `"a"` → `string`, `true` → `boolean`, for `let`/`var` bindings and mutable object/array literal members without contextual typing.
2. **`null`/`undefined` widening**: without `strictNullChecks`, `let x = null` widens `x`'s type all the way to `any` (!) because TS pre-strict-mode had no way to represent "just null" usefully. Under `strictNullChecks`, it stays `null` (still often too narrow to be useful without an explicit annotation).

**Internal behavior.** The checker computes two types internally for many expressions: the actual literal type it can prove, and the **widened type** used for the variable's declared type going forward — this is literally called "widening" in the compiler source, and it happens at variable-declaration boundaries, not at every expression.

**Compiler behavior / inference.** Widening is *contextual*: it doesn't happen if:
- The variable is `const` and the value is a bare literal (no widening needed — nothing can reassign it, §2.2).
- There's an explicit type annotation (`let x: "a" = "a"` — annotation always wins over inference).
- The literal appears in a position with an existing **contextual type** expecting something narrower (e.g., passing a literal directly as a function argument typed to a literal union — no intermediate variable, no widening opportunity).

**Real-world use cases.** This is *why* `as const` and explicit annotations exist as tools — understanding widening is what tells you *when* you need them.

**Good practices.** When you want a `let` variable to keep a literal type, annotate it explicitly rather than fighting inference.
**Bad practices.** Being surprised repeatedly by widening instead of internalizing the rule (`let` widens, `const` + no mutation possible keeps literal, objects widen per-property regardless of `const`).
**Common mistakes.** The single most common real-world bug: `const config = { status: "pending" }; someFn(config.status)` where `someFn` expects a literal union — fails because `status` widened to `string` at the object-literal level despite `const` on `config` itself.
**Edge case.** Function/method **return type inference** does *not* widen literal types the way variable declarations do in some cases — return-position literals can stay narrow when flowing into contextual typing (as seen in the generic example in §2.2, `pick("hello")` staying `"hello"`), which is inconsistent-*looking* with plain variable widening unless you understand it's driven by contextual/generic inference rules, not the same widening pass.
**Performance.** N/A — pure inference-time behavior.

### Interview Q&A — §2.8
**Junior:** Q: Why does `let x = "hi"` have type `string`, not `"hi"`? A: `let` allows reassignment, so TS widens the literal to its base type to permit future different-but-compatible values.
**Mid:** Q: Does `const obj = { a: "hi" }` give `obj.a` the type `"hi"` or `string`? A: `string` — `const` only fixes the `obj` binding itself; the property `a` is still mutable, so it independently widens.
**Senior:** Q: Explain precisely why `let x = null` behaves differently with and without `strictNullChecks`. A: Without `strictNullChecks`, `null`/`undefined` are considered assignable to (and part of) every type, so there's no useful narrow type to assign — TS widens all the way to `any`. With `strictNullChecks`, `null` is its own distinct type, so widening stops at `null` (still generally too narrow for a mutable variable meant to later hold real values, hence usually needs an explicit annotation like `let x: string | null = null`).

**Predict the inferred type**
```ts
let a = null;                     // strictNullChecks: any (no strict) / null (strict)
const b = null;                   // null in both modes (const, never reassigned)
let c: string | null = null;      // string | null — explicit annotation prevents widening surprises
```

---

## 2.9 Narrowing

**Intuition.** Narrowing is the reverse of widening: the checker watches your control flow (`if`, `switch`, `&&`, early `return`) and progressively shrinks a variable's type within each branch to only what's still logically possible there.

**Technical explanation — narrowing mechanisms:**
- `typeof x === "string"`
- `x instanceof SomeClass`
- `"prop" in x`
- Equality checks (`x === null`, `x === "literal"`)
- Truthiness (`if (x)`)
- Discriminated union tag checks (`if (shape.kind === "circle")`) — the single most important pattern for scalable union modeling (Ch. 5).
- User-defined type guards: `function isString(x: unknown): x is string`
- Assertion functions: `function assert(x: unknown): asserts x`

**Internal behavior — Control Flow Analysis (CFA).** The checker builds a **control flow graph** of your function body and, at each node, computes the narrowed type by intersecting the declared type with everything provably true along every path that reaches that node. This is why narrowing works after early returns, inside `else` branches (the *negation* of the `if` condition), and even across reassignments — the checker tracks type *per program point*, not just per variable declaration. This is genuinely one of the most sophisticated parts of the whole compiler (deep dive in Ch. 13).

**Compiler behavior / inference.**
```ts
function f(x: string | number) {
  if (typeof x === "string") {
    x.toUpperCase(); // x: string here
  } else {
    x.toFixed(2);    // x: number here — narrowed by negation of the typeof check
  }
}
```

**Exhaustiveness checking** — using `never` deliberately to catch unhandled union members at compile time:
```ts
type Shape = { kind: "circle"; r: number } | { kind: "square"; s: number };
function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle": return Math.PI * shape.r ** 2;
    case "square": return shape.s ** 2;
    default:
      const _exhaustive: never = shape; // errors if a new Shape variant is added and unhandled
      throw new Error("unreachable");
  }
}
```
This pattern is a top senior-interview signal — it turns "did you forget a case" into a compile error the moment the union grows, instead of a runtime bug.

**Real-world use cases.** API response handling (`{status: "success", data} | {status: "error", message}`), Redux reducers, form validation states, parser/AST node handling — anywhere you have "one of several known shapes."

**Good practices.** Model domain states as discriminated unions + narrow with `switch` + exhaustiveness `never` check, rather than optional-everything objects with implicit invalid states.
**Bad practices.** Modeling state as `{ status?: string; data?: T; error?: string }` (all-optional) — every combination is technically constructible even the nonsensical ones (`status: "success"` with an `error` set), and nothing forces you to handle all real cases.
**Common mistakes.** Narrowing lost across a function boundary — passing a narrowed variable into a callback/closure doesn't always preserve narrowing if the variable could theoretically be reassigned before the closure runs (the checker is conservative here — a classic "why did my narrowing disappear" bug).
**Edge case.** Narrowing via a destructured variable from an object does **not** automatically narrow the *original* object's property the same way, and vice versa — narrowing is tracked per-binding/per-access-path, so `const { status } = response; if (status === "success") { response.data }` may not narrow `response.data` the way you'd expect unless the checker's access-path tracking specifically supports it (it does for direct property access in modern TS via "discriminant property narrowing," but breaks with indirection like an extracted function).
**Performance.** CFA cost scales with function complexity (branches × variables tracked) — pathologically large functions with many reassignments and branches are a genuine, if rare, compile-time cost center.

### Interview Q&A — §2.9
**Junior:** Q: What is type narrowing? A: The compiler refining a variable's type within a code branch based on runtime checks like `typeof`, `instanceof`, or equality comparisons.
**Mid:** Q: What's a discriminated union and why is it useful? A: A union of object types sharing a common literal-typed "tag" property (e.g., `kind`); switching/narrowing on that tag lets TS precisely narrow to the exact variant in each branch, enabling exhaustiveness checking.
**Senior:** Q: How does the compiler implement narrowing internally, at a high level? A: It performs control flow analysis, building a graph of the function's execution paths; at each point it computes the type as the declared type intersected with all conditions proven true along every path reaching that point (including negated conditions in `else` branches), tracked per-binding across the flow graph — not just a single static type per variable.
**FAANG/tricky:** Q: Why might narrowing "disappear" when you pass a narrowed variable into a nested closure/callback? A: TS is conservative about closures because the outer variable could theoretically be mutated between narrowing and the closure's actual execution (especially with `let`); unless the variable is `const` or the checker can prove no mutation occurs before invocation, it won't carry the narrowed type into the closure, requiring you to re-narrow inside or copy to a new `const`.

**Predict the inferred type / debugging exercise**
```ts
function process(x: string | number | null) {
  if (!x) return;
  // x: string | number  (null AND "" AND 0 are all falsy — a real gotcha: truthiness narrowing
  //    also excludes falsy string/number values from the type's perspective? NO — 
  //    it only narrows out `null`/`undefined`/other type-level falsy possibilities;
  //    it does NOT split string into "" vs non-empty, since TS doesn't track that level of value analysis)
}
```
*Teaching point:* truthiness narrowing removes `null | undefined | false | 0 | ""` **as types**, but since `string`/`number` aren't literal types here, TS can only remove the structurally-falsy component types (`null`), not carve out "falsy values of a wide type" — a nuance worth stating explicitly in interviews.

---

## Practical Exercises — Chapter 2

1. **Predict-then-verify:** For each snippet below, write down the inferred type *before* checking in an editor:
   ```ts
   const a = [1, 2, 3] as const;
   let b = { x: 1 };
   function f() { return Math.random() > 0.5 ? "a" : 1; }
   const c = [] as string[];
   ```
2. **Refactor to discriminated union:** Take `{ status?: "loading" | "success" | "error"; data?: User; error?: string }` and redesign it as a proper discriminated union, then write an exhaustive `switch` handler with a `never` check.
3. **Enum migration:** Convert a numeric enum-based `OrderStatus` to a string-literal-union-based one; list every place behavior could change (serialization, comparison, iteration).
4. **Debug:** Given
   ```ts
   const nums: number[] = [1,2,3];
   const val = nums[10];
   val.toFixed(2); // crashes at runtime
   ```
   explain why TS didn't catch this, and name the exact compiler flag that would have.

---

## Chapter 2 Summary — Mental Model

```
                     ┌─────────────┐
                     │   unknown   │  (top: everything flows in, nothing flows out unnarrowed)
                     └──────┬──────┘
             ┌───────────────┼───────────────┐
        string/number/… (primitives)     object/array/tuple/enum (structural shapes)
             │                                │
      literal types (narrower subtypes)  more specific shapes (tuples ⊂ arrays, etc.)
             │
        widen on `let`/mutation ──▶ base primitive type
        stay narrow on `const`/`as const`/contextual typing
                     │
                     ▼
                 ┌────────┐
                 │ never  │  (bottom: nothing flows in except never itself)
                 └────────┘

Narrowing = the checker walking your control flow, computing
"declared type ∩ everything proven true so far" at each program point.
Widening = the checker's default caution when it can't prove immutability.
`as const` = manually opting out of widening, recursively.
```

---

**Next:** Chapter 3 — The Type System: structural vs nominal vs duck typing, assignability, and variance (co/contra/bivariance). Say **"next"** to continue.
