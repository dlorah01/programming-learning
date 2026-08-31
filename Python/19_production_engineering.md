---
layout: default
title: "Module 18: Production Engineering"
---

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

> **Run a snippet locally.** Copy any code block below and feed it straight to Python from your clipboard — on macOS: `pbpaste | python3 -` — or run `python3 -`, paste, and press Ctrl-D. Predict the output first, *then* run it. A few snippets reference a helper you're asked to write, or leave an input undefined — save those to a scratch file (`python3 scratch.py`) and fill in the blank first.

**Predict/reason:** Given `logger.debug(f"user data: {expensive_serialize(user)}")` running in a production service configured at `INFO` level, what actually happens at runtime, and what would change if written as `logger.debug("user data: %s", expensive_serialize(user))` instead? (Trick: does the lazy form avoid calling `expensive_serialize` too, or only avoid the *formatting*? Reason carefully.)

**Core:** Set up a `pyproject.toml` for a small package with `pytest`, `ruff`, and `mypy` as dev dependencies, and write the three corresponding CI-equivalent commands you'd run locally before pushing.

**Debugging exercise:** A team's tests pass locally for every developer but fail immediately after a package is actually released and installed by a user, with an `ImportError` for a module that clearly exists in the repo. Diagnose the likely project-structure cause and the fix.

**Interview — junior:** "Why is `print()` considered bad practice in production Python code? What would you use instead, and what does it give you that `print()` doesn't?"

**Interview — mid:** "Explain the difference between a linter and a formatter, using `Ruff` and `Black` (or `Ruff`'s formatter mode) as examples. Why are both typically CI-enforced rather than optional?"

**Interview — senior:** "Design a CI pipeline for a Python microservice from scratch — the specific stages, their order, and why that order — and explain the `src/`-layout packaging pitfall you'd guard against."

**Advanced / FAANG-style:** "A production incident traces back to a silently swallowed exception in a background worker (connecting back to Module 10's `except Exception: pass` anti-pattern) that was never logged. Redesign the worker's error handling and logging strategy so a similar incident would be immediately visible in production monitoring, referencing specific logging levels and what `exc_info=True` buys you."

---

## 18.7 Exercise Solutions

Try each exercise cold first, then check your reasoning here. Keep a short personal note on anything you got wrong — that running log is the highest-value review material as the course goes on.

**Q — Predict/reason:** Given `logger.debug(f"user data: {expensive_serialize(user)}")` running at `INFO` level in production, what actually happens? What would change with `logger.debug("user data: %s", expensive_serialize(user))` instead? Does the lazy form avoid calling `expensive_serialize` too, or only avoid the formatting?
**A:** Neither form avoids the call to `expensive_serialize(user)`. In both cases, `expensive_serialize(user)` is a plain Python function call being evaluated as part of building the arguments passed *into* `logger.debug(...)` — Python must evaluate all arguments before the function call actually happens, regardless of what that function does with them afterward or whether it decides to do anything with the result. The genuine laziness `logging`'s `%s`-style form provides is narrower than it's often assumed to be: it avoids performing the *string interpolation/formatting step* (substituting the `%s` placeholder with the argument's string representation) when the log level is disabled — it does not, and structurally cannot, avoid evaluating whatever expression was used to produce the argument in the first place, since that evaluation happens before `logger.debug` is even called. To genuinely avoid the expensive `expensive_serialize(user)` call itself when DEBUG is disabled, you need an explicit guard: `if logger.isEnabledFor(logging.DEBUG): logger.debug("user data: %s", expensive_serialize(user))`.

**Q — Core:** Set up a `pyproject.toml` for a small package with `pytest`, `ruff`, and `mypy` as dev dependencies, and the corresponding local CI-equivalent commands.
**A:**
```toml
[project]
name = "mypackage"
version = "0.1.0"

[project.optional-dependencies]
dev = ["pytest>=8.0", "ruff>=0.5", "mypy>=1.10"]
```
Local commands: `ruff check .` (lint), `ruff format --check .` (format check), `mypy .` (type check), `pytest` (tests).

**Q — Debugging:** Tests pass locally for every developer, but fail with an `ImportError` immediately after the package is released and installed by a user, for a module that clearly exists in the repo. Diagnose and fix.
**A:** Likely cause: the project isn't using a `src/` layout, so local test runs have been passing via an accidental current-working-directory-based import path that happened to work throughout development but doesn't reflect what actually ends up in the built, installed distribution — a file or `__init__.py` entry may genuinely be missing from the package due to a `pyproject.toml` package-discovery misconfiguration, and this was never caught locally because local test runs never actually went through a real install step. Fix: adopt a `src/` layout, so tests can only succeed by importing the genuinely *installed* package (`pip install -e .`), which surfaces this exact class of packaging bug during normal local development rather than only after a release ships.

**Q — Interview (junior):** "Why is `print()` considered bad practice in production Python code? What would you use instead, and what does it give you that `print()` doesn't?"
**A:** `print()` always fires unconditionally to stdout, with no concept of severity and no built-in way to redirect, filter, or structure its output without hand-rolling everything `logging` already provides. `logging` gives you severity levels (independently filterable per deployment environment without any code change), multiple simultaneous output destinations (stdout, a file, a remote log aggregator, all at once, independently configured), and consistent, structured formatting (timestamps, module name, severity) automatically attached to every entry.

**Q — Interview (mid):** "Explain the difference between a linter and a formatter, using `Ruff` and `Black`/`Ruff`'s formatter mode as examples. Why are both typically CI-enforced?"
**A:** A linter (`Ruff`, in its linting role) analyzes code for likely bugs, anti-patterns, and style violations, flagging issues without necessarily rewriting the code itself. A formatter (`Black`, or `Ruff`'s own formatter mode) deterministically rewrites code into one canonical style, with essentially no configuration knobs, specifically to end style debates entirely rather than adjudicate them. Both are typically CI-enforced because manual human review of style/lint issues wastes reviewer time and attention on things a tool can catch automatically, consistently, and instantly — the identical reasoning that makes ESLint/Prettier CI-enforced gates on any serious JS/TS codebase.

**Q — Interview (senior):** Design a CI pipeline for a Python microservice from scratch — stages, order, and reasoning — and explain the `src/`-layout pitfall to guard against.
**A:** A reasonable stage order: (1) install dependencies from a lockfile, for reproducibility; (2) lint (`ruff check .`); (3) format check (`ruff format --check .`); (4) type check (`mypy .`/`pyright`); (5) test (`pytest --cov`); (6) build (package/Docker image); (7) deploy, gated on all of the above passing on merge to main. Ordering reasoning: cheapest, fastest checks run first, so a trivial style violation fails in seconds rather than only after a multi-minute test suite finishes — fast feedback for the common case. `src/`-layout pitfall: ensure the test stage genuinely installs the package (rather than relying on an accidental current-working-directory-based import) so packaging misconfigurations are caught here, in CI, before a release — never after a user hits an `ImportError` post-install.

**Q — Advanced:** A production incident traces back to a silently swallowed exception in a background worker, never logged. Redesign the worker's error handling and logging strategy.
**A:** Replace any bare `except Exception: pass` with a handler that at minimum calls `logger.error("...", exc_info=True)` (or `logger.exception(...)`, which does the same thing implicitly), capturing the full traceback rather than discarding it silently. Distinguish genuinely recoverable failures (log at an appropriate level and continue, or route the failed item to a dead-letter queue for later inspection) from unexpected ones (log with full detail and consider re-raising or triggering an alert, since an unexpected exception may indicate a real, ongoing problem worth immediate attention). Critically, ensure the logging configuration actually ships this output to wherever the team's production monitoring/alerting reads from — a log line written only to a local file nobody watches provides no real incident visibility, regardless of how well-formed the log message is. `exc_info=True` specifically attaches the currently-handled exception's traceback to the log record, so the resulting log entry shows exactly where and why the failure occurred, not merely that something, unspecified, failed.

---

*Next: Module 19 — Interview Sprint: a cumulative, mixed-topic set of junior→senior→FAANG-style questions, debugging exercises, and code-review exercises pulling together everything from Modules 0–18, plus general interview strategy (communicating trade-offs out loud, structuring an answer, common red flags interviewers watch for).*
