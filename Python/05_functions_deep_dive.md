# Python Mastery Course — Module 4: Functions Deep Dive

---

## 4.1 Functions Are Objects — The Foundational Fact

**Intuition.** Same as JS: functions are first-class values — assignable, passable, returnable, storable in data structures. Nothing here should surprise a JS developer conceptually. What differs is Python's calling *convention* — the rules for how positional/keyword arguments map to parameters — which is considerably richer and more explicit than JS's.

```python
def greet(name):
    return f"hi {name}"

say_hi = greet          # just another name bound to the same function object (Module 1's model)
print(say_hi("Ada"))     # "hi Ada"
print(greet.__name__)    # "greet" — functions carry rich introspectable metadata
```

**Technical note, ties to Module 0:** `def` is an *executable statement*, not a declaration — it runs top-to-bottom at the point it's reached, creating a function object and binding it to a name in the enclosing namespace. This is why there's no hoisting (already covered in Module 0.4) — worth reinforcing here since it's specifically about functions.

---

## 4.2 Positional, Keyword, Default, and the Real Calling Rules

```python
def describe(name, age, city="Unknown"):
    return f"{name}, {age}, {city}"

describe("Ada", 30)                       # positional
describe(name="Ada", age=30)               # keyword — order doesn't matter
describe("Ada", city="London", age=30)     # mixed — positional args must still come first
```

**JS comparison — this is the single biggest calling-convention divergence:** JS has no true keyword arguments at all — the "named arguments" pattern in JS (`function f({name, age}) {}`) is destructuring an object, a caller-side convention, not a language-level parameter-binding mechanism. Python's keyword arguments are a genuine part of function call syntax, checked and bound by the interpreter itself, which is why they compose with defaults, `*args`/`**kwargs`, and positional/keyword-only markers in ways JS's object-destructuring trick can't replicate (no equivalent of Python raising `TypeError: describe() got an unexpected keyword argument`).

**Default argument evaluation timing (formalized here; the mutable-default bug was Module 1's motivating example) — evaluated exactly once, at `def`-execution time:**

```python
import datetime

def log(msg, timestamp=datetime.datetime.now()):   # BUG: timestamp is fixed at import time forever
    print(f"[{timestamp}] {msg}")
```
Every call to `log` without an explicit `timestamp` gets the *same* frozen moment — not "now" at call time. The idiomatic fix mirrors Module 1's `None`-sentinel pattern exactly:
```python
def log(msg, timestamp=None):
    if timestamp is None:
        timestamp = datetime.datetime.now()
    print(f"[{timestamp}] {msg}")
```
This is the same underlying mechanism as the mutable-default bug, applied to a different symptom (stale value instead of shared mutable state) — recognizing "default arguments are evaluated once, at definition time, no exceptions" as a single unifying rule is what lets you spot both bug shapes instantly rather than memorizing them as two unrelated facts.

---

## 4.3 Positional-Only and Keyword-Only Parameters (`/` and `*`) — No JS Equivalent At All

```python
def f(pos_only, /, normal, *, kw_only):
    return pos_only, normal, kw_only

f(1, 2, kw_only=3)          # OK
f(1, normal=2, kw_only=3)   # OK
f(pos_only=1, normal=2, kw_only=3)   # TypeError — pos_only can't be passed by keyword
f(1, 2, 3)                            # TypeError — kw_only MUST be passed by keyword
```

- Everything before `/` (3.8+ syntax) can **only** be passed positionally.
- Everything after a bare `*` can **only** be passed by keyword.
- Everything between them can be passed either way (the default, familiar behavior).

**Why this exists — a genuine API-design tool, not a curiosity:** forcing positional-only prevents a parameter *name* from becoming part of your function's public contract when the name isn't meaningful or is likely to change (`len(obj, /)` — you'd never call `len(obj=x)`, and the stdlib itself uses this extensively, which is *why* this syntax was finally added in 3.8, to let pure-Python code express what C-implemented builtins had always effectively done). Forcing keyword-only prevents ambiguous or easy-to-misorder positional calls for functions with several same-typed parameters (`create_user(*, name, email, is_admin=False)` — you cannot accidentally call `create_user("bob@example.com", "Bob")` with arguments swapped, because both **must** be named at every call site).

**Interview-relevant framing:** this is a genuine library-API-design signal question — "when would you use `/` or `*` in a function signature you're designing for other people to call" is a fair mid/senior question, and "never, I've never needed it" is a weak answer once you understand the rationale above.

---

## 4.4 `*args` and `**kwargs` — Variadic Parameters and Unpacking

```python
def total(*args):            # args is a tuple of all extra positional arguments
    return sum(args)

def show(**kwargs):           # kwargs is a dict of all extra keyword arguments
    for k, v in kwargs.items():
        print(k, "=", v)

total(1, 2, 3)                 # 6
show(name="Ada", age=30)       # name = Ada  \n  age = 30
```

**The names `args`/`kwargs` are pure convention, not syntax** — the actual syntax is the `*`/`**` prefix; you could legally write `*vals, **opts` and it works identically. Every experienced Python engineer still uses `args`/`kwargs` by convention because deviating from it reads as unfamiliarity, not creativity — this is exactly analogous to always naming the error parameter `err`/`error` in a JS callback.

**Unpacking at the call site — the mirror-image operation, and where real power shows up:**

```python
def add3(a, b, c):
    return a + b + c

nums = [1, 2, 3]
print(add3(*nums))              # unpacks list into 3 positional args

kwargs_dict = {"a": 1, "b": 2, "c": 3}
print(add3(**kwargs_dict))      # unpacks dict into keyword args

# Combining both directions — a common, genuinely idiomatic pattern for wrapper/decorator functions:
def logged_call(func, *args, **kwargs):
    print(f"calling {func.__name__} with {args}, {kwargs}")
    return func(*args, **kwargs)     # forwards everything through, unchanged
```

That last pattern — a function accepting `*args, **kwargs` purely to transparently forward them — is the exact mechanism decorators rely on (Module 5) and is worth internalizing now: it's how Python achieves "wrap any function regardless of its signature" without needing generics or overloads the way TS would.

**Sequence/dict unpacking outside function calls, since it's the same `*`/`**` mechanism, JS-comparable to spread/rest:**

```python
first, *rest = [1, 2, 3, 4]     # first=1, rest=[2,3,4] — like JS's `const [first, ...rest] = arr`
*init, last = [1, 2, 3, 4]       # init=[1,2,3], last=4 — no direct JS equivalent (JS rest must be last)
merged = {**dict_a, **dict_b}    # like JS's `{...a, ...b}` — later keys win on conflict, same as JS
```

**Interview trap — argument order in a signature is fixed and enforced by the grammar:** positional-only → normal → `*args` → keyword-only → `**kwargs`, e.g. `def f(a, /, b, *args, c, **kwargs)`. Getting this order wrong is a `SyntaxError`, not a runtime surprise — but being asked to write a signature combining all five categories correctly, from memory, under interview pressure, is a genuine test of whether you've internalized the model or just pattern-matched simple cases.

---

## 4.5 Recursion — What's Genuinely Different From JS Here

**The syntax is identical to JS in spirit** (a function calling itself, a base case, a recursive case) — what differs is CPython's runtime behavior:

```python
import sys
print(sys.getrecursionlimit())   # 1000 by default

def countdown(n):
    if n == 0:
        return
    countdown(n - 1)

countdown(2000)   # RecursionError: maximum recursion depth exceeded
```

**Why the limit exists and why it's low:** CPython has **no tail-call optimization**, deliberately — Guido has stated this is intentional, because TCO makes tracebacks harder to read (a tail-call-optimized stack frame vanishes, so a crash inside deep "optimized" recursion loses the call chain that would otherwise explain how you got there) — another direct expression of the Zen of Python's "explicit/readable" values, traded off deliberately against a real performance/capability cost. Every recursive call genuinely consumes a real C stack frame plus a real Python frame object, and CPython caps the count (`sys.getrecursionlimit()`, adjustable via `sys.setrecursionlimit()`, though raising it risks a genuine C-level stack overflow/segfault, not just a clean Python exception, past a point) — this is architecturally different from JS engines, which also generally lack guaranteed TCO in practice (despite it being in the ES6 spec, no major engine ships it) but don't impose an explicit, catchable, documented recursion-count ceiling the way CPython does.

**Practical consequence — idiomatic Python actively avoids deep recursion for this reason,** where a JS/functional-leaning developer might reach for it more readily: converting deep recursion to an explicit loop with your own stack (a list used as a stack, Module 9) is a completely standard, expected refactor in Python, not a "giving up on elegance" compromise.

```python
# recursive — clean, but will blow the recursion limit on deep trees
def depth(node):
    if node is None:
        return 0
    return 1 + max(depth(node.left), depth(node.right))

# iterative equivalent using an explicit stack — the idiomatic production-safe version for
# genuinely unbounded-depth structures (e.g. deeply nested untrusted JSON, huge trees)
def depth_iterative(root):
    if root is None:
        return 0
    max_d = 0
    stack = [(root, 1)]
    while stack:
        node, d = stack.pop()
        max_d = max(max_d, d)
        if node.left: stack.append((node.left, d + 1))
        if node.right: stack.append((node.right, d + 1))
    return max_d
```

---

## 4.6 Common Mistakes / Interview Traps — Consolidated

- Trying to pass a positional-only parameter by keyword, or a keyword-only parameter positionally — both hard `TypeError`s at call time.
- Forgetting default arguments are evaluated once at `def` time — applies identically to any mutable *or* "point in time" default (Module 1's list bug and this module's `datetime.now()` bug are the same root cause).
- Getting the mandatory signature order wrong (`*args` must precede keyword-only params; `**kwargs` must be last) under interview pressure.
- Reaching instinctively for deep recursion on genuinely unbounded input (parsing untrusted deeply-nested JSON, walking an arbitrary-depth tree from user data) without considering `RecursionError` risk or the iterative alternative.
- Assuming `args`/`kwargs` are keywords — they're just conventional names; the `*`/`**` prefixes are the actual syntax.

---

## 4.7 Exercises

**Predict the output:**
```python
def f(a, b=[], *args, c, **kwargs):
    b.append(a)
    return b, args, c, kwargs

print(f(1, c=2))
print(f(3, c=4))     # does this share state with the call above? why?
```

**Core:** Write a function `make_request(url, /, *, method="GET", **headers)` and demonstrate three legal call sites and two calls that raise `TypeError`, explaining each error precisely.

**Debugging exercise:** A teammate's function signature is `def process(data, options={}):` and users report that options set by one caller "leak" into calls made by a completely unrelated caller elsewhere in the codebase. Diagnose and fix, citing the exact mechanism (this should now be automatic from Module 1 + §4.2).

**Interview — junior:** "What's the difference between `*args` and `**kwargs`? Write a function that accepts both and forwards them to another function unchanged."

**Interview — mid:** "Design a function signature for a `create_widget` API that (a) requires `name` to always be passed as `name=...` for clarity at call sites, and (b) never allows a caller to accidentally pass a stray extra positional argument that gets silently absorbed. Use `/`, `*`, and explain your choices."

**Interview — senior:** "Why does CPython impose a recursion limit instead of supporting proper tail-call optimization, and what design value does that trade off against? Given a function that recurses over user-supplied JSON of unknown depth, how would you make it production-safe?"

**Advanced / FAANG-style:** "Implement a generic `retry(func, *args, retries=3, **kwargs)` wrapper that calls `func(*args, **kwargs)`, retrying on exception up to `retries` times, and explain why the `*args, **kwargs` forwarding pattern is what makes this genuinely generic across arbitrary wrapped functions — and what its limits are (e.g., how would a caller know which underlying exceptions are worth retrying vs. not)." (Direct bridge into Module 5's decorators, which formalize exactly this pattern.)

---

*Next: Module 5 — Closures & Decorators: how Python captures enclosing scope (and the classic late-binding closure bug), then building decorators from first principles up through `functools.wraps` and parameterized decorators.*
