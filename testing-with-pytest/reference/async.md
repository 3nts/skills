# pytest-asyncio

## Mode

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

`asyncio_mode = "auto"` is the right default for asyncio-only projects. It treats every
`async def test_*` as an asyncio test and every `async def` fixture as an async fixture,
with no decorators needed.

`strict` mode (the default when unset) requires `@pytest.mark.asyncio` on every test and
`@pytest_asyncio.fixture` on every async fixture. Use strict only when mixing asyncio with
another framework via anyio.

## Async fixtures

```python
@pytest_asyncio.fixture
async def mqtt_client(broker_url):
    client = AsyncMqttClient(broker_url)
    await client.connect()
    yield client
    await client.disconnect()
```

Yield-style is the canonical pattern for async resource lifecycle. In strict mode, plain
`@pytest.fixture` on an async function is deprecated and warns; treat it as already broken.

## Loop scope

Default loop scope is function (fresh loop per test). For fixtures that hold long-lived
async resources (servers, connection pools), use a wider loop:

```python
@pytest_asyncio.fixture(scope="session", loop_scope="session")
async def opcua_server():
    server = await start_server()
    yield server
    await server.stop()
```

Tests consuming a session-scoped async fixture must also run in that loop. Either annotate
per module:

```python
pytestmark = pytest.mark.asyncio(loop_scope="session")

async def test_reads_value(opcua_server):
    ...
```

Or set the defaults globally:

```toml
[tool.pytest.ini_options]
asyncio_default_fixture_loop_scope = "session"
asyncio_default_test_loop_scope = "session"
```

**Invariant: `loop_scope >= scope`.** The loop must outlive the fixture's cache lifetime. A
session-scoped fixture needs a session-scoped loop. The reverse is fine: a function-scoped
fixture on a session loop is just recreated each test on the same loop.

## Asyncio library pitfalls

Connection objects from libraries like `asyncua` and `aiomqtt` capture the running event
loop at creation. If a fixture creates the connection on the session loop but the test runs
on a function loop, you get `RuntimeError: ... bound to a different event loop`.

Fix: match `loop_scope` between the fixture and the test, or set the global default. For
session-scoped servers, every consumer test must opt into the session loop.

## Migration: the `event_loop` fixture is gone

pytest-asyncio 1.0 removed the override-the-`event_loop`-fixture pattern. Use `loop_scope=`
and the `asyncio_default_*_loop_scope` configuration instead. Old code that overrides
`event_loop` needs to migrate.

## Alternative: anyio + pytest-anyio

anyio provides `@pytest.mark.anyio` for backend-agnostic testing across asyncio and trio.
It runs fixture setup, teardown, and the test body in a single async task, which enables
proper `ContextVar` propagation and TaskGroup usage inside fixtures. For asyncio-only
projects, pytest-asyncio is the right choice; reach for anyio if you need trio
compatibility.
