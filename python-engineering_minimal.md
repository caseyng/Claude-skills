PYTHON ENGINEERING — 1 MIN REFERENCE

ARCHITECTURE
- DIP: inject dependencies, never instantiate inside
- Registry: only store/resolve by name. No construction, no if/elif
- Provider: owns construction logic (use when selection or assembly needed)
- Composition root: one place wires concretes. Config never travels past here

EXTENSIBILITY
- ABC with: name str, from_config(cls, raw), default_config(cls), abstract methods
- Adding impl = no changes to other files

CONFIG
- Raw dict at boundary for extensible sections
- Typed dataclass for fixed sections
- write_default() queries registry — no hardcoded templates

TYPE HINTS
- Use X | None. Return types required on public interfaces

DATACLASSES
- Data only. __post_init__ validates, never coerces
- Typed result: @dataclass class Result: success: bool; reason: str | None = None

__REPR__
- Every injected class: f"{ClassName}(field={self._field!r})"

RESOURCES
- with open(..., encoding="utf-8")
- Classes with resources: close() + __enter__/__exit__

MUTABLE DEFAULTS
- def f(items=None): items = items or []
- dataclass: field(default_factory=list)

LOGGING
- logger = logging.getLogger(__name__)
- Never print() for errors. Never leak secrets. Never alter flow

EXCEPTIONS
- Catch narrowest type. Broad except requires documented justification
- assert only for tests/invariants, never external input
- Invariant/programmer error: raise. Operational failure: return structured result

THREAD SAFETY
- Lock around registry writes. GIL != thread-safe for compound ops

TESTING
- Test behaviour, not implementation
- One class per behaviour group (TestXFailure, not TestX)
- Stubs inline. No mocking frameworks for simple stubs
- Integration tests: skip unless INTEGRATION env var

DESIGN SMELLS (flag before coding)
- Class instantiates own dependencies
- Config object passed >1 level deep
- Registry with if/elif or that constructs
- Hardcoded list where discovery should work
- Broad except Exception without justification
- open() without encoding
- Mutable default arg
- Comment saying "for now" or "temporary"