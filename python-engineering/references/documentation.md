# Documentation Standards

## Module Docstrings

Every module that defines an extensible component has:
1. A one-line summary
2. A full extension guide — step by step, with a minimal and a full example
3. The rules — what must be implemented, what is optional, what never to touch

```python
"""
storage/base.py
---------------
Abstract base and minimal shared config for storage backends.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
HOW TO ADD A NEW STORAGE BACKEND
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

MINIMAL BACKEND (no extra config):

    class MyBackend(StorageBackend):
        name = "my_backend"

        def write(self, key, data): ...
        def read(self, key): ...

    from_config() and default_config() are inherited.
    Your backend appears in config automatically.

BACKEND WITH EXTRA CONFIG:

    @dataclass
    class MyConfig(StorageConfig):
        region: str = "us-east-1"

    class MyBackend(StorageBackend):
        name = "my_backend"

        @classmethod
        def from_config(cls, raw):
            return cls(MyConfig(**raw))

        @classmethod
        def default_config(cls):
            base = super().default_config()
            base["region"] = "us-east-1"
            return base

        def write(self, key, data): ...
        def read(self, key): ...

RULES:
    - write() and read() must always be implemented
    - from_config() — override only if using a custom config type
    - default_config() — override only if you have extra fields
    - Never edit existing files to register a new backend
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
"""
```

---

## Class and Method Docstrings

- Classes: what the class is, what it owns, what it does not own
- Methods: args, returns, failure semantics, what it never does
- Abstract methods: the full contract including failure behaviour

Write docstrings that explain the contract — not ones that restate the signature.

```python
# Useless
def write(self, key: str, data: bytes) -> None:
    """Write data to storage."""

# Useful
def write(self, key: str, data: bytes) -> None:
    """
    Persist data under key. Overwrites silently if key exists.
    Must be implemented — no universal default exists across backends.
    Never raises — log and return on any error.
    """
```

---

## Inline Comments

Comment why, not what. The code says what. Comments say why it had to be this way, or what subtle constraint is being respected.

```python
# Fail open — the validator is advisory. A crash here would block
# all writes. The caller checks the result and decides whether to proceed.
except requests.ConnectionError:
    logger.warning("Validator unreachable — skipping validation")
    return ValidationResult(valid=True, reason="Validator unavailable")
```
