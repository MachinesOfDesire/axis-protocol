# AXIS Protocol — Version Compatibility Guide

How implementers (agents, platforms, registries, SDKs) should handle protocol version mismatch. Normative wire details live in [SPEC.md §4.3.2](./SPEC.md) (version signaling) and §6.16 (`/.well-known/axis-versions`); release-by-release deltas live in [CHANGELOG.md](./CHANGELOG.md).

## Detect

Version information appears in three places, in decreasing order of availability today:

1. **Records and discovery documents (shipped).** AIRs, OIRs, and DCs carry `axis_version` (the data-model version of the record). Every `/.well-known/axis-*` document carries `axis_version` (the schema version of that document). Read these before assuming feature support — e.g. a platform whose `axis-access` advertises `"0.2"` will not emit v0.3 PoP fields.
2. **Registry supported-versions discovery (specified for v0.4, not yet shipped).** `GET /.well-known/axis-versions` returns `current` and `minimum_supported`. Until it ships, treat a 404 on that path as "infer from the discovery documents and the CHANGELOG."
3. **AIT `axis_version` claim (specified for v0.4, not yet shipped).** Today's tokens do not carry it; do NOT reject a token for lacking it.

## Handle

- **Unknown fields: ignore.** Every version to date is additive at the record level; a consumer MUST ignore fields it does not recognize (this is how a v0.2 verifier reads a v0.3 AIT). Never reject a record solely because it carries unrecognized optional fields.
- **Declared version above yours: proceed, warn.** Verify what you can under your own version's rules and surface a warning to your operator/logs that newer-version features were present but not evaluated. The one exception is an explicit requirement signal you cannot meet (e.g. a platform advertising `proof_of_possession: "required"` when you cannot produce DPoP proofs) — that is a hard stop; tell the operator what upgrade closes the gap.
- **Declared version below the counterparty's `minimum_supported`: expect rejection.** Registries and platforms MAY reject credentials and calls older than their advertised floor. Clients SHOULD check the floor up front (once §6.16 ships) rather than discovering it via failures.
- **Pre-1.0 semantics.** Within the 0.x series, treat each minor version as potentially breaking and track the CHANGELOG; from v1.0, only major-version differences are breaking (per the versioning policy in CHANGELOG.md).

## Warn and upgrade

Implementations SHOULD make version skew visible, not silent: log a structured warning the first time a newer-version artifact is seen, and surface an operator-facing hint naming the version gap and the newest spec release. SDKs SHOULD expose the protocol version they implement as a constant so downstream code can compare it against discovery documents.

## Deprecation policy — intent

A full deprecation policy document is future work. The intent it will codify: version retirement follows an **announce → deprecate → sunset** sequence — a change is announced in the CHANGELOG (and, once shipped, reflected in `/.well-known/axis-versions`) with no behavior change; the affected version is then marked `deprecated` for a published window during which it still works but emits warnings; only after that window does it move to `sunset` (rejected), and `minimum_supported` rises. No version goes from working to rejected without passing through a deprecation window.
