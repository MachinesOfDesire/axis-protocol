# AXIS Protocol Specification

**Version:** 0.1
**Status:** Public v0.1 release candidate. This is the canonical AXIS Protocol v0.1 specification. Breaking changes are possible before v1.0; see §17 Roadmap and `CHANGELOG.md` for the versioning policy.
**Author:** Josh Ashcroft
**License:** Apache 2.0
**Repository:** https://github.com/MachinesOfDesire/axis-protocol
**Last updated:** 2026-04-19

## Abstract

AXIS (Agent Cross-system Identity Standard) is a protocol for autonomous AI agent identity, delegation, and authorization across operator boundaries. This document specifies the protocol's data model, API contract, verification procedures, and conformance criteria for v0.1.

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
  "axis_version": "0.1",
  "agent_id": "axis:widget-corp:editor",
  "did": "did:axis:prime:editor",
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

- `did` (string) — W3C DID form, typically `did:axis:{registry}:{agent-slug}`
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
  "sub": "axis:widget-corp:editor",
  "iat": 1743292800,
  "exp": 1743379200,
  "axis_version": "0.1",
  "operator_id": "widget-corp",
  "registry_url": "https://registry.axisprime.ai/agents/axis:widget-corp:editor"
}
```

**Required claims:**

- `iss` — issuer (the agent's AXIS ID)
- `sub` — subject (same as issuer for AITs)
- `iat` — issued at (Unix timestamp)
- `exp` — expiration (Unix timestamp)
- `axis_version` — protocol version
- `operator_id` — the agent's operator
- `registry_url` — URL where the agent's AIR can be resolved

Signed with the agent's private key. Verifiers resolve `registry_url` to retrieve the public key and verify the signature.

**Token lifetime.** AITs SHOULD have a maximum lifetime of 24 hours. Platforms MAY enforce shorter lifetimes. Agents re-mint AITs from their private key as needed.

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
  "scope": ["article:draft", "article:submit"],
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
    "created": "2026-04-01T00:00:00Z",
    "verificationMethod": "did:axis:prime:editor#key-1",
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
- `proof` (object) — cryptographic proof per [W3C Data Integrity]

**Optional fields:**

- `parent_credential_id` (string) — links to the credential that authorized the delegator. REQUIRED when `issued_by` is not the root operator.
- `constraints` (object) — additional limits. Semantics are out-of-band.
- `revocable` (boolean) — whether the credential can be revoked before expiry. Default `true`.
- `revocation_url` (string) — where to check revocation status for this specific credential

**The attenuation rule (MUST).** The `scope` of a DC MUST be equal to or a subset of the `scope` of its `parent_credential_id`. Verifiers MUST reject any DC where the delegate's scope exceeds the delegator's scope.

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

**Aggregation and scoring.** v0.1 does not specify how multiple Trust Attestations are aggregated. This is out of scope for v0.1 and planned for v0.3 (see §17.2).

**Design rationale — what a Trust Attestation is, and what it is not.** The v0.1 Trust Attestation is deliberately a signed container, not a prescribed rating system. AXIS does not specify what constitutes trust, how it is measured, or how multiple attestations combine. This is intentional: different verticals apply different standards of evidence, and prescribing a single scale in v0.1 would be either too narrow for most use cases or too abstract to be useful.

The design intent is to favor **measurable behavioral signals** (analogous to credit utilization — report observable facts, let consumers decide) over **opinion signals** (analogous to star ratings — subjective judgments without anchoring). A well-formed attestation reports events that happened, states that can be audited, or measurements that can be reproduced. A weakly-formed attestation reports feelings. v0.1 does not enforce this distinction — `statement` is free-form — but implementations SHOULD lean toward observable, signable events rather than editorial judgments. The v0.3 analytical layer (aggregation, weighting, gaming resistance — see §17.2) is designed to operate on behavioral signals; opinion signals degrade the aggregation's usefulness.

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

### 5.2 Presentation layer

Visible when an agent actively presents credentials to a receiving platform. The platform observes this data because the agent chose to interact with it.

For agents: `display_name`, `purpose`, `platform`, `capability_tier`
For operators: `display_name`, `verification_tier`, `domain`, `registered_at`

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
  }
}
```

Response (201):
```json
{
  "axis_id": "axis:example-xyz:editor",
  "did": "did:axis:prime:editor",
  "token": "<signed AIT JWT>",
  "registry_url": "https://registry.axisprime.ai/agents/axis:example-xyz:editor",
  "revocation_url": "https://registry.axisprime.ai/revocation/axis:example-xyz:editor"
}
```

### 6.2 `GET /agents/:agent_id`

Resolve an AIR. Public layer by default; presentation layer if a valid AIT is in the Authorization header.

Response: the AIR as specified in §4.1, subject to visibility tier.

### 6.3 `GET /operators/:operator_id`

Resolve an OIR. Same tiered visibility as §6.2.

### 6.4 `GET /verify?token=<AIT>`

Verify an AIT.

Response:
```json
{
  "valid": true,
  "agent_id": "axis:widget-corp:editor",
  "operator_id": "widget-corp",
  "verification_tier": "domain",
  "status": "active",
  "expires_at": "2026-04-16T00:00:00Z"
}
```

If invalid, returns `{ "valid": false, "reason": "<reason>" }` with a 200 response (the verification request itself succeeded; the token is just invalid).

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

### 6.10 `DELETE /delegations/:id`

Revoke a Delegation Credential. Requires delegator authentication.

### 6.11 `GET /resolve/:did`

W3C DID resolution. Returns a DID Document derived from the AIR.

Response: a DID Document per W3C DID Core.

### 6.12 `GET /.well-known/axis-access`

This registry's own access policy. See §7.

### 6.13 `GET /.well-known/axis-registry` (v0.2)

Registry self-identification manifest. See [ROADMAP.md].

## 7. Platform-side access control

Every receiving platform accepting agent interactions SHOULD publish its access requirements at `/.well-known/axis-access`.

Response:
```json
{
  "axis_version": "0.1",
  "platform_id": "ghost.example.com",
  "access_policy": {
    "minimum_verification_level": "domain",
    "required_scopes": ["content:comment"],
    "allow_unverified": false,
    "registration_url": "https://signup.axisprime.ai/signup"
  },
  "updated_at": "2026-04-13T00:00:00Z"
}
```

**Fields:**

- `minimum_verification_level` — lowest operator verification tier accepted
- `required_scopes` — scopes the agent must hold
- `allow_unverified` — boolean. If `true`, requests without an AIT are accepted (the platform chooses whether to treat them as verified or as anonymous traffic). If `false`, requests without an AIT are rejected.
- `registration_url` — where to direct unregistered agents or operators who don't meet the minimum tier

**v0.2 candidates:** `blocked_operators`, `approved_operators`, `rate_limits`.

**Design rationale.** The protocol defines how platforms *communicate* policy. Enforcement is the platform's responsibility. AXIS provides identity and trust signals; platforms make their own decisions.

## 8. Verification procedure

A verifier receiving a request from an agent MUST perform the following steps in order. If any check fails, the verifier MUST reject the request.

### Step 1 — Authenticate the agent

1. Extract the AIT from the request (typically `Authorization: Bearer <AIT>`).
2. Parse the AIT's header and payload.
3. Validate claims: `iat`, `exp`, `axis_version`, `iss`, `registry_url`.
4. Fetch the AIR from `registry_url`.
5. Verify the AIT signature using the `public_key` from the AIR.
6. Confirm `iss` matches the agent_id in the AIR.

### Step 2 — Check agent status

1. Call the `revocation_url` from the AIR.
2. If `revoked: true`, reject.
3. If `status` in the AIR is not `"active"`, reject.

### Step 3 — Verify the delegation chain

If the request requires authority beyond the agent's identity (i.e. a DC is present):

1. Walk the chain from the agent's DC back to the root:
   - For each DC, verify its `proof` signature using the issuer's public key (operator's key if `issued_by` is the operator; delegating agent's key otherwise).
   - Confirm `root_operator` is byte-for-byte identical across all credentials.
   - Confirm each DC's `scope` is a subset of its parent's `scope`.
   - Confirm `created <= now < expires` for each DC.
   - If `max_sub_delegation_depth` is specified in any constraint, confirm it is respected downward.
   - Check each DC's revocation status at its `revocation_url`.
2. Confirm the chain's top is signed by the root operator (verify with OIR's public key).
3. Confirm the chain's bottom names the acting agent.

### Step 4 — Check scope

Confirm the action being requested is within the innermost DC's `scope`. If not, reject.

### Step 5 — Accept

If all checks pass, accept the request.

### Step 6 — Retain the artifact (SHOULD)

Platforms SHOULD retain the full credential chain at time of action. The chain is the forensic artifact that answers "who authorized this?" after the fact. This is the primary evidence for regulatory frameworks like EU AI Act Article 12 and HIPAA §164.312(b).

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
  "axis_version": "0.1",
  "type": "DelegationCredential",
  "id": "dc:widget-corp:specialist-hire-2026-04",
  "issued_by": "widget-corp",
  "issued_to": "axis:external-co:specialist",
  "root_operator": "widget-corp",
  "scope": ["article:research", "article:draft", "article:submit"],
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
9. Confirms `article:publish` is within scope.

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

AXIS agent IDs are compatible with W3C Decentralized Identifiers. The `did:axis` method:

- **Identifier syntax:** `did:axis:{registry-slug}:{agent-slug}` (e.g. `did:axis:prime:editor`)
- **DID Document:** derived from the Agent Identity Record, exposing `publicKey`, `authentication`, and `service` endpoints
- **Resolution:** via the agent's `registry_url` using the `GET /resolve/:did` endpoint

Formal `did:axis` method specification planned for v0.2.

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

**Standards alignment.** AXIS is designed to sit inside existing identity and credential standards rather than supplant them. See §10.3 (W3C DID), the v0.2 roadmap (§17) for VC-compatible encoding, and the Informative references (§15) for related specifications (W3C zcap-spec, MCP).

## 11. Security considerations

### 11.1 Key management

- Private keys MUST be stored securely. Exposure of an agent's private key allows any party to impersonate the agent until revocation propagates.
- Operators SHOULD implement key rotation procedures. v0.1 does not define a formal rotation protocol; v0.2 will.
- Registries MUST NOT store private keys. Private keys are held by the operator or agent.

### 11.2 Revocation latency

Revocation is asynchronous. A verifier checking revocation at time T may miss a revocation that occurred at time T−ε if the registry's propagation is not instant. Platforms SHOULD apply short cache TTLs (<5 minutes) on revocation lookups for high-stakes actions.

### 11.3 Replay attacks

AITs carry `iat` and `exp` claims. Verifiers MUST reject AITs outside this window. Short AIT lifetimes (<24h) limit replay exposure.

### 11.4 Chain-rerooting attacks

Prevented by the `root_operator` byte-for-byte invariant. Verifiers MUST reject any chain where intermediate credentials assert a different `root_operator` than the chain's claimed root.

### 11.5 Scope widening attacks

Prevented by the attenuation rule. Verifiers MUST reject any DC whose `scope` is not a subset of its parent's `scope`.

### 11.6 Registry impersonation

In v0.1, a verifier trusts the `registry_url` declared in the AIT. A compromised agent could point at a malicious registry that issues arbitrary public keys. Mitigations:

- Verifiers MAY maintain an allowlist of trusted registries
- Verifiers SHOULD use v0.2 `/.well-known/axis-registry` manifests when available
- High-security verifiers MAY require `/.well-known/axis-access` policy alignment before accepting credentials

v0.2 closes this with registry-discovery attestations.

### 11.7 Denial of service

Public endpoints (§6.2, §6.3, §6.4, §6.5, §6.6) are rate-limit candidates. The reference implementation imposes no rate limits at v0.1 (relies on Cloudflare's edge protections). Compliant implementations MAY rate-limit at their discretion.

### 11.8 Cryptographic agility

v0.1 requires Ed25519. Future versions may add alternative algorithms. Implementations SHOULD be designed to support algorithm negotiation when v0.2 or later introduces it.

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

A registry is AXIS v0.1 compliant if and only if:

1. All endpoints in §6.1 through §6.12 are implemented and return responses matching the specified schemas
2. The Registry Data Visibility Model (§5) is correctly enforced
3. Ed25519 signatures on all records and credentials are correctly validated
4. The attenuation rule and root-operator invariant (§4.4, §8) are enforced in any delegation-chain validation the registry performs
5. Revocation endpoints return timely, accurate status
6. AIT verification follows §8

An implementation MAY extend AXIS with additional endpoints, fields, or constraints, provided the base contract remains unchanged. Implementations SHOULD document extensions.

## 14. IANA considerations

The following well-known URIs are defined by this specification:

- `/.well-known/axis-access` — platform access policy (§7)
- `/.well-known/axis-registry` — registry self-identification manifest (v0.2)
- `/.well-known/axis-scopes` — platform-recognized scope list (v0.2)

AXIS proposes registration of these URIs with IANA following the procedures in [RFC 8615].

## 15. References

**Normative:**

- [RFC 2119] — Key words for use in RFCs
- [RFC 8174] — Ambiguity of uppercase vs lowercase in RFC 2119 key words
- [RFC 7519] — JSON Web Token (JWT)
- [RFC 8037] — CFRG Elliptic Curve Diffie-Hellman (ECDH) and Signatures in JWS
- [RFC 8615] — Well-Known Uniform Resource Identifiers
- [RFC 4648] — Base64 Encodings
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

**v0.2 — in design.** `/.well-known/axis-scopes` manifest format, `/.well-known/axis-registry` discovery, `access_policy` extensions, formal `did:axis` method spec, Trust Attestation credentialSubject schema, W3C VC-compatible encoding, cross-system delegation to foreign DIDs, key rotation protocol, algorithm negotiation, client SDK specifications, registrar compliance attestations, operator slug tiering, AXIS Skills Protocol, platform registry discovery, agent rental.

**v0.3 — planned.** Standard common scope vocabulary, Trust Attestation aggregation and scoring, agent notification protocol, delegation credential constraint enhancements.

**v1.0 — stability target.** Non-breaking-change threshold. Data-model and API-contract surfaces stable under semantic versioning. Pre-v1.0 releases may contain breaking changes between minor versions; implementers should track `CHANGELOG.md` closely.

**Out of scope.** Industry-specific profiles (HIPAA, financial services, etc.), gateway middleware specification, central operator slug registry, agent runtime specification, payment processing.

For full rationale, descriptions, and context on every item above, see [ROADMAP.md](./ROADMAP.md).

---

## Appendix A — Version history

See [CHANGELOG.md]. The versioning policy appears in §17.
