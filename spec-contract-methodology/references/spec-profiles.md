# Appendix B — Spec Profile Table, Deployment Guidance, Future Directions

---

## Appendix B — Spec Profile Table

Before writing, identify which system shapes apply. Most systems span multiple shapes.
The union of required sections across all applicable shapes is the minimum complete spec.

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

---

## Deployment Guidance

The spec declares what the system must do. This section captures what operators must
do to keep the system working correctly in production over time.

Deployment guidance is not a correctness contract. Violating it does not make the spec
wrong. It exists to bridge the gap between "deployed correctly at launch" and "remains
effective over time."

### When to add

Add when any of the following apply:
- System has static defences (filters, thresholds, classifiers) that degrade against an active adversary over time
- System produces a forensic or audit record whose retention, access control, or availability has security implications
- System depends on external components (models, services, files) whose currency or availability affects security posture, not just availability
- Configuration changes are security-relevant — a misconfigured deployment silently weakens protection without producing an error

If none apply, the section is optional.

### What belongs here vs. in the spec

**Belongs in the spec (correctness contract):** behaviour that, if violated, produces
a wrong result or broken guarantee visible to callers or auditors.
Example: "The audit backend MUST NOT raise in `log()`"

**Belongs in Deployment Guidance (operational guidance):** behaviour that, if neglected,
degrades effectiveness or increases risk over time without breaking a stated contract.
Example: "Audit logs SHOULD be retained for a period sufficient for incident investigation"

When the boundary is unclear: does violating this produce a wrong result *now*, or a
worse outcome *later*? Now → spec. Later → Deployment Guidance.

### Structure

Named subsections, one concern per subsection. Each states:
- What the concern is and why it matters
- What operators MUST or SHOULD do about it
- What the consequence of neglect is, stated concretely

Reference Deployment Guidance from relevant spec sections (e.g. from §19 when noting
that static defences degrade). Do not embed operational guidance inline in spec sections.

---

## Future Directions

Add a Future Directions section (§24 or equivalent, after §23) when the system has
known, intentional extension paths that current architectural decisions are designed
to accommodate.

Not a contract. Nothing in it is a current commitment. It exists to:
- Prevent future designers from treating current architectural limits as permanent
- Document invariants that future extensions MUST preserve
- Ensure current design decisions are understood in the context of what they accommodate

For each known extension direction, a named subsection containing:
- What the extension achieves — one paragraph, no implementation detail
- What constraints it operates under — invariants from the current spec that MUST be preserved
- What design questions remain open

Future Directions MUST NOT describe implementation approaches.

When an anticipated extension is built:
1. Update relevant spec sections (§16 for new extension contracts, §7 for new failure modes, §17 for propagation paths, §20 for stability levels)
2. Mark the Future Directions entry as implemented, or remove it
3. Re-run the stress-test pass (Appendix A) against the updated spec

A Future Directions section never updated becomes fiction. Treat it as a backlog, not an archive.

---

## References

**Design by Contract (Meyer, 1992)**
Origin of preconditions, postconditions, and invariants as a formal specification method.
The component contract format in §5 is a prose adaptation of DbC.
*Bertrand Meyer, "Applying 'Design by Contract'", IEEE Computer, 1992.*

**RFC 2119 — Key words for use in RFCs to indicate requirement levels**
Defines MUST, SHOULD, MAY, MUST NOT, SHOULD NOT precisely.
*https://www.rfc-editor.org/rfc/rfc2119*

**ISO/IEC/IEEE 29148 — Systems and software engineering: Requirements engineering**
The right reference when the system will be built by multiple independent teams or must
satisfy regulatory requirements.
*ISO/IEC/IEEE 29148:2018*

**The C4 Model (Brown)**
Lightweight diagramming for system context, containers, components, and code.
Useful for §3 and §5 when prose alone is insufficient.
*https://c4model.com*

**TLA+ / Alloy**
Formal specification languages for systems where ordering, sequencing, and concurrency
constraints cannot be captured unambiguously in prose. If §11 or §13 cannot be made
falsifiable in prose, consider a formal model.
*TLA+: https://lamport.azurewebsites.net/tla/tla.html*
*Alloy: https://alloytools.org*

**Software Engineering at Google (Winters, Manshreck, Wright — O'Reilly, 2020)**
Practical guidance on how specifications fit into an engineering workflow.
