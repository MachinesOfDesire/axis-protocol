# Roadmap

This document describes planned additions to the AXIS protocol beyond v0.1. Items here are **not** committed features — they represent the current design intent and may change based on implementation feedback, real-world use cases, and standards-body engagement.

Breaking changes between minor versions (v0.1 → v0.2) are expected. v1.0 will be the first stable release.

## v0.2 — planned

### Registry discovery
**`/.well-known/axis-registry`** — Every compliant registry publishes a signed manifest at this well-known path. Verifiers can check the manifest independently of what any agent claims, closing the v0.1 gap where a compromised agent could declare a malicious registry URL.

### Registrar Compliance Attestations (RCA)
Four-tier registrar trust model:
- **Tier 0:** Unlisted / self-declared
- **Tier 1:** Listed / self-attested (in the Foundation directory)
- **Tier 2:** Audited (annual third-party compliance audit)
- **Tier 3:** Reference / Foundation-operated

RCAs are signed attestations issued by the AXIS Foundation (once formed) and carried by compliant registrars. Verifiers validate RCA signatures against the Foundation's public key (embedded in SDKs) to apply trust-tier policy. Mirrors the CA/Browser Forum trust model for TLS CAs.

### Operator slug tiering
Three tiers of operator identity verification, expressed as a format signal in the `operator_id`:
- **Domain-verified:** `axis:widget-corp:agent` (slug matches verified domain)
- **Registrar-namespaced:** `axis:kipple.joshashcroft:agent` (dot separator indicates namespace under a registrar's user system)
- **Auto-generated:** `axis:op-7f3a2b9c:agent` (`op-` prefix + random identifier)

Platforms can filter on the format pattern alone, without additional metadata lookups. Inherits DNS's global uniqueness for Tier 1, falls back to registrar-scoped uniqueness for Tier 2, UUID-based for Tier 3.

### Scope taxonomy
Structured replacement for v0.1's open scope namespace:

- **Standard common scopes** (no prefix): `content:read`, `content:write`, `content:publish`, `payment:authorize`, `data:query`, `email:send`, `file:read`, `file:write`. Universal recognition expected across compliant platforms.
- **Namespaced custom scopes** (prefix required): `shopify:inventory:update`, `ghost:theme:edit`. Platform-specific; the prefix tells the receiving platform "this scope is mine."
- **`/.well-known/axis-scopes`** — Platforms publish the scopes they recognize at this endpoint. Agents check it before presenting credentials.

### Trust Attestation aggregation and scoring
The v0.1 Trust Attestation is a signed record. v0.2 adds:

- **Scoring** — how to aggregate multiple attestations into a single trust signal
- **Time windowing** — total, last year, last 60 days, days without incident
- **Gaming resistance** — handling self-referential attestations, sock puppet farms, attestation spam
- **Weighting** — platform attestations worth more than peer attestations; long-established attestors weighted more than new ones
- **Negative feedback** — how bad outcomes are represented, disputed, and resolved
- **Operator-level tracking** — agent reputation tied to operator reputation (preventing whack-a-mole by spinning up fresh agents)

### Cross-system delegation
v0.1 supports delegation to foreign AXIS operators via the `issued_to` field accepting full AXIS IDs from any registry. v0.2 extends this to foreign DIDs:
- Delegation to `did:key:z6Mk...`, `did:web:...`, `did:ethr:...`, `did:peer:...`
- Standard resolution path for cross-system delegation
- VC-compatible encoding so foreign systems can process AXIS credentials without AXIS-specific tooling

### W3C VC-compatible encoding
Alongside the native AXIS encoding (the JSON structures in `SPEC.md`), Delegation Credentials and Trust Attestations will have a canonical W3C Verifiable Credentials encoding. Either form is valid; compliant registries MUST accept both.

### Key rotation and recovery
Formal protocol support for rotating an agent's or operator's Ed25519 keypair without losing identity continuity. Includes:
- Signed rotation events in the registry's audit log
- Grace period for old keys (reject after cutoff, warn before)
- Cross-registry rotation notification
- Recovery for lost keys (via operator-level reauthorization)

### `did:axis` DID method specification
Formal W3C DID method registration for `did:axis`. Includes:
- Canonical identifier syntax
- DID Document derivation from Agent Identity Record
- Resolution procedure
- Key algorithm negotiation

## v0.3+ — possible

### Behavioral reputation layer
Cross-operator track record for agents. Signed operational history from multiple platforms aggregated into a reputation score. Distinct from Trust Attestations (which are unsolicited endorsements); behavioral reputation is derived from verified platform events.

### Blockchain-backed registry option
For operators requiring a registry with no single point of control, a blockchain-anchored variant. Not a primary deployment target — most adoption will be conventional database registries — but the option matters for specific high-stakes use cases.

### Formal verification of the chain semantics
Proof-assisted or formally-verified reference implementations of credential chain validation. Particularly important as the protocol moves into regulated-industry deployments.

### Federated trust pools
Multiple registrars forming federations with shared trust-tier attestation. Useful for industry-specific certifications (e.g., a healthcare AXIS federation with additional HIPAA-aligned requirements beyond the base spec).

## Out of scope (not planned)

Items regularly proposed that we are not currently planning to add:

- **A central operator slug registry.** DNS + domain verification provides global uniqueness for Tier 1 slugs; registrar-namespaced slugs are globally unique via the registrar prefix. A central slug registry would recentralize the protocol.
- **A canonical scope taxonomy for all possible agent actions.** The protocol verifies that scopes attenuate correctly; it does not enforce what they mean. That is a receiving platform's job.
- **Agent runtime specification.** AXIS is identity and authorization, not execution. We integrate with MCP for the skill layer, but runtime choice is deliberately outside scope.
- **Specific encryption or transmission security beyond TLS.** Transport layer remains the deployment's concern.
- **Payment processing.** Registrars may charge for registration (AXIS Prime does); the protocol has no notion of money.

## Process

Changes to this roadmap happen via:
1. GitHub issues tagged `v0.2`, `v0.3`, or `future`
2. Discussion in the relevant issue before any PR is written
3. RFC-style design documents for major additions, linked from the corresponding issue
4. Formal commitment only when an item moves from this roadmap to `CHANGELOG.md [Unreleased]`

Suggestions and discussion welcome. Open an issue to propose a roadmap change.
