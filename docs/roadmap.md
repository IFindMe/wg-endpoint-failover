# Roadmap

A summary view. Detailed objectives, acceptance criteria, tasks and risks live in
[implementation-plan.md](implementation-plan.md); this document is the order, the gates, and
what is explicitly not being built.

## Status

Repository and architecture phase. **No engine is implemented.** Everything below the table is
a plan, not a claim.

## Milestones

| # | Milestone | Ships | Gate to start | Gate to finish |
|---|---|---|---|---|
| 0 | Repository, research, architecture | This documentation set | — | Every architectural claim has a source link; owner confirms the licence decision |
| 1 | Repository foundation | Go module, CI, invariant tests, branch protection | Milestone 0 | `go build`/`test`/`lint`/`govulncheck` green; invariant test fails on a deliberately broken fixture |
| 2 | Linux MVP | `wg-endpoint-failover` daemon + `wgefctl` + systemd unit + man pages | Milestone 1 | Idle tunnel produces no failover over 30 min; a failed primary switches within the documented bound; keys byte-identical across the switch |
| 3 | Automated failover tests | Network-namespace integration suite | Milestone 2 | Every scenario in M3's matrix passes; a known-bad variant makes it fail |
| 4 | Windows integration | Windows service, CLI, MSI | Milestone 3 + WireGuardNT spike result recorded | M2's scenarios pass on Windows; CLI works unelevated; uninstall restores the endpoint |
| 5 | Android integration | Patched `tunnel` library, engine in-process, APK | Milestone 3 + Android spike result recorded | Live tunnel switches without the tun fd being recreated; no spurious failover on Wi-Fi ↔ cellular |
| 6 | Packaging and release | `.deb`, MSI, signed APK, checksums, SBOM, provenance | Milestones 4 and 5 | Tag produces every artifact; installing on a clean machine works with no manual steps |
| 7 | Documentation and 1.0.0 | User and operations guides, reconciled docs | Milestone 6 | No `[unverified]` markers remain; documented defaults match the code |

## Why this order

1. **Linux first.** It has the best-supported control surface (`wgctrl-go` netlink), the best
   test harness (network namespaces, no hardware), and the lowest packaging cost. It is where
   the health model can be proven cheaply.
2. **The test harness before the other platforms.** The risky part of this project is the
   switching policy, not the platform adapters. Proving the policy once, on the cheapest
   platform, means Windows and Android inherit a correct policy instead of discovering their
   own.
3. **Windows before Android.** Windows needs a privileged service and an MSI — effortful, but
   the control surface already works through `wgctrl-go`
   ([platform §2](research/platform-capabilities.md#2-windows)). Android needs a patch to
   upstream code and a binding decision
   ([platform §3](research/platform-capabilities.md#3-android)), so it has the most unknowns
   and the highest chance of being reshaped by evidence.
4. **Packaging last.** Sign it once the behaviour is right.

## Risk profile across the roadmap

| Milestone | Principal risk | How it surfaces | Response |
|---|---|---|---|
| M2 | Detection latency worse than documented | Acceptance run measures it | Correct the documentation, not the measurement |
| M3 | Tests pass while the product is broken | Negative/mutation test fails | Fix the suite before trusting it |
| M4 | WireGuardNT rejects endpoint-only updates on an Up adapter | The spike fails | Milestone stops; design revisited with the measured evidence |
| M5 | `device.IpcSet` cannot change an endpoint live on Android | The spike fails | Document that Android requires a fork; do not claim support |
| M6 | Signing unavailable for Windows or Android | Release ships unsigned | Warn loudly; do not silently ship |
| M7 | Documentation drifts from behaviour | Default-table test fails | Fix in the same change |

The two spikes (M4 and M5) are deliberately scheduled *before* the implementation effort of
their milestones. Both answer a question whose answer could invalidate the milestone, and both
cost a day rather than a week.

## Non-goals

These are not "later". They are out of scope for this project.

| Not built | Why |
|---|---|
| Switching between servers with different public keys | Different product: it requires new identity, new keys and new `AllowedIPs`. See [existing-solutions §4](research/existing-solutions.md#4-privycs--adjacent-and-deliberately-out-of-scope) for the project that does this |
| NAT traversal, hole punching, STUN, signalling | Requires a broker or relay; excluded by requirement ([ADR-011](architecture/decisions.md#adr-011-no-active-endpoint-discovery)) |
| Automatic endpoint discovery | Candidates are operator-supplied |
| A WireGuard fork or protocol change | Excluded by requirement; every platform's official API suffices |
| Key management, key storage, or key rotation | Explicitly outside the trust model ([ADR-006](architecture/decisions.md#adr-006-no-key-material-enforced-structurally)) |
| Kill switch, routing, DNS redirection | Firewall policy; each platform has mechanisms |
| Tunnel lifecycle management | Composability with `wg-quick` and the Windows service ([ADR-009](architecture/decisions.md#adr-009-the-project-does-not-manage-tunnel-lifecycle)) |
| macOS, FreeBSD, OpenBSD | `wgctrl-go` makes them nearly free to add, but untested platforms must not be claimed. Revisit after 1.0.0 |

## Possible after 1.0.0

Not committed to, listed so they are not re-litigated:

- macOS / FreeBSD / OpenBSD adapters.
- L4 application-level health checks beyond the current TCP probe.
- A metrics endpoint (Prometheus) for operators who want to watch failover in a dashboard.
- Configuration migration tooling from `reresolve-dns`.
- Publishing to distribution repositories.
- A `wg-endpoint-failover up/down` lifecycle mode, if users ask for one.

## Versioning

[Semantic Versioning 2.0.0](release-plan.md#8-added-at-milestone-6--release-workflows).
Pre-1.0, breaking changes may land in a minor release and are always listed under `Changed`.
From `1.0.0`, the compatibility surface is: the companion configuration schema, the status
JSON, and the CLI.

Current version intent: `0.1.0` at the end of milestone 2 (Linux MVP), `0.2.0` at milestone 4,
`0.3.0` at milestone 5, `1.0.0` at milestone 7.
