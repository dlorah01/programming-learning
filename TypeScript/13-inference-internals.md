# Chapter 13 — Type Inference & Control Flow Internals

This chapter goes underneath everything from Chapters 2, 6, and 8 to explain the actual mechanism the checker uses to infer and narrow types — the "how," not just the "what."

## 13.1 How Inference Actually Works — The Big Picture

**Intuition.** The checker never asks "what type is this variable" as an isolated question — it asks "what type is this variable **at this exact point in the program**," because a variable's type can legitimately be different at different points (before/after narrowing, before/after reassignment). Inference and narrowing are really the same underlying machine, applied continuously.

**Technical explanation — the two inference "modes."**
1. **Bottom-up (synthesized) inference**: computing a type from an expression's own structure, working outward from literals/operators (`5 + 3` → `number`, `{x: 1}` → `{x: number}`). This is what produces a value's "natural" type with no external expectation involved.
2. **Top-down (contextual) inference**: a surrounding context provides an *expected type*, which gets pushed down into an expression to help resolve ambiguity (Ch. 4.1's contextual typing for callback parameters is exactly this — the expected function-type parameter positions flow down into the callback literal being checked).

**Internal behavior.** These two modes **cooperate**, not compete: the checker first looks for a contextual type from the surrounding position (a parameter's declared type, a variable's annotation, a `satisfies` expression, Ch. 14); if one exists, it's used to help resolve otherwise-ambiguous parts of the expression (letting object literals, array literals, and callback parameters be typed more precisely than bottom-up inference alone could manage); if no contextual type exists, pure bottom-up/synthesized inference takes over, following the widening rules from Ch. 2.8.

**Compiler behavior.** This is why the exact same object literal can be inferred completely differently depending on where it appears:
```ts
const a = { status: "active" };                    // bottom-up: { status: string } (widened)
const b: { status: "active" | "inactive" } = { status: "active" }; // top-down: literal preserved via context
function f(x: { status: "active" | "inactive" }) {}
f({ status: "active" });                              // top-down (contextual typing at call site): literal preserved
```

**Real-world use cases.** Understanding this distinction is *the* practical skill for debugging "why isn't my type as narrow as I expected" — the answer is almost always "there was no contextual type available at that position, so bottom-up widening rules applied."

**Good practices.** When you want a literal/narrow type preserved but there's no natural contextual type at the declaration site, either add an explicit annotation, use `as const` (Ch. 2.7), or restructure so the value flows directly into a context that already expects the narrow type (e.g., pass it straight into a function call rather than through an intermediate untyped variable).
**Common mistakes.** Assuming a value's inferred type is some fixed, singular thing independent of where it's declared — inference is fundamentally *positional*, not just structural.
**Edge case.** `satisfies` (Ch. 14) is specifically designed to get contextual-type-like validation *without* losing bottom-up literal inference — a genuinely important, relatively recent addition that resolves a long-standing tension between "I want type-checking against a shape" and "I don't want to lose the precise inferred type," covered fully next chapter.

### Interview Q&A — §13.1
**Junior:** Q: What's the difference between "bottom-up" and "top-down" type inference in plain terms? A: Bottom-up infers a type purely from an expression's own content (e.g., a literal's value); top-down uses an expected type from the surrounding context (a parameter type, a variable annotation) to help resolve the expression's type, often preserving more precision than bottom-up alone would.
**Mid:** Q: Why does `const a = { status: "active" }` widen `status` to `string`, while `f({ status: "active" })` (where `f` expects a literal union) doesn't? A: In the first case there's no contextual type available at the point the object literal is being checked, so pure bottom-up inference applies, following Ch. 2.8's widening rules; in the second case, `f`'s declared parameter type provides a contextual type that's pushed down into the object literal, letting the checker validate/preserve the literal `"active"` against the expected union directly, rather than first widening and then checking.
**Senior:** Q: Why is understanding contextual typing essential for correctly diagnosing "unexpectedly widened" type bugs in a real codebase? A: The overwhelming majority of "why isn't this type narrow enough" issues stem from a value being constructed or passed through a position that has no contextual type available (an untyped intermediate variable, a generic function whose parameter type doesn't pin down the specific literal shape needed) — recognizing this pattern lets you fix the *actual* cause (add an annotation, restructure the code to flow directly into a typed context, or use `as const`/`satisfies`) instead of reaching for `as` assertions as a blunt workaround.

---

## 13.2 Control Flow Analysis (CFA) — Internals

**Intuition.** The checker doesn't compute one type per variable declaration and stop — it builds something like a flowchart of your function's execution paths and recomputes the variable's type at every meaningfully different point along that flowchart, based on everything provably true so far on the path that reaches that point.

**Technical explanation — the control flow graph.** Internally, `tsc` constructs a **Control Flow Graph (CFG)** for each function/module body: nodes represent points in the code (before/after statements, branch points, loop bodies), edges represent possible execution paths between them (including the "true" and "false" edges out of an `if`, loop-back edges, `try`/`catch`/`finally` edges). This is a real, standard compiler-construction technique (not TS-specific) — TS applies **type narrowing** as a data-flow analysis over this graph, similar in spirit to how traditional compilers perform reaching-definitions or liveness analysis, just computing *types* instead of variable liveness/definite-assignment.

**Internal behavior — how a type at a point is computed.** For a given variable reference at a given CFG node, the checker walks backward along the graph's edges leading to that node, collecting every **type guard** (narrowing condition — `typeof`, `instanceof`, equality, truthiness, discriminant property checks, user-defined type guards, assignments) that's provably true along *every* path reaching that point, and intersects/refines the variable's declared type accordingly. If two different paths reach the same point with different narrowed types for the same variable (e.g., after an `if/else` where each branch narrowed differently), the types are **unioned** back together at the merge point — this is exactly why a variable's narrowed type inside an `if` branch reverts to something wider immediately after the branch closes (unless both branches happened to leave it narrowed the same way).

**Compiler behavior — why assignments reset narrowing.** A plain reassignment (`x = someNewValue`) is itself a CFA node the checker tracks — it doesn't just forget everything about `x`, it recomputes `x`'s type at that point based on the assigned value's type, and this becomes the new "known type" flowing forward from that node, potentially wider or narrower than whatever was true immediately before.

**Real-world use cases.** This is the actual mechanism behind every narrowing example from Ch. 2.9 and 3.7 — understanding CFA explicitly is what lets you correctly predict *why* narrowing survives, gets lost, or merges back to a union in genuinely non-obvious real code (nested conditionals, loops, early returns, closures).

**Good practices.** When narrowing seems to "disappear" unexpectedly, mentally reconstruct the CFG: is there a path reaching this point where the guard *wasn't* true (e.g., a loop iteration, a closure that could run later)? If so, the checker is being correctly conservative, not buggy.
**Bad practices.** Reaching for `as`/non-null assertions the instant narrowing doesn't behave as hoped, without first reasoning through why — often a small, correct restructuring (assigning to a new `const`, adding an explicit guard) resolves it without an unsafe assertion.
**Common mistakes.** Expecting narrowing to survive into a **closure** defined inside the narrowed branch but invoked later (Ch. 2.9's exact gotcha) — the checker can't prove the outer variable wasn't reassigned between the closure's *creation* and its eventual *invocation* (which might happen on a totally different, unrelated execution path later), so it conservatively widens back inside the closure unless the variable is `const` (provably never reassigned) or explicitly re-narrowed inside the closure itself.
**Edge case.** Loops present a genuinely harder CFA case: the checker must account for the possibility that a loop body runs **zero or many** times, and that narrowing established in one iteration might not hold at the start of the next (if the narrowed variable could be reassigned somewhere in the loop body) — this is why narrowing behavior inside loops can feel less "sticky" than in straight-line code, and is worth testing explicitly rather than assuming.
**Performance.** CFA cost scales with the size and branching complexity of a function (more branches, more variables tracked, more merge points to union together) — this is a real, if usually modest, contributor to overall check time, and is part of why extremely long, deeply-branching functions are a genuine (if secondary, compared to type-level generic complexity) compile-time concern, beyond the usual code-quality/readability arguments against them.

### Interview Q&A — §13.2
**Junior:** Q: Why does a variable's narrowed type (e.g., narrowed to `string` inside an `if (typeof x === "string")` block) revert to the wider original type immediately after the `if` block ends? A: Because the narrowing guard (`typeof x === "string"`) is only provably true along the path *inside* that branch; once execution passes the branch (merging with the path that skipped it, or continues past it), the checker unions the possible types from every path that could reach that point, which includes paths where the guard wasn't true.
**Mid:** Q: Mechanically, what is a "Control Flow Graph" and why does TS build one for type checking? A: A representation of a function/program's possible execution paths as nodes (points in the code) connected by edges (branches, loops, exception paths); TS builds one specifically so it can perform narrowing as a data-flow analysis — computing, at each point, the type of a variable based on everything provably true along every path that could reach that point, rather than relying on a single static type per declaration.
**Senior:** Q: Why is narrowing generally weaker/less reliable inside loop bodies compared to straight-line code, and what's the underlying reason? A: A loop body can execute zero, one, or many times, and a narrowing guard checked at the top of an iteration doesn't necessarily still hold at the same point in a *later* iteration if the narrowed variable is reassigned anywhere within the loop body — the CFA must soundly account for the loop-back edge (execution returning to the top of the loop) potentially invalidating a narrowing that was true earlier in the same iteration, so the checker is more conservative about what it can assume stays true across iterations.

**Predict/debug — CFA trace exercise**
```ts
function process(x: string | number) {
  if (typeof x === "string") {
    x = 5; // reassignment inside the narrowed branch
  }
  console.log(x); // what's x's type here?
}
```
→ `number | string` — inside the `if`, `x` was narrowed to `string`, but then explicitly reassigned to `5` (type `number`); the checker updates `x`'s tracked type at that reassignment node to `number`. At the merge point after the `if` (which also has the "guard was false, `x` stayed `string | number` minus `string`" path from the implicit `else`... more precisely: the non-string branch had `x: number` already, and the string branch now also ends with `x: number` after reassignment) — walking this through carefully is exactly the kind of exercise that builds real CFA intuition; verify your own trace against an editor.

---

## 13.3 Narrowing Mechanisms — Full Catalog

Beyond the `typeof`/`instanceof`/`in`/discriminant patterns from Ch. 2.9, the full narrowing toolkit:

- **User-defined type guards**: `function isString(x: unknown): x is string { return typeof x === "string"; }` — the `x is string` return type annotation tells the checker "if this function returns `true`, treat the argument as narrowed to `string` at the call site," a manual escape hatch for narrowing logic too complex for the checker to infer automatically from the function body alone.
- **Assertion functions**: `function assertIsString(x: unknown): asserts x is string { if (typeof x !== "string") throw new Error(); }` — narrows the argument for the rest of the *current* control flow after the call (not just inside a conditional), because throwing is itself a control-flow-terminating event the CFA understands.
- **`assert(condition)` generic assertion**: `function assert(condition: unknown): asserts condition { if (!condition) throw new Error(); }` — narrows based on an arbitrary boolean condition rather than a specific type check, useful for general invariant assertions.
- **Discriminant property narrowing** (Ch. 2.9): the most important pattern for real-world union modeling.
- **`in` operator narrowing**: `if ("bark" in animal)` narrows a union to members that actually declare a `bark` property.
- **Array/tuple length narrowing**: checking `arr.length === 2` can narrow a general array type toward a more specific tuple-like understanding in some cases (a subtler, less commonly relied-upon narrowing form).

**Internal behavior — how user-defined type guards differ from automatic narrowing.** A type guard function's `x is T` annotation is a **manual override** — the checker doesn't verify that the function body actually, correctly implements that narrowing (it type-checks the body normally, but the `is` claim itself is trusted, similar in spirit to a type assertion, Ch. 3.4's discussion of `as`) — an incorrectly implemented type guard is a real, silent unsoundness hole a developer can introduce, unlike automatic `typeof`/`instanceof` narrowing which the checker verifies is sound by construction.

**Real-world use cases.** Type guards for validating/narrowing data from untyped boundaries (`unknown` from `JSON.parse`, Ch. 2.1) into application-specific shapes; assertion functions for precondition-checking utility functions used throughout a codebase (`assertDefined(value)` patterns).

**Good practices.** Keep user-defined type guard function bodies simple and obviously correct — since the checker trusts the `is` claim without independently verifying it matches the body's actual logic, a mismatched guard is a genuine, hard-to-catch source of unsoundness; consider covering type guards with unit tests specifically because the compiler can't catch a wrong one for you.
**Bad practices.** Writing a type guard whose body doesn't actually correspond to its claimed `is` type (e.g., `function isString(x: unknown): x is string { return x !== null; }` — compiles fine, is completely wrong) — a real, dangerous anti-pattern precisely because it looks and behaves like a normal, trusted narrowing mechanism everywhere it's used.
**Common mistakes.** Forgetting that assertion functions narrow the *rest of the current scope's control flow* after the call, not just within a conditional block — a distinct, sometimes-surprising narrowing shape compared to `if`-based type guards.
**Edge case.** Type guards and assertion functions **cannot** be arrow functions assigned to a `const` in all older TS versions/contexts in some subtle scenarios around `this`-typing and generic inference — always verify a specific pattern compiles as expected in your actual TS version rather than assuming full interchangeability between function-declaration and arrow-function-expression forms for guards, especially in generic contexts.
**Performance.** Negligible — same cost class as ordinary narrowing.

### Interview Q&A — §13.3
**Junior:** Q: What does `function isString(x: unknown): x is string {...}` let you do that a plain `boolean`-returning function doesn't? A: When used in a conditional, the checker narrows the checked variable's type based on the function's `true`/`false` result, exactly as if you'd written the equivalent `typeof`/`instanceof` check inline — a plain `boolean`-returning function gives no such narrowing, even if its internal logic does the same check.
**Mid:** Q: What's the practical difference between a user-defined type guard (`x is T`) and an assertion function (`asserts x is T`) in terms of where narrowing applies? A: A type guard narrows within the branch where it's used as a condition (e.g., inside an `if`); an assertion function narrows the variable for the remainder of the *current control flow* after the call site (since the function either returns normally, meaning the assertion held, or throws, terminating that path) — no surrounding `if` is needed for the narrowing to take effect.
**Senior/FAANG:** Q: Why are user-defined type guards considered a real, non-obvious unsoundness risk compared to built-in narrowing mechanisms like `typeof`? A: The compiler independently, structurally verifies that `typeof x === "string"` genuinely corresponds to `x` being a `string` — it's sound by construction. A user-defined type guard's `x is T` return annotation is instead a **manual claim** the developer makes; the checker type-checks the function body on its own terms but does not verify that the body's actual logic is truly equivalent to "is this an instance of `T`" — an incorrectly implemented guard (e.g., checking the wrong condition, or a condition that happens to be true for other cases too) silently introduces real unsoundness that behaves, everywhere it's used, exactly like trusted, sound narrowing, making it a particularly dangerous class of bug to have slip into a codebase.

---

## 13.4 Exhaustiveness Checking — Full Treatment

**Intuition.** Once you have a discriminated union (Ch. 2.9/5.4), exhaustiveness checking is the technique for making the compiler **prove** every variant is handled — turning "did we forget a case" from a runtime discovery into a compile-time guarantee, and critically, one that **stays** a guarantee as the union evolves.

**Technical explanation — the canonical pattern (revisited with full mechanism).**
```ts
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number }
  | { kind: "triangle"; base: number; height: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle": return Math.PI * shape.radius ** 2;
    case "square": return shape.side ** 2;
    case "triangle": return 0.5 * shape.base * shape.height;
    default:
      const _exhaustive: never = shape;
      throw new Error(`Unhandled shape kind: ${(_exhaustive as Shape).kind}`);
  }
}
```

**Internal behavior — why this works precisely.** After the last handled `case`, CFA (§13.2) has narrowed `shape`'s type, along the path reaching `default`, by successively **removing** each already-checked discriminant value from the union (this is exactly the discriminant narrowing mechanism from Ch. 2.9, applied cumulatively across the `switch`'s cases) — if all three variants were handled, the type remaining at `default` is the empty union, `never`. Assigning `shape` (typed `never` at that point) to a variable explicitly annotated `never` succeeds *only* because there's genuinely nothing left it could be; if a new variant (`{ kind: "hexagon", ... }`) is added to `Shape` and forgotten in the `switch`, the type reaching `default` becomes `{ kind: "hexagon", ... }` (not `never`), and the assignment to the `never`-typed `_exhaustive` variable becomes a **compile error**, precisely and specifically flagging the missing case.

**Compiler behavior — alternative exhaustiveness idioms.**
```ts
function assertNever(x: never): never {
  throw new Error(`Unexpected value: ${JSON.stringify(x)}`);
}
// used as: default: return assertNever(shape);
```
A small reusable `assertNever` helper (rather than an inline `const _exhaustive: never = ...`) is a common, slightly more ergonomic real-world convention — same underlying mechanism, packaged as a one-line call at each exhaustiveness site instead of a repeated inline pattern.

**Real-world use cases.** Reducer functions (Redux-style action handling), state machine transition handlers, API response variant handling, AST/parser node visitors, form-field-type-specific rendering logic — anywhere a fixed, evolving set of variants must be handled completely and reliably as the codebase grows.

**Good practices.** Apply exhaustiveness checking to **every** `switch`/`if`-chain over a discriminated union in domain-critical code — the cost (one extra `default` case) is trivial relative to the value (guaranteed detection of unhandled-variant bugs at the moment a union grows, rather than discovering the gap in production).
**Bad practices.** Adding a "catch-all" `default` case that silently does something generic (e.g., `default: return 0;`) instead of the `never`-based exhaustiveness check — this defeats the entire purpose, since it makes an unhandled-variant bug compile silently and fail (or misbehave) only at runtime, exactly the failure mode exhaustiveness checking exists to prevent.
**Common mistakes.** Using `if/else if` chains without a final `else` that performs the same `never` check — `switch`-based exhaustiveness checking is the most commonly demonstrated form, but the identical technique applies equally to an `if/else if/.../else` chain's final `else` branch.
**Edge case.** Exhaustiveness checking requires the union to actually be **discriminated** (a shared, literal-typed tag property) — attempting the same pattern against a union of types with no common discriminant property doesn't narrow cleanly case-by-case in the same way, and the `never`-check technique loses most of its precision/usefulness without a clean discriminant to switch on.
**Performance.** Zero runtime cost (the `never`-typed branch is provably unreachable and can even be optimized away or left as defensive dead code); the compile-time cost is the same as ordinary discriminant narrowing across a `switch`, negligible.

### Interview Q&A — §13.4
**Junior:** Q: What does assigning a variable to a `never`-typed local variable inside a `switch`'s `default` case accomplish? A: If every case of a discriminated union has genuinely been handled, the type remaining at `default` is `never` (nothing left it could be), so the assignment compiles; if a new union variant is added and its case is forgotten, the remaining type at `default` is no longer `never`, and the assignment becomes a compile error — precisely flagging the missing case.
**Mid:** Q: Why does exhaustiveness checking specifically require the union to be *discriminated* (have a shared literal tag property)? A: The technique relies on CFA progressively narrowing the union by eliminating already-checked discriminant values case-by-case (Ch. 2.9's discriminant narrowing) — without a common, literal-typed tag property to switch/narrow on, there's no clean mechanism for the checker to track "which variants have been ruled out so far," so the type remaining in a catch-all branch wouldn't reliably collapse to `never` even after handling every meaningfully distinct case.
**Senior:** Q: Why is exhaustiveness checking considered a higher-leverage practice than a comprehensive runtime test suite covering every current union variant? A: A test suite only verifies behavior for variants that exist and are tested for *today* — it provides no automatic protection when the union itself changes (a new variant added, an old one renamed) unless someone remembers to write a new test for it. Exhaustiveness checking is enforced by the compiler on every build, automatically, for every future change to the union — it converts "did we remember to update every switch statement when this type changed" from a process/discipline problem (relying on developers remembering, and code review catching lapses) into a structural guarantee the type system enforces unconditionally, which is strictly stronger and doesn't degrade as a codebase grows and changes hands across a team over time.

**Coding challenge — capstone exercise**
```ts
type ApiResponse<T> =
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; message: string };

function render<T>(response: ApiResponse<T>, renderData: (data: T) => string): string {
  switch (response.status) {
    case "loading": return "Loading...";
    case "success": return renderData(response.data);
    case "error": return `Error: ${response.message}`;
    default:
      const _exhaustive: never = response;
      return _exhaustive;
  }
}
// Exercise: add a fourth variant { status: "cancelled" } to ApiResponse and observe
// the exact compile error that appears at the `never` assignment — then fix it.
```

---

## Practical Exercises — Chapter 13

1. **CFA trace-through:** Take a real function from a project with nested `if`/`else` and at least one reassignment, and manually annotate, at each line, what the checker's tracked type for each relevant variable is — then verify against actual editor tooltips.
2. **Closure-narrowing bug hunt:** Deliberately construct the "narrowing lost inside a closure" scenario from §13.2, observe the resulting error, then fix it two different ways (copy to a new `const`, re-narrow inside the closure).
3. **Build `assertNever`:** Write the reusable `assertNever` helper from §13.4 and retrofit it into at least two `switch` statements over discriminated unions in existing code, then deliberately add an unhandled variant to confirm it's caught.
4. **Unsound guard hunt:** Deliberately write an incorrect user-defined type guard (claims `x is string` but the body checks something else), demonstrate that it compiles without error, then show a concrete downstream bug it silently enables.
5. **Loop narrowing edge case:** Construct a small example where narrowing established at the top of a loop body does *not* reliably hold on a later iteration (due to an in-loop reassignment), and explain precisely why the checker is right to be conservative there.

---

**Next:** Chapter 14 — Architecture & Best Practices: API design, type-safe patterns, error handling, scaling TypeScript in large codebases. Say **"next"** to continue.
