# pytest core idioms

## Contents

- Test anatomy
- Fixtures: yield teardown
- Scope hierarchy
- conftest.py: fixtures search upward
- Fixture ordering
- Parametrize
- Assertions
- Markers and xfail

## Test anatomy

```python
def test_user_save_persists(user_repo):
    # arrange
    user = User(name="Ada")
    # act
    user_repo.save(user)
    # assert
    assert user_repo.get(user.id) == user
```

Three sections, one act. If the arrange section is large or shared, move it to a fixture.

## Fixtures: yield teardown

```python
@pytest.fixture
def temp_db():
    db = Database.connect()
    yield db
    db.close()
```

Code before `yield` is setup; after `yield` is teardown. Teardown runs whether the test passes
or fails. It does NOT run if setup raises before the yield.

**Corollary: split fixtures that hold multiple resources.** If setup does two state-changing
operations and the second raises, the first leaks (no teardown). Give each resource its own
composable fixture.

## Scope hierarchy

From narrowest to broadest: `function`, `class`, `module`, `package`, `session`. Lower scope
cannot depend on higher; higher can depend on lower (the broader fixture is created first and
lives longer).

```python
@pytest.fixture(scope="session")
def http_client(): ...

@pytest.fixture
def auth_token(http_client):  # function scope requests session scope: fine
    ...
```

Default scope is `function` (fresh fixture per test); keep that default for anything mutable.

## conftest.py: fixtures search upward

A test under `tests/unit/` sees fixtures from `tests/unit/conftest.py` and
`tests/conftest.py`, never from `tests/integration/conftest.py`. Same-named fixtures closer to
the test shadow ones higher up.

```
tests/
|-- conftest.py          # visible everywhere under tests/
|-- unit/
|   |-- conftest.py      # only visible to unit/
|   `-- test_x.py
`-- integration/
    |-- conftest.py      # only visible to integration/
    `-- test_y.py
```

## Fixture ordering

When two same-scope fixtures have no dependency between them, execution order is unspecified.
Encode dependencies explicitly:

```python
@pytest.fixture
def clean_db(): ...

@pytest.fixture
def seeded_db(clean_db):     # forces clean_db to run first
    insert_seed_rows()
```

## Parametrize

```python
@pytest.mark.parametrize("value,expected", [
    pytest.param("",   ValueError, id="empty_rejected"),
    pytest.param("0",  0,          id="zero_parses"),
    pytest.param("-1", -1,         id="negative_parses"),
])
def test_parse_int(value, expected): ...
```

- **Always give IDs.** Without them, CI shows `test_parse_int[--1--1]` and you cannot tell what
  failed. Use `pytest.param(..., id="...")` or pass `ids=[...]` / a callable.
- **Parameters are not copied between invocations.** Mutating a mutable parameter (dict, list)
  contaminates later runs silently. Copy inside the test if you intend to mutate.
- **Stacked parametrize is the cartesian product.** Two stacked decorators with N and M values
  generate N times M tests. Stack only when every combination is meaningful; otherwise build
  the scenario list explicitly.
- **Indirect parametrize** routes parameter values into a fixture instead of into the test
  directly. Use it when the parameter needs preparation (a built object, a connected client).

## Assertions

Use plain `assert`. pytest rewrites the bytecode to produce introspective failure output.
Never use unittest-style methods (`assertEqual`, `assertIn`).

Assertions in helper modules outside the test files are not rewritten by default. Register
them in conftest.py before they get imported:

```python
# conftest.py at the top
import pytest
pytest.register_assert_rewrite("tests.helpers")
```

**Floats: `pytest.approx`.**

```python
assert result == pytest.approx(0.1 + 0.2)
assert vector == pytest.approx([1.0, 2.0, 3.0])
```

Works for scalars, sequences, dicts, and numpy arrays. Tolerance is configurable per call
(`abs`, `rel`).

**Exceptions: `match=` is a regex search.**

```python
with pytest.raises(ValueError, match=r"out of range"):
    parse(-5)
```

`match=` uses `re.search`, so it matches anywhere in the message. `pytest.raises` accepts
subclasses. To assert the exact type:

```python
with pytest.raises(ValueError) as exc_info:
    parse(-5)
assert exc_info.type is ValueError  # not a subclass
```

## Markers and xfail

Register every custom marker and turn on strict checking:

```toml
[tool.pytest.ini_options]
addopts = "--strict-markers"
markers = [
    "slow: tests that hit the network",
    "integration: requires running services",
]
```

Without `--strict-markers`, typos silently no-op. Marks applied to fixtures have no effect
(common mistake); apply marks to tests.

**`xfail`: use `strict=True`.**

```python
@pytest.mark.xfail(strict=True, reason="upstream bug #123")
def test_known_broken(): ...
```

Without strict, a passing xfail is reported as XPASS but does not fail the suite, silently
hiding the fact that the bug got fixed.
