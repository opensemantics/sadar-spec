**SADAR**

Semantic Agent Discovery and Attribution Registry

Discovery Standards

|  |  |
| --- | --- |
| **Version** | 0.9 DRAFT |
| **Date** | April 2026 |
| **Status** | Draft for Public Comment |
| **Maintained by** | OpenSemantics.org |
| **License** | Community Specification License 1.0 |

# Table of Contents

[Table of Contents 1](#_Toc226214082)

[1. Introduction 1](#_Toc226214083)

[1.1 Design Principles 1](#_Toc226214084)

[1.2 Relationship to Other Standards 1](#_Toc226214085)

[2. Core Concepts 1](#_Toc226214086)

[2.1 The SADAR Manifest 1](#_Toc226214087)

[2.1.1 Manifest Identity Fields 1](#_Toc226214088)

[2.1.2 Manifest Semantic Fields 1](#_Toc226214089)

[2.1.3 Manifest Invocation Contract Fields 1](#_Toc226214090)

[3. Registry Protocol 1](#_Toc226214091)

[3.1 Transport Requirements 1](#_Toc226214092)

[3.2 Agent Authentication to the Registry 1](#_Toc226214093)

[3.3 Search Request 1](#_Toc226214094)

[3.3.1 Search Request Body 1](#_Toc226214095)

[3.3.2 Search Response 1](#_Toc226214096)

[4. The Resolver Contract 1](#_Toc226214097)

[4.1 Resolver Input 1](#_Toc226214098)

[4.2 Resolver Output 1](#_Toc226214099)

[5. Invocation Requirements 1](#_Toc226214100)

[5.1 Identity Requirements 1](#_Toc226214101)

[5.1.1 Agent Identity 1](#_Toc226214102)

[5.1.2 Originator Identity 1](#_Toc226214103)

[5.1.3 Transactional Context 1](#_Toc226214104)

[5.2 Transport Requirements for Invocation 1](#_Toc226214105)

[5.3 Response Handling Requirements 1](#_Toc226214106)

[6. Registry Architecture 1](#_Toc226214107)

[6.1 Registry Identity 1](#_Toc226214108)

[6.2 Federation 1](#_Toc226214109)

[6.3 Registry of Registries 1](#_Toc226214110)

[7. Conformance 1](#_Toc226214111)

[7.1 Conformance Levels 1](#_Toc226214112)

[7.2 Implementation Latitude Summary 1](#_Toc226214113)

[8. Security Considerations 1](#_Toc226214114)

[8.1 mTLS as the Trust Foundation 1](#_Toc226214115)

[8.2 Credential Completeness 1](#_Toc226214116)

[8.3 Manifest Integrity 1](#_Toc226214117)

[8.4 Business Process Scope Enforcement 1](#_Toc226214118)

[Appendix A: Normative References 1](#_Toc226214119)

[Appendix B: Example Manifest 1](#_Toc226214120)

# 1. Introduction

SADAR — the Semantic Agent Discovery and Authorization Registry — is an open standard that defines how autonomous AI agents discover callable services and are authorized to invoke them. SADAR provides a uniform protocol for registering services with semantic metadata, searching that registry, and executing invocations with verifiable identity propagation.

SADAR is intentionally narrow in scope. It specifies the interfaces, credential structures, transport requirements, and behavioral contracts that ensure interoperability across implementations. It does not specify how registries store or index manifests, how policy decisions are made, or how invocations are physically executed. These are implementation concerns deliberately left to the implementer.

|  |  |
| --- | --- |
| **SCOPE NOTE** | This document defines the SADAR standard. It normatively specifies what compliant implementations MUST, SHOULD, and MAY do. It does not describe any particular implementation. The key words MUST, MUST NOT, REQUIRED, SHALL, SHOULD, RECOMMENDED, MAY, and OPTIONAL are used as defined in RFC 2119. |

## 1.1 Design Principles

* Semantic grounding: services are described by what they do, not how they are called
* Identity completeness: every invocation carries the full three-part identity (agent, originator, context)
* Transport security: mutual TLS is mandatory for all SADAR protocol interactions
* Implementation freedom: SADAR specifies contracts, not implementations
* Auditability: the standard enables complete chain-of-custody without mandating specific audit mechanisms

## 1.2 Relationship to Other Standards

SADAR builds on and references the following standards:

* RFC 6749 / RFC 9068 — OAuth 2.0 and JWT-based authorization
* RFC 8705 — OAuth 2.0 Mutual TLS Client Authentication
* OpenTelemetry — context propagation and trace correlation
* OpenAPI 3.x — service interface description
* JSON Schema Draft 2020-12 — data contract validation
* W3C DID — decentralized identifier syntax for service IRIs

# 2. Core Concepts

## 2.1 The SADAR Manifest

The SADAR manifest is the central artifact of the standard. It is the semantic description of a callable service — what it does, what data it accepts and returns, and the policy constraints under which it may be invoked. The manifest is registered in the SADAR registry and is the basis for both search and authorization.

A manifest is immutable once registered. Updates create new versioned manifests. The registry retains all versions with their lifecycle state (ACTIVE, DEPRECATED, REVOKED).

### 2.1.1 Manifest Identity Fields

|  |  |  |
| --- | --- | --- |
| **Field** | **Required** | **Description** |
| entity\_iri | **REQUIRED** | Globally unique IRI identifying the service entity. Format: urn:sadar:{org}:{domain}:{name} |
| entry\_uuid | **REQUIRED** | Immutable UUID for this manifest version. Generated at registration time. |
| version | **REQUIRED** | Semantic version string (e.g., 2.1.0). Monotonically increasing within an entity\_iri. |
| registered\_at | **REQUIRED** | ISO 8601 UTC timestamp of registration. |
| status | **REQUIRED** | One of: ACTIVE, DEPRECATED, REVOKED |
| soft\_ttl\_seconds | **REQUIRED** | Duration after which clients SHOULD revalidate this manifest version. |
| hard\_ttl\_seconds | **REQUIRED** | Duration after which clients MUST NOT use this manifest version without revalidation. |

### 2.1.2 Manifest Semantic Fields

|  |  |  |
| --- | --- | --- |
| **Field** | **Required** | **Description** |
| display\_name | **REQUIRED** | Human-readable service name. |
| description | **REQUIRED** | Natural language description of the service's purpose and capabilities. This field is the primary input to semantic search. |
| capability\_tags | **REQUIRED** | Array of canonical capability identifiers. RECOMMENDED source: O\*NET-SOC task taxonomy or implementer-defined controlled vocabulary. |
| domain | **REQUIRED** | Business domain classification (e.g., healthcare.claims, finance.payment). |
| operations | **REQUIRED** | Array of operation descriptors. Each operation has a name, description, input\_schema, and output\_schema (JSON Schema). |
| data\_classification | **REQUIRED** | Sensitivity level of data this service handles. Implementer-defined enumeration; SADAR requires this field exist. |

### 2.1.3 Manifest Invocation Contract Fields

SADAR does not specify invocation mechanics beyond the fields below. How a client physically calls the service is an implementation concern.

|  |  |  |
| --- | --- | --- |
| **Field** | **Required** | **Description** |
| endpoint\_uri | **REQUIRED** | The URI at which this service is callable. MUST be an HTTPS URI. |
| protocol | **REQUIRED** | Transport protocol. SADAR-defined enumeration: REST, GRPC, GRAPHQL, WEBSOCKET, MESSAGE\_QUEUE, CUSTOM. |
| auth\_schemes | **REQUIRED** | Array of OIDC client credential flows accepted by this service. At minimum one MUST be present. |
| sla.timeout\_ms | **REQUIRED** | Maximum milliseconds the caller SHOULD wait before treating the invocation as failed. |
| sla.max\_retries | **REQUIRED** | Maximum retry attempts before treating the invocation as permanently failed. |
| policy\_tags | OPTIONAL | Opaque string tags consumed by the implementer's policy engine. SADAR does not define their semantics. |

# 3. Registry Protocol

## 3.1 Transport Requirements

All communication between SADAR clients and registries MUST use mutual TLS (mTLS) as defined in RFC 8705. Specifically:

* The registry MUST present a TLS certificate signed by a recognized CA or federation trust anchor.
* The client MUST present a TLS client certificate identifying the agent.
* Certificate validation MUST be performed bidirectionally. Connections where either party cannot validate the other's certificate MUST be rejected.
* TLS 1.3 is REQUIRED. TLS 1.2 MAY be accepted for legacy compatibility until December 31, 2027, after which it MUST be rejected.

## 3.2 Agent Authentication to the Registry

Before executing a search, the calling agent MUST authenticate to the registry using OIDC Client Credentials flow (RFC 6749 Section 4.4) over the established mTLS channel.

The access token obtained MUST be a JWT conforming to RFC 9068 and MUST contain the following claims:

|  |  |  |
| --- | --- | --- |
| **Field** | **Required** | **Description** |
| sub | **REQUIRED** | The agent's stable identifier. Used by the registry to scope the search. |
| iss | **REQUIRED** | The token issuer URI. MUST be reachable for JWKS validation. |
| aud | **REQUIRED** | MUST include the registry's entity IRI. |
| scope | **REQUIRED** | MUST include sadar:search. MAY include sadar:register for manifest registration. |
| exp | **REQUIRED** | Expiry time. The registry MUST reject tokens where exp is in the past. |
| sadar\_agent\_id | **REQUIRED** | The SADAR-registered agent identifier. This is the lookup key for the agent's capability manifest. |

The registry uses sadar\_agent\_id to retrieve the agent's capability manifest, which defines the scope of services this agent is permitted to discover.

|  |  |
| --- | --- |
| **NOTE** | The registry does not validate business-level authorization. It returns the candidates matching both the search criteria and the agent's registered discovery scope. Business-level enforcement is the responsibility of the implementation layer invoked after discovery. |

## 3.3 Search Request

A conforming SADAR search request MUST be submitted as an HTTP POST to the registry's search endpoint. The Authorization header MUST carry the Bearer token obtained in Section 3.2.

### 3.3.1 Search Request Body

|  |  |  |
| --- | --- | --- |
| **Field** | **Required** | **Description** |
| query | **REQUIRED** | Natural language description of the capability being sought. The registry uses this for semantic matching against manifest description and capability\_tags fields. |
| domain\_filter | OPTIONAL | Restrict results to manifests within this domain. |
| operation\_filter | OPTIONAL | Restrict results to manifests offering this named operation. |
| data\_classification\_max | OPTIONAL | Exclude manifests whose data\_classification exceeds this level. |
| max\_results | OPTIONAL | Maximum candidate count to return. Default: 10. Maximum: 50. |
| business\_process\_id | OPTIONAL | If the search occurs within a declared business process context, this IRI identifies that process. Registries MAY use this to scope results to services authorized for that process. |

### 3.3.2 Search Response

A successful search response MUST return HTTP 200 with the following body structure:

{

"request\_id": "<uuid>",

"query\_echo": "<original query string>",

"candidates": [

{

"entity\_iri": "urn:sadar:org:domain:service",

"entry\_uuid": "<uuid>",

"version": "2.1.0",

"display\_name": "...",

"description": "...",

"relevance\_score": 0.92,

"operations": [ ... ],

"endpoint\_uri": "https://...",

"protocol": "REST",

"auth\_schemes": [ ... ],

"sla": { "timeout\_ms": 5000, "max\_retries": 3 },

"soft\_ttl\_seconds": 3600,

"hard\_ttl\_seconds": 86400

}

]

}

Error responses MUST use standard HTTP status codes with a JSON error body containing error\_code, error\_description, and request\_id fields.

# 4. The Resolver Contract

After receiving search candidates, the searchAndInvoke implementation MUST execute a resolver to select the service to invoke. The resolver is a caller-supplied component. SADAR specifies the contract the resolver must honor; it does not specify resolver logic.

## 4.1 Resolver Input

The implementation MUST provide the resolver with the complete candidate list returned by the registry, including all manifest fields. The implementation MUST also provide the original search query and any context it deems relevant.

## 4.2 Resolver Output

The resolver MUST return exactly one candidate from the input list, identified by its entry\_uuid. Returning a service not present in the input list is a protocol violation.

|  |  |
| --- | --- |
| **RATIONALE** | The resolver may use any logic — LLM ranking, rule-based selection, user confirmation, or a combination. SADAR does not constrain this because selection strategy is a domain-specific concern. Constraining it would limit legitimate use cases. |

# 5. Invocation Requirements

Once a service is selected by the resolver, the implementation MUST invoke it. SADAR specifies the credential propagation and identity requirements for invocation. It does not specify the transport, transformation, or routing mechanism — these are implementation concerns.

## 5.1 Identity Requirements

Every SADAR-compliant invocation MUST carry a three-part identity structure:

### 5.1.1 Agent Identity

The calling agent MUST authenticate to the target service using OIDC Client Credentials (RFC 6749 Section 4.4) with an access token containing at minimum:

* sub: the agent's stable identifier
* sadar\_agent\_id: the SADAR-registered agent identifier
* aud: MUST include the target service's entity\_iri
* scope: MUST include at minimum the operation being invoked

### 5.1.2 Originator Identity

The identity of the human or system that initiated the workflow MUST be propagated to the target service. The originator identity MUST be carried as a JWT in the X-SADAR-Originator HTTP header. This token MUST be validated by the implementation before invocation and MUST contain:

* sub: the originator's stable identifier
* iss: the originator's identity provider
* exp: token expiry — MUST not be in the past at time of invocation
* auth\_time: the time the originator originally authenticated

|  |  |
| --- | --- |
| **REQUIREMENT** | Implementations MUST NOT allow originator credentials to be substituted, fabricated, or omitted. An invocation without valid originator credentials is a protocol violation regardless of the validity of the agent credentials. |

### 5.1.3 Transactional Context

The implementation MUST propagate an OpenTelemetry trace context per the W3C Trace Context specification (traceparent / tracestate headers). This binds the invocation to the observable execution chain.

If the invocation occurs within a declared business process, the business process manifest identifier MUST be carried as OpenTelemetry baggage:

* Key: sadar.business\_process\_id — IRI of the business process manifest
* Key: sadar.business\_process\_version — version of the business process manifest

Implementations MAY add additional baggage keys using the sadar. prefix namespace. Implementations MUST NOT strip baggage keys with the sadar. prefix received from upstream callers.

## 5.2 Transport Requirements for Invocation

* Invocations MUST be made over HTTPS.
* Implementations SHOULD use mTLS for invocation transport. If mTLS is not available for the target service, the implementation MUST log this condition.
* Implementations MUST respect the sla.timeout\_ms and sla.max\_retries values from the selected manifest.

## 5.3 Response Handling Requirements

* Implementations MUST validate that the response conforms to the selected operation's output\_schema as declared in the manifest.
* Implementations MUST NOT pass a schema-invalid response to the calling agent without logging the violation.
* Implementations MUST propagate the OTel trace context on the response path.

# 6. Registry Architecture

## 6.1 Registry Identity

A SADAR registry MUST be identified by a stable IRI in the form urn:sadar:registry:{org}:{name}. This IRI MUST be included in all registry responses.

## 6.2 Federation

SADAR supports federated registry deployments. A compliant registry MAY forward search queries to peer registries and aggregate results. Federation topology is implementation-defined. SADAR requires only that:

* Results returned to the caller indicate which registry sourced each candidate.
* mTLS is used for registry-to-registry communication.
* Deduplication is performed on (entity\_iri, entry\_uuid) pairs before results are returned to the caller.

## 6.3 Registry of Registries

Implementations MAY designate a Registry of Registries (RoR) as the authoritative trust anchor for federation topology. The RoR publishes the set of known registries, their IRIs, their trust certificates, and their replication relationships. SADAR does not mandate RoR deployment but recommends it for enterprise-scale deployments.

# 7. Conformance

## 7.1 Conformance Levels

SADAR defines three conformance levels:

|  |  |
| --- | --- |
| **SADAR Core** | Implements Sections 3 (Registry Protocol), 4 (Resolver Contract), and 5 (Invocation Requirements). Mandatory for all conforming implementations. |
| **SADAR Registry** | Implements Section 6 (Registry Architecture) in addition to Core. Required for any system acting as a SADAR registry. |
| **SADAR Federation** | Implements Section 6.2 and 6.3 in addition to Registry. Required for registries participating in federated topologies. |

## 7.2 Implementation Latitude Summary

The following aspects of a compliant SADAR implementation are explicitly left to the implementer:

* Registry storage and indexing mechanisms
* Semantic search algorithm (vector, keyword, hybrid, or other)
* Policy decision logic and policy engine integration
* Invocation execution mechanism (direct HTTP, message queue, integration engine, etc.)
* Data transformation between the caller's format and the target service's format
* Audit logging format, storage, and retention
* Error recovery and circuit breaking strategies
* Business process enforcement semantics

# 8. Security Considerations

## 8.1 mTLS as the Trust Foundation

mTLS is mandatory rather than recommended because it provides bidirectional authentication at the transport layer, independent of application-layer credential validity. This prevents registry misdirection attacks where a client is induced to authenticate to a fraudulent registry endpoint. Even if an attacker intercepts DNS or routing, they cannot present a valid server certificate for the genuine registry IRI without access to the registry's private key.

## 8.2 Credential Completeness

The three-part identity requirement (Section 5.1) exists to prevent agents from acting beyond their authorized scope by substituting their own identity for the originator's. Implementations MUST validate all three identity components independently. Failure to validate any one component MUST result in invocation rejection, not degraded-mode execution.

## 8.3 Manifest Integrity

Implementations SHOULD verify manifest integrity using a registry-provided signature before using manifest data to configure invocations. A compromised manifest could redirect invocations to attacker-controlled endpoints. Implementations MUST respect hard\_ttl\_seconds and MUST NOT use manifests beyond their hard TTL without re-fetching from the registry.

## 8.4 Business Process Scope Enforcement

When business\_process\_id is carried in OTel baggage, implementations SHOULD verify that the invoked service is authorized within the declared business process scope. This is not mandated by the standard because enforcement mechanisms are implementation-specific, but it is strongly recommended as a defense against prompt injection attacks that attempt to cause agents to invoke out-of-scope services.

# Appendix A: Normative References

* RFC 2119 — Key words for use in RFCs to Indicate Requirement Levels
* RFC 6749 — The OAuth 2.0 Authorization Framework
* RFC 8705 — OAuth 2.0 Mutual-TLS Client Authentication
* RFC 9068 — JSON Web Token (JWT) Profile for OAuth 2.0 Access Tokens
* RFC 7517 — JSON Web Key (JWK)
* RFC 7518 — JSON Web Algorithms (JWA)
* W3C Trace Context — Distributed Tracing specification
* W3C Baggage — OpenTelemetry Baggage specification
* JSON Schema 2020-12
* OpenAPI Specification 3.1

# Appendix B: Example Manifest

{

"entity\_iri": "urn:sadar:acme:healthcare.claims:eligibility-check",

"entry\_uuid": "f3a2c1d4-8b9e-4f0a-b6c7-1d2e3f4a5b6c",

"version": "3.0.1",

"status": "ACTIVE",

"registered\_at": "2026-01-15T14:30:00Z",

"soft\_ttl\_seconds": 3600,

"hard\_ttl\_seconds": 86400,

"display\_name": "Member Eligibility Check",

"description": "Verifies a member's current insurance eligibility and returns

coverage details including plan, deductible, and copay.",

"capability\_tags": ["healthcare.claims.eligibility", "insurance.member.lookup"],

"domain": "healthcare.claims",

"data\_classification": "PHI",

"operations": [{

"name": "check\_eligibility",

"description": "Check eligibility for a member by ID and date of service",

"input\_schema": { "$ref": "schemas/eligibility-request.json" },

"output\_schema": { "$ref": "schemas/eligibility-response.json" }

}],

"endpoint\_uri": "https://eligibility.claims.acme.internal/v3",

"protocol": "REST",

"auth\_schemes": ["oidc\_client\_credentials"],

"sla": { "timeout\_ms": 3000, "max\_retries": 2 },

"policy\_tags": ["requires\_phi\_handling", "hipaa\_minimum\_necessary"]

}

End of SADAR Standard Specification | OpenSemantics.org | CC BY 4.0