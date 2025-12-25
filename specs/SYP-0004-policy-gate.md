# SYP-0004: Policy Gate API

![draft](https://img.shields.io/badge/status-draft-yellow.svg?style=flat-square)

**Created:** 2025-12-24  
**Updated:** 2025-12-24  
**Version:** 0.1.0

## Abstract

This specification defines the Policy Gate interface for pre-execution governance evaluation in Synaptik-compliant systems. The Policy Gate intercepts proposed operations before they execute, coordinates evaluation against governance rules, and returns verifiable decision records. This layer enforces the fail-closed, admission-time gating semantics required by SYP-0001.

## Motivation

Without a standardized gating interface, governance enforcement becomes inconsistent: some operations might be gated while others bypass evaluation, or different subsystems might implement incompatible evaluation patterns. The Policy Gate API establishes a uniform contract for pre-execution governance across all state-mutating operations.

This specification focuses on the *interface* and *behavioral contracts* required for interoperability, not the internal implementation of evaluation logic (see SYP-0002 for contract evaluation semantics).

## Terminology

- **Policy Gate** — The component that intercepts operations and coordinates governance evaluation
- **Policy Request** — A structured description of an operation requiring evaluation
- **Decision Record** — The verifiable output of a governance evaluation
- **Evaluation Context** — The snapshot of governance rules and system state used during evaluation
- **Resource Bounds** — Limits on computation time, memory, and operations to prevent denial-of-service

## Core Interface Requirements

### 3.1 Evaluation Method

All policy gates MUST implement an evaluation method with the following semantics:

**Input:** A policy request describing the operation to evaluate  
**Output:** A decision record indicating approval or refusal  
**Behavior:** MUST NOT mutate system state; MUST complete within resource bounds; MUST fail-closed on error

**Signature (Conceptual):**
```
evaluate(request: PolicyRequest) → Result<DecisionRecord, Error>
```

The concrete signature is implementation-defined, but MUST satisfy these behavioral requirements.

### 3.2 Policy Request Structure

A policy request MUST contain at minimum:

1. **Operation Classification** — The type of operation being requested (e.g., read, write, delete)
2. **Operation Intent** — A description of what the operation aims to accomplish
3. **Operation Context** — Metadata about who is performing the operation and why (OPTIONAL)

**Request Identity:** Policy requests SHOULD include a deterministic request identifier or fingerprint:
- Systems MAY provide an explicit `request_id` field
- OR systems MUST compute a deterministic `request_fingerprint` (hash of canonicalized request fields)
- The fingerprint enables correlation across retries, caching, and audit trails
- Decision records SHOULD echo this identifier for traceability

**Rationale:** Request fingerprints enable proving that retried operations are identical, provide stable keys for decision caching, and allow third-party audit correlation without ambiguity.

**Operation Classification:** Systems MUST support at minimum:
- **Read** — Operations that retrieve existing state without modification
- **Write** — Operations that create or modify state

Systems MAY define additional classifications (update, delete, promote, recall) as needed.

**Operation Intent:** A description of the operation's purpose that enables intent-based governance rules.

**Intent Structure (RECOMMENDED):**
- **Intent Label** — Machine-readable tag or structured identifier (e.g., "store_conversation", "production_write")
- **Intent Detail** — Optional human-readable explanation for operators and auditors

Systems MAY accept freeform intent strings, but SHOULD structure them to avoid natural language matching ambiguities and intent spoofing vulnerabilities. Governance contracts that match on intent SHOULD use intent labels rather than parsing natural language.

**Operation Context (Optional Fields):**
- **Actor** — Identity of the entity requesting the operation
- **Purpose** — Machine-readable purpose tag (training, replay, production, etc.)
- **Scope** — Logical namespace or partition (e.g., "lobe" in memory systems)
- **Target** — Specific resource identifier being operated upon
- **Causal Parents** — References to prior operations this operation depends on (see SYP-0006)

### 3.3 Decision Record Structure

A decision record MUST contain at minimum:

1. **Decision Identifier** — A unique identifier for this evaluation
2. **Verdict** — Approval or refusal
3. **Timestamp** — When the decision was issued
4. **Operation Classification** — Echo of the operation type from the request

Decision records SHOULD include:
- **Request Fingerprint** — Echo of the request identifier or canonical hash for correlation
- **Intent** — Echo of the operation intent for auditability
- **Context** — Echo of actor, purpose, or scope for provenance

Decision records MAY include:
- **Cryptographic Proof** — Integrity metadata (see SYP-0005)
- **Justification** — Human-readable explanation of the verdict
- **Constraints** — Conditions or limitations applied to the approved operation
- **Risk Assessment** — Computed risk score or classification

**Verdict Values:** At minimum:
- **APPROVE** — Operation is permitted and should proceed
- **REFUSE** — Operation is denied and must not execute

Systems MAY extend with additional verdicts (e.g., CONSTRAIN, QUARANTINE) provided they map to either approval-with-conditions or refusal semantics.

### 3.4 Resource Bounds

Policy gates MUST enforce resource limits during evaluation to prevent denial-of-service attacks or unbounded processing.

**Required Bounds:**

1. **Time Limit** — Evaluation MUST timeout after a maximum duration
2. **Memory Limit** — Evaluation MUST NOT consume unbounded memory
3. **Operation Limit** — Evaluation MUST NOT perform unbounded computation (measured as: predicate evaluation steps, AST nodes traversed, interpreter operations, or equivalent evaluator-specific metric)

Upon exceeding any bound, the gate MUST:
- Terminate evaluation immediately
- Return a REFUSE verdict (fail-closed)
- Log the timeout for operator review (RECOMMENDED)

**Configuration:** The specific bound values are implementation-defined and SHOULD be configurable per deployment context. Reasonable defaults might be:
- Time: 100-1000ms
- Memory: 10-100MB
- Operations: 10,000-100,000 evaluator steps (not CPU instructions)

**Non-Normative Note:** The goal is preventing catastrophic failures (hanging, OOM), not achieving specific performance targets. Tighter bounds improve responsiveness but may limit governance expressiveness. "Operations" should be measured in terms meaningful to the contract evaluator (e.g., rule evaluations, pattern matches, predicate checks) rather than low-level CPU instructions.

### 3.5 Fail-Closed Semantics

Policy gates MUST implement fail-closed error handling.

**Error Conditions:**

1. **Evaluation Timeout** → REFUSE
2. **Resource Exhaustion** → REFUSE
3. **Contract Unavailable** → REFUSE
4. **Contract Invalid** → REFUSE
5. **Evaluator Crash** → REFUSE
6. **Unknown Error** → REFUSE

The gate MUST NOT:
- Default to approval under any error condition
- Bypass evaluation due to performance concerns
- Cache stale approvals beyond their validity period

**Exception:** Systems MAY implement explicit "emergency bypass" modes for operational recovery, provided:
- Bypass is triggered by explicit operator action, not automatic
- Bypass events are logged to audit trails with heightened priority
- Bypass does not persist across system restarts

### 3.6 Evaluation Isolation

Evaluation MUST NOT mutate system state.

**Isolation Requirements:**

1. The gate MUST NOT commit the proposed operation during evaluation
2. The gate MUST NOT update embeddings or indices during evaluation
3. The gate MUST operate on a stable snapshot of governance rules
4. The gate MUST NOT be influenced by speculative execution of the proposal

**Read-Only Access:** Evaluators MAY read existing system state (to check consistency, detect duplicates, etc.) provided reads do not trigger side effects.

## Evaluation Context Requirements

### 4.1 Governance Snapshot Stability

Policy gates MUST evaluate each request against a consistent snapshot of governance rules.

**Consistency Requirements:**

1. A single request MUST be evaluated under a single version of governance contracts
2. If contracts change during evaluation, the gate MUST either complete with the original version OR abort and retry with the new version
3. The gate MUST NOT mix predicates from multiple contract versions in a single evaluation

**Non-Normative Note:** This prevents TOCTOU vulnerabilities where rules change mid-evaluation, leading to inconsistent verdicts.

### 4.2 Contract Resolution

Policy gates MUST resolve which governance contracts apply to a given request.

**Resolution Requirements:**

1. The system MUST have a deterministic method for selecting applicable contracts
2. If no contracts are available, the gate MUST refuse the request (fail-closed)
3. If multiple contracts apply, the system MUST define a conflict resolution policy (e.g., most restrictive, explicit priority)

Systems MAY support:
- Global contracts (apply to all operations)
- Scoped contracts (apply to specific intents, actors, or namespaces)
- Hierarchical contracts (inheritance or override semantics)

## Behavioral Requirements

### 5.1 Idempotency

Evaluating the same request multiple times under the same governance context SHOULD produce the same verdict.

**Exceptions Allowed:**
- Non-deterministic contracts (explicitly using randomness or time-based rules)
- Rate-limiting contracts (where evaluation itself constitutes resource consumption)

**Rationale:** Idempotency enables safe retries and simplifies debugging.

### 5.2 Concurrency

Policy gates MAY evaluate multiple requests concurrently, provided:

1. Each evaluation is isolated (see Section 3.6)
2. Resource bounds are enforced per-evaluation, not globally
3. Concurrent evaluations do not deadlock or starve

**Non-Normative Note:** Concurrency is not required but improves throughput for systems processing high request volumes.

### 5.3 Auditability

Policy gates SHOULD emit decision records to audit trails (see SYP-0003).

**Audit Requirements:**

1. Every evaluation SHOULD produce a durable audit record
2. Audit records SHOULD include sufficient context to replay the evaluation
3. Audit records MUST NOT be mutable or deletable by application code

**Partial Compliance:** Systems without audit trails MAY still conform to SYP-0004 provided they meet the evaluation interface requirements. Full compliance with the Synaptik Protocol requires SYP-0003.

## Integration with Other Specifications

- **SYP-0001 (Memory Admission Control):** Policy gates are invoked during the suspension phase before atomic commitment
- **SYP-0002 (Contract Evaluation):** Defines how governance rules are evaluated; policy gates coordinate this process
- **SYP-0003 (Audit Trail):** Decision records emitted by policy gates are persisted to audit trails
- **SYP-0005 (Integrity & Verification):** Governance contracts used by policy gates must be integrity-checked before use
- **SYP-0006 (Causal Provenance):** Causal parent references in policy requests enable lineage-aware governance

## Conformance

A system conforms to SYP-0004 if:

1. All state-mutating operations pass through a policy gate (state-mutating includes: durable memory writes, embeddings/similarity indices, causal provenance links, and any derived retrieval structures)
2. The gate evaluates requests against governance contracts before execution
3. The gate enforces resource bounds and fails closed on errors
4. The gate produces decision records containing required metadata
5. The gate does not mutate state during evaluation

**Testing Recommendations:**
- Verify operations are intercepted before execution
- Verify evaluation timeouts trigger refusals
- Verify contract unavailability triggers refusals
- Verify concurrent evaluations maintain isolation
- Verify decision records contain required fields

## Security Considerations

### 8.1 Bypass Prevention

Implementations MUST ensure the policy gate cannot be bypassed:

1. All code paths that mutate state MUST invoke the gate
2. The gate MUST NOT be disabled by application code without explicit operator action
3. Direct database access SHOULD be restricted to prevent sidestepping the gate

**Defense in Depth:** Combining policy gates with database-level access controls provides multiple enforcement layers.

### 8.2 Evaluation Sandboxing

Contract evaluation SHOULD occur in a restricted execution environment:

1. Contracts SHOULD NOT have direct filesystem or network access
2. Contracts SHOULD NOT be able to invoke arbitrary system calls
3. Contracts SHOULD operate under least-privilege principles

**Non-Normative Note:** WASM sandboxes, language-level interpreters, or OS-level isolation (containers, seccomp) are all viable sandboxing approaches.

### 8.3 Resource Exhaustion Attacks

Malicious contracts or requests designed to consume excessive resources are mitigated by:

1. Time bounds (prevent infinite loops)
2. Memory bounds (prevent memory bombs)
3. Operation bounds (prevent combinatorial explosions)

Implementations SHOULD log resource limit violations for anomaly detection.

### 8.4 Side-Channel Leakage

Evaluation timing or resource consumption MAY reveal information about refused proposals. Implementations concerned with this SHOULD:
- Use constant-time evaluation where possible
- Normalize error messages to avoid distinguishing error types
- Rate-limit evaluation attempts per actor

## Normative References

- **RFC 2119** — Key words for use in RFCs to Indicate Requirement Levels
- **SYP-0000** — Synaptik Protocol Introduction
- **SYP-0001** — Memory Admission Control (normative for integration semantics)
- **SYP-0005** — Integrity & Verification (normative for contract integrity)

## Informative References

- **SYP-0002** — Contract Definition Language
- **SYP-0003** — Audit Trail Format
- **SYP-0006** — Causal Provenance & DAG

---

## Appendix A: Example Evaluation Flow (Non-Normative)

```
1. Application requests write operation
2. Memory system constructs PolicyRequest:
   {
     operation: WRITE,
     intent: "store conversation summary",
     actor: "user@example.com",
     purpose: "production",
     scope: "conversations",
     target: "conv-12345"
   }
3. Memory system invokes PolicyGate.evaluate(request)
4. Policy gate loads applicable contracts
5. Policy gate enforces resource bounds during evaluation
6. Contract evaluator checks constraints (see SYP-0002)
7. Policy gate constructs DecisionRecord:
   {
     decision_id: "dec-67890",
     verdict: APPROVE,
     timestamp: "2025-12-24T10:30:00Z",
     operation: WRITE,
     intent: "store conversation summary"
   }
8. Policy gate returns DecisionRecord to memory system
9. Memory system proceeds to atomic commit (SYP-0001)
```

## Appendix B: Rationale for Resource Bounds (Non-Normative)

Unbounded contract evaluation creates denial-of-service vulnerabilities:

- **Malicious contracts** could contain infinite loops
- **Malicious requests** could trigger exponential expansions
- **Buggy contracts** could hang on edge cases

Resource bounds ensure the system remains responsive even under adversarial conditions. The fail-closed default (timeout → refuse) preserves safety while protecting availability.

Deployments requiring more sophisticated contracts can increase bounds as needed, but MUST still enforce finite limits.

## Appendix C: Extensibility Patterns (Non-Normative)

Implementations may extend the policy gate interface with:

1. **Custom Verdict Types:** Beyond APPROVE/REFUSE (e.g., CONSTRAIN, QUARANTINE)
2. **Asynchronous Evaluation:** For long-running governance processes (external approval workflows)
3. **Multi-Stage Evaluation:** Chaining evaluators for layered governance (risk → ethics → compliance)
4. **Cached Decisions:** Reusing prior verdicts for identical requests (with TTL/invalidation)

Extensions SHOULD be backward-compatible with baseline SYP-0004 requirements.
