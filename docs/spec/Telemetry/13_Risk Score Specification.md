# SADAR™ Risk Score Specification

*Specification — Version 1.0.0*

Status: Draft • Tracking ID: C21

*April 2026*

*© 2026 Cognita AI Inc. All rights reserved. SADAR™ is a trademark of Cognita AI, Inc. CogniWeave™ is a trademark of Cognita AI, Inc. All other trademarks are the property of their respective owners.*

## Document Overview

This specification defines the SADAR Risk Score: a stateful, cumulative measure of the operational risk associated with an in-flight process flow. The score travels with the flow as it crosses agents, tools, resources, and trust boundaries, and is adjusted up or down by participants along the way as risk-relevant events occur.

The Risk Score is intended to provide a substrate for run-time risk-aware decision-making — interventions, escalations, guardrail invocations, audit triggers — while leaving the interpretation of the score, the intervention policies, and any weighted recomputation to the consuming enforcement layer and end-customer programs. SADAR defines the data model, the propagation pattern, the single deterministic accumulation, and the conformance obligations; SADAR does NOT define what specific score values mean for specific operations, what intervention thresholds apply, or how the scalar is weighted or acted upon. Those are the responsibility of the enforcement layer and user program, observed and tuned by use case, customer, and domain.

**Cross-references.** C28 v2 §5.1.X (Cryptographic Parity — the Risk Score participates in the parity model between Baggage and SCT). C19 v1.0 §10 (NFR Schema — impact\_score field added in §11 of this document). C20 v5 §5.1.14.X.2 (Telemetry Record — risk adjustments visible in audit trail per the v5.2 back-port note in §10). C2 v2 §B.7 (SCT structure — the per-segment risk\_adjustment claim defined in §5.1).

**Informative reference.** NIST SP 800-30 Revision 1, Appendix H (Impact). The framing for thinking about the magnitude of negative impact draws on Appendix H's qualitative impact scales as a starting point. SADAR does not adopt SP 800-30 normatively; the appendix is referenced as best-practice framing for organizations defining their own impact mappings.

— — —

# 1\. Introduction

## 1.1 Purpose

1.1.1 — This specification defines a normative substrate for risk-aware run-time control across SADAR-conformant agent flows. The substrate provides a propagation channel for incremental risk observations, a data model for those observations, and obligations on conformant implementations to accept and propagate them.

1.1.2 — The substrate exists because the deterministic-to-probabilistic shift in agentic systems introduces cumulative risk effects that no single component can detect or mitigate alone. A flow that touches sensitive data, then exhibits anomalous output, then crosses a trust boundary, then chains into another high-impact action — is, considered as a whole, riskier than the sum of its parts considered individually. The Risk Score gives flows a place to record those cumulative observations and gives governance layers a place to read them.

## 1.2 Scope

1.2.1 — In scope. This specification defines: the Risk Score data model (a current scalar in 0.0–1.0); the Risk Score Adjustment data structure (deltas, reasons, TTLs, optional decay); the starter reason enumeration; the propagation pattern (signed per-step Adjustments carried in the SCT chain, with the accumulated scalar carried in Baggage/OTel as a convenience); the single deterministic accumulation that produces the scalar; the parity requirement between the convenience scalar and the scalar recomputed from the signed chain; implementer obligations to accept and propagate adjustments; the optional impact\_score NFR for entries; integration with the Telemetry Record audit trail.

1.2.2 — Out of scope. This specification does NOT define: the specific score values that constitute "risky" in any given domain; the interpretation, weighting, or risk-sensitivity calibration applied to the accumulated scalar; intervention thresholds; the policy-driven recomputation an enforcement layer may perform over the Adjustment set; the specific mitigation and detection techniques an agent or guardrail may apply (such techniques are implementer-proprietary and may include policy-driven mitigations, drift-vs-baseline checks, attestation gating, demographic-bias detection, hallucination detection, and others); intervention orchestration logic. These concerns are user-program and enforcement-layer responsibility, tuned over time by use case, customer, and domain.

## 1.3 Normative Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 / RFC 8174 when, and only when, they appear in all capitals.

— — —

# 2\. Conceptual Model

## 2.1 The Score is Cumulative and In-Flight

2.1.1 — The Risk Score is a property of an in-flight process flow, not a static manifest attribute. It begins at zero when the flow starts. Participants in the flow — agents, tools, resources, guardrails, policy enforcement points — may issue Risk Score Adjustments that influence the score. The score travels with the flow as it progresses across components and trust boundaries.

2.1.2 — The score is a real number in the closed interval \[0.0, 1.0]. 0.0 represents no accumulated risk; 1.0 represents maximum accumulated risk. The granularity of values within the interval is implementation choice; the spec does not normatively discretize the scale.

2.1.3 — The score is the result of a single deterministic accumulation applied to the set of currently-active Risk Score Adjustments (§3) at the moment of evaluation. The accumulation is defined in §6; the same active Adjustment set yields the same scalar in every conformant implementation. Domain-specific interpretation of that scalar — weighting, sensitivity, and the decision of what a value means for a given operation — is performed above SADAR by the consuming enforcement layer, not by the accumulation.

## 2.2 Adjustments Are Net Step Contributions

2.2.1 — An agent, tool, resource, guardrail, or policy enforcement point that emits a Risk Score Adjustment SHALL emit the net result of any mitigations or compensating analyses it has applied. The Adjustment is the residual the step is contributing to the flow's ongoing risk picture, not a raw pre-mitigation observation.

2.2.2 — A net Adjustment MAY be positive (the step observed something that increases risk) or negative (the step observed something that decreases risk, such as a clean policy check, an attestation that verified, or a quality signal exceeding a baseline). Mitigation may reduce a positive raw observation to a smaller positive Adjustment, to zero, or even to a negative value if the mitigation provides positive evidence beyond the observed risk.

2.2.3 — The mitigation logic itself is internal to the step. The spec provides no schema or vocabulary for describing what mitigation was applied; the Adjustment's context field (§3) MAY be used to convey relevant detail in free-form text for audit and human review.

## 2.3 The Aggregate is Floor-Clamped

2.3.1 — While individual Adjustments may be negative and intermediate algorithm computations may produce values below 0.0, the reported aggregate score SHALL be clamped to ≥ 0.0. The floor of zero prevents "credit accumulation" — the perverse pattern where a flow banks negative adjustments against future risk-taking — and aligns with the conceptual reality that there is no state better than "no accumulated risk."

2.3.2 — The ceiling of 1.0 is an absorbing state in the sense that a flow at 1.0 represents maximum accumulated risk. Further positive Adjustments do not increase the score above 1.0; the aggregate SHALL be clamped to ≤ 1.0. Negative Adjustments may reduce a flow from 1.0 toward 0.0 per the accumulation.

## 2.4 Two Layers of Propagation

2.4.1 — The Risk Score propagates in two layers, each serving a different operational need:

||||
|-|-|-|
|**Layer**|**Lifecycle**|**Carries**|
|SCT chain|Cryptographically attested, immutable, append-only|The signed per-step Adjustments themselves — each Adjustment's delta and reason code, carried in the clear in the chain segment of the step that emitted it and signed by that step. This is the authoritative, attributable record of which step contributed what, and why.|
|Baggage / OTel|Live, in-flight, mutable|The accumulated scalar — the current score in \[0.0, 1.0] computed by the deterministic accumulation (§6). Carried as a convenience so a component can read the current score without walking the chain.|

2.4.2 — Why this split. The Adjustments-with-reasons are the evidence and must be tamper-evident and attributable: an audit must be able to prove that a specific step raised risk by a specific amount for a specific reason. Only the SCT provides this — each segment's JWS signature binds the Adjustment and its reason code to the issuing step. The scalar is a derivation of that evidence, not evidence itself; it is carried in Baggage/OTel purely so components can read the current score cheaply.

2.4.3 — Why the scalar is not the trusted value. Because the accumulation is deterministic (§6), any authorized party can walk the signed chain and recompute the exact scalar. The Baggage scalar is therefore a cache with a proof always available: a component may rely on it for convenience, or recompute from the chain when it needs assurance. Tampering with the Baggage scalar misleads only a component that chose not to verify, and that component can always verify.

2.4.4 — Runtime authority. The signed SCT chain is the source of truth for the Adjustments and for any value derived from them. A reason-aware consumer (typically the enforcement layer) walks the chain; lightweight components that only need the current number read the Baggage scalar, with recomputation from the chain available at will. At every trust boundary crossing where a new SCT chain segment is issued, the issuer SHALL carry its Adjustment, if any, as a signed claim in that segment, and SHALL compute the current scalar by the deterministic accumulation of §6 for carriage in Baggage/OTel.

2.4.5 — Confidential detail. The Adjustment carried in the SCT is in the clear — signed, not encrypted — and is limited to its structured fields and SADAR-defined reason code. Any free-form rationale a step wishes to record is written to the OTel telemetry, where it may be encrypted to the step's chosen recipients; it is never placed in the clear SCT claim.

## 2.5 Parity

2.5.1 — Cross-layer parity. The risk score is a parity instance per C28 v2 §5.1.X. The scalar carried in Baggage/OTel SHALL equal the scalar obtained by applying the deterministic accumulation of §6 to the currently-active Adjustments in the signed SCT chain. Discrepancy is a parity violation and is reported per C28 v2 §5.1.X.2.5.

2.5.2 — Universal verifiability. Because the accumulation is a single deterministic operation defined by this specification, any party authorized to read the SCT chain can recompute the scalar and check parity without knowing anything implementer-specific. There is no issuer-private algorithm to reconcile: the SCT chain is authoritative, and the Baggage scalar either matches the recomputation or is in violation.

— — —

# 3\. Risk Score Adjustment Schema

## 3.1 Structure

A Risk Score Adjustment is a structured object with the following fields:

{
"adjustment\_id": "urn:uuid:...",
"delta": -0.15,
"ttl\_seconds": 3600,
"timestamp": "2026-04-26T22:15:00Z",
"decay": { ... }, // optional
"reason": "urn:sadar:risk\_reason:v1:bias\_detection",
"context": "Demographic bias detected in agent response above 0.3 threshold; mitigation applied."
}

## 3.2 Field Definitions

||||
|-|-|-|
|**Field**|**Type**|**Definition**|
|adjustment\_id|string (URN)|Globally unique identifier for this Adjustment. Implementations MAY use UUIDs, content-addressed identifiers, or other URN forms. Required. Used for deduplication if the Adjustment is observed by multiple downstream consumers.|
|delta|decimal|Net contribution this step is making to the flow's ongoing risk picture, after the step's own mitigations. Range: \[-1.0, 1.0]. Positive values increase risk; negative values decrease risk; zero is permitted (the step ran but observed nothing notable). Required.|
|ttl\_seconds|integer ≥ 0|How long this Adjustment remains in effect, in seconds from timestamp. After TTL expiry, the Adjustment SHALL be excluded from accumulation. ttl\_seconds = 0 means the Adjustment expires immediately at issuance and is informational only. Required.|
|timestamp|RFC 3339 string|When the Adjustment was issued. Required.|
|decay|object|Optional decay parameters. Schema is implementer-extensible per §7. Implementations that do not consume the decay field SHALL pass it through unchanged.|
|reason|string (IRI)|IRI identifying the category of observation prompting this Adjustment. SHOULD be a value from the SADAR starter enumeration (§4) or an extension URN in an organization-controlled namespace. Required.|
|context|string|Free-form human-readable detail (specific observation, mitigation applied, additional audit detail). Recorded in the Telemetry Record (§10), where it MAY be encrypted to the issuer's chosen recipients. It is NOT carried in the in-the-clear SCT Adjustment claim (§5.2). SHOULD be present; concise but informative.|

## 3.3 Field Constraints

3.3.1 — Delta range. delta SHALL be in \[-1.0, 1.0]. A delta outside this range is malformed and SHALL be rejected at consumption time per §8.4.

3.3.2 — Timestamp ordering. Adjustments SHOULD be issued with monotonically non-decreasing timestamps within a flow. Receivers MAY order Adjustments by timestamp for accumulation purposes; implementations MAY use other ordering (insertion order, or by adjustment\_id) for deterministic replay.

3.3.3 — TTL minimum. While ttl\_seconds = 0 is permitted, implementations SHOULD use a non-zero TTL when an Adjustment is intended to influence accumulation downstream. Zero-TTL Adjustments are recorded for audit but contribute nothing to the live aggregate.

3.3.4 — Reason completeness. The reason field SHALL be a well-formed IRI. Receivers encountering an unrecognized reason IRI SHALL preserve the Adjustment in the chain and pass it through. The deterministic accumulation (§6) includes the Adjustment's delta regardless of whether its reason IRI is recognized; any reason-specific weighting is enforcement-layer interpretation (§6.2).

— — —

# 4\. Reason Enumeration

## 4.1 Namespace

4.1.1 — SADAR-defined reason values follow:

urn:sadar:risk\_reason:v1:{category}

The pattern parallels urn:sadar:nfr:v1:{category}:{attribute} (C19), urn:sadar:error:v1:{category}:{code} (C31), and urn:sadar:scope:v1:{category}:{operation} (C33). The risk\_reason category occupies a flat single-segment structure for v1; sub-categories MAY be added in future versions if needed.

4.1.2 — Custom reasons. Implementations and ecosystem participants MAY define custom reason IRIs in their own namespaces. Custom reasons SHALL NOT use the urn:sadar: prefix.

## 4.2 Starter Enumeration

4.2.1 — The following values are normatively defined for v1. The enumeration is deliberately small and incomplete; ecosystem participants are expected to extend with reasons relevant to their domains.

|||
|-|-|
|**Reason IRI**|**Description**|
|urn:sadar:risk\_reason:v1:sensitive\_data\_access|The step accessed data classified as sensitive (PII, PHI, financial, regulated). Use when the act of access itself is the risk-relevant event.|
|urn:sadar:risk\_reason:v1:state\_change|The step performed a state-changing operation (write, update, delete, send). Use when the change has potential downstream business impact.|
|urn:sadar:risk\_reason:v1:external\_invocation|The step invoked an external service outside the local trust boundary. Use when the invocation introduces dependency on a less-trusted system.|
|urn:sadar:risk\_reason:v1:policy\_violation|A guardrail or policy enforcement point detected a policy violation in the step's inputs, behavior, or outputs.|
|urn:sadar:risk\_reason:v1:policy\_check\_passed|A guardrail or policy enforcement point ran cleanly. Typically used with a negative delta to credit the flow for clean policy gates.|
|urn:sadar:risk\_reason:v1:attestation\_anomaly|An attestation check produced an anomalous result — inconsistent provenance, unverifiable signature, unexpected identity, or similar.|
|urn:sadar:risk\_reason:v1:attestation\_verified|An attestation check verified cleanly. Typically used with a negative delta to credit the flow for verified provenance.|
|urn:sadar:risk\_reason:v1:bias\_detection|An analysis detected demographic bias, fairness anomalies, or similar in agent output.|
|urn:sadar:risk\_reason:v1:hallucination\_detection|An analysis detected likely hallucination — output not supported by inputs, retrieval results, or grounding sources.|
|urn:sadar:risk\_reason:v1:divergence\_detection|An analysis detected divergence from expected baseline — output drift, anomalous pattern, deviation from labeled-good cluster, or similar.|
|urn:sadar:risk\_reason:v1:human\_intervention|A human intervened in the flow. May be positive (review approved) or negative (review flagged); context field clarifies.|
|urn:sadar:risk\_reason:v1:other|Reason not categorized by other values in this enumeration. Implementations SHOULD prefer specific custom IRIs in their own namespace over this catch-all.|

4.2.2 — Future versions. New reasons added to v1 are MINOR increments (backward compatible). Removal of reasons is a MAJOR increment. Receivers SHALL preserve unknown reason values; the absence of forward-compatible behavior is an interoperability failure.

## 4.3 Reason vs. Domain Vocabulary

4.3.1 — The starter enumeration is intentionally domain-neutral. Industry-specific reasons (e.g., "phi\_access\_under\_least\_privilege" for healthcare, or "sox\_significant\_change" for financial reporting) belong in domain-specific vocabularies in organization or industry namespaces, not in the SADAR starter set. The starter set provides the categorical anchors; specialization is downstream.

4.3.2 — Multiple reasons per Adjustment. The schema permits exactly one reason per Adjustment. Where a single observation reflects multiple reason categories, the issuer chooses the most-specific applicable reason and conveys additional category detail in the context field. Issuing multiple Adjustments for a single observation is an option but produces fragmented audit trails and is discouraged.

— — —

# 5\. Score Propagation

## 5.1 SCT Chain Carriage (Adjustments)

5.1.1 — The authoritative carrier of Risk Score Adjustments is the SCT chain. When a step emits an Adjustment, it SHALL include that Adjustment as a signed claim in the SCT chain segment it issues. The claim carries the Adjustment's structured fields — delta, reason, timestamp, ttl\_seconds, optional decay, and adjustment\_id — in the clear (signed, not encrypted), so that any party able to verify the chain can read and attribute the Adjustment.

5.1.2 — The per-segment Adjustment claim is risk\_adjustment. Its structure and placement within the SCT segment are defined in the SCT structure specification (Core Security §4.6); the value conforms to the Adjustment schema of §3 of this document, excluding the context field (§5.2.3).

5.1.3 — Immutability. The SCT chain is append-only and immutable: a segment's Adjustment, once signed, cannot be altered or removed without invalidating the chain. A step SHALL NOT modify or remove an Adjustment emitted by any other step. The chain therefore retains the complete history of Adjustments for the life of the flow, including Adjustments whose TTL has since expired.

5.1.4 — Active set. TTL does not remove an Adjustment from the chain; it determines whether the Adjustment contributes to the current scalar. An Adjustment is active at evaluation time T if T is at or before timestamp + ttl\_seconds, and inactive (expired) otherwise. Expired Adjustments remain in the chain as immutable history but are excluded from accumulation (§6).

## 5.2 Convenience Scalar Carriage (Baggage / OTel)

5.2.1 — The accumulated scalar propagates as a Baggage value for convenience. The Baggage key SHALL be:

urn:sadar:baggage:v1:risk\_score

5.2.2 — The value SHALL be a JSON number in \[0.0, 1.0]: the scalar computed by the deterministic accumulation (§6) over the currently-active Adjustments in the SCT chain at the time the value is written. The scalar is advisory — a component MAY rely on it or MAY recompute from the chain; it is not a trusted input and carries no signature of its own.

5.2.3 — Confidential detail. The free-form context of an Adjustment (§3) is NOT carried in the clear SCT claim. It is written to the Telemetry Record (§10), where the issuer MAY encrypt it. This keeps the provable, attributable record — delta and reason — universally readable while allowing sensitive rationale to remain confidential to authorized recipients.

## 5.3 Segment Issuance Algorithm

5.3.1 — At a trust boundary crossing where a new SCT chain segment is issued, the issuer SHALL:

1. If the issuing step has an Adjustment to contribute, construct the risk\_adjustment claim from the Adjustment's structured fields (§5.1.1) and include it in the new segment.
2. Sign the segment per the SCT structure specification, binding the Adjustment to the issuing step.
3. Determine the currently-active Adjustment set by walking the chain (including the segment just issued) and excluding expired Adjustments per §5.1.4.
4. Apply the deterministic accumulation (§6) to the active set and clamp the result to \[0.0, 1.0] per §2.3.
5. Write the clamped scalar to Baggage under urn:sadar:baggage:v1:risk\_score.

5.3.2 — No chain pruning. Issuance never removes prior Adjustments from the chain. Expired Adjustments are excluded from the active-set determination in step 3 but remain in the immutable record.

## 5.4 Cross-Layer Parity

5.4.1 — Per §2.5, the scalar in Baggage SHALL equal the result of applying the deterministic accumulation (§6) to the currently-active Adjustments in the SCT chain. Any party authorized to read the chain can recompute and verify this exactly. Parity violations are reported per C28 v2 §5.1.X with a parity-specific error code.

— — —

# 6\. Accumulation

## 6.1 The Accumulation Is Deterministic and Defined

6.1.1 — The accumulation — the function from the currently-active Adjustment set to a single scalar in \[0.0, 1.0] — is defined by this specification and is identical in every conformant implementation. It is the sum of the delta values of the currently-active (non-expired, §5.1.4) Adjustments, floor-clamped at 0.0 and ceiling-clamped at 1.0 per §2.3:

score = clamp( Σ active deltas, 0.0, 1.0 )

6.1.2 — Determinism. Applied to the same active Adjustment set, the accumulation yields the same scalar everywhere. This is what makes the Baggage scalar a verifiable cache of the signed chain (§2.4.3) and makes cross-layer parity an exact check any authorized party can perform (§2.5).

6.1.3 — SADAR performs this accumulation as a propagation convenience, not as a risk judgment. It applies no reason-specific weighting, no time discounting beyond TTL membership, and no domain sensitivity. Those are interpretation, and interpretation is out of scope (§6.2).

## 6.2 Interpretation Is the Enforcement Layer's Concern

6.2.1 — The meaning of the scalar, and any richer computation over the Adjustment set, belong to the consuming enforcement layer, not to SADAR. Reason-specific weighting, exponentially-weighted or time-discounted accumulation, Bayesian updating, predictive or learned models, and hybrid approaches are all interpretation methods an enforcement layer MAY apply over the same Adjustments. They are implementer-proprietary and are out of scope for this specification.

6.2.2 — Why interpretation is separated from accumulation. Different domains warrant different sensitivity — healthcare flows touching PHI may warrant rapid escalation; routine business flows may warrant conservative accumulation requiring multiple reinforcing observations. A single normative interpretation cannot serve both, so interpretation is left to the enforcement layer. The simple, deterministic accumulation is standardized for a different purpose: so the scalar carried on the wire is universally provable from the signed chain, independent of any layer's interpretation.

6.2.3 — Enforcement-layer participation. An enforcement layer that wishes to influence the propagated score does so by emitting its own Adjustment (§2.2, §8) after its policy evaluation or mitigation; that Adjustment enters the signed chain and the same deterministic accumulation like any other. It SHALL NOT alter another participant's Adjustment, and its internal weighted interpretation is never carried as the Baggage scalar.

— — —

# 7\. TTL and Decay

## 7.1 TTL is Normative

7.1.1 — Every Adjustment carries a ttl\_seconds (§3). At timestamp + ttl\_seconds, the Adjustment is considered expired. Implementations SHALL exclude expired Adjustments from accumulation per §5.1.4 and §6.1.3.

7.1.2 — TTL is a hard expiry. Unlike decay, which fades influence gradually, TTL is binary: before expiry, the Adjustment counts; after expiry, it does not. The pruning at trust boundary issuance (§5.1.4) ensures the SCT chain reflects the list state with expired Adjustments removed.

## 7.2 Decay is Implementer Concern

7.2.1 — The decay field on an Adjustment is reserved as an extension point for implementations that wish to apply gradual fade-in-influence over the life of an Adjustment, rather than (or in addition to) hard TTL expiry. The schema and semantics of the decay field are implementer choice. The deterministic accumulation (§6) uses each active Adjustment's delta as-is and does not apply decay; decay, where used, is an enforcement-layer interpretation concern. The spec does NOT normatively define decay strategies or decay parameters.

7.2.2 — Pass-through requirement. Implementations that do not consume the decay field SHALL pass it through unchanged when propagating Adjustments. An implementation MAY ignore decay when computing its own accumulation. An implementation SHALL NOT strip the decay field from Adjustments it does not understand.

7.2.3 — Why decay is not standardized. Decay strategies are tightly coupled to domain-specific risk dynamics — how quickly evidence stales, how reliable the original observation was, how the operational environment evolves. Different domains warrant different decay shapes. The spec leaves the choice to implementations and customer programs, which observe their flows and tune over time.

7.2.4 — Future standardization. If patterns emerge in the ecosystem around common decay approaches, future versions of this spec MAY define standard decay strategies. Until then, decay remains implementer extension.

— — —

# 8\. Implementer Obligations

## 8.1 SHALL Carry and Preserve Adjustments

8.1.1 — A SADAR-conformant implementation SHALL carry the Adjustments it receives forward in the signed SCT chain and SHALL preserve every Adjustment unchanged. An implementation that drops, alters, reorders, or removes a received Adjustment is not conformant. The chain is immutable per §5.1.3, so preservation is enforced cryptographically: an implementation cannot silently mutate prior Adjustments without invalidating the chain.

8.1.2 — A step's own Adjustment, if any, SHALL be added as a signed risk\_adjustment claim in the segment it issues (§5.3). A step that has no Adjustment to contribute issues its segment without a risk\_adjustment claim.

## 8.2 SHALL Compute the Convenience Scalar

8.2.1 — A conformant implementation that issues an SCT segment SHALL compute the convenience scalar by the deterministic accumulation of §6 over the currently-active Adjustments and write it to Baggage (§5.2, §5.3). Because the accumulation is a defined, trivial operation, there is no "degenerate" non-computing case: an implementation that carries Adjustments faithfully necessarily produces the correct scalar.

8.2.2 — Interpretation is not required for conformance. A conformant implementation is not required to interpret the scalar, apply weighting, or act on accumulated risk; those are enforcement-layer concerns (§6.2). A component that only carries Adjustments and computes the convenience scalar is fully conformant; one that also interprets and acts does so above the SADAR substrate.

## 8.3 SHALL Preserve Unknown Reasons

8.3.1 — Per §3.3.4 and §4.2.2, implementations SHALL preserve Adjustments with unrecognized reason values. The deterministic accumulation includes the Adjustment's delta regardless of reason recognition; reason-specific weighting is enforcement-layer interpretation (§6.2). Implementations SHALL NOT drop the Adjustment from the chain.

## 8.4 SHALL Reject Malformed Adjustments

8.4.1 — At consumption time, implementations SHALL reject Adjustments that fail structural validation:

delta outside \[-1.0, 1.0]

ttl\_seconds < 0

timestamp not RFC 3339

reason not a well-formed IRI

adjustment\_id not a well-formed URN

Object missing required fields

8.4.2 — Rejection on structural grounds is fail-closed and logged with the error code from §8.5. The malformed Adjustment is not carried into the chain; previously carried Adjustments are preserved.

## 8.5 Error Codes

8.5.1 — Errors emitted by Risk Score processing are namespaced under urn:sadar:error:v1:risk\_score:\* per the precedent established by C31 §4.6.X.10.7.

|||
|-|-|
|**Error Code**|**Meaning**|
|urn:sadar:error:v1:risk\_score:malformed\_adjustment|Adjustment failed structural validation per §8.4.|
|urn:sadar:error:v1:risk\_score:parity\_violation|The Baggage scalar does not equal the deterministic accumulation (§6) of the currently-active Adjustments in the SCT chain. Reported per C28 v2 §5.1.X.|
|urn:sadar:error:v1:risk\_score:invalid\_baggage|Baggage value at urn:sadar:baggage:v1:risk\_score is not a valid JSON number in \[0.0, 1.0].|

— — —

# 9\. Score Range Constraints

9.1 — All reported aggregate scores SHALL be in \[0.0, 1.0]. Reported scores include: SCT risk\_score claims, Telemetry Record risk\_score fields (§10), and any score values surfaced through inspection or query interfaces.

9.2 — The summation may produce an out-of-range intermediate value (for example, a sum above 1.0 or below 0.0). Clamping to \[0.0, 1.0] is applied at the boundary between the accumulation and the reported aggregate.

9.3 — Floor of zero. There is no operational state better than "no accumulated risk." Negative aggregate values are not meaningful. Adjustments may be negative — they may credit a flow for clean policy gates, verified attestations, or quality signals — but the credit is bounded by the floor: a flow at 0.0 cannot accumulate further "credit" against future risk. This is intentional: credit accumulation would be a perverse incentive structure.

— — —

# 10\. Telemetry Record Integration

## 10.1 Adjustments Visible in Audit

10.1.1 — Adjustments contributing to a flow are part of the flow's audit record. The Telemetry Record (C20 v5 §5.1.14.X.2) is the canonical per-invocation audit artifact and is the natural home for adjustment-level audit data.

10.1.2 — C20 v5 → v5.2 back-port. C20 v5 §5.1.14.X.2 is extended with two additional fields. The back-port is non-substantive (additive fields, backwards compatible) and rolls C20 from v5 (or v5.1, if the Part IV C33 back-port has been applied) to v5.2:

||||
|-|-|-|
|**Field**|**Type**|**Definition**|
|risk\_score\_at\_finalize|decimal in \[0.0, 1.0]|The accumulated risk score at the moment of Telemetry Record finalize, computed by the deterministic accumulation (§6) over the currently-active Adjustments in the SCT chain.|
|risk\_adjustments|array of Adjustment objects|The Adjustments active at the moment of finalize, including those carried from upstream and those issued during this invocation, recorded for at-rest audit. Adjustment objects per §3, including the context field — which is carried only here, where it MAY be encrypted, and not in the clear SCT claim (§5.2.3). The signed SCT chain remains the authoritative, attributable source of the Adjustments.|

10.1.3 — Adjustments issued during this invocation. Where the invocation itself emitted Adjustments (e.g., a guardrail step observed a policy violation and emitted +0.2 with urn:sadar:risk\_reason:v1:policy\_violation), those Adjustments SHALL be included in the risk\_adjustments field of the Telemetry Record.

10.1.4 — Helper API integration. The Helper API (C33 Part I) is extended with a method for emitting Adjustments during the invocation:

add\_risk\_adjustment(adjustment: Adjustment) → result | error

The method is admissible during Invocation and Outcome phases. The Adjustment is recorded in the Telemetry Record's risk\_adjustments field, carried as a signed risk\_adjustment claim in the SCT segment the invocation issues (§5.1), and the convenience scalar in Baggage is recomputed (§5.3). C33 v1 Part I is updated to include this method in a v1.1 back-port note tracked separately.

## 10.2 Persistence and Repatriation

10.2.1 — Persistence. Per C20 v5 §5.1.14.X.8, Telemetry Records are persisted to the implementation's storage backend at finalize. The risk\_adjustments field is persisted with the rest of the record; no separate persistence channel is required for Adjustments.

10.2.2 — Repatriation visibility. Per C20 v5 §5.1.14.X.10, repatriated telemetry includes the spans associated with the invocation; the risk\_score and risk\_adjustments fields are subject to the server's disclosure policy (repatriation\_redacted\_fields per C19 §8.3.3). Servers MAY redact risk\_score, risk\_adjustments, or specific Adjustment reasons or context per their policy. The spec does not enumerate these as non-redactable structural fields (C20 v5 §5.1.14.X.10.8.3); server discretion applies.

— — —

# 11\. Impact Score (Optional Entry-Level NFR)

## 11.1 Rationale

11.1.1 — The Risk Score addresses cumulative in-flight risk. A related but distinct concept is the impact score: the potential severity of negative consequences if a specific component (business process, agent, tool, resource) is misused.

11.1.2 — Impact and risk are related but separable. A high-impact component used correctly within its 4-way context (individual identity, functional role, business process definition, agent step + operation type) may contribute little to a flow's risk score. The same component used outside its context, or in a flow that has accumulated other risk, may contribute substantially. The impact score declares the "potential severity" — what could go wrong; the Risk Score Adjustment that the component issues at runtime conveys "what the situation actually warrants" — net of mitigation and context.

## 11.2 The Field

11.2.1 — Components MAY declare an impact score in their manifests using the following NFR field, added to C19 §8 in a v1.2 back-port:

urn:sadar:nfr:v1:protocol:impact\_score

|||
|-|-|
|**Aspect**|**Value**|
|Type|decimal|
|Range|\[0.0, 1.0]|
|Cardinality|optional, single value|
|Declarer|server (entries declare their own potential impact)|
|Default|absent (no declared impact)|

11.2.2 — Semantic. 0.0 represents no possibility of negative impact; 1.0 represents maximum severity. The publisher's intent is to convey "if this component is misused or used outside its appropriate context, how severe could the consequences be?" The publisher considers business impact, regulatory impact, reputational impact, and operational impact in arriving at a value.

11.2.3 — Optional declaration. Publishers SHOULD declare impact\_score where they can substantiate a claim. Declaring impact\_score on every entry is not required; absent declaration is interpreted as "no declared impact" rather than "zero impact."

## 11.3 Use

11.3.1 — Implementer use. The impact score is one input among many that an enforcement layer's risk-aware decision logic MAY consult. SADAR does not normatively define how impact\_score interacts with Risk Score Adjustments; the relationship is enforcement-layer concern.

11.3.2 — Patterns implementers may adopt include but are not limited to:

Use impact\_score as a multiplier on Adjustments emitted when invoking the component (a high-impact component's invocation contributes more to the flow than a low-impact one).

Use impact\_score in matching policy (refuse to invoke high-impact components from a flow already above a risk threshold).

Use impact\_score in audit and reporting (highlight high-impact invocations in operational dashboards).

Use impact\_score in attestation and certification (require human review for invocations of components above a declared impact level).

## 11.4 Limitation

11.4.1 — SADAR does not define what specific impact\_score values mean in concrete terms for any specific operation. The values are publisher-asserted. Two implementations of the "same" business operation by different publishers may declare different impact\_scores; consumers SHOULD treat impact\_score as an indicator from the publisher rather than as a normalized cross-publisher quantity.

11.4.2 — Calibration over time. The expectation is that organizations and customer programs observe their flows over time and tune impact\_score declarations and consumption logic accordingly. NIST SP 800-30 Appendix H provides best-practice framing for the qualitative scales that anchor calibration; see §12.

— — —

# 12\. References

## 12.1 Normative References

|||
|-|-|
|**Reference**|**Title / Relationship**|
|RFC 2119|Key words for use in RFCs|
|RFC 8174|Ambiguity of uppercase vs lowercase in RFC 2119|
|RFC 3339|Date and Time on the Internet: Timestamps|
|RFC 3986 / 3987|URI / IRI syntax|
|SADAR Core Security|Companion specification.|
|SADAR scope.md|Companion specification — Baggage propagation, SCT chain operations.|
|C2 v2 (SCT Operations)|SCT structure that carries the risk\_score scalar claim.|
|C19 v1.0 (NFR Schema)|NFR namespace pattern; impact\_score field defined as a protocol NFR.|
|C20 v5 (Telemetry Record)|Telemetry Record schema; risk\_score and risk\_adjustments fields added by the v5.2 back-port note in §10.|
|C28 v2 (Cryptographic Parity)|Parity between Baggage and SCT layers.|
|C31 v1 (Asserted Trust Model SCT Validation)|urn:sadar:error:v1:\* error code namespace precedent.|

## 12.2 Informative References

|||
|-|-|
|**Reference**|**Note**|
|NIST SP 800-30 Revision 1, Appendix H|"Impact." Provides the qualitative impact scales referenced as best-practice framing for impact\_score calibration. The scales are not adopted normatively; organizations using SADAR are expected to tailor SP 800-30's starting framework to their operational context.|
|NIST AI Risk Management Framework|Broader risk management framework for AI systems.|
|ISO/IEC 23894:2023|AI risk management.|

# Editorial Notes (Non-Normative)

***On standardizing accumulation but not interpretation.*** *A natural impulse is to let each implementation choose its own accumulation so it can encode domain sensitivity. This was considered and inverted. The accumulation that produces the propagated scalar is standardized as a single deterministic sum, for one reason: so the scalar on the wire is universally provable from the signed Adjustment chain, with no issuer-private algorithm to reconcile. Domain sensitivity does not disappear — it moves up, to the enforcement layer, which interprets the same signed Adjustments with whatever weighting, discounting, or learned model a domain warrants, and which may contribute its own Adjustment. The cost is that the SADAR scalar is deliberately naive; the benefit is that the evidence is attributable and the scalar is verifiable everywhere, while sophistication lives where it belongs.*

***On the relationship to Conformance Gradient.*** *Earlier C21 framing included an informative Conformance Gradient annex addressing partial implementations of the SADAR specification corpus. That annex is parked for now and will be revisited when the spec corpus is feature-complete and conformance testing concerns are in scope. The Risk Score specification is independently usable without the Gradient annex.*

***On future evolution.*** *Three areas where this specification is likely to evolve: (1) the reason enumeration in §4 will accumulate domain-specific values as ecosystem participants extend it; SADAR may absorb commonly-seen patterns into future versions of the starter set. (2) Decay strategies (§7.2) may converge on common patterns worth standardizing. (3) Connection to predictive risk modeling (beyond the passive observational pattern of v1.0) is a likely extension area as ML-based risk inference matures.*

