# Architecture and Design Patterns

## Dependency Inversion Principle

High-level modules depend on abstractions, not concrete implementations. Non-negotiable.

- Business logic depends on abstractions
- Leaf modules and infrastructure adapters may depend on concrete implementations when no variation is expected
- Dependencies are injected, not instantiated internally
- The composition root is the only place that knows about concrete classes
- Inject the value, not the container — if a component needs one field from a config object, receive that field directly

```python
# Wrong — hardcodes a concrete class
class ReportGenerator:
    def __init__(self):
        self._storage = S3Storage()

# Wrong — receives config just to extract one value
class ReportGenerator:
    def __init__(self, cfg: StorageConfig):
        self._bucket = cfg.bucket_name  # feature envy

# Right
class ReportGenerator:
    def __init__(self, storage: StorageBackend, bucket_name: str):
        self._storage     = storage
        self._bucket_name = bucket_name
```

---

## Strategy Pattern

Families of interchangeable implementations are selected via registry — never via `if/elif` chains on a config string.

```python
# Wrong
if cfg.format == "csv":
    exporter = CsvExporter(cfg)
elif cfg.format == "json":
    exporter = JsonExporter(cfg)

# Right
cls      = exporters.resolve(cfg.format)
exporter = cls.from_config(cfg)
```

---

## Registry Pattern

Registries store and retrieve classes by name. Nothing else. They do not construct, make decisions, or special-case.

```python
# Wrong — registry doing construction
def get(cfg):
    if cfg.storage == "file":
        return _registry["file"](cfg.path)
    return _registry[cfg.storage]()

# Right — registry only resolves the class
def resolve(name: str) -> type:
    if name not in _registry:
        raise PluginNotFoundError("storage", name, list(_registry.keys()))
    return _registry[name]
```

Construction belongs in provider functions.

---

## Explicit Provider Functions

Required when construction involves selection logic, configuration binding, or dependency assembly. Trivial constructions do not require provider abstraction.

```python
# Requires a provider
def provide_storage(raw: dict) -> StorageBackend:
    name = "local" if raw.get("local_path") else "s3"
    cls  = storage.resolve(name)
    return cls.from_config(raw)

# Does not require a provider
report_id = uuid4()
```

---

## Composition Root

One place wires everything together. It knows about concrete classes. Everything below depends only on abstractions. Typically `build()` in `__init__.py` or a dedicated `container.py`.

---

## Extensibility

**Adding a new implementation must not require modifying unrelated modules.**

Changes are limited to the implementation itself and its explicit registration boundary. If unrelated modules must change, that is a design smell worth discussing before proceeding.

### The Extension Protocol

Every extensible base class provides three things:

```python
class StorageBackend(ABC):

    name: str  # registry key — must be set on every subclass

    @classmethod
    def from_config(cls, raw: dict) -> "StorageBackend":
        """Default: construct from base config. Override if needed."""
        ...

    @classmethod
    def default_config(cls) -> dict:
        """Default: return base config fields. Override to add extra fields."""
        ...

    @abstractmethod
    def write(self, key: str, data: bytes) -> None:
        """Must implement — no universal default exists."""
        ...
```

- `from_config` and `default_config` have sensible base class defaults — minimal extensions inherit them
- The core operation is `@abstractmethod` — every subclass must implement it
- `default_config` feeds config generation automatically — new extensions appear in generated config without changes to config code
