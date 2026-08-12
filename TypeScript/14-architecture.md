---
layout: default
title: "Chapter 14 — Architecture & Best Practices"
---

# Chapter 14 — Architecture & Best Practices

## 14.1 The `satisfies` Operator

**Intuition.** For years, TS forced a choice: annotate a value with a type (get validation, but lose precise literal inference — Ch. 13.1's contextual-typing widening) or don't annotate it (keep precise inference, but lose validation against a known shape). `satisfies` gives you **both**: validate against a type, while keeping the expression's own naturally-inferred (narrower) type.

**Technical explanation.**
```ts
type Config = Record<string, { url: string; timeout?: number }>;

const config = {
  prod: { url: "https://api.example.com", timeout: 5000 },
  dev: { url: "http://localhost:3000" },
} satisfies Config;

config.prod.timeout.toFixed(2); // OK — TS knows `prod` specifically has `timeout`, from the literal's own inferred type
```
Compare to `const config: Config = {...}` — that annotation would widen every entry to the *general* `{ url: string; timeout?: number }` shape, and `config.prod.timeout` would be `number | undefined`, losing the fact that `prod` specifically always has a `timeout`. `satisfies` checks the object against `Config` (catching typos/missing required fields, exactly like a normal annotation would) but then **discards the annotation** for the purpose of the variable's actual resulting type, keeping the tighter, literal-preserving bottom-up-inferred type instead.

**Internal behavior.** `satisfies` performs an assignability check (Ch. 3.4) between the expression and the given type — exactly like a normal annotation or type assertion would — but, unlike an annotation, doesn't change what type is actually recorded for the expression going forward; unlike a type assertion (`as`), it's a **checked**, not a blind, operation (an `as` doesn't validate anything, it just forces the type; `satisfies` genuinely errors if the expression doesn't conform).

**Compiler behavior / inference.** `satisfies` is applied *after* the expression's own type has already been computed via normal (bottom-up, possibly-contextual) inference — it's a validation pass layered on top, not a different inference mode.

**Real-world use cases.** Configuration objects (as above) where different entries legitimately have different, narrower shapes you want to keep visible; validating that an object literal correctly implements all keys of a `Record<SomeUnion, ...>` (getting `Record`'s exhaustiveness-like key-completeness checking) while keeping each entry's own specific inferred type; validating route-handler maps, theme/design-token objects, and similar heterogeneous-but-structured config data.

**Good practices.** Reach for `satisfies` specifically whenever you'd otherwise use a type annotation purely for *validation* but the annotation's widening effect is throwing away information you actually want to keep — this is an extremely common, high-value pattern that became standard practice almost immediately after its introduction (TS 4.9).
**Bad practices.** Using `as` where `satisfies` would give the same validation with actual safety (recall Ch. 3.4: `as` doesn't check anything, it's a blind assertion) — a common, easy-to-fix modernization opportunity in codebases that predate `satisfies` and still lean on `as` for validation-adjacent purposes.
**Common mistakes.** Assuming `satisfies` changes the variable's type to the `satisfies`-clause type (like an annotation would) — it deliberately does the opposite: the variable's type remains whatever bottom-up/contextual inference already determined, satisfies is purely a compile-time check with no effect on the resulting type.
**Edge case.** `satisfies` composes naturally with `as const` (`{...} as const satisfies SomeType`) — validate the shape *and* keep it maximally literal/readonly, a common combined idiom for constant configuration data that needs both guarantees simultaneously.
**Performance.** Negligible — one extra assignability check, same cost class as a normal annotation check.

### Interview Q&A — §14.1
**Junior:** Q: What's the key difference between `const x: T = value` and `const x = value satisfies T`? A: Both validate `value` against `T`, but the annotation form (`: T`) makes `x`'s actual type become `T` (potentially widening/losing precision); `satisfies` validates against `T` but keeps `x`'s type as whatever was naturally inferred from `value` itself, preserving more precise literal/structural information.
**Mid:** Q: Why is `satisfies` considered safer than `as` for validation purposes? A: `as` is an unchecked type assertion — it doesn't verify the expression actually conforms to the asserted type at all (Ch. 3.4); `satisfies` performs a real, checked assignability comparison, and produces a compile error if the expression genuinely doesn't conform to the given type — it's validation, not a blind override.
**Senior:** Q: Give a concrete real-world scenario where `satisfies` provides meaningfully better safety than a plain type annotation, and explain precisely what's gained. A: A `Record<Environment, Config>`-style object where different environment entries have legitimately different optional fields populated — annotating with `Record<Environment, Config>` directly widens every entry to the exact same general `Config` shape, losing the fact that (say) the `prod` entry specifically always includes a `timeout` while `dev` doesn't; using `satisfies Record<Environment, Config>` instead validates that every environment key is present and every entry conforms to `Config` (catching typos/missing keys, the same safety a direct annotation would give), while still letting downstream code that specifically accesses `config.prod.timeout` see its true, narrower, always-present-for-prod inferred type rather than a needlessly widened `number | undefined`.

---

## 14.2 Type-Safe Error Handling Patterns

**Intuition.** JS exceptions are **untyped** — a function's signature gives zero indication of what it might throw, unlike checked exceptions in Java. Senior TS codebases generally compensate with one of two disciplined strategies: making expected failures part of the return type (`Result<T, E>`, Ch. 6.2), or carefully, narrowly using exceptions only for genuinely unexpected/unrecoverable conditions.

**Technical explanation — the `Result<T, E>` pattern (full production version).**
```ts
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

function ok<T>(value: T): Result<T, never> { return { ok: true, value }; }
function err<E>(error: E): Result<never, E> { return { ok: false, error }; }

function parseConfig(raw: string): Result<Config, ValidationError> {
  try {
    const parsed = JSON.parse(raw);
    if (!isValidConfig(parsed)) return err(new ValidationError("Invalid config shape"));
    return ok(parsed);
  } catch {
    return err(new ValidationError("Malformed JSON"));
  }
}

const result = parseConfig(input);
if (!result.ok) {
  console.error(result.error.message); // forced to handle the error branch before...
} else {
  useConfig(result.value); // ...accessing the success value — enforced by narrowing
}
```
Using `never` as the "other side's" type parameter in `ok`/`err` (`Result<T, never>`/`Result<never, E>`) is a deliberate, precise touch — it correctly signals "this specific result can never actually be the error/success branch," which then correctly unions when combined (`Result<T, never> | Result<never, E>` collapses sensibly when used generically).

**Internal behavior.** This pattern's entire safety value comes directly from Ch. 5.4's union-narrowing rules — `result.value` is genuinely inaccessible until `result.ok` has been checked, because it doesn't exist on the `{ ok: false, error: E }` branch, and the checker won't let you access a member not present on every currently-possible union member without first narrowing.

**Real-world use cases.** API client libraries, form/input validation, parsing untrusted input, any function where failure is an expected, common, and specifically-typed possibility (not an exceptional, rare, catch-all-worthy event) — increasingly the default recommended pattern in modern TS backend/domain-logic code, heavily influenced by Rust's `Result<T, E>` and functional-language `Either` types.

**When to still use exceptions.** Genuinely exceptional, unrecoverable, "something is broken about the program itself" conditions (programming errors, invariant violations, out-of-memory-class failures) are still reasonably modeled with `throw` — the `Result` pattern is specifically for *expected*, *recoverable*, *part of the normal domain logic* failure modes, not a wholesale replacement for exceptions everywhere.

**Good practices.** Be consistent within a codebase/team about which failure modes use `Result` vs exceptions — document the convention explicitly (e.g., "validation and business-rule failures return `Result`; infrastructure/programming errors throw") rather than mixing ad hoc per function.
**Bad practices.** Using `Result<T, E>` for genuinely exceptional conditions no caller could reasonably be expected to check for at every call site (creates enormous, tedious boilerplate for cases where a caught-at-a-higher-level exception handler is genuinely the better design) — like all patterns in this course, it's a tool for a specific job, not a universal replacement for `throw`/`try`/`catch`.
**Common mistakes.** Forgetting exceptions are **not tracked by the type system at all** — a function's signature gives zero compile-time indication it can throw, so relying purely on documentation/convention (rather than the `Result` pattern) for expected failures leaves a real, unenforced gap that's easy for a caller to forget to handle; this is precisely the gap `Result` closes for the failure modes that matter most.
**Edge case.** Async functions returning `Promise<Result<T, E>>` (rather than a rejected promise) let you apply the exact same discipline to asynchronous code — `await`ing such a function still requires the same `if (!result.ok)` handling, with no separate `try`/`catch` needed for the *expected* failure paths (real infrastructure failures like network errors are a separate, legitimate use for an actual rejected promise/`catch`).
**Performance.** Negligible runtime overhead (a plain tagged object, no exception-throwing machinery involved for the expected-failure path) — can even be marginally faster than exception-based control flow in hot paths, since thrown exceptions carry real runtime cost (stack unwinding, stack trace capture) that a plain return value doesn't.

### Interview Q&A — §14.2
**Junior:** Q: Why can't TypeScript's type system tell you what exceptions a function might throw? A: JS/TS has no "checked exceptions" concept — a function's type signature only describes its parameters and return type; anything it might `throw` is completely invisible to the type system, unlike, say, Java's `throws` clause.
**Mid:** Q: What does the `Result<T, E>` pattern give you that plain `try`/`catch` doesn't? A: It makes the possibility of failure an explicit, visible part of the function's *type signature* (its return type), and forces callers to handle both branches via type narrowing before accessing the success value — `try`/`catch` provides no compile-time enforcement that a caller actually handles a given failure; it's easy to simply forget to wrap a call, or to catch too broadly/narrowly without the compiler ever noticing.
**Senior:** Q: When would a senior engineer deliberately choose exceptions over `Result<T, E>` even in a codebase that otherwise favors the `Result` pattern? A: For genuinely exceptional, unrecoverable-at-the-call-site conditions — programming bugs (invariant violations, "this should never happen" states), infrastructure failures better handled by a централized higher-level error boundary/handler rather than individually at every call site, or conditions so rare that forcing every caller to explicitly branch on them would add pure boilerplate without meaningfully improving correctness; the `Result` pattern's value is specifically for failures that are common, expected, and meaningfully differ in how different callers should react to them.

---

## 14.3 Designing Type-Safe APIs

**Principle 1 — accept broad, return narrow.** Function parameters should generally use the widest type that's still safe to consume (`readonly T[]` over `T[]`, `Iterable<T>` over `Array<T>` when you only need to iterate, a minimal structural interface over a large concrete class type when only a few members are used) — this maximizes what callers can pass. Return types should be as *specific* as honestly possible — a precise literal/union/branded return type gives downstream code the most to work with. This mirrors, and is a direct practical consequence of, the variance principles from Ch. 3.5 (consuming positions want width, producing positions want precision).

**Principle 2 — make illegal states unrepresentable.** Revisit Ch. 2.9/5.4's core lesson at the architecture level: before reaching for runtime validation/assertions, ask whether a better type design (a discriminated union instead of several optional fields, a branded type instead of a raw primitive, Ch. 9.2) can make the invalid state impossible to construct in the first place, rather than merely detected after the fact.

**Principle 3 — minimize `any`, maximize `unknown` at boundaries.** Every external boundary (HTTP responses, `JSON.parse`, third-party libraries lacking types, user input) should type as `unknown` and be explicitly narrowed/validated (ideally via a real runtime schema validator like Zod, whose inferred TS types can then flow naturally into the rest of the codebase) before being trusted as a specific domain type — never let `any` leak in from an external boundary and silently propagate.

**Principle 4 — prefer composition of small types over large, monolithic ones.** Small, focused interfaces/branded types/utility-type building blocks (Ch. 5–10) compose more flexibly and produce clearer error messages than a small number of large, do-everything types — this is the same design instinct as small, focused functions, applied to the type level.

**Principle 5 — use the compiler as a refactoring safety net deliberately.** When changing a widely-used type (adding a required field, renaming a union member), let the resulting compile errors *guide* the refactor — this is one of TS's highest real-world productivity payoffs at scale, and is worth designing for explicitly (e.g., preferring exhaustive `switch`-based handling, Ch. 13.4, specifically because it turns a union change into an actionable, precisely-located list of compile errors rather than a silent runtime gap).

### Interview Q&A — §14.3
**Mid:** Q: Why might a senior engineer prefer a function parameter typed `readonly T[]` over `T[]`, even if the function never actually mutates the array? A: `readonly T[]` costs callers nothing (a mutable array is freely assignable to a readonly-typed parameter, Ch. 3.6/2.4) while communicating and enforcing, at the type level, a guarantee that the function won't mutate the caller's array — a small, free, real safety improvement with no downside for legitimate callers.
**Senior:** Q: Explain the "accept broad, return narrow" principle in terms of variance (Ch. 3.5). A: Function parameters are consumed by the function, so accepting the widest safe type maximizes caller flexibility — this mirrors why consuming positions (like function parameters) are contravariant-friendly, wanting "at least this much." Return types are produced by the function and consumed by callers, so returning the narrowest honest type gives callers the most specific, most useful information — mirroring why producing positions (return types) are covariant-friendly, wanting "at most/exactly this specific." The API design heuristic is a direct, practical translation of the same variance intuition that governs safe function-type substitutability.

---

## 14.4 Scaling TypeScript in Large Codebases

**Technical/architectural guidance, consolidated from earlier chapters into concrete large-codebase practice:**

- **Project references + composite builds** (Ch. 1.5) for monorepos — keep compile times bounded by the size of what actually changed, not the whole repo.
- **A shared base `tsconfig` with full strictness**, extended per-package (Ch. 1.5) — strictness should be a repo-wide default, not a per-package opt-in that quietly varies.
- **`skipLibCheck: true` + `incremental: true`** as near-universal defaults for build performance (Ch. 1.6) at scale.
- **Centralized, generated types for external boundaries** — API response types generated from an OpenAPI/GraphQL schema (rather than hand-maintained) so the type surface can't silently drift from the actual backend contract; combined with a runtime validator (Zod, io-ts) at the actual network boundary for genuine runtime safety, not just compile-time trust.
- **Avoid deeply nested, widely-imported generic/conditional utility types in hot, common-path files** (Ch. 8/9's performance notes) — a slow utility type used in one obscure file is a minor cost; the same utility type imported by hundreds of files compounds across the whole build.
- **Prefer `interface` for genuinely public, extension-point type surfaces; `type` elsewhere** (Ch. 5.1) — establish and document the convention once, apply consistently.
- **Treat exhaustiveness-checked discriminated unions as the default pattern for evolving domain state** (Ch. 13.4) — the single highest-leverage pattern for keeping a large, multi-person codebase correct as it changes over time, since it converts "did everyone update every relevant spot" into a compiler-enforced guarantee.
- **Periodically audit for `any` leakage** — `any` is viral (Ch. 2.1) and a single unguarded boundary can quietly erode type safety across a large surface of connected code; linting rules (`no-explicit-any`, `no-unsafe-*` from `@typescript-eslint`) are a standard, worthwhile investment at scale.

### Interview Q&A — §14.4
**Senior/FAANG:** Q: What's the single most impactful architectural decision for keeping a large, multi-team TypeScript codebase both fast to build and correct over years of evolution? A: There isn't truly one silver bullet, but if forced to pick the highest-leverage combination: (1) generating (not hand-maintaining) types for every external contract boundary paired with real runtime validation at the actual boundary, which prevents the single most common real-world source of "types lied to us" production bugs, and (2) defaulting to exhaustiveness-checked discriminated unions for evolving domain state, which converts the single most common real-world source of "we forgot to update this one spot" bugs into compiler-enforced, precisely-located errors at the exact moment a union changes — together these address the two dominant failure modes (external-boundary drift, internal-change propagation) that erode type safety in large, long-lived, multi-team codebases specifically as they scale in size and contributor count over time.

---

## Practical Exercises — Chapter 14

1. **`satisfies` migration:** Find (or construct) an object literal currently using a plain type annotation purely for validation, and migrate it to `satisfies`, documenting exactly what type precision was recovered.
2. **Build a `Result<T,E>` library:** Implement `ok`, `err`, `map`, `mapError`, `andThen`, and `unwrapOr` for a `Result<T, E>` type, fully generic, and refactor one real function from throwing to returning `Result`.
3. **Boundary audit:** Take a real (or hypothetical) API-consuming function, and rewrite its response handling to type the raw response as `unknown`, validate it explicitly (with or without a schema library), and only then treat it as a trusted domain type.
4. **Illegal-state redesign:** Take an existing type with several optional/boolean fields implying an informal state machine, and redesign it as a discriminated union with full exhaustiveness-checked handling.
5. **Architecture review exercise:** Given a hypothetical large monorepo's `tsconfig` setup, audit it against §14.4's checklist and identify concrete, prioritized improvements.

---

**Next:** Chapter 15 — Framework Integration: React, Node/Express, NestJS, Next.js, and designing generic libraries. Say **"next"** to continue.
