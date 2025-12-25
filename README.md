# Synaptik Protocol (SYP) Overview

This protocol defines how Synaptik components communicate, prove lineage, and enforce governance. It targets implementers of Thalamus and related tools, service integrators who embed Synaptik into their own platforms, and auditors who verify invariants. The scope covers runtime behaviors and wire formats for routing, lineage, and contract enforcement; it excludes non-Synaptik application logic. Each SYP document is versioned independently (e.g., SYP-0001) and backward-compatibility expectations are stated per section so you can evaluate upgrade impacts.

## Specification Status Labels

Each spec includes a status badge to indicate its maturity:

- ![wip](https://img.shields.io/badge/status-wip-red.svg?style=flat-square) **wip** — A work-in-progress, possibly to describe an idea before actually committing to a full draft of the spec.
- ![draft](https://img.shields.io/badge/status-draft-yellow.svg?style=flat-square) **draft** — A draft that is ready to review. It should be implementable.
- ![reliable](https://img.shields.io/badge/status-reliable-orange.svg?style=flat-square) **reliable** — A spec that has been adopted (implemented) and can be used as a reference point to learn how the system works.
- ![stable](https://img.shields.io/badge/status-stable-green.svg?style=flat-square) **stable** — We consider this spec to close to final, it might be improved but the system it specifies should not change fundamentally.
- ![permanent](https://img.shields.io/badge/status-permanent-blue.svg?style=flat-square) **permanent** — This spec will not change.
- ![deprecated](https://img.shields.io/badge/status-deprecated-grey.svg?style=flat-square) **deprecated** — This spec is no longer in use.

## Index

The specs contained in this repository are:

- **Synaptik Protocol:**
    - [SYP-0000: Introduction](specs/SYP-0000-introduction.md) - start here to understand the protocol's motivation and design principles
    - [SYP-0001: Memory Admission Control](specs/SYP-0001-memory-admission-control.md) - atomic admission semantics for governed state mutations
    - [SYP-0002: Contract Evaluation](specs/SYP-0002-contract-evaluation.md) - declarative governance rule structure and evaluation semantics
    - [SYP-0003: Audit Trail Format](specs/SYP-0003-audit-trail.md) - durable, tamper-evident decision record logging
    - [SYP-0004: Policy Gate API](specs/SYP-0004-policy-gate.md) - pre-execution governance evaluation interface
    - [SYP-0005: Integrity & Verification](specs/SYP-0005-integrity-verification.md) - governance artifact protection and decision authentication
    
**In Progress:**
    - SYP-0006: Causal Provenance & DAG (planned)
