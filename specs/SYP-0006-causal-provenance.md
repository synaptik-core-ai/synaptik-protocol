# SYP-0006: Causal Provenance & DAG

![draft](https://img.shields.io/badge/status-draft-yellow.svg?style=flat-square)

**Created:** 2025-12-24  
**Updated:** 2025-12-24  
**Version:** 0.1.0

## Abstract

This specification defines requirements for tracking causal relationships between state mutations in Synaptik-compliant systems. Causal provenance enables replay analysis, retention policies that preserve dependencies, and forensic investigation of decision chains. Unlike append-only audit trails (SYP-0003) which record linear event sequences, causal provenance captures the directed acyclic graph (DAG) of dependencies between operations, enabling systems to answer questions like "what led to this state?" and "what depends on this decision?"

## Motivation

State mutations in governed systems rarely occur in isolation. Understanding causality is critical for:

**Replay & Analysis:**
- "How did the system arrive at this conclusion?"
- "What prior decisions influenced this behavior?"
- "Can I reproduce this behavior by replaying these operations?"

**Retention & Pruning:**
- "Can I delete this old data without breaking dependencies?"
- "What downstream operations would be affected by removing this entry?"
- "Which historical states are safe to archive?"

**Incident Investigation:**
- "What sequence of events led to this policy violation?"
- "Which prior admission enabled this problematic state?"
- "What would have happened if decision X had been refused?"

**Regulatory Compliance:**
- "Demonstrate the causal chain leading to this financial transaction"
- "Prove that consent was obtained before data access"
- "Show that all preconditions were met before executing this operation"

Without causal provenance, systems can log what happened but not why, making root cause analysis and regulatory demonstration difficult or impossible.

## Terminology

- **Causal Provenance** — Metadata tracking dependencies between state mutations
- **DAG (Directed Acyclic Graph)** — A graph structure where edges represent causal dependencies and cycles are prohibited
- **Causal Parent** — An operation that another operation depends upon or derives from
- **Causal Child** — An operation that depends upon or derives from another operation
- **Causal Chain** — An ordered sequence of operations from root causes to derived effects
- **Reachability** — The set of operations accessible by following causal edges from a given node
- **Transitive Closure** — All operations directly or indirectly reachable via causal dependencies
- **Orphan** — An operation whose causal parents are no longer available (due to deletion or archiving)
- **Root Operation** — An operation with no causal parents (initial state or external trigger)

## Core Requirements

### 3.1 Causal Metadata

State-mutating operations MUST support recording causal parent references.

**Parent Reference Requirements:**

1. Each operation SHOULD declare zero or more causal parents
2. Parent references MUST be stable identifiers that do not change over time
3. Parent references MUST be resolvable to the corresponding operation records (resolvable may include tombstones or archived metadata, not necessarily full operation content)
4. Systems MUST detect and refuse operations that reference non-existent parents (dangling references)

**Parent Reference Structure:** Each parent reference MUST include:
- **Identifier** — Unique stable ID of the parent operation (UUID, content hash, sequential ID, etc.)

Parent references MAY additionally include:
- **Timestamp** — When the parent operation occurred
- **Operation Type** — Classification of the parent (for validation)
- **Content Digest** — Fingerprint of parent content (for integrity checking)

**Zero-Parent Operations:** Operations with no parents are permitted and represent:
- Initial system state
- External triggers (user input, API calls, sensor data)
- Explicitly independent operations

### 3.2 DAG Structure

Causal relationships MUST form a directed acyclic graph; cycles are prohibited.

**Acyclicity Requirements:**

1. Systems MUST detect cycles during operation admission
2. If an operation would create a cycle, the system MUST refuse it
3. Cycle detection MAY use optimistic approaches (assume no cycles, detect on traversal)
4. Cycle detection MUST eventually detect all cycles (no false negatives)

**Rationale:** Cycles prevent meaningful causal reasoning:
- "A caused B, B caused C, C caused A" is logically incoherent for temporal causality
- Replay becomes ambiguous (where does the cycle start?)
- Transitive closure algorithms may not terminate

**Non-Normative Note:** Systems may use incremental cycle detection (reject on first cycle), maintain topological orderings, or use other mechanisms. The requirement is semantic (no cycles exist), not algorithmic.

### 3.3 Causal Traversal

Systems MUST support querying causal relationships in both directions.

**Forward Traversal (Effects):**
- Given an operation, return its causal children (operations that depend on it)
- Support limiting depth (immediate children only, or N levels deep)
- Support filtering by operation type, time range, or other criteria

**Backward Traversal (Causes):**
- Given an operation, return its causal parents (operations it depends on)
- Support recursive traversal (parents of parents)
- Support transitive closure (all ancestors)

**Traversal Requirements:**

1. Traversal MUST NOT alter the graph structure (read-only)
2. Traversal MUST handle missing nodes gracefully (orphans, archived data)
3. Traversal results SHOULD be consistent (repeated queries return same results if graph unchanged)
4. Traversal MUST terminate (no infinite loops, even on malformed graphs)
5. Traversal APIs SHOULD respect the same access controls as operation content retrieval

**Query Interface (Conceptual):**
```
get_causal_parents(operation_id) → List<OperationID>
get_causal_children(operation_id) → List<OperationID>
get_causal_ancestors(operation_id, depth_limit?) → List<OperationID>
get_causal_descendants(operation_id, depth_limit?) → List<OperationID>
```

The concrete API is implementation-defined, but MUST support these semantic queries.

### 3.4 Replay Ordering

Systems SHOULD support extracting replay-compatible operation orderings from the causal graph.

**Replay Ordering Requirements:**

1. If operation B depends on operation A, A MUST be replayed before B
2. Operations with no dependency relationship MAY be replayed in any order (or concurrently)
3. The ordering MUST respect the DAG's topological structure

**Topological Sort:** Systems providing replay orderings SHOULD use topological sort or equivalent:
- Guarantees causal dependencies are respected
- Non-unique: multiple valid orderings may exist for the same DAG
- Systems MAY use timestamps, operation IDs, or other stable tiebreakers for determinism

**Partial Ordering:** Systems MAY return partial orderings when full orderings are not required:
- "These operations must come before those operations"
- Enables parallelization of independent operations during replay

**Orphan Handling:** If an operation's parents are unavailable:
- The system MAY omit the operation from replay orderings
- The system MAY include the operation with a marker indicating missing prerequisites
- The system MUST document its orphan handling policy

### 3.5 Retention & Deletion

Systems implementing retention policies MUST consider causal dependencies.

**Dependency-Aware Retention:**

1. Before deleting an operation, systems SHOULD identify all causal children
2. Systems MUST choose one of:
   - **Refuse Deletion** — Prevent deletion if children exist
   - **Cascade Deletion** — Delete operation and all descendants (with confirmation)
   - **Orphaning** — Allow deletion, mark children as orphans
   - **Tombstone** — Replace deleted operation with a minimal marker preserving causal structure

**Orphan Policy:** If systems permit orphaning:
- Orphans MUST be clearly marked (not confused with root operations)
- Orphan records SHOULD include tombstone metadata (when/why parent was deleted)
- Traversal APIs MUST handle orphans gracefully (not crash or return corrupt data)

**Archival:** Systems MAY archive old operations to cold storage, provided:
- Causal metadata remains queryable (even if content is offline)
- Archived operations can be restored if dependencies are needed
- Archive/restore events are logged in audit trails

### 3.6 Integrity Protection

Causal metadata SHOULD be protected from tampering.

**Integrity Mechanisms:**

1. Parent references SHOULD be immutable once committed
2. Systems MAY include parent fingerprints in child records (binds child to specific parent version)
3. Systems MAY use cryptographic techniques (Merkle DAGs, hash chains) to detect tampering
4. If causal metadata is altered, systems MUST invalidate dependent integrity proofs

**Non-Normative Note:** Merkle DAGs (content-addressed graphs where each node's ID is its hash including parent hashes) provide strong integrity but require careful design for mutable metadata.

## Causal Graph Properties (Informative)

### 4.1 Branching

DAGs naturally support branching causality:
- Multiple children per parent (one decision influences multiple subsequent operations)
- Multiple parents per child (operation depends on multiple prior decisions)

**Example:**
```
    A (initial state)
   / \
  B   C  (parallel operations depending on A)
   \ /
    D   (operation depending on both B and C)
```

### 4.2 Merge Points

Operations with multiple parents represent merge points:
- "Combine information from sources B and C"
- "Decision D required approval from both B and C"
- "State D is derived from prior states B and C"

### 4.3 Root Nodes

Root nodes (operations with no parents) represent:
- Initial system state
- External inputs (user queries, API requests)
- Explicitly independent operations

Systems MAY track multiple roots (multi-tenant systems, separate workflows, etc.).

### 4.4 Leaf Nodes

Leaf nodes (operations with no children) represent:
- Terminal states (end of workflow)
- Recently added operations (not yet referenced)
- Orphaned operations after deletion

Leaf nodes are not inherently significant; children may be added later.

## Example Causal Scenarios (Non-Normative)

### 5.1 Simple Linear Chain

```
Op1 (user query)
 ↓
Op2 (retrieve relevant memory)
 ↓
Op3 (generate response)
 ↓
Op4 (store interaction)
```

Each operation depends only on its immediate predecessor. Replay order is deterministic.

### 5.2 Fan-Out (Parallel Derivation)

```
       Op1 (ingest document)
      / | \
     /  |  \
   Op2 Op3 Op4 (extract entities, summarize, classify)
```

Op1 triggers multiple independent downstream operations. Op2, Op3, Op4 can execute concurrently during replay.

### 5.3 Fan-In (Synthesis)

```
  Op1     Op2     Op3  (retrieve from 3 different sources)
   \       |      /
    \      |     /
         Op4 (synthesize answer)
```

Op4 depends on Op1, Op2, and Op3. Replay must execute Op1-3 before Op4, but Op1-3 can execute in any order.

### 5.4 Diamond Pattern (Common Ancestor)

```
      Op1 (initial query)
     /   \
   Op2   Op3 (retrieve different aspects)
     \   /
      Op4 (combine results)
```

Op4 has a common ancestor (Op1) through multiple paths. Replay must ensure Op1 executes first, then Op2 and Op3, then Op4.

### 5.5 Retention Scenario

Suppose Op2 from the diamond pattern is subject to deletion (retention policy expires):

**Option A: Refuse Deletion**
- Op4 depends on Op2 (transitively), so Op2 cannot be deleted

**Option B: Cascade Deletion**
- Delete Op2, Op4, and any descendants of Op4

**Option C: Orphaning**
- Delete Op2, mark Op4 as having a missing parent
- Op4 remains queryable but flagged as incomplete

**Option D: Tombstone**
- Replace Op2 with minimal metadata: "Op2 existed, deleted at timestamp T, reason R"
- Preserves causal structure without preserving Op2's content

## Compliance Requirements

### 6.1 Baseline Compliance

A system is **baseline SYP-0006 compliant** if it:

1. Records causal parent references for state-mutating operations (Section 3.1)
2. Enforces DAG acyclicity (Section 3.2)
3. Supports forward and backward causal traversal (Section 3.3)

### 6.2 Extended Compliance

A system is **fully SYP-0006 compliant** if it additionally:

4. Provides replay-compatible operation orderings (Section 3.4)
5. Implements dependency-aware retention policies (Section 3.5)
6. Protects causal metadata integrity (Section 3.6)

### 6.3 Optional Extensions

Systems MAY implement additional features:
- Real-time causal graph visualization
- Causal query languages (graph query DSLs)
- Statistical analysis of causal patterns
- Causal graph compression or summarization
- Distributed causality tracking (multi-node systems)

## Integration with Other SYP Specifications

### 7.1 Audit Trails (SYP-0003)

Causal provenance and audit trails are complementary:
- **Audit trails** provide a linear record of what happened when
- **Causal graphs** provide a non-linear view of why and how operations relate

Systems SHOULD:
- Include causal parent references in audit trail decision entries
- Use audit trail timestamps to resolve ambiguities in causal orderings
- Cross-reference: audit entries link to causal graph nodes, graph nodes link to audit entries

### 7.2 Memory Admission Control (SYP-0001)

Causal relationships are established during admission:
- Proposals declare their causal parents
- Admission control verifies parent references are valid
- Upon commitment, causal edges are atomically added to the graph

Refused proposals MUST NOT establish causal edges (orphaned proposals have no impact on the graph).

### 7.3 Contract Evaluation (SYP-0002)

Contracts MAY reference causal metadata:
- "Refuse if this operation has more than N ancestors"
- "Require approval if operation depends on high-risk parent"
- "Rate-limit operations in the same causal chain"

Evaluators have read-only access to causal graph structure during evaluation (no mutations).

### 7.4 Integrity Verification (SYP-0005)

Causal metadata enhances integrity:
- Parent fingerprints create cryptographic chains (like blockchain)
- Merkle DAG structures provide tamper-evident provenance
- Causal traversal enables verifying full dependency chains

Systems MAY use causal structure to detect retroactive tampering (if parent hashes don't match, causality was corrupted).

## Security Considerations

### 8.1 Causal Graph Manipulation

**Threat:** Adversaries may attempt to forge causal relationships or create false dependencies.

**Mitigations:**
- Validate parent references during admission (parents must exist and be accessible)
- Use cryptographic binding (parent content hashes included in child records)
- Audit trail causal edges (record when edges are added)
- Fail-closed: invalid parent references cause admission refusal

### 8.2 Cycle Injection

**Threat:** Adversaries may attempt to create cycles to DoS replay or confuse analysis.

**Mitigations:**
- Enforce DAG acyclicity during admission (Section 3.2)
- Traverse graph with cycle detection (fail if cycle discovered)
- Limit traversal depth (prevent infinite loops in case of bugs)

### 8.3 Information Disclosure

**Threat:** Causal metadata may reveal sensitive relationships (e.g., "operation B derived from confidential operation A").

**Mitigations:**
- Apply access control to causal traversal APIs (not just to operation content)
- Redact parent references in public audit trails if needed
- Use content-addressed identifiers (hashes) instead of descriptive IDs

### 8.4 Retention Bypass

**Threat:** Adversaries may exploit causal dependencies to prevent legitimate data deletion ("make operation X depend on everything to prevent cleanup").

**Mitigations:**
- Require justification for causal parent declarations (governance evaluation)
- Implement forced deletion with tombstones (override dependencies in exceptional cases)
- Audit deletion attempts and refusals (detect abuse patterns)

## Performance Considerations (Non-Normative)

### 9.1 Graph Storage

DAG storage may use:
- **Adjacency lists** — Efficient for traversal, space-efficient for sparse graphs
- **Materialized transitive closure** — Fast ancestor/descendant queries, higher storage cost
- **Content-addressed stores** — Natural for Merkle DAGs, enables deduplication

### 9.2 Traversal Optimization

- **Caching** — Cache reachability sets for frequently queried nodes
- **Indexing** — Maintain indexes for common queries (all leaves, all roots, depth-N ancestors)
- **Pruning** — Exclude archived/deleted nodes from hot path queries

### 9.3 Cycle Detection

- **Incremental** — Check for cycles only when adding edges (efficient for admission)
- **Batch** — Periodically scan entire graph (detects cycles from imports or corruption)
- **Topological ordering** — Maintain ordering incrementally (efficient detection on updates)

## Versioning & Evolution

### 10.1 Backward Compatibility

Future versions of this specification MAY:
- Add optional causal metadata fields (e.g., edge weights, edge types)
- Define standard query languages for causal graphs
- Specify distributed causality protocols

Future versions MUST NOT:
- Remove parent reference support (breaks core requirement)
- Allow cycles (breaks DAG invariant)
- Make traversal optional (breaks baseline compliance)

### 10.2 Extensibility

Implementations MAY extend causal metadata with:
- **Edge Labels** — Annotate why operation B depends on operation A ("derived from", "approved by", "supersedes")
- **Temporal Constraints** — "B must occur within T seconds of A"
- **Quantified Dependencies** — "B depends on at least 3 of {A1, A2, A3, A4}"

Future specification profiles MAY define:
- **Content-Addressed Operation IDs** — A profile where operation identifiers are cryptographic hashes of their content and causal metadata, enabling strong Merkle-DAG integrity guarantees

Extensions SHOULD be documented and SHOULD NOT conflict with core requirements.

## Normative Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).
