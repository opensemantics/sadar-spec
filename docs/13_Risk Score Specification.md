# SADAR™ Risk Score Specification

*Specification — Version 1.0.0*

Status: Draft • Tracking ID: C21

*April 2026*

*© 2026 Cognita AI Inc. All rights reserved. SADAR™ is a trademark of Cognita AI, Inc. CogniWeave™ is a trademark of Cognita AI, Inc. All other trademarks are the property of their respective owners.*

## Document Overview

This specification defines the SADAR Risk Score: a stateful, cumulative measure of the operational risk associated with an in-flight process flow. The score travels with the flow as it crosses agents, tools, resources, and trust boundaries, and is adjusted up or down by participants along the way as risk-relevant events occur.

The Risk Score is intended to provide a substrate for run-time risk-aware decision-making — interventions, escalations, guardrail invocations, audit triggers — while leaving the specific intervention policies and accumulation algorithms to implementations and end-customer programs. SADAR defines the data model, the propagation pattern, and the conformance obligations; SADAR does NOT define what specific score values mean for specific operations, what intervention thresholds apply, or how multiple adjustments combine into a current score. Those are the responsibility of the user program, observed and tuned by use case, customer, and domain.

**Cross-references.** C28 v2 §5.1.X (Cryptographic Parity — the Risk Score participates in the parity model between Baggage and SCT). C19 v1.0 §10 (NFR Schema — impact\_score field added in §11 of this document). C20 v5 §5.1.14.X.2 (Telemetry Record — risk adjustments visible in audit trail per the v5.2 back-port note in §10). C2 v2 §B.7 (SCT structure — the risk\_score scalar claim added in §5.1).

**Informative reference.** NIST SP 800-30 Revision 1, Appendix H (Impact). The framing for thinking about the magnitude of negative impact draws on Appendix H's qualitative impact scales as a starting point. SADAR does not adopt SP 800-30 normatively; the appendix is referenced as best-practice framing for organizations defining their own impact mappings.

— — —

# 1. Introduction

## 1.1 Purpose

1.1.1 — This specification defines a normative substrate for risk-aware run-time control across SADAR-conformant agent flows. The substrate provides a propagation channel for incremental risk observations, a data model for those observations, and obligations on conformant implementations to accept and propagate them.

1.1.2 — The substrate exists because the deterministic-to-probabilistic shift in agentic systems introduces cumulative risk effects that no single component can detect or mitigate alone. A flow that touches sensitive data, then exhibits anomalous output, then crosses a trust boundary, then chains into another high-impact action — is, considered as a whole, riskier than the sum of its parts considered individually. The Risk Score gives flows a place to record those cumulative observations and gives governance layers a place to read them.

## 1.2 Scope

1.2.1 — In scope. This specification defines: the Risk Score data model (a current scalar in 0.0–1.0); the Risk Score Adjustment data structure (deltas, reasons, TTLs, optional decay); the starter reason enumeration; the propagation pattern across Baggage and SCT; the parity requirements between layers; implementer obligations to accept and propagate adjustments; the optional impact\_score NFR for entries; integration with the Telemetry Record audit trail.

1.2.2 — Out of scope. This specification does NOT define: the specific score values that constitute "risky" in any given domain; intervention thresholds; the accumulation algorithm used to compute the current scalar from a list of adjustments; the specific mitigation techniques an agent or guardrail may apply (such techniques are implementer-proprietary and may include policy-driven mitigations, drift-vs-baseline checks, attestation gating, demographic-bias detection, hallucination detection, and others); intervention orchestration logic. These concerns are user-program responsibility, tuned over time by use case, customer, and domain.

## 1.3 Normative Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 / RFC 8174 when, and only when, they appear in all capitals.

— — —

# 2. Conceptual Model

## 2.1 The Score is Cumulative and In-Flight

2.1.1 — The Risk Score is a property of an in-flight process flow, not a static manifest attribute. It begins at zero when the flow starts. Participants in the flow — agents, tools, resources, guardrails, policy enforcement points — may issue Risk Score Adjustments that influence the score. The score travels with the flow as it progresses across components and trust boundaries.

2.1.2 — The score is a real number in the closed interval [0.0, 1.0]. 0.0 represents no accumulated risk; 1.0 represents maximum accumulated risk. The granularity of values within the interval is implementation choice; the spec does not normatively discretize the scale.

2.1.3 — The score is the result of an accumulation algorithm applied to the list of currently-active Risk Score Adjustments (§3) at the moment of evaluation. Different algorithms applied to the same adjustment list MAY produce different scores; this is by design, since accumulation is implementer-defined (§6).

## 2.2 Adjustments Are Net Step Contributions

2.2.1 — An agent, tool, resource, guardrail, or policy enforcement point that emits a Risk Score Adjustment SHALL emit the net result of any mitigations or compensating analyses it has applied. The Adjustment is the residual the step is contributing to the flow's ongoing risk picture, not a raw pre-mitigation observation.

2.2.2 — A net Adjustment MAY be positive (the step observed something that increases risk) or negative (the step observed something that decreases risk, such as a clean policy check, an attestation that verified, or a quality signal exceeding a baseline). Mitigation may reduce a positive raw observation to a smaller positive Adjustment, to zero, or even to a negative value if the mitigation provides positive evidence beyond the observed risk.

2.2.3 — The mitigation logic itself is internal to the step. The spec provides no schema or vocabulary for describing what mitigation was applied; the Adjustment's context field (§3) MAY be used to convey relevant detail in free-form text for audit and human review.

## 2.3 The Aggregate is Floor-Clamped

2.3.1 — While individual Adjustments may be negative and intermediate algorithm computations may produce values below 0.0, the reported aggregate score SHALL be clamped to ≥ 0.0. The floor of zero prevents "credit accumulation" — the perverse pattern where a flow banks negative adjustments against future risk-taking — and aligns with the conceptual reality that there is no state better than "no accumulated risk."

2.3.2 — The ceiling of 1.0 is an absorbing state in the sense that a flow at 1.0 represents maximum accumulated risk. Further positive Adjustments do not increase the score above 1.0; the aggregate SHALL be clamped to ≤ 1.0. Negative Adjustments may reduce a flow from 1.0 toward 0.0 per the implementer's algorithm.

## 2.4 Two Layers of Propagation

2.4.1 — The Risk Score propagates in two layers, each serving a different operational need:

|  |  |  |
| --- | --- | --- |
| **Layer** | **Lifecycle** | **Carries** |
| Baggage | Live, in-flight, mutable | The full list of currently-active Adjustments. Carries detail sufficient for the receiving step to apply its own accumulation algorithm. Updated as steps emit new Adjustments. Fades as TTLs expire. |
| SCT | Cryptographically attested at trust boundary issuance | A scalar — the current accumulated score, point-in-time computed at chain link issuance. Single floating-point value in [0.0, 1.0]. The SCT chain across links forms an audit trail of how the score evolved across trust boundaries. |

2.4.2 — Why both. The Baggage list is the working state — it gives downstream steps the raw inputs to apply their own algorithms, lets them detect adjustments approaching expiry, and preserves the audit trail of why the score is where it is. The SCT scalar is the cryptographically-attested handoff value — it attests "at the moment this trust boundary was crossed, the accumulated score was X." The chain of SCT scalars across links provides a tamper-evident record of how the score evolved across boundaries that no in-flight Baggage manipulation can forge.

2.4.3 — Why the SCT does not carry the list. The Adjustment list in Baggage may grow large over a flow's lifetime. SCT chain links are signed and may be embedded across many spans. Carrying the full list in every SCT would be expensive and would expose detailed flow internals at every boundary crossing. The scalar preserves the essential audit fact (current cumulative state) without bloating the cryptographic record.

2.4.4 — Runtime authority. Within a process, run-time decisions SHALL use the Baggage Adjustment list as the source of truth; not all components have access to or visibility into the SCT. The SCT scalar is the boundary-handoff value and the audit snapshot, but it is not the run-time decision substrate. At every trust boundary crossing where a new SCT chain link is issued, the issuer SHALL compute a current scalar from the live Baggage list per its accumulation algorithm and embed that scalar in the new SCT.

## 2.5 Parity

2.5.1 — Cross-layer parity. The risk score is a parity instance per C28 v2 §5.1.X. The scalar in the SCT at any chain link SHALL be consistent with the value the implementer's accumulation algorithm produces from the Baggage Adjustment list at the time of SCT issuance. Discrepancy is a parity violation and is reported per C28 v2 §5.1.X.2.5.

2.5.2 — Implementation note. Parity is not a requirement that the SCT scalar match what some downstream different implementation would compute from the same Baggage list. Different implementations may use different accumulation algorithms, and the SCT scalar reflects the algorithm of the implementation that issued the chain link. Parity verification requires the verifier to use the same algorithm as the issuer; where the algorithm is not externally known, parity is verified against the issuer's declared algorithm.

— — —

# 3. Risk Score Adjustment Schema

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

|  |  |  |
| --- | --- | --- |
| **Field** | **Type** | **Definition** |
| adjustment\_id | string (URN) | Globally unique identifier for this Adjustment. Implementations MAY use UUIDs, content-addressed identifiers, or other URN forms. Required. Used for deduplication if the Adjustment is observed by multiple downstream consumers. |
| delta | decimal | Net contribution this step is making to the flow's ongoing risk picture, after the step's own mitigations. Range: [-1.0, 1.0]. Positive values increase risk; negative values decrease risk; zero is permitted (the step ran but observed nothing notable). Required. |
| ttl\_seconds | integer ≥ 0 | How long this Adjustment remains in effect, in seconds from timestamp. After TTL expiry, the Adjustment SHALL be excluded from accumulation. ttl\_seconds = 0 means the Adjustment expires immediately at issuance and is informational only. Required. |
| timestamp | RFC 3339 string | When the Adjustment was issued. Required. |
| decay | object | Optional decay parameters. Schema is implementer-extensible per §7. Implementations that do not consume the decay field SHALL pass it through unchanged. |
| reason | string (IRI) | IRI identifying the category of observation prompting this Adjustment. SHOULD be a value from the SADAR starter enumeration (§4) or an extension URN in an organization-controlled namespace. Required. |
| context | string | Free-form text describing the specific observation, mitigation applied, and any additional human-readable detail useful for audit. SHOULD be present; concise but informative. |

## 3.3 Field Constraints

3.3.1 — Delta range. delta SHALL be in [-1.0, 1.0]. A delta outside this range is malformed and SHALL be rejected at consumption time per §8.4.

3.3.2 — Timestamp ordering. Adjustments SHOULD be issued with monotonically non-decreasing timestamps within a flow. Receivers MAY order Adjustments by timestamp for accumulation purposes; implementations MAY use other ordering (insertion order, or by adjustment\_id) for deterministic replay.

3.3.3 — TTL minimum. While ttl\_seconds = 0 is permitted, implementations SHOULD use a non-zero TTL when an Adjustment is intended to influence accumulation downstream. Zero-TTL Adjustments are recorded for audit but contribute nothing to the live aggregate.

3.3.4 — Reason completeness. The reason field SHALL be a well-formed IRI. Receivers encountering an unrecognized reason IRI SHALL preserve the Adjustment in the Baggage list and pass it through; receivers MAY apply implementation-specific accumulation weight to unrecognized reasons (typically: treat as the same weight as their nearest recognized parent category, or as a default weight).

— — —

# 4. Reason Enumeration

## 4.1 Namespace

4.1.1 — SADAR-defined reason values follow:

urn:sadar:risk\_reason:v1:{category}

The pattern parallels urn:sadar:nfr:v1:{category}:{attribute} (C19), urn:sadar:error:v1:{category}:{code} (C31), and urn:sadar:scope:v1:{category}:{operation} (C33). The risk\_reason category occupies a flat single-segment structure for v1; sub-categories MAY be added in future versions if needed.

4.1.2 — Custom reasons. Implementations and ecosystem participants MAY define custom reason IRIs in their own namespaces. Custom reasons SHALL NOT use the urn:sadar: prefix.

## 4.2 Starter Enumeration

4.2.1 — The following values are normatively defined for v1. The enumeration is deliberately small and incomplete; ecosystem participants are expected to extend with reasons relevant to their domains.

|  |  |
| --- | --- |
| **Reason IRI** | **Description** |
| urn:sadar:risk\_reason:v1:sensitive\_data\_access | The step accessed data classified as sensitive (PII, PHI, financial, regulated). Use when the act of access itself is the risk-relevant event. |
| urn:sadar:risk\_reason:v1:state\_change | The step performed a state-changing operation (write, update, delete, send). Use when the change has potential downstream business impact. |
| urn:sadar:risk\_reason:v1:external\_invocation | The step invoked an external service outside the local trust boundary. Use when the invocation introduces dependency on a less-trusted system. |
| urn:sadar:risk\_reason:v1:policy\_violation | A guardrail or policy enforcement point detected a policy violation in the step's inputs, behavior, or outputs. |
| urn:sadar:risk\_reason:v1:policy\_check\_passed | A guardrail or policy enforcement point ran cleanly. Typically used with a negative delta to credit the flow for clean policy gates. |
| urn:sadar:risk\_reason:v1:attestation\_anomaly | An attestation check produced an anomalous result — inconsistent provenance, unverifiable signature, unexpected identity, or similar. |
| urn:sadar:risk\_reason:v1:attestation\_verified | An attestation check verified cleanly. Typically used with a negative delta to credit the flow for verified provenance. |
| urn:sadar:risk\_reason:v1:bias\_detection | An analysis detected demographic bias, fairness anomalies, or similar in agent output. |
| urn:sadar:risk\_reason:v1:hallucination\_detection | An analysis detected likely hallucination — output not supported by inputs, retrieval results, or grounding sources. |
| urn:sadar:risk\_reason:v1:divergence\_detection | An analysis detected divergence from expected baseline — output drift, anomalous pattern, deviation from labeled-good cluster, or similar. |
| urn:sadar:risk\_reason:v1:human\_intervention | A human intervened in the flow. May be positive (review approved) or negative (review flagged); context field clarifies. |
| urn:sadar:risk\_reason:v1:other | Reason not categorized by other values in this enumeration. Implementations SHOULD prefer specific custom IRIs in their own namespace over this catch-all. |

4.2.2 — Future versions. New reasons added to v1 are MINOR increments (backward compatible). Removal of reasons is a MAJOR increment. Receivers SHALL preserve unknown reason values; the absence of forward-compatible behavior is an interoperability failure.

## 4.3 Reason vs. Domain Vocabulary

4.3.1 — The starter enumeration is intentionally domain-neutral. Industry-specific reasons (e.g., "phi\_access\_under\_least\_privilege" for healthcare, or "sox\_significant\_change" for financial reporting) belong in domain-specific vocabularies in organization or industry namespaces, not in the SADAR starter set. The starter set provides the categorical anchors; specialization is downstream.

4.3.2 — Multiple reasons per Adjustment. The schema permits exactly one reason per Adjustment. Where a single observation reflects multiple reason categories, the issuer chooses the most-specific applicable reason and conveys additional category detail in the context field. Issuing multiple Adjustments for a single observation is an option but produces fragmented audit trails and is discouraged.

— — —

# 5. Score Propagation

## 5.1 Baggage Carriage

5.1.1 — The Risk Score Adjustment list propagates as a Baggage value within trust boundaries. The Baggage key SHALL be:

urn:sadar:baggage:v1:risk\_adjustments

5.1.2 — The value SHALL be a JSON-serialized array of Adjustment objects per §3. Implementations MAY apply Baggage-standard compression; the canonical representation prior to compression is the JSON array.

5.1.3 — Updates. When a step issues a new Adjustment, it SHALL append the Adjustment to the list and propagate the updated list in Baggage to subsequent steps. The list SHALL NOT have its existing entries modified or reordered by step issuance; new entries are appended.

5.1.4 — Pruning at trust boundary issuance. At trust boundary crossings, the issuer of a new SCT chain link SHALL prune Adjustments whose timestamp + ttl\_seconds is in the past at the moment of SCT issuance. Pruned Adjustments SHALL NOT appear in the post-boundary Baggage. The pruning is the only structural modification permitted to the list during propagation; otherwise the list is append-only within the issuer's control.

## 5.2 SCT Carriage

5.2.1 — SCT scalar claim. SCT chain links SHALL include the following claim:

|  |  |  |
| --- | --- | --- |
| **Claim** | **Type** | **Definition** |
| risk\_score | decimal in [0.0, 1.0] | The accumulated risk score at the moment this chain link was issued. Computed by the issuer's accumulation algorithm from the live Baggage Adjustment list at issuance time, after pruning per §5.1.4 and after floor-clamping per §2.3. |

5.2.2 — Default value. SCT chain links issued at the start of a flow, before any Adjustments have been emitted, SHALL carry risk\_score = 0.0.

5.2.3 — Chain link audit trail. The succession of risk\_score values across SCT chain links forms a tamper-evident audit trail of how the score evolved across trust boundaries. Verifiers reading the SCT chain receive an attested record distinct from any in-flight Baggage state.

## 5.3 Boundary Issuance Algorithm

5.3.1 — At trust boundary crossing where a new SCT chain link is issued, the issuer SHALL perform the following steps:

1. Read the current Baggage Adjustment list.

2. Prune Adjustments whose timestamp + ttl\_seconds is in the past (§5.1.4).

3. Apply the issuer's accumulation algorithm to the pruned list to compute a candidate scalar.

4. Clamp the candidate scalar to [0.0, 1.0] per §2.3.

5. Embed the clamped scalar as the risk\_score claim in the new SCT chain link.

6. Propagate the pruned (post-step-2) Baggage list to subsequent steps.

5.3.2 — Algorithm consistency. The issuer SHALL use the same accumulation algorithm at every chain link it issues within a single flow. Algorithm changes mid-flow undermine the audit trail's coherence and are not permitted.

## 5.4 Cross-Layer Parity

5.4.1 — Per §2.5, the risk\_score scalar in any SCT chain link SHALL be consistent with what the issuer's accumulation algorithm produces from the Baggage list at issuance time. Parity violations (mismatched scalar and Baggage state) are reported per C28 v2 §5.1.X.2.5 with a parity-specific error code.

— — —

# 6. Accumulation Algorithm

## 6.1 Implementer Responsibility

6.1.1 — The accumulation algorithm — the function from a list of Adjustments to a single accumulated scalar in [0.0, 1.0] — is implementer responsibility. The spec does NOT mandate a specific algorithm. Implementations are expected to evolve their algorithms based on operational experience and per-domain tuning.

6.1.2 — Algorithms MAY include but are not limited to:

Weighted sum with reason-specific weights, capped to [0.0, 1.0].

Exponentially-weighted moving averages with time-based discount.

Bayesian updating on a per-Adjustment basis, with reason-specific likelihoods.

Predictive models (e.g., ML classifiers trained on historical flows labeled by intervention outcomes).

Hybrid approaches combining several of the above.

6.1.3 — Properties algorithms SHOULD have. Algorithms SHOULD:

Be deterministic given the same Adjustment list — running the algorithm twice on the same input produces the same output.

Saturate gracefully near 1.0 — adding more high-positive Adjustments to a flow already near the ceiling SHOULD produce diminishing increases, not stay flat at exactly 1.0 (avoiding absorbing-state ambiguity).

Honor TTL semantics — Adjustments past their TTL SHALL NOT contribute. (This is normative per §5.1.4.)

Produce values in [0.0, 1.0] after clamping per §2.3.

## 6.2 Why the Algorithm Is Not Standardized

6.2.1 — Different domains warrant different sensitivity. Healthcare flows touching PHI may warrant rapid accumulation toward intervention; routine business flows may warrant conservative accumulation that requires multiple reinforcing observations to cross thresholds. A single normative algorithm cannot serve both.

6.2.2 — Algorithm disclosure. Where multiple SADAR-conformant implementations interoperate in a single flow (e.g., across federated registries), the algorithm used by each implementation is implementer-internal. The Baggage list provides the raw inputs; downstream implementations are free to apply their own algorithm rather than trusting upstream's. The SCT scalar carries the upstream's computation as an attested audit fact, but downstream implementations are not bound by it for their own decisions.

— — —

# 7. TTL and Decay

## 7.1 TTL is Normative

7.1.1 — Every Adjustment carries a ttl\_seconds (§3). At timestamp + ttl\_seconds, the Adjustment is considered expired. Implementations SHALL exclude expired Adjustments from accumulation per §5.1.4 and §6.1.3.

7.1.2 — TTL is a hard expiry. Unlike decay, which fades influence gradually, TTL is binary: before expiry, the Adjustment counts; after expiry, it does not. The pruning at trust boundary issuance (§5.1.4) ensures the SCT chain reflects the list state with expired Adjustments removed.

## 7.2 Decay is Implementer Concern

7.2.1 — The decay field on an Adjustment is reserved as an extension point for implementations that wish to apply gradual fade-in-influence over the life of an Adjustment, rather than (or in addition to) hard TTL expiry. The schema and semantics of the decay field are implementer choice. The spec does NOT normatively define decay strategies, decay parameters, or how decay interacts with the accumulation algorithm.

7.2.2 — Pass-through requirement. Implementations that do not consume the decay field SHALL pass it through unchanged when propagating Adjustments. An implementation MAY ignore decay when computing its own accumulation. An implementation SHALL NOT strip the decay field from Adjustments it does not understand.

7.2.3 — Why decay is not standardized. Decay strategies are tightly coupled to domain-specific risk dynamics — how quickly evidence stales, how reliable the original observation was, how the operational environment evolves. Different domains warrant different decay shapes. The spec leaves the choice to implementations and customer programs, which observe their flows and tune over time.

7.2.4 — Future standardization. If patterns emerge in the ecosystem around common decay approaches, future versions of this spec MAY define standard decay strategies. Until then, decay remains implementer extension.

— — —

# 8. Implementer Obligations

## 8.1 SHALL Accept Adjustments

8.1.1 — A SADAR-conformant implementation SHALL accept Adjustments received in Baggage from upstream components and SHALL preserve them in the Adjustment list propagated to downstream components. An implementation that drops, alters, or reorders received Adjustments is not conformant.

8.1.2 — Acceptance does not require processing. An implementation that accepts and propagates Adjustments without applying any accumulation logic to them is conformant for the propagation requirement. Such an implementation produces SCT scalars of 0.0 at every chain link (no accumulation = no risk reported). This is degenerate but conformant.

## 8.2 SHOULD Process Adjustments

8.2.1 — Conformant implementations SHOULD apply an accumulation algorithm and produce non-trivial SCT scalars. Doing so is the core operational value of the Risk Score mechanism. Failing to process Adjustments deprives downstream consumers and audit of risk-aware operation.

8.2.2 — Why this is SHOULD and not SHALL. End-customer programs vary in their willingness and capability to act on accumulated risk. A SADAR-conformant component that lacks the policy infrastructure to act on risk scores is still useful as a propagator (preserving the audit trail) and SHOULD not be excluded from conformance for that reason. Components that DO process Adjustments SHOULD do so with reason-aware weighting rather than treating all reasons identically.

## 8.3 SHALL Preserve Unknown Reasons

8.3.1 — Per §3.3.4 and §4.2.2, implementations SHALL preserve Adjustments with unrecognized reason values. Implementations MAY apply implementation-specific accumulation weight to unrecognized reasons but SHALL NOT drop the Adjustment from the propagated list.

## 8.4 SHALL Reject Malformed Adjustments

8.4.1 — At consumption time, implementations SHALL reject Adjustments that fail structural validation:

delta outside [-1.0, 1.0]

ttl\_seconds < 0

timestamp not RFC 3339

reason not a well-formed IRI

adjustment\_id not a well-formed URN

Object missing required fields

8.4.2 — Rejection on structural grounds is fail-closed and logged with the error code from §8.5. The malformed Adjustment is not propagated; the rest of the Adjustment list is preserved.

## 8.5 Error Codes

8.5.1 — Errors emitted by Risk Score processing are namespaced under urn:sadar:error:v1:risk\_score:\* per the precedent established by C31 §4.6.X.10.7.

|  |  |
| --- | --- |
| **Error Code** | **Meaning** |
| urn:sadar:error:v1:risk\_score:malformed\_adjustment | Adjustment failed structural validation per §8.4. |
| urn:sadar:error:v1:risk\_score:parity\_violation | SCT scalar inconsistent with Baggage list given the issuer's accumulation algorithm. Reported per C28 v2 §5.1.X. |
| urn:sadar:error:v1:risk\_score:invalid\_baggage | Baggage value at urn:sadar:baggage:v1:risk\_adjustments is not a valid JSON array of Adjustments. |

— — —

# 9. Score Range Constraints

9.1 — All reported aggregate scores SHALL be in [0.0, 1.0]. Reported scores include: SCT risk\_score claims, Telemetry Record risk\_score fields (§10), and any score values surfaced through inspection or query interfaces.

9.2 — Internal computation may produce out-of-range intermediate values per the implementer's algorithm. Clamping to [0.0, 1.0] is applied at the boundary between internal computation and the reported aggregate.

9.3 — Floor of zero. There is no operational state better than "no accumulated risk." Negative aggregate values are not meaningful. Adjustments may be negative — they may credit a flow for clean policy gates, verified attestations, or quality signals — but the credit is bounded by the floor: a flow at 0.0 cannot accumulate further "credit" against future risk. This is intentional: credit accumulation would be a perverse incentive structure.

— — —

# 10. Telemetry Record Integration

## 10.1 Adjustments Visible in Audit

10.1.1 — Adjustments contributing to a flow are part of the flow's audit record. The Telemetry Record (C20 v5 §5.1.14.X.2) is the canonical per-invocation audit artifact and is the natural home for adjustment-level audit data.

10.1.2 — C20 v5 → v5.2 back-port. C20 v5 §5.1.14.X.2 is extended with two additional fields. The back-port is non-substantive (additive fields, backwards compatible) and rolls C20 from v5 (or v5.1, if the Part IV C33 back-port has been applied) to v5.2:

|  |  |  |
| --- | --- | --- |
| **Field** | **Type** | **Definition** |
| risk\_score\_at\_finalize | decimal in [0.0, 1.0] or null | The accumulated risk score at the moment of Telemetry Record finalize per the issuing implementation's accumulation algorithm. Null if the implementation does not process Adjustments (the SHOULD-process case in §8.2). |
| risk\_adjustments | array of Adjustment objects | The list of Adjustments active in Baggage at the moment of finalize, including those propagated from upstream and those issued during this invocation. Adjustment objects per §3 of this specification. The list is the raw audit input — the set of observations that contributed to the score. |

10.1.3 — Adjustments issued during this invocation. Where the invocation itself emitted Adjustments (e.g., a guardrail step observed a policy violation and emitted +0.2 with urn:sadar:risk\_reason:v1:policy\_violation), those Adjustments SHALL be included in the risk\_adjustments field of the Telemetry Record.

10.1.4 — Helper API integration. The Helper API (C33 Part I) is extended with a method for emitting Adjustments during the invocation:

add\_risk\_adjustment(adjustment: Adjustment) → result | error

The method is admissible during Invocation and Outcome phases. The Adjustment is appended to the Telemetry Record's risk\_adjustments field and propagated to Baggage. C33 v1 Part I is updated to include this method in a v1.1 back-port note tracked separately.

## 10.2 Persistence and Repatriation

10.2.1 — Persistence. Per C20 v5 §5.1.14.X.8, Telemetry Records are persisted to the implementation's storage backend at finalize. The risk\_adjustments field is persisted with the rest of the record; no separate persistence channel is required for Adjustments.

10.2.2 — Repatriation visibility. Per C20 v5 §5.1.14.X.10, repatriated telemetry includes the spans associated with the invocation; the risk\_score and risk\_adjustments fields are subject to the server's disclosure policy (repatriation\_redacted\_fields per C19 §8.3.3). Servers MAY redact risk\_score, risk\_adjustments, or specific Adjustment reasons or context per their policy. The spec does not enumerate these as non-redactable structural fields (C20 v5 §5.1.14.X.10.8.3); server discretion applies.

— — —

# 11. Impact Score (Optional Entry-Level NFR)

## 11.1 Rationale

11.1.1 — The Risk Score addresses cumulative in-flight risk. A related but distinct concept is the impact score: the potential severity of negative consequences if a specific component (business process, agent, tool, resource) is misused.

11.1.2 — Impact and risk are related but separable. A high-impact component used correctly within its 4-way context (individual identity, functional role, business process definition, agent step + operation type) may contribute little to a flow's risk score. The same component used outside its context, or in a flow that has accumulated other risk, may contribute substantially. The impact score declares the "potential severity" — what could go wrong; the Risk Score Adjustment that the component issues at runtime conveys "what the situation actually warrants" — net of mitigation and context.

## 11.2 The Field

11.2.1 — Components MAY declare an impact score in their manifests using the following NFR field, added to C19 §8 in a v1.2 back-port:

urn:sadar:nfr:v1:protocol:impact\_score

|  |  |
| --- | --- |
| **Aspect** | **Value** |
| Type | decimal |
| Range | [0.0, 1.0] |
| Cardinality | optional, single value |
| Declarer | server (entries declare their own potential impact) |
| Default | absent (no declared impact) |

11.2.2 — Semantic. 0.0 represents no possibility of negative impact; 1.0 represents maximum severity. The publisher's intent is to convey "if this component is misused or used outside its appropriate context, how severe could the consequences be?" The publisher considers business impact, regulatory impact, reputational impact, and operational impact in arriving at a value.

11.2.3 — Optional declaration. Publishers SHOULD declare impact\_score where they can substantiate a claim. Declaring impact\_score on every entry is not required; absent declaration is interpreted as "no declared impact" rather than "zero impact."

## 11.3 Use

11.3.1 — Implementer use. The impact score is one input among many that an implementer's accumulation algorithm or risk-aware decision logic MAY consult. SADAR does not normatively define how impact\_score interacts with Risk Score Adjustments; the relationship is implementer concern.

11.3.2 — Patterns implementers may adopt include but are not limited to:

Use impact\_score as a multiplier on Adjustments emitted when invoking the component (a high-impact component's invocation contributes more to the flow than a low-impact one).

Use impact\_score in matching policy (refuse to invoke high-impact components from a flow already above a risk threshold).

Use impact\_score in audit and reporting (highlight high-impact invocations in operational dashboards).

Use impact\_score in attestation and certification (require human review for invocations of components above a declared impact level).

## 11.4 Limitation

11.4.1 — SADAR does not define what specific impact\_score values mean in concrete terms for any specific operation. The values are publisher-asserted. Two implementations of the "same" business operation by different publishers may declare different impact\_scores; consumers SHOULD treat impact\_score as an indicator from the publisher rather than as a normalized cross-publisher quantity.

11.4.2 — Calibration over time. The expectation is that organizations and customer programs observe their flows over time and tune impact\_score declarations and consumption logic accordingly. NIST SP 800-30 Appendix H provides best-practice framing for the qualitative scales that anchor calibration; see §12.

— — —

# 12. References

## 12.1 Normative References

|  |  |
| --- | --- |
| **Reference** | **Title / Relationship** |
| RFC 2119 | Key words for use in RFCs |
| RFC 8174 | Ambiguity of uppercase vs lowercase in RFC 2119 |
| RFC 3339 | Date and Time on the Internet: Timestamps |
| RFC 3986 / 3987 | URI / IRI syntax |
| SADAR Core Security | Companion specification. |
| SADAR scope.md | Companion specification — Baggage propagation, SCT chain operations. |
| C2 v2 (SCT Operations) | SCT structure that carries the risk\_score scalar claim. |
| C19 v1.0 (NFR Schema) | NFR namespace pattern; impact\_score field defined as a protocol NFR. |
| C20 v5 (Telemetry Record) | Telemetry Record schema; risk\_score and risk\_adjustments fields added by the v5.2 back-port note in §10. |
| C28 v2 (Cryptographic Parity) | Parity between Baggage and SCT layers. |
| C31 v1 (Asserted Trust Model SCT Validation) | urn:sadar:error:v1:\* error code namespace precedent. |

## 12.2 Informative References

|  |  |
| --- | --- |
| **Reference** | **Note** |
| NIST SP 800-30 Revision 1, Appendix H | "Impact." Provides the qualitative impact scales referenced as best-practice framing for impact\_score calibration. The scales are not adopted normatively; organizations using SADAR are expected to tailor SP 800-30's starting framework to their operational context. |
| NIST AI Risk Management Framework | Broader risk management framework for AI systems. |
| ISO/IEC 23894:2023 | AI risk management. |

# Editorial Notes (Non-Normative)

***On the design choice to leave accumulation open.*** *A natural impulse in spec design is to define a normative accumulation algorithm so that all SADAR implementations produce comparable scores from comparable inputs. This impulse was considered and rejected. Different domains warrant different sensitivity: a healthcare flow touching PHI may need rapid accumulation toward intervention; a routine business flow may need conservative accumulation requiring multiple reinforcing observations. A single normative algorithm cannot serve both while remaining useful for either. The Baggage list provides the raw audit inputs; downstream implementations may apply their own algorithms, or may use upstream's SCT scalar as a sufficient handoff value depending on their trust model with upstream. The cost of this openness is that cross-implementation score comparability is not guaranteed; the benefit is that implementations are not forced into a one-size-fits-all algorithm that fits no domain particularly well.*

***On the relationship to Conformance Gradient.*** *Earlier C21 framing included an informative Conformance Gradient annex addressing partial implementations of the SADAR specification corpus. That annex is parked for now and will be revisited when the spec corpus is feature-complete and conformance testing concerns are in scope. The Risk Score specification is independently usable without the Gradient annex.*

***On future evolution.*** *Three areas where this specification is likely to evolve: (1) the reason enumeration in §4 will accumulate domain-specific values as ecosystem participants extend it; SADAR may absorb commonly-seen patterns into future versions of the starter set. (2) Decay strategies (§7.2) may converge on common patterns worth standardizing. (3) Connection to predictive risk modeling (beyond the passive observational pattern of v1.0) is a likely extension area as ML-based risk inference matures.*