# Python binding — spec-driven-testing

## Shape extraction (Agent 1 additions)

Extract from Python source:
- Public functions and methods: those not prefixed with `_`
- Type annotations on parameters and return values — these are the contract; if absent, note "untyped" in the shape
- Docstrings: extract the behavioural contract (what it does, what it returns, what it raises) — not the implementation description
- `raises:` / `Raises:` sections in docstrings → include in Errors field
- Abstract base classes (`ABC`) and abstract methods are the primary API surface
- `@property` definitions are part of the public interface
- `@dataclass` and `TypedDict` fields are the shape — list all fields with types and defaults

Do not extract: methods prefixed with `_`, implementation details in docstrings ("we use a dict internally"), type: ignore comments.

**Type annotation gaps:** If a parameter has no annotation, mark it as `Any (untyped)` in the shape. This is a spec gap — Agent 2 cannot derive type-based boundary tests without it.

## Test framework (Agent 2)

Use pytest. No unittest unless the codebase already uses it.

```python
# test_module_name.py — file must be named test_*.py or *_test.py

import pytest

def test_function_name_scenario_expected_outcome():
    # Arrange
    # Act
    # Assert
```

No test class required for simple tests. Group related tests in a class only when shared setup justifies it.

## Fixtures — in-memory dependencies

```python
import pytest

@pytest.fixture
def db():
    import sqlite3
    conn = sqlite3.connect(":memory:")
    yield conn
    conn.close()

@pytest.fixture
def repository(db):
    repo = MyRepository(db)
    return repo
```

Fixtures are preferred over `setup`/`teardown` methods. Each fixture provides a fresh instance per test by default (`scope="function"`).

## Parametrize — equivalence partitions and boundary values

```python
import pytest

@pytest.mark.parametrize("input_value,expected", [
    ("valid_input",   "expected_output"),
    ("",              ""),           # empty string boundary
    (None,            None),         # null boundary
    ("MAX" * 1000,    ValueError),   # upper boundary (pass exception type to check raises)
])
def test_function(input_value, expected):
    if isinstance(expected, type) and issubclass(expected, Exception):
        with pytest.raises(expected):
            my_function(input_value)
    else:
        assert my_function(input_value) == expected
```

## Assertions

Use plain `assert` — pytest rewrites it with detailed output on failure:
```python
assert result == expected, f"Expected {expected!r}, got {result!r}"
assert result is not None
assert len(result) == 3
```

For exception testing:
```python
with pytest.raises(ValueError, match="expected message fragment"):
    function_that_should_raise(invalid_input)
```

## Format Assertion Requirement — Python specifics

For JSON or string serialization:
```python
import json

encoded = serialize(value)
# Round-trip
decoded = deserialize(encoded)
assert decoded == value, f"Round-trip failed: {decoded!r} != {value!r}"

# Format assertions
data = json.loads(encoded)
assert "expected_field" in data, f"Missing field in: {encoded}"
assert data["expected_field"] == "EXPECTED_VALUE"
assert "VariantName" in encoded  # for tagged unions / discriminated types
```

## Mocking — prefer fakes over mocks

```python
# Fake — implement only what the test needs
class FakeRepository:
    def __init__(self):
        self._items = {}

    def save(self, item):
        self._items[item.id] = item

    def find_by_id(self, id):
        return self._items.get(id)
```

Use `unittest.mock.patch` only for external I/O (network, filesystem, time) when a fake is not practical.

**Time:** Never use `time.time()` directly in production code under test — it makes tests non-deterministic. If the shape shows time-dependent behaviour, flag it in the spec gap report unless the shape exposes a way to inject a clock.

## Test structure pattern

File: `test_<module>.py`
Function: `test_<operation>_<scenario>_<expected_outcome>`

```python
def test_get_active_rules_with_one_disabled_returns_empty():
    # ...

def test_serialize_activity_trigger_encodes_variant_name():
    # ...
```
