# Schemas

JSON Schema (Draft 2020-12) definitions for the six AXIS v0.1 record types. These are direct translations of [SPEC.md §4](../SPEC.md#4-data-model). Where the spec uses MUST/SHOULD for a constraint that can be expressed syntactically, the schema enforces it; where the constraint is semantic (attenuation rule, root-operator invariant, signature validity), the schema is silent and the spec's verification procedure (§8) applies.

## Files

| Schema | Record type | Spec section |
|---|---|---|
| [agent-identity.json](./agent-identity.json) | Agent Identity Record (AIR) | §4.1 |
| [operator-identity.json](./operator-identity.json) | Operator Identity Record (OIR) | §4.2 |
| [ait.json](./ait.json) | AXIS Identity Token (decoded header + payload) | §4.3 |
| [delegation-credential.json](./delegation-credential.json) | Delegation Credential (DC) | §4.4 |
| [trust-attestation.json](./trust-attestation.json) | Trust Attestation (TA) | §4.5 |
| [content-provenance.json](./content-provenance.json) | Content Provenance Attestation (CPA) | §4.6 |

## What schemas validate, and what they don't

**Schemas validate:**
- Presence of required fields
- Field types (string, integer, boolean, array, object)
- String patterns (AXIS ID shape, DC/TA/CPA ID prefixes, base64url)
- Enumerated values (`status`, `verification_tier`, `key_algorithm`)
- Value ranges (`capability_tier` 1–3)
- Forbidden extra fields (`additionalProperties: false` everywhere)

**Schemas do NOT validate:**
- Cryptographic signatures — these are verified out-of-band per [SPEC §8](../SPEC.md#8-verification-procedure).
- The attenuation rule (DC scope ⊆ parent scope) — chain walking is a verifier responsibility.
- The root-operator invariant (byte-for-byte equality across a chain) — same reason.
- Revocation status — a runtime check against `revocation_url`.
- Whether `iat <= now < exp` — a runtime check.
- Whether a signature in a `proof` object actually matches the signer's public key.

A record that passes schema validation is *syntactically* valid v0.1. Whether it is *accepted* in a verification flow is determined by the full verification procedure.

## Usage

```bash
# Example with ajv-cli (one of many validators)
npx ajv-cli validate -s schemas/agent-identity.json -d my-air.json
```

```python
# Example with Python jsonschema
from jsonschema import validate
import json

with open("schemas/delegation-credential.json") as f:
    schema = json.load(f)

with open("my-dc.json") as f:
    dc = json.load(f)

validate(instance=dc, schema=schema)
```

## Status

These schemas are **v0.1 release-candidate**. They follow SPEC.md at the same version. When the spec changes in v0.2, schemas will be versioned alongside (likely via a `v0.1/` and `v0.2/` subdirectory pattern).

## Contributing

Schema corrections are welcome — especially pattern tightening for fields where the spec is more restrictive than this file captures, or patterns that are over-restrictive and reject valid inputs. See [../CONTRIBUTING.md](../CONTRIBUTING.md).
