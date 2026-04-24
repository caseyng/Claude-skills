# Python-Specific Patterns

## Type Hints

All public and cross-module interfaces are fully type-hinted. Return types included.
Internal helpers may omit annotations when scope is narrow and intent is clear — exception, not default.

Use `X | None` for nullable types. `Optional[X]` acceptable in existing codebases — do not mix styles within a module.

```python
# Wrong
def process(self, data, config=None):
    ...

# Right
def process(
    self,
    data:   bytes,
    config: Optional[ProcessConfig] = None,
) -> ProcessResult:
    ...
```

---

## Dataclasses for Config

Config objects are typed dataclasses. Data carriers — no logic, no type coercion in `__post_init__`. Validation only.

```python
@dataclass
class ProcessorConfig:
    batch_size:  int   = 100
    timeout_s:   float = 30.0
    max_retries: int   = 3

    def __post_init__(self):
        if self.batch_size < 1:
            raise ValueError(f"batch_size must be >= 1, got {self.batch_size}")
```

Type coercion from raw dicts belongs in `from_dict()` — explicit, visible, not hidden in constructors.

---

## ABCs

Every extensible component has an ABC. ABCs define the contract, not the implementation.

- Abstract methods only for operations with no sensible default
- Base implementations for operations with sensible defaults
- Document the full contract in the module docstring
- Include an extension guide with a minimal and full example

---

## `__repr__`

Every class that gets injected, logged, or appears in error messages implements `__repr__`. Must be informative enough to identify the instance in a log without opening a debugger.

```python
def __repr__(self) -> str:
    return f"S3StorageBackend(bucket={self._bucket!r}, region={self._region!r})"
```

A `__repr__` that returns `<MyClass object at 0x...>` is not a `__repr__`.

---

## Resource Management

Never rely on garbage collection to release external resources. Always release explicitly.

**Use context managers for any resource that must be released.**

| Resource | Leak risk | Resolution |
|---|---|---|
| File handles | File stays open, writes may not flush | `with open(path, encoding="utf-8") as f:` |
| Network connections | Socket held open, pool exhausted | `with requests.Session() as session:` |
| Database connections | Pool starved, transactions left open | `with db.connect() as conn:` |
| Thread pools | Threads not joined, process hangs | `with ThreadPoolExecutor() as pool:` |
| Subprocess handles | Zombie process, fd leak | `with subprocess.Popen(...) as proc:` |
| Locks | Deadlock if exception raised | `with self._lock:` — never acquire manually |
| Temp files | Disk accumulation | `with tempfile.NamedTemporaryFile() as f:` |

**For class-lifetime resources**, implement `close()` and support the context manager protocol:

```python
class DatabaseClient:
    def __init__(self, url: str):
        self._conn = psycopg2.connect(url)

    def close(self) -> None:
        if self._conn and not self._conn.closed:
            self._conn.close()

    def __enter__(self) -> "DatabaseClient":
        return self

    def __exit__(self, *_) -> None:
        self.close()
```

Flag resource leaks — any `open()`, connection constructor, or pool initialiser not inside a `with` block or a class that implements `close()`.

---

## Autodiscovery

Use autodiscovery when runtime extensibility is a genuine requirement. Prefer explicit registration when startup determinism, dependency visibility, or security policy matters.

When using explicit registration, document why autodiscovery was rejected.

```python
def _autodiscover() -> None:
    for _, module_name, _ in pkgutil.iter_modules(__path__):
        if module_name in ("base",):
            continue
        module = importlib.import_module(f"{__name__}.{module_name}")
        for attr_name in dir(module):
            attr = getattr(module, attr_name)
            try:
                if (
                    isinstance(attr, type)
                    and issubclass(attr, PluginBase)
                    and attr is not PluginBase
                    and hasattr(attr, "name")
                ):
                    _registry[attr.name] = attr
            except TypeError:
                # issubclass() raises TypeError on non-class objects
                continue
```

---

## Mutable Default Arguments

Default argument values are evaluated once at function definition time. Mutable defaults are shared across all calls — silent state corruption.

```python
# Wrong
def process(self, data: bytes, tags: list = []) -> None:
    tags.append("processed")  # mutates the shared default

# Right
def process(self, data: bytes, tags: Optional[List[str]] = None) -> None:
    if tags is None:
        tags = []
    tags.append("processed")
```

Dataclass fields with mutable defaults use `field(default_factory=...)`:

```python
# Wrong
@dataclass
class Config:
    tags: list = []

# Right
@dataclass
class Config:
    tags: List[str] = field(default_factory=list)
```

---

## Logging Over Print

`print()` has no level, no handler, no way to suppress in production. Use `logging` for anything not deliberate CLI output.

```python
import logging
logger = logging.getLogger(__name__)

# Wrong
print(f"  ✗ Connection failed: {e}")

# Right
logger.warning("Connection to %s failed: %s", self._url, e)
```

Use `logger.exception()` inside `except` blocks — captures traceback automatically.

Get the logger at module level. Never configure logging inside a library.

**Three constraints:**
- Logging must never alter control flow
- Logging must never leak secrets — log presence, not content: `logger.debug("key: present=%s", bool(api_key))`
- Avoid logging in hot paths — use `if logger.isEnabledFor(logging.DEBUG):` to guard expensive statements

---

## Exception Handling

Catch the narrowest exception type that covers the known failure mode. Unknown exceptions should propagate.

```python
# Wrong
try:
    result = self._client.post(url, data=payload)
    return result.json()["items"]
except Exception as e:
    return None

# Right
try:
    response = self._client.post(url, data=payload, timeout=self._timeout)
    response.raise_for_status()
    return response.json()["items"]
except requests.ConnectionError:
    logger.warning("Storage service unreachable at %s", url)
    return None
except requests.Timeout:
    logger.warning("Storage request timed out after %ss", self._timeout)
    return None
except requests.HTTPError as e:
    logger.error("Storage HTTP error: %s", e)
    return None
except (KeyError, ValueError) as e:
    logger.error("Unexpected response format: %s", e)
    return None
```

When a broad catch is genuinely necessary, document exactly why.

---

## Thread Safety

Module-level registries are singletons shared across all threads. If anything writes to a registry after startup, protect it explicitly:

```python
import threading

_registry: dict[str, type] = {}
_lock = threading.Lock()

def register(name: str, cls: type) -> None:
    with _lock:
        _registry[name] = cls
```

For class instances with mutable state accessed from multiple threads, protect every read-modify-write — not just writes.

Do not assume the GIL makes thread safety free. The GIL protects individual bytecode operations — not compound operations.

---

## Explicit Encoding

Always specify `encoding="utf-8"` when opening files. The default is platform-dependent.

```python
# Wrong
with open(path) as f:
    data = f.read()

# Right
with open(path, encoding="utf-8") as f:
    data = f.read()
```

Applies to `json.dump`, `csv.writer`, `logging.FileHandler`, and any other API that accepts an encoding argument.

---

## Failure Semantics: Raise or Return

Apply in order:

1. **Invariant violations** — raise immediately. Internal state that should be impossible is a bug.
2. **Programmer errors** — raise immediately. Wrong argument type, `None` passed to non-optional. Do not convert to `None`.
3. **Expected operational failures** — return a structured result. Network timeouts, service unavailable, item not found.
4. **Boundary-level policy** — may translate exceptions into results at architectural boundaries.

**On `assert`:** stripped by `python -O`. Never use for runtime validation of external input. Use only in tests and internal invariant checks.

---

## Typed Result Objects

Use for domain-level operations where structured failure context is needed — not universally.

```python
@dataclass
class ProcessResult:
    success: bool
    output:  Optional[str] = None
    reason:  Optional[str] = None

    def __bool__(self) -> bool:
        return self.success
```

---

## Specific Exception Types

Define specific exception types per failure mode. Never raise bare `Exception` or `ValueError` for domain errors.

```python
class AppError(Exception):
    """Base for all application exceptions."""

class PluginNotFoundError(AppError):
    def __init__(self, registry: str, name: str, available: list):
        super().__init__(
            f"No '{name}' registered in {registry}. "
            f"Available: {available}"
        )

class ConfigurationError(AppError):
    """Raised when config is invalid or missing required fields."""
```
