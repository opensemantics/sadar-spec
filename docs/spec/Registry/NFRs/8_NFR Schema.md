SADAR™ NFR Schema

*Specification — Version 1.2.0*

Status: Draft • Tracking ID: C19 • Combined back-ports from C31 §3 and C21 §11 applied

*April 2026*

*© 2026 Cognita AI Inc. All rights reserved. SADAR™ is a trademark of Cognita AI, Inc. CogniWeave™ is a trademark of Cognita AI, Inc. All other trademarks are the property of their respective owners.*

## Document Overview

This specification defines the SADAR Non-Functional Requirements (NFR) Schema: the manifest structure, NFR vocabulary across four categories (Financial, Operational, Governance, Protocol), three-tier matching strictness, business process and data field declarations, the bilateral match algorithm, and registry-side validation requirements. It is the canonical source for the SADAR manifest schema and matching semantics referenced from scope.md §5.1.2 and §5.1.3.

v1.2 incorporates two additive back-ports:

C31 §3 (April 2026): adds is\_default boolean field to the Role Declaration Object (§8.2.1) with at-most-one-per-manifest constraint (§8.2.4.4) and registry-side validation (§15.2.5). Enables servers supporting the asserted trust model to declare a default role applied when SCTs lack an explicit Role claim.

C21 §11 (April 2026): adds impact\_score Protocol NFR field (§8.4) at urn:sadar:nfr:v1:protocol:impact\_score. Optional decimal in [0.0, 1.0] declared by server entries expressing potential severity-of-misuse.

**Cross-references.** scope.md §5.1.2 (Manifest Structure umbrella). scope.md §5.1.3 (Bilateral Discovery, Compatibility Criteria, and Manifest Verification). C2 v2 (SCT Operations). C20 v5.2 (Telemetry Record schema, including the v5.2 risk score fields). C21 v1 (Risk Score Specification — origin of impact\_score in §11). C27 v3 (Trust Models — defines the four trust models referenced by supported\_trust\_models). C31 v1 (Asserted SCT Validation — origin of is\_default in §3). C33 v1.1 (searchAndInvoke).

— — —

# §1 Introduction

## §1.1 Purpose

§1.1.1 — This specification defines the normative manifest schema and NFR vocabulary used by SADAR-conformant implementations for capability advertisement, requirement declaration, and bilateral matching at discovery time.

## §1.2 Scope

§1.2.1 — In scope. Manifest structure (§2), top-level identity fields (§2.2), the symmetric requester/server section model (§2), three-tier matching strictness (§4.4), four NFR categories (§5–§8), business process declarations (§9), data field declarations (§10), compliance framework registry (§11), bilateral match algorithm (§12), unified namespace pattern (§13), versioning policy (§16), registry-side validation (§15).

§1.2.2 — Out of scope. The protocol for executing matches (handled by the registry implementation), resolver behavior (scope.md §5.1.13), the trust model definitions themselves (C27 v3), the SCT validation algorithm (C31 v1), telemetry and repatriation (C20 v5.2), risk score data model (C21 v1).

## §1.3 Normative Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 / RFC 8174.

## §1.4 Versioning Approach

§1.4.1 — Schema versioning. Every manifest declares a schema\_version IRI identifying the NFR schema version it conforms to. v1.2 manifests declare urn:sadar:nfr:v1.2.

§1.4.2 — MINOR vs MAJOR increments. New optional fields, new enum values, and new compliance framework IRIs added to the registry are MINOR-version increments (backwards compatible). Removal of fields, narrowing of value ranges, or breaking changes to existing semantics are MAJOR-version increments.

§1.4.3 — v1.2 is a MINOR increment. Both back-ports applied in v1.2 are backwards compatible with v1.0 manifests; existing v1.0 manifests remain valid under v1.2.

## §1.5 Relationship to Other SADAR Specifications

§1.5.1 — This NFR Schema spec defines the manifest data structure and matching algorithm. Other SADAR specifications consume the schema:

scope.md §5.1.2 references the manifest structure umbrella; this document is the source.

scope.md §5.1.3 references the bilateral match algorithm; §12 below is the source.

C27 v3 defines the four trust models; this document §8.1 establishes the supported\_trust\_models NFR that manifests use to declare them.

C31 v1 defines the asserted SCT validation algorithm; this document §8.2 establishes the supported\_roles NFR including the is\_default field added in v1.2.

C21 v1 defines the Risk Score Specification including impact\_score semantics; this document §8.4 establishes the impact\_score NFR field added in v1.2.

C20 v5.2 defines the Telemetry Record; this document §8.3 establishes the four repatriation Protocol NFR fields used in repatriation bilateral matching.

— — —

# §2 Manifest Structure

## §2.1 Symmetric Role Sections

§2.1.1 — A SADAR manifest may contain two top-level role sections: requester and server. Each section uses the same NFR vocabulary; interpretation is determined by which section a field appears in.

|  |  |  |
| --- | --- | --- |
| **Section** | **Role** | **Interpretation** |
| requester | Outbound role | What the entity wants when searching for and invoking other capabilities. Matched against server advertisements at discovery. |
| server | Inbound role | What the entity advertises as its own capability. What it requires of incoming requesters (in requirements\_of\_requester sub-section). Matched against requester search criteria at discovery. |

§2.1.2 — At least one section required. A manifest SHALL declare at least one of requester or server. Many entities declare both, reflecting agents that both call and are called.

§2.1.3 — Within server, an additional requirements\_of\_requester sub-section uses the same vocabulary to express what the server requires of incoming requesters.

## §2.2 Top-Level Fields

|  |  |  |  |
| --- | --- | --- | --- |
| **Field** | **Type** | **Required** | **Description** |
| manifest\_urn | string (URN) | YES | Globally unique identifier for this manifest. |
| endpoint\_urn | string (URN) | YES | Identifier of the deployment endpoint hosting this manifest. Multiple manifests may share an endpoint\_urn. |
| manifest\_version | string (semver) | YES | Version of this manifest. Bumped on any content change. |
| schema\_version | string (URN) | YES | IRI of the NFR schema version this manifest conforms to. For v1.2 conformance: urn:sadar:nfr:v1.2. |
| lifecycle\_state | enum | YES | One of: draft, active, deprecated, superseded, retired. |
| signature | JWS | YES | JWS signature over the canonicalized manifest content. |
| requester | object | CONDITIONAL | Requester-role declarations. Required if server is absent. |
| server | object | CONDITIONAL | Server-role declarations. Required if requester is absent. |

## §2.3 One Capability Per Manifest

§2.3.1 — A single manifest declares one capability. Multi-capability endpoints publish multiple manifests sharing a common endpoint\_urn. Each manifest versions independently — a change to one capability does not trigger re-versioning of others at the same endpoint.

## §2.4 Lifecycle States

|  |  |
| --- | --- |
| **State** | **Semantics** |
| draft | Manifest is being developed; not yet active. |
| active | Manifest is live and discoverable. |
| deprecated | Manifest is live but a successor is preferred. Discoverable with deprecation flag visible to requesters. |
| superseded | Manifest has been replaced by a newer version. Not discoverable in normal queries. |
| retired | Manifest is no longer supported. Not discoverable. |

— — —

# §3 Role Section Structure

§3.1 — Each role section (requester or server) carries declarations across the following elements. The four NFR categories sit alongside first-class manifest elements (business process and data fields).

## §3.2 Elements

|  |  |  |
| --- | --- | --- |
| **Element** | **Type** | **Description** |
| business\_process | object | Business process declaration per §9. |
| target\_business\_process | object | In requester section only: business process the requester wants the server to perform. |
| data\_fields | array | Data field declarations per §10. |
| financial | object | Financial NFRs per §5. |
| operational | object | Operational NFRs per §6. |
| governance | object | Governance NFRs per §7. |
| protocol | object | Protocol NFRs per §8. |
| requirements\_of\_requester | object | In server section only: requirements the server places on incoming requesters. Uses the same vocabulary. |

— — —

# §4 NFR Common Structure

## §4.1 NFR Field Identity

§4.1.1 — Every NFR field is identified by an IRI in the urn:sadar:nfr:v{version}:{category}:{attribute} pattern. The pattern enables versioned vocabulary evolution and unambiguous cross-spec reference.

## §4.2 NFR Field Object Schema

§4.2.1 — A typical NFR field is represented as an object carrying its value and a strictness flag. The schema:

{
 "value": <field-specific>,
 "strictness": "OPTIONAL" | "MANDATORY" | "MANDATORY\_STRICT",
 "comment": "<optional human-readable annotation>"
}

§4.2.2 — Some fields permit short-form (just the value) when strictness defaults are appropriate. The full-object form is always permitted; short-form acceptance is documented per-field.

## §4.3 Three-Tier Strictness

|  |  |
| --- | --- |
| **Flag** | **Match Behavior** |
| OPTIONAL | Soft preference. Mismatches affect ranking only; never cause exclusion from candidates. |
| MANDATORY | Hard requirement, but partial matches are surfaced. The resolver receives them with classification. |
| MANDATORY\_STRICT | Hard exclusion. Candidates failing this requirement are not returned in the candidate set at all. |

§4.3.1 — Default strictness. When strictness is omitted, the default is OPTIONAL.

## §4.4 Match Outcomes

§4.4.1 — Per-field match outcomes feed into per-direction and overall match outcomes per §12.

|  |  |
| --- | --- |
| **Outcome** | **Meaning** |
| FULL\_MATCH | All MANDATORY and MANDATORY\_STRICT fields satisfied; OPTIONAL preferences met or unmet without causing exclusion. |
| PARTIAL\_MATCH | At least one MANDATORY field has a partial match; no MANDATORY\_STRICT field is unsatisfied. Surfaced to resolver. |
| NO\_MATCH | At least one MANDATORY\_STRICT field is unsatisfied. Candidate excluded from results entirely. |

— — —

# §5 Financial NFRs

§5.1 — Namespace: urn:sadar:nfr:v1:financial:\*. Financial NFRs cover cost, pricing, payment methods, and billing.

|  |  |  |
| --- | --- | --- |
| **Field** | **Type** | **Description** |
| urn:sadar:nfr:v1:financial:cost\_per\_invocation | decimal | Cost per invocation in declared currency. |
| urn:sadar:nfr:v1:financial:cost\_currency | string (ISO 4217) | Currency code, e.g., USD, EUR. |
| urn:sadar:nfr:v1:financial:pricing\_model | enum | fixed, tiered, metered, subscription, freemium. |
| urn:sadar:nfr:v1:financial:payment\_methods | array of strings | Accepted payment methods, e.g., [x402, credit\_card, ach]. |
| urn:sadar:nfr:v1:financial:billing\_period | string | Billing cadence, e.g., monthly, annual, per\_invocation. |
| urn:sadar:nfr:v1:financial:merchant\_id | string | Merchant identifier for the chosen payment method. |
| urn:sadar:nfr:v1:financial:payment\_endpoint | string (URL/URN) | Payment endpoint URL. |

— — —

# §6 Operational NFRs

§6.1 — Namespace: urn:sadar:nfr:v1:operational:\*. Operational NFRs cover performance, throughput, availability, and technical limits.

|  |  |  |
| --- | --- | --- |
| **Field** | **Type** | **Description** |
| urn:sadar:nfr:v1:operational:availability | decimal in [0.0, 1.0] | Declared SLA availability (e.g., 0.999 = 99.9%). |
| urn:sadar:nfr:v1:operational:max\_latency\_ms | integer | Maximum response latency in milliseconds. |
| urn:sadar:nfr:v1:operational:max\_throughput\_qps | integer | Maximum sustained throughput in queries per second. |
| urn:sadar:nfr:v1:operational:rate\_limit\_per\_minute | integer | Per-requester rate limit per minute. |
| urn:sadar:nfr:v1:operational:max\_payload\_bytes | integer | Maximum input payload size in bytes. |
| urn:sadar:nfr:v1:operational:max\_response\_bytes | integer | Maximum response payload size in bytes. |
| urn:sadar:nfr:v1:operational:concurrency\_limit | integer | Maximum concurrent invocations from a single requester. |
| urn:sadar:nfr:v1:operational:retry\_policy | enum | idempotent, at\_least\_once, at\_most\_once. |
| urn:sadar:nfr:v1:operational:timeout\_seconds | integer | Recommended invocation timeout in seconds. |

— — —

# §7 Governance NFRs

§7.1 — Namespace: urn:sadar:nfr:v1:governance:\*. Governance NFRs cover compliance, data sovereignty, privacy, encryption, and licensing.

## §7.2 Fields

|  |  |  |
| --- | --- | --- |
| **Field** | **Type** | **Description** |
| urn:sadar:nfr:v1:governance:compliance | array of compliance IRIs | Compliance frameworks the server implements or the requester requires. IRIs from §11 registry. (See §7.2.1.) |
| urn:sadar:nfr:v1:governance:compliance\_evidence | array of objects | Compliance documentation references with optional OIDC-protected access for sensitive evidence. |
| urn:sadar:nfr:v1:governance:data\_sovereignty\_jurisdiction | array of jurisdiction codes | Permitted data jurisdictions, e.g., [US, EU]. |
| urn:sadar:nfr:v1:governance:data\_residency | string | Where data resides during processing. |
| urn:sadar:nfr:v1:governance:encryption\_in\_transit | enum | minimum\_tls\_1\_2, minimum\_tls\_1\_3, mtls\_required. |
| urn:sadar:nfr:v1:governance:encryption\_at\_rest | enum | aes\_256, aes\_128, none, implementer\_choice. |
| urn:sadar:nfr:v1:governance:license\_type | string | License identifier (commercial, open source, proprietary, etc.). |
| urn:sadar:nfr:v1:governance:license\_url | string (URL) | License document location. |
| urn:sadar:nfr:v1:governance:human\_review\_required | boolean | Whether first-use requires human review per scope.md §5.1.9. |
| urn:sadar:nfr:v1:governance:audit\_logging | enum | full, summary, none. |
| urn:sadar:nfr:v1:governance:retention\_days | integer | Data retention period in days. |

### §7.2.1 Compliance Field Detail

compliance is an array of compliance framework IRIs from the registry maintained in §11. Each entry is matched per the bilateral match algorithm. The matching is exact-IRI; equivalences between frameworks (e.g., SOC 2 Type II vs SOC 2 Type I) are not implicit and require explicit equivalence declaration per §11.4.

— — —

# §8 Protocol NFRs

§8.1 — Namespace: urn:sadar:nfr:v1:protocol:\*. Protocol NFRs cover SADAR-specific behaviors. v1.0 defines three field families (supported\_trust\_models, supported\_roles, Telemetry Repatriation Fields). v1.2 adds a fourth: impact\_score.

## §8.1 Supported Trust Models

§8.1.1 — supported\_trust\_models declares which trust models (per C27 v3) the entity supports for this capability. Bilateral matching at discovery selects the trust model used per C27 v3 §4.6.X.7 negotiation algorithm.

urn:sadar:nfr:v1:protocol:supported\_trust\_models

Type: array of trust model identifiers.

Values: direct\_auth, asserted, impersonation, deputy.

Cardinality: at least one. A manifest declaring no trust models is not conformant.

## §8.2 Supported Roles

§8.2.1 — supported\_roles declares the roles a server recognizes for asserted-trust-model invocations. Required when the server includes "asserted" in supported\_trust\_models. The complete validation algorithm is in C31 v1.

urn:sadar:nfr:v1:protocol:supported\_roles

Type: array of Role Declaration Objects.

§8.2.1 Role Declaration Object schema:

|  |  |  |  |
| --- | --- | --- | --- |
| **Field** | **Type** | **Required** | **Description** |
| role\_id | string | YES | Identifier matching ^[a-z][a-z0-9\_]\*$. Unique within this manifest. |
| description | string | YES | Human-readable description of the role. |
| permissions | array of objects | YES | Permission objects with operation, resource, and scope fields. |
| process\_authority | string (URN) or array | NO | Constraint on which business processes this role may operate within. Default: any. |
| audit\_template | object | NO | Template fields to include in audit records of invocations under this role. |
| is\_default | boolean | NO | NEW IN v1.2 (back-port from C31 §3). When true, this role applies to invocations that do not include an explicit Role claim in the SCT. At most one role per manifest may declare is\_default = true (see §8.2.4.4 constraint and §15.2.5 registry-side validation). Default: false. |

### §8.2.2 Permission Object Schema

A permission object specifies what operations and resources a role may access:

|  |  |  |
| --- | --- | --- |
| **Field** | **Type** | **Description** |
| operation | string (IRI) | Operation identifier from a published standard (X12, HL7, ISO, etc.) or an organization-defined operation. |
| resource | string (IRI) | Resource identifier — typically a data field or business object IRI. |
| scope | string | Optional access scope qualifier (read, write, read\_write, admin, etc.). |

### §8.2.3 Process Authority

process\_authority constrains the business process(es) under which a role may operate. A role with process\_authority of "urn:apqc:pcf:claim\_processing" applies only to invocations whose business process context is claim\_processing. A role without process\_authority applies to any business process.

### §8.2.4 Constraints

§8.2.4.1 — Role uniqueness. role\_id values within a single manifest SHALL be unique.

§8.2.4.2 — Asserted prerequisite. If the manifest declares "asserted" in supported\_trust\_models, the manifest SHALL declare a non-empty supported\_roles array.

§8.2.4.3 — Process authority well-formedness. process\_authority values, when present, SHALL be well-formed IRIs.

§8.2.4.4 — At-most-one default role. (NEW IN v1.2 — back-port from C31 §3.) Within supported\_roles, at most one Role Declaration Object SHALL declare is\_default = true. Manifests declaring two or more default roles are malformed and SHALL be rejected at registry-side validation per §15.2.5.

## §8.3 Telemetry Repatriation Fields

§8.3.1 — Four Protocol NFR fields establish bilateral declaration of telemetry repatriation. The bilateral match algorithm for repatriation is defined in C20 v5.2 §5.1.14.X.10.4.

|  |  |  |  |
| --- | --- | --- | --- |
| **Field** | **Type** | **Side** | **Description** |
| urn:sadar:nfr:v1:protocol:repatriation\_participation | boolean | server | Whether the server advertises repatriation capability for this entry. Default: false. |
| urn:sadar:nfr:v1:protocol:repatriation\_endpoint | string (URL/URN) | requester | Where repatriated payloads are delivered. |
| urn:sadar:nfr:v1:protocol:repatriation\_redacted\_fields | array of strings | server | Field names the server will redact in repatriated payloads. SHALL NOT contain non-redactable structural fields per C20 v5.2 §5.1.14.X.10.8.3. |
| urn:sadar:nfr:v1:protocol:repatriation\_required\_unredacted\_fields | array of strings | requester | Field names the requester requires returned unredacted. Bilateral match succeeds if intersection with repatriation\_redacted\_fields is empty. |

## §8.4 Impact Score (NEW IN v1.2)

§8.4.1 — Per back-port from C21 v1 §11. impact\_score permits server entries to declare the potential severity of negative consequences if the entity is misused or used outside its intended context. The score is publisher-asserted and static — distinct from the in-flight Risk Score (C21) which is cumulative and dynamic.

urn:sadar:nfr:v1:protocol:impact\_score

|  |  |
| --- | --- |
| **Aspect** | **Value** |
| Type | decimal |
| Range | [0.0, 1.0] |
| Cardinality | optional, single value |
| Declarer | server (entries declare their own potential impact) |
| Default | absent (no declared impact) |
| Strictness | OPTIONAL by default; MAY be set MANDATORY or MANDATORY\_STRICT by requesters. |

§8.4.2 — Semantic. 0.0 represents no possibility of negative impact; 1.0 represents maximum severity. Publishers consider business impact, regulatory impact, reputational impact, and operational impact in arriving at a value. NIST SP 800-30 Revision 1, Appendix H is referenced informatively as best-practice framing.

§8.4.3 — Optional declaration. Publishers SHOULD declare impact\_score where they can substantiate a claim. Absent declaration is interpreted as "no declared impact" rather than "zero impact."

§8.4.4 — Distinction from Risk Score. impact\_score is a static publisher-declared property of the entry. The Risk Score (C21) is a stateful, cumulative property of an in-flight process flow. Impact = "what could go wrong if misused"; Risk Score = "what the situation actually warrants given accumulated observation." Implementations may use impact\_score as one input to Risk Score accumulation algorithms (per C21 §11.3) but the relationship is implementer concern.

§8.4.5 — Calibration. Two implementations of the "same" business operation by different publishers may declare different impact\_scores. Consumers SHOULD treat impact\_score as a publisher-asserted indicator rather than a normalized cross-publisher quantity. Calibration over time is organizational concern.

— — —

# §9 Business Process Declarations

§9.1 — Business process is a first-class manifest element (not an NFR category). Declarations ground the entry in industry-standard process frameworks via IRI references.

## §9.2 Declaration Schema

|  |  |  |  |
| --- | --- | --- | --- |
| **Field** | **Type** | **Required** | **Description** |
| process\_id | string (IRI) | YES | IRI from a published process framework (APQC PCF, HL7 workflows, X12 transactions, etc.) or an organization-defined process. |
| process\_step | string (IRI) | NO | Specific step within the process, where applicable. |
| exclusions | array of objects | NO | Sub-steps not performed. Each exclusion carries its own strictness flag. |
| prerequisites | array of objects | NO | Process steps that must have completed before this entry is invoked. Each prerequisite carries its own strictness flag. |
| strictness | enum | NO | OPTIONAL / MANDATORY / MANDATORY\_STRICT applied to the entire declaration. Default: MANDATORY. |

## §9.3 Two-Process Pattern in Requester Section

§9.3.1 — Requester sections may declare two distinct business processes:

business\_process — the requester's own context. Used by enforcement layers to evaluate context-aware authorization.

target\_business\_process — the business process the requester wants the server to perform. Matched against the server's business\_process declaration at discovery.

§9.3.2 — Server sections declare only business\_process — the process step the server performs. The server may declare requirements\_of\_requester.business\_process to require the requester's upstream context.

— — —

# §10 Data Field Declarations

§10.1 — Data fields are first-class manifest elements (not an NFR category). Each declaration identifies a data field by IRI and declares strictness for matching purposes.

## §10.2 Declaration Schema

|  |  |  |  |
| --- | --- | --- | --- |
| **Field** | **Type** | **Required** | **Description** |
| field\_iri | string (IRI) | YES | IRI identifying the data field within its standard (e.g., urn:ansi.org:x12:850:SEG:01:DATE). |
| direction | enum | YES | input, output, or both. |
| strictness | enum | NO | OPTIONAL / MANDATORY / MANDATORY\_STRICT. Default: MANDATORY for output, OPTIONAL for input. |
| cardinality | enum | NO | single, multiple. Default: single. |
| comment | string | NO | Human-readable annotation. |

## §10.3 Standard Composition

§10.3.1 — Manifests may reference data fields from multiple standards in the same data\_fields array. There is no schema-level restriction on mixing standards.

§10.3.2 — Cross-standard interchangeability — equivalence mappings between fields in different standards — is governed by the forthcoming SADAR Data Classification Annex (DCA). Match-time logic at the registry is exact-IRI; transformation and interchangeability are selector concerns.

— — —

# §11 Compliance Framework Identifiers Registry

§11.1 — Namespace: urn:sadar:compliance:v1:\*. This section defines the SADAR-maintained registry of compliance framework identifiers used in the urn:sadar:nfr:v1:governance:compliance array.

## §11.2 Initial Registry

|  |  |
| --- | --- |
| **Identifier** | **Framework** |
| urn:sadar:compliance:v1:soc2\_type1 | SOC 2 Type I |
| urn:sadar:compliance:v1:soc2\_type2 | SOC 2 Type II |
| urn:sadar:compliance:v1:iso27001 | ISO 27001 |
| urn:sadar:compliance:v1:iso27017 | ISO 27017 (Cloud) |
| urn:sadar:compliance:v1:iso27018 | ISO 27018 (Cloud PII) |
| urn:sadar:compliance:v1:hipaa | HIPAA Privacy and Security Rules |
| urn:sadar:compliance:v1:hitrust | HITRUST CSF |
| urn:sadar:compliance:v1:gdpr | GDPR (EU General Data Protection Regulation) |
| urn:sadar:compliance:v1:ccpa | CCPA / CPRA (California) |
| urn:sadar:compliance:v1:fedramp\_low | FedRAMP Low |
| urn:sadar:compliance:v1:fedramp\_moderate | FedRAMP Moderate |
| urn:sadar:compliance:v1:fedramp\_high | FedRAMP High |
| urn:sadar:compliance:v1:pci\_dss | PCI DSS |
| urn:sadar:compliance:v1:nist\_800\_53 | NIST SP 800-53 |
| urn:sadar:compliance:v1:nist\_800\_171 | NIST SP 800-171 |
| urn:sadar:compliance:v1:nist\_csf | NIST Cybersecurity Framework |
| urn:sadar:compliance:v1:nist\_ai\_rmf | NIST AI Risk Management Framework |
| urn:sadar:compliance:v1:cis\_controls | CIS Critical Security Controls |
| urn:sadar:compliance:v1:eu\_ai\_act | EU AI Act |

## §11.3 Custom Compliance IRIs

§11.3.1 — Organizations MAY declare custom compliance IRIs in their own namespaces. Custom IRIs SHALL NOT use the urn:sadar: prefix.

## §11.4 Equivalences

§11.4.1 — The registry does NOT define implicit equivalences between frameworks. SOC 2 Type II is not implicitly satisfied by SOC 2 Type I; ISO 27001 is not implicitly satisfied by ISO 27017. Bilateral matching is exact-IRI.

— — —

# §12 Bilateral Match Algorithm

§12.1 — The bilateral match algorithm produces, for each candidate, a match outcome (FULL\_MATCH, PARTIAL\_MATCH, or NO\_MATCH) representing the combined evaluation of two directions.

## §12.2 Direction 1 — Requester Wants vs Server Advertises

§12.2.1 — For each field declared in the requester's requester section (or implied by its query), evaluate against the corresponding field in the candidate server's server section. Evaluation per the field's strictness flag.

## §12.3 Direction 2 — Server Requires vs Requester Self-Declares

§12.3.1 — For each field declared in the candidate server's requirements\_of\_requester sub-section, evaluate against the requester's self-declarations (in its own requester or server section, as appropriate).

## §12.4 Combination

§12.4.1 — The combined outcome is the strictest of the two directions:

If either direction is NO\_MATCH → combined is NO\_MATCH (candidate excluded from results).

If either direction is PARTIAL\_MATCH → combined is PARTIAL\_MATCH (candidate surfaced to resolver with classification).

If both are FULL\_MATCH → combined is FULL\_MATCH.

## §12.5 Resolver Consumption

§12.5.1 — Candidates returned to the resolver carry their combined match outcome. The resolver MAY prefer FULL\_MATCH candidates over PARTIAL\_MATCH; the resolver may select a PARTIAL\_MATCH if its selection logic deems the partial match acceptable. The resolver SHALL NOT escalate a NO\_MATCH candidate (which by definition is not in the candidate set).

— — —

# §13 Namespace Patterns

§13.1 — All SADAR-defined IRIs follow the unified pattern urn:sadar:{category}:v{version}:.... Six sub-namespaces:

|  |  |
| --- | --- |
| **Namespace Pattern** | **Purpose** |
| urn:sadar:nfr:v{version}:{category}:{attribute} | NFR field identifiers |
| urn:sadar:compliance:v{version}:{framework} | Compliance framework identifiers |
| urn:sadar:error:v{version}:{category}:{code} | Error code identifiers |
| urn:sadar:scope:v{version}:{category}:{operation} | OIDC scope identifiers |
| urn:sadar:risk\_reason:v{version}:{category} | Risk score reason identifiers |
| urn:sadar:baggage:v{version}:{key} | OTel baggage key identifiers |

§13.2 — Custom or implementation-specific identifiers SHALL NOT use the urn:sadar: prefix. Ecosystem participants extend the SADAR vocabulary in their own organization-controlled namespaces.

— — —

# §14 Manifest Schema Reference

§14.1 — A complete v1.2 manifest skeleton:

{
 "manifest\_urn": "urn:registry:abc:agent:claim\_processor",
 "endpoint\_urn": "urn:registry:abc:endpoint:claims\_v3",
 "manifest\_version": "2.0.0",
 "schema\_version": "urn:sadar:nfr:v1.2",
 "lifecycle\_state": "active",

 "server": {
 "business\_process": {
 "process\_id": "urn:apqc:pcf:claim\_adjudication",
 "strictness": "MANDATORY"
 },
 "data\_fields": [
 { "field\_iri": "urn:hl7:fhir:Claim", "direction": "input",
 "strictness": "MANDATORY\_STRICT" },
 { "field\_iri": "urn:hl7:fhir:ClaimResponse",
 "direction": "output", "strictness": "MANDATORY\_STRICT" }
 ],
 "financial": {
 "urn:sadar:nfr:v1:financial:cost\_per\_invocation": 0.05,
 "urn:sadar:nfr:v1:financial:cost\_currency": "USD"
 },
 "operational": {
 "urn:sadar:nfr:v1:operational:availability": 0.999,
 "urn:sadar:nfr:v1:operational:max\_latency\_ms": 500
 },
 "governance": {
 "urn:sadar:nfr:v1:governance:compliance": [
 "urn:sadar:compliance:v1:hipaa",
 "urn:sadar:compliance:v1:soc2\_type2"
 ],
 "urn:sadar:nfr:v1:governance:encryption\_in\_transit": "minimum\_tls\_1\_3"
 },
 "protocol": {
 "urn:sadar:nfr:v1:protocol:supported\_trust\_models":
 ["asserted", "deputy"],
 "urn:sadar:nfr:v1:protocol:supported\_roles": [
 {
 "role\_id": "claim\_adjudicator",
 "description": "Performs claim adjudication decisions",
 "permissions": [
 { "operation": "urn:hl7:fhir:Claim:adjudicate",
 "resource": "urn:hl7:fhir:Claim",
 "scope": "read\_write" }
 ],
 "is\_default": true
 }
 ],
 "urn:sadar:nfr:v1:protocol:repatriation\_participation": true,
 "urn:sadar:nfr:v1:protocol:repatriation\_redacted\_fields":
 ["audit\_metadata", "input\_summary"],
 "urn:sadar:nfr:v1:protocol:impact\_score": 0.7
 },

 "requirements\_of\_requester": {
 "governance": {
 "urn:sadar:nfr:v1:governance:compliance": [
 "urn:sadar:compliance:v1:hipaa"
 ]
 }
 }
 },

 "signature": "<JWS over canonicalized content>"
}

— — —

# §15 Registry-Side Validation

§15.1 — A SADAR-conformant registry SHALL validate manifests on ingestion (registration and update). Manifests failing validation SHALL be rejected with a structured error in the urn:sadar:error:v1:\* namespace.

## §15.2 Validation Requirements

§15.2.1 — Schema validation. The manifest SHALL conform to the schema defined in this specification, including all required fields and well-formed values.

§15.2.2 — Repatriation field validation. Server manifests declaring repatriation\_participation = true SHALL declare repatriation\_redacted\_fields. The redacted fields list SHALL NOT contain any non-redactable structural field per C20 v5.2 §5.1.14.X.10.8.3 (trace\_id, span\_id, parent\_span\_id, start\_time, end\_time, duration, status, name, telemetry.origin.environment, telemetry.origin.agent, telemetry.repatriated, telemetry.disclosure.policy\_id).

§15.2.3 — Asserted-trust-model role validation. Server manifests declaring "asserted" in supported\_trust\_models SHALL declare a non-empty supported\_roles array. Manifests failing this constraint are rejected.

§15.2.4 — Role uniqueness. Within supported\_roles, role\_id values SHALL be unique. Manifests with duplicate role\_id are rejected.

§15.2.5 — At-most-one default role. (NEW IN v1.2 — back-port from C31 §3.) Within supported\_roles, at most one Role Declaration Object SHALL declare is\_default = true. Manifests declaring two or more default roles are rejected with error urn:sadar:error:v1:nfr\_schema:multiple\_default\_roles.

§15.2.6 — Cryptographic integrity. The manifest signature SHALL verify against the publisher's registry-published signing key. Manifests with invalid signatures are rejected.

§15.2.7 — Lifecycle state validity. lifecycle\_state transitions SHALL follow the lifecycle model in §2.4. Invalid transitions (e.g., active → draft) are rejected.

§15.2.8 — IRI well-formedness. All IRI fields SHALL be well-formed per RFC 3987. Manifests with malformed IRIs are rejected.

§15.2.9 — impact\_score range. (NEW IN v1.2 — back-port from C21 §11.) When present, impact\_score SHALL be a decimal in [0.0, 1.0]. Out-of-range values are rejected with error urn:sadar:error:v1:nfr\_schema:invalid\_impact\_score.

## §15.3 Error Codes

|  |  |
| --- | --- |
| **Error Code** | **Meaning** |
| urn:sadar:error:v1:nfr\_schema:invalid\_schema | Manifest fails schema validation. §15.2.1. |
| urn:sadar:error:v1:nfr\_schema:invalid\_redacted\_fields | repatriation\_redacted\_fields contains non-redactable structural field. §15.2.2. |
| urn:sadar:error:v1:nfr\_schema:asserted\_without\_roles | "asserted" declared without supported\_roles. §15.2.3. |
| urn:sadar:error:v1:nfr\_schema:duplicate\_role\_id | Role uniqueness violated. §15.2.4. |
| urn:sadar:error:v1:nfr\_schema:multiple\_default\_roles | NEW IN v1.2. More than one role declares is\_default = true. §15.2.5. |
| urn:sadar:error:v1:nfr\_schema:invalid\_signature | Manifest signature does not verify. §15.2.6. |
| urn:sadar:error:v1:nfr\_schema:invalid\_lifecycle\_transition | lifecycle\_state transition is not permitted. §15.2.7. |
| urn:sadar:error:v1:nfr\_schema:malformed\_iri | IRI value is not well-formed. §15.2.8. |
| urn:sadar:error:v1:nfr\_schema:invalid\_impact\_score | NEW IN v1.2. impact\_score is outside [0.0, 1.0]. §15.2.9. |

— — —

# §16 Versioning Policy

§16.1 — MINOR increments. Adding optional fields, adding enum values to existing fields, adding entries to the compliance framework registry, and adding new compliance framework registry entries are MINOR-version increments. Manifests at the prior MINOR version remain valid under the new MINOR version.

§16.2 — MAJOR increments. Removing fields, narrowing value ranges in ways that invalidate existing values, removing enum values, or breaking changes to existing semantics are MAJOR-version increments. Existing manifests at the prior MAJOR version SHALL be migrated.

§16.3 — Schema version field. Manifests declare schema\_version per §2.2. The value reflects the schema version the manifest was authored against. Registries validating manifests SHOULD verify schema\_version matches the validation logic applied; manifests at older MINOR versions remain valid.

— — —

# §17 References

## §17.1 Normative References

|  |  |
| --- | --- |
| **Reference** | **Title / Relationship** |
| RFC 2119 | Key words for use in RFCs |
| RFC 8174 | Ambiguity of uppercase vs lowercase in RFC 2119 |
| RFC 3987 | Internationalized Resource Identifiers |
| RFC 7515 | JSON Web Signature (JWS) |
| RFC 7517 | JSON Web Key (JWK) |
| SADAR scope.md | §5.1.2 (Manifest Structure umbrella), §5.1.3 (Bilateral Discovery), §5.1.8 (Authentication Baseline). |
| C2 v2 (SCT Operations) | SCT structure used by trust models referenced in §8.1. |
| C20 v5.2 (Telemetry Record and Repatriation) | Schema for repatriation NFRs in §8.3 and the non-redactable structural field set in §15.2.2. |
| C21 v1 (Risk Score Specification) | Source of impact\_score in §8.4 and the v1.2 back-port. |
| C27 v3 (Trust Models) | The four trust models referenced in §8.1. |
| C31 v1 (Asserted SCT Validation) | Source of is\_default in §8.2.1 and the v1.2 back-port. |
| C33 v1.1 (searchAndInvoke Telemetry and Authentication) | Helper API surface and authentication scopes. |

## §17.2 Informative References

|  |  |
| --- | --- |
| **Reference** | **Note** |
| NIST SP 800-30 Revision 1, Appendix H | Best-practice framing for impact\_score calibration. Not adopted normatively. |
| OpenTelemetry specification | OTel spans, baggage, trace context. |
| OIDC Core 1.0 | OIDC scope claim mechanism. |

— — —

# Appendix A — Complete Field Index

A consolidated index of all NFR fields defined in v1.2.

|  |  |  |
| --- | --- | --- |
| **Category** | **Fields / Notes** | **Section** |
| Financial | 7 fields | §5 |
| Operational | 9 fields | §6 |
| Governance | 11 fields | §7 |
| Protocol — Trust Models | 1 field (supported\_trust\_models) | §8.1 |
| Protocol — Roles | 1 field (supported\_roles, with is\_default sub-field NEW IN v1.2) | §8.2 |
| Protocol — Repatriation | 4 fields | §8.3 |
| Protocol — Impact (NEW IN v1.2) | 1 field (impact\_score) | §8.4 |
| Total Protocol fields | 7 | §8 total |
| First-Class Manifest Elements | 2 (business\_process, data\_fields) | §9, §10 |

— — —

# Appendix B — Change Log

|  |  |  |
| --- | --- | --- |
| **Version** | **Date** | **Changes** |
| 1.2.0 | April 2026 | Combined back-port transition from v1.0. Two backwards-compatible additions: (1) From C31 v1 §3: is\_default boolean field added to Role Declaration Object (§8.2.1). At-most-one-per-manifest constraint added at §8.2.4.4 and registry-side validation at §15.2.5 with error code urn:sadar:error:v1:nfr\_schema:multiple\_default\_roles. Enables servers supporting the asserted trust model to declare a default role applied when SCTs lack an explicit Role claim. (2) From C21 v1 §11: impact\_score Protocol NFR field added as new §8.4 at urn:sadar:nfr:v1:protocol:impact\_score. Optional decimal in [0.0, 1.0] declared by server entries expressing potential severity-of-misuse. Distinct from the in-flight Risk Score (C21). Registry-side validation at §15.2.9 with error code urn:sadar:error:v1:nfr\_schema:invalid\_impact\_score. No intermediate v1.1 issued. Both back-ports are backwards compatible — manifests conformant to v1.0 remain conformant to v1.2. |
| 1.0.0 | April 2026 | Initial v1 release. Establishes manifest structure with symmetric requester/server sections, four NFR categories (Financial, Operational, Governance, Protocol), three-tier matching strictness (OPTIONAL/MANDATORY/MANDATORY\_STRICT), business process and data field declarations as first-class manifest elements, bilateral match algorithm with strictest-of-two combination, unified urn:sadar:\* namespace pattern, and registry-side validation requirements. |

*© 2026 Cognita AI Inc. All rights reserved. SADAR™ is a trademark of Cognita AI, Inc. CogniWeave™ is a trademark of Cognita AI, Inc. All other trademarks are the property of their respective owners. Licensed under the Community Specification License 1.0.*