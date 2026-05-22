# pytest built-in plugins

## Contents

- monkeypatch
- pytest-mock note (`mocker` fixture)
- tmp_path and tmp_path_factory
- capsys / capfd
- caplog

## monkeypatch

Function-scoped. Every `setattr`, `setenv`, `setitem`, `chdir`, and related call is reversed
in LIFO order at test exit. Direct `os.environ` mutation leaks across the session; always go
through monkeypatch.

```python
def test_uses_proxy(monkeypatch):
    monkeypatch.setenv("HTTP_PROXY", "http://proxy:8080")
    monkeypatch.setattr("app.services.http_client", FakeClient())
    result = run_fetch("/items")
    assert result.status == 200
```

**Patch at the import site, not the definition site.** If `services.py` does `from app.clients
import http_client`, patch `"app.services.http_client"` (where `services` looks it up), not
`"app.clients.http_client"`.

**Minimal-blast-radius patching: `monkeypatch.context()`.**

```python
def test_partial_swap(monkeypatch):
    with monkeypatch.context() as m:
        m.setattr("time.time", lambda: 1234)
        result = under_test()
    # time.time restored at end of the `with`, not at end of test
```

## pytest-mock note (`mocker` fixture)

The `mocker` fixture (from the third-party `pytest-mock` package) wraps
`unittest.mock.patch` with pytest-style auto-cleanup. It is the most common interface for
building Mock objects in pytest projects:

```python
def test_calls_collaborator(mocker):
    fake_publisher = mocker.patch("app.services.publisher", autospec=True)
    run()
    fake_publisher.send.assert_called_once_with(expected_payload)
```

Always pass `autospec=True` so the mock rejects calls that the real object would not accept.
Without autospec, `Mock` silently accepts any signature and any attribute, which masks API
drift.

## tmp_path and tmp_path_factory

`tmp_path` is function-scoped (one dir per test). `tmp_path_factory` is session-scoped
(shared across tests).

```python
def test_writes_log(tmp_path):
    log_file = tmp_path / "app.log"
    write_log(log_file, "hello")
    assert log_file.read_text() == "hello\n"

@pytest.fixture(scope="session")
def shared_config_dir(tmp_path_factory):
    return tmp_path_factory.mktemp("config")
```

pytest keeps the last 3 runs of temp dirs by default (configurable via
`tmp_path_retention_count` / `tmp_path_retention_policy`). Useful for post-failure
inspection.

Avoid the legacy `tmpdir` / `tmpdir_factory` fixtures: they return `py.path.local`, not
`pathlib.Path`, and they are deprecated.

Requesting function-scoped `tmp_path` from a session-scoped fixture raises ScopeMismatch.
Use `tmp_path_factory` in broader-scope fixtures.

## capsys / capfd

`capsys` intercepts at the Python `sys.stdout` layer. `capfd` intercepts at the OS
file-descriptor layer, catching output from subprocesses and C extensions that bypass Python
streams.

```python
def test_prints_summary(capsys):
    print_summary({"a": 1})
    out, err = capsys.readouterr()  # snapshots and clears the buffer
    assert "a: 1" in out
```

Use `capfd` (not `capsys`) for subprocess output, otherwise it goes uncaptured.
`capsys.disabled()` lets output through to the real terminal inside a `with` block, useful
for debugging without removing the fixture.

Both have binary variants (`capsysbinary`, `capfdbinary`) for asserting on raw bytes.

## caplog

```python
def test_logs_warning_on_retry(caplog):
    with caplog.at_level(logging.WARNING, logger="myapp.publisher"):
        publish_with_retry(message)
    matching = [r for r in caplog.records
                if r.levelno == logging.WARNING and "retry" in r.message]
    assert len(matching) == 1
```

**The most common caplog mistake: not setting the level before the code runs.** Loggers below
their effective level drop records before caplog ever sees them. Always set the level
explicitly.

For loggers with `propagate=False`, target them by name:
`caplog.set_level(logging.INFO, logger="my.specific.logger")`.

**Prefer `caplog.records` over `caplog.text`.** Records expose structured attributes (level,
logger name, message, exc_info). String matching on `caplog.text` is brittle to log format
changes.

**Hazard: `logging.config.dictConfig` calls that replace handlers evict caplog's handler.**
If the code under test reconfigures logging, the captured records dry up. Either use
incremental configs in the code under test, or override the logging configuration via a
fixture.
