# SYP-0001: Memory Admission Control

![draft](https://img.shields.io/badge/status-draft-yellow.svg?style=flat-square)

**Created:** 2025-12-24  
**Updated:** 2025-12-24  
**Version:** 0.1.0

## Abstract

This specification defines the atomic admission semantics governing state mutations in Synaptik-compliant systems. It establishes the suspend-evaluate-commit pattern that ensures governance constraints are enforced before state changes occur, preventing unsafe proposals from influencing system memory, embeddings, or causal history. Unlike post-hoc filtering approaches, Memory Admission Control prevents unsafe content from ever entering system memory or influencing downstream reasoning.

## Motivation

Traditional memory systems commit state changes immediately, allowing post-hoc validation to detect but not prevent policy violations. By the time a violation is detected, the unsafe content has already influenced embeddings, graph structures, and downstream reasoning.

Memory Admission Control inverts this: proposed state mutations are suspended pending governance evaluation, and only committed atomically upon approval. This pattern is analogous to transactional admission control in databases, adapted for cognitive state management. This ensures systems cannot "forget rules" because violations are prevented at the point of admission.

## Terminology

- **Proposal** — A candidate state mutation awaiting governance evaluation
- **Suspension** — The act of deferring state commitment pending evaluation
- **Evaluation** — The process of assessing a proposal against active governance rules
- **Admission** — The atomic commitment of an approved proposal to durable state
- **Refusal** — The rejection of a proposal without any state mutation
- **Atomic Commitment** — A state transition that either completes entirely or not at all, with no partial mutations

## Core Requirements

### 3.1 Suspension Semantics

A Synaptik-compliant system MUST suspend all state-mutating operations prior to commitment.

**Suspension Requirements:**

1. The system MUST NOT modify durable state before evaluation completes
2. The system MUST NOT update vector embeddings before evaluation completes
3. The system MUST NOT establish causal relationships before evaluation completes
4. The system MUST preserve the proposed state in a transient context accessible to evaluators

**Non-Normative Note:** Suspension does not require specific implementation mechanisms. Systems may use transactions, staging buffers, copy-on-write structures, or other patterns that satisfy the semantic requirements.

### 3.2 Evaluation Isolation

Evaluation MUST occur without side effects on system state.

**Isolation Requirements:**

1. Evaluators MUST NOT mutate the state being evaluated
2. Evaluators MUST operate on a consistent snapshot of governance rules
3. Evaluators MUST NOT be influenced by the suspended proposal (no speculative execution)
4. Evaluation results MUST be deterministic for identical proposals and governance contexts

**Non-Normative Note:** "Deterministic" means the same proposal evaluated under the same governance rules produces the same verdict. Non-determinism from timestamps, randomness, or external state is acceptable if explicitly declared in governance contracts.

### 3.3 Atomic Commitment

Upon approval, the system MUST commit the proposal atomically with associated decision metadata.

**Atomicity Requirements:**

1. If commitment succeeds, the proposal and its decision metadata MUST both be durable
2. If commitment fails, neither the proposal nor its metadata may be partially persisted
3. Concurrent proposals MUST NOT observe each other's uncommitted state
4. The committed state MUST reflect exactly the approved proposal without modification

**Decision Metadata:** The system MUST record at minimum:
- Decision identifier (unique per evaluation)
- Timestamp of decision
- Operation classification (read, write, or implementation-defined)
- Approval or refusal verdict

Additional metadata (actor, intent, cryptographic proofs) is RECOMMENDED but not required for baseline compliance.

### 3.4 Refusal Semantics

Upon refusal, the system MUST ensure no state mutation occurs.

**Refusal Requirements:**

1. Refused proposals MUST NOT modify durable storage
2. Refused proposals MUST NOT influence vector embeddings or similarity indices
3. Refused proposals MUST NOT establish causal links to other state
4. Refused proposals MUST NOT be observable to subsequent operations as part of system memory

**Audit Exception:** Systems MAY record refusal events in audit trails separate from operational state, provided:
- The audit trail is isolated from memory retrieval paths
- The audit trail does not influence embeddings or similarity search
- The audit trail is clearly marked as containing refused (not admitted) content

### 3.5 Fail-Closed Behavior

Evaluation errors or timeouts MUST be treated as refusals.

**Error Handling Requirements:**

1. If evaluation cannot complete within configured resource bounds, the system MUST refuse the proposal
2. If governance rules are unavailable or invalid, the system MUST refuse the proposal
3. If the evaluator crashes or raises an unhandled exception, the system MUST refuse the proposal
4. The system MUST NOT default to approval under any error condition

**Non-Normative Note:** This "fail-closed" design ensures safety under degraded conditions. Systems requiring high availability should implement fallback governance rules rather than disabling gating.

## Admission Flow Requirements

A conformant implementation MUST follow this logical sequence:

```
1. Receive proposal
2. Suspend state mutation
3. Invoke governance evaluation (see SYP-0004)
4. Upon approval:
   a. Atomically commit proposal and decision metadata
   b. Establish causal links (if SYP-0006 implemented)
   c. Return success to proposer
5. Upon refusal:
   a. Discard proposed state
   b. Record refusal in audit trail (if implemented)
   c. Return failure to proposer
6. Upon evaluation error:
   a. Treat as refusal (fail-closed)
   b. Log error for operator review
   c. Return failure to proposer
```

The system MAY optimize or reorder internal operations provided the observable semantics match this flow.

## Proposal Structure

Systems MUST support proposals containing at minimum:

- **Operation Type** — Classification of the mutation (e.g., write, update, delete)
- **Content** — The state to be admitted upon approval
- **Context** — Metadata describing intent, actor, or purpose (OPTIONAL)

Systems MAY extend proposals with additional fields (causal parents, timestamps, tags) as needed.

**Content Normalization:** Systems MAY normalize content (whitespace, encoding, canonicalization) before evaluation, provided:
- Normalization is deterministic and reversible
- The committed content matches the normalized form
- Normalization rules are documented

## Concurrency and Ordering

### 6.1 Concurrent Proposals

Systems MAY evaluate multiple proposals concurrently, provided:

1. Each proposal's evaluation is isolated (see Section 3.2)
2. Commitment order respects causal dependencies (if SYP-0006 implemented)
3. Concurrent commits maintain atomicity guarantees (see Section 3.3)

### 6.2 Ordering Guarantees

Systems MUST provide at minimum:

- **Per-proposal consistency** — A proposal's evaluation reflects a consistent governance snapshot
- **No retroactive invalidation** — Committed proposals cannot be "uncommitted" by later governance changes

Systems MAY provide stronger guarantees (serializability, linearizability) but these are not required for baseline compliance.

## Interaction with Other Specifications

- **SYP-0002 (Contract Evaluation):** Defines how governance rules are expressed and interpreted during evaluation
- **SYP-0003 (Audit Trail):** Defines requirements for recording decisions if audit trails are implemented
- **SYP-0004 (Policy Gate):** Defines the interface for invoking governance evaluation
- **SYP-0005 (Integrity & Verification):** Defines requirements for protecting governance rules from tampering
- **SYP-0006 (Causal Provenance):** Defines optional causal linking during atomic commitment

## Conformance

A system conforms to SYP-0001 if:

1. All state mutations are suspended pending evaluation
2. Evaluations are isolated and do not mutate state
3. Approved proposals are committed atomically with decision metadata
4. Refused proposals produce no state mutations
5. Evaluation errors trigger fail-closed refusal

**Testing Recommendations:**
- Verify proposals are not observable before commitment completes
- Verify refused proposals do not influence embeddings or retrieval
- Verify evaluation errors result in refusal, not approval
- Verify concurrent proposals maintain isolation and atomicity

## Security Considerations

### 9.1 Denial of Service

Malicious proposals designed to exhaust evaluation resources (CPU, memory, time) MUST trigger fail-closed refusal. Implementations SHOULD enforce resource bounds at the policy gate layer (see SYP-0004).

### 9.2 Time-of-Check-Time-of-Use (TOCTOU)

Implementations MUST ensure governance rules evaluated during suspension remain effective at commitment time. If rules change between evaluation and commitment, the system MUST either:
- Re-evaluate under current rules, OR
- Refuse the proposal and require re-submission

### 9.3 Information Leakage

Refused proposals MUST NOT leak into operational query results, but MAY appear in operator-facing audit logs. Implementations SHOULD ensure audit trails are access-controlled separately from application data.

## Normative References

- **RFC 2119** — Key words for use in RFCs to Indicate Requirement Levels
- **SYP-0000** — Synaptik Protocol Introduction
- **SYP-0004** — Policy Gate API (normative for evaluation invocation)
- **SYP-0005** — Integrity & Verification (normative for governance protection)

## Informative References

- **SYP-0002** — Contract Definition Language
- **SYP-0003** — Audit Trail Format
- **SYP-0006** — Causal Provenance & DAG

---

## Appendix A: Example Admission Sequence (Non-Normative)

```
Proposer → Memory System: Propose(content="...", intent="...")
Memory System: Suspend write operation
Memory System → Policy Gate: Evaluate(proposal)
Policy Gate → Contract Evaluator: Check constraints
Contract Evaluator → Policy Gate: Verdict = APPROVE
Policy Gate → Memory System: Decision(approved, proof={...})
Memory System: Atomic commit [content + decision metadata]
Memory System → Proposer: Success(memory_id="...")
```

## Appendix B: Rationale for Fail-Closed Default (Non-Normative)

Traditional systems often fail-open (default to permitting) to maintain availability. Synaptik inverts this because:

1. **Safety-critical contexts** — Leaked PII or unsafe actions are more costly than service degradation
2. **Explicit governance** — If rules are unavailable, the system cannot prove safety
3. **Defense in depth** — Fail-closed ensures partial deployment doesn't create vulnerabilities

Systems requiring high availability should deploy redundant governance infrastructure rather than disabling safety checks.
