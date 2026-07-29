# Python Mastery Course — Module 8: Descriptors & Metaclasses

This module goes below `@property` and below `class` itself. It's advanced, genuinely optional for day-to-day application code, but it's exactly the material that separates "comfortable Python user" from "understands why the language behaves the way it does" — and it shows up in senior/staff-level interviews specifically to test that depth.

---

## 8.1 `__getattr__` vs. `__getattribute__` — Two Different Interception Points

**`__getattribute__`** is called for **every single attribute access** on an instance, unconditionally — `obj.x` always goes through it, even for attributes that exist normally. You almost never override this directly (it's easy to break infinite-recurse into itself, and every attribute access on every instance pays its cost) — but knowing it exists explains the next fact:

**`__getattr__`** is called **only as a fallback**, when normal attribute lookup (instance `__dict__`, then class, then MRO — the actual lookup chain `__getattribute__` implements) has already failed to find the attribute:

```python
class LazyConfig:
    def __init__(self):
        self._cache = {}

    def __getattr__(self, name):        # only fires for attributes NOT found normally
        print(f"computing {name}...")
        value = expensive_lookup(name)    # e.g. hit a database or remote config service
        self._cache[name] = value          # cache it as a REAL instance attribute
        return value
        # NOTE: setting self._cache[name] as a real attribute means next time,
        # __getattribute__ finds it directly and __getattr__ never fires again for that name
```

**Why this two-tier design exists:** it lets you implement "attribute access as a computed fallback" (lazy loading, proxying, dynamic attribute generation — think of an ORM row object exposing database columns as attributes without knowing the schema ahead of time) without paying interception overhead on *every* attribute access — only on the ones that don't already exist. This is architecturally similar in spirit to a JS `Proxy`'s `get` trap, except Python's fallback-only `__getattr__` is cheaper by default and doesn't require wrapping the object in a `Proxy` — it's a first-class hook any class gets for free.

**Common mistake:** implementing `__getattr__` and forgetting it will also be called for dunder-adjacent lookups and typo'd attribute names, potentially masking real `AttributeError`s as silently-computed nonsense — always raise `AttributeError(name)` explicitly inside `__getattr__` for names you genuinely don't recognize, rather than letting it error some other, more confusing way.

---

## 8.2 The Descriptor Protocol — What `@property` Actually Is

**Intuition:** a descriptor is any object implementing `__get__` (and optionally `__set__`/`__delete__`) that's stored as a **class attribute** — Python's attribute lookup machinery (that `__getattribute__` from §8.1) specifically checks for these methods and, if present, calls them instead of just handing back the raw stored value. `@property` is not special syntax with its own separate implementation — it's a built-in class that implements exactly this protocol.

```python
class Celsius:
    def __get__(self, instance, owner):
        print("getting!")
        return instance._celsius

    def __set__(self, instance, value):
        print("setting!")
        instance._celsius = value

class Temperature:
    celsius = Celsius()      # a descriptor INSTANCE, stored as a CLASS attribute

t = Temperature()
t.celsius = 25       # prints "setting!" — Python sees `celsius` is a descriptor and calls __set__
print(t.celsius)      # prints "getting!" then 25 — calls __get__
```

This is *precisely* what `@property` generates for you automatically — `@property`/`@x.setter` is sugar for building exactly this kind of descriptor object behind the scenes. Once you see this, `@property` stops looking like magic syntax and becomes "a convenient constructor for a very common descriptor."

**Why this matters beyond satisfying curiosity — descriptors are genuinely reusable, unlike a one-off `@property`:** a hand-rolled descriptor can be defined once and attached to *many* classes/attributes, enforcing a validation rule (e.g., "always a positive number," "always a valid email format") consistently across a whole codebase, without copy-pasting the same `@property` getter/setter pair repeatedly. This is exactly the mechanism behind libraries like Django's ORM fields (`models.CharField()` is, at its core, a descriptor) and `dataclasses`' own field machinery under the hood — recognizing "this framework's declarative field syntax is probably descriptors" is a genuine, transferable piece of senior-level pattern recognition.

**Data descriptors vs. non-data descriptors — a real, tested subtlety:** a descriptor implementing `__set__` (or `__delete__`) is a **data descriptor** and takes priority over even an instance's own `__dict__` entry of the same name; a descriptor implementing only `__get__` is a **non-data descriptor**, and an instance `__dict__` entry of the same name takes priority over it instead. This is precisely why **methods** (which are non-data descriptors under the hood — a function object implements `__get__`, which is what turns `instance.method` into a bound method with `self` already filled in) can be "shadowed" by an instance attribute of the same name, while a `@property` (a data descriptor, since it implements `__set__`) cannot be shadowed that way — a fine-grained distinction, but a real one that occasionally explains genuinely confusing bugs.

---

## 8.3 Metaclasses — Customizing Class *Creation* Itself

**Intuition, the reframe you need first:** in Python, **classes are themselves objects**, created by calling something — and that "something" is, by default, `type`. `type` is simultaneously the built-in function you already use for introspection (`type(5)` → `<class 'int'>`) *and* the default metaclass every class is an instance of:

```python
print(type(5))            # <class 'int'>       — 5 is an instance of int
print(type(int))          # <class 'type'>       — int (the class itself) is an instance of type
print(type(Temperature))  # <class 'type'>       — YOUR classes are also instances of type, by default
```

`class Foo: ...` is itself sugar — mechanically, it's roughly `Foo = type('Foo', (bases,), {namespace_dict})`, i.e. calling `type` (the metaclass) with the class's name, base classes, and body-as-a-dict. A **metaclass** is simply a custom replacement for `type` — a class that inherits from `type` and overrides `__new__`/`__init__` to customize *what happens when a class statement executes*, exactly the same way overriding `__new__`/`__init__` on an ordinary class customizes what happens when you *instantiate* it (Module 7.2) — same mechanism, one meta-level up.

```python
class UppercaseAttrMeta(type):
    def __new__(mcs, name, bases, namespace):
        uppercase_namespace = {
            (key.upper() if not key.startswith("__") else key): value
            for key, value in namespace.items()
        }
        return super().__new__(mcs, name, bases, uppercase_namespace)

class Config(metaclass=UppercaseAttrMeta):
    debug = True
    version = "1.0"

print(Config.DEBUG)     # True — the metaclass rewrote attribute names at CLASS CREATION time
```

**Why they exist / real production uses (this is not just a party trick):** enforcing structural rules across an entire family of classes at definition time (e.g. "every subclass must define a `validate()` method, or fail at import time, not at first use"), auto-registering every subclass in a global registry (plugin systems), and — genuinely the most common real-world encounter — this is exactly the mechanism ORMs (Django models, SQLAlchemy declarative base) and validation libraries (Pydantic, before its newer Rust-core versions) use to turn plain-looking class bodies with type annotations into fully validated, schema-aware classes automatically.

**The honest, senior-level guidance every experienced Python engineer gives here, and a genuine, common interview question in itself:** "metaclasses are deep magic; 99% of problems that look like they need one are better solved with a class decorator, `__init_subclass__` (a simpler 3.6+ hook specifically for 'run code when a subclass is defined,' without needing a full custom metaclass), or plain composition." Reaching for a metaclass when a simpler tool would do is a real anti-pattern flagged in code review — know the mechanism deeply, reach for it rarely.

```python
# __init_subclass__ — the lighter-weight tool that covers most "I thought I needed a metaclass" cases
class Plugin:
    registry = []
    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        Plugin.registry.append(cls)     # auto-register every subclass, no metaclass required

class MyPlugin(Plugin):
    pass

print(Plugin.registry)   # [<class '__main__.MyPlugin'>]
```

**JS/TS comparison:** JS/TS has no metaclass equivalent at all — the closest conceptual cousin is a class decorator (TS/JS's `@decorator` on a class, or manually wrapping a class definition), which can modify a class after creation but doesn't intercept the *process* of class construction itself the way a Python metaclass's `__new__` does. If you're coming from TS, treat metaclasses as a genuinely new capability, not a renamed familiar one.

---

## 8.4 Common Mistakes / Interview Traps — Consolidated

- Overriding `__getattribute__` naively and causing infinite recursion (any attribute access inside it, including `self.x`, re-triggers itself unless you route through `object.__getattribute__(self, name)`).
- Implementing `__getattr__` without raising `AttributeError` for unrecognized names, silently masking real typos/bugs.
- Confusing data vs. non-data descriptors and being surprised an instance attribute can shadow a method but not a `@property`.
- Reaching for a full metaclass where `__init_subclass__` or a simple class decorator would be simpler, more readable, and less magic-feeling to the next engineer.
- Reading library code that uses declarative-looking class bodies (Django models, Pydantic) without recognizing metaclasses/descriptors as the underlying mechanism — a real "can you read unfamiliar production code" gap.

---

## 8.5 Exercises

**Predict the output:**
```python
class Meta(type):
    def __new__(mcs, name, bases, ns):
        print(f"creating class {name}")
        return super().__new__(mcs, name, bases, ns)

class Foo(metaclass=Meta):
    pass

print("---")
f = Foo()
```
*(Pay attention to exactly when "creating class Foo" prints relative to instantiation.)*

**Core:** Implement a `PositiveNumber` descriptor (raising `ValueError` in `__set__` for non-positive values) and attach it to two unrelated classes, demonstrating the reuse that a one-off `@property` per class wouldn't give you.

**Debugging exercise:** A class defines both `__getattr__` (for lazy-loaded config values) and, separately, a genuine typo accesses `self.confg` instead of `self.config` somewhere deep in the codebase. Instead of a clear `AttributeError: confg`, the bug manifests as a mysterious remote-lookup attempt for a nonexistent key. Explain precisely why, and how correct `__getattr__` implementation would have surfaced the typo immediately instead.

**Interview — junior:** "What is `type` in Python, really? What does `type(SomeClass)` return, and why?"

**Interview — mid:** "Explain what `@property` actually is, mechanically, in terms of the descriptor protocol — don't just say 'it makes a getter/setter.'"

**Interview — senior:** "You need every subclass of a base `Handler` class to automatically register itself in a dispatch table at class-definition time. Compare implementing this via a metaclass vs. `__init_subclass__`, and justify which you'd actually ship in a production codebase and why."

**Advanced / FAANG-style:** "A junior engineer is confused why `instance.some_method` works even though they never explicitly bound `self`. Explain, using the descriptor protocol precisely, how plain functions stored as class attributes become bound methods on instance access — and why this means functions are technically non-data descriptors themselves."

---

*Next: Module 9 — Collections: the real internals of `list`/`dict`/`set` (hash tables, amortized growth), plus `tuple`, `deque`, `defaultdict`, `Counter`, `OrderedDict`, and `heapq` — what each buys you over a naive `list`/`dict`, and the Big-O of the operations you actually use daily.*
