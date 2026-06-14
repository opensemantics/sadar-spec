**SADAR**

Semantic Agent Discovery and Attribution Runtime

Discovery Standards

|                   |                                     |
| ----------------- | ----------------------------------- |
| **Version**       | 0.10 DRAFT                          |
| **Date**          | June 2026                           |
| **Status**        | Draft for Public Comment            |
| **Maintained by** | OpenSemantics.org                   |
| **License**       | Community Specification License 1.0 |

# Table of Contents

- [1. Introduction](#1-introduction)
  - [1.1 Design Principles](#11-design-principles)
  - [1.2 Relationship to Other Standards](#12-relationship-to-other-standards)
- [2. Core Concepts](#2-core-concepts)
  - [2.1 The SADAR Manifest](#21-the-sadar-manifest)
    - [2.1.1 Manifest Identity Fields](#211-manifest-identity-fields)
    - [2.1.2 Manifest Semantic Fields](#212-manifest-semantic-fields)
    - [2.1.3 Manifest Invocation Contract Fields](#213-manifest-invocation-contract-fields)
  - [2.2 Deterministic Candidate Generation](#22-deterministic-candidate-generation)
  - [2.3 Capability and Data References](#23-capability-and-data-references)
  - [2.4 Enforcement and Match Semantics](#24-enforcement-and-match-semantics)
- [3. Registry Protocol](#3-registry-protocol)
  - [3.1 Transport Requirements](#31-transport-requirements)
  - [3.2 Agent Authentication to the Registry](#32-agent-authentication-to-the-registry)
  - [3.3 Search Request](#33-search-request)
    - [3.3.1 Search Request Body](#331-search-request-body)
    - [3.3.2 Search Response](#332-search-response)
    - [3.3.3 Match Metadata](#333-match-metadata)
- [4. The Resolver Contract](#4-the-resolver-contract)
  - [4.1 Resolver Input](#41-resolver-input)
  - [4.2 Resolver Output](#42-resolver-output)
- [5. Invocation Requirements](#5-invocation-requirements)
  - [5.1 Identity Requirements](#51-identity-requirements)
    - [5.1.1 Agent Identity](#511-agent-identity)
    - [5.1.2 Originator Identity and the SADAR Context Token](#512-originator-identity-and-the-sadar-context-token)
    - [5.1.3 Transactional Context](#513-transactional-context)
  - [5.2 Transport Requirements for Invocation](#52-transport-requirements-for-invocation)
  - [5.3 Response Handling Requirements](#53-response-handling-requirements)
- [6. Registry Architecture](#6-registry-architecture)
  - [6.1 Registry Identity](#61-registry-identity)
  - [6.2 Federation](#62-federation)
  - [6.3 Directory of Authorized Registries](#63-directory-of-authorized-registries)
- [7. Conformance, Certification, and Authorization](#7-conformance-certification-and-authorization)
  - [7.1 Functional Conformance Tiers](#71-functional-conformance-tiers)
  - [7.2 The Governance Ladder](#72-the-governance-ladder)
  - [7.3 Implementation Latitude Summary](#73-implementation-latitude-summary)
- [8. Security Considerations](#8-security-considerations)
  - [8.1 mTLS as the Trust Foundation](#81-mtls-as-the-trust-foundation)
  - [8.2 Credential Completeness](#82-credential-completeness)
  - [8.3 Manifest Integrity](#83-manifest-integrity)
  - [8.4 Business Process Scope Enforcement](#84-business-process-scope-enforcement)
  - [8.5 Determinism as a Federation Trust Property](#85-determinism-as-a-federation-trust-property)
- [Appendix A: Normative References](#appendix-a-normative-references)
- [Appendix B: Example Manifest](#appendix-b-example-manifest)
- [Appendix C: Example Search and Candidate Response](#appendix-c-example-search-and-candidate-response)

# 1. Introduction

SADAR — the Semantic Agent Discovery and Attribution Runtime — is an open standard that defines how autonomous AI agents discover callable services and are authorized to invoke them. SADAR provides a uniform protocol for registering services with framework-anchored semantic metadata, searching that registry deterministically, and executing invocations with verifiable identity propagation.

SADAR is intentionally narrow in scope. It specifies the interfaces, credential structures, transport requirements, and behavioral contracts that ensure interoperability across implementations. It does not specify how registries physically store or index manifests, how a caller selects among returned candidates, or how invocations are physically executed. These are implementation concerns deliberately left to the implementer.

A central property of SADAR is that discovery is **deterministic**: the production of the candidate list is a pure function of its inputs. SADAR specifies the rules by which candidates are derived; it does not rank them and does not choose among them. Selection is delegated to a caller-supplied resolver (Section 4).

|                |                                                                                                                                                                                                                                                                                                        |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **SCOPE NOTE** | This document defines the SADAR standard. It normatively specifies what compliant implementations MUST, SHOULD, and MAY do. It does not describe any particular implementation. The key words MUST, MUST NOT, REQUIRED, SHALL, SHOULD, RECOMMENDED, MAY, and OPTIONAL are used as defined in RFC 2119. |

## 1.1 Design Principles

- **Semantic grounding**: services are described by what they do, expressed as references into published, versioned frameworks — not by free text and not by how they are called.
- **Deterministic discovery**: given identical inputs, every conformant registry produces an identical candidate list with identical match metadata.
- **Identity completeness**: every invocation carries the full three-part identity (agent, originator, context).
- **Transport security**: mutual TLS is mandatory for all SADAR protocol interactions.
- **Implementation freedom**: SADAR specifies contracts and the matching result, not internal data structures or selection strategy.
- **Auditability**: the standard enables complete chain-of-custody without mandating specific audit mechanisms.

## 1.2 Relationship to Other Standards

SADAR builds on and references the following standards:

- RFC 6749 / RFC 9068 — OAuth 2.0 and JWT-based authorization
- RFC 7523 — JWT Profile for OAuth 2.0 Client Authentication
- RFC 8705 — OAuth 2.0 Mutual TLS Client Authentication
- RFC 7515 / RFC 7516 — JSON Web Signature (JWS) and JSON Web Encryption (JWE)
- OpenTelemetry — context propagation and trace correlation
- OpenAPI 3.x — service interface description
- JSON Schema Draft 2020-12 — data contract validation

SADAR references, but does not redefine, the following companion SADAR specifications:

- **Governance and Conformance** — defines the conformance, certification, and authorization processes and the role of the Authorizing Body.
- **Federation Establishment and Policy** — defines bilateral federation agreements and forwarding policy.
- **Replication and Manifest Provenance** — defines replication relationships and provenance chains.
- **SADAR Context Token (SCT)** — defines the JWS-inside-JWE token that carries originator authorization context across trust boundaries.

Capability and data frameworks (for example ASC X12, the APQC Process Classification Framework, FHIR, and ISO 20022) are referenced by manifests but are published and versioned by their respective authorities, not by SADAR.

# 2. Core Concepts

## 2.1 The SADAR Manifest

The SADAR manifest is the central artifact of the standard. It is the semantic description of a callable component — what it does, what data it accepts and returns, the non-functional properties it asserts, and the constraints under which it may be invoked. The manifest is registered in a SADAR registry and is the basis for both discovery and invocation.

A manifest is immutable once registered. Any change to its content — including signing method, authentication endpoint, capabilities, or non-functional properties — produces a new version with a new signature. The registry retains all versions with their lifecycle state (ACTIVE, DEPRECATED, REVOKED). Manifest immutability is a precondition of deterministic discovery (Section 2.2): the registry's contents must be a stable input to the candidate-generation function.

The canonical referent of a component is the composition **Publisher + Component + Version**. Human-readable labels (display names, descriptions) are ephemeral and are never used for matching.

### 2.1.1 Manifest Identity Fields

|                    |              |                                                                                                            |
| ------------------ | ------------ | ---------------------------------------------------------------------------------------------------------- |
| **Field**          | **Required** | **Description**                                                                                            |
| publisher\_iri     | **REQUIRED** | Globally unique IRI identifying the owning entity (publisher). Each component is owned by exactly one entity. |
| component\_name    | **REQUIRED** | Component name within the publisher's namespace. Alphanumeric with underscores and dashes; no spaces.      |
| version            | **REQUIRED** | Semantic version string (e.g., 2.1.0). Monotonically increasing within a (publisher\_iri, component\_name). |
| entry\_uuid        | **REQUIRED** | Immutable UUID for this manifest version. Generated at registration time.                                  |
| registered\_at     | **REQUIRED** | ISO 8601 UTC timestamp of registration.                                                                    |
| status             | **REQUIRED** | One of: ACTIVE, DEPRECATED, REVOKED.                                                                        |
| soft\_ttl\_seconds | **REQUIRED** | Duration after which clients SHOULD revalidate this manifest version.                                      |
| hard\_ttl\_seconds | **REQUIRED** | Duration after which clients MUST NOT use this manifest version without revalidation.                      |

The triple (publisher\_iri, component\_name, version) is the immutable component referent. The registry MUST treat it as unique.

### 2.1.2 Manifest Semantic Fields

A component exposes exactly one **Provider interface** and MAY expose one or more **Requester interfaces**. The Provider interface is how the component is discovered and invoked. Requester interfaces declare the candidate providers the component may seek; they are informational inputs to a planner or an enforcement layer and are not gated by the registry at registration time.

|                      |              |                                                                                                                                              |
| -------------------- | ------------ | -------------------------------------------------------------------------------------------------------------------------------------------- |
| **Field**            | **Required** | **Description**                                                                                                                              |
| display\_name        | **REQUIRED** | Human-readable component name. Not used for matching.                                                                                        |
| description          | OPTIONAL     | Natural-language description for human consumption and search UIs. Not used for matching.                                                    |
| domain               | **REQUIRED** | Business domain classification (e.g., healthcare.claims, finance.payment).                                                                   |
| data\_classification | **REQUIRED** | Sensitivity level of data this component handles. Implementer-defined enumeration; SADAR requires the field exist.                           |
| provider\_interface  | **REQUIRED** | The single Provider interface. See below.                                                                                                    |
| requester\_interfaces | OPTIONAL    | Array of zero or more Requester interfaces. See below.                                                                                       |

**Provider interface** object:

|                |              |                                                                                                                                                                       |
| -------------- | ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Field**      | **Required** | **Description**                                                                                                                                                        |
| capabilities   | **REQUIRED** | Array of capability references (Section 2.3) the component advertises. On a Provider interface, capabilities are advertised; functional Mandatory does not apply.       |
| nfrs           | OPTIONAL     | Array of non-functional property assertions (Section 2.4). A provider MAY mark an NFR Mandatory, meaning a requester MUST satisfy it (e.g., accepted payment methods). |
| data\_inputs   | OPTIONAL     | Array of data references (Section 2.3) the component consumes. A provider MAY mark a data input Mandatory, meaning the requester MUST supply it.                       |
| data\_outputs  | OPTIONAL     | Array of data references the component produces.                                                                                                                       |
| operations     | **REQUIRED** | Array of operation descriptors. Each operation has a name, description, input\_schema, and output\_schema (JSON Schema 2020-12) describing the wire contract.          |

Operation `input_schema` / `output_schema` describe the wire payload for validation. The `data_inputs` / `data_outputs` references describe the same data semantically (framework-anchored) for matching purposes. The two are complementary: JSON Schema validates the payload; data references make the data dimension matchable and reportable.

**Requester interface** object:

|                   |              |                                                                                                                                   |
| ----------------- | ------------ | --------------------------------------------------------------------------------------------------------------------------------- |
| **Field**         | **Required** | **Description**                                                                                                                    |
| interface\_name   | OPTIONAL     | Human-readable label. Not used for matching.                                                                                       |
| sought\_capabilities | **REQUIRED** | Array of capability references the requester may seek, each with an enforcement value (Section 2.4) and a version expression.   |
| nfr\_constraints  | OPTIONAL     | Array of NFR constraints the requester places on candidate providers, each with an enforcement value.                             |
| data\_constraints | OPTIONAL     | Array of data references the requester requires of, or expects to receive from, a candidate provider, each with enforcement.      |

### 2.1.3 Manifest Invocation Contract Fields

SADAR does not specify invocation mechanics beyond the fields below. How a client physically calls the component is an implementation concern.

|                  |              |                                                                                                            |
| ---------------- | ------------ | ---------------------------------------------------------------------------------------------------------- |
| **Field**        | **Required** | **Description**                                                                                            |
| endpoint\_uri    | **REQUIRED** | The URI at which this component is callable. MUST be an HTTPS URI.                                         |
| protocol         | **REQUIRED** | Transport protocol. SADAR-defined enumeration: REST, GRPC, GRAPHQL, WEBSOCKET, MESSAGE\_QUEUE, CUSTOM.     |
| auth\_schemes    | **REQUIRED** | Array of OIDC client-credential flows accepted by this component. At minimum one MUST be present.          |
| sla.timeout\_ms  | **REQUIRED** | Maximum milliseconds the caller SHOULD wait before treating the invocation as failed.                     |
| sla.max\_retries | **REQUIRED** | Maximum retry attempts before treating the invocation as permanently failed.                              |
| policy\_tags     | OPTIONAL     | Opaque string tags consumed by the implementer's policy engine. SADAR does not define their semantics.    |

## 2.2 Deterministic Candidate Generation

A conformant registry MUST generate the candidate list deterministically. Given identical search inputs, identical registry contents, and identical forwarding rules, a conformant registry MUST return an identical candidate list, with identical match metadata, on every evaluation.

Determinism is quantified over the inputs, not over outputs across registries. Two registries with different contents will return different candidate lists; this is correct and is not a violation. What is guaranteed is that the candidate-generation *function* is identical across all conformant implementations. The function's inputs are:

1. the search request (Section 3.3.1),
2. the registry contents (the set of registered, in-scope manifest versions), and
3. the forwarding rules in effect for the request (Section 6).

The registry MUST NOT introduce nondeterminism into candidate generation — for example, through ranking, sampling, time-of-day behavior, or load-dependent truncation. Selection among candidates is the resolver's responsibility (Section 4) and is explicitly nondeterministic by permission.

|          |                                                                                                                                                                                                            |
| -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **NOTE** | Determinism is what certification attests (Section 7.2). Because only certified registries may be authorized to federate, every node in a federation provably generates candidates by the same function. |

## 2.3 Capability and Data References

Capabilities and data definitions are both expressed as references into a published, versioned framework. A reference is resolved to a canonical IRI at evaluation time by composition; the composed IRI is never stored, so it cannot drift from its parts.

A reference consists of:

|                       |              |                                                                                                                  |
| --------------------- | ------------ | ---------------------------------------------------------------------------------------------------------------- |
| **Field**             | **Required** | **Description**                                                                                                  |
| authority\_iri        | **REQUIRED** | Base IRI of the framework authority, including the framework variant and version (e.g., `https://x12.org/5010.x`). |
| version\_expression   | **REQUIRED** | The version segment as declared. MAY be concrete (`5010.003`), partially wildcarded (`5010.x`), or fully wildcarded (`x`). |
| framework\_local\_id  | **REQUIRED** | The native key of the term within the framework (e.g., the X12 transaction-set code `850`, or an APQC element ID). |

The canonical IRI is `authority_iri` + `framework_local_id`, with the version carried inside `authority_iri`. The natural key of a definition is the triple (authority, version, framework\_local\_id); the registry MUST treat it as unique so that two references to the same term resolve to the same identity.

**Capabilities** answer "what do I do" and are functional. For frameworks whose functional unit is atomic (e.g., an X12 transaction set), capabilities are flat. **Data definitions** answer "what flows in and out" and MAY be hierarchical (e.g., transaction → segment → element); hierarchy is expressed by a parent reference, and a node's local id is its own segment/element code, with structural position reconstructed by walking parents. Display labels on either are not used for matching.

## 2.4 Enforcement and Match Semantics

Every requester reference carries an **enforcement** value: `MANDATORY`, `PREFERRED`, or `OPTIONAL`. Enforcement is interpreted by direction.

**Requester enforcement** applies across all dimensions (functional capability, NFR, data):

- `MANDATORY` — the constraint is a gate. A candidate that does not satisfy it is not a candidate.
- `PREFERRED` — the constraint does not gate. Whether it is satisfied is reported in match metadata for the resolver to use; it does not affect candidacy.
- `OPTIONAL` — informational only.

**Provider enforcement** applies only to NFRs and data inputs, never to functional capabilities. A provider advertises its capabilities; it asserts no functional gate on requesters. A provider `MANDATORY` NFR or data input is a gate the requester MUST satisfy (e.g., "you must match my accepted payment methods," "you must supply this field").

The composite gate for pairing a requester against a Provider interface is therefore:

1. every requester-Mandatory functional capability is present in the provider's advertised capabilities (matched via downward subsumption — a provider's more-specific capability satisfies a requester's more-general one; the reverse does not), **and**
2. every requester-Mandatory NFR and data constraint is satisfied by the provider, **and**
3. every provider-Mandatory NFR and data input is satisfied by the requester.

Version matching is enforcement-governed subsumption on the version segment. A wildcard subsumes more specific versions (`x` subsumes everything; `5010.x` subsumes any `5010` release). Two version expressions match if the sets they denote intersect. A requester-Mandatory concrete version pins the gate to that version regardless of provider permissiveness.

Structured NFR predicates the registry MUST evaluate deterministically include exact match, within-range (e.g., cost between x and y), and one-of / set-intersection (e.g., acceptable regions, hosting models, TLS versions). For any constraint that admits a range of valid answers, the registry MUST report the candidate's actual value in match metadata (Section 3.3.3), including for a Mandatory comparison gate that a candidate passes with margin (e.g., a `≤10ms` requirement met at `8ms`). Non-discriminating exact-equality matches are not individually reported.

# 3. Registry Protocol

## 3.1 Transport Requirements

All communication between SADAR clients and registries MUST use mutual TLS (mTLS) as defined in RFC 8705. Specifically:

- The registry MUST present a TLS certificate signed by a recognized CA or federation trust anchor.
- The client MUST present a TLS client certificate identifying the agent.
- Certificate validation MUST be performed bidirectionally. Connections where either party cannot validate the other's certificate MUST be rejected.
- TLS 1.3 is REQUIRED. TLS 1.2 MAY be accepted for legacy compatibility until December 31, 2027, after which it MUST be rejected.

## 3.2 Agent Authentication to the Registry

Before executing a search, the calling agent MUST authenticate to the registry using OIDC Client Credentials flow (RFC 6749 Section 4.4) over the established mTLS channel.

The access token obtained MUST be a JWT conforming to RFC 9068 and MUST contain the following claims:

|                  |              |                                                                                                    |
| ---------------- | ------------ | -------------------------------------------------------------------------------------------------- |
| **Field**        | **Required** | **Description**                                                                                    |
| sub              | **REQUIRED** | The agent's stable identifier. Used by the registry to scope the search.                           |
| iss              | **REQUIRED** | The token issuer URI. MUST be reachable for JWKS validation.                                       |
| aud              | **REQUIRED** | MUST include the registry's entity IRI.                                                            |
| scope            | **REQUIRED** | MUST include sadar:search. MAY include sadar:register for manifest registration.                   |
| exp              | **REQUIRED** | Expiry time. The registry MUST reject tokens where exp is in the past.                             |
| sadar\_agent\_id | **REQUIRED** | The SADAR-registered agent identifier. This is the lookup key for the agent's capability manifest. |

The registry uses sadar\_agent\_id to retrieve the agent's capability manifest, which defines the scope of components this agent is permitted to discover.

|          |                                                                                                                                                                                                                                                                            |
| -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **NOTE** | The registry does not validate business-level authorization. It returns the candidates matching the search criteria, the agent's registered discovery scope, and the deterministic match rules. Business-level enforcement is the responsibility of the layer invoked after discovery. |

## 3.3 Search Request

A conforming SADAR search request MUST be submitted as an HTTP POST to the registry's search endpoint. The Authorization header MUST carry the Bearer token obtained in Section 3.2.

### 3.3.1 Search Request Body

The search request expresses requester criteria as structured, enforcement-tagged references. Matching is performed against these structured criteria, not against natural-language text.

|                           |              |                                                                                                                                                       |
| ------------------------- | ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Field**                 | **Required** | **Description**                                                                                                                                       |
| sought\_capabilities      | **REQUIRED** | Array of capability references (Section 2.3), each with an enforcement value and a version expression. At least one MUST be present.                  |
| nfr\_constraints          | OPTIONAL     | Array of NFR constraints (within-range, one-of, exact, set), each with an enforcement value.                                                          |
| data\_constraints         | OPTIONAL     | Array of data references the requester requires or expects, each with an enforcement value.                                                           |
| domain\_filter            | OPTIONAL     | Restrict results to components within this domain.                                                                                                    |
| data\_classification\_max | OPTIONAL     | Exclude components whose data\_classification exceeds this level.                                                                                     |
| max\_results              | OPTIONAL     | Maximum candidate count to return. Default: 10. Maximum: 50. Truncation, if applied, MUST be deterministic and ordered by the component referent.     |
| business\_process\_id     | OPTIONAL     | If the search occurs within a declared business process context, this IRI identifies that process. Registries MAY use it to scope results.            |
| intent\_hint              | OPTIONAL     | Non-normative natural-language description of intent, for logging or client-side translation. MUST NOT affect candidacy or match metadata.            |

### 3.3.2 Search Response

A successful search response MUST return HTTP 200. The response MUST NOT include any registry-assigned ranking or relevance score. The candidate list is a set; ordering, if present, carries no normative meaning and MUST NOT be relied upon by the resolver.

```
{
  "request_id": "",
  "query_echo": { ... echo of structured criteria ... },
  "registry_iri": "urn:sadar:registry:org:name",
  "candidates": [
    {
      "publisher_iri": "urn:sadar:acme",
      "component_name": "eligibility-check",
      "version": "3.0.1",
      "entry_uuid": "",
      "display_name": "...",
      "matched_capabilities": [ "https://x12.org/5010.x/270" ],
      "operations": [ ... ],
      "endpoint_uri": "https://...",
      "protocol": "REST",
      "auth_schemes": [ ... ],
      "sla": { "timeout_ms": 3000, "max_retries": 2 },
      "soft_ttl_seconds": 3600,
      "hard_ttl_seconds": 86400,
      "source_registry": "urn:sadar:registry:org:name",
      "match_metadata": [ ... see 3.3.3 ... ]
    }
  ]
}
```

Error responses MUST use standard HTTP status codes with a JSON error body containing error\_code, error\_description, and request\_id fields.

### 3.3.3 Match Metadata

Each candidate MUST carry match metadata reporting, for every dimension on which candidates can validly differ, the constraint as asked, the candidate's actual value, and the disposition. Dimensions that all candidates satisfy identically (non-discriminating exact-equality gates) MUST NOT be reported.

Each match-metadata entry has:

|                |              |                                                                                                                          |
| -------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------ |
| **Field**      | **Required** | **Description**                                                                                                          |
| dimension      | **REQUIRED** | One of: CAPABILITY, NFR, DATA.                                                                                           |
| reference      | **REQUIRED** | The capability/NFR/data reference the entry concerns.                                                                   |
| constraint     | **REQUIRED** | The requester or provider constraint as declared (e.g., version expression, range, one-of set).                        |
| actual\_value  | **REQUIRED** | The candidate's actual value for this dimension.                                                                        |
| disposition    | **REQUIRED** | One of: WITHIN\_RANGE, ONE\_OF\_SELECTED, SUBSUMPTION\_MATCH, MANDATORY\_MARGIN, PREFERRED\_MET, PREFERRED\_UNMET.       |

Match metadata MUST be deterministic: identical inputs produce identical metadata. The resolver MAY use these annotations under any policy; the registry attaches no weight to them.

# 4. The Resolver Contract

After receiving search candidates, the searchAndInvoke implementation MUST execute a resolver to select the component to invoke. The resolver is a caller-supplied component. SADAR specifies the contract the resolver must honor; it does not specify resolver logic. SADAR does not rank candidates and expresses no preference among them.

## 4.1 Resolver Input

The implementation MUST provide the resolver with:

- the complete candidate list returned by the registry, including all manifest fields, and
- the per-candidate **match metadata** (Section 3.3.3), and
- the original structured search criteria, and
- any additional context the implementation deems relevant.

## 4.2 Resolver Output

The resolver MUST return exactly one candidate from the input list, identified by its (publisher\_iri, component\_name, version) referent and entry\_uuid. Returning a component not present in the input list is a protocol violation.

|               |                                                                                                                                                                                                                                                   |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **RATIONALE** | The resolver may use any method — first match, random choice, LLM ranking, rule-based selection, weighting of match metadata, user confirmation, or any combination. SADAR does not constrain this because selection strategy is a domain-specific concern, and because the candidate list is already guaranteed to contain only valid candidates. |

# 5. Invocation Requirements

Once a component is selected by the resolver, the implementation MUST invoke it. SADAR specifies the credential propagation and identity requirements for invocation. It does not specify the transport, transformation, or routing mechanism — these are implementation concerns.

## 5.1 Identity Requirements

Every SADAR-compliant invocation MUST carry a three-part identity structure.

### 5.1.1 Agent Identity

The calling agent MUST authenticate to the target component using OIDC Client Credentials (RFC 6749 Section 4.4) with an access token containing at minimum:

- sub: the agent's stable identifier
- sadar\_agent\_id: the SADAR-registered agent identifier
- aud: MUST include the target component's referent
- scope: MUST include at minimum the operation being invoked

### 5.1.2 Originator Identity and the SADAR Context Token

The identity and authorization context of the human or system that initiated the workflow MUST be propagated to the target component. This context MUST be carried as a **SADAR Context Token (SCT)** — a JWS-inside-JWE token as defined in the SADAR Context Token specification — transported in the X-SADAR-Context HTTP header. The SCT MUST be validated by the implementation before invocation. Its signed payload MUST contain at minimum:

- sub: the originator's stable identifier
- iss: the originator's identity provider
- exp: token expiry — MUST NOT be in the past at time of invocation
- auth\_time: the time the originator originally authenticated

|                 |                                                                                                                                                                                                                                  |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **REQUIREMENT** | Implementations MUST NOT allow originator context to be substituted, fabricated, or omitted. An invocation without a valid SADAR Context Token is a protocol violation regardless of the validity of the agent credentials. |

### 5.1.3 Transactional Context

The implementation MUST propagate an OpenTelemetry trace context per the W3C Trace Context specification (traceparent / tracestate headers). This binds the invocation to the observable execution chain.

If the invocation occurs within a declared business process, the business process manifest identifier MUST be carried as OpenTelemetry baggage:

- Key: sadar.business\_process\_id — IRI of the business process manifest
- Key: sadar.business\_process\_version — version of the business process manifest

Implementations MAY add additional baggage keys using the sadar. prefix namespace. Implementations MUST NOT strip baggage keys with the sadar. prefix received from upstream callers.

## 5.2 Transport Requirements for Invocation

- Invocations MUST be made over HTTPS.
- Implementations SHOULD use mTLS for invocation transport. If mTLS is not available for the target component, the implementation MUST log this condition.
- Implementations MUST respect the sla.timeout\_ms and sla.max\_retries values from the selected manifest.

## 5.3 Response Handling Requirements

- Implementations MUST validate that the response conforms to the selected operation's output\_schema as declared in the manifest.
- Implementations MUST NOT pass a schema-invalid response to the calling agent without logging the violation.
- Implementations MUST propagate the OTel trace context on the response path.

# 6. Registry Architecture

## 6.1 Registry Identity

A SADAR registry MUST be identified by a stable IRI in the form urn:sadar:registry:{org}:{name}. This IRI MUST be included in all registry responses.

## 6.2 Federation

SADAR supports federated registry deployments. A compliant registry MAY forward search queries to peer registries and aggregate results, subject to the governance constraints of Section 7.2 and the Federation Establishment and Policy specification. SADAR requires that:

- Only registries that are certified and authorized (Section 7.2) participate in public federation.
- Federation is established by bilateral agreement; forwarding rules are an explicit input to candidate generation (Section 2.2) and MUST be well-defined and stable for a given request.
- A federated search composes deterministically: given the same request, the same union of in-scope registry contents, and the same forwarding topology, the aggregated candidate list MUST be identical.
- Results returned to the caller MUST indicate which registry sourced each candidate (source\_registry).
- mTLS MUST be used for registry-to-registry communication.
- Deduplication MUST be performed on the component referent (publisher\_iri, component\_name, version) together with entry\_uuid before results are returned to the caller.

## 6.3 Directory of Authorized Registries

The Directory of Authorized Registries (the Registry of Registries, RoR) is the canonical record of registries authorized for public federation. It is the authorization mechanism and the source of truth for federation topology.

- The Directory publishes the set of authorized registries, their IRIs, their trust certificates, and their replication relationships.
- Listing in the Directory constitutes authorization (Section 7.2). A registry MUST hold current certification to be listed.
- Because the Directory defines the forwarding topology, it is the source of truth for the forwarding-rules input over which federated determinism is quantified.

A private registry MAY consume content from authorized registries but MUST NOT serve as the home registry for public-federation content and MUST NOT be listed in the Directory. Deauthorization (removal from the Directory) and certification revocation are decoupled acts; either may occur without the other.

# 7. Conformance, Certification, and Authorization

## 7.1 Functional Conformance Tiers

SADAR defines three functional conformance tiers describing which sections an implementation realizes:

|                      |                                                                                                                                                  |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| **SADAR Core**       | Implements Sections 3 (Registry Protocol), 4 (Resolver Contract), and 5 (Invocation Requirements). Mandatory for all conforming implementations. |
| **SADAR Registry**   | Implements Section 6 (Registry Architecture) in addition to Core, including deterministic candidate generation (Section 2.2). Required for any system acting as a SADAR registry. |
| **SADAR Federation** | Implements Sections 6.2 and 6.3 in addition to Registry. Required for registries participating in federated topologies.                          |

## 7.2 The Governance Ladder

Functional conformance is necessary but not sufficient for public federation. SADAR defines a three-step governance ladder, specified in detail in the Governance and Conformance specification:

1. **Conformance** — functional compliance with this standard, including deterministic candidate generation.
2. **Certification** — attestation by an Authorizing Body that an implementation conforms, specifically including verification that it generates candidates deterministically per Section 2.2.
3. **Authorization** — listing of the registry in the Directory of Authorized Registries (Section 6.3), which is a prerequisite for participating in public federation. Authorization requires current certification.

Because certification verifies deterministic candidate generation, and because authorization to federate requires certification, every registry in a public federation provably evaluates the same candidate-generation function. Federated determinism is therefore a property produced by the governance structure, not an unverified runtime assumption. Certification revocation and deauthorization are decoupled, as stated in Section 6.3.

## 7.3 Implementation Latitude Summary

The following aspects of a compliant SADAR implementation are explicitly left to the implementer:

- Internal storage and indexing structures used to evaluate matches (the match *result* is normative; the data structures are not)
- Policy decision logic and policy engine integration
- Resolver selection strategy
- Invocation execution mechanism (direct HTTP, message queue, integration engine, etc.)
- Data transformation between the caller's format and the target component's format
- Audit logging format, storage, and retention
- Error recovery and circuit breaking strategies
- Business process enforcement semantics

Deterministic candidate generation (Section 2.2) is **not** within implementation latitude; it is normatively required and is the subject of certification.

# 8. Security Considerations

## 8.1 mTLS as the Trust Foundation

mTLS is mandatory rather than recommended because it provides bidirectional authentication at the transport layer, independent of application-layer credential validity. This prevents registry misdirection attacks where a client is induced to authenticate to a fraudulent registry endpoint. Even if an attacker intercepts DNS or routing, they cannot present a valid server certificate for the genuine registry IRI without access to the registry's private key.

## 8.2 Credential Completeness

The three-part identity requirement (Section 5.1) exists to prevent agents from acting beyond their authorized scope by substituting their own identity for the originator's. Implementations MUST validate all three identity components independently. Failure to validate any one component MUST result in invocation rejection, not degraded-mode execution.

## 8.3 Manifest Integrity

A manifest is signed by its publisher and is immutable per version (Section 2.1). Implementations SHOULD verify manifest integrity using the publisher's signature before using manifest data to configure invocations; the key and signing methodology are recorded with the manifest for reverification. A compromised manifest could redirect invocations to attacker-controlled endpoints. Implementations MUST respect hard\_ttl\_seconds and MUST NOT use manifests beyond their hard TTL without re-fetching from the registry. Because immutable signed content is the stable input to candidate generation, manifest integrity also underpins deterministic discovery.

## 8.4 Business Process Scope Enforcement

When business\_process\_id is carried in OTel baggage, implementations SHOULD verify that the invoked component is authorized within the declared business process scope. This is not mandated by the standard because enforcement mechanisms are implementation-specific, but it is strongly recommended as a defense against prompt-injection attacks that attempt to cause agents to invoke out-of-scope components.

## 8.5 Determinism as a Federation Trust Property

Deterministic candidate generation is a security property as well as a correctness property. Because a candidate list is reproducible from declared inputs, federated results are auditable and a misbehaving registry is detectable: the same inputs that produced a given list can be replayed to confirm it. Certification (Section 7.2) converts this property into a federation trust anchor — authorized registries are those an Authorizing Body has attested behave deterministically — so a caller can rely on federated discovery without trusting each peer individually.

# Appendix A: Normative References

- RFC 2119 — Key words for use in RFCs to Indicate Requirement Levels
- RFC 6749 — The OAuth 2.0 Authorization Framework
- RFC 7515 — JSON Web Signature (JWS)
- RFC 7516 — JSON Web Encryption (JWE)
- RFC 7517 — JSON Web Key (JWK)
- RFC 7518 — JSON Web Algorithms (JWA)
- RFC 7523 — JWT Profile for OAuth 2.0 Client Authentication and Authorization Grants
- RFC 8705 — OAuth 2.0 Mutual-TLS Client Authentication
- RFC 9068 — JSON Web Token (JWT) Profile for OAuth 2.0 Access Tokens
- W3C Trace Context — Distributed Tracing specification
- W3C Baggage — OpenTelemetry Baggage specification
- JSON Schema 2020-12
- OpenAPI Specification 3.1
- SADAR Governance and Conformance specification
- SADAR Federation Establishment and Policy specification
- SADAR Replication and Manifest Provenance specification
- SADAR Context Token (SCT) specification

# Appendix B: Example Manifest

```
{
  "publisher_iri": "urn:sadar:acme",
  "component_name": "eligibility-check",
  "version": "3.0.1",
  "entry_uuid": "f3a2c1d4-8b9e-4f0a-b6c7-1d2e3f4a5b6c",
  "status": "ACTIVE",
  "registered_at": "2026-01-15T14:30:00Z",
  "soft_ttl_seconds": 3600,
  "hard_ttl_seconds": 86400,
  "display_name": "Member Eligibility Check",
  "description": "Verifies a member's insurance eligibility and returns coverage details.",
  "domain": "healthcare.claims",
  "data_classification": "PHI",
  "provider_interface": {
    "capabilities": [
      {
        "authority_iri": "https://x12.org/5010.x",
        "version_expression": "5010.x",
        "framework_local_id": "270"
      },
      {
        "authority_iri": "https://x12.org/5010.x",
        "version_expression": "5010.x",
        "framework_local_id": "271"
      }
    ],
    "nfrs": [
      { "reference": "nfr:latency_ms", "predicate": "max", "value": 3000, "enforcement": "PREFERRED" },
      { "reference": "nfr:region", "predicate": "one_of", "value": ["us-east", "us-west"], "enforcement": "MANDATORY" }
    ],
    "data_inputs": [
      {
        "authority_iri": "https://x12.org/5010.x",
        "version_expression": "5010.x",
        "framework_local_id": "270/NM1/09",
        "label": "MemberID",
        "enforcement": "MANDATORY"
      }
    ],
    "data_outputs": [
      {
        "authority_iri": "https://x12.org/5010.x",
        "version_expression": "5010.x",
        "framework_local_id": "271/EB",
        "label": "EligibilityBenefit"
      }
    ],
    "operations": [
      {
        "name": "check_eligibility",
        "description": "Check eligibility for a member by ID and date of service",
        "input_schema": { "$ref": "schemas/eligibility-request.json" },
        "output_schema": { "$ref": "schemas/eligibility-response.json" }
      }
    ]
  },
  "endpoint_uri": "https://eligibility.claims.acme.internal/v3",
  "protocol": "REST",
  "auth_schemes": ["oidc_client_credentials"],
  "sla": { "timeout_ms": 3000, "max_retries": 2 },
  "policy_tags": ["requires_phi_handling", "hipaa_minimum_necessary"]
}
```

# Appendix C: Example Search and Candidate Response

**Search request body:**

```
{
  "sought_capabilities": [
    {
      "authority_iri": "https://x12.org/5010.x",
      "version_expression": "5010.x",
      "framework_local_id": "270",
      "enforcement": "MANDATORY"
    }
  ],
  "nfr_constraints": [
    { "reference": "nfr:latency_ms", "predicate": "max", "value": 10, "enforcement": "MANDATORY" },
    { "reference": "nfr:region", "predicate": "one_of", "value": ["us-east"], "enforcement": "PREFERRED" }
  ],
  "domain_filter": "healthcare.claims",
  "max_results": 10,
  "intent_hint": "find a member eligibility inquiry service"
}
```

**Candidate (excerpt) with match metadata:**

```
{
  "publisher_iri": "urn:sadar:acme",
  "component_name": "eligibility-check",
  "version": "3.0.1",
  "matched_capabilities": ["https://x12.org/5010.x/270"],
  "source_registry": "urn:sadar:registry:acme:east",
  "match_metadata": [
    {
      "dimension": "NFR",
      "reference": "nfr:latency_ms",
      "constraint": "max 10",
      "actual_value": 8,
      "disposition": "MANDATORY_MARGIN"
    },
    {
      "dimension": "NFR",
      "reference": "nfr:region",
      "constraint": "one_of [us-east]",
      "actual_value": "us-east",
      "disposition": "PREFERRED_MET"
    }
  ]
}
```

The Mandatory capability (X12 270) is a non-discriminating exact gate and is not individually reported; the latency gate is reported with its margin (passed at 8ms against a ≤10ms requirement), and the preferred region is reported as met. The resolver decides what, if anything, these annotations are worth.

End of SADAR Discovery Standard | OpenSemantics.org | Community Specification License 1.0
