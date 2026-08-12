---
layout: default
title: "Python Mastery Course — Answer Key: Modules 0–18"
---

# Python Mastery Course — Answer Key: Modules 0–18

Every exercise from Modules 0–18, in order, with its answer and reasoning. Use it the way described alongside this file: attempt each exercise cold first, check here, then write your own one-line note on anything you missed.

---

## Module 0 — Foundations

**Predict the output:**
```python
print(hello())
def hello(): return "hi"
```
`NameError: name 'hello' is not defined`. Python has no hoisting — `def` is an executable statement that binds the name at the point it runs, top to bottom. At the `print(hello())` line, `hello` doesn't exist in the namespace yet.

**Core (dis.dis on if/else):** An `if/else` compiles to a comparison opcode followed by a conditional jump (`POP_JUMP_IF_FALSE` or version-equivalent) — if the condition is false, execution jumps past the `if` block's bytecode straight to the `else` block's bytecode (or past both, if there's no `else`); the operand stack holds the comparison result briefly, then it's popped by the jump instruction itself, leaving the stack in the same state either branch is taken.

**Debugging (stale `utils.py`):** Priority order: (1) check for a stale `__pycache__/*.pyc` that wasn't invalidated — delete `__pycache__` and rerun; (2) check whether they're editing a different copy of `utils.py` than the one actually being imported (wrong virtual environment, a duplicate file earlier in `sys.path`, or an installed package shadowing the local file); (3) confirm the process was actually restarted — a long-running process (REPL, server with autoreload disabled) holds the *already-imported* module object in `sys.modules` and won't re-read the file at all without a restart.

**Interview — junior:** Python source is compiled to bytecode (an intermediate representation), and that bytecode is interpreted by CPython's C evaluation loop — it's neither "purely interpreted" (re-parsing text) nor "compiled" in the sense of producing native machine code ahead of time (by default). Same overall shape as the JVM, without a default JIT step to native code.

**Interview — mid:** CPython has no JIT — every loop iteration re-dispatches through the bytecode interpreter, opcode by opcode, with real per-operation overhead that never gets cheaper no matter how many times the loop runs. The actual production fix is pushing the hot loop into C — vectorizing with NumPy/pandas, or using a C extension — not switching interpreters.

**Interview — senior:** Interpreter starts, initializes builtins/`sys`. Script source is tokenized → parsed to AST → compiled to a code object (bytecode + metadata). That code object executes inside the `__main__` module's namespace, top to bottom — every `def`/`class`/statement runs once, in order (no hoisting). Any `import` first checks `sys.modules`; if not cached, the module is located via `sys.path`, its own top-level code runs once inside a fresh namespace, and the result is cached. Business logic starts executing once its containing `def`/statement is reached in this top-to-bottom pass.

**Advanced (PyPy 5x proposal):** PyPy's JIT only helps pure-Python code — if the service relies on C extensions (most real services do: DB drivers, crypto libs, some web frameworks' C-accelerated bits), those extensions may not be PyPy-compatible at all, or may run without PyPy's JIT benefit, capping or negating the expected speedup. There's also real migration risk: subtle behavioral differences in edge cases, and PyPy's own warm-up time (the JIT needs "hot" code to kick in — short-lived scripts may see no benefit or even a regression).

---

## Module 1 — Names, Objects, Mutability

**Predict the output:**
```python
def f(x, y=[]):
    y.append(x)
    return y
print(f(1))       # [1]
print(f(2, []))     # [2] — caller supplied a fresh list, bypassing the shared default
print(f(3))         # [1, 3] — reuses the SAME default list object from the first call
```

**Core (deep_flatten without deepcopy):**
```python
def deep_flatten(nested):
    result = []
    for item in nested:
        if isinstance(item, list):
            result.extend(deep_flatten(item))
        else:
            result.append(item)
    return result
```
No `copy.deepcopy` needed because you're building a genuinely new flat list from scratch, not trying to preserve/duplicate the original nested structure — new objects are created by `append`/`extend`, never mutating the input.

**Debugging (`self.items = items.copy()` fix):** This does **not** fix the bug. The default argument `items=[]` is still evaluated once, at class-definition time, and still shared as the same object every instance that doesn't pass its own `items` receives — `.copy()` runs on that same shared default object every time, producing a fresh top-level list *each call*, which actually does resolve the mutation-sharing symptom, but only by accident of also copying. It's a working-but-non-idiomatic fix — the correct, expected fix is still `items=None` + `if items is None: items = []`, because relying on `.copy()` masks the underlying mutable-default anti-pattern rather than addressing it, and a reviewer would still flag it.

**Interview — junior:** `is` checks identity (same object); `==` checks value equality. Example: `a = [1,2]; b = [1,2]; a == b` is `True`, `a is b` is `False` — same contents, different objects.

**Interview — mid:** Default arguments are evaluated once, at `def` time, not per call — a mutable default (like `[]`) is one shared object reused and mutated across every call that doesn't override it. Fix: `def f(x=None): if x is None: x = []`.

**Interview — senior:** Likely cause: the function mutates its list argument in place (`.append`/`.extend`/`+=`) rather than treating it as read-only, and the caller didn't expect their original list to change (pass-by-object-reference — Module 1.4). Fix 1 (in-place contract, if mutation is intentional): document it clearly and name the function/parameter to signal mutation (e.g. `sort_in_place`). Fix 2 (defensive copy): `def f(lst): lst = lst.copy(); ...` inside the function, so the caller's original is never touched — costs a copy, but eliminates surprise for callers.

**Advanced:** A list is unhashable (mutable, no `__hash__`), so a tuple/frozenset containing a list is also unhashable — Python can't guarantee a hash stays constant if reachable contents can mutate underneath it. This is precisely why dict keys must be hashable: a dict's internal hash table relies on a key's hash never changing after insertion, or lookups would silently break; the mutability model directly constrains what's usable as a key.

---

## Module 2 — Numbers, Strings, Booleans, None

**Predict:**
```python
print(-7 // 2, -7 % 2, 7 // -2, 7 % -2)
# -4 1 -4 -1  (floor division rounds toward -inf; % sign follows the DIVISOR)
print(bool(""), bool("0"), bool([0]), bool({}))
# False True True False  ("0" is a non-empty string -> truthy; [0] is a non-empty list -> truthy)
print(True + True + True == 3)   # True — bool is an int subclass
```

**Core (safe divide):**
```python
def safe_divide(a, b):
    if b == 0:
        return None
    return a / b

result = safe_divide(x, y)
if result is None:      # `is None`, not `== None` — Module 1's idiom
    handle_division_by_zero()
```

**Debugging (currency cent drift):** Root cause: `float` is IEEE-754 binary floating point, which cannot represent most decimal fractions (like `0.1`) exactly in binary — repeated addition/multiplication of these inexact approximations accumulates small rounding errors over many operations. Fix: use `decimal.Decimal` for currency, which represents decimal fractions exactly, or work in integer cents throughout and only convert to a display format at the boundary.

**Interview — junior:** `False` — an empty list is falsy in Python (truthiness is based on emptiness for containers). JS's `Boolean([])` is `true` — empty arrays/objects are always truthy in JS, a genuine divergence.

**Interview — mid:** `/` is true division, always returns `float`. `//` is floor division — rounds toward negative infinity, not toward zero, so it diverges from truncation for negative operands (`-7 // 2 == -4`, not `-3`).

**Interview — senior:** `dict.get('count')` returns `None` for both "key absent" and, if the stored value is genuinely `0`, it correctly returns `0` — so `.get` alone actually *does* distinguish them correctly as long as you check `is None` specifically rather than checking falsiness (`if not d.get('count')` would incorrectly treat a real `0` the same as absent). The real trap is `if not result:` — the fix is `result = d.get('count'); if result is None: ... else: # 0 is a valid, meaningful value`.

**Advanced:** `bool` was retrofitted as an `int` subclass in 2.3 specifically to preserve backward compatibility with code that used plain `0`/`1` as booleans before a dedicated type existed. Consequence for `sum(1 for x in items if predicate(x))`: this is a real, idiomatic Python counting idiom — but you could also write `sum(predicate(x) for x in items)` directly, since `True`/`False` sum as `1`/`0` — a genuine, exploitable convenience some code relies on, and worth recognizing when reading terser code.

---

## Module 3 — Control Flow, Comprehensions, `match`

**Predict:**
```python
for i in range(3):
    if i == 1: continue
    print(i)
else:
    print("done")
# 0
# 2
# done   (the loop completed without `break`, so the else clause runs)

x = [i for i in range(3)]
print(i)
```
The second `print(i)` prints `2` — not from the comprehension (which has its own scope in Python 3 and doesn't leak `i`), but from the **for loop above it**, which does leak its loop variable (loops aren't scopes). This is a genuine trick — easy to misattribute to the comprehension.

**Core (idiomatic rewrite):**
```python
result = [x * 2 for x in data if x > 0]
```

**Debugging (`break` inside `match` inside `for`):** `break` inside a `case` block applies to the nearest enclosing **loop**, not the `match` statement — but the actual bug here is usually the opposite misconception: `match` has no fallthrough at all, so a `break` right after a matched case is redundant, not broken — the real issue is more likely that the `case` never actually matched what was expected (a pattern mismatch), so the `break` never executes at all. Diagnose by checking the actual pattern being matched against the actual data shape, not the `break` placement.

**Interview — junior:** Handlers dict → `match`:
```python
match command:
    case "start": handle_start()
    case "stop": handle_stop()
    case _: handle_unknown()
```
Prefer the dict version when handlers need to be iterated, tested independently, or extended/registered dynamically at runtime — `match` is better for structural destructuring, not simple flat value dispatch.

**Interview — mid:** Comprehensions compile to an inlined loop pattern directly in bytecode; `map()`/`filter()` with a `lambda` require a real Python function call per element (frame creation overhead) for each invocation of the lambda. The comprehension avoids that per-element call overhead entirely.

**Interview — senior:** (Open design exercise — no single correct answer; key elements: a `match` on `command.split()` with at least one guard clause like `case ["go", d] if d in valid_directions`, and a `case ["take", *items]` rest-capture. Refactor trigger: once command grammar grows complex enough to need real parsing/validation/help-text generation, a class-based Command pattern with a registry becomes more maintainable than an ever-growing `match`.)

**Advanced:** `for x in obj` calls `iter(obj)` (→ `obj.__iter__()`) once, then calls `next()` on the resulting iterator repeatedly until `StopIteration`. Because this only ever needs "give me the next item," a `for line in file` never needs the whole file in memory — each `next()` call reads just the next line off disk. `zip`/`enumerate` wrap this same protocol, producing values lazily one at a time rather than building intermediate lists.

---

## Module 4 — Functions Deep Dive

**Predict:**
```python
def f(a, b=[], *args, c, **kwargs):
    b.append(a)
    return b, args, c, kwargs
print(f(1, c=2))   # ([1], (), 2, {})
print(f(3, c=4))    # ([1, 3], (), 4, {}) — same shared default `b` from the first call
```

**Core (`make_request`):**
```python
def make_request(url, /, *, method="GET", **headers):
    ...
```
Legal: `make_request("http://x")`; `make_request("http://x", method="POST")`; `make_request("http://x", method="POST", Authorization="Bearer x")`.
Illegal: `make_request(url="http://x")` → `TypeError` (`url` is positional-only); `make_request("http://x", "POST")` → `TypeError` (`method` is keyword-only, can't be passed positionally).

**Debugging (`options={}` leaking):** Same root cause as Module 1's mutable default — one shared dict object created at `def` time, mutated by one caller's usage, visible to every other caller that doesn't pass its own `options`. Fix: `options=None`, then `if options is None: options = {}`.

**Interview — junior:** `*args` captures extra positional args as a tuple; `**kwargs` captures extra keyword args as a dict. Forwarding: `def wrapper(*args, **kwargs): return target(*args, **kwargs)`.

**Interview — mid:**
```python
def create_widget(*, name, color="blue", size=1):
    ...
```
`*` forces `name` (and everything after it) to be passed by keyword only, so a caller can never accidentally supply positional args in the wrong order — every call site is self-documenting (`create_widget(name="x", size=2)`), eliminating the "which positional arg was that" ambiguity entirely.

**Interview — senior:** No TCO because it would make tracebacks harder to read/debug — an optimized-away frame erases the call chain that would otherwise explain how a deep crash was reached, which conflicts with Python's readability/explicitness values. Production-safe fix for unknown-depth JSON recursion: convert to an iterative approach using an explicit stack (a list), or use a library (e.g. `json` with an iterative parser, or bump `sys.setrecursionlimit` cautiously only as a last resort, aware it risks a real C stack overflow past a point).

**Advanced:**
```python
def retry(func, *args, retries=3, **kwargs):
    last_exc = None
    for _ in range(retries):
        try:
            return func(*args, **kwargs)
        except Exception as e:
            last_exc = e
    raise last_exc
```
Generic because `*args, **kwargs` forwards any call signature transparently — works regardless of what `func` actually expects. Limit: it can't distinguish which exceptions are worth retrying (a `ValueError` from bad input vs. a `ConnectionError` from a flaky network) — a caller would need to pass an explicit exception type/predicate to retry on selectively.

---

## Module 5 — Closures & Decorators

**Predict:**
```python
def make_multipliers():
    return [lambda x: x * i for i in range(3)]
fns = make_multipliers()
print([f(10) for f in fns])   # [20, 20, 20] — same late-binding bug, all three close over the same i (=2)
```

**Core (`@timer`):**
```python
import functools, time
def timer(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        print(f"{func.__name__} took {time.perf_counter() - start:.4f}s")
        return result
    return wrapper
```

**Debugging (`help()` shows wrapper's generic signature):** Missing `@functools.wraps(func)` on the inner wrapper — add it, one line, above `def wrapper(...)`.

**Interview — junior:** `@decorator` above `def f(): ...` is sugar for `f = decorator(f)`, applied immediately after `f` is defined.

**Interview — mid:** Bug: loop variables aren't scoped per-iteration — all closures share the same variable, whose final value is used at call time (after the loop ends). Fix 1: default-argument capture (`lambda i=i: i`). Fix 2: a factory function returning a fresh closure per call (`def make(i): return lambda: i`).

**Interview — senior:**
```python
def cache_for(seconds):
    def decorator(func):
        cache = {}
        @functools.wraps(func)
        def wrapper(*args):
            now = time.time()
            if args in cache and now - cache[args][1] < seconds:
                return cache[args][0]
            result = func(*args)
            cache[args] = (result, now)
            return result
        return wrapper
    return decorator
```
Unhashable arguments (e.g. a `list`) would raise `TypeError` when used as a dict key (`args in cache`) — would need to require hashable args explicitly, or serialize arguments to a hashable key (e.g. via `repr`) with the tradeoffs that implies.

**Advanced:** Auth should generally wrap outermost (check auth before even considering cache or logging, so unauthorized requests never touch a cache lookup or generate log noise for data they shouldn't access) — so `@require_auth` on top, then `@cache`, then `@log` innermost, is a reasonable order; if `@log` were outermost above `@cache`, it would log every call including cache hits, which may or may not be desired — the concrete "correct" order is genuinely a requirements question, but the reasoning (auth must gate everything else) is the graded part.

---

## Module 6 — Generators & Iterators

**Predict:**
```python
def gen():
    print("start"); yield 1
    print("middle"); yield 2
    print("end")
g = gen()
print("created")   # prints immediately — body hasn't run yet
print(next(g))       # "start" then 1
print(next(g))       # "middle" then 2
```

**Core (`chunked`):**
```python
def chunked(iterable, size):
    chunk = []
    for item in iterable:
        chunk.append(item)
        if len(chunk) == size:
            yield chunk
            chunk = []
    if chunk:
        yield chunk
```

**Debugging (10GB file, two passes):** Root cause: `list(open(path))` materializes the entire file in memory at once. Redesign: since the data is now needed for two passes but generators are single-use, either (a) reopen the file for the second pass (two separate `for line in open(path):` loops — cheap, since file objects are lazy and reads are sequential), or (b) if validation and processing genuinely need to happen together per-line, combine them into a single pass instead of two.

**Interview — junior:** List comprehension eagerly builds the full list now; generator expression is lazy, producing values on demand. Wrong choice: using a generator expression when you need `len()`, indexing, or multiple passes (it only supports one forward pass, once); using a list comprehension when the sequence is huge and you only need one pass (wastes memory).

**Interview — mid:** `iter([1,2,3])` calls `__iter__()`, which explicitly constructs and returns a **new** iterator object each time — the list itself isn't an iterator, just iterable. `next()` on an iterator advances *that specific object's* internal position; the list itself holds no position state, so two separate `iter()` calls never interfere with each other.

**Interview — senior:** (Design exercise — key elements: chain generators for read → filter → transform stages, each a generator function/expression consuming the previous lazily; memory stays roughly constant throughout since nothing is materialized until consumed. Break laziness only where genuinely required — e.g. sorting needs the full sequence, so a `sorted()` call would be a deliberate, isolated point of full materialization.)

**Advanced:** `yield from sub` correctly propagates `.send()`/`.throw()` calls into the sub-generator and forwards its `return` value as the `yield from` expression's own result — a manual `for x in sub: yield x` loop does neither. Bridge to `await`: awaiting a coroutine that itself awaits another coroutine behaves the same way — the "delegation" chains correctly through multiple levels, preserving the ability to send values/exceptions down and get a final return value back up.

---

## Module 7 — OOP & the Data Model

**Predict:** `Child(Left, Right).who()` → `"Left"`. MRO for `class Child(Left, Right)` where both inherit from `Base`: `(Child, Left, Right, Base, object)` — C3 linearization checks `Child`'s own bases in the order listed, so `Left` (listed first) wins before falling through to `Right`, then `Base`.

**Core (`Vector`):**
```python
class Vector:
    def __init__(self, *components): self.components = components
    def __add__(self, other): return Vector(*(a+b for a,b in zip(self.components, other.components)))
    def __sub__(self, other): return Vector(*(a-b for a,b in zip(self.components, other.components)))
    def __eq__(self, other): return self.components == other.components
    def __repr__(self): return f"Vector{self.components}"
    def __len__(self): return len(self.components)
```

**Debugging (`__eq__` breaks `set()` usage):** Defining `__eq__` without `__hash__` causes Python to set `__hash__ = None` on the class automatically (making instances unhashable), because the old identity-based hash would no longer be consistent with the new value-based equality. Fix: define `__hash__` explicitly, consistent with `__eq__` (typically hash the same fields being compared) — don't just delete `__eq__`, since that abandons the actual feature (value-based comparison) that was being added.

**Interview — junior:** `__new__` creates and returns the instance; `__init__` initializes an already-created instance and returns nothing. Override `__new__` when subclassing an immutable type (setting state in `__init__` is too late) or implementing patterns like a Singleton.

**Interview — mid:**
```python
class Account:
    def __init__(self, balance): self._balance = balance
    @property
    def balance(self): return self._balance
    @balance.setter
    def balance(self, value):
        if value < 0: raise ValueError("negative balance")
        self._balance = value
```
Good practice because the call-site syntax (`acct.balance`, `acct.balance = x`) is identical whether backed by a plain attribute or a property — you can add validation later without ever touching calling code, so starting simple costs nothing.

**Interview — senior:** `@dataclass` (plain): per-instance `__dict__`, most memory overhead, most flexible. `@dataclass(slots=True)`: fixed attribute slots, meaningfully less memory per instance, slightly faster attribute access, loses dynamic attribute addition. `namedtuple`/plain `tuple`: most memory-compact (no per-instance dict at all, fixed C-level layout), immutable, positional access — best raw memory efficiency at some ergonomics cost. At 50M instances, `slots=True` or `namedtuple` are the realistic choices; plain `dataclass` would likely be memory-prohibitive at that scale.

**Advanced:** (Design exercise — key elements: mixins should be small, single-purpose, explicitly designed to combine (`class Handler(Loggable, Serializable, Cacheable, Base)`); reasoning about MRO stays tractable as long as mixins don't override the same method names as each other or the base. Switch to composition once mixins start needing to coordinate/depend on each other's state, or once the mixin count/interaction complexity makes MRO reasoning genuinely hard to predict at a glance.)

---

## Module 8 — Descriptors & Metaclasses

**Predict:** `"creating class Foo"` prints immediately when the `class Foo(metaclass=Meta): pass` statement executes — at **class-definition time**, before any instance is ever created — not when `f = Foo()` runs. The `"---"` prints after, then instantiation happens with no further metaclass-level output (unless `Meta` also overrides `__call__`).

**Core (`PositiveNumber` descriptor):**
```python
class PositiveNumber:
    def __set_name__(self, owner, name): self.name = "_" + name
    def __get__(self, instance, owner): return getattr(instance, self.name)
    def __set__(self, instance, value):
        if value <= 0: raise ValueError("must be positive")
        setattr(instance, self.name, value)

class Product:
    price = PositiveNumber()
class Order:
    quantity = PositiveNumber()
```
Reuse demonstrated: the same `PositiveNumber` class enforces the rule on two unrelated classes without duplicating validation logic in either.

**Debugging (typo masked by `__getattr__`):** `self.confg` (typo) triggers `__getattr__` because normal lookup fails to find `confg` — if `__getattr__` treats *any* unrecognized name as "go fetch it remotely," it silently attempts a remote lookup for a nonexistent key instead of raising `AttributeError`. Correct `__getattr__` implementation should explicitly check whether `name` is a recognized/expected config key and raise `AttributeError(name)` for anything else — which would have surfaced the typo immediately as a normal, familiar error.

**Interview — junior:** `type` is the default metaclass — the class that all ordinary classes are instances of. `type(SomeClass)` returns `<class 'type'>` (assuming `SomeClass` doesn't use a custom metaclass), because `SomeClass` itself is an object, created by calling `type`.

**Interview — mid:** `@property` is a built-in class implementing the descriptor protocol (`__get__`, and `__set__` if a setter is defined) and storing itself as a class attribute; accessing `instance.attr` is intercepted by the descriptor's `__get__` instead of returning a plain stored value — it's not special syntax, it's a reusable object that happens to have convenient decorator sugar for construction.

**Interview — senior:** `__init_subclass__` is the better production choice for straightforward auto-registration — it's simpler, more readable to teammates unfamiliar with metaclasses, and doesn't require understanding the full metaclass machinery just to grasp "every subclass gets added to a list." A full metaclass is justified only when you need to intercept/modify the class *namespace itself* at creation time (renaming/injecting attributes, enforcing structural rules across the whole class body) — auto-registration alone doesn't need that.

**Advanced:** A plain function is a non-data descriptor (implements only `__get__`, no `__set__`) — when accessed via an instance (`instance.method`), `__get__` returns a *bound method* object with `self` already partially applied, which is exactly what makes `instance.method(x)` equivalent to `Class.method(instance, x)` without any special binding syntax being needed.

---

## Module 9 — Collections

**Predict:**
```python
d = {}
d.setdefault("a", []).append(1)
d.setdefault("a", []).append(2)
print(d)   # {'a': [1, 2]} — second setdefault call finds the existing list and returns it, doesn't overwrite
```

**Core (top-5 status codes):**
```python
from collections import Counter
Counter(log["status_code"] for log in logs).most_common(5)
```
Complexity: `Counter.most_common(n)` uses a heap-based selection, O(n log k) for top-k, versus a hand-rolled `defaultdict(int)` + full `sorted()` on all unique keys, which is O(u log u) where u is the number of unique statuses — `Counter` is typically at least as good and considerably more concise/expressive.

**Debugging (`queue.pop(0)` hot path):** `list.pop(0)` is O(n) — every remaining element shifts left one slot — making a "pop from front in a loop" pattern O(n²) overall for n items. Fix: use `collections.deque` and `.popleft()`, which is O(1).

**Interview — junior:** `x in list` is O(n) (linear scan); `x in set` is O(1) average (hash table lookup). The difference is the underlying data structure — a set hashes the value directly to a bucket rather than scanning.

**Interview — mid:** Dict keys must be hashable because the hash table implementation requires a stable hash to locate a key's bucket — if the key's hash could change after insertion (as it could for a mutable object), lookups would silently break. A `list` can never be hashable (mutable, no `__hash__`); a `tuple` is hashable *only if every element it contains is itself hashable* — `(1, 2)` is fine, `(1, [2,3])` is not, because the tuple's own "shape" is fixed but its hash would need to account for the unhashable list inside it.

**Interview — senior:** Sorted list: O(n) insert (shifting), O(1) top-k via slicing — bad for frequent updates. `heapq`: O(log n) insert/update, O(k log n) to get top-k via `nlargest` — good balance for frequent updates with occasional top-k reads. A balanced structure (e.g., a skip list or a database-backed sorted set like Redis's) trades implementation complexity for both O(log n) updates and O(k) ordered range reads — justified if update *and* read frequency are both very high at scale; `heapq` is usually the pragmatic middle ground for moderate scale.

**Advanced:** `dict` gives O(1) key→node lookup; `deque` (or `OrderedDict.move_to_end`) gives O(1) reordering to mark "most recently used" and O(1) eviction from the opposite end. Combined: `get` does an O(1) dict lookup then an O(1) move-to-end; `put` does an O(1) dict insert plus O(1) move-to-end, evicting the O(1) front/back element if over capacity. A naive list-based approach would need an O(n) scan to find/reorder the accessed item, degrading the whole scheme to O(n) per operation.

---

## Module 10 — Error Handling & Context Managers

**Predict:**
```python
def f():
    try:
        raise ValueError("oops")
    except ValueError:
        return "caught"
    finally:
        print("finally ran")
print(f())
# "finally ran"    <- prints first, even though the function already has a return value queued
# "caught"           <- then the actual return happens
```

**Core (payment exception hierarchy):**
```python
class PaymentError(Exception): pass
class InsufficientFundsError(PaymentError): pass
class CardDeclinedError(PaymentError): pass

try:
    process_payment()
except InsufficientFundsError as e:
    handle_insufficient_funds(e)
except PaymentError as e:
    generic_payment_error_handler(e)
```

**Debugging (`except Exception: pass` masking corruption):** This pattern silently discards *any* exception, including ones signaling real, silent data corruption — the pipeline keeps running on bad data with zero visible trace of what went wrong, making the bug both possible (nothing stops corrupted records from proceeding) and hard to diagnose (no log, no traceback, no signal at the time of failure). Rewrite: catch only the specific, genuinely recoverable exception types, log every caught exception with context (`logger.exception(...)`), and let unexpected exception types propagate rather than being silently absorbed.

**Interview — junior:** `except Exception:` catches ordinary application errors while deliberately excluding `SystemExit`/`KeyboardInterrupt`/`GeneratorExit` (which inherit from `BaseException`, not `Exception`); a bare `except:` catches everything, including those, which is almost always wrong — you don't want to accidentally swallow a deliberate `sys.exit()` or Ctrl-C. Prefer `except Exception:`, essentially always.

**Interview — mid:** `else` runs only if the `try` block raised nothing — its purpose is keeping success-path logic out of `try`, so it isn't accidentally covered by an `except` clause meant only for the specific risky call. Rewrite:
```python
try:
    f = open(path)
except FileNotFoundError:
    return {}
else:
    return parse(f.read())   # a failure here won't be mistaken for a FileNotFoundError
```

**Interview — senior:**
```python
@contextmanager
def get_connection():
    conn = pool.acquire()
    try:
        yield conn
    finally:
        pool.release(conn)
```
Should **not** suppress exceptions raised inside the `with` block (no `return True` semantics needed here) — a DB error inside the block is a real failure the caller needs to see; suppressing it would hide genuine bugs. Cleanup (releasing the connection) still needs to happen either way, which `finally` guarantees regardless.

**Advanced:** (a) Abort-on-first-failure: simplest, but one bad record blocks 9,999 good ones — appropriate for a strict, small, correctness-critical batch. (b) Catch-and-log-per-record: maximizes throughput, appropriate for a nightly job where partial success is fine and failures get reviewed later — the standard choice for large, tolerant batch jobs. (c) Exception groups: appropriate when you want the batch to *look* atomic to the caller (fail loudly, with full detail on everything that went wrong) while still processing everything — better fit for a user-facing synchronous request where the caller needs one clear pass/fail signal with complete diagnostic detail, rather than a nightly job where (b)'s continuous logging is more operationally useful.

---

## Module 11 — Functional Tools

**Predict:**
```python
from itertools import groupby
data = [1, 1, 2, 2, 1, 1]
for key, group in groupby(data):
    print(key, list(group))
# 1 [1, 1]
# 2 [2, 2]
# 1 [1, 1]     <- NOT merged with the first group of 1s — groupby only groups CONSECUTIVE runs
```

**Core (pairs summing to target via `combinations`):**
```python
from itertools import combinations
[(a, b) for a, b in combinations(nums, 2) if a + b == target]
```
Clarity/complexity: roughly equivalent O(n²) to a nested loop, but avoids hand-writing the double-loop-with-index-guard-against-duplicates logic — `combinations` already guarantees each unordered pair exactly once, which a naive nested loop needs an explicit `j > i` guard to replicate correctly.

**Debugging (fragmented `groupby`):** Data wasn't sorted by `category` first — `groupby` only merges consecutive matching keys, so identical categories separated by other categories in the list produce separate group entries. Fix: `sorted(data, key=lambda x: x["category"])` before passing to `groupby`.

**Interview — junior:** `permutations([1,2,3], 2)` → `[(1,2),(1,3),(2,1),(2,3),(3,1),(3,2)]` (order matters, no repeats). `combinations([1,2,3], 2)` → `[(1,2),(1,3),(2,3)]` (order doesn't matter, no repeats — half as many results).

**Interview — mid:** `@functools.lru_cache(maxsize=None)` above the recursive function definition. Works because it memoizes each unique argument combination's result — overlapping subproblems (like `fib(n-2)` being recomputed many times in naive recursion) are computed once and reused, turning exponential time into linear. Constraint: all arguments must be hashable (no `list`/`dict` arguments).

**Interview — senior:** (Design exercise — key elements: `itertools.groupby` genuinely helps only if the stream is pre-sorted/pre-grouped by user ID as it arrives, which a live event stream usually isn't — more realistically, a `defaultdict(list)` (or a streaming aggregation structure) accumulating per-user state as events arrive is the practical choice, deviating from pure `itertools` laziness specifically because grouping by non-consecutive keys in an unsorted stream structurally requires either buffering/sorting or a dict-based accumulator.)

**Advanced:**
```python
def my_chain(*iterables):
    for it in iterables:
        yield from it

def my_islice(iterable, stop):
    it = iter(iterable)
    for _ in range(stop):
        yield next(it)
```
Lazy because each only pulls from its source iterator on demand, one `next()` at a time — eager versions (materializing full lists first) would hang forever on an infinite source like `itertools.count()`, since there'd be no way to ever finish building the eager intermediate list.

---

## Module 12 — Modules, Packaging, Environments

**Predict (circular import trace):** If `a.py` is imported first: `sys.modules['a']` is created and Python begins executing `a.py` top to bottom; it hits `import b`, which begins executing `b.py`; `b.py` hits `import a`, finds `a` already in `sys.modules` (even though only partially executed so far — only whatever ran before the `import b` line exists in its namespace) and uses that partial module object rather than re-running `a.py`. If `b.py` then tries to access a name from `a` defined *after* the `import b` line in `a.py`, that name doesn't exist yet on the partial module object — `AttributeError: module 'a' has no attribute '...'`.

**Core (`__init__.py` re-export):**
```
mypkg/__init__.py:  from .core import CoreThing
mypkg/core.py:       class CoreThing: ...
```
`from mypkg import CoreThing` now works because `__init__.py` runs on `import mypkg` and pulls `CoreThing` into the package's own namespace.

**Debugging (`ModuleNotFoundError` despite `pip install`):** Priority order: (1) the install happened in a different virtual environment than the one currently active/being run (most common cause); (2) no virtual environment was active at all and it installed to a different Python installation's site-packages than the one on `PATH`; (3) IDE/terminal is using a different interpreter than expected — check `which python`/`which pip` match, and confirm with `python -m pip show requests`.

**Interview — junior:** Checks whether the current module is being run directly (`__name__` is `"__main__"`) versus imported by something else (`__name__` is the module's dotted name). Wrapping entry-point logic in it lets a file be both a reusable importable module and a runnable script without import triggering script-only side effects.

**Interview — mid:** `pip install` is global/system-or-user-wide by default with no automatic per-project scoping — two projects needing incompatible versions of the same library would directly conflict. Node's `npm install` writes into a project-local `node_modules/` automatically, so isolation is the default behavior with no extra step; Python requires deliberately opting into that isolation via a virtual environment.

**Interview — senior:** (Open justification — reasonable answer: `uv` for a new service prioritizing CI speed and modern standards-compliant tooling with low onboarding friction for a Node-background team (its `pyproject.toml`+lockfile model maps closely to `package.json`+`package-lock.json`); `poetry` is an equally defensible, more battle-tested alternative; raw `pip`+`requirements.txt` is the weakest choice for a new production service specifically because of the reproducibility gap without a proper lockfile.)

**Advanced:** Fix 1 (deferred/local import): move the `import a` inside the specific function in `b.py` that needs it, so it only executes at call time (after both modules have fully finished their top-level execution), not at import time — quick, minimally invasive, but can feel like a code smell if overused. Fix 2 (extract shared logic to a third module `shared.py`, imported by both `a.py` and `b.py` instead of each other): more structurally correct, avoids the circular dependency entirely, but requires identifying and extracting exactly what's genuinely shared. Tradeoff: local imports are faster to apply but leave the underlying circular *design* dependency in place; extraction is more work but actually resolves the structural issue.

---

## Module 13 — Concurrency

**Predict:**
```python
async def main():
    await asyncio.gather(task(2), task(1), task(3))
```
Print order: `task 1 done`, `task 2 done`, `task 3 done` — in order of their `sleep` duration finishing, not the order they were listed in `gather()`, since all three start concurrently and each simply finishes whenever its own sleep completes. Total time: roughly 3 seconds (the longest individual sleep), not 6 (the sum) — they run concurrently.

**Core:** (Design exercise — `threading` version uses a `ThreadPoolExecutor` or manual `Thread`s calling a synchronous `requests.get` per URL; `asyncio` version uses `aiohttp`/`httpx.AsyncClient` with `asyncio.gather`. Choice given a synchronous ORM: `threading` — since the ORM is synchronous and can't be awaited natively, using it inside `async def` code would hit the blocking-call trap; threading avoids that entirely by not requiring the whole call chain to be async-native.)

**Debugging (`time.sleep(0.1)` degrading whole service):** `asyncio` is single-threaded and cooperative — a synchronous, blocking call like `time.sleep()` inside any `async def` handler does not yield control back to the event loop at all; it blocks the *entire* thread the event loop is running on, so every other in-flight coroutine (every other concurrent request) is stalled for that full 0.1 seconds, not just the request that called it. Under load, this compounds across many concurrent requests, degrading the whole service's throughput, not just one handler's.

**Interview — junior:** The GIL prevents more than one thread from executing Python bytecode *at the same instant*, even on a multi-core machine. It does not prevent creating multiple threads, does not prevent I/O-bound concurrency (GIL releases during blocking I/O), and does not make compound operations like `+=` atomic.

**Interview — mid:** `multiprocessing` — pure-Python CPU-heavy work gets no real parallelism from threading due to the GIL; separate processes each with their own GIL genuinely run in parallel across cores. If the filter is actually a NumPy/OpenCV call, `threading` becomes viable too, because those libraries' C implementations release the GIL during the heavy computation — the parallelism happens inside C, outside the GIL's reach.

**Interview — senior:** (Design exercise — key elements: hundreds of concurrent API calls → `asyncio` with an async HTTP client, since it's I/O-bound and scales to many more concurrent operations cheaply than threads would; CPU-heavy validation per response → `multiprocessing`, since it's genuinely CPU-bound pure-Python work needing real parallelism; DB writes → likely batched async writes if the driver supports it, or a thread pool if using a synchronous driver, since writes are I/O-bound.)

**Advanced:** `counter += 1` compiles to multiple bytecode operations (roughly: load `counter`, load `1`, add, store back to `counter`). The GIL guarantees each *individual* bytecode instruction executes atomically, but not that this whole sequence executes atomically — a thread switch can occur between, say, the "load" and the "store," so Thread A reads `counter=5`, gets switched out before storing, Thread B reads the same `counter=5`, increments and stores `6`, then Thread A resumes and stores its own computed `6` — overwriting Thread B's increment. One update is lost. Repeated across many switches, the final count ends up below the true total.

---

## Module 14 — Memory

**Predict:**
```python
gc.disable()
a = Node(); b = Node()
a.ref = b; b.ref = a
del a, b
print(gc.collect())   # returns 2 (a nonzero count) — even with gc "disabled," calling
                        # gc.collect() manually still runs a full collection cycle and
                        # finds/frees the two cyclically-referencing Node objects, since
                        # refcounting alone could never reach 0 for either (each still
                        # holds a reference from the other).
```

**Core (weakref-based observer pattern):**
```python
import weakref
class Subject:
    def __init__(self): self._observers = []
    def subscribe(self, obs): self._observers.append(weakref.ref(obs))
    def notify(self, event):
        for ref in self._observers:
            obs = ref()
            if obs is not None: obs.update(event)
```
Difference vs. a plain list: a plain `list` of strong references would keep every subscribed observer alive forever, even after nothing else in the program references them — a real, silent memory leak in long-running services. The `weakref` version lets observers be garbage collected normally once their only other references are gone, with `Subject` simply skipping dead entries on notify.

**Debugging (`TreeNode` leak):** `.parent`/`.children` references form exactly the reference-cycle shape from Module 14.2 — each node's parent points down to it while it points back up, and if the cycle collector isn't running effectively (or is disabled, or the objects are otherwise reachable in a chain that's slow to be scanned), these cycles accumulate faster than they're collected. Fix: either ensure the cycle collector remains enabled and is actually running (check `gc.isenabled()`), or break the cycle explicitly by making one direction (typically `.parent`) a `weakref` instead of a strong reference, since a parent-child tree genuinely doesn't need the parent pointer to keep the parent alive.

**Interview — junior:** Every object's refcount is tracked; the instant a reference-losing event (rebinding, `del`, going out of scope, removal from a container) drops that count to zero, the object is deallocated immediately, synchronously, at that exact point in execution — no separate GC pass needed for the common, cycle-free case.

**Interview — mid:** Reference cycles — two or more objects referencing each other (directly or through a chain) with no external reference to any of them. Example: `a.ref = b; b.ref = a` after both `a` and `b` names are deleted — each object's refcount is still 1 (from the other), never reaching 0, so pure refcounting would leak them forever. CPython's separate generational cycle-detecting `gc` module exists specifically to find and collect exactly this pattern.

**Interview — senior:** Plain `dict`: simplest, but grows unbounded, keeps every cached object alive forever regardless of memory pressure — bad fit given stated memory concerns. `lru_cache(maxsize=N)`: bounded by count, evicts least-recently-used, but eviction is based purely on access recency, not actual memory pressure, and a fixed `maxsize` may be wrong-sized if object sizes vary a lot. `WeakValueDictionary`: lets cached large objects be reclaimed automatically under real memory pressure (as soon as nothing else references them) rather than by an arbitrary count-based policy — best fit specifically because the concern is memory size, not access pattern; downside is cached values disappear as soon as nothing else holds a strong reference, which may evict things sooner than desired if nothing else in the program happens to reference them.

**Advanced:** The free-threaded (no-GIL) build has to make reference counting itself thread-safe without the GIL's blanket protection — every increment/decrement across every thread now needs to be an atomic operation (or otherwise synchronized), which is real, measurable overhead per reference operation, compounded by the fact that refcount updates happen constantly, on nearly every operation in the language (Module 14.1). This is precisely why the free-threaded build currently costs meaningful single-threaded performance — the atomic operations needed to make refcounting safe without the GIL aren't free, and the CPython team has spent years specifically mitigating this cost (biased reference counting, deferred reference counting schemes) rather than it being a simple "just remove the lock" change.

---

## Module 15 — Performance & Profiling

**Predict/reason:** `if x in results_list:` inside a loop over `n` items is O(n) per check × n iterations = **O(n²)** overall. One-line fix: convert `results_list` to a `set` before the loop (`results_set = set(results_list)`), reducing the membership check to O(1) average, making the whole loop O(n).

**Core:** (Exercise — rewrite pairwise distance loop using NumPy broadcasting, e.g. `np.linalg.norm(points_a[:, None, :] - points_b[None, :, :], axis=-1)`, then benchmark both versions with `timeit` at a realistic `n` — expect a large, measurable speedup as `n` grows, since the vectorized version avoids per-element Python bytecode dispatch entirely.)

**Debugging (`cProfile` total calls vs. cumulative time):** "Total calls" just counts how many times a function was invoked — a tiny, fast helper called a million times can have a high call count but negligible total contribution to runtime. "Cumulative time" (`cumtime`) includes time spent in that function *and everything it calls* — this is generally the more useful column for finding the actual bottleneck, since a function with high cumulative time (even if called only once) is where the real time is being spent, possibly deep in its own call tree. The teammate should have sorted by cumulative time, not call count.

**Interview — junior:** A single `time.time()` delta captures one noisy sample — affected by OS scheduling jitter, cache warm-up state, and whether a GC pause happened to land during that specific run — any of which can make a single measurement misleading. `timeit` runs many iterations and reports statistically sound aggregate timing, specifically designed for reliable micro-benchmarks.

**Interview — mid:** Diagnosis: `x in some_list` inside a loop makes the whole operation O(n²) (an O(n) scan repeated n times). Fix: convert the list to a `set` once before the loop; membership checks become O(1) average. Before: O(n²); after: O(n).

**Interview — senior:** (Process answer — key elements, in order: profile first with `cProfile` to find the actual hot function(s), not guess; once located, check whether it's an algorithmic complexity issue (Big-O smell, per Module 9/15.4) — fix that first, it's usually highest-leverage; if the hot code is genuinely numeric/array-shaped, vectorize with NumPy; if it's expensive-but-repeatable pure computation, consider `lru_cache`; if none of those apply and the work is embarrassingly parallel CPU-bound work, consider `multiprocessing` — always re-profile after each change to confirm the fix actually helped before moving to the next.)

**Advanced:** Bytecode-dispatch overhead: a hand-written Python loop summing array elements one at a time executes real bytecode instructions (`LOAD_FAST`, `BINARY_OP`, etc.) per element, each going through CPython's interpreter loop — real, per-element interpretation cost with no JIT to eliminate it (Module 0.2). Boxed-vs-unboxed: each element accessed from a NumPy array this way still gets converted to/from a full boxed Python `int`/`float` object (with its own refcount, type pointer — Module 14.1) for the Python-level arithmetic, adding allocation/deallocation overhead per element that `np.sum()` — operating entirely on the raw contiguous unboxed C buffer inside compiled C code — never pays at all.

---

## Module 16 — Type Hints & Static Analysis

**Predict:** `f("hello")` runs completely fine at runtime — Python ignores type hints entirely at execution time; the function just returns `"hello"` unchanged (whatever `x` actually is). Running `mypy` on this file would flag the call site: `error: Argument 1 to "f" has incompatible type "str"; expected "int"`.

**Core (`Drawable` Protocol):**
```python
from typing import Protocol
class Drawable(Protocol):
    def draw(self) -> str: ...

class Circle:
    def draw(self) -> str: return "○"
class Square:
    def draw(self) -> str: return "□"

def render(shape: Drawable) -> None:
    print(shape.draw())

render(Circle()); render(Square())   # both satisfy Drawable structurally, no shared base class
```

**Debugging (`mypy --strict` + `# type: ignore` masking a bug):** Likely cause: a genuinely mistyped code path was suppressed with `# type: ignore` (either to silence a real, unresolved error under time pressure, or because a third-party dependency lacked type stubs and the whole call chain fell back to effectively-`Any`), which disabled checking for exactly the code that later broke at runtime. Change: treat `# type: ignore` as requiring a specific, reviewed justification (many linters support requiring an error code alongside it) rather than a blanket silence-and-move-on, and audit/add stubs for untyped third-party dependencies rather than letting `Any` leak silently through them.

**Interview — junior:** No — Python type hints are never checked or enforced at runtime by default; CPython simply ignores them during execution. Static analysis tools (`mypy`, `pyright`) are what actually check them, as a separate, opt-in step outside normal program execution.

**Interview — mid:** `Generic`/class-based typing is **nominal** — a type satisfies a generic interface via explicit inheritance/declared relationship. `Protocol` is **structural** — any object with the right method/attribute shape satisfies it, with zero required inheritance, mirroring how Python's actual runtime behavior already works (duck typing: `for x in obj` works on anything with `__iter__` regardless of what it inherits from). `Protocol` lets the static type system express the same "if it quacks like a duck" philosophy Python has always had at runtime.

**Interview — senior:** Realistically achievable: strong static-analysis-time confidence that internally-typed code paths match their declared types, IDE-level real-time error catching, and meaningfully fewer type-related bugs shipped, especially in code the team fully controls. Gaps that remain regardless of strictness: zero runtime enforcement (a mistyped value can still flow through if the checker was skipped, a dependency wasn't checked, or a suppression comment was used), and any interaction with untyped or partially-typed third-party libraries defaults toward `Any`, silently disabling checking through that boundary — a structurally different, weaker guarantee than TypeScript's compile-gate, which should be stated plainly rather than downplayed.

**Advanced:**
```python
class Plugin(Protocol):
    def run(self, config: dict) -> Result: ...
```
`Protocol` is the right choice here specifically because plugins are independently authored (possibly by separate teams/packages) and shouldn't need to share a common base class just to satisfy the registry's expectations — a `Generic`/ABC-based approach would force every plugin author to explicitly inherit from a shared base, adding coupling that isn't actually necessary; `Protocol` gets the same static guarantee (the registry can type-check that anything passed in has a compatible `run` method) with zero inheritance requirement, matching how the plugins are actually structured and maintained.

---

## Module 17 — Testing

**Predict:** `Mock()` with no spec: `mock.nonexistent_method()` does **not** raise — it silently succeeds, auto-creating and returning yet another `Mock` object, since a bare `Mock` accepts any attribute/method access by design. `create_autospec(RealClass)` with the same call: **raises `AttributeError`**, because the autospec constrains the mock's interface to match `RealClass`'s actual, real attributes/methods — a call to something that doesn't genuinely exist on the real class fails loudly instead of silently succeeding.

**Core (fixture + parametrized tests):**
```python
@pytest.fixture
def cart():
    return ShoppingCart()

@pytest.mark.parametrize("item, price, expected_total", [
    ("apple", 1.0, 1.0),
    ("banana", 0.5, 0.5),
])
def test_add_item(cart, item, price, expected_total):
    cart.add(item, price)
    assert cart.total() == expected_total

def test_remove_item(cart):
    cart.add("apple", 1.0)
    cart.remove("apple")
    assert cart.total() == 0
```

**Debugging (mocked `PaymentGateway` masking real breakage):** The test suite mocked `PaymentGateway` broadly enough (likely a bare `Mock()`, not autospec'd) that the tests never actually exercised the real interaction contract between the code and the payment gateway — a refactor that changed how the real `PaymentGateway` needed to be called (a renamed method, a changed argument shape) wouldn't be caught, because the mock happily accepted whatever calls were made against it regardless of whether they matched the real object's actual interface. Fix: use `create_autospec(PaymentGateway)` (or `patch(..., autospec=True)`) so the mock's interface is verified against the real class, and/or add a small number of genuine integration tests against a real (test-environment) payment gateway to catch contract drift that pure mocking can't.

**Interview — junior:** `unittest` requires class-based `TestCase` subclasses and `assertX` methods (`assertEqual`, `assertTrue`, etc.); `pytest` allows plain functions with plain `assert` statements, which it rewrites to give rich, informative failure output automatically. Two concrete `pytest` advantages: (1) fixtures — a more composable, explicit dependency-injection-style setup/teardown system than `setUp`/`tearDown`; (2) it can run existing `unittest`-based suites unmodified, so adoption doesn't require a rewrite.

**Interview — mid:** Fixtures are reusable setup/teardown functions injected into tests by matching parameter names to fixture names; `yield` inside a fixture splits it into setup (before `yield`) and teardown (after, typically in `finally`). `scope="session"` creates the fixture once for the entire test run (appropriate for something genuinely expensive to set up, like a real database connection, where sharing across tests is an acceptable and desired tradeoff); the default `scope="function"` creates a fresh instance per test (appropriate whenever tests need full isolation from each other). Risk of the broader scope: shared mutable state can leak between tests, causing order-dependent test failures that are hard to diagnose.

**Interview — senior:** (Design exercise — key elements: mock the external payment API entirely — it's a true external boundary, real calls would be slow, flaky, cost money, and third-party-availability-dependent; use a real test database (not mocked) for DB interaction — DB logic (queries, transactions, schema interaction) is exactly the kind of internal-but-I/O-adjacent logic where mocking would hide real bugs, and a test database is cheap/fast/fully controllable, so the cost-benefit favors testing it for real rather than mocking it away.)

**Advanced:** (Diagnosis exercise — likely root causes: over-mocking internal collaborators (tests validate that internal code called a mock correctly, not that real behavior is correct), brittle assertions that check implementation details rather than observable outcomes, and/or a missing integration-testing layer entirely — a suite that's 100% unit tests with heavy mocking can be fast and green while genuinely broken end-to-end behavior slips through every seam between mocked components. Propose: add a thin layer of real integration tests covering the actual seams between major components, and audit existing mocks for whether they're mocking true external boundaries or just internal collaborators that should be tested for real.)

---

## Module 18 — Production Engineering

**Predict/reason:** At `INFO` level, `logger.debug(...)` calls are suppressed entirely — but the f-string form (`logger.debug(f"user data: {expensive_serialize(user)}")`) still **evaluates the f-string, and therefore still calls `expensive_serialize(user)`**, before `logger.debug()` even gets invoked — Python must construct the string argument before passing it, regardless of what the logger does with it afterward. The lazy `%s` form (`logger.debug("user data: %s", expensive_serialize(user))`) — wait, careful: this *also* still evaluates `expensive_serialize(user)` eagerly as a Python argument, for the same reason (arguments are evaluated before the function call happens). The genuine laziness `logging` provides is in *not formatting the final string* (the `%s` substitution itself) when the level is disabled — it does **not** avoid evaluating expensive arguments passed into the call. To truly avoid the expensive computation itself when DEBUG is disabled, you'd need an explicit guard: `if logger.isEnabledFor(logging.DEBUG): logger.debug("user data: %s", expensive_serialize(user))`.

**Core:** (Exercise — `pyproject.toml` with `[project.optional-dependencies] dev = ["pytest", "ruff", "mypy"]`; corresponding local commands: `ruff check .`, `ruff format --check .` (or `black --check .`), `mypy .`, `pytest`.)

**Debugging (`ImportError` only after install):** Likely cause: the package isn't using a `src/` layout, so local tests were passing via an accidental current-working-directory-based import path that happened to work during development but doesn't reflect what actually gets packaged and installed — a file, module, or `__init__.py` entry may be missing from the built distribution (misconfigured `pyproject.toml` package discovery) without it ever being caught locally, since local test runs never went through the real install step. Fix: adopt a `src/` layout so tests can only succeed by importing the genuinely installed package (`pip install -e .`), catching this exact class of packaging bug during normal local development instead of after release.

**Interview — junior:** `print()` always fires unconditionally to stdout with no severity concept and no way to redirect/filter/structure it without hand-rolling everything `logging` already provides. `logging` gives you severity levels (filterable per-deployment without code changes), multiple simultaneous destinations, and consistent structured output (timestamps, module, severity) automatically.

**Interview — mid:** A linter (`Ruff`) analyzes code for likely bugs, anti-patterns, and style violations, flagging issues without necessarily changing the code. A formatter (`Black`, or `Ruff`'s formatter mode) deterministically rewrites code into one canonical style, eliminating style debate entirely. Both are CI-enforced because manual review of style/lint issues wastes reviewer time on things a tool can catch automatically and consistently — the same reasoning that makes ESLint/Prettier CI-enforced on serious JS/TS codebases.

**Interview — senior:** (Design exercise — reasonable stage order: install deps from lockfile → lint (`ruff check`) → format check (`ruff format --check`) → type check (`mypy`/`pyright`) → tests (`pytest --cov`) → build → deploy on merge. Order reasoning: cheapest/fastest checks first, so a trivial style violation fails in seconds rather than after a multi-minute test suite runs. `src/`-layout pitfall: ensure the CI test stage genuinely installs the package (rather than relying on CWD-based imports) so packaging misconfigurations are caught before release, not after.)

**Advanced:** (Design exercise — key elements: replace the bare `except Exception: pass` with a handler that logs the full exception with `logger.error("...", exc_info=True)` at minimum, capturing the traceback; consider whether the specific failure is genuinely recoverable (log and continue) or should trigger an alert/retry/dead-letter-queue depending on severity; ensure the logging output actually reaches production monitoring — i.e., the logger is configured with a handler shipping to wherever the team's alerting/observability tooling reads from, not just to a local file nobody watches. `exc_info=True` specifically attaches the current exception's traceback to the log record, so the log entry shows exactly where and why the failure happened, not just that "something" failed.)

---

*This answer key covers every exercise in Modules 0–18. Module 19 already has its own answers built in. Recommended use: attempt the exercise in the module file first, then check here — and keep your own short note on anything you got wrong, since that personal log is the highest-value review material as you go.*
