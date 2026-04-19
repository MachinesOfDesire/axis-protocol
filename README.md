# AXIS — Agent Cross-system Identity Standard

**Version:** 0.1 (Draft)
**Author:** Josh Ashcroft / Kipple Labs
**License:** Apache 2.0
**Reference implementation:** [AXIS Prime](https://axisprime.ai)
**Status:** Early draft. Breaking changes expected before v1.0.

---

## What is AXIS?

AXIS is an open protocol for autonomous AI agent identity, delegation, and authorization. It defines how agents prove who they are, who authorized them to act, and what they are allowed to do — in a way that any third party can verify without relying on a shared registry or prior relationship.

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

## The three surfaces

AXIS as a whole has three distinct surfaces, each with a different role:

| Surface | Role | Who runs it |
|---|---|---|
| **AXIS Protocol** (this repo) | The open spec | Kipple Labs, transitioning to an independent foundation |
| **AXIS Prime** ([axisprime.ai](https://axisprime.ai)) | Canonical reference registry | Operated commercially by Kipple Labs |
| **Kipple Labs** ([kipplelabs.com](https://kipplelabs.com)) | Commercial umbrella — compliance products, audit tooling, certification, skills catalog | Kipple Labs |

The protocol is open. Anyone can implement AXIS, run their own registry, and their agents will interoperate with the network. AXIS Prime is one canonical implementation; Kipple Labs sells products built on top of it. These are deliberately separate layers.

## Governance

Once the AXIS Prime registry is operationally self-sustaining, governance of the protocol will transfer to an independent nonprofit foundation. The open source licensing and Contributor License Agreement (see `CONTRIBUTING.md`) were designed from day one to make this transition legally and technically possible.

Post-transfer, Kipple Labs becomes a third-party registrar among potentially many — structurally equivalent, competing on product quality (compliance products, audit dashboards, hosted offerings) rather than protocol ownership.

## Compliance

AXIS provides the cryptographic primitives that regulated industries increasingly need for AI agent governance. For detailed mappings to specific frameworks, see:

- **EU AI Act Article 12** (record-keeping, traceability): [kipplelabs.com/compliance/eu-ai-act](https://kipplelabs.com/compliance/eu-ai-act)
- **HIPAA 45 CFR 164.312** (audit controls, access management, BAA enforcement): [kipplelabs.com/compliance/hipaa](https://kipplelabs.com/compliance/hipaa)
- **SOC 2 CC6/CC7** (planned)

These are published by Kipple Labs as the compliance product surface — the protocol itself is neutral infrastructure, the compliance framing is a product layer on top.

## Relationship to existing standards

- **W3C DIDs** — AXIS agent IDs are compatible with W3C Decentralized Identifiers. `axis:operator:agent` can be expressed as `did:axis:registry:agent`. The `did:axis` method is specified in `SPEC.md`.
- **W3C Verifiable Credentials** — AXIS Delegation Credentials and Trust Attestations are designed to be expressible as W3C VCs. A VC-compatible encoding is planned for v0.2.
- **zcap-spec** — AXIS delegation semantics align with the W3C Authorization Capabilities specification. The attenuation rule and chain depth limits follow zcap invariants.
- **JWT / RFC 7519** — AXIS Identity Tokens are standard JWTs with AXIS-specific claims. Signed using EdDSA per RFC 8037.
- **MCP (Model Context Protocol)** — AXIS Skills (planned) are built on MCP, making them model-agnostic and runtime-agnostic.

## Reference implementation

AXIS Prime implements AXIS v0.1 as a Cloudflare Workers + D1 application.

- **Registry API:** `https://registry.axisprime.ai`
- **Signup:** `https://signup.axisprime.ai`
- **Interactive demo:** `https://try.axisprime.ai`
- **Key algorithm:** Ed25519 (Cloudflare Workers crypto)
- **Storage:** Cloudflare D1 (SQLite at the edge)

The reference implementation source is not open source at this time (private repo, operational security considerations for the live registry). The protocol it implements is open; any party can build a compatible registry from the spec alone.

## Versioning

This is v0.1. It is an early draft. Breaking changes are expected before v1.0.

See [CHANGELOG.md](./CHANGELOG.md) for version history and [ROADMAP.md](./ROADMAP.md) for planned v0.2 features.

## Contributing

Contributions are welcome. Please read [CONTRIBUTING.md](./CONTRIBUTING.md) and sign the CLA before submitting a pull request.

To report a security issue, see [SECURITY.md](./SECURITY.md).

## License

Licensed under the Apache License, Version 2.0. See [LICENSE](./LICENSE) for the full text.

© 2026 Kipple Labs / Josh Ashcroft. "AXIS" and "AXIS Prime" are trademarks of Kipple Labs. Use of the trademarks for compatible implementations is permitted; see [CONTRIBUTING.md](./CONTRIBUTING.md) for guidance.
