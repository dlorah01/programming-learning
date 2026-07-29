# Python Mastery Course — Module 18: Production Engineering

---

## 18.1 Logging — Why Not Just `print()`

```python
import logging

logger = logging.getLogger(__name__)     # __name__ gives each module its own logger, hierarchical
logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(name)s: %(message)s")

logger.debug("verbose diagnostic detail")     # suppressed at INFO level — configurable per-deployment
logger.info("user %s logged in", user_id)      # note: lazy %-formatting, not an f-string — see below
logger.warning("cache miss rate high")
logger.error("payment failed", exc_info=True)   # exc_info=True attaches the current exception traceback
logger.critical("database unreachable")
```

**Why `logging` over `print()`, concretely, a genuinely common junior-to-mid interview question:** `print()` always fires, always goes to stdout, has no severity concept, and can't be redirected/filtered/structured without hand-rolling everything logging already gives you for free — **levels** (DEBUG/INFO/WARNING/ERROR/CRITICAL, individually filterable per deployment environment without code changes), **handlers** (send to stdout, a file, a remote log aggregator, all simultaneously, independently configured), and **structured, consistent formatting** (timestamps, module name, severity, all automatic). Production code with scattered `print()` statements is a genuine, common code-review flag — `print()` belongs in quick scripts/debugging sessions, not shipped service code.

**Why `logger.info("user %s logged in", user_id)` and not an f-string, precisely — a real, non-obvious performance/safety point:** the `%s`-style lazy formatting only actually formats the string **if the log record is going to be emitted** at the configured level — `logger.debug(f"expensive: {compute_expensive_debug_info()}")` always evaluates `compute_expensive_debug_info()` regardless of whether DEBUG logging is even enabled, wasting real work in production where DEBUG is typically off; the lazy `%s`-argument form defers formatting until logging has already confirmed the record will actually be used. A genuine, checkable, common code-review-level distinction.

**JS comparison:** conceptually parallel to `winston`/`pino` (structured, leveled logging) vs. scattered `console.log` — the same underlying "structured, filterable, production-appropriate logging vs. ad-hoc print debugging" cultural distinction exists in both ecosystems; if you already avoid `console.log` in production Node code, you already have the right instinct here.

---

## 18.2 Project Structure — The Idiomatic Modern Layout

```
myproject/
    pyproject.toml           # modern project metadata + dependencies (Module 12.5) — the package.json analog
    README.md
    src/
        myproject/
            __init__.py
            core.py
            services/
                __init__.py
                payment.py
    tests/
        test_core.py
        test_payment.py
    .gitignore
```

**The `src/` layout, specifically, and why it's the modern recommended default over putting the package directly at the project root — a real, subtle, genuinely important detail:** with a `src/` layout, your tests can only import `myproject` via the actually-*installed* package (`pip install -e .`), not accidentally via a stray current-working-directory import — this catches real packaging bugs (missing files in the built distribution, incorrect `pyproject.toml` configuration) *during local testing*, rather than only discovering them after a broken release ships. Without `src/` (package directly at repo root), tests can pass locally while silently relying on the CWD-based import path, masking packaging bugs that only surface once installed elsewhere — a genuine, real-world lesson learned that's now widely adopted as best practice, worth knowing the *reasoning* behind, not just the convention.

**`pyproject.toml` — the modern, standardized single source of truth (Module 12.5's `package.json` analog), replacing the older, more fragmented `setup.py`/`setup.cfg`/`requirements.txt` combination:**

```toml
[project]
name = "myproject"
version = "0.1.0"
dependencies = ["requests>=2.31", "pydantic>=2.0"]

[project.optional-dependencies]
dev = ["pytest>=8.0", "ruff>=0.5", "mypy>=1.10"]
```

---

## 18.3 `Ruff` and `Black` — Modern Linting and Formatting

**`Black`** — an **opinionated, non-configurable** (deliberately, almost entirely) code formatter — direct spiritual analog to `Prettier` in the JS ecosystem: the entire point is ending style debates in code review by having a tool that reformats code into one canonical style, with essentially no knobs to argue about. Running `black .` reformats your whole codebase consistently; most teams run it as a pre-commit hook or CI check (`black --check .` failing the build on unformatted code) rather than a manual, optional step.

**`Ruff`** — a newer, Rust-implemented linter (from Astral, the same team behind `uv`, Module 12.5) that has rapidly become the dominant choice, largely replacing the older `flake8`/`isort`/`pylint`/`pyupgrade` combination with one dramatically faster, single tool implementing (and actively expanding) most of their combined rule sets — genuinely one to two orders of magnitude faster than the tools it replaces, which matters a great deal for large codebases and CI turnaround time. `Ruff` can also *format* code (a Black-compatible formatter mode), increasingly letting teams replace both `Black` and the older linting stack with a single tool.

**JS/TS comparison, a clean mapping:** `Black`/`Ruff`'s formatting role ≈ `Prettier`; `Ruff`'s linting role ≈ `ESLint`; the underlying professional practice — automated, CI-enforced, non-negotiable formatting/linting rather than manual style review — is identical in spirit across both ecosystems, and genuinely expected on any serious production Python codebase, exactly as it would be on a serious TS codebase.

---

## 18.4 CI/CD Basics — What a Real Pipeline Checks

A representative, realistic CI pipeline for a Python service, run on every pull request:

1. **Install dependencies** — typically from a lockfile (`uv.lock`/`poetry.lock`, Module 12.5) for reproducibility.
2. **Lint** — `ruff check .`, failing the build on violations.
3. **Format check** — `ruff format --check .` or `black --check .`, failing on unformatted code (never auto-fixing in CI silently — formatting issues should be visible/fixed locally, not silently patched by the pipeline).
4. **Type check** — `mypy .` or `pyright`, per Module 16.
5. **Test** — `pytest --cov`, typically with a minimum coverage threshold gate.
6. **Build** — package the application (Docker image, wheel/sdist for a library).
7. **Deploy** — on merge to main, typically gated behind the above all passing.

**Why each stage exists as a *separate, explicit gate* rather than one big script, worth articulating clearly in an interview:** fast-failing on cheap checks first (linting/formatting are seconds; full test suites can be minutes) gives faster feedback for the common case of a simple style violation, and keeping stages independently reportable (a PR showing "lint: pass, types: fail, tests: pass" rather than one opaque pass/fail) makes failures immediately diagnosable rather than requiring someone to dig through a monolithic log.

---

## 18.5 Common Mistakes / Interview Traps — Consolidated

- Shipping production code with scattered `print()` debugging statements instead of proper leveled logging.
- Using f-strings inside log calls (`logger.debug(f"...")`) instead of lazy `%s` formatting, paying for expensive string construction even when that log level is disabled.
- Not using a `src/` layout, allowing tests to pass locally via accidental CWD-based imports while masking real packaging bugs.
- Treating `Black`/`Ruff` formatting as optional/manual rather than a CI-enforced, non-negotiable gate — reintroducing exactly the style-debate overhead these tools exist to eliminate.
- Running the full test suite before cheap linting/type checks in CI, wasting time on slow feedback for what's often a trivial style issue.

---

## 18.6 Exercises

**Predict/reason:** Given `logger.debug(f"user data: {expensive_serialize(user)}")` running in a production service configured at `INFO` level, what actually happens at runtime, and what would change if written as `logger.debug("user data: %s", expensive_serialize(user))` instead? (Trick: does the lazy form avoid calling `expensive_serialize` too, or only avoid the *formatting*? Reason carefully.)

**Core:** Set up a `pyproject.toml` for a small package with `pytest`, `ruff`, and `mypy` as dev dependencies, and write the three corresponding CI-equivalent commands you'd run locally before pushing.

**Debugging exercise:** A team's tests pass locally for every developer but fail immediately after a package is actually released and installed by a user, with an `ImportError` for a module that clearly exists in the repo. Diagnose the likely project-structure cause and the fix.

**Interview — junior:** "Why is `print()` considered bad practice in production Python code? What would you use instead, and what does it give you that `print()` doesn't?"

**Interview — mid:** "Explain the difference between a linter and a formatter, using `Ruff` and `Black` (or `Ruff`'s formatter mode) as examples. Why are both typically CI-enforced rather than optional?"

**Interview — senior:** "Design a CI pipeline for a Python microservice from scratch — the specific stages, their order, and why that order — and explain the `src/`-layout packaging pitfall you'd guard against."

**Advanced / FAANG-style:** "A production incident traces back to a silently swallowed exception in a background worker (connecting back to Module 10's `except Exception: pass` anti-pattern) that was never logged. Redesign the worker's error handling and logging strategy so a similar incident would be immediately visible in production monitoring, referencing specific logging levels and what `exc_info=True` buys you."

---

*Next: Module 19 — Interview Sprint: a cumulative, mixed-topic set of junior→senior→FAANG-style questions, debugging exercises, and code-review exercises pulling together everything from Modules 0–18, plus general interview strategy (communicating trade-offs out loud, structuring an answer, common red flags interviewers watch for).*
