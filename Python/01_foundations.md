---
layout: default
title: "Module 0: Foundations"
---

# Python Mastery Course — Module 0: Foundations

**Course map** (so you have the shape of what's coming): 0. Foundations → 1. Names/Objects/Mutability → 2. Numbers/Strings/Booleans/None → 3. Control Flow & Comprehensions → 4. Functions Deep Dive → 5. Closures & Decorators → 6. Generators/Iterators → 7. OOP & the Data Model → 8. Descriptors/Metaclasses → 9. Collections → 10. Error Handling & Context Managers → 11. Functional Tools (itertools/functools) → 12. Modules/Packaging/Envs → 13. Concurrency (threads/async/multiprocessing/GIL) → 14. Memory Model & GC → 15. Performance & Profiling → 16. Type Hints & Static Analysis → 17. Testing → 18. Production Engineering → 19. Interview Sprint (cumulative).

Each file stands alone but assumes everything before it. Let's start.

---

## 0.1 Python Philosophy & The Zen of Python

**Intuition.** JS/TS grew up permissive: multiple ways to do everything, and the ecosystem (linters, Prettier, TypeScript itself) exists partly to impose order after the fact. Python inverted that bet: the language itself tries to make one way obviously best, and it's opinionated at the syntax level, not just via tooling.

**The text itself** — run `import this` in a REPL and Python prints 19 aphorisms written by Tim Peters in 1999 (it's `PEP 20`). I won't reproduce the poem verbatim (it's copyrighted text), but the load-bearing ideas are:

- *Explicit is better than implicit* — no hidden coercions like JS's `"5" + 3 === "53"` but `"5" - 3 === 2`. Python's `+` between `str` and `int` raises `TypeError`, full stop.
- *There should be one obvious way to do it* — contrast with JS having callbacks, promises, async/await, generators, and observables all live simultaneously as "the" async story.
- *Readability counts* — significant whitespace isn't an aesthetic quirk; it's a deliberate bet that formatting-as-syntax eliminates an entire class of style bikeshedding (Python programmers don't debate brace placement).
- *Errors should never pass silently, unless explicitly silenced* — this is why Python has no `undefined`-style "just returns nothing weird and keeps going"; missing dict keys throw, missing attributes throw, and you opt into swallowing exceptions with an explicit empty `except`.

**Why it exists / history.** Guido van Rossum started Python in December 1989 as a Christmas project, a descendant of the ABC teaching language, intended to fix ABC's extensibility problems while keeping its readability goals. The "one obvious way" philosophy is a direct reaction against Perl's TMTOWTDI ("there's more than one way to do it") motto, which Guido considered a readability disaster at scale. This history matters for how you read Python code today: when you see something verbose where JS would have a one-liner, it's usually not an oversight — it's the community consciously avoiding cleverness.

**How experienced Python devs think differently from JS devs here:** an experienced Python engineer treats "is this Pythonic" as a real design question, not a style nitpick — the same way a Rust engineer treats "does this satisfy the borrow checker." If your code works but reads like transliterated JavaScript (e.g., manual index loops instead of iteration, `lambda` chains instead of comprehensions, deeply nested callbacks instead of flat control flow), a Python reviewer will flag it even though it's correct — that's a genuine cultural gap you're crossing, not a syntax detail.

**Common mistake for JS/TS transplants:** writing Python with JS idioms — `for i in range(len(arr)): print(arr[i])` instead of `for item in arr: print(item)`. It runs, it's just not Python. You'll unlearn this over the next few modules; I'll flag it every time it shows up.

---

## 0.2 Interpreters, CPython, and Alternative Implementations

**Intuition.** "Python" is a language *specification* (behavior, syntax, semantics) — not one binary. This is exactly like "JavaScript" (ECMAScript spec) vs. V8/SpiderMonkey/JavaScriptCore. When you type `python3` in your terminal, you are almost certainly running **CPython**, the reference implementation, written in C, maintained by the CPython core team and the Python Software Foundation.

**Technical explanation — what "interpreter" means here.** CPython is not a pure tree-walking interpreter (it doesn't re-parse and re-execute source text line by line at runtime). It's a **bytecode interpreter**: source → AST (Abstract Syntax Tree) → compiled to bytecode (a compact instruction set for a stack-based virtual machine) → bytecode is executed by a C loop (`ceval.c`'s main interpreter loop) that dispatches on opcode. This is architecturally the same shape as the JVM or (pre-JIT) V8: compile to an intermediate representation once, then interpret that IR repeatedly, which is cheaper than reparsing text.

**The key difference from V8 that will surprise you coming from JS:** V8 has a tiered JIT (Ignition bytecode interpreter → TurboFan JIT compiler that produces actual machine code for hot functions). Standard CPython, as of the versions you'll be using, has **no JIT** — every single execution goes through the bytecode interpreter loop, opcode by opcode, forever. This is the single biggest reason equivalent numeric/loop-heavy code is often 10-50x slower in CPython than in Node. (Python 3.13 introduced an experimental JIT behind a build flag, and 3.14 continues that work, but it's not the default production story yet — treat "CPython has no JIT by default" as the operating assumption for the versions you'll actually deploy.)

**Alternative implementations (know these exist, know when they matter):**

| Implementation | What it is | When you'd reach for it |
|---|---|---|
| **CPython** | Reference impl, C-based, the default | 99% of real-world usage; richest C-extension ecosystem (NumPy, pandas, etc. depend on the C API) |
| **PyPy** | Alternative impl with a genuine tracing JIT, written in RPython | Pure-Python CPU-bound workloads with poor C-extension dependence; can be 4-10x faster for long-running numeric/algorithmic Python loops |
| **Jython** | Python on the JVM | Legacy; JVM interop; largely dormant now |
| **IronPython** | Python on .NET/CLR | .NET interop; niche |
| **MicroPython** | Minimal Python for microcontrollers | Embedded systems |
| **Cython** | Not really an "implementation" — a compiler that turns Python-like source (optionally with C type annotations) into C, then into a genuine compiled extension | Hot loops in an otherwise-CPython codebase (NumPy internals use this pattern) |

**Why this matters practically:** "Python is slow" is really "CPython's interpreter loop is slow for CPU-bound pure-Python code." That's why the actual professional answer to Python performance is almost never "switch implementations" — it's "push the hot loop into C" (NumPy/pandas vectorized ops, or a C extension), because the C code doesn't pay the bytecode-dispatch tax at all. We'll return to this in the Performance module with real profiling.

**JS/TS comparison, concretely:** Node's V8 will happily JIT a hand-written `for` loop summing a million numbers into near-native machine code after a few thousand iterations (function gets "hot," TurboFan kicks in). CPython will interpret that same loop's bytecode a million times, opcode by opcode, with no escape hatch — which is exactly why idiomatic Python performance advice is "let a C-implemented function do the loop for you" (`sum()`, `numpy`, list comprehensions which are marginally faster than manual loops due to opcode count, not magic).

---

## 0.3 Compilation Pipeline & Bytecode

**Intuition.** Think of this as Python's answer to "what does `tsc` do to your `.ts` file," except it happens automatically, silently, every time you run a script, and the "compiled" artifact is bytecode, not another text format.

**The pipeline, concretely:**

1. **Tokenizing** — source text → token stream.
2. **Parsing** — tokens → an AST (Abstract Syntax Tree). Since Python 3.9, this uses a PEG parser (`pegen`), replacing the older LL(1) parser — this is *why* features like the walrus operator (`:=`, 3.8) and structural pattern matching (`match`/`case`, 3.10) became syntactically feasible: the old parser genuinely couldn't express their grammars cleanly.
3. **Compiling** — AST → bytecode, a sequence of instructions for the CPython VM (a stack machine — most opcodes push/pop values off an operand stack, unlike a register machine).
4. **Caching** — the compiled bytecode for imported modules is cached to disk as `.pyc` files inside a `__pycache__/` directory, keyed by source hash/mtime, so re-imports skip recompilation. (Your entry-point script itself is *not* cached this way — only imported modules are.)
5. **Execution** — the bytecode is fed to the C-implemented evaluation loop, which executes it instruction by instruction, manipulating Python objects on the heap.

**See it yourself** (this is the single most useful habit for demystifying Python — do this whenever behavior confuses you):

```python
import dis

def add(a, b):
    return a + b

dis.dis(add)
```

```
  2           0 RESUME                   0
              2 LOAD_FAST                0 (a)
              4 LOAD_FAST                1 (b)
              6 BINARY_OP                0 (+)
             10 RETURN_VALUE
```

Read this like a stack machine trace: `LOAD_FAST` pushes local variable `a`, then `b`, onto the operand stack; `BINARY_OP` pops two values, applies `+` (which itself dispatches to `a.__add__(b)` — more on this in Module 7's data model coverage), pushes the result; `RETURN_VALUE` pops and returns it. Every Python operation you write ultimately bottoms out in traces like this — internalizing `dis` output is what separates people who *guess* about Python performance from people who *know*.

**JS/TS comparison:** V8 does something conceptually similar (source → AST via its parser → Ignition bytecode), but you basically never look at Ignition bytecode in daily JS work because tooling doesn't expose it casually the way `dis` is a stdlib one-liner in Python. Python's compilation model is deliberately transparent and inspectable — this reflects the philosophy again: nothing hidden.

**Why `.pyc` caching trips up JS developers:** in Node, `require`/`import` re-parses on every process start (unless you're using a snapshot tool) — there's no persistent cross-run compiled-artifact cache by default. Python's `__pycache__` is a genuine cross-run cache: if you `import numpy` in ten different scripts, the compile step only reruns when numpy's source changes. If you ever see stale behavior after editing a file that's *imported* elsewhere, and deleting `__pycache__` fixes it, this is why — it's almost always a stale-hash edge case, not a "haunted computer."

**Common pitfall / interview trap:** people say "Python is an interpreted language" as if that's a clean binary distinction from "compiled languages" like Java. It's not that simple, and a senior interviewer will notice if you parrot the simplification. The accurate framing: Python source is compiled to bytecode (an intermediate representation), and *that* is interpreted — same overall shape as the JVM, just without an additional JIT-to-machine-code step by default in CPython.

---

## 0.4 Execution Model — What Actually Happens When You Run `python script.py`

1. The interpreter starts, initializes core built-in types and the `sys`/`builtins` modules.
2. Your script is compiled to a **code object** (bytecode + metadata: constants, variable names, etc.) via the pipeline above.
3. That code object is executed in the `__main__` module's namespace — a real dict, accessible as `globals()`. This is *directly analogous* to the fact that top-level `var`s in an old-school Node CommonJS module live in that module's scope object — except Python's is a plain, inspectable, mutable dict you can literally read and write at runtime.
4. Every `def`, `class`, and top-level statement executes top-to-bottom, in order, at *import/run time* — not "hoisted" in any sense. This is a major JS divergence:

```python
print(greet())   # NameError: name 'greet' is not defined

def greet():
    return "hi"
```

In JS, `function` declarations hoist; calling `greet()` before its textual definition works fine for `function greet(){}` (though not for `const greet = () => {}`, which behaves more like Python here — TDZ). In Python there is **no hoisting of any kind**. Names are bound at the point their binding statement executes, full stop. This single fact resolves a large fraction of "why doesn't Python see my function/class" confusion for JS transplants — the answer is always "because the def/class statement hasn't run yet at that point in program order."

5. Modules are executed exactly once per process (subsequent `import`s of the same module return the cached module object from `sys.modules` — same amortization idea as Node's module cache).

---

## 0.5 Python Versions — What You Actually Need to Track

**The Python 2 vs 3 split (historical context, briefly):** Python 3.0 shipped in 2008 with deliberate, non-backward-compatible changes (`print` became a function, not a statement; strings became Unicode by default instead of bytes-by-default; integer division changed). Python 2 reached official end-of-life January 1, 2020. You will essentially never target Python 2 today — but you *will* encounter the vocabulary "py2/py3 compat," `six`, and `from __future__ import` in older codebases, so recognize it rather than being confused by it.

**What matters going forward is the 3.x minor version**, because unlike most JS runtime version bumps (which mostly just track the evolving ECMAScript spec plus V8 improvements), Python minor versions ship genuinely load-bearing *language* features you need to know are version-gated:

- **3.6** — f-strings, variable annotation syntax
- **3.7** — `dataclasses`, `from __future__ import annotations` groundwork, dicts formally guaranteed insertion-ordered (was a CPython implementation detail in 3.6)
- **3.8** — walrus operator `:=`, positional-only parameters (`/` in signatures)
- **3.9** — dict merge operators (`|`, `|=`), the new PEG parser, generic collection types usable directly (`list[int]` instead of `typing.List[int]`)
- **3.10** — structural pattern matching (`match`/`case`), much better error messages, `X | Y` union syntax without importing `Optional`/`Union`
- **3.11** — big interpreter speedups (adaptive specializing interpreter, "Faster CPython" project), exception groups (`except*`)
- **3.12** — further perf work, more type-system refinements (`type` statement for type aliases), improved f-strings (PEP 701, arbitrary nesting)
- **3.13** — experimental free-threaded build (no-GIL, opt-in), experimental JIT (opt-in)

**Why this matters practically, unlike JS:** in JS/TS you rarely gate code on "does the runtime support this syntax" beyond broad ES-version/browser-target concerns handled transparently by Babel/tsc. In Python, if you write `match`/`case` or `X | Y` union syntax and your production environment (or a library's minimum-supported-version) is 3.9, it's a hard `SyntaxError`, not a polyfillable gap — there is no Babel-equivalent transpilation culture in mainstream Python. Checking a project's `pyproject.toml`/`setup.cfg` `python_requires` before using a "nice new feature" is a real, everyday professional habit, not paranoia.

**What to install/use today:** for a course starting now, target **Python 3.12 or 3.13** — recent enough to get all the syntax above, old enough that every major library (NumPy, pandas, Django, FastAPI) fully supports it.

---

## 0.6 Real-World Example Tying This Together

Say you're debugging "why is this endpoint slow" in a Flask/FastAPI service. An experienced Python engineer's instinct sequence, informed directly by everything above:

1. Is the hot path pure-Python (interpreter-bound) or C-extension-bound (NumPy/pandas/DB driver)? If pure Python and loop-heavy, that's the "no JIT" tax — the fix is vectorize/push into C, not "optimize the Python."
2. Check `sys.version_info` / the `python_requires` in `pyproject.toml` before reaching for a nice 3.11+ feature in a library that must run on 3.9.
3. If behavior seems to "not see" a recently added function/class, check textual program order — likely a hoisting-instinct bug carried over from JS.
4. If import behavior seems stale after an edit, suspect `__pycache__` staleness before suspecting anything exotic.

---

## 0.7 Exercises

> **Run a snippet locally.** Copy any code block below and feed it straight to Python from your clipboard — on macOS: `pbpaste | python3 -` — or run `python3 -`, paste, and press Ctrl-D. Predict the output first, *then* run it. A few snippets reference a helper you're asked to write, or leave an input undefined — save those to a scratch file (`python3 scratch.py`) and fill in the blank first.

**Warm-up (predict the output):**
```python
print(hello())

def hello():
    return "hi"
```
*What happens, and why — name the specific mechanism.*

**Core:**
Run `dis.dis` on a function containing an `if/else` and identify the jump opcodes (`POP_JUMP_IF_FALSE` or your version's equivalent). Explain in your own words what the operand stack looks like at each step.

**Debugging exercise:**
A teammate says: "I edited `utils.py`, but my script still runs the old version even after I saved." Given only what you know about `.pyc` caching and module-execution-once semantics, list three concrete hypotheses to check, in priority order.

**Interview-style — junior:** "Is Python compiled or interpreted?" Give the precise answer (not the pop-sci one), in under 60 seconds, as if speaking out loud.

**Interview-style — mid:** "Why is CPython slow for tight numeric loops, and what's the actual production fix — not 'switch to PyPy'?"

**Interview-style — senior:** "Walk me through what happens, mechanically, from `python app.py` to your first line of business logic executing — mention code objects, `__main__`, and `sys.modules`."

**Advanced / FAANG-style:** "A junior engineer proposes rewriting a CPU-bound Python microservice in PyPy for a 5x speedup with zero code changes. What are the real risks to that plan?" (Hint: think about what depends on the CPython C-API.)

---

## 0.8 Exercise Solutions

Try each exercise cold first, then check your reasoning here. Keep a short personal note on anything you got wrong — that running log is the highest-value review material as the course goes on.

**Q — Predict the output:**
```python
print(hello())
def hello(): return "hi"
```
**A:** `NameError: name 'hello' is not defined`. Python has no hoisting — `def` is an executable statement that binds the name at the point it runs, top to bottom. At the `print(hello())` line, `hello` doesn't exist in the namespace yet, since its `def` statement hasn't executed.

**Q — Core:** Run `dis.dis` on a function containing an `if`/`else` and identify the jump opcodes. Explain what the operand stack looks like at each step.
**A:** An `if`/`else` compiles to a comparison opcode followed by a conditional jump (`POP_JUMP_IF_FALSE`, or the version-current equivalent). The comparison result sits briefly on the operand stack; the jump instruction pops it and, if false, redirects execution past the `if` block's bytecode straight to the `else` block's bytecode (or past both if there's no `else`). Whichever branch runs, the stack ends up in the same state — this consistency is required, since the two branches converge back to shared code afterward.

**Q — Debugging:** A teammate says: "I edited `utils.py`, but my script still runs the old version even after I saved." List three hypotheses to check, in priority order.
**A:** (1) A stale `__pycache__/*.pyc` wasn't invalidated — delete `__pycache__` and rerun. (2) They're editing a different copy of `utils.py` than the one actually being imported (wrong virtual environment, a duplicate file earlier in `sys.path`, or an installed package shadowing the local file). (3) The running process was never actually restarted — a long-lived process (REPL, a server without autoreload) holds the already-imported module object in `sys.modules` and won't re-read the file without a restart, since modules only execute once per process.

**Q — Interview (junior):** "Is Python compiled or interpreted?"
**A:** Neither in the pop-science sense. Python source is compiled to bytecode — an intermediate representation, produced once — and that bytecode is what's interpreted, opcode by opcode, by CPython's C evaluation loop. Architecturally the same shape as the JVM: compile to an IR, then interpret the IR, with no default JIT step to native machine code.

**Q — Interview (mid):** "Why is CPython slow for tight numeric loops, and what's the actual production fix — not 'switch to PyPy'?"
**A:** CPython has no JIT by default — every loop iteration re-dispatches through the bytecode interpreter loop with real per-operation overhead that never amortizes away, no matter how many times the loop runs. The real production fix is pushing the hot loop into compiled C — vectorizing with NumPy/pandas, or writing a small C extension — not switching interpreters, which brings its own compatibility risk (see the advanced question below).

**Q — Interview (senior):** "Walk me through what happens, mechanically, from `python app.py` to your first line of business logic executing — mention code objects, `__main__`, and `sys.modules`."
**A:** The interpreter starts and initializes core built-in types and `sys`/`builtins`. The script's source is tokenized, parsed to an AST, and compiled into a code object (bytecode plus metadata: constants, variable names). That code object executes inside the `__main__` module's namespace, top to bottom, in program order — every `def`/`class`/statement runs exactly once, at this point, with no hoisting. Any `import` encountered checks `sys.modules` first; if the module isn't cached, it's located via `sys.path`, its own top-level code executes once inside a fresh namespace, and the result is cached — so a module imported from multiple places only executes once per process. Business logic starts executing once its containing statement is reached in this pass.

**Q — Advanced:** "A junior engineer proposes rewriting a CPU-bound Python microservice in PyPy for a 5x speedup with zero code changes. What are the real risks?"
**A:** PyPy's JIT only helps genuinely pure-Python code. If the service depends on C extensions (most real services do — DB drivers, crypto libraries, some web framework internals), those extensions may be PyPy-incompatible outright, or may run without benefiting from PyPy's JIT at all, capping or even negating the expected speedup. There's also real behavioral-compatibility risk (subtle edge-case differences between CPython and PyPy) and a warm-up cost — PyPy's JIT needs sustained "hot" execution to kick in, so short-lived request/response cycles may see little or no benefit.

---

*Next: Module 1 — Names, Objects, References, Identity vs Equality, and Mutability. This is the module that resolves nearly every "but it worked differently in JS" surprise about assignment.*
