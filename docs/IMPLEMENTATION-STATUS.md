# AXIS v0.3 — reference implementation status

**As of:** 2026-07-18 · **Spec:** [SPEC.md](../SPEC.md) v0.3.1 · **Reference registry:** `registry.axisprime.ai`

The v0.3 specification is *additive* over v0.2 and deliberately ships ahead of its
own enforcement in places — several v0.3 mechanisms are **specified now, enforced
later**, and the SPEC flags each one inline. This page consolidates those flags into
one at-a-glance view so an implementer or evaluator can see, without reading the whole
spec, exactly what the **reference implementation (AXIS Prime)** does today versus what
is written down for a future release.

This is about *the reference implementation's* current behavior. The protocol is open:
another compliant registry MAY ship more or less of v0.3, and advertises what it speaks
through the `axis_version` fields on its discovery documents (§4.3.2). A conformant
implementation is judged against the SPEC's conformance criteria (§13/§14), not against
this table.

## Legend

- **Live** — implemented and serving in the reference registry / reference platform verifier today.
- **Partial** — some of the mechanism ships; the rest is specified but not yet serving.
- **Specified** — written in the SPEC, not yet implemented anywhere in the reference stack.

## v0.3 mechanisms

| Mechanism | SPEC | Status | Notes |
|---|---|---|---|
| Standard scope **vocabulary** (two-layer namespace, `x-<vendor>:` prefix) | §4.4.1, App. B | **Partial** | Classifier ships and the vocabulary is published; enforced at **verify/authorize** time by platforms. The registry **write** path still validates the v0.2 grammar only — a grammar-valid non-standard scope is accepted on `POST /delegations` today. Mint-time enforcement is tracked for a later release. |
| `/.well-known/axis-scopes` discovery manifest | §6.14 | **Live** | Registry publishes the standard vocabulary. |
| Registry-legitimacy CA-trust model — `/.well-known/axis-registry` self-manifest + `/.well-known/axis-directory` root directory, pinned-root-key step | §6.13, §6.15, §8 Step 1 | **Live** | Both documents served and signed (registry-signing key / Prime ROOT key); verifiers pin the root public key. |
| `access_policy` extensions — `blocked_operators` / `approved_operators` / `rate_limits` | §7 | **Live (schema)** | Fields published on `/.well-known/axis-access`. The registry-as-platform does not operator-gate its own access, so its lists are empty/absent; platforms that set them enforce them. |
| Chain-by-`dlg` verification (`GET /delegations/:id/chain`, verdict pinned to the credential) | §4.3, §6.9.1 | **Live** | AITs reference a DC by `dlg` id; the chain resolves and its effective scope authorizes. |
| Signed Delegation Credentials (JCS proof, chain proof verification) | §4.4, §6 | **Live (verify) / opt-in (enforce)** | The registry verifies DC proofs and walks the signed chain. Hard *rejection* of unsigned/invalid DCs is a deliberate operator opt-in during the v0.2→v0.3 migration, not yet default-on. |
| `did:axis` method (formal) | §10.3 | **Live** | Resolve surfaces the canonical `did`. |
| Inline challenge-on-refusal (401/403 body) | §7.1 | **Partial** | Platform-side verifiers emit the deny taxonomy: parameterized `WWW-Authenticate: AXIS` + structured `axis_error` (`no_credential`, `expired`, `agent_revoked`, `operator_revoked`, `insufficient_scope`+`missing_scopes`, `insufficient_verification`). The richer `required`/`current`/`remedy` blocks and the `challenge` nonce are **specified, not shipped**; the reference registry itself does not emit the inline-challenge body. |
| Sender-constrained AITs — `cnf.jkt` proof-of-possession + DPoP (+ RFC 9421 message-signature profile) + `proof_of_possession: "required"` advertisement | §4.3.1, §7, §8 | **Specified** | Design landed in the SPEC; enforcement not shipped. The registry's `/verify` and `/.well-known/axis-access` still present the v0.2 surface (no `proof_of_possession`, no DPoP enforcement). Presenting a v0.3 `cnf` AIT to today's verifier degrades to a bearer token. |
| Ephemeral within-runtime sub-agent delegates (`issued_to_public_key`, `task_spec_hash`) | §4.4.2, §8 Step 3 | **Specified** | Inline-only (never registry-served) by design; verification path exists in the reference platform but is not enabled by default. |
| AIT `axis_version` claim | §4.3.2 | **Specified (v0.4)** | No shipped issuer emits it; no shipped verifier checks it. Becomes a SHOULD-emit from v0.4. |
| `GET /.well-known/axis-versions` supported-versions discovery | §6.16 | **Specified (v0.4)** | Not implemented today. Until it ships, infer version support from the `axis_version` fields on the discovery documents and the CHANGELOG. |

## What this means for launch messaging

- **True today:** v0.3 identity, delegation, scoped authorization, standard-vocabulary
  discovery, registry-legitimacy verification, and chain verification are live on the
  reference registry. A platform can discover the protocol, register an operator + agent,
  mint and present AITs, and verify a signed delegation chain end-to-end.
- **Not yet true:** sender-constrained AITs (DPoP/PoP), ephemeral delegates, mint-time
  vocabulary enforcement, and the v0.4 version-negotiation surface. Do not describe these
  as available; describe them as specified in v0.3 / planned for v0.4.

When in doubt, the SPEC's inline status paragraphs (§4.3.1 "Status", §4.4.1 "Enforcement
status", §6.16 "Status", §7.1 "Status") are authoritative; this page is a convenience
index over them.
