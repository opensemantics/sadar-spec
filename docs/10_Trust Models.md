SADAR™ Trust Models

*Specification — Version 1.0.0*

Status: Draft • Combines C27 v3 (Trust Model Definitions) + C31 v1 (Asserted SCT Validation)

*April 2026*

*© 2026 Cognita AI Inc. All rights reserved. SADAR™ is a trademark of Cognita AI, Inc. CogniWeave™ is a trademark of Cognita AI, Inc. All other trademarks are the property of their respective owners. Licensed under the Community Specification License 1.0.*

## Document Overview

This specification defines the SADAR Trust Models — the four normative patterns by which an originator's identity propagates through a multi-agent flow — and the validation rules a server applies to incoming Signed Context Tokens (SCTs) under each model. The document combines two related concerns:

Part I — Trust Model Definitions and Negotiation. The four trust models (direct\_auth, asserted, impersonation, deputy), their semantics, the originating\_user\_trust SCT claim, the supported\_trust\_models Protocol NFR, and the bilateral negotiation algorithm with tie-break rules.

Part II — Asserted Trust Model SCT Validation. For the asserted model specifically, the server-side validation algorithm covering asserter identity validation, originator namespacing validation, Role claim validation with default-role handling, and trust model match verification.

**Cross-references.** scope.md §5.1.8 (Identity, Authentication, and Trust). C2 v2 (SCT Operations — defines the SCT structure carrying the originating\_user\_trust claim and Role claim). C19 v1.2 §8.1 (supported\_trust\_models Protocol NFR) and §8.2 (supported\_roles Protocol NFR including is\_default field). C24 v2 §5.1.3.X (Bilateral Manifest Verification — provides the framework into which SCT validation integrates).

— — —

# Part I — Trust Model Definitions and Negotiation

## §I.1 Purpose

§I.1.1 — In agentic systems, an originator (typically a human user, but possibly a system-of-record service) initiates an operation that may pass through multiple agents before reaching a final action. At each handoff, the originator's identity and authority must be propagated in some form. There are multiple legitimate patterns for this propagation; this specification establishes the four normative patterns SADAR recognizes and the means by which they are negotiated and declared.

§I.1.2 — Why a normative taxonomy. Without a normative taxonomy, agent-to-agent invocations have no shared vocabulary for "in what sense is this caller acting on behalf of whom." Servers cannot reliably enforce policy because the enforcement rules differ depending on whether the caller is the originator (direct\_auth), an asserter making a claim about an originator (asserted), the originator under a setuid pattern (impersonation), or a delegate carrying the originator's authority while retaining its own identity (deputy). Each pattern has different security properties, different audit implications, and different fits with downstream policy decisions.

## §I.2 The Four Trust Models

|  |  |
| --- | --- |
| **Model Identifier** | **Semantics** |
| direct\_auth | The originator authenticated directly with an Identity Provider (IdP) within this session. The agent making the invocation acts on the originator's own session authority. Most familiar pattern; the SCT carries the originator's directly-verified identity. |
| asserted | The calling application asserts the originator's identity and Role under its own service credentials. The originator did NOT authenticate to the receiving service in this session; the asserter is presenting an authenticated claim "originator X is performing this action under role Y." Used where direct authentication is impractical (e.g., back-office batch processing, system-of-record initiation) but attribution must be preserved. |
| impersonation | The agent operates AS the originator at downstream services. A setuid-style pattern; the downstream service sees the originator's identity, not the agent's. Carries operational risks (loss of agent attribution at the immediate hop) but preserves originator-as-actor semantics for policy that depends on the originator's authority. |
| deputy | The agent operates with the originator's authority but its own identity. A constrained delegation pattern; the downstream service sees both identities and may apply policy based on either or both. Generally preferred over impersonation where both identities have meaning, since attribution is preserved at every hop. |

§I.2.1 — Lowercase identifiers. The model identifiers (direct\_auth, asserted, impersonation, deputy) are case-sensitive lowercase. They appear in this exact form in SCT claims and manifest NFR arrays.

§I.2.2 — Mutually exclusive per invocation. Each invocation operates under exactly one trust model. The model is determined by bilateral negotiation at discovery time per §I.6.

## §I.3 The originating\_user\_trust SCT Claim

§I.3.1 — Every SCT chain link SHALL carry an originating\_user\_trust claim identifying the trust model under which the chain was opened. Once set at the Open operation (per C2 v2), the value persists through Continue, Hold, Authoritative Carry, and Close operations. The chain's trust model is fixed at Open.

§I.3.2 — Claim format. The claim value is one of the four lowercase identifiers from §I.2:

"originating\_user\_trust": "direct\_auth"
"originating\_user\_trust": "asserted"
"originating\_user\_trust": "impersonation"
"originating\_user\_trust": "deputy"

§I.3.3 — Mid-chain trust model changes. The originating\_user\_trust value MUST NOT change mid-chain. If a flow needs to operate under a different trust model partway through, the chain SHALL be Closed (per C2 v2) and a new chain SHALL be Opened with the new trust model. The cryptographic relationship between the chains is preserved through the chain-of-chains mechanism in C2 v2 §B.7.5.

## §I.4 The supported\_trust\_models Protocol NFR

§I.4.1 — Manifests declare which trust models the entity supports for each capability. This is a Protocol NFR per C19 v1.2 §8.1:

urn:sadar:nfr:v1:protocol:supported\_trust\_models

§I.4.2 — The value is an array of trust model identifiers. The order of the array IS significant — it expresses the declaring entity's preference ordering, used in the negotiation algorithm of §I.6.

§I.4.3 — Examples:

// A server preferring deputy over asserted, also accepting direct\_auth
"urn:sadar:nfr:v1:protocol:supported\_trust\_models":
 ["deputy", "asserted", "direct\_auth"]

// A requester only willing to operate under deputy
"urn:sadar:nfr:v1:protocol:supported\_trust\_models": ["deputy"]

§I.4.4 — Cardinality requirement. Manifests SHALL declare at least one trust model in supported\_trust\_models. A manifest declaring no trust models is malformed and rejected at registry-side validation per C19 v1.2 §15.

§I.4.5 — In requester section. Indicates which trust models the requester is willing to operate under when calling other capabilities.

§I.4.6 — In server section. Indicates which trust models the server accepts for incoming invocations of this capability.

## §I.5 Per-Model Behavior

### §I.5.1 direct\_auth

§I.5.1.1 — Properties. The originator's identity is established through direct authentication with an IdP. The SCT carries authentication evidence (typically the IdP's ID token or derived assertion). The originator and the calling agent are either the same identity (originator-driven) or the agent acts under the originator's session authority (session-bound).

§I.5.1.2 — Use cases. User-initiated agent flows where the user directly logs in and the agent operates within the user's session. Familiar from web application architectures.

§I.5.1.3 — Server policy implications. The server typically applies user-level policy (authorization based on the originator's roles and permissions). Audit attribution is to the originator.

### §I.5.2 asserted

§I.5.2.1 — Properties. The calling application authenticates with its own service credentials (typically OAuth Client Credentials) and asserts a claim about the originator. The originator does NOT authenticate to the receiving service in this session. The SCT carries the asserter's identity and the asserter's claim about the originator's identity, plus a Role claim drawn from the server's declared supported\_roles.

§I.5.2.2 — Use cases. Back-office batch processing where system-of-record services initiate operations on behalf of historical originators. Service-to-service calls where the originator is logically present but not authenticatable in this session. ETL pipelines processing user-attributed records.

§I.5.2.3 — Server policy implications. The server validates the assertion and applies role-based policy per the Role claim. Audit attribution names both the asserter and the asserted originator. The server-side validation algorithm is defined in Part II of this document.

§I.5.2.4 — Asserter trust prerequisite. The asserted model depends on the server trusting the asserter's authority to make claims about originators. This trust is established organizationally (the server trusts certain asserters) and enforced by per-server policy on which asserters are accepted. SADAR does not normatively define how asserter trust is established; it provides the protocol substrate.

### §I.5.3 impersonation

§I.5.3.1 — Properties. The agent presents the originator's identity (not the agent's) at downstream services. setuid-style semantics — the agent steps into the originator's shoes. The SCT carries the originator's identity in the position normally occupied by the caller, and may include an indication of which agent is impersonating.

§I.5.3.2 — Use cases. Where downstream services apply policy based exclusively on the originator's identity and the agent is essentially a stand-in. Legacy integration where downstream services have no notion of agents and only understand users.

§I.5.3.3 — Operational considerations. Impersonation loses agent attribution at the immediate hop — the downstream service sees the originator, not the agent. This makes operational forensics harder when agent-specific behavior is at issue. Logging at the impersonating agent SHOULD preserve the agent's identity locally even though it does not appear in the SCT forwarded downstream.

§I.5.3.4 — Risk profile. impersonation typically has the highest operational risk among the four models because it allows agents to act as users with the users' full authority. Servers SHOULD consider whether deputy (which preserves agent identity) is sufficient before accepting impersonation.

### §I.5.4 deputy

§I.5.4.1 — Properties. The agent presents both its own identity AND the originator's identity. The SCT carries both. The agent operates with the originator's authority but downstream services see the agent as the actor. A constrained delegation pattern.

§I.5.4.2 — Use cases. Agent-driven flows where preserving attribution at every hop is important. Multi-step business processes where an agent reads from one system on behalf of a user and writes to another. Audit-rich environments.

§I.5.4.3 — Server policy implications. The server may apply policy based on either identity or both. Common patterns:

Originator-based authorization with agent identity logged for audit.

Both-identity authorization where the operation requires both the originator's authority AND the agent being on a trusted-agent allowlist.

Agent-bounded scope — the originator may have broad authority, but operations under the deputy SCT are constrained to a subset appropriate to the agent.

§I.5.4.4 — Recommended default. Where multiple models are feasible, deputy is generally preferred over impersonation because it preserves attribution. The negotiation algorithm of §I.6 reflects this preference in tie-break behavior.

## §I.6 Negotiation Algorithm

§I.6.1 — At discovery time, the requester and the candidate server's supported\_trust\_models arrays are evaluated to select the trust model used for the invocation. The algorithm:

Step 1 — Compute the intersection of the requester's and the server's supported\_trust\_models.

Step 2 — If the intersection is empty, the candidate is NO\_MATCH on this dimension. Bilateral matching per C19 §12 excludes the candidate from results.

Step 3 — If the intersection has exactly one element, that is the negotiated trust model.

Step 4 — If the intersection has multiple elements, compute combined preference rank for each candidate model: rank = (requester's position) + (server's position) where position 0 is most-preferred. Lower combined rank wins.

Step 5 — If multiple models tie on combined rank, apply the tie-break rule: prefer deputy over other models. If deputy is not in the tied set, the resolver receives the tied candidates and selects per its policy.

§I.6.2 — Why deputy wins ties. deputy is the more constrained pattern: it preserves agent identity downstream, enabling fuller audit and per-hop policy. Where multiple models would work, the more-attributable choice is preferred.

§I.6.3 — Negotiation result encoding. The negotiated trust model is encoded in the discovery result so the requester knows which model to operate under. SAI invokes the target with an SCT chain Opened under the negotiated model.

§I.6.4 — Worked example.

Requester supported\_trust\_models: ["deputy", "asserted", "direct\_auth"]

Server supported\_trust\_models: ["asserted", "deputy"]

Step 1 — Intersection: {asserted, deputy}.

Step 4 — Combined ranks: deputy = 0 (req) + 1 (svr) = 1; asserted = 1 (req) + 0 (svr) = 1.

Step 5 — Tie. Prefer deputy. Negotiated: deputy.

## §I.7 Originator Namespacing

§I.7.1 — Across the four trust models, originator identifiers appear in different forms in the SCT. SADAR defines a normative namespacing pattern that handles all four uniformly:

|  |  |  |
| --- | --- | --- |
| **Model** | **Originator Identifier Format** | **Notes** |
| direct\_auth | <idp\_urn>:<originator\_id> | IdP identifier prefixed to the originator ID. The IdP is the naming authority for direct-auth originators. |
| asserted | <asserter\_urn>:<originator\_id> | Asserter identifier prefixed to the originator ID. The asserter is the naming authority for asserted originators. |
| impersonation | <idp\_urn>:<originator\_id> | Same as direct\_auth — the originator is presented as having authenticated even though the agent is the actor. |
| deputy | <idp\_urn>:<originator\_id> | Same as direct\_auth — the originator's authenticated identity is preserved alongside the agent's identity. |

§I.7.2 — Why the asserter is the naming authority for asserted. In the asserted model, there is no IdP for the originator in this session. The asserter is making a claim about an originator whose identity exists in the asserter's domain (employee ID, customer ID, account number, etc.). Namespacing under the asserter prevents collision when multiple asserters use the same local identifiers — asserter-A:emp\_123 and asserter-B:emp\_123 are distinct originators.

§I.7.3 — Originator URN structure. The complete originator identifier in the SCT is a URN of the form:

urn:sadar:originator:<naming\_authority>:<originator\_id>

where <naming\_authority> is the IdP URN for direct\_auth, impersonation, and deputy, or the asserter URN for asserted.

— — —

# Part II — Asserted Trust Model SCT Validation

## §II.1 Purpose

§II.1.1 — When a server receives an invocation under the asserted trust model, it SHALL apply the validation algorithm specified in this Part. Validation is fail-closed: a server that does not validate, or that validates incorrectly, is not conformant.

§II.1.2 — Why asserted requires explicit validation. The asserted model carries inherent risks not present in direct\_auth: the originator's identity is asserter-asserted, not directly verified. Without explicit, normative validation rules, servers might accept malformed or malicious assertions; audit trails might attribute operations to wrong originators; role-based policy might apply to fabricated roles. Part II closes these gaps.

## §II.2 Scope

§II.2.1 — In scope. Validation rules applied by a server when processing an asserted-model SCT (§II.4 algorithm), the four rejection conditions (§II.5), the role declaration mechanism including the is\_default field per C19 v1.2 §8.2.1 (§II.6), and the structured error response namespace (§II.7).

§II.2.2 — Out of scope. The negotiation that selected the asserted model in the first place (Part I §I.6 above). The manifest verification at invocation that establishes the caller's identity (C24 v2 §5.1.3.X). The trust relationship between server and asserter (organizational policy, not protocol).

## §II.3 Asserter Trust Prerequisite

§II.3.1 — A server accepting asserted-model invocations maintains a list (explicit or policy-based) of asserters whose assertions it will honor. SADAR does not define this list's format or maintenance; it is the server's organizational concern. The validation algorithm (§II.4) consults this list as a prerequisite step.

## §II.4 Validation Algorithm

§II.4.1 — At invocation, the server SHALL perform the following validation steps in order. Failure at any step rejects the invocation with the corresponding error code from §II.7.

### §II.4.1 Step 1 — Trust Model Match

§II.4.1.1 — Verify that the SCT's originating\_user\_trust claim is "asserted". If the claim is any other value, this algorithm does not apply (a different validation path runs for other trust models).

§II.4.1.2 — Verify that "asserted" is in the server's supported\_trust\_models. If the server has not declared asserted support, the invocation is rejected with urn:sadar:error:v1:trust\_model:invalid\_trust\_model.

### §II.4.2 Step 2 — Asserter Identity Validation

§II.4.2.1 — The SCT's sub claim (or equivalent caller identity field) is the asserter's identity. The server SHALL:

Verify the asserter URN is well-formed.

Resolve the asserter URN to an active, signed manifest in the registry. If resolution fails (no manifest, manifest in non-active lifecycle state), reject with urn:sadar:error:v1:trust\_model:asserter\_not\_resolvable.

Verify the asserter's manifest signature is valid per the asserter's registry-published key material. Reject with urn:sadar:error:v1:trust\_model:invalid\_asserter\_identity on failure.

Verify the asserter is on the server's trusted-asserter list (per §II.3). If not, reject with urn:sadar:error:v1:trust\_model:invalid\_asserter\_identity.

### §II.4.3 Step 3 — Originator Namespacing Validation

§II.4.3.1 — The SCT carries an originator identifier in the urn:sadar:originator:<asserter>:<originator\_id> form per §I.7. The server SHALL:

Parse the originator URN and extract the <asserter> segment.

Verify the <asserter> segment matches the asserter identity validated in Step 2. Originators may be namespaced only under their actual asserter; namespacing under a different asserter is rejected with urn:sadar:error:v1:trust\_model:invalid\_originator\_namespacing.

### §II.4.4 Step 4 — Role Claim Validation

§II.4.4.1 — The SCT may carry a Role claim drawn from the server's supported\_roles per C19 v1.2 §8.2. The server SHALL:

Look for a Role claim in the SCT.

If a Role claim is present, verify that its value is a role\_id declared in the server's supported\_roles. If not, reject with urn:sadar:error:v1:trust\_model:unrecognized\_role.

If a Role claim is absent, apply default-role handling per §II.4.5.

If the Role claim is structurally malformed (not a string, not matching the role\_id format), reject with urn:sadar:error:v1:trust\_model:malformed\_role\_claim.

### §II.4.5 Default-Role Handling (uses C19 v1.2 is\_default)

§II.4.5.1 — When no Role claim is present in the SCT, the server applies one of four cases:

|  |  |  |
| --- | --- | --- |
| **Case** | **Condition** | **Action** |
| Case A | No supported\_roles declared | Server has not declared asserted support per §II.4.1; this case should not occur (Step 1 rejects). If reached anyway, reject with internal\_error. |
| Case B | supported\_roles declared but no role has is\_default=true | Reject with urn:sadar:error:v1:trust\_model:missing\_role\_claim. The server requires an explicit Role claim because no default is declared. |
| Case C | supported\_roles declared with exactly one role having is\_default=true | Apply that role as the role for this invocation. Continue validation. |
| Case D | supported\_roles declared with multiple roles having is\_default=true | This is a malformed manifest — registry-side validation per C19 v1.2 §15.2.5 rejects manifests with multiple defaults. If reached anyway (e.g., manifest published before validation was enforced), the server SHOULD reject with urn:sadar:error:v1:trust\_model:multiple\_default\_roles, mirroring the registry-side error code from C19. |

§II.4.5.2 — Case C is the operational expectation. A server supporting asserted invocations from systems that may not always include explicit Role claims declares a default role (e.g., "claim\_processor" with is\_default=true) so that invocations without explicit role assignment still receive meaningful policy-applicable role context.

§II.4.5.3 — Case B is the strict-policy alternative. A server requiring every invocation to specify a role explicitly declares no default role. Invocations without a Role claim are rejected; the asserter must include explicit role assignment.

### §II.4.6 Successful Validation

§II.4.6.1 — If all four steps succeed (with Case C applied in §II.4.5 if needed), the invocation proceeds to authorization policy evaluation. The validated context for policy purposes comprises:

asserter\_identity — the validated asserter URN.

originator\_identity — the namespaced originator URN.

role\_id — the validated role identifier (either explicit from Role claim or default per §II.4.5 Case C).

role\_permissions — the permission objects, process\_authority, and audit\_template fields associated with the validated role.

## §II.5 Rejection Conditions Summary

§II.5.1 — The validation algorithm has six distinct rejection conditions (one per step plus default-role cases B and D). Each maps to a specific error code in the urn:sadar:error:v1: namespace per §II.7. Errors SHALL be structured per the standard SADAR error format:

{
 "code": "urn:sadar:error:v1:trust\_model:invalid\_asserter\_identity",
 "message": "Asserter manifest signature failed verification.",
 "asserter\_urn": "urn:registry:abc:asserter:foo",
 "trace\_id": "..."
}

§II.5.2 — Fail-closed. Any uncertainty in validation rejects rather than passes. A server cannot conformantly choose to "give the benefit of the doubt" on validation failure.

## §II.6 Role Declaration Object Schema

§II.6.1 — Reference. The Role Declaration Object schema is defined in C19 v1.2 §8.2.1. Reproduced here for convenience and to call attention to the is\_default field that supports the default-role handling in §II.4.5.

|  |  |  |  |
| --- | --- | --- | --- |
| **Field** | **Type** | **Required** | **Description** |
| role\_id | string | YES | Identifier matching ^[a-z][a-z0-9\_]\*$. Unique within this manifest. |
| description | string | YES | Human-readable description of the role. |
| permissions | array of objects | YES | Permission objects with operation, resource, and scope fields. |
| process\_authority | string (URN) or array | NO | Constraint on which business processes this role may operate within. |
| audit\_template | object | NO | Template fields to include in audit records of invocations under this role. |
| is\_default | boolean | NO | When true, this role applies to invocations without an explicit Role claim per §II.4.5 Case C. At most one role per manifest may declare is\_default=true (C19 v1.2 §8.2.4.4 constraint). |

§II.6.2 — Registry-side validation. C19 v1.2 §15.2.5 rejects manifests declaring more than one role with is\_default=true. Servers receiving manifests that pre-date this validation (or from non-conformant registries) SHOULD apply Case D handling per §II.4.5.

## §II.7 Error Code Namespace

|  |  |
| --- | --- |
| **Error Code** | **Meaning** |
| urn:sadar:error:v1:trust\_model:invalid\_trust\_model | "asserted" not in server's supported\_trust\_models. §II.4.1.2. |
| urn:sadar:error:v1:trust\_model:invalid\_asserter\_identity | Asserter identity validation failed (manifest signature, trusted-asserter list). §II.4.2.1. |
| urn:sadar:error:v1:trust\_model:asserter\_not\_resolvable | Asserter URN does not resolve to an active manifest in the registry. §II.4.2.1. |
| urn:sadar:error:v1:trust\_model:invalid\_originator\_namespacing | Originator URN's asserter segment does not match the validated asserter identity. §II.4.3.1. |
| urn:sadar:error:v1:trust\_model:unrecognized\_role | Role claim value is not in server's supported\_roles. §II.4.4.1. |
| urn:sadar:error:v1:trust\_model:malformed\_role\_claim | Role claim is structurally invalid. §II.4.4.1. |
| urn:sadar:error:v1:trust\_model:missing\_role\_claim | No Role claim present and no default role declared. §II.4.5 Case B. |
| urn:sadar:error:v1:trust\_model:multiple\_default\_roles | Server's manifest declares multiple roles with is\_default=true. §II.4.5 Case D. |

— — —

# Change Log

|  |  |  |
| --- | --- | --- |
| **Version** | **Date** | **Changes** |
| 1.0.0 | April 2026 | Initial publication. Combines C27 v3 (originator trust flag taxonomy, four trust models, supported\_trust\_models NFR, bilateral negotiation algorithm with deputy tie-break, originator namespacing) and C31 v1 (asserted trust model SCT validation with four rejection conditions, default-role handling integrating with C19 v1.2 is\_default field, urn:sadar:error:v1:trust\_model:\* error codes). Per the integration push Q6 decision, the two are combined into a single companion document because they address related concerns and read more coherently together. Y1 normalization applied — "server" used throughout in preference to "responder." |

*© 2026 Cognita AI Inc. All rights reserved. SADAR™ is a trademark of Cognita AI, Inc. CogniWeave™ is a trademark of Cognita AI, Inc. All other trademarks are the property of their respective owners. Licensed under the Community Specification License 1.0.*