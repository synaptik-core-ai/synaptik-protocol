# SYP-0002: Contract Evaluation

![draft](https://img.shields.io/badge/status-draft-yellow.svg?style=flat-square)

**Created:** 2025-12-24  
**Updated:** 2025-12-24  
**Version:** 0.1.0

## Abstract

This specification defines how governance contracts are structured, evaluated, and composed in Synaptik-compliant systems. Contracts are declarative rule sets that assess proposals against constraints, compute risk scores, and signal interventions. This layer enables governance logic to be expressed independently of application code, making policies auditable, portable, and verifiable without executing arbitrary code.

## Motivation

Embedding governance rules directly in application logic creates several problems:
- Rules are scattered across codebases, making audits difficult
- Changes require code modifications and redeployment
- Non-technical stakeholders cannot review or approve rules
- Different implementations may interpret rules inconsistently

Declarative contracts address these by:
- Centralizing governance logic in structured documents
- Enabling non-developer review and approval
- Facilitating portability across implementations
- Supporting formal verification and testing

## Terminology

- **Contract** — A declarative specification of governance rules applied to proposals
- **Constraint** — A condition that proposals must satisfy (e.g., "no PII", "budget < $1000")
- **Evaluation** — The process of checking a proposal against contract rules
- **Risk Score** — A numeric assessment of proposal risk level
- **Intervention** — An action signaled by a contract (approve, refuse, quarantine, constrain)
- **Predicate** — A boolean expression evaluated against proposal content or metadata
- **Scope** — The domain to which a contract applies (global, namespace-specific, intent-specific)

## Core Requirements

### 3.1 Declarative Representation

Contracts MUST be expressed in a declarative format separate from executable application code.

**Declarativity Requirements:**

1. Contracts MUST describe *what* to enforce, not *how* to enforce it
2. Contracts MUST be parsable without executing arbitrary code
3. Contracts MUST be representable in human-readable structured data formats
4. Contracts MUST NOT contain imperative control flow (loops, gotos, mutable state)

**Acceptable Formats:** Implementations MAY use:
- Structured markup languages (XML, TOML, YAML, JSON)
- Domain-specific languages with constrained syntax
- Logic programming languages with restricted predicates
- Constraint satisfaction problem (CSP) specifications

**Non-Acceptable:** Contracts MUST NOT be:
- General-purpose programming languages without sandboxing
- Binary formats without structured schemas
- Formats requiring code execution to parse

**Rationale:** Declarative formats enable static analysis, formal verification, and human review without executing potentially malicious code.

### 3.2 Constraint Specification

Contracts MUST support defining constraints on proposal content and metadata.

**Minimum Constraint Types:**

1. **Content-Based** — Rules matching on proposal text or structure (pattern matching, keyword detection)
2. **Metadata-Based** — Rules matching on actor, intent, purpose, scope, or timestamps
3. **Contextual** — Rules referencing existing system state (quotas, rate limits, historical patterns)

**Constraint Structure:** Each constraint SHOULD specify:
- **Condition** — When the constraint applies (predicate or selector)
- **Requirement** — What must be true for satisfaction
- **Severity** — How violations are classified (error, warning, info)
- **Message** — Human-readable explanation for violations

**Example Constraint (Conceptual):**
```
Constraint: NoPersonalIdentifiers
Condition: operation=WRITE AND purpose=production
Requirement: content MUST NOT match SSN_PATTERN
Severity: ERROR
Message: "Personal identifiers prohibited in production writes"
```

### 3.3 Risk Assessment

Contracts SHOULD support computing numeric risk scores for proposals.

**Risk Scoring Requirements:**

1. Risk scores SHOULD be normalized to a consistent range (e.g., 0.0-1.0 or 0-100)
2. Higher scores SHOULD indicate higher risk
3. Score computation SHOULD be deterministic for identical proposals
4. Score thresholds MAY trigger different interventions (approve if <0.3, quarantine if 0.3-0.7, refuse if >0.7)

**Risk Factors:** Contracts MAY consider:
- Constraint violation count and severity
- Proposal content characteristics (length, complexity, entropy)
- Metadata attributes (unknown actors, unusual intents, high-privilege operations)
- Contextual signals (rate of recent refusals, time-of-day anomalies)

**Composability:** When multiple contracts apply, systems MUST define how scores combine:
- Maximum (most conservative)
- Average (balanced)
- Weighted sum (configurable priorities)

### 3.4 Intervention Signaling

Contracts MUST indicate what action should be taken for a proposal.

**Minimum Intervention Types:**

1. **APPROVE** — Proposal is permitted unconditionally
2. **REFUSE** — Proposal is denied; no state mutation occurs

**Optional Intervention Types:**

Systems MAY support:
- **APPROVE_WITH_CONDITIONS** — Approve but apply constraints (redaction, tagging, expiration)
- **QUARANTINE** — Isolate for manual review before final decision
- **ESCALATE** — Route to higher-privilege decision-maker or review process
- **DEFER** — Delay decision pending additional context

**Mapping to Binary Decision:** Systems implementing only approve/refuse MUST map optional interventions:
- APPROVE_WITH_CONDITIONS → implementation-defined (approve with logging OR refuse if conditions unsupported)
- QUARANTINE/ESCALATE/DEFER → REFUSE (fail-closed)

### 3.5 Scope and Applicability

Contracts MUST define their scope of application.

**Scope Dimensions:**

1. **Global vs. Scoped** — Applies to all operations OR only specific namespaces/intents
2. **Operation Types** — Applies to reads, writes, both, or specific operation subtypes
3. **Contexts** — Applies in all contexts OR only specific purposes (production, training, replay)

**Scope Specification:** Contracts SHOULD include:
- **Applies To** — Selector defining matching proposals
- **Precedence** — Priority when multiple contracts match
- **Effective Period** — Optional start/end timestamps for time-limited rules

**Default Behavior:** If no contracts match a proposal, systems MUST either:
- Have a default global contract (e.g., "approve all"), OR
- Refuse the proposal (fail-closed)

Systems SHOULD document which approach they use.

## Evaluation Semantics

### 4.1 Evaluation Order

When multiple contracts apply, systems MUST define evaluation order.

**Ordering Strategies:**

1. **Sequential** — Evaluate in declaration order; first refusal terminates
2. **Priority-Based** — Evaluate by precedence; highest-priority verdict wins
3. **Exhaustive** — Evaluate all contracts; aggregate results

Systems MUST document their ordering strategy in conformance documentation.

**Short-Circuit Behavior:** Systems MAY short-circuit evaluation (stop after first REFUSE) for performance, provided:
- All applicable contracts would eventually be evaluated if none refused
- The short-circuit behavior is documented
- Audit trails record which contracts were evaluated vs. skipped

### 4.2 Predicate Evaluation

Contract predicates (conditions, requirements) MUST be evaluated against proposal data.

**Evaluation Requirements:**

1. Evaluation MUST be deterministic for identical inputs (proposal content, governance snapshot, and evaluation context)
2. Evaluation MUST complete within resource bounds (see SYP-0004)
3. Evaluation MUST NOT mutate the proposal or system state
4. Evaluation failures MUST be treated as constraint violations (fail-closed)

**Non-Normative Note:** "Identical inputs" means the same proposal evaluated under the same governance contracts with the same contextual state (historical decisions, quotas, timestamps). Contextual rules (rate limits, time-based policies) remain deterministic because context is part of the input.

**Expression Complexity:** Systems SHOULD limit predicate complexity:
- Maximum nesting depth for logical operators (AND/OR/NOT)
- Maximum number of terms per expression
- Restricted function/operator set to prevent infinite loops

### 4.3 Context Access

Contracts MAY access contextual information during evaluation.

**Permitted Context:**

1. **Proposal Content** — Text, structure, metadata being evaluated
2. **Historical State** — Read-only access to prior decisions, quotas, statistics (snapshot at evaluation time)
3. **System Metadata** — Timestamps, system identifiers, environment tags

**Snapshot Consistency:** Contextual state MUST reflect a consistent point-in-time snapshot. Systems MUST NOT allow partial reads where some context reflects one moment and other context reflects another, as this breaks determinism guarantees.

**Prohibited Context:**

1. **External Network** — No HTTP requests, DNS lookups, or external API calls
2. **Mutable State** — No writes to databases, files, or shared memory
3. **Ambient Authority** — No access to secrets, credentials, or privileged operations

**Rationale:** Restricting context ensures evaluations are reproducible, auditable, and cannot exfiltrate data or exhaust resources.

### 4.4 Composition and Inheritance

Systems MAY support composing contracts from smaller rules or inheriting from templates.

**Composition Patterns:**

1. **Inclusion** — Reference shared constraint libraries
2. **Extension** — Specialize a base contract with additional rules
3. **Override** — Refine parent contract rules for specific scopes

**Conflict Resolution:** When composed contracts conflict:
- Systems MUST define precedence rules (most-specific wins, explicit priority, etc.)
- Systems MUST NOT silently ignore conflicts
- Systems SHOULD emit warnings when conflicts are detected

## Contract Lifecycle

### 5.1 Validation

Contracts MUST be validated before activation.

**Validation Requirements:**

1. **Syntax** — Contract format must conform to schema
2. **Semantics** — Predicates must be well-formed and type-safe
3. **Resource Bounds** — Contract complexity must not exceed evaluator limits
4. **Consistency** — No internal contradictions (e.g., "approve AND refuse")

Validation failures MUST prevent contract activation.

### 5.2 Versioning

Contracts SHOULD support versioning for controlled updates.

**Version Management:**

1. Each contract SHOULD have a version identifier
2. Version changes SHOULD be tracked in audit logs
3. Systems SHOULD support multiple contract versions concurrently during transitions
4. Systems SHOULD NOT retroactively apply new contracts to old decisions

**Breaking Changes:** Updates that change evaluation semantics SHOULD increment major version numbers and require explicit migration.

### 5.3 Deprecation

Contracts SHOULD support graceful deprecation.

**Deprecation Workflow:**

1. Mark contract as deprecated with effective end date
2. Emit warnings when deprecated contracts are evaluated
3. After end date, refuse to load deprecated contracts
4. Retain deprecated contracts in archives for audit/replay purposes

## Security Considerations

### 6.1 Injection Attacks

Contract formats MUST be resistant to injection attacks.

**Protection Mechanisms:**

1. Use structured formats with strict parsing (not string concatenation)
2. Validate all inputs before evaluation
3. Escape or sanitize user-provided strings in predicates
4. Use parameterized queries if accessing databases

### 6.2 Resource Exhaustion

Contracts MUST NOT enable denial-of-service through unbounded evaluation.

**Resource Controls:**

1. Evaluation timeout (enforced by SYP-0004)
2. Maximum predicate complexity
3. Limits on contextual data access
4. Prohibition of recursive or infinite structures

### 6.3 Information Disclosure

Contracts MUST NOT leak sensitive information through side channels.

**Leakage Vectors:**

1. **Timing** — Evaluation time should not reveal proposal content
2. **Error Messages** — Failures should not expose internal state
3. **Risk Scores** — Scores should not encode sensitive data

Systems SHOULD use constant-time evaluation where feasible and sanitize error messages.

### 6.4 Privilege Escalation

Contracts MUST NOT enable unauthorized actions.

**Restrictions:**

1. Contracts operate with least privilege (read-only access to context)
2. Contracts cannot invoke privileged operations (write, delete, escalate permissions)
3. Contract authors MUST NOT be able to exempt themselves from governance (no self-approval rules)
4. Contract modifications MUST be subject to the same governance as operational changes or to independent approval workflows

**Rationale:** If contract authors can modify contracts governing their own operations without oversight, governance becomes security theater. Systems SHOULD enforce separation of duties between contract authorship and operational execution.

## Integration with Other Specifications

- **SYP-0001 (Memory Admission Control):** Contracts are evaluated during the suspension phase; results determine admission
- **SYP-0003 (Audit Trail):** Contract evaluations generate decision records persisted to audit trails
- **SYP-0004 (Policy Gate):** Policy gates load and invoke contracts within resource bounds
- **SYP-0005 (Integrity & Verification):** Contracts are integrity-checked before evaluation to prevent tampering
- **SYP-0006 (Causal Provenance):** Contracts may reference causal parent metadata in evaluations

## Conformance

A system conforms to SYP-0002 if:

1. Governance rules are expressed in declarative contracts separate from application code
2. Contracts support defining constraints on content and metadata
3. Evaluation is deterministic, isolated, and resource-bounded
4. Contracts signal interventions (approve/refuse at minimum)
5. Contracts define scope and applicability
6. Contracts are validated before activation

**Testing Recommendations:**
- Verify contracts can be parsed without code execution
- Verify malformed contracts are rejected during validation
- Verify evaluation respects resource bounds
- Verify multiple contracts compose according to documented strategy

## Normative References

- **RFC 2119** — Key words for use in RFCs to Indicate Requirement Levels
- **SYP-0000** — Synaptik Protocol Introduction
- **SYP-0001** — Memory Admission Control (normative for evaluation integration)
- **SYP-0004** — Policy Gate API (normative for evaluation invocation)
- **SYP-0005** — Integrity & Verification (normative for contract integrity)

## Informative References

- **SYP-0003** — Audit Trail Format
- **SYP-0006** — Causal Provenance & DAG

---

## Appendix A: Example Contract Structure (Non-Normative)

A conceptual contract structure:

```
Contract: PrivacyProtection
Version: 1.2.0
Scope:
  Operations: [WRITE]
  Purpose: [production]
  
Constraints:
  - Name: NoSSN
    Condition: purpose = production
    Requirement: content MUST NOT match \d{3}-\d{2}-\d{4}
    Severity: ERROR
    Message: "Social Security Numbers prohibited"
    
  - Name: NoEmail
    Condition: purpose = production AND actor NOT IN trusted_users
    Requirement: content MUST NOT match email_pattern
    Severity: WARNING
    Message: "Email addresses discouraged"

Risk:
  Base: 0.0
  PerViolation:
    ERROR: +0.5
    WARNING: +0.2
  Threshold:
    REFUSE: 0.7
    QUARANTINE: 0.4

Intervention:
  Default: APPROVE
  IfRiskAbove(0.7): REFUSE
  IfRiskAbove(0.4): QUARANTINE
```

This is illustrative; actual formats are implementation-defined.

## Appendix B: Evaluation Algorithm (Non-Normative)

A reference evaluation algorithm:

```
1. Load all contracts matching proposal scope
2. Sort contracts by precedence/priority
3. Initialize risk_score = 0.0
4. For each contract:
   a. Evaluate all constraints
   b. Collect violations
   c. Update risk_score based on violations
   d. Determine intervention from risk thresholds
   e. If intervention = REFUSE, short-circuit and return
5. If no contract refused:
   a. Compute final intervention from risk_score
   b. Return decision record with verdict and metadata
6. If any evaluation errors occurred:
   a. Return REFUSE (fail-closed)
```

## Appendix C: Constraint Expression Language Considerations (Non-Normative)

When designing constraint expression languages, consider:

**Simplicity:**
- Minimal syntax reduces parsing errors
- Limited operators prevent complexity exploits
- Clear semantics aid human review

**Expressiveness:**
- Pattern matching for content analysis
- Logical operators (AND/OR/NOT) for composition
- Comparators (<, >, =, IN, MATCHES) for various data types

**Safety:**
- No Turing-complete computation (prevent halting problems)
- No recursion or unbounded iteration
- No external side effects

**Example minimal language:**
```
field op value
  where op = {=, !=, <, >, IN, MATCHES, CONTAINS}
expression AND expression
expression OR expression
NOT expression
```

This allows rich constraints while maintaining decidability and resource bounds.

## Appendix D: Rationale for Declarative Contracts (Non-Normative)

**Why not executable code?**

Executable code governance (JavaScript, Python, etc.) offers maximum flexibility but creates severe risks:

1. **Security:** Arbitrary code can exfiltrate data, escalate privileges, or DOS the system
2. **Auditability:** Reviewing code requires programming expertise; logic may be obfuscated
3. **Portability:** Code tied to specific runtimes/libraries is not portable across implementations
4. **Verification:** Proving code properties requires formal methods; declarative rules enable lighter-weight analysis

**Hybrid approaches:**

Systems MAY support embedded scripting for advanced rules, provided:
- Scripts execute in sandboxes with strict resource limits
- Scripts are treated as higher-risk artifacts requiring extra review
- Scripts are optional; core governance works without them

Synaptik protocols prioritize safety and auditability over expressiveness, hence the declarative bias.
