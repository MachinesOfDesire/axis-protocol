# Changelog

All notable changes to the AXIS protocol specification are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

**Versioning policy.** Pre-v1.0 releases (all 0.x versions) may contain breaking changes between minor versions. v1.0 will freeze the stable contract under stricter discipline: patch releases additive only, minor releases may add optional fields or endpoints, major releases may break backward compatibility only with explicit governance approval. Track this file closely until v1.0.

## [Unreleased]

### Corrected (spec text aligned to shipped behavior)

- **§7.1 — "partially shipped" status notes updated: the shipped deny taxonomy now covers the full enum.** Production platform-side verifiers previously emitted only the `no_credential` / `insufficient_scope` skeleton alongside implementation-specific codes (`missing_token`, `tier_too_low`, `verification_failed`, …); they have consolidated onto this section's taxonomy. Shipped behavior now: every `401` carries the parameterized `WWW-Authenticate: AXIS` challenge; `axis_error` is emitted wherever the enum names the condition (`no_credential`, `expired`, `agent_revoked`, `operator_revoked`, `insufficient_scope` + `missing_scopes`, `insufficient_verification`); a granular platform `code` rides alongside for diagnostic detail. The layering note now names three distinguishable layers in shipped bodies (protocol scope, protocol verification, platform policy) and drops `tier_too_low` from the policy-code examples — verification-tier denials are the enum's `insufficient_verification`, not a platform code. The `required`/`current`/`remedy` blocks and `challenge` nonce remain specified-not-shipped. No change to any normative requirement — these are status-note corrections.

## [0.3.1] — 2026-07-07

Patch release: a spec-to-shipped reconciliation pass plus version-discipline scaffolding. No new protocol mechanisms ship in this release; every correction below aligns the spec text with what the reference implementations (registry, protocol SDK, platform adapters) actually do on the wire today, and every genuinely new item is explicitly marked **specified, not yet shipped**.

### Corrected (spec text aligned to shipped behavior)

- **§4.4 / §8 Step 3 — DC proof canonicalization made normative.** The bytes signed for a Delegation Credential proof are now explicitly the RFC 8785 JCS canonicalization of the DC minus its `proof` field (what the reference SDK signs and the reference registry verifies). The proof envelope documents `proofType: "jcs-eddsa-2026"` (RECOMMENDED), the JCS-first/legacy-fallback rule when `proofType` is absent, and the MUST-reject-distinctly rule for unrecognized `proofType` values. Previously §4.4 referenced W3C Data Integrity without naming the canonicalization — the exact fragility (nested-key legacy canonicalization) that JCS was adopted to close.
- **§4.3 — AIT claims table reconciled with shipped tokens.** `iss`, `iat`, `exp`, `aud` are the required claims. `sub`, `operator_id`, `registry_url` reclassified OPTIONAL/advisory (shipped issuers do not emit them; the reference registry resolves the operator linkage authoritatively from the agent record and ignores payload claims for it). Added `jti` (RECOMMENDED; reference platforms require it and enforce single-use). `axis_version` in the AIT moved to §4.3.2 as a v0.4 addition, specified not shipped.
- **§4.3 — presentation channels documented.** `X-AXIS-Token` is the shipped, RECOMMENDED header for presenting an AIT to a platform; `Authorization: Bearer` and `?ait=` are the SDK-accepted alternates; `Authorization: DPoP` remains the specified-not-shipped v0.3 PoP form. Previously §8 only described the Authorization-header forms.
- **§6.4 `GET /verify` — response shape updated to shipped reality.** `operator_id` returns in canonical `axis:{operator}:operator` form; invalid tokens return a stable `code` (`invalid_signature`, `token_expired`, `agent_revoked`, `agent_suspended`); structural failures are 400 `invalid_request` / `missing_aud`; the endpoint requires a non-empty `aud`; and when the AIT carries `dlg` the response includes `delegation_resolved` / `delegation_valid` / `delegation_bound_to_agent` / `delegation_depth` / `effective_scope`. The previously-documented `verification_tier` field is not in the shipped response.
- **§6.9.1 (new) — `GET /delegations/:id/chain` documented, dual-mode.** The shipped chain endpoint accepts a delegation-credential id (`dc:`-prefixed or issuer-chosen — the form an AIT's `dlg` carries) and resolves upward via `parent_credential_id` with the verdict pinned to that credential (`delegation` block + `effective_scope`), or an agent identifier (newest-active-chain walk). Detection rules, 404 semantics, per-link checks, and the depth-16 cap documented. §4.3's `dlg` resolution pointer updated to this endpoint.
- **§7.1 — inline challenge status updated from "design only" to "partially shipped (platform-side)."** Production platform verifiers now emit the 401 + parameterized `WWW-Authenticate: AXIS` challenge (params `realm`, `audience`, `registry`, `discovery` — now specified) with `axis_error: "no_credential"`, and 403 `insufficient_scope` denials naming `missing_scopes` (new OPTIONAL body field). The `required`/`current`/`remedy` blocks and `challenge` nonce remain specified-not-shipped. Added the shipped two-layer deny distinction: protocol-layer scope denials vs platform-policy denials.
- **§4.4.1 — vocabulary enforcement status.** The reference registry ships the two-layer scope classifier and the `/.well-known/axis-scopes` manifest, but its delegation write path validates grammar only; vocabulary enforcement happens platform-side at verify time. Stated honestly, with the write-path tightening tracked for v0.4.
- **§9 — worked-example scopes fixed.** `article:*` scopes violated §4.4.1's own closed-namespace rule (unprefixed non-standard); replaced with standard vocabulary (`content:read` / `content:create` / `content:update` / `content:publish`).
- **§11.3 — shipped bearer-mode replay hardening documented.** `aud` match + platform max-TTL ceiling + single-use `jti` (409 `ait_replay` on re-presentation at the reference commenting platform).
- **§7 — `actions` manifest documented** (OPTIONAL, shipped by the reference commenting platform); `platform_id` noted OPTIONAL (`audience` is the normative identifier).
- **schemas/ait.json, schemas/delegation-credential.json — brought up to date.** The AIT schema no longer requires claims shipped tokens lack, accepts `aud`/`jti`/`dlg`/`cnf`/`axis_version`, and permits additional claims. The DC schema accepts the shipped `proofType` field (previously rejected by `additionalProperties: false`), makes `proof.created` optional (the SDK does not emit it), adds the v0.3 ephemeral fields (`issued_to` sub-form, `issued_to_public_key`, `task_spec_hash`), and drops the `axis_version` const `"0.1"`.

### Added

- **§4.3.2 — protocol version signaling.** Where versions appear on the wire today (records, discovery documents) and the v0.4 additions (AIT `axis_version` claim; supported-versions discovery), each marked shipped vs specified-not-shipped, with the MUST-NOT-reject-on-absence rule for the transition.
- **§6.16 — `GET /.well-known/axis-versions` (specified, not yet shipped).** Registry supported-versions discovery: `current`, `minimum_supported`, per-version status (`current` / `supported` / `deprecated` / `sunset`). Added to §14 IANA reservations.
- **Appendix B — reserved own-content comment scopes (specified, not yet enforced).** `content:comment:read`, `content:comment:edit-own`, `content:comment:delete-own`: the holder may read comment threads and edit/retract only comments it authored (platform matches stored authoring `agent_id`). Appendix B now also states which standard scopes are enforced in production today (`content:comment`; `data:read`/`data:write`/`data:delete`) — and reiterates that the commenting scope is `content:comment`, never `comments:write`.
- **COMPATIBILITY.md.** Implementer guidance for version mismatch (detect / handle / warn-and-upgrade) and the one-paragraph intent of the future announce → deprecate → sunset deprecation policy.
- **README.md** version/staleness fixes (was still describing v0.2-era status).

### Not changed

No wire-format changes. Anything valid against v0.3.0 remains valid; the corrections change the spec's description of behavior, not the behavior.

## [0.3.0] — 2026-06-14

Minor release. Additive over v0.2. v0.3 introduces sender-constrained AITs, ephemeral within-runtime sub-agent delegates, a standard scope vocabulary over the v0.2 grammar, a CA-trust registry-legitimacy model, the scope-discovery manifest, an optional inline challenge-on-refusal response, and the formal `did:axis` method. Every v0.3 mechanism degrades gracefully against v0.2: a v0.3 AIT presented to a v0.2 verifier is treated as a bearer token; v0.2 DCs and AITs remain valid. The single behavioral change is that a v0.3 verifier advertising `proof_of_possession: "required"` MUST reject a v0.2 (bearer-only) AIT.

### Added

- **§4.3.1 AIT — sender-constrained AITs (proof of possession).** New `cnf.jkt` confirmation claim (RFC 7638 JWK thumbprint of the agent's existing key) and a per-request DPoP proof header (RFC 9449), bound to the AIT via `ath`. Closes the same-platform leaked-AIT replay class that the v0.2 `aud` claim did not cover. Includes a `jti × jkt` replay cache (fail-closed), a 60-second default freshness window (120-second ceiling), permitted-not-required DPoP server-nonce mode, and an OPTIONAL RFC 9421 HTTP Message Signature profile (canonical AXIS component set, RFC 9530 Content-Digest) for high-stakes scopes. Deployment-configurable opt-out via `proof_of_possession`.
- **§4.4.1 / Appendix B — standard scope vocabulary.** A two-layer namespace over the v0.2 grammar: standard scopes (closed enum across reserved domains `content`, `social`, `commerce`, `data`, `comms`, `account`, `scheduling`, `compute`) carry no prefix; custom scopes MUST be prefixed `x-<vendor>:`. Domain wildcards (e.g. `content:*`) are NOT grantable standard scopes — the single-segment `*` stays a matching/attenuation grammar feature only — and unprefixed non-standard scopes are invalid. Seeded W3C-vocabulary-first (ActivityStreams 2.0, schema.org Actions, ODRL).
- **§4.4.2 DC — ephemeral sub-agent delegates.** New ephemeral `issued_to` form `axis:{operator}:{agent}:sub:{task-id}` with REQUIRED `issued_to_public_key` and OPTIONAL `task_spec_hash`. A task-bound delegate that is self-proving via the DC chain, never written to the registry, authenticated through the parent's AIT (`dlg` → ephemeral DC id). All v0.2 invariants (attenuation, root-operator, expiry, scope-subset) apply unchanged.
- **§6.13 `/.well-known/axis-registry` — registry self-manifest.** Replaces the v0.2 stub. Each registry publishes its signing-key set and an Ed25519 self-signature; rotation is expressed via overlapping key validity windows.
- **§6.14 `/.well-known/axis-scopes` — scope vocabulary discovery.** Publishes the scopes a platform recognizes, each flagged `standard` with a description; custom scopes MAY include `maps_to` a standard scope.
- **§6.15 `/.well-known/axis-directory` — root registrar directory.** AXIS Prime publishes a signed directory of certified registrars (monotonic `directory_version`, `expires_at`, per-registrar key fingerprints), signed by the pinned Prime root key.
- **§8 Step 1 — registry-legitimacy verification.** Before trusting any record from a declared `registry_url`, a v0.3 verifier verifies the registry manifest's self-signature, verifies the root directory against the pinned Prime root key (rejecting expired or rolled-back directories), and confirms the registry is listed `certified` with a matching key fingerprint. Fail-closed. Closes the registry-impersonation class (§11.6).
- **§8 Step 1.8 / Step 4.5 — verification additions.** DPoP proof-of-possession verification; ephemeral-delegate chain-walk branch; message-signature verification for scopes a platform marks `requires_message_signature`.
- **§7 — access-policy extensions.** `proof_of_possession`, `requires_message_signature`, and the v0.3 `blocked_operators` / `approved_operators` / `rate_limits` fields.
- **§7.1 — inline challenge-on-refusal (OPTIONAL).** Structured 401/403 body (`axis_error`, `message`, `action`, `required`/`current`, `remedy`, `audience`, optional `challenge` nonce) so gated endpoints can tell an agent exactly what was required and how to obtain it. Strictly an optimization over the pull model; degrades gracefully.
- **§10.3 — formal `did:axis` method.** Method name, identifier syntax (v0.2 canonical + v0.1 legacy), registry-anchored create/update/deactivate, resolution gated on the registry-legitimacy chain, DID Document derivation, and security/privacy considerations.

### Changed

- **§4.3 / §11.3 Replay attacks.** Documented that v0.3 PoP closes the same-platform replay class the v0.2 `aud` claim left open.
- **§11.6 Registry impersonation.** Rewritten: v0.3 closes it with the CA-trust legitimacy model and pinned root key (was an open v0.1/v0.2 risk with allowlist-only mitigation).
- **§11.1 Key management.** Registry signing-key rotation is specified (manifest `keys` windows); agent/operator key rotation remains a planned v0.3 deliverable.
- **§13 Conformance.** Added v0.3 registry criteria (legitimacy manifest, scope namespace, ephemeral acceptance) and the v0.3 verifier criteria (legitimacy check, DPoP enforcement).
- **§14 IANA.** `/.well-known/axis-scopes` and `/.well-known/axis-registry` reclassified to v0.3; `/.well-known/axis-directory` added.
- **§17 Roadmap.** v0.3 shipped items moved into "specified in this release." W3C VC encoding and agent/operator key rotation remain planned (designed as roadmap, not specified here). Algorithm agility/negotiation explicitly deferred (no current driver).

### Planned but not specified in this release

- **W3C VC-compatible encoding (deliverable #2).** Optional JSON-LD envelope for AIT/DC/TA alongside the canonical JWT form, with a lossless round-trip mapping and Ed25519 proof-suite alignment. Problem statement and open questions are tracked; the full design is not yet in this spec.
- **Agent/operator key rotation (deliverable #4).** Multi-key AIR with validity windows and/or signed rotation records, including reconciliation with the `cnf.jkt` PoP binding. Tracked; not yet specified.

### Resolved open vocabulary questions (per shipped registry behavior)

- Domain wildcards (`content:*`) are NOT grantable standard scopes — the standard vocabulary is a closed enum; the single-segment `*` stays a matching/attenuation grammar feature only.
- Custom scopes MUST carry the `x-<vendor>:` prefix; unprefixed non-standard scopes are invalid.

### Not changed

No changes to: AIT signing algorithm (still Ed25519 only), JWT envelope shape, AIR/DC record structure other than the additive fields above, the revocation mechanism, or v0.1/v0.2 endpoint paths. A v0.2-conformant consumer reads a v0.3 record without modification (it ignores `cnf`, `issued_to_public_key`, and the new well-known docs).

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
