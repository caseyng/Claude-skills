# Backend Enrichment Checklist

System-level concerns specific to backend services (Go, Python, Node.js, Rust, or any
process that handles requests, owns persistent state, or communicates over a network).
Load alongside `enrichment-universal.md` for any project that includes a backend component.

For each concern: identify the owner component, decide the policy, record it in
`technical_decisions` or `cross_cutting_concerns` as appropriate.

---

## 1. Database Selection and Ownership

**Questions to answer:**
- Which database engine? (SQLite, PostgreSQL, MySQL, Redis, etc.) Why this choice?
- Which component owns the database connection pool?
- Which component owns the schema and runs migrations?
- If multiple components need data from the same store: who owns the store, and what interface
  do others use to access it?

**Concern:** A database owned by two components will have competing migration strategies.
A connection pool not owned by anyone will be created ad hoc, resulting in connection leaks.

**Defaults to challenge:**
- SQLite: appropriate for embedded/local use, single writer. Multiple concurrent writers need PostgreSQL.
- Redis: appropriate for cache/pub-sub; wrong as a primary store unless data loss is acceptable.
- "The service connects directly to the DB" — fine for a single-service deployment; wrong if
  multiple services share a schema (introduces a coupling point that's invisible until both evolve).

---

## 2. Connection Management

**Questions to answer:**
- Is there a connection pool? Who owns and sizes it?
- What happens when the database is unreachable at startup? (fail-fast vs wait-and-retry)
- What happens when a query times out? (cancel vs wait indefinitely)
- Are connections returned to the pool on error? (connection leak prevention)

**Defaults to challenge:**
- "Unlimited connection pool" — the database has a finite connection limit; an unlimited pool
  lets the service exhaust it under load.
- "No connection timeout" — a hung database query hangs a goroutine/thread indefinitely.

---

## 3. API Design

**Questions to answer:**
- What protocol? (REST/JSON, gRPC/protobuf, GraphQL, raw TCP, message queue)
- Versioning strategy: how are breaking changes handled? (URL versioning `/v1/`, header, none)
- Pagination: cursor-based or offset-based?
- Authentication: where does it happen, and which component enforces it?
- Rate limiting: per-user, per-IP, or none? Which component enforces it?

**Decision record:** Record the protocol choice and versioning strategy in `technical_decisions`.
These decisions affect every consuming component — record them even if the answer is "REST,
no versioning for now."

---

## 4. Authentication and Authorisation Boundary

**Questions to answer:**
- Which component is the auth boundary? (the one that validates tokens / verifies credentials)
- Which component issues tokens?
- What does an unauthenticated request look like to internal components? (rejected at gateway,
  or does every component check independently?)
- Is there a service-to-service auth mechanism, or do internal services trust each other?

**Single ownership rule:** Exactly one component enforces authentication. Internal components
receive only authenticated requests. If two components check credentials independently, they
will diverge.

---

## 5. Background Processing

**Questions to answer:**
- Are there background jobs (scheduled tasks, queue workers, cleanup routines)?
- Who owns the job scheduler? (cron, message queue consumer, in-process goroutine)
- What happens if a job fails? (retry, dead-letter, alert)
- What happens if two instances run the same job simultaneously? (idempotent? lock required?)

**Deployment concern:** A job that assumes single-instance will fail silently in a
multi-instance deployment. Decide now whether the deployment is always single-instance,
or whether jobs must be designed for multi-instance safety.

---

## 6. Configuration Management

**Questions to answer:**
- Where does configuration come from? (environment variables, config file, secrets manager)
- Which component reads configuration, and how does it expose values to others?
- Which configuration values are secrets? (database password, API key)
- What happens if a required configuration value is absent at startup? (fail-fast vs default)

**Fail-fast on missing configuration:** A service that silently uses a default for a missing
required value will behave unexpectedly in production. Validate all required configuration at
startup and fail with a clear error.

---

## 7. Graceful Shutdown

**Questions to answer:**
- What is the shutdown sequence? (stop accepting requests → drain in-flight → close connections → exit)
- How long does the service wait for in-flight requests to complete before forcing exit?
- What happens to background jobs on shutdown? (cancel, checkpoint, complete)
- Which component owns the shutdown signal handler?

**Why this matters:** A service that exits immediately on SIGTERM will lose in-flight requests
and potentially corrupt partial writes. A service with no shutdown timeout will block deploys.

---

## 8. Health and Readiness

**Questions to answer:**
- Is there a health check endpoint? (`/healthz`, `/ready`, or equivalent)
- What does healthy mean? (process is running? database is reachable? dependencies are up?)
- What does ready mean? (accepting traffic? migrations complete? cache warm?)
- Who calls these endpoints? (load balancer, orchestrator, monitoring)

**Health vs Readiness:** Health = "the process is alive." Readiness = "the process can serve
traffic." They are different. A service that fails its own migration is alive but not ready.

---

## 9. Observability

**Questions to answer:**
- Structured logs: format (JSON), where emitted (stdout/stderr), who collects them
- Metrics: which metrics are emitted, what system collects them (Prometheus, CloudWatch, etc.)
- Distributed tracing: if multiple services exist, is there a trace propagation mechanism?
- Alerts: which failure conditions produce an alert, and who receives it?

**Minimum bar:** Structured logs to stdout. Every error log includes the error value and
enough context to identify the request or operation that failed. Log the request ID on every
log line for a given request.

---

## 10. External Service Dependencies

**Questions to answer:**
- Which external services does this backend call? (third-party APIs, databases, message queues)
- What happens when an external service is unavailable? (fail request, queue, degrade gracefully)
- Are there circuit breakers or retry budgets?
- Is there a timeout on every outbound call? (a missing timeout turns a slow dependency
  into a goroutine/thread leak)

**Record in `interaction_contracts`:** Every external service is an interaction contract with
an `on_failure` field. Do not leave this unspecified — the failure mode of an external dependency
is often the most important design decision for a backend service.

---

## 11. Data Retention and Purge

**Questions to answer:**
- Is there any data that ages out? (logs, events, sessions, cache entries)
- Who runs the purge? (application code, database TTL, cron job)
- Is purge configuration exposed to operators, or hardcoded?
- What is the impact of purge on running queries? (lock contention, performance impact)

---

## 12. Deployment Model

**Questions to answer:**
- Single instance or horizontally scalable?
- Stateless or stateful? (stateful services cannot scale horizontally without shared state)
- Containerised? (Docker, OCI) — affects how configuration, secrets, and volumes work
- Runs as a long-lived process or invoked per-request? (affects connection pooling, startup cost)

**Record in `technical_decisions`:** The deployment model affects almost every other decision
on this checklist. It belongs in `technical_decisions` even if the answer seems obvious.
