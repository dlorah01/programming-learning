---
layout: default
title: "Appendix — Master Exercise Reference"
---

# Appendix — Master Exercise Reference

Every practical exercise from Chapters 1–15, worked in full. This is the single document to drill against: if you can produce (or reconstruct) every answer below from first principles — not from memory of the answer, but from the underlying reasoning — you have working mastery of the concept, not just recognition of it.

Organized by chapter. Each answer states **what** the correct solution/answer is and **why**, tied back to the specific mechanism (assignability, variance, CFA, distribution, etc.) responsible.

---

# Chapter 1 — Foundations

### 1. Setup drill: tsconfig for Node 20 ESM backend vs. Vite-bundled browser app

**Node 20 ESM backend:**
```jsonc
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022"],
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "skipLibCheck": true,
    "outDir": "dist",
    "declaration": true
  }
}
```
**Why:** `module`/`moduleResolution: "NodeNext"` because that's the only setting that accurately models Node's real dual CJS/ESM resolution algorithm (`package.json` `"exports"`, mandatory `.js` extensions on relative imports) — anything else risks passing type-checking while failing at actual runtime resolution (Ch. 11.1). `lib: ["ES2022"]` with no `"DOM"` because a backend has no `window`/`document` — omitting DOM types turns accidental browser-API usage into a compile error instead of a runtime surprise. `target: "ES2022"` because Node 20 natively supports it — no reason to downlevel and pay a bundle/perf cost for nothing (Ch. 1.4).

**Vite-bundled browser app:**
```jsonc
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "strict": true,
    "jsx": "react-jsx",
    "noEmit": true
  }
}
```
**Why:** `moduleResolution: "Bundler"` because Vite (not `tsc`) actually resolves and bundles modules — this setting matches Vite's permissive, non-Node-ESM-strict resolution behavior (no mandatory extensions) rather than fighting it. `noEmit: true` because `tsc` here is a pure type-checking gate; Vite/esbuild does the actual transpile+bundle (Ch. 1.3's checking-vs-emitting separation, applied practically). `lib` includes `"DOM"` because browser globals are genuinely used.

### 2. Diagnosis checklist: "TypeScript didn't catch this bug in production"

Systematic checklist, each item a distinct mechanism from Ch. 1–3:
- **Type erasure** (Ch. 1.2) — was the bug a runtime-only concern (e.g., `instanceof` on an interface) that TS structurally cannot catch?
- **`any` leakage** (Ch. 2.1) — did an `any` (explicit, or implicit via missing `noImplicitAny`) silently disable checking somewhere upstream of the bug?
- **Unchecked external data** — was the value asserted via `as` (Ch. 3.4/14.1) from `JSON.parse`/an API response without runtime validation, i.e. was the type simply a lie the developer told the compiler?
- **`noEmitOnError` off + ignored CI failure** (Ch. 1.3) — did `tsc` actually error, but the build shipped anyway because emit isn't blocked by default and CI didn't gate on it?
- **Wrong file set** (Ch. 1.5) — was the buggy file actually excluded from the checked program (wrong `include` glob), so it was never type-checked at all despite living in the repo?
- **Deliberate unsoundness** (Ch. 16.1's list) — was it array covariance, bivariant methods, a numeric enum accepting an arbitrary number, or an incorrect user-defined type guard (Ch. 13.3)?

### 3. Build-speed audit: 5-package linear-dependency monorepo

Introduce `composite: true` + `references` in each package's `tsconfig.json`, pointing at its direct dependency; add a root "solution" `tsconfig.json` with `files: []` and `references` to all 5. Run `tsc --build`. **Why this helps mechanically:** each package emits a `.tsbuildinfo` cache file recording its own + dependencies' state (Ch. 1.5); `tsc --build` only reprocesses a package if its own source or an upstream dependency's output actually changed — a change to package 5 (leaf) no longer forces rechecking packages 1–4, turning an O(whole repo) rebuild into O(changed subgraph). Measure by touching only the leaf package and timing before/after — the win should be dramatic once references are wired correctly.

### 4. Conceptual proof: why `interface Foo {}` emits zero bytes but `class Foo {}` doesn't

An `interface` is a pure compile-time structural contract with no runtime existence at all (Ch. 1.2/3.1) — the checker uses it purely for assignability comparisons, and it corresponds to nothing that needs to exist when the program actually runs. A `class`, by contrast, is dual: it declares both a *type* (the instance shape, used the same way an interface is for checking) **and** a real runtime value — the constructor function itself, which real code calls via `new`, which real `instanceof` checks reference, and which may carry real static members. Because the value side has genuine runtime behavior, the emitter cannot erase it; it emits the actual constructor function (and any class-field initialization logic). This value/type duality is unique to `class` among the constructs covered in Ch. 3.1/12.4.

---

# Chapter 2 — Types: Primitives to Narrowing

### 1. Predict-then-verify
```ts
const a = [1, 2, 3] as const;      // readonly [1, 2, 3]
let b = { x: 1 };                   // { x: number } — property widens regardless of let/const on b
function f() { return Math.random() > 0.5 ? "a" : 1; } // return type: string | number
const c = [] as string[];           // string[]
```
**Why:** `a` — `as const` recursively freezes and literal-narrows both the tuple shape and each element (Ch. 2.7). `b` — object literal property widening (Ch. 2.8) happens independently of whether the *outer* binding is `let`/`const`, since `const` only fixes the `b` binding itself, not the mutability of `x`. `f` — function return type inference unions the types across every `return` statement via control-flow analysis (Ch. 13.2). `c` — an explicit type assertion on an empty array literal fixes its element type immediately, bypassing the "implicit `any[]`" widening that a bare `const c = []` would otherwise suffer (Ch. 2.4).

### 2. Refactor to discriminated union
```ts
// Before: { status?: "loading" | "success" | "error"; data?: User; error?: string }
type FetchState =
  | { status: "loading" }
  | { status: "success"; data: User }
  | { status: "error"; error: string };

function handle(state: FetchState): string {
  switch (state.status) {
    case "loading": return "Loading…";
    case "success": return state.data.name;
    case "error": return state.error;
    default:
      const _exhaustive: never = state;
      return _exhaustive;
  }
}
```
**Why:** The original all-optional shape permits nonsensical combinations (`status: "loading"` with `data` populated) that are structurally impossible to construct in the union version (Ch. 2.9/5.4) — "make illegal states unrepresentable." The `never`-typed `default` branch only compiles if every variant is genuinely handled (Ch. 13.4); adding a fourth status later without updating the `switch` produces a precise compile error at that exact spot.

### 3. Enum migration: numeric → string-literal union
```ts
// Before: enum OrderStatus { Pending, Shipped, Delivered }
type OrderStatus = "pending" | "shipped" | "delivered";
```
**Behavior changes to document:** (1) **Serialization** — the numeric enum serializes to JSON as `0`/`1`/`2` (opaque, order-dependent); the literal union serializes as the readable string itself, self-documenting and safe even if enum members are reordered in source later. (2) **Comparison** — `status === OrderStatus.Shipped` (numeric) vs `status === "shipped"` (string) — the string version is trivially debuggable in logs/devtools; the numeric version requires cross-referencing the enum definition. (3) **Runtime footprint** — the numeric enum emits a real bidirectional-mapping object (Ch. 2.6); the string union emits **zero bytes**, pure compile-time erasure. (4) **Openness/soundness** — the numeric enum unsoundly accepts any arbitrary `number` (Ch. 2.6's edge case); the string union strictly rejects anything not exactly one of the three literals.

### 4. Debug: why doesn't TS catch `nums[10]` being out of bounds?
```ts
const nums: number[] = [1,2,3];
const val = nums[10]; // TS types this as `number`, not `number | undefined`
val.toFixed(2); // crashes: val is actually undefined
```
**Why TS misses it:** by default, array index access is typed unsoundly — `arr[i]` is always `T`, regardless of whether `i` is provably in bounds (Ch. 2.4's edge case). **The fix:** enable `noUncheckedIndexedAccess: true` (Ch. 1.6) — this changes `nums[10]`'s type to `number | undefined`, forcing a narrowing check before `.toFixed()` is callable, catching exactly this class of bug at compile time.

---

# Chapter 3 — The Type System

### 1. Structural proof exercise
```ts
interface Flyer { fly(): void; }
interface Bird { fly(): void; }
function launch(f: Flyer) { f.fly(); }
const b: Bird = { fly() {} };
launch(b); // OK — identical structure, zero declared relationship

interface FlyerV2 { fly(): void; altitude: number; }
function launch2(f: FlyerV2) { f.fly(); }
launch2(b); // Error — Bird lacks `altitude`, the required member FlyerV2 now demands
```
**Why:** Assignability is purely member-by-member structural comparison (Ch. 3.1/3.4) — adding a required member to only one side breaks compatibility precisely because width subtyping requires the source to be a superset-or-equal of the target's required members; `Bird` no longer satisfies that once `FlyerV2` demands `altitude`.

### 2. Build a branded type
```ts
declare const brand: unique symbol;
type Brand<T, B extends string> = T & { readonly [brand]: B };
type Email = Brand<string, "Email">;
type Username = Brand<string, "Username">;

function toEmail(s: string): Email {
  if (!s.includes("@")) throw new Error("invalid email");
  return s as Email;
}
function toUsername(s: string): Username {
  if (s.length < 3) throw new Error("too short");
  return s as Username;
}
function sendEmail(to: Email) {}
sendEmail("not-validated@example.com"); // Error — plain string, not Email
sendEmail(toEmail("real@example.com"));  // OK
```
**Why:** the `unique symbol`-keyed intersection is structurally unsatisfiable by any ordinary string (Ch. 3.2/9.2) — only values that pass through `toEmail`/`toUsername` (the "smart constructors") can ever produce the branded type, which is precisely why a raw string literal is rejected even though it's *structurally* just a string underneath.

### 3. Variance trace: `Comparator<T> = (a: T, b: T) => number`
Is `Comparator<Animal>` assignable to `Comparator<Cat>`? **Yes.** A function that can compare any two `Animal`s can certainly compare two `Cat`s (a `Cat` is an `Animal`) — the parameter position is contravariant (Ch. 3.5/3.6), so accepting a *wider* parameter type than strictly required is safe, meaning `Comparator<Animal>` can stand in wherever `Comparator<Cat>` is expected. The reverse — `Comparator<Cat>` assignable to `Comparator<Animal>` — is **not** safe: a function that only knows how to compare `Cat`s can't safely be called with a `Dog` (also an `Animal`), so it cannot substitute for something requiring `Comparator<Animal>`.

### 4. Bivariance hunt
```ts
interface Repo { save(item: { id: string }): void; }        // method — bivariant
interface RepoProp { save: (item: { id: string }) => void; } // property — strict contravariant

const r: Repo = { save(item: { id: string; ownerId: string }) { } }; // compiles (bivariant, narrower param accepted)
const rp: RepoProp = { save(item: { id: string; ownerId: string }) { } }; // Error under strictFunctionTypes
```
**Why the difference exists at all:** documented deliberately in Ch. 3.7 — method-shorthand signatures are always bivariantly checked because narrower-parameter method overrides are an extremely common, usually-safe real-world OOP pattern, and the TS team judged strict contravariance there would reject too much legitimate code relative to the actual bugs caught; function-typed properties get no such exception.

---

# Chapter 4 — Functions

### 1. Overload design + deliberate shadowing bug
```ts
function createElement(tag: "input"): HTMLInputElement;
function createElement(tag: "button"): HTMLButtonElement;
function createElement(tag: string): HTMLElement;
function createElement(tag: string): HTMLElement { return document.createElement(tag); }

// Shadowing bug — general overload placed FIRST:
function createElementBuggy(tag: string): HTMLElement;
function createElementBuggy(tag: "input"): HTMLInputElement; // unreachable for "input" calls!
function createElementBuggy(tag: string): HTMLElement { return document.createElement(tag); }
const el = createElementBuggy("input"); // el: HTMLElement, NOT HTMLInputElement
```
**Why:** overload resolution is strictly first-match-wins, top to bottom (Ch. 4.3) — it never searches for the "best" match. Placing the general `string` overload before the specific `"input"` overload means every call resolves against the general one first, silently losing precision with no error.

### 2. Options-object refactor
```ts
interface CreateUserOptions {
  name: string;
  active?: boolean;
  admin?: boolean;
  verified?: boolean;
}
function createUser({ name, active = true, admin = false, verified = false }: CreateUserOptions) {}
createUser({ name: "Alice", admin: true });
```
**Why this is strictly better:** positional booleans give zero readability at the call site about which flag is which (Ch. 4.2) and are trivially easy to transpose by accident with no compiler warning (both are `boolean`, structurally interchangeable positionally); an options object makes every argument self-labeling and order-independent.

### 3. `this`-safety exercise
```ts
interface Sized { readonly length: number; }
function reportSize(this: Sized) { console.log(this.length); }
const arr = { length: 5, reportSize };
arr.reportSize();           // OK — `this` matches Sized
const detached = arr.reportSize;
detached();                    // Error — bare call has no valid `this` matching Sized
```
**Why:** the `this: Sized` parameter (Ch. 4.5) is erased at runtime but checked at every call site — calling as a method supplies `arr` itself as `this`; a detached bare call has no receiver at all, failing the check — a compile-time catch for a class of runtime `this`-loss bug.

### 4. Variadic rest `pipe` function
```ts
function pipe<A, B>(f1: (a: A) => B): (a: A) => B;
function pipe<A, B, C>(f1: (a: A) => B, f2: (b: B) => C): (a: A) => C;
function pipe<A, B, C, D>(f1: (a: A) => B, f2: (b: B) => C, f3: (c: C) => D): (a: A) => D;
function pipe(...fns: Array<(x: any) => any>) {
  return (x: any) => fns.reduce((acc, fn) => fn(acc), x);
}
```
**Why this needs Ch. 6's variadic tuples for full generality:** the overload-per-arity approach only covers a fixed, manually-enumerated set of chain lengths. A fully general solution needs a single generic signature using variadic tuple types to recursively express "an arbitrary-length chain where each function's input matches the previous function's output" (Ch. 6.6/8.5).

---

# Chapter 5 — Objects

### 1. Convention exercise
Rule applied: plain object shape *intended as an extension point* → `interface`; anything computed, unioned, tupled, or otherwise not a plain extensible shape → `type`. Converting a union or function-type alias to `interface` syntax is simply **impossible** (Ch. 5.1) — concretely demonstrating a real capability gap, not just a style preference.

### 2. Module augmentation drill
```ts
// types/express-augment.d.ts
import "express";
declare module "express" {
  interface Request { requestId: string; }
}
```
Verify: any file importing `Request` from `"express"` anywhere in the program now sees `requestId` — including inside Express's own middleware type signatures (Ch. 5.2/11.5). If it doesn't appear, the likely cause is the augmenting file not being included in the compiled program (Ch. 1.5) or a module-path resolution mismatch (Ch. 11.5).

### 3. Extends vs. intersect trap
```ts
type Base = { status: "active" };
type Override = { status: "inactive" };
type Broken = Base & Override; // status: never — silent collapse
const x: Broken = { status: "active" }; // Error, confusingly deep inside a `never` member

// Refactored with extends — fails IMMEDIATELY at declaration:
interface BaseI { status: "active"; }
interface OverrideI extends BaseI { status: "inactive"; } // Error right here
```
**Why the refactor is strictly better for maintainability:** `extends` triggers an upfront compatibility check at the exact point the conflicting declaration is written (Ch. 5.3) — the error is immediately local. The `&` version defers failure to wherever the resulting type is later *used*, arbitrarily far from the actual conflicting declarations.

### 4. Union modeling
```ts
type PaymentState =
  | { kind: "unpaid" }
  | { kind: "paid"; amount: number; paidAt: Date }
  | { kind: "refunded"; amount: number; refundedAt: Date; reason: string };

function describe(state: PaymentState): string {
  switch (state.kind) {
    case "unpaid": return "Not yet paid";
    case "paid": return `Paid $${state.amount} on ${state.paidAt.toDateString()}`;
    case "refunded": return `Refunded $${state.amount}: ${state.reason}`;
    default: const _e: never = state; return _e;
  }
}
```
**Why:** fields previously modeled with all-optional flags permitted invalid combinations like `paid: true` with no `amount`; the union removes that possibility structurally, and exhaustiveness guarantees the handler stays complete as new states are added (Ch. 2.9/13.4).

---

# Chapter 6 — Generics

### 1. Generic vs. `unknown` audit
Heuristic (Ch. 6.1): does a type parameter appear in more than one position (linking two parameters, or a parameter and the return type)? If yes — genuinely generic. If a type parameter appears exactly once with nothing downstream depending on it matching anything else, it can typically be replaced by `unknown` with no loss of real safety.

### 2. `Result<T, E>` with `map`/`flatMap`/`unwrapOr`
```ts
type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E };
const ok = <T>(value: T): Result<T, never> => ({ ok: true, value });
const err = <E>(error: E): Result<never, E> => ({ ok: false, error });

function map<T, U, E>(r: Result<T, E>, f: (v: T) => U): Result<U, E> {
  return r.ok ? ok(f(r.value)) : r;
}
function flatMap<T, U, E>(r: Result<T, E>, f: (v: T) => Result<U, E>): Result<U, E> {
  return r.ok ? f(r.value) : r;
}
function unwrapOr<T, E>(r: Result<T, E>, fallback: T): T {
  return r.ok ? r.value : fallback;
}
```
**Why fully generic and `any`-free is achievable here:** every operation's type flows entirely from `T`/`E`/`U`, linked through the function signatures (Ch. 6.1) — `map`'s `U` is inferred purely from `f`'s return type, requiring no explicit type arguments at any call site.

### 3. `pluck<T, K extends keyof T>`
```ts
function pluck<T, K extends keyof T>(obj: T, key: K): T[K] { return obj[key]; }
pluck({ name: "Alice", age: 30 }, "name");   // valid
pluck({ name: "Alice", age: 30 }, "email");  // Error — "email" not in keyof T
const obj = { id: 1, active: true };
const val = pluck(obj, "active");             // T inferred from the object literal
```
**Why:** `T` is inferred first from the `obj` argument (Ch. 6.1), and `K extends keyof T` is then checked against *that specific* `T` — this is why `pluck(obj, "email")` correctly fails without any hardcoded key list; `keyof T` is recomputed per call site (Ch. 6.4/7.1).

### 4. Generic `Stack<T>` with static factory
```ts
class Stack<T> {
  private items: T[] = [];
  push(item: T): void { this.items.push(item); }
  pop(): T | undefined { return this.items.pop(); }
  peek(): T | undefined { return this.items[this.items.length - 1]; }
  isEmpty(): boolean { return this.items.length === 0; }
  static of<U>(...items: U[]): Stack<U> {
    const s = new Stack<U>();
    items.forEach(i => s.push(i));
    return s;
  }
}
```
**Why `of` needs its own `U`:** static members exist independent of any instance and have no bound `T` at all (Ch. 6.3) — a static factory generic over its output type must introduce and infer its own, separate type parameter.

### 5. `Head`/`Tail`/`Last` tuple utilities
```ts
type Head<T extends unknown[]> = T extends [infer H, ...unknown[]] ? H : never;
type Tail<T extends unknown[]> = T extends [unknown, ...infer R] ? R : never;
type Last<T extends unknown[]> = T extends [...unknown[], infer L] ? L : never;
```
**Why:** each uses `infer` at a specific structural position within a tuple pattern (Ch. 6.6/8.2) — `Head` captures the first element with a trailing `...unknown[]` rest; `Last` reverses the pattern, placing the rest first.

---

# Chapter 7 — `keyof`, `typeof`, Indexed Access, Mapped Types

### 1. `MyPick`/`MyOmit`/`MyRecord` from scratch
```ts
type MyPick<T, K extends keyof T> = { [P in K]: T[P] };
type MyExclude<T, U> = T extends U ? never : T;
type MyOmit<T, K extends keyof any> = MyPick<T, MyExclude<keyof T, K>>;
type MyRecord<K extends string | number | symbol, V> = { [P in K]: V };
```
**Why:** `MyPick` directly applies the `in`-mapped-type pattern over a constrained key subset (Ch. 7.3/7.5); `MyOmit` computes "all keys except excluded ones" by combining `MyExclude` (a distributive conditional, Ch. 8.3) with `MyPick`; `MyRecord` iterates directly over an arbitrary key-type union.

### 2. Single source of truth drill
```ts
const config = { apiUrl: "https://api.example.com", timeout: 5000, retries: 3 } as const;
type ConfigKey = keyof typeof config;
```
**Why:** replacing a hand-written parallel union with `keyof typeof config` (Ch. 7.1/7.2) ties the type permanently to the real runtime object — adding/removing/renaming a key automatically updates every usage, eliminating drift.

### 3. `Getters`/`Setters` key remapping
```ts
type Getters<T> = { [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K] };
type Setters<T> = { [K in keyof T as `set${Capitalize<string & K>}`]: (value: T[K]) => void };
```
**Why:** the `as` clause (Ch. 7.5) renames each mapped key using a template literal type (Ch. 9.1) combined with `Capitalize`; `string & K` filters `K` down to string-compatible members, since `keyof T` could in principle include `symbol`/`number` keys.

### 4. Filter-by-value-type
```ts
type StringKeysOnly<T> = { [K in keyof T as T[K] extends string ? K : never]: T[K] };
```
**Why:** mapping a key to `never` inside the `as` clause drops that property entirely (Ch. 7.5) — a genuine type-level filter checking each property's *value* type.

### 5. Indexed access over a union
```ts
type A = { kind: "a"; val: number };
type B = { kind: "b"; val: string };
type AllVals = (A | B)["val"]; // number | string
```
**Why:** indexed access with a fixed key on a union computes the type at that key for *each* member and unions results (Ch. 7.4) — distinct from `keyof (A|B)`'s intersection behavior (Ch. 7.1), since the two operations have different safety requirements.

---

# Chapter 8 — Conditional Types, `infer`, Recursion

### 1. Implement from memory
```ts
type MyExclude<T, U> = T extends U ? never : T;
type MyExtract<T, U> = T extends U ? T : never;
type MyNonNullable<T> = T extends null | undefined ? never : T;
type MyReturnType<T extends (...a: any) => any> = T extends (...a: any) => infer R ? R : never;
type MyParameters<T extends (...a: any) => any> = T extends (...a: infer P) => any ? P : never;
```
**Why these work:** each is a conditional type (Ch. 8.1) whose branching depends on structural pattern matching against `T`; the `infer`-based ones additionally capture a piece of the matched structure (Ch. 8.2).

### 2. Distribution trap
```ts
type Wrap<T> = T extends unknown ? [T] : never;
type R1 = Wrap<never>;          // never
type R2 = Wrap<string>;          // [string]
type R3 = Wrap<1 | 2 | "a" | "b">; // [1] | [2] | ["a"] | ["b"]
```
**Why:** `T` is naked on the left of `extends` (Ch. 8.3), so a union instantiation distributes into one application per member, then re-unions; `never` distributes over zero members, collapsing to `never`.

### 3. Non-distributive rewrite
```ts
type WrapNonDist<T> = [T] extends [unknown] ? [T] : never;
type R4 = WrapNonDist<1 | 2 | "a" | "b">; // [1 | 2 | "a" | "b"]
```
**Why:** wrapping `T` in a tuple on both sides removes the naked-type-parameter trigger (Ch. 8.3's opt-out), so the union is tested as one indivisible unit.

### 4. Recursive `Json` + conceptual `flatten`
```ts
type Json = string | number | boolean | null | Json[] | { [key: string]: Json };
```
**Why the base case matters:** recursion bottoms out the moment a matched value is a primitive rather than `Json[]`/`{...}` — without primitives as a genuine non-recursive alternative, the type couldn't terminate (Ch. 8.5).

### 5. Trigger and fix `TS2589`
```ts
type Broken<T> = Broken<T>; // TS2589

type CountDown<N extends number, Acc extends unknown[] = []> =
  Acc["length"] extends N ? Acc : CountDown<N, [...Acc, unknown]>;
```
**Why the fix works:** each recursive call makes real, checkable progress toward termination (`Acc["length"]` growing); `Broken<T>` re-invokes itself with an unchanged argument forever, correctly triggering the recursion-depth guard (Ch. 8.5).

---

# Chapter 9 — Template Literal Types & Nominal Patterns

### 1. Route parser + params object
```ts
type ExtractParam<T extends string> =
  T extends `${string}:${infer P}/${infer Rest}` ? P | ExtractParam<Rest> :
  T extends `${string}:${infer P}` ? P : never;

type RouteParamsObj<T extends string> = { [K in ExtractParam<T>]: string };
```
**Why:** `ExtractParam` recursively pattern-matches (`infer` inside a template literal, Ch. 9.1/8.2) each `:segment`, producing a union of names; `RouteParamsObj` feeds that union into a mapped type (Ch. 7.5) as the key set.

### 2. Combinatorial cost audit
Three interpolated unions of size 15, 20, 10 → **3,000** materialized literal members (Ch. 9.1's cross-product). Verdict: a real design smell at that scale — prefer a parsing conditional type over exhaustive enumeration, or reduce union sizes.

### 3. Branded-type library
```ts
declare const brand: unique symbol;
type Brand<T, B extends string> = T & { readonly [brand]: B };
type Email = Brand<string, "Email">;
type UserId = Brand<string, "UserId">;
type PositiveInt = Brand<number, "PositiveInt">;
```
Confirms (Ch. 3.2/9.2) the brand's marker property blocks any value not routed through the smart constructor, even though structurally each is just a primitive.

### 4. Event handler prop derivation
```ts
type HandlerProps<T> = { [K in keyof T as `on${Capitalize<string & K>}`]: (e: T[K]) => void };
```
**Why:** identical mechanism to `Getters<T>` (Ch. 7.5/9.1 combined) — `Capitalize` plus template literal key renaming, with each event's specific type flowing through via `T[K]`.

---

# Chapter 10 — Utility Types

### 1. Full reimplementation (remaining ones)
```ts
type MyPartial<T> = { [K in keyof T]?: T[K] };
type MyRequired<T> = { [K in keyof T]-?: T[K] };
type MyReadonly<T> = { readonly [K in keyof T]: T[K] };
type MyRecord<K extends keyof any, V> = { [P in K]: V };
type MyAwaited<T> = T extends Promise<infer U> ? MyAwaited<U> : T;
```
**Why `Awaited` must be recursive:** real `await` fully unwraps nested promises (Ch. 8.4/10.6) — a single-level version would leave `Promise<Promise<T>>` half-unwrapped.

### 2. The `Omit`-on-unions trap
```ts
type Shape = { kind: "circle"; radius: number } | { kind: "square"; side: number };
type Broken = Omit<Shape, "kind">; // { radius: number; side: number } — WRONG
type DistributiveOmit<T, K extends keyof any> = T extends unknown ? Omit<T, K> : never;
type Fixed = DistributiveOmit<Shape, "kind">; // { radius: number } | { side: number }
```
**Why:** `Omit` relies on `keyof T`, and `keyof` on a union is an *intersection* of keys (Ch. 7.1/10.2) — forcing distribution first processes each member independently before `Omit` runs.

### 3. Generic `logged` decorator
```ts
function logged<F extends (...args: any[]) => any>(fn: F): (...args: Parameters<F>) => ReturnType<F> {
  return (...args: Parameters<F>): ReturnType<F> => {
    const result = fn(...args);
    return result;
  };
}
```
**Why:** `Parameters<F>`/`ReturnType<F>` (Ch. 10.4) derive the wrapper's signature from whatever function is passed in, for any arity, with zero manual overloads.

### 4. `NoInfer` audit
```ts
function setupRolesFixed<R extends string>(roles: R[], defaultRole?: NoInfer<R>) {}
```
**Why:** without `NoInfer` (Ch. 10.8), every argument position referencing `R` contributes to inferring it, silently widening; `NoInfer` excludes that parameter from inference.

### 5. `AsyncReturnType<F> = Awaited<ReturnType<F>>`
**Why:** `ReturnType<F>` alone gives `Promise<{...}>`; composing with `Awaited` (Ch. 8.4/10.6) unwraps the promise layer — a direct demonstration of utility-type composition.

---

# Chapter 11 — Modules & Declarations

### 1. Ambient global drill
```ts
// globals.d.ts — no top-level import/export
declare const __BUILD_TIME__: string;
// assets.d.ts
declare module "*.svg" { const content: string; export default content; }
```
**Why `globals.d.ts` must have no import/export:** adding one turns it from a global script into a module (Ch. 11.4), scoping the declaration incorrectly.

### 2. Express augmentation, verified
```ts
import "express";
declare module "express" { interface Request { user?: { id: string }; } }
```
Confirmed globally without re-importing the augmentation file anywhere, because merging happens at the binder level program-wide (Ch. 5.2/11.5).

### 3. Module resolution audit
- Pure ESM Node 20 backend: `NodeNext`/`NodeNext`.
- Dual CJS/ESM library: `package.json` `"exports"` conditional map with separate typed builds per format (Ch. 11.1's edge case).
- Vite frontend: `Bundler`/`ESNext`, `noEmit: true`.

### 4. Diagnose a broken augmentation
Omit the anchoring `import "express";` and mistarget the module string — compiles cleanly, `req.user` never appears anywhere (Ch. 11.5). Fix: match the library's own internal module specifier exactly, add the anchoring import.

---

# Chapter 12 — Decorators, Metadata, Mixins, Merging

### 1. Decorator system audit
`experimentalDecorators: true` in `tsconfig` + `(target, key, descriptor)` signature → legacy; `(originalMethod, context)` with `ClassMethodDecoratorContext` → TC39 Stage 3 (Ch. 12.1). Presence of `reflect-metadata` confirms legacy, since TC39 has no equivalent (Ch. 12.2).

### 2. Logging decorator, once-per-definition proof
A shared closure variable (e.g., `callCount`) incremented across multiple instances' method calls proves the decorator itself ran once, at class-definition time, not per instance (Ch. 12.1).

### 3. Mixin composition, `instanceof` verified
```ts
type Constructor<T = {}> = new (...args: any[]) => T;
class Full extends Disposable(Timestamped(Serializable(Base))) {}
```
`instanceof Base` succeeds because each mixin genuinely `extends` its input at runtime (Ch. 12.3), forming a real prototype chain.

### 4. Declaration merging showcase
Namespace+function and namespace+enum merges (Ch. 12.4) — prefer real `static` class members for new code; reserve this pattern for legacy-compatible or non-class-extendable targets.

---

# Chapter 13 — Inference & Control Flow Internals

### 1. CFA trace-through
Tracking a variable's type at each labeled point in nested `if`/`typeof` checks confirms the checker intersects the declared type with every guard provably true along every path reaching that point (Ch. 13.2).

### 2. Closure-narrowing bug, fixed two ways
```ts
const narrowed = x; // Fix 1: copy to const
setTimeout(() => { if (x !== null) x.toString(); }, 1000); // Fix 2: re-narrow inside
```
**Why the original fails:** the checker can't prove `x` wasn't reassigned between closure creation and invocation (Ch. 13.2).

### 3. Reusable `assertNever`
```ts
function assertNever(x: never): never { throw new Error(`Unexpected value: ${JSON.stringify(x)}`); }
```
Equivalent to but cleaner than the inline `never`-assignment pattern (Ch. 13.4).

### 4. Unsound guard hunt
```ts
function isString(x: unknown): x is string { return x !== null; } // WRONG
```
Compiles because the checker trusts the `is` claim without verifying the body (Ch. 13.3) — a genuine, silent unsoundness hole.

### 5. Loop narrowing edge case
A `while`-condition narrowing re-checked only at loop-entry, combined with an in-body reassignment, demonstrates why the checker can't assume the same narrowing holds mid-iteration after a reassignment (Ch. 13.2).

---

# Chapter 14 — Architecture & Best Practices

### 1. `satisfies` migration
```ts
const routesFixed = { home: {path:"/"}, admin: {path:"/admin", auth:true} } satisfies Record<string, {path:string; auth?:boolean}>;
```
Precision recovered: each entry keeps its own narrower inferred type instead of collapsing to the general shape (Ch. 14.1).

### 2. `Result<T,E>` refactor
Converting a throwing validator to return `Result<T,E>` forces every caller to explicitly branch on success/failure (Ch. 14.2), unlike an invisible-to-the-type-system thrown exception.

### 3. Boundary audit
```ts
const raw: unknown = await res.json();
if (!isValidUser(raw)) return err("Malformed user response");
```
Typing the raw response as `unknown` forces genuine runtime validation before trust (Ch. 14.3), unlike a bare `as User` assertion.

### 4. Illegal-state redesign
A boolean-flag form permitting `isSaving && isEditing` simultaneously redesigned as a `mode`-discriminated union removes that possibility structurally (Ch. 2.9/14.3).

### 5. Architecture review
Prioritized: (1) unify strictness via a shared base config, (2) add `composite`/`references`, (3) `skipLibCheck: true`, (4) audit `any` leakage at package boundaries specifically (Ch. 14.4).

---

# Chapter 15 — Framework Integration

### 1. Generic `<Table<T>>`
`T` links `data`, each `Column<T>`'s `render`, and `keyExtractor` together (Ch. 6.1/15.1) — inferred entirely from the `data` argument.

### 2. Express auth middleware, both ways
Scoped `AuthenticatedRequest extends Request` when only some routes are authenticated (visible distinction per handler); global module augmentation when `user` is a universal, always-possibly-present concern (Ch. 11.5/15.2).

### 3. NestJS injection token
```ts
const NOTIFICATION_SERVICE = Symbol("NOTIFICATION_SERVICE");
constructor(@Inject(NOTIFICATION_SERVICE) private notifier: NotificationService) {}
```
Required because `NotificationService` is an erased interface with nothing for `reflect-metadata` to capture (Ch. 1.2/12.2).

### 4. Library API design review
Apply infer-first design, minimal structural constraints, a deliberate `interface`-vs-`type` choice per exported shape, and a type-only smoke-test file exercising both valid and intentionally-invalid usage (Ch. 15.5).

---

## How to Use This Reference

Don't just read the answers — for each exercise, attempt it cold first, then compare. Where your answer differs from the one here, the useful question isn't "was I right" but **"which mechanism explains the difference"** — trace it back to one of the six primitives from the Chapter 16 mental model (assignability, variance, inference/CFA, composition, type-level introspection/transformation/branching, or the deliberate-unsoundness list). If you can always answer that follow-up, you have the depth this course was built to give you.
