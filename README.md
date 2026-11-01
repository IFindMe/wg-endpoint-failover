# wg-endpoint-failover

Automatic endpoint failover for a single WireGuard peer.

`wg-endpoint-failover` is a companion userspace project that gives one WireGuard peer
several alternative network endpoints and switches between them automatically when the
active one stops working. It does not modify the WireGuard protocol, does not fork
WireGuard, and does not change peer identity.

> **Status: planning.** This repository currently contains research, architecture and an
> implementation plan only. No failover engine has been implemented yet. Nothing in this
> repository should be read as a claim of working software.

---

## The problem

A WireGuard peer has exactly one `Endpoint`. If that endpoint becomes unreachable — the
host's address changes, a DNS record goes stale, a provider rotates an IP, a route
preference shifts — the tunnel stops passing traffic until an operator runs
`wg set <if> peer <key> endpoint <new>`. WireGuard is deliberately silent when idle, so
there is no automatic signal that anything is wrong.

WireGuard already supports *roaming*: if a peer's authenticated packets arrive from a new
source address, the endpoint is updated automatically
([wireguard.com — Built-in Roaming](https://www.wireguard.com/#built-in-roaming),
[`wg.8`](https://git.zx2c4.com/wireguard-tools/about/src/man/wg.8)). Roaming only helps
when the *peer* moves. It does not help when the peer stays put but the path to it breaks,
and it cannot choose between known alternatives.

## What this project adds

```text
WireGuard Interface  (one interface, unchanged)
    |
    +-- One Peer  (PublicKey, PresharedKey, AllowedIPs — never modified)
    |
    +-- Endpoint Candidates  (ordered by priority)
    |     +-- Primary:   endpoint-a.example.com:51820
    |     +-- Backup 1:  173.1.1.40:51820
    |     +-- Backup 2:  backup.example.com:51624
    |
    +-- Health Monitor  ->  Endpoint Manager  ->  active endpoint
                                                    |
                                              WireGuard tunnel
```

Only one field is ever written: the peer's `endpoint`. Keys, preshared keys, `AllowedIPs`,
the interface address and the interface itself are left untouched.

## Design invariants

1. **Peer identity is immutable.** No code path writes `private_key`, `preshared_key`,
   `allowed_ips`, `listen_port` or interface addresses.
2. **No private keys.** The daemon never reads, stores or logs a private key or preshared
   key. Endpoint candidates are associated with peers by *public* key.
3. **No protocol changes.** Failover is a configuration operation on an existing peer, not
   a cryptographic one.
4. **No forked WireGuard.** Upstream repositories are dependencies and integration
   surfaces, never forks.
5. **The original configuration file is never rewritten.** Companion state lives in its own
   file and the pre-existing endpoint is restored on clean shutdown.
6. **Failure is proven, not assumed.** An idle tunnel is not a broken tunnel
   ([Quick Start — "WireGuard tries to be as silent as possible when not being used"](https://www.wireguard.com/quickstart/)).

## Documentation

| Document | Contents |
|---|---|
| [docs/research/upstream-analysis.md](docs/research/upstream-analysis.md) | What upstream WireGuard actually supports, with source references |
| [docs/research/platform-capabilities.md](docs/research/platform-capabilities.md) | Linux / Windows / Android: how an endpoint can be changed on each |
| [docs/research/existing-solutions.md](docs/research/existing-solutions.md) | Prior art: `reresolve-dns`, wgrtc, PrivyCS |
| [docs/architecture/overview.md](docs/architecture/overview.md) | Component architecture and the recommended option |
| [docs/architecture/endpoint-manager.md](docs/architecture/endpoint-manager.md) | Endpoint selection, switching, adapter interface |
| [docs/architecture/health-monitoring.md](docs/architecture/health-monitoring.md) | Health model, thresholds, state machine |
| [docs/architecture/configuration.md](docs/architecture/configuration.md) | Companion configuration schema |
| [docs/architecture/security.md](docs/architecture/security.md) | Threat model, privilege model, secret handling |
| [docs/architecture/failure-modes.md](docs/architecture/failure-modes.md) | Failure-mode and mitigation analysis |
| [docs/architecture/android-integration.md](docs/architecture/android-integration.md) | Android integration strategy |
| [docs/architecture/decisions.md](docs/architecture/decisions.md) | Architecture decision records |
| [docs/implementation-plan.md](docs/implementation-plan.md) | Dependency-ordered milestones |
| [docs/release-plan.md](docs/release-plan.md) | CI/CD and release architecture |
| [docs/roadmap.md](docs/roadmap.md) | Milestone timeline and non-goals |

Diagrams are Mermaid sources in [`design/diagrams/`](design/diagrams/).

## Scope

**In scope (MVP):** one WireGuard interface, one peer identity, N endpoint candidates,
automatic failover and failback on Linux, then Windows, then Android.

**Explicitly out of scope:** switching between WireGuard servers that have *different*
public keys; multi-hop or relay topologies; NAT traversal / hole punching; any change to
WireGuard cryptography.

## Platform status

| Platform | Status |
|---|---|
| Linux | Planned — design complete, implementation not started |
| Windows | Planned — design complete, feasibility validated at source level, not built |
| Android | Planned — requires patching the official `tunnel` library; not built |

No platform is supported until it is implemented, tested and released.

## License

Apache License 2.0 — see [LICENSE](LICENSE) and the rationale in
[docs/architecture/decisions.md](docs/architecture/decisions.md#adr-008-license-apache-20).

WireGuard is a registered trademark of Jason A. Donenfeld. This project is not affiliated
with or endorsed by the WireGuard project.
