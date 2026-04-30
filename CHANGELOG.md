# Changelog

All notable changes to the AXIS protocol specification are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

**Versioning policy.** Pre-v1.0 releases (all 0.x versions) may contain breaking changes between minor versions. v1.0 will freeze the stable contract under stricter discipline: patch releases additive only, minor releases may add optional fields or endpoints, major releases may break backward compatibility only with explicit governance approval. Track this file closely until v1.0.

## [0.1.1] — 2026-04-24

Patch release. Clarifications and corrections only; no new wire fields, no behavior changes for v0.1.0-conformant implementations.

### Clarified

- **§5.1 Public layer.** Added `service_endpoints` (W3C DID `service` array shape, optional, agents only) to the public-layer enumeration. The reference implementation already returned this field; the spec now documents it.
- **§5.1 Public layer.** Added a normative MUST that public-layer identifiers carry no PII. Specifically, `operator_id` derived from any part of an email address is non-conformant; email-tier operators receive opaque random identifiers. Domain-derived identifiers remain acceptable (the domain is already public). Cross-references the new Registry Conformance §8.1.1.
- **§5.2 Presentation layer.** Spelled out the three paths that unlock the presentation layer: (1) valid AIT in the request, (2) the owning registrar reading its own data, (3) admin/super_admin role. v0.1.0 only described path 1; the reference implementation has supported paths 2 and 3 since April 23, 2026.
- **§13 Conformance criteria.** Added a clarifying note distinguishing wire-format compliance (this document) from runtime-behavior conformance (separate document at github.com/MachinesOfDesire/axis-conformance). Production registries should satisfy both.

### Editorial

- Status banner moved from "Public v0.1 release candidate" to "Public release" on README and SPEC. The release-candidate hedging predated the v0.1.1 patch pass; it is no longer accurate.
- "Last updated" bumped to 2026-04-24.

### Not changed

No changes to: AIT signing algorithm, AIT structural format, endpoint paths, request or response schemas (other than documenting the existing `service_endpoints` field), delegation envelope, revocation mechanism, or any v0.1.0 claim name.

A consumer that implements v0.1.0 reads a v0.1.1 record without modification.

---

## [0.1.0] — Public release

This is the first public version of the AXIS Protocol specification. The spec has been through internal review and the reference implementation (AXIS Prime, operated by Kipple Labs at `registry.axisprime.ai`) has been running in production with the Offworld News autonomous publication since March 2026.

### Included in v0.1

- Agent Identity Record (AIR), Operator Identity Record (OIR)
- AXIS Identity Token (AIT) signed-JWT format (Ed25519 / EdDSA per RFC 8037)
- Delegation Credential (DC) with monotonic attenuation and root_operator invariant
- Trust Attestation (TA) basic structure
- Content Provenance Attestation (CPA) basic structure
- Registry Data Visibility Model (public / presentation / private tiers)
- Registry API contract (§6.1 through §6.12)
- Platform-side Access Control (`/.well-known/axis-access`)
- Verification procedure (six-step flow)
- Worked example: cross-operator agent hire
- Compliance cross-references: EU AI Act Article 12, HIPAA 45 CFR 164, W3C DID Core
- Compliance posture and GDPR considerations (§10.4)
- Security and Privacy considerations
- Conformance criteria
- IANA well-known URI reservations for `/.well-known/axis-access` (v0.1), `/.well-known/axis-registry` (v0.2), `/.well-known/axis-scopes` (v0.2)

### Explicitly deferred to v0.2

See SPEC.md §17 Roadmap for the full list. Key items:

- AXIS Skills Protocol (capability marketplace layer — pulled from v0.1 to keep the base spec focused on identity + authorization + revocation)
- `/.well-known/axis-scopes` endpoint and manifest format
- `/.well-known/axis-registry` registry self-identification manifest
- Formal `did:axis` method specification
- Trust Attestation `credentialSubject` schema standardization
- Verifiable Credential-compatible encoding for AITs and DCs
- Key rotation protocol
- Client SDK specifications
- Registrar Compliance Attestations (four-tier trust model)
- Operator slug tiering

### Breaking changes expected before v1.0

None planned today, but v0.x permits them. Anyone implementing against v0.1 should track this changelog and subscribe to spec-related issues on the repository.

---

## Change history leading up to 0.1.0

### 2026-04-19 — pre-release review pass

- Added §10.4 Compliance posture (AXIS as infrastructure, not AI system; GDPR for registrars; explicit non-goals).
- Added §17 Roadmap consolidating all v0.2 / v0.3 / v1.0 forward references that were previously scattered through the body.
- Promoted AXIS Skills Protocol from "not in spec" to explicit v0.2 roadmap item as the future capability marketplace layer.
- Clarified `allow_unverified` field in §7 from ambiguous to strict boolean.
- Updated Status banner from "Early draft" to "Public v0.1 release candidate."
- Expanded Appendix A with semantic versioning discipline note; moved full release schedule to the Notion release plan (out-of-repo operational artifact).

### 2026-04-18 — architecture clarifications (non-spec)

- Entity structure finalized: three-entity model (AXIS Protocol / AXIS Prime / Kipple Labs) documented in README and reference-implementation context.
- Reference implementation URL updated from `axis-registry.editor-9a4.workers.dev` to `registry.axisprime.ai`.

### 2026-04-13 — pre-v0.1 additions

- Registry Data Visibility Model — public / presentation / private tiered visibility for agent and operator records.
- Platform-side Access Control — `/.well-known/axis-access` endpoint for platforms to publish their accept-requirements.
- Operator verification tier clarification — lives on the Operator Identity Record, not in delegation credentials.
- Scope taxonomy guidance — open namespace in v0.1, structured taxonomy in v0.2.
- Full chain presentation — agents present the complete delegation bundle; receiving platform does not chase individual links.
- Platform retention recommendation — SHOULD retain the credential chain at time of action (HIPAA §164.312(b), EU AI Act Art. 12).

### 2026-03-29 — initial draft

- Three-layer architecture: Identity (AIR + AIT), Authorization (DC), Reputation (TA + CPA).
- Ed25519 keypair as cryptographic identity.
- Dual identifier format: `axis:operator:agent-name` and `did:axis:registry:agent-id`.
- API contract: `/register`, `/agents/:id`, `/verify`, `/revocation/:id`, `/operators/:id`, `/delegations`, `/delegations/:id`, `/delegations/:agent_id/chain`.
- Delegation Credentials with monotonic scope attenuation.
- `root_operator` field byte-for-byte identical across chain (prevents re-rooting).
- Cross-registry delegation via self-locating `registry_url` in AITs.
- Registrar model distinct from registry (consumer-facing vs backend).
