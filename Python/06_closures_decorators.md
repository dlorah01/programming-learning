---
layout: default
title: "Module 5: Closures & Decorators"
---

# Python Mastery Course — Module 5: Closures & Decorators

---

## 5.1 Closures — What They Actually Capture

**Intuition:** identical concept to JS closures — an inner function remembers variables from its enclosing scope even after that enclosing function has returned. If you're solid on JS closures, you already have the right mental model; the differences here are mechanical/CPython-specific, not conceptual.

```python
def make_counter():
    count = 0
    def increment():
        nonlocal count       # explicit — see below, this is the real divergence from JS
        count += 1
        return count
    return increment

counter = make_counter()
print(counter())   # 1
print(counter())   # 2
```

**The `nonlocal` keyword — the genuine syntactic divergence from JS.** In JS, an inner arrow function or closure can freely reassign an outer `let`-scoped variable with no special syntax. In Python, by default, **assignment inside a function creates a new local variable**, even if a name with the same spelling exists in an enclosing scope — the compiler decides, at compile time, whether a name inside a function body is local or not, based purely on whether that name is ever assigned to anywhere in the function body. This means:

```python
def make_counter_broken():
    count = 0
    def increment():
        count += 1     # UnboundLocalError! `count += 1` is `count = count + 1`,
        return count    # and the assignment makes Python treat `count` as LOCAL to
    return increment     # increment(), so `count + 1` reads an unbound local before it's set
```

`nonlocal count` tells the compiler "when I assign to `count` in this function, mean the enclosing function's `count`, not a new local" — you only need it when you plan to **rebind** the outer name; reading an outer variable without assigning to it (e.g. just `print(count)`) never needs `nonlocal`, because without an assignment anywhere in the function body, the compiler has no reason to treat the name as local, and it resolves outward through the enclosing scope automatically (this scope-resolution order — local → enclosing → global → builtin — is Python's **LEGB rule**, worth knowing by name for interviews).

**Why this design exists, historically:** Python's scoping was originally simpler and stricter (no closures over mutable outer variables at all in very old Python); `nonlocal` (added in Python 3.0) was the deliberate, explicit mechanism added specifically because Guido's philosophy resists *implicit* scope-crossing assignment — JS lets you silently rebind an outer variable from an inner closure with zero indication at the call site; Python forces you to declare that intent up front, in the function signature area, which is the Zen of Python's "explicit is better than implicit" applied directly to scoping.

---

## 5.2 The Classic Late-Binding Closure Bug (Loop Variable Capture)

**This is one of the single most common real bugs in Python code, and a near-universal interview trap:**

```python
funcs = []
for i in range(3):
    funcs.append(lambda: i)

print([f() for f in funcs])   # [2, 2, 2] — NOT [0, 1, 2]!
```

**Why:** closures in Python capture *variables by reference to the enclosing scope's cell*, not by value at closure-creation time. All three lambdas close over the *same* `i` — the loop variable, which (per Module 3's "loops don't create new scopes") is one single variable that gets mutated on every iteration and ends at `2` once the loop finishes. By the time any lambda is actually *called*, the loop has finished and `i` is `2` for all of them.

**JS developers get bitten by this specifically because modern JS `let` fixed exactly this problem** — `for (let i = 0; ...)` creates a fresh binding of `i` *per iteration* in JS (this was the entire motivating reason `let` was introduced over `var`, whose old `var`-based closures had this identical bug historically). Python's `for` loop variable has no per-iteration rebinding at all — there's only ever one `i`, matching old pre-`let` JS `var` behavior, not modern JS `let` behavior. If you learned JS after `let` became idiomatic, this bug will genuinely surprise you.

**The idiomatic fix — default-argument capture, exploiting §4.2's "evaluated once, at def time" rule deliberately, as a feature rather than a bug this time:**

```python
funcs = []
for i in range(3):
    funcs.append(lambda i=i: i)   # i=i default forces evaluation NOW, capturing the current value

print([f() for f in funcs])   # [0, 1, 2]
```

This works precisely *because* default argument values are evaluated once at the moment the `lambda`/`def` statement executes (inside the loop iteration, here) — each lambda gets its *own* default value baked in at creation time, sidestepping the shared-cell problem entirely. Recognizing that Module 4's "default args evaluated once, at definition time" rule is the *fix* for this bug, not just a footgun elsewhere, is the kind of connective understanding that separates "knows Python facts" from "understands Python."

---

## 5.3 `lambda` — Anonymous Functions, Deliberately Limited

```python
square = lambda x: x * x
sorted(words, key=lambda w: len(w))
```

**`lambda` is restricted to a single expression** — no statements, no assignments, no multiple lines, by design. This is stricter than JS arrow functions, which can have full statement bodies (`() => { let x = 1; return x; }`). Python's restriction is deliberate, not a limitation nobody noticed: the community consensus (and Guido's stated position) is that if your logic needs more than one expression, it deserves a proper named `def` — a real name in a traceback, docstring capability, and testability that an anonymous inline lambda can never have. This is *why* idiomatic Python style guides (PEP 8) explicitly discourage assigning a `lambda` to a name (`f = lambda x: x + 1` — just write `def f(x): return x + 1` instead; if it's worth naming, it's worth being a real function).

**Where `lambda` is genuinely idiomatic:** short, throwaway, inline use as an argument to `sorted`, `min`, `max`, `filter` (rare — comprehensions usually win, per Module 3), or `key=` parameters generally — anywhere the function is used exactly once, immediately, and naming it would add ceremony without adding clarity.

---

## 5.4 Decorators — Built From First Principles

**Intuition:** a decorator is a function that takes a function and returns a (usually wrapped) function — "wrapping" behavior around a call without modifying the original function's source. Conceptually this is the same pattern as a JS higher-order function wrapper (`const logged = fn => (...args) => { console.log('calling'); return fn(...args); }`) — Python just gives this pattern first-class `@` syntax at the language level, which JS has no equivalent for (JS decorators are a TC39 stage proposal used mainly via TypeScript/Babel transforms for classes; nothing as universal or stdlib-native as Python's function decorators).

**Building one manually, then revealing the sugar:**

```python
def logged(func):
    def wrapper(*args, **kwargs):        # the *args/**kwargs forwarding pattern from Module 4
        print(f"calling {func.__name__}")
        result = func(*args, **kwargs)
        print(f"{func.__name__} returned {result}")
        return result
    return wrapper

def add(a, b):
    return a + b

add = logged(add)      # manually "decorating" — this IS what @ sugar does
print(add(2, 3))
```

```python
@logged                 # pure syntactic sugar for `add = logged(add)`, applied at def-time
def add(a, b):
    return a + b
```

**This is a closure, full stop** — `wrapper` closes over `func` from `logged`'s enclosing scope, exactly per §5.1's mechanics. Decorators aren't a separate language feature bolted on top of closures; they're closures plus one line of sugar, and understanding them as "just a closure returning a closure" is what makes advanced decorator patterns (below) feel inevitable rather than magical.

**The `functools.wraps` fix — a real, expected-in-code-review detail, not decoration:**

```python
def add(a, b):
    """Adds two numbers."""
    return a + b

add = logged(add)
print(add.__name__)   # "wrapper" — WRONG; you've lost the original function's identity/metadata
print(add.__doc__)    # None — lost the docstring too
```

```python
import functools

def logged(func):
    @functools.wraps(func)     # copies __name__, __doc__, __module__, etc. from func onto wrapper
    def wrapper(*args, **kwargs):
        print(f"calling {func.__name__}")
        return func(*args, **kwargs)
    return wrapper
```

Omitting `functools.wraps` is a genuine, frequently-flagged code review issue — it silently corrupts introspection (`help()`, debuggers, `inspect` module, some testing frameworks that key off `__name__`), and it's a routine mid-level interview probe ("what's missing from this decorator?").

**Parameterized decorators — a decorator that itself takes arguments requires one more level of nesting:**

```python
def retry(times):                       # outer: takes the decorator's OWN arguments
    def decorator(func):                # middle: takes the function being decorated
        @functools.wraps(func)
        def wrapper(*args, **kwargs):    # inner: the actual replacement function
            last_exc = None
            for attempt in range(times):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    last_exc = e
            raise last_exc
        return wrapper
    return decorator

@retry(times=3)
def flaky_call():
    ...
```

Reading this correctly under interview pressure ("`@retry(times=3)` calls `retry(times=3)`, which *returns* the actual decorator, which is then applied to `flaky_call`") is a genuine, common senior-level checkpoint — the three-level nesting is exactly parallel to a JS curried higher-order-function factory (`const retry = times => fn => (...args) => {...}`), if that framing helps.

**Multiple stacked decorators apply bottom-up:**

```python
@a
@b
def f(): ...
# equivalent to: f = a(b(f))  — b wraps first, then a wraps the result
```

A genuine, common bug: decorator order matters (e.g., a caching decorator above a logging decorator behaves differently than the reverse — the logging decorator would only fire on cache misses if it's innermost) — always reason about stacked decorators as literal nested function calls, inside-out.

**Class-based decorators (worth knowing exist, less common in practice):** any callable can be a decorator, including a class instance implementing `__call__` (Module 7's data model) — used when the decorator needs to hold persistent state more naturally than a closure variable, e.g. a call-counting decorator storing `self.count`.

---

## 5.5 Common Mistakes / Interview Traps — Consolidated

- Forgetting `nonlocal` when a closure needs to *rebind* (not just read) an enclosing variable, hitting `UnboundLocalError`.
- The loop-variable late-binding bug (§5.2) — extremely common in real code, near-guaranteed interview question.
- Assigning multi-line logic to a `lambda` (isn't even legal for statements) or over-using `lambda` where a named `def` would be clearer and more debuggable.
- Forgetting `functools.wraps`, silently breaking `__name__`/`__doc__`/introspection on every decorated function.
- Misreading parameterized decorator nesting order, especially with multiple stacked decorators.
- Assuming a decorator "modifies the function in place" rather than understanding it *replaces the name* with a new wrapper object (ties directly back to Module 1's rebind-not-mutate model).

---

## 5.6 Exercises

> **Run a snippet locally.** Copy any code block below and feed it straight to Python from your clipboard — on macOS: `pbpaste | python3 -` — or run `python3 -`, paste, and press Ctrl-D. Predict the output first, *then* run it. A few snippets reference a helper you're asked to write, or leave an input undefined — save those to a scratch file (`python3 scratch.py`) and fill in the blank first.

**Predict the output:**
```python
def make_multipliers():
    return [lambda x: x * i for i in range(3)]

fns = make_multipliers()
print([f(10) for f in fns])
```

**Core:** Write a `@timer` decorator (using `functools.wraps`) that prints how long the wrapped function took to execute, and apply it to a function that also takes `*args, **kwargs`.

**Debugging exercise:** A decorated function's `help(my_func)` output shows the wrapper's generic signature instead of the original function's docstring and parameters. Diagnose and fix in one line.

**Interview — junior:** "What does `@decorator` above a function definition actually do, in terms of plain assignment? Rewrite it without the `@` syntax."

**Interview — mid:** "Explain the classic `for i in range(n): funcs.append(lambda: i)` bug. Why does it happen, and what are two different ways to fix it?" (Default-argument capture, and a helper function that returns a fresh closure per call, are both acceptable — expect to name at least the default-arg fix.)

**Interview — senior:** "Write a decorator factory `@cache_for(seconds)` that caches a function's return value for a configurable duration, correctly handling different arguments as different cache keys. Discuss what happens with unhashable arguments and how you'd handle that."

**Advanced / FAANG-style:** "You have three decorators — `@log`, `@cache`, `@require_auth` — stacked on an API handler. Reason through the correct stacking order for production correctness (e.g., should auth checking happen before or after a cache lookup?), and explain what silently breaks if the order is wrong." (This is a real production-architecture question disguised as a decorator-ordering puzzle.)

---

## 5.7 Exercise Solutions

Try each exercise cold first, then check your reasoning here. Keep a short personal note on anything you got wrong — that running log is the highest-value review material as the course goes on.

**Q — Predict the output:**
```python
def make_multipliers():
    return [lambda x: x * i for i in range(3)]

fns = make_multipliers()
print([f(10) for f in fns])
```
**A:** `[20, 20, 20]`. All three lambdas close over the same variable `i` by reference, not by its value at creation time — since loops don't create a new scope per iteration, there is only ever one `i`, and it holds `2` (its final value) by the time any lambda is actually called. Every call computes `10 * 2`.

**Q — Core:** Write a `@timer` decorator (using `functools.wraps`) that prints how long the wrapped function took, and apply it to a function that also takes `*args, **kwargs`.
**A:**
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

@timer
def compute(a, b, factor=1):
    return (a + b) * factor
```

**Q — Debugging:** A decorated function's `help(my_func)` output shows the wrapper's generic signature instead of the original function's docstring and parameters. Diagnose and fix in one line.
**A:** The decorator's inner `wrapper` is missing `@functools.wraps(func)`. Add it directly above `def wrapper(*args, **kwargs):` inside the decorator — this copies `__name__`, `__doc__`, and other metadata from the original function onto the wrapper.

**Q — Interview (junior):** "What does `@decorator` above a function definition actually do, in terms of plain assignment? Rewrite it without the `@` syntax."
**A:**
```python
@logged
def add(a, b): return a + b
```
is exactly equivalent to:
```python
def add(a, b): return a + b
add = logged(add)
```
`@decorator` is pure sugar for calling `decorator` on the function immediately after it's defined and rebinding the original name to the result.

**Q — Interview (mid):** "Explain the classic `for i in range(n): funcs.append(lambda: i)` bug. Why does it happen, and what are two different ways to fix it?"
**A:** All appended lambdas close over the same shared loop variable `i` (loops don't scope per iteration in Python), so every one of them sees whatever `i`'s final value ended up being once the loop finished, regardless of what it was during the iteration that created that particular lambda. Fix 1 — default-argument capture: `lambda i=i: i`, since default values are evaluated once, immediately, at the point the `lambda` statement itself executes, baking in the current value of `i` at that moment. Fix 2 — a factory function returning a fresh closure per call: `def make(i): return lambda: i`, where each invocation of `make` creates its own independent local `i`, with no sharing across calls.

**Q — Interview (senior):** "Write a decorator factory `@cache_for(seconds)` that caches a function's return value for a configurable duration, correctly handling different arguments as different cache keys. Discuss what happens with unhashable arguments."
**A:**
```python
import time, functools

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
`args` (a tuple) is used directly as the cache dict's key, which correctly gives different argument combinations different cache entries — but this requires every element of `args` to be hashable. A `list` argument would raise `TypeError: unhashable type: 'list'` the moment `args in cache` is evaluated. Handling this properly requires either documenting that arguments must be hashable, or converting unhashable arguments to a hashable representation (e.g. `tuple(sorted(some_dict.items()))`) before using them as a key — with the tradeoff that this adds real complexity and a place for subtle bugs if two logically-different unhashable inputs happen to serialize to the same key.

**Q — Advanced:** "You have three decorators — `@log`, `@cache`, `@require_auth` — stacked on an API handler. Reason through the correct stacking order for production correctness, and explain what silently breaks if the order is wrong."
**A:** Stacked decorators apply bottom-up (`@a @b def f()` means `f = a(b(f))`, so `b` wraps first). For this trio, `@require_auth` should sit outermost (applied last, so it runs first on each call) — an unauthorized request should never even reach the cache lookup or generate a log entry for data the caller shouldn't have touched. Below that, ordering `@cache` above `@log` (i.e., `log` wraps closer to the real function, `cache` wraps around `log`) means every call — cache hit or miss — gets logged consistently; ordering it the other way (`@log` outermost, `@cache` closer in) would only log actual cache misses, silently under-reporting real traffic if that's not the intent. The concrete "correct" placement between `cache`/`log` genuinely depends on what you want logged — but auth belonging outermost, gating everything else, is the one piece that's a correctness requirement rather than a preference, and getting it wrong (e.g. `@cache` outermost, above `@require_auth`) risks serving a cached response to an unauthorized caller if the cache key doesn't account for identity.

---

*Next: Module 6 — Generators & Iterators: laziness, `yield`, the full iterator protocol mechanics, generator expressions, and coroutine-style `send`/`throw` basics — the concept that quietly underlies async/await later on.*
