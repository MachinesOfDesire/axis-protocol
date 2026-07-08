# AXIS Protocol Specification

**Version:** 0.3.1
**Status:** Public release. This is the canonical AXIS Protocol v0.3 specification. Breaking changes are possible before v1.0; see §17 Roadmap and `CHANGELOG.md` for the versioning policy. Version-mismatch handling guidance for implementers lives in [COMPATIBILITY.md](./COMPATIBILITY.md).
**Author:** Josh Ashcroft
**License:** Apache 2.0
**Repository:** https://github.com/MachinesOfDesire/axis-protocol
**Last updated:** 2026-07-07

## Abstract

AXIS (Agent Cross-system Identity Standard) is a protocol for autonomous AI agent identity, delegation, and authorization across operator boundaries. This document specifies the protocol's data model, API contract, verification procedures, and conformance criteria. v0.3 is additive over v0.2: it introduces sender-constrained AITs (proof-of-possession via `cnf.jkt` + DPoP), ephemeral within-runtime sub-agent delegates, a standard scope vocabulary layered over the v0.2 grammar, a CA-trust registry-legitimacy model (`/.well-known/axis-registry` + `/.well-known/axis-directory`), the `/.well-known/axis-scopes` discovery manifest, an optional inline challenge-on-refusal response, and the formal `did:axis` method. Every v0.3 mechanism degrades gracefully against v0.2 verifiers; the one place a v0.3 verifier MUST reject a v0.2 presentation is when the platform advertises `proof_of_possession: "required"` (see §4.3). v0.2 itself was additive over v0.1.1 with one behavioral change: the `aud` claim on AITs became REQUIRED (see §4.3).

AXIS addresses the agent accountability problem: when an autonomous AI agent takes a consequential action, third parties — receiving platforms, auditors, courts, regulators — must be able to verify who authorized that action, what scope they authorized, and whether that authority is still in force. Today's identity tooling does not produce the evidence regulatory frameworks and live incidents demand. AXIS provides that evidence as a first-class protocol artifact: a cryptographically signed, self-contained credential chain that any third party can verify locally, without prior relationship to the agent's operator.

The defining features are a lean credential surface (identity token plus optional delegation chain), a self-locating registry model where credentials carry the pointer to the registry that issued them, and monotonic delegation attenuation with a root-operator invariant. Credential chain validation is performed locally by the verifier; the only network dependencies are standard lookups against the agent's self-declared registry (public key retrieval and revocation checks), and that registry may be any AXIS-compliant implementation.

The reference implementation is AXIS Prime at `registry.axisprime.ai`. The protocol itself is open; any party may implement a compliant registry.

## 0. Motivation

Autonomous AI agents are taking consequential actions across organizational boundaries — producing content, making commitments, accessing regulated data, executing transactions — at a rate that outpaces the identity infrastructure supporting them. When an agent acts, three questions need clean answers, and today none of them have clean mechanisms:

1. **Who authorized this agent to act?** A chain of authority back to a human operator.
2. **What scope were they authorized for?** What the agent was, and was not, permitted to do.
3. **Can we prove it later, to an auditor or a court?** Tamper-evident evidence that survives the agent, the platform, and organizational memory.

Organizations are being held liable for agent actions they cannot later explain — their agent made a commitment, the organization is bound by it, and no one can produce the chain of authority that authorized it. Regulatory frameworks increasingly assume this evidence can be produced: EU AI Act Article 12 requires automatic event recording for high-risk AI systems; HIPAA 45 CFR 164.312(b) requires audit controls over electronic Protected Health Information access; SOC 2 requires evidence of authorization for Trust Services Criteria controls. Agent actions fall inside all of these frameworks, and today's identity tooling — API keys, session tokens, OAuth scopes — does not produce the right kind of evidence.

AXIS addresses this gap at the protocol layer. It defines portable, cryptographically verifiable agent identities; delegation credentials that encode scope and time constraints; revocation with known propagation semantics; and a verification procedure that produces a tamper-evident artifact at the time of action — the evidence that regulatory frameworks, auditors, and courts can reason about after the fact.

## 1. Requirements language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119] and [RFC 8174] when, and only when, they appear in all capitals.

## 2. Terminology

**Agent** — An autonomous software system that takes actions on behalf of a human operator or another agent. Every agent has a persistent cryptographic identity (an Ed25519 keypair). Identity is bound to the keypair, not to the underlying model, memory, or runtime: an agent may change models, memory stores, or execution environments while retaining its AXIS identity as long as its keypair remains under the operator's control.

**Operator** — A human or organization controlling one or more agents. The operator is the root of every delegation chain.

**Verification tier** — A property of the Operator Identity Record indicating how thoroughly the registrar verified the operator's identity. One of `email`, `domain`, `kyb_individual`, `kyb_business`. The tier is NOT a field in delegation credentials; it is observable by resolving the operator record.

**Registry** — Backend infrastructure that stores identity records and credential chains. Defined by its API contract, not by any specific implementation.

**Registrar** — A consumer-facing service that handles registration, verification, pricing, and support on behalf of end users. Registrars write to a registry.

**Agent Identity Record (AIR)** — A persistent, cryptographically verifiable record of an agent's existence, public key, and status.

**Operator Identity Record (OIR)** — The equivalent for an operator.

**AXIS Identity Token (AIT)** — A signed JWT that an agent presents to prove its identity.

**Delegation Credential (DC)** — A signed document authorizing one party to act on behalf of another, with scope and time constraints.

**Trust Attestation (TA)** — A signed reputation statement about an agent or operator. Distinct from authorization.

**Content Provenance Attestation (CPA)** — A signed record binding specific content to the agent, delegation, and reviewer that produced it.

**Verifier (Relying Party)** — Any party receiving a request from an agent that needs to confirm the agent's identity and authorization. Verifiers MAY be platforms, services, or other agents.

**Attenuation** — The property that each link in a delegation chain can only narrow (never widen) the scope of the parent link.

**Root operator** — The operator at the top of a delegation chain. The `root_operator` field MUST be byte-for-byte identical across every credential in the chain.

## 3. Architecture overview

AXIS defines three credential types in three layers:

1. **Identity (Layer 1)** — AIR + AIT. Answers "who is this agent?"
2. **Authorization (Layer 2)** — Delegation Credential. Answers "what is this agent allowed to do, and who said so?"
3. **Reputation (Layer 3)** — Trust Attestation + Content Provenance Attestation. Answers "how has this agent performed?"

Layers 1 and 2 are mandatory for any verification. Layer 3 is advisory.

Verification is stateless from the registry's perspective: the receiving platform does all credential-chain validation locally, calling the registry only to fetch public keys and check revocation status. Credential chains are self-contained bundles; the receiving platform does not chase individual links.

## 4. Data model

All records are JSON. Field names are lowercase with underscores. Timestamps are ISO 8601 in UTC. Base64 encoding is URL-safe (base64url, RFC 4648 §5) for all cryptographic material.

### 4.1 Agent Identity Record (AIR)

```json
{
  "axis_version": "0.2",
  "agent_id": "axis:widget-corp:editor",
  "did": "did:axis:prime:widget-corp:editor",
  "operator_id": "widget-corp",
  "public_key": "lDwxSH896YH5IlqxAHaZmKFAI-32qIiLBTdTPOcTVCE",
  "key_algorithm": "Ed25519",
  "status": "active",
  "registry_url": "https://registry.axisprime.ai/agents/axis:widget-corp:editor",
  "revocation_url": "https://registry.axisprime.ai/revocation/axis:widget-corp:editor"
}
```

**Required fields:**

- `axis_version` (string) — protocol version, e.g. `"0.1"`
- `agent_id` (string) — globally unique, format `axis:{operator}:{agent-name}`
- `operator_id` (string) — the operator controlling this agent
- `public_key` (string) — Ed25519 public key, base64url-encoded
- `key_algorithm` (string) — MUST be `"Ed25519"` for v0.1
- `status` (string) — one of: `"active"`, `"suspended"`, `"revoked"`
- `registry_url` (string) — absolute URL where this record can be resolved
- `revocation_url` (string) — absolute URL where revocation status can be checked

**Optional fields** (presentation layer; see §5):

Optional fields enable presentation-layer functionality (display name, purpose, capability signals) and interoperability with W3C DID tooling. They are not required for cryptographic verification — a minimal AIR with only the required fields above is sufficient for the core verification procedure in §8.

- `did` (string) — W3C DID form. v0.2 canonical shape is `did:axis:{registry}:{operator-slug}:{agent-slug}` (operator-namespaced; see §9 for the v0.2 change). v0.1 form `did:axis:{registry}:{agent-slug}` remains resolvable; registries SHOULD return both forms during the transition window. Registries MUST return the v0.2 form for newly-registered agents.
- `display_name` (string) — human-readable name
- `platform` (string) — agent runtime identifier
- `purpose` (string) — plain-language description of the agent's function
- `capability_tier` (integer, 1-3) — advisory capability signal

### 4.2 Operator Identity Record (OIR)

```json
{
  "axis_version": "0.1",
  "operator_id": "widget-corp",
  "public_key": "eBjj90caoNPHtKqCAn4OvDJ8_s03HT5iSYyEXZvkaTA",
  "key_algorithm": "Ed25519",
  "status": "active",
  "registry_url": "https://registry.axisprime.ai/operators/widget-corp",
  "revocation_url": "https://registry.axisprime.ai/revocation/operator/widget-corp"
}
```

**Required fields:**

- `axis_version` (string)
- `operator_id` (string) — the operator's slug (globally unique within a registry, or globally unique via domain verification at Tier 1)
- `public_key` (string) — Ed25519 public key used for signing delegation credentials
- `key_algorithm` (string) — MUST be `"Ed25519"` for v0.1
- `status` (string) — one of: `"active"`, `"suspended"`, `"deactivated"`
- `registry_url` (string)
- `revocation_url` (string)

**Optional fields** (presentation layer; see §5):

- `display_name` (string)
- `verification_tier` (string) — one of `"email"`, `"domain"`, `"kyb_individual"`, `"kyb_business"`
- `domain` (string) — verified domain (when `verification_tier` is `"domain"` or higher)
- `registered_at` (string, ISO 8601)

### 4.3 AXIS Identity Token (AIT)

The AIT is a JWT per [RFC 7519], signed using EdDSA per [RFC 8037].

**Header:**
```json
{
  "alg": "EdDSA",
  "typ": "AIT",
  "kid": "axis:widget-corp:editor"
}
```

**Payload:**
```json
{
  "iss": "axis:widget-corp:editor",
  "aud": "ghost.example.com",
  "iat": 1743292800,
  "exp": 1743379200,
  "jti": "8f3a1c2e-9b47-4d15-a02b-6c1e5f88d2a1",
  "dlg": "dc:widget-corp:editor-2026-04"
}
```

**Required claims:**

- `iss` — issuer (the agent's AXIS ID)
- `aud` — **(REQUIRED in v0.2)** the platform identifier the token is intended for. Verifiers MUST reject tokens whose `aud` does not match their own identifier (advertised in `/.well-known/axis-access` under `audience`; see §6.12). The reference registry's `/verify` endpoint additionally rejects any AIT whose `aud` is missing or empty (the registry cannot know each platform's audience, so it enforces presence; the value match is the platform's job). The `aud` requirement closes the cross-platform replay class: a token minted for platform A cannot be presented to platform B. v0.1 verifiers continue to accept tokens without `aud` for backward compatibility; v0.2 verifiers MUST reject.
- `iat` — issued at (Unix timestamp)
- `exp` — expiration (Unix timestamp)

**Optional claims:**

- `sub` — subject. When present, MUST equal `iss` (an AIT always speaks for the agent that signed it). Verifiers accept `iss` alone; the reference registry resolves the agent from `iss`, falling back to `sub`.
- `jti` — unique token identifier, RECOMMENDED. Platforms MAY treat AITs as single-use by recording consumed `jti` values until `exp` and rejecting re-presentation. The reference platform implementations REQUIRE `jti` and enforce single-use (see §11.3); issuers SHOULD always set it.
- `dlg` (string) — `credential_id` of a Delegation Credential the agent is acting under. Verifiers resolve the chain (see §6.9.1, `GET /delegations/:id/chain`, which accepts the `dlg` value directly and pins the verdict to that credential) and use the chain's effective scope for authorization. Absent means no specific delegation is claimed; the agent is asserting its own native scope. Pointing at a record by ID rather than embedding the credential keeps the claim shape stable as the delegation envelope evolves; v0.3+ multi-signature delegation can ship without changing the `dlg` shape.
- `operator_id` — the agent's operator. ADVISORY only: verifiers MUST treat the registry's agent record as authoritative for the operator linkage (the reference registry returns the operator resolved from the AIR, ignoring this claim). Issuers MAY omit it.
- `registry_url` — URL where the agent's AIR can be resolved. ADVISORY convenience for verifiers that have not yet resolved the agent; subject to the registry-legitimacy check (§8 Step 1) like any self-declared registry pointer.
- `axis_version` — protocol version signaling. **Specified for v0.4, not yet shipped:** current issuers (including the reference SDK) do not emit this claim, and shipped verifiers do not check it. See §4.3.2.
- `cnf` — proof-of-possession confirmation (v0.3, §4.3.1). Specified, not yet shipped in the reference implementations.

Signed with the agent's private key. Verifiers resolve the agent's AIR (via `registry_url` when present, otherwise via the agent's declared registry) to retrieve the public key and verify the signature.

**Presentation channels.** An agent presents an AIT to a platform through one of:

1. `X-AXIS-Token: <AIT>` request header — the channel shipped and documented by the reference platform implementations, and the RECOMMENDED default. Keeping the AIT out of `Authorization` leaves that header free for the platform's own operator/user sessions.
2. `Authorization: Bearer <AIT>` — accepted by the reference platform SDK (checked before `X-AXIS-Token`).
3. `?ait=<AIT>` query parameter — accepted as a last resort; NOT RECOMMENDED (URLs leak into logs).
4. `Authorization: DPoP <AIT>` + `DPoP` proof header — the v0.3 sender-constrained form (§4.3.1); specified, not yet shipped.

Platforms SHOULD advertise their accepted channel in `/.well-known/axis-access` (the reference commenting platform does so in its `actions` manifest, §7).

**Token lifetime.** AITs SHOULD have a maximum lifetime of 24 hours. Platforms MAY enforce shorter lifetimes, and production verifiers are encouraged to: the reference SDK mints 5-minute tokens by default, and shipped platform verifiers apply a configurable max-TTL ceiling. Agents re-mint AITs from their private key as needed.

#### 4.3.1 Sender-constrained AITs — `cnf.jkt` and DPoP (v0.3)

The v0.2 AIT is a bearer JWT: verification (§8 Step 1) confirms the signature was valid at signing time but does NOT confirm the presenter has live control of the agent's private key at request time. v0.2's REQUIRED `aud` closes cross-platform replay (a token minted for platform A cannot be presented to platform B); it does NOT close the same-platform leaked-AIT replay class. AITs leak in places private keys do not: proxy/WAF/CDN logs, the receiving platform's own storage, browser memory (XSS), crash dumps, and support traces. Any capture yields a working agent identity until `exp`.

v0.3 closes this by binding the AIT to a proof of possession of the agent's key, following [RFC 7800] (PoP key semantics for JWTs) and [RFC 9449] (OAuth 2.0 DPoP). No new credential type, no keypair rotation: the binding is derived from the agent's existing Ed25519 keypair.

**`cnf` claim with `jkt`.** v0.3 AITs carry a confirmation claim whose `jkt` is the [RFC 7638] JWK thumbprint of the agent's public key — the same key that verifies the AIT signature and is published in the AIR:

```jsonc
"cnf": {
  "jkt": "<base64url SHA-256 JWK thumbprint per RFC 7638>"
}
```

**DPoP proof header.** The agent presents the AIT with a per-request proof JWT:

```
Authorization: DPoP <AIT>
DPoP: <proof-JWT>
```

The proof JWT carries the holder's public key inline so the verifier needs no extra lookup:

```jsonc
// Header
{ "typ": "dpop+jwt", "alg": "EdDSA",
  "jwk": { "kty": "OKP", "crv": "Ed25519", "x": "<base64url public key>" } }
// Payload
{ "jti": "<unique per request, [A-Za-z0-9_-]+>",
  "htm": "POST",                              // HTTP method, uppercase
  "htu": "https://verifier.example.com/path", // HTTP target URI (RFC 9449 §4.2)
  "iat": 1748432840,
  "ath": "<base64url SHA-256 of the AIT bytes>" }
```

The proof is signed by the agent's private key (the same key behind the AIT). The verifier confirms `jkt` matches the embedded JWK's thumbprint, verifies the proof signature against that JWK, and checks `htm`/`htu`/`iat`/`ath` and the replay cache (§8 Step 1). The chain inline JWK → `cnf.jkt` → AIR `public_key` → registry-published identity holds together.

**Freshness window.** Verifiers MUST reject DPoP proofs whose `iat` is more than 60 seconds in the past or future (the default). Implementations MAY use tighter windows and MUST NOT exceed 120 seconds without documented operational justification.

**Replay cache.** Verifiers MUST maintain a `jti × jkt` replay cache with TTL ≥ the freshness window, consistent across horizontally-scaled verifier instances. The cache SHOULD fail closed: if it is unavailable, the verifier SHOULD reject (HTTP 503) rather than accept without the replay check.

**DPoP server-nonce mode** ([RFC 9449] §9) is PERMITTED but NOT REQUIRED. A verifier MAY respond `WWW-Authenticate: DPoP error="use_dpop_nonce", nonce="..."` on first contact; an agent receiving this MUST retry with the nonce as a `nonce` claim in the proof JWT. This also composes with the inline-challenge `challenge` nonce (§7.1).

**Deployment-configurable opt-out.** A platform advertises its PoP requirement in `/.well-known/axis-access` via `proof_of_possession` (§7):

- `"required"` (the v0.3 default, RECOMMENDED for any internet-facing verifier) — the verifier MUST reject AITs lacking `cnf.jkt` AND requests lacking a valid DPoP proof.
- `"bearer_allowed"` — the verifier accepts both v0.2 bearer presentations and v0.3 DPoP presentations. The platform documents the threat-model acceptance; internet-facing platforms SHOULD NOT use this. Closed deployments (home LAN, private corporate networks) MAY opt out here and own the decision.

When the field is absent, a v0.3 verifier defaults to `"required"`; a v0.2 verifier reading a v0.3 document treats the absence as `"bearer_allowed"` (it has no PoP machinery).

**Optional message-signature profile for high-stakes scopes.** A platform MAY require [RFC 9421] HTTP Message Signatures for specific scopes, advertised per-scope in `/.well-known/axis-access` under `requires_message_signature`. When required, the request carries DPoP AND a message signature. The canonical AXIS profile signs exactly this component set, in order — `"@method"`, `"@target-uri"`, `"@authority"`, `"content-digest"`, `"date"`, `"axis-ait"` — where `Content-Digest` is per [RFC 9530] (`sha-256` over the body), `Axis-Ait` is a request header carrying base64url SHA-256 of the AIT (identical to the DPoP `ath`), the signature algorithm is `ed25519`, `keyid` is the agent's AXIS ID, and `created` is present within the same freshness window. The component set is normative and MUST NOT be redefined per-platform; alternate profiles may be introduced later via a `proofType`-style selector. See §8 Step 4.5.

**Status — design landed; enforcement not yet shipped in the reference registry.** The `cnf.jkt` binding, DPoP verification, and message-signature profile are specified here for v0.3. The reference registry's `/verify` path and `/.well-known/axis-access` still advertise the v0.2 surface (they do not yet emit `proof_of_possession` or enforce DPoP); PoP enforcement is verifier-/platform-side and lands per the migration path in §8 and the CHANGELOG. Backward compatibility: a v0.3 AIT (with `cnf`) presented to a v0.2 verifier degrades to a bearer token (the verifier ignores the unknown claim); a v0.2 AIT (no `cnf`) is rejected only by a v0.3 verifier advertising `proof_of_possession: "required"`.

#### 4.3.2 Protocol version signaling

Where a protocol version appears on the wire, and what each occurrence means:

**Shipped today:**

- **Records** — AIRs, OIRs, and DCs carry `axis_version`: the protocol version whose data model the record conforms to. Registries echo the version a signed DC was minted under; registry-generated records carry the current record-format version (`"0.2"` — AIR/DC record formats have been stable since v0.2; v0.3 added discovery and verification features, not record-format changes).
- **Discovery documents** — `/.well-known/axis-access`, `/.well-known/axis-scopes`, `/.well-known/axis-registry`, and `/.well-known/axis-directory` each carry `axis_version`: the protocol version of that document's own schema. The reference registry currently advertises `"0.2"` on `axis-access` and `"0.3"` on the v0.3-introduced documents.

**Specified for v0.4 — not yet shipped:**

- **AIT `axis_version` claim** — the protocol version the issuer minted the token against. No shipped issuer emits it and no shipped verifier checks it today. From v0.4, issuers SHOULD emit it on every AIT. Verifiers MUST NOT reject a token solely for lacking `axis_version` until their advertised minimum-supported version (see `/.well-known/axis-versions`, §6.16) is v0.4 or later; a verifier MAY reject a token that *declares* a version below the verifier's minimum-supported version.
- **Registry supported-versions discovery** — `GET /.well-known/axis-versions` (§6.16) advertises the registry's current and minimum-supported protocol versions so implementers can detect mismatch before hitting a hard failure. See COMPATIBILITY.md for the mismatch-handling procedure.

Version comparison follows semantic versioning: within the 0.x series, treat each minor version as potentially breaking (per the CHANGELOG versioning policy); from v1.0, only major-version differences are breaking.

### 4.4 Delegation Credential (DC)

```json
{
  "axis_version": "0.1",
  "type": "DelegationCredential",
  "id": "dc:widget-corp:editor-researcher-2026-04",
  "issued_by": "axis:widget-corp:editor",
  "issued_to": "axis:widget-corp:researcher",
  "root_operator": "widget-corp",
  "parent_credential_id": "dc:widget-corp:operator-editor-2026-03",
  "scope": ["content:create", "content:update"],
  "constraints": {
    "max_sub_delegation_depth": 0,
    "beat": "ai-infrastructure"
  },
  "created": "2026-04-01T00:00:00Z",
  "expires": "2026-05-01T00:00:00Z",
  "revocable": true,
  "revocation_url": "https://registry.axisprime.ai/revocation/credentials/dc:widget-corp:editor-researcher-2026-04",
  "proof": {
    "type": "Ed25519Signature2020",
    "proofType": "jcs-eddsa-2026",
    "created": "2026-04-01T00:00:00Z",
    "verificationMethod": "did:axis:prime:widget-corp:editor#key-1",
    "proofPurpose": "assertionMethod",
    "proofValue": "<Ed25519 signature, base64url>"
  }
}
```

**Required fields:**

- `axis_version` (string)
- `type` (string) — MUST be `"DelegationCredential"`
- `id` (string) — unique credential identifier
- `issued_by` (string) — the delegator (agent or operator AXIS ID)
- `issued_to` (string) — the delegate (AXIS ID or foreign DID)
- `root_operator` (string) — MUST be byte-for-byte identical across all credentials in a chain
- `scope` (array of strings) — permitted actions
- `created` (string, ISO 8601) — issuance timestamp
- `expires` (string, ISO 8601) — expiration timestamp
- `proof` (object) — cryptographic proof per [W3C Data Integrity]. See "DC proof canonicalization" below for the normative signing bytes.

**DC proof canonicalization (normative).** The bytes signed are the [RFC 8785] JCS canonicalization of the complete DC document **minus its `proof` field**, UTF-8 encoded; `proofValue` is the base64url Ed25519 signature over those bytes. This is the same JCS regime §6.1 establishes for all signed canonical bodies in the protocol, and it is what the reference SDK signs and the reference registry verifies. JCS canonicalization is REQUIRED here because DCs contain nested objects (`constraints`, `proof`) — the legacy v0.1 top-level-sort canonicalization (`JSON.stringify(body, Object.keys(body).sort())`) silently *filtered* nested keys rather than recursively sorting them, producing different canonical bytes for the same document and breaking nested-document signing. The proof envelope:

- `type` — `"Ed25519Signature2020"` (the Data Integrity suite name).
- `proofType` — `"jcs-eddsa-2026"`, RECOMMENDED. Explicitly selects the JCS regime. When `proofType` is absent, verifiers MUST attempt JCS verification first and MAY fall back to the deprecated legacy canonicalization (per §6.1); the legacy form is removed in v1.0. A proof carrying an *unrecognized* `proofType` MUST be rejected distinctly (e.g. `unsupported_proof_type`), never silently treated as legacy — accepting an unknown suite name would bypass the canonicalization contract.
- `verificationMethod` — `<issuer-identifier>#key-1`, where the issuer identifier MAY be the DID form (`did:axis:prime:widget-corp:editor#key-1`) or the AXIS ID form (`axis:widget-corp:editor#key-1`; the form the reference SDK emits). Verifiers resolve the issuer's key from the registry regardless (§8 Step 3); the field is informative about *which* key, not a resolution instruction.
- `proofPurpose` — `"assertionMethod"`.
- `proofValue` — the signature, base64url.

**Optional fields:**

- `parent_credential_id` (string) — links to the credential that authorized the delegator. REQUIRED when `issued_by` is not the root operator.
- `constraints` (object) — additional limits. Semantics are out-of-band.
- `revocable` (boolean) — whether the credential can be revoked before expiry. Default `true`.
- `revocation_url` (string) — where to check revocation status for this specific credential. OPTIONAL for ephemeral-delegate DCs (§4.4.2); REQUIRED unchanged from v0.2 for registered DCs.
- `issued_to_public_key` (string, v0.3) — Ed25519 public key of an ephemeral delegate, base64url. REQUIRED when `issued_to` matches the ephemeral pattern; FORBIDDEN otherwise. See §4.4.2.
- `task_spec_hash` (string, v0.3) — SHA-256 hex of a canonical serialization of the task spec the issuer authorized. OPTIONAL; binds an ephemeral DC to a specific task instance for audit. See §4.4.2.

**Scope grammar (v0.2, unchanged in v0.3).** Scopes are colon-separated strings where each segment is an ASCII word matching `[a-zA-Z0-9_-]+`. Examples: `read:articles`, `write:drafts`, `admin:users`. A scope `granted` matches a scope `requested` if, segment-by-segment, each `granted` segment is identical to the corresponding `requested` segment, OR the `granted` segment is the literal wildcard `*`. The wildcard matches exactly one segment, not multiple: `admin:*` covers `admin:users` and `admin:roles` but does NOT cover `admin:users:delete`. Verifiers computing effective scope across a delegation chain MUST intersect per the same matching rules. The wildcard `*` is a **matching/attenuation grammar feature only** — it is NOT a grantable standard scope (see §4.4.1).

**Scope vocabulary (v0.3).** v0.2 standardized the scope *grammar* (shape) and left the *vocabulary* (meaning) platform-defined. v0.3 layers a standard vocabulary over the grammar so interoperating parties agree on what common scopes mean. The full standard table and the two-layer namespace rules are in §4.4.1 and Appendix B. Platforms continue to advertise the strings they require in `/.well-known/axis-access` under `required_scopes` and publish the vocabulary they recognize at `/.well-known/axis-scopes` (§6.14).

#### 4.4.1 Standard scope vocabulary — two-layer namespace (v0.3)

Scopes fall into two layers, both of which MUST satisfy the v0.2 grammar above:

- **Layer 1 — Standard scopes.** Carry NO vendor prefix; the first segment is one of the reserved standard domains (`content`, `social`, `commerce`, `data`, `comms`, `account`, `scheduling`, `compute`) and the whole scope is one of the recognized strings in the standard table (Appendix B). The standard vocabulary is a **CLOSED enum**: a grammar-valid scope whose first segment is a reserved standard domain but which is not a recognized standard string is **INVALID** (domain squatting is rejected — e.g. `content:frobnicate` is invalid because `content` is reserved). This keeps adding a standard action a deliberate vocabulary change, not something any caller can mint.

  Domain wildcards such as `content:*` are **NOT grantable standard scopes.** `content` is a reserved domain and `content:*` is not a recognized standard string, so it classifies as invalid in the standard namespace. The single-segment `*` remains available as a matching/attenuation grammar feature (above), not as a vocabulary entry.

- **Layer 2 — Custom scopes.** Anything outside the standard set MUST be prefixed `x-<vendor>:` where `<vendor>` matches `[a-z0-9-]+` (e.g. `x-ghost:newsletter:send`). The `x-` prefix is RESERVED for custom strings so a vendor extension can never collide with a future standard domain. **Unprefixed non-standard scopes are INVALID** — a grammar-valid scope that is neither a recognized standard string nor `x-`-prefixed is rejected.

Classification (matching the reference registry's `classifyScope`, in order): (1) recognized standard string → `standard`; (2) malformed grammar → `invalid`; (3) first segment is a reserved standard domain but not a recognized string → `invalid` (domain squatting); (4) first segment matches `x-<vendor>` → `custom`; (5) otherwise (grammar-valid, neither standard-domain nor `x-`-prefixed) → `invalid`.

This is adoption pull, not enforcement at the protocol layer: an agent holding `content:comment` works at every platform that speaks the standard, while a platform that invents `x-acme:commenting:add` only interoperates with agents that learned that string. The standard set is a versioned seed (Appendix B), grown by proposal, W3C-vocabulary-first (ActivityStreams 2.0, schema.org Actions, ODRL).

**Enforcement status.** The reference registry ships the two-layer classifier and publishes the vocabulary at `/.well-known/axis-scopes`, but its delegation **write** path currently validates the v0.2 grammar only — a grammar-valid unprefixed non-standard scope (e.g. `article:draft`) is still accepted on `POST /delegations` today. Vocabulary-layer classification is therefore enforced at *verify/authorize* time by platforms (which check for the specific standard strings they require, e.g. `content:comment`), not yet at credential-mint time. Wiring the classifier into the registry write path is tracked for a subsequent release; issuers SHOULD already restrict themselves to standard or `x-<vendor>:` scopes so their credentials survive that tightening.

#### 4.4.2 Ephemeral sub-agent delegates (v0.3)

The v0.2 DC requires the delegate (`issued_to`) to be a registered AXIS agent (with an AIR and persistent key) or a foreign DID. This does not fit the **within-runtime sub-agent case** common in agent runtimes: a registered agent spawns a transient, task-bound sub-execution to do one piece of work. Today that sub-execution either inherits the parent's identity (no audit granularity, no surgical interrupt, capability laundering) or is registered as its own persistent agent (heavy: a registry write and dead identity per task). v0.3 adds a third delegate form: a **task-bound delegate that exists only for the lifetime of one delegation, derived from the parent's signature, with no registry footprint.**

**`issued_to` ephemeral form.** In addition to the three existing patterns (operator, registered agent, foreign DID), `issued_to` MAY take the ephemeral form:

```
axis:{operator}:{agent}:sub:{task-id}
```

where `{operator}` and `{agent}` reference the parent registered agent and `{task-id}` matches `[A-Za-z0-9_-]+`. Implementations SHOULD generate `{task-id}` deterministically (e.g. `base64url(sha256(issued_to_public_key || issued_by || created || canonical(scope)))[:16]`) so a verifier can recompute the binding, but the protocol does NOT mandate a derivation; the DC's `proof.proofValue` is what cryptographically binds the identifier. Ephemeral DIDs are NEVER written to the registry — they are self-proving via the DC chain.

**`issued_to_public_key`.** REQUIRED when `issued_to` is ephemeral, FORBIDDEN otherwise: the base64url Ed25519 public key of the ephemeral delegate (43 chars, unpadded). The verifier uses it to verify the delegate's action signature without registry resolution.

**`task_spec_hash`.** OPTIONAL SHA-256 hex binding the DC to a specific task instance for audit. The canonical task-spec schema is deferred; issuers hash whatever structured task description they emit.

Example:

```json
{
  "axis_version": "0.3",
  "type": "DelegationCredential",
  "id": "dc:offworldnews-ai:bram-research-1748c3a2",
  "issued_by": "axis:offworldnews-ai:bram",
  "issued_to": "axis:offworldnews-ai:bram:sub:research-1748c3a2",
  "issued_to_public_key": "h3jK9lQ8xH5p2dN-VrYqZmKFAI_32qIiLBTdTPOcTVE",
  "root_operator": "offworldnews-ai",
  "parent_credential_id": "dc:offworldnews-ai:operator-bram-2026-04",
  "scope": ["x-bram:web:search", "x-bram:library:search"],
  "constraints": { "max_sub_delegation_depth": 0 },
  "task_spec_hash": "a3b8c2e1f4d5...",
  "created": "2026-05-25T14:25:00Z",
  "expires": "2026-05-25T14:30:00Z",
  "revocable": true,
  "proof": { "type": "Ed25519Signature2020", "verificationMethod": "did:axis:prime:offworldnews-ai:bram#key-1", "proofPurpose": "assertionMethod", "proofValue": "<Ed25519 signature by bram's persistent key>" }
}
```

**Normative requirements:**

- A DC with an ephemeral `issued_to` MUST include `issued_to_public_key`; a DC without one MUST NOT include it.
- The `issued_by` of an ephemeral DC MUST be a registered AXIS agent (operators delegate to registered agents; registered agents delegate to ephemeral sub-tasks).
- Ephemeral delegates SHOULD NOT mint AITs. Authentication flows through the parent: the parent mints an AIT in its own name with the `dlg` claim pointing at the ephemeral DC's `id`, and presents that AIT plus the inline DC chain (including the ephemeral DC with `issued_to_public_key`). The verifier walks the chain (§8 Step 3); the innermost ephemeral DC's `scope` is what the request must fit inside, and its `id` is the authorizing credential in the audit row. The agent stays "at the desk" on behalf of the sub-task.
- All v0.2 invariants apply unchanged across links with ephemeral delegates: attenuation, the `root_operator` byte-for-byte invariant, `created <= now < expires`, and scope-subset.
- Ephemeral DCs are NOT written to the registry's `delegations` table; they live in the issuer's local audit log and travel inline with the action envelope. `revocation_url` is OPTIONAL for them — revocation is driven primarily by short `expires` and parent re-issuance rather than registry-mediated revocation.
- Ephemeral-to-ephemeral nesting is permitted, gated by `max_sub_delegation_depth` per §4.4 (an open question flagged for review; default proposal is to permit it).

**The attenuation rule (MUST).** The `scope` of a DC MUST be equal to or a subset of the `scope` of its `parent_credential_id` per the matching rules above. Verifiers MUST reject any DC where the delegate's scope exceeds the delegator's scope.

**The root-operator invariant (MUST).** The `root_operator` field MUST be byte-for-byte identical across every credential in a chain. Verifiers MUST reject any chain where intermediate credentials assert a different `root_operator`. This prevents chain-rerooting attacks.

**Sub-delegation depth (SHOULD).** Delegators SHOULD include `max_sub_delegation_depth` in `constraints`. When present, the delegate MAY further delegate only if the new credential's depth is `max_sub_delegation_depth - 1` or higher. A value of `0` forbids sub-delegation.

**Cross-system delegation.** The `issued_to` field MAY reference a foreign identifier, e.g. `did:key:z6Mk...` or `did:web:example.com`. Verifiers resolving foreign delegates follow the DID method's resolution procedure to retrieve the delegate's public key.

### 4.5 Trust Attestation (TA)

```json
{
  "axis_version": "0.1",
  "type": "TrustAttestation",
  "id": "ta:widget-corp:researcher-2026-04",
  "issued_by": "axis:widget-corp:editor",
  "subject": "axis:widget-corp:researcher",
  "issued_at": "2026-04-15T00:00:00Z",
  "scope": "editorial:research",
  "level": 3,
  "statement": "Consistently produces well-cited research briefs on time.",
  "signature": "<Ed25519 signature, base64url>"
}
```

**Required fields:**

- `axis_version` (string)
- `type` (string) — MUST be `"TrustAttestation"`
- `id` (string)
- `issued_by` (string) — AXIS ID of the attestor
- `subject` (string) — AXIS ID of the agent being attested
- `issued_at` (string, ISO 8601)
- `scope` (string) — domain of the attestation
- `signature` (string) — Ed25519 signature by the attestor

**Optional fields:**

- `level` (integer) — attestor-defined level (semantics out-of-band)
- `statement` (string) — plain-language description

**Storage.** Trust Attestations MUST NOT be stored in the registry. They are stored by the issuer, the agent (as a portfolio), or an external attestation index. The registry's job is identity, not reputation.

**Aggregation and scoring.** v0.1 does not specify how multiple Trust Attestations are aggregated. This is out of scope for v0.1 and planned for a future version (see §17).

**Design rationale — what a Trust Attestation is, and what it is not.** The v0.1 Trust Attestation is deliberately a signed container, not a prescribed rating system. AXIS does not specify what constitutes trust, how it is measured, or how multiple attestations combine. This is intentional: different verticals apply different standards of evidence, and prescribing a single scale in v0.1 would be either too narrow for most use cases or too abstract to be useful.

The design intent is to favor **measurable behavioral signals** (analogous to credit utilization — report observable facts, let consumers decide) over **opinion signals** (analogous to star ratings — subjective judgments without anchoring). A well-formed attestation reports events that happened, states that can be audited, or measurements that can be reproduced. A weakly-formed attestation reports feelings. v0.1 does not enforce this distinction — `statement` is free-form — but implementations SHOULD lean toward observable, signable events rather than editorial judgments. The planned analytical layer (aggregation, weighting, gaming resistance — see §17) is designed to operate on behavioral signals; opinion signals degrade the aggregation's usefulness.

### 4.6 Content Provenance Attestation (CPA)

```json
{
  "axis_version": "0.1",
  "type": "ContentProvenanceAttestation",
  "id": "cpa:widget-corp:article-2026-04-15",
  "content_id": "https://example.com/articles/ai-agent-identity",
  "produced_by": "axis:widget-corp:researcher",
  "produced_under_credential": "dc:widget-corp:editor-researcher-2026-04",
  "reviewed_by": "axis:widget-corp:editor",
  "approved_at": "2026-04-15T12:00:00Z",
  "root_operator": "widget-corp",
  "signature": "<Ed25519 signature by reviewer, base64url>"
}
```

A CPA binds content, production agent, delegation credential, review agent, and approval timestamp into a single signed artifact. Third parties verifying content governance can validate a CPA without any prior relationship to the operator or the content pipeline.

## 5. Registry Data Visibility Model

Registry data is organized into three visibility tiers. This model applies to both AIRs and OIRs.

### 5.1 Public layer

Available to anyone, no authentication required. Contains only what is necessary for cryptographic verification:

- `public_key` and `key_algorithm`
- `agent_id` or `operator_id`
- `status`
- `revocation_url`
- `registry_url`
- `axis_version`
- `service_endpoints` (agents only; OPTIONAL; W3C DID `service` array shape)

Public-layer identifiers MUST NOT carry personally identifiable information. `operator_id` derived from the verified `domain` is acceptable (the domain is already public). `operator_id` derived from any part of an email address is NOT acceptable; email-tier operators MUST receive an opaque random identifier. See Registry Conformance §8.1.1 for the full normative rule.

### 5.2 Presentation layer

Returned in addition to the public layer when ANY of the following holds:

1. The request includes a valid AIT in the `Authorization: Bearer` header or `?ait=` query parameter (see §5.4 Presentation context). This is the original case: an agent actively presents credentials to a receiving platform.
2. The caller is the registrar that owns the record, authenticated per the registry's authentication mechanism. Registrars reading their own data MUST receive the presentation layer without needing to mint an AIT.
3. The caller holds an `admin` or `super_admin` role (see Registry Conformance §2). Cross-tenant administrative reads receive the presentation layer.

Otherwise the registry MUST return only the public layer.

**Presentation-layer fields:**

- For agents: `display_name`, `purpose`, `platform`, `capability_tier`
- For operators: `display_name`, `verification_tier`, `domain`, `registered_at`

### 5.3 Private layer

Only visible through the registrar relationship or explicit operator consent. Never exposed through registry API.

- Operator contact information
- Business registration details
- KYB verification documents
- Billing information

### 5.4 Presentation context

A receiving platform requests the presentation layer by including an AIT in its query to the registry:

```
GET /agents/{agent_id}
Authorization: Bearer <AIT of the agent presenting credentials>
```

The registry verifies the AIT, confirms the agent belongs to the requested operator (or matches the requested agent), and returns the union of public + presentation layers. If no AIT is provided, only the public layer is returned.

## 6. API contract

Every AXIS-compliant registry MUST expose the following endpoints. Response bodies are JSON unless noted. Error responses follow the shape:

```json
{
  "error": {
    "code": "invalid_request",
    "message": "Human-readable error description"
  }
}
```

### 6.1 `POST /register`

Register a new agent. Registrar-authenticated (Bearer token).

Request body:
```json
{
  "public_key": "<Ed25519 public key, base64url>",
  "operator": {
    "email": "alice@example.com"
  },
  "metadata": {
    "name": "editor",
    "description": "Editorial director."
  },
  "proof": {
    "proofType": "jcs-eddsa-2026",
    "proofValue": "<Ed25519 signature over JCS-canonicalized request body, base64url>"
  }
}
```

Response (201):
```json
{
  "axis_id": "axis:example-xyz:editor",
  "did": "did:axis:prime:example-xyz:editor",
  "token": "<signed AIT JWT>",
  "registry_url": "https://registry.axisprime.ai/agents/axis:example-xyz:editor",
  "revocation_url": "https://registry.axisprime.ai/revocation/axis:example-xyz:editor"
}
```

**Proof of key ownership (v0.2).** The optional `proof` field demonstrates the registrant controls the private key matching `public_key`. Two proof types are recognized:

- `proofType: "jcs-eddsa-2026"` (v0.2, RECOMMENDED) — the request body MINUS the `proof` field is canonicalized per [RFC 8785 JCS](https://datatracker.ietf.org/doc/html/rfc8785), and `proofValue` is the base64url Ed25519 signature over the canonical bytes. Closes the proof-canonicalization fragility in v0.1, where nested object keys were not deterministically ordered. JCS canonicalization is the v0.2 default for all signed canonical bodies in the protocol (registration proof, delegation envelope, action envelope).
- `proofType` absent — legacy v0.1 canonicalization (`JSON.stringify(body, Object.keys(body).sort())`). Registries SHOULD verify JCS-signed proofs first and fall back to the legacy form when JCS verification fails. Legacy canonicalization is deprecated in v0.2 and will be removed in v1.0.

Registries MUST verify the proof when present. Registries MAY require proof for new registrations; the canonical reference implementation requires it.

### 6.2 `GET /agents/:agent_id`

Resolve an AIR. Public layer by default; presentation layer if a valid AIT is in the Authorization header.

Response: the AIR as specified in §4.1, subject to visibility tier.

### 6.3 `GET /operators/:operator_id`

Resolve an OIR. Same tiered visibility as §6.2.

### 6.4 `GET /verify?token=<AIT>`

Verify an AIT. The registry checks token structure, the presence of a non-empty `aud` claim (value matching is platform-side; see §4.3), the Ed25519 signature against the agent's registered key, expiry, and agent status.

Response (valid):
```json
{
  "valid": true,
  "agent_id": "axis:widget-corp:editor",
  "operator_id": "axis:widget-corp:operator",
  "status": "active",
  "expires_at": "2026-04-16T00:00:00Z",
  "dlg": "dc:widget-corp:editor-2026-04",
  "delegation_id": "dc:widget-corp:editor-2026-04",
  "scope": null,
  "delegation_resolved": true,
  "delegation_valid": true,
  "delegation_bound_to_agent": true,
  "delegation_depth": 2,
  "effective_scope": ["content:comment"]
}
```

- `operator_id` is returned in the canonical `axis:{operator}:operator` form, resolved authoritatively from the agent's registry record — never from the token payload.
- When the AIT carries a `dlg` claim, the registry resolves the delegation chain to root (§6.9.1) and returns the `delegation_*` fields plus the chain's trustworthy `effective_scope` (root-to-leaf intersection). Top-level `valid` remains the *identity* assertion (signature + agent status); the delegation's trustworthiness is surfaced separately as `delegation_valid` for the platform to act on. `delegation_bound_to_agent` confirms the named credential's leaf `issued_to` is the presenting agent — a delegation issued to someone else does not authorize this presenter. `delegation_id` mirrors `dlg` for legacy consumers.
- `scope` echoes any self-declared `scope` claim from the token; it is NOT trustworthy authorization data — platforms enforce against `effective_scope`.

If invalid, returns HTTP 200 (the verification request itself succeeded; the token is just invalid) with a stable machine-readable `code`:

```json
{ "valid": false, "code": "token_expired", "agent_id": "axis:widget-corp:editor", "reason": "Token expired", "expired_at": "2026-04-15T00:00:00Z" }
```

`code` is one of: `invalid_signature`, `token_expired`, `agent_revoked` (covers revoked and deactivated), `agent_suspended`. Structural failures return 400 with the standard error envelope: `invalid_request` (not a JWT, or header is not `typ: AIT` / `alg: EdDSA`) or `missing_aud` (the AIT lacks a non-empty `aud` claim). An unknown agent returns 404 (`agent_not_found`).

### 6.5 `GET /revocation/:agent_id`

Check revocation status of an agent. Public endpoint.

Response:
```json
{
  "agent_id": "axis:widget-corp:editor",
  "revoked": false,
  "checked_at": "2026-04-15T12:00:00Z"
}
```

### 6.6 `GET /revocation/credentials/:credential_id`

Check revocation status of a specific Delegation Credential.

Response:
```json
{
  "credential_id": "dc:widget-corp:editor-researcher-2026-04",
  "revoked": false,
  "checked_at": "2026-04-15T12:00:00Z"
}
```

### 6.7 `DELETE /agents/:agent_id`

Revoke an agent. Requires operator or registrar authentication.

### 6.8 `POST /delegations`

Issue a Delegation Credential. Agent- or operator-authenticated.

Request body: a DC as specified in §4.4, minus the `proof` field (the registry signs or the caller provides their own signed DC, per registry policy).

### 6.9 `GET /delegations/:id`

Retrieve a Delegation Credential by ID. Public endpoint.

#### 6.9.1 `GET /delegations/:id/chain` — chain resolution (dual-mode)

Resolve and verify a delegation chain. Public endpoint. The `:id` parameter accepts BOTH identifier forms a verifier may hold, and the two forms have different resolution semantics:

1. **Delegation-credential id** (the form an AIT's `dlg` claim carries, §4.3) — the chain is resolved **upward from that credential** via `parent_credential_id` to the root, and the verdict is **pinned** to it: `chainValid` reflects only the named credential's chain, and the response carries a top-level `delegation` block (id, scope, parties, status, expiry) plus the chain's `effective_scope` (the root-to-leaf scope intersection per §4.4 matching rules). A verifier holding a token's `dlg` claim MUST enforce against this pinned scope — never against the union of the delegatee's other delegations.
2. **Agent identifier** (AXIS ID, DID, or bare slug) — the registry finds the agent's newest active delegation and walks issuer-to-issuer back to the root operator. This form answers "does this agent have a valid chain?" without pinning to a specific credential.

**Detection.** Identifiers with the `dc:` prefix (the registry-generated credential-id form) are treated as delegation ids outright — no agent-identifier form starts with `dc:`, so the prefix is unambiguous. Other identifiers try agent resolution first, then fall back to a direct delegation lookup (issuer-chosen ids on signed submissions may use any scheme). Unknown `dc:`-prefixed ids return 404 `delegation_not_found`; other unknown identifiers return 404 `agent_not_found`.

Response (delegation-id form; the agent form omits the `delegation` and `effective_scope` blocks):

```json
{
  "agent": "did:axis:prime:widget-corp:editor",
  "axis_id": "axis:widget-corp:editor",
  "delegation": {
    "id": "dc:widget-corp:editor-2026-04",
    "scope": ["content:comment"],
    "issued_by": "axis:widget-corp:operator",
    "issued_to": "axis:widget-corp:editor",
    "status": "active",
    "expires": "2026-05-01T00:00:00Z"
  },
  "effective_scope": ["content:comment"],
  "chainValid": true,
  "chainDepth": 1,
  "chain": [
    { "delegation": "dc:widget-corp:editor-2026-04", "from": "axis:widget-corp:operator", "to": "axis:widget-corp:editor", "scope": ["content:comment"], "signatureValid": true, "expired": false, "status": "active" }
  ],
  "rootOperator": { "domain": "widget-corp.com", "verified": true },
  "verifiedAt": "2026-07-07T00:00:00Z"
}
```

Per-link checks: proof signature against the resolved issuer key (`signatureValid` reported truthfully per link; legacy/unsigned rows report `false`), status, expiry (`created <= now < expires`), root-operator consistency, scope attenuation, cycle detection, and a chain-depth cap of 16.

**Trust note.** This endpoint is a *convenience* — the registry walking its own stored chain. A verifier performing full §8 verification of an inline-presented chain does the same checks locally; a verifier that received only an AIT with `dlg` uses this endpoint (or `GET /verify`, which embeds the same resolution) to obtain the trustworthy effective scope.

### 6.10 `DELETE /delegations/:id`

Revoke a Delegation Credential. Requires delegator authentication.

### 6.11 `GET /resolve/:did`

W3C DID resolution. Returns a DID Document derived from the AIR.

Response: a DID Document per W3C DID Core.

### 6.12 `GET /.well-known/axis-access`

This registry's own access policy. See §7.

### 6.13 `GET /.well-known/axis-registry` (v0.3)

Registry self-identification manifest. Each registry publishes its signing-key set and an Ed25519 self-signature over the canonical manifest (minus the `signature` field). Public, static, edge-cacheable. Used by the registry-legitimacy verifier (§8 Step 1).

Response (the exact shipped shape):
```json
{
  "axis_version": "0.3",
  "registry_id": "prime",
  "registry_url": "https://registry.axisprime.ai",
  "keys": [
    {
      "kid": "prime-registry-2026",
      "alg": "Ed25519",
      "public_key": "<base64url Ed25519 public key>",
      "status": "active",
      "not_before": "2026-06-14T21:38:39Z",
      "not_after": "2027-06-14T21:38:39Z"
    }
  ],
  "endpoints": {
    "agents": "/agents",
    "operators": "/operators",
    "verify": "/verify",
    "revocation": "/revocation",
    "delegations": "/delegations"
  },
  "directory_url": "https://registry.axisprime.ai/.well-known/axis-directory",
  "signature": "<Ed25519 self-signature over JCS-canonicalized manifest minus `signature`>"
}
```

- `keys` carries the signing key set. Key rotation is expressed here through overlapping `not_before`/`not_after` windows and per-key `status` (`active` is the relevant value for verification; rotation lives in the manifest, not in re-issued identities). The legitimacy verifier accepts the manifest if its self-signature verifies against ANY active Ed25519 key.
- `signature` proves control of the keys (domain control). It does NOT by itself establish legitimacy — that requires the manifest's keys to be listed `certified` in the root directory (§6.15, §8).

### 6.14 `GET /.well-known/axis-scopes` (v0.3)

Standard scope vocabulary discovery. Publishes the scopes the platform recognizes, each flagged `standard` and described, so an agent can reason about an unfamiliar scope. Public, no auth, edge-cacheable.

Response (the exact shipped shape — one entry per standard scope, in domain/insertion order; see Appendix B for the full table):
```json
{
  "axis_version": "0.3",
  "platform_id": "registry.axisprime.ai",
  "scopes": [
    { "scope": "content:read", "standard": true, "description": "Read content items." },
    { "scope": "content:create", "standard": true, "description": "Create new content items." }
  ],
  "updated_at": "2026-06-14T00:00:00Z"
}
```

A platform MAY additionally publish custom scopes it recognizes with `standard: false` and an optional `maps_to` pointing at the nearest standard scope, so an agent can reason about an unfamiliar custom string:

```jsonc
{ "scope": "x-ghost:newsletter:send", "standard": false, "maps_to": "comms:send", "description": "Send a newsletter issue to subscribers." }
```

`axis-scopes` says what vocabulary a platform RECOGNIZES (the dictionary); `axis-access` (§7) says what it REQUIRES (the bar). They are complementary and independent.

### 6.15 `GET /.well-known/axis-directory` (v0.3)

AXIS Prime's signed root directory of certified registrars. Signed by the Prime ROOT key, which verifiers pin out of band. Public, static, edge-cacheable. Used by the registry-legitimacy verifier (§8 Step 1).

Response (the exact shipped shape):
```json
{
  "axis_version": "0.3",
  "directory_version": 1,
  "issued_at": "2026-06-14T21:38:39Z",
  "expires_at": "2027-06-14T21:38:39Z",
  "registrars": [
    {
      "registry_id": "prime",
      "registry_url": "https://registry.axisprime.ai",
      "key_fingerprints": ["<sha256 hex of raw Ed25519 public-key bytes>"],
      "status": "certified",
      "certified_at": "2026-06-14T21:38:39Z"
    }
  ],
  "root_signature": "<Ed25519 signature by the AXIS Prime root key over JCS-canonicalized directory minus `root_signature`>"
}
```

- `status` is one of `certified`, `suspended`, `revoked`. Suspended/revoked registries are LISTED (so verifiers stop trusting them), not silently dropped.
- `directory_version` is a monotonic integer. Verifiers reject a cached directory whose version is lower than one already seen (rollback protection).
- `key_fingerprints` are the lowercase hex SHA-256 of the RAW public-key bytes (the 32 bytes the `public_key` base64url-decodes to), taken over the raw bytes — not over the base64url text — so the fingerprint is encoding-independent. At least one manifest key fingerprint must appear here for the registry to be trusted.
- `root_signature` is verified against the pinned Prime root key. The root public key is the single out-of-band trust anchor (pinned in the conformance suite and SDKs, the browser-ships-a-root-store pattern).

Both the manifest (§6.13) and the directory are static and signed, so steady-state verification adds zero per-lookup round trips to Prime. The directory carries `expires_at` (suggested 7-day window, daily refresh).

Registrar admission criteria (what a registry must do to be certified) are a governance/conformance matter (`axis-conformance`), not part of this wire protocol.

### 6.16 `GET /.well-known/axis-versions` (v0.4 — specified, not yet shipped)

Registry supported-versions discovery. Publishes the protocol versions the registry currently speaks so implementers can detect a version mismatch up front rather than through a hard failure mid-flow. Public, no auth, edge-cacheable.

**Status — specified, not yet shipped.** The reference registry does not implement this endpoint today; it is specified here as a v0.4 addition. Until it ships, implementers infer version support from the `axis_version` fields on the discovery documents (§4.3.2) and the CHANGELOG.

Response:
```json
{
  "axis_version": "0.4",
  "current": "0.4",
  "minimum_supported": "0.2",
  "versions": [
    { "version": "0.4", "status": "current" },
    { "version": "0.3", "status": "supported" },
    { "version": "0.2", "status": "supported" },
    { "version": "0.1", "status": "sunset" }
  ],
  "updated_at": "2026-07-07T00:00:00Z"
}
```

- `current` — the newest protocol version this registry fully implements.
- `minimum_supported` — the oldest version whose credentials and API calls this registry still accepts. Anything older MAY be rejected.
- `versions[].status` — one of `current`, `supported`, `deprecated` (still works; scheduled for removal — see COMPATIBILITY.md for the announce → deprecate → sunset intent), `sunset` (no longer accepted).

Clients SHOULD check this document when they encounter an unrecognized field, a rejected credential, or a new deployment, and SHOULD warn their operator when the client's own protocol version falls outside the `[minimum_supported, current]` window. See COMPATIBILITY.md for the full mismatch-handling procedure.

## 7. Platform-side access control

Every receiving platform accepting agent interactions SHOULD publish its access requirements at `/.well-known/axis-access`. v0.2 introduces a REQUIRED `audience` field that platforms publish so issuers can populate the `aud` claim on AITs intended for that platform (see §4.3).

Response:
```json
{
  "axis_version": "0.2",
  "platform_id": "ghost.example.com",
  "audience": "ghost.example.com",
  "proof_of_possession": "required",
  "requires_message_signature": ["content:publish", "commerce:pay"],
  "access_policy": {
    "minimum_verification_level": "domain",
    "required_scopes": ["content:comment"],
    "allow_unverified": false,
    "registration_url": "https://signup.axisprime.ai/signup",
    "blocked_operators": [],
    "approved_operators": null,
    "rate_limits": null
  },
  "updated_at": "2026-06-14T00:00:00Z"
}
```

**Fields:**

- `audience` — **(REQUIRED)** the platform's stable audience identifier. AIT issuers populate the `aud` claim with this value when minting tokens intended for this platform; verifiers reject tokens whose `aud` does not match. The identifier MUST be a non-empty string and MUST be stable across the platform's lifetime. Convention is subdomain-shaped (`<service>.<domain>`) but the format is not normative; any stable string works.
- `proof_of_possession` (v0.3) — `"required"` or `"bearer_allowed"`. Controls whether the platform demands sender-constrained AITs (§4.3.1). Default when absent: `"required"` for v0.3 verifiers, `"bearer_allowed"` for v0.2 verifiers reading a v0.3 document.
- `requires_message_signature` (v0.3) — OPTIONAL array of scope strings; requests for those scopes MUST additionally carry an RFC 9421 message signature per the canonical AXIS profile (§4.3.1, §8 Step 4.5). Absent or empty means no scope on this platform requires message signing.
- `minimum_verification_level` — lowest operator verification tier accepted
- `required_scopes` — scopes the agent must hold
- `allow_unverified` — boolean. If `true`, requests without an AIT are accepted (the platform chooses whether to treat them as verified or as anonymous traffic). If `false`, requests without an AIT are rejected.
- `registration_url` — where to direct unregistered agents or operators who don't meet the minimum tier
- `blocked_operators`, `approved_operators`, `rate_limits` (v0.3, OPTIONAL) — operator allow/block lists and rate-limit hints, enforced platform-side. Published for schema completeness; a platform that does not operator-gate leaves the lists empty/absent.
- `actions` (OPTIONAL, shipped in the reference commenting platform) — an array describing how to *invoke* this platform, not just what it requires: per-action `id`, `description`, `method`, `path`, `auth` (e.g. `{ "header": "X-AXIS-Token", "scheme": "AIT" }`), `required_scopes`, a minimal `request` field schema, and expected `responses`. Strictly additive and advisory — consumers that predate the field ignore it, and its absence remains valid. Deliberately minimal; not a full OpenAPI document.
- `platform_id` (OPTIONAL) — a human-oriented platform label. `audience` is the normative identifier; shipped platform implementations omit `platform_id` and remain conformant.

**Reference-registry note.** The reference registry, acting as a platform for its own endpoints, advertises `axis_version: "0.2"` on this document and a permissive policy (`minimum_verification_level: "email"`, `allow_unverified: true`, empty `required_scopes`). It does not yet emit `proof_of_possession`; the v0.3 PoP fields above are the spec design for platforms that adopt sender-constrained AITs.

**Design rationale.** The protocol defines how platforms *communicate* policy. Enforcement is the platform's responsibility. AXIS provides identity and trust signals; platforms make their own decisions.

### 7.1 Inline challenge-on-refusal (v0.3, OPTIONAL)

v0.2 supports only the pull model: an agent reads `/.well-known/axis-access` proactively. There is no defined way for a gated endpoint to tell an agent, at the moment of refusal, exactly what was required and how to obtain it. v0.3 defines an OPTIONAL structured refusal body so failures are loud, specific, and actionable.

Status codes carry funnel meaning and are used precisely:

- **No AIT / no AXIS identity → `401 Unauthorized`**, with a `WWW-Authenticate: AXIS` challenge. ("Retry with credentials" — the sign-up funnel.) The challenge header carries discovery parameters so a refused agent can self-serve the fix:

  ```
  WWW-Authenticate: AXIS realm="workspace", audience="files.example.com", registry="https://registry.axisprime.ai", discovery="/.well-known/axis-access"
  ```

  - `realm` — the protected surface's label.
  - `audience` — the platform's stable audience identifier (§4.3, §7), so a retrying agent mints the AIT with the right `aud` without a separate fetch.
  - `registry` — the registry the platform verifies against (where an unregistered agent should go).
  - `discovery` — the path to the platform's `/.well-known/axis-access` document.
- **Valid AIT but tier too low or scope insufficient → `403 Forbidden`**. ("Authenticated but not authorized" — the level-up funnel, the deny-and-show-why moment.)

Both carry the same body, scoped to the refused action:

```json
{
  "axis_error": "insufficient_verification",
  "message": "Posting an article requires a domain-verified operator. Your operator is email-verified.",
  "action": "content:publish",
  "required": { "minimum_verification_level": "domain", "required_scopes": ["content:publish"], "allow_unverified": false },
  "current": { "verification_level": "email", "held_scopes": ["content:comment"] },
  "remedy": {
    "what": "Upgrade this operator to domain verification.",
    "how": "Add the AXIS DNS TXT record shown at the link, then retry.",
    "url": "https://signup.axisprime.ai/upgrade?operator=acme&to=domain&return=https%3A%2F%2Fghost.example.com%2Fpublish"
  },
  "audience": "ghost.example.com",
  "challenge": "<opaque nonce, optional>"
}
```

- `axis_error` (machine-readable): one of `no_credential`, `insufficient_verification`, `insufficient_scope`, `operator_revoked`, `agent_revoked`, `expired`.
- `message` — human-readable and specific.
- `action` — the scope the refused action maps to.
- `missing_scopes` (OPTIONAL array) — on `insufficient_scope` denials, the exact scope strings the request required but the presented chain did not cover. Shipped platform verifiers emit this alongside `axis_error`; it is the minimal machine-actionable form of the `required`/`current` gap.
- `required` vs `current` — the gap, as data.
- `remedy` — `what`/`how` in plain language; `url` deep-links to the exact upgrade/registration flow carrying the operator id, target tier/scope, and a `return` URL so the operator lands back at the original action. The `remedy.url` SHOULD derive from the platform's `registration_url` (§7) plus gap-describing params, so platforms already publishing `registration_url` get most of this for free.
- `audience` — echoes the platform's `aud` (§4.3) so a retrying agent mints the AIT with the right `aud` without a separate fetch.
- `challenge` — OPTIONAL opaque nonce. If present, a retrying agent SHOULD echo it in the presented AIT, binding the presentation to this request (per-request replay protection atop `aud`). This is the seam to sender-constrained AITs (§4.3.1): when DPoP/`cnf` are in play, `challenge` becomes the nonce carrier (the DPoP server-nonce mode).

**Graceful degradation.** The inline challenge is strictly an OPTIMIZATION over the pull model, never a replacement, and neither side is REQUIRED to implement it. An agent that only reads `/.well-known/axis-access` still works everywhere (it just misses the inline remedy); a platform that returns a bare 401/403 still works with every agent (it just costs the extra fetch). Cannot break anything built against v0.2.

**Agent behavior.** Agents SHOULD surface `message` + `remedy` to their operator and MAY auto-follow `remedy.url` only for human-gated authority-creating steps — an agent may present authority, never grant itself authority.

**Status — partially shipped (platform-side).** Production platform-side verifiers now implement this section's deny taxonomy: every `401` carries the parameterized `WWW-Authenticate: AXIS` challenge with a structured body, and `axis_error` is emitted wherever the enum above names the condition — `no_credential`, `expired`, `agent_revoked`, `operator_revoked`, `insufficient_scope` (with `missing_scopes` named), and `insufficient_verification`. Shipped bodies also carry a granular platform `code` alongside `axis_error` (e.g. `agent_suspended` with `axis_error: "agent_revoked"`), so the enum is the stable machine surface and the code preserves diagnostic detail. The richer `required`/`current`/`remedy` blocks and the `challenge` nonce are specified but not yet shipped anywhere. The reference registry itself (acting as a platform for its own endpoints) does not emit the inline-challenge body.

**Layering note (shipped behavior).** Platform verifiers in production keep deny layers explicitly distinguishable in the body (a `layer` field in shipped implementations): a *protocol-layer scope denial* (`axis_error: "insufficient_scope"` — the credential chain does not cover the required scope), a *protocol-layer verification denial* (`axis_error: "insufficient_verification"` — the operator tier is below the platform's advertised `minimum_verification_level`), and a *platform-policy denial* (the platform's own admission rules — operator blocklists, closed or unconfigured gates, resource-level policy — rejected an otherwise-valid, in-scope credential). Policy denials use platform-defined codes (`operator_blocked`, `gate_closed`, etc.) and carry no `axis_error`; `axis_error` values are reserved for the protocol-layer conditions enumerated above. Platforms SHOULD keep the layers distinguishable in their deny bodies so agents can tell "get a better credential" from "this platform said no."

## 8. Verification procedure

A verifier receiving a request from an agent MUST perform the following steps in order. If any check fails, the verifier MUST reject the request.

### Step 1 — Authenticate the agent

1. Extract the AIT and (v0.3) DPoP proof from the request:
   - AIT, bearer presentation (shipped): from the `X-AXIS-Token` header, `Authorization: Bearer <AIT>`, or the `?ait=` query parameter (§4.3 Presentation channels; the reference platform SDK checks them in the order Bearer → `X-AXIS-Token` → query). Bearer presentations are accepted when the platform advertises `proof_of_possession: "bearer_allowed"` (or predates PoP).
   - AIT, sender-constrained presentation (v0.3, specified not shipped): from `Authorization: DPoP <AIT>`, with the DPoP proof from the `DPoP` header.
2. Parse the AIT's header and payload.
3. Validate claims: `iat`, `exp`, `aud`, `axis_version`, `iss`, `registry_url`.
4. **(v0.3) Verify the declared registry is legitimate** before trusting any record from `registry_url`. This extends Step 1 with the CA-trust check:
   - a. Extract `registry_url` (and the `registry` slug from the `did:axis` form).
   - b. Fetch that registry's `/.well-known/axis-registry` (§6.13); verify the manifest's self-signature against any active key in its `keys`.
   - c. Fetch (or read cached) Prime's signed root directory `/.well-known/axis-directory` (§6.15); verify `root_signature` against the **pinned Prime root public key**; reject if the directory is expired or its `directory_version` is lower than one already seen (rollback protection).
   - d. Confirm the registry's `registry_id` is listed `certified` in the directory with at least one key fingerprint matching the manifest's keys (fingerprint = lowercase hex SHA-256 over the raw public-key bytes). A self-declared registry that is absent, suspended, or revoked fails here. The whole check fails closed: any missing/malformed input or failed signature yields rejection.
   - Verifiers MAY maintain an allowlist of trusted registries as an alternative or supplement to the directory check; the directory is the protocol's federated-trust mechanism.
5. Fetch the AIR from `registry_url`.
6. Verify the AIT signature using the `public_key` from the AIR.
7. Confirm `iss` matches the agent_id in the AIR.
8. **(v0.3) Proof of possession.** If the platform requires PoP (`proof_of_possession: "required"`, OR the AIT carries `cnf.jkt`):
   - a. Reject if the `DPoP` header is missing.
   - b. Parse the DPoP proof JWT.
   - c. Compute the RFC 7638 JWK thumbprint of the proof's embedded `jwk` and compare to the AIT's `cnf.jkt`. Reject on mismatch.
   - d. Verify the proof signature against the embedded `jwk`. Reject on failure.
   - e. Confirm `htm` matches the HTTP method and `htu` matches the request URI (normalized per RFC 9449 §4.2; query string excluded by default unless the platform documents otherwise).
   - f. Confirm `iat` is within the freshness window (default ±60 s).
   - g. Compute base64url SHA-256 of the AIT and compare to `ath`. Reject on mismatch.
   - h. Reject if `jti × jkt` is already in the replay cache; otherwise record it with TTL ≥ the remaining freshness window. If the replay cache is unavailable, reject (fail closed).

### Step 2 — Check agent status

1. Call the `revocation_url` from the AIR.
2. If `revoked: true`, reject.
3. If `status` in the AIR is not `"active"`, reject.

### Step 3 — Verify the delegation chain

If the request requires authority beyond the agent's identity (i.e. a DC is present):

1. Walk the chain from the agent's DC back to the root:
   - For each DC, verify its `proof` signature using the issuer's public key (operator's key if `issued_by` is the operator; delegating agent's key otherwise). The verified bytes are the RFC 8785 JCS canonicalization of the DC minus its `proof` field (§4.4 DC proof canonicalization); apply the `proofType` regime rules from §4.4 (JCS-first with deprecated legacy fallback when `proofType` is absent; reject unrecognized `proofType` values distinctly).
   - Confirm `root_operator` is byte-for-byte identical across all credentials.
   - Confirm each DC's `scope` is a subset of its parent's `scope`.
   - Confirm `created <= now < expires` for each DC.
   - If `max_sub_delegation_depth` is specified in any constraint, confirm it is respected downward.
   - Check each DC's revocation status at its `revocation_url`.
   - **(v0.3) Ephemeral-delegate branch.** If a DC's `issued_to` matches the ephemeral pattern (`axis:{operator}:{agent}:sub:{task-id}`, §4.4.2):
     - Confirm `issued_to_public_key` is present (reject if absent).
     - Verify the delegate's action signature against `issued_to_public_key`. Do NOT attempt registry resolution for the ephemeral `issued_to`, and do NOT expect the delegate to present its own AIT — the parent's AIT (with `dlg` pointing at this ephemeral DC) carries the chain.
     - `revocation_url` MAY be absent for an ephemeral DC; when absent, rely on `expires`. All other invariants (attenuation, `root_operator`, expiry, scope-subset) apply unchanged.
2. Confirm the chain's top is signed by the root operator (verify with OIR's public key).
3. Confirm the chain's bottom names the acting agent (for an ephemeral leaf, the bottom is the ephemeral DC, and its `scope` is the effective scope the request must fit inside).

### Step 4 — Check scope

Confirm the action being requested is within the innermost DC's `scope` (and, per §4.4.1, that the requested scope is a recognized standard scope or a valid `x-<vendor>:` custom scope). If not, reject.

### Step 4.5 — Verify message signature when required (v0.3)

If the requested scope appears in the platform's `requires_message_signature` list (§7):

1. Reject if `Signature-Input` and `Signature` headers are missing.
2. Parse the signature base per RFC 9421 §2 using the canonical AXIS component set (`"@method"`, `"@target-uri"`, `"@authority"`, `"content-digest"`, `"date"`, `"axis-ait"`).
3. Recompute `Content-Digest` from the received body; reject on mismatch with the `Content-Digest` header.
4. Recompute `Axis-Ait` as base64url SHA-256 of the AIT; reject on mismatch with the request header.
5. Verify the signature against the agent's public key from the AIR (the `keyid` in `Signature-Input` MUST equal `iss`).
6. Confirm `created` is within the freshness window (±60 s).

### Step 5 — Accept

If all checks pass, accept the request.

### Step 6 — Retain the artifact (SHOULD)

Platforms SHOULD retain the full credential chain at time of action. The chain is the forensic artifact that answers "who authorized this?" after the fact. This is the primary evidence for regulatory frameworks like EU AI Act Article 12 and HIPAA §164.312(b). (v0.3) When PoP is in use, the retained artifact SHOULD also include the DPoP proof JWT and, when present, the `Signature`, `Signature-Input`, `Content-Digest`, and `Axis-Ait` headers — these are the per-request signed evidence that proves what the agent committed at the moment of action, not merely that it held a token.

## 9. Worked example: cross-operator agent hire

Scenario: Operator A hires an external specialist (Agent B) from a different AXIS registry, who produces work reviewed by Operator A's internal editor (Agent C), which is then published to a third-party platform (Ghost CMS) with no prior relationship to any party.

**Actors:**

- `widget-corp` — hiring operator, at `registry.axisprime.ai`
- `axis:widget-corp:editor` (Agent C) — internal editor
- `axis:external-co:specialist` (Agent B) — external specialist, at `registry.externalco.com`
- `ghost.example.com` — receiving platform

### Step 1 — Operator A evaluates external specialist

Operator A fetches `axis:external-co:specialist` from `registry.externalco.com`. Retrieves TAs about the specialist. Verifies each TA using the specialist's public key (resolvable from the foreign registry).

### Step 2 — Operator A issues a DC to the specialist

```json
{
  "axis_version": "0.2",
  "type": "DelegationCredential",
  "id": "dc:widget-corp:specialist-hire-2026-04",
  "issued_by": "widget-corp",
  "issued_to": "axis:external-co:specialist",
  "root_operator": "widget-corp",
  "scope": ["content:read", "content:create", "content:update"],
  "constraints": {
    "max_sub_delegation_depth": 0,
    "require_editor_review_before_publish": true
  },
  "created": "2026-04-01T00:00:00Z",
  "expires": "2026-04-30T00:00:00Z",
  "revocable": true,
  "revocation_url": "https://registry.axisprime.ai/revocation/credentials/dc:widget-corp:specialist-hire-2026-04",
  "proof": { "...": "signed by widget-corp operator key" }
}
```

### Step 3 — Specialist produces content

Agent B uses its AIT (signed from `registry.externalco.com`) plus the DC from Step 2 to submit a draft to Agent C.

### Step 4 — Editor reviews and publishes

Agent C accepts the draft, produces a Content Provenance Attestation signed by Agent C, and sends a publish request to `ghost.example.com` carrying:

- Agent C's AIT
- Agent C's DC from the operator
- Agent B's DC (Step 2)
- Agent B's AIT
- The CPA binding content to Agent B's delegation and Agent C's review

### Step 5 — Ghost verifies

Ghost has no prior relationship to any party. Following §8:

1. Resolves `axis:widget-corp:editor` from `registry.axisprime.ai`. Verifies Agent C's AIT signature.
2. Checks Agent C's revocation.
3. Verifies Agent C's DC (signed by `widget-corp` operator, scope valid, `root_operator = widget-corp`).
4. Fetches `widget-corp` OIR from `registry.axisprime.ai`. Retrieves operator public key.
5. Verifies Agent B's DC using operator public key. Confirms `root_operator` matches (byte-for-byte).
6. Resolves `axis:external-co:specialist` from `registry.externalco.com` (foreign registry). Verifies Agent B's AIT signature using the specialist's public key from that foreign registry.
7. Checks revocation on both agents and both DCs.
8. Verifies the CPA's signature.
9. Confirms `content:publish` is within the scope of Agent C's DC (the publish request is Agent C's; the specialist's DC covered only `content:read` / `content:create` / `content:update`).

All checks pass. Ghost accepts the publish request. At no point did it call back to any proprietary registry or require pre-configuration with either operator.

## 10. Compliance cross-references

AXIS provides primitives that regulated industries increasingly need for AI agent governance. This section maps AXIS concepts to specific regulatory frameworks. See the detailed mapping articles for full treatment.

### 10.1 EU AI Act Article 12 (record-keeping for high-risk AI systems)

AXIS supports Article 12 obligations via:

- **Automatic event recording** — every DC-authorized action produces a signed artifact
- **Traceability to a living person** — delegation chains root at a human operator's signing key
- **Authority scope at time of action** — embedded in the DC
- **Delegation chain integrity** — signed chain is a single tamper-evident artifact
- **Revocation-aware verification** — revocation state checked at time of action
- **Cross-system audit reconstruction** — the signed chain reconstructs the event without distributed system state

AXIS does NOT solve: risk analysis (Art. 9), data governance (Art. 10), technical documentation (Art. 11), accuracy/robustness (Art. 15), fundamental rights impact assessment (Art. 27).

### 10.2 HIPAA 45 CFR 164.312 (Technical Safeguards)

AXIS supports HIPAA Security Rule obligations via:

- **Unique user identification** (164.312(a)(2)(i)) — each agent has a unique canonical ID bound to a public key
- **Person or entity authentication** (164.312(d)) — every action signed with a verifiable key
- **Audit controls** (164.312(b)) — signed credential + action + verification result as one artifact
- **Access management** (164.308(a)(4)) — scope enforcement via DCs
- **Termination procedures** (164.308(a)(3)(ii)(C)) — revocation takes immediate effect
- **BAA enforcement** (164.502(e), 164.504(e)) — CE → BA → subcontractor chain as cryptographic DCs

AXIS does NOT solve: encryption at rest (164.312(a)(2)(iv)), transmission security (164.312(e)), integrity of ePHI itself (164.312(c)), physical safeguards (164.310), risk analysis (164.308(a)(1)), training (164.308(a)(5)).

### 10.3 W3C DID Core

AXIS agent IDs are compatible with W3C Decentralized Identifiers. v0.3 formalizes the `did:axis` method.

**Method name.** The method name is `axis`. A `did:axis` DID conforms to W3C DID Core.

- **Identifier syntax (v0.2 canonical, unchanged in v0.3):** `did:axis:{registry-slug}:{operator-slug}:{agent-slug}` (e.g. `did:axis:prime:widget-corp:editor`). The operator namespace prevents global slug squatting in the registry's DID space. Each method-specific identifier segment matches `[a-z0-9-]+`; the `registry-slug` is the certified registry id (the `registry_id` in that registry's `/.well-known/axis-registry` manifest, §6.13).
- **Identifier syntax (v0.1 legacy):** `did:axis:{registry-slug}:{agent-slug}` (e.g. `did:axis:prime:editor`). Remains resolvable; registries SHOULD return both forms in agent records during the transition window. After v0.3, v0.1 DID forms become resolve-only and new registrations only accept the v0.2 form.
- **Create / Update / Deactivate.** Creation happens via registry registration (`POST /register`, §6.1); the DID is minted by the certifying registry. Update and deactivation follow the AIR's `status` lifecycle (`active` / `suspended` / `revoked`) and the registry's revocation surface (§6.5, §6.7). There is no on-ledger create/update operation — `did:axis` is a registry-anchored method, and the registry's legitimacy is itself verifiable via the CA-trust model (§6.13, §6.15, §8 Step 1).
- **Resolution.** Resolve via the agent's `registry_url` using `GET /resolve/:did` (§6.11). The resolver first establishes the declared registry's legitimacy (§8 Step 1, the manifest + pinned-root-directory check), then returns a DID Document derived from the AIR. Resolvers MUST accept both v0.1 and v0.2 DID forms.
- **DID Document.** Derived from the AIR, exposing the verification key (`verificationMethod` with `Ed25519VerificationKey2020`-shape material from the AIR `public_key`), `authentication` and `assertionMethod` referencing that key, and the optional `service` array (§5.1 `service_endpoints`). The relative key id convention is `#key-1` (matching the DC `proof.verificationMethod`, e.g. `did:axis:prime:widget-corp:editor#key-1`).
- **Security and privacy.** Resolution integrity rests on the registry-legitimacy chain (§8 Step 1) and the AIT/DC signature checks (§8); privacy follows the Registry Data Visibility Model (§5) — a resolved DID Document exposes only public-layer material unless presentation context is supplied.

### 10.4 Compliance posture

AXIS is identity infrastructure, not an AI system or a data processor. The protocol itself does not make compliance claims; it provides primitives (cryptographic attribution, delegation, revocation) that platforms and registries can use to meet regulatory obligations.

**EU AI Act.** AXIS is not a "high-risk AI system" under the Act and is not itself subject to it. However, AXIS enables platforms building on top of it to meet Act requirements around agent traceability, human oversight, and transparency. See §10.1 for the Article 12 mapping.

**GDPR and data protection.** Registries handling EU operators' personal data (email, phone, KYB documents) act as data controllers under GDPR. Three implications for registry implementations:

- The Registry Data Visibility Model (§5) supports data minimization by default: only material needed for cryptographic verification is publicly exposed.
- Right-to-be-forgotten interacts with permanent cryptographic attribution. Registry operators SHOULD separate PII (name, contact, KYB evidence) from identity records (public keys, revocation state) so erasure requests can remove PII without invalidating the audit trail. Revocation, not record deletion, is the protocol's canonical response to account closure.
- Cross-border transfer obligations (e.g. Standard Contractual Clauses for non-EU registrars handling EU subjects) are a registrar concern, not a protocol concern.

The protocol is largely agnostic to GDPR mechanics; compliance lives at the registrar implementation layer.

**Explicit non-goals.** AXIS v0.1 does not provide:

- HIPAA conformance for the protocol itself. §10.2 maps AXIS primitives to HIPAA Security Rule obligations; implementations serving covered entities or business associates are responsible for the full set of administrative, physical, and technical safeguards under 45 CFR 164 Subparts C and D. AXIS is one technical control among many.
- Industry-specific profiles (financial services, healthcare, government). Profiles on top of AXIS for specific verticals may be defined by domain groups; they are out of scope for the base protocol.
- SOC 2, ISO 27001, or other organizational certifications. These attest to a registrar's operational practices, not the protocol.

**Standards alignment.** AXIS is designed to sit inside existing identity and credential standards rather than supplant them. See §10.3 (formal `did:axis` method), the §17 roadmap for the planned W3C VC-compatible encoding, and the Informative references (§15) for related specifications (W3C zcap-spec, MCP).

## 11. Security considerations

### 11.1 Key management

- Private keys MUST be stored securely. Exposure of an agent's private key allows any party to impersonate the agent until revocation propagates.
- Operators SHOULD implement key rotation procedures. A formal agent/operator key-rotation protocol is a planned v0.3 deliverable, not yet specified (see §17). Registry signing-key rotation IS specified via the `/.well-known/axis-registry` manifest's `keys` array with overlapping validity windows (§6.13); it is distinct from agent/operator rotation.
- Registries MUST NOT store private keys. Private keys are held by the operator or agent.

### 11.2 Revocation latency

Revocation is asynchronous. A verifier checking revocation at time T may miss a revocation that occurred at time T−ε if the registry's propagation is not instant. Platforms SHOULD apply short cache TTLs (<5 minutes) on revocation lookups for high-stakes actions.

### 11.3 Replay attacks

AITs carry `iat` and `exp` claims. Verifiers MUST reject AITs outside this window. Short AIT lifetimes (<24h) limit replay exposure. v0.2's REQUIRED `aud` claim closes cross-platform replay.

**Bearer-mode same-platform hardening (shipped).** Until sender-constrained AITs ship, the reference platform implementations layer three bearer-mode controls: (1) the `aud` match, (2) a platform-enforced max-TTL ceiling on `exp` (rejecting long-lived tokens regardless of what the issuer set), and (3) **single-use `jti`**: the platform records each consumed `jti` until the token's `exp` and rejects re-presentation (the reference commenting platform returns `409` with code `ait_replay`, and rejects tokens missing `jti` outright). Platforms adopting this pattern SHOULD fail closed when the replay store is unavailable. This makes a captured AIT worthless after first use at that platform, at the cost of requiring issuers to mint per-request tokens — cheap, since minting is a local signature.

v0.3's sender-constrained AITs (`cnf.jkt` + DPoP, §4.3.1) close the remaining same-platform leaked-AIT replay class: a captured AIT presented without simultaneous control of the agent's private key is rejected, and the `jti × jkt` replay cache rejects a re-presented DPoP proof. Platforms advertising `proof_of_possession: "required"` enforce this; the optional RFC 9421 message-signature profile additionally binds the request body.

### 11.4 Chain-rerooting attacks

Prevented by the `root_operator` byte-for-byte invariant. Verifiers MUST reject any chain where intermediate credentials assert a different `root_operator` than the chain's claimed root.

### 11.5 Scope widening attacks

Prevented by the attenuation rule. Verifiers MUST reject any DC whose `scope` is not a subset of its parent's `scope`.

### 11.6 Registry impersonation

In v0.1/v0.2, a verifier trusts the `registry_url` declared in the AIT. A compromised agent could point at a malicious registry that issues arbitrary public keys. v0.3 closes this with the CA-trust registry-legitimacy model (§6.13, §6.15, §8 Step 1): a verifier trusts records from a declared registry only when the registry's self-signed manifest verifies AND the registry is listed `certified`, with a matching key fingerprint, in Prime's root directory, whose `root_signature` verifies against the **pinned Prime root public key**. A self-declared, uncertified registry fails the legitimacy check before any AIR is trusted. Supplementary mitigations:

- Verifiers MAY maintain an allowlist of trusted registries in addition to the directory check.
- High-security verifiers MAY require `/.well-known/axis-access` policy alignment before accepting credentials.

The root public key is the single out-of-band trust anchor (pinned in the conformance suite and SDKs); directory rollback is prevented by the monotonic `directory_version`.

### 11.7 Denial of service

Public endpoints (§6.2, §6.3, §6.4, §6.5, §6.6) are rate-limit candidates. The reference implementation imposes no rate limits at v0.1 (relies on Cloudflare's edge protections). Compliant implementations MAY rate-limit at their discretion.

### 11.8 Cryptographic agility

v0.1 through v0.3 require Ed25519. Future versions may add alternative algorithms. Algorithm agility / negotiation is deferred — there is no current driver, and it is explicitly NOT part of v0.3 (see §17). Implementations SHOULD nonetheless avoid hard-coding assumptions that would make a later algorithm addition costly.

## 12. Privacy considerations

### 12.1 Registry Data Visibility Model

See §5. Public endpoints expose only what is required for cryptographic verification. Personally identifiable operator details are gated behind presentation context or registrar relationships.

### 12.2 Query logging

Verification queries carry the agent being verified + the querying party's IP. Registries SHOULD implement data minimization: query logs SHOULD NOT be retained beyond what is operationally necessary. The reference implementation retains query logs for 7 days for rate-limiting and abuse detection only.

### 12.3 Presentation-context authentication

The presentation layer is gated by the presenter's AIT. Verifiers fetching the presentation layer attest to having received credentials from the agent. This aligns with the "you see my details because I showed up at your door" model.

### 12.4 Cross-system delegation and PII

When an AXIS operator issues a DC to a foreign DID (e.g. `did:key:...`), the delegation credential is signed and publishable. Operators SHOULD NOT embed PII about the delegate in DC constraints; constraints are operational parameters, not identity statements.

## 13. Conformance criteria

A registry is AXIS-compliant if and only if:

1. All endpoints in §6.1 through §6.12 are implemented and return responses matching the specified schemas
2. The Registry Data Visibility Model (§5) is correctly enforced
3. Ed25519 signatures on all records and credentials are correctly validated
4. The attenuation rule and root-operator invariant (§4.4, §8) are enforced in any delegation-chain validation the registry performs
5. Revocation endpoints return timely, accurate status
6. AIT verification follows §8

**v0.3 additions.** A v0.3-compliant registry additionally:

7. Publishes a self-signed registry manifest at `/.well-known/axis-registry` (§6.13) and a standard scope vocabulary at `/.well-known/axis-scopes` (§6.14); the certifying root publishes the signed root directory at `/.well-known/axis-directory` (§6.15).
8. Enforces the two-layer scope namespace (§4.4.1) when validating scope sets: standard scopes are a closed enum, custom scopes carry the `x-<vendor>:` prefix, domain wildcards and unprefixed non-standard scopes are rejected.
9. Accepts the ephemeral `issued_to` form with `issued_to_public_key` in delegation-chain validation (§4.4.2, §8 Step 3).

A v0.3-conformant *verifier* additionally performs the registry-legitimacy check against the pinned Prime root (§8 Step 1) and, when it advertises `proof_of_possession: "required"`, enforces DPoP proof-of-possession (§4.3.1, §8 Step 1.8).

An implementation MAY extend AXIS with additional endpoints, fields, or constraints, provided the base contract remains unchanged. Implementations SHOULD document extensions.

**Spec compliance vs. registry conformance.** This document defines the wire-format contract: record shapes, AIT structure, endpoint schemas, verification semantics. Runtime-behavior requirements (authentication mechanism, ownership-scoped authorization, audit logging, retention, key management) are normative for any registry that wants to call itself "AXIS-conformant" and live in a separate document, [Registry Conformance v0.1](https://github.com/MachinesOfDesire/axis-conformance). Spec-compliant + conformance-compliant is the bar for production registries.

## 14. IANA considerations

The following well-known URIs are defined by this specification:

- `/.well-known/axis-access` — platform access policy (§7)
- `/.well-known/axis-registry` — registry self-identification manifest (v0.3, §6.13)
- `/.well-known/axis-scopes` — recognized scope vocabulary (v0.3, §6.14)
- `/.well-known/axis-directory` — root registrar directory (v0.3, §6.15)
- `/.well-known/axis-versions` — registry supported protocol versions (v0.4, §6.16; specified, not yet shipped)

AXIS proposes registration of these URIs with IANA following the procedures in [RFC 8615].

## 15. References

**Normative:**

- [RFC 2119] — Key words for use in RFCs
- [RFC 8174] — Ambiguity of uppercase vs lowercase in RFC 2119 key words
- [RFC 7519] — JSON Web Token (JWT)
- [RFC 8037] — CFRG Elliptic Curve Diffie-Hellman (ECDH) and Signatures in JWS
- [RFC 8615] — Well-Known Uniform Resource Identifiers
- [RFC 4648] — Base64 Encodings
- [RFC 8785] — JSON Canonicalization Scheme (JCS)
- [RFC 7638] — JSON Web Key (JWK) Thumbprint
- [RFC 7800] — Proof-of-Possession Key Semantics for JWTs
- [RFC 9449] — OAuth 2.0 Demonstrating Proof of Possession (DPoP)
- [RFC 9421] — HTTP Message Signatures (optional message-signature profile, §4.3.1)
- [RFC 9530] — Digest Fields (Content-Digest for the message-signature profile)
- [W3C DID Core] — Decentralized Identifiers v1.0
- [W3C Data Integrity] — Verifiable Credential Data Integrity

**Informative:**

- EU AI Act (Regulation (EU) 2024/1689), particularly Articles 9-17, 26, 27, 50, 72, 79
- HIPAA 45 CFR Part 164, Subpart C (Security Rule)
- W3C zcap-spec — Authorization Capabilities
- MCP (Model Context Protocol) — Anthropic
- AIP (Agent Identity Protocol) — provai-dev, IETF Internet-Draft format. Sibling effort in the agent-identity space, with a different design balance: OAuth 2.1-aligned grant ceremonies, W3C DID-first foundation, threat-model-driven tiering, and a registry push-notification channel. AXIS optimizes instead for leanness, cross-registry portability, and pull-based verification.

## 16. Acknowledgements

AXIS was extracted from a production system and refined through discussion in the W3C DID Working Group (issue #155), NIST NCCoE concept paper submission, and technical review by collaborators. Specific thanks to contributors listed in `CONTRIBUTORS.md`.

---

## 17. Roadmap

AXIS is in active development. Forward-looking work is planned for future versions; all items are targets, not commitments.

**v0.2 — shipped.** `dlg` claim in AIT payload (§4.3); REQUIRED `aud` claim with `audience` advertisement in `/.well-known/axis-access` (§4.3, §7); scope grammar stabilization (§4.4); operator-namespaced DIDs `did:axis:{registry}:{operator}:{agent}` (§4.1, §10.3); RFC 8785 JCS canonicalization for proof bodies with `proofType` field (§6.1); presentation-layer unlock clarification (§5.2; landed in v0.1.1); `service_endpoints` documented in public layer (§5.1; landed in v0.1.1); reference to Registry Conformance v0.1 (§13).

**v0.3 — specified in this release.** Sender-constrained AITs — `cnf.jkt` proof-of-possession + DPoP, with the optional RFC 9421 message-signature profile and `proof_of_possession` advertisement (§4.3.1, §7, §8 Step 1.8, Step 4.5); ephemeral within-runtime sub-agent delegates — ephemeral `issued_to` + `issued_to_public_key` + `task_spec_hash` (§4.4.2, §8 Step 3); standard scope vocabulary, two-layer namespace, and the `x-<vendor>:` custom prefix (§4.4.1, Appendix B); `/.well-known/axis-scopes` discovery manifest (§6.14); registry-legitimacy CA-trust model — `/.well-known/axis-registry` self-manifest and `/.well-known/axis-directory` root directory with the pinned-root-key verification step (§6.13, §6.15, §8 Step 1); inline challenge-on-refusal 401/403 body (§7.1, OPTIONAL); formal `did:axis` method (§10.3); `access_policy` extensions `blocked_operators`/`approved_operators`/`rate_limits` (§7).

**v0.3 — planned (designed as roadmap, not specified here).** W3C VC-compatible encoding for AIT/DC/TA — optional JSON-LD envelope alongside the native JWT form (deliverable #2; full design TBD — see the VC-encoding candidate stub for problem statement and open questions); agent/operator key rotation protocol — multi-key AIR with validity windows or signed rotation records, plus reconciliation with `cnf.jkt` PoP binding (deliverable #4; full design TBD — see the key-rotation candidate stub). Both remain to be specified before v0.3 ratifies. Also planned: Trust Attestation aggregation and scoring, agent notification protocol, delegation credential constraint enhancements, **Evidence Record Types** (signed wire-format records for AI-disclosure receipts, AI-decision notification ledger entries, human-oversight assertions, and Fundamental Rights Impact Assessment attestations — sized for EU AI Act Art. 50, Art. 14, Art. 26(11), and Art. 27 evidence respectively; see ROADMAP.md), client SDK specifications, registrar compliance attestations, AXIS Skills Protocol, agent rental, deprecation of v0.1 DID forms (resolve-only), removal of legacy proof canonicalization.

**v0.4 — specified in this document, not yet shipped.** Protocol version signaling: the AIT `axis_version` claim as a SHOULD-emit (§4.3.2) and the `/.well-known/axis-versions` supported-versions discovery endpoint (§6.16). Also targeted for v0.4: wiring the two-layer scope-vocabulary classifier into the registry delegation write path (§4.4.1 enforcement-status note) and promoting the reserved own-content comment scopes (Appendix B) to enforced standard scopes as platform support lands.

**Deferred — no current driver.** Cryptographic algorithm agility / negotiation (§11.8): AXIS remains Ed25519-only through v0.3; algorithm negotiation is explicitly out of v0.3 scope and revisited only when a concrete driver (e.g. a post-quantum migration mandate) appears.

**v1.0 — stability target.** Non-breaking-change threshold. Data-model and API-contract surfaces stable under semantic versioning. Pre-v1.0 releases may contain breaking changes between minor versions; implementers should track `CHANGELOG.md` closely.

**Out of scope.** Industry-specific profiles (HIPAA, financial services, etc.), gateway middleware specification, central operator slug registry, agent runtime specification, payment processing.

For full rationale, descriptions, and context on every item above, see [ROADMAP.md](./ROADMAP.md).

---

## Appendix A — Version history

See [CHANGELOG.md]. The versioning policy appears in §17.

## Appendix B — Standard scope vocabulary (v0.3)

The standard scope vocabulary layers a shared *meaning* over the v0.2 scope grammar (§4.4, §4.4.1). Standard scopes carry NO vendor prefix; their first segment is one of the reserved domains below and the whole scope is one of the recognized strings. The standard set is a **closed enum** — a grammar-valid scope under a reserved domain that is not listed here is INVALID (domain squatting is rejected). Anything outside the standard set MUST be a `x-<vendor>:`-prefixed custom scope.

This table is a versioned **seed**, grown by proposal (W3C-vocabulary-first: ActivityStreams 2.0 Activity Vocabulary, schema.org Actions, W3C ODRL). It is not yet frozen canon and may change before v0.3 ratification. The `/.well-known/axis-scopes` endpoint (§6.14) publishes this table's `scope` and `description` text verbatim.

| Domain | Standard scopes | Description |
|---|---|---|
| `content` | `content:read`, `content:create`, `content:update`, `content:delete`, `content:comment`, `content:publish` | Read / create / update / delete / comment on / publish content items |
| `content` (own-content, RESERVED) | `content:comment:read`, `content:comment:edit-own`, `content:comment:delete-own` | Read comment threads / edit / retract **only comments the agent itself authored** — see below |
| `social` | `social:follow`, `social:like`, `social:announce`, `social:invite`, `social:join`, `social:leave`, `social:block`, `social:flag` | Follow, like/react, boost/re-share, invite, join, leave, block, flag |
| `commerce` | `commerce:quote`, `commerce:order`, `commerce:purchase`, `commerce:pay`, `commerce:sell`, `commerce:refund`, `commerce:offer`, `commerce:accept`, `commerce:reject` | Quote, order, purchase, pay, sell, refund, offer, accept, reject |
| `data` | `data:read`, `data:export`, `data:write`, `data:modify`, `data:share`, `data:delete` | Read, bulk-export, write, modify, share, delete data records |
| `comms` | `comms:send`, `comms:read` | Send / read messages |
| `account` | `account:read`, `account:manage` | Read account info / manage account settings |
| `scheduling` | `scheduling:read`, `scheduling:book`, `scheduling:cancel` | Read availability/bookings, create a booking, cancel a booking |
| `compute` | `compute:invoke`, `compute:read` | Invoke a compute task/function, read results or status |

**Reserved domains:** `content`, `social`, `commerce`, `data`, `comms`, `account`, `scheduling`, `compute`. A scope whose first segment is one of these but which is not a recognized string above (including domain wildcards like `content:*`) is INVALID. Custom scopes use `x-<vendor>:` where `<vendor>` matches `[a-z0-9-]+`.

**Shipped vs reserved.**

- **Shipped and enforced today:** `content:comment` is the canonical commenting scope, enforced in production — the reference commenting platform requires it (`required_scopes: ["content:comment"]` in its `/.well-known/axis-access`) and denies requests whose delegation chain does not cover it. The scope for commenting is `content:comment`, never a platform-invented string like `comments:write`. Production platform-side verifiers also enforce `data:read`, `data:write`, and `data:delete` for resource-access surfaces. The remaining standard strings above are recognized vocabulary (published via `/.well-known/axis-scopes`) that platforms adopt as they gate the corresponding actions.
- **Reserved — own-content comment scopes (specified, not yet enforced by any shipped platform):** `content:comment:read`, `content:comment:edit-own`, `content:comment:delete-own`. Semantics: the holder may read comment threads, and may edit or retract (delete) **only comments authored by the same agent identity** — the platform matches the stored authoring `agent_id` against the presenting agent and MUST reject own-content operations on comments the agent did not author. `content:comment:edit-own` and `content:comment:delete-own` are deliberately narrower than `content:update` / `content:delete` (which cover content items generally, without the authorship constraint); a platform gating comment edits SHOULD require the own-content scope, not the general one. These strings join the closed standard enum as of this revision; implementations updating their vocabulary tables MUST classify them `standard`. Note the shipped reference registry's vocabulary table predates this revision (its classifier does not yet recognize them, and its `/.well-known/axis-scopes` does not yet list them); no shipped platform enforces them yet. Enforcement is targeted alongside comment-editing surfaces (§17 v0.4).
