# pytest anti-patterns

## Contents

- 1. Over-mocking / mock mirrors implementation
- 2. God fixtures and autouse abuse
- 3. Test interdependence
- 4. Brittle exact-string assertions
- 5. sleep-based waiting in async tests
- 6. `request.getfixturevalue` indirection
- 7. Parametrize without IDs
- 8. Patching at definition site instead of import site
- 9. conftest.py sprawl
- 10. Marker drift

## 1. Over-mocking / mock mirrors implementation

**Signals:** three or more levels of `mock.return_value` chaining; `assert_called_with` on
internal methods of the system under test; patching the ORM session directly; mocks that
have to be updated on every refactor even though the contract is unchanged.

**Fix:** ask "is there a real-but-fast fake I could use here?" (in-memory dict, fake repo,
test double from the same package). Use `autospec=True` on every mock. Reserve mocks for
outgoing command messages at process boundaries (Sandi Metz's rule).

## 2. God fixtures and autouse abuse

**Signals:** a fixture longer than most tests; `autouse=True` on something that is not pure
cleanup or reset; one fixture that sets up the database, mocks two services, and configures
env vars in a single block.

**Fix:** decompose into composable fixtures, one resource each. Reserve `autouse` for
cross-cutting cleanup (e.g., reset a global registry between tests). Scope to the lowest
conftest.py level where all consumers can reach it.

## 3. Test interdependence

**Signals:** tests that pass alone but fail in the suite (or vice versa); session-scoped
fixtures that mutate shared state across tests; module-level mutable helpers.

**Detection:** run with `pytest-randomly` or `pytest-random-order` for a few CI cycles.
Tests with hidden coupling start failing intermittently.

**Fix:** function-scope mutable fixtures; explicit yield teardown that resets shared state.

## 4. Brittle exact-string assertions

**Signals:**

```python
assert str(exc.value) == "User not found: id=42"
assert caplog.text == "WARNING:app:retry attempt 1"
```

**Fix:**

```python
with pytest.raises(UserNotFound, match=r"id=42"):
    ...
assert any(r.levelno == logging.WARNING and "retry" in r.message
           for r in caplog.records)
```

Assert on type plus a key substring for exceptions; use record attributes for logs.

## 5. sleep-based waiting in async tests

**Signals:** `await asyncio.sleep(0.5)` followed by an assertion; "works on my machine"
sleep durations that fail under CI load.

**Fix:** poll a condition with a timeout, or `await` the actual coroutine that produces the
state:

```python
async def wait_for(predicate, timeout=2.0, interval=0.05):
    loop = asyncio.get_running_loop()
    deadline = loop.time() + timeout
    while loop.time() < deadline:
        if predicate():
            return
        await asyncio.sleep(interval)
    raise TimeoutError
```

## 6. `request.getfixturevalue` indirection

**Signals:**

```python
def test_x(request):
    db = request.getfixturevalue("db")
```

**Why it bites:** the dependency is invisible to readers and to pytest's static dependency
resolver. Known consequences: incorrect teardown order, broken parametrize support, crashes
on already-finalized fixtures.

**Fix:** declare the fixture as a function argument:

```python
def test_x(db): ...
```

Use indirect parametrize if you need parametrize to drive fixture inputs.

## 7. Parametrize without IDs

**Signal:** CI output shows `test_publish[True-0-{}-something]` and you cannot tell what
failed.

**Fix:** `pytest.param(..., id="case_name")` on every case, or pass `ids=[...]` /
`ids=callable` to `parametrize`.

## 8. Patching at definition site instead of import site

**Signal:** `monkeypatch.setattr("requests.get", ...)` patches the canonical location, but
if `app.services` did `from requests import get`, the bound name in `services` still
references the original.

**Fix:** patch where the name is looked up: `monkeypatch.setattr("app.services.get", ...)`.

## 9. conftest.py sprawl

**Signals:** conftest.py at every directory level; the same fixture redefined three times
at different scopes; fixtures defined far from the tests that use them.

**Fix:** consolidate to the highest level where all consumers can see them, following the
upward-search rule. One conftest.py per real domain split, not one per directory.

## 10. Marker drift

**Signals:** `@pytest.mark.smoke` on a test but `smoke` is not in the registered markers
list; marks applied to fixtures (which have no effect).

**Fix:** turn on `--strict-markers` so unregistered marks become collection errors. Register
every custom marker in `[tool.pytest.ini_options].markers`. Apply marks to tests, not
fixtures.
