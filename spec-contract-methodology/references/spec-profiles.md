# Section Requirements by Component Type

Section requirements for each component shape. Most components span multiple shapes.
The union of Required and Conditional-that-applies entries across all applicable shapes
is the minimum complete spec for that component.

In the pipeline, the Stage 3 agent determines applicable shapes from the component
definition in the System Design Document — not from user input.

**Key:** R = Required. C = Conditional (required if stated property applies).
N = Not applicable (write one sentence declaring inapplicable and why).

| Section | Stateless Transform | Stateful Single-Caller | Stateful Shared Service | Streaming / Event-Driven | Extension Point | Security-Sensitive |
|---|---|---|---|---|---|---|
| §1 Purpose and Scope | R | R | R | R | R | R |
| §2 Concepts and Vocabulary | R | R | R | R | R | R |
| §3 System Boundary | R | R | R | R | R | R |
| §4 Data Contracts | C (structured I/O) | C (structured I/O) | C (structured I/O) | R | C (structured I/O) | R |
| §5 Component Contracts | R | R | R | R | R | R |
| §6 Lifecycle | N (declare stateless) | R | R | R | R | R |
| §7 Failure Taxonomy | R | R | R | R | R | R |
| §8 Boundary Conditions | R | R | R | R | R | R |
| §9 Sentinel Values | R | R | R | R | R | R |
| §10 Atomicity and State on Failure | N (declare stateless) | R | R | R | C (if stateful) | R |
| §11 Ordering and Sequencing | N | C (if pipeline) | C (if pipeline) | R | C (if pipeline) | R |
| §12 Interaction Contracts | C (if >1 component) | C (if >1 component) | R | R | R | R |
| §13 Concurrency and Re-entrancy | R | R | R | R | R | R |
| §14 External Dependencies | C (if any) | C (if any) | C (if any) | C (if any) | C (if any) | R |
| §15 Configuration | C (if configurable) | C (if configurable) | C (if configurable) | C (if configurable) | C (if configurable) | R |
| §16 Extension Contracts | N | N | N | N | R | C (if extensible) |
| §17 Error Propagation | R | R | R | R | R | R |
| §18 Observability Contract | C (if emits) | C (if emits) | C (if emits) | R | C (if emits) | R |
| §19 Security Properties | C (untrusted input) | C (security-relevant) | R | C (security-relevant) | R | R |
| §20 Versioning and Evolution | R | R | R | R | R | R |
| §21 What Is Not Specified | R | R | R | R | R | R |
| §22 Assumptions | R | R | R | R | R | R |
| §23 Performance Contracts | N | C (timeouts/ordering) | C (timeouts/ordering) | R | C (if bounded) | R |

**Note on Extension Point shape.** §16 is required and typically the most detailed
section. Stress-test Vector 5 is the most important to run for this shape.

**Note on Security-Sensitive shape.** Any system that handles untrusted input, makes
access control decisions, or sits on a security boundary should be treated as
security-sensitive regardless of primary shape. §19, §22 (caller trust assumptions),
and §14 (dependency availability on auth failure) are all required.
