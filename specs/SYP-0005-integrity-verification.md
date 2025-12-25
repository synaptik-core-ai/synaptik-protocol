# SYP-0005: Integrity & Verification

![draft](https://img.shields.io/badge/status-draft-yellow.svg?style=flat-square)

**Created:** 2025-12-24  
**Updated:** 2025-12-24  
**Version:** 0.1.0

## Abstract

This specification defines requirements for protecting governance artifacts (contracts, policies, rules) from unauthorized modification and enabling third-party verification of governance decisions. Integrity protection ensures that the governance rules evaluated during admission control (SYP-0001) and policy gating (SYP-0004) are authentic and have not been tampered with, while verification mechanisms enable external auditors to validate decision records without trusting the evaluating system.

## Motivation

Governance-integrated systems are only as trustworthy as their governance rules. If contracts can be silently modified, bypassed, or forged, the entire admission control framework becomes security theater. Similarly, if decision records can be altered after the fact, audit trails lose their evidentiary value.

This specification establishes:
1. How governance artifacts are protected from tampering
2. How systems verify artifact integrity before use
3. How decision records can be cryptographically authenticated
4. How third parties can verify decisions without system access

## Terminology

- **Governance Artifact** — Any file or data structure containing governance rules (contracts, policies, constraint definitions)
- **Integrity Digest** — A cryptographic fingerprint of an artifact's content
- **Trust Anchor** — A reference point for verifying artifact authenticity (public key, certificate, root hash)
- **Binding** — The process of associating an artifact with its integrity metadata
- **Verification** — The process of checking that an artifact matches its expected integrity digest
- **Authentication** — Proving a decision record was issued by a specific system or key

## Core Requirements

### 3.1 Artifact Integrity Protection

Systems MUST ensure governance artifacts cannot be modified without detection.

**Protection Requirements:**

1. Each governance artifact MUST have an associated integrity digest
2. The digest MUST be computed using a collision-resistant hash function
3. The digest MUST be computed over a canonical serialization of the artifact content
4. The digest MUST be verified before the artifact is used for evaluation
5. If verification fails, the system MUST refuse operations depending on that artifact

**Canonical Serialization:** To ensure cross-implementation compatibility, systems MUST define and document their canonicalization procedure for hashed artifacts:
- Deterministic field ordering (if structured formats)
- Explicit character encoding (e.g., UTF-8)
- Normalization rules (e.g., newline handling, whitespace, trailing data)
- The procedure MUST be documented to enable independent verification

**Rationale:** Without canonical serialization, two implementations may hash different byte sequences for semantically identical artifacts, breaking verification interoperability.

**Digest Binding:** Implementations MUST bind artifacts to their digests using one of:
- **Detached signatures** — Separate signature files referencing the artifact
- **Embedded signatures** — Signatures included within the artifact structure
- **Manifest files** — Index files mapping artifact names to expected digests

**Collision Resistance:** The hash function MUST provide at least 128 bits of collision resistance (e.g., SHA-256, SHA3-256, BLAKE3). Cryptographically broken functions (MD5, SHA-1) MUST NOT be used.

### 3.2 Artifact Verification Process

Before using a governance artifact, systems MUST verify its integrity.

**Verification Steps:**

1. Compute the artifact's current digest
2. Compare against the expected digest from binding metadata
3. If they match, proceed with evaluation
4. If they differ, refuse the operation and alert operators

**Verification Frequency:**

- **On Load:** Artifacts MUST be verified when first loaded into memory
- **On Use:** Systems SHOULD verify artifacts before each evaluation in high-assurance deployments or when artifacts may be hot-reloaded; in-memory tampering detection is most relevant when process integrity is suspect
- **Periodic:** Systems MAY periodically re-verify artifacts to detect file-level corruption

**Failure Handling:** If verification fails:
- The system MUST NOT use the compromised artifact
- The system MUST refuse any operation whose evaluation depends on the failed artifact
- Deployments MAY choose to refuse all operations system-wide (fail-closed global)
- The system SHOULD emit an alert or alarm for operator review
- The system MAY attempt to reload from a backup or canonical source

**Rationale:** Scoped refusal preserves partial availability when only specific contracts fail, while still preventing unsafe evaluations. Global fail-closed is permitted for high-assurance deployments.

### 3.3 Artifact Authenticity

Systems SHOULD verify that artifacts originate from a trusted source.

**Authenticity Mechanisms (RECOMMENDED):**

1. **Digital Signatures** — Artifacts signed by a trusted key
2. **Certificate Chains** — Hierarchical trust rooted in a certificate authority
3. **Out-of-Band Verification** — Manual operator approval or checksums published via secure channels

**Signature Verification:** If signatures are used:
- Systems MUST verify signatures before trusting artifacts
- Systems MUST check signature validity (expiration, revocation if applicable)
- Systems MUST fail-closed if signature verification fails

**Key Management:** Trust anchors (public keys, certificates, root hashes) MUST be:
- Stored separately from signed artifacts
- Protected from unauthorized modification
- Accessible to the verification process at runtime

**Non-Normative Note:** Authenticity is RECOMMENDED but not strictly required for baseline compliance. Systems without signing can still achieve integrity protection via operator-verified digests.

### 3.4 Artifact Immutability

Once bound and verified, governance artifacts SHOULD be treated as immutable during their active lifetime.

**Immutability Practices:**

1. Artifacts SHOULD be stored in read-only or append-only storage
2. Updates SHOULD create new versions rather than modifying existing artifacts
3. Version changes SHOULD require explicit operator approval
4. Active artifacts SHOULD NOT be deleted while referenced by running evaluations

**Versioning:** Systems MUST support multiple versions of artifacts to enable safe updates:
- Old versions remain valid until explicitly deprecated
- New versions are verified independently before activation
- Transitions between versions follow a controlled rollout process

## Decision Record Authentication

### 4.1 Decision Integrity

Decision records produced by policy gates (SYP-0004) SHOULD include cryptographic proofs.

**Proof Types:**

Systems SHOULD include one or more:
- **Integrity Proofs** — Evidence of tamper-detection (hashes, Merkle trees, chained digests)
- **Authenticity Proofs** — Evidence of issuer identity (digital signatures, MACs)

**Integrity Proof Mechanisms:**
- **Chained Hashes** — Each record includes hash of prior record
- **Merkle Inclusion Proofs** — Hierarchical hashes for efficient batch verification

**Authenticity Proof Mechanisms:**
- **Digital Signatures** — Public-key signatures enabling third-party verification
- **Message Authentication Codes (MACs)** — Keyed hashes proving origin to key holders

**Minimum Requirements:**

1. Each decision record SHOULD contain an integrity proof and/or authenticity proof
2. Proofs MUST be computed over a canonical serialization of covered fields (deterministic field ordering, encoding, and representation)
3. The canonicalization procedure MUST be documented to enable independent verification
4. Proofs MUST cover all critical fields (decision ID, verdict, timestamp, intent)
5. Tampering with any covered field MUST invalidate the proof
6. Proofs SHOULD be verifiable without access to the evaluating system

**Rationale:** Canonical serialization prevents verification incompatibilities where different implementations produce different byte sequences for logically identical decision records.

**Field Coverage:** At minimum, integrity proofs SHOULD cover:
- Decision identifier
- Verdict (approve/refuse)
- Operation classification
- Timestamp
- Intent or content fingerprint

### 4.2 Proof Verification

Third parties MUST be able to verify decision records independently.

**Verification Requirements:**

1. Systems MUST document the proof format and verification procedure
2. Verification MUST NOT require access to private keys or internal system state
3. Verification SHOULD be implementable by external auditors in different programming languages
4. Verification failure MUST be detectable (invalid proofs don't silently pass)

**Trust Anchors:** Systems using public-key signatures MUST publish:
- The public keys used for signing
- Key validity periods (if applicable)
- Key identifiers to support rotation

Systems using symmetric MACs MAY share verification keys with auditors via secure channels, but this limits third-party verifiability.

### 4.3 Replay Protection

Decision records SHOULD include mechanisms to prevent replay attacks.

**Replay Prevention Mechanisms:**

1. **Unique Identifiers** — Each decision has a globally unique ID (UUID, sequential counter with namespace)
2. **Timestamps** — Records include precise issuance time
3. **Nonces** — Random values prevent identical records from having identical proofs
4. **Sequence Numbers** — Monotonic counters detect reordering or omissionDo 

Systems SHOULD include at least two of these mechanisms to provide defense in depth.

**Anti-Replay Verification:** Verifiers SHOULD reject decision records with duplicate decision IDs within the same issuer namespace or audit trail, preventing replay of old decisions as new approvals.

## Governance Update Procedures

### 5.1 Safe Update Protocol

Updating governance artifacts MUST NOT create exploitable windows where rules are unenforced.

**Update Requirements:**

1. New artifacts MUST be verified before activation
2. The system MUST NOT enter a state where no valid artifacts are loaded
3. If activation fails, the system MUST roll back to previous artifacts
4. Updates SHOULD be atomic (all-or-nothing across related artifacts)

**Deployment Pattern (RECOMMENDED):**
```
1. Verify new artifact integrity and authenticity
2. Stage new artifact in a non-active location
3. Test new artifact in a sandbox or staging environment
4. Atomically swap active artifact pointer
5. Monitor for evaluation failures after activation
6. Retain old artifact for emergency rollback
```

### 5.2 Emergency Revocation

Systems SHOULD support rapid revocation of compromised artifacts.

**Revocation Mechanisms:**

1. **Blocklist** — List of artifact digests that MUST NOT be used
2. **Expiration** — Artifacts automatically invalidate after a configured lifetime
3. **Kill Switch** — Emergency operator command to disable specific artifacts

Upon revocation:
- The system MUST immediately stop using the revoked artifact
- The system MUST fail-closed until a replacement artifact is verified
- The system SHOULD alert operators to the revocation event

### 5.3 Version Rollback

Systems SHOULD support controlled rollback to previous artifact versions.

**Rollback Requirements:**

1. Previous versions MUST be retained for a configured retention period
2. Rollback MUST verify the old artifact's integrity before reactivation
3. Rollback SHOULD emit an audit event documenting the version change
4. Rollback SHOULD NOT bypass normal integrity verification

## Cryptographic Requirements (Recommendations)

### 6.1 Hash Functions

Systems MUST use collision-resistant hash functions for integrity digests.

**Required Support:**
- SHA-256 (256-bit security) — REQUIRED to support for interoperability
- SHA3-256 (256-bit security, alternative construction) — REQUIRED to support for interoperability

**Recommended:**
- BLAKE3 (256-bit security, high performance) — OPTIONAL but RECOMMENDED for performance-critical deployments

**Deprecated Functions:** MUST NOT use:
- MD5 (broken collision resistance)
- SHA-1 (broken collision resistance)

**Rationale:** Requiring support for both SHA-256 and SHA3-256 ensures cross-implementation compatibility. BLAKE3 is permitted for performance but not required to avoid forcing implementations to adopt emerging algorithms before widespread adoption.

### 6.2 Signature Schemes (If Used)

Systems using digital signatures SHOULD use standard, well-analyzed schemes.

**Recommended Signature Schemes:**
- Ed25519 (high performance, small keys)
- ECDSA with P-256 (widely supported)
- RSA-PSS with 3072-bit keys (conservative choice)

**Post-Quantum Considerations:** Systems with long-term security requirements MAY additionally use post-quantum signature schemes (e.g., SPHINCS+, Dilithium), but SHOULD maintain classical signatures for interoperability.

### 6.3 Key Management

Cryptographic keys used for signing or MAC generation MUST be protected.

**Key Protection Requirements:**

1. Private keys MUST NOT be embedded in application code
2. Private keys SHOULD be stored in hardware security modules (HSMs) or secure enclaves when available
3. Private keys SHOULD be backed up with offline or split-key recovery mechanisms
4. Key material SHOULD be rotated periodically (recommended: annually)

**Key Rotation:** When rotating keys:
- Old keys MUST remain available to verify historical records
- New artifacts MUST be signed with new keys
- Systems MUST support multiple concurrent public keys for transition periods

## Integration with Other Specifications

- **SYP-0001 (Memory Admission Control):** Integrity verification occurs before evaluation; failures trigger fail-closed refusal
- **SYP-0002 (Contract Evaluation):** Contracts are verified before evaluation; invalid contracts cannot be loaded
- **SYP-0003 (Audit Trail):** Decision records persisted to audit trails include integrity proofs for tamper-evidence
- **SYP-0004 (Policy Gate):** Policy gates verify artifact integrity before invoking evaluation
- **SYP-0006 (Causal Provenance):** Provenance chains can reference decision proofs for end-to-end verification

## Conformance

A system conforms to SYP-0005 if:

1. Governance artifacts have integrity digests computed with collision-resistant hash functions
2. Artifacts are verified before use in evaluation
3. Verification failures trigger fail-closed behavior
4. Decision records include integrity proofs (RECOMMENDED) or reference verifiable artifacts

**Testing Recommendations:**
- Verify tampered artifacts are detected and rejected
- Verify missing artifacts trigger fail-closed behavior
- Verify signature/MAC verification failures prevent artifact use
- Verify decision proofs can be verified independently

## Security Considerations

### 9.1 Trust Anchor Protection

Trust anchors (public keys, root hashes) are critical security components:

1. Compromise of trust anchors allows forging arbitrary artifacts
2. Trust anchors SHOULD be embedded at build time or loaded from secure hardware
3. Trust anchors SHOULD NOT be modifiable by application code
4. Systems SHOULD support multiple trust anchors for key rotation and vendor diversity

### 9.2 Downgrade Attacks

Attackers may attempt to force systems to use older, vulnerable artifacts:

1. Systems SHOULD track the latest known version of each artifact
2. Systems MAY refuse to load artifacts older than a configured threshold
3. Version rollback (Section 5.3) SHOULD require explicit operator approval
4. Audit trails SHOULD log all version changes for anomaly detection

### 9.3 Timing Attacks

Verification operations may leak information through timing side-channels:

1. Signature verification SHOULD use constant-time implementations
2. Hash computation is generally safe but implementations should avoid data-dependent branches
3. Error messages SHOULD NOT reveal partial verification results

### 9.4 Supply Chain Attacks

Compromised build or distribution processes may inject malicious artifacts:

1. Artifacts SHOULD be signed by developers at build time
2. Signatures SHOULD be verified by operators before deployment
3. Reproducible builds enable independent verification of artifact contents
4. Artifact provenance (who built, when, from which source) SHOULD be documented

## Normative References

- **RFC 2119** — Key words for use in RFCs to Indicate Requirement Levels
- **SYP-0000** — Synaptik Protocol Introduction
- **SYP-0001** — Memory Admission Control (normative for fail-closed integration)
- **SYP-0004** — Policy Gate API (normative for pre-evaluation verification)

## Informative References

- **SYP-0002** — Contract Definition Language
- **SYP-0003** — Audit Trail Format
- **SYP-0006** — Causal Provenance & DAG
- **NIST FIPS 180-4** — Secure Hash Standard (SHA-2)
- **NIST FIPS 202** — SHA-3 Standard
- **RFC 8032** — Edwards-Curve Digital Signature Algorithm (EdDSA)

---

## Appendix A: Example Integrity Verification Flow (Non-Normative)

```
1. System starts and needs to load governance contracts
2. System reads contract file: contracts/financial.toml
3. System computes digest:
   digest = HASH(file_contents)
   → "a3f2...89bc"
4. System reads binding metadata: contracts/.signatures
5. System finds entry:
   financial.toml → expected_digest="a3f2...89bc", signature="..."
6. System compares computed vs. expected digest → MATCH
7. System verifies signature (if present) → VALID
8. System loads contract into evaluation engine
9. Policy gate uses verified contract for evaluation
```

If step 6 fails (digest mismatch), system refuses all operations requiring that contract.

## Appendix B: Decision Record Proof Example (Non-Normative)

A decision record with integrity proof might look like:

```json
{
  "decision_id": "dec-a1b2c3d4",
  "verdict": "APPROVE",
  "operation": "WRITE",
  "intent": "store user preference",
  "timestamp": "2025-12-24T15:45:00Z",
  "integrity_proof": {
    "algorithm": "ED25519",
    "signature": "3a7f...d9e2",
    "signing_key_id": "key-2025-prod",
    "covered_fields": ["decision_id", "verdict", "operation", "intent", "timestamp"]
  }
}
```

A third-party verifier can:
1. Reconstruct the signed message from covered fields
2. Fetch the public key for "key-2025-prod"
3. Verify the signature cryptographically
4. Confirm the record hasn't been tampered with

## Appendix C: Artifact Manifest Example (Non-Normative)

A manifest file binding artifacts to digests:

```toml
# contracts/.manifest.toml
version = "1.0"
created_at = "2025-12-24T10:00:00Z"

[[artifact]]
name = "financial.toml"
digest_algorithm = "BLAKE3"
digest = "a3f2...89bc"
size_bytes = 4096

[[artifact]]
name = "privacy.toml"
digest_algorithm = "BLAKE3"
digest = "7c8d...34ef"
size_bytes = 2048

[signature]
algorithm = "ED25519"
public_key_id = "key-2025-prod"
signature = "9f3a...6b1c"
```

Systems load the manifest, verify its signature, then verify each artifact against its expected digest before use.

## Appendix D: Rationale for Fail-Closed on Integrity Failures (Non-Normative)

If integrity verification fails, three options exist:

1. **Proceed anyway** (fail-open) → Attacker can bypass governance by tampering artifacts
2. **Use cached/old rules** → Attacker forces stale policy enforcement
3. **Refuse all operations** (fail-closed) → Service degrades but safety is preserved

Synaptik chooses fail-closed because:
- Governance bypass is an unacceptable security failure
- Stale policies may permit now-forbidden operations
- Service degradation is preferable to silent compromise
- Operators can resolve integrity failures and restore service

High-availability deployments should maintain redundant, verified artifact sources rather than compromising integrity enforcement.
