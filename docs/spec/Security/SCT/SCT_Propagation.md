# SADAR Context Token — Propagation

*Design specification — Draft, June 2026*

*Companion to **SCT Structure** (segment schema), **9. SCT Operations** (chain operations), and Core Security §4.6 (orientation). Where SCT Structure defines the segment and SCT Operations defines the operations over a chain, this document defines how the chain is **carried**: across hops within an environment, across organizational/registry boundaries, and how environment-local state and context propagation participate (§8).*

*© 2026 Cognita AI Inc. All rights reserved. SADAR™ is a trademark of Cognita AI, Inc. CogniWeave™ is a trademark of Cognita AI, Inc.*

---

## 1. Scope and Relationship to Other Documents

This document specifies SCT **propagation** — the transport and custody of the SADAR Context Token (SCT) across an invocation flow. It is distinct from:

- **9. SCT Operations** — defines the abstract chain operations (Validate, Extract Claims, Append Segment, Verify Chain Integrity). Propagation *uses* those operations; it does not redefine them.
- **§4.6 (SCT structure)** — defines the JWS-inside-JWE segment format. Propagation carries that structure; it does not alter it.
- **10. Trust Models** — defines how originator authority (direct_auth, asserted, impersonation, deputy) is expressed within the chain. Propagation carries the chain that bears those claims.

The defining concern here is: **the SCT is per-hop mutable (re-signed at each step) and multi-recipient (each segment encrypted to the next party). How does it move, who holds it between hops, and how does it cross trust boundaries without any agent or framework needing to understand it?**

---

## 2. Transport: The `SADAR-SCT` Header

### 2.1 Canonical transport

The SCT is transmitted in the **SCT Header** — the term of art for the dedicated HTTP header whose literal field name is **`SADAR-SCT`** — on every invocation, **alongside** the OIDC usage token (in the `Authorization` header), never embedded within it. (Prose refers to "the SCT Header"; the literal wire token is `SADAR-SCT`, exactly as "the Authorization header" names the literal `Authorization` field.) Both are required on every request:

- The **usage token** (OAuth2/OIDC, DPoP-bound, mTLS channel) provides authentication and replay protection at the configured credential scope. Minted by the serving entity as sovereign issuer; stable and cacheable.
- The **SCT** (`SADAR-SCT` header) provides per-invocation authorization context and chain-of-custody. Constructed and re-signed per invocation by SAI; accrues across hops independently of any authorization server.

The two are deliberately decoupled siblings, not nested.

### 2.2 Why the SCT is NOT an OIDC claim or claim extension

Embedding the accruing SCT inside the OIDC token breaks three invariants:

- **Lifecycle mismatch.** The usage token is minted once at first-use and cached. The SCT is re-signed every hop. Embedding the SCT would force token re-minting through the authorization server on every chain extension — putting the AS back in the runtime path and contradicting service-sovereign issuance.
- **Issuer mismatch.** The usage token is issued by the *serving* entity (callee); the SCT is issued by *SAI on the requester side*. Different parties, keys, and clocks. A claim inside a token is signed by the token's issuer — so embedding would require the serving entity to sign the requester's SCT, collapsing non-repudiation.
- **Structure mismatch.** A JWS-inside-JWE chain with per-segment recipients does not fit a single-audience OIDC claim value. Segment N is wrapped for hop-N's recipient; the token is wrapped for the current callee. The confidentiality model forbids the embedding.

**The only originator-related element that legitimately belongs in an OIDC claim** is the originator-propagation *reference* that seeds the chain root at genesis — never the accruing SCT itself.

> **Note — "natural propagation" is illusory.** Putting the SCT in the token would not make it propagate for free. OIDC tokens do not self-propagate; the framework still forwards the token hop-to-hop. Embedding the SCT only changes what is in the suitcase, not who carries it — while breaking the three invariants above.

---

## 3. Custody Between Hops: Stateless SAI over a Shared Persistence Layer

### 3.1 The propagation-burden problem

Within an environment, *something* must carry the SCT from one hop to the next. The agent is SCT-blind by design; the orchestration framework (LangChain, CrewAI, etc.) is SADAR-unaware. Two models resolve who carries it:

| Model | Framework burden | SAI burden | Where cross-hop state lives |
|---|---|---|---|
| **SAI propagates on the wire** | SADAR-aware adapter per framework | None across hops | On the wire (`SADAR-SCT` header) |
| **SAI over shared persistence** (recommended in-environment) | None for the SCT (only the flow handle / context) | Read-append-write per call | In the persistence layer, keyed by the composite key (§10) |

### 3.2 Stateless SAI, stateful store

**Each SAI instance is stateless; the persistence layer holds the chain state.** On each hop, SAI reads the relevant SCT link from the store, appends its segment, and writes the result back — holding nothing between calls. This is what lets SAI scale horizontally and lets each link be resolved and signed independently (no SAI instance depends on another's memory). The store is stateful; SAI is not.

The lookup is keyed by the **composite key** defined in §10 — not by TraceID alone. (TraceID-alone is insufficient the moment a flow has parallel branches or iterates; see §10.3.)

**Implementation note (non-normative).** The reference implementation may use Valkey, Redis, or — for a single-process demo — an in-memory map or DiskCache. *None of these is a normative requirement.* The normative requirement is only that cross-hop chain custody within an environment be maintained by SAI without requiring agent or framework SCT-awareness, keyed such that parallel branches do not collide (§10.3). A single-process in-memory deployment (the in-address-space singleton, §10.6) needs no external store at all; persistence is required only for **durability** — surviving process restart across an async gap and providing a crash-proof audit record.

### 3.3 Statelessness does not weaken confidentiality

The SCT is JWS-inside-JWE. Even when SAI holds the chain in the store, it can only **append and re-sign**; it cannot read prior segments for which it is not the JWE recipient. The store holds the chain as ciphertext-bearing structure. **The store saves propagation, not decryption authority.** Confidentiality is preserved exactly as on the wire.

---

## 4. Session-Store Security: The SADAR SCT Key

### 4.1 The threat

Keying the session store by TraceID creates an authorization surface: **the TraceID is not a secret** — it propagates in OTel headers and logs across every hop and (across boundaries) across organizations. If possession of the TraceID alone suffices to retrieve or extend a chain, an attacker who observes the TraceID can:

- **Not** decrypt content (JWE protects confidentiality even on theft), but
- **Hijack continuity** — retrieve a validly-rooted chain and append a malicious segment that passes Verify Chain Integrity because the parent linkage is genuine. The damage is *unauthorized chain extension under a real originator's authority*, not disclosure.

**Principle:** going stateful created a second path to obtain the SCT (ask the store) in addition to receiving the header. That path MUST be at least as authenticated as receiving the header would have been. *Never let "knows the TraceID" be sufficient to retrieve or extend a chain.*

### 4.2 The SADAR SCT Key

The SCT stored in the session store is **encrypted at rest under a SADAR SCT Key** — a first-class SADAR key type whose **issuance scope is the deployer's choice**:

| Scope | Blast radius on key compromise |
|---|---|
| Platform-wide | All sessions |
| Per-flow | One flow's sessions |
| Per-user | One user's sessions |
| Per-session | One session |

The specification defines the key and its role; the customer selects the scope by issuance policy. This is the blast-radius control. The holder of the SADAR SCT Key is, by design, "the something that can read" — security comes from controlling *who holds it* and *at what scope*, not from trying to eliminate a readable key (something, somewhere, must be able to read).

### 4.3 Key binding and lookup authorization

- **Key binding.** The stored SCT references its SADAR SCT Key via a `kid` in cleartext metadata (standard JWE pattern), so an SAI knows which key decrypts a given entry and key rotation is survivable. This matches how the rest of SADAR resolves keys (JWKS, signed manifests).
- **Lookup authorization (the hijack fix).** The SADAR SCT Key provides confidentiality and decryption-authority; in addition, SAI MUST authorize the **caller** before serving or extending a stored chain — verifying the authenticated caller identity is the party entitled to continue this chain. The index (`intent_instance_id` / TraceID) is the **index**; the authenticated identity is the **authorization**. The key scopes *who can decrypt*; the authz check scopes *who SAI will decrypt for*.
- **Replay binding.** Two complementary mechanisms apply. At the SCT level, the **`cnf` (proof-of-possession) claim defined in Core Security §4.6** binds each SCT to the authentication credential presented on the invocation (DPoP key thumbprint, SPIFFE SVID, or mTLS client certificate), so a captured SCT cannot be replayed under a different caller — this is the primary anti-hijack binding and SADAR does not re-derive it here. At the store level, entries additionally carry the `jti + exp` seen-set (per 9. SCT Operations / RFC 9449 pattern) so a captured-and-replayed chain fails even if other layers were bypassed.

### 4.4 On double-encryption (documented option, not default)

A two-part scheme (encrypt under the SADAR SCT Key *and* under a platform-chosen second key) is possible but **not the default**. It forces every read and write to be a two-key rendezvous — an availability and coupling cost that reintroduces the cross-hop coupling statefulness was meant to avoid — while adding confidentiality only against the SADAR SCT Key issuer itself, a narrow threat. Layered encryption earns its keep only when the two layers defend against genuinely different, non-overlapping adversaries. Retain as a documented option for deployments that must distrust the SADAR SCT Key issuer; do not default to it.

### 4.5 Key derivation vs. propagation (resolved)

A naive scheme keys the store on `platform_secret + nonce + traceid` and propagates the nonce. This is circular: if the next SAI needs the nonce to decrypt and the nonce cannot ride the public wire, either it is propagated as a secret (the burden statefulness avoided) or it is derived. Two resolutions:

- **Derived** (`base_nonce = KDF(platform_key, traceid)`): nothing secret travels; each SAI recomputes locally. But this collapses blast radius back to the single platform key, since the nonce becomes a public function of one secret.
- **Per-session entropy from the SCT itself**: the genesis SCT root segment already carries fresh per-session entropy that propagates only to entitled parties (JWE-protected). Deriving the store key partly from a secret salt **inside the encrypted root payload** gives per-session scoping that rides the existing SCT propagation — no new wire secret, no circularity. (The root `jti` is unsuitable: it is cleartext metadata, visible to anyone who sees the chain. The salt must be in the encrypted payload so only a party that can decrypt the root can derive the key.)

This is the recommended direction: **the SCT is itself the per-session secret** — unique per session, genuinely secret (JWE-wrapped), and already propagated to the right parties by construction.

---

## 5. The Genesis Case (First Agent in a Flow)

At genesis, no inbound SCT exists; SAI **mints** the root rather than extending a chain.

1. The platform holds a registry ID and the originator's token.
2. Platform code calls a helper (the **trust-boundary injector**) to authenticate to SAI. The helper builds the root SCT segment (SCT0), signed with the platform's key, carrying the **originator as an asserted claim** injected via the trust boundary — agent application logic never has direct access to key material or constructs the SCT.
3. The call to SAI authenticates via OAuth2 (transport), with SCT0 in the **`SADAR-SCT`** header.
4. SAI appends the next segment for the dispatch it is about to make, **signed with the issuer identity's key** and **JWE-wrapped to the resolved downstream recipient** (known only after discovery/selection — not "Agent 1").

### 5.1 Identifier roles at genesis

Three identifiers do three jobs and MUST NOT be conflated:

| Identifier | Role | Scope |
|---|---|---|
| **Intent** | Seeds the SCT chain root; semantic identity of what is being accomplished | Stable across the whole flow |
| **Span ID** | One operation's segment of the trace | One operation per hop |
| **TraceID** | Correlation envelope / transaction instance | Whole flow |

Intent binds to the **SCT chain root jti**, *not* to a span ID. Fusing intent with span ID is an error: the span changes every hop while the intent does not.

### 5.2 Initiate Chain (genesis operation)

The four operations originally in 9. SCT Operations assume a prior chain (Append references a parent); root creation was not enumerated. A genesis-agent implementation invokes exactly that operation. **B.0 Initiate Chain** has been drafted for 9. SCT Operations (inputs: intent, asserted originator, signing identity, crypto suite; output: a single-segment chain whose Verify Chain Integrity returns VALID; the root is marked structurally by the absence of `parent_sct_jti`, with `segment_action` defaulting to `executed`). It references Core Security §4.6 `intent_instance_id` seeding so the two align.

---

## 6. The Signing Invariant

**Every SCT segment is signed with the key of the entity whose action that segment attests. SAI performs the signing mechanically, selecting the issuing identity per segment.**

| Segment / step | Actor | Signing key |
|---|---|---|
| Manifest (static, published) | Home registry | Registry key |
| Boundary crossing (GW routing) | Home registry (as boundary agent) | Registry key |
| Agent executes its function | The agent | Agent's key |
| Tool / resource acts | The tool / resource | Its key |
| SAI authority/dispatch segment | SAI / platform identity | SAI / platform key |

SAI is the signing *machinery*; the *identity* whose key it uses varies by step. No party ever signs for another — that is what preserves per-hop non-repudiation. "The step's key" means **the actor's key**, never universally one key.

### 6.1 The registry is a peer entry, not a root

The home registry's key is **not** a CA-style root that vouches for the keys of entities it homes. It is an **entry key** — exactly like an agent's, tool's, or resource's key. The registry has a manifest, an endpoint (its **GW**, see §7), and a key, the same as any other entry. Its key signs only the registry's own actions (its manifest and its routing segments), the same way an agent's key signs only the agent's actions. Therefore there is **no "root key on the hot path" concern**: the registry key's runtime exposure profile is identical to every other entry's key — used when, and only when, that entity acts. No delegation machinery is required.

---

## 7. Crossing Boundaries: The SAI Gateway

### 7.1 Endpoints

Three distinct endpoints, separated:

| Endpoint | Whose | Used by | Purpose |
|---|---|---|---|
| **Component auth endpoint** | The component (e.g., Y) | The component's **owning** SAI | Mint the component's token (stays in the owning environment) |
| **Component true/business endpoint** | The component (Y) | The owning SAI | Invoke the component's actual function |
| **GW (SAI) endpoint** | The **home registry** | A **calling** SAI | Where to hand the SCT + boundary auth; pinned to the registry |

The calling SAI only ever needs the **GW endpoint**. The component's true and auth endpoints are consumed only by the owning SAI, locally — which is *why the component's token never leaves its environment* (minted by the owning SAI at the component's auth endpoint, used at its true endpoint, both internal).

### 7.2 SAI endpoint pinned to the home registry

The SAI/GW endpoint is a **per-registry** property, not per-component. If you host a component, you hold its home-registry manifest (a federation-normative requirement); you resolve and inject **the home registry's GW endpoint**. One lookup, no per-agent SAI data. Consequence: **everything homed in a registry shares that registry's GW endpoint** — SAI is a per-environment *service tier*, not a per-agent sidecar.

**Local vs. remote** reduces to a single test: **is the target's home registry my home registry?** Registry identity is the trust-domain discriminator. There is no separate "is it federated" rule. A component homed at a remote URL but in *my* registry is local; a component in a *different* registry is remote — regardless of where it physically runs.

### 7.3 SAI as a role; GW as a specialized SAI

SAI is a **role**, realizable as an in-process library, a sidecar, or a service — interchangeable for local calls (the SCT never crosses a network boundary locally). The distinction only materializes at startup and when calling from a remote client.

**All calls are SAI-to-SAI** (every hop needs the SCT append + authentication, which only SAI does). Therefore local vs. remote is not an architectural fork — it is a **routing lookup**: "where is the next SAI?" When the target's GW endpoint resolves to my own SAI, the next SAI is me (local fast path, shared session store). When it differs, the next SAI is remote (wire transport).

The **GW** is a specialized SAI with **no discovery process**. Its only function is **routing**:

1. Terminate and authenticate the inbound boundary call (token audience = the registry's GW/SAI; mTLS).
2. **Validate** the inbound SCT chain against the caller's published keys (integrity verification needs no decryption — public keys, cleartext metadata).
3. **Look up**, in a **registry-local routing table**, which **internal** SAI Flow handles the target component. (This table is *private to the environment* — only the GW endpoint federates; internal SAI topology is never published.)
4. **Append a boundary segment**, signed with the **home registry's key** (the GW's own identity, per §6) — attesting "the home registry received this at its boundary and routed it internally." The segment carries an explicit **boundary action enum** (defined in 9. SCT Operations; seed value `ROUTED`, extensible) rather than leaving the action to be inferred. This is a true, registry-scoped claim; it does not claim to be the component.
5. **Hand off** to the internal SAI Flow.

### 7.4 The boundary segment is a real hop

Because a genuine trust boundary was crossed, the boundary segment is a real, signed hop — and local calls have none (no boundary crossed). This asymmetry is **truthful information**, not an inconsistency: the chain records "authority entered this registry's authority domain here." The registry-key segment *is* the boundary marker; the forensic crossing record falls out of the actor-signs invariant for free, with no special boundary logic.

Crucially, the recorded boundary is the **registry** boundary, not the hosting-location boundary. A component's home registry is the authority that vouches for its manifest and keys regardless of where the component runs, so crossing into the home registry's authority *is* the trust event.

### 7.5 GW → SAI Flow handoff

The GW hands off to the internal **SAI Flow**, which performs the genesis-in-environment work: lands the chain in the environment's session store (keyed by the shared TraceID, encrypted under *this* environment's SADAR SCT Key), resolves the target's true and auth endpoints locally, mints the target's token (which never leaves the environment), appends the dispatch segment (signed by the appropriate actor identity per §6), and invokes the target.

### 7.6 What crosses, what stays

| | Crosses A → B (on the wire) | Stays within an environment |
|---|---|---|
| `SADAR-SCT` chain (JWE-wrapped) | ✔ | |
| Boundary auth token (audience = B's registry GW) | ✔ | |
| TraceID (shared, for correlation) | ✔ | |
| Target component's token | | ✔ (minted by owning SAI Flow) |
| SADAR SCT Key | | ✔ (per-environment; store encryption is local) |
| Session store contents | | ✔ (each environment owns its own) |
| Internal SAI-Flow routing table | | ✔ (registry-local, non-federated) |

No store is shared across the boundary; no SADAR SCT Key crosses; no agent or framework touches the SCT. Each environment re-establishes its own session-store custody independently after receiving the chain on the wire.

### 7.7 Cross-boundary chain shape

```
… A-side segments (SAI-A, last one JWE-wrapped to B's GW)
   → B GW boundary segment        (signed: home-registry key; "received at boundary, routed to Y")
   → B Flow dispatch segment       (signed: appropriate actor; JWE-wrapped to Y)
   → Y execution segment           (signed: Y's key; "Y performed this")
   → …
```

B can **verify** the entire chain's integrity (public keys) and walk the `parent_sct_jti` linkage to know the chain's shape and root, but can **decrypt** only the segments wrapped to B-side recipients. A's internal segment *contents* remain confidential to A unless A wrapped them for B. The designed disclosure level: B learns provenance, integrity, and root/originator context, not A's internal segment payloads.

### 7.8 Conformance, Locality, Address-Space, and Synchronicity

**SAI is always the caller.** Whatever is invoked — a conformant agent, a non-conformant tool, a networked local service, a resource — the call is made by SAI. What varies is three *independent* axes, plus a synchronicity dimension. Holding them independent is what keeps the model free of special cases.

#### 7.8.1 Three independent axes

| Axis | Values | What it determines |
|---|---|---|
| **Conformance** | conformant / non-conformant | Whether the SCT is passed (conformant) or the call is straight with SAI recording the step (non-conformant) |
| **Locality** | local (same home registry) / remote (different registry) | Whether a boundary segment is produced (remote) or not (local) |
| **Address-space** | in-process / out-of-process / networked | Deployment only: lib call vs. sidecar vs. HTTP — no effect on chain semantics |

These do not collapse into one another. An out-of-address-space-but-**local** agent is reached by SAI over HTTP (networked deployment), is in *my* registry (local — no boundary segment), and may be conformant or not. SAI is invoked in every combination; only the three flags change what SAI does.

#### 7.8.2 Federation is the precondition for conformant remote calls

A SADAR-conformant **remote** call requires the target to be **federated** — i.e., to have a home-registry manifest. This is not an added constraint; it is a consequence of need: the GW endpoint, the recipient public key for JWE-wrap, and the signature-verification keys all come from that manifest. No manifest → no key to wrap to, no GW to route through, no boundary segment to verify. *Conformant remote* and *federated* are the same condition viewed two ways.

#### 7.8.3 The conformance NFR

`sadar_conformant` is an ordinary NFR, matched like any other. If either side declares it **mandatory**, a counterparty that does not also declare conformance produces an empty intersection and **NO_MATCH** — identical to a mandatory compliance-framework or repatriation NFR. There is no special enforcement model: the bootstrap fact that a non-conformant server cannot run the matching algorithm is not SADAR's concern, because a non-conformant server is not performing SADAR matching by definition.

#### 7.8.4 Non-conformant calls

A call MAY be made to a non-conformant agent, tool, or resource (subject to §7.8.3 matching). It works "like today" — a straight call with no SCT passed to the target — **but still via SAI**, and SAI **still records the step into the session store** so the next SAI picks up a chain that truthfully shows the non-conformant operation occurred. Continuity survives because SAI, always present, bridges the gap.

**Signer of the non-conformant segment: the home registry key.** The segment attests: *"here was an operation, defined in my registry, selected by the resolver, that I executed."* The actor is the **registry** — it defined the operation and its resolver selected it — so the registry's key signs, exactly as for the GW boundary segment (§7.3) and consistent with the §6 actor-signs invariant. The segment carries an explicit **non-conformant action enum** (defined in 9. SCT Operations; seed value `EXECUTED_NONCONFORMANT`, extensible). The segment does **not** claim the non-conformant target attested anything (it cannot).

**Audit consequence — signer identity plus action enum encode the conformance boundary.** A conformant hop carries the **component's own key** (the component attests its action). A non-conformant hop carries the **home registry's key** with the `EXECUTED_NONCONFORMANT` action enum. An auditor sees the signing identity flip from component to registry, disambiguated from a boundary-routing segment by the action enum (`ROUTED` vs `EXECUTED_NONCONFORMANT`) — so the two registry-signed segment kinds are structurally distinct, not merely distinguishable by prose. Key resolution is a local lookup: every replicated entry carries its **home-registry ID**, and federation replicates the home-registry manifest (with its key) into the host registry, so a verifier connected to the host registry reads the home-registry ID off the segment's entry and looks up the key locally.

**What is lost and retained.** A non-conformant hop gives up the target's own non-repudiation (it did not sign), its participation in confidentiality (no JWE wrapped to it), and any downstream SCT it would have propagated. What is retained is the registry's attestation that the operation was executed. The degradation is precise and bounded: lose the target-as-signer, keep the registry-as-witness.

#### 7.8.5 Resources are accessed via tools (deferred, known shape)

A resource is **non-conformant** and is **not** a direct call target. A resource is accessed **via a tool**, which may or may not be conformant. Like an SPD node (see Process Flow Specification, SPDComponents), a resource carries **criteria for discovering an appropriate tool**. Resource access therefore reduces to: discover a tool for the resource → call the tool (conformant or not, per §7.8.4) → the tool acts on the resource. Conformance applies to the **resolved tool**, not the resource. Full resource-access mechanics are deferred; the known shape is recorded here so the conformance model stays consistent.

#### 7.8.6 Synchronous vs. asynchronous

**Synchronous** is the base case: SAI calls, blocks, receives the result on the same channel, appends the outcome segment inline; the session store holds the chain across the brief in-flight window.

**Asynchronous** introduces a result that arrives later, on a different connection. Correlation and triggering of that result is a **platform concern** — the platform must do exactly what it would do without SADAR. SADAR does not mint, carry, expose, or define the platform's correlation mechanism.

**The correlation id — one value, two separate purposes.** The platform supplies a correlation id. It serves the platform's purpose (route the async result back, as it always would) and, separately, SADAR's purpose (tie the incoming result to the SCT chain that created the request, so the result is attributed and the chain continues). Same value; distinct purposes. SADAR **stores the platform-given correlation id on the dispatching SCT link** and, when the response arrives, looks up that link by the id and **continues the chain** (appends via `parent_sct_jti` — continuation, not a new chain).

**Platform obligation (the entire contract).** The platform's trigger-on-response **must call SAI** as a **DIRECT lookup** (the target is already known from the original dispatch — no discovery), and must pass the **correlation id on both** the original request SAI call (so SADAR records it on the link) **and** the triggered response SAI call (so SADAR finds the link). How the platform generates, carries, and matches that id is entirely the platform's business. SADAR cannot fetch the id from the platform — it is given to SAI in the call, never retrieved.

**Lookup key.** SADAR keys the async lookup on the composite **`(intent_instance_id, correlation_id)`**. Per Core Security §4.6, `intent_instance_id` — seeded from the root TraceID at flow start, propagated by value, immutable end-to-end and identical across trust boundaries — is the stable join key; the **local TraceID may legitimately differ in a remote segment** and SHALL NOT be relied upon for cross-boundary correlation. The composite supports multiple concurrent async dispatches in one flow (each bound to its own SCT link) and prevents two flows that coincidentally reuse a correlation-id value from intertwining at the SADAR layer. The correlation id and `intent_instance_id` are separate concepts; a platform MAY use equal values, but SADAR always treats them as a tuple.

**Access and change rights.** The correlation id, once held within the SCT, is **not accessible through SADAR** to the platform or any other party — it exists for internal SADAR continuation and audit purposes only. The platform's own copy of the id is the platform's; SADAR's stored copy is sealed. SADAR reserves the right to change the internal mechanics of how it holds and uses the id. *A non-conformant layer must not depend on, or piggyback on, the SADAR-internal correlation id; doing so would create a semi-conformant dependency outside any conformance contract that SADAR-internal changes could silently break.*

**Conformant async.** A conformant async result is itself a SADAR invocation carrying `parent_sct_jti`; it continues the chain by the SCT linkage directly. The platform correlation id ties the platform's delivery to the dispatch; the `parent_sct_jti` ties the SCT continuation — both point at the same chain.

**Non-conformant async.** The non-conformant target knows nothing of SADAR and handles its own correlation separately. The platform's correlation mechanism routes the result back and the platform's trigger calls SAI with the correlation id; SAI continues the chain with a **registry-signed** segment (per §7.8.4). The non-conformant target never receives or echoes a SADAR construct.

**The one firm requirement, regardless of conformance:** the async result MUST be **processed through SAI** to rejoin the chain. SADAR specifies only the lookup-and-continue and the two-place correlation-id obligation; the platform specifies everything about how the result is correlated and delivered.

**Store-lifetime consequence.** The session store entry and the `jti + exp` replay set (§4.3) must persist for the **async window**, not merely the synchronous in-flight window — bounded by `exp`, not unbounded. Async windows that exceed practical store TTLs are a deployment constraint to size for.

*(Deferred thought: whether an async response trigger might legitimately invoke discovery rather than DIRECT lookup is an interesting case to revisit later; for now the response path is DIRECT lookup only.)*

---

## 8. Chain Lifecycle and Context Propagation

### 8.0 Framing principle — SADAR mirrors OpenTelemetry's patterns

SADAR's chain-of-custody propagation **deliberately mirrors the architectural design patterns of OpenTelemetry (OTel)**. It adopts OTel's interval-bookend structure, its ambient in-process context propagation, and its inject/extract wire-propagation model — and therefore inherits OTel's **capabilities** (polyglot propagation, parent/child parenting, fan-out via copy-on-fork) and its **limitations** (context drops at un-instrumented thread-of-control forks, degrading to recorded orphan segments).

SADAR **diverges** from OTel in exactly the dimensions that distinguish *authority* from *observability*:

1. **Signed.** Each SCT segment is signed by its actor (non-repudiation); spans are unsigned.
2. **Encrypted per recipient.** Segments are JWE-wrapped to their recipient (confidentiality); spans are not.
3. **SAI-scoped.** SADAR propagates at **SAI boundaries**, not at every span. The point-sets overlap but are not identical — SADAR's are sparser and different.
4. **Independent.** SADAR has **no functional dependency on OTel** — it uses the same *kind* of context mechanism as a *separate instance*, never OTel's own channel, and functions with OTel absent.

> The mirror is of **patterns, not plumbing.** Implementers may reason about SADAR propagation using their OTel mental model, while understanding that the authority lane is signed, encrypted, SAI-scoped, and OTel-independent. The corollary for operations: *wherever your traces are clean, your chains are clean* — and wherever a trace breaks (un-instrumented fork), the chain breaks in the same place.

### 8.1 The bookend model

Each SAI invocation produces **two** SCT appends, not one:

- An **open bookend** at **dispatch** — created when SAI is about to call a target. It mints the call's `jti`, records "calling X", and carries `parent_sct_jti` linking to the caller. It exists for the entire in-flight window.
- A **close bookend** at **return or timeout** — references the open bookend's `jti` and carries the outcome (`step_status`, final risk adjustment, return time).

A single-append model (write only on return) is rejected: it leaves no record while a call is in flight, gives children nothing to parent against during a long-running or async call, and cannot attribute a timeout. The open bookend at dispatch is what provides in-flight visibility, the parent reference for children, and the anchor a late async result reconnects to.

`step_status` (Core Security §4.6) is a **close-bookend** claim — the open bookend cannot know an outcome that has not happened. Therefore an **open bookend with no matching close** is precisely the signal of an in-flight or timed-out call. A TTL/timeout sweeper synthesizes a close bookend (`step_status` = failure/timeout) when an open bookend's window expires with no return.

The close bookend is a **separate, immutable append**, not an in-place update of the open bookend — consistent with the SCT chain's append-only nature. "Find in-flight/timed-out calls" is a scan for open bookends whose `jti` has no matching close.

### 8.2 Structural mirror of OTel spans

The bookend pair is the authority-lane mirror of an OTel span's start/end. The topology is identical:

| OTel span | SCT bookend pair |
|---|---|
| span start | open bookend (dispatch) |
| span end | close bookend (return/timeout) |
| span id | the call's `jti` |
| parent span id | `parent_sct_jti` |
| trace id | the flow handle / `intent_instance_id` |
| span status | `step_status` (on close) |
| unclosed span (process died) | open bookend with no close |
| nested spans = call tree | nested `jti`s = the chain tree |
| sibling spans under one parent (fan-out) | sibling open-bookends under one parent `jti` |

**Do not collapse the two lanes.** The shapes match; the trust does not. The span tree records *what happened* (unsigned, observability); only the SCT records *under whose authority, signed*. The SCT MUST NOT be re-derived from the trace, and the `jti` is **SADAR-minted, not the span id** — a span id is OTel-controlled and (cross-process) wire-exposed; binding it into the signed chain would import a non-authoritative, externally-influenced value into the authority lane. The `jti` *corresponds* to a span (same interval) but is its own identifier; the two MAY be cross-referenced for audit joins, never equated.

### 8.3 The composite key

A flow's persisted SCT links are keyed by four fields, each defeating a collision the others cannot:

| Field | Defeats |
|---|---|
| **flow handle** (in-environment) / **`intent_instance_id`** (cross-environment) | cross-flow collision |
| **calling step** (the caller's component id) | same flow, different caller |
| **current step** (the called component id) | same caller, different target |
| **call `jti`** (per-invocation, the open bookend's id) | same caller, same target, **multiple or iterated invocations** |

The fourth field is essential: a step may launch the *same* target multiple times (parallel multi-call) or iterate over it (loop); those calls share the first three fields and are separated only by a per-invocation discriminator. A **UUID/`jti` is used, never a timestamp** — timestamps are neither guaranteed unique nor monotonic under concurrency. The call's open-bookend `jti` already provides this unique per-invocation id, so it serves as the fourth key directly.

The first field is the flow's stable identity: a **handle** within an environment (§8.6) or **`intent_instance_id`** across environments (the local TraceID may differ in a remote segment, per Core Security §4.6).

### 8.4 Parenting and fan-out via ambient context

A step does not "discover" which chain it belongs to; it **inherits** its parent from the **ambient context** at the moment its segment is created — exactly as an OTel child span reads the current span from the propagation context.

- **The open bookend's `jti` is the parent reference.** Because the open bookend is minted at **dispatch** (before the callee runs), the predecessor `jti` a child needs *already exists* when the child calls SAI. This is a second reason bookends open at dispatch (the first being in-flight visibility): the dispatch-time `jti` is the parenting anchor.
- **Ambient carrier.** The current SCT `jti` rides the host language's context carrier — the *same kind* OTel uses, a *separate instance* (§8.5). A step calling SAI reads the current `jti` from that carrier as its predecessor; SAI mints the child's new `jti` and sets it as current.
- **Fan-out = sibling subtrees, no collision.** At a fork, the context is **copied per branch** (copy-on-fork). Each parallel branch sees the *same* parent `jti` (read from its copy) and sets its *own* new `jti` as current *in its own copy*. N parallel branches all parent to the fork's open bookend and diverge by their own `jti`s — identical to OTel sibling spans, and exactly the per-branch isolation the composite key (§8.3) requires.

### 8.5 SADAR-parallel context — no OTel dependency

SADAR maintains its **own** context, carried by the host language's context primitive — the same *kind* OTel uses, a **separate instance** holding the current SCT `jti`:

| Language | Context carrier (SADAR uses its own instance of the same kind) |
|---|---|
| Python | `contextvars.ContextVar` |
| Java | `ThreadLocal` / `Context` + `Scope` |
| Go | explicit `context.Context` (no ambient storage; threaded by hand) |
| Node / JS | `AsyncLocalStorage` |
| .NET | `AsyncLocal<T>` |
| Rust | async-runtime task-locals / explicit passing |

SADAR has **no functional dependency on OpenTelemetry**: it neither reads OTel's context nor writes OTel's baggage, and functions with OTel absent. (Dropping baggage as a transport, earlier in this document, removed the last functional tie to OTel; this completes the separation.) Implementations **MAY** hook the same lifecycle points an OTel integration uses (task spawn, pool submit, request send/receive) to *trigger* SADAR context propagation, but SADAR's correctness **SHALL NOT** depend on OTel being deployed.

The wire transport is language-independent (the SCT Header, like W3C `traceparent`); only the in-process carrier is per-language.

### 8.6 Propagation regimes and the two deployment/trust modes

**SADAR sets and reads its context at SAI boundaries — which SAI owns.** The propagation points are SAI's own call sites, not OTel's spans (the two sets are not the same). Two regimes:

- **In-process (ambient).** When SAI runs in the caller's thread of control, the current `jti` is ambient — the next SAI call reads it for free. **SAI owns its own forks:** when SAI dispatches to an internal target on a different thread, SAI performs the context copy itself (`copy_context()` / `ctx.run()` or the language equivalent), so its own dispatches never drop context.
- **Across a boundary (explicit).** When the call crosses a process or environment boundary, the reference rides the **SCT Header** on the authenticated invocation request; the **receiving SAI extracts it and re-establishes the local context carrier** (inject on send, extract-and-establish on receive — the OTel symmetry). After extraction the receiver's internal flow is back in the ambient regime.

**The one residual, and it is fail-soft.** If an *agent* (not SAI) forks a thread of control that does **not** carry the SADAR context and then calls SAI from it, SAI reads an empty context and records an **orphan segment** — unparented, but still recorded and visible in the logs. This is not data loss; it is graceful degradation, and it is the identical failure point where an OTel span would orphan. Agent-internal concurrency that does **not** call SAI is correctly **out of scope** (no SAI boundary, nothing to record). Framework propagation of SADAR context across the framework's *own* forks is therefore an **optional quality-of-lineage improvement**, not a correctness obligation SADAR imposes.

**Two deployment/trust modes** (the choice between them *is* a security decision):

- **In-address-space singleton.** A singleton SAI object, bootstrapped by the framework, generating an internal UUID handle. Every SAI call resolves to the same object — provided as an in-memory MCP server *and/or* a direct tool, both redirecting to the singleton. Non-in-memory SAI calls are disallowed **within** an environment (no mixed model); the singleton may still call out to non-in-memory agents/tools/resources, and may hand off to a remote SAI at a registry GW (§7). The SADAR key is held module-private (encapsulation against honest access). **Trust premise: all code in the address space is trusted** — there is no memory isolation from malicious in-process code, and encapsulation is a visibility convention, not a memory barrier. A single-process singleton needs no external store for the happy path; the in-memory map *is* the state.
- **Separate-service SAI.** SAI runs as its own service behind a well-known API endpoint; the bootstrap call returns a handle (UUID) the client uses on every subsequent call. Under the covers, stateless SAIs share a persistence layer (§3). This gives **OS-enforced process isolation** — the SADAR key lives in SAI's address space, unreachable by agent code; lookup keys cross to the endpoint over an authenticated, encrypted call. This is the mode for **untrusted agent code**: the protection the singleton mode cannot provide in-process is provided here by the process boundary.

The flow handle (singleton UUID, or endpoint-issued UUID) **replaces the first field of the composite key (§8.3)** within an environment; cross-environment, `intent_instance_id` is the stable join key, re-established on the far side by the GW (§7).

### 8.7 Context-pointer confidentiality (parked)

Whether to encrypt the in-process context pointer is **parked**. The reasoning recorded for resumption: in one address space, code that can read a contextvar can also reach the SADAR key (heap walk, closure introspection, monkeypatch), so encrypting the pointer protects nothing against the only adversary it would face — in-process malicious code — while honest code reads neither. Encryption becomes meaningful only where a real boundary exists: across the wire (handled by the authenticated SAI call / SCT Header) and at rest in a shared store (handled by the SADAR SCT Key, §4). The path to protecting the key *from agents* is therefore the **separate-service mode** (OS isolation), not contextvar encryption. The persistence-layer encryption of §4 is unaffected and remains in force.

---

## 9. Open Decisions

| # | Decision | Status |
|---|---|---|
| 1 | **Header name** | **`SADAR-SCT`** (locked this session; supersedes the `SADAR-Context` placeholder in the transport-canonical note). Apply identically in §4.6, the Overview, and any wire-format/conformance text. |
| 2 | **Operations vocabulary** | Two distinct layers, kept and labeled separately: the **crypto/chain operations** (Validate, Extract Claims, Append Segment, Verify Chain Integrity — the four in 9. SCT Operations) vs. the **lifecycle operations** (Open, Continue, Hold, Authoritative Carry, Close — referenced in 10. Trust Models). The Context Token orientation page's "five vs four" self-contradiction is a symptom of conflating these; reconcile by labeling the layers explicitly. |
| 3 | **Initiate Chain** | *Drafted.* Genesis root-creation added as **B.0 Initiate Chain** in 9. SCT Operations (root has no `parent_sct_jti`; `segment_action` default `executed`; references §4.6 `intent_instance_id` seeding). Pending final insertion/commit. |
| 4 | **Session-store key derivation** | Recommended: derive the store key partly from a **secret salt inside the encrypted SCT root payload** (per-session entropy that already propagates to entitled parties), not from the cleartext root `jti` and not solely from a platform-wide key (§4.5). **Parked** this session pending the encryption decision (§10.7). |
| 5 | **Boundary / non-conformant action enums** | *Resolved.* Two extensible enums, defined normatively in 9. SCT Operations and referenced here: boundary action (seed `ROUTED`) and non-conformant action (seed `EXECUTED_NONCONFORMANT`). Minimal now; async variants added when that section firms up. |
| 6 | **`sadar_conformant` NFR** | Define the field formally (URN, requester/server placement) as an ordinary mandatory-capable NFR, matched like compliance-framework / repatriation. NO_MATCH on mismatch (§7.8.3). |
| 7 | **Async correlation contract** | *Resolved.* Platform owns correlation entirely (does what it would without SADAR); its only obligation is to pass one correlation id on both the request and response SAI calls (DIRECT lookup). SADAR stores the id on the dispatching SCT link, keys lookup on **`(intent_instance_id, correlation_id)`** (TraceID may differ in a remote segment), continues via `parent_sct_jti`. The id is sealed within SADAR (internal + audit only); non-conformant parties must not piggyback on it (§7.8.6). |
| 8 | **Resource-via-tool** | Resource access reduces to tool discovery via resource-borne criteria; conformance applies to the resolved tool (§7.8.5). Full mechanics deferred — confirm the known shape is correct before deferring. |
| 9 | **Context-propagation dependency** | *Resolved.* SADAR maintains its **own** context (same *kind* of carrier as OTel — `contextvars`/`ThreadLocal`/`context.Context`/`AsyncLocalStorage` — a *separate* instance), with **no functional dependency on OTel**. SADAR neither reads OTel's context nor writes OTel's baggage (§10.5). |
| 10 | **Demo persistence durability** | Open. A single-process in-memory singleton needs no store for the happy path; persistence returns only for durability (async-across-restart, crash-proof audit). Confirm whether the demo must survive async-across-restart (which pulls persistence back in) or may accept in-memory-only with a stated limitation (§10.6). |
| 11 | **Context-pointer encryption** | *Parked.* In-process contextvars are trusted with the address space (§10.7); encrypting them adds nothing against in-process code that can also reach the key. Cross-boundary the pointer rides the authenticated `SADAR-SCT`/SAI call. Revisit only if a deployment requires untrusted code in SAI's address space (→ separate-service mode). |

---

## 10. Reconciliation Notes (for the SCT document set)

These edits, identified during this design thread, should be applied to the existing documents:

- **SCT Structure** — *Created this session.* The canonical normative segment schema (envelope, claim table with carriage, enumerations including `segment_action`, required-claim-by-kind matrix). Consolidates Core Security §4.6 into formalized tables; §4.6 should be reduced to an orientation pointer to it (Open Decision in SCT Structure §9). Pending commit to `docs/spec/Security/SCT/SCT_Structure.md`.
- **Core Security §4.6 (SCT structure)** — *Applied in v0.4.1.* Named the transport header **`SADAR-SCT`** (was "dedicated SCT HTTP header"), with the "alongside the OIDC usage token, both required" phrasing. Added the **`segment_action`** claim valued from `urn:sadar:segment_action:v1:*` (`executed`, `routed`, `executed_nonconformant`; extensible). §4.6 already correctly says "JWS-inside-JWE." *Next:* reduce §4.6 to an orientation intro + pointer to SCT Structure, to avoid schema drift.
- **9. SCT Operations** — needs the new **B.0 Initiate Chain** operation (genesis root creation; the existing four operations all assume a prior chain — Append requires a parent). Should reference §4.6's `intent_instance_id` root-seeding (`startFlow`) so the two align. *Pending.* (The header name and enums do NOT belong here — they are §4.6 structure content, now applied there.)
- **Context Token orientation page** — add the canonical transport block (§2.1 here); resolve the "five chain operations" vs "four abstract operations" self-contradiction per Open Decision 2; it already uses "SADAR Context Token" and "JWS-inside-JWE" correctly.
- **10. Trust Models** — already uses "SADAR Context Token" correctly (the earlier "Signed Context Token" concern is resolved in the live file). Its Open/Continue/Hold/Authoritative Carry/Close set is the **lifecycle** layer per Open Decision 2.
- **Specification Overview v3 (public PDF)** and **coalition-era material** — replace "OIDC claims extension" transport phrasing with the canonical block; narrow the OIDC-claim role to originator-propagation *seeding* only. *(Not in the repo; tracked here for the editor.)*

*Terminology lock:* SCT = **SADAR Context Token** (never "Signed Context Token"); structure = **JWS-inside-JWE**; transport = **`SADAR-SCT` header** alongside the usage token, both required per request; OIDC claim extension role = **originator-propagation seeding only**.
