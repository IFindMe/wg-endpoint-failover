# Contributing

This project is in the **planning phase**. The most useful contributions right now are
corrections to the research and challenges to the architecture, not code.

## Before you start

Read, in order:

1. [docs/architecture/overview.md](docs/architecture/overview.md)
2. [docs/architecture/decisions.md](docs/architecture/decisions.md)
3. [docs/architecture/health-monitoring.md](docs/architecture/health-monitoring.md)

If your proposal contradicts an ADR, say so explicitly and provide evidence. "It is
simpler" is not evidence.

## Non-negotiable invariants

Any change that violates one of these will be rejected regardless of merit elsewhere:

- No code path writes `private_key`, `preshared_key`, `allowed_ips`, `listen_port` or
  interface addresses.
- No private key or preshared key is read, stored, transmitted or logged.
- No fork of, or patch to, the WireGuard protocol implementation's cryptography.
- The user's WireGuard configuration file is never modified.
- No new dependency is added to the trusted computing base without an ADR.

## Development setup

Toolchains are pinned per platform. As of the planning phase:

| Tool | Version | Notes |
|---|---|---|
| Go | 1.24 or newer | Linux and Windows engine, and the shared core |
| `wireguard-tools` | 1.0.20210914+ | `wg(8)` used in integration tests |
| Linux kernel | 5.6+ | In-tree WireGuard; netlink generic API |
| JDK | 17 | Android build |
| Android SDK | API 26+ | `VpnService` minimum used by wireguard-android |
| `gh` | 2.x | Release and repository automation |

Install `golangci-lint` and `markdownlint-cli2` for local checks.

## Commit messages

[Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/) is enforced on
pull request titles and required by convention for commits.

```text
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

Allowed types: `feat`, `fix`, `docs`, `test`, `build`, `ci`, `refactor`, `perf`, `chore`,
`revert`.

Scopes: `linux`, `windows`, `android`, `engine`, `config`, `health`, `dns`, `cli`,
`daemon`, `docs`, `ci`, `deps`.

Breaking changes use `!` after the type/scope and a `BREAKING CHANGE:` footer.

## Branching

- `main` is protected and always releasable.
- Work happens on `feat/<slug>`, `fix/<slug>`, `docs/<slug>`.
- Squash-merge only. The squashed subject must be a valid Conventional Commit.

## Pull requests

Every pull request must:

- fill in the template,
- state which milestone and which task it advances,
- include evidence for behavioural claims (command output, test output, packet capture),
- include tests for behaviour changes,
- update the affected documentation in the same pull request.

Reviewers check for scope expansion. A pull request that does more than its stated task
will be asked to split.

## Testing expectations

- Unit tests for pure logic (state machine, DNS cache, config validation) must be
  deterministic and must not require privileges.
- Linux integration tests use network namespaces and virtual WireGuard interfaces and are
  expected to run in CI on the standard GitHub-hosted runner.
- Tests must assert that peer public keys, preshared keys, `AllowedIPs` and interface
  addresses are byte-identical before and after every endpoint switch. This assertion is
  the core safety property of the project and must never be weakened.

## Documentation

- Markdown is linted with `markdownlint-cli2`.
- Diagrams are Mermaid sources under `design/diagrams/`, embedded by reference.
- Every non-obvious technical claim in a research document needs a source link.
- Do not add a document that restates another document; link to it instead.

## Security

Do not open a public issue for a vulnerability. See [SECURITY.md](SECURITY.md).

## Licensing

Contributions are accepted under the Apache License 2.0. By submitting a pull request you
agree that your contribution is licensed under those terms. Add the following SPDX header
to new source files:

```text
// SPDX-License-Identifier: Apache-2.0
// Copyright <year> <holder>
```

Do not copy code from GPL-2.0 sources (for example `wireguard-tools`) into this repository.
Behaviour may be reimplemented from documentation; code may not be translated.
