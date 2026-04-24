---
version: 1.1.0
date: 2026-04-24
skill: code-integrity-guardrail / python
---

# Python Guardrail Bindings

Concrete implementations of all universal code-integrity-guardrail concepts for
Python 3.10+. This file is read after invoking `/skill:code-integrity-guardrail python`.
It answers "how" for Python specifically.

---

## Security Bindings (Category 1)

### Hardcoded Secrets

**Python syntax to detect:**

```python
# All of these are violations
API_KEY = "sk-abc123"
password = "hunter2"
SECRET = "my-secret-value"
requests.get(url, headers={"Authorization": "Bearer abc123"})
psycopg2.connect("postgresql://user:password@host/db")
```

**Correct pattern:**

```python
import os
api_key = os.environ["API_KEY"]           # raises if missing — intentional
api_key = os.environ.get("API_KEY", "")   # silent fallback — only for optional values
```

**Tools:** `bandit -t B105,B106,B107` (hardcoded password checks), `trufflehog`,
`detect-secrets`.

---

### SQL Injection

**Python syntax to detect:**

```python
# All of these are violations — any driver, any ORM
cursor.execute(f"SELECT * FROM users WHERE id = {user_id}")
cursor.execute("SELECT * FROM users WHERE name = '%s'" % name)
cursor.execute("DELETE FROM logs WHERE date = " + date_str)
db.query("UPDATE users SET role = '" + role + "' WHERE id = " + id)
```

**Correct pattern:**

```python
# psycopg2 / sqlite3 — always use parameterized queries
cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
cursor.execute("SELECT * FROM users WHERE name = %s", (name,))

# SQLAlchemy ORM — use ORM methods
user = session.query(User).filter(User.id == user_id).first()

# SQLAlchemy Core — use bound parameters
stmt = select(users).where(users.c.id == bindparam("user_id"))
result = conn.execute(stmt, {"user_id": user_id})
```

**Tools:** `bandit -t B608` (SQL injection), `semgrep` with Python security ruleset.

---

### XSS (Cross-Site Scripting)

**Python syntax to detect:**

```python
# Flask — Markup() or |safe bypasses auto-escaping
return Markup(f"<div>{user_input}</div>")
return render_template_string(f"<p>{comment}</p>")

# Django — mark_safe() on unvalidated content
from django.utils.safestring import mark_safe
html = mark_safe(user_content)

# Any f-string or concatenation building HTML from user input
html = f"<h1>Welcome, {username}!</h1>"
```

**Correct pattern:**

```python
# Flask — Jinja2 auto-escapes by default in .html templates
# Use render_template(), not render_template_string() with f-strings
return render_template("profile.html", username=username)

# Django — templates auto-escape; never use mark_safe on user content
# If HTML content is intentional, sanitize first
import bleach
safe_html = bleach.clean(user_html, tags=ALLOWED_TAGS, attributes=ALLOWED_ATTRS)

# For APIs returning HTML — always escape
from markupsafe import escape
safe = escape(user_input)
```

**Empirical note:** 86% failure rate in AI-generated code. Treat as default-present
in any code that renders user input. Check every template variable.

**Tools:** `bandit -t B703,B704`, `semgrep` XSS rules.

---

### Log Injection

**Python syntax to detect:**

```python
# Any logging statement with raw user input — newlines enable log forgery
logger.info(f"User action: {user_input}")
logger.warning("Request from: " + request.headers.get("User-Agent"))
logger.debug("Processing: %s", raw_query_param)
```

**Correct pattern:**

```python
# Sanitize before logging — strip or replace control characters
def sanitize_for_log(value: str) -> str:
    return value.replace("\n", "\\n").replace("\r", "\\r")

logger.info("User action: %s", sanitize_for_log(user_input))

# Or use structured logging that serializes values safely
import structlog
log = structlog.get_logger()
log.info("user_action", input_length=len(user_input), action_type=action_type)
```

**Empirical note:** 88% failure rate in AI-generated code. Any logging of
request data, headers, query parameters, or form fields is a candidate.

---

### Command Injection / Insecure Subprocess

**Python syntax to detect:**

```python
# All of these are violations
os.system(f"grep {pattern} {filename}")
subprocess.run(f"convert {input_file} {output_file}", shell=True)
subprocess.call("ls " + directory, shell=True)
os.popen(f"wc -l {user_file}")
```

**Correct pattern:**

```python
# Use list arguments, shell=False (default)
subprocess.run(["grep", pattern, filename], capture_output=True, timeout=30)
subprocess.run(["convert", input_file, output_file], timeout=60)

# Always set timeout — never allow infinite subprocess execution
result = subprocess.run(
    ["ffmpeg", "-i", input_path, output_path],
    capture_output=True,
    timeout=300,
    check=True,
)
```

**Tools:** `bandit -t B602,B603,B604,B605,B606,B607`.

---

### Unsafe Dynamic Execution

**Python syntax to detect:**

```python
# All violations — any user-controlled input to these
eval(user_expression)
exec(user_code)
eval(request.args.get("formula"))
exec(f"result = {expression}")
compile(user_source, "<string>", "exec")
```

**No safe alternative for user input.** If dynamic evaluation is genuinely required,
use a restricted AST evaluator (e.g., `asteval`) with an explicit allowlist. Never
`eval()` or `exec()` on user-controlled strings.

**Tools:** `bandit -t B307`.

---

### Disabled TLS Verification

**Python syntax to detect:**

```python
# requests
requests.get(url, verify=False)
session.verify = False

# urllib / httpx
ssl_context = ssl.create_default_context()
ssl_context.check_hostname = False
ssl_context.verify_mode = ssl.CERT_NONE

# aiohttp
connector = aiohttp.TCPConnector(ssl=False)
```

**Correct pattern:**

```python
# Fix the certificate issue; never disable verification
# For custom CA:
requests.get(url, verify="/path/to/ca-bundle.crt")

# For internal services with self-signed certs in development:
# Set REQUESTS_CA_BUNDLE env var to the cert path
# Never disable in production
```

**Tools:** `bandit -t B501,B502`.

---

### Weak Cryptography

**Python syntax to detect:**

```python
# Password "hashing" with MD5 or SHA — never acceptable
import hashlib
hashlib.md5(password.encode()).hexdigest()
hashlib.sha1(password.encode()).hexdigest()
hashlib.sha256(password.encode()).hexdigest()  # also wrong for passwords — no salt/KDF

# DES, RC4, ECB mode — never acceptable
from Crypto.Cipher import DES
from Crypto.Cipher import ARC4
AES.new(key, AES.MODE_ECB)
```

**Correct pattern:**

```python
# Password hashing
import bcrypt
hashed = bcrypt.hashpw(password.encode(), bcrypt.gensalt(rounds=12))
bcrypt.checkpw(password.encode(), hashed)

# Or argon2
from argon2 import PasswordHasher
ph = PasswordHasher()
hash = ph.hash(password)
ph.verify(hash, password)

# Symmetric encryption — AES-GCM
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
key = AESGCM.generate_key(bit_length=256)
aesgcm = AESGCM(key)
ciphertext = aesgcm.encrypt(nonce, data, associated_data)
```

**Tools:** `bandit -t B303,B304,B305,B306`.

---

### Over-Permissive CORS

**Python syntax to detect:**

```python
# Flask-CORS
from flask_cors import CORS
CORS(app)                             # wildcard on all routes
CORS(app, origins="*")               # explicit wildcard
cors.init_app(app, origins="*")

# Django CORS headers
CORS_ALLOW_ALL_ORIGINS = True
CORS_ORIGIN_ALLOW_ALL = True

# FastAPI
app.add_middleware(CORSMiddleware, allow_origins=["*"])
```

**Correct pattern:**

```python
# Explicit origin allowlist
CORS(app, origins=["https://app.example.com", "https://admin.example.com"])

# FastAPI
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.example.com"],
    allow_credentials=True,
    allow_methods=["GET", "POST"],
    allow_headers=["Authorization", "Content-Type"],
)
```

---

### Path Traversal

**Python syntax to detect:**

```python
# Looks safe but allows ../../etc/passwd
filepath = os.path.join(BASE_DIR, user_filename)
with open(os.path.join(upload_dir, filename)) as f: ...
```

**Correct pattern:**

```python
import pathlib

def safe_path(base: str, user_input: str) -> pathlib.Path:
    base_path = pathlib.Path(base).resolve()
    requested = (base_path / user_input).resolve()
    if not str(requested).startswith(str(base_path)):
        raise ValueError(f"Path traversal attempt: {user_input!r}")
    return requested
```

---

### Information Disclosure in Logs

**Python syntax to detect:**

```python
logger.debug("API key: %s", api_key)
logger.info(f"Connecting with password {db_password}")
logger.error("Auth failed for token: %s", bearer_token)
print(f"Secret: {secret}")
```

**Correct pattern:**

```python
logger.debug("API key: present=%s", bool(api_key))
logger.info("DB connection: host=%s, user=%s", db_host, db_user)  # no password
logger.error("Auth failed for token: present=%s, prefix=%s", bool(token), token[:4] if token else "none")
```

---

### Missing Auth/Authz

**Python syntax to detect:**

```python
# Flask — route with no auth decorator
@app.route("/api/admin/users")
def list_users():
    return jsonify(User.query.all())

# FastAPI — endpoint with no dependency
@router.get("/admin/reports")
async def get_reports():
    return reports
```

**Correct pattern:**

```python
# Flask
@app.route("/api/admin/users")
@login_required
@require_role("admin")
def list_users():
    return jsonify(User.query.all())

# FastAPI
@router.get("/admin/reports")
async def get_reports(current_user: User = Depends(require_admin)):
    return reports
```

---

## Logic Bindings (Category 2)

### Boundary Errors

```python
# Off-by-one — check every range and slice
items[0]         # fails on empty list — guard: if items: ...
items[-1]        # fails on empty list — guard: if items: ...
items[len(items)] # IndexError — should be items[len(items)-1]
for i in range(1, len(items)):  # skips index 0 — intentional?

# Correct patterns
first = items[0] if items else default
last = items[-1] if items else default
```

### Null/None Handling

```python
# Violations — dereference before guard
result = db.query(User).filter_by(id=user_id).first()
name = result.name  # AttributeError if result is None

# Correct
result = db.query(User).filter_by(id=user_id).first()
if result is None:
    raise UserNotFoundError(user_id)
name = result.name
```

### Type Comparison

```python
# Violations
if x == None:    # wrong — use `is None`
if x == True:    # wrong — use `if x:` or `if x is True:`
if type(x) == int:  # fragile — use isinstance()

# Correct
if x is None:
if x:
if isinstance(x, int):
```

### Mutable Default Arguments

```python
# Violation — shared across all calls
def process(self, items: list = []) -> None:
    items.append(self._result)

# Correct
def process(self, items: list | None = None) -> None:
    if items is None:
        items = []
    items.append(self._result)

# Dataclass — use field()
from dataclasses import dataclass, field
@dataclass
class Config:
    tags: list[str] = field(default_factory=list)  # correct
    # tags: list[str] = []  # violation
```

### Resource Management

```python
# Violations — relying on GC
f = open(path, encoding="utf-8")
data = f.read()
# f never closed if exception occurs

conn = psycopg2.connect(dsn)
# conn never closed

# Correct — always context managers
with open(path, encoding="utf-8") as f:
    data = f.read()

with psycopg2.connect(dsn) as conn:
    with conn.cursor() as cursor:
        cursor.execute(query)

# Thread pools, subprocess, locks — same rule
with ThreadPoolExecutor(max_workers=4) as pool:
    futures = [pool.submit(task, item) for item in items]

with self._lock:  # never acquire manually
    self._shared_state[key] = value
```

### Thread Safety

```python
# Violation — GIL does not protect compound operations
class Counter:
    def __init__(self):
        self._count = 0

    def increment(self):
        self._count += 1  # read-modify-write — not atomic

# Correct
import threading

class Counter:
    def __init__(self):
        self._count = 0
        self._lock = threading.Lock()

    def increment(self):
        with self._lock:
            self._count += 1
```

### Numeric Precision

```python
# Violation — float comparison
if balance == 0.0: ...
if price * quantity == expected_total: ...

# Correct
import math
if math.isclose(balance, 0.0, abs_tol=1e-9): ...

# For financial values — use Decimal
from decimal import Decimal
price = Decimal("19.99")
quantity = Decimal("3")
total = price * quantity  # exact
```

---

## Architecture Bindings (Category 3)

### Dependency Inversion

```python
# Violation — hardcoded concrete class
class ReportGenerator:
    def __init__(self):
        self._storage = S3Storage()  # hardcoded

# Violation — receives config to extract one field
class ReportGenerator:
    def __init__(self, cfg: StorageConfig):
        self._bucket = cfg.bucket_name  # feature envy

# Correct
class ReportGenerator:
    def __init__(self, storage: StorageBackend, bucket_name: str) -> None:
        self._storage = storage
        self._bucket_name = bucket_name

    def __repr__(self) -> str:
        return f"ReportGenerator(storage={self._storage!r}, bucket={self._bucket_name!r})"
```

### Registry Pattern

```python
# Violation — registry doing construction
def get_storage(cfg):
    if cfg.storage == "local":
        return _registry["local"](cfg.path)
    return _registry[cfg.storage]()

# Correct — registry only resolves
_registry: dict[str, type] = {}

def resolve(name: str) -> type:
    if name not in _registry:
        raise PluginNotFoundError("storage", name, list(_registry.keys()))
    return _registry[name]

# Construction lives in a provider
def provide_storage(raw: dict) -> StorageBackend:
    name = raw.get("type", "local")
    cls = storage_registry.resolve(name)
    return cls.from_config(raw)
```

### Strategy / String `if/elif` Chains

```python
# Violation — grows with every new format
def export(data, format):
    if format == "csv":
        return CsvExporter(data).run()
    elif format == "json":
        return JsonExporter(data).run()
    elif format == "parquet":
        return ParquetExporter(data).run()

# Correct — closed to modification, open to extension
cls = exporter_registry.resolve(format)
return cls.from_config(cfg).run(data)
```

### Config Propagation

```python
# Violation — config travels past composition root
class Pipeline:
    def __init__(self, config: AppConfig):
        self._processor = Processor(config)  # config passed down

class Processor:
    def __init__(self, config: AppConfig):
        self._timeout = config.processor.timeout  # feature envy 2 levels deep

# Correct — extract at composition root
processor_timeout = config.processor.timeout
processor = Processor(timeout=processor_timeout)
pipeline = Pipeline(processor=processor)
```

### Resilience

```python
# Violation — no timeout, no retry
response = requests.get(url)
result = db.execute(query)

# Correct
import time

def _get_with_retry(url: str, retries: int = 3, backoff: float = 1.0) -> requests.Response:
    for attempt in range(retries):
        try:
            response = requests.get(url, timeout=10)
            response.raise_for_status()
            return response
        except (requests.Timeout, requests.ConnectionError) as e:
            if attempt == retries - 1:
                raise
            time.sleep(backoff * (2 ** attempt))
    raise RuntimeError("unreachable")
```

---

## Quality Bindings (Category 4)

### `__repr__`

```python
# Violation
class DatabaseClient:
    pass  # repr: <DatabaseClient object at 0x7f...>

# Correct
class DatabaseClient:
    def __init__(self, host: str, port: int, database: str) -> None:
        self._host = host
        self._port = port
        self._database = database

    def __repr__(self) -> str:
        return f"DatabaseClient(host={self._host!r}, port={self._port}, db={self._database!r})"
```

### Logging vs Print

```python
# Violations
print(f"Processing {item}")
print(f"Error: {e}")
sys.stdout.write(f"Result: {result}\n")

# Correct
import logging
logger = logging.getLogger(__name__)

logger.debug("Processing item: id=%s", item.id)
logger.error("Processing failed: %s", e, exc_info=True)  # captures traceback
```

### Encoding Explicitness

```python
# Violations
with open(path) as f: ...
with open(path, "w") as f: ...
json.dump(data, open(path, "w"))

# Correct
with open(path, encoding="utf-8") as f: ...
with open(path, "w", encoding="utf-8") as f: ...
with open(path, "w", encoding="utf-8") as f:
    json.dump(data, f, ensure_ascii=False)

# Also applies to:
logging.FileHandler(path, encoding="utf-8")
csv.writer(open(path, "w", encoding="utf-8", newline=""))
```

### Mirror Test Detection

```python
# Mirror test — duplicates implementation logic
def test_calculate_total():
    items = [Item(price=10), Item(price=20)]
    # mirrors implementation: sum(item.price for item in items)
    expected = sum(item.price for item in items)
    assert calculate_total(items) == expected

# Correct — tests behavior with known values
def test_calculate_total():
    items = [Item(price=10), Item(price=20)]
    assert calculate_total(items) == 30

def test_calculate_total_empty():
    assert calculate_total([]) == 0

def test_calculate_total_single():
    assert calculate_total([Item(price=7)]) == 7
```

### Exception Handling

```python
# Violation — broad catch, silent failure
try:
    result = self._client.post(url, data=payload)
    return result.json()["items"]
except Exception:
    return None

# Correct — narrow catch, logged failure
try:
    response = self._client.post(url, data=payload, timeout=self._timeout)
    response.raise_for_status()
    return response.json()["items"]
except requests.ConnectionError:
    logger.warning("Service unreachable at %s", url)
    return None
except requests.Timeout:
    logger.warning("Request timed out after %ss", self._timeout)
    return None
except requests.HTTPError as e:
    logger.error("HTTP error: %s", e)
    return None
except (KeyError, ValueError) as e:
    logger.error("Unexpected response format: %s", e)
    return None
```

---

## Python Security Tooling

Run before delivering any Python code:

| Tool | Command | Checks |
|------|---------|--------|
| bandit | `bandit -r src/ -ll` | Security issues across all categories |
| safety | `safety check` | Known CVEs in dependencies |
| pip-audit | `pip-audit` | Dependency vulnerability audit |
| semgrep | `semgrep --config=p/python` | Code pattern security rules |
| mypy | `mypy src/` | Type errors that hide logic bugs |
| ruff | `ruff check src/` | Lint including common bug patterns |

**Minimum acceptable bar before delivery:** bandit with no HIGH severity findings,
mypy with no errors on public interfaces.

---

## Common Python Hallucinations

These package names have appeared as AI hallucinations. Verify before using:

| Hallucinated name | Real package (if exists) |
|------------------|--------------------------|
| `python-dotenv-config` | `python-dotenv` |
| `flask-security` | `flask-security-too` (fork) |
| `requests-retry` | `urllib3.util.retry` (built-in) |
| `sqlalchemy-async` | `sqlalchemy[asyncio]` (extra) |
| `pydantic-settings-env` | `pydantic-settings` |
| `fastapi-auth` | no canonical package — use `python-jose`, `passlib` |
| `celery-redis` | `celery[redis]` (extra) |
| `pytest-async` | `pytest-asyncio` |
| `logging-structured` | `structlog` |

**Rule:** Any package you cannot find in a 5-second PyPI search is a red flag.
Do not suggest it. Look up the correct name or describe what the user needs to
install manually.

---

## Python Pre-Flight Checklist

Run in order before delivering any Python code.

**Phase 1 — Security:**
- [ ] No string concatenation in SQL queries — parameterized queries only
- [ ] No `eval()` or `exec()` on any variable (not just user input)
- [ ] No `subprocess` with `shell=True` and variable arguments
- [ ] No `requests.get(verify=False)` or equivalent TLS disable
- [ ] No credentials assigned as string literals
- [ ] No `hashlib.md5` or `hashlib.sha1` for password hashing
- [ ] No `CORS(app)` or `allow_origins=["*"]` on authenticated routes
- [ ] No path joins with user input without canonicalization check
- [ ] No logging of secret values, tokens, or passwords
- [ ] All routes/endpoints have explicit auth decorators or dependencies
- [ ] No rendering of user input as raw HTML (XSS)
- [ ] No raw user input in log statements (log injection)

**Phase 2 — Logic:**
- [ ] No list/string indexing without empty-collection guards
- [ ] No `.first()`, `.get()`, or dict access without None-check
- [ ] All comparisons use `is` for None/True/False, `==` for values
- [ ] No mutable default arguments (`def f(items=[])`)
- [ ] All `open()` calls inside `with` blocks
- [ ] All DB connections, thread pools, locks inside `with` blocks
- [ ] No module-level registry writes after startup without `threading.Lock()`
- [ ] No float equality comparison for values that matter

**Phase 3 — Architecture:**
- [ ] No class instantiates its own dependencies (except in composition root)
- [ ] No registry contains `if/elif` construction logic
- [ ] No config object passed more than one level past extraction point
- [ ] No `if format == "x": ... elif format == "y":` chains — use registry
- [ ] All external HTTP calls have `timeout=` set
- [ ] All retry logic uses exponential backoff

**Phase 4 — Quality:**
- [ ] Every injected class has a meaningful `__repr__`
- [ ] No `print()` for diagnostics — `logger = logging.getLogger(__name__)`
- [ ] All `open()` calls specify `encoding="utf-8"`
- [ ] All tests use known expected values, not implementation-derived values
- [ ] At least one error/edge-case test per public method
- [ ] No broad `except Exception` without documented justification
- [ ] No `assert` for runtime validation of external input

**Phase 5 — Triggers:**
- [ ] Scan output for: "for now", "temporary", "assumes valid input",
      "works in most cases", "simplified for brevity", "I verified"
- [ ] Any trigger phrase found → resolve or explicitly justify

---

## Python Pressure Scenarios

### "Just use pickle for caching"

**Risk:** `pickle.loads()` on untrusted data executes arbitrary code.
**Principled alternative:** Use `json` for primitive types; `msgpack` for binary
efficiency; explicitly typed serialization (Pydantic, dataclasses + json) for
structured data. If the data comes from your own system only, pickle is lower-risk
but still requires integrity verification (HMAC the payload).

---

### "Just use `shell=True`, it's simpler"

**Risk:** Any variable argument in a shell=True call is a command injection vector.
**Principled alternative:** Always use list arguments. The overhead is one extra
character per argument. There is no legitimate performance reason to use `shell=True`.

---

### "Just use `eval()` to parse the config expression"

**Risk:** `eval()` executes arbitrary Python. Config expressions from files or
environment variables are an attack surface.
**Principled alternative:** Use `ast.literal_eval()` for simple Python literals.
Use a config library (Pydantic, dataclasses) for structured config. For expression
evaluation, use `asteval` with an explicit allowlist.

---

### "Just disable verify=False for the internal service"

**Risk:** TLS verification disabled allows MITM attacks on internal traffic.
Internal services are a common pivot point in lateral movement.
**Principled alternative:** Install the internal CA certificate. Set
`REQUESTS_CA_BUNDLE` to the cert path. Use mutual TLS for service-to-service
auth.

---

### "Just use `except Exception: pass` to make it robust"

**Risk:** Silently swallows all errors including programming errors, creating
silent failure with no diagnostic information.
**Principled alternative:** Catch the specific exceptions you can handle. Log
at minimum. Let unknown exceptions propagate — a crash with a traceback is
better than silent corruption.

---

### "We don't need auth on the internal API, it's not public"

**Risk:** "Internal" is a perimeter assumption. Compromised service, misconfigured
network, or SSRF can reach internal APIs. Every endpoint that touches data needs
auth.
**Principled alternative:** Add auth. If speed is the constraint, use a shared
secret header checked at the application layer — one function, applied to all
internal routes.

---

## Python Scope Calibration

### One-Off Scripts (personal, single-use, no external input)

**Can relax:** `__repr__`, formal logging (print acceptable), full docstrings,
ABCs, registry pattern.

**Cannot relax:** Input validation if the script touches external data, explicit
encoding on file operations, resource cleanup (context managers still required),
no hardcoded credentials even in scripts.

---

### Prototypes (demo, throwaway, no production data)

**Can relax:** Full architecture patterns (DI, registry), `__repr__`, test coverage.

**Cannot relax:** Secrets management (use env vars even in demos), no `shell=True`
with variable input, no `eval()` on any input, explicit encoding.

**Required:** A prototype file or module must have a comment at the top:

```python
# PROTOTYPE — not for production use
# Missing: input validation, error handling, auth, logging, resilience
# Do not deploy without full review
```

---

### Internal Utilities (team-facing, controlled input, not customer-facing)

**Can relax:** Formal resilience patterns (retries/circuit breaker), full test
coverage on happy path.

**Cannot relax:** Auth if the tool accesses shared data, secrets management,
SQL parameterization, encoding explicitness.

---

### Shared Libraries (imported by other code, customer-facing indirectly)

**Cannot relax anything.** Full checklist applies. Libraries have no control over
how they are used. Input validation is mandatory. Side effects must be documented.
No `print()`. No hardcoded anything.

---

### Production Services (customer-facing, data-touching)

**Cannot relax anything.** All four checklist phases run. Bandit with no HIGH.
mypy with no errors on public interfaces. Auth on every endpoint. No TODO
deviations without removal triggers.

---

## Post-Generation Verification Statement (Python)

Before delivering any Python code:

> "Verification complete.
> Security scan: [pass / findings — named and resolved].
> Logic verification: [pass / issues — named and resolved].
> Architecture review: [pass / issues — named and resolved].
> Quality audit: [pass / issues — named and resolved].
> Self-correction triggers: [none / resolved].
> Behavioral checks: all dependencies verified against PyPI; no sycophantic
> agreement with incorrect premises."

If bandit or mypy were run, state the result. If they were not run (e.g., code
was reasoned about rather than executed), state that and note any patterns that
would be flagged.
