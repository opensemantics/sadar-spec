SADAR™ Telemetry Record and Repatriation

*Specification — Version 5.2.0*

Status: Draft • Tracking ID: C20 • Combined back-ports from C33 IV and C21 §10 applied

*April 2026*

*© 2026 Cognita AI Inc. All rights reserved. SADAR™ is a trademark of Cognita AI, Inc. CogniWeave™ is a trademark of Cognita AI, Inc. All other trademarks are the property of their respective owners.*

## Document Overview

This specification defines the SADAR Telemetry Record — the canonical per-invocation persistent audit artifact — and the governed cross-trust-boundary Telemetry Repatriation mechanism. Together they form the §5.1.14.X subsection of scope.md's observability requirements.

v5.2 incorporates two additive back-ports:

C33 Part IV (April 2026): clarifies trigger timing for repatriation as OTel span close, distinct from Telemetry Record finalize. New subsection §5.1.14.X.10.7.4.

C21 §10 (April 2026): adds risk\_score\_at\_finalize and risk\_adjustments fields to the Telemetry Record schema for audit visibility of in-flight risk score evolution.

**Cross-references.** C19 §8.3 (Repatriation NFR fields). C28 (Cryptographic parity). C33 v1.1 Parts I and II (Helper API including add\_risk\_adjustment, repatriation trigger specification). C21 v1 (Risk Score Specification). scope.md §5.1.14 (Observability umbrella).

— — —

# §5.1.14.X.1 Purpose and Scope

§5.1.14.X.1.1 — Every SAI invocation produces a Telemetry Record: a structured audit artifact capturing the search, selection, invocation, and outcome of the operation. The Telemetry Record is persisted to an implementer-chosen storage backend and is the canonical audit artifact for that invocation.

§5.1.14.X.1.2 — Repatriation extends the Telemetry Record's audit value across trust boundaries. Where a server's invocation is itself part of a longer flow originating with a requester in a different trust domain, repatriation enables the server to return policy-controlled trace fragments to its caller — preserving the originator's observability while respecting the server's disclosure policy.

§5.1.14.X.1.3 — Both concerns (Telemetry Record and Repatriation) are part of the SADAR observability normative scope established in scope.md §5.1.14. Implementer behaviors beyond the normative requirements (storage backend choice, retention policy, telemetry routing) are implementer concern.

# §5.1.14.X.2 Telemetry Record Schema

§5.1.14.X.2.1 — Every Telemetry Record SHALL include the following fields. Fields are organized by lifecycle phase per §5.1.14.X.3.

## Identification and Version

|  |  |  |
| --- | --- | --- |
| **Field** | **Type** | **Description** |
| schema\_version | string (URN) | Version of this Telemetry Record schema. v5.2 manifests declare urn:sadar:telemetry:v5.2. |
| record\_id | string (URN) | Globally unique identifier for this Telemetry Record. Set at Helper construction. |
| calling\_manifest\_id | string (URN) | Manifest URN of the calling entity (the SAI invoker). Set at Helper construction. |

## Search Phase

|  |  |  |
| --- | --- | --- |
| **Field** | **Type** | **Description** |
| timestamp\_search | RFC 3339 string | When the search began. |
| criteria | object | The search criteria — capability requirements and NFR constraints — as submitted to the registry. May include application-supplied extensions per Helper API set\_criteria\_extension. |
| candidate\_count | integer | Number of candidates returned by the registry pre-filter. |
| qualified\_count | integer | Number of candidates surviving NFR matching with at least PARTIAL\_MATCH classification. |

## Selection Phase

|  |  |  |
| --- | --- | --- |
| **Field** | **Type** | **Description** |
| resolver\_id | string (URN) | Identifier of the resolver invoked for selection. |
| resolver\_version | string (semver) | Resolver version. |
| resolver\_location | string (URL/URN) | Where the resolver was loaded from. |
| resolver\_endpoint | string (URL) | How the resolver was invoked. |
| selected\_entity\_id | string (URN) or null | The entity URN of the selected target, or null if selection failed. |
| selected\_agent\_id | string (URN) or null | The agent URN of the selected target, or null if selection failed. |
| registry\_source | string (URN) | The registry from which the selected entry was discovered. |
| third\_party\_registry | boolean | Whether the registry\_source is the local or a federated registry. |
| sadar\_process\_id | string (URN) or null | The SADAR process identifier from the SCT, if any. |
| selection\_exception | object or null | If selection failed, the structured exception per resolver contract. Null on success. |

## Invocation Phase

|  |  |  |
| --- | --- | --- |
| **Field** | **Type** | **Description** |
| timestamp\_invoked | RFC 3339 string | When the call to the selected target was dispatched. |
| timestamp\_returned | RFC 3339 string | When the response (or final failure signal) was received. |
| duration\_seconds | decimal | timestamp\_returned - timestamp\_invoked. |
| input\_summary | object or null | Structured summary of input data per §5.1.14.X.4. Subject to redaction per the application's policy. |
| output\_summary | object or null | Structured summary of output data per §5.1.14.X.4. Subject to redaction per the application's policy. |
| invocation\_status | enum | SUCCESS, FAILED, TIMED\_OUT, NOT\_INVOKED. |

## Outcome Phase

|  |  |  |
| --- | --- | --- |
| **Field** | **Type** | **Description** |
| audit\_metadata | object | Custom audit fields added via Helper API add\_audit\_metadata. IRI-namespaced keys; SADAR namespace reserved. |
| risk\_score\_at\_finalize | decimal in [0.0, 1.0] or null | NEW IN v5.2 (back-port from C21 §10). The accumulated risk score at the moment of finalize, computed by the issuing implementation's accumulation algorithm per C21 §6 over the Adjustment list. Null if the implementation does not process Adjustments (the SHOULD-process case in C21 §8.2). Floor-and ceiling-clamped to [0.0, 1.0]. |
| risk\_adjustments | array of Adjustment objects | NEW IN v5.2 (back-port from C21 §10). The list of Risk Score Adjustments active in OTel Baggage at the moment of finalize, including those propagated from upstream and those issued during this invocation. Adjustment objects per C21 §3. The list is the raw audit input — the set of observations that contributed to risk\_score\_at\_finalize. |

§5.1.14.X.2.2 — Required vs optional. All fields above are required for finalization, with the following exceptions:

selected\_entity\_id, selected\_agent\_id, registry\_source are null when selection\_exception is non-null (selection failed).

sadar\_process\_id is null when no SADAR process is declared in the SCT.

input\_summary, output\_summary, audit\_metadata are optional — application code MAY omit these.

risk\_score\_at\_finalize is null when the implementation does not process Risk Score Adjustments (C21 §8.2 SHOULD-process is unsatisfied).

risk\_adjustments is an empty array when no Adjustments were active in Baggage at finalize.

§5.1.14.X.2.3 — Field immutability. Once a phase ends per §5.1.14.X.3 phase model, fields finalized at the end of that phase SHALL NOT be modified. The Helper API enforces this per C33 §I.3.

# §5.1.14.X.3 Lifecycle and Helper API Requirement

§5.1.14.X.3.1 — Telemetry Record construction follows the four-phase lifecycle of an SAI invocation: Search, Selection, Invocation, Outcome. Each phase populates its own subset of fields per §5.1.14.X.2, and finalization seals the record.

§5.1.14.X.3.2 — Helper API requirement. Implementations SHALL access the Telemetry Record only through a SADAR-defined Helper API. The Helper API enforces field validation, phase-immutability, and the boundary between SAI-internal fields and application-facing fields. The complete Helper API surface is defined in C33 §I.

§5.1.14.X.3.3 — Application-driven Risk Score Adjustments. The Helper API's add\_risk\_adjustment method (C33 §I.4.6) is the canonical surface through which application-driven Adjustments enter the Telemetry Record's risk\_adjustments field. SAI-internal Adjustments (e.g., emitted when SAI detects a verification failure) bypass the application-facing Helper API and use SAI's internal access path.

# §5.1.14.X.4 Input and Output Summaries

§5.1.14.X.4.1 — input\_summary and output\_summary fields capture structured summaries of the data passed to and returned from the invocation. They are application-set per C33 §I.4.2 and §I.4.3.

§5.1.14.X.4.2 — Application-driven redaction. The application is responsible for redacting sensitive content before submitting summaries to the Helper API. The Helper validates structure (must be an object, JSON-serializable, with valid redaction markers if used) but does not enforce redaction policy. Implementations are encouraged to use standard redaction markers ([[REDACTED]], "\*\*\*", or implementation-specific patterns) for human-readable audit output.

§5.1.14.X.4.3 — IRI-based field tagging. Where the input and output summaries reference fields whose semantics are governed by published standards, summaries SHOULD use the IRI form for field identification. This anchors the summary to the same semantic vocabulary used in the manifest's data field declarations.

§5.1.14.X.4.4 — Cross-standard composition. Summaries may reference data from multiple standards (X12, HL7, ISO, D&B, etc.). Detailed semantics for cross-standard composition are governed by the forthcoming SADAR Data Classification Annex. This specification establishes only the structural placement.

# §5.1.14.X.5 Layering with OTel

§5.1.14.X.5.1 — Telemetry Records and OTel spans are distinct artifacts with distinct purposes:

|  |  |
| --- | --- |
| **Artifact** | **Purpose** |
| Telemetry Record | Per-invocation persistent audit artifact. One per SAI invocation. Captures structured outcome and audit metadata. Persisted to local storage at finalize. |
| OTel Spans | Per-operation observability traces. Multiple per SAI invocation (one for SAI itself, additional spans for downstream calls SAI initiates). Carry runtime trace context, baggage, and operation telemetry. Emitted via OTel SDK. |

§5.1.14.X.5.2 — Correlation. The Telemetry Record's record\_id is correlated with the SAI invocation's root span via OTel trace context. The SCT chain root jti correlates with both. This three-way correlation enables cross-artifact audit reconstruction.

# §5.1.14.X.6 OTel Span Structure

§5.1.14.X.6.1 — SAI emits OTel spans for both its own operations and the downstream calls it initiates. Span naming follows OTel semantic conventions where applicable; SADAR-specific span attributes are defined in this section.

§5.1.14.X.6.2 — Required SADAR span attributes:

|  |  |  |
| --- | --- | --- |
| **Attribute** | **Type** | **Description** |
| telemetry.origin.environment | string (structured ID) | The issuing entity, agent, and environment in <entity\_urn>:<agent\_urn>:<environment\_id> form. Per §5.1.14.X.7 provenance attribution. SHALL be present on every SADAR-conformant span. |
| telemetry.origin.agent | string (URN) | The agent identifier within the issuing entity. SHALL be present. |
| telemetry.repatriated | boolean | Whether this span has been (or will be) repatriated to the caller. Used for audit-trail reconstruction. |
| telemetry.disclosure.policy\_id | string (URN) or null | When telemetry.repatriated is true, the identifier of the disclosure policy applied. Null when repatriation does not apply. |
| sadar.process\_id | string (URN) or null | The SADAR process identifier from the SCT, if any. Permits cross-trace correlation by business process. |
| sadar.sct\_chain\_root\_jti | string (JWT ID) | The chain root jti of the SCT under which this span was emitted. Permits cross-trace correlation by SCT chain. |

§5.1.14.X.6.3 — Standard OTel span attributes (trace\_id, span\_id, parent\_span\_id, timestamps, status, name) follow OTel conventions and are required as for any conformant OTel span.

§5.1.14.X.6.4 — Risk Score baggage. The OTel Baggage value at urn:sadar:baggage:v1:risk\_adjustments carries the live list of Risk Score Adjustments per C21 §5.1. The list propagates with W3C Baggage across trust boundaries. The Helper API's add\_risk\_adjustment method (C33 §I.4.6) updates this baggage value.

§5.1.14.X.6.5 — Helper API access. The Telemetry Record is NOT a span attribute. Application code SHALL access the Telemetry Record only through the Helper API per §5.1.14.X.3.2. Spans and Telemetry Records are coordinated by SAI internals but the application boundary is the Helper API surface only.

# §5.1.14.X.7 Provenance Attribution

§5.1.14.X.7.1 — Every SADAR-conformant span SHALL carry the telemetry.origin.environment attribute identifying the issuing entity. The attribute uses the structured form:

<entity\_urn>:<agent\_urn>:<environment\_id>

Where:

entity\_urn — registry URN of the publishing entity.

agent\_urn — agent URN within the entity.

environment\_id — implementation-specific environment identifier (e.g., production, staging, development), permitting attribution to specific deployment environments within an entity.

§5.1.14.X.7.2 — Symmetry. Both ends of every SADAR-conformant interaction emit spans with this attribute. The receiving entity's span (capturing receipt of an invocation) and the sending entity's span (capturing dispatch of the invocation) each carry their own telemetry.origin.environment value. Cross-checking these enables tamper-evident audit.

§5.1.14.X.7.3 — Non-redactable. telemetry.origin.environment and telemetry.origin.agent are non-redactable structural fields per §5.1.14.X.10.8.3. Repatriation servers cannot redact these values.

# §5.1.14.X.8 Persistence Backend

§5.1.14.X.8.1 — At finalize (per C33 §I.6), the Helper persists the Telemetry Record to a storage backend. The storage backend choice is implementer concern; conformant choices include relational databases, document stores, time-series databases, append-only logs, and others.

§5.1.14.X.8.2 — Persistence guarantees. The implementation SHALL provide durability appropriate to its operational context. The spec does not mandate specific durability guarantees (e.g., synchronous fsync) since storage backends vary; the implementation SHOULD document its durability profile.

§5.1.14.X.8.3 — Failed-invocation persistence. Records for failed invocations (invocation\_status of FAILED, TIMED\_OUT, or NOT\_INVOKED) SHALL be persisted on the same path as successful invocations. Audit-trail completeness requires that failures are not silently dropped.

§5.1.14.X.8.4 — Persistence failure handling. If the storage backend rejects or fails to acknowledge the write, finalize() SHALL return an error per C33 §I.7. The SAI invocation outcome itself is not affected (the response has already been returned to the application). Audit-trail completeness in the face of storage failure is implementer concern; retry-with-backoff or queueing for asynchronous reattempt are recommended but not mandated.

# §5.1.14.X.9 Sensitivity Classification

§5.1.14.X.9.1 — Telemetry Records may contain sensitive business information. Implementations SHOULD apply sensitivity classification appropriate to their operational context, restricting access to Telemetry Records based on classification. The spec does not normatively define classification levels — those are organizational concern — but recognizes the operational reality that not all audit records should be visible to all administrators.

§5.1.14.X.9.2 — Risk Score implications. The risk\_adjustments field may contain reason codes and context strings that describe sensitive risk-relevant observations (e.g., detected bias, attestation anomalies, policy violations). Where the implementation classifies Telemetry Records by sensitivity, risk\_adjustments content SHOULD be considered in the classification.

# §5.1.14.X.10 Telemetry Repatriation

§5.1.14.X.10.1 — Telemetry Repatriation is the asynchronous, governed return of trace fragments from a server to the requester at trust boundary crossings. It enables observability across organizational boundaries while preserving each side's control over what is disclosed.

## §5.1.14.X.10.2 Bilateral Declaration

§5.1.14.X.10.2.1 — Repatriation participation is bilaterally declared via four Protocol NFRs in C19 §8.3:

repatriation\_participation (server) — boolean. Whether the server participates.

repatriation\_endpoint (requester) — URL/URN. The requester's endpoint for receiving repatriated payloads.

repatriation\_redacted\_fields (server) — array of field names. Fields the server will redact in repatriated payloads.

repatriation\_required\_unredacted\_fields (requester) — array of field names. Fields the requester requires returned unredacted.

§5.1.14.X.10.2.2 — Capability vs advertisement. Every conformant server has the technical capability for repatriation. The repatriation\_participation flag controls advertisement (and therefore matching), not capability.

## §5.1.14.X.10.3 Bilateral Match

§5.1.14.X.10.3.1 — Repatriation is admissible iff:

Phase 1 (participation): server.repatriation\_participation = true AND requester.repatriation\_endpoint is non-empty.

Phase 2 (field intersection): requester.repatriation\_required\_unredacted\_fields ∩ server.repatriation\_redacted\_fields = ∅.

§5.1.14.X.10.3.2 — Match failure. Repatriation is not admissible if either phase fails. There is no partial repatriation; either it applies fully or not at all.

§5.1.14.X.10.4 — The bilateral match is evaluated at discovery time, when the requester selects the server. Repatriation applicability is determined for the duration of the discovery TTL.

## §5.1.14.X.10.5 Repatriation Endpoint

§5.1.14.X.10.5.1 — The repatriation\_endpoint is the URL or service URN where the requester accepts repatriated payloads. The endpoint authenticates incoming repatriations per scope.md §5.1.8.1, with the server presenting an OIDC token bearing the urn:sadar:scope:v1:telemetry:repatriate scope.

§5.1.14.X.10.5.2 — Asynchronous return. Repatriation is asynchronous to the original invocation's response. The server MAY return its primary invocation response and then initiate repatriation as a separate operation; the requester's repatriation endpoint receives the payload on its own timeline.

## §5.1.14.X.10.6 Repatriation Payload

§5.1.14.X.10.6.1 — A repatriation payload is structured as OTel span data. Specifically, the payload is a collection of spans the server emitted in the course of processing the invocation, with redaction applied per the server's declared repatriation\_redacted\_fields.

§5.1.14.X.10.6.2 — Trace continuity. Repatriated spans preserve their original trace\_id and span\_id values, enabling the requester to assemble a complete trace at its observability backend. The requester's OTel collector integrates repatriated spans with its locally-emitted spans by trace\_id correlation.

§5.1.14.X.10.6.3 — All-spans-returned. The server SHALL return all spans it emitted in the course of processing the invocation, not a curated subset. Where the server wishes to limit disclosure, the mechanism is field-level redaction (repatriation\_redacted\_fields), not span-level filtering.

## §5.1.14.X.10.7 Trigger Conditions

§5.1.14.X.10.7.1 — The server triggers repatriation when an outgoing call from its searchAndInvoke implementation completes (success or failure), per the bilateral match evaluated at discovery.

§5.1.14.X.10.7.2 — Per-link semantics. In multi-link chains, each server evaluates repatriation against its immediate caller (not against the chain originator). Each link repatriates only to its own immediate caller. The full chain assembles at the originator through the natural propagation of repatriated payloads up the chain.

§5.1.14.X.10.7.3 — Failure inclusion. Repatriation is required even when the primary invocation produced a failed response. A failed invocation's span carries diagnostic value (timing, error status, partial completion); repatriation preserves that diagnostic value across boundaries. Internal error detail in the spans is subject to the server's disclosure policy via repatriation\_redacted\_fields.

§5.1.14.X.10.7.4 — Trigger timing. (NEW IN v5.2, per back-port from C33 v1 Part IV.) Repatriation is triggered upon each agent/tool/resource call return — success or failure — when bilateral applicability conditions of §5.1.14.X.10.3.1 are satisfied. The trigger is keyed to the close of the OTel span representing the outgoing invocation, NOT to Telemetry Record finalize (which governs local audit-artifact persistence per §5.1.14.X.8). The detection mechanism for span close is implementer choice; three acceptable patterns are documented in C33 §II.3 (OTel span-close subscription, SAI-internal span lifecycle tracking, explicit Finalize-with-span call). C33 §II is the authoritative specification of repatriation trigger semantics.

The distinction between Telemetry Record finalize and repatriation trigger is important: Telemetry Record finalize is local audit artifact persistence and occurs at a single point per invocation; repatriation trigger occurs at each span close (which may be multiple per invocation if SAI initiates multiple downstream calls). Conflating the two causes implementations to miss repatriation opportunities on intermediate spans.

## §5.1.14.X.10.8 Redaction

§5.1.14.X.10.8.1 — Redaction policy is server-declared via repatriation\_redacted\_fields. The list enumerates field names of OTel span attributes the server will redact in repatriated payloads.

§5.1.14.X.10.8.2 — Redaction implementation. Redaction may be applied as null replacement, marker substitution (e.g., "[[REDACTED]]"), or other documented forms. The redaction marker selection is implementer choice. Implementations SHOULD document their redaction marker convention.

§5.1.14.X.10.8.3 — Non-redactable structural fields. The following fields are non-redactable; they MUST NOT appear in repatriation\_redacted\_fields, and the server SHALL preserve them in repatriated payloads regardless of redaction policy:

trace\_id, span\_id, parent\_span\_id — for trace reconstruction.

start\_time, end\_time, duration — for timing analysis.

status — for success/failure determination.

name — for operation identification.

telemetry.origin.environment, telemetry.origin.agent, telemetry.repatriated — for provenance.

telemetry.disclosure.policy\_id — for disclosure audit.

Registry-side validation (C19 §15.2.2) enforces the non-redactable rule at manifest publication. Manifests declaring any non-redactable structural field in repatriation\_redacted\_fields are rejected.

## §5.1.14.X.10.9 Replay Protection

§5.1.14.X.10.9.1 — Repatriation payloads are signed by the server using its registry-published signing key. The requester's repatriation endpoint verifies the signature on receipt; payloads with invalid signatures are rejected.

§5.1.14.X.10.9.2 — Replay timestamps. Repatriation payloads include a sender timestamp; the requester's endpoint may reject payloads outside an acceptance window (typically minutes to hours, implementer-chosen) to prevent replay attacks on the repatriation channel.

## §5.1.14.X.10.10 Conformance Profile

§5.1.14.X.10.10.1 — Conformance to repatriation requires all of: bilateral declaration accuracy (manifests reflect actual behavior), non-redactable structural field preservation (§5.1.14.X.10.8.3), trigger correctness per §5.1.14.X.10.7 with the trigger-timing clarification at §5.1.14.X.10.7.4, payload signature verification, and asynchronous transmission.

§5.1.14.X.10.10.2 — Per-link conformance. Conformance is evaluated per-link, not per-chain. Each server is independently conformant or non-conformant to the repatriation requirements; chain composition does not affect per-link evaluation.

— — —

# Change Log

|  |  |  |
| --- | --- | --- |
| **Version** | **Date** | **Changes** |
| 5.2.0 | April 2026 | Combined back-port transition from v5.0. Adds two backwards-compatible additions: (1) From C33 v1 Part IV: §5.1.14.X.10.7.4 Trigger timing — clarifies that repatriation is keyed to OTel span close, distinct from Telemetry Record finalize. Resolves an implicit gap in v5.0 where readers might conflate the two artifacts. (2) From C21 v1 §10: Two new fields in Telemetry Record schema (§5.1.14.X.2 Outcome Phase) — risk\_score\_at\_finalize (decimal in [0.0, 1.0] or null) and risk\_adjustments (array of Adjustment objects). Captures Risk Score state at finalize for audit visibility per C21 §10. No intermediate v5.1 issued. Both back-ports are backwards compatible — implementations conformant to v5.0 remain conformant to v5.2; v5.2 implementations gain the new fields and the trigger-timing clarification. |
| 5.0.0 | April 2026 | Major revision establishing field-list bilateral matching for repatriation (replaces Compliance Levels), symmetric provenance attribution, non-redactable structural field set, binary participation flag, OTel span structure with provenance attribution, Helper API requirement, all-spans-returned requirement, and conformance profile. |
| 1.0–4.0 | March–April 2026 | Iterative drafts. See prior versions for historical increments. |

*© 2026 Cognita AI Inc. All rights reserved. SADAR™ is a trademark of Cognita AI, Inc. CogniWeave™ is a trademark of Cognita AI, Inc. All other trademarks are the property of their respective owners. Licensed under the Community Specification License 1.0.*