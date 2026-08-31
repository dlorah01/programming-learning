---
layout: default
title: "Module 7: OOP & the Python Data Model"
---

# Python Mastery Course — Module 7: OOP & the Python Data Model

---

## 7.1 The Central Idea: "The Data Model" — Everything Is Protocol Dispatch

**This is the single most important reframe in the entire course.** In JS, operators (`+`, `[]`, `in`) are mostly fixed language behavior with a few overridable escape hatches (`valueOf`, `Symbol.iterator`, `Proxy`). In Python, **almost every piece of syntax is sugar for a dunder ("double underscore") method call on an object**, uniformly, for built-in types and your own classes alike:

| Syntax | Actually calls |
|---|---|
| `a + b` | `a.__add__(b)` (falls back to `b.__radd__(a)`) |
| `len(a)` | `a.__len__()` |
| `a[i]` | `a.__getitem__(i)` |
| `a in b` | `b.__contains__(a)` |
| `for x in a` | `a.__iter__()` then repeated `__next__()` (Module 6) |
| `str(a)` | `a.__str__()` |
| `repr(a)` | `a.__repr__()` |
| `bool(a)` | `a.__bool__()` (Module 2) |
| `a == b` | `a.__eq__(b)` |
| `with a:` | `a.__enter__()` / `a.__exit__()` (Module 10) |
| `a()` | `a.__call__()` |

This is why Module 3's `for` loop worked identically over lists, dicts, and files (Module 6), why `bool()` was configurable via `__bool__`/`__len__` (Module 2), and it's the entire reason you can make *your own classes* participate fully in native Python syntax — sortable with `sorted()`, addable with `+`, iterable with `for`, indexable with `[]` — just by implementing the right dunder methods. Guido's own framing: "the data model" is Python's actual specification for what an object *is* — every built-in type is just a particularly well-optimized (usually C-implemented) example of implementing this same protocol set that you have full access to yourself.

---

## 7.2 Classes — `__init__` vs `__new__`, and Basic Mechanics

```python
class Point:
    def __init__(self, x, y):     # initializes an ALREADY-CREATED instance
        self.x = x
        self.y = y

    def __repr__(self):            # controls repr(p) and default REPL/debugger display
        return f"Point({self.x!r}, {self.y!r})"

    def __eq__(self, other):        # without this, == falls back to identity (Module 1)
        if not isinstance(other, Point):
            return NotImplemented
        return self.x == other.x and self.y == other.y

p1 = Point(1, 2)
print(p1)              # Point(1, 2) — thanks to __repr__
print(p1 == Point(1, 2))  # True — thanks to __eq__
```

**`self` is not special syntax — it's just the first parameter, by convention, receiving the instance explicitly.** JS's `this` is implicit and context-dependent (famously error-prone — `this` inside a regular function callback vs. an arrow function vs. a method differ, source of endless JS bugs); Python has no equivalent footgun because **every method receives its instance as an explicit, ordinary parameter** — `p1.method(x)` is sugar for `Point.method(p1, x)`, full stop, no dynamic-binding ambiguity ever. This is another instance of "explicit is better than implicit" resolving an entire bug class JS developers know well.

**`__new__` vs `__init__` — the real construction sequence, a genuine and common interview gap:** `__new__(cls, ...)` is a **static method** (implicitly) responsible for actually *creating* and returning the new instance (typically via `super().__new__(cls)`); `__init__(self, ...)` receives that already-created instance and *initializes* it — it returns nothing (`None`) and mutates `self` in place. `SomeClass(args)` calls `__new__` first, then, *if* `__new__` returned an instance of `cls`, calls `__init__` on it with the same args. You almost never override `__new__` in everyday code — it matters for immutable-type subclassing (you can't set attributes in `__init__` on an already-constructed immutable `int`/`str`/`tuple` subclass, so customization must happen in `__new__`, before the object is frozen) and is the mechanism behind the Singleton pattern and metaclasses (Module 8).

---

## 7.3 Inheritance vs. Composition — Same Debate as JS/TS, Python-Specific Mechanics

```python
class Animal:
    def __init__(self, name):
        self.name = name
    def speak(self):
        raise NotImplementedError

class Dog(Animal):
    def speak(self):
        return f"{self.name} says Woof"

class Cat(Animal):
    def speak(self):
        return f"{self.name} says Meow"
```

Conceptually identical to JS `class Dog extends Animal`. The genuine Python-specific complexity is **multiple inheritance** (Python allows it fully; JS/TS classes don't at all — TS uses interfaces/mixins-via-composition instead) and the **Method Resolution Order (MRO)** that resolves it:

```python
class A:
    def greet(self): return "A"
class B(A):
    def greet(self): return "B"
class C(A):
    def greet(self): return "C"
class D(B, C):
    pass

print(D().greet())           # "B" — MRO, not naive left-to-right depth-first
print(D.__mro__)              # (D, B, C, A, object) — C3 linearization order
```

**Why MRO exists / historical context:** Python uses the **C3 linearization algorithm** (adopted in Python 2.3) specifically to resolve the classic "diamond problem" — where `D(B, C)` and both `B`/`C` inherit from `A`, which parent's method wins, and in what order do you search? C3 guarantees a consistent, monotonic ordering that respects each class's own local MRO and the order bases were listed in `class D(B, C)` — `super()` calls walk *this* MRO chain, not naive parent-then-grandparent. Real interview question: "why is multiple inheritance considered risky in production code" — the honest answer is MRO complexity grows fast and becomes genuinely hard to reason about beyond simple mixin patterns, which is *why* "composition over inheritance" is at least as strong a cultural default in Python as it is in modern JS/TS/Go design guidance, despite Python's syntax fully supporting deep multiple inheritance.

**Composition, the idiomatic default for "has-a" relationships:**

```python
class Engine:
    def start(self): return "vroom"

class Car:
    def __init__(self):
        self.engine = Engine()     # Car HAS AN Engine, doesn't inherit from it
    def start(self):
        return self.engine.start()
```

**Mixins — the one multiple-inheritance pattern that stays genuinely idiomatic and low-risk:** small, focused classes providing one capability (`class JSONSerializableMixin: def to_json(self): ...`), designed *specifically* to be combined via multiple inheritance, never instantiated alone — this is the pattern the stdlib itself uses (`http.server`'s handler mixins) and is the recommended way to get multiple-inheritance's benefits without its diamond-problem risk, because mixins are designed from the start to not collide.

---

## 7.4 `dataclasses` (3.7+) — Eliminating Boilerplate, TS-Interface-Adjacent

```python
from dataclasses import dataclass, field

@dataclass
class Point:
    x: float
    y: float
    tags: list[str] = field(default_factory=list)   # NOT `tags: list = []` — Module 1's bug, solved for you
```

This single decorator auto-generates `__init__`, `__repr__`, and `__eq__` (structural, field-by-field, unlike the identity-default `object.__eq__`) based purely on the class-level type annotations — exactly the boilerplate you'd otherwise hand-write per §7.2. `field(default_factory=list)` exists *specifically* because a plain `tags: list = []` class attribute would be Module 1/4's mutable-default bug all over again (shared across every instance) — `dataclasses` bakes in the fix as the only spelling it accepts for a mutable default, which is a genuinely good, deliberate API design choice worth calling out.

**JS/TS comparison:** closest analog is a TS `interface`/`type` plus a hand-written constructor, or a class with parameter properties — but TS interfaces are purely a compile-time construct (erased at runtime, zero runtime behavior), while a Python `dataclass` is a real, runtime class with real generated methods; `isinstance()`, `repr()`, structural `==`, and (optionally) `frozen=True` immutability all genuinely exist at runtime, not just in a type checker.

```python
@dataclass(frozen=True)   # makes instances immutable after __init__ — raises on attribute reassignment
class ImmutablePoint:
    x: float
    y: float
```

**When to reach for a plain class instead:** once you need custom validation logic beyond simple defaults, complex inheritance interactions, or behavior that doesn't fit the "just a bag of typed fields" shape — `dataclass` is for data containers, not a replacement for OOP generally.

---

## 7.5 Properties — Controlled Attribute Access, No Getter/Setter Boilerplate

```python
class Temperature:
    def __init__(self, celsius):
        self._celsius = celsius

    @property
    def celsius(self):
        return self._celsius

    @celsius.setter
    def celsius(self, value):
        if value < -273.15:
            raise ValueError("below absolute zero")
        self._celsius = value

    @property
    def fahrenheit(self):                # computed, read-only — no setter defined
        return self._celsius * 9/5 + 32

t = Temperature(25)
print(t.celsius)         # 25 — looks like plain attribute access...
t.celsius = 30            # ...but this actually calls the setter, running validation
print(t.fahrenheit)       # 86.0 — computed on access, not stored
t.celsius = -300          # ValueError
```

**Why this exists / the design philosophy behind it, a genuinely important Python-vs-Java/C# contrast too:** unlike languages where you're expected to write `getX()`/`setX()` boilerplate from day one "just in case" validation is ever needed, idiomatic Python starts with a **plain public attribute** (`self.celsius = celsius`, no property at all) and only *upgrades* to a `@property` later, if and when validation/computed-access is genuinely needed — because the call-site syntax (`obj.celsius`, `obj.celsius = x`) is **identical either way**. This is a real, practical consequence of the data-model philosophy: callers never need to know or care whether `.celsius` is a plain attribute or a property-backed computed value, so you can convert one to the other without ever touching calling code — a genuine "you ain't gonna need it" argument that's much weaker in languages without this transparent upgrade path.

---

## 7.6 `__slots__` — A Real Memory/Performance Tool, Not a Curiosity

```python
class PointDict:      # default: instances back their attributes with a per-instance __dict__
    def __init__(self, x, y):
        self.x = x
        self.y = y

class PointSlots:
    __slots__ = ("x", "y")     # no per-instance __dict__ — fixed, fast attribute slots instead
    def __init__(self, x, y):
        self.x = x
        self.y = y
```

**Why it matters:** by default, every instance's attributes live in a per-instance `__dict__` (a real hash table) — flexible (you can add arbitrary new attributes at runtime, `p.new_attr = 5` just works) but memory-hungry, since every single instance carries its own dict overhead. `__slots__` tells CPython to allocate a fixed-size C-level array of attribute slots instead — meaningfully less memory per instance (genuinely significant at scale — millions of small objects, e.g. parsing millions of graph nodes or data records) and slightly faster attribute access (array-offset lookup vs. hash-table lookup), at the cost of losing the ability to dynamically add new attributes not listed in `__slots__` (and complicating multiple inheritance across slotted classes). This is a real, production-relevant optimization lever — a senior engineer processing large in-memory object collections should know `__slots__` exists and when to reach for it, not just "use `dataclass` and move on."

---

## 7.7 Common Mistakes / Interview Traps — Consolidated

- Confusing `__new__` and `__init__`'s responsibilities — a genuine, common gap even among intermediate Python developers.
- Writing `tags: list = []` in a `dataclass` instead of `field(default_factory=list)` — the exact Module 1 bug, reintroduced.
- Assuming multiple inheritance "just works" left-to-right without understanding MRO/C3 linearization — a classic senior interview probe.
- Defining `__eq__` without also considering `__hash__` — Python auto-sets `__hash__ = None` (making instances unhashable) if you define `__eq__` without also defining `__hash__`, since two "equal" objects must hash equal, and the default identity-based hash would violate that once equality is overridden. This bites people trying to put custom objects in a `set`/dict key after adding `__eq__`.
- Over-reaching for inheritance where composition would be simpler and more testable — a real, common code-review pushback in both Python and JS/TS/Go shops alike.

---

## 7.8 Exercises

> **Run a snippet locally.** Copy any code block below and feed it straight to Python from your clipboard — on macOS: `pbpaste | python3 -` — or run `python3 -`, paste, and press Ctrl-D. Predict the output first, *then* run it. A few snippets reference a helper you're asked to write, or leave an input undefined — save those to a scratch file (`python3 scratch.py`) and fill in the blank first.

**Predict the output:**
```python
class Base:
    def who(self): return "Base"
class Left(Base):
    def who(self): return "Left"
class Right(Base):
    def who(self): return "Right"
class Child(Left, Right):
    pass

print(Child().who())
print(Child.__mro__)
```

**Core:** Implement a `Vector` class supporting `+`, `-`, `==`, `repr`, and `len` (as its dimensionality) purely through dunder methods, then demonstrate it working with native `+`/`==`/`len()` syntax.

**Debugging exercise:** A developer adds `__eq__` to an existing class so instances compare by value, and existing code that stored instances in a `set()` starts raising `TypeError: unhashable type`. Explain precisely why, and fix it correctly (not by just deleting `__eq__`).

**Interview — junior:** "What's the difference between `__init__` and `__new__`? When would you ever need to override `__new__`?"

**Interview — mid:** "Convert this hand-written class with a getter/setter pair enforcing a non-negative `balance` into an idiomatic Python class using `@property`, and explain why starting with a plain attribute and 'upgrading' later is considered good Python practice."

**Interview — senior:** "You're processing 50 million small immutable coordinate objects in memory for a geospatial pipeline. Compare a `@dataclass`, a `@dataclass(slots=True)` (3.10+ shortcut for `__slots__`), and a plain `namedtuple`/`tuple` for this use case, in terms of memory and access performance, and justify a choice."

**Advanced / FAANG-style:** "Design a small mixin-based plugin system (e.g., `Loggable`, `Serializable`, `Cacheable` mixins combined via multiple inheritance into concrete classes) and explain, using MRO reasoning, how you'd guarantee predictable method resolution as the number of mixins grows — and at what point you'd recommend switching to composition instead."

---

## 7.9 Exercise Solutions

Try each exercise cold first, then check your reasoning here. Keep a short personal note on anything you got wrong — that running log is the highest-value review material as the course goes on.

**Q — Predict the output:**
```python
class Base:
    def who(self): return "Base"
class Left(Base):
    def who(self): return "Left"
class Right(Base):
    def who(self): return "Right"
class Child(Left, Right):
    pass

print(Child().who())
print(Child.__mro__)
```
**A:** `"Left"`, then `(Child, Left, Right, Base, object)`. Under C3 linearization, `Child`'s MRO walks its own listed bases in order — `Left` first, then `Right` — before falling through to their shared ancestor `Base`, so `Left.who()` is found and used first.

**Q — Core:** Implement a `Vector` class supporting `+`, `-`, `==`, `repr`, and `len` purely through dunder methods, then demonstrate it working with native syntax.
**A:**
```python
class Vector:
    def __init__(self, *components):
        self.components = components
    def __add__(self, other):
        return Vector(*(a + b for a, b in zip(self.components, other.components)))
    def __sub__(self, other):
        return Vector(*(a - b for a, b in zip(self.components, other.components)))
    def __eq__(self, other):
        return isinstance(other, Vector) and self.components == other.components
    def __repr__(self):
        return f"Vector{self.components}"
    def __len__(self):
        return len(self.components)

v1, v2 = Vector(1, 2), Vector(3, 4)
print(v1 + v2)          # Vector(4, 6)
print(v1 == Vector(1, 2))  # True
print(len(v1))            # 2
```

**Q — Debugging:** A developer adds `__eq__` to an existing class so instances compare by value, and existing code that stored instances in a `set()` starts raising `TypeError: unhashable type`. Explain precisely why, and fix it correctly.
**A:** By default, `object.__hash__` is identity-based, matching `object.__eq__`'s default identity comparison — the two stay consistent. Overriding `__eq__` to mean structural/value equality breaks that consistency (two distinct objects can now be "equal" while still having different identity-based hashes), so Python automatically sets `__hash__ = None` on the class the moment `__eq__` is defined without also defining `__hash__` — making instances unhashable specifically to prevent silently-broken dict/set behavior. Correct fix: define `__hash__` explicitly, based on the same fields used in `__eq__` (e.g. `def __hash__(self): return hash((self.x, self.y))`) — not deleting the new `__eq__`, which would abandon the actual feature being added.

**Q — Interview (junior):** "What's the difference between `__init__` and `__new__`? When would you ever need to override `__new__`?"
**A:** `__new__(cls, ...)` actually creates and returns the instance (implicitly a static method); `__init__(self, ...)` receives that already-created instance and initializes its state, returning nothing. `SomeClass(...)` calls `__new__` first, then `__init__` on whatever `__new__` returned (if it's an instance of `cls`). You'd override `__new__` when subclassing an immutable built-in type (there's no way to set state in `__init__` after the fact, since the object is already frozen by then) or to implement patterns like a Singleton that need to control instance creation itself.

**Q — Interview (mid):** Convert a hand-written getter/setter pair enforcing a non-negative `balance` into idiomatic Python using `@property`, and explain why starting with a plain attribute and "upgrading" later is good practice.
**A:**
```python
class Account:
    def __init__(self, balance):
        self._balance = balance
    @property
    def balance(self):
        return self._balance
    @balance.setter
    def balance(self, value):
        if value < 0:
            raise ValueError("balance cannot be negative")
        self._balance = value
```
Starting with a plain public attribute is good practice because the call-site syntax (`account.balance`, `account.balance = x`) is *identical* whether it's backed by a plain attribute or a `@property` — callers never need to know or care which. This means you can add validation/computed behavior later, precisely when it's actually needed, without ever touching any code that already calls into the class.

**Q — Interview (senior):** Compare `@dataclass`, `@dataclass(slots=True)`, and a plain `namedtuple`/`tuple` for 50 million small immutable coordinate objects, in terms of memory and access performance.
**A:** Plain `@dataclass`: each instance carries a per-instance `__dict__` (a real hash table) for its attributes — the most memory overhead of the three, but the most flexible (dynamic attribute addition still works). `@dataclass(slots=True)`: fixed, array-style attribute slots instead of a `__dict__` — meaningfully less memory per instance and marginally faster attribute access (array-offset lookup instead of hash-table lookup), at the cost of losing dynamic attribute addition. `namedtuple`/plain `tuple`: no per-instance `__dict__` at all, and no descriptor-based attribute machinery — generally the most memory-compact and fastest-to-construct of the three, immutable by nature, with positional (and, for `namedtuple`, named) access. At 50 million instances, the memory difference between these options is genuinely significant in aggregate — `slots=True` or `namedtuple` are the realistic choices; a plain `@dataclass` would likely be memory-prohibitive at that scale.

**Q — Advanced:** Design a mixin-based plugin system and explain, using MRO reasoning, how you'd guarantee predictable method resolution as the number of mixins grows — and when you'd switch to composition instead.
**A:** Design each mixin to be small and single-purpose (`LoggableMixin`, `SerializableMixin`, `CacheableMixin`), each implementing exactly one clearly-named method or small set of methods, and deliberately avoid having two mixins define methods with the same name unless one is explicitly meant to override the other in a well-understood way — this keeps C3 linearization's resulting MRO predictable, since there's no genuine ambiguity about which mixin's implementation should "win." As the number of mixins grows, MRO reasoning stays tractable exactly as long as this "no accidental name collisions" discipline holds; once mixins start needing to coordinate with each other's internal state, or once enough of them combine that reasoning about resolution order stops being obvious at a glance, that's the concrete signal to switch to composition instead — each capability becomes an object the main class holds a reference to (`self.logger`, `self.serializer`) and delegates to explicitly, trading a bit of call-site verbosity for a design that no longer depends on inheritance-order reasoning at all.

---

*Next: Module 8 — Descriptors & Metaclasses: the mechanism `@property` is actually built on, how `__getattr__`/`__getattribute__` differ, and how metaclasses let you customize class *creation* itself.*
