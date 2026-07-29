# Chapter 11 — Modules & Declarations

## 11.1 ES Modules in TypeScript

**Intuition.** TS's `import`/`export` is (mostly) just typed ES modules — but TS also needs to know, at compile time, where to find the *types* for whatever you're importing, which is a genuinely separate problem from where Node/the browser finds the *runtime* code.

**Technical explanation.** Two things happen for every `import`: (1) **module resolution** — the compiler locates a matching `.ts`/`.d.ts` file (or a `package.json` `types`/`exports` field) to pull type information from; (2) **emit** — the import statement itself gets rewritten into whatever the target `module` setting requires (`CommonJS` `require()`, or left as `import` for `ESNext`/`NodeNext`).

**Internal behavior — `import type` / `export type`.** Because TS types are erased (Ch. 1.2), an import that's *only* used for types shouldn't produce any runtime `require`/`import` at all — the compiler's emitter normally figures this out automatically (elides type-only imports), but this inference can fail in ambiguous cases, especially under `isolatedModules` (Ch. 1.4), where each file is transpiled independently with no cross-file knowledge of whether an imported name is a type or a value. `import type { Foo } from "./foo"` / `export type { Foo }` make the type-only intent **explicit**, guaranteeing correct elision regardless of transpiler.
```ts
import type { User } from "./types";      // guaranteed erased, zero runtime import
import { type User, fetchUser } from "./api"; // inline type-only specifier, mixed with a value import
```

**Compiler behavior — `moduleResolution` strategies.**
- `Node10` (legacy, was called `"node"`): mimics old CommonJS `require()` resolution — extension-less imports, `index.js` folder resolution, no `exports` field awareness.
- `Node16`/`NodeNext`: mirrors modern Node's actual dual ESM/CJS resolution algorithm, respects `package.json` `"exports"` maps and requires explicit file extensions in relative ESM imports (`./foo.js`, even when importing from a `.ts` file — a famous, frequently-confusing modern-Node-ESM requirement).
- `Bundler`: designed for bundler-based toolchains (Vite, esbuild, webpack) that have their own resolution logic more permissive than Node's — allows extension-less imports and `package.json` conditions similar to bundlers' actual behavior, without requiring the strict Node ESM extension rules.

**Real-world use cases.** Choosing `moduleResolution: "NodeNext"` for a real Node.js backend that ships both/either ESM and CJS; choosing `"Bundler"` for frontend apps built with Vite/webpack where the bundler (not `tsc`) does the actual resolution/bundling.

**Good practices.** Match `moduleResolution` to your actual runtime/bundler, not just "whatever makes red squiggles go away" — a mismatched setting can hide real resolution bugs that only surface at actual runtime/build time. Use explicit `import type` for type-only imports in shared/library code to guarantee correct behavior regardless of the consumer's transpiler.
**Bad practices.** Leaving `moduleResolution` on a legacy/default setting while actually deploying to modern Node with `"type": "module"` — a very common real-world source of "it type-checks fine but crashes at runtime with a resolution error."
**Common mistakes.** Forgetting the `.js` extension requirement in relative imports under `NodeNext`/`Node16` even though you're importing from `.ts` source files — this is intentional (matches what the *emitted* `.js` file will actually need to resolve at runtime) and trips up almost everyone migrating an existing Node project to native ESM.
**Edge case.** A single package can ship **dual CJS/ESM builds** with different type definitions per format via `package.json`'s `"exports"` conditional map (separate `"import"`/`"require"` entries each pointing at different `.d.ts` files) — genuinely advanced, real-world library-authoring knowledge, and a frequent source of "why do the types look different depending on how this package is imported" bugs when done incorrectly.
**Performance.** Module resolution itself is usually fast, but misconfigured resolution (e.g., accidentally resolving into a huge `node_modules` type-checking pass) combined with `skipLibCheck: false` can be a real, measurable build-time cost.

### Interview Q&A — §11.1
**Junior:** Q: What does `import type { User } from "./types"` do differently from a normal `import`? A: It explicitly marks the import as type-only, guaranteeing the compiler erases it completely at emit time with no runtime `require`/`import` produced, regardless of how the transpiler analyzes the rest of the file.
**Mid:** Q: Why might `import type` matter more under `isolatedModules`? A: Single-file transpilers (Babel, esbuild, swc) can't see across files to determine whether an imported name is a type or a value, so they rely on explicit signals; without `import type`, an ambiguous import might either be incorrectly left in (harmless but wasteful) or, in genuinely ambiguous cases, mishandled — explicit `import type` removes the ambiguity entirely.
**Senior:** Q: Why do relative imports need explicit `.js` extensions under `NodeNext` moduleResolution even in `.ts` source files? A: TS models Node's actual ESM resolution algorithm, which requires explicit extensions for relative specifiers (unlike CommonJS's implicit extension-guessing) — since the `.ts` file will be emitted as a `.js` file, the import specifier must already reference the *emitted* filename's extension (`.js`) so the real runtime resolution succeeds; TS enforces this at compile time specifically to prevent a category of "works in the type checker, breaks at actual runtime" bug.

---

## 11.2 Namespaces

**Intuition.** Namespaces are TypeScript's **pre-ES-modules** answer to organizing code into logical groups — largely a legacy feature now that ES modules are universal, but still load-bearing for a few specific things (declaring the shape of non-modular global libraries, and merging into classes/functions/enums, Ch. 5.2).

**Technical explanation.**
```ts
namespace Geometry {
  export interface Point { x: number; y: number; }
  export function distance(a: Point, b: Point): number { /* ... */ return 0; }
}
const p: Geometry.Point = { x: 0, y: 0 };
```
Compiles (pre-modern targets) to a real runtime IIFE-wrapped object, similar in spirit to the regular-`enum` emit pattern from Ch. 2.6 — another one of the rare non-fully-erased TS constructs.

**Compiler behavior.** Namespaces can span multiple files via triple-slash directives (`/// <reference path="..." />`) — a pre-ES-modules file-linking mechanism, now essentially obsolete for application code but still occasionally seen in older/`.d.ts`-only codebases.

**Real-world use cases (modern, legitimate).** Declaring types for old-style global (non-module, script-tag-loaded) JavaScript libraries in ambient `.d.ts` files, where the library itself attaches things to the global scope rather than exporting via ES modules — namespaces model this global, nested structure naturally. Also used for the namespace+class/function/enum merging patterns from Ch. 5.2 (e.g., adding static-like helper types alongside a class).

**Good practices.** For all new, module-based application code, use ES modules (files + `import`/`export`), not namespaces, for code organization — namespaces for *application* code organization is considered outdated/discouraged by the TS team itself. Reserve namespaces for their legitimate remaining niches: ambient global library typings and declaration-merging-based augmentation patterns.
**Bad practices.** Introducing namespaces in a modern, ES-module-based codebase purely out of unfamiliarity with modules, or mixing namespaces and ES modules within the same file in confusing ways (generally discouraged, can produce surprising interactions).
**Common mistakes.** Confusing a namespace's `export` (scoped to the namespace) with a module's top-level `export` (scoped to the file) — they look similar syntactically but mean different things structurally.
**Edge case.** `declare global { ... }` (Ch. 5.2's Window-augmentation example) is technically a form of ambient (non-namespaced) global augmentation, distinct from `namespace`, but the two are historically/conceptually related — both deal with declaring things outside the normal module-scoped system.
**Performance.** Emitted namespace IIFE code adds real (if small) runtime bytes, similar to regular enums (Ch. 2.6) — a minor but real consideration for bundle-size-sensitive contexts, and one more reason modern application code prefers ES modules (zero namespace-wrapper overhead).

### Interview Q&A — §11.2
**Junior:** Q: Are namespaces still recommended for organizing modern TypeScript application code? A: No — ES modules (separate files with `import`/`export`) are the modern standard; namespaces are a largely legacy feature retained mainly for ambient global library typing and specific declaration-merging scenarios.
**Mid:** Q: Give one legitimate modern use case for `namespace`. A: Typing an old-style global JavaScript library that attaches a nested object structure to the global scope (rather than using ES module exports) — an ambient `.d.ts` file can model that structure naturally with namespaces, matching how the library actually behaves at runtime.
**Senior:** Q: Why do namespaces, unlike interfaces/type aliases, produce real runtime JavaScript output? A: Because a namespace can contain actual runtime values (functions, variables, nested namespaces), not just type declarations — the compiler must emit a real object (via an IIFE pattern) to hold those runtime members and support the dot-access syntax (`Geometry.distance(...)`) at runtime, exactly analogous to why regular (non-const) `enum`s also produce real runtime objects (Ch. 2.6) — anything with genuine runtime members can't be fully erased.

---

## 11.3 Declaration Files (`.d.ts`)

**Intuition.** A `.d.ts` file contains **only type information, zero implementation** — it's a pure "here's the shape of this code" contract, used either to describe a JS library that has no types of its own, or as the published type-surface of your own compiled TS library.

**Technical explanation.**
```ts
// math-lib.d.ts
export function add(a: number, b: number): number;
export function subtract(a: number, b: number): number;
declare const VERSION: string;
export { VERSION };
```
No function bodies, no implementation — just signatures. The `declare` keyword (§11.4) marks something as "exists elsewhere at runtime, trust me, just here's its type."

**Internal behavior.** When `declaration: true` (Ch. 1.6) is set, `tsc` **generates** `.d.ts` files automatically from your real `.ts` source as part of emit — this is how published TS libraries typically produce their public type surface: you write real `.ts`, the compiler strips implementation and keeps signatures for the `.d.ts` output, which ships alongside the compiled `.js` in the published package.

**Compiler behavior / real-world use cases.**
- **Auto-generated** (`declaration: true`): the standard way a TS library publishes types for its own compiled JS.
- **Hand-written, for an existing untyped JS library**: when consuming a plain-JS package with no types, you write (or install from **DefinitelyTyped**, the massive community-maintained `@types/*` package repository) a `.d.ts` describing its actual shape.
- **`package.json` `"types"`/`"typings"` field** points consumers' compilers at the right `.d.ts` entry point for a published package.

**Good practices.** For your own libraries, prefer auto-generated `.d.ts` (via `declaration: true`) over hand-writing/maintaining them separately — guarantees they never drift from the real implementation. When consuming an untyped JS library, check DefinitelyTyped (`npm install -D @types/some-lib`) before hand-writing your own ambient declarations.
**Bad practices.** Hand-maintaining a `.d.ts` file in parallel with real `.ts` implementation files for your *own* code (rather than auto-generating) — near-guaranteed to drift out of sync over time.
**Common mistakes.** Forgetting `.d.ts` files can't contain any actual runtime implementation code (no function bodies, no executable statements) — attempting to include real logic in a `.d.ts` is simply invalid.
**Edge case.** `declarationMap: true` (Ch. 1.6) generates source maps linking generated `.d.ts` files back to the original `.ts` source, so "go to definition" in an editor lands you in the real, readable source rather than the flattened declaration file — a meaningful DX improvement for consumers of your published library working in a monorepo.
**Performance.** Generating `.d.ts` output adds real (usually modest) time to the build; consuming a package's `.d.ts` files during type-checking is subject to the same `skipLibCheck` considerations as any other dependency type (Ch. 1.6).

### Interview Q&A — §11.3
**Junior:** Q: What goes inside a `.d.ts` file? A: Only type declarations/signatures — no actual implementation code (no function bodies, no runnable statements).
**Mid:** Q: How do you get TypeScript to consume a plain JavaScript library that has no types of its own? A: Install community-maintained types from DefinitelyTyped (`@types/package-name`) if available, or hand-write a `.d.ts` describing the library's actual runtime shape if not.
**Senior:** Q: Why is auto-generating `.d.ts` from real `.ts` source (via `declaration: true`) strongly preferred over hand-writing/maintaining separate `.d.ts` files for your own library's types? A: Hand-maintained, parallel `.d.ts` files have no compiler-enforced connection to the actual implementation — any change to the real code's signatures requires a human to remember to update the separate declaration file, which reliably drifts out of sync over time in any actively-developed codebase; auto-generation makes the published type surface a direct, compiler-verified derivative of the real source, eliminating that entire class of bug.

---

## 11.4 Ambient Declarations

**Intuition.** "Ambient" means "declared to exist, without providing an implementation here" — you're telling the compiler about something that's real at runtime (a global variable, a non-JS asset import, a library loaded via `<script>`) but whose actual definition lives somewhere the compiler can't see directly.

**Technical explanation.**
```ts
declare const __VERSION__: string;              // a global injected by a bundler at build time
declare function gtag(...args: any[]): void;      // a script-tag-loaded global analytics function
declare module "*.svg" {                            // non-JS asset imports (bundler-specific)
  const content: string;
  export default content;
}
declare global {                                    // augmenting the actual global scope
  interface Window { myGlobalFlag: boolean; }
}
```

**Internal behavior.** `declare` tells the checker "trust that this exists and has this shape at runtime — don't require an implementation, and don't emit anything for this declaration itself." Ambient declarations produce **zero runtime output** on their own; they're pure compile-time type information layered on top of something that exists through some other mechanism (a bundler's global injection, a script tag, a webpack asset loader).

**Compiler behavior — ambient module wildcard patterns.** `declare module "*.svg"` uses a wildcard to match *any* import path ending in `.svg`, letting bundler-processed non-JS asset imports (common in frontend build tooling) type-check without TS needing to understand SVG files at all — it's purely a type-level stand-in for what the bundler actually does at build/runtime.

**Real-world use cases.** Typing bundler-injected build-time globals (`__VERSION__`, `process.env.NODE_ENV` patterns), typing non-JS asset imports (`.svg`, `.css`, `.png` module declarations for webpack/Vite-style asset imports), typing third-party globals loaded via `<script>` tags (analytics snippets, ad SDKs), global test-runner globals (`describe`/`it`/`expect` when not imported explicitly, via `@types/jest`/`@types/mocha`-style ambient declaration packages).

**Good practices.** Keep ambient declarations centralized in a small number of clearly-named files (e.g., `globals.d.ts`, `assets.d.ts`) rather than scattered inline throughout application code — makes the "what's actually ambient/implicit here" surface easy to audit.
**Bad practices.** Using `declare` as an escape hatch to silence a legitimate "cannot find module"/"cannot find name" error without actually verifying the thing genuinely exists at runtime with the shape you've declared — this is a pure, unchecked promise to the compiler; if it's wrong, you get no safety at all and a runtime crash instead.
**Common mistakes.** Forgetting ambient `.d.ts` files (not written as ES modules — no top-level `import`/`export`) are treated as **global scope** declarations by default, while a `.d.ts` file that *does* contain a top-level `import`/`export` becomes a module and its declarations are scoped to that module, not global — accidentally adding an unrelated `import` to what was meant to be a global ambient file can silently change its scoping behavior, a genuinely common, confusing gotcha.
**Edge case.** `declare global { ... }` specifically requires being inside a file that's *already* a module (has some top-level `import`/`export`) — using it in a genuinely global (non-module) ambient file is unnecessary/invalid, since such a file is already implicitly global; this inverse relationship (needing `declare global` specifically *because* your file is a module, to "reach out" to the global scope from within module scope) is a frequently-confusing detail worth internalizing precisely.
**Performance.** Negligible — pure compile-time constructs with no runtime footprint of their own.

### Interview Q&A — §11.4
**Junior:** Q: What does `declare const API_URL: string;` mean, with no assigned value? A: It tells the compiler that a value named `API_URL` of type `string` exists at runtime (injected by some external mechanism, e.g. a bundler), without providing or requiring an actual implementation in this file — purely a type-level promise.
**Mid:** Q: Why does a `.d.ts` file with no top-level `import`/`export` behave differently (in terms of scoping) from one that has them? A: A file with no top-level `import`/`export` is treated as a **script**, and its declarations are automatically global; a file with any top-level `import`/`export` becomes a **module**, and its declarations are scoped to that module by default — requiring an explicit `declare global { ... }` block if you specifically want to reach out and augment the true global scope from within a module file.
**Senior:** Q: What's the real risk of using `declare` to silence a "cannot find name" error, and how would you responsibly verify it's safe? A: `declare` is an entirely unchecked assertion — the compiler takes your word for it with zero verification, so if the declared shape doesn't actually match what exists at runtime (wrong type, wrong existence at all, e.g. a global only injected in some environments/build configs), you get silent type-checking success and a genuine runtime failure with no compiler safety net; before adding a `declare`, verify (through documentation, runtime logging, or the actual injecting mechanism's source) that the declared name and shape are accurate and consistently present in every environment the code will actually run in.

---

## 11.5 Module Augmentation (deep dive, building on Ch. 5.2)

**Intuition.** Module augmentation is declaration merging (Ch. 5.2) specifically applied *across module boundaries* — reaching into a library's own module-scoped types from your own code and adding to them, without editing the library's source at all.

**Technical explanation.**
```ts
// your-app/types/express-augment.d.ts
import "express"; // ensures this file is treated as touching that module, not just declaring a new one
declare module "express" {
  interface Request {
    user?: { id: string; role: string };
  }
}
```
Now every `Request` type throughout your application (including inside Express's own middleware type signatures) sees the combined shape — `req.user` is available and correctly typed everywhere, with zero changes to Express's actual source.

**Internal behavior — why the module specifier must match exactly.** The `declare module "express"` string must resolve to the **exact same module** the library itself uses internally to declare `Request` — if your ambient file's module path doesn't correctly resolve to that same underlying module (e.g., due to a subtly different package resolution path, or targeting a re-exporting wrapper module instead of the true source), TS creates an entirely new, unrelated ambient module declaration instead of merging with the real one — your augmentation silently does nothing where you expected it, and you won't necessarily get an error, just missing functionality. This is one of the most genuinely confusing real-world TS gotchas, and understanding *why* it happens (declaration merging is fundamentally Symbol-based — Ch. 5.2 — and requires resolving to the literal same Symbol) is a strong senior-level signal.

**Compiler behavior.** For augmentation to correctly target a specific existing module (rather than accidentally creating a new global ambient module), the augmenting file typically needs at least one top-level `import`/`export` of its own (making it a module in TS's own scoping model, §11.4) — a bare `declare module "express" {...}` with no surrounding module context can behave differently than one nested inside a proper module file; the common, robust convention is exactly the `import "express";` side-effect import shown above, which both establishes module context and confirms/anchors the resolution target.

**Real-world use cases.** Adding custom properties to Express's `Request`/`Response` (auth middleware, request-scoped context), extending Redux's `Store` types, adding custom Jest matcher types (`expect.extend` + a matching `declare global { namespace jest { interface Matchers<R> { ... } } }` augmentation — a very common real pattern), extending React's built-in `JSX.IntrinsicElements` for custom elements/web components.

**Good practices.** Centralize module augmentations in clearly-named files (`types/express.d.ts`, `types/jest.d.ts`) and include them in your `tsconfig`'s `include`/`files` explicitly if they're not naturally picked up — a common real-world setup mistake is an augmentation file that's technically correct but simply never gets loaded into the program because it's outside the configured `include` globs (tying back to Ch. 1.5's "which files are actually in the program" discussion).
**Bad practices.** Silently duplicating/redefining a property that might conflict with a *future* version of the library's own types (an unmanaged upgrade risk) — document augmentations clearly and revisit them when upgrading the augmented library's major version.
**Common mistakes.** Omitting the anchoring `import "the-module";` (or any top-level import/export) in an augmentation file and being confused why the augmentation "compiles but doesn't seem to apply anywhere."
**Edge case.** You can also augment your **own** first-party modules from a different file in the same project (not just third-party libraries) — occasionally useful for plugin-style architectures where a core module's type needs to be extended by separately-loaded feature modules, though for first-party code, simply editing the original interface directly is usually clearer when it's available to you.
**Performance.** No special cost beyond ordinary declaration merging (Ch. 5.2).

### Interview Q&A — §11.5
**Junior:** Q: What is module augmentation used for? A: Adding new properties/members to a type defined in a third-party (or separate first-party) module, without modifying that module's own source — commonly used to add custom properties like `req.user` to Express's `Request` type.
**Mid:** Q: Why does a module augmentation file often need a top-level `import "the-target-module";` even if nothing from it is actually used? A: It's a "side-effect import" that establishes the file as a module (rather than a global script) and anchors the `declare module "the-target-module" {...}` block to correctly target and merge with that library's real, already-existing module declaration, rather than accidentally creating a new, unrelated ambient module of the same name.
**Senior/FAANG:** Q: Why can module augmentation silently "not work" — compiling with no errors but the augmented properties never actually appearing where expected — and how would you diagnose it? A: Declaration merging is fundamentally based on resolving to the exact same underlying Symbol (Ch. 5.2); if the augmenting `declare module "..."` string doesn't resolve through TS's module resolution to the literal same module the library itself uses internally (e.g., resolving through a different path, a re-export barrel, or a mismatched package version), TS creates a brand-new, separate ambient module declaration under that name instead of merging into the real one — this compiles cleanly (it's valid syntax) but has no effect anywhere the library's real types are actually used. Diagnosing it typically means verifying the exact resolved module path (e.g., checking the library's own `.d.ts` module declaration string matches character-for-character) and confirming the augmentation file is actually included in the program (Ch. 1.5).

---

## Practical Exercises — Chapter 11

1. **Ambient global drill:** Write ambient declarations for a hypothetical bundler-injected `__BUILD_TIME__: string` global and a `*.svg` asset module, and use both in a small sample file.
2. **Express augmentation (or equivalent):** Practice the full `declare module "express" { interface Request { ... } }` pattern (or the equivalent for any library you use), including the anchoring import, and verify it applies correctly across your codebase.
3. **Module resolution audit:** For a real (or hypothetical) Node.js project, decide the correct `module`/`moduleResolution` pairing for (a) a pure ESM Node 20 backend, (b) a dual CJS/ESM published library, (c) a Vite-bundled frontend app — justify each.
4. **Diagnose a broken augmentation:** Deliberately misconfigure a module augmentation's target path so it silently fails to merge, observe the "no error but no effect" symptom, then fix it — document exactly what you changed.

---

**Next:** Chapter 12 — Compiler-Level Features: decorators, metadata, mixins, and a full declaration-merging deep dive. Say **"next"** to continue.
