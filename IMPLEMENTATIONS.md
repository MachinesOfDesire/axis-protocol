# AXIS-compatible implementations

This file lists registries that implement AXIS v0.3. For the reference
registry's per-mechanism shipped-vs-specified status, see
[docs/IMPLEMENTATION-STATUS.md](./docs/IMPLEMENTATION-STATUS.md).

To add an implementation: open a pull request adding your registry to the list below. See [CONTRIBUTING.md](./CONTRIBUTING.md#registering-a-compatible-implementation) for the format and basic conformance expectations.

---

## AXIS Prime

- **Operator:** Kipple Labs
- **Registry URL:** `https://registry.axisprime.ai`
- **Signup / registrar:** `https://app.axisprime.ai/signup`
- **Spec version:** v0.3 (see [implementation status](./docs/IMPLEMENTATION-STATUS.md))
- **Storage:** Cloudflare D1 (SQLite at the edge)
- **Signature algorithm:** Ed25519
- **Role:** Canonical reference registry, launch registrar
- **Contact:** `hello@axisprime.ai`

AXIS Prime is the launch implementation of AXIS, built and operated by Kipple Labs. Intended as the canonical reference and public-facing registry during v0.x, with migration to independent foundation operation planned alongside the v0.2+ governance transition.

---

## Your registry here

AXIS is an open protocol. Running a compliant registry does not require permission, payment, or accreditation. Register yours by opening a PR.
