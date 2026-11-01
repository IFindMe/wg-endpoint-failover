# Security Policy

## Reporting a vulnerability

Report security issues privately to the maintainers via GitHub's
[private vulnerability reporting](https://docs.github.com/en/code-security/security-advisories/guidance-on-reporting-and-writing-information-about-vulnerabilities/privately-reporting-a-security-vulnerability)
on this repository.

Do **not** open a public issue, and do not include exploit details in a pull request.

Please include:

- affected component and version or commit,
- platform (Linux / Windows / Android),
- reproduction steps or a proof of concept,
- impact assessment: what an attacker gains,
- whether private key material, tunnel plaintext, or traffic metadata is exposed.

We aim to acknowledge within 5 working days and to publish an advisory once a fix is
available.

## Supported versions

The project has no released version yet. Once 0.1.0 ships, only the latest minor release
receives security fixes.

## What this project handles

Understanding the blast radius matters more than the policy text:

| Data | Handled? | Notes |
|---|---|---|
| WireGuard private keys | **No** | Never read, never stored, never logged |
| WireGuard preshared keys | **No** | Never read, never stored, never logged |
| Peer public keys | Yes | Not secret; used as the peer identifier |
| Endpoint addresses | Yes | Not secret, but treated as sensitive metadata in logs |
| Tunnel plaintext | **No** | The daemon never sees decrypted traffic |
| Companion config | Yes | Contains no secret material by design |
| Health-check payloads | Yes | Application-level probes only; no user data |

Because the daemon never holds key material, a full compromise of the daemon does not
directly yield the ability to decrypt recorded traffic or impersonate the peer. It does
yield the ability to redirect the tunnel to an attacker-chosen endpoint — which is a
denial of service and a traffic-redirection primitive, not a decryption primitive. This
distinction drives the privilege model in
[docs/architecture/security.md](docs/architecture/security.md).

## Known structural risks

Documented, accepted, and mitigated — see
[docs/architecture/failure-modes.md](docs/architecture/failure-modes.md):

- An attacker who can modify the companion configuration can redirect the tunnel. The
  configuration must be root-owned and not group-writable.
- Endpoint redirection is unauthenticated by design: WireGuard authenticates peers, not
  paths. Redirecting to an endpoint that does not hold the peer's private key results in a
  dead tunnel, not a compromised one — the handshake fails.
- DNS is an unauthenticated input. A poisoned DNS answer causes failover to an address that
  cannot complete a handshake; it does not expose plaintext.
