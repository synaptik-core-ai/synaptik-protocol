# SYP-0000: Synaptik Protocol Introduction

![draft](https://img.shields.io/badge/status-draft-yellow.svg?style=flat-square)

**Created:** 2025-12-22  
**Updated:** 2025-12-22

## Abstract

The Synaptik Protocol defines a standard for governance-integrated durable memory systems where safety constraints are enforced by construction rather than by prompting. It addresses a fundamental pattern in AI system failures: autonomous agents that cannot remember—or enforce—what they should not do.

## Motivation

An AI trading bot loses $50M in 4 minutes.  
An LLM leaks PII in production.  
An autonomous system makes a decision you can't explain to regulators.

**The pattern:** Not one of them remembered what they shouldn't do.

Traditional approaches rely on:
- Pre-training alignment (degraded by fine-tuning or drift)
- Prompt engineering (bypassed or forgotten in long contexts)
- Post-hoc filtering (too late if the action already executed)
- External guardrails (disconnected from the memory/reasoning substrate)

These fail because **constraints live outside the decision loop**. By the time a system recalls a memory, proposes an action, or generates a token, it has already "forgotten" the rules.

## Core Innovation

Synaptik inverts this: **constraints are enforced at admission, not at output.**

The protocol centers on admission-time governance: memory writes MUST pass through a **policy gate** that suspends the operation, evaluates governance contracts under resource bounds, and only commits if the decision trace proves compliant. Recalls are typically read-only and are not re-evaluated per access; implementers MAY add post-hoc sweeps or session-level audits if they need additional assurance without slowing the hot path. Unsafe proposals are expected to be refused before they influence the memory graph, embeddings, or causal history, and refusals MUST be auditable.

This creates systems that:
1. **Cannot forget rules** — Contracts are cryptographically locked and integrity-checked at runtime
2. **Explain every decision** — Immutable audit trails with cryptographic authentication enable regulatory replay
3. **Fail safely** — Refused proposals never corrupt the knowledge base or influence future reasoning

## Protocol Scope

The Synaptik Protocol specifies:

1. **Memory Admission Control** (SYP-001) — How proposals are suspended, evaluated, and atomically committed with decision traces
2. **Contract Definition Format** (SYP-002) — Declarative rule syntax for constraints, risk assessment, and intervention policies
3. **Audit Trail Structure** (SYP-003) — Append-only logbook schemas, cryptographic proofs, and tamper-evidence
4. **Policy Gate Interface** (SYP-004) — Standard APIs for pre-execution gating across memory, retrieval, and action systems
5. **Integrity Verification** (SYP-005) — Contract pinning, digest validation, and runtime lock enforcement
6. **Causal Provenance** (SYP-006) — DAG lineage, parent tracking, and replay-aware retention

The protocol is **implementation-agnostic** and designed for interoperability across systems.

## Design Principles

1. **Enforcement by construction** — Safety is a property of the substrate, not a bolt-on layer
2. **Pre-execution gating** — Evaluation happens before state mutation, not after
3. **Immutable provenance** — Every decision leaves a cryptographic trace that cannot be altered
4. **Resource-bounded contracts** — Governance rules execute under strict time/memory/instruction limits to prevent DoS
5. **Offline-first operation** — No network dependencies; contracts are embedded at build time
6. **Fail-closed semantics** — Ambiguous or timed-out decisions default to refusal

## Out of Scope

This protocol does NOT specify:
- Storage backends (SQLite, Postgres, distributed stores)
- Embedding models or vector search algorithms
- LLM providers or inference infrastructure
- Specific domain contracts (PII, financial, medical — those are policy libraries)
- Network protocols for multi-node coordination

## Document Structure

- **SYP-0000** (this document) — Introduction and overview
- **SYP-0001** — Memory Admission Control specification
- **SYP-0002** — Contract Definition Language
- **SYP-0003** — Audit Trail Format
- **SYP-0004** — Policy Gate API
- **SYP-0005** — Integrity & Verification
- **SYP-0006** — Causal Provenance & DAG

Each subsequent spec is self-contained and can be implemented independently, though full compliance requires SYP-0001 through SYP-0005.

## Terminology

- **Proposal** — A candidate operation (write, recall, promotion) awaiting governance approval
- **Contract** — A declarative ruleset defining constraints, risk thresholds, and interventions
- **Policy Gate** — The enforcement layer that evaluates proposals against active contracts
- **Decision Trace** — Cryptographic record of a governance evaluation (authenticated, timestamped, verdict, constraints)
- **Admission** — The atomic commit of a proposal after passing all gates
- **Refusal** — Rejection of a proposal; no state mutation occurs

## Versioning

Protocol specs follow semantic versioning:
- **Major version** — Breaking changes to wire formats, APIs, or semantics
- **Minor version** — Backward-compatible additions (new optional fields, extended enums)
- **Patch version** — Clarifications, typos, non-normative changes

Current version: **0.1.0** (pre-release; subject to breaking changes before 1.0)

## Normative Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this and subsequent specifications are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).