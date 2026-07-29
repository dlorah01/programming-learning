# Python Mastery Course — Module 3: Control Flow, Comprehensions, `match`/`case`, Iteration

---

## 3.1 `if`/`elif`/`else` — Mostly Familiar, Two Real Divergences

Syntax aside (colons + indentation instead of braces), two things genuinely differ from JS:

**No switch statement until 3.10** (see `match`/`case` below) — for years the idiomatic Python replacement for a JS `switch` was an `if/elif` chain or a dict-based dispatch table:

```python
# classic pre-3.10 idiom — dict as dispatch table, still very common and often preferred
handlers = {
    "start": handle_start,
    "stop": handle_stop,
}
handler = handlers.get(command, handle_unknown)
handler()
```

This dict-dispatch pattern is genuinely idiomatic Python (not a workaround) — it's O(1), it's data (so it's extensible/testable independent of control flow), and experienced Python engineers reach for it even now that `match` exists, when the cases are simple value→action mappings rather than structural.

**Ternary expression order is reversed from JS**, and trips people up constantly when transliterating:

```python
# JS: condition ? ifTrue : ifFalse
# Python: ifTrue if condition else ifFalse
status = "adult" if age >= 18 else "minor"
```

Get the order backwards under interview pressure and it's an immediate, visible tell that you're translating from another language rather than thinking natively in Python.

**No block scoping** — this is a structural, not cosmetic, divergence and a real footgun:

```python
if True:
    x = 5
print(x)   # 5 — `if` does NOT create a new scope; x leaks into the enclosing scope

for i in range(3):
    pass
print(i)   # 2 — the loop variable survives the loop, fully accessible afterward
```

JS's `let`/`const` are block-scoped (`{}` creates a scope); Python's only scopes are module/function/class/comprehension (comprehensions get their own scope since Python 3 — more below). `if`, `for`, `while`, `with`, and `try` blocks are **not** scopes. This is occasionally convenient (checking a loop variable's final value after the loop) and occasionally a real bug source (accidentally reusing a name across what you assumed were isolated blocks).

---

## 3.2 Loops — `for` Is Iterator-Based, Not Index-Based

**The single most important reframe for a JS/TS developer:** Python's `for` is not a C-style counting loop with an escape hatch (`for...in`/`for...of` in JS terms) — it's the *only* loop construct for iteration, and it always works by repeatedly calling `next()` on an iterator (formalized in §3.5) until `StopIteration`. There is no C-style `for (let i = 0; i < n; i++)` syntax in Python at all.

```python
# ANTI-PATTERN for a JS transplant — works, but immediately flags you as non-native
arr = ["a", "b", "c"]
for i in range(len(arr)):
    print(arr[i])

# IDIOMATIC
for item in arr:
    print(item)

# Need the index too? Don't manually track it — enumerate() (idiomatic, JS has no direct equivalent)
for i, item in enumerate(arr):
    print(i, item)

# Need to walk two sequences together? zip() (JS: closest is manual index loop, or lodash zip)
names = ["Ada", "Alan"]
langs = ["Python", "Haskell"]
for name, lang in zip(names, langs):
    print(f"{name} -> {lang}")
```

**Why this matters beyond style:** `enumerate`/`zip` aren't just nicer syntax — they're lazy iterators (Module 6), so they don't materialize an intermediate list, and they compile to fewer/cheaper bytecode operations than manual indexing (each `arr[i]` is a separate `__getitem__` dispatch; direct iteration avoids that per-element dispatch overhead). Reviewers flag manual-index loops both for readability *and* because they're measurably slower in the common case.

**`while`, `break`, `continue`** — behave exactly as in JS, no surprises.

**The `for...else` / `while...else` clause — genuinely has no JS equivalent, worth knowing exists:**

```python
def find_first_even(nums):
    for n in nums:
        if n % 2 == 0:
            print(f"found {n}")
            break
    else:
        print("no evens found")   # runs iff the loop completed WITHOUT hitting `break`
```

The `else` here means "if the loop was not exited via `break`" — genuinely confusing name choice (a frequent community complaint), but useful for exactly this "search, and handle the not-found case" pattern without a separate found-flag variable. Rare in the wild but a legitimate "have you actually studied Python deeply" interview signal.

---

## 3.3 Comprehensions — Python's Idiomatic Replacement for `map`/`filter`

**Intuition:** JS's `arr.map(x => x * 2).filter(x => x > 5)` chains — Python has a single syntactic construct that does both at once, more efficiently and (once you're fluent) more readably than chained higher-order function calls.

```python
squares = [x * x for x in range(10)]
evens = [x for x in range(20) if x % 2 == 0]
pairs = [(x, y) for x in range(3) for y in range(3) if x != y]   # nested loops, left-to-right order

# dict and set comprehensions — same syntax shape, different brackets
squares_dict = {x: x * x for x in range(5)}
unique_lengths = {len(w) for w in ["a", "bb", "ccc", "dd"]}
```

**Why comprehensions exist / why they're preferred over `map`/`filter` in idiomatic Python** (a real, common interview question): comprehensions compile to a dedicated bytecode pattern (essentially an inlined loop with an implicit accumulator) that's measurably faster than the equivalent `map`/`filter`/`lambda` chain, because the latter requires actual function-call overhead per element (each `lambda` invocation is a real Python function call, with all the frame-creation cost that implies), while a comprehension's loop body is inlined directly. This is a genuine, benchmarkable performance difference, not just style — a rare case where "more Pythonic" and "objectively faster" point the same direction. `map`/`filter` still exist and are still used, especially when you already have a named function (`map(str.upper, words)` reads cleanly) — but a `lambda` passed to `map`/`filter` is the specific pattern idiomatic Python avoids in favor of a comprehension.

**Comprehensions have their own scope (Python 3 only — this was a Python 2→3 fix):**

```python
x = "outer"
result = [x for x in range(5)]
print(x)   # "outer" — the comprehension's `x` does NOT leak into the enclosing scope
```

This is a genuine Python 2 → 3 improvement: in Python 2, list comprehensions leaked their loop variable into the enclosing scope (exactly like the `for` loop leakage from §3.1); generator/dict/set comprehensions never did, and Python 3 unified list comprehensions to match. Know this if you ever read old Python 2 code and the leaked variable looks like a bug — it was a real, documented wart that got fixed.

**Readability limit — a genuine best-practice line, not just taste:** nested comprehensions beyond 2 levels, or comprehensions with more than one `if`, are widely considered an anti-pattern — code review will ask you to unfold it into an explicit loop or `itertools` chain (Module 11) once it stops being scannable in one glance. "Because you *can* nest comprehensions doesn't mean you should" is a genuine senior-engineer instinct, and interviewers sometimes deliberately show you an unreadable nested comprehension and ask you to refactor it.

---

## 3.4 `match`/`case` (3.10+) — Structural Pattern Matching, Not Just a Switch

**Intuition:** superficially resembles a JS `switch`, but it's dramatically more powerful — it does structural destructuring + type checking + guard conditions in one construct, closer to Rust/Scala pattern matching than to C-family `switch`.

```python
def handle(command):
    match command.split():
        case ["go", direction] if direction in ("north", "south", "east", "west"):
            return f"moving {direction}"
        case ["go", _]:
            return "invalid direction"
        case ["look"]:
            return "looking around"
        case ["take", *items]:                 # captures remaining items into a list
            return f"taking {items}"
        case _:                                 # wildcard — like `default` in JS switch
            return "unknown command"
```

```python
# Structural matching against types/shapes — no direct JS equivalent at all
match shape:
    case {"type": "circle", "radius": r}:
        area = 3.14159 * r ** 2
    case {"type": "rectangle", "width": w, "height": h}:
        area = w * h
    case _:
        raise ValueError("unknown shape")
```

**Why it exists / why 3.10 specifically:** this required the PEG parser rewrite mentioned in Module 0 — the old parser's grammar genuinely couldn't express `match`'s syntax (notably, `match` and `case` are "soft keywords" — they're only keywords in this specific grammatical position, and remain valid identifiers everywhere else, e.g. you can still have a variable named `match`). It exists to bring genuine structural destructuring/pattern-matching to Python — the community had wanted a switch-equivalent for two decades, but resisted a plain switch because Python already had dict-dispatch and if/elif chains covering that simpler need; what actually shipped is strictly more powerful than what JS's `switch` offers.

**Real interview trap:** unlike a JS `switch`, there's **no fallthrough** between cases — each matched `case` runs its block and the `match` statement ends; you never need (or can use) `break`. People with switch-statement scar tissue sometimes reflexively add a `break` — harmless but a tell that you're not thinking in `match` natively (and syntactically it's fine there, just pointless, since `break` there would apply to an *enclosing loop*, not the match — a subtle gotcha in itself if a `match` is nested inside a `for`).

**When to actually reach for it vs. `if/elif` vs. dict-dispatch (a real design judgment call, and a fair interview question):** use `match` when you're doing genuine structural destructuring (matching on shape/type of nested data, like the shape-area example) — for simple flat value→action mapping, dict-dispatch is still often cleaner and more testable (it's just data, so you can iterate over it, extend it, unit test individual handlers in isolation).

---

## 3.5 The Iteration Protocol — Conceptual Preview (Full Depth in Module 6)

**Why `for` works uniformly over lists, dicts, strings, files, and custom objects:** every one of them implements the **iterable protocol** — `__iter__` returns an **iterator** (an object implementing `__next__`, which returns the next value or raises `StopIteration` when exhausted). `for x in obj:` is, under the hood, exactly:

```python
it = iter(obj)          # calls obj.__iter__()
while True:
    try:
        x = next(it)      # calls it.__next__()
    except StopIteration:
        break
    # loop body
```

This is precisely why `enumerate`, `zip`, `reversed`, dict `.keys()`/`.values()`/`.items()`, file objects (`for line in f:`), and generators (Module 6) all "just work" with `for` — they all speak the same two-method protocol, an instance of the Zen of Python's implicit theme: uniform protocols over special-cased syntax. JS's iteration story (`Symbol.iterator`, generators, `for...of`) is architecturally the *same idea*, introduced later (ES6) and clearly influenced by exactly this kind of protocol-based design in older languages including Python — if you already understand `Symbol.iterator`, you already understand 90% of `__iter__`/`__next__` conceptually; Module 6 will fill in the remaining CPython-specific mechanics and generators.

---

## 3.6 Common Mistakes / Interview Traps — Consolidated

- Writing `for i in range(len(arr)): arr[i]` instead of direct iteration — works, immediately reads as non-native.
- Getting ternary order backwards (`x if cond else y`, not `cond ? x : y`) under time pressure.
- Assuming `if`/`for` create new scopes — they don't; only functions/classes/comprehensions do.
- Reaching for nested `lambda`+`map`/`filter` chains where a single comprehension (or a named function + comprehension) would be clearer and faster.
- Expecting `match`/`case` fallthrough like a JS/C `switch` — there is none.
- Forgetting comprehensions have their own scope in Python 3 (a fixed Python 2 wart) — mostly relevant for reading old code correctly, not writing new bugs.

---

## 3.7 Exercises

**Predict the output:**
```python
for i in range(3):
    if i == 1:
        continue
    print(i)
else:
    print("done")

x = [i for i in range(3)]
print(i)   # does this raise, or print something? explain precisely
```

**Core:** Rewrite this JS-style transliteration into idiomatic Python:
```python
result = []
for i in range(len(data)):
    if data[i] > 0:
        result.append(data[i] * 2)
```

**Debugging exercise:** A `match` block inside a `for` loop has a `case` ending in `break`, intended to exit the loop early once a match is found — but the loop keeps running. Explain precisely why, using what you now know about `match` semantics.

**Interview — junior:** "Convert this dict-dispatch pattern to a `match` statement, and explain a scenario where you'd prefer to keep the dict version instead."

**Interview — mid:** "Why are list comprehensions generally faster than the equivalent `map()` + `lambda` call in CPython? Be specific about what's different at the bytecode/call-overhead level."

**Interview — senior:** "Design the command-parsing logic for a simple text-adventure game's input handler using `match`/`case`, including at least one guard clause and one wildcard/rest-capture pattern. Then explain when you'd refactor this into a class-based command pattern instead — what user-facing complexity would trigger that refactor?"

**Advanced / FAANG-style:** "Explain, mechanically, what `for x in obj` actually does in terms of dunder method calls, and why this design lets `zip`, `enumerate`, and a `for line in file` loop over a multi-gigabyte file without ever loading it entirely into memory." (Direct bridge into Module 6's generators/laziness.)

---

*Next: Module 4 — Functions Deep Dive: parameters, positional/keyword/keyword-only/positional-only, unpacking, `*args`/`**kwargs`, default-argument evaluation timing, and how Python's calling convention differs from JS's.*
