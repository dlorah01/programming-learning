# Python Mastery Course — Module 12: Modules, Packaging, and Environments

---

## 12.1 The Import System — What `import` Actually Does

**Mechanics, precisely, connecting back to Module 0's execution model:** `import foo` does three things, in order: (1) check `sys.modules` — if `foo` was already imported anywhere in this process, return the cached module object immediately, no re-execution; (2) if not cached, locate `foo`'s source via `sys.path` (a list of directories searched in order); (3) execute `foo`'s entire top-level code (Module 0's execution model — every `def`/`class`/statement runs top-to-bottom, once) inside a fresh module namespace, then cache that namespace as the module object in `sys.modules` and bind it to the name `foo` in the importing scope.

```python
import sys
print(sys.path)   # the actual search order: script's own dir, PYTHONPATH entries,
                    # installed site-packages, stdlib — in a specific, checkable order
```

**Why "modules execute exactly once" matters practically:** any module-level code with side effects (opening a connection, printing something, registering a plugin) runs exactly once, the first time it's imported anywhere in the process — subsequent imports elsewhere in the codebase are free, cached lookups. This is architecturally identical to Node's CommonJS `require` caching (and ESM's module singleton behavior) — if you already understand why a Node module's top-level `console.log` only fires once regardless of how many files `require` it, you already understand this.

**Circular imports — a genuine, common real bug class, worth understanding mechanically:**

```python
# a.py
import b
def foo(): return b.helper()

# b.py
import a          # circular!
def helper(): return "hi"
```

If `a.py` is imported first, `sys.modules['a']` is created (partially populated, execution in progress) *before* `a.py`'s `import b` line even finishes running — when `b.py` then does `import a`, it gets that same, still-incomplete module object from `sys.modules` rather than re-running `a.py` from scratch (which would infinite-loop). This often works by accident for simple cases but breaks the moment `b.py` tries to access something from `a` that hasn't been defined yet at that point in `a.py`'s own execution — the fix is almost always restructuring (move the shared logic to a third module both import, or move the import inside the function body so it's deferred until call time rather than import time) rather than fighting the import system directly.

---

## 12.2 Packages, `__init__.py`, and Relative vs. Absolute Imports

A **package** is just a directory containing an `__init__.py` (its presence is what historically marked a directory as importable as a package; since Python 3.3, "namespace packages" without `__init__.py` are also supported, but explicit `__init__.py` remains the clear, conventional default for anything beyond simple namespace-splitting use cases).

```
myapp/
    __init__.py
    models.py
    services/
        __init__.py
        payment.py
```

```python
# absolute import — always resolves from the top of sys.path, unambiguous, generally preferred
from myapp.services.payment import charge_card

# relative import — resolves relative to the CURRENT module's package location
# inside myapp/services/payment.py:
from ..models import User        # ".." = parent package (myapp), then .models
from . import other_service       # "." = current package (myapp.services)
```

**Idiomatic guidance, and a real, common team-style debate:** absolute imports are generally preferred for clarity (you can tell exactly what's being imported without knowing the current file's location) — PEP 8 itself expresses this preference. Relative imports are more common *within* a single tightly-coupled package for internal cross-references, since they survive the package being renamed/relocated as a whole. Mixing styles inconsistently within one codebase is a real, common lint/review flag.

**`__init__.py`'s actual role beyond "marks this as a package":** it's the module executed when the package itself is imported (`import myapp` runs `myapp/__init__.py`), commonly used to curate a clean public API by re-exporting selected internals (`from .models import User` inside `__init__.py`, so callers can do `from myapp import User` instead of the deeper `from myapp.models import User`) — a genuine, deliberate API-design tool, not boilerplate.

---

## 12.3 `if __name__ == "__main__":` — Why It Exists, Precisely

```python
def main():
    print("running as a script")

if __name__ == "__main__":
    main()
```

**Mechanics:** every module has a `__name__` attribute — when a file is run directly (`python script.py`), Python sets `__name__ = "__main__"` for that module's namespace; when the same file is *imported* by something else, `__name__` is instead set to the module's actual dotted name (e.g. `"myapp.utils"`). This guard means a file can be both a reusable, importable module (its functions/classes usable elsewhere without side effects firing) *and* a runnable script (with its own entry-point logic) — without the import triggering the script-only behavior.

**JS/Node comparison:** closest analog is Node's `require.main === module` check (also historically common for the same "is this the entry point or an import" question), or in modern practice, just having a clearly separate `main.js`/CLI entry file — Python's convention is more universal/idiomatic across the ecosystem than Node's equivalent check tends to be in practice. This exact snippet appears in a large fraction of real-world Python files and is a genuine "have you actually written/read Python scripts" fluency signal.

---

## 12.4 Virtual Environments — Why Python Needs Them, Unlike Node

**The core problem `venv` solves, and why it's structurally different from `node_modules`:** Node installs dependencies *per-project*, inside a local `node_modules/` folder, by default and automatically — there's no separate "environment" concept needed because dependency isolation is baked into how `npm install` works out of the box. Python's default `pip install` behavior is **global** (or user-wide) by default — installing a package makes it available to *every* Python script on the system using that interpreter, which means two projects needing different, incompatible versions of the same library would conflict directly without some isolation mechanism.

```bash
python -m venv .venv              # creates an isolated Python installation copy in .venv/
source .venv/bin/activate          # (Windows: .venv\Scripts\activate)
pip install requests               # installs ONLY into this isolated environment, not globally
```

A virtual environment is, mechanically, a lightweight, mostly-symlinked copy of the Python interpreter plus an isolated `site-packages` directory — activating it modifies your shell's `PATH` (and a few environment variables) so `python`/`pip` resolve to the venv's copies instead of the system ones. This is the conceptual (though not mechanical) equivalent of `node_modules` being automatically project-scoped — Python just requires you to explicitly opt into that scoping per-project, rather than getting it by default.

**Why this matters practically, and a real, common "gotcha" for beginners:** installing packages without an activated virtual environment (or accidentally activating the wrong one) is one of the most common sources of "works on my machine"/"ModuleNotFoundError even though I definitely installed it" confusion for people new to Python — always know which environment is active (`which python`, or check the shell prompt prefix venv tools typically add) before installing or running anything project-specific.

---

## 12.5 `pip`, `poetry`, `uv` — The Dependency-Management Landscape, Compared to `npm`

**`pip`** — the baseline installer, roughly analogous to raw `npm install`, but historically with a weaker built-in story for lockfiles/reproducible installs. `requirements.txt` (a flat list of package==version pins) is the traditional, still extremely common way to declare dependencies — conceptually similar to a `package.json`'s dependencies list, but without npm's automatic, integrated lockfile (`package-lock.json`) generation baked in by default; `pip freeze > requirements.txt` is the traditional (somewhat manual, blunt) way to snapshot exact installed versions.

**`poetry`** — a more complete, `npm`-like experience: a `pyproject.toml` (the modern, standardized Python project-metadata file — think `package.json`) declares dependencies with version *ranges*, and `poetry.lock` pins exact resolved versions automatically, generated and checked-in the same way `package-lock.json` is — genuinely closes most of the gap with the Node experience, including a proper dependency resolver, virtual-environment management built in, and publishing tooling.

**`uv`** — a newer (Astral, the Ruff developers), Rust-implemented tool explicitly aiming to replace `pip`+`venv`+`pip-tools`+parts of `poetry`'s job, with a specific, heavily-marketed focus on raw speed (often reported as one to two orders of magnitude faster dependency resolution/installs than `pip`, due to being compiled and using aggressive caching/parallelism) — increasingly the recommended modern default for new projects as of the versions you'll actually be using, while still being interoperable with the standard `pyproject.toml` format `poetry` also uses.

**A genuinely fair, common interview/practical question, and the honest answer:** "which should I use" — for a course/greenfield project today, `uv` is the strongest default recommendation for pure speed and modern ergonomics while remaining standards-compliant; `poetry` remains extremely common in existing production codebases and has a longer track record; raw `pip` + `requirements.txt` remains ubiquitous in simpler scripts, some legacy codebases, and is still worth understanding since you'll encounter it constantly reading other people's code and CI configs.

---

## 12.6 Common Mistakes / Interview Traps — Consolidated

- Installing packages globally instead of into an activated virtual environment, causing cross-project version conflicts.
- Forgetting `if __name__ == "__main__":` and having script-only logic (like a CLI argument parse-and-run) fire unexpectedly when the file is imported elsewhere for its utility functions.
- Circular imports caused by two modules needing each other's top-level names at import time, rather than deferring the import or restructuring shared logic out to a third module.
- Mixing relative and absolute imports inconsistently within one package.
- Confusing `requirements.txt` (a flat, sometimes-unpinned list) with a genuine lockfile — not realizing `pip` alone doesn't give you the same reproducibility guarantee `npm`/`poetry`/`uv` lockfiles do by default.

---

## 12.7 Exercises

**Predict the output:** Given `a.py` importing `b.py` which imports `a.py` back, where `b.py` tries to call a function from `a.py` defined *after* the `import b` line in `a.py` — trace through the exact `ImportError`/`AttributeError` that results, referencing `sys.modules` state precisely.

**Core:** Structure a small package (`mypkg/__init__.py`, `mypkg/core.py`, `mypkg/utils.py`) such that `from mypkg import CoreThing` works, even though `CoreThing` is actually defined in `core.py` — using `__init__.py`'s re-export role correctly.

**Debugging exercise:** A teammate reports `ModuleNotFoundError: No module named 'requests'` despite having run `pip install requests` minutes earlier. Walk through the most likely causes in priority order, tied to virtual environment mechanics specifically.

**Interview — junior:** "What does `if __name__ == '__main__':` actually check, and why is it idiomatic to wrap script entry-point logic in it?"

**Interview — mid:** "Explain why Python needs virtual environments as an explicit, separate concept, when Node's `npm install` gives you project-scoped dependencies automatically."

**Interview — senior:** "Your team is starting a new production service. Justify a dependency-management tool choice (`pip`+`requirements.txt`, `poetry`, or `uv`), considering reproducibility, CI speed, and onboarding friction for engineers coming from a Node/JS background."

**Advanced / FAANG-style:** "Diagnose and fix a real circular import between two modules that genuinely need functions from each other, without simply merging them into one file — discuss at least two structurally different fixes (deferred/local import vs. extracting shared logic to a third module) and the tradeoffs between them."

---

*Next: Module 13 — Concurrency: threading, multiprocessing, asyncio, and the GIL — what each model actually buys you in CPython, and how Python's concurrency story fundamentally differs from Node's single-threaded event loop.*
