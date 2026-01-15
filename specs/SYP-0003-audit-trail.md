# SYP-0003: Audit Trail Format

![draft](https://img.shields.io/badge/status-draft-yellow.svg?style=flat-square)

**Created:** 2025-12-24  
**Updated:** 2025-12-24  
**Version:** 0.1.0

## Abstract

This specification defines requirements for durable, tamper-evident audit trails that record governance decisions in Synaptik-compliant systems. Audit trails provide cryptographic evidence of what was evaluated, when, by which rules, and with what outcome. This enables regulatory compliance, forensic analysis, incident investigation, and third-party verification without trusting the operating system.

## Motivation

Governance without auditability is unverifiable. Organizations need to demonstrate:
- What rules were in effect at any point in time
- Which operations were approved or refused
- Why each decision was made
- That historical records haven't been altered

Traditional logging systems are mutable, easily deleted, and lack cryptographic guarantees. Synaptik audit trails address this by requiring:
- Append-only semantics (no retroactive changes)
- Tamper-evident construction (modifications are detectable)
- Cryptographic authentication (decisions provably originate from specific systems/keys)
- Causal linking (decisions reference their context and dependencies)

## Terminology

- **Audit Trail** — A durable, append-only log of governance decision records
- **Decision Entry** — A single record documenting one governance evaluation
- **Append-Only** — Writes add new entries; existing entries cannot be modified or deleted
- **Tamper-Evidence** — Modifications to entries or their ordering are cryptographically detectable
- **Batch** — A collection of decision entries sealed together with a single cryptographic proof
- **Seal** — The process of finalizing a batch with integrity metadata
- **Anchor** — An optional external timestamp or immutable reference proving a record existed at a specific time

## Core Requirements

### 3.1 Append-Only Semantics

Audit trails MUST be append-only: new entries are added, but existing entries cannot be modified or deleted.

**Append-Only Requirements:**

1. Once written, decision entries MUST remain unchanged
2. Deletion of individual entries MUST NOT be possible through normal operations
3. Entry ordering MUST be preserved (no reordering of past decisions)
4. Implementations MUST detect and refuse attempts to modify historical entries

**Operational Exceptions:**

Systems MAY support:
- **Archive & Rotate** — Moving old trail segments to cold storage while preserving integrity
- **Retention Policies** — Deleting trail file bytes after configured periods, provided:
  - Deletions are recorded as cryptographically committed tombstones in an immutable retention ledger
  - The tombstone records what was deleted, when, by whom, and why
  - Verification can confirm the deletion was authorized and logged
  - In other words: bytes may be removed from hot storage, but history cannot be erased without trace
- **Emergency Purge** — Operator-initiated removal of specific entries, provided:
  - Purge events are logged in a separate append-only audit of audits (retention ledger)
  - Gaps in sequence numbers are detectable
  - Purges require explicit authorization and justification
  - Purge tombstones are themselves tamper-evident

**Non-Normative Implementation Note:** Append-only file systems, write-once media, or database table constraints can enforce append-only semantics.

### 3.2 Decision Entry Structure

Each decision entry MUST contain at minimum:

1. **Decision Identifier** — Unique ID for this evaluation (UUID, sequential counter, etc.)
2. **Timestamp** — When the decision was made (ISO 8601 or Unix epoch)
3. **Operation Classification** — Type of operation evaluated (read, write, etc.)
4. **Verdict** — Outcome of evaluation (approve, refuse)
5. **Intent** — Description of what was being attempted

Decision entries SHOULD additionally include:

- **Actor** — Who initiated the operation (user ID, service account, etc.)
- **Purpose** — Context for the operation (production, training, replay, etc.)
- **Scope** — Namespace or partition (e.g., database schema)
- **Target** — Specific resource identifier (memory key, file path, etc.)
- **Contract References** — Which contracts were evaluated (name, version, digest)
- **Constraint Violations** — Which specific rules failed (if any)
- **Risk Score** — Computed risk assessment (if applicable)
- **Content Fingerprint** — Hash of proposal content (enables correlation without storing sensitive data)

Decision entries MAY include:

- **Justification** — Human-readable explanation of verdict
- **Execution Duration** — Time spent in evaluation
- **Causal Parents** — References to prior decisions this depends on (see SYP-0006)
- **System Metadata** — Version, environment, deployment identifiers

### 3.3 Tamper-Evidence

Audit trails MUST be tamper-evident: modifications are cryptographically detectable.

**Tamper-Evidence Mechanisms:**

Systems MUST implement cryptographic techniques to make modifications detectable. Various approaches exist including cryptographic chains, authenticated data structures, and verifiable logging schemes.

**Non-Normative Note:** Consult cryptographic literature and standards (e.g., RFC 6962) for specific tamper-evident construction techniques.

**Detection Requirements:**

1. Tampering with entry content MUST invalidate its integrity proof
2. Deletion of entries MUST be detectable via sequence gaps or broken hash chains
3. Reordering of entries MUST be detectable via timestamp or sequence inconsistencies
4. Verification MUST NOT require access to the system that created the trail

**Verification Process:** Systems MUST provide a method for external verifiers to:
- Compute expected integrity values (hashes, signatures, proofs)
- Compare against recorded integrity metadata
- Detect discrepancies indicating tampering

### 3.4 Batching and Sealing

To reduce overhead, systems MAY batch multiple decision entries before computing integrity proofs.

**Batching Requirements:**

1. Batches MUST have deterministic boundaries (fixed size, time window, or explicit flush)
2. Batch sealing MUST be atomic (all entries in batch get same integrity metadata)
3. Batch metadata MUST include:
   - Batch identifier or sequence number
   - Timestamp of seal operation
   - Count of entries in batch
   - Integrity proof (implementation-defined cryptographic commitment)

**Batch Size Tradeoffs:**

- **Small batches** — Lower latency, more frequent integrity operations, higher overhead
- **Large batches** — Higher latency, fewer integrity operations, lower overhead

Systems SHOULD support configurable batch sizes with reasonable defaults (e.g., 128 entries or 1 second, whichever comes first).

**Flush Semantics:** Systems MUST flush batches:
- When batch size limit is reached
- When time limit is reached
- On graceful shutdown
- On explicit operator request

Emergency shutdowns MAY leave incomplete batches; systems SHOULD handle these gracefully on restart.

### 3.5 Cryptographic Authentication

Decision entries SHOULD include cryptographic proofs of authenticity.

**Authentication Mechanisms:**

Systems MAY use various cryptographic authentication techniques to prove the origin and integrity of audit entries. Implementation choices depend on threat model, performance requirements, and regulatory needs.

**Proof Requirements:**

1. Proofs MUST cover all critical fields (decision ID, verdict, timestamp, intent)
2. Proofs MUST be computed over a canonical serialization of covered fields (deterministic field ordering, encoding, and representation)
3. The canonicalization procedure MUST be documented to enable independent verification
4. Proofs MUST be verifiable by intended auditors (internal, external, regulatory)
5. Proof algorithms MUST use cryptographically strong primitives (see SYP-0005)
6. Proof metadata MUST identify which key/algorithm was used (key ID, algorithm name)

**Rationale:** Canonical serialization prevents "it verifies on my machine" failures where different implementations hash different field orderings or representations, producing incompatible proofs for identical logical entries.

**Key Rotation:** Systems MUST support key rotation:
- Old entries remain verifiable with old keys
- New entries use new keys
- Key validity periods are documented
- Both old and new keys are available during transition periods

### 3.6 Sequential Consistency

Audit trails MUST preserve decision ordering.

**Ordering Requirements:**

1. Entries MUST be appended in the order decisions were made
2. Timestamps MUST establish a consistent ordering (monotonically increasing, or via logical clocks, or batch sealing)
3. Sequence numbers (if used) MUST be contiguous (no gaps except documented purges)
4. Replaying the trail MUST reconstruct governance history accurately

**Timestamp Options:** Systems MAY use:
- **Wall-Clock Timestamps** — Physical time (UTC), monotonic within tolerance for small clock skew
- **Logical Clocks** — Causally-ordered timestamps for establishing happened-before relationships
- **Batch Timestamps** — All entries in a batch share the seal timestamp (ordering within batch may be by sequence number)

**Concurrency:** Systems processing concurrent decisions MAY:
- Serialize concurrent decisions to a single trail
- Maintain separate trails per evaluation context, with synchronization metadata
- Use causally-ordered timestamps to establish happened-before relationships

The chosen approach MUST be documented and MUST enable reconstructing causal relationships.

## Audit Trail Access

### 4.1 Read Access Control

Audit trails contain sensitive information; access MUST be controlled.

**Access Requirements:**

1. Audit trail reads SHOULD require authentication
2. Access SHOULD be logged (audit of audits)
3. Different roles MAY have different access levels:
   - **Operators** — Full read access for debugging
   - **Auditors** — Read access to decision metadata (may exclude sensitive content)
   - **External Regulators** — Access to cryptographic proofs and verdicts (may exclude internal details)

**Confidentiality vs. Verifiability:** Systems balancing these concerns MAY:
- Store sensitive content separately from decision metadata
- Use content fingerprints (hashes) in audit trails instead of plaintext
- Encrypt audit trails with keys available only to authorized auditors

### 4.2 Query and Retrieval

Systems SHOULD support querying audit trails.

**Query Capabilities:**

1. **By Time Range** — Retrieve decisions within a date/time window
2. **By Verdict** — Filter for approvals, refusals, or specific intervention types
3. **By Actor** — Find decisions for a specific user or service
4. **By Intent** — Search for operations matching intent patterns
5. **By Target** — Find decisions affecting specific resources
6. **By Contract** — Locate uses of specific governance rules

**Performance Considerations:**

- Systems MAY index trails for query performance
- Indexes SHOULD NOT compromise tamper-evidence (index separately from raw trail)
- Large queries SHOULD be paginated or streamed to avoid memory exhaustion

### 4.3 Retention and Archival

Systems MUST define audit trail retention policies.

**Retention Requirements:**

1. Trails MUST be retained long enough to satisfy compliance requirements (industry/jurisdiction-specific)
2. Retention periods SHOULD be configurable
3. Archived trails MUST preserve tamper-evidence
4. Archived trails SHOULD remain queryable (possibly with degraded performance)

**Archival Workflow:**
```
1. Identify trail segments exceeding active retention period
2. Verify integrity before archival (detect corruption before moving)
3. Copy to archival storage (S3, tape, cold storage)
4. Verify integrity after copy
5. Delete from active storage only after verification
6. Document archive location in metadata
```

### 4.4 Export and Portability

Audit trails SHOULD be exportable for external analysis.

**Export Requirements:**

1. Exports MUST preserve integrity metadata (signatures, hashes, proofs)
2. Exports SHOULD use standard formats (JSONL, CSV with signatures, etc.)
3. Exports MUST be verifiable independently of the source system
4. Export documentation MUST describe verification procedures

**Interoperability:** Exported trails SHOULD be usable by:
- Different Synaptik implementations
- Third-party audit tools
- Regulatory reporting systems
- Forensic analysis platforms

## Failure Handling

### 5.1 Write Failures

Systems MUST handle audit trail write failures gracefully.

**Write Failure Responses:**

1. If audit write fails, the decision MUST NOT be lost
2. Systems MUST retry writes with exponential backoff
3. Systems SHOULD buffer failed writes in memory or temporary storage
4. If buffering fails, systems MUST either:
   - Fail-closed (refuse new operations until audit trail is restored), OR
   - Emit critical alerts and continue (degraded mode)

The chosen approach MUST be documented.

**Backpressure:** Systems SHOULD implement backpressure:
- Slow down evaluation rate if audit trail is lagging
- Refuse new operations if audit buffer is full
- Alert operators before critical thresholds are reached

### 5.2 Storage Exhaustion

Systems MUST monitor audit trail storage capacity.

**Capacity Management:**

1. Systems SHOULD emit warnings when storage is approaching capacity
2. Systems SHOULD automatically rotate/archive old trails to free space
3. Systems MUST NOT silently drop audit entries when storage is full
4. Systems MUST fail-closed or alert operators when capacity is exhausted

### 5.3 Corruption Detection

Systems MUST detect and respond to audit trail corruption.

**Detection Mechanisms:**

1. Verify integrity on startup (detect corruption from crashes or disk failures)
2. Periodically verify integrity of recent entries (detect in-flight corruption)
3. Verify integrity before archival (detect corruption before it propagates)

**Corruption Response:**

1. Alert operators immediately
2. Isolate corrupted trail segments
3. Attempt recovery from backups or replicas
4. Document corruption events in a separate log-of-logs

## Integration with Other Specifications

- **SYP-0001 (Memory Admission Control):** Audit entries are created during atomic commitment phase
- **SYP-0002 (Contract Evaluation):** Audit entries reference which contracts were evaluated
- **SYP-0004 (Policy Gate):** Policy gates generate decision records appended to audit trails
- **SYP-0005 (Integrity & Verification):** Audit entries include cryptographic proofs per SYP-0005 requirements
- **SYP-0006 (Causal Provenance):** Audit entries MAY reference causal parent decisions for lineage tracking

## Conformance

A system conforms to SYP-0003 if:

1. Governance decisions are recorded in an append-only audit trail
2. Audit entries contain required metadata (ID, timestamp, verdict, intent)
3. The trail is tamper-evident via cryptographic techniques (implementation-defined)
4. Entries are recorded in chronological order with sequential consistency
5. The trail is durable and survives system restarts
6. External verifiers can detect tampering without system access

**Testing Recommendations:**
- Verify entries cannot be modified after writing
- Verify tampering with entries is detectable via integrity checks
- Verify trail survives crashes and restarts
- Verify external verification procedures work correctly

## Security Considerations

### 8.1 Confidentiality

Audit trails may contain sensitive information:

**Sensitive Data:**
- Content fingerprints may reveal patterns
- Intents may describe confidential operations
- Actors may be privileged accounts

**Protection Mechanisms:**
- Encrypt trails at rest with strong ciphers
- Use content hashes instead of plaintext when possible
- Redact or pseudonymize sensitive fields
- Apply access controls to trail files

### 8.2 Integrity

Integrity failures compromise the entire audit system:

**Threat Vectors:**
- Filesystem corruption (disk failures)
- Software bugs (crashes during writes)
- Malicious modification (attackers with filesystem access)

**Defenses:**
- Use checksums/signatures for tamper-evidence
- Replicate trails to multiple storage systems
- Store trails on write-once media when possible
- Separate audit trail storage from application data

### 8.3 Availability

Audit trails must remain accessible:

**Availability Threats:**
- Storage exhaustion
- Denial-of-service attacks (log flooding)
- Accidental deletion

**Mitigations:**
- Capacity monitoring and alerting
- Rate limiting on audit entry generation
- Backup and archival procedures
- Retention policies with operator approval for deletions

### 8.4 Non-Repudiation

Audit trails serve as evidence; they must be trustworthy:

**Requirements for Legal/Regulatory Use:**
- Digital signatures provide stronger non-repudiation than MACs
- Signatures must be timestamped (internal timestamp or external anchor)
- Key management must be documented and auditable
- Verification procedures must be reproducible

## Normative References

- **RFC 2119** — Key words for use in RFCs to Indicate Requirement Levels
- **SYP-0000** — Synaptik Protocol Introduction
- **SYP-0001** — Memory Admission Control (normative for when entries are created)
- **SYP-0004** — Policy Gate API (normative for decision record format)
- **SYP-0005** — Integrity & Verification (normative for cryptographic proofs)

## Informative References

- **SYP-0002** — Contract Evaluation
- **SYP-0006** — Causal Provenance & DAG
- **RFC 6962** — Certificate Transparency (verifiable log structure)
- **RFC 3161** — Time-Stamp Protocol (external anchoring)

---

## Implementation Complexity (Non-Normative)

Implementing tamper-evident audit trails at scale presents several challenges:

**Write Throughput:**
- How to append audit entries at sustained production rates without becoming a bottleneck?
- What batching strategies balance latency with cryptographic overhead?
- How to maintain append-only semantics under concurrent load?

**Tamper-Evidence Performance:**
- How to compute cryptographic proofs without blocking audit writes?
- What data structures enable efficient verification of individual entries in large batches?
- How to generate inclusion proofs for entries in multi-terabyte audit logs?

**Storage & Retention:**
- How to manage audit trail growth over years of continuous operation?
- What archival strategies preserve tamper-evidence after moving to cold storage?
- How to verify integrity of archived segments without rehydrating all data?

**Coordination Under Concurrency:**
- How to assign globally unique sequence numbers when multiple writers operate concurrently?
- What mechanisms prevent conflicting batch seals when writes occur in parallel?
- How to maintain sequential consistency when audit entries are created concurrently?

Naive implementations either sacrifice performance (synchronous cryptographic operations per entry) or sacrifice integrity (buffering without crash recovery). Production systems require sophisticated techniques to achieve both.

## Verification Requirements

Compliant audit trail systems MUST provide:

1. **Tamper Detection Tests** — Demonstrate modification of entries, deletion of entries, and reordering of entries are all detectable
2. **Independent Verification** — Provide verification tools that work without access to the system that created the trail
3. **Performance Benchmarks** — Document achieved write throughput, verification time, and storage efficiency
4. **Crash Recovery Evidence** — Demonstrate no entries are lost or corrupted after crashes during writes

Claims of compliance without demonstrable tamper detection and independent verification should be treated skeptically.

## Adversarial Scenarios (Non-Normative)

Implementations should demonstrate resilience against:

**Scenario 1: Silent Entry Modification**
- After trail creation, modify entry content directly in storage
- Verify: Tampering is detected via integrity check failures
- Verify: Modified entry is identified specifically

**Scenario 2: Entry Deletion**
- Delete individual entries from middle of audit trail
- Verify: Deletion is detected via sequence gaps or broken chains
- Verify: System can identify which entries are missing

**Scenario 3: Reordering Attack**
- Reorder entries within a batch or across batches
- Verify: Reordering is detected via timestamp or sequence inconsistencies
- Verify: Original order can be proven from timestamps/chains

**Scenario 4: Rollback Attack**
- Restore audit trail to earlier state, discarding recent entries
- Verify: Rollback is detectable (missing expected entries)
- Verify: External anchors or timestamps prove later entries existed

**Scenario 5: Write Flood Denial of Service**
- Flood audit trail at 10x sustained production rate
- Verify: System applies backpressure without data loss
- Verify: Audit trail remains verifiable after flood
- Verify: No memory exhaustion or crash under sustained load

**Scenario 6: Concurrent Tamper During Verification**
- Modify entries while external verifier is computing proofs
- Verify: Verifier detects inconsistency
- Verify: No false-positive verification from race conditions

Most implementations will fail Scenarios 4-6. Systems passing all six demonstrate cryptographically sound tamper-evidence under adversarial conditions.

## Appendix A: Example Audit Entry (Non-Normative)

A conceptual audit entry includes:

- Core decision fields: decision_id, timestamp, operation, verdict, intent
- Context fields: actor, purpose, scope, target
- Evaluation metadata: contracts_evaluated, risk_score, execution_time
- Content fingerprint: hash of proposal content
- Integrity proof: cryptographic authentication data (format is implementation-defined)

The exact structure and format are implementation-defined. See commercial and reference implementations for specific formats.

## Appendix B: Tamper Detection Example (Non-Normative)

Tamper-evident audit trails use cryptographic techniques to link entries such that modifications break verifiable relationships.

General verification approach:
1. Recompute cryptographic commitments from entry content
2. Check that computed values match recorded values
3. Verify cryptographic relationships between entries
4. Any mismatch indicates tampering

The specific cryptographic construction is implementation-defined. Consult standards like RFC 6962 or commercial implementations for concrete techniques.

## Appendix C: Batch Verification (Non-Normative)

When entries are batched together, tamper-evident constructions can provide efficient verification of individual entries using compact proofs.

General properties of batch verification:
- Individual entries can be verified without accessing all batch entries
- Proof size is typically logarithmic relative to batch size
- Batch commitments cryptographically bind all entries together

The specific construction and proof format are implementation-defined. Various authenticated data structures provide these properties with different tradeoffs.

## Appendix D: Rationale for Append-Only Design (Non-Normative)

**Why not allow updates?**

Mutable logs create several problems:
1. **Tampering:** Attackers can silently alter past decisions to hide violations
2. **Disputes:** Without immutability, parties can claim logs were retroactively changed
3. **Forensics:** Investigators cannot trust that evidence reflects actual events

**Cost of immutability:**

- Storage grows indefinitely (mitigated by archival and retention policies)
- Errors cannot be corrected (mitigated by append-only corrections that reference the original entry)
- Privacy compliance is harder (mitigated by content fingerprints instead of plaintext, and documented purge procedures)

The Synaptik protocol prioritizes evidence integrity over operational convenience. Systems requiring mutability should maintain separate operational logs and immutable audit trails.

## Appendix E: Performance Optimization Strategies (Non-Normative)

High-throughput systems may find synchronous audit writes costly:

**Optimization Approaches:**

Various techniques exist for optimizing audit trail performance:
- Buffering and asynchronous writes (with crash recovery considerations)
- Batching to amortize cryptographic costs (with latency tradeoffs)
- Sharding across multiple storage systems (with coordination complexity)
- Deferred proof computation (with eventual consistency tradeoffs)

Each optimization involves specific tradeoffs between performance, durability, and latency. Systems SHOULD document which optimizations are used and their failure modes. Consult cryptographic literature and commercial implementations for specific techniques.
