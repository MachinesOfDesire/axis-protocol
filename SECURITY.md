# Security Policy

## Reporting a vulnerability

If you believe you've found a security vulnerability in the AXIS protocol specification or in the AXIS Prime reference implementation, please report it privately.

**Do not** open a public GitHub issue for security vulnerabilities.

### How to report

Email: **`security@kipplelabs.com`**

Include:
- Description of the vulnerability
- Steps to reproduce (or, for spec-level issues, the attack scenario)
- Your assessment of the impact and affected parties
- Any suggested mitigations
- Your name and affiliation (if you'd like to be credited)

### What to expect

- **Acknowledgement within 3 business days.** We'll confirm we received your report and are investigating.
- **Triage within 7 business days.** We'll assess severity and likely response timeline.
- **Fix and coordinated disclosure timeline** proportional to severity. Critical vulnerabilities in the reference implementation are typically patched within 7 days. Spec-level issues often require coordinated disclosure across multiple implementations.
- **Public credit** if desired, after the fix is deployed and the fix is safe to discuss publicly. We maintain a credits section in `CHANGELOG.md` under each affected version.

## Scope

### In scope

- **AXIS protocol specification** — cryptographic weaknesses, logical flaws in delegation chains, authorization bypasses, registry enumeration attacks, identity impersonation vectors, replay or forgery attacks against AITs or DCs, privacy leaks in the Registry Data Visibility Model
- **AXIS Prime reference implementation** — vulnerabilities in the deployed registry at `registry.axisprime.ai` and related services (`signup.axisprime.ai`, `admin.axisprime.ai`, `demo.axisprime.ai`, `try.axisprime.ai`)
- **Supply chain** — compromise paths through build tooling, dependencies, or CI/CD that would affect protocol-level integrity

### Out of scope

- **Third-party AXIS-compliant registries** — report vulnerabilities in those to the respective operator. We will coordinate with the operator if a spec-level issue is involved.
- **Phishing / social engineering / physical attacks** against individuals
- **Denial of service against the reference registry** — we run on Cloudflare; resource-exhaustion DDoS reports are not in scope unless they reveal an infrastructure bug
- **Issues already publicly disclosed** or fixed in the current version
- **Best-practices audits** that do not identify specific vulnerabilities (e.g., "you should enable X feature") — these are welcome as GitHub issues rather than security reports

## Severity model

We use a CVSS-inspired informal severity scale:

- **Critical:** Allows impersonation of any agent without the key, forging of delegation credentials, registry takeover, or mass compromise
- **High:** Allows impersonation or privilege escalation under specific conditions, or exposure of private-layer data
- **Medium:** Information disclosure beyond what the Registry Data Visibility Model intends, or denial-of-service of a specific registrar
- **Low:** Minor implementation bugs, edge cases with negligible real-world impact

## Safe harbor

We will not pursue legal action against researchers acting in good faith under the following conditions:

- You make a good-faith effort to avoid privacy violations, destruction of data, or disruption of service
- You only interact with accounts you own or for which you have explicit permission from the account holder
- You do not exploit a vulnerability beyond what is necessary to confirm its existence
- You provide us a reasonable opportunity to respond before disclosing publicly (we aim for 90 days from acknowledgement, shorter for critical issues by mutual agreement)
- You do not publicly disclose the vulnerability until we have confirmed a fix is deployed or reached the end of the coordinated disclosure window

Researchers acting in good faith under these conditions will be credited in the `CHANGELOG.md` unless they request anonymity.

## Bug bounty

We do not currently offer a formal bug bounty program. This may change as the protocol reaches wider adoption. Reporters of critical or high-severity issues may receive discretionary thanks (swag, acknowledgement, introductions to standards-body communities) at our option.

## Cryptographic assumptions

AXIS assumes the following:

- **Ed25519** is used for all signatures (AITs, Delegation Credentials, Trust Attestations). Breaking Ed25519 or a specific deployment's implementation of it breaks the protocol's security assumptions at that deployment.
- **SHA-256** is used for key hashing (api key storage) and credential integrity. Second-preimage attacks against SHA-256 would require a protocol update.
- **TLS 1.3+** is used for transport between agents, registries, and verifiers. TLS compromise is a separate concern from AXIS but affects the protocol's privacy guarantees.
- **Time synchronization within ~5 minutes** across all parties for credential `iat`/`exp` validation. Clock skew beyond this window may cause legitimate credentials to fail verification.

Deployments that use non-Ed25519 keys, weakened signature suites, or bypass TLS are NOT AXIS-compliant and fall outside the protocol's security model.

## Contact

- Security issues: `security@kipplelabs.com`
- General inquiries: `hello@kipplelabs.com`

PGP/age keys for encrypted communication are available on request. We aim to publish these alongside this file once the keys are generated and rotated through a proper ceremony (planned before v1.0 launch).
