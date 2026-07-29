# Python Mastery Course — Module 17: Testing

---

## 17.1 `unittest` — The Stdlib, xUnit-Style Baseline

```python
import unittest

class TestMath(unittest.TestCase):
    def setUp(self):                       # runs before EVERY test method — like beforeEach in Jest
        self.calculator = Calculator()

    def test_addition(self):
        self.assertEqual(self.calculator.add(2, 3), 5)

    def test_division_by_zero(self):
        with self.assertRaises(ZeroDivisionError):
            self.calculator.divide(1, 0)

if __name__ == "__main__":
    unittest.main()
```

**JS comparison:** structurally close to Jest/Mocha's class-based or `describe`/`beforeEach` style — class-based, `setUp`/`tearDown` lifecycle hooks, `assertX` methods instead of `expect().toBe()` chains. Genuinely fine, stdlib-included (zero extra dependency), and still common in older/enterprise codebases, but the ecosystem has broadly moved toward `pytest` as the idiomatic default for new code — know `unittest` well enough to read/maintain existing suites, but expect to write new tests in `pytest`.

---

## 17.2 `pytest` — The De Facto Modern Standard

```python
def test_addition():
    calc = Calculator()
    assert calc.add(2, 3) == 5     # plain `assert` — no assertEqual/assertTrue ceremony needed

def test_division_by_zero():
    calc = Calculator()
    with pytest.raises(ZeroDivisionError):
        calc.divide(1, 0)
```

**Why `pytest` won out, concretely — real, substantive advantages over `unittest`, not just fashion:**

1. **Plain `assert` with rich failure introspection.** `pytest` rewrites the `assert` statement at import time (via bytecode manipulation) to capture and display the actual values of subexpressions on failure — `assert result == [1,2,3]` failing shows you the *actual* list contents in a diff-style output, without needing `self.assertEqual` or any special assertion method at all. This is a genuine, significant ergonomics win — no memorizing/looking-up a dozen `assertX` method names.
2. **No mandatory class-based structure** — plain functions work fine, reducing boilerplate for simple tests (classes remain available and useful for grouping/sharing setup, just not required).
3. **Fixtures (§17.3)** — a genuinely more powerful, more composable dependency-injection-style setup/teardown system than `unittest`'s `setUp`/`tearDown`.
4. **A vast, high-quality plugin ecosystem** (`pytest-cov` for coverage, `pytest-mock`, `pytest-asyncio` for testing async code, `pytest-xdist` for parallel test runs) — genuinely comparable in spirit to the Jest/Vitest plugin ecosystems JS developers already expect.
5. **`unittest`-based test suites run unmodified under `pytest`** — `pytest` can discover and run `unittest.TestCase`-based tests directly, so migration is typically additive/incremental, not a rewrite.

---

## 17.3 Fixtures — `pytest`'s Composable Setup/Teardown System

```python
import pytest

@pytest.fixture
def calculator():
    calc = Calculator()
    yield calc                    # everything before yield = setup; after = teardown (Module 10.4's
    calc.close()                    # @contextmanager pattern, reused here — same yield-as-boundary idea)

def test_addition(calculator):     # pytest matches the PARAMETER NAME to the fixture name automatically
    assert calculator.add(2, 3) == 5

def test_subtraction(calculator):   # each test gets its OWN fresh instance — no shared state between tests
    assert calculator.subtract(5, 3) == 2
```

**The mechanism, precisely, and why it's more powerful than `setUp`:** `pytest` inspects a test function's parameter names and, for each one matching a registered fixture, calls that fixture function and injects its (pre-`yield`) return/yielded value as the argument — genuinely a dependency-injection pattern, letting you compose fixtures (a fixture can itself depend on other fixtures as its own parameters), share fixtures across an entire test file or project via `conftest.py`, and control fixture lifetime precisely via `scope`:

```python
@pytest.fixture(scope="session")     # created ONCE for the entire test run, not per-test
def database_connection():
    conn = connect_to_test_db()
    yield conn
    conn.close()
```

`scope="function"` (the default — fresh per test), `"class"`, `"module"`, `"session"` — a real, meaningful lever for balancing test isolation against setup cost (e.g., an expensive database connection you genuinely want to share across many tests, vs. a cheap object you want strictly isolated per test to avoid cross-test state leakage).

**JS comparison:** conceptually closer to a dependency-injection-flavored version of Jest's `beforeEach`/`beforeAll` — the real difference is `pytest`'s fixtures are explicitly *requested* per-test via parameter names (so a given test's actual dependencies are visible right in its signature) rather than implicitly applied to every test in a block via `beforeEach`, which is a genuinely different, more explicit (Zen of Python, again) design.

---

## 17.4 Mocking — `unittest.mock`, Used Identically From Either Test Framework

```python
from unittest.mock import Mock, patch

def test_sends_email(mocker=None):
    email_service = Mock()
    user_service = UserService(email_service)
    user_service.register("ada@example.com")
    email_service.send.assert_called_once_with("ada@example.com", "Welcome!")

# patch() — replaces a real dependency with a Mock for the duration of the test, restoring after
@patch("myapp.services.requests.get")
def test_fetch_user(mock_get):
    mock_get.return_value.json.return_value = {"id": 1, "name": "Ada"}
    result = fetch_user(1)
    assert result["name"] == "Ada"
    mock_get.assert_called_once_with("https://api.example.com/users/1")
```

**`Mock` objects are permissive by default — a genuine, common gotcha:** any attribute or method access on a `Mock()` automatically succeeds and returns another `Mock` (auto-speccing children on demand), which means a typo in a mocked method name (`mock.sned_email()` instead of `mock.send_email()`) **won't raise an error** — it just silently creates and calls a new, different mock attribute, and your test can pass despite testing the wrong thing entirely. The fix, worth knowing and reaching for on anything beyond the simplest mocks:

```python
from unittest.mock import create_autospec
mock_service = create_autospec(RealEmailService)   # only allows attributes/methods RealEmailService
mock_service.sned_email()                            # AttributeError — genuinely catches the typo now
```

`create_autospec`/`patch(..., autospec=True)` constrain the mock's interface to match the real object's actual signature, converting a class of silent, hard-to-spot mocking bugs into loud, immediate failures — a real, senior-level testing hygiene practice, directly parallel to Jest's `jest.mock()` with explicit type checking via TS, if that's a useful bridge.

**`patch` as a decorator vs. a context manager — both valid, situational:**

```python
def test_a():
    with patch("myapp.module.some_function") as mock_fn:    # scoped tightly to this block
        mock_fn.return_value = 42
        ...
```

---

## 17.5 What a Genuinely Good Test Suite Looks Like — Production Guidance

- **Arrange-Act-Assert structure** per test — same universal principle as any good test suite in any language; `pytest`'s plain-function style makes this especially readable.
- **One logical assertion focus per test** — a test named `test_division_by_zero_raises` should test exactly that, not incidentally also verify five unrelated things; failures should immediately tell you what broke.
- **Fixtures for shared setup, not copy-pasted setup code** — the whole point of §17.3.
- **Mock external boundaries (network, database, filesystem, time), not your own internal logic** — over-mocking internal collaborators makes tests brittle (they break on refactors that don't change actual behavior) and is a genuine, common anti-pattern; mock at the true I/O boundary, test real internal logic directly wherever feasible.
- **Parametrized tests over copy-pasted near-duplicate test functions:**

```python
@pytest.mark.parametrize("a, b, expected", [
    (2, 3, 5),
    (-1, 1, 0),
    (0, 0, 0),
])
def test_addition(a, b, expected):
    assert Calculator().add(a, b) == expected
```
Genuinely eliminates copy-paste test duplication (direct analog to Jest's `test.each`) — a real, checkable code-review-level improvement over three near-identical `test_addition_1`/`_2`/`_3` functions.

---

## 17.6 Common Mistakes / Interview Traps — Consolidated

- Over-mocking internal collaborators, producing tests that pass despite real behavior being broken, or fail on harmless refactors.
- Using a bare `Mock()` where `create_autospec`/`autospec=True` would have caught a typo'd method name immediately.
- Copy-pasting near-identical test functions instead of `@pytest.mark.parametrize`.
- Fixture `scope` misuse — sharing a `session`-scoped fixture that actually needs per-test isolation, causing subtle cross-test state leakage/order-dependence.
- Testing implementation details (private methods, internal call counts on your own code) instead of observable behavior — brittle, high-maintenance tests that resist legitimate refactoring.

---

## 17.7 Exercises

**Predict:** Given a `Mock()` with no `spec`, calling `mock.nonexistent_method()` — does this raise an error? What about `create_autospec(RealClass)` with the same call, assuming `RealClass` has no such method?

**Core:** Write a `pytest` fixture providing a fresh, empty `ShoppingCart` per test, and three parametrized tests covering adding items, removing items, and computing totals.

**Debugging exercise:** A test suite mocks a `PaymentGateway` dependency, and after a refactor genuinely breaks the real payment integration, every relevant test still passes. Diagnose why (connecting to §17.4/§17.5) and describe what change to the test suite would have caught the real bug.

**Interview — junior:** "What's the difference between `unittest` and `pytest`? Name at least two concrete advantages of `pytest`."

**Interview — mid:** "Explain `pytest` fixtures and fixture scopes. When would you use `scope='session'` vs the default `scope='function'`, and what risk does the broader scope introduce?"

**Interview — senior:** "Design the test strategy for a service that calls an external payment API and writes to a database. What do you mock, what do you test with a real (test) database, and why — justify the boundary."

**Advanced / FAANG-style:** "A team's test suite is fast and green but the service still ships regressions regularly. Diagnose likely root causes in their testing discipline (connecting to over-mocking, brittle assertions, or missing integration-level coverage) and propose concrete changes to their test pyramid."

---

*Next: Module 18 — Production Engineering: logging, packaging/project structure, linting/formatting (Ruff, Black), and CI/CD basics — the practices that separate "runs on my machine" from a maintainable production codebase.*
