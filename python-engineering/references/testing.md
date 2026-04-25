# Testing Standards

## What to Test

- Behaviour, not implementation. Test what the component does, not how it does it.
- Every failure mode. If it can return `None`, test that. If it can fail open, test that.
- Every contract stated in a docstring. If it says "never raises", test that it does not raise on adversarial input.
- The extension protocol. Test that `from_config()` and `default_config()` work on every extensible component.

---

### What Not to Test

- **Internal helpers** — functions that exist only to decompose a public method. Test the public method; the helper is an implementation detail.
- **Trivial getters/setters** — `def get_name(self): return self._name` adds no behaviour to test.
- **Framework glue** — if Flask/FastAPI/Django does something, assume it works. Test your callbacks, not the routing registration.
- **The registry’s storage mechanics** — test that `resolve(name)` returns the right class, not that the internal dict works. That’s Python’s job.
- **Compile-time type checks disguised as tests** — `isinstance()` assertions on injected dependencies are usually testing your annotation, not behaviour. Inject a stub and test what the component does with it.

**Smell:** A test that changes after refactoring that didn’t change behaviour. That test was testing implementation, not behaviour.

---

## Test Structure

One test class per behaviour group, not per method. Test class names describe the scenario.

```python
# Right
class TestWriteFailure(unittest.TestCase):
    def test_connection_error_returns_none(self): ...
    def test_timeout_returns_none(self): ...
    def test_write_never_raises(self): ...

# Wrong
class TestWrite(unittest.TestCase):
    def test_write_1(self): ...
    def test_write_2(self): ...
```

---

## Stubs

Stubs are minimal. They implement exactly the interface needed — nothing more. Defined inline in the test file unless genuinely shared across multiple test files.

```python
class _StubStorage:
    name = "stub"
    def write(self, key, data): pass
    def read(self, key): return b"data"
    def close(self): pass
    def __repr__(self): return "StubStorage()"
```

No mocking frameworks for simple stubs. Use `unittest.mock` only for verifying interactions — call counts, argument capture.

---

## Separating Unit and Integration Tests

Tests that require a running external service, a real database, or a network call are integration tests. They live in a separate directory or are marked with a skip decorator. They never run in the default `python -m unittest discover` pass.

```python
@unittest.skipUnless(os.getenv("INTEGRATION"), "integration tests disabled")
class TestS3BackendIntegration(unittest.TestCase):
    ...
```
