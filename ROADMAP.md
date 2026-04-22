# Roadmap

This document is the expanded narrative treatment of the protocol's forward roadmap. For the terse manifest inside the specification, see [SPEC.md §17](./SPEC.md#17-roadmap). Items listed here are **not** committed features — they represent current design intent and may change based on implementation feedback, real-world use cases, and standards-body engagement.

Breaking changes between minor versions (v0.1 → v0.2) are expected. v1.0 will be the first stable release; see [SPEC.md §17.3](./SPEC.md#173-target-v10-stability) for the versioning policy.

## v0.2 — planned

The v0.2 release is focused on **hardening the cross-registry trust model, standardizing platform-facing discovery, and expanding the credential surface**. All items below are captured as terse bullets in [SPEC.md §17.1](./SPEC.md#171-planned-for-v02); this document adds rationale.

### Registry discovery — `/.well-known/axis-registry`

Every compliant registry publishes a signed manifest at this well-known path. Verifiers can check the manifest independently of what any agent claims, closing the v0.1 gap where a compromised agent could declare a malicious registry URL. The manifest identifies the registry's operator, its governance model, its accreditation tier (see RCA below), and its public key.

### Platform-side access policy extensions

The v0.1 `/.well-known/axis-access` endpoint (§7) publishes minimum verification level, required scopes, and an `allow_unverified` boolean. v0.2 adds:

- `blocked_operators` — explicit deny list for specific operators
- `approved_operators` — explicit allow list (when the platform wants a whitelist model)
- `rate_limits` — per-agent or per-operator rate-limit declarations

### Registrar Compliance Attestations (RCA)

Four-tier registrar trust model mirroring the CA/Browser Forum pattern for TLS CAs:

- **Tier 0** — Unlisted / self-declared
- **Tier 1** — Listed / self-attested in the Foundation directory
- **Tier 2** — Audited (annual third-party compliance audit)
- **Tier 3** — Reference / Foundation-operated

RCAs are signed attestations issued by the AXIS Foundation (once formed) and carried by compliant registrars. Verifiers validate RCA signatures against the Foundation's public key (embedded in SDKs) to apply trust-tier policy.

### Operator slug tiering

Three tiers of operator identity verification, expressed as a format signal in the `operator_id` itself — no additional metadata lookup required for a platform to filter on verification level:

- **Domain-verified:** `axis:widget-corp:agent` — slug matches a verified domain
- **Registrar-namespaced:** `axis:kipple.joshashcroft:agent` — dot separator indicates a namespace under a registrar's user system
- **Auto-generated:** `axis:op-7f3a2b9c:agent` — `op-` prefix + random identifier

Inherits DNS's global uniqueness for Tier 1, falls back to registrar-scoped uniqueness for Tier 2, UUID-based for Tier 3.

### Scope discovery — `/.well-known/axis-scopes`

Platforms publish the scopes they recognize at this endpoint. Agents check it before presenting credentials; gateway products read it to render per-platform authorization UIs dynamically. This unblocks the Tier C (Scope Enforcement) adoption pattern from the positioning frame — a gateway can handle a new platform by dropping in a manifest rather than writing custom code.

The v0.3 release adds the **standard common scope vocabulary** on top of this discovery mechanism (see v0.3 below).

### Trust Attestation `credentialSubject` schema

The v0.1 Trust Attestation is a signed record with a minimal structure (§4.5). v0.2 standardizes the `credentialSubject` schema so attestations from different issuers are comparable. Aggregation, scoring, and gaming resistance are deferred to v0.3 — v0.2 just nails down the format.

### Cross-system delegation to foreign DIDs

v0.1 supports delegation to foreign AXIS operators via the `issued_to` field accepting full AXIS IDs from any registry. v0.2 extends this to foreign DIDs:

- Delegation to `did:key:z6Mk...`, `did:web:...`, `did:ethr:...`, `did:peer:...`
- Standard resolution path for cross-system delegation
- VC-compatible encoding so foreign systems can process AXIS credentials without AXIS-specific tooling

### W3C VC-compatible encoding

Alongside the native AXIS encoding (the JSON structures in `SPEC.md`), Delegation Credentials and Trust Attestations get a canonical W3C Verifiable Credentials encoding. Either form is valid; compliant registries MUST accept both. This unlocks interoperability with existing VC tooling (issuer libraries, wallets, verifiers) without forcing the AXIS ecosystem onto the full VC stack.

### Key rotation and recovery

Formal protocol support for rotating an agent's or operator's Ed25519 keypair without losing identity continuity:

- Signed rotation events in the registry's audit log
- Grace period for old keys (reject after cutoff, warn before)
- Cross-registry rotation notification
- Recovery for lost keys via operator-level reauthorization

v0.1 states only that operators SHOULD rotate; v0.2 defines how.

### Algorithm negotiation

v0.1 fixes Ed25519. v0.2 introduces a negotiation mechanism so additional signing algorithms (e.g. post-quantum candidates, platform-native hardware-backed algorithms) can be advertised in identity records and selected by verifiers. Backwards-compatible: Ed25519 remains the required baseline.

### `did:axis` DID method specification

Formal W3C DID Method Registry submission:

- Canonical identifier syntax
- DID Document derivation from Agent Identity Record
- Resolution procedure
- Key algorithm negotiation (ties to the item above)

### AXIS Skills Protocol

Deferred from v0.1 to keep the base spec focused on identity + authorization + revocation. v0.2 adds the capability marketplace layer:

- **Skill Record** — a signed declaration that an agent has a specific capability
- **Skill Credential** — a signed attestation from a platform or certifying body that the agent exercised that capability successfully in production

Built on MCP (Model Context Protocol) so skills are model-agnostic and runtime-agnostic. Mirrors the existing delegation and trust-attestation patterns.

### Platform registry public/private discovery

Discovery mechanisms for whether a platform operates a private or public registry of accepted agents, and how agents learn which platforms have pre-approved their operator.

### Agent rental / operational control transfer

Protocol-level handling for transferring operational control of an agent between operators — e.g., a hosted-agent service that hands operational control back to the customer after a trial. v0.1 has no mechanism for this; today it requires deactivating one agent and registering a new one, losing identity continuity.

### Client SDK specifications

Reference interfaces for operator-side and platform-side libraries. The specs define the interface; implementations remain the domain of ecosystem contributors. Intent: lower the bar for new registries and verifiers to achieve v0.2 conformance.

## v0.3 — planned

v0.3 extends depth and analytics on top of the v0.2 surface. Captured as terse bullets in [SPEC.md §17.2](./SPEC.md#172-planned-for-v03).

### Standard common scope vocabulary

Built on top of v0.2's `/.well-known/axis-scopes` discovery. Defines the universal scopes every compliant platform SHOULD recognize:

- `content:read`, `content:write`, `content:publish`
- `payment:authorize`
- `data:query`
- `email:send`
- `file:read`, `file:write`
- (list may grow based on implementation feedback)

Platform-specific scopes remain namespaced (`shopify:inventory:update`, `ghost:theme:edit`, etc.) and are not centrally defined — the prefix tells the receiving platform "this scope is mine."

### Trust Attestation aggregation, scoring, and gaming resistance

Builds on the v0.2 TA schema. Adds the analytical layer:

- **Aggregation** — how multiple attestations about the same agent combine into a single trust signal
- **Time windowing** — total, last year, last 60 days, days without incident
- **Weighting** — platform attestations worth more than peer attestations; long-established attestors weighted more than new ones
- **Gaming resistance** — handling self-referential attestations, sock-puppet attestor farms, attestation spam
- **Negative feedback** — how bad outcomes are represented, disputed, and resolved
- **Operator-level tracking** — agent reputation tied to operator reputation (prevents whack-a-mole by spinning up fresh agents under the same operator)

This is a significant analytical and research effort; scoping it tight to v0.3 protects v0.2 from dragging on while this work matures.

### Agent notification / asynchronous action protocol

Standardized mechanism for platforms to push events back to agents (action completed, review required, credential expiring). v0.1 and v0.2 assume agent-initiated request flow; v0.3 opens the reverse direction.

### Delegation credential enhancements

Constraint extensions requested by deployed systems:

- **Conditional constraints** — e.g. "approved iff the monthly spend cap has not been hit"
- **Multi-party authorization** — e.g. "valid only if countersigned by Operator B"
- **Time-bounded renewal** — automatic DC re-issuance with shortened validity

Driven by real-world requests from adopters, not speculative.

## v0.3+ — possible

Items under active consideration but not committed to any specific minor version.

### Behavioral reputation layer

Cross-operator track record for agents. Signed operational history from multiple platforms aggregated into a reputation score. Distinct from Trust Attestations (unsolicited endorsements); behavioral reputation is derived from verified platform events rather than explicit attestations.

### Blockchain-backed registry option

For operators requiring a registry with no single point of control, a blockchain-anchored variant. Not a primary deployment target — most adoption will be conventional database registries — but the option matters for specific high-stakes use cases (trustless cross-registry audit, immutable historical record).

### Formal verification of chain semantics

Proof-assisted or formally-verified reference implementations of credential chain validation. Particularly important as the protocol moves into regulated-industry deployments.

### Federated trust pools

Multiple registrars forming federations with shared trust-tier attestation. Useful for industry-specific certifications (e.g., a healthcare AXIS federation with additional HIPAA-aligned requirements beyond the base spec).

## Out of scope (not planned)

Items regularly proposed that we are not currently planning to add to the base protocol. Some may live in companion specifications; others are deliberately not AXIS's problem.

- **A central operator slug registry.** DNS + domain verification provides global uniqueness for Tier 1 slugs; registrar-namespaced slugs are globally unique via the registrar prefix. A central slug registry would recentralize the protocol.
- **A canonical scope taxonomy for all possible agent actions.** v0.3's common scopes are the universal minimum. Beyond that, the protocol verifies that scopes attenuate correctly; it does not enforce what they mean semantically. That is a receiving platform's job.
- **Agent runtime specification.** AXIS is identity and authorization, not execution. We integrate with MCP for the skill layer, but runtime choice is deliberately outside scope.
- **Specific encryption or transmission security beyond TLS.** Transport layer remains the deployment's concern.
- **Payment processing.** Registrars may charge for registration (AXIS Prime does); the protocol has no notion of money.
- **HIPAA, financial-services, or other industry-specific profiles.** These may emerge as companion specifications authored by domain groups, but the base protocol stays vertical-neutral.
- **Gateway middleware specification.** Gateway products are deployment patterns built *on* the protocol (using `/.well-known/axis-scopes` defined in v0.2); they are not part of the protocol itself.
- **SOC 2, ISO 27001, or other organizational certifications.** These attest to a registrar's operational practices, not the protocol.

## Process

Changes to this roadmap happen via:

1. GitHub issues tagged `v0.2`, `v0.3`, or `future`
2. Discussion in the relevant issue before any PR is written
3. RFC-style design documents for major additions, linked from the corresponding issue
4. Formal commitment only when an item moves from this roadmap into `CHANGELOG.md`

Suggestions and discussion welcome. Open an issue to propose a roadmap change.
