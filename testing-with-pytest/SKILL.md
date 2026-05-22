---
name: testing-with-pytest
description: >
  Patterns and anti-patterns for writing and reviewing Python tests that use pytest.
  Covers core idioms (fixtures, parametrize, assertions, markers, conftest scoping),
  built-in isolation plugins (monkeypatch, tmp_path, capsys, caplog), first-class
  pytest-asyncio coverage (asyncio_mode, loop scope, async fixtures, common pitfalls),
  and review heuristics. Invoke when authoring or reviewing test_*.py / *_test.py /
  conftest.py, debugging fixture-scope or event-loop issues, or designing a Python
  project's test architecture. Skip for unittest-only or doctest-only modules.
---

# pytest patterns

Reference for authoring and reviewing pytest tests. This file holds the high-frequency
content (design principles, anti-pattern scan, quick reference). Detailed material is
one level deep under `reference/`.

## Design principles

- **One act per test.** The act is the single state-changing call. Arrange and cleanup
  belong in fixtures. Multi-act tests are harder to diagnose; split them.
- **Test behavior, not implementation.** Assert on outputs and observable side effects,
  not on which internal methods got called. `assert_called_with` on internal
  collaborators couples tests to the implementation.
- **Descriptive names beat compact ones.** `test_publish_retries_on_broker_timeout`
  documents intent; `test_publish_3` does not.
- **Sociable tests over isolated ones when a real fake is cheap.** Mock only at process
  boundaries you cannot control (network, time, randomness). For internal collaborators,
  prefer in-memory fakes.
- **Compose small fixtures.** A fixture should own one resource. Avoid god fixtures.
- **Always use `autospec=True`** when mocking. Plain `Mock()` silently masks API drift.

## Anti-patterns to flag in review

Quick scan list. Each one has signals and concrete fix code in
`reference/anti-patterns.md`.

1. **Over-mocking.** Mocks chained 3+ levels, `assert_called_with` on internals,
   patching the ORM session directly.
2. **God fixtures + autouse abuse.** One fixture does DB + service mocks + env;
   `autouse=True` on non-cleanup.
3. **Test interdependence.** Tests that pass alone but fail in suite; session-scope
   fixtures with mutable state.
4. **Brittle exact-string assertions.** `assert str(exc) == "..."` or full
   `caplog.text` matches.
5. **sleep-based waiting in async.** `await asyncio.sleep(0.5)` before an assertion.
6. **`request.getfixturevalue` indirection.** Dynamic lookup hides dependencies from
   readers and from pytest's resolver.
7. **Parametrize without IDs.** CI shows `test[1-2-3]` and you cannot tell what failed.
8. **Patching at definition site.** `monkeypatch.setattr("requests.get", ...)` instead
   of `app.services.get`.
9. **conftest.py sprawl.** conftest at every directory; same fixture redefined.
10. **Marker drift.** Unregistered marks; marks on fixtures (no effect).

## Quick reference

| Need                          | Pattern                                                  |
|-------------------------------|----------------------------------------------------------|
| Setup + teardown              | `@pytest.fixture` with `yield`                           |
| Same logic, many cases        | `@pytest.mark.parametrize(..., ids=[...])`               |
| Float compare                 | `pytest.approx(x)`                                       |
| Exception assert              | `pytest.raises(T, match=r"...")`                         |
| Env var                       | `monkeypatch.setenv(name, value)`                        |
| Module global swap            | `monkeypatch.setattr("pkg.mod.name", new)`               |
| Time-bounded swap             | `with monkeypatch.context() as m:`                       |
| Filesystem (per test)         | `tmp_path`                                               |
| Filesystem (shared)           | `tmp_path_factory.mktemp("name")`                        |
| Capture stdout (Python)       | `capsys.readouterr()`                                    |
| Capture stdout (subprocess)   | `capfd.readouterr()`                                     |
| Log assertion                 | `caplog.records` + `caplog.set_level(level, logger=...)` |
| Mock with autospec            | `mocker.patch("pkg.mod.name", autospec=True)`            |
| Async test (auto mode)        | `async def test_x(...)`                                  |
| Async resource fixture        | `@pytest_asyncio.fixture` + yield                        |
| Session-scoped async fixture  | `scope="session", loop_scope="session"`                  |
| Async test on session loop    | `pytestmark = pytest.mark.asyncio(loop_scope="session")` |
| Skip combinator               | `pytest.param(..., marks=pytest.mark.skipif(...))`       |
| Expected failure              | `@pytest.mark.xfail(strict=True, reason="...")`          |
| Custom marker                 | Register in `[tool.pytest.ini_options].markers`          |
| Catch leaks early             | `pytest-randomly` in CI                                  |

## Deeper references

- **`reference/core-idioms.md`**: test anatomy, fixture yield + scope hierarchy,
  conftest upward search, fixture ordering, parametrize (IDs, mutation hazard,
  cartesian, indirect), assertions (rewrite, approx, raises), markers + xfail.
- **`reference/plugins.md`**: monkeypatch, pytest-mock note, tmp_path /
  tmp_path_factory, capsys vs capfd, caplog.
- **`reference/async.md`**: asyncio_mode, async fixtures, loop scope invariant, library
  pitfalls (asyncua / aiomqtt loop binding), event_loop migration, anyio alternative.
- **`reference/anti-patterns.md`**: the 10 anti-patterns above with signals, why each
  bites, and concrete fix code.

## Sources

- pytest docs: https://docs.pytest.org/en/stable/
- pytest-asyncio docs: https://pytest-asyncio.readthedocs.io/en/latest/
- Martin Fowler, "Mocks Aren't Stubs":
  https://martinfowler.com/articles/mocksArentStubs.html
- Sandi Metz on testing seams; James Shore, "Testing Without Mocks"; Brian Okken,
  "Python Testing with pytest" (pythontest.com)
