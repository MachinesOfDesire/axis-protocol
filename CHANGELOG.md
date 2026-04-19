# Changelog

All notable changes to the AXIS protocol specification are documented in this
file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

Breaking changes are expected throughout the v0.x series. v1.0 will freeze the
stable contract.

## [Unreleased]

### Planned for v0.2
- Cross-system VC schema for Trust Attestations
- Trust Attestation aggregation, scoring, and evaluation
- Standard resolution path for cross-system delegation to foreign DIDs
- Scope taxonomy: standard vocabulary + namespace convention + `/.well-known/axis-scopes` discovery
- Registry discovery via `/.well-known/axis-registry`
- Registrar Compliance Attestations (RCA) — four-tier registrar trust model
- Operator slug tiering (domain-verified, registrar-namespaced, auto-generated)
- W3C VC-compatible encoding for Delegation Credentials
- Key rotation and recovery protocols
- `did:axis` DID method formal specification

## [0.1] — 2026-03-29

Initial draft. Published to GitHub pending external licensing review.

### Added
- Three-layer architecture: Identity (AIR + AIT), Authorization (DC), Reputation (TA + CPA)
- Ed25519 keypair as cryptographic identity
- Dual identifier format: `axis:operator:agent-name` and `did:axis:registry:agent-id`
- API contract: `/register`, `/agents/:id`, `/verify`, `/revocation/:id`, `/operators/:id`, `/delegations`, `/delegations/:id`, `/delegations/:agent_id/chain`
- Delegation Credentials with monotonic scope attenuation
- `root_operator` field byte-for-byte identical across chain (prevents re-rooting)
- Cross-registry delegation via self-locating `registry_url` in AITs
- Registrar model distinct from registry (consumer-facing vs backend)

### 2026-04-13 updates to v0.1
- Registry Data Visibility Model — public / presentation / private tiered visibility for agent and operator records
- Platform-side Access Control — `/.well-known/axis-access` endpoint for platforms to publish their accept-requirements
- Operator verification tier clarification — lives on the Operator Identity Record, not in delegation credentials
- Scope taxonomy guidance — open namespace in v0.1, structured taxonomy in v0.2
- Full chain presentation — agents present the complete delegation bundle; receiving platform does not chase individual links
- Platform retention recommendation — SHOULD retain the credential chain at time of action (HIPAA §164.312(b), EU AI Act Art. 12)

### 2026-04-18 updates (architecture / non-spec)
- Entity structure finalized: three-entity model (AXIS Protocol / AXIS Prime / Kipple Labs) documented in README and reference-implementation context. No changes to spec semantics; reference implementation URL updated from `axis-registry.editor-9a4.workers.dev` to `registry.axisprime.ai`.
