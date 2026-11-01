# Architecture decision records

Numbered decisions with evidence. A decision is reversed only by a new record that names the
one it supersedes and states what evidence changed.

| ADR | Decision | Status |
|---|---|---|
| [001](#adr-001-companion-configuration-not-extended-wireguard-config) | Companion configuration, not extended WireGuard config | Accepted |
| [002](#adr-002-shared-go-core-with-platform-adapters) | Shared Go core with platform adapters | Accepted |
| [003](#adr-003-commit-probe-observe-no-independent-prober) | Commit, probe, observe — no independent prober | Accepted |
| [004](#adr-004-layered-health-model-with-an-inconclusive-state) | Layered health model with an `INCONCLUSIVE` state | Accepted |
| [005](#adr-005-wgctrl-go-as-the-control-layer-for-linux-and-windows) | `wgctrl-go` as the control layer for Linux and Windows | Accepted |
| [006](#adr-006-no-key-material-enforced-structurally) | No key material, enforced structurally | Accepted |
| [007](#adr-007-adopt-roaming-dont-fight-it) | Adopt roaming, don't fight it | Accepted |
| [008](#adr-008-license-apache-20) | License (Apache-2.0) | **Needs your confirmation** |
| [009](#adr-009-the-project-does-not-manage-tunnel-lifecycle) | The project does not manage tunnel lifecycle | Accepted |
| [010](#adr-010-platform-specific-default-parameters) | Platform-specific default parameters | Accepted |
| [011](#adr-011-no-active-endpoint-discovery) | No active endpoint discovery | Accepted |

---

## ADR-001: Companion configuration, not extended WireGuard config

**Status:** Accepted

**Decision.** Endpoint candidates live in a separate TOML file. The WireGuard configuration
file is never read, never written, never parsed.

**Evidence.** `[Peer]` sections with unknown keys are rejected by `wg setconf` and `wg-quick`,
and `wg-quick strip` forwards them to `wg setconf`. A `Endpoints =` list would therefore make
the file invalid for every existing WireGuard client. WireGuard's data model has exactly one
`WGPEER_A_ENDPOINT` attribute and no list type
([upstream §1](../research/upstream-analysis.md#1-a-peer-has-exactly-one-endpoint)).

Reading the WireGuard config file would mean reading a private key, which violates ADR-006.
The engine obtains peer facts from the live device instead.

**Consequences.** Two files to keep in sync, joined only by interface name and peer public key.
The companion file is optional and additive: deleting it leaves a perfectly valid WireGuard
configuration.

Full schema: [configuration.md](configuration.md).

## ADR-002: Shared Go core with platform adapters

**Status:** Accepted

**Decision.** One Go module with a platform-independent core (policy, state machine, DNS,
validation, status) and one thin adapter per platform. Android runs the same core inside the
app process.

**Evidence.** `wgctrl-go` exposes the identical `ConfigureDevice` call against the identical
`wgtypes` model on Linux (netlink) and Windows (WireGuardNT `DeviceIoControl` with
`PeerHasEndpoint | PeerUpdateOnly`)
([overview §3](overview.md#3-evaluation)). Android has no external control surface at all and
requires in-process integration
([platform §4](../research/platform-capabilities.md#4-cross-platform-conclusion)), so a
binding is unavoidable there.

**Why not native per platform (Option B).** Duplicating the hysteresis and backoff logic in
Kotlin creates the largest correctness risk in the project: two implementations of the
failure/recovery decision that will diverge, and the divergence will surface as
"failover behaves differently on my phone" — expensive to diagnose and impossible to test
centrally.

**Why not shared-core-with-bindings everywhere (Option C).** Go runs natively on Linux and
Windows; a `gomobile`/FFI layer there would add a build and runtime boundary for no gain.

**Consequences.** Three adapters, each ~200–400 lines and policy-free. Android requires the
patch in [android-integration.md](android-integration.md).

## ADR-003: Commit-probe-observe, no independent prober

**Status:** Accepted

**Decision.** The engine never probes a candidate before selecting it. It commits the
endpoint, stimulates the tunnel, observes the result, and rolls back if wrong.

**Evidence.** Three independent blockers:

1. A WireGuard handshake initiation is constructed from the sender's **static private key**
   (`handshake.precomputedStaticStatic = device.staticIdentity.privateKey.sharedSecret(pk)`,
   [`device/peer.go`](https://github.com/WireGuard/wireguard-go/blob/master/device/peer.go)).
   An external prober cannot elicit a handshake response without it.
2. The listen port is owned by the implementation, so a second socket's responses go nowhere
   useful.
3. There is no UDP reachability primitive; silence is indistinguishable from filtering.

**Consequences.** The state machine has a `Probation` state, and failback is an *attempt* with
rollback rather than a measurement. This is the cost of holding no keys, and it is stated in
user-facing documentation rather than hidden.

**Rejected alternative:** synthesising handshake initiations with a retained key. It would
require storing the peer's private key, which violates ADR-006 and would make the daemon a
high-value target.

## ADR-004: Layered health model with an `INCONCLUSIVE` state

**Status:** Accepted

**Decision.** Health is evaluated in layers (L0 name, L1 reachability, L2 handshake, L3 tunnel,
L4 application). Only L2 and above can cause failover. The evaluation result set includes
`INCONCLUSIVE`, which does not count toward `failure_threshold`.

**Evidence.** WireGuard "tries to be as silent as possible when not being used"
([Quick Start](https://www.wireguard.com/quickstart/)), so an old `last_handshake` is normal
on an idle tunnel. Protocol constants: `RejectAfterTime = 180 s`, `RekeyAttemptTime = 90 s`,
`RekeyTimeout = 5 s`
([`device/constants.go`](https://github.com/WireGuard/wireguard-go/blob/master/device/constants.go)).
Upstream's own `reresolve-dns` uses a 135 s heuristic.

**Why `INCONCLUSIVE` matters.** While transport keys are still valid, a broken path and a peer
that simply does not answer a probe look identical. Declaring failure would produce false
failovers on any network that filters ICMP — the flapping the project is required to avoid.
Failure becomes *provable* only once the session has expired.

**Consequences.** Honest detection-latency bounds: immediate in `target` mode, up to ~195 s in
`keepalive` mode, unbounded in `passive` mode. Sub-second failover is not achievable without
holding keys, and is not promised anywhere.

## ADR-005: `wgctrl-go` as the control layer for Linux and Windows

**Status:** Accepted, with a validation task in milestone 4

**Decision.** Use `golang.zx2c4.com/wireguard/wgctrl` for both platforms. Do not shell out to
`wg(8)` and do not call `wireguard.dll` directly.

**Evidence.** On Linux it speaks netlink (`internal/wglinux`) and encodes only non-nil
fields, which is what makes an endpoint-only update structurally safe
([platform §1](../research/platform-capabilities.md#1-linux)). On Windows it enumerates
WireGuardNT adapters via SetupAPI and issues configuration IOCTLs
(`internal/wgwindows`), so no dependency on the WireGuard for Windows service or its private
gob IPC exists ([platform §2](../research/platform-capabilities.md#2-windows)).

**Risk.** `wgctrl-go` has no Windows CI job upstream (its workflows cover Linux and static
analysis). The Windows client is source-correct but not continuously exercised. Mitigated by
keeping the adapter thin and validating it ourselves in milestone 4.

**Rejected alternative:** `wg(8)` subprocess. It makes the public key a command-line argument
(visible in `/proc/<pid>/cmdline` and audit logs), returns coarse exit codes, and depends on a
binary being present.

## ADR-006: No key material, enforced structurally

**Status:** Accepted

**Decision.** The daemon never reads, stores, transmits or logs a private key or preshared key.
This is enforced by types and by tests, not only by review.

**Evidence.** The Linux netlink API and the WireGuardNT IOCTL both support partial updates
(`WGPEER_F_UPDATE_ONLY` / `PeerUpdateOnly`), so an endpoint change genuinely requires no key
material ([upstream §4](../research/upstream-analysis.md#4-updating-an-endpoint-is-peer-scoped-and-non-destructive)).

**How it is enforced.**

- The adapter interface exposes `Read() (PeerState, error)` where `PeerState` has no key field,
  and `SetEndpoint(iface, peer, addr)`. There is no function that *could* write a key.
- `wgtypes.Device` contains `PrivateKey` and `Peer.PresharedKey`. Status is a separate
  projection type, and a test fails the build if `wgtypes.Device` is ever embedded in a status
  or log structure.
- The configuration validator rejects `private_key`, `preshared_key` and `psk` anywhere in the
  document, with an explanatory message.
- The daemon never calls `wg showconf`, which would return the private key.

**Consequences.** Compromise of the daemon yields a traffic-redirection and denial-of-service
primitive, never a decryption or impersonation one. That distinction determines advisory
severity.

## ADR-007: Adopt roaming, don't fight it

**Status:** Accepted

**Decision.** When WireGuard's observed endpoint diverges from our desired endpoint, adopt a
known candidate rather than re-asserting. Re-assert only for unknown observed endpoints, and
only after a long delay.

**Evidence.** `wg(8)` documents that the endpoint "will be updated automatically to the most
recent source IP address and port of correctly authenticated packets from the peer", and
`wgtypes.Peer.Endpoint` is described as "the most recent source address used for
communication"
([upstream §5](../research/upstream-analysis.md#5-roaming-silently-overwrites-a-manually-set-endpoint)).

**Why adopt.** Availability beats preference. If the peer is reachable at a different address,
re-asserting our preferred one breaks a working tunnel to satisfy policy.

**Why re-assert at all.** Because the endpoint is operator-controlled policy, not an automatic
consequence — but the delay defaults to 300 s, so legitimate mobility is never fought.

**Security note.** Roaming only happens on *correctly authenticated* packets, so an unknown
observed endpoint is not evidence of an attack. It means the peer genuinely sent from there.

## ADR-008: License (Apache-2.0)

**Status:** Accepted — **confirmation requested from the project owner**

**Context.** No single license preference can be established for the owner. The owner's
public repositories use MIT (`ScaleTail`) and Apache-2.0 (`yolo-tflite-android`), with several
repositories unlicensed. There is no organisation-level policy to inherit.

Upstream components differ, and their licences constrain the choice:

| Component | License | Can we copy code? |
|---|---|---|
| `wireguard-tools` | GPL-2.0 | **No** — would force the whole project to GPL-2.0 |
| `wireguard-linux` (kernel) | GPL-2.0 | No |
| `wireguard-go` | MIT | Yes, with notice preserved |
| `wireguard-windows` | MIT | Yes, with notice preserved |
| `wgctrl-go` | MIT | Yes — it is a direct dependency |
| `wireguard-android` | Apache-2.0 | Yes, with notice preserved |
| wgrtc | Apache-2.0 | Yes, with notice preserved |
| PrivyCS | GPL-3.0 | **No** |

**Decision.** Apache License 2.0, with the following reasoning:

1. The Android integration patches an **Apache-2.0** component. MIT code can be included in an
   Apache-2.0 work with notices preserved; the reverse direction is not clean, because
   Apache-2.0 code cannot be re-licensed as MIT. Apache-2.0 therefore keeps the Android patch
   legally straightforward, including if it is proposed upstream.
2. Apache-2.0 carries an explicit patent grant, which is worth having in a networking and
   security tool.
3. It matches one of the two licences the owner already uses.
4. All MIT dependencies (`wgctrl-go`, `wireguard-go`) remain includable.

**What this explicitly is not.** It is not an inheritance of WireGuard's GPL-2.0 licensing,
and no WireGuard code is copied. `reresolve-dns`'s behaviour is reimplemented from documented
semantics, never transliterated.

**Consequences.** Every source file carries `SPDX-License-Identifier: Apache-2.0`. Contributing
means licensing under these terms.

**If the owner prefers MIT instead:** it is workable for Linux and Windows (all MIT
dependencies), but the Android patch would then need to stay a separate Apache-2.0-licensed
patch file applied at build time rather than code in this repository. This is a real, if
minor, ongoing cost, and it is the reason Apache-2.0 is recommended rather than MIT.

## ADR-009: The project does not manage tunnel lifecycle

**Status:** Accepted

**Decision.** The daemon attaches to an existing interface and an existing peer. It does not
create interfaces, set addresses, add routes, start or stop tunnels, or install WireGuard
itself.

**Evidence.** `wg-quick@.service` already owns lifecycle on Linux
([`src/systemd/wg-quick@.service`](https://github.com/WireGuard/wireguard-tools/blob/master/src/systemd/wg-quick%40.service)),
and WireGuard for Windows already owns it on Windows. Competing with either would make the
project unusable alongside the standard tooling.

**Consequences.** The daemon is composable with `wg-quick`, NetworkManager, systemd units, and
the Windows service. It reports `Halted` when the interface or peer is missing rather than
trying to create either. A future `up`/`down` mode is possible but would be a separate,
explicitly requested feature.

## ADR-010: Platform-specific default parameters

**Status:** Accepted

**Decision.** The schema is identical across platforms; compiled-in defaults differ. Android
uses materially slower intervals.

**Evidence.** Android checks are battery-sensitive, timers are subject to Doze, and a `VpnService`
already keeps the radio alive. wgrtc's Android side faces the same trade-off
([existing solutions §3](../research/existing-solutions.md#3-wgrtc--closest-third-party-project)).

| Parameter | Desktop default | Android default |
|---|---|---|
| `check_interval` | 5 s | 30 s |
| `failback_interval` | 60 s | 300 s |
| `dns.refresh_interval` | 60 s | 300 s |
| `probe.allow_keepalive` | true | false |

**Consequences.** A configuration copied between platforms behaves correctly but with
different timing. Documented in [configuration.md §6](configuration.md#6-defaults), and
printed by `wgefctl validate` so an operator can see which defaults are in effect.

## ADR-011: No active endpoint discovery

**Status:** Accepted

**Decision.** Candidates are operator-supplied. The project performs no STUN, no broker, no
signalling and no NAT traversal.

**Evidence.** The requirement excludes mandatory relay servers. wgrtc implements broker-based
discovery and its dependency on a WebSocket broker is precisely what makes it a different
product ([existing solutions §3](../research/existing-solutions.md#3-wgrtc--closest-third-party-project)).

**Consequences.** If all candidates become unreachable because the server's address changed in
a way not reflected in DNS, this project cannot recover — it has nothing to try. That is a
stated limitation, not a bug. It is the reason `docs/roadmap.md` lists "operator hook on
all-candidates-failed" as the escape hatch.

**Rejected alternative:** automatic endpoint discovery via STUN, which would add a third-party
dependency, a privacy cost (a third party learns the endpoint), and a failure mode the project
is designed to avoid.
