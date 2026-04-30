# SADAR Specification Scope

**Specification:** Semantic Agent Discovery and Attribution Registry (SADAR)
**Version:** 0.10 DRAFT
**Steward:** Cognita AI Inc.
**License:** Community Specification License 1.0
**Status:** Draft — subject to change prior to Approved Specification designation

---

## Companion Documents

This Scope document is part of the SADAR specification publication set.
The following companion documents are published concurrently and are
referenced throughout this document. Where this Scope document
references normative content by section, the authoritative content
lives in the cited companion document; this Scope document carries
the umbrella requirement and the IP licensing boundary.

| Document | Description |
|----------|-------------|
| 8. NFR Schema | Non-Functional Requirements vocabulary, manifest structure, four NFR categories, three-tier matching strictness, bilateral match algorithm, registry-side validation. |
| 9. SCT Operations | Signed Context Token chain operations (Open, Continue, Hold, Authoritative Carry, Close), structure, claims, and chain semantics. |
| 10. Trust Models | The four originator trust models (direct_auth, asserted, impersonation, deputy), bilateral negotiation, tie-break rules, and asserted trust model role declarations. |
| 11. Telemetry Record and Repatriation | Per-invocation Telemetry Record schema, OTel span structure with provenance attribution, Helper API requirement, governed cross-trust-boundary repatriation. |
| 12. searchAndInvoke Telemetry and Authentication | Telemetry Helper API surface, repatriation trigger specification, OIDC scope namespace and initial scope set. |
| 13. Risk Score Specification | Cumulative in-flight risk score, Adjustment data model, Baggage and SCT propagation, implementer accumulation algorithm, NIST SP 800-30 framing. |
| SADAR Conformance Specification | Conformance criteria, test suites, and validation rules. |
| OpenSemantics.org Charter | Governance, working group structure, contribution process, and the SADAR Certification Program. |
| OpenSemantics.org Trademark Policy | SADAR mark usage rights, certification mark authorization, and enforcement. |
| SADAR Reference Architecture | Non-normative implementation patterns, including the searchAndInvoke handler architecture, selection strategy patterns, and input/output validation approaches. |
| SADAR Implementation Guidance | Non-normative operational guidance including availability, high availability, geographic distribution, and deployment patterns. |
| Community Specification License 1.0 | The IP licensing framework governing this specification. |
| Notices.md | Authoritative record of contributor copyright ownership and patent licensing commitments. |

---

## 1. Overview

SADAR defines a standards-based framework for the discovery,
negotiation, and invocation of agents, tools, and services using
pre-existing industry standard definitions expressed in a structured
semantic format with non-functional requirements (NFRs) spanning
Financial, Operational, Governance, and Protocol categories. The
specification includes a Security Architecture for agent, tool, and resource
authentication and preservation of originating scope of authority.
Further, the standard provides a means of defining processes — business
or technical — as multi-step flows, each carrying a unique transaction
instance identifier throughout the flow execution.

The specification enables interoperable, explainable, and policy-aware
interactions across heterogeneous AI systems, enterprise platforms,
and service providers.

The specification defines required compatibility evaluation criteria
such that conformant registry implementations MUST produce consistent
results given identical inputs and normative fields, regardless of the
registry operator. The adjudication of the candidate result set to the
final selected entry is the sole responsibility of the requesting
agent's organization.

The registry operates exclusively at discovery time. It is not in the
runtime call path and does not hold sensitive operational data,
credentials, or usage tokens. After discovery is complete, consumers
and servers communicate directly.

A SADAR-conformant registry holds three categories of top-level
records, all of which are signed manifests:

| Record | Description |
|--------|-------------|
| Entity | The organizational unit that owns Entries, Registries, and other Entities. Every Entry and every Registry record traces ownership through the entity hierarchy to a single legally accountable root entity. |
| Entry | A discoverable item published by an owning Entity. Entry subtypes are defined below. |
| Registry | A peer registry record, used for federation and replication. Registries are themselves signed manifests, owned by an Entity, and discoverable on the same protocol surface as Entries. |

The discoverable **Entry** subtypes are:

| Entry Subtype | Description |
|---------------|-------------|
| Agent | An autonomous or semi-autonomous actor exhibiting the nature of agents in Agentic AI (planning, autonomous execution). An Agent's manifest carries a `requester` section, a `server` section, or both, depending on the roles the Agent plays. An Agent that exposes multiple server endpoints (methods, functions, or APIs) publishes one Agent manifest per endpoint. |
| Tool | A general-purpose technical capability not tied to specific business functionality. Tools advertise as servers. |
| Resource | An accessible artifact made available to Agents or Tools — databases, files, configurations, data sets, and similar. Resources may be technical or business in nature. Resources advertise as servers. |
| Process | A definition of a multi-step (or, in the degenerate case, single-step) flow, grounded in standard process taxonomies. A Process declares its steps, sequencing, gating conditions, exclusions, and prerequisites. Processes may be business processes (e.g., order-to-cash, claims adjudication) or technical processes (e.g., document ingestion, ETL workflows). Process entries provide scaffolding for planner Agents and sequencing for dependency guardrails. |

All top-level records — Entity, Entry, and Registry — carry a
`home_registry_id` identifying the registry of origin. This field is
set at first registration and is preserved across all forwarding,
replication, and federation operations to maintain an unambiguous
provenance trail.

**NOTE:** Within SADAR and its documentation, *semantic* shall not be confused with vector comparision such as cosine similarity.  Within SADAR, the term *semantic* refers to the contextual, business or technical meaning. Semantic meaning is defined using industry standard definitions such as the American Productivity & Quality Center process definitions, standard transactions such as X12, HL7, SWIFT, etc. or fully declared APIs.

A *semantic match* in SADAR does not mean that the names are semantically similar.  Instead it means that the semantic **meaning** of the definitions is grounded in the same definition.
---

## 2. Purpose

- Provide a common language for intelligent system interoperability
- Enable semantic discovery of agents, tools, resources, and processes
- Support negotiation of functional and non-functional requirements
- Ensure alignment with business processes and regulatory constraints
- Facilitate explainable, auditable decision-making
- Establish verifiable identity and attribution for every registry
  interaction and invocation
- Provide a substrate for risk-aware run-time control through
  cumulative in-flight risk observation

---

## 3. Guiding Principles

- **Interoperability** — conformant implementations work across
  heterogeneous platforms
- **Semantic Precision** — capabilities and constraints are expressed
  using standardized, externally grounded taxonomies
- **Explainability** — discovery and selection decisions are
  attributable and auditable; execution flows across conforming implementations is attributable with transaction context
- **Consistency** — conformant registry implementations MUST produce
  consistent results given identical inputs and normative fields;
  compatibility evaluation criteria are defined by the specification,
  not by implementer discretion
- **Extensibility** — conformant implementations may add capabilities
  beyond normative requirements without affecting conformance
- **Neutrality** — the specification does not prescribe execution
  runtimes, policy engines, or enforcement mechanisms beyond the
  normative interface contracts defined herein
- **Security and Trust** — identity, authentication, authorization,
  and nonrepudiation are first-class elements of the specification; originating scope of authority is preserved
- **Registry Isolation** — the registry is a discovery-time component
  only; it is never in the runtime call path and never holds sensitive
  operational data
- **Risk Awareness** — the specification provides normative substrate
  for cumulative observation of in-flight risk; how implementations
  consume that substrate is implementer concern, but the substrate
  itself is normative

---

## 4. Relationship to Other Standards

SADAR provides a semantic and governance discovery layer that
complements existing standards and frameworks. It does not replace
them.

### Transport, MCP, and A2A

SADAR does not define transport protocols. Those are outside the
specification scope. SADAR is intended to be fully interoperable with
Model Context Protocol (MCP) and Agent-to-Agent (A2A) specifications.
The SADAR specification defines this interoperability, including the
normative SADAR tool interface definitions for each delivery
mechanism.

### Industry Categories, Process Definitions, and Operations

SADAR does not define standards for industry categories, process
definitions, or domain-specific operations and documents. Instead, it
provides mechanisms for namespacing these dimensions using
pre-existing standards (O*NET, NAICS, APQC PCF, HL7, X12, ISO 20022,
and others). The specification does not impose requirements on those
external standards; it defines how they are represented as uniquely
identified namespaces denoted by Internationalized Resource
Identifiers (IRIs) that directly reference those standards for
semantic and syntactical clarity. SADAR further introduces a
specification for identifying references within these standards.

**NOTE:** Where industry standards do not exist or, particuarly at the operation level are not sufficiently granular, published API documentation can be used as the namespace.

### Identity, Authorization, Signing, and Encryption

SADAR uses existing standards — including OIDC, OAuth, JWS, and JWE —
for security and encryption purposes. It establishes usage patterns,
normative requirements, and claims extensions on top of those
standards.

### Observability

SADAR normatively requires the use of OpenTelemetry (OTel) for
traceability, consistent with the precedent established by the
Agent-to-Agent (A2A) specification. SADAR extends OTel through
normatively defined baggage fields that carry business process
context, risk score state, and other invocation-time context required
for monitoring and attribution across conformant implementations.

### Risk and Impact Frameworks

SADAR does not adopt any specific risk or impact framework
normatively. NIST SP 800-30 Revision 1, Appendix H is referenced
informatively in the Risk Score Specification companion document as
best-practice framing for impact-magnitude calibration; organizations
using SADAR are expected to tailor SP 800-30's qualitative scales (or
equivalent frameworks) to their operational context.

---

## 5. Specification Scope and IP Licensing Boundaries

### 5.0 Intellectual Property Rights and Obligations

Contributors to the SADAR specification retain copyright in their
individual contributions. As contributions are incorporated into the
specification, SADAR becomes a collective work with distributed
copyright ownership. Copyright notices of all contributors are
permanent and may not be removed.

Under the Community Specification License 1.0, Contributors grant all
Licensees a perpetual, worldwide, royalty-free license to reproduce,
use, and create derivative works from the specification. This grant
covers:

- **Copyright** — the right to reproduce, adapt, and distribute the
  specification text and normative artifacts
- **Patent** — a royalty-free license to Necessary Claims, defined as
  those patent claims necessarily infringed by implementing the
  normative requirements of this Specification and for which no
  non-infringing alternative implementation exists

Licensees reproducing or implementing this specification must:

- Retain all existing copyright notices
- Include the Community Specification License 1.0 with any
  distribution
- Attribute the work to the **SADAR Specification** stewarded by
  **OpenSemantics.org**

Attribution runs to the specification and its steward, not to
individual contributors by name. The authoritative record of
contributor copyright ownership is maintained in `Notices.md` and the
specification repository commit history.

**Licensing commitments apply only to the normative elements of the specification. Elements not normatively defined herein are outside the scope of these commitments regardless of whether a particular implementation includes them.**

---

### 5.1 In Scope (Normative Elements)

The following elements are within Scope for purposes of the IP
licensing commitments described in Section 5.0.

---

#### §5.1.1 Entity Model

The normative definition of the organizational unit that publishes
or consumes registry entries, including:

- Entity manifest structure — signed manifests defining entity
  identity, OIDC endpoints, JWKS, payment methods, and default
  handlers
- Entity hierarchy — a tree structure with a single legally
  accountable root entity; all non-root entities have exactly one
  parent entity
- Record ownership — every Entry and every Registry record in a
  SADAR-conformant registry has exactly one owning Entity. An Entity
  may also own subordinate Entities, forming the entity hierarchy
  defined above.
- Least-privilege administration — normative delegation of entity
  administration across organizational boundaries, legal
  jurisdictions, and operational teams
- Namespacing — entity-scoped naming that prevents collision of
  identically named entries across entities, required for correct
  disambiguation in federated registries
- Discoverability levels — a normative enumerated type governing
  the visibility of an entity and its entries:

  | Level | Description |
  |-------|-------------|
  | Entity-Only | Discoverable only by agents within this entity |
  | Entity-Peers | Discoverable across entities sharing the same parent, within the same root |
  | Entity-Tree | Discoverable within the entity and any of its children |
  | Root-Wide | Discoverable within all entities under the common root |
  | Public | Discoverable within the registry |
  | Federated | Eligible for replication and/or discovery via forwarding |

- Accountability chain — every registry entry traces through the
  entity hierarchy to a legally accountable root entity

---

#### §5.1.2 Manifest Structure

The normative schema, required fields, optional fields, data types,
and serialization format of the SADAR Manifest. Every top-level
record in a SADAR-conformant registry — Entity, Entry, and Registry
— is a publisher-signed, immutable manifest. Any change to a record
produces a new versioned manifest with a new signature.

**Common Manifest Fields.** Every manifest, regardless of record
type, carries the following normative top-level fields:

- `manifest_urn` — globally unique manifest identifier
- `manifest_version` — monotonically increasing version of this
  manifest
- `schema_version` — version of the SADAR manifest schema in use
- `lifecycle_state` — one of {draft, active, deprecated, superseded,
  retired}
- `home_registry_id` — identifier of the registry of origin; set at
  first registration and preserved across all replication and
  forwarding operations
- `owning_entity` — reference to the owning Entity manifest
- Cryptographic signature structure using JWS

**Per-Record-Type Manifest Structure.**

| Record | Manifest Sections |
|--------|-------------------|
| Entity | Identity, OIDC endpoints, JWKS, payment methods, default handlers, discoverability level, parent entity reference (if non-root). Defined in §5.1.1. |
| Registry | Registry role/type (Provider, Marketplace, Industry, Community, Internal, Registry of Registries), federation endpoints, NFR declarations for operational controls, licensing, and billing. Defined in §5.1.11. |
| Agent | One or both of the role sections defined below: `requester`, `server`. Each section follows the Entry NFR vocabulary. An Agent carries an `endpoint_urn` only when its manifest declares a `server` section. |
| Tool | A `server` section only. Carries `endpoint_urn`. |
| Resource | A `server` section only. Carries `endpoint_urn`. |
| Process | A process declaration: grounding standard reference, ordered or partially ordered step list, sequencing metadata, gating conditions, prerequisites, and exclusions. Process entries do not themselves carry an `endpoint_urn`; execution of a Process is performed by Agents that reference the Process declaration. |

**Role Sections (Agent Entries).** An Agent manifest may carry one
or both of the following role sections:

| Section | Direction | Interpretation |
|---------|-----------|----------------|
| `requester` | Outbound | What the Agent wants when searching for and invoking other capabilities; matched against server advertisements at discovery |
| `server` | Inbound | What the Agent advertises as its own callable capability; what it requires of incoming requesters (in `requirements_of_requester`); matched against requester search criteria at discovery |

The same NFR vocabulary applies in both sections; interpretation is
determined by which section a field appears in. Many Agents act in
both roles simultaneously and declare both sections within a single
manifest. An Agent that exposes multiple distinct server endpoints
publishes one Agent manifest per endpoint, each with its own
`endpoint_urn` and its own `server` section. Multi-capability
Agents that share a deployment endpoint may publish multiple
manifests sharing a common `endpoint_urn`.

**Section Element Vocabulary.** Each `requester` or `server`
section, and the analogous advertisement structure of Tool and
Resource manifests, carries declarations across the following
elements:

| Element | Normative Content |
|---------|-------------------|
| Process Reference | Reference to the Process(es) the manifest is associated with, the step(s) involved, prerequisites, and exclusions. In an Agent's `requester` section, two distinct process references are present: the requester's own context, and the target process the requester wants the server to perform. |
| Data Fields | IRI-based references to data definitions across published standards (X12, HL7, ISO, D&B, and others), with three-tier matching strictness and exclusions |
| Financial NFRs | Cost, pricing, payment methods, billing |
| Operational NFRs | Performance, throughput, availability, technical limits |
| Governance NFRs | Compliance frameworks, data sovereignty, privacy, encryption, licensing |
| Protocol NFRs | SADAR-specific behaviors: supported trust models, role declarations, telemetry repatriation participation and field-list declarations, impact score |

**Strictness, Lifecycle, and Schema.** The normative manifest
schema additionally includes:

- Three-tier matching strictness flag (`OPTIONAL`, `MANDATORY`,
  `MANDATORY_STRICT`) applicable at every NFR field, process
  reference, data field declaration, and exclusion
- Lifecycle state transitions and the constraints governing them

The complete manifest schema, including the normative vocabulary
for all NFR fields and matching semantics, is defined in the
**3. NFR Schema** companion document. The umbrella requirement is
that SADAR-conformant manifests SHALL conform to the schema defined
there.

---

#### §5.1.3 Bilateral Discovery, Compatibility Criteria, and Manifest Verification

The normative compatibility evaluation criteria and protocol by which
a requester's requirements and a server's declarations are matched,
verified, and admitted as candidate invocations.

**Compatibility Evaluation.** The match operates bilaterally across
two directions:

- **Direction 1** — the requester's wants (declared in `requester`)
  are evaluated against the server's advertisements (declared in
  `server`)
- **Direction 2** — the server's requirements of the requester
  (declared in `server.requirements_of_requester`) are evaluated
  against the requester's self-declarations

Both directions SHALL succeed for a candidate to be returned. The
match outcome is one of FULL_MATCH, PARTIAL_MATCH, or NO_MATCH per
the strictness flags on individual fields; combination across the
two directions uses the strictest-of-two rule. Candidates failing
MANDATORY_STRICT requirements are excluded from the candidate set
without notification. Candidates passing with PARTIAL_MATCH on
MANDATORY (non-strict) fields are surfaced to the resolver for final
adjudication.

Conformant implementations MUST produce consistent results given
identical inputs and normative fields. The match algorithm is
defined normatively in the **3. NFR Schema** companion document.

**Bilateral Manifest Verification (server-side).** A providing
entity SHALL verify the manifest of any incoming requester at
invocation time, independent of any registry pre-screen the
requester may have performed. The verification SHALL confirm:

- The requester's manifest signature verifies against the requester's
  registry-published key material
- The requester's manifest is in an active lifecycle state
- The bilateral match succeeds for the invocation context

Manifest verification at invocation is a SADAR conformance
requirement; the registry pre-screen is a discovery-time convenience,
not a substitute for invocation-time verification. Manifest
verification results MAY be cached within the manifest's declared
Discovery TTL. Verification failure is fail-closed — the providing
entity SHALL reject the invocation with structured error and SHALL
NOT silently proceed under reduced trust.

**SCT Claim Validation (server-side).** A providing entity SHALL
validate the SCT claims carried with each invocation. SCT-claim
validation is keyed to the negotiated trust model:

- Where the SCT's originating user trust claim is `asserted`, the
  server SHALL apply the validation rules defined in the
  **5. Trust Models** companion document. These include asserter
  identity validation, originator namespacing validation, Role claim
  validation with default-Role handling, and trust model match
- For other trust models, validation requirements are at the
  providing entity's discretion subject to the requirements of
  §5.1.8 and §5.1.15

Validation failure is fail-closed. Structured errors use the
`urn:sadar:error:v1:*` namespace.

Server requirements prevent discovery and invocation by
non-qualifying requesters; this is the right-of-refusal at both the
discovery and invocation layers.

---

#### §5.1.4 Compliance Posture Matching

The normative bilateral compliance framework evaluated at discovery
time, prior to any invocation. Compliance is one of the Governance
NFRs and is matched per the bilateral match algorithm of §5.1.3.

- Requesters declare applicable compliance frameworks in their
  manifests using IRIs from the SADAR-maintained compliance framework
  registry, in the form `urn:sadar:compliance:v1:{framework}`
- Servers configure the compliance posture requirements requesters
  must assert to receive discovery results, and declare their own
  compliance posture
- Matching is bilateral — a requester can require compliant servers;
  a server can restrict visibility to requesters asserting a
  compatible posture
- Compliance alignment is verified at discovery, not assumed after
  invocation
- Normative format for referencing compliance documentation and
  proof of compliance, including authenticated access via OIDC for
  sensitive documents such as audit results
- Licensing metadata may specify that first use of a
  compliance-scoped capability requires human review or out-of-band
  verification before automated provisioning

The complete compliance framework registry, with normative IRIs for
SOC 2, ISO 27001, HIPAA, GDPR, FedRAMP, and others, is maintained
in the **3. NFR Schema** companion document.

---

#### §5.1.5 Process Metadata

The normative requirements for declaring process context within
manifests, and the normative structure of standalone Process
entries. The specification defines what MUST be declared; how
implementations consume or act upon this metadata beyond the
normative discovery criteria is left to implementer discretion.

A **Process** in SADAR is a definition of an ordered or partially
ordered set of steps, grounded in a standard process taxonomy. A
Process may describe a business flow (e.g., APQC PCF activities,
HL7 workflows, X12 transactional flows) or a technical flow (e.g.,
document ingestion pipeline, ETL workflow, multi-stage validation).
Single-step operations are representable as degenerate Processes
with one step.

- Manifests declare process step(s) using the structured process
  declaration schema, grounded in standard frameworks (APQC PCF,
  HL7 workflows, X12 transactions, and others as normatively
  defined; technical flows may be grounded in published API or
  operational schemas where industry standards are insufficient
  per §4)
- Process declarations support exclusions (sub-steps not
  performed) and prerequisites (steps required to be in scope),
  each with their own strictness flag
- Agents acting as requesters declare two distinct process
  contexts: their own (used by enforcement layers to evaluate
  context-aware authorization) and the target (what the requester
  wants the server to do)
- Agents, Tools, and Resources acting as servers declare the
  process step(s) they offer
- Discovery MUST evaluate process context compatibility per the
  bilateral match algorithm of §5.1.3
- The process identifier MUST be carried in OTel baggage as
  defined in §5.1.14
- Normative structure for `process` registry entries — top-level
  Process records defining complete (multi-step) processes
  independently of any single Agent, Tool, or Resource

The complete process declaration schema, including exclusion and
prerequisite semantics and matching behavior, is defined in the
**3. NFR Schema** companion document.

---

#### §5.1.6 Registry Protocol

The normative protocol governing interaction with a SADAR-conformant
registry, including:

- Registration, update, and deregistration request and response
  formats
- Registry acknowledgment and error response formats
- Validation rules for manifest acceptance, including key and hash
  validation on ingestion
- Cross-field validation requirements including:
  - Asserted trust model prerequisite — manifests declaring
    `asserted` in `supported_trust_models` SHALL declare a
    non-empty `supported_roles` array
  - Repatriation redacted fields validation — server
    manifests declaring `repatriation_participation = true` SHALL
    declare `repatriation_redacted_fields` containing no
    non-redactable structural fields per the Telemetry Record and
    Repatriation companion document
  - Role uniqueness — within `supported_roles`, `role_id` values
    SHALL be unique and at most one role SHALL declare
    `is_default = true`
- Minimum required registry behaviors for acceptance, rejection,
  and error signaling
- Manifest immutability enforcement — changes must produce new
  versioned entries with new signatures
- Structured error codes under `urn:sadar:error:v1:*` for validation
  failures

The complete registry-side validation specification is defined in
the **3. NFR Schema** companion document.

---

#### §5.1.7 Resolver Contract

The normative interface by which consumers query a SADAR-conformant
registry to discover entries by capability, trust level, or identity,
applying the required compatibility evaluation criteria defined in
§5.1.3:

- Query request structure and required and optional parameters
- Response payload format and required fields
- Result set — candidate entries with match classification
  (FULL_MATCH or PARTIAL_MATCH); candidates failing MANDATORY_STRICT
  requirements are excluded from the candidate set entirely
- Resolver receives the classified candidate set; the resolver's
  selection logic acts on classification (it MAY prefer FULL_MATCH
  candidates and adjudicate PARTIAL_MATCH candidates per its policy)
- Pagination and result ordering requirements
- Defined error conditions and required error response formats

---

#### §5.1.8 Identity, Authentication, and Trust

The normative architecture for establishing identity, authenticating
to other entities, propagating authorization context across trust
boundaries, and validating that context at invocation. This Section
is structured into six sub-subsections, each cross-referencing the
companion document where the detailed normative content resides.

##### §5.1.8.1 Authentication Baseline

The minimum required authentication mechanism for all SADAR
interactions:

- OIDC Client Credentials over mTLS as normative baseline
- TLS 1.2 minimum
- Algorithm and minimum key strength requirements: minimum 256-bit
  encryption; algorithm explicitly declared in the manifest
- Manifest fields normatively required for authentication: OIDC
  endpoint, JWKS endpoint or in-manifest key reference, signing key
  (`use: sig`) for manifest verification, encryption key
  (`use: enc`) for usage key delivery
- Key rotation by JWKS endpoint update without manifest republication
- All registry-to-registry, agent-to-registry, and agent-to-agent
  authentication SHALL conform to this baseline

##### §5.1.8.2 Signed Context Token (SCT)

Authorization context propagated across trust boundaries. The SCT is
a JWS-inside-JWE token carrying authorization claims attested at
each trust boundary crossing. The SCT chain across multiple
boundaries forms the cryptographically attested record of how
authorization context evolved through the flow.

The SCT supports five chain operations:

| Operation | Purpose |
|-----------|---------|
| Open | First-link issuance establishing the originator's authorization context |
| Continue | Mid-chain re-issuance preserving and propagating context |
| Hold | Suspension of the chain for asynchronous continuation |
| Authoritative Carry | Continuation with elevated authority appropriate to the carrying entity |
| Close | Terminal issuance ending the chain |

The complete SCT schema, chain operation semantics, claim
specifications, and the canonical conformance pattern are defined in
the **4. SCT Operations** companion document.

##### §5.1.8.3 Originator Trust Models

The trust model under which an originator's identity is propagated
through the SCT chain. SADAR defines four trust models:

| Model | Semantics |
|-------|-----------|
| direct_auth | Originator authenticated directly with an IdP this session |
| asserted | Calling application asserts originator identity and Role under its own service credentials |
| impersonation | Agent operates as the originator at downstream services (setuid-style) |
| deputy | Agent operates with originator authority but its own identity (constrained delegation) |

Each invocation operates under one trust model, negotiated bilaterally
between requester and server at discovery time per the
`supported_trust_models` Protocol NFR. Where multiple trust models
are mutually supported, the negotiation algorithm selects the model
of closest combined preference rank, with deputy preferred in ties.

The complete trust model definitions, negotiation algorithm,
asserted-model role schema, and tie-break rules are defined in the
**5. Trust Models** companion document.

##### §5.1.8.4 Server-Side SCT Claim Validation

Per §5.1.3, providing entities validate SCT claims at
invocation. For the asserted trust model, validation includes
asserter identity verification, originator namespacing verification,
Role claim presence and recognition with default-Role handling, and
trust model match per §5.1.15.

The complete asserted-model validation algorithm, error codes, and
fail-closed semantics are defined in the **5. Trust Models** companion
document.

##### §5.1.8.5 Authentication Scopes

OIDC scope claim values used by SADAR. The `urn:sadar:scope:v1:*`
namespace is reserved for SADAR-defined scopes. The initial scope
set covers registry interactions, telemetry repatriation, and
invocation. Token TTL bounds: minimum 60 seconds, maximum 24 hours,
recommended default 15 minutes. Lazy refresh — tokens are refreshed
on next use after expiry, not proactively.

The token issuer is the **target** authentication endpoint declared
in the target's manifest; the registry is NOT the token issuer.
Federated registries follow the same authentication pattern as any
other agent or service.

The complete scope set, TTL bounds, refresh semantics, and the
relationship to the §5.1.8.1 authentication baseline are defined in
the **7. searchAndInvoke Telemetry and Authentication** companion
document.

##### §5.1.8.6 Nonrepudiation

Every SADAR invocation generates an attributed, time-limited usage
record. No invocation is anonymous. The persistent audit artifact
for each invocation is the Telemetry Record (§5.1.14). The
SCT chain provides the cryptographically attested context history;
the Telemetry Record provides the per-invocation ground-truth audit
fact. Where a value appears in both, parity is required per
§5.1.17.

The Telemetry Record schema and SCT chain root jti correlation are
defined in the **6. Telemetry Record and Repatriation** companion
document. The SCT chain audit semantics are defined in the
**4. SCT Operations** companion document.

---

#### §5.1.9 Controlled First-Use Authorization

The normative mechanics for establishing new server relationships
between previously unknown parties:

- Standard protocol for first-use interactions, including acceptance
  of existing API keys or other licensing credentials
- Credential exchange format and sequence
- Licensing acceptance protocol and record format
- Payment authorization workflow for commercial capabilities
- Normative option for either party to require human review before
  automated provisioning proceeds
- First-use attribution record format — establishes an auditable
  authorization record for all subsequent interactions
- Normative format for exposing compliance documentation and proof
  of compliance, including authenticated access for sensitive
  documents

---

#### §5.1.10 Usage Payment

The normative manifest elements supporting direct consumer-to-server
payment settlement:

- Manifest fields for cost model, pricing structure, payment terms,
  and payment methods, defined within the Financial NFR category
- Public merchant code and payment endpoint declarations in the
  manifest, enabling direct consumer-to-server payment after
  discovery
- Normative requirement that the registry is not involved in payment
  settlement — payment occurs directly between parties
- Registries and registry entries may define usage charges in their
  manifests; these are subject to the same NFR matching as other
  manifest dimensions

The complete Financial NFR vocabulary is defined in the
**3. NFR Schema** companion document.

---

#### §5.1.11 Registry Topology, Federation, and Directory of Authorized Registries

The normative structure of the SADAR registry network and the
governance layer that controls federation participation:

- Defined registry roles and types: Provider, Marketplace, Industry,
  Community, Internal, and Registry of Registries
- **Directory of Authorized Registries** — the normative governance
  construct that controls which registries may participate in public
  federation; structure, registration protocol, and query interface
  are normatively defined
- **Federation access control** — conformant registries MUST reject
  unauthorized forwarding and replication requests from registries
  not listed in the Directory of Authorized Registries; this is a
  mandatory protection for federation integrity and is not subject to
  implementer discretion
- Between authorized registries, forwarding and replication requests
  may be declined only if the request violates the receiving
  registry's declared NFRs
- Private registries — normative support for registries operating
  internally without participation in the public Directory, whether
  standalone or internally federated
- Inter-registry communication and replication message formats
- **Provenance preservation across replication** — every Entity,
  Entry, and Registry record carries a `home_registry_id`
  identifying its registry of origin. Conformant registries SHALL
  preserve `home_registry_id` unchanged across all forwarding,
  replication, and re-publication operations. Replicated records
  SHALL be discoverable by `home_registry_id` such that consumers
  may distinguish records native to a queried registry from those
  replicated into it.
- Query forwarding protocol — registries may forward queries to other
  authorized registries within their terms of service
- TTL semantics governing replication scope (`replication_seconds`)
  and session scope (`session_seconds`); TTL expiry triggers
  rediscovery, which may be a full query or direct re-query to the
  prior selected service
- Registry-of-Registries resolution protocol
- Registry manifests — registries follow the same manifest model as
  other top-level records and are registered and managed accordingly;
  registries MUST declare NFRs for operational controls, licensing,
  and billing in their manifests
- Federation authentication — authentication to any federated
  registry follows the §5.1.8.1 authentication baseline; there is no
  separate federation token system. The federated registry's manifest
  declares its authentication endpoint per the standard pattern
- Forwarding and replication access controls — local registry
  administrators configure all inbound and outbound forwarding and
  replication settings
- Terms of service — replication and discovery forwarding requests
  are subject to the same licensing, billing, and rate-limiting
  controls as direct usage
- Discovery result caching — registries and agentic frameworks may
  cache discovery results; the discovery TTL governs cache validity;
  time-of-use tokens enforce reauthentication

---

#### §5.1.12 Registry Isolation

The normative constraints on registry behavior establishing it as a
discovery-time component only:

- The registry facilitates the exchange of signed manifests; after
  discovery, consumer and server communicate directly
- The registry MUST NOT have access to usage tokens, credentials,
  API keys, or sensitive operational data
- The registry MUST NOT issue authentication tokens for invocations;
  authentication tokens are issued by the target's authentication
  endpoint per §5.1.8.1
- Runtime interactions MUST NOT be dependent on registry availability
  once discovery is complete, except at discovery TTL expiry
- No sensitive information is held by the registry; it is not part
  of the runtime process beyond discovery

---

#### §5.1.13 SADAR Discovery Tool Interface

The SADAR specification normatively defines the discovery tool
interface through which conformant implementations expose SADAR
discovery capabilities to requesting agents. This interface has
three normative layers.

*Layer 1 — Tool Discovery*
Conformant implementations MUST register and advertise the SADAR
discovery tool in a normatively defined manner for each supported
delivery mechanism. The specification defines, per delivery
mechanism (MCP, A2A, LangChain, direct integration, and others as
normatively defined):

- The required tool registration location and format
- The canonical tool identifier and required metadata that make
  the tool discoverable across frameworks
- Required capability declarations for the tool itself as a
  registry entry, subject to the same manifest and signing
  requirements as any other registry entry

*Layer 2 — Resolver Contract*
The SADAR discovery tool accepts a consumer-supplied resolver that
performs final selection from the candidate set returned by
bilateral matching. The specification normatively defines:

- How the resolver is declared and located by a conformant
  implementation — including registration format and location
  requirements
- **Input contract** — the required structure and content of what
  the tool passes to the resolver, including the candidate set
  format with FULL_MATCH/PARTIAL_MATCH classification per §5.1.3
- **Output contract** — what the resolver MUST return; the resolver
  MUST return exactly one selection from the provided candidate
  set or a defined error; the resolver MUST NOT modify, augment, or
  substitute candidates outside the set returned by bilateral
  matching
- **Behavioral contract** — the resolver operates on the candidate
  set only; it MUST NOT perform independent registry queries or
  bypass bilateral matching results; selection logic is implementer
  discretion within these constraints

*Layer 3 — Error Contract*
The specification normatively defines the complete error surface of
the discovery tool interface, using the `urn:sadar:error:v1:*`
namespace. The error contract includes:

- Defined error conditions, their codes, and required semantics
- Required error response structure and required fields
- Terminal vs. retriable error classification
- Error propagation requirements to the calling agent

*Implementation Surface — Helper API and Authentication*
A SADAR-conformant `searchAndInvoke` implementation provides a
Telemetry Helper API as the application-facing boundary for
constructing the per-invocation Telemetry Record (§5.1.14),
and it authenticates to invoked targets per §5.1.8.1 with scopes
from the `urn:sadar:scope:v1:*` namespace. SAI also performs
telemetry repatriation upon each agent/tool/resource call return
when bilateral matching at discovery indicated repatriation is
applicable. The complete Helper API surface, repatriation trigger
specification, and scope set are defined in the **7. searchAndInvoke
Telemetry and Authentication** companion document. Implementation
patterns for the resolver and SAI internals — including strategy
patterns, handler architectures, and validation approaches — are
non-normative and are described in the SADAR Reference Architecture
companion document.

---

#### §5.1.14 Observability Requirements

SADAR normatively requires the use of OpenTelemetry (OTel) or an
OpenTelemetry-compatible alternative for traceability within
conformant implementations, consistent with the precedent
established by the A2A specification. The normative requirements
are:

- Conformant implementations MUST instrument using OpenTelemetry
- The process identifier MUST be carried in OTel baggage
- The risk score adjustment list MUST be carried in OTel baggage
  per §5.1.18
- W3C Trace Context propagation is required for any invocation
  participating in cross-trust-boundary repatriation per
  §5.1.14.X.10; W3C Baggage propagation is required across all trust
  boundaries
- Span and Trace IDs MUST be generated and propagated for
  attribution and nonrepudiation
- Every SADAR-conformant span SHALL carry the
  `telemetry.origin.environment` attribute identifying the issuing
  entity using the structured environment identifier format defined
  in the Telemetry Record and Repatriation companion document
- Implementations SHALL access the per-invocation Telemetry Record
  only through a SADAR-defined Helper API per §5.1.13

How implementations consume, process, route, store, monitor, or
act upon telemetry data beyond these normative requirements is left
to implementer discretion and is outside the scope of this
specification.

##### §5.1.14.X SADAR Telemetry Record and Repatriation

A normative subsection of §5.1.14 covering the per-invocation
Telemetry Record and the governed cross-trust-boundary repatriation
mechanism.

The Telemetry Record is the canonical per-invocation persistent
audit artifact. Every invocation of `searchAndInvoke` produces a
structured Telemetry Record. The record's required fields cover
search metadata, resolver metadata, selection metadata, invocation
metadata, and outcome metadata (including risk score state). The
Telemetry Record is persisted to an implementer-chosen storage
backend at finalize.

Telemetry Repatriation is the asynchronous, governed return of
trace fragments from a server to the requester at trust boundary
crossings. Repatriation is bilaterally declared in manifests via
four Protocol NFRs (`repatriation_participation`,
`repatriation_endpoint`, `repatriation_redacted_fields`,
`repatriation_required_unredacted_fields`) and is matched at
discovery time. Where applicable, repatriation occurs at OTel span
close per the searchAndInvoke Telemetry and Authentication
companion document. Repatriated telemetry preserves a normative set
of non-redactable structural fields and is governed by the
server's declared disclosure policy through field-level
redaction. The Compliance Levels mechanism of earlier drafts is
superseded by field-list bilateral matching.

The complete Telemetry Record schema, OTel span structure,
repatriation endpoint requirements, redaction semantics,
non-redactable structural field set, and replay protection are
defined in the **6. Telemetry Record and Repatriation** companion
document.

---

#### §5.1.15 Authorization Metadata

The normative requirements for authorization information that must
be present and conveyed at invocation time:

- The trust model under which the invocation operates is conveyed
  through the `originating_user_trust` claim of the SCT per
  §5.1.8.3
- For asserted-trust-model invocations, the SCT carries asserter
  identity, originator identity (namespaced under asserter), and a
  Role claim drawn from the server's declared `supported_roles`
- Server validation of authorization metadata is fail-closed per
  §5.1.8.4; the structured error namespace is
  `urn:sadar:error:v1:trust_model:*`
- Authorization policy enforcement — how a server acts on
  validated authorization metadata to permit or deny specific
  operations — is server-internal and outside SADAR's normative
  scope per §5.2.6

The complete authorization metadata schema, claim semantics, and
server-side validation are defined in the **5. Trust Models** and
**4. SCT Operations** companion documents.

---

#### §5.1.16 Semantic Contracts and Identifiers

The normative use of IRIs and URNs for capability grounding,
versioning semantics, and compatibility declarations.

SADAR defines an IRI namespace pattern for spec-defined identifiers:

| Namespace | Scope |
|-----------|-------|
| `urn:sadar:nfr:v{version}:{category}:{attribute}` | NFR field identifiers |
| `urn:sadar:compliance:v{version}:{framework}` | Compliance framework identifiers |
| `urn:sadar:error:v{version}:{category}:{code}` | Error code identifiers |
| `urn:sadar:scope:v{version}:{category}:{operation}` | OIDC scope identifiers |
| `urn:sadar:risk_reason:v{version}:{category}` | Risk score reason identifiers |
| `urn:sadar:baggage:v{version}:{key}` | OTel baggage key identifiers |

When referring to an entire namespace (e.g., the set of all error
codes), the convention is `urn:sadar:{category}:v{version}:*` with a
trailing `*`. When referring to a specific identifier, the asterisk
is omitted: `urn:sadar:scope:v1:invocation:invoke` is a specific
scope; `urn:sadar:scope:v1:*` refers to the SADAR-defined scope
namespace as a whole.

All SADAR-defined IRIs include a version segment. Custom or
implementation-specific identifiers SHALL NOT use the `urn:sadar:`
prefix; ecosystem participants extending the SADAR vocabulary use
their own organization-controlled namespaces.

The SADAR specification for identifying references within external
standards (NAICS, APQC PCF, HL7, X12, ISO 20022, and others) is
defined in the SADAR Specification corpus, with usage examples in
the NFR Schema companion document.

---

#### §5.1.17 Cryptographic Parity

A normative requirement that values appearing in multiple
propagation layers (OTel baggage, OTel span attributes, SCT claims,
Telemetry Record fields) SHALL be consistent at any point of
observation. Parity instances include:

- Process identifier across baggage, SCT claim, and
  Telemetry Record
- SCT chain root jti across baggage and Telemetry Record
- Trust model claim across SCT and discovery-time negotiation
  result
- Risk score scalar across Baggage-derived computation and SCT
  chain link claim
- Transaction Instance ID across SCT, OTel trace, and Telemetry
  Record

Parity violations are reported with structured error in the
`urn:sadar:error:v1:*` namespace. Cryptographic parity is the
foundation for tamper-evident audit reconstruction across trust
boundaries.

The complete parity vocabulary, instance-by-instance specification,
and verification rules are defined in the SADAR Specification
corpus, with the foundational requirement preserved in this
Section.

---

#### §5.1.18 Risk Score

A normative substrate for risk-aware run-time control across SADAR-
conformant agent flows. The risk score is a stateful, cumulative
property of an in-flight process flow — a real number in `[0.0, 1.0]`
that begins at zero and is adjusted up or down by participants in
the flow as risk-relevant events occur.

- Adjustments are emitted as structured objects carrying `delta`,
  `ttl_seconds`, `timestamp`, optional `decay`, `reason` (IRI from
  the `urn:sadar:risk_reason:v1:*` namespace or custom), and
  `context` (free-form text)
- Adjustments propagate as a list in OTel Baggage under the key
  `urn:sadar:baggage:v1:risk_adjustments`; the computed accumulated
  scalar propagates in the SCT chain at each link
- The accumulation algorithm is implementer concern; the
  specification does not mandate a specific algorithm, recognizing
  that different domains warrant different risk sensitivity
- Conformant implementations SHALL accept and propagate Adjustments
  without dropping, modifying, or reordering them; SHALL preserve
  unknown reason IRIs; SHALL reject malformed Adjustments
- Conformant implementations SHOULD apply an accumulation algorithm
  and produce non-trivial scalars; not doing so is conformant but
  degenerate (always reports score = 0.0)
- The aggregate score is floor-clamped at 0.0 and ceiling-clamped
  at 1.0; intermediate algorithm values may exceed these bounds but
  reported aggregates do not
- The Telemetry Record carries the score and the Adjustment list at
  finalize for audit per §5.1.14
- The optional Protocol NFR `urn:sadar:nfr:v1:protocol:impact_score`
  permits servers to declare static potential-severity-of-misuse on
  their entries, distinct from the in-flight Risk Score

The complete Risk Score data model, propagation rules, error codes,
and impact_score semantics are defined in the **8. Risk Score
Specification** companion document.

---

#### §5.1.19 Conformance

The normative definition of conformance to this Specification,
including:

- Conformance levels and associated required behaviors
- Conformance assertion structure and required content
- Requirements for self-declaration or third-party attestation

---

### 5.2 Out of Scope (Excluded from IP Licensing Commitments)

The following are explicitly excluded from the IP licensing
commitments described in Section 5.0. No Necessary Claims arising
solely from these elements are licensed under the Community
Specification License 1.0.

---

#### §5.2.1 Runtime Selection and Decision Logic

SADAR defines the protocol by which a conformant registry reflects
a candidate set of entries in response to a discovery query. The
specification ends at that reflection. Runtime validation, ranking,
filtering, and final selection from the candidate set is the sole
responsibility of the requesting agent's organization. Any
implementation of such selection or enforcement logic — beyond the
normative resolver input, output, and behavioral contracts defined
in §5.1.13 — is out of scope.

---

#### §5.2.2 Proprietary Registry Extensions

A conformant SADAR registry implementation may support capabilities
beyond those normatively required by this specification, provided
that:

- All normative SADAR manifest fields, schema, and serialization
  requirements are satisfied without modification
- All normative SADAR query interface behaviors, request formats,
  and response formats are preserved
- Any proprietary extension to the manifest schema or query
  interface does not alter the result set that would be returned by
  a fully-conformant SADAR registry processing the same query
  against the same normative manifest fields

These constraints are necessary to preserve interoperability and
federation consistency. In a federated topology, queries may be
resolved locally, forwarded to a peer registry, or satisfied from a
replicated cache. A result set MUST NOT vary based on which registry
in the topology processes the query. Proprietary extensions that
introduce additional selection criteria, modify manifest discovery
fields, or alter query semantics in ways that affect federation
behavior are not permitted under a conformance claim.

The IP licensing grant extends only to those elements required for
SADAR conformance. Proprietary extensions to internal implementation
— including storage, indexing, administrative tooling, and
operational management — are out of scope provided they do not
affect the normative query or manifest behavior described above.

---

#### §5.2.3 User Interface and Tooling

Any user interface, developer tooling, administrative console, or
management surface built on top of a SADAR-conformant registry is
out of scope for purposes of the IP licensing grant. The design,
layout, and implementation approach of such surfaces are left to
implementer discretion.

However, any interface that provides human or programmatic access
to a SADAR-conformant registry MUST implement and enforce the
normative SADAR security model, including:

- Authentication and identity requirements as defined in §5.1.8
- Authorization metadata requirements as defined in §5.1.15
- All access control requirements normatively specified in the
  SADAR Specification

An interface that exposes registry functions without enforcing the
normative security model does not qualify as part of a conformant
SADAR implementation, regardless of whether the underlying registry
is otherwise conformant. Security is a foundational requirement
that cannot be satisfied at the registry layer alone and bypassed
at the interface layer.

---

#### §5.2.4 Agent Orchestration and Workflow

A system that invokes agents discovered via SADAR must satisfy the
following normative requirements to qualify as part of a conformant
SADAR implementation:

- **Observability** — OTel instrumentation as defined in §5.1.14,
  including the process identifier, the risk score
  Adjustment list, all required baggage fields, and provenance
  attribution on every span
- **Discovery Tool** — use of a SADAR-conformant discovery tool
  implementation as normatively defined in §5.1.13
- **Resolver** — provision of a resolver satisfying the normative
  input, output, and behavioral contracts defined in §5.1.13
- **Telemetry Record Helper API** — application access to the
  Telemetry Record only through the SADAR-defined Helper API per
  §5.1.13
- **Risk Score Propagation** — accept-and-propagate behavior on
  Risk Score Adjustments per §5.1.18

The IP licensing grant extends to these normative requirements.

All other aspects of agent orchestration, workflow execution,
process coordination, fan-out/fan-in patterns, and execution
substrate are out of scope. The grant does not extend to a consuming
system merely because it interacts with a SADAR-conformant registry,
nor to the internal implementation of any orchestration mechanism
that satisfies the normative requirements above.

---

#### §5.2.5 Observability Implementation

The IP licensing grant extends to the normative OTel requirements
defined in §5.1.14, including the Telemetry Record schema,
span structure with provenance attribution, and Telemetry
Repatriation. How an implementation consumes, processes, routes,
stores, monitors, or acts upon that telemetry data beyond those
normative requirements is out of scope.

---

#### §5.2.6 Authorization Policy Implementation

The IP licensing grant extends to the normative authorization
metadata requirements defined in §5.1.15, including SCT
chain validation per §5.1.8.4. How an implementation
evaluates, enforces, or acts upon validated authorization metadata
— including any RBAC, ABAC, zero-trust, or other policy model
applied to the invocation — is out of scope.

---

#### §5.2.7 Payment Settlement Implementation

The IP licensing grant extends to the normative payment manifest
fields defined in §5.1.10. The mechanics of actual payment
settlement between consumer and server — including payment
processing systems, financial institution integrations, and
settlement networks — are out of scope. The registry is not in the
payment path.

---

#### §5.2.8 Risk Score Accumulation and Intervention Logic

The IP licensing grant extends to the normative Risk Score
Adjustment data model, propagation rules, and accept-and-propagate
behavior defined in §5.1.18. The accumulation algorithm by
which a list of Adjustments produces a current scalar, the
intervention thresholds at which an implementation acts on
accumulated risk, and the specific mitigation techniques an agent
or guardrail applies are all out of scope.

---

#### §5.2.9 Transport, Persistence, and Execution

*Registry Implementation:*
The SADAR specification normatively defines transport and persistence
*requirements* for conformant registry implementations, including:

- **Transport** — required protocol support, message format
  requirements, and communication security requirements for registry
  interactions
- **Persistence** — durability, consistency, and integrity
  requirements for stored manifests and registry state

The IP licensing grant extends to these normative requirements. The
*implementation* of these requirements — including choice of
transport stack, database engine, storage architecture, or
persistence technology — is left to implementer discretion and is
out of scope. A conformant registry may satisfy persistence
requirements using any appropriate technology, whether relational,
document, vector, or graph store, provided all normative requirements
are met.

*Agent Runtime and Execution:*
Execution runtimes, model architectures, inference substrates, and
all transport and persistence concerns of the consuming agent system
are out of scope in their entirety. The IP licensing grant does not
extend to the agent runtime merely because it interacts with a
SADAR-conformant registry.

---

## 6. Scope Interpretation

A patent claim is within Scope only if:

> It is necessarily infringed by implementing the normative
> requirements of this Specification, and no alternative
> implementation exists that avoids such infringement.

Claims covering specific implementations, optimizations, or optional
approaches — where alternative implementations exist that satisfy
the same normative requirement — are not considered Necessary Claims
and are excluded from the IP licensing grant.

---

## 7. Conformance

Conformance is a technical determination that an implementation
satisfies the normative requirements of the SADAR specification.
Conformance is evaluated against the normative elements defined in
§5.1 in their entirety. Partial conformance — satisfying some but
not all normative requirements — does not constitute conformance.

SADAR defines the following conformance tiers:

- **Self-Declared Conformance** — the implementer asserts that their
  implementation satisfies all normative SADAR requirements.
  Self-declaration does not imply review or attestation by
  OpenSemantics.org or any delegated body. Self-declared conformance
  carries no authorization to use the SADAR certification mark.

- **Assessed Conformance** — conformance has been evaluated against
  the SADAR conformance test suite by OpenSemantics.org or a
  delegated conformance body. Assessed conformance is a prerequisite
  for certification.

Conformance criteria, test suites, and validation rules are defined
in the SADAR Conformance Specification companion document.

Certification of conformant implementations and authorization to use
the SADAR mark are governed by the OpenSemantics.org Charter and
OpenSemantics.org Trademark Policy companion documents respectively.
The IP licensing grants of the Community Specification License 1.0
apply to all conformant implementations regardless of certification
or mark authorization status.

---

## 8. Changes to Scope

**The Scope is currently in *draft* form as a work-in-progress.  Until it is released in its initial, published, form, it shall not be considered binding and will be subject to revision.**

Once the initial, non-draft version is published, further changes to this Scope shall not apply retroactively. Contributors' IP
licensing commitments apply only to the Scope in effect at the time
of their Contribution.

---

## 9. Terminology

Key terms used in this document — including Entity, Entry, Agent,
Tool, Resource, Process, Registry, Requester, Server, SCT, Helper
API, Risk Score Adjustment, Necessary Claims, Directory of
Authorized Registries, and Non-Functional Requirement (NFR) — are
defined in the SADAR Specification Terminology section.

---

## 10. Evolution

SADAR will evolve through community contributions and real-world
implementation feedback. The specification repository and
`Notices.md` serve as the authoritative record of contributor
copyright ownership and patent licensing commitments.

---

*© 2026 Cognita AI Inc. All rights reserved.*
*SADAR™ is a trademark of Cognita AI Inc. stewarded through
OpenSemantics.org. Use of the SADAR™ name or certification mark is
governed by the OpenSemantics.org Trademark Policy.*
*This specification is licensed under the Community Specification
License 1.0.*
