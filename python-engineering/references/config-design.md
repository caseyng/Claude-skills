# Config Design

## Raw Dicts at the Boundary

Top-level config carries raw dicts for sections where implementations vary. Providers deserialise them via `from_config()`.

```python
@dataclass
class AppConfig:
    storage  : dict           = field(default_factory=dict)   # raw — provider owns
    exporter : dict           = field(default_factory=dict)   # raw — provider owns
    pipeline : PipelineConfig = field(default_factory=PipelineConfig)  # typed — no variation
```

`storage` is raw because the provider selects which class to use and which fields to extract.
`pipeline` is typed because there is no per-implementation variation.

---

## Discovery-Driven Config Generation

`write_default()` never contains a hardcoded template. It queries registered components.

```python
def write_default(path: str) -> None:
    from . import storage, exporters

    template = {
        "storage" : storage.resolve("local").default_config(),
        "exporter": exporters.resolve("json").default_config(),
    }

    with open(path, "w", encoding="utf-8") as f:
        json.dump(template, f, indent=4)
```

New implementation registered → appears in config automatically. The config module never needs editing.
