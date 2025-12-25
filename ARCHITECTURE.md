# Synaptik Protocol Architecture

The Synaptik Protocol defines a governance-first memory substrate in which constraints are enforced **at admission time**, not retroactively at output. This document provides a **non-normative reference model** that explains how the protocol’s guarantees compose, without prescribing a specific implementation.

The authoritative requirements of the protocol are defined in the individual SYP specifications. This document exists to orient readers and implementers.

## Protocol Guarantees (Overview)

A system is considered Synaptik-aligned if it satisfies the following governance guarantees:

1. **Governed State Admission** — State mutations are evaluated against active governance rules before commitment.
2. **Declarative Governance Rules** — Constraints are expressed independently of application logic.
3. **Cryptographic Governance Integrity** — Governance rules cannot be silently modified or bypassed.
4. **Tamper‑Evident Decision Records** — Governance decisions are durably auditable.
5. **Causal Traceability (Recommended)** — State transitions preserve lineage for replay and analysis.

These guarantees are normatively defined by the SYP specifications referenced throughout this document.

---

## Reference Model (Non‑Normative)

The following layered model illustrates one way the protocol guarantees may be composed. Alternative architectures are permitted so long as the required semantics are preserved.

```
+------------------------------------------+
|         Application Layer                |
|  (agents, services, interfaces)          |
+------------------------------------------+
|        Policy Gate (SYP-0004)            |
|  Pre-execution evaluation & gating       |
+------------------------------------------+
|   Memory Admission Control (SYP-0001)    |
|  Suspend → Evaluate → Commit             |
+------------------------------------------+
|   Contract Evaluation (SYP-0002)         |
|  Declarative governance rules            |
+------------------------------------------+
|    Integrity Layer (SYP-0005)            |
|  Governance identity & verification      |
+------------------------------------------+
|   Audit Trail (SYP-0003)                 |
|  Tamper-evident decision records         |
+------------------------------------------+
|   Causal Provenance (SYP-0006)           |
|  Lineage & replay semantics              |
+------------------------------------------+
|         Storage Substrate                |
|  (implementation-defined)                |
+------------------------------------------+
```

This stack is **illustrative**, not prescriptive.

---

## Reference Responsibilities (Informative)

### Policy Gate — *SYP‑0004*

Intercepts proposed state‑mutating operations prior to execution and coordinates governance evaluation.

**Semantic responsibilities:**

* Ensure all proposals are evaluated before execution
* Enforce fail‑closed behavior on error or timeout
* Bind decisions to verifiable decision records

---

### Memory Admission Control — *SYP‑0001*

Defines the atomic admission semantics governing state mutation.

**Semantic guarantees:**

* Evaluation occurs without mutating state
* Rejected proposals produce no side effects
* Approved proposals commit atomically with decision metadata

---

### Contract Evaluation — *SYP‑0002*

Evaluates proposals against declarative governance rules.

**Semantic capabilities:**

* Constraint evaluation independent of application logic
* Risk assessment and intervention signaling
* Composable rule evaluation

---

### Integrity Layer — *SYP‑0005*

Ensures governance rules are verifiable and protected against unauthorized modification.

**Semantic guarantees:**

* Governance identity is verifiable
* Invalid or altered governance artifacts cause failure
* Enforcement cannot be silently bypassed

---

### Audit Trail — *SYP‑0003*

Provides durable, tamper‑evident records of governance decisions.

**Semantic properties:**

* Append‑only decision recording
* Cryptographic authentication
* Causal linkage to governed operations

---

### Causal Provenance — *SYP‑0006* (Recommended)

Tracks lineage between state transitions to support replay, retention, and analysis.

**Semantic properties:**

* Parent‑child relationships between operations
* Dependency‑aware traversal and pruning
* Replay‑compatible ordering guarantees

---

## Example Admission Flow (Non‑Normative)

```
1. Proposal submitted to Policy Gate
2. Operation suspended
3. Governance rules evaluated
4. Decision produced (approve / refuse)
5. If approved:
   - Atomic commit
   - Decision recorded
   - Provenance linked
6. If refused:
   - No state mutation
   - Refusal recorded
```

This flow illustrates **one possible execution model**. Implementations MAY vary so long as admission‑time enforcement and auditability are preserved.

---

## Compliance Summary (Normative)

A system is **Synaptik‑compliant** if it implements:

* **SYP‑0001** — Governed admission semantics
* **SYP‑0002** — Declarative contract format
* **SYP‑0003** — Tamper‑evident audit records
* **SYP‑0004** — Pre‑execution policy gating
* **SYP‑0005** — Governance integrity verification

**SYP‑0006** (Causal Provenance) is RECOMMENDED but not required for baseline compliance.

---

## Implementation Freedom (Informative)

The protocol intentionally avoids mandating specific:

* algorithms
* cryptographic primitives
* storage backends
* execution environments

Implementations are free to choose appropriate mechanisms provided the **protocol semantics** are preserved: admission‑time enforcement, verifiable governance integrity, and auditable decisions.
