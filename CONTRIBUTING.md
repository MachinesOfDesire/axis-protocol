# Contributing to AXIS

Thank you for your interest in contributing to the AXIS protocol.

## Before you contribute

Contributions are welcome — issues, pull requests, proposed spec changes, example implementations, conformance tests, schema additions.

We ask contributors to sign a Contributor License Agreement (CLA) before submitting a pull request. The CLA keeps the project legally clean and lets us transition stewardship to an independent foundation in the future without having to re-negotiate rights with every past contributor.

## Contributor License Agreement (CLA)

By submitting a pull request or other contribution to this repository, you agree to the following terms:

1. **You grant us a license.** You grant the AXIS project maintainer (currently Josh Ashcroft) and any future foundation or successor organization that inherits stewardship of the AXIS protocol a perpetual, worldwide, non-exclusive, royalty-free, irrevocable license to use, reproduce, modify, distribute, and sublicense your contribution as part of the AXIS protocol, under the Apache 2.0 license or any future license the project adopts.

2. **You retain your copyright.** Signing this CLA does not transfer ownership of your contribution. You keep your copyright. We just need the right to use it.

3. **Your contribution is your own work.** You represent that you have the right to submit the contribution — it is your original work, or you have permission from the original author, and it does not infringe any third party's rights.

4. **No warranty.** You provide your contribution as-is, without warranties of any kind.

5. **Relicensing optionality.** The license granted above includes the right to relicense your contribution under a different open source license if the project's governance determines relicensing is appropriate (for example, transitioning to an OSI-approved license better suited to the eventual foundation's stewardship model).

To indicate your agreement, add your name to the `CONTRIBUTORS.md` file as part of your pull request, with the following format:

```
Your Name <your@email.com> — agreed to CLA on YYYY-MM-DD
```

If you are contributing on behalf of an organization, contact us at `contrib@axisprime.ai` before submitting so we can arrange an entity-level CLA if needed.

## How to contribute

### Proposing a spec change

1. **Open an issue** describing the problem you want to solve and your proposed approach. Spec changes benefit from design discussion before implementation.
2. **Wait for discussion** before writing a PR. This avoids wasted effort on approaches that won't be accepted, and it lets us flag whether something should be a v0.1 clarification, a v0.2 addition, or out of scope.
3. **Submit a PR** that references the issue. Include:
   - Change to `SPEC.md` (the canonical spec document)
   - Update to `CHANGELOG.md` under `[Unreleased]`
   - Test cases or examples demonstrating the change
   - Rationale section in the PR description

### Registering a compatible implementation

If you've built an AXIS-compliant registry, open a PR adding your registry to `IMPLEMENTATIONS.md`. Include:

- Registry URL
- Operator of the registry (company or individual)
- Storage backend (D1, Postgres, blockchain, etc.)
- Compliance statement (which v0.x version you implement and any known deviations)
- Contact for registry operations

We'll do a basic conformance check and merge the PR. The directory of compliant registries is public and helps verifiers decide which registries to trust.

### Reporting a bug or inconsistency in the spec

Open an issue with:
- Clear description of the problem
- Exact citation in `SPEC.md` (section and paragraph)
- Where possible, a proposed correction

### Reporting a security vulnerability

Do NOT open a public issue for security vulnerabilities. See `SECURITY.md` for our security disclosure process.

## What we're looking for

- **Clarifications** that make the spec easier to implement correctly
- **Real-world use cases** that expose gaps in the current spec (especially cross-industry or regulated-industry scenarios)
- **Compatible implementation registrations** in `IMPLEMENTATIONS.md`
- **Corrections** to errors, ambiguities, or inconsistencies
- **Conformance test cases** — signed examples of valid and invalid records, credentials, and verification flows
- **JSON Schema additions** under `schemas/` for the core record types
- **Industry-specific profiles** (healthcare, finance, government) that extend but do not break the core

## What we're not looking for (yet)

- **Breaking changes to the v0.1 data model mid-version.** Between versions (v0.1 → v0.2) breaking changes are expected; within a version we hold the contract stable so implementers can ship against it. Significant data-model changes should target v0.2 — see [ROADMAP.md](./ROADMAP.md).
- **New credential types without a real-world use case.** Describe the use case first; the credential type comes after.
- **Infrastructure or tooling contributions.** The reference implementation (AXIS Prime) lives in a separate (currently private) repository. This repo is the spec only.
- **PRs that conflate spec changes with implementation details.** Separate them.

## Style and conventions

### Spec-document style

- **Use SHOULD, MUST, MAY, REQUIRED, OPTIONAL** per [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119) and [RFC 8174](https://datatracker.ietf.org/doc/html/rfc8174). Capitalize when used normatively.
- **Examples use `axis:operator:agent-name` format.** Pick illustrative operator names that are clearly fictional (`widget-corp`, `acme`, `some-rando`).
- **JSON examples are formatted with 2-space indentation** and trailing commas are prohibited (strict JSON).
- **Cryptographic examples use Ed25519** unless demonstrating a specific alternative algorithm case.

### Commit messages

Use conventional commits:

```
spec: add scope taxonomy discovery endpoint
schema: tighten delegation credential validation rules
docs: fix example in worked-example section
chore: update CONTRIBUTORS list
```

Prefix with `spec:` for normative spec changes, `schema:` for schema file changes, `docs:` for README/CHANGELOG/ROADMAP, `chore:` for housekeeping.

### PR size

Keep PRs focused. One conceptual change per PR. If you're also fixing typos, put the typos in a separate PR.

## Review process

PRs are reviewed by the current maintainer (Josh Ashcroft) during the pre-foundation period. Review SLA is best-effort; there are no guaranteed response times during v0.x.

Once a foundation is formed, review will transition to the foundation's governance structure (expected v0.2 or v0.3 timeframe).

## Contact

- **General questions:** open a GitHub issue
- **CLA / organization-level contributions:** `contrib@axisprime.ai`
- **Security issues:** see `SECURITY.md`

## License

By contributing, you agree that your contributions will be licensed under the Apache License, Version 2.0. See `LICENSE`.
