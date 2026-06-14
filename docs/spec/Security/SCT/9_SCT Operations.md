SADAR Context Token Operations

*Draft Appendix B for SADAR Core Security v0.5 — Version 2*

*Companion to §4.6 (SCT structure) — abstract operations and conformance requirements*

*April 2026*

*© 2026 Cognita AI Inc. All rights reserved. SADAR™ is a trademark of Cognita AI, Inc. CogniWeave™ is a trademark of Cognita AI, Inc. All other trademarks are the property of their respective owners.*

**Changes from v1 (April 25, 2026)**

This v2 incorporates feedback from review of v1. Substantive changes:

**B.2 Validate.** Optional `n` parameter for last-n-segment validation. Structured return shape with per-segment results. INDETERMINATE removed — public keys for SCT segments are available via signed registry manifests, so key unavailability is an infrastructure failure (KEY\_UNAVAILABLE error) rather than a validity question.

**B.3 Extract Claims.** Restructured. Validation is now an inherent part of execution — no separate prerequisite call required. Default behavior returns the full chain with per-segment annotations indicating signature validity and decryption accessibility. Optional `return\_own` flag restricts output to caller-issued segments. For segments that fail signature verification, only metadata is returned; decrypted contents are withheld even if the caller holds the decryption key (preventing ingestion of unverified content).

**B.5 Verify Chain Integrity.** Returns VALID or INVALID only — no INDETERMINATE. New B.5.2 explicitly states that chain integrity verification operates on cleartext metadata (signatures, headers, bindings) and does not require decryption. Confidentiality and integrity are independent properties; a party that cannot decrypt segments can still verify chain integrity end-to-end.

**Read-only requirements removed.** B.2.4, B.3.3 (read-only sentence), and B.5.5 dropped. Chain immutability is a cryptographic property of the format, not an API contract. Replaced with a single editorial paragraph at the top of the appendix.

**B.6 Error Contract.** CLAIM\_STRUCTURE\_INVALID retained — "claim" terminology aligns with §4.6 and JWS/JWT standards.

**B.7 Conformant Realizations (substantially expanded).** B.7.1 reference primitives now specified in a table with NIST/RFC standards references. B.7.2 introduces the Chain Crypto Suite — a chain-level (not per-segment) assertion of cryptographic primitives. B.7.3 enumerates acceptable suites (SADAR-CRYPTO-1 through 4). B.7.4 defines required security properties (non-repudiation per ISO/IEC 13888-3, replay protection per RFC 9449, confidentiality per NIST SP 800-38D). B.7.5 introduces the Conformance Declaration as a verifiable artifact in the issuer's signed manifest.

**B.8 additions.** B.8.1 defines jti and bounds cache TTL by segment exp. New B.8.6 addresses post-quantum migration with NIST FIPS 203 (ML-KEM) and FIPS 204 (ML-DSA), including hybrid scheme guidance for the transition period.

**Placement**

Recommended placement: new Appendix B in Core Security v0.5, paralleling Appendix A. Cross-referenced from §4.6 (SCT structure).

# Appendix B: Normative Requirements — SADAR Context Token Operations

This appendix defines the abstract operations any conformant implementation SHALL support over the SADAR Context Token (SCT) defined in Section 4.6. Requirements are expressed as SHALL (mandatory), SHOULD (strongly recommended), and SHALL NOT (prohibited). Conformance is assessed against these statements, not against specific implementation mechanisms.

The operations are defined in abstract terms. Implementations MAY expose them through any interface idiom appropriate to the implementation language or runtime, provided the defined behavioral and security properties are preserved. Language bindings, method signatures, and API ergonomics are ecosystem-level concerns and are not normative.

***On chain immutability and read-only operations.*** *SCT chains are structurally immutable. Any modification to a prior segment's content invalidates that segment's JWS signature and is detected by Verify Chain Integrity (Section B.5). The four operations defined in this appendix do not include explicit "read-only" requirements because chain immutability is a property of the format, not a behavioral constraint on operations. An implementation cannot mutate a prior segment without producing a chain that fails subsequent integrity verification. The Append Segment operation produces a new chain that includes the prior segments unchanged; it does not "modify" the prior chain in any meaningful sense.*

## B.1 Operation Set

A conformant implementation SHALL support each of the four operations defined in Sections B.2 through B.5: Validate, Extract Claims, Append Segment, and Verify Chain Integrity. Each operation is defined by its inputs, outputs, preconditions, postconditions, and error classification.

## B.2 Validate

B.2.1 — A conformant implementation SHALL provide a Validate operation that, given an SCT chain, verifies for each segment within scope: (a) the JWS signature against the segment issuer's published key material, and (b) the structural conformance of the segment to the SCT format defined in Section 4.6.

B.2.2 — Validate MAY accept an optional parameter n indicating that only the most recent n segments in the chain are to be validated. If n is omitted, the entire chain SHALL be validated. Implementations SHOULD support the n parameter as an optimization for append-time validation; implementations that do not support partial validation SHALL ignore n and validate the entire chain.

B.2.3 — Validate SHALL return a structured result containing at minimum:

chain\_valid: a boolean — true if and only if all in-scope segments validated successfully

segments: an enumerated list, one entry per in-scope segment, each containing:

– index: the position of the segment in the chain

– jti: the segment's JWT ID

– signature\_valid: a boolean indicating signature verification result

– structure\_valid: a boolean indicating structural conformance to Section 4.6

– failure\_reason: present iff signature\_valid OR structure\_valid is false; identifies the specific failure category from Section B.6

B.2.4 — Implementations SHALL NOT silently drop or ignore segments that fail validation, nor return chain\_valid = true for a chain containing any in-scope segment whose signature\_valid or structure\_valid is false.

B.2.5 — If the issuer's public key for any in-scope segment cannot be retrieved, Validate SHALL fail with KEY\_UNAVAILABLE (Section B.6). Public key material for SCT segments is published in the issuer's signed registry manifest (Core Security Section 4.2) and is not access-controlled; key unavailability indicates a registry connectivity or manifest-validity failure rather than an authorization issue.

## B.3 Extract Claims

B.3.1 — A conformant implementation SHALL provide an Extract Claims operation that, given an SCT chain, performs validation per Section B.2 as part of execution and returns a structured per-segment result containing both validity status and, where the caller is authorized, the decrypted claim set. Validation is an inherent part of Extract Claims execution; no separate prerequisite call is required.

B.3.2 — Extract Claims SHALL return a structured result containing one entry per segment in the chain with at minimum:

index: position of the segment in the chain

jti: the segment's JWT ID

signature\_valid: whether the segment's signature verifies

decryption\_status: one of:

– DECRYPTED — the caller holds the private key corresponding to the segment's JWE recipient and the payload was decrypted successfully

– ENCRYPTED\_NOT\_AUTHORIZED — the caller does not hold the private key corresponding to the segment's JWE recipient and cannot decrypt the payload

claims: present if and only if decryption\_status is DECRYPTED AND signature\_valid is true

failure\_reason: present iff signature\_valid is false; identifies the failure category from Section B.6

B.3.3 — Extract Claims SHALL accept an optional return\_own flag. When return\_own is true, the result list SHALL be filtered to segments where the caller is also the segment's issuer. When return\_own is false or omitted, all segments in the chain SHALL be returned with appropriate per-segment annotations.

B.3.4 — When a segment fails signature verification, Extract Claims SHALL NOT return decrypted claims for that segment, even if the caller holds the corresponding decryption key. Returning unverified content creates a security risk; structural metadata (jti, claimed issuer, signature failure reason) is sufficient for diagnostics. Decrypted claims are returned only when both signature\_valid and decryption\_status indicate success.

B.3.5 — Authorization to decrypt a segment SHALL be determined exclusively by JWE recipient targeting: the caller is authorized to decrypt segment N iff the caller holds the private key corresponding to the public key used to encrypt segment N. There is no separate authorization layer for claim access. ENCRYPTED\_NOT\_AUTHORIZED is a structural condition (the caller is not the JWE recipient), not a policy decision.

B.3.6 — Decryption keys SHALL be obtained through the agent's trust boundary as defined in Core Security Appendix A.1.3. Agent application logic SHALL NOT have direct access to decryption key material. The implementation accesses the caller's decryption keys via the deployment's provisioned trust boundary (workload identity, credential broker sidecar, or framework-level injection).

B.3.7 — Extract Claims SHALL preserve claim type fidelity: returned claim values SHALL have the types declared in Section 4.6, not their serialized forms.

## B.4 Append Segment

B.4.1 — A conformant implementation SHALL provide an Append Segment operation that, given a prior SCT chain, a new claim set, and the issuer signing key and target recipient public key required to construct a new segment, produces a new SCT chain with one additional segment cryptographically chained to the prior tail.

B.4.2 — Append Segment SHALL invoke Validate (Section B.2) on the prior chain as part of execution. If validation fails (chain\_valid = false), Append Segment SHALL fail with PRECONDITION\_FAILED (Section B.6) and SHALL NOT produce an output chain.

B.4.3 — The new segment SHALL be bound to the prior tail segment via the chain structure defined in Section 4.6, including a parent\_sct\_jti claim referencing the prior tail's jti.

B.4.4 — The new segment's claim set SHALL satisfy the structural and semantic requirements of Section 4.6, including required claim presence and type correctness.

B.4.5 — Append Segment SHALL produce an output chain whose Verify Chain Integrity (Section B.5) returns VALID when invoked by any party with access to the issuers' public keys.

B.4.6 — Append Segment SHALL use the Chain Crypto Suite asserted by the input chain (Section B.7.2). It SHALL NOT introduce a different cryptographic suite within an existing chain. To use a different suite, a new chain SHALL be initiated.

## B.5 Verify Chain Integrity

B.5.1 — A conformant implementation SHALL provide a Verify Chain Integrity operation that, given an SCT chain, verifies end-to-end cryptographic integrity across all segments, including JWS signature verification of each segment and parent-child binding integrity between successive segments.

B.5.2 — Chain integrity verification operates exclusively on cleartext metadata (JWS signatures, JWE headers, chain binding fields). It SHALL NOT require access to encrypted payloads. Any party with access to the issuers' public keys — which are publicly available via signed registry manifests per Core Security Section 4.2 — SHALL be able to verify chain integrity end-to-end, regardless of whether they can decrypt any individual segment's content. Confidentiality and chain integrity are independent properties.

B.5.3 — Verify Chain Integrity SHALL return one of two defined outcomes:

VALID — every segment's signature verifies and every parent-child binding is intact

INVALID — at least one segment's signature fails OR at least one parent-child binding is broken

B.5.4 — On INVALID, the operation SHALL identify the specific failing segment(s) and the nature of the failure (signature failure or parent-child binding failure), per Section B.6.

B.5.5 — A party that cannot decrypt one or more segment payloads SHALL still be able to invoke Verify Chain Integrity and receive a VALID or INVALID result. A chain may contain segments encrypted to multiple distinct recipients while remaining cryptographically valid as a chain.

## B.6 Error Contract

B.6.1 — All four operations SHALL signal errors through a defined error classification rather than through implementation-specific exception types or status codes. The classification SHALL distinguish at minimum the following categories:

|  |  |
| --- | --- |
| **Error Category** | **Defined Conditions** |
| PRECONDITION\_FAILED | Operation invoked without satisfying a stated precondition (e.g., Append Segment on a chain whose Validate returned chain\_valid = false). |
| SIGNATURE\_INVALID | One or more segment signatures fail cryptographic verification. |
| CHAIN\_INTEGRITY\_BROKEN | Parent-child cryptographic binding is broken between adjacent segments, or chain ordering invariants are violated. |
| CLAIM\_STRUCTURE\_INVALID | A segment claim set is missing required claims, contains type-incorrect values, or violates the structural constraints of Section 4.6. (Note: "claim" follows JWS/JWT usage per RFC 7519 — the named fields within an SCT segment.) |
| KEY\_UNAVAILABLE | Required key material — signing key for Append Segment, decryption key for Extract Claims, verification key for Validate or Verify Chain Integrity — is not accessible to the implementation. |
| ACCESS\_DENIED | The caller is not authorized to perform the requested operation. Note: ENCRYPTED\_NOT\_AUTHORIZED in Extract Claims (Section B.3.2) is NOT this error — it is a structural condition reported in the per-segment result, not a failure of the operation itself. |

B.6.2 — The mapping from these error categories to language-specific error mechanisms (typed exceptions, error codes, result types, etc.) is an implementation choice and is not normative.

## B.7 Conformant Realizations and Cryptographic Primitives

### B.7.1 Reference Cryptographic Primitives

The reference cryptographic primitives for SCT segments are listed below. These primitives are the v1 reference baseline. Alternative primitives MAY be used within the constraints of Section B.7.2 (Chain Crypto Suite Assertion).

|  |  |  |
| --- | --- | --- |
| **Function** | **Reference Primitive** | **Standard** |
| Segment signature | ECDSA with P-256 curve and SHA-256 (ES256) | RFC 7515 (JWS); FIPS 186-5 |
| Segment payload encryption | AES-256 in GCM mode (A256GCM) | RFC 7516 (JWE); FIPS 197; NIST SP 800-38D |
| Recipient key wrap | ECDH-ES with P-256 + AES-256 Key Wrap (ECDH-ES+A256KW) | RFC 7518; NIST SP 800-56A |
| Chain binding hash | SHA-256 | FIPS 180-4 |

### B.7.2 Chain Crypto Suite Assertion

B.7.2.1 — Each SCT chain SHALL declare a Chain Crypto Suite at the chain header level, identifying the cryptographic primitives in use across the chain. The Chain Crypto Suite assertion is structural and applies chain-wide: all segments in a chain SHALL use the same cryptographic primitives. Mixed-primitive chains are not conformant.

B.7.2.2 — The Chain Crypto Suite assertion SHALL identify, at minimum:

The signature algorithm (e.g., ES256, RS256, EdDSA)

The content encryption algorithm (e.g., A256GCM)

The recipient key wrap algorithm (e.g., ECDH-ES+A256KW, RSA-OAEP-256)

The chain binding hash algorithm (e.g., SHA-256, SHA-384)

B.7.2.3 — A party receiving an SCT chain SHALL inspect the Chain Crypto Suite assertion before performing any cryptographic operation on the chain, and SHALL reject the chain if the asserted suite is not in the receiver's accepted-suite list. Acceptance-list policy is a deployment configuration concern and is not normative.

B.7.2.4 — Once a Chain Crypto Suite is declared at chain initiation, it SHALL NOT change for the lifetime of the chain. A new chain SHALL be initiated to use a different suite.

B.7.2.5 — The Chain Crypto Suite assertion SHALL be covered by the JWS signature on the chain header (or the first segment, depending on chain framing per Section 4.6) so that the suite assertion itself cannot be altered without invalidating the chain.

***Editorial note.*** *Chain-level suite assertion is a deliberate simplification over per-segment assertion. Per-segment suites would allow each publisher to self-select cryptographic primitives, enabling per-publisher risk management. The trade-off is significantly more complex conformance testing, harder interoperability verification, and a more difficult debugging and audit surface — particularly when investigating a chain that has been signed by multiple publishers using different primitives. Chain-level assertion preserves a single cryptographic context per chain, which is the right default for v1. Per-segment crypto may be revisited in a future spec version if production demand emerges.*

### B.7.3 Acceptable Cryptographic Suites for SADAR v1

The following Chain Crypto Suites are conformant for SADAR v1. Implementations SHALL support SADAR-CRYPTO-1 (the reference suite) for both producing and consuming SCT chains. Support for additional suites is OPTIONAL.

|  |  |  |  |  |
| --- | --- | --- | --- | --- |
| **Suite ID** | **Signature** | **Encryption** | **Key Wrap** | **Hash** |
| SADAR-CRYPTO-1 (reference) | ES256 | A256GCM | ECDH-ES+A256KW | SHA-256 |
| SADAR-CRYPTO-2 | ES384 | A256GCM | ECDH-ES+A256KW | SHA-384 |
| SADAR-CRYPTO-3 | EdDSA (Ed25519) | A256GCM | ECDH-ES+A256KW | SHA-256 |
| SADAR-CRYPTO-4 | RS256 (≥3072 bit) | A256GCM | RSA-OAEP-256 | SHA-256 |

B.7.3.1 — Implementations SHALL support SADAR-CRYPTO-1 for both producing and consuming SCT chains. Support for additional suites is OPTIONAL but SHOULD be documented in the implementation's Conformance Declaration (Section B.7.5).

B.7.3.2 — An implementation MAY validate inbound SCT chains using a suite it does not support for production, provided it can verify signatures and chain integrity in that suite. Inability to produce chains in a given suite does not prevent validation of inbound chains in that suite, subject to acceptance-list policy per B.7.2.3.

B.7.3.3 — Additional suites MAY be added to this list in subsequent spec versions. Suites SHALL NOT be silently removed; deprecated suites are marked deprecated for at least one spec version before removal, with a documented migration path.

### B.7.4 Required Security Properties

The cryptographic primitives selected for any conformant SADAR implementation SHALL provide the following properties. Specific reference standards are provided as normative anchors for each property.

**Non-repudiation.** A receiver SHALL be able to demonstrate to a third party that a specific issuer signed a specific segment, where the issuer's public key is verifiable through their signed registry manifest. The required strength is digital signature non-repudiation per ISO/IEC 13888-3 (mechanisms using asymmetric techniques) with the issuer's private signing key bound exclusively to a single registered SADAR identity per Core Security Section A.1.4.

**Replay protection.** A receiver SHALL be able to detect reuse of a previously-seen SCT or segment within the validity window of the segment's exp claim. The required mechanism is per-segment unique jti (RFC 7519 §4.1.7) tracked by the receiver in a seen-set bounded by the segment's exp. This is the same pattern used for DPoP per RFC 9449. Combined with the segment's exp, this provides bounded replay detection without requiring a persistent global jti store.

**Confidentiality.** Only the intended JWE recipient(s) SHALL be able to decrypt segment payloads. The required strength is authenticated encryption with security level equivalent to AES-256-GCM (NIST SP 800-38D), with key material protected per FIPS 140-3 Level 1 or higher in conformant production deployments. Recipient targeting is established by JWE key wrap to the recipient's public key per RFC 7516.

### B.7.5 Conformance Declaration

B.7.5.1 — A conformant SADAR implementation SHALL publish a Conformance Declaration in its registry manifest. The Conformance Declaration is itself a verifiable artifact: it is included in the issuer's signed manifest, and any consumer or peer can retrieve it as part of standard discovery.

B.7.5.2 — The Conformance Declaration SHALL list, at minimum:

Supported Chain Crypto Suites for production

Supported Chain Crypto Suites for validation (where wider than production support)

The set of normative requirements from this appendix the implementation satisfies, with any documented exceptions

The reference implementation version (where applicable) used as the foundation

Conformance test suite results (Section B.8.5) where available

## B.8 Non-Normative Implementation Guidance

The following guidance is provided to assist implementers and is not normative. Any approach satisfying the requirements in Sections B.1 through B.7 is conformant.

### B.8.1 Validation Result Caching

Implementations may cache the result of Validate against the SCT segment's jti (JWT ID, RFC 7519 §4.1.7) and content hash for performance. Cache entries SHOULD be bounded by the segment's exp claim — cached results MUST NOT be used past segment expiry. Cache management policy below this ceiling (LRU eviction, capacity limits, refresh on demand) is at implementer's discretion. Cache invalidation on Append Segment is recommended: producing a new chain that includes the cached segment as ancestor does not invalidate the cache entry, since the cached segment's validity is unchanged.

### B.8.2 Idempotency of Validate and Verify Chain Integrity

Validate and Verify Chain Integrity are naturally idempotent: invoking either operation multiple times on the same input produces the same result, provided no key material has changed between invocations. Implementations SHOULD treat these operations as idempotent for caching, retry, and replay-safety purposes.

### B.8.3 Determinism of Append Segment

Append Segment is non-deterministic by design: the new segment carries a fresh jti and (typically) a fresh issued-at timestamp. Implementations MAY accept an optional caller-supplied jti for deterministic output (idempotency-token pattern) but SHALL ensure that any caller-supplied jti is unique within the appending party's issuance scope.

### B.8.4 Reference Implementation

The SADAR open-source reference implementation, published under Apache License 2.0, demonstrates these four operations idiomatically. Initial reference implementation language is Python; additional language implementations will be enumerated as they are released. Implementations in other languages MAY adopt the reference implementation's API patterns or define language-idiomatic alternatives, provided the normative requirements of Sections B.1 through B.7 are satisfied. The reference implementation is non-normative; it serves as a conformance demonstration and a starting point for new implementations.

### B.8.5 Conformance Test Suite

A conformance test suite covering the operations and error contract defined in this appendix will be published alongside the reference implementation. Implementations claiming conformance to this appendix SHOULD pass the conformance test suite. Conformance test results SHOULD be published in the implementation's Conformance Declaration per Section B.7.5.

### B.8.6 Post-Quantum Migration

NIST has standardized post-quantum cryptographic primitives (FIPS 203 ML-KEM and FIPS 204 ML-DSA, both finalized August 2024). Implementations preparing for post-quantum migration SHOULD plan for the addition of:

ML-DSA-65 or ML-DSA-87 (FIPS 204) as a signature primitive, replacing ES256/ES384/RS256

ML-KEM-768 or ML-KEM-1024 (FIPS 203) as a key encapsulation mechanism, replacing ECDH-ES key wrap

Hybrid suites combining classical and post-quantum primitives in parallel as a forward-compatible migration path during the transition period

Specific Chain Crypto Suite identifiers (Section B.7.3) for post-quantum and hybrid configurations will be added once the IETF JOSE working group finalizes JWS/JWE algorithm registrations for ML-DSA and ML-KEM. Implementations supporting an experimental post-quantum suite SHOULD document the IETF draft version they implement and the transitional nature of the support in their Conformance Declaration.

# Editorial Notes (Non-Normative)

Items below are drafting notes for the spec editor and Working Group; they are not part of the proposed normative text.

**On chain immutability framing**

The preamble paragraph at the top of Appendix B explains that chain immutability is a cryptographic property, not an API contract. This allows the four operations to be specified without redundant "SHALL NOT modify" requirements, which were dead weight in v1 and weakened the architectural framing.

**On Validate vs Verify Chain Integrity**

These two operations have overlapping but distinct purposes. Validate is about individual segment correctness (signature + structure); Verify Chain Integrity is about end-to-end chain coherence (signatures + parent-child bindings). An implementation may share code between them, but the operations are conceptually distinct and serve different conformance questions.

**On Extract Claims as the primary entry point**

Most production callers will use Extract Claims rather than Validate directly, because Extract Claims provides both validation and claim retrieval in a single call. Validate exists for cases where a caller wants to check chain validity without retrieving claims (e.g., a preflight check before deciding whether to proceed with a more expensive operation).

**On Chain Crypto Suite chain-wide vs per-segment**

Per-segment crypto allows each publisher to self-manage cryptographic risk but at significant cost: complex conformance testing, harder interoperability verification, difficult audit reasoning. Chain-level assertion is the right v1 default. Per-segment may be revisited in v2 if production demand emerges.

**On post-quantum migration timing**

NIST FIPS 203 and 204 were finalized August 2024. JOSE working group registrations for JWS/JWE post-quantum algorithms are still in draft as of April 2026. Section B.8.6 documents the migration path without committing to specific drafts. When JOSE finalizes, B.7.3 should add post-quantum suite identifiers.

**On Conformance Declaration as a verifiable artifact**

Section B.7.5 introduces the Conformance Declaration as part of the registry manifest. This makes conformance claims verifiable: any consumer or peer can retrieve the declaration and check supported suites and exception sets at discovery time. This is a defensive measure against vendors claiming conformance without supporting reality.

**On AGT mapping under v2**

Under B.7.2 (Chain Crypto Suite), AGT's Ed25519/DID substrate maps to SADAR-CRYPTO-3 (EdDSA + A256GCM + ECDH-ES+A256KW + SHA-256). With a JWKS shim translating between AGT's key publication and SADAR's expected JWKS format, the cryptographic substrate is conformant. Remaining gaps (originator-per-invocation, business process context) are content/structure gaps rather than cryptographic gaps. See matrix INC-6 and INC-7.