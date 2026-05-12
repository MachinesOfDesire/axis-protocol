# Changelog

All notable changes to the AXIS protocol specification are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

**Versioning policy.** Pre-v1.0 releases (all 0.x versions) may contain breaking changes between minor versions. v1.0 will freeze the stable contract under stricter discipline: patch releases additive only, minor releases may add optional fields or endpoints, major releases may break backward compatibility only with explicit governance approval. Track this file closely until v1.0.

## [0.2.0] — 2026-05-10

Minor release. Six wire-format additions and one clarification. Five additions are backward-compatible (additive optional fields, additive endpoint fields, additive DID alias). One — required `aud` claim on AITs — is a behavioral change for verifiers: v0.1 verifiers continue accepting tokens without `aud`; v0.2 verifiers MUST reject. Issuers SHOULD set `aud` on all new tokens immediately; the change closes the cross-platform AIT replay class.

### Added

- **§4.3 AIT — required `aud` claim.** Verifiers MUST reject AITs whose `aud` does not match their advertised `audience` (see §7). Closes cross-platform replay: a token minted for platform A cannot be presented to platform B. Backward-compatibility behavior documented inline.
- **§4.3 AIT — optional `dlg` claim.** Carries the `credential_id` of a Delegation Credential the agent is acting under. Verifiers resolve the chain via `GET /delegations/:id` and use the chain's effective scope for authorization. Forward-compatible by design: `dlg` is a reference to a record, not a copy of one, so future delegation envelope evolution (multi-signature, etc.) does not require changing the AIT claim shape.
- **§4.4 DC — formal scope grammar.** Scopes are colon-separated `[a-zA-Z0-9_-]+` segments; wildcard `*` matches exactly one segment; verifiers intersect across the chain per the same rules. Vocabulary remains platform-defined (deferred to v0.3).
- **§4.1 AIR — operator-namespaced DIDs.** v0.2 canonical DID shape is `did:axis:{registry}:{operator-slug}:{agent-slug}`. v0.1 forms remain resolvable; registries SHOULD return both forms during the transition window. Closes the global-DID-namespace squatting class. Backfill is mechanical (a single SQL UPDATE on existing records).
- **§6.1 POST /register — `proof.proofType` field.** New value `"jcs-eddsa-2026"` selects RFC 8785 JCS canonicalization for the proof body. Closes the v0.1 nested-key canonicalization fragility (the legacy `JSON.stringify(body, Object.keys(body).sort())` filtered keys at every nesting level rather than recursively sorting). Legacy form remains accepted for v0.1 registrants and is removed in v1.0. Registries SHOULD verify JCS first and fall back to legacy.
- **§7 `/.well-known/axis-access` — required `audience` field.** Each platform that consumes AITs MUST advertise its stable audience identifier so issuers can populate `aud` correctly.

### Changed

- **§17 Roadmap.** v0.2 list moved from "in design" to "shipped." v0.3 list expanded to receive the deferrals (formal `did:axis` method spec, W3C VC encoding, scope vocabulary, key rotation, algorithm negotiation, Evidence Record Types, and others). Removal of legacy proof canonicalization and v0.1 DID-form deprecation are tracked v0.3 → v1.0 items.
- **§7 access policy `v0.2 candidates` → `v0.3 candidates`.** `blocked_operators`, `approved_operators`, `rate_limits` deferred to v0.3.

### Migration notes (0.1.x → 0.2.0)

| Concern | Action |
|---|---|
| AIT issuers | Set `aud` on every new token to match the target platform's advertised `audience`. Reference SDK does this when the caller passes `aud` in the claims object. |
| AIT verifiers (v0.2-conformant) | Read `audience` from `/.well-known/axis-access`; reject tokens whose `aud` does not match. Reference SDK 0.2.2 supports this via the `audience` opt on `verifyAITLocally`. |
| Registries | Add `audience` to the platform's `/.well-known/axis-access` response. Begin returning the v0.2 DID form on new registrations; backfill v0.2 forms for existing rows (single SQL UPDATE; no operator action required). Accept both proof types on `/register`, prefer JCS. |
| SDKs | Implement JCS canonicalization. The reference JS SDK constraint is hand-rolled, zero-dependency. |
| Reference apps consuming AITs | Verify `aud`. Configure the platform's `audience` identifier (recommended convention: `<service>.<domain>`). |

### Compliance window

Registries MUST accept v0.2 DID resolution within 60 days of v0.2 publication. v0.1 DID forms remain resolvable indefinitely; the v0.3 spec will narrow them to resolve-only.

### Not changed

No changes to: AIT signing algorithm (still Ed25519 only), JWT envelope shape (still header.payload.signature), agent record structure (other than `did` shape and the new claims), delegation envelope, revocation mechanism, endpoint paths.

A v0.1.1-compliant consumer reads a v0.2 record without modification — they will not parse the new `dlg` or `aud` claims, will not enforce `aud`, and will see the v0.1 DID form returned alongside the v0.2 form.

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
