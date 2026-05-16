# Backend Service / API Enrichment Checklist

Platform-specific concerns for backend services and APIs. Apply after `enrichment-universal.md`.

---

## API Design

**API style**
REST, GraphQL, gRPC, or event-driven? This is an early architectural decision.
Default: REST for most CRUD-heavy services. gRPC for high-performance internal services
or when strong typing across service boundaries matters. GraphQL when clients need
flexible querying of a complex data graph.

**Versioning strategy**
How are breaking API changes handled?
Default: version in the URL path (`/v1/`, `/v2/`) for REST. Declare a deprecation policy
before the first external consumer is added.

**API documentation**
Is API documentation required? Who consumes this API?
Default: if the API has external consumers, OpenAPI/Swagger documentation is required.
Internal-only APIs need at minimum a contract document.

---

## Authentication and Authorisation

**Client authentication**
How do API callers authenticate? API keys, OAuth tokens, mutual TLS, JWT?
Default: if external clients call this API, API keys or OAuth with short-lived tokens.
Never long-lived tokens without a revocation mechanism.

**Authorisation model**
What can each caller do? Is it the same for all callers or role-based?
Default: even simple services need an explicit authorisation model. "Anyone with an API key
can do anything" is a model — state it explicitly.

---

## Reliability and Scalability

**Expected load and scaling requirements**
What request volume must the service handle? Is load variable or steady?
Default: state the expected order of magnitude. A service handling 10 requests/day has
different design constraints than one handling 10,000/second.

**Stateless vs stateful**
Does the service hold state between requests?
Default: stateless services scale horizontally. If the service must hold state, identify
the state store and its consistency requirements.

**Failure handling for callers**
What should callers do when this service is unavailable?
Default: define the retry policy. Services must document whether endpoints are idempotent
(safe to retry) or not.

---

## Data

**Database selection**
Relational, document, key-value, time-series? What consistency guarantees are required?
Default: relational database for most services. Switch only when the data model genuinely
does not fit the relational model, not for novelty.

**Schema migrations**
How are database schema changes deployed? Can migrations run without downtime?
Default: migrations must be backward-compatible with the prior version of the service
for the duration of a deployment window (blue-green or rolling). Destructive migrations
are a deployment risk — plan them explicitly.

**Data residency and compliance**
Does data need to stay in a specific region? Are there compliance requirements (GDPR, HIPAA)?
Default: state the data residency requirements early — they constrain cloud provider and
region selection, and affect the architecture significantly.

---

## Operational Concerns

**Health checks and readiness**
Does the service expose health and readiness endpoints?
Default: yes. A health endpoint is required for load balancers and orchestrators.
Readiness signals when the service is ready to accept traffic after startup.

**Structured logging**
What format, what fields, where do logs go?
Default: structured JSON logs with at minimum: timestamp, level, service name, trace ID,
message. Logs go to a centralised store queryable by the team.

**Rate limiting**
Is rate limiting required to protect the service from abuse?
Default: any externally accessible API should have rate limiting. Define limits per
authenticated caller, not globally.

**Distributed tracing**
Is request tracing across service boundaries needed?
Default: if this service calls other services or is called by other services, trace IDs
must be propagated. Required for debugging latency issues in multi-service architectures.

---

## Deployment

**Containerisation**
Is the service containerised? Is orchestration (Kubernetes, ECS) in scope?
Default: state the deployment mechanism early. It affects configuration management,
secret injection, and scaling strategy.

**Secret management**
How are credentials (database passwords, API keys) injected at runtime?
Default: secrets from a secrets manager (Vault, AWS Secrets Manager, etc.) or environment
variables injected at deploy time. Never hardcoded, never in source control.
