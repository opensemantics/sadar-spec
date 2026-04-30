SADAR™ searchAndInvoke Telemetry and Authentication

*Specification — Version 1.1.0*

Status: Draft • Tracking ID: C33 • Back-port from C21 §10.1.4 applied

*April 2026*

*© 2026 Cognita AI Inc. All rights reserved. SADAR™ is a trademark of Cognita AI, Inc. CogniWeave™ is a trademark of Cognita AI, Inc. All other trademarks are the property of their respective owners.*

## Document Overview

This document specifies three concerns related to the operation of a SADAR-conformant searchAndInvoke (SAI) implementation:

Part I — Telemetry Record Helper API. The boundary between application code and SAI for constructing the per-invocation Telemetry Record (defined in C20 §5.1.14.X.2). The Helper API enforces field validation and phase-immutability so that the record meets §5.1.14.X.6.5 conformance. v1.1 adds the add\_risk\_adjustment method per §I.4.6.

Part II — Repatriation Trigger Specification. The normative requirement that SAI repatriates telemetry on each agent/tool/resource call return, plus three acceptable implementation patterns for detecting that return.

Part III — Authentication Scopes. The urn:sadar:scope:v1:\* namespace and initial set of OIDC scope values used by SAI when authenticating to agents/tools/resources via the §5.1.8 authentication baseline.

**Cross-references.** C20 §5.1.14.X.2 (Telemetry Record schema, including risk\_score\_at\_finalize and risk\_adjustments fields per C20 v5.2). C20 §5.1.14.X.6 (OTel span structure). C20 §5.1.14.X.6.5 (Helper API requirement). C20 §5.1.14.X.10 (Telemetry Repatriation). C19 (NFR Schema, including the is\_default and impact\_score fields per C19 v1.2). C21 (Risk Score Specification — origin of the add\_risk\_adjustment method). scope.md §5.1.8 (authentication baseline). scope.md §5.1.13 (searchAndInvoke). C31 §4.6.X.10.7 (urn:sadar:error:v1:\* namespace precedent).

— — —

# Part I — Telemetry Record Helper API

## I.1 Purpose and Scope

I.1.1 — This Part defines the Helper API surface required by C20 §5.1.14.X.6.5: implementations SHALL access the Telemetry Record only through a SADAR-defined Helper API that validates field types, enforces phase-immutability, and prevents application code from setting fields that are system-managed.

I.1.2 — Scope of Part I. The Helper API governs how application code interacts with the Telemetry Record during an SAI invocation. It does NOT govern repatriation (Part II), persistence-backend choice (C20 §5.1.14.X.8), span emission (C20 §5.1.14.X.6.1), or any concern unrelated to application-driven Telemetry Record population.

I.1.3 — Scope boundary with SAI internal logic. The Helper API is the application/SAI boundary. SAI itself populates many Telemetry Record fields directly without application involvement (timestamps, candidate counts, resolver outcomes, invocation status, etc.). The Helper API surfaces methods only for fields that application code may legitimately set or augment; SAI-only fields are not exposed through the application-facing Helper API surface and are populated through SAI's internal access path.

## I.2 Helper Instance Lifecycle

I.2.1 — Per-invocation scoping. A Helper instance SHALL be bound to exactly one SAI invocation. SAI creates a Helper at the beginning of the invocation, the Helper accumulates the Telemetry Record over the invocation's lifetime, and the Helper is finalized at the end of the invocation. Helper instances SHALL NOT be shared across invocations or reused after finalization.

I.2.2 — Concurrent access. The Helper instance SHALL be safe against single-invocation sequential use only. Concurrent operations on a single Helper instance from multiple threads or goroutines are programmer error; the Helper SHALL detect such use and reject with the error code defined in §I.7. Multi-invocation concurrency uses multiple Helper instances; the Helper API does not provide cross-instance synchronization.

I.2.3 — Construction is SAI-internal. Application code SHALL NOT construct Helper instances directly. The Helper is provided to application code by SAI as part of the SAI invocation context. Implementations may surface this through dependency injection, a context object passed to the application handler, language-idiomatic patterns, or other mechanisms appropriate to the host environment; the choice is implementer's.

## I.3 Phase Model

I.3.1 — The Telemetry Record is populated across four phases corresponding to the SAI invocation lifecycle. Each phase admits specific Helper API operations; each phase finalizes a defined set of fields, after which they SHALL NOT be modified.

|  |  |  |  |
| --- | --- | --- | --- |
| **Phase** | **What's happening** | **Application interaction** | **Fields finalized at end of phase** |
| Search | SAI is searching the registry and applying NFR matching to produce the candidate set. | Application MAY contribute search criteria refinements via the set\_criteria\_extension method (§I.4.1). Most fields in this phase are SAI-populated. | timestamp\_search, criteria, candidate\_count, qualified\_count |
| Selection | SAI is invoking the resolver to produce the final selection from the candidate set, or recording a selection failure. | Application typically does not interact with the Helper in this phase. SAI records resolver outcomes directly. | resolver\_id, resolver\_version, resolver\_location, resolver\_endpoint, selected\_entity\_id, selected\_agent\_id, registry\_source, third\_party\_registry, sadar\_process\_id, selection\_exception |
| Invocation | SAI is dispatching the call to the selected target and awaiting response. | Application MAY set input\_summary via set\_input\_summary (§I.4.2) at the start of this phase, set output\_summary via set\_output\_summary (§I.4.3) when the response is received, and add Risk Score Adjustments via add\_risk\_adjustment (§I.4.6) in response to risk-relevant observations during the invocation. | timestamp\_invoked, timestamp\_returned, duration\_seconds, invocation\_status |
| Outcome | SAI is finalizing the record and persisting it to the storage backend. | Application MAY set output\_summary if not already set in Invocation phase, add custom audit metadata via add\_audit\_metadata (§I.4.4), or add Risk Score Adjustments via add\_risk\_adjustment (§I.4.6). Helper.finalize() closes the phase and triggers persistence. | (record sealed; no further fields admitted) |

I.3.2 — Phase transitions are SAI-driven. SAI moves the Helper between phases based on its own lifecycle. Application code does not request phase transitions; application code only invokes phase-appropriate methods at appropriate points in the invocation flow.

I.3.3 — Out-of-phase calls. Calling a Helper method in a phase other than the one in which it is admissible is a validation error. The Helper SHALL reject the call with the error code in §I.7 and SHALL NOT modify the Telemetry Record. The error response SHALL identify both the method and the current phase.

I.3.4 — Once a phase ends and its fields are finalized, those fields SHALL be immutable. Subsequent calls attempting to modify finalized fields are out-of-phase calls per §I.3.3.

## I.4 Application-Facing Methods

The Helper API exposes six application-facing methods. Each is admissible only in specific phases; out-of-phase calls are rejected per §I.3.3.

I.4.1 — set\_criteria\_extension. Permits application code to augment the search criteria SAI passes to the registry with application-specific extensions. The method takes a structured extension object and merges it into the criteria field.

Signature (language-neutral):

set\_criteria\_extension(extension: object) → result | error

Parameters:

|  |  |  |
| --- | --- | --- |
| **Parameter** | **Type** | **Definition** |
| extension | object | Structured criteria extension. Implementer-defined schema; the Helper validates structure (must be an object, no null, no circular references) but not content. |

Phase: Search.

Errors: out-of-phase (called outside Search), invalid extension (structurally invalid object).

I.4.2 — set\_input\_summary. Application code SHOULD set input\_summary at the beginning of the Invocation phase, providing the structured summary of input data per C20 §5.1.14.X.4. Application code is responsible for redaction per the redaction rules of C20 §5.1.14.X.4.2.

Signature:

set\_input\_summary(summary: object) → result | error

|  |  |  |
| --- | --- | --- |
| **Parameter** | **Type** | **Definition** |
| summary | object | Structured input summary. Field structure per C20 §5.1.14.X.4 and the data classification annex (forthcoming). The Helper validates that the value is an object and that any redaction markers are syntactically valid; semantic redaction policy enforcement is the application's responsibility. |

Phase: Invocation.

Errors: out-of-phase, invalid summary, summary already set (input\_summary may be set at most once per invocation).

I.4.3 — set\_output\_summary. Application code SHOULD set output\_summary when the response is received, providing the structured summary of output data per C20 §5.1.14.X.4.

Signature:

set\_output\_summary(summary: object) → result | error

Parameters: as for set\_input\_summary, but for the output side.

Phase: Invocation or Outcome.

Errors: out-of-phase, invalid summary, summary already set.

I.4.4 — add\_audit\_metadata. Application code MAY add custom audit metadata fields not defined in the C20 schema. Custom fields are stored in a designated audit\_metadata sub-object on the Telemetry Record. Custom field names SHALL be IRI-namespaced per the application's namespace; SADAR-defined field names are reserved.

Signature:

add\_audit\_metadata(key: string (IRI), value: any) → result | error

|  |  |  |
| --- | --- | --- |
| **Parameter** | **Type** | **Definition** |
| key | string (IRI) | IRI-namespaced metadata key. SHALL NOT use the urn:sadar: prefix. |
| value | any JSON-serializable | Metadata value. The Helper validates that the value is JSON-serializable (no circular references, no functions/classes). |

Phase: Outcome (not earlier — audit metadata is finalization-time content).

Errors: out-of-phase, invalid key (uses reserved namespace, not a valid IRI), invalid value, key already set.

I.4.5 — get\_invocation\_context. Application code MAY read certain invocation-context fields from the Helper. Read access is granted only for fields the application has legitimate need to know — specifically, the SCT chain root jti (for application-side correlation), the resolver-selected target identity (for application-side observability), and the sadar\_process\_id. Other fields are not exposed through the read surface.

Signature:

get\_invocation\_context() → context\_object

Returned context\_object fields:

|  |  |  |
| --- | --- | --- |
| **Field** | **Type** | **Definition** |
| sct\_chain\_root\_jti | string (JWT ID) | The chain root jti for application-side correlation. Available after Selection phase completes. |
| selected\_entity\_id | string (URN) | The selected target entity URN. Available after Selection phase. |
| selected\_agent\_id | string (URN) | The selected target agent URN. Available after Selection phase. |
| sadar\_process\_id | string (URN) or null | The SADAR process identifier carried in the SCT, if any. |

Phase: Selection, Invocation, or Outcome (returns nulls during Search).

Errors: none under correct usage. Returns a snapshot — repeated calls during the same phase return consistent values.

I.4.6 — add\_risk\_adjustment. (NEW IN v1.1, per back-port from C21 §10.1.4.) Application code MAY emit a Risk Score Adjustment during the invocation in response to risk-relevant observations. The Helper appends the Adjustment to the Telemetry Record's risk\_adjustments field (per C20 v5.2 §5.1.14.X.2) and propagates it to OTel Baggage at urn:sadar:baggage:v1:risk\_adjustments per C21 §5.1.

Signature:

add\_risk\_adjustment(adjustment: Adjustment) → result | error

|  |  |  |
| --- | --- | --- |
| **Parameter** | **Type** | **Definition** |
| adjustment | Adjustment object | Risk Score Adjustment per the schema in C21 §3.1. The Helper validates structure (delta in [-1.0, 1.0], ttl\_seconds ≥ 0, timestamp RFC 3339, reason a well-formed IRI, adjustment\_id a well-formed URN, required fields present); semantic content (reason categorization, context phrasing) is application concern. |

Phase: Invocation or Outcome.

Errors: out-of-phase (called during Search or Selection), malformed Adjustment (per C21 §8.4 — delta out of range, malformed timestamp, malformed reason IRI, missing required fields), duplicate adjustment\_id (same adjustment\_id added previously in this invocation).

Behavior: on success, the Helper (a) appends the Adjustment to the local risk\_adjustments field of the Telemetry Record; (b) updates the OTel Baggage value at urn:sadar:baggage:v1:risk\_adjustments to include the new Adjustment, preserving any pre-existing Adjustments propagated from upstream; (c) preserves the Adjustment list ordering (append-only). The Helper does NOT compute a current accumulated scalar at this point; accumulation is implementer concern per C21 §6 and is performed at SCT chain link issuance per C21 §5.3.

Multiple Adjustments per invocation are permitted. An invocation may emit zero, one, or many Adjustments through this method.

Note: This method makes the Helper API the canonical surface through which application-driven Risk Score Adjustments enter the flow. Adjustments emitted by SAI internals (e.g., when SAI detects a manifest verification failure) bypass this method and are written through SAI's internal access path.

## I.5 SAI-Internal Population

I.5.1 — The following Telemetry Record fields are populated by SAI directly and are NOT exposed through the application-facing Helper API:

|  |  |
| --- | --- |
| **Field(s)** | **How and when populated** |
| schema\_version | Set at Helper construction. |
| timestamp\_search | Set at Search phase start. |
| calling\_manifest\_id | Set at Helper construction. |
| criteria | Populated by SAI from the search criteria; application contributions arrive via set\_criteria\_extension and are merged in. |
| candidate\_count | Set by SAI on receiving registry response. |
| qualified\_count | Set by SAI after NFR matching. |
| resolver\_id, resolver\_version, resolver\_location, resolver\_endpoint | Set by SAI when the resolver is invoked. |
| selected\_entity\_id, selected\_agent\_id, registry\_source, third\_party\_registry | Set by SAI when the resolver returns its selection. |
| sadar\_process\_id | Set by SAI from the SCT. |
| timestamp\_invoked, timestamp\_returned, duration\_seconds | Set by SAI based on Invocation phase boundaries. |
| selection\_exception | Set by SAI if the resolver raises a SelectionException. |
| invocation\_status | Set by SAI at Outcome phase based on invocation result. |
| risk\_score\_at\_finalize | Set by SAI at finalize per C20 v5.2 §5.1.14.X.2 and C21 §10. Computed from the live Adjustment list using the implementer's accumulation algorithm. |

I.5.2 — SAI internal access. SAI implementations have direct mutable access to the Telemetry Record at construction time and through a SAI-internal access path. This access path is implementer choice and is not exposed to application code. The integrity of the SAI-internal access path is implementer's responsibility; it is outside the scope of this specification.

## I.6 Finalization

I.6.1 — Finalize triggers persistence. The Helper.finalize() method, called by SAI at the end of the Outcome phase, SHALL:

Validate that all required fields are populated. Fields with declared defaults that have not been set may be filled with their defaults at finalization. Fields without defaults that have not been set cause finalization failure per §I.7.

Compute any derived fields (e.g., duration\_seconds from timestamp\_invoked and timestamp\_returned).

Compute risk\_score\_at\_finalize by applying the implementer's accumulation algorithm to the live Adjustment list (per C21 §6) and clamping the result to [0.0, 1.0] (per C21 §2.3). The Adjustment list itself (in the risk\_adjustments field) is preserved for audit.

Seal the record against further modification.

Trigger persistence to the configured storage backend per C20 §5.1.14.X.8.

I.6.2 — Persistence is required for both successful and failed invocations. Per C20 §5.1.14.X.1.1, every invocation produces a Telemetry Record regardless of outcome. Finalize SHALL persist records with invocation\_status of SUCCESS, FAILED, TIMED\_OUT, or NOT\_INVOKED.

I.6.3 — Finalize and Repatriation. The finalize() method governs Telemetry Record persistence (the local audit artifact). Repatriation — the cross-trust-boundary delivery of telemetry to the immediate caller's repatriation endpoint — is governed separately by Part II of this document and is keyed to the OTel span lifecycle, not to the Telemetry Record finalize. The two are related (both are invocation-end actions) but distinct (local persistence vs. cross-boundary delivery; Telemetry Record vs. OTel spans).

I.6.4 — Persistence failure. If the storage backend fails to persist the record, finalize() SHALL return an error. The SAI invocation outcome itself is not affected — the invocation has already returned to the application by this point. Audit-trail completeness in the face of storage failure is the implementer's durability concern; the spec recommends but does not mandate retry-with-backoff or queueing for asynchronous reattempt.

## I.7 Error Codes

I.7.1 — The following error codes are normatively defined for Helper API errors. Codes are under the urn:sadar:error:v1:helper:\* subnamespace, following the precedent established in C31 §4.6.X.10.7.

|  |  |
| --- | --- |
| **Error Code** | **Meaning** |
| urn:sadar:error:v1:helper:out\_of\_phase | Method called in a phase that does not admit it. §I.3.3. |
| urn:sadar:error:v1:helper:concurrent\_access | Concurrent operations detected on a single Helper instance. §I.2.2. |
| urn:sadar:error:v1:helper:invalid\_value | A method received a value that fails structural validation (not an object where required, not JSON-serializable, IRI where required but not well-formed, etc.). For Risk Score Adjustments, this includes the malformed-Adjustment cases of C21 §8.4. |
| urn:sadar:error:v1:helper:reserved\_namespace | add\_audit\_metadata called with a key in the urn:sadar: reserved namespace. §I.4.4. |
| urn:sadar:error:v1:helper:already\_set | Single-set field (input\_summary, output\_summary, audit\_metadata key) was already set and cannot be overwritten. For Risk Score Adjustments, also raised when an adjustment\_id is reused within the same invocation. |
| urn:sadar:error:v1:helper:missing\_required\_field | finalize() called but a required field without a default remains unset. §I.6.1. |
| urn:sadar:error:v1:helper:persistence\_failure | Storage backend rejected or failed to acknowledge the record write. §I.6.4. |

I.7.2 — Error response structure. Errors SHALL include the machine-readable error code, a human-readable message, and the method name and current phase where applicable. The Helper SHALL NOT silently no-op on validation failure.

— — —

# Part II — Repatriation Trigger Specification

## II.1 Purpose and Scope

II.1.1 — This Part specifies the normative requirement governing when SAI repatriates telemetry data, and enumerates the acceptable implementation patterns by which SAI may detect that requirement's trigger condition.

II.1.2 — Scope of Part II. This Part specifies WHEN repatriation occurs. The WHAT (which fields are returned, redaction policy, transmission protocol) is governed by C20 §5.1.14.X.10. The WHO (bilateral matching for repatriation eligibility) is governed by C19 §8.3 and C20 §5.1.14.X.10.4. Part II adds the trigger semantics and resolves a gap left implicit in C20.

## II.2 Normative Trigger Requirement

II.2.1 — Per-call repatriation trigger. SAI SHALL repatriate telemetry upon each agent/tool/resource call return — success or failure — when bilateral matching at discovery indicated repatriation is applicable to the invocation.

II.2.2 — Bilateral applicability. Repatriation is applicable iff both conditions hold per C20 §5.1.14.X.10.4.2:

The server declared repatriation\_participation = true (C19 §8.3.1).

The requester (calling SAI) declared a non-empty repatriation\_endpoint (C19 §8.3.2).

And the field-list intersection check of C20 §5.1.14.X.10.4.2 Phase 2 succeeded — the requester's repatriation\_required\_unredacted\_fields and the server's repatriation\_redacted\_fields have empty intersection.

II.2.3 — When applicability fails. If repatriation is not applicable, SAI SHALL NOT repatriate. No partial-repatriation, no fallback, no opt-in-by-default. The bilateral declaration is the gating condition.

II.2.4 — Per-link semantics. In multi-link chains, each link's SAI invocation independently evaluates applicability against its immediate caller (not against the chain originator). Each link repatriates only to its own immediate caller. The full chain assembles at the originator through the natural propagation of repatriated payloads up the chain. SAI does NOT need knowledge of chain length, position, or completion to satisfy this requirement.

II.2.5 — Success and failure. Repatriation triggers on both successful and failed downstream returns. A failed invocation's span carries its own diagnostic value (timing, error status, partial completion); per C20 §5.1.14.X.10.7.3, repatriation is required even when the primary invocation produced a failed response, with internal error detail subject to the server's disclosure policy.

## II.3 Acceptable Implementation Patterns

II.3.1 — The spec is implementation-pattern-agnostic for the trigger detection mechanism. Three patterns are acceptable. Implementers MAY use any of them; mixing patterns within a single implementation is permitted where appropriate. Other patterns satisfying the §II.2 requirement are also acceptable; these three are documented as a non-exhaustive catalog of reasonable approaches.

### II.3.2 Pattern A — OTel Span-Close Subscription

SAI subscribes to OTel span-close events for the spans it creates. When a span representing an outgoing agent/tool/resource call closes, the subscription handler evaluates applicability (§II.2.2) and, if applicable, triggers repatriation.

Properties:

Decoupled — the trigger fires automatically on span close; application code is unaware.

Failure-inclusive — span close fires regardless of invocation success or failure.

OTel-SDK-dependent — requires the host OTel SDK to expose a span-close hook surface. Java, Python, Go OTel SDKs each support this through different APIs; portability across languages requires SDK abstraction at the SAI layer.

No application action required — the application code invoking the agent/tool/resource has no responsibility for triggering repatriation.

Recommended for implementations using OTel SDKs that expose lifecycle hooks. The reference implementation provided alongside this specification uses Pattern A.

### II.3.3 Pattern B — SAI-Internal Span Lifecycle Tracking

SAI tracks the spans it creates within its own invocation lifetime, observes their close transitions through its own internal span-management code (without subscribing to OTel hooks), and triggers repatriation when each closes.

Properties:

No OTel hook dependency — works on any OTel SDK regardless of hook surface.

SAI-managed lifecycle — SAI must explicitly track span open/close in its own data structures, which adds bookkeeping.

Failure-inclusive — span close triggered by SAI works the same regardless of underlying invocation outcome.

No application action required.

Recommended for implementations on OTel SDKs without lifecycle hook support, or where SAI already maintains explicit span tracking for other reasons.

### II.3.4 Pattern C — Explicit Finalize-with-Span Call

SAI exposes a Finalize-with-span function that takes the span as input, evaluates applicability (§II.2.2), and triggers repatriation. SAI calls this function from its own internal flow when an outgoing call returns. The function may also be called by the application directly, but doing so is not the canonical pattern.

Properties:

No OTel hook dependency.

No internal span lifecycle tracking required by SAI; the function handles each span as it is presented.

Application-direct use risk — if the application is expected to call this and an exception path bypasses the call, repatriation is missed. SAI-internal use does not have this failure mode. Implementations MAY expose Pattern C as a SAI-internal mechanism only and not surface it to applications.

Compatible with any OTel SDK.

Recommended where SAI invokes a call-completion handler that naturally maps to a Finalize-with-span function, and where the function is invoked by SAI internally (not by application code).

## II.4 Pattern Selection Guidance (Non-Normative)

II.4.1 — The choice among Patterns A, B, and C is implementer discretion. Considerations include:

OTel SDK hook surface in the host language and runtime.

Whether SAI already maintains explicit span tracking for other reasons (favors Pattern B).

Whether the host environment surfaces span-close hooks naturally and reliably (favors Pattern A).

Operational concerns about asynchronous handler reliability under high load (consider Pattern C with SAI-internal invocation for deterministic timing).

II.4.2 — Conformance is established by satisfying §II.2 regardless of pattern. The pattern itself is not a conformance criterion.

## II.5 Repatriation Mechanics Reference

The mechanics of repatriation — the protocol for transmitting data to the requester's repatriation endpoint, the redaction rules, the OTel span structure, the failure handling — are defined in C20 §5.1.14.X.10. Part II of this document does not duplicate that content; it specifies only the trigger condition.

Implementers SHOULD treat C20 §5.1.14.X.10 (with the v5.2 trigger-timing clarification at §5.1.14.X.10.7.4) and Part II of this document as a single conceptual unit when implementing SAI repatriation behavior.

— — —

# Part III — Authentication Scopes

## III.1 Purpose and Scope

III.1.1 — This Part establishes the urn:sadar:scope:v1:\* namespace for SADAR-defined OIDC scope claim values, and enumerates the initial set of scopes used by SAI in its authentication interactions with agents, tools, and resources.

III.1.2 — Scope of Part III. This Part defines scope vocabulary and TTL bounds. It does NOT define the authentication baseline — OIDC Client Credentials and mTLS per scope.md §5.1.8 — which is foundational and unchanged by this document. SADAR scope values are layered on top of the §5.1.8 OIDC flow as the standard OIDC scope claim mechanism.

## III.2 Authentication Architecture

III.2.1 — Token issuer is the target. When SAI invokes an agent, tool, or resource, SAI authenticates to the target's authentication endpoint as declared in the target's registry manifest per scope.md §5.1.8. The target's authentication endpoint issues the access token. The registry is NOT the token issuer; the registry only stores the target's manifest, which in turn declares where the target's authentication endpoint lives.

III.2.2 — Federation / Registry of Registries. In federated topologies, registries themselves have manifests in the Registry of Registries (per scope.md §5.1.11). A registry's manifest declares its own authentication endpoints, NFRs, and operational details. When SAI needs to interact with a federated registry, SAI authenticates to that registry's authentication endpoint per §5.1.8 — same pattern as authenticating to any other agent/tool/resource. Federation does not introduce a separate token system.

III.2.3 — Registry isolation preserved. SAI authenticates to each target as needed; the registry does not hold credentials, usage tokens, or runtime authentication state per scope.md §5.1.12. Token storage and lifecycle are local to the SAI implementation.

## III.3 Scope Namespace

III.3.1 — All SADAR-defined OIDC scopes follow:

urn:sadar:scope:v1:{category}:{operation}

Where {category} groups related operations and {operation} names the specific authorized action. The pattern parallels urn:sadar:nfr:v1:{category}:{attribute} (C19) and urn:sadar:error:v1:{category}:{code} (C31).

III.3.2 — Custom scopes. Implementations and ecosystem participants MAY define custom scopes using their own namespace; custom scopes SHALL NOT use the urn:sadar: prefix. Targets receiving tokens with unrecognized custom scopes SHOULD ignore them rather than reject the token, unless the target's policy explicitly requires a recognized scope.

## III.4 Initial Scope Set

III.4.1 — The following scopes are normatively defined for SAI use in v1 of this specification. Future versions MAY add scopes; such additions are MINOR-version increments per the C19 §16 versioning policy.

|  |  |
| --- | --- |
| **Scope Value** | **Authorized Operation** |
| urn:sadar:scope:v1:registry:search | Submit a discovery query to the target registry. Used when SAI or a registry queries another registry in a federated topology. |
| urn:sadar:scope:v1:registry:resolve\_manifest | Retrieve a specific manifest by URN from the target registry. |
| urn:sadar:scope:v1:registry:list\_registries | Query the target registry's view of the Directory of Authorized Registries. |
| urn:sadar:scope:v1:registry:health | Check the target registry's health/availability endpoints. |
| urn:sadar:scope:v1:telemetry:repatriate | Send repatriated telemetry to a target's repatriation endpoint. Used by SAI when delivering repatriated payloads per Part II of this document and C20 §5.1.14.X.10. |
| urn:sadar:scope:v1:invocation:invoke | Invoke a target agent, tool, or resource through its declared invocation endpoint. The default scope for SAI invocation operations. |

III.4.2 — Scope minimization. SAI SHOULD request only the scopes required for the immediate operation. A token authorizing urn:sadar:scope:v1:invocation:invoke does NOT implicitly authorize urn:sadar:scope:v1:telemetry:repatriate; both are obtained as needed.

III.4.3 — Scope multiplicity. A single token MAY carry multiple scopes per OIDC convention. SAI MAY request multiple scopes in a single token request when the same target authentication endpoint will issue tokens covering related operations.

## III.5 Token TTL Bounds

III.5.1 — Minimum TTL. Token TTLs SHALL NOT be less than 60 seconds. Below this bound, tokens become operationally meaningless — every operation effectively re-authenticates, which adds round-trip cost without security benefit.

III.5.2 — Maximum TTL. Token TTLs SHALL NOT exceed 24 hours (86,400 seconds). Above this bound, tokens outlive most reasonable threat-response windows; revocation latency becomes unacceptable for operational security.

III.5.3 — Recommended default. Implementations SHOULD use a default TTL of 15 minutes (900 seconds) absent specific reason to deviate. The recommended default is non-normative; implementations MAY use any value within the bounds of §III.5.1–.5.2.

III.5.4 — Per-scope TTL variation. Authentication endpoints MAY issue different TTLs for different scope sets. Higher-risk scopes (those granting persistence write access, repatriation, or other operations with cross-trust-boundary effects) SHOULD use shorter TTLs. Lower-risk scopes (health, list\_registries) MAY use longer TTLs up to the §III.5.2 maximum.

III.5.5 — TTL is in the token. Per OIDC convention, the TTL is expressed as the exp claim in the JWT. SAI determines expiration from the token itself, not from local-clock tracking against issuance time.

## III.6 Lazy Refresh

III.6.1 — Refresh on next use. SAI SHALL NOT proactively refresh tokens before their expiration. A token is requested anew when it is needed and either (a) no cached token for the required scope exists, or (b) the cached token has expired.

III.6.2 — No refresh-vs-acquire distinction. Token refresh requires re-authenticating to the target's authentication endpoint per §5.1.8. There is no cheaper "refresh" path; every fresh token requires a fresh authentication. The cost of refreshing ahead of expiration is therefore the same as the cost of acquiring at expiration; lazy refresh is preferred because it minimizes total round trips.

III.6.3 — Long-running processes. For long-running SAI invocations or processes that may pause for extended periods, tokens may expire between operations. SAI implementations SHOULD handle expired-token rejections from targets by re-authenticating and retrying once. Repeated failures after re-authentication indicate a real authorization problem and are reported as errors. The retry-on-401 behavior is implementation defensiveness; the spec does not normatively mandate it.

## III.7 Token Format

III.7.1 — JWT. SADAR scopes are carried in JWTs per OIDC convention. The scope claim in the JWT is a space-separated list of scope values per RFC 8693 / OIDC core. The token is signed by the target's authentication endpoint using key material declared in the target's manifest per §5.1.8.

III.7.2 — JWT verification. SAI verifies tokens it presents and tokens it receives using the target's registry-published JWKS. Verification follows the same path as for any other §5.1.8 JWT. There is no SADAR-specific verification step.

— — —

# Change Log

|  |  |  |
| --- | --- | --- |
| **Version** | **Date** | **Changes** |
| 1.1.0 | April 2026 | Back-port from C21 §10.1.4 applied. Adds add\_risk\_adjustment method to Helper API §I.4.6, admissible during Invocation and Outcome phases. Method appends Adjustments to the Telemetry Record's risk\_adjustments field (C20 v5.2) and propagates to OTel Baggage at urn:sadar:baggage:v1:risk\_adjustments. Phase descriptions in §I.3 and §I.4 updated to reflect six methods. Finalize logic in §I.6.1 updated to mention risk\_score\_at\_finalize computation. Method count in §I.4 intro changed from five to six. Backwards compatible — implementations conformant to v1.0 remain conformant to v1.1; v1.1 implementations gain the add\_risk\_adjustment surface. |
| 1.0.0 | April 2026 | Initial v1 release. Three-part document: Telemetry Record Helper API (Part I), Repatriation Trigger Specification (Part II), Authentication Scopes (Part III). Establishes urn:sadar:scope:v1:\* namespace. |

*© 2026 Cognita AI Inc. All rights reserved. SADAR™ is a trademark of Cognita AI, Inc. CogniWeave™ is a trademark of Cognita AI, Inc. All other trademarks are the property of their respective owners. Licensed under the Community Specification License 1.0.*