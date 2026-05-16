# Integration Testing Enrichment — Backend Services / APIs

Backend-specific tooling and approach for integration tests. Apply in Phases 3 and 4.

---

## Test Type: In-Process Integration Tests

For backend services, integration tests typically start the server in-process against a
real test database, then send real HTTP requests and assert on responses and side effects.

This is distinct from end-to-end testing (which uses a deployed environment). Integration
tests run locally or in CI with a controlled test database.

---

## Test Database

Use a real database instance — not an in-memory SQLite substitute for a Postgres service.
The test database must match the production database type.

**Options:**
- Local instance: start a real database server on a dedicated test port, run migrations, tear down after tests
- Test containers: `testcontainers` library starts a real database in Docker for the test run

**Database isolation between tests:**
- Wrap each test in a transaction that is rolled back after the test
- Or: truncate all tables in `@Before` and `@After`
- Never run integration tests against shared state — they will interfere with each other

---

## External API Stubs

External APIs (payment providers, email services, third-party data APIs) are stubbed.
Use a stub server that returns pre-configured responses:

- **WireMock**: starts a real HTTP server, responds to configured routes with stub responses
- **MockServer**: similar to WireMock
- **Recorded responses**: capture real API responses once, replay in tests

Configure the service under test to point to the stub server's URL in the test environment.
Use environment variables or test configuration to inject the stub URL.

---

## HTTP Integration Tests

Test the assembled service by sending real HTTP requests:

```python
# Example using httpx (Python)
def test_create_rule_stores_and_returns_id(client, db):
    response = client.post("/v1/rules", json={"trigger": "IN_VEHICLE", "action": "reply"})
    assert response.status_code == 201
    body = response.json()
    assert "id" in body
    assert db.query(Rule).filter_by(id=body["id"]).one()  # side effect verified
```

Test both the response body AND the side effects (database state, events emitted, messages
sent to stubs). A test that only checks the status code is not an integration test.

---

## Failure Path Testing

Every interaction contract has an `on_failure` field. Test it explicitly.

For each contract's failure path:
- Configure the stub or database to produce the failure condition
- Send the request that triggers the failing path
- Assert the behavior matches the contract's `on_failure` specification

Example patterns:
- Database unavailable: take the test database offline mid-request (connection pool exhausted)
- External API failure: configure WireMock to return a 500 or timeout
- Invalid data from upstream: send malformed payloads to the consuming endpoint

---

## Message Queue Integration

If components communicate via a message queue (Kafka, RabbitMQ, SQS):
- Use a real queue instance (test containers or local) — not an in-memory mock
- Publish messages and assert the consuming component processes them correctly
- Assert the side effects of processing (database writes, downstream events)
- Test queue failure: publish a malformed message, verify dead-letter routing or error handling

---

## Test Isolation

Each integration test must start from a clean state. Use one of:

```python
@pytest.fixture(autouse=True)
def clean_db(db_session):
    yield
    db_session.rollback()  # undo all writes from this test
```

Or:
```python
@pytest.fixture(autouse=True)
def clean_db(db):
    yield
    db.execute("TRUNCATE rules, schedules, log_entries RESTART IDENTITY CASCADE")
    db.commit()
```

Tests that share mutable state will produce order-dependent failures that are extremely
difficult to diagnose.

---

## MANUAL_REQUIRED Conditions on Backend

| Condition | Why it cannot be automated |
|---|---|
| Network partition between services | Simulating a real network split requires infrastructure control beyond test environment |
| Disk full / out of space | Cannot reliably reproduce in standard CI environment |
| External API rate limiting | Third-party rate limits require real timing and account state |
| Production traffic patterns | Load characteristics not reproducible in isolation |

For these, write explicit test procedures documenting: how to reproduce the condition manually,
what to observe, and the pass/fail criteria.
