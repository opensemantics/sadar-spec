# SADAR Context Token — Structure

*Normative specification — Draft, June 2026*

*This document is the canonical definition of the SADAR Context Token (SCT) segment structure: the JWS-inside-JWE envelope, the complete claim schema, and the controlled enumerations. It is the structural source of truth referenced by 9. SCT Operations (which defines the operations over a chain) and SCT Propagation (which defines how the chain is carried). Registry Core Security §4.6 introduces the SCT at orientation depth and references this document for the full schema.*

*© 2026 Cognita AI Inc. All rights reserved. SADAR™ is a trademark of Cognita AI, Inc. CogniWeave™ is a trademark of Cognita AI, Inc.*

---

## 1. Scope and Conventions

### 1.1 Scope

This document specifies the internal structure of an SCT segment and of the SCT chain assembled from segments. It defines the cryptographic envelope, the claim set, the type and cardinality of each claim, which claims are signed-in-the-clear versus encrypted, and the SADAR-controlled IRI enumerations. It does **not** define the operations performed over a chain (see 9. SCT Operations), nor how the chain is transported or held between hops (see SCT Propagation), nor the authentication layer the SCT accompanies (see Core Security §4.x).

### 1.2 Relationship to the SCT document set

| Document | Defines |
|---|---|
| **SCT Structure** (this document) | The segment envelope and claim schema — the structural source of truth |
| **9. SCT Operations** | The operations over a chain: Initiate Chain, Validate, Extract Claims, Append Segment, Verify Chain Integrity |
| **SCT Propagation** | Transport (`SADAR-SCT` header), cross-hop custody, session-store security, and cross-boundary crossing |
| **Core Security §4.x** | The authentication layer (OIDC usage token, first-use provisioning, DPoP) the SCT accompanies; §4.6 introduces the SCT and references this document |

### 1.3 Conventions

The key words MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT, RECOMMENDED, MAY, and OPTIONAL are to be interpreted as described in RFC 2119 and RFC 8174 when, and only when, they appear in all capitals. Claim names and IRIs appear in regular type in prose and unmarked in tables. Code-style monospace is reserved for the rendered Web version of this document.

---

## 2. Overview

The OIDC usage token obtained during first-use provisioning establishes authentication at the configured `credential_scope` granularity — *who* is permitted to call a service at the infrastructure level. It does not carry the context of *what* is being requested, *why*, *on whose authority*, and *as part of which business process*. That context is the SCT.

The SCT is a per-invocation authorization-context token constructed and signed by searchAndInvoke and transmitted alongside the authentication token on every invocation (see SCT Propagation for transport). The separation between the authentication layer (usage token) and the authorization-context layer (SCT) is intentional: the authentication layer is stable across an invocation session and represents a registered trust relationship; the SCT is specific to each invocation and represents the runtime assertion of what is being done. Keeping them separate prevents the authentication credential from being overloaded with policy context that varies per call, and lets the `credential_scope` mapping operate independently of the authorization policy enforced at each service.

An SCT **chain** is the ordered sequence of SCT **segments** produced as an invocation flow proceeds across hops. Each segment is independently signed by its issuing actor and independently encrypted to its receiving party; segments are linked into a chain by the `parent_sct_jti` claim. The chain is the chain-of-custody record for the entire flow.

---

## 3. Segment Envelope

### 3.1 JWS-inside-JWE

Each SCT segment SHALL be structured as a **JWS-inside-JWE**:

- an **inner JWS**, signed by the issuing actor (the entity whose action the segment attests; see the signing invariant in SCT Propagation §6), providing integrity and non-repudiation; and
- an **outer JWE**, encrypted to the receiving party's public key, providing confidentiality of carried context across the hop.

The two properties are independent: chain integrity (verifiable from public signing keys) does not require decryption, and confidentiality (one recipient per segment) does not weaken integrity. A party able to verify a chain's integrity is not thereby able to read segments not encrypted to it.

### 3.2 Signed-clear versus encrypted claims

Certain claims are carried **in the clear** (inside the signed JWS but outside any additional encryption, so they are integrity-protected and signer-attributable but readable by any party that can verify the signature) because downstream parties and auditors must read them without holding the segment's decryption key. All other claims are carried within the encrypted payload. Section 5 marks each claim's disposition in the **Carriage** column (`signed-clear` or `encrypted`).

### 3.3 Chain Crypto Suite

The cryptographic algorithms (signature, key-wrap, content-encryption) used across a chain are governed by a **Chain Crypto Suite** asserted at chain initiation (Initiate Chain) and bound under the root segment's signature. The suite SHALL govern the entire chain; Append Segment SHALL NOT alter it. The enumerated suites and their binding are normatively defined in 9. SCT Operations §B.7; this document references that definition rather than restating it.

### 3.4 Chain linkage

Segments are linked by `parent_sct_jti` (§5). The **root** segment carries no `parent_sct_jti` — its absence is the structural marker of a chain root. Every non-root segment SHALL carry `parent_sct_jti` referencing the `jti` of the immediately preceding segment. A segment lacking `parent_sct_jti` at any position other than the root renders the chain structurally invalid.

---

## 4. Standard JWS Header and Identity Claims

Each segment is a JWS and carries the standard registered claims required for verification and replay protection:

| Claim | Type | Cardinality | Carriage | Description |
|---|---|---|---|---|
| `jti` | string | MUST | signed-clear | Unique identifier for this segment. The basis for `parent_sct_jti` linkage and for the replay seen-set (§6). |
| `iss` | URN | MUST | signed-clear | The issuing actor's registered SADAR identity (the entity whose action this segment attests). Determines the signing key. |
| `iat` | numeric date | MUST | signed-clear | Issued-at time. |
| `exp` | numeric date | MUST | signed-clear | Expiry. Bounds the replay seen-set window and the maximum lifetime of the segment. |

> The `iss` is the *actor* of the segment, not necessarily searchAndInvoke. Per the signing invariant (SCT Propagation §6), an execution segment is issued by the executing component, a routing segment by the home registry, and so on. searchAndInvoke performs the signing mechanically using the actor's key.

---

## 5. SCT Claim Schema

The following claims comprise the SCT authorization-context payload. **Required** is relative to an ordinary execution segment unless otherwise noted.

| Claim | Type | Required | Carriage | Description |
|---|---|---|---|---|
| `agent_id` | URN | MUST | encrypted | The specific agent registry URN — the actual caller — regardless of the `credential_scope` mapping. The service always knows which agent is calling even when the authentication token presents a broader scope. |
| `originating_user` | structure | MUST | encrypted | The identity of the human user whose session initiated the invocation chain. Either a directly authenticated user JWT or an asserted user claim; carries a flag indicating which trust model applies (see Trust Models). |
| `business_process_id` | IRI | SHOULD | encrypted | The SADAR-registered identifier of the business process under which this invocation occurs. Absent for invocations not bound to a registered process. |
| `agent_step` | string | SHOULD | encrypted | The step identifier within the business process at which this invocation occurs. Used by services enforcing step-level operation constraints. |
| `authorized_operations` | list of IRIs | MUST | encrypted | The set of operation types this invocation is authorized to request, as declared in the agent's manifest for this step. This is the `actions` component of `authority` (below). |
| `parent_sct_jti` | string | Conditional | signed-clear | The `jti` of the parent segment in the chain. **MUST be absent on the root segment** and **MUST be present on every non-root segment** (§3.4). Carried in the clear so chain integrity is verifiable without decryption. |
| `risk_score_adjustment` | structure | MUST | signed-clear | The step's net contribution to the risk score, as `{ delta, reason }`. `delta` is a decimal in [-1.0, 1.0]; `reason` is an IRI from `urn:sadar:risk_reason:v1:*` (§6.4). Free-form rationale is recorded only in OTel telemetry, never in this clear claim. See 13. Risk Score Specification. |
| `step_status` | structure | MUST | signed-clear | The terminal outcome the step reports, as `{ status, reason, description }`, mandatory on every segment. See §6.2–6.3 for the enumerations and the `description` sanitization rule. The step reports; the enforcement layer decides. |
| `intent_instance_id` | string | MUST | signed-clear | The stable identifier of this process instance. Seeded from the root TraceID at flow start, propagated by value, **immutable end-to-end and identical across trust boundaries**. The join key correlating repatriated telemetry to the originating flow. Distinct from the local TraceID (which may legitimately differ in a remote segment) and SHALL NOT be re-read from the ambient trace downstream. |
| `authority` | structure (RFC 9396) | MUST | encrypted | The scope of authority for this invocation, structured as RFC 9396 `authorization_details` with SADAR type `urn:sadar:authority:v1:` and components `actions`, `datatypes`, `locations`, `privileges`. SADAR does not set, compute, interpret, or attenuate this authority — an IAM/policy/authorization server establishes it and the enforcement layer checks it; the SCT propagates it, signed. Authority always travels in the SCT regardless of how the caller authenticated. |
| `cnf` | structure | MUST | signed-clear | A confirmation (proof-of-possession) binding to the authentication credential presented on this invocation (DPoP key thumbprint, SPIFFE SVID, or mTLS client certificate), so that a captured SCT cannot be replayed under a different caller. Cryptographically ties authentication (the auth token) to authority (this SCT). The primary anti-hijack binding referenced by SCT Propagation §4. |
| `segment_action` | IRI | MUST | signed-clear | The action this segment records, from `urn:sadar:segment_action:v1:*` (§6.5). Makes the role of the segment explicit rather than inferred, and disambiguates the registry-signed segment kinds (`routed` vs `executed_nonconformant`). |

### 5.1 Structured-claim shapes

**`originating_user`** — `{ identity, trust_model }`, where `identity` is a user JWT (direct authentication) or an asserted user reference, and `trust_model` is the applicable trust-model flag (see Trust Models: direct authentication, assertion, impersonation, deputy).

**`risk_score_adjustment`** — `{ delta, reason }`; `delta` ∈ [-1.0, 1.0]; `reason` ∈ `urn:sadar:risk_reason:v1:*`.

**`step_status`** — `{ status, reason, description }`; `status` ∈ `urn:sadar:step_status:v1:*` (§6.2, default `success`); `reason` ∈ `urn:sadar:status_reason:v1:*` (required for any non-clean outcome); `description` per §6.3 (sanitized, display-only).

**`authority`** — RFC 9396 `authorization_details`, SADAR type `urn:sadar:authority:v1:`, with components `actions` (operations — identical to `authorized_operations`), `datatypes` (data scope), `locations` (targets and resources), `privileges` (level).

**`cnf`** — a proof-of-possession confirmation per the established `cnf` convention (e.g., `{ jkt }` for a DPoP key thumbprint), or the equivalent binding for a SPIFFE SVID or mTLS client certificate.

---

## 6. Controlled Enumerations

All SADAR-controlled enumerations are IRI-valued in a versioned `urn:sadar:*:v1:*` namespace. Implementations MAY define custom values in an organization-controlled namespace where a claim's definition permits it. Implementations SHALL ignore enumeration IRIs they do not recognize for control purposes while preserving them for audit.

### 6.1 Namespaces

| Namespace | Claim |
|---|---|
| `urn:sadar:step_status:v1:*` | `step_status.status` |
| `urn:sadar:status_reason:v1:*` | `step_status.reason` |
| `urn:sadar:risk_reason:v1:*` | `risk_score_adjustment.reason` |
| `urn:sadar:authority:v1:*` | `authority` (RFC 9396 type) |
| `urn:sadar:segment_action:v1:*` | `segment_action` |

### 6.2 `step_status.status`

`success` (default), `success_with_information`, `success_with_warnings`, `partial_success`, `failure`, `failure_compensated`, `failure_uncompensated`, `cancelled`. `step_status.reason` (from `urn:sadar:status_reason:v1:*`) is REQUIRED for any non-clean outcome.

### 6.3 `step_status.description` sanitization

`description` is sanitized free text: ≤ 256 characters; the permitted set is `[A-Za-z0-9]`, space, and `. , - _ :` only. Markup, quotes, brackets, URLs, and control characters are stripped on ingestion. It is display-only and SHALL NOT be parsed or used in any control decision. Longer or sensitive narrative is recorded in OTel telemetry (where it may be encrypted), never in this clear claim.

### 6.4 `risk_score_adjustment.reason`

An IRI normatively from `urn:sadar:risk_reason:v1:*`; custom reasons MAY be carried in an organization-controlled namespace. The accumulated scalar risk score is carried in OTel baggage as a convenience and is recomputable from the signed chain; baggage is not authoritative (see SCT Propagation §5.4 and 13. Risk Score Specification).

### 6.5 `segment_action`

| IRI | Meaning | Typical signer |
|---|---|---|
| `urn:sadar:segment_action:v1:executed` | A component performed its operation; the default for an ordinary execution segment. | The executing component's key |
| `urn:sadar:segment_action:v1:routed` | A registry gateway received an inbound cross-boundary call and routed it internally. | The home registry's key |
| `urn:sadar:segment_action:v1:executed_nonconformant` | searchAndInvoke executed a registry-defined operation against a non-conformant agent, tool, or resource that cannot itself sign. | The home registry's key (on the registry's behalf) |

The enumeration is extensible; additional actions MAY be added in subsequent versions. See SCT Propagation §7 (boundary crossing) and §7.8 (conformance) for the semantics of `routed` and `executed_nonconformant`.

---

## 7. Required-Claim Matrix by Segment Kind

The base schema (§5) is relative to an ordinary execution segment. Segment kinds differ in a small number of claims:

| Claim | Root (genesis) | Execution | Routing (`routed`) | Non-conformant (`executed_nonconformant`) |
|---|---|---|---|---|
| `parent_sct_jti` | MUST be absent | MUST be present | MUST be present | MUST be present |
| `segment_action` | `executed` (or kind-specific) | `executed` | `routed` | `executed_nonconformant` |
| `iss` (signer) | opening identity | the component | the home registry | the home registry |
| `cnf` | MUST | MUST | MUST | MUST |
| `step_status` | MUST | MUST | MUST | MUST |
| all other §5 claims | per §5 | per §5 | per §5 | per §5 |

Genesis (root) segment construction is the **Initiate Chain** operation (9. SCT Operations §B.0); the root is distinguished structurally by the absence of `parent_sct_jti`.

---

## 8. Conformance

A segment is structurally conformant when:

- It is a well-formed JWS-inside-JWE per §3, signed by the actor named in `iss` and verifiable against that identity's published key, and encrypted to the intended recipient.
- All claims marked MUST in §5 (subject to the per-kind matrix in §7) are present and well-formed; structured claims conform to §5.1; enumerated claims carry IRIs from the namespaces in §6.
- `parent_sct_jti` is absent on the root and present on every non-root segment (§3.4).
- Signed-clear claims (§3.2, §5 Carriage column) are carried in the clear and are integrity-protected by the segment signature.
- The Chain Crypto Suite asserted at the root governs the segment (§3.3).

A chain is structurally conformant when every segment is conformant, the linkage is unbroken from root to tail via `parent_sct_jti`, and Verify Chain Integrity (9. SCT Operations §B.5) returns VALID.

---

## 9. Open Items

| # | Item |
|---|---|
| 1 | **Canonical serialization.** This document specifies the logical structure and carriage of each claim but defers the exact canonical JSON serialization (member ordering, encoding) used for signing to 9. SCT Operations §B.7 (Chain Crypto Suite). Confirm whether the canonicalization rule should be stated here as the structural source of truth, or remain in Operations. |
| 2 | **`business_process_id` / `agent_step` cardinality.** Marked SHOULD (absent when not process-bound). Confirm whether a registered-process invocation makes them MUST conditionally. |
| 3 | **`originating_user.identity` form.** Confirm the normative representations (user JWT vs asserted reference) and whether the asserted form carries its own signature/vouching structure. |
| 4 | **Core Security §4.6 reduction.** With this document canonical, §4.6 should be reduced to an orientation introduction plus a pointer here, to avoid schema drift between two documents. Confirm the reduction (and the version bump that accompanies it). |
