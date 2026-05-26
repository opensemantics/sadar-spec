**SADAR**

Semantic Agent Discovery and Attribution Registry

Discovery and Invocation

|                   |                                     |
| ----------------- | ----------------------------------- |
| **Version**       | 0.10 DRAFT                          |
| **Date**          | May 2026                            |
| **Status**        | Draft for Public Comment            |
| **Maintained by** | OpenSemantics.org                   |
| **License**       | Community Specification License 1.0 |

> **Revision note (non-normative).** This document supersedes the prior
> 0.9 Discovery draft, which described a probabilistic / natural-language
> discovery model, a simplified manifest, an `X-SADAR-Originator` header
> identity model, and a CC BY 4.0 license. Those elements are obsolete.
> The current architecture is deterministic bilateral matching, the
> requester/server manifest model with four NFR categories, Signed
> Context Token (SCT) authorization-context propagation, and the
> Community Specification License 1.0. Where authoritative detail lives
> in a companion document, this document references it rather than
> restating it.

---

# 1. Introduction

SADAR — the Semantic Agent Discovery and Attribution Registry — is an
open specification that defines how autonomous AI agents discover
callable agents, tools, resources, and processes, and how they invoke
them with verifiable identity and attribution propagated end-to-end.
SADAR provides a uniform protocol for registering capabilities with
machine-readable, standards-grounded metadata, matching that metadata
deterministically and bilaterally, and executing invocations that carry
the originator's scope of authority across every trust boundary.

SADAR is intentionally narrow in scope. It specifies interfaces,
manifest and credential structures, transport requirements, and
behavioral contracts that ensure interoperability across
implementations. It does not specify how registries store or index
manifests, how authorization decisions are enforced, how a resolver
ranks candidates, or how invocations are physically executed. Those are
implementation concerns deliberately left to the implementer.

| | |
| --- | --- |
| **SCOPE NOTE** | This document is a protocol-level reading of discovery and invocation within the SADAR specification. It normatively specifies what conformant implementations MUST, SHOULD, and MAY do, and references the companion documents listed in §1.4 for authoritative detail. The key words MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT, RECOMMENDED, MAY, and OPTIONAL are used as defined in RFC 2119 / RFC 8174 when, and only when, they appear in all capitals. |

## 1.1 Design Principles

- **Semantic grounding** — capabilities and data are described by
  reference to published industry standards, not by free-text prose.
- **Determinism** — given identical inputs and normative fields,
  conformant registries produce identical candidate sets, regardless of
  operator.
- **Bilateral agreement** — discovery returns a candidate only when both
  the requester's requirements and the provider's requirements are
  satisfied.
- **Attribution completeness** — every invocation carries verifiable
  agent identity, originator authorization context, and transactional
  context.
- **Registry isolation** — the registry is a discovery-time component
  only; it is never in the runtime call path and never holds sensitive
  operational data.
- **Implementation freedom** — SADAR specifies contracts, not
  implementations; storage, indexing, ranking, enforcement, and
  execution are implementer concerns.
- **Neutrality** — discovery and invocation are independent of cloud,
  model provider, agentic framework, and hosting model.

## 1.2 Relationship to Other Standards

SADAR builds on and references existing standards rather than replacing
them:

- **MCP / A2A** — SADAR provides semantic discovery and attribution
  above MCP tool invocation and A2A agent interaction; it defines the
  normative tool interface for each delivery mechanism (§5.6) but does
  not define transport.
- **OIDC / OAuth 2.0** (RFC 6749, RFC 9068) — agent and originator
  authentication and token structure.
- **OAuth 2.0 Mutual-TLS** (RFC 8705) — channel-bound client
  authentication.
- **JWS / JWE** (RFC 7515 / RFC 7516) — manifest signing and SCT
  structure.
- **JWK / JWKS** (RFC 7517) — published key material for verification.
- **OpenTelemetry / W3C Trace Context / W3C Baggage** — transactional
  context propagation and trace correlation.
- **Industry grounding standards** — O\*NET, NAICS, APQC PCF, HL7, X12,
  ISO 20022, and others, referenced by IRI (§2.3).

## 1.3 Normative Language

The key words are interpreted per RFC 2119 / RFC 8174 as noted in the
Scope Note above.

## 1.4 Companion Documents

This document is part of the SADAR specification publication set. Where
this document references content by section, the authoritative content
lives in the cited companion.

| Document | Authoritative for |
| --- | --- |
| `2_Scope.md` | Specification scope, normative element catalog (§5.1), and IP licensing boundary. |
| NFR Schema | Manifest schema, the four NFR categories, three-tier matching strictness, the bilateral match algorithm, and registry-side validation. |
| SCT Operations | Signed Context Token structure, claims, and the five chain operations. |
| Trust Models | The four originator trust models, bilateral negotiation, and asserted-model server-side validation. |
| Telemetry Record and Repatriation | Per-invocation Telemetry Record schema, OTel span structure, provenance attribution, and governed repatriation. |
| searchAndInvoke Telemetry and Authentication | Telemetry Helper API, repatriation trigger, and the OIDC scope namespace. |
| Risk Score Specification | Cumulative in-flight risk score data model and propagation. |
| Conformance Specification | Conformance criteria, test suites, and validation rules. |

---

# 2. Core Concepts

## 2.1 Records in the Registry

A SADAR-conformant registry holds three categories of top-level record,
all of which are publisher-signed manifests:

| Record | Description |
| --- | --- |
| **Entity** | The organizational unit that owns Entries, Registries, and subordinate Entities. Every record traces ownership through the entity hierarchy to a single, legally accountable root entity. |
| **Entry** | A discoverable item published by an owning Entity. Entry subtypes are Agent, Tool, Resource, and Process (§2.2). |
| **Registry** | A peer registry record, used for federation and replication. Registries are themselves signed manifests, owned by an Entity, and discoverable on the same protocol surface as Entries. |

The discoverable **Entry** subtypes:

| Subtype | Description |
| --- | --- |
| **Agent** | An autonomous or semi-autonomous actor. An Agent manifest carries a `requester` role section, a `server` role section, or both. An Agent exposing multiple server endpoints publishes one manifest per endpoint. |
| **Tool** | A general-purpose technical capability not tied to a specific business function. Tools advertise as servers. |
| **Resource** | An accessible artifact — database, file, configuration, data set — made available to agents or tools. Resources advertise as servers. |
| **Process** | A definition of a multi-step (or, in the degenerate case, single-step) flow, grounded in a standard process taxonomy. A Process declares steps, sequencing, gating conditions, prerequisites, and exclusions. Process entries provide scaffolding for planner agents and a basis for dependency enforcement; they do not carry an invocation endpoint and are executed by Agents that reference them. |

Every record carries a `home_registry_id` identifying its registry of
origin, set at first registration and preserved across all forwarding,
replication, and federation operations (§6.4).

## 2.2 The Manifest

The manifest is the central artifact of SADAR: the publisher-signed,
machine-readable description of a record. A manifest is immutable once
registered; any change produces a new versioned manifest with a new
signature. The registry retains versions with their lifecycle state.

> The complete, authoritative manifest schema — field names, data types,
> serialization, the four NFR categories, the data-field and
> process-reference vocabularies, and matching semantics — is defined in
> the **NFR Schema** companion document. This section gives the
> structural overview required to read the discovery and invocation
> protocol; it is not the authoritative schema.

**Common fields** (every manifest, per `2_Scope.md` §5.1.2): a globally
unique manifest identifier, manifest version, schema version, lifecycle
state (one of draft, active, deprecated, superseded, retired),
`home_registry_id`, a reference to the owning entity, and a JWS
signature over the canonicalized content.

**Role sections.** An Agent manifest may carry one or both of two
symmetric role sections, expressed in the same vocabulary; interpretation
depends on the section a field appears in:

| Section | Direction | Interpretation |
| --- | --- | --- |
| `requester` | Outbound | What the agent wants when searching for and invoking other capabilities; matched against provider advertisements at discovery. |
| `server` | Inbound | What the agent advertises as its own callable capability, and — in a `requirements_of_requester` sub-section — what it requires of incoming requesters. |

Tools and Resources carry a `server` section only. Many agents declare
both sections, reflecting the operational reality that agents both call
and are called.

**Section element vocabulary.** Each role section carries declarations
across:

| Element | Content |
| --- | --- |
| Process Reference | The process(es) and step(s) the manifest is associated with, with prerequisites and exclusions. In `requester`, two distinct references appear: the requester's own upstream context, and the target process it wants the provider to perform. |
| Data Fields | IRI-based references to data definitions in published standards (X12, HL7, ISO 20022, and others), each carrying a strictness flag. |
| Financial NFRs | Cost, pricing, payment methods, billing. |
| Operational NFRs | Performance, throughput, availability, technical limits. |
| Governance NFRs | Compliance frameworks, data sovereignty, privacy, encryption, licensing. |
| Protocol NFRs | SADAR-specific behaviors: supported trust models, role declarations, telemetry-repatriation participation and field lists, optional impact score. |

**Strictness.** Every NFR field, process reference, data-field
declaration, and exclusion carries one of three matching strictness
flags — `OPTIONAL`, `MANDATORY`, or `MANDATORY_STRICT` — governing how a
mismatch affects discovery (§3.4).

## 2.3 Semantic Grounding and IRIs

Capabilities and data are grounded by Internationalized Resource
Identifier (IRI) in published standards. SADAR does not define those
standards; it defines how they are referenced as uniquely identified
namespaces and how specific elements within them are addressed.

| Dimension | Example grounding standards |
| --- | --- |
| Industry / entity | NAICS, ISIC, NACE, GICS; entity scoping via Moody's, S&P, and similar |
| Process / task | APQC PCF; industry frameworks such as eTOM (telecom), BIAN (banking), SCOR (supply chain); O\*NET task taxonomy |
| Operation / data | HL7, X12, ISO 20022, SWIFT, NCPDP; published APIs where standards are insufficient |

SADAR-defined identifiers use the namespace pattern
`urn:sadar:{category}:v{version}:{...}`. Ecosystem participants extending
the vocabulary SHALL use their own organization-controlled namespaces and
SHALL NOT use the `urn:sadar:` prefix.

> **Semantic match is not similarity.** Within SADAR, *semantic* refers
> to contextual business or technical meaning grounded in a shared
> standard definition. A semantic match means two declarations are
> grounded in the same definition — not that their names are
> lexically or vectorally similar. SADAR matching does not use cosine
> similarity or any probabilistic name comparison.

---

# 3. Registry Protocol

## 3.1 Transport

All communication between SADAR clients and registries SHALL use mutual
TLS (mTLS) per RFC 8705:

- The registry SHALL present a TLS certificate signed by a recognized CA
  or federation trust anchor.
- The client SHALL present a TLS client certificate identifying the
  agent.
- Certificate validation SHALL be bidirectional; connections where
  either party cannot validate the other SHALL be rejected.
- TLS 1.2 is the minimum; TLS 1.3 is RECOMMENDED. The algorithm and
  minimum key strength are as declared in the relevant manifest per
  `2_Scope.md` §5.1.8.1.

## 3.2 Authentication to the Registry

Before searching, the calling agent SHALL authenticate to the registry
using OIDC Client Credentials over the established mTLS channel, per the
authentication baseline (`2_Scope.md` §5.1.8.1). The access token is a
JWT carrying, at minimum, the agent's stable identifier, issuer,
audience (including the registry identifier), expiry, and the SADAR scope
authorizing the requested registry operation.

SADAR registry-interaction scopes live in the `urn:sadar:scope:v1:*`
namespace and include registry search, manifest resolution, registry
listing, and health (see the **searchAndInvoke Telemetry and
Authentication** companion for the complete scope set and TTL bounds).

> The registry does not make or enforce business-level authorization
> decisions. It returns candidates that satisfy bilateral matching and
> the requester's registered discoverability scope. Authorization
> enforcement is the responsibility of the providing entity and the
> consuming implementation (§5, and `2_Scope.md` §5.2.6).

## 3.3 Search Request

A search request conveys the requester's matching criteria so the
registry can evaluate both directions of the bilateral match (§3.4). At
minimum, the request conveys:

- The requester's identity, sufficient for the registry to resolve and
  verify the requester's published manifest and to apply its
  discoverability scope.
- The **target process** and capability the requester seeks, grounded by
  IRI.
- The requester's relevant `requester`-section declarations — data-field
  requirements, NFR constraints, and supported trust models — against
  which the provider's `requirements_of_requester` will be evaluated.

> The authoritative request and response wire formats are defined in the
> Registry Protocol and Resolver Contract sections of `2_Scope.md`
> (§5.1.6, §5.1.7) and the NFR Schema companion. Implementations SHALL
> NOT introduce request or response fields that alter the candidate set
> a fully conformant registry would return for the same normative inputs
> (`2_Scope.md` §5.2.2).

## 3.4 Bilateral Matching

The match operates in two directions, both of which SHALL succeed for a
candidate to be returned:

- **Direction 1** — the requester's declared wants (its `requester`
  section) are evaluated against the provider's advertisements (its
  `server` section).
- **Direction 2** — the provider's requirements of the requester (its
  `server.requirements_of_requester` sub-section) are evaluated against
  the requester's self-declarations.

Each field is evaluated per its strictness flag, producing one of three
per-direction outcomes:

| Outcome | Condition |
| --- | --- |
| **FULL_MATCH** | All `MANDATORY` and `MANDATORY_STRICT` fields satisfied; `OPTIONAL` preferences met or unmet without exclusion. |
| **PARTIAL_MATCH** | At least one `MANDATORY` (non-strict) field is a partial match, surfaced to the resolver for adjudication; no `MANDATORY_STRICT` field is unsatisfied. |
| **NO_MATCH** | At least one `MANDATORY_STRICT` field is unsatisfied; the candidate is excluded from results entirely, without notification. |

The combined outcome across the two directions is the stricter of the
two. Conformant registries SHALL produce consistent results for
identical inputs and normative fields, regardless of operator. The
authoritative match algorithm is defined in the NFR Schema companion.

Provider requirements prevent discovery by non-qualifying requesters:
this is the **right of refusal** at the discovery layer (a corresponding
right of refusal applies at invocation, §5.2).

## 3.5 Search Response

The response returns the candidate set surviving bilateral matching, each
candidate carrying its `FULL_MATCH` or `PARTIAL_MATCH` classification.
Candidates failing a `MANDATORY_STRICT` requirement are excluded
entirely. The response conveys, per candidate, the information the
resolver and the subsequent invocation require — including the selected
target's identity, its declared invocation and authentication endpoints
(resolved from its manifest), its negotiated trust model context, and
applicable TTLs. Error responses use the `urn:sadar:error:v1:*`
namespace.

Each entry declares a discovery TTL governing how long a discovered
result may be honored before rediscovery, and federation-related TTLs
govern replication scope (§6.3).

---

# 4. The Resolver Contract

After receiving the classified candidate set, the `searchAndInvoke`
implementation SHALL execute a resolver to select the target to invoke.
The resolver is supplied by the requesting organization. SADAR specifies
the contract the resolver SHALL honor; it does not specify resolver
logic.

## 4.1 Resolver Input

The implementation SHALL provide the resolver with the complete
classified candidate set returned by bilateral matching, including the
`FULL_MATCH` / `PARTIAL_MATCH` classification on each candidate, together
with the original search criteria and any context the implementation
deems relevant.

## 4.2 Resolver Output

The resolver SHALL return exactly one candidate from the input set, or a
defined error. Returning a target not present in the input set is a
protocol violation.

## 4.3 Resolver Constraints

The resolver:

- MAY prefer `FULL_MATCH` candidates and adjudicate `PARTIAL_MATCH`
  candidates per its own policy.
- SHALL NOT modify, augment, or substitute candidates outside the set
  returned by bilateral matching.
- SHALL NOT perform independent registry queries or otherwise bypass
  bilateral matching results.

Selection logic — rule-based, model-assisted, human-in-the-loop, or
otherwise — is implementer discretion within these constraints.
Implementation patterns for the resolver are non-normative and described
in the SADAR Reference Architecture.

---

# 5. Invocation Requirements

Once the resolver selects a target, the implementation SHALL invoke it.
SADAR specifies the identity, authorization-context, and verification
requirements for invocation; it does not specify the transport,
transformation, or routing mechanism, which are implementation concerns.

## 5.1 Identity and Authorization Context

Every conformant invocation carries three complementary elements.

### 5.1.1 Agent Authentication

The calling agent SHALL authenticate to the target using OIDC Client
Credentials over mTLS per the authentication baseline (`2_Scope.md`
§5.1.8.1), with a token scoped to the operation being invoked
(`urn:sadar:scope:v1:invocation:invoke` by default). The token issuer is
the **target's** authentication endpoint declared in the target's
manifest; the registry is never the token issuer.

### 5.1.2 Originator Authorization Context (SCT)

The originator's scope of authority SHALL be propagated using a Signed
Context Token (SCT) — a JWS-inside-JWE token carrying authorization
claims attested at each trust-boundary crossing. The SCT chain across
boundaries is the cryptographically attested record of how authorization
context evolved through the flow.

The trust model under which the originator's identity propagates is one
of four — `direct_auth`, `asserted`, `impersonation`, or `deputy` —
negotiated bilaterally at discovery via the `supported_trust_models`
Protocol NFR. The SCT and its five chain operations (Open, Continue,
Hold, Authoritative Carry, Close) are defined in the **SCT Operations**
companion; the trust models, negotiation algorithm, and asserted-model
validation are defined in the **Trust Models** companion.

> This replaces the bespoke originator-header mechanism of earlier
> drafts. Originator identity and authority travel in the SCT, not in a
> standalone HTTP header.

### 5.1.3 Transactional Context

The implementation SHALL propagate OpenTelemetry context per W3C Trace
Context (`traceparent` / `tracestate`) and W3C Baggage, binding the
invocation to the observable execution chain. Where the invocation occurs
within a declared process, the process identifier SHALL be carried in
OTel Baggage. The risk-score adjustment list SHALL be carried in OTel
Baggage per the Risk Score companion. Implementations SHALL NOT strip
`urn:sadar:`-prefixed baggage received from upstream callers.

## 5.2 Bilateral Manifest Verification at Invocation

A providing entity SHALL verify the manifest of any incoming requester at
invocation time, independent of any registry pre-screen. Verification
SHALL confirm that the requester's manifest signature verifies against
the requester's registry-published key material, that the manifest is in
an active lifecycle state, and that the bilateral match succeeds for the
invocation context. For `asserted`-model invocations, the provider SHALL
additionally apply the SCT-claim validation of the **Trust Models**
companion (asserter identity, originator namespacing, role-claim
validation with default-role handling, trust-model match).

Verification is fail-closed: on any failure the provider SHALL reject the
invocation with a structured `urn:sadar:error:v1:*` error and SHALL NOT
proceed under reduced trust. The registry pre-screen is a discovery-time
convenience, not a substitute for invocation-time verification.
Verification results MAY be cached within the manifest's declared
discovery TTL.

This is the **right of refusal** at the invocation layer.

## 5.3 Transport for Invocation

- Invocations SHALL be made over a secure transport; mTLS is RECOMMENDED
  for the invocation channel, and where it is not available the condition
  SHOULD be logged.
- Implementations SHALL respect the operational NFRs declared in the
  selected manifest (timeouts, retry limits, rate limits).

## 5.4 Response Handling

- Implementations SHOULD validate responses against the selected
  operation's declared data contract before passing them to the calling
  agent, and SHOULD log violations.
- Implementations SHALL propagate OTel trace context on the response
  path.

## 5.5 Telemetry and Repatriation

Every invocation produces a per-invocation Telemetry Record — the
canonical persistent audit artifact — accessed only through the SADAR
Telemetry Helper API. Every conformant span carries provenance
attribution identifying the issuing entity, agent, and environment. Where
repatriation is bilaterally declared and matched at discovery, the
provider returns redacted trace fragments to its immediate caller at
trust-boundary crossings, preserving a normative set of non-redactable
structural fields. The Telemetry Record schema, span structure,
provenance attribution, and repatriation mechanics are defined in the
**Telemetry Record and Repatriation** and **searchAndInvoke Telemetry and
Authentication** companions.

---

# 6. Registry Architecture and Federation

## 6.1 Registry Identity

A registry SHALL be identified by a stable identifier and SHALL itself be
a signed manifest owned by an accountable Entity, discoverable on the
same protocol surface as Entries.

## 6.2 Registry Types

SADAR defines six registry roles: Provider, Marketplace, Industry,
Community, Internal, and Registry of Registries. A registry's role and
its operational, licensing, and billing NFRs are declared in its
manifest.

## 6.3 Federation

Registries may operate independently or participate in a federated
network governed by the Directory of Authorized Registries.

- Conformant registries SHALL reject forwarding and replication requests
  from registries not listed in the Directory of Authorized Registries.
- Between authorized registries, a forwarding or replication request may
  be declined only if it violates the receiving registry's declared NFRs.
- **Query forwarding** — registries may forward queries to other
  authorized registries within their terms of service.
- **Content replication** — pull-based replication of specific content;
  replicated entries are augmented with their home registry reference so
  discovery and invocation flows are identical for local and replicated
  entries.
- All forwarding and replication settings are configured by local
  registry administrators, inbound and outbound.
- Federation authentication follows the same OIDC-over-mTLS baseline as
  any other interaction; there is no separate federation token system.
- Private, non-authorized registries are fully supported for internal use
  — standalone or internally federated — without participation in the
  public Directory.

The authoritative federation specification is `2_Scope.md` §5.1.11.

## 6.4 Provenance Preservation

Every Entity, Entry, and Registry record carries a `home_registry_id`
identifying its registry of origin. Conformant registries SHALL preserve
`home_registry_id` unchanged across all forwarding, replication, and
re-publication. Replicated records SHALL remain discoverable by
`home_registry_id` so consumers can distinguish records native to a
queried registry from those replicated into it.

## 6.5 Registry of Registries

A Registry of Registries (RoR) MAY serve as the authoritative trust
anchor for federation topology, publishing the set of authorized
registries, their identifiers, their trust material, and their
replication relationships. The RoR holds registry descriptors only; it
holds no credentials, keys, or operational data.

## 6.6 Registry Isolation

The registry — and the RoR — facilitate the exchange of signed manifests
and nothing more. They SHALL NOT hold usage tokens, credentials, keys, or
sensitive operational data, SHALL NOT alter manifests, and SHALL NOT
issue authentication tokens for invocations. After discovery, consumer
and provider communicate directly; runtime interactions do not depend on
registry availability until the discovery TTL expires.

## 6.7 Availability

The Directory of Authorized Registries and individual registries follow
standard enterprise high-availability and geographic-distribution
patterns and MAY be distributed across multiple clouds, geographies, and
jurisdictions. Non-normative deployment guidance is provided in the SADAR
Implementation Guidance companion.

---

# 7. Conformance

Conformance to SADAR is evaluated against the normative elements of
`2_Scope.md` §5.1 that apply to the role an implementation plays.
Conformance criteria, test suites, and validation rules are defined in
the Conformance Specification companion; verification tiers
(Self-Declared and Assessed) and certification are governed by
`2_Scope.md` §7 and the OpenSemantics.org Charter.

> **Note (non-normative).** Earlier drafts defined implementation-subset
> conformance "levels" (Core / Registry / Federation). That model is not
> carried forward, because `2_Scope.md` §7 evaluates conformance against
> the applicable normative elements in their entirety rather than as
> selectable tiers. Role-based applicability is the reconciliation: an
> implementation that is not a registry operator need not satisfy
> registry-operator requirements, but SHALL satisfy every normative
> requirement applicable to the role it does play.

## 7.1 Implementation Latitude

The following are explicitly left to the implementer and are out of
scope for the IP licensing commitments per `2_Scope.md` §5.2:

- Registry storage and indexing mechanisms.
- The resolver's selection logic.
- Authorization policy decision and enforcement (RBAC, ABAC, zero-trust,
  or other) applied to validated identity and authorization context.
- The risk-score accumulation algorithm and any intervention thresholds.
- Invocation execution, transformation, and routing mechanisms.
- Telemetry storage, retention, and downstream processing.
- Payment settlement mechanics.

---

# 8. Security Considerations

## 8.1 mTLS as the Trust Foundation

mTLS is mandatory rather than recommended because it provides
bidirectional authentication at the transport layer, independent of
application-layer credential validity. It defends against registry- and
target-misdirection attacks: even with intercepted DNS or routing, an
attacker cannot present a valid certificate for the genuine identifier
without the corresponding private key.

## 8.2 Manifest Integrity

Manifests are publisher-signed (JWS) and immutable. Consumers SHOULD
verify a manifest's signature against the publisher's registry-published
key material before using it to configure an invocation; a compromised or
substituted manifest could otherwise redirect invocations. Manifests
SHALL NOT be used beyond their declared discovery TTL without
revalidation.

## 8.3 Bilateral Verification and Right of Refusal

Discovery-time bilateral matching prevents non-qualifying requesters from
discovering entries they could not invoke, and invocation-time bilateral
manifest verification (§5.2) re-confirms the match fail-closed,
independent of any registry pre-screen. Together these provide
right-of-refusal at both the discovery and invocation layers.

## 8.4 SCT Validation and Trust Models

The originator's authority is carried in the SCT under a negotiated trust
model. Providers SHALL validate incoming SCT claims at invocation;
for the `asserted` model the normative validation algorithm of the Trust
Models companion applies, and all validation is fail-closed. The
`deputy` model is preferred over `impersonation` where both are feasible,
because it preserves agent attribution at every hop.

## 8.5 Process Scope and Out-of-Sequence Defense

Where a process context is declared, discovery evaluates process-context
compatibility, and prerequisites declared in manifests allow a provider
to require that prior steps be in scope before invocation. This is a
defense against out-of-sequence invocation and against prompt-injection
attempts to drive an agent to invoke out-of-context capabilities. SADAR
supplies the context and the declarations; enforcement is the
implementer's responsibility.

## 8.6 Registry Isolation as Attack-Surface Reduction

Because the registry is never in the runtime call path and never holds
credentials, keys, or usage tokens, it is neither a runtime bottleneck
nor a high-value single point of compromise. Compromise of a registry
exposes signed, already-public manifests — not operational secrets.

---

# Appendix A — Normative References

- RFC 2119 / RFC 8174 — Requirement-level key words
- RFC 6749 — OAuth 2.0 Authorization Framework
- RFC 8705 — OAuth 2.0 Mutual-TLS Client Authentication
- RFC 9068 — JWT Profile for OAuth 2.0 Access Tokens
- RFC 7515 / RFC 7516 — JSON Web Signature / JSON Web Encryption
- RFC 7517 — JSON Web Key (JWK)
- RFC 3339 — Date and Time on the Internet: Timestamps
- W3C Trace Context; W3C Baggage
- OpenTelemetry specification

# Appendix B — Cross-Reference to Scope

| This document | `2_Scope.md` |
| --- | --- |
| §2.1 Records | §1 Overview; §5.1.1 Entity Model |
| §2.2 Manifest | §5.1.2 Manifest Structure |
| §2.3 Semantic grounding | §4; §5.1.16 Semantic Contracts and Identifiers |
| §3.4 Bilateral matching | §5.1.3 |
| §4 Resolver contract | §5.1.7; §5.1.13 Layer 2 |
| §5.1.2 SCT / trust models | §5.1.8.2, §5.1.8.3 |
| §5.2 Invocation verification | §5.1.3 |
| §5.5 Telemetry & repatriation | §5.1.14 |
| §6 Federation | §5.1.11; §5.1.12 |
| §7 Conformance | §5.1.19; §7 |

---

*© 2026 Cognita AI Inc. SADAR™ is a trademark of Cognita AI Inc.,
stewarded through OpenSemantics.org. Use of the SADAR™ name or
certification mark is governed by the OpenSemantics.org Trademark Policy.
This specification is licensed under the Community Specification License
1.0.*
