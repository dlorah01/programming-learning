---
layout: default
title: "Module 16: Type Hints & Static Analysis"
---

# Python Mastery Course — Module 16: Type Hints & Static Analysis

---

## 16.1 The Fundamental Framing: Gradual, Optional, Erased Typing

**The single most important fact to get right, and a genuine, common point of confusion for TS developers:** Python type hints are **entirely optional, unenforced at runtime by default, and erased/ignored by the interpreter** — `def f(x: int) -> str:` does **not** raise an error if you call `f("not an int")`; CPython simply doesn't check. Type hints exist purely for **static analysis tools** (`mypy`, `pyright`), IDEs (autocomplete, inline errors), and human readers — they're documentation with tooling superpowers, not language-enforced contracts.

```python
def add(a: int, b: int) -> int:
    return a + b

print(add("hello", "world"))   # runs FINE at runtime — "helloworld" — no TypeError, hints ignored
```

**JS/TS comparison, precisely — this is the core divergence:** TypeScript is *also* erased at runtime (compiles away entirely, `tsc` produces plain JS with zero runtime type checks) — so in that specific sense, Python's hints and TS's types are architecturally similar: both are purely compile-time/static-analysis constructs with zero runtime cost or enforcement by default. The real difference is *cultural and tooling maturity*: TS's type system is mandatory-by-convention in nearly all serious codebases and its compiler (`tsc`) is a required build step that blocks bad code from shipping; Python's type hints remain genuinely optional in practice — huge amounts of production Python code have partial or no type coverage, `mypy`/`pyright` are opt-in tools you must deliberately wire into CI, and Python code runs identically whether or not it passes type checking. If you're coming from a TS-disciplined team, expect Python type coverage to be culturally patchier by default, even on serious production codebases.

---

## 16.2 Basic Syntax — Variables, Functions, Collections

```python
name: str = "Ada"
age: int = 30
scores: list[int] = [90, 85, 100]        # 3.9+ — built-in generics, no typing.List needed anymore
mapping: dict[str, int] = {"a": 1}

def greet(name: str, age: int = 0) -> str:
    return f"{name} is {age}"

from typing import Optional
# Optional[X] means "X or None" — equivalent to `X | None` (3.10+ preferred syntax)
def find_user(id: int) -> Optional[str]:      # or: -> str | None
    ...
```

**Historical note, ties to Module 0's version timeline:** before 3.9, generic collection hints required importing from `typing` (`typing.List[int]`, `typing.Dict[str, int]`) since builtin collection types themselves weren't subscriptable — 3.9's PEP 585 let you write `list[int]` directly; before 3.10, unions required `typing.Union[X, Y]`/`typing.Optional[X]` instead of the now-preferred `X | Y` syntax (PEP 604). Reading older Python code with `List`/`Dict`/`Union` imports is extremely common and not a sign of bad style — just an older-target-version codebase; know both spellings.

---

## 16.3 `Generic` — Writing Your Own Parameterized Types

```python
from typing import TypeVar, Generic

T = TypeVar("T")

class Stack(Generic[T]):
    def __init__(self) -> None:
        self._items: list[T] = []
    def push(self, item: T) -> None:
        self._items.append(item)
    def pop(self) -> T:
        return self._items.pop()

int_stack: Stack[int] = Stack()
int_stack.push(5)
int_stack.push("oops")   # mypy/pyright flag this — str isn't int; runtime doesn't care, still runs
```

**Modern syntax (3.12+, PEP 695) — genuinely nicer, worth knowing exists even if targeting older versions:**

```python
class Stack[T]:              # no more TypeVar/Generic boilerplate needed
    def push(self, item: T) -> None: ...
```

**JS/TS comparison:** structurally identical concept to TS generics (`class Stack<T>`) — if you're fluent in TS generics, `Generic[T]`/`TypeVar` (or the new 3.12 syntax) will feel immediately familiar; the underlying idea (parameterize a class/function over a type, let the checker verify consistent usage) transfers directly.

---

## 16.4 `Protocol` — Structural Typing, Genuinely Different From Nominal `Generic`/Inheritance

**This is the single most important, most distinctly "Pythonic" typing concept in this module, and the one most worth understanding deeply, because it's philosophically tied to duck typing.**

```python
from typing import Protocol

class Sized(Protocol):
    def __len__(self) -> int: ...

def print_size(obj: Sized) -> None:
    print(len(obj))

print_size([1, 2, 3])      # OK — list has __len__, satisfies Sized structurally, no inheritance needed
print_size("hello")         # OK — str has __len__ too
print_size(42)               # mypy/pyright flag this — int has no __len__
```

**Why this exists, and why it fits Python specifically:** Python has always been a **duck-typing** language — "if it walks like a duck and quacks like a duck" — `for x in obj` works on anything with `__iter__` regardless of inheritance (Module 6), `len(obj)` works on anything with `__len__` (Module 7.1) regardless of what it inherits from. `Protocol` (PEP 544, structural typing) lets the *type system* express this same duck-typing philosophy statically — a class satisfies a `Protocol` by having the right shape/methods, with **zero required inheritance relationship**, exactly mirroring how the runtime behavior already worked. This is **structural** typing — TS's `interface` works the same way (structural, not nominal — any object with matching shape satisfies a TS interface, no explicit `implements` needed) — so `Protocol` is genuinely the most direct, familiar-feeling TS-to-Python typing bridge, more so than `Generic`/class-based typing generally, which leans closer to nominal (Java/C#-style, explicit-inheritance) typing by contrast.

---

## 16.5 `TypedDict` — Typed Dictionaries, Direct TS-Interface-for-Object-Shapes Analog

```python
from typing import TypedDict

class Movie(TypedDict):
    title: str
    year: int

m: Movie = {"title": "Inception", "year": 2010}   # checker verifies keys/types match exactly
bad: Movie = {"title": "X"}                          # flagged — missing required key `year`
```

**Direct TS comparison, closest analog in this entire module:** structurally almost identical to a TS `interface { title: string; year: number }` used to type a plain object literal — `TypedDict` exists specifically because a plain Python `dict` is otherwise typed as `dict[str, Any]`-ish (or a specific uniform value type) with no per-key type distinction, mirroring the exact gap TS interfaces fill for JS object literals. Use it when you're working with dict-shaped data (very common — JSON payloads, config dicts) that you want typed precisely without upgrading to a full `dataclass`/class (Module 7.4) — `TypedDict` instances are still, at runtime, genuinely plain `dict` objects (fully erased, like everything in this module), just with static shape-checking layered on top.

---

## 16.6 `Literal` — Restricting to Specific Values, Direct TS String-Literal-Union Analog

```python
from typing import Literal

def set_mode(mode: Literal["read", "write", "append"]) -> None: ...

set_mode("read")       # OK
set_mode("delete")      # flagged — not one of the allowed literal values
```

Directly equivalent to TS's `type Mode = "read" | "write" | "append"` — genuinely one of the cleanest, most direct 1:1 mappings between the two type systems in this whole module. Idiomatic for representing a closed set of string/int "mode" or "status" values without needing a full `Enum` (though `Enum`, covered briefly below, is often the better runtime-enforced choice when the value needs to be a real, validated runtime object, not just a static-checking construct).

---

## 16.7 `mypy` vs. `pyright` — The Two Real Tools, Compared

**`mypy`** — the original, reference-implementation-adjacent type checker, Python-implemented, historically the default/most-referenced tool, integrates deeply with the `typing` module's evolution (many PEPs are co-developed with mypy's maintainers).

**`pyright`** — Microsoft's checker (the same engine powering VS Code's Python type-checking/IntelliSense), TypeScript-implemented, generally faster, and — a genuinely relevant point for a TS developer — built by the same organizational lineage that builds TypeScript's own checker, which shows in its inference quality and editor-integration polish; increasingly the default recommendation for day-to-day IDE feedback, with `mypy` still extremely common in CI pipelines and larger, established codebases.

**A fair, practical recommendation, and a legitimate interview/team-tooling question:** many teams run `pyright` (or its VS Code integration, Pylance) for fast, real-time editor feedback during development, and `mypy` in CI as the stricter, more configurable gate before merge — not mutually exclusive, genuinely complementary in practice, similar in spirit to how a team might use both an editor's live TS checking and a separate `tsc --noEmit` CI step.

---

## 16.8 What Python's Type System Genuinely Can't Guarantee, vs. TypeScript — Honest Limits

- **No runtime enforcement whatsoever by default (§16.1)** — TS at least guarantees whatever passed `tsc` is what actually ships (modulo `any`/unsafe casts); Python code with wildly wrong types passing right through `mypy`-checked code paths can still run at runtime if the checker was never actually run, skipped in CI, or the specific code path used `# type: ignore`.
- **Partial/gradual adoption is far more common and far less disciplined in practice** — a Python codebase can have `mypy --strict` on some modules and zero type coverage on others, with no build-level guarantee of consistency; large, long-lived Python codebases with genuinely comprehensive, strict typing throughout remain less common than the TS equivalent, culturally.
- **`Any` is a much easier, much more tempting escape hatch** — untyped legacy code and third-party libraries without type stubs default effectively to `Any` everywhere they're touched, silently disabling checking through that entire chain, and this happens far more pervasively in the Python ecosystem than TS's comparatively more disciplined `@types` ecosystem.

**The honest, senior-level framing for an interview:** Python's type hints are a genuinely powerful, increasingly mainstream tool for catching real bugs and improving IDE experience/documentation — but treating them as an equivalent safety guarantee to TypeScript's compile-gate is a real category error; they're best understood as "very good, IDE-integrated, optional documentation with a checker," not "the compiler will not let broken types ship."

---

## 16.9 Common Mistakes / Interview Traps — Consolidated

- Believing type hints are enforced at runtime — they are not, by default, ever (Runtime enforcement requires separate tools/patterns, like Pydantic's runtime validation, which is a deliberately different, additional thing).
- Using `typing.List`/`Dict`/`Union` reflexively in new 3.9+/3.10+ code instead of the now-preferred `list`/`dict`/`X | Y` builtin syntax.
- Reaching for a full class hierarchy/`Generic` where a `Protocol` would express the actual (structural, duck-typed) requirement more precisely and flexibly.
- Assuming `mypy`/`pyright` passing means the code is actually type-safe in production — untyped third-party dependencies, `Any` leakage, and un-run checkers in CI all create real, common gaps.
- Confusing `TypedDict` (a typing-only construct, still a real plain `dict` at runtime) with a `dataclass` (a real class with real runtime behavior) — they solve related but genuinely different problems.

---

## 16.10 Exercises

> **Run a snippet locally.** Copy any code block below and feed it straight to Python from your clipboard — on macOS: `pbpaste | python3 -` — or run `python3 -`, paste, and press Ctrl-D. Predict the output first, *then* run it. A few snippets reference a helper you're asked to write, or leave an input undefined — save those to a scratch file (`python3 scratch.py`) and fill in the blank first.

**Predict:** Given `def f(x: int) -> int: return x`, called as `f("hello")` — what happens when this code actually runs, with no type checker involved? What happens when you run `mypy` on it?

**Core:** Define a `Protocol` called `Drawable` requiring a `draw() -> str` method, write two unrelated classes (no shared base class) that both structurally satisfy it, and a function accepting `Drawable` that works with both.

**Debugging exercise:** A codebase has `mypy --strict` configured but a specific module is full of `# type: ignore` comments and still ships a runtime `AttributeError` that a correct type hint would have caught. Explain what likely went wrong in the team's typing discipline, and what you'd change.

**Interview — junior:** "Do Python type hints get checked at runtime? What actually enforces them, if anything?"

**Interview — mid:** "Explain the difference between `Generic`/class-based typing and `Protocol`-based structural typing in Python, and connect `Protocol` to Python's existing duck-typing philosophy."

**Interview — senior:** "Your team, coming from a strict TypeScript codebase, is adopting Python for a new service and wants the same level of type safety guarantee. What would you tell them is realistically achievable with `mypy`/`pyright`, and what gaps remain no matter how strict the configuration is?"

**Advanced / FAANG-style:** "Design the type hints for a small plugin-registry system where plugins must implement a `run(config: dict) -> Result` shape, without requiring plugins to inherit from a common base class. Justify using `Protocol` over `Generic`/ABC-based inheritance for this specific requirement."

---

## 16.11 Exercise Solutions

Try each exercise cold first, then check your reasoning here. Keep a short personal note on anything you got wrong — that running log is the highest-value review material as the course goes on.

**Q — Predict the output:**
```python
def f(x: int) -> int:
    return x
f("hello")
```
**A:** At runtime, this executes completely fine and returns `"hello"` unchanged — Python ignores type hints entirely during execution; there is no built-in check enforcing that `x` is actually an `int`. Running `mypy` (or `pyright`) on this file, by contrast, flags the call: something like `error: Argument 1 to "f" has incompatible type "str"; expected "int"` — the checker catches it statically, but nothing about running the program itself does.

**Q — Core:** Define a `Protocol` called `Drawable` requiring `draw() -> str`, and write two unrelated classes that both structurally satisfy it.
**A:**
```python
from typing import Protocol

class Drawable(Protocol):
    def draw(self) -> str: ...

class Circle:
    def draw(self) -> str:
        return "○"

class Square:
    def draw(self) -> str:
        return "□"

def render(shape: Drawable) -> None:
    print(shape.draw())

render(Circle())
render(Square())
```
Neither `Circle` nor `Square` inherits from `Drawable` or from each other — both satisfy the protocol purely by having a matching `draw() -> str` method, which is exactly the structural-typing point.

**Q — Debugging:** A codebase has `mypy --strict` configured, but a specific module full of `# type: ignore` comments still ships a runtime `AttributeError` a correct type hint would have caught. What went wrong, and what would you change?
**A:** Likely, `# type: ignore` was used to silence a real, unresolved type error under time pressure (or because a dependency lacked type stubs and the whole call chain effectively degraded to `Any`), which disabled checking for exactly the code path that later broke. Change: require a specific, reviewed justification for every `# type: ignore` (many linters can enforce requiring a specific error code alongside it, rather than a blanket suppression), and audit/add proper type stubs for untyped third-party dependencies rather than letting `Any` silently propagate through them unchecked.

**Q — Interview (junior):** "Do Python type hints get checked at runtime? What actually enforces them, if anything?"
**A:** No — never, by default. CPython simply ignores type hints during execution; nothing about `def f(x: int):` restricts what's actually passed at runtime. Enforcement, such as it is, comes entirely from separate, opt-in static analysis tools (`mypy`, `pyright`) run outside normal program execution — and even then, only for whatever code those tools actually check.

**Q — Interview (mid):** "Explain the difference between `Generic`/class-based typing and `Protocol`-based structural typing, and connect `Protocol` to Python's existing duck-typing philosophy."
**A:** `Generic`/class-based typing is **nominal** — a type is only considered to satisfy an interface through an explicit, declared inheritance relationship. `Protocol` is **structural** — any object whose methods/attributes match the required shape satisfies it, with zero required inheritance at all. This maps directly onto how Python's runtime has always behaved: `for x in obj` works on anything implementing `__iter__`, `len(obj)` works on anything implementing `__len__`, regardless of what that object actually inherits from (duck typing). `Protocol` lets the *static type checker* verify this same "if it has the right shape, it qualifies" philosophy ahead of time, rather than only discovering a shape mismatch at runtime.

**Q — Interview (senior):** A team from a strict TypeScript codebase wants equivalent type safety in Python. What's realistically achievable, and what gaps remain regardless of strictness?
**A:** Realistically achievable: strong static-analysis-time confidence within code the team fully controls, real-time IDE error catching, and a meaningfully reduced rate of type-related bugs shipped in that internally-typed code. Gaps that remain no matter how strict the configuration: zero runtime enforcement whatsoever — a value of the wrong type can still flow through in production if a checker was skipped in CI, a code path was suppressed with `# type: ignore`, or an untyped third-party dependency defaults to `Any` and silently disables checking through that entire boundary. This is a structurally weaker guarantee than TypeScript's compile-gate (where `tsc` genuinely blocks a build), and it should be stated to the team plainly rather than implied to be equivalent.

**Q — Advanced:** Design type hints for a plugin registry where plugins must implement `run(config: dict) -> Result`, without requiring inheritance from a common base class — justify `Protocol` over `Generic`/ABC-based inheritance.
**A:**
```python
from typing import Protocol

class Plugin(Protocol):
    def run(self, config: dict) -> Result: ...

def register(plugin: Plugin) -> None: ...
```
`Protocol` is the right choice specifically because plugins may be authored independently, possibly by separate teams or external packages, and shouldn't need to import and inherit from a shared base class just to be usable by the registry — a `Generic`/ABC-based approach would force that inheritance coupling on every plugin author. `Protocol` gets the same static guarantee (the registry can verify any given plugin object has a compatible `run` method before accepting it) with zero required inheritance relationship, which matches how genuinely decoupled plugin authorship actually works in practice.

---

*Next: Module 17 — Testing: `unittest` vs. `pytest`, fixtures, mocking, and what a genuinely good test suite looks like in a production Python codebase.*
