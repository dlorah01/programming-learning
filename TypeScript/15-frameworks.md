---
layout: default
title: "Chapter 15 — Framework Integration"
---

# Chapter 15 — Framework Integration

Each of these frameworks stresses a different part of the type system covered so far — this chapter is about *applying* Chapters 1–14, not introducing new TS features.

## 15.1 React + TypeScript

**Component props — the core pattern.**
```tsx
interface ButtonProps {
  label: string;
  onClick: () => void;
  variant?: "primary" | "secondary";
  children?: React.ReactNode;
}
function Button({ label, onClick, variant = "primary", children }: ButtonProps) {
  return <button onClick={onClick} className={variant}>{label}{children}</button>;
}
```
Note the union-of-literals for `variant` (Ch. 2.2/5.4), the optional prop with a default value destructured directly (Ch. 4.2), and `React.ReactNode` (a wide, deliberately permissive type covering everything JSX can render — strings, numbers, elements, fragments, arrays, `null`).

**Generic components.**
```tsx
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
  keyExtractor: (item: T) => string;
}
function List<T>({ items, renderItem, keyExtractor }: ListProps<T>) {
  return <ul>{items.map(item => <li key={keyExtractor(item)}>{renderItem(item)}</li>)}</ul>;
}
```
This is a direct, practical application of Ch. 6's generics — `T` links `items`, `renderItem`, and `keyExtractor` together so a caller passing `User[]` gets `renderItem`/`keyExtractor` correctly typed to receive `User`, with no manual annotation needed at the call site (inference from the `items` argument, Ch. 6.1).

**Hooks and generics.**
```tsx
function useToggle(initial = false): [boolean, () => void] { // tuple return, Ch. 2.5
  const [value, setValue] = useState(initial);
  const toggle = useCallback(() => setValue(v => !v), []);
  return [value, toggle];
}
```
`useState<T>`'s tuple return (Ch. 2.5's canonical real-world tuple example) is worth re-grounding here in a fully worked custom-hook context — the same reasoning (arbitrary local naming via destructuring, exact per-position types) applies to any custom hook returning multiple related values.

**Discriminated unions for component state.**
```tsx
type FetchState<T> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: string };
```
Directly applies Ch. 2.9/13.4's "make illegal states unrepresentable" + exhaustiveness principle to the extremely common "loading/error/data" component state shape — a strong, frequently-cited senior React+TS practice, replacing all-optional state objects that permit nonsensical combinations (e.g., `loading: true` and `data` simultaneously populated).

**Event typing.** `React.MouseEvent<HTMLButtonElement>`, `React.ChangeEvent<HTMLInputElement>` — these are library-provided generic types (structurally similar in spirit to `keyof`-constrained generics, Ch. 6.4) parameterized by the specific DOM element type, giving precise `event.target` typing (a plain DOM `Event` alone wouldn't know `target` is specifically an `HTMLInputElement` with a `.value`).

**Good practices.** Model component state as a discriminated union whenever more than one boolean/optional flag could otherwise combine into a nonsensical state; use generics for genuinely reusable components (lists, tables, selects) rather than duplicating near-identical components per data type; prefer `interface` for public component prop types (Ch. 5.1's extensibility argument — consumers may want to extend/merge, especially in shared component libraries).
**Bad practices.** Typing props as `any` to avoid friction with a complex generic component; using booleans for what's really a multi-state enum-like value (`isLoading`/`isError`/`hasData` as three separate optional booleans instead of one discriminated `status` field).
**Common mistakes.** Forgetting `children` isn't automatically part of a props interface — must be explicitly declared (`children?: React.ReactNode` or, in some setups, `React.PropsWithChildren<Props>`, a small utility type wrapping this exact addition).
**Edge case.** Generic components (`function List<T>(...)`) using the arrow-function form (`const List = <T,>(props: ListProps<T>) => ...`) need a trailing comma after the type parameter (`<T,>`) in `.tsx` files specifically, to disambiguate from JSX opening-tag syntax — a small, genuinely `.tsx`-specific parser gotcha worth knowing exists.
**Performance.** No TS-specific runtime cost (types are erased, Ch. 1.2, same as anywhere else); the type-checking cost of large prop interfaces/generic components is subject to the same structural-comparison considerations as Ch. 3.1 generally, usually negligible for realistic component sizes.

### Interview Q&A — §15.1
**Junior:** Q: How do you type a React component's props? A: Define an `interface` (or `type`) describing the prop shape, and use it to annotate the function's single parameter (destructured props object).
**Mid:** Q: Why is `React.MouseEvent<HTMLButtonElement>` more useful than the plain DOM `MouseEvent` type in an `onClick` handler? A: It's a generic type parameterized by the specific element type, so `event.currentTarget` (and related properties) are typed precisely as that element (e.g., `HTMLButtonElement`, with its specific properties like `.disabled`), rather than the generic `EventTarget` a plain DOM event type would give.
**Senior:** Q: Why is modeling async component state as a discriminated union (`{status: "loading"} | {status: "success", data} | ...`) preferred over several independent optional/boolean fields? A: It directly applies the "make illegal states unrepresentable" principle (Ch. 2.9/14.3) — with independent booleans/optionals, the type permits nonsensical combinations (loading and having data simultaneously, an error present alongside a success state) that must be prevented by runtime discipline alone; a discriminated union makes those combinations structurally impossible to construct, and pairs with exhaustive `switch`/conditional rendering (Ch. 13.4) to guarantee every state is explicitly handled in the render logic, with compiler enforcement if a new state is added later.

---

## 15.2 Node.js & Express

**Typing the request/response cycle.**
```ts
import { Request, Response, NextFunction } from "express";

interface AuthenticatedRequest extends Request {
  user?: { id: string; role: string };
}
function requireAuth(req: AuthenticatedRequest, res: Response, next: NextFunction) {
  if (!req.user) { res.status(401).json({ error: "Unauthorized" }); return; }
  next();
}
```
Extending `Request` via `interface ... extends` (a scoped, local extension, Ch. 5.3) is one option; module augmentation (Ch. 11.5's full `declare module "express"` deep dive) is the alternative for adding `user` globally to *every* `Request` throughout the app rather than only where this specific extended interface is explicitly used — know both, and choose deliberately based on whether the extension should be universal or scoped.

**Typed route handlers with generics (Express 5 / typed-route patterns).**
```ts
function typedHandler<P = {}, ResBody = unknown, ReqBody = unknown>(
  handler: (req: Request<P, ResBody, ReqBody>, res: Response<ResBody>) => void
) { return handler; }

app.post<{ id: string }, User, { name: string }>("/users/:id", (req, res) => {
  req.params.id;   // string
  req.body.name;    // string
  res.json({ id: req.params.id, name: req.body.name } as User);
});
```
Express's own `Request<P, ResBody, ReqBody, ReqQuery>` generic signature is a direct, practical application of Ch. 6.2's generic interfaces, letting route params/body/response all be precisely typed per-route rather than defaulting to `any`.

**Good practices.** Validate request bodies at the actual runtime boundary (Zod/similar, Ch. 14.3's boundary principle) — TS generics on `Request<P, ResBody, ReqBody>` only assert a *shape*, they don't verify the actual incoming JSON matches it; pairing compile-time route typing with runtime validation closes the real safety gap. Prefer module augmentation for genuinely global request extensions (auth middleware's `req.user`) over scattered local interface extensions, for consistency across the whole app.
**Bad practices.** Typing `req.body` as `any` (Express's default without explicit generics) and accessing properties without any runtime validation — a very common, very real source of production bugs where the type system provides zero actual protection because the underlying `any` was never replaced with a real, validated type.
**Common mistakes.** Forgetting Express middleware types require an explicit `void` (or `Promise<void>` for async middleware) return — accidentally returning a value from a middleware function can cause confusing downstream type errors in the middleware chain's composed types.
**Edge case.** Async route handlers that throw need explicit error-handling wiring (Express doesn't automatically catch promise rejections in older major versions) — this is a runtime/framework concern more than a typing one, but it's a common, real gotcha specifically because TS's typing of the handler doesn't hint at this missing safety net at all (the types compile fine either way).
**Performance.** No meaningful TS-specific performance concern beyond normal structural typing costs for larger generic route-handler signatures.

### Interview Q&A — §15.2
**Junior:** Q: What TS mechanism from Ch. 11 would you use to add a custom `req.user` property to Express's `Request` type globally, across the whole app? A: Module augmentation (`declare module "express" { interface Request { user?: ...; } }`), which merges into Express's own `Request` interface via declaration merging.
**Mid:** Q: Does typing `req.body` with a generic type parameter actually validate the incoming JSON at runtime? A: No — it only asserts, at compile time, the *shape* TS should assume `req.body` has; it provides zero runtime verification that the actual parsed JSON matches that shape, so real safety against malformed/malicious input still requires an explicit runtime validator at the boundary.
**Senior:** Q: When would you choose a locally-scoped `interface AuthenticatedRequest extends Request` over global module augmentation for adding `req.user`? A: When the extension genuinely should be scoped — e.g., only certain routes/middleware chains actually have `user` populated (others might not, if auth is optional on some routes), and you want the type system to reflect that distinction explicitly (some handlers declared to accept plain `Request`, others specifically `AuthenticatedRequest`) rather than making `user` universally (and often inaccurately, as an always-possibly-undefined field) present on every single `Request` throughout the entire application via global augmentation.

---

## 15.3 NestJS

NestJS is the most decorator-and-metadata-dependent mainstream TS framework (Ch. 12.1/12.2 applied at full scale) — understanding it deeply *is* understanding legacy decorators + `reflect-metadata` in practice.

**Dependency injection, tied directly to Ch. 12.2.**
```ts
@Injectable()
class UserService {
  findById(id: string): User { /* ... */ return {} as User; }
}

@Controller("users")
class UserController {
  constructor(private readonly userService: UserService) {} // constructor parameter property (Ch. 4/12)

  @Get(":id")
  getUser(@Param("id") id: string): User {
    return this.userService.findById(id);
  }
}
```
`UserService` being automatically injected purely from the constructor parameter's *type* is the exact `emitDecoratorMetadata` + `reflect-metadata` mechanism from Ch. 12.2 — NestJS's entire DI container is built on reading that reflected constructor-parameter-type metadata.

**Why interface-typed dependencies need injection tokens (direct application of Ch. 12.2's limitation).**
```ts
const USER_REPOSITORY = Symbol("USER_REPOSITORY");
interface UserRepository { findById(id: string): User; }

@Injectable()
class UserService {
  constructor(@Inject(USER_REPOSITORY) private repo: UserRepository) {} // explicit token required
}
```
Because `UserRepository` is an `interface` (fully erased, Ch. 1.2/12.2 — no runtime representation for reflection to capture), NestJS cannot automatically resolve it from type reflection alone; the `@Inject(TOKEN)` pattern supplies an explicit, runtime-real key instead — this is precisely the "class-shaped types work automatically, everything else needs an explicit token" limitation from Ch. 12.2, now seen as the actual, standard, everyday NestJS idiom it produces.

**Good practices.** Use interface-based dependencies (with explicit injection tokens) for genuinely swappable implementations (e.g., a `UserRepository` interface with separate real/in-memory-test implementations) — this is good architecture *and* forces you to understand and correctly apply the token pattern, rather than defaulting to concrete classes everywhere purely because it's the path of least typing friction.
**Bad practices.** Depending on concrete classes everywhere purely to avoid the injection-token ceremony, when an interface-based, swappable dependency would be the better architectural choice (harder to unit-test in isolation, tighter coupling).
**Common mistakes.** Forgetting `experimentalDecorators: true` (and, historically, `emitDecoratorMetadata: true`) must be set in `tsconfig.json` for NestJS's DI to function at all — a common "why is dependency injection silently failing/returning undefined" onboarding issue traceable directly back to a missing/misconfigured compiler flag.
**Edge case.** NestJS's migration path (or lack thereof) toward TC39 Stage 3 decorators is directly constrained by exactly the `reflect-metadata` limitation discussed in Ch. 12.1/12.2 — verify NestJS's current documented decorator-system support if this is directly relevant to a real project, since it's an actively evolving area of the framework's own architecture.

### Interview Q&A — §15.3
**Mid:** Q: How does NestJS know which service to inject into a controller's constructor, purely from the parameter's type annotation? A: Via `emitDecoratorMetadata` + `reflect-metadata` (Ch. 12.2) — the compiler emits runtime-accessible metadata recording the constructor's parameter types (for class-shaped types), which NestJS's DI container reads at runtime to resolve and inject the correct instance.
**Senior:** Q: Why can't NestJS automatically inject a dependency typed only as an `interface`, and what's the standard fix? A: Interfaces have no runtime representation at all — full erasure (Ch. 1.2) means there's nothing for `reflect-metadata` to capture for such a parameter. The standard fix is an explicit injection token (commonly a `Symbol`, sometimes a string constant) associated with the interface, supplied via `@Inject(TOKEN)` on the constructor parameter, giving the DI container a real, runtime-resolvable key instead of relying on (for interfaces, impossible) automatic type-based reflection.

---

## 15.4 Next.js

**Typed routing/data-fetching (App Router conventions).**
```ts
interface PageProps {
  params: { slug: string };
  searchParams: { [key: string]: string | string[] | undefined };
}
export default async function Page({ params, searchParams }: PageProps) {
  const post = await getPost(params.slug);
  return <article>{post.title}</article>;
}
```
Dynamic route segment types (`params.slug`) are essentially hand-written structural contracts matching the file-system route convention (`app/blog/[slug]/page.tsx`) — Next.js doesn't (at least not universally across versions) auto-derive these purely from the filename in a fully type-safe way without additional tooling, so precisely matching the declared `params` shape to the actual route's dynamic segments is a real, manual discipline point worth verifying against current Next.js documentation and version-specific typed-routes tooling.

**API routes.**
```ts
import type { NextApiRequest, NextApiResponse } from "next"; // Pages Router
export default function handler(req: NextApiRequest, res: NextApiResponse<{ message: string }>) {
  res.status(200).json({ message: "ok" });
}
```
`NextApiResponse<T>`'s generic parameter (Ch. 6.2 again) types the shape of `.json()`'s expected argument, catching a mismatched response shape at compile time.

**Good practices.** Keep server-only vs client-only type boundaries explicit and correctly configured (React Server Components' type-checking surface, and Next's own `"use client"`/`"use server"` directive-driven boundaries) — verify current Next.js documentation for the specific TS configuration guidance for your version/router, since this area has evolved significantly across major versions and is genuinely worth checking fresh rather than relying on older training-data assumptions.
**Common mistakes.** Mismatched `params`/`searchParams` shape assumptions vs. the actual file-system route structure — since (depending on version/tooling) this correspondence may not be fully compiler-enforced, a renamed route segment can silently produce a runtime `undefined` rather than a caught compile error, unless typed-routes tooling is specifically enabled and verified for your setup.

### Interview Q&A — §15.4
**Mid/Senior:** Q: Why is it worth explicitly verifying whether your Next.js version's dynamic route `params` typing is compiler-enforced against the actual file-system route structure, rather than assuming it always is? A: The correspondence between a file-system route path (e.g., `app/blog/[slug]/page.tsx`) and a hand-declared `params: { slug: string }` type is, in many setups, a structural convention maintained by the developer rather than something the compiler automatically derives and checks against the actual folder structure — a renamed dynamic segment (`[slug]` → `[postId]`) without a corresponding update to the `params` type can silently type-check while being wrong at runtime, unless the project has specifically enabled and correctly configured typed-routes tooling; this is exactly the kind of framework-specific "does the type system actually protect me here, or just look like it does" question worth verifying directly against current docs rather than assuming.

---

## 15.5 Designing Generic Libraries

Consolidating Chapters 6–10 into the practical discipline of writing a reusable, publishable generic library.

**Principle: infer where possible, require explicitly only where necessary.**
```ts
// Good: T inferred from usage
function createStore<T>(initial: T) { /* ... */ }
const store = createStore({ count: 0 }); // T inferred, no ceremony

// Necessary: no argument to infer from, must be explicit
function createEmptyStore<T>(): Store<T> { /* ... */ }
const store2 = createEmptyStore<{ count: number }>();
```

**Principle: constrain generics to exactly what's needed (Ch. 6.4), no more.** A library function constrained to `T extends { id: string }` is usable by far more callers than one requiring a specific concrete `Entity` base class — structural, minimal constraints maximize a library's applicability (a direct, library-scale application of structural typing's core value, Ch. 3.1).

**Principle: provide precise overloads for genuinely irregular call shapes (Ch. 4.3), but prefer conditional-type-driven single signatures when the relationship between input/output shape is expressible as a computation.** Overloads don't scale well past a handful of cases and don't automatically stay consistent with each other; a well-designed conditional-type-driven generic signature (Ch. 8) computes the correct return type for *any* valid input in one signature, correctly and automatically, at the cost of more upfront type-level design effort.

**Principle: ship generated `.d.ts` (Ch. 11.3), never hand-maintained.** For any library published for others to consume, `declaration: true` (auto-generated, guaranteed-in-sync types) is non-negotiable — a hand-maintained `.d.ts` for a real library is a maintenance liability from day one.

**Principle: design the public API surface around `interface` for extension points, `type` for closed/computed shapes (Ch. 5.1).** Genuinely think through, for every exported type, "should a consumer be able to merge/extend this" — and choose the declaration form that matches that deliberate decision, not habit.

**Principle: validate the library's own types against realistic consumer code before publishing.** Writing a small "smoke test" `.ts` file that imports and uses the library the way a real consumer would (including deliberately-wrong usage that should produce errors) catches type-design mistakes that pure implementation-focused testing misses entirely — type-level "tests" are a real, valuable, underused practice for library authors.

### Interview Q&A — §15.5
**Senior/FAANG:** Q: What's the tradeoff between using function overloads versus a single conditional-type-driven generic signature when designing a library function whose return type depends on its input? A: Overloads (Ch. 4.3) are simpler to write and reason about individually, and work well for a small, fixed number of genuinely distinct call shapes, but they don't scale cleanly — each new input variant needs a new overload, they must be manually kept mutually consistent, and TS's first-match resolution order (Ch. 4.3) can silently produce the wrong overload's return type if ordering isn't carefully maintained as the set grows. A single conditional-type-driven generic signature (Ch. 8) instead *computes* the correct return type for any valid input via one expression, scales to arbitrarily many input shapes without additional signatures, and can't drift into an inconsistent state the way a hand-maintained overload list can — at the cost of genuinely harder upfront type-level design work and potentially less readable error messages for consumers who pass invalid input. Library authors generally reach for conditional types once the number of "genuinely different" call shapes grows past a small, fixed handful, or when the input-to-output relationship is a clean, expressible computation rather than a small set of unrelated special cases.

---

## Practical Exercises — Chapter 15

1. **Generic React component:** Build a fully generic, type-safe `<Table<T>>` component (columns, row data, custom cell renderers per column) using the patterns from §15.1.
2. **Express auth middleware:** Implement `req.user` typing both ways — a locally-scoped `AuthenticatedRequest extends Request` interface, and a global module augmentation — and articulate when you'd choose each.
3. **NestJS injection token:** Design a swappable `NotificationService` interface (email/SMS/push implementations) with a proper injection-token-based DI setup, demonstrating why the interface-typed dependency needs `@Inject(TOKEN)`.
4. **Library API design review:** Take a small utility function you've written, and redesign its public type signature applying every principle from §15.5 — infer-first, minimal constraints, `interface` vs `type` decision, and a smoke-test file exercising both valid and intentionally-invalid usage.

---

**Next:** Chapter 16 — The Interview Capstone: the complete mental model connecting every concept, plus a full mixed mock interview spanning junior through FAANG-level. Say **"next"** to continue.
