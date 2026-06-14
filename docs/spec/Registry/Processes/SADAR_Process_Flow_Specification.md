# Process Flow Specification

**SADAR Companion Document**

*Normative specification for process flow semantics*

Draft — June 2026

# **1. Introduction**

## **1.1 Scope**

This companion document is the normative specification for SADAR process flow semantics. It defines: the Process Definition Model (the structural form of a process flow), process composition (how processes reference other processes), layered relations between process-level and manifest-level declarations, compensating transactions, error and exception logic, and the discovery-time and runtime evaluation rules that apply to each.

The Process page covers process declarations at orientation depth: what a process is, the structural properties of declarations, and the bilateral matching pattern. This document covers everything that page defers.

## **1.2 Normative Basis**

The normative basis lives across multiple sources. Scope §5.1.5 establishes process as a first-class manifest element. NFR Schema §9 defines the field-level structure of the *process* NFR. This document defines the schema and semantics of Process Definitions themselves and of all flow content not covered in the manifest-level *process* NFR.

## **1.3 Conventions**

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in RFC 2119 and RFC 8174 when, and only when, they appear in all capitals.

Field names and IRI fragments are presented in regular type within prose; in schema tables they appear unmarked. Code-style monospace formatting is reserved for the rendered Web version of this document.

## **1.4 Audience**

This document is intended for: implementers writing or evaluating SADAR-conformant process engines; authors of SADAR Process Definitions in industry, vertical, or implementer-specific namespaces; standards practitioners reasoning about cross-organizational process composition; and audit/compliance teams scoping process-flow forensics.

# **2. Process Definition Model**

## **2.1 Overview**

A **Process Definition** is a structured, signed document that formalizes a single process flow. It is the specification of what the process is, how it composes with other processes, what it requires of participants, what error and compensation semantics it carries, and how it should be evaluated.

Process Definitions are **signed manifests in the registry**. They share the registry isolation, manifest signing, JWKS-published key, and immutability-per-version properties of every other SADAR manifest. Authoring is open: a Process Definition MAY be authored by the grounding taxonomy authority, an industry consortium, an organization, or any conformant entity. Multiple Process Definitions MAY exist for the same grounding taxonomy concept (e.g., for different industry-specific realizations); each Definition has its own *definition\_iri* and is independently referenceable.

Where the Process overview treats a process as the IRI a manifest references, this document treats the Process Definition as the structured artifact *behind* that IRI. The two views are reconcilable: a manifest's process declaration carries an IRI; the IRI dereferences (via registry resolution) to the Process Definition; the Definition is the source of process-level structural content.

## **2.2 Schema**

A Process Definition has the schema below. Required fields MUST be present; optional fields MAY be omitted (their absence has defined semantics described in subsequent sections).

| **Field** | **Required** | **Type** | **Description** |
| --- | --- | --- | --- |
| definition\_iri | MUST | IRI | The IRI that uniquely identifies this Process Definition. Distinct from the underlying taxonomy concept IRI (see grounding\_iri). |
| grounding\_iri | MUST | IRI | The IRI of the underlying process concept this definition realizes. Typically a taxonomy reference (e.g., APQC PCF) or implementer-namespace identifier. |
| version | MUST | string | Version identifier for this definition. SemVer is RECOMMENDED. Process Definitions are immutable per version. |
| author\_entity | MUST | URN | The URN of the entity that authored and signed this Process Definition. Basis for trust evaluation. |
| title | MUST | string | Human-readable name for this process. Free-form. |
| description | SHOULD | string | Human-readable description of what this process accomplishes. |
| prerequisites | MAY | list of IRIs | Process Definition IRIs of processes that MUST have completed before this process can be entered. Process-level prerequisites; manifests MAY add additional prerequisites but MUST NOT remove these. |
| sequencing | MAY | list of constraints | Temporal ordering relations against other processes. See §2.4. |
| exclusions | MAY | list of exclusions | Other processes that MUST NOT coexist with this one. Each carries a strictness flag (strict | non-strict). |
| components | MAY | list of components | Component processes composing this one. See §3. |
| compensation | MAY | compensation block | Compensation declarations. See §5. |
| error\_handling | MAY | error block | Error and exception handling. See §6. |
| lifecycle\_state | MUST | enum | active | deprecated | revoked. Drives consumer behavior. |
| signature | MUST | JWS | Author's cryptographic signature over the definition content. |

Process Definitions MUST be JSON-serializable for registry storage and transmission. The wire form is a JSON object with field names matching the schema above. Implementations MAY use other serializations internally provided round-trip equivalence is preserved.

## **2.3 Identity and Versioning**

Every Process Definition MUST be uniquely identified by its *definition\_iri*. The IRI MUST conform to RFC 3987 (Internationalized Resource Identifiers) and SHOULD use a URN namespace pattern (urn:<authority>:process:v<version>:<name>) or HTTP-URI rooted in a domain controlled by the author.

The *grounding\_iri* field references the process concept this Definition realizes. For example, a Definition realizing APQC PCF process 5.1.2 (Manage Customer Service Requests) would set grounding\_iri to the APQC PCF IRI for that concept and definition\_iri to a separate IRI representing this specific Definition (e.g., urn:cognita-ai:process:v1:customer-service-request:retail-banking).

Process Definitions are immutable per version. Any change to definition content MUST produce a new version with a new signature. Consumers caching Definitions by IRI may continue to reference older versions until cache expiration; lifecycle transitions (deprecation, revocation) propagate as new versions through normal registry replication.

## **2.4 Sequencing Schema**

The *sequencing* field carries temporal ordering relations against other processes. Each sequencing entry MUST declare:

* **target\_iri** — the Process Definition IRI of the process this sequencing applies to.
* **relation** — one of: must\_complete\_before, must\_not\_begin\_until, may\_run\_in\_parallel\_with, must\_complete\_before\_workflow\_advances. Implementations MAY define additional relations in implementer-namespace IRIs; consumers ignore relations they do not recognize.
* **scope** — one of: invocation (this single invocation chain) or workflow (the broader workflow context). Default is invocation.

Sequencing relations are bilateral by convention: if Process A declares a sequencing relation against Process B, Process B SHOULD declare the symmetric relation against Process A. Definitions are not required to be symmetric (registries do not enforce symmetry), but asymmetric sequencing produces matching ambiguity that implementers SHOULD avoid.

## **2.5 Publication and Discovery**

Process Definitions are published into a SADAR-conformant registry by their author. Registries MUST validate Process Definitions on ingestion against the schema in §2.2 and the additional validation rules in §3 (composition acyclicity), §4 (relation conflicts within the definition), and §10 (conformance). Definitions that fail validation are rejected with a structured error in the *urn:sadar:error:v1:process:\** namespace.

Discovery queries MAY return Process Definition manifests for inspection. A consuming entity wishing to evaluate process compatibility resolves the relevant Process Definition IRIs through registry queries (using the existing search and resolve\_manifest scopes) before completing bilateral matching against another entity's manifest.

# **3. Process Composition**

## **3.1 Overview**

A Process Definition MAY declare **component processes**: other Process Definitions that compose to form this one. A composed process is functionally a higher-level orchestration of its components; an entity claiming participation in the composed process is participating in (some or all of) its components.

The composition mechanism is the structural feature that gives SADAR's process model its scaling property. An entity does not declare an entire procurement chain in its manifest; it declares the specific processes it implements. The chain itself lives in the Process Definition that composes those processes. Other entities can declare different participation patterns against the same composed process without coordination.

Components are listed in the *components* field of the parent Process Definition. The list ordering does not by itself define execution order; the *relation* and *order* fields on each component carry the structural composition.

## **3.2 Composition Relations**

Each component declares a *relation* describing how it composes with the parent and with sibling components:

* **sequential** — the component is part of an ordered sequence. The component's order field defines its position. Sequential components run one after another in order; later components MUST NOT begin until earlier components have completed.
* **parallel** — the component runs concurrently with other parallel components. Parallel components MUST NOT have hard sequencing dependencies among themselves; the parent process completes when all parallel components complete.
* **conditional** — the component runs only if its condition evaluates to true at the point of evaluation. Conditional components have a condition expression; if the condition is false the component is skipped without failing the parent.
* **loop** — the component repeats according to its termination criteria. Termination MUST specify either a max iteration count, an exit condition expression, or both. Loops without explicit termination are non-conformant.

## **3.3 Component Schema**

Each entry in the components field MUST conform to the schema below.

| **Field** | **Required** | **Type** | **Description** |
| --- | --- | --- | --- |
| component\_iri | MUST | IRI | Process Definition IRI of the component process. |
| relation | MUST | enum | sequential | parallel | conditional | loop. See §3.2. |
| order | Conditional | integer | Required for sequential composition. Defines position in the sequence (1-based). Order values MUST be unique within a single sequential composition. |
| condition | Conditional | expression | Required for conditional and loop compositions. Boolean expression evaluated against process state to determine inclusion or termination. |
| termination | Conditional | criteria block | Required for loop compositions. Specifies max iterations, exit conditions, and timeout. |
| optional | MAY | boolean | If true, this component MAY be skipped without failing the parent. Default false. |
| compensation\_iri | MAY | IRI | Override compensation for this specific component. Defaults to the component's own compensation declaration if any. |

## **3.4 Composition Validation**

Registries MUST validate the following at ingestion of any Process Definition declaring components:

* **Acyclicity.** A Process Definition MUST NOT directly or transitively contain itself as a component. Validators MUST traverse the composition graph and reject Definitions whose closure includes the Definition itself. Cycles produce urn:sadar:error:v1:process:composition\_cycle.
* **Reachability.** Every component MUST be reachable in the composition flow under at least one execution path. Components that cannot be reached (unreachable conditional branches, dead loop bodies) MUST be either declared optional or rejected.
* **Termination.** Loop components MUST declare termination criteria. Validators MUST verify presence of either max\_iterations or exit\_condition.
* **Sequencing coherence.** Parallel components MUST NOT declare sequencing relations against each other that would force them into ordered execution. Validators MUST check that parallel siblings are mutually non-sequenced.
* **Order uniqueness.** Within a single sequential composition, order values MUST be unique. Duplicate orders are rejected.

## **3.5 Composition Evaluation at Discovery**

When bilateral matching evaluates a process declaration that references a composed process, the matching algorithm MUST consider both the composed process and its components. The specific evaluation depends on which side declares the composed process:

* **Requester declares composed target\_process.** The requester intends to invoke a composed flow; matching evaluates whether the server provides the components the requester will invoke. Servers typically declare participation in individual components rather than the full composed process.
* **Server declares composed process.** The server orchestrates the composed flow itself; matching evaluates whether the requester's own\_process satisfies the server's requirements\_of\_requester.process for the composed flow.

Discovery does not require both sides to declare identical composition; the bilateral matching tolerates participation at different levels of the composition tree, provided the resulting interaction is coherent.

## **3.6 Worked Example: Procurement Composition**

Consider a procurement Process Definition composed of seven sequential components: Request for Information, Request for Proposal, Request for Quotation, Purchase Order, Invoice, Ship Notice, Payment. Each is itself a Process Definition; each is grounded in APQC PCF (with industry-specific variants where appropriate).

The procurement Process Definition declares each as a component with relation = sequential and order values 1 through 7. The Definition's prerequisites field MAY require an upstream process (e.g., Vendor Onboarding); the Definition's exclusions field MAY exclude a competing one (e.g., Emergency Direct Purchase, with strict = true).

An entity participating in procurement does not declare procurement itself in its manifest. It declares participation in the components it actually implements: a sourcing system might declare Request for Quotation; an ERP might declare Purchase Order, Invoice, and Payment; a logistics partner might declare Ship Notice. Bilateral matching at discovery composes these declarations across organizations to produce the end-to-end procurement flow.

# **4. Layered Relations: Process-Level vs Manifest-Level**

## **4.1 The Two Layers**

Prerequisites, sequencing, and exclusions can be declared at two layers. Both layers MAY contribute to the effective constraint set evaluated at discovery and runtime; their interaction is governed by the rules in §4.2.

* **Process-level relations** live in the Process Definition. They apply to every entity participating in the process by virtue of participating in it. A Purchase Order Definition declaring Request for Quotation as a prerequisite means every entity participating in Purchase Order operates under that prerequisite.
* **Manifest-level relations** live in the entity's manifest declaration of its participation. They apply only to that specific entity's specific declaration. A particular ERP's manifest may add additional prerequisites reflecting its own operational policy without affecting other ERPs' participation in the same process.

## **4.2 Interaction Rules**

The effective constraint set evaluated at discovery and runtime is the combination of process-level and manifest-level relations under the following rules:

| **Operation** | **Permitted?** | **Description** |
| --- | --- | --- |
| Manifest adds prerequisites not in process-level | MAY | Manifests can require additional prerequisites beyond the process-level set. The combined prerequisite set is the union of process-level and manifest-level. |
| Manifest adds exclusions not in process-level | MAY | Manifests can exclude additional processes not excluded by the process definition. The combined exclusion set is the union, with strictness max-merge (strict overrides non-strict). |
| Manifest adds sequencing not in process-level | MAY | Manifests can introduce additional sequencing constraints. Combined sequencing is the union; conflicting sequencing produces a validation error. |
| Manifest narrows process-level options | MAY | Where a process-level relation is permissive (e.g., a list of acceptable predecessors), the manifest can require a subset. |
| Manifest removes a process-level prerequisite | MUST NOT | Manifests cannot relax process-level requirements. Attempts produce urn:sadar:error:v1:process:relation\_conflict on validation. |
| Manifest weakens a strict exclusion to non-strict | MUST NOT | Strictness can be raised by the manifest but never lowered against process-level. |
| Manifest contradicts process-level sequencing | MUST NOT | Where process-level says “must complete before X” and manifest says “may run in parallel with X”, the manifest is rejected. |
| Manifest is silent (no relations declared) | MAY | Process-level relations apply unmodified. This is the default for manifests with no operational policy beyond participation. |

In summary: manifests can supplement, narrow, or strengthen; manifests cannot weaken, remove, or contradict. The asymmetry is deliberate. Process-level relations are commitments that the process makes to all participants; allowing manifests to relax them would defeat the consistency that grounding in a Definition provides.

## **4.3 Conflict Resolution**

When a manifest declaration appears to conflict with the referenced Process Definition, registry validation MUST detect the conflict at ingestion and reject the manifest. Specific conflict cases produce specific error codes in the *urn:sadar:error:v1:process:\** namespace; see Appendix A.

Process Definition updates that introduce new constraints can retroactively conflict with previously-valid manifests. Registries SHOULD detect this on Process Definition publication and notify affected manifest authors. Affected manifests remain valid until expiration but new versions of those manifests MUST conform to the updated Definition.

## **4.4 Validation at Registry Ingestion**

Registries MUST validate manifest process declarations against referenced Process Definitions at ingestion. Validation MUST cover:

* **Resolvability** — the referenced process IRI resolves to a published, non-revoked Process Definition.
* **Schema conformance** — manifest-level relations conform to the schema in §2.
* **Interaction conformance** — manifest-level relations satisfy the rules in §4.2 against process-level relations.
* **Cross-reference consistency** — IRIs referenced in manifest-level relations themselves resolve to published Process Definitions where they identify processes.

Validation failures produce structured errors. The manifest is rejected; the publishing entity addresses the validation issues and republishes a new manifest version.

# **5. Compensating Transactions**

## **5.1 Overview**

A compensating transaction reverses, undoes, or otherwise mitigates the side effects of a previously-completed process. The pattern is the SAGA pattern from distributed-systems literature: rather than holding a long-running global transaction (which is infeasible across organizational boundaries), each step is committed locally, and failure later in the flow triggers compensation in reverse order.

In SADAR, compensation is **optional**. A Process Definition that does not declare compensation does not support rollback; failure of a downstream step in a flow involving such a process leaves the completed step's effects intact. Compensation declarations are how a process opts in to rollback support.

## **5.2 Compensation Declaration**

A Process Definition MAY declare a *compensation* block. The block specifies the compensation process, its scope, and operational parameters.

| **Field** | **Required** | **Type** | **Description** |
| --- | --- | --- | --- |
| compensation\_iri | MUST | IRI | Process Definition IRI of the compensating process. The compensation is itself a process; running compensation means invoking that process. |
| scope | MUST | enum | step (compensate this individual process only) | composition (compensate the broader composed flow). |
| triggers | MAY | list of triggers | Conditions under which compensation initiates. Default: any non-success terminal state of this process. |
| idempotency\_required | SHOULD | boolean | If true (default), compensation MUST be idempotent. Implementations MAY enforce this through idempotency keys carried in compensation invocations. |
| max\_attempts | MAY | integer | Maximum retry attempts for compensation invocation. Distinct from forward-flow retry. Default 3. |
| timeout\_seconds | MAY | integer | Maximum wall-clock time before compensation is considered failed and escalated. Default 300. |

Compensation declarations live at the process level by preference. Manifest-level compensation declarations MAY override the process-level compensation\_iri to substitute an entity-specific compensating process; this is permitted because compensation is fundamentally an operational policy that can vary by implementer.

## **5.3 Compensation Triggers**

Compensation is initiated through one of three trigger paths:

* **Explicit failure** — a step of the flow returns an error that exceeds local retry/recovery and propagates up to a compensation-enabled boundary. The completed prior steps in the flow are then compensated in reverse order, beginning with the most recently completed step.
* **Cascading compensation** — a downstream compensation request itself fails or is incomplete, requiring compensation of additional upstream steps. Cascades MUST be bounded; implementations SHOULD NOT cascade compensation beyond the originating composition's depth.
* **External request** — a compensation invocation arrives from outside the original flow (e.g., a regulatory rollback request, a customer-initiated cancellation). External compensation requests MUST be authorized like any other invocation through the standard authentication baseline.

## **5.4 Compensation Execution**

Compensation execution follows the rules below.

* **Reverse order.** Compensations of multiple completed steps proceed in reverse order of original completion. The most recently completed step is compensated first; the originally first-completed step is compensated last.
* **Idempotency.** Compensation invocations MUST be idempotent unless the compensation declaration explicitly disables this requirement. Receivers MUST handle repeated compensation requests for the same step gracefully (typically by recognizing the step is already compensated and returning success).
* **Partial compensation.** If some compensations succeed and others fail, the result is partial compensation. The failure MUST be visible in the audit fabric; downstream consumers MAY apply additional remediation. SADAR does not mandate a global all-or-nothing semantics; the SAGA pattern explicitly admits partial outcomes.
* **Concurrent compensation.** Multiple compensations against the same chain MUST NOT run concurrently. Implementations MUST serialize compensation attempts through a chain-level lock or equivalent.

## **5.5 Compensation in the SCT Chain**

Compensating invocations issue SCT chain links following the standard SCT operations (see **4. SCT Operations**). Compensation links MUST reference the original invocation's chain link via parent\_jti, producing a continuous audit chain that includes both the forward flow and the compensation. The link's trust model is typically the same as the original; deputy is common, since the compensating entity acts on behalf of the originator.

Compensation links MUST carry an additional claim, *compensation\_of*, set to the chain root jti of the original invocation chain being compensated. This claim is in the urn:sadar:claim:v1:compensation\_of namespace and is used by auditors to walk from a compensation chain to its corresponding forward chain.

## **5.6 Compensation in Telemetry Records and Risk Score**

Each compensating invocation produces its own Telemetry Record (see **Telemetry Record**) and is repatriable under the same Repatriation rules as forward-flow records. The Telemetry Record's compensation\_of field references the original chain root jti, mirroring the SCT claim.

Risk Score (see **Risk Score**) accumulates compensation events. Implementations SHOULD apply Risk Score adjustments for compensation triggers (a compensation event indicates something failed) and MAY apply additional adjustments for partial compensation, repeated compensation attempts, or compensation timeout.

## **5.7 Worked Example: Order Rollback**

Consider an order-fulfillment composition with four sequential components: reserve inventory, charge payment, schedule shipping, notify customer. The Order Process Definition declares compensation; each component Process Definition also declares its own compensation.

During execution, components 1 and 2 (reserve inventory, charge payment) complete successfully. Component 3 (schedule shipping) fails with an error indicating shipping unavailable. Local retry exhausts. The failure propagates to the Order composition's compensation handler.

Compensation runs in reverse order: first, refund payment (the compensation declared by component 2); second, release inventory reservation (the compensation declared by component 1). Component 4 (notify customer) was not reached and requires no compensation. After both compensations succeed, the original Order chain is in a fully-compensated terminal state. A separate notification step (“notify customer of cancellation”) MAY be invoked by orchestration policy; it is a forward-flow invocation, not strictly part of compensation.

If refund payment fails (e.g., the payment provider is unavailable), the compensation enters partial state. Audit records both the successful inventory release and the failed refund. Operational policy decides remediation (retry the refund later, escalate to manual handling, or notify the customer of the partial outcome). Risk Score reflects the cascade.

# **6. Error and Exception Logic**

## **6.1 Error Categories**

Errors in process flows are categorized to support consistent handling across implementers. The three normative categories are below; implementations MAY define additional categories in implementer-namespace IRIs but MUST recognize the three normative categories first.

| **Category** | **Description** | **Examples** |
| --- | --- | --- |
| business | Errors arising from violations of business rules or domain constraints. Generally not retry-eligible because the business rule is unlikely to change on retry. | Insufficient credit, customer not eligible, prerequisite document missing. |
| technical | Errors arising from implementation, infrastructure, or transport failures. Often retry-eligible because the underlying issue may be transient. | Service unavailable, timeout, network reset, transient database error. |
| governance | Errors arising from authorization, policy, or compliance enforcement. Generally not retry-eligible without policy or authorization change. | Authentication denied, scope insufficient, exclusion violated, jurisdictional restriction. |

The category is recorded as a property of the error itself (in the error response envelope) and is used by the matching algorithm and by retry/alternate-path/fallback logic to determine handling.

## **6.2 Error Propagation**

Errors propagate up the composition tree by default. A failure in a component process bubbles up to the parent composition; if unhandled there, it bubbles further up to the parent of the parent; until either a handler resolves it or it reaches the root and terminates the chain.

Propagation can be intercepted at each level by alternate paths (§6.3), fallback (§6.4), retry (§6.5), or compensation (§5). The first matching handler resolves the error; subsequent handlers do not see it. If no handler matches, propagation continues.

## **6.3 Alternate Paths**

A Process Definition MAY declare alternate paths for specific error categories or codes. An alternate path is itself a Process Definition (referenced by IRI); when the originating process encounters a matching error, the alternate is invoked in lieu of failing the parent.

Alternate paths are declared as a list. Each entry specifies a match criterion (error category, error code IRI, or condition expression) and the alternate Process Definition IRI. Matching is first-match-wins; entries are evaluated in declaration order.

Alternate paths are evaluated at the same composition level where the error occurred. If no alternate matches, the error continues to propagate.

## **6.4 Fallback Behavior**

A Process Definition MAY declare a single fallback Process Definition. Fallback is invoked when error propagation reaches the Definition's level and no alternate path matches. Fallback is functionally a final-chance alternate.

Default behavior in the absence of declared fallback is propagation to the parent. A Definition wishing to terminate propagation locally without invoking any alternate flow MAY declare an explicit terminate fallback whose Process Definition IRI is urn:sadar:process:v1:terminate (a SADAR-defined sentinel).

## **6.5 Retry Policy**

A Process Definition MAY declare retry policy for handling transient errors. Retry is distinct from compensation: retry repeats the failing operation hoping it will succeed; compensation reverses completed prior operations.

| **Field** | **Required** | **Type** | **Description** |
| --- | --- | --- | --- |
| max\_attempts | MUST | integer | Maximum total attempts including the initial. MUST be ≥ 1. |
| backoff | MUST | enum | constant | linear | exponential | exponential\_jitter. |
| initial\_delay\_seconds | MUST | integer | Delay before first retry. |
| max\_delay\_seconds | MAY | integer | Cap on delay between retries (relevant for exponential backoffs). |
| retryable\_categories | MAY | list of enums | Error categories eligible for retry. Default: technical only. |
| retryable\_codes | MAY | list of IRIs | Specific error code IRIs eligible for retry. Permits fine-grained retry policy beyond categories. |

Each retry attempt is a separate invocation and produces its own Telemetry Record. Retries are visible in the audit fabric; a failed-then-succeeded sequence is reconstructible from the records. Risk Score MAY accumulate retry-attempt adjustments per the implementer's risk model (multiple retries can indicate fragility worth flagging).

## **6.6 Exception Escalation**

When alternate paths, fallback, retry, and compensation have all been exhausted (or none was declared) and the error continues to propagate to the chain root, the error reaches exception escalation. Escalation MAY be declared at the root Process Definition level via an escalation handler.

* **Escalation handler** — a Process Definition IRI invoked when escalation triggers. Typical handlers initiate human review, notify operational ownership, or hold the chain pending external resolution.
* **Escalation contents** — the chain's full error context (root error, propagation path, attempted resolutions) is passed to the escalation handler as input. The handler is a normal SADAR process invocation; standard authentication and SCT semantics apply.

Default behavior in the absence of declared escalation is chain termination with the root error preserved in the Telemetry Record. No automatic escalation is invoked.

## **6.7 Error Code Namespace**

Process flow errors are emitted in the *urn:sadar:error:v1:process:\** namespace. Specific codes appear throughout this document and are catalogued in Appendix A. Implementations MUST emit these codes for the conditions described; implementations MAY emit additional codes in implementer-namespace IRIs for conditions not covered by the SADAR-defined set.

# **7. Discovery-Time Evaluation**

## **7.1 Overview**

Discovery-time evaluation is the process by which bilateral matching at registry-query time validates that a candidate requester-server pair is compatible across all process declarations. This section specifies the evaluation steps that apply specifically to processes; standard NFR matching is covered in 3. NFR Schema.

## **7.2 Process Reference Resolution**

For each process IRI referenced in the requester's manifest (own\_process, target\_process) and the server's manifest (process, requirements\_of\_requester.process, prerequisites, exclusions, sequencing.target\_iri, components.component\_iri, alternate paths, fallback, etc.), the matching algorithm MUST:

* **Resolve the IRI** to the corresponding Process Definition via registry resolution.
* **Verify lifecycle state** is active (not deprecated or revoked).
* **Verify schema conformance** of the resolved Definition.
* **Surface unresolvable references** as urn:sadar:error:v1:process:reference\_unresolvable. Such references typically arise from cross-registry visibility issues or stale caches.

## **7.3 Composition Matching**

Where either party's declarations involve composed processes (§3), the matching algorithm MUST recursively apply matching to component processes. The depth of recursion is bounded by the registry's configured maximum composition depth (default 10; deeper compositions produce urn:sadar:error:v1:process:composition\_depth\_exceeded).

Composition matching is tolerant: the requester and server need not declare participation at the same level of the composition tree. A requester invoking the composed parent matches a server providing one of the components, provided the component is reachable in the requester's intended invocation path.

## **7.4 Layered Relation Evaluation**

For each manifest-level process declaration, the matching algorithm MUST:

* **Compute the effective constraint set** as the union of process-level (from the resolved Process Definition) and manifest-level (from the manifest declaration) prerequisites, exclusions, and sequencing.
* **Detect interaction conflicts** per §4.2; emit urn:sadar:error:v1:process:relation\_conflict on detection.
* **Apply the effective constraint set** to the candidate match: the effective prerequisites must be satisfiable by the requester's known process state; the effective exclusions must not conflict with any active processes; the effective sequencing must be satisfiable in the intended invocation order.

## **7.5 Compensation and Error Compatibility**

Where a requester's intended invocation requires compensation support (declared via a manifest-level compensation\_required flag), the server's process declarations MUST include compensation declarations for the relevant processes. Servers without declared compensation are not matched as candidates for requesters requiring compensation. This is a hard match constraint; it is not advisory.

Error and exception logic compatibility is informational at discovery rather than a hard match constraint. The matching algorithm MAY surface error-policy differences (e.g., the requester's expected retry policy versus the server's declared retry policy) as match annotations; the resolver decides whether to accept the difference.

# **8. Runtime Evaluation**

## **8.1 Process State in OTel Baggage**

Process state propagates in OTel baggage during in-flight execution as parity-bearing content (see Observability Overview, Cryptographic Parity in Scope §5.1.17). The state carries:

* **active\_processes** — set of process IRIs currently in execution within the invocation chain.
* **completed\_processes** — set of process IRIs successfully completed within the chain.
* **compensated\_processes** — set of process IRIs originally completed but subsequently compensated. Members of this set are also members of completed\_processes (a process is compensated only after completing). The set's content is informative for runtime decisions and audit.

Baggage values are carried under SADAR baggage keys in the *urn:sadar:baggage:v1:process:\** namespace. Specific keys: urn:sadar:baggage:v1:process:active, urn:sadar:baggage:v1:process:completed, urn:sadar:baggage:v1:process:compensated.

## **8.2 Prerequisite Verification**

At each invocation, the receiving entity MUST verify that the referenced target process's effective prerequisites (process-level union manifest-level) are satisfied by the inbound completed\_processes set. Specifically: each prerequisite IRI MUST appear in completed\_processes; prerequisites that appear in compensated\_processes count as not-completed and fail verification.

Failure to satisfy prerequisites MUST cause the invocation to be rejected with urn:sadar:error:v1:process:prerequisite\_not\_satisfied. The error response envelope MUST identify the specific unsatisfied prerequisites.

## **8.3 Exclusion Verification**

At each invocation, the receiving entity MUST verify that no excluded process is currently active. Specifically: the intersection of the target process's effective exclusions with the inbound active\_processes set MUST be empty.

Strict exclusion violation MUST cause invocation rejection with urn:sadar:error:v1:process:strict\_exclusion\_violated. Non-strict exclusion violation MUST NOT reject invocation but MUST contribute a Risk Score adjustment under the urn:sadar:risk-reason:v1:non\_strict\_exclusion\_violation reason and SHOULD surface as a structured warning in the response and the Telemetry Record.

## **8.4 Sequencing Verification**

At each invocation, the receiving entity MUST verify that the target process's effective sequencing constraints are satisfiable given the current active\_processes and completed\_processes state. Specifically:

* **must\_complete\_before(X)** — X MUST NOT be in active\_processes nor (if scope is workflow) in completed\_processes-without-this-process. Violation produces urn:sadar:error:v1:process:sequencing\_violated.
* **must\_not\_begin\_until(X)** — X MUST be in completed\_processes. Violation as above.
* **may\_run\_in\_parallel\_with(X)** — informative; no enforcement. Asserts compatibility for orchestration but does not constrain runtime.
* **must\_complete\_before\_workflow\_advances(X)** — evaluated at workflow-level state transitions; depends on workflow tracking, which is operational rather than spec-mandated.

## **8.5 Process State Transitions**

Process state transitions MUST be recorded in the Telemetry Record (see Telemetry Record) and propagated in subsequent baggage. The canonical transitions:

* **Process entry** — the target process is added to active\_processes when invocation succeeds verification. Recorded in the Telemetry Record's invocation section.
* **Successful completion** — the process is moved from active\_processes to completed\_processes when its terminal state is success. Recorded in the Telemetry Record's outcome section.
* **Failed completion (no compensation)** — the process is removed from active\_processes; not added to completed\_processes. The error continues per error-handling logic.
* **Compensation** — the process IRI is added to compensated\_processes (in addition to its existing membership in completed\_processes). Recorded in the Telemetry Record of the compensating invocation.

# **9. Worked Examples**

## **9.1 Multi-Stage Procurement**

This example illustrates composition (§3), participation across organizations, and effective constraint evaluation.

A Procurement Process Definition (urn:industry-consortium:process:v1:procurement) is composed sequentially of seven components: RFI, RFP, RFQ, PO, Invoice, Ship Notice, Payment. The Definition declares Vendor Onboarding as a process-level prerequisite for the entire flow.

Three organizations participate: a sourcing platform implements RFI, RFP, and RFQ; an ERP implements PO, Invoice, and Payment; a logistics partner implements Ship Notice. Each organization's manifest declares only the components it implements; none declares the composed Procurement process.

At runtime, an invocation chain begins with the sourcing platform invoking RFQ. Vendor Onboarding has been completed earlier; completed\_processes carries vendor\_onboarding's IRI. RFQ enters; matching verifies the prerequisite; the chain proceeds. RFQ completes successfully and moves to completed\_processes. The chain advances to PO at the ERP. Bilateral matching at this boundary verifies the ERP's requirements\_of\_requester.process accepts the sourcing platform's own\_process. The PO process's effective prerequisites (potentially including RFQ from process-level sequencing) are satisfied. The chain proceeds through the remaining components similarly.

If at any point a component fails and the Procurement Definition's compensation declares the originally-completed components must be reversed, compensation runs as described in §5.4: Payment compensation first, then Invoice, then PO, then RFQ, then RFP, then RFI — in reverse order of completion.

## **9.2 Compensation: Cancelled Order**

This example illustrates compensation triggers, partial compensation, and audit reconstruction.

An Order Definition is composed of: reserve inventory (component 1), charge payment (component 2), schedule shipping (component 3), notify customer (component 4). Each component declares its own compensation: release inventory reservation, refund payment, cancel shipping reservation, notify customer of cancellation.

Components 1 and 2 complete successfully; the chain reaches component 3 (schedule shipping). The shipping carrier returns service unavailable. Component 3's retry policy attempts twice with exponential backoff; both retries fail. The error propagates to the Order Definition. The Order Definition's compensation declaration triggers; compensation initiates.

Compensation runs in reverse order of completion: refund payment (compensation for component 2) first; release inventory reservation (compensation for component 1) second. Both succeed. The Order chain reaches a fully-compensated terminal state. Component 4 was never reached and requires no compensation.

Audit reconstruction: an authorized auditor walks the SCT chain. The chain shows components 1 and 2 succeeded; component 3 failed; the chain branched to compensation; compensations of components 2 and 1 succeeded. Telemetry Records corroborate at each step. Risk Score reflects the failure-and-rollback pattern; the operational team reviews because shipping failures with full rollback may indicate a carrier issue.

## **9.3 Error Handling: Alternate Path**

This example illustrates alternate-path selection and the difference between alternate, fallback, and escalation.

A Document Approval Definition is composed of: validate input, check policy, route for approval. The check policy component MAY return either a business error (policy\_violated) or a technical error (policy\_engine\_unavailable). The Definition declares two alternate paths:

* **On policy\_violated (business):** alternate is request\_human\_review (a Definition that escalates the document to a human reviewer).
* **On policy\_engine\_unavailable (technical):** alternate is route\_for\_default\_approval (a Definition that uses cached fallback policy).

Definition also declares retry policy for technical errors (max\_attempts = 3; exponential backoff) and a fallback Definition (terminate; chain ends with the original error preserved). No escalation handler is declared.

During execution, validate input succeeds; check policy returns policy\_engine\_unavailable. Retry attempts twice (after backoff); both fail. The error propagates to the Document Approval level. Alternate-path matching finds policy\_engine\_unavailable in the second alternate's match criterion. route\_for\_default\_approval is invoked; it succeeds. The chain proceeds with the alternate's outcome substituting for the failed component's. The Telemetry Record captures both the original failure and the alternate's substitution. Risk Score reflects the technical-error fallback (lower severity than a business-policy violation).

# **10. Conformance**

## **10.1 Process Definition Conformance**

A Process Definition is conformant when:

* **Schema.** All required fields per §2.2 are present and well-formed.
* **IRI.** definition\_iri and grounding\_iri conform to RFC 3987.
* **Composition.** Where components are declared, the composition validation rules of §3.4 are satisfied.
* **Internal consistency.** Process-level prerequisites, exclusions, and sequencing do not contradict each other within the Definition.
* **Compensation.** Where compensation is declared, the compensation\_iri resolves and the schema of §5.2 is satisfied.
* **Error handling.** Where error\_handling is declared, alternate paths, fallback, retry, and escalation conform to §6.
* **Signature.** The Definition is signed by author\_entity using a JWS over the canonical Definition content; the signature verifies against author\_entity's published JWKS.

## **10.2 Implementation Conformance**

A SADAR-conformant implementation participating in process flows MUST:

* **At discovery:** perform reference resolution (§7.2), composition matching (§7.3), and layered relation evaluation (§7.4).
* **At runtime:** verify prerequisites (§8.2), exclusions (§8.3), and sequencing (§8.4) at each invocation, recording state transitions per §8.5.
* **Where compensation is declared and compensation triggers fire:** execute compensation per §5.4, including reverse order, idempotency, and chain integrity per §5.5.
* **Where error handling is declared:** evaluate alternate paths, fallback, retry, and escalation per §6.
* **Emit structured errors** in the urn:sadar:error:v1:process:\* namespace for all conditions where this document specifies error emission.

## **10.3 Interaction with Other Companion Documents**

This document interacts with other companion documents as follows:

* **4. SCT Operations:** compensating invocations issue SCT chain links per the standard chain operations; the compensation\_of claim adds an additional cross-reference but does not modify chain semantics.
* **6. Telemetry Record and Repatriation:** process state transitions and compensation events are recorded in the standard Telemetry Record schema; the schema's existing fields accommodate process flow content without extension.
* **8. Risk Score Specification:** process flow events (compensation, retry, exclusion violations, escalation) contribute Risk Score adjustments under reasons defined in 8. Risk Score Specification's reason catalog.
* **3. NFR Schema:** the manifest-level process NFR (covered there) carries the manifest-level relations evaluated against the process-level relations defined here.

# **Appendix A: Error Code Catalog**

All error codes defined in this companion document are enumerated below. Implementations MUST emit these codes for the conditions described.

| **Error Code IRI** | **Category** | **Description** |
| --- | --- | --- |
| urn:sadar:error:v1:process:reference\_unresolvable | technical | A process IRI referenced in a manifest or another Process Definition cannot be resolved. |
| urn:sadar:error:v1:process:composition\_cycle | governance | A Process Definition's composition closure includes the Definition itself. |
| urn:sadar:error:v1:process:composition\_depth\_exceeded | governance | Composition recursion depth exceeded the registry's configured maximum. |
| urn:sadar:error:v1:process:relation\_conflict | governance | Manifest-level relations conflict with process-level relations under the rules of §4.2. |
| urn:sadar:error:v1:process:prerequisite\_not\_satisfied | business | A target process's effective prerequisites are not satisfied by the inbound completed\_processes state. |
| urn:sadar:error:v1:process:strict\_exclusion\_violated | governance | A target process's effective strict exclusions intersect with active\_processes. |
| urn:sadar:error:v1:process:sequencing\_violated | governance | A target process's effective sequencing constraints are not satisfied by current process state. |
| urn:sadar:error:v1:process:compensation\_failed | technical | A compensation invocation failed after exhausting compensation max\_attempts. |
| urn:sadar:error:v1:process:retry\_exhausted | technical | A retry policy's max\_attempts was reached without success. |
| urn:sadar:error:v1:process:escalation\_failed | technical | An escalation handler invocation failed. |
| urn:sadar:error:v1:process:lifecycle\_state\_invalid | governance | A referenced Process Definition is in deprecated or revoked state. |
| urn:sadar:error:v1:process:schema\_invalid | governance | A Process Definition or manifest declaration fails schema validation. |

# **Appendix B: Glossary**

| **Term** | **Definition** |
| --- | --- |
| Process Definition | A signed structured document formalizing a single process flow. Identified by definition\_iri; references the underlying canonical concept via grounding\_iri. |
| grounding\_iri | The IRI of the canonical process concept a Process Definition realizes (typically a taxonomy reference). |
| definition\_iri | The IRI uniquely identifying a Process Definition. Distinct from grounding\_iri. |
| Process-level relation | A prerequisite, exclusion, sequencing constraint, compensation declaration, or error-handling declaration that lives in the Process Definition itself and applies to all participants. |
| Manifest-level relation | A relation declared in an entity's manifest, applying only to that entity's specific participation. Supplements or narrows but cannot relax process-level relations. |
| Composition | Declaration of component processes within a parent Process Definition. Components compose sequentially, in parallel, conditionally, or in loops. |
| Compensation | An optional process-flow feature for SAGA-pattern rollback. Compensation is itself a process invoked to reverse the side effects of a completed process when downstream failure makes the original work invalid. |
| Compensation chain | The SCT chain produced by compensating invocations. Linked to the original chain via the compensation\_of claim. |
| Alternate path | A Process Definition invoked in lieu of failing when a specific error matches the alternate's match criterion. |
| Fallback | A final-chance alternate invoked when no other alternate matches. |
| Escalation handler | A Process Definition invoked when error propagation reaches the chain root with no resolution; typical handlers initiate human review. |
| Effective constraint set | The combined set of prerequisites, exclusions, and sequencing produced by union of process-level and manifest-level relations under the interaction rules of §4.2. |
