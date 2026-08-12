---
layout: default
title: "Chapter 1 — Foundations"
---

# Chapter 1 — Foundations

## 1.1 Why TypeScript Exists

**Intuition.** JavaScript was designed in 10 days for tiny scripts in browsers. It scaled to run entire banks, airlines, and OSes (VS Code, Slack, Node backends) anyway. Dynamic typing is great for a 50-line script and a liability at 500,000 lines with 40 engineers, because the only way to know if `user.name` exists is to run the code and see. TypeScript exists to move that discovery from **runtime** (in production, in front of a user) to **compile time** (in your editor, before you even save).

**Technical explanation.** TypeScript is a **superset** of JavaScript that adds a static, structural, gradual type system. "Superset" means: every valid `.js` file is (almost) valid `.ts` — TS adds syntax and checking on top, it doesn't remove JS semantics. It compiles ("transpiles") down to plain JavaScript that runs anywhere JS runs.

**Internal behavior.** Under the hood, TS is a pipeline:
```
Source (.ts) → Scanner (tokens) → Parser (AST) → Binder (symbols/scopes)
            → Checker (type analysis) → Emitter (.js output)
```
The **Checker** is where "TypeScript-ness" lives — it's ~40k+ lines of code that walks the AST built by the parser and computes/validates types. Critically: **the emitter does not need the checker to succeed.** Type errors are *diagnostics*, not blockers, by default — TS will still emit JS even with type errors (unless you set `noEmitOnError`).

**Compiler behavior.** `tsc` performs two largely independent jobs:
1. Type-check (structural analysis, produces diagnostics)
2. Emit (strip types, downlevel syntax, produce `.js`)

This split is why TS type errors don't stop your app from running in dev tools like `ts-node` (with transpile-only) or `esbuild`/`swc` — those tools skip the checker entirely and just do step 2, because type erasure is a purely syntactic transformation.

**Type inference role here:** N/A directly, but this sets up *why* inference matters — TS was designed so you rarely need to annotate everything; the checker infers types from initializers, so gradual adoption (`// @ts-check` on plain JS, `any` escape hatches) is possible.

**Real-world use cases.** Large teams, public APIs/libraries (types are documentation that can't go stale), refactoring safety nets, IDE tooling (autocomplete, "find all references", safe renames) — all powered by the same checker.

**Good practices.** Treat TS as a design tool, not just a linter — model your domain with types first, then implement.
**Bad practices.** Sprinkling `any` to silence errors instead of modeling the actual shape.
**Common mistake.** Believing TS types exist at runtime. They don't — see erasure below.
**Edge case.** TS *can* affect runtime semantics minimally (e.g., `enum` emits real JS objects, parameter property shorthand in classes emits assignments) — these are the few non-erased constructs, covered in Ch. 4/5.
**Performance implication.** Type-checking large codebases is CPU/memory heavy; project references (`composite`, `incremental`) exist specifically to cache and parallelize this (covered §1.5).

---

## 1.2 Relationship With JavaScript

**Intuition.** Think of TS as JS wearing a harness during a workout (development) that gets removed before the performance (production). The harness catches mistakes; it's never part of the actual show.

**Technical explanation — Type Erasure.** TypeScript types are **entirely erased** at compile time. There is no TS runtime. `interface`, `type`, generic parameters `<T>`, type annotations `: string` — none of it exists in the emitted `.js`. This is fundamentally different from languages like Java/C# where types affect runtime dispatch, or from Flow/Reason which have partial runtime representations in some modes.

```ts
function greet(name: string): string {
  return `Hello ${name}`;
}
```
compiles to:
```js
function greet(name) {
  return `Hello ${name}`;
}
```
Nothing else changed. This is the single most important mental model in TS: **TS = JS + a type-checking phase that vanishes.**

**Internal behavior.** Because of erasure:
- You **cannot** do `if (x instanceof SomeInterface)` — interfaces don't exist at runtime.
- You **cannot** do `typeof T === 'string'` inside a generic function — `T` is gone by runtime.
- Runtime type checks must use JS-native mechanisms: `typeof`, `instanceof` (real classes), `in`, custom type guards, or runtime validation libraries (Zod, io-ts) that TS can *infer types from* but that do the actual runtime work themselves.

**Compiler behavior.** `tsc` (or Babel's TS plugin, or `esbuild`/`swc`) strips type-only syntax as a **syntactic** transform. Babel's TS plugin, notably, does *no type-checking at all* — it just deletes anything that looks like a type. This is why Babel can't catch type errors and can't understand some TS features requiring type info (e.g., `const enum` inlining, which needs the checker).

**Type inference:** JS gives TS its raw material — TS infers types by watching JS values flow (literals, initializers, return statements) rather than requiring you to write Java-style declarations everywhere.

**Real-world use cases.** Gradual migration of JS codebases (`allowJs` + `checkJs`), publishing libraries usable from both TS and plain JS consumers, using any JS library via type declarations.

**Good practices.** Write TS as if you're writing better-documented JS, not a different language — idioms (closures, prototypes, `this`) are unchanged.
**Bad practices.** Trying to use types for runtime logic (`if (typeof T === 'number')` inside generics — doesn't compile as intended, `T` isn't a value).
**Common mistakes.**
- Assuming an interface can be used with `instanceof`.
- Assuming a type annotation *validates* external data (e.g., `JSON.parse(x) as User` — this is a **lie you tell the compiler**, not a check).
**Edge case.** `class` is the one place JS and TS overlap at runtime — a TS `class` declares both a type (the shape) *and* a real JS value (the constructor function), so `instanceof MyClass` **does** work, unlike `interface`.
**Performance.** Zero runtime cost from types themselves — this is a major reason TS adoption has near-zero runtime overhead vs plain JS.

### Interview Q&A — §1.1/1.2

**Junior**
- Q: What is TypeScript? A: A statically-typed superset of JavaScript that compiles to plain JS, adding compile-time type checking.
- Q: Does TypeScript run in the browser? A: No — browsers run JS. TS is compiled/transpiled to JS first.

**Mid**
- Q: If TS has a type error, will it still produce JS output? A: Yes, by default (`noEmitOnError: false`). Type-checking and emitting are separate phases.
- Q: Can you use `instanceof` with a TS `interface`? A: No — interfaces are erased; only real runtime constructs (classes, functions) work with `instanceof`.

**Senior**
- Q: Why can tools like esbuild/swc compile TS so much faster than `tsc`? A: They perform only syntactic type-stripping (erasure) with no semantic analysis — no symbol resolution, no type checking. `tsc` runs the full checker (binding, control-flow analysis, structural comparison), which is algorithmically far more expensive.
- Q: Explain why `const enum` cannot be used with `isolatedModules: true` or Babel. A: `const enum` requires the compiler to know all enum member values at compile time to inline them — a whole-program, type-aware transform. Single-file transpilers (Babel, isolatedModules-compatible tools) process files independently without cross-file type info, so they can't safely inline it.

**FAANG-style / tricky**
- Q: "TypeScript's type system is Turing-complete." Is this actually true, and why would an interviewer ask this? A: Yes, via conditional types + recursive type aliases you can compute (people have implemented JSON parsers, regex engines in TS types). It matters because it explains *why* type-level infinite loops/`Excessive stack depth` errors are possible, and why the checker imposes recursion depth limits.
- Q: Two modules both declare `interface Window { foo: string }`. What happens, and why is this *not* a TS bug? A: Declaration merging — TS interfaces support merging by design (unlike `type`), so both declarations combine into one interface. This is intentional (used heavily for ambient global augmentation, e.g. extending `window`).

**Predict the inferred type**
```ts
const a = "hello";       // type: "hello" (literal), not string — const + no annotation
let b = "hello";         // type: string — let widens
const c = { x: 1 };      // type: { x: number } — object literal props widen number literals but not the shape
function f(x = 5) { return x; } // param x: number, return type: number
```

**Debugging challenge**
```ts
interface User { id: number; }
const raw: any = JSON.parse('{"id": "not-a-number"}');
const u = raw as User;
console.log(u.id.toFixed(2)); // crashes at runtime — why does TS not catch this?
```
*Answer:* `as` is a type **assertion**, not a conversion or validation — you're telling the compiler "trust me," bypassing structural checking. `any` also disables checking on `raw` entirely. Fix: validate with a runtime schema (Zod/io-ts) before trusting the shape.

---

## 1.3 The TypeScript Compiler (`tsc`)

**Intuition.** `tsc` is a program that reads your `.ts` files, builds a full mental model of every type in your program (like a spreadsheet cross-referencing every cell), checks that model for contradictions, then produces `.js` with the type "annotations" scrubbed off.

**Technical / internal behavior — the real pipeline:**

1. **Scanner (Lexer).** Converts raw text into a token stream (`const`, identifier `x`, `=`, number `5`, `;`...).
2. **Parser.** Consumes tokens, builds an **AST** (Abstract Syntax Tree). Pure syntax — no type knowledge yet. A syntax error (`const x = ;`) is caught here.
3. **Binder.** Walks the AST once, creates **Symbols** — a Symbol is the compiler's internal representation of a named entity (variable, function, type, namespace) linking all its declarations together (this is what enables declaration merging) and establishing scope.
4. **Checker.** The heart of TS. On demand (lazily, as needed — not eagerly for the whole program), it:
   - Resolves each expression's type (`getTypeOfSymbol`, `getTypeAtLocation`).
   - Performs **control flow analysis** (narrowing types based on `if`, `typeof`, `&&`, assignments — Ch. 13).
   - Checks **assignability**/compatibility between types (structural comparison — Ch. 3).
   - Emits diagnostics (errors/warnings) when incompatible.
5. **Emitter.** Walks the AST again, using checker info only where needed (e.g. to decide whether to elide an unused import, or inline a `const enum`), and prints `.js` (+ optionally `.d.ts`, `.map`).

**Key internal fact:** the checker is largely **lazy and on-demand** — it doesn't necessarily type-check every single node eagerly in file order; it resolves types as they're requested (e.g., when checking an assignment, it asks "what's the type of the RHS?" which recursively triggers resolution). This laziness is part of why huge, deeply-generic types can blow up compile time unpredictably — cost is incurred where types are *consumed*, not just where they're *declared*.

**Compiler behavior — the two ways to run it:**
- `tsc` — full pipeline: parse → bind → check → emit. Errors reported, exit code non-zero on error (still emits unless `noEmitOnError`).
- `tsc --noEmit` — checking only, no output files. Common in CI as a pure "type-check gate," separate from the actual build (which is often delegated to esbuild/swc/webpack for speed, since they skip step 4 entirely).

**Real-world use cases.** `tsc --noEmit` in CI/pre-commit; `tsc -w` (watch mode, incremental recheck of changed files only via a dependency graph); `tsc --build` for multi-project (monorepo) builds with project references.

**Good practices.** In production toolchains, separate *type-checking* (`tsc --noEmit`, can run in parallel/CI) from *bundling/transpiling* (esbuild/swc/babel, fast, no type awareness) — get both speed and safety without serializing them.
**Bad practices.** Relying on a bundler's TS integration alone and never running `tsc --noEmit` — many fast bundlers silently ignore type errors.
**Common mistake.** Assuming `tsc` won't produce output if there are errors — it will, unless configured not to.
**Edge case.** Circular type references are legal and common (e.g., recursive types, mutually-referencing interfaces) — the checker handles this via lazy resolution + internal recursion depth guards, but *circular imports* combined with `isolatedModules` + certain const-enum/type-only patterns can break single-file transpilers even though `tsc` handles them fine (because `tsc` sees the whole program graph).
**Performance implications.** Whole-program checking is inherently `O(files × types-touched)`-ish and non-linear with generic complexity; mitigations: `incremental`, `composite` project references, `skipLibCheck` (skip re-checking `.d.ts` of dependencies), isolating hot generic utility types.

### Interview Q&A — §1.3

**Junior:** Q: What does `tsc` do? A: Type-checks TS code and compiles it to JavaScript.
**Mid:** Q: What's the difference between `tsc` and `tsc --noEmit`? A: `--noEmit` runs parsing/binding/checking only, producing diagnostics but no `.js` output — used as a pure verification step.
**Senior:** Q: Why is TS type-checking described as "lazy"? What's the practical consequence? A: The checker resolves types on demand as they're referenced rather than eagerly precomputing everything, so pathological cost shows up at *usage sites* of complex generic/conditional types (e.g., a deeply nested conditional type used in a hot function signature can slow the whole build even if the type is "declared" cheaply).
**FAANG/tricky:** Q: Two files import each other and both declare interfaces referencing each other recursively. Why does `tsc` compile this fine but a single-file transpiler might error on "cannot find name"? A: `tsc` builds the whole-program symbol/module graph before checking (binder sees all files), so forward/circular references resolve; single-file tools (Babel, isolatedModules mode) process files independently and can't see across the circular import boundary for type-only info, so they require you to avoid patterns that need cross-file type resolution (this is exactly what `isolatedModules` flags).

**Predict/Debug:** "Why does my editor show an error instantly but running `node dist/index.js` after `tsc` still executes the buggy code path?" → Because emit happened despite errors (no `noEmitOnError`), and Node just runs plain JS with no awareness that TS ever objected.

---

## 1.4 Transpilation vs Type Checking (and "Downleveling")

**Intuition.** "Transpiling" = translating source-to-source (TS syntax → JS syntax, or modern JS → older JS). It's a *syntax* operation. "Type-checking" is a *semantic* operation asking "is this program logically consistent." People conflate them because `tsc` happens to do both, but they are independent capabilities.

**Technical explanation.**
- **Transpilation / Downleveling**: controlled by `target` (e.g., `ES2022` → `ES5`) and `lib`. TS rewrites newer JS syntax into older equivalents (e.g., `async/await` → generator+state-machine polyfill pattern for old targets, optional chaining `?.` → conditional expressions, class fields → `Object.defineProperty`/constructor assignments).
- **Type erasure**: removes TS-only syntax (types, interfaces, generics params) — not really "downleveling," just deletion.

**Internal behavior.** These happen in the **same emit pass** in `tsc`, but are conceptually separable — which is exactly what `isolatedModules` enforces: it requires every file to be independently transpilable without needing whole-program type info, because tools like Babel/esbuild/swc only transpile (erase + downlevel), they never type-check.

**Compiler behavior.**
```
target: "ES2022" → syntax downleveled to ES2022 features max
module: "ESNext" | "CommonJS" | "NodeNext" ... → import/export syntax rewritten accordingly
lib: which ambient global APIs (DOM, ES2022, etc.) are assumed to exist for type-checking purposes only
```
`lib` affects **only type-checking** (what global types like `Array.prototype.flatMap` or `fetch` are known to exist) — it does **not** polyfill anything at runtime. Setting `lib: ["ES2022"]` but running on an old engine without `Promise` will type-check fine and crash at runtime. This trips people up constantly.

**Type inference angle.** `lib` files are literally `.d.ts` declaration files shipped with TS (`lib.es2022.d.ts`, `lib.dom.d.ts`) — the checker treats them as ambient truth. Inference for `document.querySelector(...)` comes entirely from `lib.dom.d.ts`, not from any real analysis of a browser.

**Real-world use cases.** Targeting `ES2020` for modern evergreen browsers vs `ES5` for legacy IE11 support; using `NodeNext`/`Node16` module mode to correctly model Node's dual ESM/CJS resolution; excluding `dom` from `lib` in a Node-only project so accidental use of `window`/`document` is a compile error.

**Good practices.** Set `target` to the actual runtime you support (don't default to `ES5` if you only ship to modern Node/evergreen browsers — you pay bundle-size and performance cost downleveling `async/await`, classes, etc. for nothing). Match `lib` to your actual runtime capabilities, not just "whatever compiles."
**Bad practices.** Using `lib: ["ESNext", "DOM"]` in a Node backend project — you'll get false green lights on `fetch`/`localStorage` usage that doesn't exist in your actual runtime (unless it genuinely does, e.g. modern Node has `fetch`).
**Common mistakes.**
- Thinking `target: "ES2022"` polyfills anything. It doesn't — you still need `core-js` or similar for missing runtime APIs on old engines; `target`/`lib` only affects syntax downleveling and type knowledge.
- Confusing `module` (output module format) with `target` (output syntax level) — orthogonal settings.
**Edge case.** `useDefineForClassFields` interacts with `target`: for `target >= ES2022`, class fields default to real `[[Define]]` semantics (matches JS spec) vs the older TS-specific `[[Set]]`-based emulation — this is a genuine semantic difference (affects behavior with inherited accessors), not just syntax, showing transpilation *can* leak semantics if misconfigured.
**Performance.** Lower `target` = more downlevel emit code (bigger bundles, slower runtime, e.g. generator-based async emulation) — always target as high as your actual support matrix allows.

### Interview Q&A — §1.4

**Junior:** Q: Does setting `target: "ES5"` make `Promise` work in IE11? A: No — you'd still need a `Promise` polyfill; `target` only changes *syntax*, `lib` only changes *type knowledge*, neither adds runtime behavior.
**Mid:** Q: What's the difference between `target` and `lib`? A: `target` controls output JS syntax version; `lib` controls which ambient API type declarations the checker assumes are available.
**Senior:** Q: Why does `isolatedModules: true` exist, and what does it forbid? A: It ensures every `.ts` file can be transpiled in isolation (needed for Babel/esbuild/swc pipelines), so it forbids constructs that require whole-program type info to emit correctly — e.g., re-exporting a type without `export type`, or non-const `enum` merging across files in ways that need cross-file resolution, or `const enum` inlining.
**FAANG/tricky:** Q: A team upgrades `target` from `ES5` to `ES2020` and a subtle bug appears in class field initialization order relative to a parent's overridden accessor. Explain. A: This is the `useDefineForClassFields` semantic change — `ES2022`+ class fields use `[[Define]]` (spec-accurate) which can bypass a parent's setter/accessor, whereas the ES5-emulation used `[[Set]]` and would trigger it — a genuine behavior change from raising `target`, not just a cosmetic syntax change.

**Predict the output.**
```ts
// target: ES5
const greet = (name: string) => `Hi ${name}`;
```
→ Arrow function downlevels to a regular `function` expression (no `this`-binding needed here, but TS still downlevels syntax); template literal downlevels to string concatenation on ES5 target.

---

## 1.5 `tsconfig.json`

**Intuition.** `tsconfig.json` is the constitution for your project's compiler — it says which files are in scope, what JS environment you're targeting, and how strict the checker should be.

**Technical explanation — structure:**
```jsonc
{
  "compilerOptions": { /* §1.6 */ },
  "include": ["src/**/*"],       // glob patterns to include
  "exclude": ["node_modules"],   // glob patterns to exclude (default already excludes node_modules)
  "files": ["src/main.ts"],      // explicit file list (rare; for small/precise sets)
  "extends": "./tsconfig.base.json", // inherit from another config (monorepos)
  "references": [{ "path": "../shared" }] // project references (composite builds)
}
```

**Internal behavior.** `tsc` resolves the "program" — the exact set of files to check — by combining `files`/`include`/`exclude` and then **transitively following all imports** from those root files (even files not matched by `include` get pulled in and checked if they're imported — a common surprise). Presence of a `tsconfig.json` also implicitly signals "this directory is a TS project root" to editors (VS Code's TS server looks for the nearest `tsconfig.json` upward from an open file).

**Compiler behavior — resolution order.** When you run bare `tsc`, it looks for `tsconfig.json` in the current directory, then walks up parent directories. `extends` performs a deep merge (child overrides parent) — this enables a monorepo pattern: one `tsconfig.base.json` with shared strictness settings, each package extending it with its own `outDir`/`rootDir`.

**Project references (`composite: true` + `references`)** let `tsc --build` treat a monorepo as a DAG of projects, type-checking/emitting each once and caching (`.tsbuildinfo`), only rechecking what changed — this is the main lever for keeping large-codebase compile times sane.

**Real-world use cases.** Monorepos (Nx/Turborepo/Lerna) using base config + references; separate `tsconfig.build.json` (stricter, excludes tests) vs `tsconfig.json` (used by editor, includes tests) — a very common senior-level pattern.

**Good practices.**
- Keep one strict `tsconfig.base.json`, extend per-package.
- Separate the "editor/dev" config from the "production build" config when you need different `include`/emit settings.
- Use `composite`/`references` once you have >1 internal package depending on another.

**Bad practices.**
- One giant `tsconfig.json` with no `include`, accidentally checking `node_modules` types or test fixtures in prod builds.
- Forgetting that imported-but-not-included files still get checked (surprise errors from a stray script).

**Common mistakes.**
- Assuming `exclude` stops a file from being *checked* if something still imports it — `exclude` only removes it from the **initial root set**; if a root file imports it, it's included anyway. `exclude` mainly matters for the initial file-globbing and for `--build` project boundaries.
- Not realizing VS Code's red squiggles use the **editor's own config resolution** (nearest `tsconfig.json`), which can differ from what your build script actually points at.

**Edge case.** `files: []` combined with `references` is a common "solution-style" root tsconfig pattern — a root config with no files of its own, just referencing sub-projects, used purely to `tsc --build` the whole repo.

**Performance.** `composite`/`incremental` + `.tsbuildinfo` caching is the single biggest lever for large-repo compile speed; `skipLibCheck: true` avoids re-checking every dependency's `.d.ts` files (usually safe — those are assumed already valid).

### Interview Q&A — §1.5

**Junior:** Q: What's the purpose of `tsconfig.json`? A: Configures which files TS compiles/checks and how (target, strictness, module system, etc.).
**Mid:** Q: If a file is in `exclude`, is it guaranteed never to be type-checked? A: No — if a non-excluded file imports it, it's pulled into the program and checked anyway. `exclude` only affects the initial include-glob resolution.
**Senior:** Q: How do project references (`composite`) improve build performance in a monorepo, mechanically? A: Each referenced project is built independently with `.tsbuildinfo` emitted, recording file/type dependency state; `tsc --build` only rebuilds a project if its own files or *its declared dependencies'* outputs changed, turning an O(whole repo) recheck into O(changed subgraph).
**FAANG/tricky:** Q: You have `strict: true` in `tsconfig.base.json` but a package extends it and still reports no errors on obviously unsafe code. What's a likely cause? A: The child config's `include`/`files` might not actually be picking up those source files (wrong `rootDir`, wrong glob), so `tsc` is silently checking an empty/wrong file set — always verify with `tsc --listFiles` or `--showConfig`.

**Debugging exercise.** Given this config, predict what happens:
```jsonc
{
  "compilerOptions": { "strict": true },
  "include": ["src"],
  "exclude": ["src/**/*.test.ts"]
}
```
`src/index.ts` imports `./helpers.ts`, and `helpers.ts` imports a type from `src/legacy.test.ts` (a mistake). Will `legacy.test.ts` be checked? → **Yes** — even though excluded by glob, being imported by an included file pulls it into the program.

---

## 1.6 Compiler Options — Deep Dive

Group by purpose rather than alphabetically (this is how you should reason about them in interviews/config decisions):

### A. Type-checking strictness
| Option | What it actually does |
|---|---|
| `strict` | Master switch enabling the bundle below. Always start with this `true`. |
| `noImplicitAny` | Errors when a type can't be inferred and silently falls back to `any`. Forces explicit typing at true boundaries. |
| `strictNullChecks` | `null`/`undefined` become distinct types, not silently assignable to everything. **The single highest-value flag** — without it, `string` secretly means `string \| null \| undefined` everywhere. |
| `strictFunctionTypes` | Enforces contravariant checking of function parameters (Ch. 3) instead of unsound bivariant checking, for non-method function types. |
| `strictBindCallApply` | Type-checks `.bind/.call/.apply` arguments against the function's real signature. |
| `strictPropertyInitialization` | Class properties must be initialized in the constructor or have a definite assignment assertion — catches "declared but never set" bugs. Requires `strictNullChecks`. |
| `noImplicitThis` | Errors when `this` has an implicit `any` type. |
| `alwaysStrict` | Emits `"use strict"` and parses in strict JS mode. |
| `useUnknownInCatchVariables` | `catch (e)` gives `e: unknown` instead of `any` (default `true` under `strict` in modern TS) — forces you to narrow before using the error. |

### B. Additional checks (not in `strict`, but recommended)
- `noUnusedLocals` / `noUnusedParameters` — flags dead code.
- `noImplicitReturns` — every code path in a function must explicitly return (or none do).
- `noFallthroughCasesInSwitch` — catches accidental `switch` fallthrough.
- `exactOptionalPropertyTypes` — `{ a?: string }` means "a is absent OR a string," **not** "a can be explicitly `undefined`" — a subtle but real distinction (`obj.a = undefined` becomes an error).
- `noUncheckedIndexedAccess` — `arr[i]` and `obj[key]` return `T | undefined` instead of just `T`, modeling reality (index access can miss). Extremely high-value, rarely enabled by default teams.

### C. Modules & interop
- `module`: output module format (`CommonJS`, `ESNext`, `NodeNext`...).
- `moduleResolution`: algorithm for resolving `import` specifiers to files (`Node10`(legacy `node`), `Bundler`, `NodeNext`). Must generally match `module`.
- `esModuleInterop`: makes `import x from 'cjs-module'` work sanely against CommonJS packages that don't have a real default export, by synthesizing one — turns off a lot of confusing `import * as x` requirements.
- `allowSyntheticDefaultImports`: type-checking-only version of the above (no runtime change) — usually implied by `esModuleInterop`.
- `resolveJsonModule`: allows `import data from './data.json'` with inferred types.
- `isolatedModules`: (§1.4) require every file to be independently transpilable.
- `verbatimModuleSyntax`: forces you to explicitly write `import type`/`export type` for type-only imports so single-file transpilers know exactly what to erase, with zero ambiguity.

### D. Emit
- `outDir`/`rootDir`: where output goes / the input root used to mirror folder structure.
- `declaration`: emit `.d.ts` files (mandatory for publishing a library).
- `declarationMap`: source maps for `.d.ts` → original `.ts`, enabling "go to definition" landing in your real source instead of the generated declaration.
- `sourceMap`: `.js.map` for debugging compiled output against original source.
- `noEmitOnError`: don't emit `.js` if there were type errors — flips the "TS emits anyway" default behavior discussed in §1.3.
- `removeComments`, `importHelpers` (dedupe downlevel helper code via `tslib` instead of inlining per-file).

### E. Performance / project structure
- `incremental` + `tsBuildInfoFile`: cache type info between builds.
- `composite`: enables project references, forces `declaration: true`.
- `skipLibCheck`: skip type-checking of `.d.ts` files (usually just dependencies) — big speed win, standard in most real-world configs.

**Good practices.** Start every new project from `strict: true` + `noUncheckedIndexedAccess` + `exactOptionalPropertyTypes` + `skipLibCheck: true`. Retrofitting strictness onto a large loose codebase later is far more painful than starting strict.
**Bad practices.** Disabling `strictNullChecks` "temporarily" — it never gets re-enabled, and it's the flag with the highest bug-catching value.
**Common mistakes.** Assuming `strict: true` implies *every* useful check — `noUncheckedIndexedAccess` and `exactOptionalPropertyTypes` are **not** part of `strict` and must be opted into separately.
**Edge case.** `skipLibCheck: true` means a genuinely broken `.d.ts` in a dependency can go unnoticed until it causes a *downstream* type error in your code that resolves through it — the error surfaces at the use-site, not the source, which can be confusing to debug.
**Performance.** `skipLibCheck` + `incremental` + `composite` are the three biggest levers; also, deeply recursive/generic utility types in hot paths (widely-imported files) dominate check time more than raw file count does.

### Interview Q&A — §1.6

**Junior:** Q: What does `strictNullChecks` do? A: Makes `null`/`undefined` explicit, non-implicit members of a type, so you must handle them intentionally rather than TS silently allowing them everywhere.
**Mid:** Q: What's the difference between `target` and `module`? A: `target` = ECMAScript syntax version of the emitted JS; `module` = the module system syntax (`import/export` vs `require`) of the emitted JS.
**Senior:** Q: Why would you enable `noUncheckedIndexedAccess` even under `strict: true`? A: `strict` doesn't make array/object index access account for out-of-bounds/missing keys — `arr[i]` is typed `T`, not `T | undefined`, which is unsound (arrays can be shorter than assumed). This flag closes that soundness gap at the cost of extra narrowing/assertions at call sites.
**FAANG/tricky:** Q: Your team enables `exactOptionalPropertyTypes` and CI suddenly fails on previously "fine" code doing `const obj: { a?: string } = { a: undefined }`. Explain precisely what changed semantically. A: Without the flag, `a?: string` is treated (unsoundly) as if it also permits explicit `undefined` assignment, conflating "key absent" with "key present with value undefined." With the flag, TS distinguishes them precisely (matching real JS semantics like `'a' in obj` vs `obj.a === undefined`), so explicitly assigning `undefined` to an optional property is now correctly rejected unless the type explicitly says `a?: string | undefined`.

**Predict-the-error exercise.**
```ts
// strict: true, noUncheckedIndexedAccess: true
const arr: string[] = ["a", "b"];
const first: string = arr[0]; // error?
```
→ **Error.** With `noUncheckedIndexedAccess`, `arr[0]` has type `string | undefined`, not assignable to `string` without a check/assertion (`arr[0]!` or a guard).

**Refactoring exercise.** Take this loose config and make it senior-grade:
```jsonc
{ "compilerOptions": { "target": "ES5", "noImplicitAny": false } }
```
→ Discuss: raise `target` to match real runtime, enable full `strict`, add `noUncheckedIndexedAccess`, `skipLibCheck`, set explicit `include`/`exclude`, decide `module`/`moduleResolution` based on Node/bundler target.

---

## Practical Exercises — Chapter 1

1. **Setup drill:** From scratch, write a `tsconfig.json` for a Node 20 backend (ESM), and a separate one for a browser app bundled by Vite. Justify each differing option (`module`, `moduleResolution`, `lib`, `target`).
2. **Diagnosis:** A teammate says "TypeScript didn't catch this bug in production." Walk through the checklist of reasons that could be true (erasure, `any`, unchecked external data via `as`, `noEmitOnError` off + ignored CI failure, wrong file included in program, etc.).
3. **Build-speed audit:** Given a monorepo with 5 packages where each depends on the previous, describe how you'd introduce `composite`/`references` and measure the win.
4. **Conceptual proof:** Explain, without looking anything up, why `interface Foo {}` produces zero bytes of runtime JS but `class Foo {}` does not — tie it back to erasure and symbol/value duality.

---

## Chapter 1 Summary — Mental Model

```
 .ts source
     │
     ▼
 [Scanner] → tokens
     │
     ▼
 [Parser] → AST (syntax only, no meaning yet)
     │
     ▼
 [Binder] → Symbols (names ↔ declarations ↔ scopes)  ← declaration merging happens here
     │
     ▼
 [Checker] → resolves types lazily, control-flow narrows, compares structurally,
             emits diagnostics                        ← "TypeScript" really lives here
     │
     ▼
 [Emitter] → erase types, downlevel syntax per target/module → .js (+ .d.ts, .map)
```
Everything in this course is really about **one thing**: what the Checker considers two types "compatible enough" for, and how it infers types when you don't write them explicitly. Chapters 2–3 build that foundation directly.

---

**Next:** Chapter 2 — Types: primitives, literals, objects, arrays, tuples, enums, const assertions, widening & narrowing. Say **"next"** to continue.
