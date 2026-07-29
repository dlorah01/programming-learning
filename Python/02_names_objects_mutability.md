# Python Mastery Course — Module 1: Names, Objects, References, Identity vs Equality, Mutability

This module resolves the single biggest category of "Python did something weird" surprises for JS/TS developers. Almost all of them trace back to one fact: **Python variables are not boxes that hold values — they're labels attached to objects that live independently on the heap.**

---

## 1.1 Names Are Not Variables — The Model

**Intuition.** In C-family languages (and how most people mentally model JS), you picture a variable as a labeled box: `x = 5` puts `5` "into" the box named `x`. Python's actual model is closer to sticky notes on objects: every object lives on the heap the moment it's created, and `x = 5` just sticks a note reading "x" onto the `5` object that already exists. Assignment is **binding a name to an object**, never copying data into a slot.

This isn't just a metaphor — it's operationally how CPython works. Every name lives in a namespace (a dict-like mapping: `globals()`, a function's local frame, a class body, etc.), and that namespace maps `str` name → object reference (a pointer to a `PyObject` on the heap). `x = 5` does exactly one thing: `namespace['x'] = <pointer to the int object 5>`.

**Why this exists / historical framing:** Python's data model made *everything* — including `5`, `True`, and functions — a first-class heap object with a type, an identity, and (for reference-counted objects) a refcount from day one. This is more uniform than JS's split between primitive values (numbers, strings, booleans — copied by value, no identity) and objects (reference semantics). In Python, there is no primitive/object split at the language-semantics level: `5` is a full `PyObject` of type `int`, same conceptual category as a list or a custom class instance. (CPython does cache and special-case small ints for performance — more below — but that's an implementation detail, not a semantic one.)

**JS/TS comparison, concretely:**

```js
// JS: primitives copy by value
let a = 5;
let b = a;
b = 10;
console.log(a); // 5 — untouched, 'a' held its own copy

// JS: objects are references
let obj1 = { val: 5 };
let obj2 = obj1;
obj2.val = 10;
console.log(obj1.val); // 10 — same underlying object
```

```python
# Python: EVERYTHING is a name bound to an object — no primitive/object split
a = 5
b = a
b = 10
print(a)  # 5 — but NOT because ints "copy like primitives"; it's because
          # `b = 10` REBINDS the name b to a *new* int object 10.
          # 'a' still points at the original int object 5. No mutation occurred.
```

The behavior looks identical to JS's primitive case, but the *reason* is different, and the difference becomes visible the moment you use a mutable type:

```python
list1 = [1, 2, 3]
list2 = list1        # list2 is bound to the SAME object as list1
list2.append(4)
print(list1)          # [1, 2, 3, 4] — same object, mutated in place
```

That's identical to the JS object example. The unifying rule: **assignment in Python never copies. It's always "rebind this name to point at this object."** Whether you observe "value-like" or "reference-like" behavior depends entirely on whether the object itself is mutable and whether you *mutated* it vs. *rebound the name* to a different object. This one rule replaces the entire mental "which things are primitives" question you're used to from JS.

---

## 1.2 Identity vs. Equality — `is` vs. `==`

**Intuition.** `is` asks "are these two names pointing at the exact same object in memory?" `==` asks "do these objects compare as equal, according to whatever `__eq__` logic their type defines?" This maps directly onto JS's `===` on objects (reference identity) vs. a deep-equality check (JS has no built-in structural equality operator for objects — you'd reach for `_.isEqual` or `JSON.stringify` comparisons; Python bakes structural equality into `==` via `__eq__`).

**Technical explanation.** `is` compares `id(a) == id(b)` — `id()` returns (in CPython specifically) the object's memory address, guaranteed unique and constant for the object's lifetime. `==` calls `a.__eq__(b)` (falling back to `b.__eq__(a)` if the first returns `NotImplemented`), which for built-in containers (`list`, `dict`, `tuple`, `str`) is defined to mean *recursive structural equality*, and for custom classes defaults to identity (`object.__eq__` = `is`) unless you override it (Module 7 territory).

```python
a = [1, 2, 3]
b = [1, 2, 3]
c = a

print(a == b)   # True  — same contents, different objects
print(a is b)   # False — genuinely two separate list objects
print(a is c)   # True  — c is literally the same object as a
```

**The infamous small-int/string interning trap** — this is a near-guaranteed interview trap:

```python
x = 256
y = 256
print(x is y)   # True

x = 257
y = 257
print(x is y)   # False (usually — implementation/CPython-version dependent!)
```

**Why:** CPython pre-creates and caches small integers in the range **-5 to 256** at startup (a fixed array of singleton `int` objects) purely as a memory/perf optimization — every reference to, say, `100` anywhere in your program reuses the same cached object. Outside that range, `256 + 1` and `256 + 1` typed separately may or may not produce the same object — it's an implementation detail of CPython's peephole optimizer/compiler, **not a language guarantee**. Similarly, short string literals that look like identifiers get "interned" (deduplicated) by the compiler for performance, so identical short strings often (not always, not guaranteedly) `is` each other, but this is never something correct code should rely on.

**The rule an experienced Python engineer actually follows, without exception:** *use `==` for value comparison, always. Use `is` only for identity checks that are semantically about identity* — specifically `is None`, `is True`/`is False` (never `== None`), sentinel checks, and "is this the exact same cached/singleton object" checks. Never use `is` for numbers or strings expecting value-equality semantics; that you occasionally get lucky due to interning is exactly the trap.

**Why `is None` specifically, not `== None`:** it's faster (identity check, no method dispatch), it's unambiguous (a broken `__eq__` override on some class could theoretically make `x == None` true for a non-None `x`; `is` can never be fooled), and it's the community-enforced idiom — linters (`Ruff`, `flake8`) flag `== None` as a style violation, not just a preference.

---

## 1.3 Mutability — The Actual Taxonomy You Need Memorized

This is the single most consequential list in this module. Get it cold.

| Immutable | Mutable |
|---|---|
| `int`, `float`, `complex`, `bool` | `list` |
| `str` | `dict` |
| `tuple` *(shallow — see caveat below)* | `set` |
| `frozenset` | `bytearray` |
| `bytes` | most user-defined classes (by default) |
| `range` | |
| `None` | |

**Technical explanation of what "immutable" actually means at the object level:** an immutable object's identity-bearing internal state can never change after construction. Any operation that "looks like" mutation on an immutable type — `s = s + "x"`, `n += 1`, `t = t + (4,)` — actually constructs a **brand new object** and rebinds the name to it. The old object is unchanged and, if nothing else references it, becomes eligible for garbage collection.

```python
s = "hello"
s_id = id(s)
s += " world"
print(id(s) == s_id)   # False — a genuinely new str object was created
```

Contrast with mutable types, where in-place operations really do mutate the same object:

```python
lst = [1, 2, 3]
lst_id = id(lst)
lst += [4]              # for lists specifically, += calls __iadd__, which DOES mutate in place
print(id(lst) == lst_id)  # True — same object
```

**Interview trap hidden in that last example:** `+=` is *not* uniformly "mutate in place" or uniformly "rebind" — it depends on whether the type implements `__iadd__`. For `list`, `+=` mutates in place (calls `list.__iadd__`, equivalent to `.extend()`). For `tuple` or `str` (immutable, no `__iadd__`), `+=` falls back to `__add__` plus rebind — new object every time. This is a real gotcha:

```python
def append_bad(lst, item):
    lst += [item]   # mutates the caller's list in place — surprising if you expected value semantics

def append_worse(lst, item):
    lst = lst + [item]   # rebinds the LOCAL name lst to a new list — caller's list is untouched
```

Both look nearly identical; they have opposite observable effects on the caller. This single distinction is a very common mid-level interview probe.

**The tuple caveat (genuinely important, frequently tested):** tuples are immutable *as containers* — you cannot reassign what index 0 points to, and the tuple can never grow or shrink — but if a tuple holds a mutable object, that inner object can still be mutated:

```python
t = (1, 2, [3, 4])
t[2].append(5)
print(t)   # (1, 2, [3, 4, 5]) — the tuple's "shape" (which objects it references) never changed;
           # the LIST it references was mutated, which is a completely different object's business.
t[0] = 99  # TypeError: 'tuple' object does not support item assignment
```

The precise mental model: **"immutable" describes whether the container's own reference-slots can be reassigned, not whether the entire object graph reachable from it is frozen.** This exact distinction is what "hashable" hinges on too (Module 9) — a tuple containing a list is unhashable, because Python can't guarantee its hash stays constant if the contents can mutate underneath it, even though the tuple itself never "changes."

---

## 1.4 Function Arguments — Why "Pass by Reference vs. Value" Is the Wrong Question

**This is where 1.1 through 1.3 collide and produce the classic JS-transplant bug.**

Python is neither "pass by value" nor "pass by reference" in the traditional sense — the accurate term is **"pass by object reference"** (also called "pass by assignment," since parameter binding uses exactly the same rebind-a-name-to-an-object mechanics as `x = ...`). Every argument passed to a function is a new local name inside that function's frame, bound to the *same object* the caller's variable pointed to. What happens next depends entirely on §1.3:

```python
def mutate_in_place(lst):
    lst.append(99)          # mutates the shared object — visible to caller

def rebind_only(lst):
    lst = [1, 2, 3]          # rebinds the LOCAL name — caller unaffected

my_list = [1, 2]
mutate_in_place(my_list)
print(my_list)   # [1, 2, 99]

rebind_only(my_list)
print(my_list)   # [1, 2, 99] — completely unchanged; rebind_only only reassigned its own local name
```

**JS/TS comparison:** this is *exactly* analogous to how JS passes objects and arrays — `function mutate(arr) { arr.push(99) }` mutates the caller's array; `function rebind(arr) { arr = [1,2,3] }` does not affect the caller. If you already have correct intuitions about JS object/array parameter passing, you already have correct intuitions about Python — the trap is purely that Python numbers, strings, and tuples behave like JS *primitives* here (no observable mutation possible, since they can't be mutated at all), while Python lists, dicts, and sets behave like JS *objects/arrays* (shared mutation visible). The rule to internalize: **it's not about "pass by X" — it's just §1.3's mutability table, applied at a function boundary.**

**The classic footgun this produces — mutable default arguments:**

```python
def add_item(item, bucket=[]):   # DANGER
    bucket.append(item)
    return bucket

print(add_item(1))   # [1]
print(add_item(2))   # [1, 2]  <-- surprise! Not [2].
```

**Why:** default argument values are evaluated **exactly once**, at function-definition time (when the `def` statement executes), not on every call. That single default `list` object is created once and reused across every call that doesn't supply its own `bucket` — so mutating it persists across calls, permanently, for the lifetime of the function object. This is a top-tier, near-universal Python interview question at every level from junior to senior, and it's a real production bug pattern (Django/Flask codebases have shipped this exact bug).

**The idiomatic fix, which every experienced Python engineer writes reflexively:**

```python
def add_item(item, bucket=None):
    if bucket is None:      # the `is None` idiom from §1.2, in its most common real use
        bucket = []
    bucket.append(item)
    return bucket
```

---

## 1.5 `copy` vs `deepcopy` — Resolving the Mutability Problem Deliberately

Given everything above, "I want a genuinely independent copy of this mutable object" is a real, frequent need — Python makes you ask for it explicitly (Zen of Python: explicit is better than implicit).

```python
import copy

original = [1, 2, [3, 4]]

shallow = original.copy()          # equivalently: list(original), or original[:]
shallow[0] = 99                    # top-level rebind — independent, fine
print(original[0])                 # 1 — unaffected

shallow[2].append(5)               # mutates the SHARED inner list object
print(original[2])                 # [3, 4, 5] — affected! shallow copy only copies one level deep

deep = copy.deepcopy(original)
deep[2].append(6)
print(original[2])                 # unaffected — deepcopy recursively copies the entire object graph
```

**JS/TS comparison:** this maps directly onto the `{...obj}` / `Array.from`/`slice()` (shallow) vs. `structuredClone()` or `JSON.parse(JSON.stringify(x))` (deep, with the same caveats about non-serializable content) distinction you already know. Same underlying problem, same two-tier solution shape.

**Performance/production note:** `deepcopy` is meaningfully slower (it recursively walks the object graph and uses a memo dict to handle cycles correctly) — reach for it only when you actually need independence at every level, not reflexively.

---

## 1.6 Debugging Example — Reading a Real Bug With This Model

```python
class ShoppingCart:
    def __init__(self, items=[]):     # bug planted
        self.items = items

cart1 = ShoppingCart()
cart1.items.append("apple")

cart2 = ShoppingCart()
print(cart2.items)   # ['apple'] — a brand new cart already has a stranger's apple in it
```

Walk the model: `items=[]` is a default argument, evaluated once at class-definition time (§1.4). Every `ShoppingCart` instance that doesn't pass its own `items` gets `self.items` bound to that *same* list object. `cart1.items.append(...)` mutates it in place (§1.3); `cart2.items` is `is` the same object, so it "sees" the mutation. Fix: `items=None`, then `self.items = items if items is not None else []`.

---

## 1.7 Common Mistakes / Interview Traps — Consolidated

- Using `==` where `is` was intended for `None`/sentinel checks, or vice versa (relying on `is` for value equality of ints/strings).
- Mutable default arguments (§1.4) — expect this in nearly every mid-level+ Python interview.
- Assuming `t[2].append(x)` fails on a tuple because "tuples are immutable" — it doesn't fail; only slot reassignment fails (§1.3).
- Writing a function that mutates its list/dict argument in place and being surprised the caller's data changed (or the reverse — expecting mutation and getting silent rebind-only no-ops).
- Assuming `a = b` "copies" a list/dict the way it copies an int — it never copies; it always binds.
- Believing CPython's small-int caching (`-5` to `256`) is a language guarantee rather than an implementation detail you should never depend on.

---

## 1.8 Exercises

**Predict the output:**
```python
def f(x, y=[]):
    y.append(x)
    return y

print(f(1))
print(f(2, []))
print(f(3))
```

**Core:** Write a function `deep_flatten(nested_list)` that flattens arbitrarily nested lists without using `copy.deepcopy`, reasoning explicitly about which objects you're creating vs. referencing.

**Debugging exercise:** Given the `ShoppingCart` bug in §1.6, a teammate "fixes" it by writing `self.items = items.copy()` instead of changing the default argument. Does this actually fix the bug? Why or why not? (Trace through what `items` is bound to at the time `.copy()` runs.)

**Interview — junior:** "What's the difference between `is` and `==`? Give an example where they'd return different results for equal-looking objects."

**Interview — mid:** "Explain why mutable default arguments are dangerous, and show the idiomatic fix."

**Interview — senior:** "A function takes a `list` argument, conditionally mutates it in a loop, and returns it. A caller reports intermittent bugs where their original list has extra items they didn't expect. Diagnose the likely cause and propose two different fixes with different tradeoffs (in-place contract vs. defensive copy)."

**Advanced / FAANG-style:** "Explain, at the object-model level, why `frozenset({1, 2, [3,4]})` — wait, why can't you even construct that? What does 'hashable' require in terms of the mutability model from this module, and how does that connect to why dict keys must be hashable?" (This bridges directly into Module 9.)

---

*Next: Module 2 — Numbers, Strings, Booleans, None, and Truthiness: the concrete built-in types, their internal representations, and where Python's type coercion rules diverge sharply from JS's.*
