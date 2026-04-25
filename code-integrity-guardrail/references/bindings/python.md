---
version: 1.2.2
skill: code-integrity-guardrail / python
---

# Python Guardrail Bindings

Python 3.10+ concrete additions to the universal code-integrity-guardrail.
This file only records what the universal skill cannot express without
language-specific syntax or tooling.

---

## Phase 0: Python Security Tooling

Run before delivering any Python code. State results in the post-generation
verification statement.

| Tool | Command | What it catches |
|---|---|---|
| bandit | `bandit -r src/ -ll` | Security issues across all categories |
| mypy | `mypy src/` | Type errors that hide logic bugs |
| pip-audit | `pip-audit` | Known CVEs in dependencies |
| ruff | `ruff check src/` | Lint and common bug patterns |

**Minimum bar:** bandit with no HIGH severity findings, mypy with no errors on
the files being delivered.

**When bash is unavailable:** State this. Cap all findings at FLAG or UNCERTAIN.
Note which bandit checks would apply to the generated patterns.

---

## Security Syntax (Category 1)

Only patterns where Python-specific syntax is required to detect the violation.

### SQL Injection

```python
# Violations — any driver, any ORM
cursor.execute(f"SELECT * FROM users WHERE id = {user_id}")
cursor.execute("SELECT * FROM users WHERE name = '%s'" % name)
cursor.execute("DELETE FROM logs WHERE date = " + date_str)
```

```python
# Correct
cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
stmt = select(users).where(users.c.id == bindparam("user_id"))
```

**Exception:** An f-string query is not a violation if every interpolated value
is a non-interpolated string literal defined in the same file (e.g., a table
name constant). Any variable interpolation disqualifies the exception.

**Bandit:** `B608`

---

### XSS

```python
# Violations
return Markup(f"<div>{user_input}</div>")          # Flask — bypasses escaping
return render_template_string(f"<p>{comment}</p>") # Flask — bypasses escaping
html = mark_safe(user_content)                     # Django — on unvalidated content
html = f"<h1>Welcome, {username}!</h1>"            # any raw HTML f-string
```

```python
# Correct
return render_template("profile.html", username=username)  # Jinja2 auto-escapes
safe_html = bleach.clean(user_html, tags=ALLOWED_TAGS)     # Django intentional HTML
safe = escape(user_input)                                   # markupsafe
```

**Bandit:** `B703, B704`

---

### Log Injection

```python
# Violations — newlines in user input enable log forgery
logger.info(f"User action: {user_input}")
logger.warning("Request from: " + request.headers.get("User-Agent"))
```

```python
# Correct
def sanitize_for_log(value: str) -> str:
    return value.replace("\n", "\\n").replace("\r", "\\r")

logger.info("User action: %s", sanitize_for_log(user_input))

# Or structured logging — serializes values safely
log.info("user_action", input_length=len(user_input), action_type=action_type)
```

---

### Command Injection

```python
# Violations
os.system(f"grep {pattern} {filename}")
subprocess.run(f"convert {input_file} {output_file}", shell=True)
subprocess.call("ls " + directory, shell=True)
```

```python
# Correct — list arguments, shell=False (default), always set timeout
subprocess.run(["grep", pattern, filename], capture_output=True, timeout=30)
subprocess.run(["convert", input_file, output_file], timeout=60)
```

**Exception:** `shell=True` is not a violation if the full shell string is a
string literal with no variable interpolation — e.g., `subprocess.run("ls -la", shell=True)`.
Any variable part disqualifies the exception.

**Bandit:** `B602, B603, B604, B605, B606, B607`

---

### Disabled TLS

```python
# Violations
requests.get(url, verify=False)
session.verify = False
ssl_context.check_hostname = False
ssl_context.verify_mode = ssl.CERT_NONE
connector = aiohttp.TCPConnector(ssl=False)
```

```python
# Correct — fix the cert, never disable verification
requests.get(url, verify="/path/to/ca-bundle.crt")
# Or set REQUESTS_CA_BUNDLE env var
```

**Bandit:** `B501, B502`

---

### Weak Cryptography

```python
# Violations
hashlib.md5(password.encode()).hexdigest()
hashlib.sha1(password.encode()).hexdigest()
hashlib.sha256(password.encode()).hexdigest()  # also wrong — no KDF, no salt
AES.new(key, AES.MODE_ECB)
```

```python
# Correct
import bcrypt
hashed = bcrypt.hashpw(password.encode(), bcrypt.gensalt(rounds=12))

from cryptography.hazmat.primitives.ciphers.aead import AESGCM
ciphertext = AESGCM(key).encrypt(nonce, data, aad)
```

**Bandit:** `B303, B304, B305, B306`

---

### Over-Permissive CORS

**Violation:** wildcard origin combined with credentials enabled.

```python
# Violations — wildcard + credentials
CORS(app, origins="*", supports_credentials=True)
app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_credentials=True)

# Django
CORS_ALLOW_ALL_ORIGINS = True
CORS_ALLOW_CREDENTIALS = True
```

```python
# Correct
CORS(app, origins=["https://app.example.com"])

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.example.com"],
    allow_credentials=True,
    allow_methods=["GET", "POST"],
    allow_headers=["Authorization", "Content-Type"],
)
```

**Note:** Wildcard alone (`allow_credentials=False` or absent) is lower risk
but should still be restricted to explicit origins on authenticated endpoints.

---

### Path Traversal

```python
# Looks safe but allows ../../etc/passwd
filepath = os.path.join(BASE_DIR, user_filename)
```

```python
# Correct
def safe_path(base: str, user_input: str) -> pathlib.Path:
    base_path = pathlib.Path(base).resolve()
    requested = (base_path / user_input).resolve()
    if not str(requested).startswith(str(base_path)):
        raise ValueError(f"Path traversal attempt: {user_input!r}")
    return requested
```

---

### Hardcoded Secrets

```python
# Violations
API_KEY = "sk-abc123"
requests.get(url, headers={"Authorization": "Bearer abc123"})
psycopg2.connect("postgresql://user:password@host/db")
```

```python
# Correct
api_key = os.environ["API_KEY"]         # raises if missing — intentional
api_key = os.environ.get("API_KEY", "") # silent fallback — only for optional values
```

**Bandit:** `B105, B106, B107`

---

## Logic Syntax (Category 2)

Only patterns where Python-specific syntax distinguishes the violation.

### Type Comparison

```python
# Violations
if x == None:       # use `is None`
if x == True:       # use `if x:` or `if x is True:`
if type(x) == int:  # use isinstance()

# Correct
if x is None: ...
if x: ...
if isinstance(x, int): ...
```

### Mutable Default Arguments

```python
# Violation — shared across all calls
def process(self, items: list = []) -> None: ...

# Correct
def process(self, items: list | None = None) -> None:
    if items is None:
        items = []

# Dataclass
@dataclass
class Config:
    tags: list[str] = field(default_factory=list)  # correct
    # tags: list[str] = []  # violation
```

### Numeric Precision

```python
# Violations
if balance == 0.0: ...
if price * quantity == expected_total: ...

# Correct
import math
if math.isclose(balance, 0.0, abs_tol=1e-9): ...

from decimal import Decimal
total = Decimal("19.99") * Decimal("3")  # exact
```

---

## Architecture Syntax (Category 3)

### Dependency Inversion

```python
# Violation
class ReportGenerator:
    def __init__(self):
        self._storage = S3Storage()  # hardcoded concrete

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

# Correct — registry only resolves
def resolve(name: str) -> type:
    if name not in _registry:
        raise PluginNotFoundError("storage", name, list(_registry.keys()))
    return _registry[name]

# Provider constructs
def provide_storage(raw: dict) -> StorageBackend:
    return storage_registry.resolve(raw["type"]).from_config(raw)
```

### Resilience

```python
# Violation
response = requests.get(url)

# Correct
def _get_with_retry(url: str, retries: int = 3, backoff: float = 1.0) -> requests.Response:
    for attempt in range(retries):
        try:
            response = requests.get(url, timeout=10)
            response.raise_for_status()
            return response
        except (requests.Timeout, requests.ConnectionError):
            if attempt == retries - 1:
                raise
            time.sleep(backoff * (2 ** attempt))
```

---

## Quality Syntax (Category 4)

### Logging vs Print

```python
# Violations (outside # SCRIPT-NOT-A-LIBRARY files)
print(f"Processing {item}")
print(f"Error: {e}")

# Correct
logger = logging.getLogger(__name__)
logger.debug("Processing item: id=%s", item.id)
logger.error("Processing failed", exc_info=True)
```

### Encoding Explicitness

```python
# Violations
with open(path) as f: ...
json.dump(data, open(path, "w"))

# Correct
with open(path, encoding="utf-8") as f: ...
with open(path, "w", encoding="utf-8") as f:
    json.dump(data, f, ensure_ascii=False)
logging.FileHandler(path, encoding="utf-8")
```

### Mirror Test Detection

```python
# Mirror test — computes expected from same logic as implementation
def test_calculate_total():
    items = [Item(price=10), Item(price=20)]
    expected = sum(item.price for item in items)  # mirrors impl
    assert calculate_total(items) == expected

# Correct — uses known values
def test_calculate_total():
    assert calculate_total([Item(price=10), Item(price=20)]) == 30

def test_calculate_total_empty():
    assert calculate_total([]) == 0
```

### Exception Handling

```python
# Violation — broad catch, silent failure
try:
    return self._client.post(url, data=payload).json()["items"]
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
except (KeyError, ValueError) as e:
    logger.error("Unexpected response format: %s", e)
    return None
```

---

## Common Python Hallucinations

| Hallucinated name | Correct package |
|---|---|
| `python-dotenv-config` | `python-dotenv` |
| `flask-security` | `flask-security-too` (fork) |
| `requests-retry` | `urllib3.util.retry` (built-in) |
| `sqlalchemy-async` | `sqlalchemy[asyncio]` (extra) |
| `pydantic-settings-env` | `pydantic-settings` |
| `fastapi-auth` | no canonical package — use `python-jose`, `passlib` |
| `celery-redis` | `celery[redis]` (extra) |
| `pytest-async` | `pytest-asyncio` |
| `logging-structured` | `structlog` |

Any package not findable in a 5-second PyPI search is a red flag. Do not suggest
it. Look up the correct name or describe what the user needs to install.

---

## Python Pre-Flight Checklist

Extends the universal phases. Items here are Python-specific.

**Phase 0 — Tools:**
- [ ] `bandit -r src/ -ll` — no HIGH findings (MEDIUM reviewed and documented)
- [ ] `mypy src/` — no errors on public interfaces
- [ ] `pip-audit` — no known CVEs in dependencies
- [ ] If tools unavailable — stated explicitly; findings capped at UNCERTAIN

**Phase 1 — Security (Python-specific):**
- [ ] No SQL f-strings or concatenation — parameterized queries only (exception documented if constant-only)
- [ ] No `eval()` or `exec()` on any variable
- [ ] No `subprocess` with `shell=True` and variable arguments (exception documented if literal-only)
- [ ] No `requests.get(verify=False)` or equivalent
- [ ] No credentials as string literals
- [ ] No `hashlib.md5/sha1` for password hashing
- [ ] No wildcard CORS with `allow_credentials=True`
- [ ] No path joins with user input without canonicalization
- [ ] No logging of secret values or tokens
- [ ] All routes have explicit auth decorators or dependencies
- [ ] No `Markup()`, `mark_safe()`, or `render_template_string()` with user input
- [ ] No raw user input in log statements

**Phase 2 — Logic (Python-specific):**
- [ ] No list/string indexing without empty-collection guards
- [ ] No `.first()`, `.get()`, or dict access without None-check
- [ ] `is` for None/True/False; `==` for values; `isinstance()` for types
- [ ] No mutable default arguments
- [ ] All `open()` calls inside `with` blocks
- [ ] All DB connections, locks, thread pools inside `with` blocks
- [ ] No module-level registry writes after startup without `threading.Lock()`
- [ ] No float equality for values that matter

**Phase 3 — Architecture (Python-specific):**
- [ ] No class instantiates its own dependencies (except composition root)
- [ ] No registry contains construction logic
- [ ] No config object passed more than one level past extraction point
- [ ] No `if format == "x": elif format == "y":` chains — use registry
- [ ] All external HTTP calls have `timeout=` set
- [ ] All retry logic uses exponential backoff

**Phase 4 — Quality (Python-specific):**
- [ ] Every injected class has meaningful `__repr__`
- [ ] No `print()` for diagnostics outside `# SCRIPT-NOT-A-LIBRARY` files
- [ ] All `open()` calls specify `encoding="utf-8"`
- [ ] Tests use known expected values, not implementation-derived values
- [ ] At least one error/edge-case test per public method
- [ ] No broad `except Exception` without documented justification
- [ ] No `assert` for runtime validation of external input

---

## Python Pressure Scenarios

### "Just use pickle for caching"

**Risk:** `pickle.loads()` on untrusted data executes arbitrary code.
**Alternative:** `json` for primitives; `msgpack` for binary efficiency;
Pydantic/dataclasses + json for structured data. If data comes from your own
system only, pickle is lower-risk but still requires integrity verification
(HMAC the payload).

---

### "Just use `shell=True`, it's simpler"

**Risk:** Any variable argument is a command injection vector.
**Alternative:** List arguments. One extra character per argument. No legitimate
performance reason to use `shell=True` with variable input.

---

### "Just use `eval()` to parse the config expression"

**Risk:** `eval()` executes arbitrary Python. Config from files or env vars is
an attack surface.
**Alternative:** `ast.literal_eval()` for simple Python literals. Pydantic for
structured config. `asteval` with an explicit allowlist for expression evaluation.

---

### "Just disable `verify=False` for the internal service"

**Risk:** Disabling TLS allows MITM on internal traffic. Internal services are
a common lateral movement pivot.
**Alternative:** Install the internal CA cert. Set `REQUESTS_CA_BUNDLE`. Use
mutual TLS for service-to-service auth.

---

### "Just use `except Exception: pass` to make it robust"

**Risk:** Silently swallows all errors including programming errors. Creates
silent failure with no diagnostic.
**Alternative:** Catch specific exceptions. Log at minimum. Let unknown
exceptions propagate — a crash with a traceback is better than silent corruption.

---

### "We don't need auth on the internal API, it's not public"

**Risk:** "Internal" is a perimeter assumption. Compromised service,
misconfigured network, or SSRF can reach internal APIs.
**Alternative:** Add auth. If speed is the constraint, a shared secret header
checked at the application layer is one function applied to all internal routes.
