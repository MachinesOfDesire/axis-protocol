# AXIS — Agent Cross-system Identity Standard

**Version:** 0.1.1
**License:** Apache 2.0
**Reference implementation:** [AXIS Prime](https://axisprime.ai)
**Status:** Public release. Breaking changes possible before v1.0; see [CHANGELOG.md](./CHANGELOG.md) for the versioning policy.

---

## What is AXIS?

AXIS is an open protocol for autonomous AI agent identity, delegation, and authorization. It defines how agents prove who they are, who authorized them to act, and what they are allowed to do — in a way that any third party can verify without any prior relationship with the agent or its operator. Verification is cryptographic and self-locating: the agent carries a pointer to its registry, and any verifier can resolve, verify, and make a trust decision with no pre-shared infrastructure. Multiple independent registries can coexist and interoperate.

AXIS is designed for the cross-system case: an agent built and operated by one organization needs to be verified by a platform or service operated by a completely different organization, possibly using a different agent runtime, different identity infrastructure, and with no pre-existing relationship between any of the parties.

## The problem AXIS solves

AI agents today have no portable identity. They are identified by API keys, session tokens, or platform-specific credentials — all of which are scoped to a single system and meaningless outside it.

This breaks down the moment agents operate across organizational boundaries. When Agent A (from Org A) is authorized to take an action on Platform B (operated by Org B), Platform B has no standard way to answer:

- Is this really Agent A, and not something pretending to be Agent A?
- Who authorized Agent A to take this action?
- What is Agent A actually authorized to do, and what are the limits?
- Is that authorization still valid, or has it been revoked?
- If Agent A is acting on behalf of a sub-agent, is that sub-delegation legitimate?

AXIS answers all of these questions with cryptographic proof, without requiring Platform B to have any prior knowledge of Org A's systems.

## The three-layer architecture

AXIS defines three distinct credential types that together solve the cross-operator trust problem:

### Layer 1 — Identity (who is this agent?)

The **Agent Identity Record** and **AXIS Identity Token (AIT)**. Persistent, cryptographically verifiable identity anchored to a registry. Any party can verify any agent without prior knowledge of the operator's systems.

- Agent Identity Records are compatible with W3C DID Documents (`did:axis` method)
- AITs are signed JWTs that agents present as bearer credentials
- Dual ID format: human-readable (`axis:operator:agent-name`) and W3C DID (`did:axis:registry:agent-id`)

### Layer 2 — Authorization (what is this agent allowed to do?)

**Delegation Credentials**. Scoped, time-limited, revocable authority chains with monotonic attenuation — each level can only narrow the scope, never widen it. The `root_operator` field is byte-for-byte identical across every link in the chain, preventing re-rooting attacks.

### Layer 3 — Reputation (how has this agent performed?)

**Trust Attestations** and **Content Provenance Attestations**. Signed records of an agent's track record, skill verifications, and content production history. Advisory, not mandatory — but the basis for the agent labor market.

## Core concepts

**Agent** — An autonomous software system that takes actions on behalf of a human operator or another agent. An agent has a persistent identity that survives restarts, platform migrations, and key rotations.

**Operator** — A human or organization that controls one or more agents. The operator is the root of the trust chain. Every agent's authority ultimately traces back to an operator. The operator's verification tier (e.g. email-verified, domain-verified, KYB-individual, KYB-business) is a property of the Operator Identity Record stored in the registry, NOT a field in delegation credentials or agent identity records. Receiving platforms see the operator's verification tier by querying the registry when resolving the operator's identity. This separates the identity verification signal (how thoroughly the operator proved who they are) from the authorization signal (what the agent is allowed to do). Agents do not have an independent verification tier. Agent trust history is captured through Trust Attestations (Layer 3), not through registry-level verification.

**Registry** — Backend infrastructure that stores identity records and credential chains. A registry is defined by its API contract, not by any specific implementation. Multiple independent registries can coexist and interoperate.

**Registrar** — A consumer-facing service that handles agent registration, operator verification, pricing, and support on behalf of end users. Registrars write to a registry. Multiple registrars can write to the same registry. Agents registered through any compliant registrar are interoperable across the entire network.

**Agent Identity Record** — A persistent, cryptographically verifiable record of an agent's existence, public key, capabilities, and current status. Stored in a registry and resolvable by any party. Conforms to W3C DID Core.

**AXIS Identity Token (AIT)** — A signed JWT that an agent presents to prove its identity. Contains the agent's ID, operator, and registry URL. The verifier resolves the registry URL to retrieve the public key and verify the signature.

**Delegation Credential (DC)** — A signed document issued by an operator or agent (the delegator) to another agent (the delegate), specifying what actions the delegate is authorized to take, under what constraints, and for how long. Delegation can be chained but can only narrow authority — never widen it.

**Trust Attestation (TA)** — A signed statement from one party about another agent's trustworthiness, track record, or verified capabilities. Distinct from delegation — attestations say "I trust this agent" not "I authorize this agent to act."

**Content Provenance Attestation (CPA)** — A signed record proving specific content was produced by a specific agent under a specific delegation, reviewed by a specific party. Enables third-party verification of content governance chains.

**Verifier (Relying Party)** — Any party that receives a request from an agent and needs to verify the agent's identity and authorization. The verifier has no prior relationship with the agent or its operator.

## Quick start

If you want to integrate with AXIS as a verifier (platform accepting agent traffic):

```javascript
// Receive an AIT from the agent (typically in Authorization header)
const ait = request.headers.get('Authorization')?.replace(/^Bearer /, '');

// Verify with the registry
const res = await fetch(`https://registry.axisprime.ai/verify?token=${ait}`);
const { valid, agent_id, operator_id, verification_tier } = await res.json();

if (!valid) return new Response('Invalid agent credentials', { status: 401 });

// Apply your policy — e.g. minimum verification tier
if (verification_tier === 'email') {
  return new Response('Domain-verified operators only', { status: 403 });
}

// Pass through with agent context
```

For a full worked example of cross-operator verification, see [SPEC.md](./SPEC.md#worked-example-cross-operator-agent-hire).

## Repository structure

```
axis-protocol/
├── README.md                        # This file — overview + quick start
├── SPEC.md                          # Full protocol specification
├── CONTRIBUTING.md                  # How to contribute + CLA
├── CONTRIBUTORS.md                  # Signed CLA record
├── SECURITY.md                      # Security disclosure policy
├── IMPLEMENTATIONS.md               # Compatible registry implementations
├── ROADMAP.md                       # v0.2 and beyond
├── CHANGELOG.md                     # Version history
├── LICENSE                          # Apache 2.0
└── schemas/                         # JSON schemas (planned)
    ├── agent-identity.json
    ├── operator-identity.json
    ├── ait.json
    ├── delegation-credential.json
    ├── trust-attestation.json
    └── content-provenance.json
```

## Protocol, registry, and implementations

AXIS is an open protocol. This repository contains the specification and any party may implement a compliant registry. Multiple independent registries can coexist; agents registered through any compliant registrar are interoperable across the network.

A reference implementation exists (see below). Other implementations are welcome — see [IMPLEMENTATIONS.md](./IMPLEMENTATIONS.md) to register one.

## Governance

The AXIS Protocol specification is owned by Kipple Labs, Inc. and published under Apache 2.0. The protocol is currently maintained by Kipple Labs. At a future date when adoption justifies it, day-to-day maintenance and governance may be delegated to an independent nonprofit foundation, with Kipple Labs retaining ownership of the specification and trademarks. The open source licensing and Contributor License Agreement (see [CONTRIBUTING.md](./CONTRIBUTING.md)) were designed to make this transition legally and technically possible.

## Relationship to existing standards

- **W3C DIDs** — AXIS agent IDs follow W3C Decentralized Identifier syntax: `axis:operator:agent` can be expressed as `did:axis:registry:agent`. Formal `did:axis` method registration with W3C is planned for v0.2.
- **W3C Verifiable Credentials** — AXIS Delegation Credentials and Trust Attestations are designed to be expressible as W3C VCs. A VC-compatible encoding is planned for v0.2.
- **zcap-spec** — AXIS delegation semantics align with the W3C Authorization Capabilities specification. The attenuation rule and chain depth limits follow zcap invariants.
- **JWT / RFC 7519** — AXIS Identity Tokens are standard JWTs with AXIS-specific claims. Signed using EdDSA per RFC 8037.
- **MCP (Model Context Protocol)** — AXIS Skills (planned for v0.2 as the future capability marketplace layer) will build on MCP, making them model-agnostic and runtime-agnostic.

## Reference implementation

AXIS Prime implements AXIS v0.1 as a Cloudflare Workers + D1 application. It is currently in **closed beta**: the registry API, verification endpoint, and interactive demo are publicly readable, but self-service operator registration is gated while the billing surface and operator onboarding are finalized.

- **Registry API (read-only public):** `https://registry.axisprime.ai`
- **Interactive demo:** `https://try.axisprime.ai`
- **Operator signup:** `https://signup.axisprime.ai` (closed beta — contact for access)
- **Key algorithm:** Ed25519 (Cloudflare Workers crypto)
- **Storage:** Cloudflare D1 (SQLite at the edge)

The reference implementation source is not open source at this time. The protocol it implements is open; any party can build a compatible registry from the spec alone.

## Versioning

This is v0.1 — the first public release candidate. Breaking changes are possible within the 0.x series; v1.0 will freeze the stable contract under strict semantic versioning discipline.

See [CHANGELOG.md](./CHANGELOG.md) for version history and the versioning policy. The consolidated forward-looking roadmap lives in [SPEC.md §17](./SPEC.md#17-roadmap); [ROADMAP.md](./ROADMAP.md) holds the longer-form narrative.

## Contributing

Contributions are welcome. Please read [CONTRIBUTING.md](./CONTRIBUTING.md) and sign the CLA before submitting a pull request.

To report a security issue, see [SECURITY.md](./SECURITY.md).

## License

Licensed under the Apache License, Version 2.0. See [LICENSE](./LICENSE) for the full text.

© 2026 Kipple Labs, Inc. "AXIS," "AXIS Protocol," "AXIS Prime," "N7," and "Kipple Labs" are pending trademarks of Kipple Labs, Inc. Use of the names for compatible implementations is permitted; see [CONTRIBUTING.md](./CONTRIBUTING.md) for guidance.
